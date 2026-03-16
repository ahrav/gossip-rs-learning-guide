# The Faithful Mirror -- In-Memory Persistence Backends for Tests and Simulation

*A CI pipeline runs 4,000 integration tests against the scan engine. Test number 2,871 writes three findings through `FindingsSink`, calls `handle.wait()`, then asserts the durable count is three. The test passes 499 times out of 500. On the 500th run, the assertion fires: durable count is two. The engineer reruns the suite. It passes. She reruns it again. It passes. Weeks later, the same test fails in a different developer's branch. The findings were written through a mock that applied them synchronously inside `upsert_batch` but returned the handle before the internal `HashMap` insert completed in a background task. The auto-commit raced with the assertion. The window was roughly 50 microseconds wide -- enough to fail once every few hundred runs, never reproducibly, never on the same machine twice. The team spends 14 hours bisecting before realizing the mock itself was the bug.*

---

Test doubles for persistence must not lie. A mock that returns `Ok(())` without enforcing referential integrity will let tests pass that would fail against PostgreSQL. A mock that applies writes synchronously inside `batch_upsert` eliminates the two-phase commit protocol that production backends implement. A mock that cannot inject failures leaves error-handling paths permanently untested.

The `gossip-persistence-inmemory` crate solves these problems with two full reference implementations -- `InMemoryDoneLedger` and `InMemoryFindingsSink` -- that enforce the same invariants a production backend would. Done-ledger writes use monotonic lattice merge. Findings writes enforce referential integrity between all three layers. Durability acknowledgement remains explicit through `CommitHandle::wait()`. And the entire system supports deterministic fault injection and delayed completion for simulation testing.

These are not mocks. They are the persistence contract, implemented in memory.

The design achieves three properties simultaneously. First, correctness: the in-memory backends enforce the same lattice merge, referential integrity, and commit protocol that production backends must implement. Second, controllability: test code can inject failures at submission time, at commit time, or delay writes indefinitely and release them in any order. Third, observability: snapshot methods expose durable state for assertions without breaking encapsulation of the internal data structures.

## The Generic Store Infrastructure

Both backends share a single coordination engine: `InMemoryStoreCore<B>`, parameterized over a `StoreBackend` trait that injects domain-specific apply logic. The shared infrastructure handles queuing, condvar signaling, fault injection, and commit-handle lifecycle management. Without this factoring, the done-ledger and findings backends would duplicate hundreds of lines of synchronization code.

### The StoreBackend Trait

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) trait StoreBackend: Send + 'static {
    /// The buffered write payload (e.g. a vec of records).
    type Payload: Send;

    /// The receipt returned to callers after a successful commit.
    type Receipt: Send;

    /// The mutable durable state that payloads are applied to.
    type Durable: Send + Default;

    /// Discriminator used in error variants to identify which backend
    /// produced the error without requiring downcasting.
    const STORE_KIND: InMemoryStoreKind;

    /// Apply `payload` to `durable`, producing a receipt on success.
    fn apply(
        durable: &mut Self::Durable,
        fail_commit_remaining: &mut usize,
        payload: &Self::Payload,
    ) -> Result<Self::Receipt, InMemoryPersistenceError>;
}
```

Five items define a backend: the payload buffered by a pending write, the receipt returned after a successful commit, the durable state that payloads mutate, a store-kind discriminator for error reporting, and the `apply` function that performs the domain-specific mutation. The `apply` function receives a mutable reference to `fail_commit_remaining` so it can check and decrement the fault-injection counter before touching durable state. This keeps injected failures deterministic: if the counter is non-zero, the apply function returns `InjectedCommitFailure` without mutating anything.

The trait is `Send + 'static` because the store core is wrapped in `Arc` and shared across threads. The `Durable` type requires `Default` so that `StoreState` can be default-constructed. These bounds propagate to the public wrappers: both `InMemoryDoneLedger` and `InMemoryFindingsSink` are `Send + Sync` by construction, making them safe to share between test threads and the system under test.

### StoreState: The Shared Interior

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) struct StoreState<B: StoreBackend> {
    pub(crate) durable: B::Durable,
    ops: HashMap<PendingWriteId, PendingOp<B::Payload, B::Receipt>>,
    order: VecDeque<PendingWriteId>,
    next_op_id: u64,
    pub(crate) auto_complete: bool,
    delay_next: usize,
    fail_submit_remaining: usize,
    fail_commit_remaining: usize,
}
```

Operations live in two parallel structures. The `ops` map stores every operation by id -- both pending and recently finished -- so that `StoreHandle::wait` can look up results regardless of completion mode. The `order` queue tracks submission order for delayed operations only; auto-completed operations never enter this queue. Three independent fault-injection counters control failure behavior at different points in the operation lifecycle.

The `next_op_id` counter starts at 1 so that id 0 is never issued, providing a useful sentinel value in tests. The `auto_complete` flag defaults to `true`, meaning most tests get synchronous behavior without configuration. The `delay_next` counter overrides `auto_complete` for a specific number of writes, enabling mixed-mode operation. The `fail_submit_remaining` counter rejects writes before any state change. The `fail_commit_remaining` counter fails writes at apply time, after a handle has been issued. All counters use `saturating_add` when incremented and simple decrement when consumed, making their behavior predictable and idempotent across multiple configuration calls.

### InMemoryStoreCore: Mutex + Condvar

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) struct InMemoryStoreCore<B: StoreBackend> {
    pub(crate) state: Mutex<StoreState<B>>,
    pub(crate) cv: Condvar,
}
```

The core wraps `StoreState` with a single mutex and a single condvar. There is exactly one mutex per `InMemoryStoreCore` instance. No code path acquires a second lock while holding this one, eliminating deadlock risk. The condvar is signaled with `notify_all` after every state transition to `Finished`, even for auto-completed operations. Multiple handles may be waiting on different operations simultaneously, so `notify_all` is necessary. Waiters re-check their specific operation id after each wake, tolerating spurious wakes.

### The Submit Path

The `submit` method is the primary entry point for all write operations. It follows a strict four-phase sequence:

```rust
// From crates/gossip-persistence-inmemory/src/store.rs (simplified)

pub(crate) fn submit(
    self: &Arc<Self>,
    payload: B::Payload,
) -> Result<StoreHandle<B>, InMemoryPersistenceError> {
    let mut guard = self.lock_state()?;

    // Phase 1: Submission fault injection — reject before any state change.
    if guard.fail_submit_remaining > 0 {
        guard.fail_submit_remaining -= 1;
        return Err(InMemoryPersistenceError::InjectedSubmissionFailure {
            store: B::STORE_KIND,
        });
    }

    // Phase 2: Allocate a unique, monotonically increasing op id.
    let op_id = PendingWriteId::from_raw(guard.next_op_id);
    guard.next_op_id = guard.next_op_id.checked_add(1).ok_or(
        InMemoryPersistenceError::InjectedSubmissionFailure {
            store: B::STORE_KIND,
        },
    )?;

    // Phase 3: delay_next takes priority over auto_complete.
    let should_delay = if guard.delay_next > 0 {
        guard.delay_next -= 1;
        true
    } else {
        !guard.auto_complete
    };

    // Phase 4: Enqueue or apply.
    if should_delay {
        guard.order.push_back(op_id);
        guard.ops.insert(op_id, op);
    } else {
        let state = &mut *guard;
        let result = B::apply(
            &mut state.durable,
            &mut state.fail_commit_remaining,
            &op.payload,
        );
        op.state = PendingState::Finished(Some(result));
        state.ops.insert(op_id, op);
        self.cv.notify_all();
    }

    Ok(StoreHandle { inner: Arc::clone(self), op_id })
}
```

Phase 1 checks the submission-failure counter. If non-zero, the request is rejected before any state mutation -- no handle is created, no op id is allocated. Phase 2 allocates a monotonic operation id using `checked_add` to prevent wraparound. Phase 3 determines delay mode: the `delay_next` counter takes priority over the global `auto_complete` flag, allowing per-submission delay overrides without toggling global state. Phase 4 either enqueues the operation as pending or applies it immediately.

For auto-completed writes, errors from `B::apply()` do not cause `submit` to fail. The error is stored inside the operation's `Finished` state and surfaced when the handle's `wait` method is called. This preserves the two-phase contract: submission succeeds, durability may fail.

## InMemoryDoneLedger

The done-ledger backend tracks which `(tenant, policy, ovid)` triples have been scanned, and at what status. Its durable state is a `HashMap<DoneLedgerKey, DoneLedgerRecord>`.

### Construction

```rust
// From crates/gossip-persistence-inmemory/src/done_ledger.rs

#[derive(Clone, Default)]
pub struct InMemoryDoneLedger {
    core: Arc<InMemoryStoreCore<DoneLedgerBackend>>,
}

impl InMemoryDoneLedger {
    /// Create an empty done-ledger with auto-complete enabled.
    pub fn new() -> Self {
        Self::default()
    }

    /// Create an empty done-ledger with an explicit auto-complete mode.
    pub fn with_auto_complete(auto_complete: bool) -> Self {
        Self {
            core: Arc::new(InMemoryStoreCore::with_auto_complete(auto_complete)),
        }
    }
}
```

Cloning is cheap -- an `Arc` bump -- and produces a handle to the *same* shared state. This is intentional: test harnesses inject faults on one handle while the system under test uses another. The `Default` implementation creates an empty ledger with auto-complete enabled, which is the common case for tests that do not need to exercise delayed-completion scenarios.

### DoneLedger Trait Implementation

```rust
// From crates/gossip-persistence-inmemory/src/done_ledger.rs

impl DoneLedger for InMemoryDoneLedger {
    type Error = InMemoryPersistenceError;
    type CommitHandle = InMemoryDoneLedgerHandle;

    fn batch_get(
        &self,
        tenant_id: TenantId,
        policy_hash: PolicyHash,
        ovid_hashes: &[OvidHash],
    ) -> Result<Vec<Option<DoneLedgerRecord>>, Self::Error> {
        let guard = self.core.lock_state()?;
        Ok(ovid_hashes
            .iter()
            .map(|ovid_hash| {
                let key = DoneLedgerKey::new(tenant_id, policy_hash, *ovid_hash);
                guard.durable.rows.get(&key).cloned()
            })
            .collect())
    }

    fn batch_upsert(
        &self,
        records: &[DoneLedgerRecord],
    ) -> Result<Self::CommitHandle, Self::Error> {
        let payload = DoneLedgerPayload {
            records: records.to_vec(),
        };
        let handle = self.core.submit(payload)?;
        Ok(InMemoryDoneLedgerHandle { handle })
    }
}
```

`batch_get` returns positionally aligned results: `result[i]` corresponds to `ovid_hashes[i]`, with `None` for absent keys. `batch_upsert` clones the records into a payload and delegates to the generic `submit` path.

### Lattice Merge Semantics

The critical behavior lives in the `StoreBackend::apply` implementation for the done-ledger. Rather than reimplementing merge logic, the in-memory backend delegates to `DoneLedgerRecord::merge()` on the contracts crate -- the same method that defines the canonical merge semantics:

```rust
// From crates/gossip-persistence-inmemory/src/done_ledger.rs (inside StoreBackend::apply)

// Validate each record before mutation.
for record in &payload.records {
    record.validate()?;

    let prov = record.provenance();
    if prov.started_at().as_raw() > prov.finished_at().as_raw() {
        return Err(PersistenceInputError::ProvenanceOrdering {
            started_at: prov.started_at().as_raw(),
            finished_at: prov.finished_at().as_raw(),
        }.into());
    }
}

// Deduplicate within the batch using DoneLedgerRecord::merge().
let mut by_key: HashMap<DoneLedgerKey, DoneLedgerRecord> =
    HashMap::with_capacity(payload.records.len());
for record in &payload.records {
    let key = record.key();
    match by_key.get(&key) {
        Some(existing) => {
            let merged = existing.merge(record)?;
            by_key.insert(key, merged);
        }
        None => {
            by_key.insert(key, record.clone());
        }
    }
}

// Apply deduplicated batch to durable state.
for (key, record) in &by_key {
    match durable.rows.get(key) {
        Some(existing) => {
            let merged = existing.merge(record)?;
            durable.rows.insert(*key, merged);
        }
        None => {
            durable.rows.insert(*key, record.clone());
        }
    }
}
```

The apply function performs three phases. First, it validates every record using `DoneLedgerRecord::validate()` and checks provenance temporal ordering -- rejecting records where `started_at` exceeds `finished_at`. Second, it deduplicates within the batch by merging records that share the same key via `DoneLedgerRecord::merge()`. Third, it applies the deduplicated batch to durable state, again using `DoneLedgerRecord::merge()` for any keys that already exist.

The `DoneLedgerRecord::merge()` method on the contracts crate implements the full per-field merge: status uses the lattice join (higher rank wins), `bytes_scanned` takes the max, `findings_count` is status-aware (forced to zero for `ScannedClean`, clamped to at least one for `ScannedWithFindings`), provenance is selected from the winner record using a multi-field comparison key, and error codes are cleared on scanned status or taken from the provenance winner with fallback to the loser.

This delegation pattern is deliberate. The in-memory backend does not reimplement merge logic -- it calls the same `merge()` method that defines the canonical semantics. If a production backend disagrees with the contracts crate on the result of any merge, the conformance harness (covered in the next chapter) will identify which.

### Snapshot and Query Methods

The done-ledger exposes `get_record` for single-key lookups and `snapshot` for full-state dumps:

```rust
// From crates/gossip-persistence-inmemory/src/done_ledger.rs

pub fn get_record(
    &self,
    key: DoneLedgerKey,
) -> Result<Option<DoneLedgerRecord>, InMemoryPersistenceError> {
    let guard = self.core.lock_state()?;
    Ok(guard.durable.rows.get(&key).cloned())
}

pub fn snapshot(&self) -> Result<Vec<DoneLedgerRecord>, InMemoryPersistenceError> {
    let guard = self.core.lock_state()?;
    let mut rows: Vec<_> = guard.durable.rows.values().cloned().collect();
    rows.sort_by(|lhs, rhs| {
        lhs.key().tenant_id().as_bytes().cmp(rhs.key().tenant_id().as_bytes())
            .then_with(|| lhs.key().policy_hash().as_bytes()
                .cmp(rhs.key().policy_hash().as_bytes()))
            .then_with(|| lhs.key().ovid_hash().as_bytes()
                .cmp(rhs.key().ovid_hash().as_bytes()))
    });
    Ok(rows)
}
```

Snapshots are sorted by `(tenant_id, policy_hash, ovid_hash)` in byte-lexicographic order so that comparisons are deterministic regardless of `HashMap` iteration order. This matters for tests that assert on the full durable state: without sorting, two semantically identical states could produce different assertion output depending on hash randomization.

## InMemoryFindingsSink

The findings backend stores the three-layer hierarchy: findings, occurrences, and observations. Its durable state consists of three `HashMap`s keyed by `(TenantId, EntityId)` tuples.

### Construction and Trait Implementation

```rust
// From crates/gossip-persistence-inmemory/src/findings.rs

#[derive(Clone, Default)]
pub struct InMemoryFindingsSink {
    core: Arc<InMemoryStoreCore<FindingsBackend>>,
}

impl FindingsSink for InMemoryFindingsSink {
    type Error = InMemoryPersistenceError;
    type CommitHandle = InMemoryFindingsHandle;

    fn upsert_batch(
        &self,
        batch: FindingsUpsertBatch<'_>,
    ) -> Result<Self::CommitHandle, Self::Error> {
        batch.validate_observation_identity()?;
        validate_batch_tenant_consistency(batch)?;

        let payload = FindingsPayload {
            findings: batch.findings().to_vec(),
            occurrences: batch.occurrences().to_vec(),
            observations: batch.observations().to_vec(),
        };
        let handle = self.core.submit(payload)?;
        Ok(InMemoryFindingsHandle { handle })
    }
}
```

Validation is split into two phases. Phase 1, before the lock, checks batch-local invariants: observation identity consistency and single-tenant enforcement. Phase 2, inside `apply_findings_payload` under the lock, performs referential integrity checks that require access to the union of durable and same-batch rows.

### Validate-Then-Mutate: Referential Integrity

The `apply_findings_payload` function processes the three layers in parent-to-child order so that child referential checks can look up parents in both the already-durable state and the same-batch maps:

```rust
// From crates/gossip-persistence-inmemory/src/findings.rs (abbreviated)

fn apply_findings_payload(
    durable: &mut FindingsDurable,
    fail_commit_remaining: &mut usize,
    payload: &FindingsPayload,
) -> Result<FindingsCommitReceipt, InMemoryPersistenceError> {
    if *fail_commit_remaining > 0 {
        *fail_commit_remaining -= 1;
        return Err(InMemoryPersistenceError::InjectedCommitFailure {
            store: InMemoryStoreKind::Findings,
        });
    }

    // Layer 1: Findings (immutable insert-or-check)
    let mut batch_findings: HashMap<FindingKey, FindingRecord> =
        HashMap::with_capacity(payload.findings.len());
    for finding in &payload.findings {
        let key = (finding.tenant_id(), finding.finding_id());
        if let Some(existing) = batch_findings.get(&key) {
            if existing != finding {
                return Err(InMemoryPersistenceError::FindingConflict {
                    tenant_id: finding.tenant_id(),
                    finding_id: finding.finding_id(),
                });
            }
            continue;
        }
        if let Some(existing) = durable.findings.get(&key)
            && existing != finding
        {
            return Err(InMemoryPersistenceError::FindingConflict {
                tenant_id: finding.tenant_id(),
                finding_id: finding.finding_id(),
            });
        }
        batch_findings.insert(key, finding.clone());
    }

    // Layer 2: Occurrences (immutable, with parent reference check)
    // ... each occurrence must reference a finding in batch_findings
    //     or durable.findings ...

    // Layer 3: Observations (upsert with identity check)
    // ... identity fields must match; provenance may change ...

    // Validation passed — commit new rows into durable state.
    for (key, record) in batch_findings {
        durable.findings.entry(key).or_insert(record);
    }
    // ...
    Ok(FindingsCommitReceipt::new(
        findings_count, occurrences_count, observations_count,
    ))
}
```

The critical design is the validate-then-mutate approach. Temporary `batch_*` maps accumulate validated rows. Every invariant is checked as read-only lookups against the union of `batch_*` (same-batch rows) and `durable.*` (already-durable rows). Only after the entire payload passes validation are new rows inserted into durable state. A validation error at any point aborts without mutating anything.

Findings and occurrences are immutable: duplicate keys with identical content are silently deduplicated, but differing content produces a `FindingConflict` or `OccurrenceConflict` error. Observations use upsert semantics: identity fields must match, but provenance (`seen_at`, location) may be updated through a merge that prefers the freshest record.

### The FindingsConformanceProbe Implementation

The in-memory findings sink also implements `FindingsConformanceProbe`, providing the read-side surface needed by the conformance harness:

```rust
// From crates/gossip-persistence-inmemory/src/findings.rs

impl FindingsConformanceProbe for InMemoryFindingsSink {
    type Error = InMemoryPersistenceError;

    fn durable_counts(&self) -> Result<DurableFindingsCounts, Self::Error> {
        let guard = self.core.lock_state()?;
        Ok(DurableFindingsCounts::new(
            guard.durable.findings.len() as u64,
            guard.durable.occurrences.len() as u64,
            guard.durable.observations.len() as u64,
        ))
    }
}
```

This is a direct count of the three internal `HashMap` sizes, cast to `u64`. Since the counts are taken under the same lock that protects mutations, they represent a consistent snapshot of durable state at a single point in time. The conformance harness uses these counts to verify that replaying an identical batch produces zero additional rows.

## Fault Injection

Three independent counters control failure behavior, checked in strict order during each submission:

```
submit()
  |
  +-- fail_submit_remaining > 0 --> InjectedSubmissionFailure (no state change)
  |
  +-- delay_next > 0 or !auto_complete
  |     --> op enqueued as Pending, wait() blocks on condvar
  |
  +-- auto_complete (default)
        --> B::apply() runs immediately
            +-- fail_commit_remaining > 0 --> InjectedCommitFailure (no mutation)
            +-- otherwise --> durable state mutated, Finished
```

### Submission Failures

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) fn fail_next_submissions(
    &self,
    count: usize,
) -> Result<(), InMemoryPersistenceError> {
    let mut guard = self.lock_state()?;
    guard.fail_submit_remaining = guard.fail_submit_remaining.saturating_add(count);
    Ok(())
}
```

Submission failures are checked before any state mutation. No handle is created, no op id is allocated, and the durable state is untouched. This simulates backend unavailability at the request-acceptance layer. The caller sees an immediate error from `batch_upsert` or `upsert_batch`.

### Commit Failures

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) fn fail_next_commits(
    &self,
    count: usize,
) -> Result<(), InMemoryPersistenceError> {
    let mut guard = self.lock_state()?;
    guard.fail_commit_remaining = guard.fail_commit_remaining.saturating_add(count);
    Ok(())
}
```

Commit failures occur at apply time. The caller receives a valid handle, but `wait()` returns the injected error. This simulates fsync or replication failures in real backends. The durable state is untouched because each backend's `apply` function checks the counter before performing any mutation.

### Delayed Writes

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) fn delay_next_writes(
    &self,
    count: usize,
) -> Result<(), InMemoryPersistenceError> {
    let mut guard = self.lock_state()?;
    guard.delay_next = guard.delay_next.saturating_add(count);
    Ok(())
}
```

The `delay_next` counter takes priority over `auto_complete`. Even if auto-complete is enabled, the next `count` writes are enqueued as pending. This allows mixed-mode operation within a single test without toggling the global auto-complete flag back and forth. Calls are additive: `delay_next_writes(2)` followed by `delay_next_writes(3)` delays five writes total (saturating addition).

All three counters use `saturating_add` when incremented, preventing overflow when tests configure multiple rounds of failures.

## PendingWriteId and Completion Control

When writes are delayed, the test controls exactly when each operation becomes durable. The `PendingWriteId` is an opaque handle referencing a specific pending write:

```rust
// From crates/gossip-persistence-inmemory/src/error.rs

#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct PendingWriteId(u64);
```

The `CompletionOrder` enum controls which end of the queue to drain:

```rust
// From crates/gossip-persistence-inmemory/src/error.rs

pub enum CompletionOrder {
    /// Release the oldest pending operation first (FIFO drain).
    OldestFirst,
    /// Release the newest pending operation first (LIFO / reverse drain).
    NewestFirst,
}
```

`OldestFirst` is the natural drain order. `NewestFirst` lets tests simulate out-of-order durability where a later write becomes durable before an earlier one -- a scenario that occurs in production when multiple fsync operations complete in non-deterministic order.

### Release Methods

Three release methods provide different levels of control:

```rust
// From crates/gossip-persistence-inmemory/src/done_ledger.rs (same API on InMemoryFindingsSink)

/// Release one delayed operation in the requested order.
pub fn release_next(
    &self,
    order: CompletionOrder,
) -> Result<Option<PendingWriteId>, InMemoryPersistenceError> {
    self.core.release_next(order)
}

/// Release a specific delayed operation by id.
pub fn release_specific(
    &self,
    op_id: PendingWriteId,
) -> Result<bool, InMemoryPersistenceError> {
    self.core.release_specific(op_id)
}

/// Release every currently delayed operation.
pub fn release_all(
    &self,
    order: CompletionOrder,
) -> Result<usize, InMemoryPersistenceError> {
    self.core.release_all(order)
}
```

`release_next` pops from the front (`OldestFirst`) or back (`NewestFirst`) of the order queue until it finds a still-pending operation, then applies its payload. Already-finished entries are silently skipped -- this can happen when `release_specific` completed an operation out of queue order.

`release_specific` targets a single operation by id and removes it from the order queue via `retain`, preventing stale IDs from accumulating.

`release_all` snapshots the set of pending operations before mutation so that concurrent submitters cannot extend the set being drained. Only operations that are pending at snapshot time are released. The order parameter controls the apply sequence: `OldestFirst` for FIFO drain, `NewestFirst` for out-of-order durability testing.

### The PendingState Lifecycle

Each operation transitions through exactly two states:

```rust
// From crates/gossip-persistence-inmemory/src/pending.rs

pub(crate) enum PendingState<R> {
    Pending,
    Finished(Option<Result<R, InMemoryPersistenceError>>),
}
```

`Pending` to `Finished(Some(...))` is the only valid transition. The `Option` wrapper inside `Finished` exists so that `CommitHandle::wait` can `take()` the result exactly once. A second wait on the same operation sees `None` and returns `UnknownOperation`. The `PendingOp` struct pairs a payload with its lifecycle state:

```rust
// From crates/gossip-persistence-inmemory/src/pending.rs

pub(crate) struct PendingOp<P, R> {
    pub(crate) payload: P,
    pub(crate) state: PendingState<R>,
}
```

This design ensures that the payload is available for apply when the operation transitions from `Pending` to `Finished`, and that the result is available for extraction when the handle's `wait` method is called.

### The Condvar Wait Loop

Commit handles block on the shared condvar until their specific operation transitions to `Finished`:

```rust
// From crates/gossip-persistence-inmemory/src/store.rs

pub(crate) fn wait(self) -> Result<B::Receipt, InMemoryPersistenceError> {
    let mut guard = self.inner.state.lock().map_err(|_| {
        InMemoryPersistenceError::Poisoned { store: B::STORE_KIND }
    })?;

    loop {
        let finished = match guard.ops.get_mut(&self.op_id) {
            Some(op) => match &mut op.state {
                PendingState::Pending => None,
                PendingState::Finished(result) => Some(result.take().ok_or(
                    InMemoryPersistenceError::UnknownOperation {
                        store: B::STORE_KIND,
                        op_id: self.op_id,
                    },
                )),
            },
            None => {
                return Err(InMemoryPersistenceError::UnknownOperation {
                    store: B::STORE_KIND,
                    op_id: self.op_id,
                });
            }
        };

        if let Some(result) = finished {
            guard.ops.remove(&self.op_id);
            return result?;
        }

        guard = self.inner.cv.wait(guard).map_err(|_| {
            InMemoryPersistenceError::Poisoned { store: B::STORE_KIND }
        })?;
    }
}
```

The `wait` method takes `self` by value -- a handle can only be consumed once. The loop re-acquires the mutex after each condvar wake and checks whether this specific operation has transitioned to `Finished`. The `Finished` result is extracted via `take()`, consuming the `Option`. The operation entry is then removed from the `ops` map to prevent unbounded growth. A second wait on the same operation (impossible at the type level since `wait` consumes `self`) would see `None` and return `UnknownOperation`.

## NoOpCommitSink

Not all test scenarios require full persistence semantics. The `CommitSink` trait defines a per-item lifecycle (begin, upsert findings, finish), and `NoOpCommitSink` is its zero-cost test double:

```rust
// From crates/gossip-scanner-runtime/src/commit_sink.rs

#[derive(Clone, Copy, Debug, Default)]
pub struct NoOpCommitSink;

impl CommitSink for NoOpCommitSink {
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

`NoOpCommitSink` is zero-sized, `Copy`-able, and available in all builds -- not behind a feature gate. It serves both tests that do not need persistence verification and CLI-mode production scanning where findings flow through `EventOutput` instead.

## Error Taxonomy

The `InMemoryPersistenceError` enum provides ten variants organized into three categories:

```rust
// From crates/gossip-persistence-inmemory/src/error.rs

pub enum InMemoryPersistenceError {
    // — Fault injection —
    InjectedSubmissionFailure { store: InMemoryStoreKind },
    InjectedCommitFailure { store: InMemoryStoreKind },

    // — Infrastructure —
    Poisoned { store: InMemoryStoreKind },
    UnknownOperation { store: InMemoryStoreKind, op_id: PendingWriteId },

    // — Data integrity —
    MissingFinding { tenant_id: TenantId, finding_id: FindingId, occurrence_id: OccurrenceId },
    MissingOccurrence { tenant_id: TenantId, occurrence_id: OccurrenceId, observation_id: ObservationId },
    FindingConflict { tenant_id: TenantId, finding_id: FindingId },
    OccurrenceConflict { tenant_id: TenantId, occurrence_id: OccurrenceId },
    ObservationConflict { tenant_id: TenantId, observation_id: ObservationId },
    BatchValidation(PersistenceInputError),
}
```

**Fault injection** variants (`InjectedSubmissionFailure`, `InjectedCommitFailure`) are deterministic, test-controlled failures that exercise error-handling paths. **Infrastructure** variants (`Poisoned`, `UnknownOperation`) indicate internal consistency violations -- a poisoned mutex or a commit handle that references a non-existent operation. **Data integrity** variants cover referential integrity failures (`MissingFinding`, `MissingOccurrence`), immutability violations (`FindingConflict`, `OccurrenceConflict`, `ObservationConflict`), and pre-submission validation failures (`BatchValidation`).

Every variant that involves a specific backend carries an `InMemoryStoreKind` discriminator (`DoneLedger` or `Findings`) so error messages identify the source without requiring downcasting.

## The Lifecycle Diagram

The complete operation lifecycle, from submission through durability, follows this path:

```
submit()
  |
  +-- fail_submit_remaining > 0
  |     --> InjectedSubmissionFailure (no state change)
  |
  +-- allocate monotonic op id
  |
  +-- delay_next > 0 or !auto_complete
  |     --> op inserted as Pending into ops + order queue
  |         --> caller gets StoreHandle
  |             --> wait() blocks on condvar
  |                 --> release_next / release_specific / release_all
  |                       --> finish_op() applies payload
  |                           --> condvar.notify_all()
  |
  +-- auto_complete (default)
        --> B::apply() runs immediately under the lock
            --> op inserted as Finished
                --> condvar.notify_all()
                    --> wait() returns at once
```

In auto-complete mode (the default), writes apply immediately during `batch_upsert` or `upsert_batch`, so `handle.wait()` returns at once. In delayed mode, writes are enqueued as pending operations and only applied when explicitly released. The `finish_op` helper uses a remove-apply-reinsert sequence to work around the borrow checker: it removes the operation from the map, destructures state to get independent mutable references to `durable` and `fail_commit_remaining`, calls `B::apply()`, stores the result, and reinserts the operation.

## Summary

| Component | Role |
|---|---|
| `StoreBackend` trait | Injects domain-specific apply logic into the generic store |
| `InMemoryStoreCore<B>` | Mutex + condvar + fault injection + queue management |
| `StoreState<B>` | Durable state + ops map + order queue + counters |
| `InMemoryDoneLedger` | Reference done-ledger with lattice merge semantics |
| `InMemoryFindingsSink` | Reference findings sink with referential integrity |
| `PendingWriteId` | Opaque handle to a specific pending write |
| `CompletionOrder` | FIFO or LIFO drain direction for release methods |
| `NoOpCommitSink` | Zero-sized `CommitSink` double for tests and CLI mode |
| `InMemoryPersistenceError` | Ten-variant error enum covering faults, infrastructure, and data integrity |

The in-memory backends are not simplifications of the persistence contract. They are the contract, enforced with the same invariants that production backends must satisfy. Tests that pass against `InMemoryDoneLedger` and `InMemoryFindingsSink` exercise the same lattice merge, referential integrity, and two-phase commit protocol that a PostgreSQL or SQLite backend must implement. The conformance harness, covered in the next chapter, makes this equivalence formally verifiable.

## What's Next

Chapter 6 examines the persistence conformance harness -- a backend-agnostic test suite that every `DoneLedger` and `FindingsSink` implementation must pass. The harness verifies idempotent upserts, lattice merge dominance, referential integrity rejection, observation merge semantics, and secret redaction in `Debug` output. The in-memory backends are the first implementations to run through it.
