# The Lattice That Never Forgets -- Done-Ledger Deduplication and Monotonic Status Merge

*A scanning cluster has 48 workers processing an S3 bucket containing 12 million objects. Worker A picks up object `reports/quarterly-2025.xlsx` (version `e3b0c44298fc`, policy hash `a1f9...03d2`) and scans it clean -- no secrets detected, zero findings. Fourteen milliseconds later, Worker B picks up the same object under policy hash `b7c4...91e0` -- the policy was updated mid-batch with a new regex rule for API keys. Worker B detects 3 findings. Both workers submit done-ledger records concurrently. Worker A's record arrives at the persistence backend 2 milliseconds after Worker B's. A naive last-writer-wins store overwrites `ScannedWithFindings` with `ScannedClean`. The 3 API key findings are now invisible to every downstream consumer -- the triage pipeline, the notification system, the compliance dashboard. The done-ledger claims the object was scanned and clean. Nobody re-scans it. The leaked API keys remain in production for 47 days until an unrelated audit catches them.*

---

This failure mode is not hypothetical. Any system that tracks "was this item processed?" across concurrent workers faces the write-ordering problem. The done-ledger in Gossip-rs eliminates it entirely by making status a **join-semilattice**: a mathematical structure where concurrent writes always converge to the same value regardless of arrival order. The merge rule is simple -- the higher-ranked status wins -- and the rank ordering guarantees that a scanned-with-findings status can never be downgraded by a concurrent scanned-clean write.

The done-ledger is not a simple boolean "seen/unseen" flag. It is a fully typed persistence contract with validated keys, monotonic status progression, cross-field invariants, structured error codes, worker provenance tracking, and a backend-neutral trait that enforces lattice merge semantics at the type level. Every component exists in the codebase today, implemented and tested.

This chapter walks through every type in the done-ledger contract, from the composite key that identifies what was scanned, through the lattice that governs how concurrent writes merge, to the trait that backends must implement.

## The Composite Key: `DoneLedgerKey`

A done-ledger entry answers a precise question: "Has this exact object-version been processed under this exact policy for this exact tenant?" The key must capture all three dimensions. Missing any one of them collapses distinct scan scenarios into a single row, either suppressing re-scans that should happen or permitting duplicates that should not.

From `done_ledger.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct DoneLedgerKey {
    tenant_id: TenantId,
    policy_hash: PolicyHash,
    ovid_hash: OvidHash,
}
```

Three fields, three scoping roles:

**`tenant_id: TenantId`** -- Tenant isolation boundary. Every row in the done-ledger belongs to exactly one tenant. Two tenants scanning the same S3 bucket with the same policy produce separate done-ledger entries. There is no cross-tenant visibility, no shared deduplication state, no information leakage through key collisions.

**`policy_hash: PolicyHash`** -- Scan-policy version. When the security team adds a new detection rule or adjusts a threshold, the policy hash changes. A stale done-ledger entry from a previous policy version must not suppress a re-scan under the updated policy. Including the policy hash in the key ensures that policy changes automatically invalidate prior results -- without requiring an explicit "invalidate all entries" migration.

**`ovid_hash: OvidHash`** -- Object-version identity. This is where the key design diverges from the naive approach. Instead of storing `StableItemId` and `ObjectVersionId` as separate fields, the done-ledger compresses them into a single 32-byte hash. The next section explains why.

The key implements `CanonicalBytes` for content-addressed identity derivation, writing all three fields in a fixed, unambiguous order:

From `done_ledger.rs`:

```rust
impl CanonicalBytes for DoneLedgerKey {
    #[inline]
    fn write_canonical(&self, h: &mut Hasher) {
        self.tenant_id.write_canonical(h);
        self.policy_hash.write_canonical(h);
        self.ovid_hash.write_canonical(h);
    }
}
```

The constructor is `const` and infallible -- all three inputs are pre-validated types:

From `done_ledger.rs`:

```rust
impl DoneLedgerKey {
    #[inline]
    #[must_use]
    pub const fn new(
        tenant_id: TenantId,
        policy_hash: PolicyHash,
        ovid_hash: OvidHash,
    ) -> Self {
        Self {
            tenant_id,
            policy_hash,
            ovid_hash,
        }
    }
}
```

## OvidHash: Compressing Identity Into 32 Bytes

A raw done-ledger key storing `StableItemId` (32 bytes) + `ObjectVersionId` (32 bytes) + the version-strength discriminant (1 byte) would consume 65 bytes per row just for the object-version portion. At scale -- tens of millions of done-ledger entries -- that overhead matters. `OvidHash` compresses the object-version identity into a fixed 32-byte BLAKE3 hash, derived in a dedicated domain-separated context.

The domain constant lives in the authoritative registry:

From `domain.rs`:

```rust
pub const OVID_V1: &str = "gossip/persistence/v1/ovid";
```

**OVID** stands for **O**bject-**V**ersion **Id**entity.

The derivation inputs are captured in a structured type:

From `ovid.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct OvidHashInputs {
    /// Stable item identity (connector-scoped).
    pub stable_item_id: StableItemId,
    /// Version claim (strong or weak) from the connector.
    pub version: VersionId,
}
```

Two fields feed into the hash:

- **`stable_item_id`** -- The content-addressed identifier for the item's logical identity, already scoped by connector tag and connector instance. This is derived upstream via `ItemIdentityKey::stable_id()`.
- **`version`** -- A `VersionId` enum that wraps an `ObjectVersionId` with claim-strength semantics.

The `VersionId` type distinguishes between strong and weak version claims:

From `types.rs`:

```rust
pub enum VersionId {
    /// Strong version identity that should reliably reference immutable content.
    Strong(ObjectVersionId),
    /// Weak version identity where content may change without version changes.
    Weak(ObjectVersionId),
}
```

A strong version (e.g., an S3 ETag for a non-multipart upload) guarantees that the same version identifier always references the same bytes. A weak version (e.g., a last-modified timestamp) provides no such guarantee. This distinction matters for the done-ledger: the same `ObjectVersionId` bytes under `Strong` and `Weak` produce different `OvidHash` values. The canonical serialization prefixes `0u8` for strong and `1u8` for weak, ensuring the two cases never collide:

From `ovid.rs`:

```rust
fn write_version_id_canonical(version: VersionId, h: &mut Hasher) {
    match version {
        VersionId::Strong(version_id) => {
            0u8.write_canonical(h);
            version_id.write_canonical(h);
        }
        VersionId::Weak(version_id) => {
            1u8.write_canonical(h);
            version_id.write_canonical(h);
        }
    }
}
```

The derivation function uses a cached `LazyLock` hasher clone for performance -- the BLAKE3 derive-key setup cost is paid once per process:

From `ovid.rs`:

```rust
static OVID_HASHER: LazyLock<Hasher> =
    LazyLock::new(|| Hasher::new_derive_key(identity::domain::OVID_V1));

pub fn derive_ovid_hash(inputs: &OvidHashInputs) -> OvidHash {
    OvidHash::from_bytes(derive_from_cached(&OVID_HASHER, inputs))
}
```

The `derive_from_cached` function clones the pre-initialized hasher, feeds the canonical bytes, and finalizes to 32 bytes. This pattern appears throughout the identity subsystem -- it amortizes the derive-key context setup across millions of hash operations.

**Why hash instead of storing fields separately?** The done-ledger key already includes `TenantId` (for tenant-scoped queries) and `PolicyHash` (for policy-scoped lookups). The object-version identity is only needed for equality comparison -- "is this the same object-version?" -- not for decomposition back into its constituent parts. A 32-byte hash answers that question with 256-bit collision resistance and cuts per-row storage roughly in half compared to storing both fields separately.

There is a second, subtler benefit: uniform key width. When all object-version identities are exactly 32 bytes, backend storage engines can use fixed-width key columns, aligned B-tree nodes, and predictable index page sizes. Variable-width keys complicate storage layout and make worst-case sizing harder to predict. The done-ledger never needs to decompose an `OvidHash` back into its `StableItemId` and `VersionId` constituents -- the hash is strictly a deduplication token, not a retrieval key.

## The Status Lattice: `DoneLedgerStatus`

The status field is where the done-ledger solves the concurrent-write problem from the opening scenario. `DoneLedgerStatus` is not a simple enum -- it is a join-semilattice with a total order defined by discriminant rank:

From `done_ledger.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
#[repr(u8)]
pub enum DoneLedgerStatus {
    /// Temporary failure; the item may be retried and succeed later.
    FailedRetryable = 1,
    /// Terminal failure for the current tenant + policy + object-version triple.
    FailedPermanent = 2,
    /// Intentionally skipped (unsupported format, filtered by policy, size cap, etc.).
    Skipped = 3,
    /// Successfully scanned and committed; no secrets or findings detected.
    ScannedClean = 10,
    /// Successfully scanned and committed; one or more findings were persisted.
    ScannedWithFindings = 11,
}
```

The discriminant values define a total order:

```
FailedRetryable (1) < FailedPermanent (2) < Skipped (3) < ScannedClean (10) < ScannedWithFindings (11)
```

There is an intentional gap between `Skipped` (3) and `ScannedClean` (10). This reserves discriminant space for future non-terminal states without changing the relative ordering of existing variants. Backends store the rank as a `u8`, so adding a variant between 3 and 10 is a backwards-compatible schema change.

The rank ordering encodes a deliberate information hierarchy:

- **Failures (1, 2)** carry the least information -- they mean "something went wrong, we do not know the scan result."
- **Skipped (3)** carries slightly more -- "we intentionally chose not to scan this, but it is not an error."
- **ScannedClean (10)** is a terminal success -- "we scanned it, and found nothing."
- **ScannedWithFindings (11)** is the highest-ranked status -- "we scanned it, and found secrets."

The merge function is trivially simple:

From `done_ledger.rs`:

```rust
impl DoneLedgerStatus {
    pub const fn rank(self) -> u8 {
        self as u8
    }

    pub const fn merge(self, other: Self) -> Self {
        if self.rank() >= other.rank() {
            self
        } else {
            other
        }
    }
}
```

`merge` takes the maximum rank. This single rule produces all three lattice properties:

**Idempotence**: `merge(x, x) = x`. Merging a status with itself is a no-op. Re-upserting the same record produces no state change.

**Commutativity**: `merge(a, b) = merge(b, a)`. The result does not depend on which write arrives first. Worker A's `ScannedClean` and Worker B's `ScannedWithFindings` merge to `ScannedWithFindings` regardless of arrival order.

**Monotonicity**: Once a status reaches a higher rank, it never moves backwards. A `ScannedWithFindings` entry can never be downgraded to `ScannedClean`, `Skipped`, or any failure state, no matter how many concurrent or replayed writes arrive.

Returning to the opening scenario: Worker B writes `ScannedWithFindings` (rank 11). Worker A writes `ScannedClean` (rank 10). The backend applies `merge(ScannedWithFindings, ScannedClean) = ScannedWithFindings`. The 3 API key findings are preserved. The arrival order is irrelevant.

Consider every possible pair of concurrent writes and the merge outcome:

| Write A | Write B | `merge(A, B)` | Correct? |
|---------|---------|----------------|----------|
| `FailedRetryable` (1) | `ScannedClean` (10) | `ScannedClean` | Yes -- a successful scan dominates a transient failure |
| `FailedPermanent` (2) | `FailedRetryable` (1) | `FailedPermanent` | Yes -- a permanent failure is more definitive than a retryable one |
| `Skipped` (3) | `ScannedWithFindings` (11) | `ScannedWithFindings` | Yes -- a successful scan with findings dominates a skip |
| `ScannedClean` (10) | `ScannedWithFindings` (11) | `ScannedWithFindings` | Yes -- findings must never be erased by a clean scan |
| `ScannedWithFindings` (11) | `ScannedWithFindings` (11) | `ScannedWithFindings` | Yes -- idempotent |

Every row in the table holds because the rank ordering mirrors the information content: more informative statuses always dominate less informative ones.

The `from_rank` method reconstitutes the enum from a persisted discriminant, returning `None` for unknown values:

From `done_ledger.rs`:

```rust
pub const fn from_rank(rank: u8) -> Option<Self> {
    match rank {
        1 => Some(Self::FailedRetryable),
        2 => Some(Self::FailedPermanent),
        3 => Some(Self::Skipped),
        10 => Some(Self::ScannedClean),
        11 => Some(Self::ScannedWithFindings),
        _ => None,
    }
}
```

Backends should treat an unknown rank as a data-corruption signal rather than silently mapping it to a default. The type system enforces this by returning `Option<Self>` instead of panicking or inventing a fallback.

Helper methods provide semantic queries without exposing the rank arithmetic:

From `done_ledger.rs`:

```rust
pub const fn is_scanned(self) -> bool {
    matches!(self, Self::ScannedClean | Self::ScannedWithFindings)
}

pub const fn is_failure(self) -> bool {
    matches!(self, Self::FailedRetryable | Self::FailedPermanent)
}

pub const fn is_skipped(self) -> bool {
    matches!(self, Self::Skipped)
}
```

## The Full Record: `DoneLedgerRecord`

A done-ledger row is more than a key and a status. It carries scan metrics, worker provenance, and an optional error code. The `DoneLedgerRecord` struct bundles all six fields:

From `done_ledger.rs`:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct DoneLedgerRecord {
    key: DoneLedgerKey,
    status: DoneLedgerStatus,
    bytes_scanned: u64,
    findings_count: u32,
    provenance: DoneLedgerProvenance,
    error_code: Option<DoneLedgerErrorCode>,
}
```

Field by field:

- **`key: DoneLedgerKey`** -- The composite deduplication key `(tenant, policy, ovid)`.
- **`status: DoneLedgerStatus`** -- The lattice status governing merge behavior.
- **`bytes_scanned: u64`** -- Total bytes consumed while scanning the object-version. Useful for throughput metrics and billing.
- **`findings_count: u32`** -- Number of distinct findings detected during the scan. This field has cross-field invariants with `status`.
- **`provenance: DoneLedgerProvenance`** -- Which worker, shard, and time window produced this entry.
- **`error_code: Option<DoneLedgerErrorCode>`** -- Structured error code, present for failure and skip statuses.

### Construction Validation

The constructor `try_new` enforces the critical invariant: `findings_count` must be consistent with `status`.

From `done_ledger.rs`:

```rust
pub fn try_new(
    key: DoneLedgerKey,
    status: DoneLedgerStatus,
    bytes_scanned: u64,
    findings_count: u32,
    provenance: DoneLedgerProvenance,
    error_code: Option<DoneLedgerErrorCode>,
) -> Result<Self, PersistenceInputError> {
    match status {
        DoneLedgerStatus::ScannedWithFindings if findings_count == 0 => {
            return Err(PersistenceInputError::InconsistentFindingsCount {
                status: "ScannedWithFindings",
                findings_count,
            });
        }
        DoneLedgerStatus::ScannedClean if findings_count > 0 => {
            return Err(PersistenceInputError::InconsistentFindingsCount {
                status: "ScannedClean",
                findings_count,
            });
        }
        _ => {}
    }
    Ok(Self { key, status, bytes_scanned, findings_count, provenance, error_code })
}
```

The rules are strict:

- `ScannedWithFindings` with `findings_count == 0` is a contradiction -- the status says findings exist, but the count says they do not. Rejected.
- `ScannedClean` with `findings_count > 0` is equally contradictory. Rejected.
- Failure and skip statuses accept any `findings_count` -- a partially scanned object might have detected findings before encountering a fatal error.

### Write-Path Validation

The `validate()` method enforces additional cross-field invariants that `try_new` intentionally does not -- keeping the constructor flexible for deserialization from storage where some combinations may have been persisted by earlier code versions:

From `done_ledger.rs`:

```rust
pub fn validate(&self) -> Result<(), PersistenceInputError> {
    let status_name = match self.status {
        DoneLedgerStatus::ScannedClean => "ScannedClean",
        DoneLedgerStatus::ScannedWithFindings => "ScannedWithFindings",
        DoneLedgerStatus::FailedRetryable => "FailedRetryable",
        DoneLedgerStatus::FailedPermanent => "FailedPermanent",
        DoneLedgerStatus::Skipped => "Skipped",
    };

    match self.status {
        DoneLedgerStatus::ScannedClean => {
            if self.findings_count != 0 {
                return Err(PersistenceInputError::InconsistentFindingsCount {
                    status: status_name,
                    findings_count: self.findings_count,
                });
            }
            if self.error_code.is_some() {
                return Err(PersistenceInputError::UnexpectedErrorCode {
                    status: status_name,
                });
            }
        }
        DoneLedgerStatus::ScannedWithFindings => {
            if self.findings_count == 0 {
                return Err(PersistenceInputError::InconsistentFindingsCount {
                    status: status_name,
                    findings_count: self.findings_count,
                });
            }
            if self.error_code.is_some() {
                return Err(PersistenceInputError::UnexpectedErrorCode {
                    status: status_name,
                });
            }
        }
        DoneLedgerStatus::FailedRetryable
        | DoneLedgerStatus::FailedPermanent
        | DoneLedgerStatus::Skipped => {
            if self.error_code.is_none() {
                return Err(PersistenceInputError::MissingErrorCode {
                    status: status_name,
                });
            }
        }
    }
    Ok(())
}
```

The full invariant set:

| Status | `findings_count` | `error_code` |
|--------|-------------------|--------------|
| `ScannedClean` | Must be 0 | Must be `None` |
| `ScannedWithFindings` | Must be > 0 | Must be `None` |
| `FailedRetryable` | Any | Must be `Some` |
| `FailedPermanent` | Any | Must be `Some` |
| `Skipped` | Any | Must be `Some` |

Success statuses must not carry error codes. Failure and skip statuses must carry them. Write-path callers invoke `validate()` before persisting a record; the constructor `try_new` enforces only the findings-count consistency to keep deserialization flexible.

### Record-Level Merge

The record itself provides a `merge` method that implements per-field merge logic using the status lattice as the foundation:

From `done_ledger.rs`:

```rust
pub fn merge(&self, incoming: &DoneLedgerRecord) -> Result<Self, PersistenceInputError> {
    self.validate()?;
    incoming.validate()?;

    if self.key != incoming.key {
        return Err(PersistenceInputError::KeyMismatch {
            existing: Box::new(self.key),
            incoming: Box::new(incoming.key),
        });
    }

    let merged_status = self.status.merge(incoming.status);
    let bytes_scanned = self.bytes_scanned.max(incoming.bytes_scanned);
    // ... per-field merge logic (see below) ...
}
```

Both records are validated first -- unvalidated records can break associativity because the error-code fallback mechanism may pick different losers depending on merge grouping. The method returns `Result<Self, PersistenceInputError>`, catching key mismatches and validation failures at the earliest possible point.

The merge rules are applied per-field:

1. **Status** -- lattice join via `DoneLedgerStatus::merge`. The higher-ranked status wins, so scanned states can never be downgraded to failed or skipped.

2. **`bytes_scanned`** -- non-regressing maximum of both records.

3. **`findings_count`** -- status-aware:
   - `ScannedClean`: forced to 0.
   - `ScannedWithFindings`: takes the max of the findings counts from whichever records have `ScannedWithFindings` status, clamped to `>= 1`. Lower-status records do not contribute findings once a `ScannedWithFindings` result exists.
   - Otherwise: max of both values.

4. **Provenance** (`run_id`, `shard_id`, `fence_epoch`, timestamps) -- the winner is chosen by comparing a multi-field key: `(status rank, finished_at, started_at, fence_epoch, run_id, shard_id, error_code)` in lexicographic order. `run_id` and `shard_id` use the same signed-integer bit-pattern ordering as the PostgreSQL backend, so both implementations resolve exact ties identically. All provenance fields travel together from the winner.

5. **`error_code`** -- cleared when the merged status is a scanned state (success absorbs prior errors). Otherwise taken from the provenance winner, falling back to the loser's error code if the winner has none.

## Worker Provenance: `DoneLedgerProvenance`

Every done-ledger entry records which worker produced it. This is not part of the deduplication key -- it is write-side metadata for debugging, auditing, and stale-entry diagnostics.

From `done_ledger.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct DoneLedgerProvenance {
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    started_at: LogicalTime,
    finished_at: LogicalTime,
}
```

Five fields:

- **`run_id: RunId`** -- Globally unique identifier for the scanning run. Distinguishes entries produced by different deployments or restarts.
- **`shard_id: ShardId`** -- The shard being processed when this entry was written. Combined with `run_id`, this pinpoints the exact worker and workload that produced a result.
- **`fence_epoch: FenceEpoch`** -- The lease epoch under which the writing worker held its shard lease. This is the critical field for stale-write detection. If a worker's lease expires and a new worker acquires the shard at a higher epoch, the backend can detect and reject writes from the stale (pre-fence) worker. Without fence epochs, a slow worker that lost its lease could overwrite entries from the worker that replaced it.
- **`started_at: LogicalTime`** -- When scanning of the object-version began.
- **`finished_at: LogicalTime`** -- When scanning completed.

The constructor enforces a debug-mode invariant:

From `done_ledger.rs`:

```rust
pub fn new(
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    started_at: LogicalTime,
    finished_at: LogicalTime,
) -> Self {
    debug_assert!(
        started_at.as_raw() <= finished_at.as_raw(),
        "provenance: started_at ({started_at:?}) must not exceed finished_at ({finished_at:?})"
    );
    Self { run_id, shard_id, fence_epoch, started_at, finished_at }
}
```

`started_at` must not exceed `finished_at`. This is enforced as a `debug_assert` -- it catches logic errors during development without paying the branch cost in release builds.

When a higher-ranked status overwrites an existing row during lattice merge, the provenance is replaced along with it. The provenance always describes the worker that produced the currently dominant status. This means audit queries can answer "who produced this result?" with a single row read, without needing to traverse a write history log.

## Error Codes: `DoneLedgerErrorCode`

Failure and skip statuses carry a structured error code. This is intentionally not a free-form message field -- it exists for short, bounded, internal values like `HTTP_403`, `TIMEOUT`, or `S3:ACCESS_DENIED`.

From `done_ledger.rs`:

```rust
pub const MAX_DONE_LEDGER_ERROR_CODE_SIZE: usize = 128;

#[derive(Clone, PartialEq, Eq, Hash)]
pub struct DoneLedgerErrorCode(Box<str>);
```

The constructor validates three properties:

From `done_ledger.rs`:

```rust
pub fn try_new(code: impl Into<String>) -> Result<Self, PersistenceInputError> {
    let code = code.into();
    if code.is_empty() {
        return Err(PersistenceInputError::Empty {
            field: "DoneLedgerErrorCode",
        });
    }
    if code.len() > MAX_DONE_LEDGER_ERROR_CODE_SIZE {
        return Err(PersistenceInputError::TooLarge {
            field: "DoneLedgerErrorCode",
            size: code.len(),
            max: MAX_DONE_LEDGER_ERROR_CODE_SIZE,
        });
    }
    let mut all_spaces = true;
    for (index, byte) in code.bytes().enumerate() {
        let ok = byte.is_ascii_alphanumeric()
            || matches!(byte, b' ' | b'_' | b'-' | b'.' | b':' | b'/');
        if !ok {
            return Err(PersistenceInputError::InvalidByte {
                field: "DoneLedgerErrorCode",
                index,
                byte,
            });
        }
        if byte != b' ' {
            all_spaces = false;
        }
    }
    if all_spaces {
        return Err(PersistenceInputError::Empty {
            field: "DoneLedgerErrorCode",
        });
    }
    Ok(Self(code.into_boxed_str()))
}
```

The allowed character set is ASCII alphanumeric plus six punctuation characters: space (` `), underscore (`_`), hyphen (`-`), period (`.`), colon (`:`), and forward slash (`/`). This alphabet covers structured codes like `HTTP_403`, `TIMEOUT`, and `S3:ACCESS_DENIED` without admitting arbitrary user input, raw connector output, or binary data.

The validation also rejects all-whitespace strings as effectively empty. Maximum length is 128 bytes -- sufficient for any structured code, too small for accidental message dumps.

The error types that `DoneLedgerErrorCode::try_new` can return come from the shared `PersistenceInputError` enum:

From `error.rs`:

```rust
pub enum PersistenceInputError {
    Empty { field: &'static str },
    TooLarge { field: &'static str, size: usize, max: usize },
    InvalidByte { field: &'static str, index: usize, byte: u8 },
    InconsistentFindingsCount { status: &'static str, findings_count: u32 },
    MissingErrorCode { status: &'static str },
    UnexpectedErrorCode { status: &'static str },
    // ... additional variants for other persistence types
}
```

This enum is shared across all persistence-boundary value types -- findings, occurrences, observations, and done-ledger records all validate through the same error surface.

## The Backend Contract: `DoneLedger` Trait

The `DoneLedger` trait defines the backend-neutral interface that any durable storage implementation must satisfy. It has two methods and two associated types:

From `done_ledger.rs`:

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

**Associated types:**

- `Error` -- Backend-specific error type (database connection failure, I/O error, etc.). Bounded by `Error + Send + Sync + 'static` so callers can propagate it through async task boundaries.
- `CommitHandle` -- A handle for eventual durable acknowledgement. Constrained to `CommitHandle<Receipt = DoneLedgerCommitReceipt, Error = Self::Error>`, which ties the handle's receipt type to the done-ledger receipt and its error type to the backend's error type.

**`batch_get`** looks up done-ledger rows for a batch of object-versions within one tenant and policy. The return value is positional: the *i*-th element corresponds to `ovid_hashes[i]`, with `None` for keys not yet present. This positional contract avoids the need for callers to perform a secondary lookup to match results to inputs.

**`batch_upsert`** submits a batch of records for durable upsert, applying monotonic merge semantics for any keys that already exist. The critical contract points:

1. **Monotonic lattice merge.** When upserting a key that already exists, the backend must compare incoming and existing statuses via `DoneLedgerStatus::merge` and persist only the dominant value.
2. **Idempotent writes.** Re-upserting the same `(key, status)` pair must be a no-op.
3. **No downgrade.** A scanned status must never be overwritten by a failure or skip status.
4. **Duplicate key handling.** If the batch contains duplicate keys, the implementation must merge them -- not produce duplicate rows.
5. **Submission is not durability.** Returning `Ok(handle)` means the backend accepted the batch. Durability is established only when `CommitHandle::wait` returns a `DoneLedgerCommitReceipt`.

The two-phase submit/wait design lets storage implementations batch or defer fsync/replication work without weakening the caller-visible contract. A synchronous SQLite backend returns a `ReadyCommitHandle` whose `wait()` is instant. A distributed backend might batch multiple upserts into a single replication round and block on `wait()` until the replica quorum acknowledges.

The `CommitHandle` trait itself is defined in the commit module:

From `commit.rs`:

```rust
#[must_use = "durability is not established until wait() returns a receipt"]
pub trait CommitHandle: Send + 'static {
    type Receipt: CommitReceipt;
    type Error: Error + Send + Sync + 'static;

    fn wait(self) -> Result<Self::Receipt, Self::Error>;
}
```

`wait` consumes `self`, preventing double-waits and making the single-use nature of durable acknowledgement explicit in the type system. If a caller tries to wait twice, the compiler rejects it.

## The Durability Proof: `DoneLedgerCommitReceipt`

When `CommitHandle::wait` returns successfully, it produces a `DoneLedgerCommitReceipt` -- a proof object summarizing what became durable:

From `commit.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct DoneLedgerCommitReceipt {
    record_count: u64,
    scanned_count: u64,
    findings_count: u64,
}
```

Three aggregate metrics:

- **`record_count`** -- Total number of done-ledger rows durably acknowledged. Includes all statuses: failures, skips, and scanned.
- **`scanned_count`** -- Subset of `record_count` whose status reached a terminal scanned state (`ScannedClean` or `ScannedWithFindings`). This is the number that matters for deduplication correctness -- these entries will suppress future re-scans.
- **`findings_count`** -- Aggregate number of findings referenced across all committed rows. This is the metric that feeds the pipeline's progress tracking: "how many findings has this batch contributed to the durable store?"

The receipt is backend-neutral. It carries no transaction identifiers, no implementation-specific metadata, no retry handles. It is purely a summary of the durable outcome. Callers that need more detail (e.g., per-record merge results) must query the done-ledger after the receipt is obtained.

The `DoneLedgerCommitReceipt` composes into higher-level receipts. An `ItemCommitReceipt` bundles a `FindingsCommitReceipt` and a `DoneLedgerCommitReceipt` to prove that both findings and done-ledger rows for a page's items are durable. A `PageCommitReceipt` adds a `CheckpointCommitReceipt` on top, proving the cursor checkpoint is also durable. This compositional hierarchy mirrors the page-commit protocol covered in Chapter 4.

## Summary

The done-ledger is a single-purpose persistence contract: track which object-versions have been processed, under which policy, for which tenant, and ensure that concurrent writes converge deterministically. The type system encodes every invariant:

| Type | Role |
|------|------|
| `DoneLedgerKey` | Composite deduplication key: `(tenant, policy, ovid_hash)` |
| `OvidHash` | 32-byte BLAKE3 hash of `(StableItemId, VersionId)` with claim-strength discrimination |
| `DoneLedgerStatus` | Join-semilattice with 5 variants, `merge = max(rank)` |
| `DoneLedgerRecord` | Full row: key + status + metrics + provenance + error code, with cross-field invariants |
| `DoneLedgerProvenance` | Audit trail: run, shard, fence epoch, time window |
| `DoneLedgerErrorCode` | Validated ASCII-safe string, max 128 bytes, restricted character set |
| `DoneLedger` trait | Backend contract: `batch_get` + `batch_upsert` with lattice merge enforcement |
| `DoneLedgerCommitReceipt` | Durability proof: record count, scanned count, findings count |

The lattice merge rule -- `max(rank)` -- is the single property that makes all of this safe under concurrency. No locks, no ordering requirements, no conflict resolution logic. Two workers can write contradictory statuses for the same key, and the system converges to the correct answer because the merge function is idempotent, commutative, and monotonic.

## What's Next

The done-ledger records what was scanned. But scanning produces findings -- secrets, credentials, API keys -- that must themselves be persisted durably before the done-ledger entry can be written. Chapter 3 covers the **findings sink**: the three-layer data model (`FindingRecord`, `OccurrenceRecord`, `ObservationRecord`) and the trait contract that governs how findings reach durable storage. The done-ledger and findings sink are co-committed as part of the page-commit protocol: findings are written first, then done-ledger entries, then the cursor checkpoint. This ordering ensures that a done-ledger entry is never written for an object whose findings have not yet been durably persisted.
