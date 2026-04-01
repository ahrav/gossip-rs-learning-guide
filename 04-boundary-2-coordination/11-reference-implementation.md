# Chapter 11: The Reference Implementation -- InMemoryCoordinator Walk-Through

## Introduction

The `InMemoryCoordinator` is the executable specification of the shard
coordination protocol. Every protocol rule -- fencing, leases, idempotency,
cursor monotonicity, split coverage, invariant enforcement -- is implemented
here first. Production backends (PostgreSQL, DynamoDB, etc.) must produce
identical observable behavior for the same input sequences.

The implementation makes three deliberate design choices that simplify
reasoning about correctness:

**Single-threaded (`&mut self`).**
Every operation takes `&mut self`, which means the Rust borrow checker
serializes all mutations at compile time. There are no locks, no
`RwLock<InnerState>` wrappers, no `Arc` indirection. This eliminates
concurrency concerns entirely, so invariants can be verified inline
without worrying about interleaving.

**Purely in-memory (`AHashMap`).**
The data structures are two-level hash maps: `AHashMap<TenantId,
AHashMap<ShardKey, ShardRecord>>` for shards and `AHashMap<(TenantId,
RunId), RunRecord>` for runs. No disk I/O, no transactions, no WAL. Every
state transition is a direct field mutation on a struct.

**Tiger-style invariant enforcement.**
Every mutation path calls `ShardRecord::assert_invariants(&slab)` before
returning. If any of the ten structural invariants are violated, the
coordinator panics immediately. This is crash-to-prevent-corruption:
the invariant violation is caught before any corrupt state can be
persisted, and on crash-recovery the shard returns to its pre-operation
state.

```text
crates/gossip-coordination/src/in_memory.rs
```

---

## Data Structures

### The `InMemoryCoordinator` Struct

```rust
pub struct InMemoryCoordinator {
    // Two-level shard map: tenant -> (shard_key -> record).
    pub(crate) shards: AHashMap<TenantId, AHashMap<ShardKey, ShardRecord>>,
    // Global shard count, maintained inline on insert/remove.
    pub(crate) total_shard_count: usize,
    // Run records keyed by (tenant, run).
    pub(crate) runs: AHashMap<(TenantId, RunId), RunRecord>,
    // Secondary index: run -> shard IDs (root + split children).
    pub(crate) run_shards: AHashMap<(TenantId, RunId), HashSet<ShardId, ahash::RandomState>>,
    // Duration applied to every new lease.
    pub(crate) default_lease_duration: u64,
    // Per-tenant shard cap.
    pub(crate) max_shards_per_tenant: usize,
    // Global shard cap.
    pub(crate) max_total_shards: usize,
    // Per-worker claim cooldown: worker -> last successful claim time.
    pub(crate) claim_cooldowns: AHashMap<WorkerId, LogicalTime>,
    // Minimum time between successive claims by the same worker.
    pub(crate) claim_cooldown_interval: u64,
    // Arena allocator for pooled ShardRecord fields (spec + cursor).
    pub(crate) slab: ByteSlab,
    // Reusable shard-id candidate buffer for claim hot path.
    pub(crate) claim_candidates_scratch: Vec<ShardId>,
}
```

### The Two-Level Shard Map

The outer map is keyed by `TenantId`. This provides O(1) tenant isolation:
a lookup with the wrong tenant misses at the outer map without scanning any
shard records. No cross-tenant information leaks.

The inner map is keyed by `ShardKey` (a composite of `RunId` and `ShardId`).
This reduces hash input from 48 bytes (if we used a composite key including
the tenant) to 16 bytes.

`total_shard_count` is maintained inline -- incremented on insert, decremented
on remove -- so that global limit checks remain O(1) instead of O(T) where
T is the number of tenants. An assertion after every mutation verifies that
the counter stays synchronized:

```rust
fn assert_shard_count(&self) {
    assert_eq!(
        self.total_shard_count,
        self.shards.values().map(|m| m.len()).sum::<usize>(),
        "total_shard_count drift detected"
    );
}
```

### Runtime Configuration and Slab Sizing

The coordinator is constructed via `CoordinatorRuntimeConfig`, which includes
a `slab_capacity` field controlling the `ByteSlab` size:

```rust
pub struct CoordinatorRuntimeConfig {
    pub default_lease_duration: u64,
    pub max_shards_per_tenant: usize,
    pub max_total_shards: usize,
    pub claim_cooldown_interval: u64,
    /// Byte slab capacity. Zero uses auto-sized default:
    /// min(max_total_shards * 4096, 64 MiB).
    pub slab_capacity: usize,
}
```

When `slab_capacity` is zero (the default), the coordinator derives a capacity
from `max_total_shards * 4_096` (the default per-shard budget), capped at 64 MiB
to prevent pathological eager allocation. Explicit values are clamped to the
`ByteSlab` backend's `u32`-addressable maximum.

### Helper Methods

The two-level map is accessed through four internal methods that maintain
the `total_shard_count` invariant:

```rust
fn shard_get(&self, tenant: &TenantId, key: &ShardKey) -> Option<&ShardRecord>;
fn shard_get_mut(&mut self, tenant: &TenantId, key: &ShardKey) -> Option<&mut ShardRecord>;
fn shard_insert(&mut self, tenant: TenantId, key: ShardKey, record: ShardRecord);
fn shard_remove(&mut self, tenant: &TenantId, key: &ShardKey) -> Option<ShardRecord>;
```

`shard_insert` has upsert semantics: if a record already exists at
`(tenant, key)`, it is replaced and the count is unchanged. This is
intentional for the remove-mutate-restore pattern used by split operations,
where a parent is removed, mutated, then re-inserted at the same key.

`shard_remove` uses `checked_sub` for the decrement. An underflow would
indicate the counter drifted from the map contents, which is always a bug:

```rust
fn shard_remove(&mut self, tenant: &TenantId, key: &ShardKey) -> Option<ShardRecord> {
    let inner = self.shards.get_mut(tenant)?;
    let removed = inner.remove(key);
    if removed.is_some() {
        self.total_shard_count = self
            .total_shard_count
            .checked_sub(1)
            .expect("total_shard_count underflow");
    }
    self.assert_shard_count();
    removed
}
```

### Shard Count Limits

`check_shard_limits` enforces both per-tenant and global shard caps before
creating new shards. It takes a `temporarily_removed` parameter to account
for the remove-mutate-restore pattern in split operations:

```rust
fn check_shard_limits(
    &self,
    tenant: TenantId,
    additional: usize,
    temporarily_removed: usize,
) -> Result<(), SplitValidationError> {
    let tenant_count = self.shards.get(&tenant)
        .map_or(0, |m| m.len())
        .saturating_add(temporarily_removed);
    if tenant_count.saturating_add(additional) > self.max_shards_per_tenant {
        return Err(SplitValidationError::ShardLimitExceeded { /* ... */ });
    }
    let total_count = self.total_shard_count.saturating_add(temporarily_removed);
    if total_count.saturating_add(additional) > self.max_total_shards {
        return Err(SplitValidationError::ShardLimitExceeded { /* ... */ });
    }
    Ok(())
}
```

When `split_replace` removes the parent from the map before validating,
the parent's record no longer contributes to the map-based count.
`temporarily_removed=1` adds it back to the accounting so that the limit
check is accurate.

---

## Tracing a Complete Operation Sequence

Let us trace a worker's lifecycle through the in-memory backend: create a
run, register shards, acquire a shard, checkpoint progress, and complete
the shard.

### Step 1: `create_run` + `register_shards`

Before any shard operations can occur, a run must be created and populated
with its initial shard manifest. This is a two-phase process:

1. `create_run(now, tenant, run, config)` creates a `RunRecord` in
   `Initializing` status with no shards.
2. `register_shards(now, tenant, run, shards, op_id)` validates the
   manifest, creates `ShardRecord`s for each initial shard, updates the
   `run_shards` secondary index, and transitions the run to `Active`.

The two-phase design separates run identity creation from shard population.
This allows the orchestrator to create a run, compute the shard manifest
externally (e.g., by enumerating a repository's file tree), and then
register the shards in a separate call. If the shard enumeration fails, the
`Initializing` run can be cleaned up via `cancel_run` without ever having
created any shard records.

The register step is idempotent. The run op-log stores the `RegisterShards`
entry with the result (`RegisteredShards { shard_ids }`) so that retries
return the same shard IDs without re-creating records. This is important
because `register_shards` creates `ShardRecord`s with real state -- if a
retry re-created them, you would have duplicate shards covering the same
key ranges.

The validation sequence within `register_shards` is worth noting because
it follows the same idempotency-first principle as shard operations:

1. Lookup + tenant isolation
2. Idempotency (before status check)
3. Status check (`Initializing` only)
4. Manifest validation (uniqueness, non-empty)
5. Shard count limit guard
6. Shard record creation + index update
7. Run status transition to `Active`

Step 2 before step 3 means a replay succeeds even if the run has since
transitioned to `Active` (which is the expected state after a successful
first execution).

### Step 2: `acquire_and_restore_into`

This is where a worker takes exclusive ownership of a shard. The
implementation follows the six-step sequence documented in the trait:

```rust
fn acquire_and_restore_into<'a>(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    key: ShardKey,
    worker: WorkerId,
    out: &'a mut AcquireScratch,
) -> Result<AcquireResultView<'a>, AcquireError> {
    let lease_duration = self.default_lease_duration;
    let record = self
        .shard_get_mut(&tenant, &key)
        .ok_or(AcquireError::ShardNotFound { shard: key })?;

    // 1) Tenant isolation (defense-in-depth -- the two-level map
    //    already provides structural tenant separation).
    if record.tenant != tenant {
        return Err(AcquireError::TenantMismatch { expected: tenant });
    }

    // 2) Terminal shards cannot be acquired.
    if record.status != ShardStatus::Active {
        return Err(AcquireError::ShardTerminal {
            shard: key,
            status: record.status,
        });
    }

    // 3) Active lease -- must wait for expiry rather than preempt.
    if record.is_leased_at(now) {
        let (owner, deadline) = record
            .lease
            .as_ref()
            .map(|h| (h.owner(), h.deadline()))
            .expect("lease must exist when is_leased_at returns true");
        return Err(AcquireError::AlreadyLeased {
            current_owner: owner,
            lease_deadline: deadline,
        });
    }

    // 4) Bump fence epoch (Kleppmann fencing token).
    let new_fence = record.advance_fence();

    // 5) Grant a new lease with a fresh deadline.
    let deadline = now
        .checked_add(lease_duration)
        .unwrap_or(LogicalTime::from_raw(u64::MAX));
    record.lease = Some(LeaseHolder::new(worker, deadline));

    // 6) Return the fencing lease + shard snapshot + capacity hint.
    let lease = Lease::new(tenant, key.run(), key.shard(), worker, new_fence, deadline);
    out.reset();
    out.write_spec(
        record.spec.key_range_start(&self.slab),
        record.spec.key_range_end(&self.slab),
        record.spec.metadata(&self.slab),
    );
    out.write_cursor(
        record.cursor.last_key(&self.slab),
        record.cursor.token(&self.slab),
    );
    out.write_spawned_iter(record.spawned.iter(&self.slab));
    let snapshot = out.view(record.status, record.cursor_semantics, record.parent);
    let capacity = self.count_available_for_run(now, tenant, key.run());
    Ok(AcquireResultView { lease, snapshot, capacity })
}
```

Walking through the six steps:

**Shard lookup.** The two-level map provides O(1) lookup with structural
tenant isolation. If the shard does not exist under this tenant, the outer
map miss returns `ShardNotFound` without revealing anything about other
tenants.

**Tenant check (defense-in-depth).** Even though the two-level map
structurally isolates tenants, the record's `tenant` field is checked
against the request's tenant. This catches internal consistency bugs
where the record's tenant field disagrees with its map key.

**Terminal check.** Only `Active` shards can be acquired. Note that
`Parked` shards are terminal within the coordination protocol -- they
can only be resumed via the admin `unpark_shard` operation on the
`RunManagement` trait, not through `acquire_and_restore_into`.

**Live lease check.** `is_leased_at(now)` checks whether another worker
currently holds a valid lease (`now < deadline`). If so, the operation
fails with `AlreadyLeased` rather than preempting. This preserves the
at-most-once processing guarantee within a lease window. The caller
receives the deadline so it knows when to retry.

**Fence epoch bump.** `record.advance_fence()` increments the monotonic
fence epoch. Any worker from a prior epoch that tries to checkpoint or
complete will be rejected by `validate_lease`'s fence check. This is
the Kleppmann fencing token pattern.

**Lease installation.** A new `LeaseHolder` is created with the
worker's identity and a deadline computed as `now + lease_duration`.
The deadline computation saturates at `u64::MAX` rather than panicking
on overflow -- a very-long lease is safe because it will still expire
eventually or be superseded by a fence bump.

**Snapshot creation.** The caller-owned `AcquireScratch` (`out`) is used to
materialize the shard's spec, cursor, and lineage into a read-only
`ShardSnapshotView`. The three-step write pattern (`write_spec`,
`write_cursor`, `write_spawned_iter`) populates the scratch buffer, and
`out.view()` produces a borrowed view over it. The snapshot borrows from
`out`, so callers must consume or copy it before reusing the scratch buffer.

**Capacity hint.** `count_available_for_run` scans all shards in the
run to count how many are active and unleased. This is O(S) but
acceptable for a reference implementation. Production backends should
maintain a running counter.

### Step 3: `checkpoint`

Checkpoint persists a new cursor position. It is the canonical example
of the three-step validation chain
(see [Chapter 10: The Validation Layer](./10-validation-layer.md)):

```rust
fn checkpoint(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
    new_cursor: &CursorUpdate<'_>,
    op_id: OpId,
) -> Result<IdempotentOutcome<()>, CheckpointError> {
    let key = lease.shard_key();
    let record = self
        .shard_get_mut(&tenant, &key)
        .ok_or(CheckpointError::ShardNotFound { shard: key })?;

    // Step 1: Idempotency check BEFORE lease validation.
    let payload_hash = hash_checkpoint_payload(&new_cursor);
    if check_op_idempotency(record, op_id, payload_hash)?.is_some() {
        return Ok(IdempotentOutcome::Replayed(()));
    }

    // Step 2: Lease validation (tenant, terminal, fence, expiry, owner).
    validate_lease(now, tenant, lease, record)?;

    // Step 3: Operation-specific validation (cursor monotonicity + bounds).
    // Uses the `_pooled` variant to borrow key bytes directly from the slab,
    // avoiding per-call materialization of owned ShardSpec values.
    validate_cursor_update_pooled(
        &new_cursor,
        record.cursor.last_key(&self.slab),
        record.spec.key_range_start(&self.slab),
        record.spec.key_range_end(&self.slab),
    )?;

    // --- All validation passed. Mutate state. ---
    record.cursor.update_from_ref(new_cursor, &mut self.slab)?;
    record.op_log_push(OpLogEntry::new(
        op_id,
        OpKind::Checkpoint,
        OpResult::Completed,
        payload_hash,
        now,
    ));

    record.assert_invariants(&self.slab);
    Ok(IdempotentOutcome::Executed(()))
}
```

The mutation sequence is minimal: update the cursor, push an op-log entry,
assert invariants. The op-log push is always the last mutation step before
the invariant check. This ordering matters because `op_log_push` itself
asserts that the OpId is not already in the log (defense-in-depth against
the idempotency check being bypassed).

Notice how the `?` operator on `check_op_idempotency` and `validate_lease`
converts `CoordError` into `CheckpointError` via the `From<CoordError>`
impl. The validation functions return `CoordError`; the operation converts
it to its specific error type. This is the shared building block
architecture described in
[Chapter 10](./10-validation-layer.md).

### Step 4: `complete`

Complete marks a shard as fully processed. It follows the same three-step
pattern as checkpoint, with an additional status transition:

```rust
fn complete(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
    final_cursor: &CursorUpdate<'_>,
    op_id: OpId,
) -> Result<IdempotentOutcome<()>, CompleteError> {
    let key = lease.shard_key();
    let record = self
        .shard_get_mut(&tenant, &key)
        .ok_or(CompleteError::ShardNotFound { shard: key })?;

    // Idempotency first.
    let payload_hash = hash_complete_payload(&final_cursor);
    if check_op_idempotency(record, op_id, payload_hash)?.is_some() {
        return Ok(IdempotentOutcome::Replayed(()));
    }

    // Lease validation.
    validate_lease(now, tenant, lease, record)?;

    // Cursor validation (pooled variant -- borrows slab bytes directly).
    validate_cursor_update_pooled(
        &final_cursor,
        record.cursor.last_key(&self.slab),
        record.spec.key_range_start(&self.slab),
        record.spec.key_range_end(&self.slab),
    )?;

    // --- Mutate state ---
    record.cursor.update_from_ref(final_cursor, &mut self.slab)?;
    record.assert_transition_legal(ShardStatus::Done);
    record.status = ShardStatus::Done;
    record.lease = None;  // Terminal shards release their lease.
    record.op_log_push(OpLogEntry::new(
        op_id,
        OpKind::Complete,
        OpResult::Completed,
        payload_hash,
        now,
    ));

    record.assert_invariants(&self.slab);
    Ok(IdempotentOutcome::Executed(()))
}
```

The differences from checkpoint:

1. **`assert_transition_legal(ShardStatus::Done)`** verifies that
   transitioning from the current status to Done is legal. Only Active
   shards can transition to Done. If someone calls complete on a Parked
   shard (which would have been caught by the terminal check in
   `validate_lease`), this assertion provides a second line of defense.

2. **`record.status = ShardStatus::Done`** transitions the shard to its
   terminal state.

3. **`record.lease = None`** releases the lease. INV-3 (terminal shards
   must not hold a lease) is enforced by `assert_invariants(&slab)` on the
   next line. If we forgot to clear the lease, the invariant check would
   panic.

4. **The op-log entry uses `OpKind::Complete`** instead of
   `OpKind::Checkpoint`. Because the shard is now terminal, no further
   operations can push entries to the op-log. The complete entry will
   never be evicted, which means idempotent replays are guaranteed to
   find it regardless of how many subsequent operations occur (there
   are none).

### Step 5: `park_shard`

Park suspends a shard so it is no longer eligible for acquisition:

```rust
fn park_shard(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
    reason: ParkReason,
    op_id: OpId,
) -> Result<IdempotentOutcome<()>, ParkError> {
    let key = lease.shard_key();
    {
        let record = self
            .shard_get_mut(&tenant, &key)
            .ok_or(ParkError::ShardNotFound { shard: key })?;

        let payload_hash = hash_park_payload(reason);
        if check_op_idempotency(record, op_id, payload_hash)?.is_some() {
            return Ok(IdempotentOutcome::Replayed(()));
        }

        validate_lease(now, tenant, lease, record)?;

        record.assert_transition_legal(ShardStatus::Parked);
        record.status = ShardStatus::Parked;
        record.park_reason = Some(reason);
        record.lease = None;
        record.op_log_push(OpLogEntry::new(
            op_id, OpKind::Park, OpResult::Completed, payload_hash, now,
        ));
    }
    self.shard_get(&tenant, &key)
        .expect("parked shard must exist")
        .assert_invariants(&self.slab);
    Ok(IdempotentOutcome::Executed(()))
}
```

Park follows the same pattern as complete, but with two differences:

1. **No cursor validation.** Park does not advance the cursor -- it
   freezes the shard at its current position. The `ParkError` type
   accordingly has no `CursorRegression` or `CursorOutOfBounds` variants.

2. **`park_reason` is set.** INV-1 requires that Parked shards have a
   reason. The `ParkReason` enum captures coordination-level categories:
   `PermissionDenied`, `NotFound`, `Poisoned`, `TooManyErrors`, or
   `Other`. These are broad enough to inform retry decisions (e.g.,
   `TooManyErrors` is suitable for time-delayed auto-retry) without
   encoding detailed error descriptions in the coordination state.

### Step 6: `renew`

Renew extends an existing lease without modifying shard progress:

```rust
fn renew(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
) -> Result<RenewResult, RenewError> {
    let key = lease.shard_key();
    let lease_duration = self.default_lease_duration;
    let deadline = {
        let record = self
            .shard_get_mut(&tenant, &key)
            .ok_or(RenewError::ShardNotFound { shard: key })?;

        validate_lease(now, tenant, lease, record)?;

        let deadline = now
            .checked_add(lease_duration)
            .unwrap_or(LogicalTime::from_raw(u64::MAX));
        record.lease = Some(LeaseHolder::new(lease.owner(), deadline));
        deadline
    };
    self.shard_get(&tenant, &key)
        .expect("renewed shard must exist")
        .assert_invariants(&self.slab);
    let capacity = self.count_available_for_run(now, tenant, key.run());
    Ok(RenewResult { new_deadline: deadline, capacity })
}
```

Renew is the simplest lease-gated operation. It has no idempotency check
(renew is not idempotent -- each call extends the deadline anew). It has
no operation-specific validation (no cursor, no status transition). Just
lease validation followed by a deadline update.

Note the scoped borrow pattern: the mutable borrow of `record` is confined
inside the `{ ... }` block. After the block ends, `assert_invariants` is
called via a fresh shared borrow from `shard_get`. This avoids holding
`&mut record` (which borrows the shard map) across the `assert_invariants`
call that needs `&self.slab`. The same pattern appears in `park_shard`
and `unpark_shard`.

The `RenewError` type is correspondingly narrow: it carries the
five common precondition variants (`ShardNotFound`, `TenantMismatch`,
`StaleFence`, `LeaseExpired`, `ShardTerminal`) plus `BackendError(InfraError)`
for infrastructure failures (`InfraError` classifies errors as `Transient` or
`Corruption`). No `OpIdConflict`, no cursor variants, no split variants.

---

## The Remove-Mutate-Restore Pattern for Splits

Both `split_replace` and `split_residual` face a Rust borrow checker
challenge: they need a mutable reference to the parent record (to mutate
it) and simultaneous mutable access to the shard map (to insert child
records). With a standard `HashMap`, calling `get_mut` to get `&mut
ShardRecord` borrows the entire map, preventing any other map operations.

The solution is the **remove-mutate-restore** pattern:

```rust
fn split_replace(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
    plan: SplitReplacePlan<'_>,
    op_id: OpId,
) -> Result<IdempotentOutcome<SplitReplaceResult>, SplitReplaceError> {
    let key = lease.shard_key();

    // Remove the parent from the map (remove-mutate-restore pattern).
    self.with_removed_parent(
        tenant,
        key,
        SplitReplaceError::ShardNotFound { shard: key },
        |coordinator, parent| {
            // Phase 1: Validate preconditions.
            let payload_hash = hash_split_replace_payload(&plan);
            if check_op_idempotency(parent, op_id, payload_hash)?.is_some() {
                let children = split_replace_replay_child_ids(parent, &plan, op_id);
                return Ok(IdempotentOutcome::Replayed(SplitReplaceResult { children }));
            }
            validate_lease(now, tenant, lease, parent)?;
            let sorted =
                split_replace_validate_preconditions(parent, &plan, &coordinator.slab)?;

            // Shard count limit guard (temporarily_removed=1 for parent).
            coordinator
                .check_shard_limits(tenant, sorted.len(), 1)
                .map_err(SplitReplaceError::SplitInvalid)?;

            // Phase 2: Build + insert children (may allocate into slab).
            let child_ids = split_replace_insert_children(
                coordinator,
                parent,
                &plan,
                &sorted,
                tenant,
                op_id,
            )?;

            // Phase 3: Apply parent mutation + index updates.
            if let Err(e) = split_replace_apply_parent(
                parent,
                child_ids.as_slice(),
                op_id,
                payload_hash,
                now,
                &mut coordinator.slab,
            ) {
                split_replace_rollback_inserted_children(
                    coordinator, tenant, parent.run, child_ids.as_slice(),
                );
                return Err(e);
            }

            for &child_id in &child_ids {
                coordinator.index_shard(tenant, parent.run, child_id);
            }

            Ok(IdempotentOutcome::Executed(SplitReplaceResult {
                children: child_ids,
            }))
        },
    )
}
```

The key insight is the `with_removed_parent` helper. The parent is removed
from the map at the top, which releases the borrow on `self.shards`. Inside
the closure, `&mut parent` and `&mut coordinator` can coexist because `parent` is a
local variable passed by the helper, not a borrow from the map.

The parent is restored at the bottom unconditionally -- both on success
and on error -- by the `with_removed_parent` helper. The only exception is
if `assert_invariants` panics inside `split_replace_apply_parent`. In that
case, the parent is intentionally *not* restored because the invariant
violation indicates irrecoverable corruption, and the process is crashing
anyway.

Why not just leave the parent out of the map if it fails? Because the
error path needs to return a clean error to the caller. If the parent
were missing from the map, subsequent operations would see `ShardNotFound`
for a shard that should exist. The restore-on-error ensures the map is
consistent regardless of the operation outcome.

The `temporarily_removed=1` parameter to `check_shard_limits` accounts
for the parent being out of the map during validation. Without it, the
limit check would undercount by one.

### Extracted Free Functions

The split helpers are extracted as free functions (not methods on
`InMemoryCoordinator`) for two reasons:

1. **Borrow splitting.** `split_replace` temporarily removes the parent
   from `self.shards` and passes `&mut parent` into a closure. Free
   functions that take `&ShardRecord` / `&mut ShardRecord` avoid borrowing
   `self`, which would conflict with the closure's `&mut self` captures
   for `shard_insert` and `index_shard`.

2. **Purity.** Validation and record construction are pure computations.
   Keeping them free of `self` makes this explicit and simplifies unit
   testing -- no coordinator setup required.

The three phases of split_replace (validate, build, apply) map to
separate functions:

- `split_replace_validate_preconditions` -- sorts children, validates
  coverage, checks spawn cap. Takes `slab: &ByteSlab` to borrow parent
  bounds directly from the pooled spec.
- `split_replace_insert_children` -- derives child IDs via BLAKE3,
  constructs `ShardRecord`s, inserts each child into the map with
  rollback on mid-build failure. Takes `&mut InMemoryCoordinator`
  for map insertion and slab allocation.
- `split_replace_apply_parent` -- transitions parent to `Split` status,
  records spawned children, pushes op-log entry, asserts invariants.
  Takes `slab: &mut ByteSlab` for spawned lineage allocation.

---

## `find_replayed_residual`: The Secondary Idempotency Defense

`split_residual` has a unique idempotency challenge. Unlike `split_replace`
(which makes the parent terminal, freezing its op-log), `split_residual`
keeps the parent `Active`. Subsequent checkpoints can evict the
`split_residual` op-log entry.

The solution is a two-tier replay detection:

```rust
fn split_residual_check_replay(
    parent: &ShardRecord,
    op_id: OpId,
    payload_hash: u64,
    slab: &ByteSlab,
) -> Result<Option<IdempotentOutcome<SplitResidualResult>>, SplitResidualError> {
    // Tier 1: Op-log check (primary).
    if check_op_idempotency(parent, op_id, payload_hash)?.is_some() {
        let replayed = find_replayed_residual(parent, op_id, slab).expect(
            "op-log hit implies residual exists in parent.spawned"
        );
        return Ok(Some(IdempotentOutcome::Replayed(SplitResidualResult {
            residual: replayed,
        })));
    }

    // Tier 2: Spawned probe (defense-in-depth).
    // If the op-log entry was evicted but the residual was already created,
    // detect via parent.spawned (permanent, never evicted).
    if let Some(existing) = find_replayed_residual(parent, op_id, slab) {
        return Ok(Some(IdempotentOutcome::Replayed(SplitResidualResult {
            residual: existing,
        })));
    }

    Ok(None)
}
```

Tier 1 is the standard op-log check. If the entry is present and the hash
matches, return `Replayed`. If the hash differs, return `OpIdConflict`.

Tier 2 kicks in when the op-log entry has been evicted (16+ intervening
operations). `find_replayed_residual` scans `parent.spawned` -- which is
permanent and never evicted -- for a residual derived from the given OpId:

```rust
fn find_replayed_residual(parent: &ShardRecord, op_id: OpId, slab: &ByteSlab) -> Option<ShardId> {
    assert!(
        parent.spawned.len() <= MAX_SPAWNED_PER_SHARD,
        "spawned count {} exceeds bound {}",
        parent.spawned.len(), MAX_SPAWNED_PER_SHARD,
    );
    for (idx, spawned) in parent.spawned.iter(slab).enumerate() {
        let candidate = derive_split_shard_id(
            parent.run,
            parent.shard,
            op_id,
            DerivedShardKind::Residual,
            idx as u32,
        );
        if spawned == candidate {
            return Some(candidate);
        }
    }
    None
}
```

The algorithm iterates `parent.spawned` via the slab-backed iterator. For
each `(index, spawned_id)` pair, it re-derives the BLAKE3-based ShardId for
that index and compares directly. The complexity is O(S * D) where S =
`spawned.len()` (bounded by 1024) and D = BLAKE3 hash cost (constant).
No heap allocation -- the slab iterator borrows directly from the arena.

The spawned check comes *after* the op-log check so that `OpIdConflict`
(same OpId, different payload) is not masked. The source code documents
a known limitation:

> The spawned-probe tier cannot verify payload hash after op-log
> eviction. If a client replays op_id=X with a *different* plan after
> the op-log entry is evicted, this path returns Replayed (matching the
> original residual) instead of OpIdConflict.

This is acceptable because: (1) eviction requires 16+ intervening ops,
(2) OpIds are CSPRNG-generated so accidental reuse is astronomically
unlikely, and (3) production backends with durable op-logs do not have
this window.

---

## Capacity Hint Computation

After every acquire and renew, the coordinator computes a `CapacityHint`:

```rust
fn count_available_for_run(
    &self,
    now: LogicalTime,
    tenant: TenantId,
    run: RunId,
) -> CapacityHint {
    let shard_ids = match self.run_shards.get(&(tenant, run)) {
        Some(ids) => ids,
        None => return CapacityHint::ZERO,
    };

    let mut available_count: u32 = 0;
    let mut earliest_deadline: Option<LogicalTime> = None;

    for &shard_id in shard_ids {
        let key = ShardKey::new(run, shard_id);
        let record = self.shard_get(&tenant, &key).unwrap_or_else(|| {
            panic!("run_shards index desynchronization")
        });

        if record.status != ShardStatus::Active {
            continue;
        }

        if record.is_leased_at(now) {
            let deadline = record.lease().expect("lease exists").deadline();
            earliest_deadline = Some(match earliest_deadline {
                Some(prev) => core::cmp::min(prev, deadline),
                None => deadline,
            });
        } else {
            available_count = available_count
                .checked_add(1)
                .expect("available_count overflow");
        }
    }

    CapacityHint { available_count, earliest_deadline }
}
```

The scan iterates all shards in the run. For each active shard:
- If it has a live lease, its deadline is compared against the running
  minimum (`earliest_deadline`).
- If it is unleased, `available_count` is incremented.

Terminal shards (Done, Split, Parked) are skipped entirely.

The result is order-independent: both sum and min are commutative, so
the `HashSet` iteration order does not affect the output.

A missing `run_shards` entry (no registered shards yet) returns
`CapacityHint::ZERO` rather than panicking. This handles the case where
`acquire_and_restore_into` is called before `register_shards` has populated
the index.

---

## Key Design Patterns

Eight patterns recur across every operation in the `InMemoryCoordinator`.
Understanding them lets you read any operation's implementation quickly.

### 1. Idempotency-First

Every idempotent operation checks `check_op_idempotency` before any
other validation. This ordering guarantee means:

- Replays succeed even after lease expiry.
- Replays succeed even after the shard becomes terminal.
- Replays succeed even after the lease is transferred to another worker.

If the worker sent a checkpoint, the network ate the response, the lease
expired, another worker acquired the shard, and the original worker
retries -- the retry still succeeds because the op-log entry is found
before the lease is validated.

### 2. Invariant Assertion After Every Mutation

`record.assert_invariants(&slab)` appears at the end of every mutation path.
Without exception. This catches bugs in the mutation logic before they
propagate to persistent storage. The assertions check all ten structural
invariants documented in
[Chapter 10](./10-validation-layer.md).

### 3. Lease Release on Terminal Transitions

Every operation that transitions a shard to a terminal state clears the
lease:

```rust
// In complete:
record.status = ShardStatus::Done;
record.lease = None;

// In park_shard:
record.status = ShardStatus::Parked;
record.park_reason = Some(reason);
record.lease = None;

// In split_replace_apply_parent:
parent.status = ShardStatus::Split;
parent.lease = None;
```

INV-3 (terminal shards must not hold a lease) makes this mandatory. If
any path forgets the `lease = None`, the subsequent `assert_invariants(&slab)`
panics. The invariant check is the enforcement mechanism; the pattern is
the convention that makes the enforcement predictable.

### 4. Op-Log Recording as the Last Mutation Step

The op-log push is always the last mutation before `assert_invariants(&slab)`:

```rust
record.cursor.update_from_ref(new_cursor, &mut self.slab)?;  // State mutation
record.op_log_push(OpLogEntry::new(..));                      // Op-log recording (last)
record.assert_invariants(&self.slab);                         // Invariant check
```

This ordering ensures that `op_log_push`'s internal duplicate check
(`assert!(!self.op_log.iter().any(|e| e.op_id() == entry.op_id()))`) runs
after all other mutations. If the idempotency check were somehow bypassed,
the push assertion catches the duplicate.

### 5. The `?` Operator and `From<CoordError>` Conversion

Every validation call uses `?` to convert `CoordError` into the
operation-specific error type:

```rust
// In checkpoint:
check_op_idempotency(record, op_id, payload_hash)?;       // CoordError -> CheckpointError
validate_lease(now, tenant, lease, record)?;                // CoordError -> CheckpointError
validate_cursor_update_pooled(&new_cursor, ...slab...)?;   // CoordError -> CheckpointError
```

This is the shared building block architecture in action. The validation
functions are generic (return `CoordError`), and the `?` operator
automatically routes through the `From<CoordError> for CheckpointError`
impl.

### 6. Payload Hashing Before Idempotency Check

Every idempotent operation hashes its payload before calling
`check_op_idempotency`:

```rust
let payload_hash = hash_checkpoint_payload(&new_cursor);
if check_op_idempotency(record, op_id, payload_hash)?.is_some() {
    return Ok(IdempotentOutcome::Replayed(()));
}
```

The hash function is deterministic and specific to each operation type.
`hash_checkpoint_payload` hashes the cursor; `hash_complete_payload`
hashes the final cursor; `hash_park_payload` hashes the park reason;
`hash_split_replace_payload` hashes the split plan. Different payloads
produce different hashes, so reusing an OpId with a different payload
triggers `OpIdConflict`.

### 7. Consistent Status Check Patterns

Acquire uses `record.status != ShardStatus::Active` (positive check for
the one allowed state). Lease-gated operations use
`record.status.is_terminal()` inside `validate_lease` (negative check
for all disallowed states). Terminal transitions use
`record.assert_transition_legal(ShardStatus::Done)` (state machine
validation). The three patterns target different audiences:

- Acquire needs to specifically distinguish Active from non-Active
  because it has a unique `ShardTerminal` error.
- Lease-gated operations only care whether the shard is alive or dead.
- Terminal transitions need to verify the state machine edge exists.

### 8. Slab Lifecycle and Drop

The coordinator owns the `ByteSlab` that backs all `PooledShardSpec` and
`PooledCursor` fields. On drop, the coordinator must release all slab-backed
fields to avoid leak assertions.

Individual records expose `deallocate_fields(&mut self, slab: &mut ByteSlab)`
(defined at `record.rs:803-807`) which releases spec, cursor, and
spawned-lineage storage:

```rust
pub fn deallocate_fields(&mut self, slab: &mut ByteSlab) {
    self.spec.release_fields(slab);
    self.cursor.release_fields(slab);
    self.spawned.release_fields(slab);
}
```

This method is used by coordinator rollback/drop paths and by simulation
wrappers via `SimIntrospection::release_record_fields`. If a record is removed
while the slab stays live, the coordinator must call `deallocate_fields` to
prevent pooled slots from consuming slab capacity indefinitely.

The coordinator's `Drop` impl uses `deallocate_fields` in debug builds:
impl Drop for InMemoryCoordinator {
    fn drop(&mut self) {
        #[cfg(debug_assertions)]
        {
            for (_tenant, tenant_map) in self.shards.drain() {
                for (_key, mut record) in tenant_map {
                    record.deallocate_fields(&mut self.slab);
                }
            }
            // ByteSlab::Drop will assert live_count == 0
        }
        #[cfg(not(debug_assertions))]
        self.slab.clear();
    }
}
```

The dual strategy is intentional:

- **Debug builds** iterate all records and call `deallocate_fields` on each,
  which releases spec, cursor, and spawned lineage slab slots individually.
  `ByteSlab::Drop`'s `live_count == 0` assertion then verifies that no
  handles were leaked.
- **Release builds** call `slab.clear()` for O(1) bulk reset. This is
  safe because the slab outlives all records, and no handles escape the
  coordinator.

### Why These Patterns Matter

These eight patterns are not accidental conventions. They are load-bearing
structural decisions that make the codebase auditable:

- **Idempotency-first** means you can verify the idempotency guarantee by
  reading the first three lines of any operation.
- **Invariant assertion after every mutation** means bugs are caught at the
  mutation site, not propagated to some downstream consumer that sees
  corrupt state days later.
- **Lease release on terminal transitions** with invariant enforcement means
  you can never accidentally leave a lease on a dead shard.
- **Op-log as last mutation** means the op-log push's internal duplicate
  check provides a second line of defense against idempotency bypass.

When reviewing a new operation implementation, check for these patterns.
If any is missing, that is a bug.

---

## The `unpark_shard` Operation

`unpark_shard` lives on the `RunManagement` trait, not
`CoordinationBackend`, because it is an administrative operation. It
transitions a Parked shard back to Active:

```rust
fn unpark_shard(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    key: ShardKey,
    op_id: OpId,
) -> Result<IdempotentOutcome<()>, UnparkError> {
    // ... lookup, tenant check, run terminal check ...

    // Idempotency via shard op-log.
    if check_op_idempotency(record, op_id, payload_hash)?.is_some() {
        return Ok(IdempotentOutcome::Replayed(()));
    }

    if record.status != ShardStatus::Parked {
        return Err(UnparkError::NotParked { status: record.status });
    }

    {
        // Re-borrow mutably for the actual mutation.
        let record = self
            .shard_get_mut(&tenant, &key)
            .expect("shard record must exist: verified above");

        // Bump fence first -- invalidate zombie workers from prior lease.
        record.advance_fence();

        record.park_reason = None;
        record.status = ShardStatus::Active;
        record.lease = None;

        record.op_log_push(OpLogEntry::new(
            op_id, OpKind::Unpark, OpResult::Completed, payload_hash, now,
        ));
    }
    self.shard_get(&tenant, &key)
        .expect("unparked shard must exist")
        .assert_invariants(&self.slab);
    Ok(IdempotentOutcome::Executed(()))
}
```

Two notable details:

1. **Fence bump before status change.** The fence epoch is incremented
   before setting status back to Active. This invalidates any zombie
   workers that might still hold a reference to the shard from a prior
   lease (before the shard was parked). Without the fence bump, a zombie
   worker could present a lease from epoch N, and if the unparked shard
   still had epoch N, the zombie's checkpoint would succeed.

2. **Run terminal check.** Before unparking, the implementation checks
   whether the parent run is terminal. Unparking a shard in a Cancelled,
   Done, or Failed run would create an orphaned Active shard that no
   run-level operation will ever collect. The check prevents this waste.

---

## The Secondary Index: `run_shards`

The `run_shards` map (`AHashMap<(TenantId, RunId), HashSet<ShardId>>`)
is a secondary index that enables run-level queries without scanning the
entire tenant shard map. Without it, `list_shards` and `get_run_progress`
would need to iterate all shards for a tenant and filter by run -- O(S_t)
where S_t is the total shard count for the tenant.

The index is maintained at three points:

1. **`register_shards`** -- inserts root shard IDs when the manifest is
   registered.
2. **`split_replace`** -- inserts child shard IDs when a parent is split.
3. **`split_residual`** -- inserts the residual shard ID when a parent
   sheds a portion of its key range.

The `index_shard` method is idempotent (`HashSet::insert` is a no-op for
duplicates), so calling it on both register and split paths is safe.

A debug-only consistency check validates the index after every mutation:

```rust
#[cfg(debug_assertions)]
fn debug_assert_run_shards_consistent(&self, tenant: TenantId, run: RunId) {
    if let Some(shard_ids) = self.run_shards.get(&(tenant, run)) {
        for &shard_id in shard_ids {
            let key = ShardKey::new(run, shard_id);
            debug_assert!(
                self.shard_get(&tenant, &key).is_some(),
                "run_shards index contains {:?} but primary map has no record",
                shard_id,
            );
        }
    }
}
```

This catches a specific class of bug: a shard ID added to the index but
not inserted into the primary map. In release builds, the check is compiled
out entirely (`#[cfg(debug_assertions)]`) to avoid the O(S) scan on every
mutation.

---

## Connecting to Production Backends

The `InMemoryCoordinator` is the golden reference, but it is not the only
backend. Production deployments use PostgreSQL, DynamoDB, or other durable
stores. The relationship between the reference and production backends is:

1. **Same observable behavior.** Given the same sequence of operations
   (same `now`, same `tenant`, same `lease`, same `op_id`), both must
   produce the same results, the same error variants, and the same state
   transitions.

2. **Same validation functions.** Production backends call the same
   `validate_lease`, `check_op_idempotency`, and `validate_cursor_update_pooled`
   functions. These are pure and backend-agnostic.

3. **Different storage.** The in-memory backend uses hash maps; PostgreSQL
   uses rows and transactions; DynamoDB uses items and conditional writes.
   The storage mechanism differs, but the logical state machine is
   identical.

4. **Same invariant checks.** Production backends can call
   `ShardRecord::validate_invariants()` (the non-panicking variant) after
   deserializing a record from storage, catching corruption introduced by
   storage-layer bugs.

The conformance test suite (described in
[Chapter 12: Proving Correctness](./12-proving-correctness.md)) feeds
identical operation sequences to every backend and compares outputs. The
deterministic simulation (described in the simulation harness docs) goes
further, injecting failures and verifying that invariants hold across
crash-recovery boundaries.

---

## Summary: Reading the InMemoryCoordinator

When you open `in_memory.rs` for the first time, the file is large. Here
is a reading strategy:

1. **Start with the struct definition.** Understand the
   two-level map, the secondary index, and the configuration fields.

2. **Read `acquire_and_restore_into`** (the `CoordinationBackend` impl). This
   is the entry point for a worker's lifecycle and demonstrates the
   non-idempotent operation pattern.

3. **Read `checkpoint`**. This demonstrates the idempotent operation
   pattern with the three-step validation chain. Once you understand
   checkpoint, every other idempotent operation (complete, park, split)
   follows the same structure.

4. **Read `split_replace`**. This demonstrates the remove-mutate-restore
   pattern and the extracted free function design. `split_residual` is
   the same pattern with the addition of `find_replayed_residual`.

5. **Skip the run-level operations** (`RunManagement` impl) on first
   reading. They follow the same idempotency-first pattern but operate
   on `RunRecord` instead of `ShardRecord`.

6. **Read the `SimIntrospection` impl** last. It is a read-only
   observation interface for the deterministic simulation harness,
   providing iterator access to all shards without mutation.

The key insight is that every operation follows one of two templates:

**Non-idempotent (acquire, renew):**
lookup -> validate -> mutate -> assert_invariants

**Idempotent (checkpoint, complete, park, split, register_shards):**
lookup -> idempotency_check -> validate -> mutate -> op_log_push -> assert_invariants

Once you recognize the template, you can read any operation by focusing
on what is unique to it (the specific mutation logic) rather than
re-analyzing the validation boilerplate.

---

**Source files referenced in this chapter:**

- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/in_memory.rs`
- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/validation.rs`
- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/record.rs`
- `/Users/ahrav/Projects/Gossip-rs/crates/gossip-coordination/src/error.rs`
