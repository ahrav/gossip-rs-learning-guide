# Chapter 10: The Validation Layer -- How Safety Is Enforced

## Introduction

The coordination protocol has many safety invariants. Leases must not be used
after expiry. Cursors must never regress. Terminal shards must reject all
mutations. Tenants must never see each other's data. Fence epochs must match.
Op-log entries must be unique.

How does the implementation enforce all of these? Through a **layered
validation architecture** that separates shared precondition checks from
operation-specific logic. Instead of each operation reimplementing the same
validation sequence, the implementation extracts every shared check into pure
functions in a dedicated module:

> `crates/gossip-coordination/src/validation.rs`

These functions take immutable references to the shard record and return
`Result<(), CoordError>`. They perform no mutations. They have no side
effects. Backend implementations call them in a prescribed order, convert the
shared `CoordError` into operation-specific error types via `From` impls, and
only then proceed to mutate state.

This chapter walks through the complete validation architecture: the
composition order, the ten structural invariants enforced on every shard
record, the lease validation decision tree, the error taxonomy with its
compile-time exhaustiveness guarantees, the security-conscious redaction
policy, and the idempotent outcome wrapper.

---

## Validation Composition Order

A lease-gated mutation (checkpoint, complete, park, split) follows a strict
three-step validation chain. The ordering is not arbitrary -- each step's
position is chosen to guarantee specific safety and security properties.

```text
  Incoming operation (e.g., checkpoint)
           |
           v
  +---------------------------+
  | 1. check_op_idempotency   |  <-- Replay detection FIRST
  |    Is this OpId in the    |      Replays succeed even after
  |    op-log?                |      lease expiry or terminal status
  +---------------------------+
           |
           | Ok(None) = new op
           v
  +---------------------------+
  | 2. validate_lease         |  <-- Shared preconditions
  |    tenant -> terminal ->  |      Priority-ordered checks
  |    fence -> expiry ->     |
  |    owner                  |
  +---------------------------+
           |
           | Ok(()) = lease valid
           v
  +---------------------------+
  | 3. Operation-specific     |  <-- e.g., validate_cursor_update_pooled
  |    validation             |      for checkpoint/complete
  +---------------------------+
           |
           | Ok(()) = all clear
           v
       Mutate state
```

### Why Idempotency Comes First

Step 1 (`check_op_idempotency`) is checked first on every idempotent path
so that a successful replay is never blocked by an expired lease or terminal
status. Consider the sequence:

1. Worker sends checkpoint with OpId=X. It succeeds.
2. The lease expires.
3. Worker retries checkpoint with the same OpId=X (network hiccup).

If we checked the lease before idempotency, the retry would fail with
`LeaseExpired`. But the operation already succeeded -- the worker just did not
receive the acknowledgment. Checking idempotency first returns the cached
result, and the worker proceeds without needing to re-acquire.

The module documentation makes this explicit:

```rust
//! A lease-gated mutation (e.g. checkpoint, complete) typically chains:
//!
//! 1. **[`check_op_idempotency`]** -- replay detection first, so replays
//!    succeed even after lease expiry or terminal status.
//! 2. **[`validate_lease`]** -- tenant, terminal, fence, expiry checks.
//! 3. **Operation-specific validation** (e.g. [`validate_cursor_update_pooled`]
//!    for checkpoint).
```

### The Three Validation Functions

**`check_op_idempotency`** searches the shard record's op-log for the
given `OpId`:

```rust
pub fn check_op_idempotency(
    record: &ShardRecord,
    op_id: OpId,
    payload_hash: u64,
) -> Result<Option<&OpLogEntry>, CoordError> {
    // Precondition: payload hash must be non-zero.
    assert!(
        payload_hash != 0,
        "check_op_idempotency: payload_hash must be non-zero"
    );

    let Some(entry) = record.op_log_lookup(op_id) else {
        return Ok(None);
    };

    if entry.payload_hash() == payload_hash {
        Ok(Some(entry))
    } else {
        Err(CoordError::OpIdConflict {
            op_id,
            expected_hash: entry.payload_hash(),
            actual_hash: payload_hash,
        })
    }
}
```

Three outcomes:
- `Ok(None)` -- the OpId is not in the log. This is a new operation;
  proceed to step 2.
- `Ok(Some(entry))` -- the OpId is found and the payload hash matches.
  This is an idempotent replay. Return the cached result immediately.
- `Err(CoordError::OpIdConflict)` -- the OpId is found but the payload
  hash differs. This is a mutation conflict -- the caller reused an OpId
  for a semantically different operation. Reject.

Note the precondition assertion: `payload_hash` must be non-zero. A zero
hash indicates broken hashing at the call site, which is a programming
error, not a protocol error. The function panics rather than silently
accepting a sentinel value that could match anything.

**`validate_lease`** performs five checks in priority order (detailed in
the next section).

**`validate_cursor_update_pooled`** enforces cursor constraints for checkpoint
and complete operations. It performs four checks (with a token-size sub-check) in sequence:

```rust
pub fn validate_cursor_update_pooled(
    new_cursor: &CursorUpdate<'_>,
    old_last_key: Option<&[u8]>,
    spec_start: &[u8],
    spec_end: &[u8],
) -> Result<(), CoordError> {
    // 1. Key presence -- an initial (keyless) cursor means no data
    //    has been processed yet, so there is nothing to checkpoint.
    let Some(new_last_key) = new_cursor.last_key() else {
        return Err(CoordError::CheckpointMissingKey);
    };

    // 2. Key size -- defense-in-depth against oversized keys.
    if new_last_key.len() > MAX_KEY_SIZE {
        return Err(CoordError::CursorKeyTooLarge {
            size: new_last_key.len(),
            max: MAX_KEY_SIZE,
        });
    }

    // 2b. Token size -- defense-in-depth against oversized tokens.
    if let Some(token) = new_cursor.token()
        && token.len() > MAX_TOKEN_SIZE
    {
        return Err(CoordError::CursorTokenTooLarge {
            size: token.len(),
            max: MAX_TOKEN_SIZE,
        });
    }

    // 3. Monotonicity -- new.last_key must not regress below old.last_key.
    if let Some(old_key) = old_last_key
        && new_last_key < old_key
    {
        return Err(CoordError::CursorRegression {
            old_key: Some(old_key.len()),
            new_key: Some(new_last_key.len()),
        });
    }

    // 4. Bounds checking -- new.last_key must be in [spec.start, spec.end).
    if (!spec_start.is_empty() && new_last_key < spec_start)
        || (!spec_end.is_empty() && new_last_key >= spec_end)
    {
        return Err(CoordError::CursorOutOfBounds(
            CursorOutOfBoundsDetail {
                last_key: new_last_key.len(),
                spec_start: spec_start.len(),
                spec_end: spec_end.len(),
            },
        ));
    }

    Ok(())
}
```

This function operates entirely on borrowed slices -- it takes the previous
cursor key and shard spec boundaries as raw `&[u8]` rather than owned
`&Cursor` and `&ShardSpec` references. This avoids materializing
heap-allocated types on every checkpoint, consistent with the project's
allocation policy for the HOT checkpoint path. The `_pooled` suffix in the
name reflects this slab-friendly design.

The four checks (with a token-size sub-check on check 2) correspond to
five distinct failure modes: the worker tried to checkpoint without
processing anything (missing key), the key exceeds `MAX_KEY_SIZE` (key
too large), the cursor token exceeds `MAX_TOKEN_SIZE` (token too large),
the worker regressed to an earlier position (cursor regression), or the
worker advanced past the end of its assigned key range (out of bounds).
Each produces a different `CoordError` variant, giving the caller precise
diagnostics about what went wrong.

### How the Three Steps Work Together

The three validation functions are designed to compose without overlap.
Each function checks a distinct set of preconditions:

| Function                  | What it checks                     | State accessed |
|---------------------------|------------------------------------|--------------------|
| `check_op_idempotency`   | Op-log replay/conflict             | `record.op_log`    |
| `validate_lease`          | Tenant, terminal, fence, expiry    | `record.tenant`, `record.status`, `record.fence_epoch`, `record.lease` |
| `validate_cursor_update_pooled`  | Key presence, key size, token size, monotonicity, bounds | `record.cursor`, `record.spec` |

No field is checked twice across the three functions. This means the
ordering is not just about priority -- it is about separation of concerns.
The idempotency check reads the op-log. The lease check reads ownership
fields. The cursor check reads progress fields. They are orthogonal.

This orthogonality has a practical consequence: if you are adding a new
operation that does not advance a cursor (like `park_shard`), you skip step
3 entirely. The first two steps still apply identically. The validation
layer does not force you to check things that are irrelevant to your
operation.

---

## The Ten ShardRecord Invariants

Every `ShardRecord` carries ten structural invariants, checked by
`assert_invariants()` after every state transition. The implementation
splits them into two groups: lifecycle invariants (INV-1 through INV-5)
and lineage invariants (INV-6 through INV-10).

### Lifecycle Invariants

```rust
fn assert_lifecycle_invariants(&self) {
    // INV-1: park_reason consistency.
    match self.status {
        ShardStatus::Parked => {
            assert!(
                self.park_reason.is_some(),
                "Parked shard {:?} must have park_reason",
                self.shard,
            );
        }
        _ => {
            assert!(
                self.park_reason.is_none(),
                "Non-parked shard {:?} (status: {:?}) must not have park_reason",
                self.shard, self.status,
            );
        }
    }

    // INV-2: (structural -- Option<LeaseHolder> makes paired-ness implicit)

    // INV-3: Terminal shards must not hold a lease.
    if self.status.is_terminal() {
        assert!(
            self.lease.is_none(),
            "Terminal shard {:?} (status: {:?}) must not have a lease",
            self.shard, self.status,
        );
    }

    // INV-4: Fence epoch minimum.
    assert!(
        self.fence_epoch >= FenceEpoch::INITIAL,
        "Shard {:?}: fence_epoch must be >= INITIAL (1)",
        self.shard,
    );

    // INV-5: Op-log bounded.
    assert!(
        self.op_log.len() <= Self::OP_LOG_CAP,
        "Shard {:?}: op_log length {} exceeds cap {}",
        self.shard, self.op_log.len(), Self::OP_LOG_CAP,
    );
}
```

Let us walk through each:

**INV-1: `park_reason.is_some()` iff `status == Parked`.**
This is a biconditional. If a shard is Parked, there must be a reason
(PermissionDenied, NotFound, Poisoned, TooManyErrors, or Other). If a
shard is not Parked, there must be no reason. The "iff" structure catches
two classes of bugs: forgetting to set the reason when parking, and
forgetting to clear the reason when unparking.

**INV-2: Structural -- enforced by `Option<LeaseHolder>`.**
This invariant is not checked at runtime because the type system
enforces it. `Option<LeaseHolder>` guarantees that the lease holder's
owner (WorkerId) and deadline (LogicalTime) are always present together
or both absent.

**INV-3: Terminal shards must not hold a lease.**
Done, Split, and Parked shards have no active owner. If a shard becomes
terminal while leased, the operation must clear the lease as part of the
transition.

**INV-4: `fence_epoch >= FenceEpoch::INITIAL`.**
The fence epoch starts at `INITIAL` (value 1) and only goes up.

**INV-5: `op_log.len() <= OP_LOG_CAP`.**
The op-log is a `RingBuffer<OpLogEntry, 16>`, which structurally
enforces capacity. This assertion is defense-in-depth.

### Lineage Invariants

```rust
fn assert_lineage_invariants(&self, slab: &ByteSlab) {
    // INV-6: Split implies spawned is non-empty.
    if self.status == ShardStatus::Split {
        assert!(
            !self.spawned.is_empty(),
            "Split shard {:?} must have spawned children",
            self.shard,
        );
    }

    // INV-7: parent.is_some() iff shard.is_derived().
    if self.parent.is_some() {
        assert!(
            self.shard.is_derived(),
            "Shard {:?} claims parentage but is not derived (bit 63 not set)",
            self.shard,
        );
    }
    if self.shard.is_derived() {
        assert!(
            self.parent.is_some(),
            "Shard {:?}: derived (bit 63 set) but has no parent",
            self.shard,
        );
    }

    // INV-8: All spawned entries must be derived.
    for (i, spawned_id) in self.spawned.iter(slab).enumerate() {
        assert!(
            spawned_id.is_derived(),
            "Shard {:?}: spawned[{i}] ({:?}) is not derived (bit 63 not set)",
            self.shard, spawned_id,
        );
    }

    // INV-9: Op-log entries have unique OpId values.
    // (Enforced in two places: here via an O(n²) scan, and in
    // `ShardRecord::op_log_push` via an internal duplicate assertion.)

    // INV-10: Spawned count bounded.
    assert!(
        self.spawned.len() <= MAX_SPAWNED_PER_SHARD,
        "Shard {:?}: spawned count {} exceeds cap {}",
        self.shard, self.spawned.len(), MAX_SPAWNED_PER_SHARD,
    );
}
```

### The Crash-to-Prevent-Corruption Philosophy

All invariant violations cause panics. This is intentional. An invariant
violation means the coordinator's in-memory state is inconsistent with its
design contract -- continuing would risk persisting corrupt data. By
crashing *before* persistence, the coordinator ensures crash-recovery
returns to the last valid state.

---

## The `validate_lease` Decision Tree

When a lease-gated operation passes the idempotency check (step 1 returned
`Ok(None)`), it proceeds to `validate_lease`. This function performs five
checks in strict priority order:

```rust
pub fn validate_lease(
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
    record: &ShardRecord,
) -> Result<(), CoordError> {
    // Precondition: time must be positive.
    assert!(now > LogicalTime::ZERO, "validate_lease: now must be > ZERO");

    // 1. Tenant isolation.
    if record.tenant != tenant {
        return Err(CoordError::TenantMismatch { expected: tenant });
    }

    // 2. Terminal status.
    if record.status.is_terminal() {
        return Err(CoordError::ShardTerminal {
            shard: ShardKey::new(record.run, record.shard),
            status: record.status,
        });
    }

    // 3. Fence epoch.
    if lease.fence() != record.fence_epoch {
        return Err(CoordError::StaleFence {
            presented: lease.fence(),
            current: record.fence_epoch,
        });
    }

    // 4. Lease expiry (with missing-deadline defense).
    let Some(deadline) = record.lease_deadline() else {
        return Err(CoordError::StaleFence {
            presented: lease.fence(),
            current: record.fence_epoch,
        });
    };
    if now >= deadline {
        return Err(CoordError::LeaseExpired { deadline, now });
    }

    // 5. Owner divergence.
    if record.lease_owner() != Some(lease.owner()) {
        return Err(CoordError::StaleFence {
            presented: lease.fence(),
            current: record.fence_epoch,
        });
    }

    Ok(())
}
```

### Why This Ordering Matters

The check ordering is security-first. If tenant isolation were checked
after fencing, a caller who presented the wrong tenant but a valid fence
token would receive `TenantMismatch`. But a caller who presented the wrong
tenant and a stale fence would receive `StaleFence`. The difference reveals
information about shard state that should not leak across tenant boundaries.

By checking tenant first, the caller always receives `TenantMismatch`
regardless of what other conditions are violated. No cross-tenant
information leaks.

---

## Error Taxonomy

The error system is defined in:

> `crates/gossip-coordination/src/error.rs`

### The Shared Building Block: `CoordError`

`CoordError` is the central enum. All validation functions return it. Each
operation error type accepts a subset of its variants via `From<CoordError>`
impls:

```rust
#[derive(Clone, PartialEq, Eq)]
#[non_exhaustive]
pub enum CoordError {
    ShardNotFound { shard: ShardKey },
    TenantMismatch { expected: TenantId },
    StaleFence { presented: FenceEpoch, current: FenceEpoch },
    LeaseExpired { deadline: LogicalTime, now: LogicalTime },
    ShardTerminal { shard: ShardKey, status: ShardStatus },
    OpIdConflict { op_id: OpId, expected_hash: u64, actual_hash: u64 },
    CursorRegression { old_key: Option<usize>, new_key: Option<usize> },
    CursorOutOfBounds(CursorOutOfBoundsDetail),
    CursorKeyTooLarge { size: usize, max: usize },
    CursorTokenTooLarge { size: usize, max: usize },
    SplitInvalid(SplitValidationError),
    CheckpointMissingKey,
}
```

Twelve variants, covering every possible coordination failure. The enum is
`#[non_exhaustive]` so that adding variants is a non-breaking change for
downstream crate consumers.

Note the key design choices:
- **`CursorRegression`** fields are `Option<usize>` -- byte **lengths** of the old and new keys, not the raw key data. This is the redaction policy in action: raw cursor keys contain user data and must not leak via error messages.
- **`CursorOutOfBounds`** is **not boxed**: `CursorOutOfBounds(CursorOutOfBoundsDetail)`. The `CursorOutOfBoundsDetail` struct holds only three `usize` fields (byte lengths), so it is small enough to inline directly.
- **`SplitInvalid`** is also **not boxed**: `SplitInvalid(SplitValidationError)`. The `SplitValidationError` enum stores only `usize` byte lengths (not `Box<[u8]>` byte data), keeping it compact enough to inline.
- **`CursorKeyTooLarge`** and **`CursorTokenTooLarge`** guard against oversized keys and tokens before they reach slab storage, emitted by `validate_cursor_update_pooled`.

### Per-Operation Error Types

Each coordination operation has a dedicated error type:

| Operation          | Error Type        | Unique traits                                    |
|--------------------|-------------------|--------------------------------------------------|
| `acquire`          | `AcquireError`    | No `From<CoordError>` -- has `AlreadyLeased`, `BackendError(InfraError)` |
| `renew`            | `RenewError`      | Common precondition variants + `BackendError(InfraError)` |
| `checkpoint`       | `CheckpointError` | Widest surface (tied with Complete) + `BackendError(InfraError)` |
| `complete`         | `CompleteError`   | Same as Checkpoint + `BackendError(InfraError)`  |
| `park`             | `ParkError`       | No cursor variants + `BackendError(InfraError)`  |
| `split`            | `SplitError`      | Has `SplitInvalid`, no cursor variants + `BackendError(InfraError)` |

### The Variant Routing Matrix

The module documentation includes a routing matrix that shows exactly which
`CoordError` variants map to which operation errors:

```text
| CoordError variant   | Renew | Checkpoint | Complete | Park | Split |
|----------------------|:-----:|:----------:|:--------:|:----:|:-----:|
| ShardNotFound        |  yes  |    yes     |   yes    | yes  |  yes  |
| TenantMismatch       |  yes  |    yes     |   yes    | yes  |  yes  |
| StaleFence           |  yes  |    yes     |   yes    | yes  |  yes  |
| LeaseExpired         |  yes  |    yes     |   yes    | yes  |  yes  |
| ShardTerminal        |  yes  |    yes     |   yes    | yes  |  yes  |
| OpIdConflict         |   --  |    yes     |   yes    | yes  |  yes  |
| CursorRegression     |   --  |    yes     |   yes    |  --  |   --  |
| CursorOutOfBounds    |   --  |    yes     |   yes    |  --  |   --  |
| CursorKeyTooLarge    |   --  |    yes     |   yes    |  --  |   --  |
| CursorTokenTooLarge  |   --  |    yes     |   yes    |  --  |   --  |
| SplitInvalid         |   --  |     --     |    --    |  --  |  yes  |
| CheckpointMissingKey |   --  |    yes     |   yes    |  --  |   --  |
```

In addition to the `CoordError` variants routed via `From<CoordError>`, the
`CheckpointError`, `CompleteError`, and `SplitError` types each carry a
`ResourceExhausted(SlabFull)` variant with a direct `From<SlabFull>` impl
(not routed through `CoordError`). This fires when the byte slab cannot
satisfy an allocation request during a cursor or spawned-lineage update. The
error is recoverable: the caller may retry after freeing slab space.
`ParkError` does not have this variant because parking does not update cursor
or lineage slab storage.

Additionally, every operation-specific error type carries a
`BackendError(InfraError)` variant for coordination infrastructure errors.
`InfraError` is a structured enum with two variants:

- **`Transient { operation, message }`** -- retryable infrastructure failures
  (network timeouts, gRPC errors, CAS retry budget exhaustion).
- **`Corruption { operation, message }`** -- permanent data inconsistencies
  (codec decode failures, missing records, invariant violations in stored state).

The `operation` field identifies the failing step as a human-readable label
(e.g., `"acquire.load_shard"`, `"renew.txn"`). The affected error types are:
`AcquireError`, `RenewError`, `CheckpointError`, `CompleteError`, `ParkError`,
`SplitError`, `CreateRunError`, `RegisterShardsError`, `GetRunError`, and
`ClaimError`. `BackendError` is not routed through `CoordError` -- it is
defined directly on each error type. Callers can use `InfraError::is_transient()`
to decide whether to retry.

The five common variants (`ShardNotFound` through `ShardTerminal`) appear
in every column. The remaining seven are operation-specific: `OpIdConflict`
appears wherever idempotency is checked, cursor variants (including the
size-limit guards `CursorKeyTooLarge` and `CursorTokenTooLarge`) appear
only on operations that advance the cursor, `SplitInvalid` only on splits,
and `CheckpointMissingKey` only where a cursor key is required.

### Explicit Rejection Arms (No Wildcards)

The `From<CoordError>` impls enumerate all rejected variants explicitly
rather than using `_ => unreachable!()`. Here is the impl for
`CheckpointError`:

```rust
impl From<CoordError> for CheckpointError {
    fn from(e: CoordError) -> Self {
        match e {
            CoordError::ShardNotFound { shard } => Self::ShardNotFound { shard },
            CoordError::TenantMismatch { expected } => Self::TenantMismatch { expected },
            CoordError::StaleFence { presented, current } =>
                Self::StaleFence { presented, current },
            CoordError::LeaseExpired { deadline, now } =>
                Self::LeaseExpired { deadline, now },
            CoordError::ShardTerminal { shard, status } =>
                Self::ShardTerminal { shard, status },
            CoordError::OpIdConflict { op_id, expected_hash, actual_hash } =>
                Self::OpIdConflict { op_id, expected_hash, actual_hash },
            CoordError::CursorRegression { old_key, new_key } =>
                Self::CursorRegression { old_key, new_key },
            CoordError::CursorOutOfBounds(detail) =>
                Self::CursorOutOfBounds(detail),
            CoordError::CursorKeyTooLarge { size, max } =>
                Self::CursorKeyTooLarge { size, max },
            CoordError::CursorTokenTooLarge { size, max } =>
                Self::CursorTokenTooLarge { size, max },
            CoordError::CheckpointMissingKey => Self::CheckpointMissingKey,
            // Explicitly reject -- adding a new variant triggers a compile error here.
            CoordError::SplitInvalid(_) => {
                unreachable!("CoordError::{e} is not valid for CheckpointError")
            }
        }
    }
}
```

The rejected variant (`SplitInvalid`) is listed explicitly. If someone
adds a new variant to `CoordError`, every single `From` impl will produce
a compile error because the new variant is not handled. This is the
compile-time exhaustiveness guarantee.

### Type Aliases for Split Errors

Both split operations share the same error surface:

```rust
pub type SplitReplaceError = SplitError;
pub type SplitResidualError = SplitError;
```

### The `std::error::Error` Chain

`CoordError` and `SplitError` implement `std::error::Error::source()`
for the `SplitInvalid` variant:

```rust
impl std::error::Error for CoordError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::SplitInvalid(inner) => Some(inner),
            _ => None,
        }
    }
}
```

---

## Security-Conscious Redaction

Custom `Debug` and `Display` impls on error types redact sensitive fields
to prevent information leakage in logs and error messages.

### `OpIdConflict`: Hash Values Redacted

```rust
Self::OpIdConflict { op_id, .. } => f
    .debug_struct("OpIdConflict")
    .field("op_id", op_id)
    .field("expected_hash", &"<redacted>")
    .field("actual_hash", &"<redacted>")
    .finish(),
```

### `AlreadyLeased`: Current Owner Redacted

```rust
Self::AlreadyLeased { lease_deadline, .. } => f
    .debug_struct("AlreadyLeased")
    .field("current_owner", &"<redacted>")
    .field("lease_deadline", lease_deadline)
    .finish(),
```

### `CursorRegression` / `CursorOutOfBounds`: Byte Lengths Only

```rust
Self::CursorRegression { old_key, new_key } => {
    f.debug_struct("CursorRegression")
        .field("old_key", &RedactedOptionalLen(*old_key))
        .field("new_key", &RedactedOptionalLen(*new_key))
        .finish()
}
```

Since `CursorRegression` stores `Option<usize>` (byte lengths, not raw
bytes), the redaction is structural -- the raw key data never enters the
error type at all. The `RedactedOptionalLen` wrapper formats `Some(47)`
as `Some([47 bytes])` for clarity.

### `TenantMismatch`: Only Expected Shown

```rust
TenantMismatch { expected: TenantId },
```

Note the conspicuous absence of an `actual` field. The error only carries
the caller's tenant (`expected`). The record's actual tenant is
deliberately omitted to prevent cross-tenant enumeration.

---

## `IdempotentOutcome`

Every idempotent operation (checkpoint, complete, park, split_replace,
split_residual) returns its result wrapped in `IdempotentOutcome<T>`:

```rust
#[non_exhaustive]
#[must_use = "idempotent outcome should be inspected"]
pub enum IdempotentOutcome<T> {
    /// The operation was executed for the first time.
    Executed(T),
    /// The operation was a retry -- result replayed from op-log.
    Replayed(T),
}
```

The wrapper carries the same `T` in both variants. Callers who do not
care about the execution path can use `into_inner()`:

```rust
impl<T> IdempotentOutcome<T> {
    pub fn into_inner(self) -> T {
        match self {
            Self::Executed(v) | Self::Replayed(v) => v,
        }
    }

    pub fn is_replay(&self) -> bool {
        matches!(self, Self::Replayed(_))
    }

    pub fn is_executed(&self) -> bool {
        matches!(self, Self::Executed(_))
    }

    pub fn map<U>(self, f: impl FnOnce(T) -> U) -> IdempotentOutcome<U> {
        match self {
            Self::Executed(v) => IdempotentOutcome::Executed(f(v)),
            Self::Replayed(v) => IdempotentOutcome::Replayed(f(v)),
        }
    }
}
```

---

## Connecting the Layers

The validation layer is the enforcement backbone of the coordination
protocol. Every mutation passes through the same sequence:

1. Idempotency check (pure, read-only scan of op-log)
2. Lease validation (pure, priority-ordered check sequence)
3. Operation-specific validation (pure, cursor or split checks)
4. State mutation (writes to `ShardRecord` fields)
5. Op-log push (records the operation for future idempotency)
6. Invariant assertion (panics if any of the 10 invariants are violated)

Steps 1-3 are the validation layer. Step 4 is the operation. Steps 5-6
are the post-mutation safety net. The validation functions are shared
across all backends. The invariant assertions run identically in the
in-memory reference, in a PostgreSQL backend, and in the deterministic
simulation.

In the next chapter, we will see this layered validation in action as we
walk through the `InMemoryCoordinator`'s complete implementation of every
coordination operation.

---

**Source files referenced in this chapter:**

- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/validation.rs`
- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/record.rs`
- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/error.rs`
