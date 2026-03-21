# The Three-Layer Witness -- Findings Persistence as a Normalized Identity Model

*A GitHub connector reports an AWS key at byte offset 1,024 in `infrastructure/secrets.yaml`, version `v3`, tenant `t-9281`. The security team triages the finding and marks it "resolved." Eight days later, a developer pushes version `v4` of the same file. The same AWS key now sits at byte offset 2,048 — the developer added a 1,024-byte header above it, shifting every line down. The scanner fires again. What happens next determines whether the organization sees a clean dashboard or a dangerous blind spot. If the system treats this as an entirely new finding, the resolved triage state is lost; the team must re-investigate a secret they already handled. If the system collapses the new detection into the old finding and ignores the positional change, it silently swallows the fact that the secret has moved — and any tooling that relies on byte offsets to remediate (automated rotation, line-level comments, patch generation) points at stale coordinates. Neither outcome is acceptable. The system needs three distinct layers of identity: one that is version-stable (the secret exists in this item), one that is version-and-position-specific (the secret appears at these exact bytes in this exact version), and one that is run-specific (this scan policy observed this occurrence during this run). Without all three, the model either duplicates or collapses information that downstream consumers need to remain distinct.*

---

The persistence layer in Gossip-rs does not store "findings" as flat rows. It stores a normalized three-layer hierarchy where each layer answers a different question about a detected secret. The layering is not an accident of schema evolution or a premature optimization for query patterns. It exists because the three identity lifetimes — version-stable, version-specific, and run-specific — are fundamentally different, and conflating any two of them creates data loss.

The design lives in a single file: `crates/gossip-contracts/src/persistence/findings.rs`. The module documentation states the contract directly:

> Scan results are persisted in three normalized layers, each with different stability and scoping properties.

This chapter walks through each layer, its constructor discipline, its identity derivation, and the batch and trait abstractions that tie the three layers together for durable persistence.

## Layer 1: FindingRecord — Version-Stable Identity

A `FindingRecord` answers the question: *does this secret exist in this item?* It deliberately excludes any version-specific or run-specific information. The same secret detected by the same rule in the same logical item produces the same `FindingRecord` regardless of which object version or scan policy surfaced it.

```rust
// crates/gossip-contracts/src/persistence/findings.rs:66-73

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct FindingRecord {
    tenant_id: TenantId,
    finding_id: FindingId,
    stable_item_id: StableItemId,
    rule_fingerprint: RuleFingerprint,
    secret_hash: SecretHash,
}
```

Five fields, all fixed-width 32-byte identity types (except `TenantId`, which is also 32 bytes). No strings, no variable-length data, no timestamps.

The field semantics:

| Field | Width | Purpose |
|-------|-------|---------|
| `tenant_id` | 32 bytes | Tenant isolation boundary — every record is tenant-scoped |
| `finding_id` | 32 bytes | Content-addressed identity derived from the other three fields |
| `stable_item_id` | 32 bytes | Version-stable item identity (a repository, a bucket, a drive) |
| `rule_fingerprint` | 32 bytes | Hash of the detection rule that matched |
| `secret_hash` | 32 bytes | Tenant-keyed one-way hash of the secret material — raw bytes are never stored |

### The Constructor: Auto-Derived Identity

`FindingRecord::new()` does not accept a `FindingId`. It derives one deterministically from the natural key fields:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:78-98

#[must_use]
pub fn new(
    tenant_id: TenantId,
    stable_item_id: StableItemId,
    rule_fingerprint: RuleFingerprint,
    secret_hash: SecretHash,
) -> Self {
    let finding_id = derive_finding_id(&FindingIdInputs {
        tenant: tenant_id,
        item: stable_item_id,
        rule: rule_fingerprint,
        secret: secret_hash,
    });
    Self {
        tenant_id,
        finding_id,
        stable_item_id,
        rule_fingerprint,
        secret_hash,
    }
}
```

The caller cannot supply a `FindingId`. There is no `from_raw()` or `with_id()` escape hatch. The identity is a pure function of the four natural key fields, derived through BLAKE3 derive-key mode with the domain tag `"gossip/finding/v1"`. This makes `FindingId` a content-addressed hash: identical inputs always produce the same output, and no coordination (counters, sequences, UUID generation) is required.

The critical design property: `ObjectVersionId` is *not* an input to `FindingId`. The same secret in the same item under the same rule maps to the same `FindingId` across all versions of that item. Triage state attached to a `FindingId` survives version churn.

### Integrity Verification: `verify_id()`

For write-path sanity checking, `FindingRecord` exposes a verification method that re-derives the `FindingId` and compares it to the stored value:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:147-155

#[must_use]
pub fn verify_id(&self) -> bool {
    let expected = derive_finding_id(&FindingIdInputs {
        tenant: self.tenant_id,
        item: self.stable_item_id,
        rule: self.rule_fingerprint,
        secret: self.secret_hash,
    });
    self.finding_id == expected
}
```

This returns a simple `bool` rather than a `Result` because `FindingRecord` always constructs its own ID — there is no path where a caller-supplied ID could diverge. The check exists as a defense-in-depth checkpoint, not as a validation gate. Compare this to the richer `Result`-returning `validate_identity()` on `ObservationRecord`, where a stored ID from persistence *can* legitimately mismatch.

## Layer 2: OccurrenceRecord — Version-and-Position-Specific

An `OccurrenceRecord` answers the question: *where exactly does this secret appear in this specific version?* It pins a `FindingRecord` to a concrete byte range within a specific object version.

```rust
// crates/gossip-contracts/src/persistence/findings.rs:167-175

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct OccurrenceRecord {
    tenant_id: TenantId,
    occurrence_id: OccurrenceId,
    finding_id: FindingId,
    object_version_id: ObjectVersionId,
    byte_offset: u64,
    byte_length: NonZeroU64,
}
```

Six fields. The `finding_id` is a foreign key back to layer 1. The `object_version_id` ties this occurrence to a specific version (a commit SHA, an object etag, an S3 version ID). The byte span `[byte_offset, byte_offset + byte_length)` identifies the exact location of the secret material within that version's content.

### Why `NonZeroU64` for `byte_length`

`byte_length` is `NonZeroU64`, not `u64`. A zero-length secret is physically impossible — there must be at least one byte of secret material for the detection rule to have matched. Rather than enforcing this invariant through documentation or runtime assertions scattered across callers, the type itself makes the illegal state unrepresentable. The `NonZeroU64` guarantee is established at construction time and propagated through the type system.

### The Constructor: `try_new()` with Validation

Unlike `FindingRecord::new()`, which is infallible, `OccurrenceRecord` offers a fallible `try_new()` that accepts a raw `u64` for `byte_length` and validates it:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:185-201

pub fn try_new(
    tenant_id: TenantId,
    finding_id: FindingId,
    object_version_id: ObjectVersionId,
    byte_offset: u64,
    byte_length: u64,
) -> Result<Self, PersistenceInputError> {
    let byte_length =
        NonZeroU64::new(byte_length).ok_or(PersistenceInputError::ZeroSpanLength)?;
    Ok(Self::new(
        tenant_id,
        finding_id,
        object_version_id,
        byte_offset,
        byte_length,
    ))
}
```

The `PersistenceInputError::ZeroSpanLength` variant is defined in the error module:

```rust
// crates/gossip-contracts/src/persistence/error.rs:23

/// Occurrence span length must be non-zero.
ZeroSpanLength,
```

When the caller already holds a validated `NonZeroU64`, the infallible `new()` path is available — it skips the zero-check and derives the `OccurrenceId` directly:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:206-227

#[must_use]
pub fn new(
    tenant_id: TenantId,
    finding_id: FindingId,
    object_version_id: ObjectVersionId,
    byte_offset: u64,
    byte_length: NonZeroU64,
) -> Self {
    let occurrence_id = derive_occurrence_id(&OccurrenceIdInputs {
        finding: finding_id,
        version: object_version_id,
        byte_offset,
        byte_length: byte_length.get(),
    });
    Self {
        tenant_id,
        occurrence_id,
        finding_id,
        object_version_id,
        byte_offset,
        byte_length,
    }
}
```

Like `FindingRecord`, the `OccurrenceId` is derived, not supplied. The derivation inputs are `(finding_id, object_version_id, byte_offset, byte_length)` — the natural key for "this finding at this position in this version." The same finding at the same bytes in the same version always produces the same `OccurrenceId`.

### `verify_id()`: Same Pattern as Layer 1

```rust
// crates/gossip-contracts/src/persistence/findings.rs:280-288

#[must_use]
pub fn verify_id(&self) -> bool {
    let expected = derive_occurrence_id(&OccurrenceIdInputs {
        finding: self.finding_id,
        version: self.object_version_id,
        byte_offset: self.byte_offset,
        byte_length: self.byte_length.get(),
    });
    self.occurrence_id == expected
}
```

Again a simple `bool` — the ID is always self-derived, so mismatch indicates memory corruption or serialization bugs, not caller error.

## Layer 3: ObservationRecord — Run-Specific Witness

An `ObservationRecord` answers the question: *when and how was this occurrence observed?* It records the fact that a specific occurrence was seen during a particular scan run, under a particular scan policy, in a specific shard and fence epoch. This is the most granular layer — it captures *when* and *under what conditions* a secret was detected, not just *where*.

```rust
// crates/gossip-contracts/src/persistence/findings.rs:301-313

#[derive(Clone, Debug, PartialEq, Eq)]
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
    location: Option<Location>,
}
```

Ten fields. This is the widest record in the findings model, and deliberately so. Each field serves a distinct purpose:

| Field | Purpose |
|-------|---------|
| `tenant_id` | Tenant isolation boundary |
| `observation_id` | Content-addressed identity derived from `(tenant, policy_hash, occurrence_id)` |
| `occurrence_id` | Foreign key to layer 2 — which occurrence was observed |
| `policy_hash` | Hash of the scan policy active during detection |
| `ovid_hash` | Object-version identity linking back to the done-ledger |
| `run_id` | Which scan run produced this observation |
| `shard_id` | Which shard was being processed |
| `fence_epoch` | Lease epoch under which the writing worker held its shard lease |
| `seen_at` | Logical timestamp when the occurrence was observed |
| `location` | Optional display metadata — path and URL for presentation |

### Why Location Lives on Observation, Not on Occurrence

The `Location` type is defined in the connector module:

```rust
// crates/gossip-contracts/src/connector/types.rs:833-837

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Location {
    display: String,
    url: Option<String>,
}
```

A `Location` carries a human-readable display path and an optional deep-link URL. This is presentation context — it describes how to show the finding to a user, not the finding's identity. The display path can change when a repository renames a file; the URL can change when a hosting provider restructures its link format. Neither change affects the underlying identity of the finding or its occurrence.

Attaching `Location` to the observation layer means each policy+run combination can produce its own presentation context without polluting the stable identity layers. It also means the `Location` field is `Option<Location>` — observations from batch-mode scanners that do not produce display metadata simply omit it.

### The Constructor: `new()`

```rust
// crates/gossip-contracts/src/persistence/findings.rs:327-354

#[must_use]
pub fn new(
    tenant_id: TenantId,
    occurrence_id: OccurrenceId,
    policy_hash: PolicyHash,
    ovid_hash: OvidHash,
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    seen_at: LogicalTime,
) -> Self {
    let observation_id = derive_observation_id(&ObservationIdInputs {
        tenant: tenant_id,
        policy: policy_hash,
        occurrence: occurrence_id,
    });
    Self {
        tenant_id,
        observation_id,
        occurrence_id,
        policy_hash,
        ovid_hash,
        run_id,
        shard_id,
        fence_epoch,
        seen_at,
        location: None,
    }
}
```

The `ObservationId` is derived from `(tenant_id, policy_hash, occurrence_id)` — three fixed-width 32-byte fields totaling 96 bytes of canonical input. Location defaults to `None`; the builder method `with_location()` attaches it:

```rust
// crates/gossip-contracts/src/persistence/findings.rs

#[must_use]
pub fn with_location(self, location: Arc<Location>) -> Self {
    Self {
        location: Some(location),
        ..self
    }
}
```

The method accepts `Arc<Location>` so multiple observations sharing the same
location can avoid cloning the underlying `String` fields.

### Round-Trip Reconstruction: `from_persisted()`

When reading an observation back from storage, the system must verify that the stored `ObservationId` still matches the canonical derivation. `from_persisted()` handles this:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:363-395

pub fn from_persisted(
    tenant_id: TenantId,
    stored_observation_id: ObservationId,
    occurrence_id: OccurrenceId,
    policy_hash: PolicyHash,
    ovid_hash: OvidHash,
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    seen_at: LogicalTime,
    location: Option<Location>,
) -> Result<Self, PersistenceInputError> {
    let mut record = Self::new(
        tenant_id,
        occurrence_id,
        policy_hash,
        ovid_hash,
        run_id,
        shard_id,
        fence_epoch,
        seen_at,
    );

    if record.observation_id != stored_observation_id {
        return Err(PersistenceInputError::ObservationIdMismatch {
            expected: record.observation_id,
            actual: stored_observation_id,
        });
    }

    record.location = location;
    Ok(record)
}
```

The pattern: construct the record fresh (which re-derives the canonical `ObservationId`), compare to the stored value, and reject on mismatch. This catches data corruption, accidental schema migration errors, and any code path that might have written an `ObservationId` computed from incorrect inputs. The error type is specific:

```rust
// crates/gossip-contracts/src/persistence/error.rs:25-28

/// A provided observation id does not match the canonical derived value.
ObservationIdMismatch {
    expected: ObservationId,
    actual: ObservationId,
},
```

### `validate_identity()`: Richer Than `verify_id()`

Unlike the `bool`-returning `verify_id()` on layers 1 and 2, `ObservationRecord` provides a `Result`-returning `validate_identity()`:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:399-410

pub fn validate_identity(&self) -> Result<(), PersistenceInputError> {
    let expected = self.derived_observation_id();

    if self.observation_id != expected {
        return Err(PersistenceInputError::ObservationIdMismatch {
            expected,
            actual: self.observation_id,
        });
    }

    Ok(())
}
```

The distinction matters. `FindingRecord` and `OccurrenceRecord` always derive their own IDs at construction — there is no code path that accepts a caller-supplied ID, so mismatch is theoretically impossible and a `bool` suffices. `ObservationRecord::from_persisted()` *does* accept a stored ID from an external source (the database), making mismatch a realistic failure mode that callers need to handle with proper error propagation, not a silent boolean check.

## The Three-Layer Identity Hierarchy

The three layers form a strict containment hierarchy. Each layer's identity has a different stability lifetime:

```text
┌──────────────────────────────────────────────────────────────────┐
│                        FindingRecord                             │
│  FindingId = H(tenant, stable_item_id, rule_fingerprint,         │
│               secret_hash)                                       │
│  Lifetime: version-independent — survives across all versions    │
│            of the same item                                      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    OccurrenceRecord                        │  │
│  │  OccurrenceId = H(finding_id, object_version_id,           │  │
│  │                   byte_offset, byte_length)                │  │
│  │  Lifetime: version+position-specific — a new version or    │  │
│  │            a shifted offset produces a new OccurrenceId    │  │
│  │                                                            │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │                ObservationRecord                     │  │  │
│  │  │  ObservationId = H(tenant, policy_hash,              │  │  │
│  │  │                    occurrence_id)                     │  │  │
│  │  │  Lifetime: run+policy-specific — the same             │  │  │
│  │  │            occurrence observed under different         │  │  │
│  │  │            policies produces different                │  │  │
│  │  │            ObservationIds                             │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

Returning to the opening scenario: the AWS key in `secrets.yaml` version `v3` at byte 1,024 and the same key in version `v4` at byte 2,048 share the same `FindingId` (same tenant, same item, same rule, same secret hash). They produce different `OccurrenceId` values (different version, different offset). Each scan run that detects them produces its own `ObservationRecord`. The triage state attached to the `FindingId` is preserved. The byte-level coordinates for each version are precise. The audit trail of which scan run observed what is complete.

This separation also explains why an occurrence's `finding_id` is a foreign key, not a denormalized copy of the finding's natural key fields. The `FindingId` is already a content-addressed hash — it *is* the stable reference. Storing the natural key fields again on the occurrence would be redundant and would create a consistency burden.

## FindingsUpsertBatch: Referentially Closed Batches

The three record layers are submitted together in a single batch. `FindingsUpsertBatch` is a borrowed, zero-copy view over three slices:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:516-521

#[derive(Clone, Copy, Debug, Default)]
pub struct FindingsUpsertBatch<'a> {
    findings: &'a [FindingRecord],
    occurrences: &'a [OccurrenceRecord],
    observations: &'a [ObservationRecord],
}
```

The batch borrows slices rather than owning them. The caller retains control over allocation and can reuse buffers across flushes — critical for hot-path performance where allocation churn must be avoided. An empty batch (all three slices empty) is a valid no-op.

### Construction and Accessors

```rust
// crates/gossip-contracts/src/persistence/findings.rs:527-537

#[inline]
#[must_use]
pub const fn new(
    findings: &'a [FindingRecord],
    occurrences: &'a [OccurrenceRecord],
    observations: &'a [ObservationRecord],
) -> Self {
    Self {
        findings,
        occurrences,
        observations,
    }
}
```

The `const fn` constructor and `Copy` derive mean batch creation involves no allocation — just three pointer-length pairs on the stack.

### Referential Integrity Validation

The batch provides a method that validates the parent-child relationships across all three layers within the batch:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:612-660

pub fn validate_referential_integrity(self) -> Result<(), PersistenceInputError> {
    // Tenant consistency: all records must share the same tenant.
    let expected_tenant = self
        .findings
        .first()
        .map(|f| f.tenant_id())
        .or_else(|| self.occurrences.first().map(|o| o.tenant_id()))
        .or_else(|| self.observations.first().map(|o| o.tenant_id()));

    if let Some(expected) = expected_tenant {
        for f in self.findings {
            if f.tenant_id() != expected {
                return Err(PersistenceInputError::InconsistentTenant);
            }
        }
        for occ in self.occurrences {
            if occ.tenant_id() != expected {
                return Err(PersistenceInputError::InconsistentTenant);
            }
        }
        for obs in self.observations {
            if obs.tenant_id() != expected {
                return Err(PersistenceInputError::InconsistentTenant);
            }
        }
    }

    // Referential integrity: occurrences → findings, observations → occurrences.
    let finding_ids: HashSet<_> = self.findings.iter().map(|f| f.finding_id()).collect();
    for occ in self.occurrences {
        if !finding_ids.contains(&occ.finding_id()) {
            return Err(PersistenceInputError::OrphanedReference {
                child_type: "OccurrenceRecord",
                parent_type: "FindingRecord",
            });
        }
    }
    let occurrence_ids: HashSet<_> =
        self.occurrences.iter().map(|o| o.occurrence_id()).collect();
    for obs in self.observations {
        if !occurrence_ids.contains(&obs.occurrence_id()) {
            return Err(PersistenceInputError::OrphanedReference {
                child_type: "ObservationRecord",
                parent_type: "OccurrenceRecord",
            });
        }
    }
    Ok(())
}
```

This method enforces three invariants:

1. **Tenant consistency**: Every record in the batch — across all three layers — must belong to the same tenant. The first non-empty layer's first record establishes the expected `TenantId`; every subsequent record is checked against it.

2. **Occurrence → Finding referential integrity**: Every `OccurrenceRecord.finding_id` must reference a `FindingRecord` present in the same batch.

3. **Observation → Occurrence referential integrity**: Every `ObservationRecord.occurrence_id` must reference an `OccurrenceRecord` present in the same batch.

The orphaned-reference error captures which child type failed to find its parent:

```rust
// crates/gossip-contracts/src/persistence/error.rs:42-45

/// A child record references a parent that does not exist in the batch.
OrphanedReference {
    child_type: &'static str,
    parent_type: &'static str,
},
```

An important caveat from the source documentation: this validates within-batch closure only. Cross-batch references to already-persisted records cannot be checked here — the batch has no access to the backend's existing state. Backends that accept partial batches (occurrences referencing previously-persisted findings) must handle that validation internally.

The method is classified as a COLD-path diagnostic tool. Backends call it in debug mode or tests, not on every hot-path upsert. The cost is O(n) in the number of occurrences and observations, bounded by the recommended maximum batch size.

### Observation Identity Validation

A separate validation method checks that every observation's stored `ObservationId` matches the canonical derivation:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:585-591

pub fn validate_observation_identity(self) -> Result<(), PersistenceInputError> {
    for observation in self.observations {
        observation.validate_identity()?;
    }

    Ok(())
}
```

Only observations are checked because `FindingRecord` and `OccurrenceRecord` always derive their IDs at construction — no caller-supplied ID is accepted. `ObservationRecord::from_persisted()` is the only path that accepts a stored ID, making observations the only layer where mismatch is possible.

## FindingsSink: The Persistence Trait

All findings persistence goes through a single trait:

```rust
// crates/gossip-contracts/src/persistence/findings.rs:686-709

pub trait FindingsSink: Send + Sync {
    /// Backend-specific immediate submit or wait error type.
    type Error: Error + Send + Sync + 'static;
    /// Handle returned for eventual durable acknowledgement of submitted rows.
    type CommitHandle: CommitHandle<Receipt = FindingsCommitReceipt, Error = Self::Error>;

    /// Submit all rows in `batch`, inserting new records and ignoring
    /// duplicates (upsert semantics).
    fn upsert_batch(
        &self,
        batch: FindingsUpsertBatch<'_>,
    ) -> Result<Self::CommitHandle, Self::Error>;
}
```

The trait design encodes several important contracts:

**Two associated types, not concrete types.** `Error` and `CommitHandle` are backend-specific. An SQLite backend returns synchronous errors and a `ReadyCommitHandle` (which resolves immediately). A distributed backend might return an async handle that waits for replication quorum. The trait abstracts over both.

**Upsert semantics, not insert.** The method is named `upsert_batch`, and the implementor contract in the documentation is explicit: "Replaying the same batch, or overlapping batches, must not create duplicate rows." Content-addressed IDs (`FindingId`, `OccurrenceId`, `ObservationId`) serve as natural deduplication keys. This makes the entire persistence path idempotent — a critical property for retry-safe crash recovery.

**Submission is not durability.** Returning `Ok(handle)` means the backend *accepted* the batch. Durability is established only when `CommitHandle::wait()` yields a `FindingsCommitReceipt`. This two-phase design lets storage implementations batch or defer fsync/replication work without weakening the caller-visible contract. The `CommitHandle` trait consumes `self` on `wait()` to prevent double-waits:

```rust
// crates/gossip-contracts/src/persistence/commit.rs:63-76

#[must_use = "durability is not established until wait() returns a receipt"]
pub trait CommitHandle: Send + 'static {
    type Receipt: CommitReceipt;
    type Error: Error + Send + Sync + 'static;

    fn wait(self) -> Result<Self::Receipt, Self::Error>;
}
```

The `#[must_use]` annotation on `CommitHandle` prevents callers from silently discarding the handle — a fire-and-forget `upsert_batch()` call that ignores the return value triggers a compiler warning.

**Referential integrity is a backend obligation.** The trait documentation states: "The backend must ensure that every `OccurrenceRecord.finding_id` references a persisted or in-batch `FindingRecord`, and every `ObservationRecord.occurrence_id` references a persisted or in-batch `OccurrenceRecord`." Whether this is enforced through foreign keys, application-level checks, or implicit batch closure is a backend decision.

## FindingsCommitReceipt: The Durability Proof

When `CommitHandle::wait()` succeeds, it returns a `FindingsCommitReceipt`:

```rust
// crates/gossip-contracts/src/persistence/commit.rs:136-141

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct FindingsCommitReceipt {
    finding_count: u64,
    occurrence_count: u64,
    observation_count: u64,
}
```

Three counters, mirroring the three-layer model. Each count reflects the number of distinct records in the submitted batch that the backend has durably acknowledged — not the number of newly inserted rows. An idempotent replay of an already-durable batch returns the same counts as the original write.

```rust
// crates/gossip-contracts/src/persistence/commit.rs:146-153

#[inline]
#[must_use]
pub const fn new(finding_count: u64, occurrence_count: u64, observation_count: u64) -> Self {
    Self {
        finding_count,
        occurrence_count,
        observation_count,
    }
}
```

The receipt is `Clone + Copy + Debug + Send + Sync` — it implements the `CommitReceipt` marker trait and can be embedded into composite receipts. In the broader commit protocol, `FindingsCommitReceipt` feeds into `ItemCommitReceipt` (which combines findings + done-ledger durability) and ultimately into `PageCommitReceipt` (which adds checkpoint durability). The receipt hierarchy mirrors the page-commit protocol: no page is considered durable until all three sub-receipts — findings, done-ledger, and checkpoint — have been collected.

## Design Properties: Why Three Layers

The three-layer decomposition creates several properties that a flat schema cannot provide. Each property addresses a concrete operational requirement — not a theoretical purity concern.

**Stable triage.** Marking a `FindingId` as resolved applies across all versions of the item. When the secret moves to a new byte offset or appears in a new version, the triage state is not lost — the new `OccurrenceRecord` still references the same `FindingId`. A security team that resolves a finding in version `v3` does not need to re-triage it when version `v4` surfaces the same secret at different coordinates. The `FindingId` acts as a durable anchor for all downstream workflow state.

**Precise remediation.** Automated tools that need byte-level coordinates (rotation scripts, line-level PR comments, patch generators) consume `OccurrenceRecord` data. Each version gets its own precise coordinates without overwriting the coordinates from other versions. A remediation bot targeting version `v4` reads `byte_offset=2048, byte_length=40` from the `v4` occurrence, not the stale `byte_offset=1024` from `v3`. Both occurrences coexist under the same finding.

**Complete audit trail.** Each scan run that detects a secret produces its own `ObservationRecord`. The audit trail shows which policy, which run, which shard, and which fence epoch produced each detection — information required for compliance reporting and debugging scan coverage gaps. If a finding was detected by run `R-001` on Monday but not by run `R-002` on Tuesday, the observation layer makes that absence visible without ambiguity.

**Policy-aware deduplication.** The `ObservationId` includes `policy_hash`, so the same occurrence observed under two different policies produces two distinct observations. This prevents a strict policy from silently collapsing its detections into a permissive policy's observation records. An organization running both a "full-entropy" scan policy and a "high-confidence-only" policy gets separate audit trails for each, even when both policies detect the same secret at the same location.

**Idempotent replay.** All three IDs are content-addressed. Replaying a batch produces the same records with the same IDs. Upsert semantics at the backend level mean duplicates are silently absorbed. Crash recovery requires no special reconciliation — just replay the batch. If a worker crashes after submitting a batch but before advancing the cursor, the next worker picks up the same page and re-submits the same records. The backend absorbs the duplicates, the cursor advances, and no data is lost or doubled.

**No N+1 write amplification.** Because the batch groups all three layers into a single `upsert_batch()` call, the backend can persist findings, occurrences, and observations in a single transaction rather than making three separate round-trips. The borrowed-slice design of `FindingsUpsertBatch` means the caller assembles the batch in-place without copying records into a new allocation for the submission.

## Summary

The findings persistence model is a normalized three-layer hierarchy:

| Layer | Record Type | Identity Inputs | Stability |
|-------|------------|-----------------|-----------|
| 1 | `FindingRecord` | `(tenant, item, rule, secret_hash)` | Version-independent — survives across all versions |
| 2 | `OccurrenceRecord` | `(finding, version, offset, length)` | Version+position-specific — new version or offset = new ID |
| 3 | `ObservationRecord` | `(tenant, policy, occurrence)` | Run+policy-specific — same occurrence under different policies = different IDs |

Records are submitted in `FindingsUpsertBatch` — a borrowed, zero-copy view over three slices with intra-batch referential integrity validation. The `FindingsSink` trait accepts these batches with idempotent upsert semantics, returning a `CommitHandle` that must be waited on for durability proof. The resulting `FindingsCommitReceipt` carries three counters mirroring the three layers.

## What's Next

Chapter 4 covers the **commit protocol** that ties findings persistence to the done-ledger and checkpoint layers. The `FindingsCommitReceipt` produced by `FindingsSink::upsert_batch()` is one of three receipts that must be collected before a page is considered fully durable. The page-commit typestate machine enforces the ordering: findings first, then done-ledger, then checkpoint — and only a complete `PageCommitReceipt` proves the cursor can safely advance.
