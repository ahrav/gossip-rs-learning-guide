# Crash Math -- The Persistence Problem Space

*A worker in shard 0x3A begins scanning a page of 10,000 items under tenant `T-0018` and policy hash `ab91c4...`. By item 9,500, the worker has persisted 237 findings across three layers (findings, occurrences, observations) and flushed 9,500 done-ledger rows marking each object-version as `ScannedClean` or `ScannedWithFindings`. The cursor checkpoint sits at item 9,000 -- the last full flush boundary. At item 9,501, the process takes a `SIGKILL`. On restart, the coordinator resumes from the checkpointed cursor at item 9,000. Items 9,001 through 9,500 have durable done-ledger entries and durable findings, but the cursor does not reflect that work. Items 9,501 through 10,000 have neither. The replacement worker must re-scan 1,000 items, and the persistence layer must ensure that replaying findings for items 9,001-9,500 does not create duplicate rows, that re-writing done-ledger entries for already-scanned items does not downgrade their status, and that the 500 items beyond the crash point are correctly recognized as "never scanned."*

*Without a done-ledger, the replacement worker has no way to distinguish "scanned and clean" from "never scanned." It rescans all 10,000 items. Without idempotent upsert semantics, the 237 findings become 474. Without a monotonic status lattice, a retried `FailedRetryable` write for item 9,100 could overwrite its existing `ScannedWithFindings` status, erasing evidence that the item contains a secret. The persistence boundary exists to make all three failure modes impossible.*

---

The failure scenario above distills into a single architectural question: how does a crash-recovering system distinguish completed work from incomplete work, without losing data and without duplicating data? The answer requires three cooperating mechanisms -- a done-ledger for deduplication, a findings sink for idempotent secret persistence, and a commit protocol that orders writes so crash recovery always sees a consistent prefix.

Gossip-rs implements these mechanisms across multiple crates and two distinct trait surfaces. This chapter maps the full persistence problem space: the two interfaces, the three record layers, the commit ordering protocol, and the conformance harness that proves backends implement the contracts correctly.

## Two Persistence Surfaces

Gossip-rs separates persistence into two trait surfaces at different abstraction levels. The split is not accidental -- each surface serves a different caller, operates at a different granularity, and enforces different invariants.

### CommitSink: The Scan-Time Interface

The `CommitSink` trait lives in `gossip-scanner-runtime/src/commit_sink.rs` and defines the per-item lifecycle that scan drivers call during execution. Drivers know nothing about done-ledgers, findings sinks, or commit ordering -- they call three methods in sequence for each scanned item.

From `gossip-scanner-runtime/src/commit_sink.rs`:

```rust
pub trait CommitSink: Send + Sync {
    /// Open a new item transaction. Must be called before any findings
    /// are upserted for this key.
    fn begin_item(&self, item_key: &ItemKey, meta: &ItemMeta) -> Result<()>;

    /// Record one batch of findings for an in-progress item. May be
    /// called multiple times if findings arrive incrementally.
    fn upsert_findings(&self, item_key: &ItemKey, batch: &FindingsBatch) -> Result<()>;

    /// Close the item transaction. After this call, no further findings
    /// may be upserted for this key within the current scan.
    fn finish_item(&self, item_key: &ItemKey) -> Result<()>;
}
```

The protocol is strict: `begin_item` before any `upsert_findings`, then `finish_item` to close. A scan driver for an S3 connector or a git pack walker calls these methods and never touches persistence internals.

The `FindingRecord` at this level is deliberately compact -- five fields carrying only the data needed for downstream identity derivation:

From `gossip-scanner-runtime/src/commit_sink.rs`:

```rust
pub struct FindingRecord {
    pub rule_id: u32,
    pub start: u64,
    pub end: u64,
    pub norm_hash: [u8; 32],
    pub confidence_score: i8,
}
```

**Rationale:** Keeping `CommitSink` simple serves two purposes. First, scan drivers should not need to understand tenant scoping, policy hashing, or identity chain derivation -- those are runtime concerns. Second, `CliNoOpCommitSink` provides a zero-cost test double for CLI mode where findings flow only through the event stream:

From `gossip-scanner-runtime/src/commit_sink.rs`:

```rust
/// No-op sink used by CLI and direct-mode scans.
pub struct CliNoOpCommitSink;

impl CommitSink for CliNoOpCommitSink {
    fn begin_item(&self, _item_key: &ItemKey, _meta: &ItemMeta) -> Result<()> {
        Ok(())
    }
    fn upsert_findings(&self, _item_key: &ItemKey, _batch: &FindingsBatch) -> Result<()> {
        Ok(())
    }
    fn finish_item(&self, _item_key: &ItemKey) -> Result<()> {
        Ok(())
    }
}
```

### Persistence Traits: The Storage-Time Interface

The `gossip-contracts/src/persistence/` module defines the durable storage contracts that backend implementations compile against. Two traits form the core surface:

**`DoneLedger`** -- deduplication tracking for scanned object-versions. From `done_ledger.rs`:

```rust
pub trait DoneLedger: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    type CommitHandle: CommitHandle<Receipt = DoneLedgerCommitReceipt, Error = Self::Error>;

    fn batch_get(
        &self,
        tenant_id: TenantId,
        policy_hash: PolicyHash,
        ovid_hashes: &[OvidHash],
    ) -> Result<Vec<Option<DoneLedgerRecord>>, Self::Error>;

    fn batch_upsert(
        &self,
        records: &[DoneLedgerRecord],
    ) -> Result<Self::CommitHandle, Self::Error>;
}
```

**`FindingsSink`** -- idempotent persistence for the three-layer findings model. From `findings.rs`:

```rust
pub trait FindingsSink: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    type CommitHandle: CommitHandle<Receipt = FindingsCommitReceipt, Error = Self::Error>;

    fn upsert_batch(
        &self,
        batch: FindingsUpsertBatch<'_>,
    ) -> Result<Self::CommitHandle, Self::Error>;
}
```

Both traits share a critical design property: they separate *submission* from *durability*. Returning `Ok(handle)` means the backend accepted responsibility for the write. Durability is established only when `handle.wait()` returns a receipt. This two-phase design lets backends batch or defer fsync work without weakening the caller-visible contract.

**Rationale:** The associated `CommitHandle` type is consumed by `wait()`, preventing double-waits and making the single-use nature of durable acknowledgement explicit in the type system. From `commit.rs`:

```rust
#[must_use = "durability is not established until wait() returns a receipt"]
pub trait CommitHandle: Send + 'static {
    type Receipt: CommitReceipt;
    type Error: Error + Send + Sync + 'static;

    fn wait(self) -> Result<Self::Receipt, Self::Error>;
}
```

### How the Two Surfaces Connect

The runtime bridges `CommitSink` to the persistence traits through the receipt-driven execution pipeline in `gossip-scanner-runtime`. Distributed execution routes the `CommitSink` lifecycle into the receipt-driven commit pipeline (`ReceiptCommitSink` feeding `ResultCommitter`), while direct CLI scans use `CliNoOpCommitSink`. When a scan driver calls `begin_item` / `upsert_findings` / `finish_item`, the durable pipeline derives the full identity chain (`norm_hash` -> `secret_hash` -> `finding_id` -> `occurrence_id`), constructs the three-layer findings records, and flushes them through `FindingsSink` and `DoneLedger`.

The key transformation: `commit_sink::FindingRecord` (five flat fields, no tenant context) becomes `FindingRecord` + `OccurrenceRecord` + `ObservationRecord` (fully scoped, content-addressed, referentially closed) before reaching the persistence traits.

**Rationale:** The two-surface split exists because scan drivers and persistence backends have fundamentally different concerns. A git pack walker cares about byte offsets and rule matches. A SQLite backend cares about referential integrity and monotonic merge semantics. Forcing both into a single trait would either expose backend details to drivers (leaking abstraction upward) or hide persistence invariants from backends (weakening contracts downward). The receipt-driven commit pipeline sits at the boundary where domain knowledge from both sides converges.

## Architecture: The Full Persistence Stack

```
┌───────────────────────────────────────────────────────────┐
│                      Scan Driver                          │
│   (S3, Git, Filesystem -- source-specific scan loop)      │
└──────────────────────────┬────────────────────────────────┘
                           │  begin_item / upsert_findings / finish_item
                           ▼
┌───────────────────────────────────────────────────────────┐
│          CommitSink  (gossip-scanner-runtime)              │
│                                                           │
│  ┌─────────────────┐          ┌────────────────────────┐  │
│  │CliNoOpCommitSink│          │  ReceiptCommitSink     │  │
│  │ (CLI mode)      │          │  (distributed mode)    │  │
│  └─────────────────┘          └───────────┬────────────┘  │
└───────────────────────────────────────────┼───────────────┘
                                            │  identity derivation
                                            │  norm_hash → secret_hash → finding_id → occurrence_id
                                            ▼
┌───────────────────────────────────────────────────────────┐
│          Persistence Traits  (gossip-contracts)            │
│                                                           │
│  ┌────────────────────────┐  ┌─────────────────────────┐  │
│  │  FindingsSink trait    │  │  DoneLedger trait        │  │
│  │  upsert_batch()        │  │  batch_get()             │  │
│  │  → FindingsCommitReceipt│ │  batch_upsert()          │  │
│  │                        │  │  → DoneLedgerCommitReceipt│ │
│  └────────────┬───────────┘  └──────────┬──────────────┘  │
└───────────────┼─────────────────────────┼─────────────────┘
                │                         │
                ▼                         ▼
┌───────────────────────────────────────────────────────────┐
│              Concrete Backends                             │
│  (In-memory test doubles, SQLite, PostgreSQL)             │
└───────────────────────────────────────────────────────────┘
```

Every arrow represents a trait boundary. Scan drivers depend only on `CommitSink`. The runtime depends on `CommitSink` + persistence traits. Backends depend only on persistence traits. No layer reaches through to another's internals.

## The Three Record Layers

The findings data model uses three normalized layers with different stability and scoping properties. This normalization is not arbitrary -- each layer answers a different question about a detected secret.

### Layer 1: FindingRecord -- "What Was Found Where"

From `findings.rs`:

```rust
pub struct FindingRecord {
    tenant_id: TenantId,
    finding_id: FindingId,
    stable_item_id: StableItemId,
    rule_fingerprint: RuleFingerprint,
    secret_hash: SecretHash,
}
```

A `FindingRecord` is keyed by `(tenant, stable_item_id, rule_fingerprint, secret_hash)`. The `FindingId` is content-addressed -- derived from those four fields via BLAKE3, so identical inputs always produce the same identity without coordination. Construction via `FindingRecord::new` derives the ID internally, and `verify_id()` lets callers confirm the stored ID matches a fresh derivation.

**Rationale:** Findings are version-independent and policy-independent. The same secret detected by the same rule in the same logical item produces the same `FindingRecord` regardless of which object version or scan policy surfaced it. This means triage decisions (false positive, accepted risk) survive rescans: the `FindingId` is stable across version changes. The `secret_hash` field is a one-way keyed hash -- raw secret bytes never appear in any persistence record. The `SecretHash` newtype's `Debug` implementation is redacted, preventing accidental exposure in structured logs or error messages.

### Layer 2: OccurrenceRecord -- "In Which Version, at What Offset"

From `findings.rs`:

```rust
pub struct OccurrenceRecord {
    tenant_id: TenantId,
    occurrence_id: OccurrenceId,
    finding_id: FindingId,
    object_version_id: ObjectVersionId,
    byte_offset: u64,
    byte_length: NonZeroU64,
}
```

An occurrence pins a finding to a specific byte range `[byte_offset, byte_offset + byte_length)` within a specific object version. The same finding can have multiple occurrences across different versions of the same item, or at different offsets within the same version.

**Rationale:** Separating occurrences from findings means a secret that moves from line 50 to line 80 across versions creates a new occurrence but not a new finding. The finding identity (and any triage state attached to it) remains stable. The `NonZeroU64` constraint on `byte_length` is enforced at construction -- a zero-length span is meaningless for a secret occurrence.

### Layer 3: ObservationRecord -- "When and How It Was Seen"

From `findings.rs`:

```rust
pub struct ObservationRecord {
    tenant_id: TenantId,
    observation_id: ObservationId,
    occurrence_id: OccurrenceId,
    policy_hash: PolicyHash,
    ovid_hash: OvidHash,
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    seen_at: LogicalTime,
    location: Option<Arc<Location>>,
}
```

An observation records that a particular occurrence was seen during a specific scan run, under a specific policy, in a specific shard and fence epoch. The `ObservationId` is derived from `(tenant_id, policy_hash, occurrence_id)`. The `ObservationRecord::new` constructor derives the ID canonically; `ObservationRecord::from_persisted` accepts a stored ID and rejects it if it does not match the canonical derivation, catching data corruption or migration errors at read time.

**Rationale:** `Location` metadata (display path and optional URL) lives on the observation rather than the finding or occurrence because it is presentation context tied to the specific policy and run. Different policies scanning the same item may present different location strings (an S3 policy shows `s3://bucket/key`, a filesystem policy shows `/mnt/data/file.txt`). Observations also carry the `fence_epoch`, enabling backends to detect and reject writes from stale workers whose lease has expired. The builder pattern `with_location(location)` keeps `Location` optional without adding constructor complexity.

### Referential Integrity

The three layers form a strict hierarchy: each occurrence references a finding; each observation references an occurrence. The `FindingsUpsertBatch` type bundles all three layers into a single submission unit:

From `findings.rs`:

```rust
pub struct FindingsUpsertBatch<'a> {
    findings: &'a [FindingRecord],
    occurrences: &'a [OccurrenceRecord],
    observations: &'a [ObservationRecord],
}
```

The batch provides `validate_referential_integrity()` to verify that every occurrence references a finding in the batch, every observation references an occurrence in the batch, and all records share the same tenant. A separate `validate_observation_identity()` check confirms that every observation's stored `observation_id` matches the canonical derivation. Backends must enforce referential integrity -- either through foreign keys or by accepting referential closure within a single batch.

**Rationale:** The batch borrows slices rather than owning them, so callers retain control over allocation and can reuse buffers across flushes. Both traits recommend keeping batches at or below `RECOMMENDED_MAX_BATCH_SIZE` (10,000 records), defined in `persistence/mod.rs`.

## Object-Version Identity: OvidHash

Before the done-ledger can track whether an object-version has been scanned, it needs a stable hash of the object-version identity. `OvidHash` serves this role.

From `ovid.rs`:

```rust
pub struct OvidHashInputs {
    pub stable_item_id: StableItemId,
    pub version: VersionId,
}

pub fn derive_ovid_hash(inputs: &OvidHashInputs) -> OvidHash {
    OvidHash::from_bytes(derive_from_cached(&OVID_HASHER, inputs))
}
```

The hash is derived from the stable item identity plus the version claim (strong or weak). The same stable item under two different version claims produces different `OvidHash` values -- a strong version `v3` and a weak version `v3` for the same item are distinct entries in the done-ledger.

**Rationale:** The `VersionId` enum distinguishes `Strong` (content-hash, ETag with strong semantics) from `Weak` (timestamp-derived, monotonic counter). A strong version claim means "these exact bytes"; a weak claim means "this is probably the same content." Keeping them separate in the hash prevents a weak claim from satisfying a strong claim's deduplication entry.

## The Done-Ledger Lattice

The done-ledger records whether a specific object-version has been processed under a given tenant and scan policy. Its primary purpose is at-most-once scan semantics: before scanning an object, the pipeline checks the done-ledger to avoid redundant work. Each row is keyed by `DoneLedgerKey`:

From `done_ledger.rs`:

```rust
pub struct DoneLedgerKey {
    tenant_id: TenantId,
    policy_hash: PolicyHash,
    ovid_hash: OvidHash,
}
```

This triple ensures that the same object scanned under different policies produces separate entries, and the same policy applied to different object versions produces separate entries. The `DoneLedgerKey` implements `CanonicalBytes` for content-addressed identity derivation, writing all three fields in a fixed, unambiguous order.

The status of each entry follows a join-semilattice defined by `DoneLedgerStatus`:

From `done_ledger.rs`:

```rust
#[repr(u8)]
pub enum DoneLedgerStatus {
    FailedRetryable = 1,
    FailedPermanent = 2,
    Skipped = 3,
    ScannedClean = 10,
    ScannedWithFindings = 11,
}
```

The merge rule is `max(self.rank(), other.rank())`. This guarantees three properties:

1. **Idempotence** -- merging a value with itself is a no-op.
2. **Commutativity** -- `a.merge(b) == b.merge(a)`.
3. **Monotonicity** -- once an object reaches a scanned state (rank 10 or 11), no failure write (rank 1, 2, or 3) can downgrade it.

The intentional gap between `Skipped` (rank 3) and `ScannedClean` (rank 10) reserves discriminant space for future non-terminal states. Backends store the rank as a `u8`, so adding a variant between 3 and 10 is a backwards-compatible schema change.

**Rationale:** The lattice property is what makes the crash scenario from the opening solvable. When the replacement worker re-scans items 9,001-9,500 and writes `ScannedClean` for an item that already has `ScannedWithFindings`, the merge yields `ScannedWithFindings` (rank 11 > rank 10). The higher-ranked status always wins, regardless of write order. The `DoneLedger` trait mandates this merge behavior in its implementor contract: backends must compare incoming and existing statuses via `DoneLedgerStatus::merge` and persist only the dominant value.

Each `DoneLedgerRecord` also carries a `DoneLedgerProvenance` struct recording which worker, shard, and time window produced the entry:

From `done_ledger.rs`:

```rust
pub struct DoneLedgerProvenance {
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    started_at: LogicalTime,
    finished_at: LogicalTime,
}
```

Provenance is not part of the deduplication key -- it is write-side metadata used for debugging, auditing, and stale-entry diagnostics. When a higher-ranked status overwrites an existing row, the provenance is replaced along with it. The `fence_epoch` ties the write to a specific lease epoch, enabling backends to detect and reject writes from workers whose lease has been revoked. Chapter 2 covers the done-ledger in depth.

## The Commit Protocol: PageCommit Typestate

A page's data must become durable in a strict order so that crash recovery never observes a checkpointed cursor without the corresponding findings and done-ledger rows already on disk. The `PageCommit<S>` typestate machine enforces this ordering at compile time.

The four states:

```
AwaitingFindings ──record_findings──▸ FindingsDurable
                                         │
                             record_done_ledger (validates item count)
                                         │
                                         ▾
                                     ItemDurable
                                         │
                             record_checkpoint (validates full scope)
                                         │
                                         ▾
                                  CheckpointDurable
```

`CheckpointBoundary` is a family-neutral enum that tags each checkpoint with its source family:

From `page_commit.rs`:

```rust
pub enum CheckpointBoundaryKind {
    /// Ordered-content item frontier.
    OrderedContent,
    /// Repo-frontier progress (e.g., Git repos).
    RepoFrontier,
}

pub enum CheckpointBoundary {
    /// Ordered-content checkpoint boundary.
    OrderedContent(Cursor),
    /// Repo-frontier checkpoint boundary.
    RepoFrontier(Cursor),
}
```

Both variants wrap a [`Cursor`] because ordered-content and repo-frontier source families both reduce to a monotonic frontier position plus optional opaque token. The enum tag carries the semantic domain so runtime code cannot accidentally mix them.

From `page_commit.rs`:

```rust
#[must_use = "page commits must be driven to a durable receipt or explicitly dropped"]
pub struct PageCommit<S> {
    scope: Arc<CommitScope>,
    state: S,
}
```

Each state is a separate type (`AwaitingFindings`, `FindingsDurable`, `ItemDurable`, `CheckpointDurable`), and transition methods exist only on the correct state. Calling `record_done_ledger` on a `PageCommit<AwaitingFindings>` is a compile error, not a runtime panic.

Each transition offers two advancement paths: `record_*` (caller already holds the receipt) and `wait_*` (caller passes a `CommitHandle`; the method waits and validates). The `CommitScope` captures the immutable context for one page commit -- tenant, run, shard, fence epoch, item count, and cursor:

From `page_commit.rs`:

```rust
pub struct CommitScope {
    tenant_id: TenantId,
    policy_hash: PolicyHash,
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    committed_units: NonZeroU64,
    checkpoint_boundary: CheckpointBoundary,
}
```

The done-ledger transition validates that the receipt's `record_count` matches `scope.committed_units()`. The checkpoint transition validates that the receipt's embedded `CommitScope` matches the page's scope exactly. These validations catch receipt mix-ups between concurrent pages at the earliest possible point.

**Rationale:** The alternative -- a runtime state enum with panics on invalid transitions -- trades compile-time safety for simpler generics. Given that incorrect ordering is a data-loss bug (findings committed after a cursor advance that is already visible to restart logic), compile-time enforcement is worth the generics cost. The `wait_*` methods consume the state machine on failure: after a backend wait error, the page's I/O state is unknown, so the only safe action is to reconstruct a new `PageCommit` and retry from `AwaitingFindings`. Chapter 4 covers the typestate machine in depth.

## Receipt Hierarchy

The commit protocol produces a hierarchy of receipts proving what became durable at each stage:

| Receipt | Proves | Contents |
|---------|--------|----------|
| `FindingsCommitReceipt` | Three-layer findings data is durable | `finding_count`, `occurrence_count`, `observation_count` |
| `DoneLedgerCommitReceipt` | Done-ledger rows are durable | `record_count`, `scanned_count`, `findings_count` |
| `CheckpointCommitReceipt` | Cursor checkpoint is durable | `CommitScope`, `checkpointed_at` |
| `ItemCommitReceipt` | Composite: findings + done-ledger | Contains both sub-receipts |
| `PageCommitReceipt` | Terminal: entire page is durable | `ItemCommitReceipt` + `CheckpointCommitReceipt` |

Holding a `PageCommitReceipt` is sufficient proof that the cursor can be safely advanced -- no further persistence work is required for the page. For synchronous backends and test doubles, `ReadyCommitHandle` wraps a pre-computed `Result<R, E>` so `wait()` returns immediately without blocking:

From `commit.rs`:

```rust
pub struct ReadyCommitHandle<R, E>(Result<R, E>);

impl<R, E> CommitHandle for ReadyCommitHandle<R, E>
where
    R: CommitReceipt,
    E: Error + Send + Sync + 'static,
{
    type Receipt = R;
    type Error = E;

    fn wait(self) -> Result<Self::Receipt, Self::Error> {
        self.0
    }
}
```

**Rationale:** Receipt counts in `FindingsCommitReceipt` and `DoneLedgerCommitReceipt` reflect the number of distinct records the backend has durably acknowledged, not the number of newly inserted rows. An idempotent replay of an already-durable batch returns the same counts as the original write. This makes receipts suitable as progress counters without requiring backends to distinguish "new insert" from "idempotent no-op."

## Input Validation: PersistenceInputError

The persistence boundary enforces strict input validation on all record constructors. `PersistenceInputError` covers twelve validation failure modes:

From `error.rs`:

```rust
pub enum PersistenceInputError {
    Empty { field: &'static str },
    TooLarge { field: &'static str, size: usize, max: usize },
    InvalidByte { field: &'static str, index: usize, byte: u8 },
    ZeroSpanLength,
    ObservationIdMismatch { expected: ObservationId, actual: ObservationId },
    InconsistentFindingsCount { status: &'static str, findings_count: u32 },
    MissingErrorCode { status: &'static str },
    UnexpectedErrorCode { status: &'static str },
    OrphanedReference { child_type: &'static str, parent_type: &'static str },
    InconsistentTenant,
    ProvenanceOrdering { started_at: u64, finished_at: u64 },
    KeyMismatch { existing: Box<DoneLedgerKey>, incoming: Box<DoneLedgerKey> },
}
```

**Rationale:** Each variant catches a specific contract violation at the earliest possible point. `DoneLedgerRecord::try_new` rejects a `ScannedWithFindings` status with `findings_count == 0`, and vice versa for `ScannedClean` with `findings_count > 0`. `OccurrenceRecord::try_new` rejects zero-length byte spans. `ObservationRecord::from_persisted` rejects observation IDs that do not match the canonical derivation. The `DoneLedgerErrorCode` constructor restricts error codes to a small ASCII-safe alphabet (alphanumeric plus `' '`, `'_'`, `'-'`, `'.'`, `':'`, `'/'`), with a maximum size of 128 bytes -- raw connector output or user-supplied strings never enter this field.

## Conformance Testing

The persistence contracts ship with a reusable conformance harness in `conformance.rs` that every backend must pass. The harness executes 11 checks across three sub-suites:

**Done-ledger (4 checks):**
1. Idempotent upsert -- writing the same record twice does not alter state.
2. Fail-then-scan dominance -- `FailedRetryable` followed by `ScannedWithFindings` yields scanned.
3. Scan-then-fail dominance -- the reverse order yields the same result.
4. Batch-get positional semantics -- `result[i]` corresponds to `ovid_hashes[i]`.

**Findings (4 checks):**
1. Idempotent upsert -- replay produces zero additional durable rows.
2. Orphan occurrence rejection -- an occurrence without its parent finding is rejected.
3. Orphan observation rejection -- an observation without its parent occurrence is rejected.
4. Observation upsert merge -- replaying with a different `seen_at` does not create duplicates.

**Redaction (3 checks):**
1. `NormHash` debug output does not leak raw bytes.
2. `SecretHash` debug output does not leak raw bytes.
3. `FindingRecord` debug output does not leak raw secret bytes.

The harness entry point:

From `conformance.rs`:

```rust
pub fn run_conformance<L, F>(
    done_ledger: &L,
    findings: &F,
) -> Result<PersistenceConformanceReport, PersistenceConformanceError>
where
    L: DoneLedger,
    F: FindingsSink + FindingsConformanceProbe,
```

The `FindingsConformanceProbe` trait adds a narrow read-side surface for test verification. The production `FindingsSink` API is write-only by design, so a backend that silently duplicates rows on replay still returns `Ok(())`. The probe lets the harness snapshot durable row counts before and after writes, proving replay is truly idempotent:

From `conformance.rs`:

```rust
pub trait FindingsConformanceProbe: Send + Sync {
    type Error: Error + Send + Sync + 'static;

    fn durable_counts(&self) -> Result<DurableFindingsCounts, Self::Error>;
}
```

The `DurableFindingsCounts` type mirrors the three-layer structure (`findings`, `occurrences`, `observations`), and the harness uses `saturating_sub` to compute deltas between snapshots. A first write should produce a delta of `(1, 1, 1)` for a single-item batch; a replay should produce `(0, 0, 0)`.

Each conformance test case constructs its fixtures from a distinct seed byte, producing unique `(tenant, policy, ovid)` keys. Test cases are independent of each other and of any pre-existing backend state -- they can run against a shared backend instance without cross-contamination. The `PersistenceConformanceReport` aggregates check counts across all three sub-suites, and `PersistenceConformanceError` carries structured `case` labels (formatted as `"sub-suite/test:phase"`) for precise failure diagnostics.

Chapter 6 covers the conformance harness in depth.

## Summary: Persistence Types at a Glance

| Type | Module | Purpose |
|------|--------|---------|
| `CommitSink` | `gossip-scanner-runtime` | Per-item commit lifecycle for scan drivers |
| `CliNoOpCommitSink` | `gossip-scanner-runtime` | Zero-cost CLI/test double |
| `DoneLedger` | `gossip-contracts/persistence` | Deduplication tracking trait |
| `DoneLedgerKey` | `gossip-contracts/persistence` | Composite key: `(tenant, policy, ovid)` |
| `DoneLedgerStatus` | `gossip-contracts/persistence` | Join-semilattice status enum |
| `DoneLedgerRecord` | `gossip-contracts/persistence` | Complete done-ledger row |
| `DoneLedgerProvenance` | `gossip-contracts/persistence` | Write-side audit metadata |
| `DoneLedgerErrorCode` | `gossip-contracts/persistence` | Bounded ASCII error code |
| `FindingsSink` | `gossip-contracts/persistence` | Three-layer findings persistence trait |
| `FindingRecord` | `gossip-contracts/persistence` | Layer 1: stable finding identity |
| `OccurrenceRecord` | `gossip-contracts/persistence` | Layer 2: version-specific occurrence |
| `ObservationRecord` | `gossip-contracts/persistence` | Layer 3: policy/run-scoped observation |
| `FindingsUpsertBatch` | `gossip-contracts/persistence` | Borrowed batch view over three layers |
| `OvidHash` | `gossip-contracts/persistence` | Object-version identity hash |
| `CommitHandle` | `gossip-contracts/persistence` | Two-phase durable acknowledgement trait |
| `ReadyCommitHandle` | `gossip-contracts/persistence` | Pre-resolved handle for sync backends |
| `FindingsCommitReceipt` | `gossip-contracts/persistence` | Proof of durable findings write |
| `DoneLedgerCommitReceipt` | `gossip-contracts/persistence` | Proof of durable done-ledger write |
| `CheckpointCommitReceipt` | `gossip-contracts/persistence` | Proof of durable cursor checkpoint |
| `ItemCommitReceipt` | `gossip-contracts/persistence` | Composite: findings + done-ledger |
| `PageCommitReceipt` | `gossip-contracts/persistence` | Terminal: full page durable |
| `PageCommit<S>` | `gossip-contracts/persistence` | Typestate machine for commit ordering |
| `CommitScope` | `gossip-contracts/persistence` | Immutable scope for one page commit |
| `PersistenceInputError` | `gossip-contracts/persistence` | Input validation error enum |
| `RECOMMENDED_MAX_BATCH_SIZE` | `gossip-contracts/persistence` | Batch size guidance (10,000) |
| `run_conformance` | `gossip-contracts/persistence` | Conformance harness entry point (11 checks) |

## What Comes Next

This chapter mapped the problem space and the full type inventory. The remaining chapters drill into each subsystem:

- **Chapter 2: The Done-Ledger** -- The join-semilattice in detail. `DoneLedgerStatus::merge`, the monotonicity invariant, `DoneLedgerProvenance` for audit trails, and how `batch_get` enables pre-scan deduplication checks.
- **Chapter 3: The Findings Sink** -- The three-layer data model in depth. Content-addressed identity derivation, referential integrity validation, `FindingsUpsertBatch` construction, and the write-only API surface.
- **Chapter 4: The Commit Protocol** -- `PageCommit<S>` typestate machine. Compile-time ordering enforcement, receipt validation, `wait_*` vs `record_*` transitions, and error handling when backend waits fail.
- **Chapter 5: In-Memory Test Doubles** -- Building backends that pass the conformance harness without touching disk.
- **Chapter 6: Persistence Conformance** -- The reusable conformance harness that every backend must pass.

### PostgreSQL Backends

Three crates provide PostgreSQL-backed persistence, proving the contract traits work against a real database -- not just in-memory test doubles:

- **`gossip-done-ledger-postgres`** -- PostgreSQL done-ledger backend. Implements `DoneLedger` with proper lattice-merge semantics using SQL `GREATEST` for status rank and per-field maximums.
- **`gossip-findings-postgres`** -- PostgreSQL findings backend. Implements `FindingsSink` with idempotent upserts across all three record layers (findings, occurrences, observations) using `ON CONFLICT` merge logic.
- **`gossip-pg-common`** -- Shared PostgreSQL utilities: connection pool management, migration runner, and common schema helpers used by both backends.

Both the done-ledger and findings Postgres backends pass the same eleven-check conformance harness described in Chapter 6, validating lattice merge, idempotent replay, referential integrity, and redaction against a live database.
