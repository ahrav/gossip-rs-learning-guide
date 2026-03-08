# The Zombie Worker -- CAS Transactions and Owner Key Fencing in etcd

*Worker A holds shard 42 with fence epoch 3. Its owner key is attached to etcd lease `0x5a01`, which has a 60-second TTL. A JVM sidecar in the same pod triggers a 90-second garbage collection pause. Worker A's process stalls completely — no heartbeats, no lease keep-alive pings. After 60 seconds, etcd's TTL mechanism fires: lease `0x5a01` expires and the owner key at `/gossip/v1/tenants/.../shards/000000000000002a/owner` is automatically deleted. The shard is now unowned. Worker B claims shard 42, receiving fence epoch 4 and a fresh etcd lease `0x7b02`. Worker B begins scanning from the last checkpoint. Thirty seconds later, Worker A's GC pause ends. Its in-memory state still holds a valid `Lease` object: worker ID 7, fence epoch 3, shard 42. Worker A attempts to checkpoint cursor progress at key `"m"`. Without the three-clause owner guard, the checkpoint would need only to match the shard record's `mod_revision` — a value that has changed since Worker B acquired, so the CAS would fail. But imagine a narrower window: Worker A's stale checkpoint arrives between Worker B's acquire and Worker B's first checkpoint, when `mod_revision` has not yet advanced again. A two-clause guard checking only value equality would pass because Worker A still knows the correct `(WorkerId(7), FenceEpoch(3))` encoding. The third clause — lease identity — rejects the write unconditionally. The etcd lease ID on the owner key is now `0x7b02`, not `0x5a01`. Worker A's transaction fails. The zombie is fenced out.*

---

The previous chapter established the etcd backend's configuration, keyspace layout, and binary codec. Those are the foundation. This chapter covers the live operations — the code that runs on every acquire, renew, and checkpoint call against a real etcd cluster. Every mutating operation follows the same pattern: load the current record, validate preconditions locally, build a conditional transaction, submit it, and retry on CAS failure. The pattern is simple to describe and difficult to get right. The subtlety lies entirely in what the conditions check and what happens when they fail.

The etcd backend must solve two problems simultaneously. First, it must implement optimistic concurrency: multiple workers racing to modify the same shard record must serialize correctly without distributed locks. Second, it must implement lease-based fencing: a worker whose etcd lease has expired must be rejected even if it presents credentials that were valid moments ago. The combination of `mod_revision` guards on shard records and three-clause owner key comparisons achieves both properties in a single etcd transaction.

## The EtcdCoordinator Struct

The backend centers on a single struct that owns all connection state and provides the `CoordinationBackend`, `RunManagement`, and `ShardClaiming` trait implementations.

From `backend.rs`:

```rust
pub struct EtcdCoordinator {
    config: EtcdCoordinatorConfig,
    keyspace: EtcdKeyspace,
    runtime: tokio::runtime::Runtime,
    client: etcd_client::Client,
    claim_candidates_scratch: Vec<ShardId>,
}
```

Five fields. `config` carries the validated connection parameters and tuning knobs (owner lease TTL, CAS retry budget). `keyspace` generates deterministic etcd key paths from identity tuples. `client` is the gRPC handle to the etcd cluster. `claim_candidates_scratch` is a reusable buffer for the `claim_next_available` path — it avoids allocating a fresh `Vec` on every claim attempt.

The `runtime` field is a Tokio `current_thread` runtime. The etcd client library (`etcd_client`) is async, but the `CoordinationBackend` trait is synchronous. The backend bridges this gap by owning a single-threaded Tokio runtime and calling `block_on` for every etcd RPC. This is a deliberate architectural choice: the coordination backend runs in a synchronous context (the coordination facade), and embedding async into the trait surface would infect every backend implementation with an executor dependency.

The `connect()` constructor performs a two-phase fail-fast initialization: first it connects to etcd, then it issues a `status` RPC to verify the cluster is reachable and healthy. If either phase fails, the caller gets an error before any coordination operations are attempted.

From `backend.rs`:

```rust
pub fn connect(config: EtcdCoordinatorConfig) -> Result<Self, EtcdCoordinatorError> {
    config.validate()?;

    let runtime = tokio::runtime::Builder::new_current_thread()
        .enable_io()
        .enable_time()
        .build()
        .map_err(EtcdCoordinatorError::RuntimeBuild)?;

    let endpoints = config.endpoints().to_vec();
    let mut connect_opts =
        etcd_client::ConnectOptions::new().with_connect_timeout(DEFAULT_CONNECT_TIMEOUT);
    if let Some((user, password)) = config.auth() {
        connect_opts = connect_opts.with_user(user, password);
    }

    debug_assert!(
        tokio::runtime::Handle::try_current().is_err(),
        "connect() must not be called from within an active Tokio runtime"
    );

    let mut client = runtime
        .block_on(etcd_client::Client::connect(endpoints, Some(connect_opts)))
        .map_err(|source| EtcdCoordinatorError::Etcd {
            operation: EtcdOperation::Connect,
            source,
        })?;

    runtime
        .block_on(client.status())
        .map_err(|source| EtcdCoordinatorError::Etcd {
            operation: EtcdOperation::Status,
            source,
        })?;

    let keyspace = EtcdKeyspace::new(config.namespace_prefix())?;

    Ok(Self {
        config,
        keyspace,
        runtime,
        client,
        claim_candidates_scratch: Vec::new(),
    })
}
```

The `debug_assert` on `Handle::try_current` catches a lethal bug at development time: calling `block_on` from inside an existing Tokio runtime deadlocks the current thread. The assertion fires only in debug builds, keeping production overhead at zero. The two-phase pattern — connect, then status — ensures that a successful `connect()` return means the cluster responded to at least one RPC. A backend that only connects but never verifies could mask a firewall rule that blocks the etcd port but allows the TCP handshake.

## Internal Persistence Types

Every mutating operation begins by loading the current record from etcd. The raw etcd response — a key-value pair with metadata — is decoded into one of three internal types that pair the domain record with its etcd revision.

From `backend.rs`:

```rust
struct PersistedRun {
    record: RunRecord,
    mod_revision: i64,
}
```

```rust
struct PersistedOwner {
    binding: OwnerLeaseValue,
    lease_id: i64,
}
```

```rust
struct PersistedShard {
    record: ShardRecord,
    slab: ByteSlab,
    mod_revision: i64,
    owner: Option<PersistedOwner>,
}
```

`PersistedRun` is straightforward: the decoded `RunRecord` and the `mod_revision` that etcd stamped on the key when it was last written. The `mod_revision` serves as the CAS precondition — a transaction conditioned on `mod_revision == X` will succeed only if no other writer has modified the key since it was read.

`PersistedOwner` pairs the decoded `OwnerLeaseValue` (a `WorkerId` and `FenceEpoch`) with the etcd lease ID that controls the key's TTL. The lease ID is not stored in the value blob itself — it is etcd metadata attached to the key at write time. When the lease expires, etcd deletes the key automatically. The lease ID is critical for the third clause of the fencing guard.

`PersistedShard` combines a `ShardRecord`, the `ByteSlab` that backs its pooled fields (spec, cursor, spawned), the `mod_revision` for CAS, and an optional `PersistedOwner`. The slab travels with the record because pooled fields reference offsets within the slab — separating them would leave dangling slot references. Each `PersistedShard` owns its own slab. There is no shared slab across shards.

`PersistedShard` provides helper methods that the CAS operations use repeatedly:

```rust
impl PersistedShard {
    fn owner_is_live_at(&self, now: LogicalTime) -> bool {
        self.owner.is_some()
            && self
                .record
                .lease_deadline()
                .is_some_and(|deadline| now < deadline)
    }

    fn expected_owner_value(&self) -> Option<Vec<u8>> {
        self.owner
            .as_ref()
            .map(|owner| encode_owner_value(owner.binding.worker, owner.binding.fence))
    }

    fn owner_matches_lease(&self, lease: &Lease) -> bool {
        self.owner.as_ref().is_some_and(|owner| {
            owner.binding.worker == lease.owner() && owner.binding.fence == lease.fence()
        })
    }
}
```

`owner_is_live_at` checks both that an owner key exists and that the logical lease deadline has not passed. `expected_owner_value` re-encodes the current owner binding for use as a CAS comparison value. `owner_matches_lease` validates that the presented lease's worker and fence epoch match the persisted owner.

## The Owner Key and Lease Identity Pattern

Each shard has a separate `/owner` key stored as a child of the shard record key. The etcd keyspace layout places it at `{shard_record_key}/owner`:

```text
/gossip/v1/tenants/{tenant}/runs/{run}/shards/{shard}         # shard record
/gossip/v1/tenants/{tenant}/runs/{run}/shards/{shard}/owner   # owner binding
```

The owner key value is a small binary blob containing just two fields:

From `codec.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct OwnerLeaseValue {
    pub worker: WorkerId,
    pub fence: FenceEpoch,
}
```

```rust
pub fn encode_owner_value(worker: WorkerId, fence: FenceEpoch) -> Vec<u8> {
    let mut out = Vec::with_capacity(3 + 16);
    out.extend_from_slice(VERSION_PREFIX_V1);
    out.push(BlobKind::ShardOwner.as_u8());
    put_u64(&mut out, worker.as_raw());
    put_u64(&mut out, fence.as_raw());
    out
}
```

The encoding is 19 bytes: a 2-byte version prefix (`b"v1"`), a 1-byte blob kind tag (`ShardOwner = 3`), an 8-byte worker ID, and an 8-byte fence epoch. The decoder validates that the fence epoch is at least `FenceEpoch::INITIAL`, rejecting zero-fence blobs as corrupt.

The critical property is that the owner key is attached to a *real etcd lease* with a wall-clock TTL. When `acquire_and_restore_into` creates the owner key, it passes a `PutOptions` with the lease ID. etcd associates the key with that lease. When the lease expires — because the worker died, stalled, or failed to send keep-alive pings — etcd deletes the owner key automatically. No coordination code needs to run. No heartbeat thread needs to detect the failure. The TTL mechanism is etcd's responsibility.

The shard record itself stores the *logical* lease deadline for coordinators to make availability decisions. The logical deadline is computed from the coordination protocol's `LogicalTime` and the run's configured `lease_duration`. It is distinct from the etcd lease TTL: the logical deadline governs protocol-level decisions (is this shard available for claiming?), while the etcd TTL governs the physical owner key lifetime. Both must agree that the lease has expired before a new worker can acquire the shard.

## The Three-Clause Fencing Guard

The core fencing mechanism is a function that returns three `Compare` clauses for an etcd transaction:

From `backend.rs`:

```rust
fn compare_owner_present(
    owner_key: String,
    owner_value: Vec<u8>,
    lease_id: i64,
) -> Vec<Compare> {
    vec![
        Compare::version(owner_key.clone(), CompareOp::Greater, 0),
        Compare::value(owner_key.clone(), CompareOp::Equal, owner_value),
        Compare::lease(owner_key, CompareOp::Equal, lease_id),
    ]
}
```

Three clauses, each checking a different dimension:

**Clause 1 — Existence (`version > 0`).** The owner key must exist. If it has been deleted — by lease expiry, explicit revocation, or another worker's acquire — the transaction fails immediately. This catches the common case where a worker's etcd lease expired while it was stalled.

**Clause 2 — Value equality.** The owner key's value must exactly match the expected `(WorkerId, FenceEpoch)` encoding. If another worker has acquired the shard and written a new owner binding with a different worker ID or a higher fence epoch, this clause rejects the stale worker's write.

**Clause 3 — Lease identity (`lease == expected_lease_id`).** The etcd lease attached to the owner key must be the same lease the current operation expects. This is the clause that closes the zombie worker gap from the opening scenario.

The third clause is necessary because clauses 1 and 2 alone are insufficient in a specific edge case: worker ID reuse. If Worker A crashes, its etcd lease expires, the owner key is deleted, and then Worker A restarts with the same worker ID. If Worker A happens to acquire the shard again before any other worker, the new owner key would have the same `(WorkerId, FenceEpoch)` value as the old one (the fence epoch advances on acquire, so this particular scenario requires a more complex setup). But consider a subtler race: Worker A's checkpoint arrives during the narrow window after Worker B acquired but before Worker B's first mutation changes the `mod_revision`. Even if the value check passes, the lease ID will differ because Worker B received a fresh etcd lease. The lease identity clause is the final defense.

The backend also provides simpler guards for other CAS patterns:

```rust
fn compare_shard_revision(shard_record_key: String, mod_revision: i64) -> Compare {
    Compare::mod_revision(shard_record_key, CompareOp::Equal, mod_revision)
}

fn compare_absent(key: String) -> Compare {
    Compare::version(key, CompareOp::Equal, 0)
}
```

`compare_shard_revision` guards the shard record against concurrent modification — the standard optimistic concurrency check. `compare_absent` ensures a key does not yet exist, used for `create_run` and `register_shards` to prevent duplicate creation.

## The CAS Transaction Pattern

Every mutating operation in the etcd backend follows the same structural pattern:

1. Load the current record from etcd, capturing its `mod_revision`.
2. Validate preconditions locally (lease validity, status, fencing epoch).
3. Mutate the in-memory record to reflect the desired change.
4. Build an etcd `Txn` with `when(compares).and_then(ops)`.
5. Submit the transaction via `etcd_txn()`.
6. If the transaction succeeds, return the result.
7. If the transaction fails (CAS mismatch), sleep with exponential backoff and retry from step 1.
8. If retries exhaust, re-read the record and return whatever domain error is appropriate.

The backoff function prevents thundering-herd contention when multiple workers race on the same shard:

From `backend.rs`:

```rust
fn cas_retry_delay(attempt: usize) -> Duration {
    let base_ms: u64 = 5;
    let max_ms: u64 = 200;
    let exp_ms = base_ms.saturating_mul(1u64 << attempt.min(6)).min(max_ms);

    // +/-50% jitter from sub-second nanoseconds (no RNG dependency).
    let jitter_source = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap_or(Duration::ZERO)
        .subsec_nanos();
    let jitter_frac = (jitter_source % 1000) as f64 / 1000.0; // 0.0 .. 1.0
    let jittered = (exp_ms as f64) * (0.5 + jitter_frac); // 50% .. 150%

    Duration::from_micros((jittered * 1000.0) as u64)
}
```

The delay starts at 5 ms, doubles on each attempt (capped at 200 ms), and applies ±50% jitter derived from the current system time's sub-second nanoseconds. The jitter source avoids pulling in an RNG dependency — sub-second nanoseconds provide sufficient entropy for backoff decorrelation. The retry budget is configured via `optimistic_txn_retries` (default: 8 attempts). After exhaustion, the operation re-reads the shard to determine the actual error. If the shard was acquired by another worker, the re-read produces `AlreadyLeased` or `StaleFence`. If contention is genuinely unexplainable, the backend panics with `fatal_storage_error` — this is a bug, not an expected failure.

The RPC wrappers that the CAS loop calls are thin shims over the async etcd client:

```rust
fn etcd_txn(&self, txn: Txn) -> Result<etcd_client::TxnResponse, EtcdCoordinatorError> {
    let mut client = self.client.clone();
    self.runtime
        .block_on(client.txn(txn))
        .map_err(|source| EtcdCoordinatorError::Etcd {
            operation: EtcdOperation::Txn,
            source,
        })
}
```

Each wrapper clones the client handle (cheap — it is an `Arc` internally), calls `block_on` against the owned single-threaded runtime, and maps errors into the unified `EtcdCoordinatorError` with the appropriate `EtcdOperation` discriminant.

## acquire_and_restore_into

The most complex operation in the backend. It takes an unowned shard and returns a fully populated `AcquireResultView` containing the lease, the shard snapshot (spec, cursor, spawned children), and a capacity hint for the claiming subsystem.

The operation spans approximately 180 lines because it must handle four concerns atomically: creating an etcd lease, writing the owner key with that lease, updating the shard record with a new fence epoch and lease deadline, and cleaning up the etcd lease on failure.

The CAS loop:

From `backend.rs`:

```rust
for attempt in 0..self.config.optimistic_txn_retries() {
    let persisted = match self.load_shard_record(tenant, key) {
        Ok(Some(shard)) => shard,
        Ok(None) => return Err(AcquireError::ShardNotFound { shard: key }),
        Err(err) => {
            return Err(AcquireError::BackendError {
                message: format!("acquire.load_shard: {err}"),
            });
        }
    };

    if persisted.record.tenant != tenant {
        return Err(AcquireError::TenantMismatch { expected: tenant });
    }
    if persisted.record.status != ShardStatus::Active {
        return Err(AcquireError::ShardTerminal {
            shard: key,
            status: persisted.record.status,
        });
    }
    if persisted.owner_is_live_at(now) {
        return Err(AcquireError::AlreadyLeased {
            current_owner: persisted
                .record
                .lease_owner()
                .expect("live owner key must match shard record lease"),
            lease_deadline: persisted
                .record
                .lease_deadline()
                .expect("live owner key must match shard record deadline"),
        });
    }
```

The precondition checks are deterministic. If the shard does not exist, is terminal, belongs to a different tenant, or already has a live owner, the error is returned immediately without entering the CAS retry loop. Only races between concurrent acquirers trigger retries.

After preconditions pass, the operation creates a fresh etcd lease and builds the mutation:

```rust
    let grant = match self.etcd_lease_grant(self.config.owner_lease_ttl_secs()) {
        Ok(g) => g,
        Err(err) => {
            return Err(AcquireError::BackendError {
                message: format!("acquire.lease_grant: {err}"),
            });
        }
    };
    let new_lease_id = grant.id();
    let prior_lease_id = persisted.owner.as_ref().map(|owner| owner.lease_id);

    let mut persisted = persisted;
    let new_fence = persisted.record.advance_fence();
    persisted.record.lease = Some(LeaseHolder::new(worker, new_deadline));
    persisted.record.assert_invariants(&persisted.slab);
    let shard_blob = encode_shard_record(&persisted.record, &persisted.slab)
        .unwrap_or_else(|err| self.fatal_storage_error("acquire.encode_shard", err));
    let owner_blob = encode_owner_value(worker, new_fence);
```

The fence epoch is advanced *before* encoding. `advance_fence` increments the epoch atomically on the in-memory record. The shard record is then encoded with the new lease holder and fence epoch. The owner blob is encoded with the new worker and fence.

The transaction uses two operations: a put on the shard record key (without a lease — the record is permanent) and a put on the owner key (with the new lease ID):

```rust
    let txn = Txn::new().when(compares).and_then(vec![
        TxnOp::put(shard_record_key.into_bytes(), shard_blob, None),
        TxnOp::put(
            owner_key.into_bytes(),
            owner_blob,
            Some(PutOptions::new().with_lease(new_lease_id)),
        ),
    ]);
```

The `compares` vector contains either four clauses (shard revision + three-clause owner guard, if a prior owner exists) or two clauses (shard revision + owner absent, if the shard was unowned). This branching handles the transition from "no owner" to "has owner" as well as "expired owner" to "new owner."

On CAS failure, the freshly granted etcd lease is immediately revoked to prevent lease accumulation:

```rust
    if !response.succeeded() {
        self.best_effort_revoke_lease(new_lease_id);
        std::thread::sleep(cas_retry_delay(attempt));
        continue;
    }
```

On success, the prior lease (if any) is revoked as cleanup, the capacity hint is computed via a lightweight count-only RPC, and the `AcquireResultView` is assembled from the mutated in-memory record:

```rust
    if let Some(old_lease_id) = prior_lease_id {
        self.best_effort_revoke_lease(old_lease_id);
    }

    let lease = Lease::new(
        tenant,
        key.run(),
        key.shard(),
        worker,
        new_fence,
        new_deadline,
    );
    out.reset();
    out.write_spec(
        persisted.record.spec.key_range_start(&persisted.slab),
        persisted.record.spec.key_range_end(&persisted.slab),
        persisted.record.spec.metadata(&persisted.slab),
    );
    out.write_cursor(
        persisted.record.cursor.last_key(&persisted.slab),
        persisted.record.cursor.token(&persisted.slab),
    );
    out.write_spawned_iter(persisted.record.spawned.iter(&persisted.slab));
```

The `AcquireScratch` buffer (`out`) is caller-owned and reusable across acquire calls. The `reset()` / `write_*` / `view()` sequence fills it with the shard's current spec, cursor, and spawned children without allocating a new buffer.

## renew

Renew extends the logical lease deadline in the shard record and refreshes the etcd lease TTL. It does not modify cursor state, fence epoch, or any other shard field.

From `backend.rs`:

```rust
fn renew(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    lease: &Lease,
) -> Result<RenewResult, RenewError> {
    let key = lease.shard_key();

    for attempt in 0..self.config.optimistic_txn_retries() {
        let persisted = match self.load_shard_record(tenant, key) {
            Ok(Some(shard)) => shard,
            Ok(None) => return Err(RenewError::ShardNotFound { shard: key }),
            Err(err) => {
                return Err(RenewError::BackendError {
                    message: format!("renew.load_shard: {err}"),
                });
            }
        };

        validate_lease(now, tenant, lease, &persisted.record)?;
        if !persisted.owner_matches_lease(lease) {
            return Err(RenewError::StaleFence {
                presented: lease.fence(),
                current: persisted.record.fence_epoch,
            });
        }
```

The precondition validation uses the shared `validate_lease` function (defined in `gossip-coordination`) followed by an owner-binding match. If the owner key has been deleted (lease expired) or the fence epoch has advanced (another worker acquired), the renew fails immediately.

The transaction updates only the shard record with a new lease deadline. The owner key value does not change — only the etcd lease TTL is refreshed:

```rust
        let txn = Txn::new().when(compares).and_then(vec![TxnOp::put(
            shard_record_key.into_bytes(),
            shard_blob,
            None,
        )]);
```

The compares include the shard `mod_revision` and the three-clause owner guard. After CAS success, a best-effort `keep_alive_once` call extends the etcd lease:

```rust
        if response.succeeded() {
            let _ = self.etcd_lease_keep_alive_once(old_lease_id);
            // ...
            return Ok(RenewResult {
                new_deadline,
                capacity,
            });
        }
```

The keep-alive is deliberately best-effort. If it fails (transport blip, lease expired in the tiny window between CAS and the keep-alive call), the CAS already committed the new deadline to the shard record. Returning an error would lie about the outcome — the shard record IS updated. The next renew cycle will detect ownership loss via `owner_matches_lease` if the owner key was deleted.

## checkpoint

Checkpoint updates the cursor in the shard record while preserving all fencing invariants. It is the most frequently called operation during normal scanning — every batch of records processed triggers a checkpoint to persist cursor progress.

From `backend.rs`:

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
    let payload_hash = hash_checkpoint_payload(new_cursor);

    for attempt in 0..self.config.optimistic_txn_retries() {
        let persisted = match self.load_shard_record(tenant, key) {
            // ...
        };

        if check_op_idempotency(&persisted.record, op_id, payload_hash)?.is_some() {
            return Ok(IdempotentOutcome::Replayed(()));
        }
        validate_lease(now, tenant, lease, &persisted.record)?;
        if !persisted.owner_matches_lease(lease) {
            return Err(CheckpointError::StaleFence {
                presented: lease.fence(),
                current: persisted.record.fence_epoch,
            });
        }
        validate_cursor_update_pooled(
            new_cursor,
            persisted.record.cursor.last_key(&persisted.slab),
            persisted.record.spec.key_range_start(&persisted.slab),
            persisted.record.spec.key_range_end(&persisted.slab),
        )?;
```

Checkpoint adds two checks that renew does not need. First, `check_op_idempotency` scans the shard's op log for a matching `(op_id, payload_hash)` pair. If found, the checkpoint was already applied — the operation returns `Replayed` without modifying anything. This is the idempotency mechanism described in chapter 5. Second, `validate_cursor_update_pooled` verifies that the new cursor is within the shard's key range bounds — a structural invariant that prevents cursor corruption.

After validation, the cursor is updated in-place on the in-memory record, an op log entry is pushed, and the CAS transaction is built:

```rust
        let mut persisted = persisted;
        persisted
            .record
            .cursor
            .update_from_ref(new_cursor, &mut persisted.slab)?;
        persisted.record.op_log_push(OpLogEntry::new(
            op_id,
            OpKind::Checkpoint,
            OpResult::Completed,
            payload_hash,
            now,
        ));
        persisted.record.assert_invariants(&persisted.slab);
```

The transaction guards are identical to renew: shard `mod_revision` plus the three-clause owner guard. The puts include only the shard record — the owner key is not modified by checkpoint.

## RunManagement Implementation

### create_run

Run creation uses the simplest CAS pattern: a single `compare_absent` guard ensuring the run key does not already exist.

From `backend.rs`:

```rust
fn create_run(
    &mut self,
    now: LogicalTime,
    tenant: TenantId,
    run: RunId,
    config: RunConfig,
) -> Result<RunRecord, CreateRunError> {
    config.assert_valid();

    let record = RunRecord {
        tenant,
        run,
        config,
        status: RunStatus::Initializing,
        created_at: now,
        completed_at: None,
        root_shards: Vec::new(),
        op_log: RingBuffer::new(),
    };
    record.assert_invariants();

    let key = self.keyspace.run_record_key(tenant, run);
    let blob = encode_run_record(&record);
    let txn = Txn::new()
        .when(vec![Self::compare_absent(key.clone())])
        .and_then(vec![TxnOp::put(key.into_bytes(), blob, None)]);
```

If the key already exists, the transaction fails and the operation returns `RunAlreadyExists`. There is no retry loop — duplicate run creation is a caller error, not a contention scenario.

### register_shards

Shard registration is the most complex run-management operation. It must atomically create all shard records, transition the run from `Initializing` to `Active`, and populate the active-index entries — all in a single etcd transaction.

The operation enforces a hard limit derived from etcd's default `--max-txn-ops` cap of 128:

From `backend.rs`:

```rust
const MAX_SHARDS_PER_ETCD_TXN: usize = 41;
```

Each shard generates 3 ops (compare-absent, put-record, put-active-index) plus 3 fixed ops for the run record (compare-run-revision, put-run-record, put-run-active-index), giving `(128 - 3) / 3 = 41` as the maximum shard count per transaction. Exceeding this limit returns `ResourceExhausted { resource: "etcd_txn_ops" }` immediately.

The compares vector ensures atomicity: one `compare_run_revision` guard on the run record, plus one `compare_absent` per shard key:

```rust
    let mut compares = Vec::with_capacity(1 + shards.len());
    let run_key = self.keyspace.run_record_key(tenant, run);
    compares.push(Self::compare_run_revision(
        run_key.clone(),
        persisted_run.mod_revision,
    ));

    for shard in shards {
        let shard_key = self.keyspace.shard_record_key(tenant, run, shard.shard());
        compares.push(Self::compare_absent(shard_key.clone()));
        let shard_blob =
            self.build_root_shard_blob(tenant, run, cursor_semantics, shard)?;
        txn_ops.push(TxnOp::put(shard_key.into_bytes(), shard_blob, None));

        let active_index = self
            .keyspace
            .shard_active_index_key(tenant, run, shard.shard());
        txn_ops.push(TxnOp::put(
            active_index.into_bytes(),
            Vec::<u8>::new(),
            None,
        ));
    }
```

The run record is updated to `Active` status, the root shards list is populated, and a `RegisterShards` op log entry is pushed — all before the transaction is submitted. If the CAS fails, the retry loop re-reads the run and validates idempotency before trying again.

### get_run, list_shards_into, collect_claim_candidates_into

The read-side operations are simpler. `get_run` loads a single run record by exact key. `list_shards_into` performs a prefix scan of all shard records under a run, filters them against a `ShardFilter`, and sorts the results by key range start. `collect_claim_candidates_into` scans for active shards without a live owner, returning them as claim candidates sorted by shard ID.

Both scan operations use `scan_run_shards`, which issues a single etcd prefix-range GET on the `shards/` subtree and partitions the response into record KVs and owner KVs. The trailing-slash convention in the scan prefix (`shards/` not `shards`) is critical — without it, an etcd prefix scan would also match `shards_active/` keys, pulling in active-index entries that are not shard records.

```rust
fn scan_run_shards(
    &self,
    tenant: TenantId,
    run: RunId,
) -> Result<Vec<PersistedShard>, EtcdCoordinatorError> {
    let prefix = self
        .keyspace
        .shard_records_scan_prefix(tenant, run)
        .into_bytes();
    let response = self.etcd_get(prefix, Some(GetOptions::new().with_prefix()))?;

    let mut owner_map = HashMap::<Vec<u8>, PersistedOwner>::new();
    let mut record_kvs = Vec::<(Vec<u8>, Vec<u8>, i64)>::new();

    for kv in response.kvs() {
        if kv.key().ends_with(b"/owner") {
            let owner = self.decode_owner_kv(kv)?;
            owner_map.insert(kv.key().to_vec(), owner);
        } else {
            record_kvs.push((kv.key().to_vec(), kv.value().to_vec(), kv.mod_revision()));
        }
    }
```

Owner bindings are matched to their parent shard record by key suffix convention: the code builds the expected owner key by appending `/owner` to the record key. Orphaned owner keys (owner with no matching shard record) trigger an invariant-violation error.

## Slab Management

Each `PersistedShard` owns an independent `ByteSlab` sized as a heuristic of the blob length:

From `backend.rs`:

```rust
fn make_decode_slab(blob_len: usize) -> ByteSlab {
    let cap = blob_len
        .saturating_mul(4)
        .clamp(MIN_DECODE_SLAB_CAPACITY, MAX_DECODE_SLAB_CAPACITY);
    ByteSlab::with_capacity(cap)
}
```

The 4× multiplier accounts for pooled field overhead (slab headers, alignment padding) relative to the raw wire bytes. The capacity is clamped between 4 KiB (`MIN_DECODE_SLAB_CAPACITY`) and 256 KiB (`MAX_DECODE_SLAB_CAPACITY`). There is no shared slab across reads — each `load_shard_record` call creates a fresh slab. This avoids the complexity of slab lifetime management across multiple records and makes each `PersistedShard` independently droppable.

For encoding during `register_shards`, a separate capacity estimator computes the slab size needed for a single initial shard:

```rust
fn build_slab_capacity_for_initial_shard(input: &InitialShardInput<'_>) -> usize {
    let spec = input.spec();
    let cursor = input.cursor();
    let cursor_last = cursor.last_key().map_or(0, |key| key.len());
    let cursor_token = cursor.token().map_or(0, |token| token.len());
    let needed = spec.key_range_start().len()
        + spec.key_range_end().len()
        + spec.metadata().len()
        + cursor_last
        + cursor_token
        + 256;
    needed.max(DEFAULT_BUILD_SLAB_FLOOR)
}
```

The 256-byte padding covers slab header overhead and alignment. The floor (`DEFAULT_BUILD_SLAB_FLOOR = 1024`) ensures tiny shards still get a workable slab.

## Unimplemented Operations

The etcd backend does not yet persist all coordination operations. Eight operations panic with a descriptive message when called, signaling that the code path must not be reached until persistence logic is added:

```rust
fn fail_unimplemented<T>(&self, operation: &'static str) -> T {
    panic!(
        "EtcdCoordinator::{operation} is not yet persisted to etcd; \
         this operation must be implemented before it is callable"
    );
}
```

The affected operations are `complete`, `park_shard`, `split_replace`, `split_residual`, `complete_run`, `fail_run`, `cancel_run`, and `unpark_shard`. These are the terminal and lifecycle transition operations — they change shard or run status in ways that require additional CAS patterns beyond what the hot path uses. `complete` must atomically update the shard status and remove the active-index entry. `split_replace` and `split_residual` must create child shard records in the same transaction that marks the parent as split. The run transition operations (`complete_run`, `fail_run`, `cancel_run`) must verify that all shards have reached a terminal state before transitioning the run. The acquire/renew/checkpoint hot path and the run creation/registration path are fully persisted; the terminal operations will follow as the coordination surface is extended.

## BackendError Propagation

Every domain error type in the coordination crate (`AcquireError`, `RenewError`, `CheckpointError`, `CreateRunError`, `RegisterShardsError`, `GetRunError`, `ClaimError`) carries a `BackendError { message: String }` variant. The etcd backend uses this variant to surface infrastructure failures without exposing `EtcdCoordinatorError` through the trait boundary.

The pattern is consistent across all operations:

```rust
Err(err) => {
    return Err(AcquireError::BackendError {
        message: format!("acquire.load_shard: {err}"),
    });
}
```

The message prefix (`acquire.load_shard`, `renew.txn`, `checkpoint.encode_shard`) identifies which operation and which internal step failed. The `EtcdCoordinatorError`'s `Display` implementation includes the etcd operation tag and the underlying gRPC error, so the formatted message carries enough context for diagnosis without requiring callers to depend on the etcd-specific error type.

This design preserves the trait's backend-agnostic contract: the in-memory coordinator never produces `BackendError` (it has no infrastructure to fail), while the etcd coordinator translates every infrastructure failure into the same variant. Callers can treat `BackendError` as a transient failure eligible for retry after backoff, regardless of which backend is in use.

## Summary

The etcd backend implements coordination operations through a disciplined pattern: load, validate, mutate, CAS-submit, retry. Three internal types (`PersistedRun`, `PersistedShard`, `PersistedOwner`) pair domain records with their etcd metadata. The three-clause owner guard (`compare_owner_present`) checks existence, value equality, and lease identity in a single transaction — closing the zombie worker gap that simpler guards leave open. The CAS retry loop uses exponential backoff with jitter to prevent thundering-herd contention. The `acquire_and_restore_into` operation creates etcd leases, writes owner keys with TTL, and assembles full shard snapshots in a single transactional flow. The `renew` and `checkpoint` operations use the same three-clause guard to ensure that only the current lease holder can extend or advance shard state. Run management operations (`create_run`, `register_shards`) use simpler CAS patterns appropriate to their contention profile. Eight terminal operations remain unimplemented as panicking stubs.

## What's Next

The next chapter explores the integration test suite that validates these operations against a live etcd cluster — including lease expiry scenarios, concurrent acquire races, and the transaction size guard that caps `register_shards` at 41 shards per batch. Those tests exercise the exact CAS patterns described here, confirming that the three-clause guard rejects stale workers and that the retry loop converges under contention.
