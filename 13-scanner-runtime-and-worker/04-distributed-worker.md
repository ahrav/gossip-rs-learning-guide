# The Distributed Loop -- ShardLease, CoordinationFacade, and the Claim-Execute-Complete Cycle

*Worker 12 acquires shard `fs-0xf1` from the coordinator. The scan completes successfully: 340 files scanned, 7 findings emitted, all identity records persisted through the receipt-driven commit pipeline. The `PrefixCheckpointAggregator` confirms that all 340 sequence numbers produced durable receipts. The worker calls `complete` on the coordinator with a receipt-derived cursor. The coordinator records progress. Three seconds later, Worker 12 claims another shard. If the process had crashed between the last durable commit and the `complete` call, the coordination backend would reclaim the lease when its deadline expires. Another worker would acquire the shard, resume from the last durable checkpoint, and reprocess only the uncommitted items. The persistence backends are idempotent for duplicate writes, so double-committed items produce no data corruption. This is at-least-once delivery: the system guarantees that every item is scanned and committed at least once, tolerating duplication at the persistence layer rather than risking missed items.*

---

The `distributed` module in `gossip-scanner-runtime` implements the production worker loop. It connects the runtime's scan dispatch to the `CoordinationFacade`: claiming shard leases, executing filesystem scans with receipt-driven durability, and completing shards with cursor advancement. This chapter examines the full claim-execute-complete cycle.

## 1. The Architecture

The distributed runtime diagram from the module documentation:

```text
┌──────────────┐    claim     ┌────────────────────┐
│ Coordinator  │ ──────────>  │  run_worker loop   │
│ (CoordFacade)│ <────────── │  (claim/scan/done) │
└──────────────┘   complete   └────────┬───────────┘
                                       │
                             ┌─────────▼──────────┐
                             │ run_filesystem_lease│
                             │  (per shard)        │
                             └─────────┬──────────┘
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
         ┌────────────────┐  ┌──────────────────┐ ┌─────────────┐
         │ scan engine    │  │ ReceiptCommitSink │ │ commit      │
         │ (scheduler)    │──│ (CommitSink impl) │─│ pipeline    │
         └────────────────┘  └──────────────────┘ │ + drainer   │
                                                  └──────┬──────┘
                                                         ▼
                                                ┌────────────────┐
                                                │ Checkpoint     │
                                                │ Aggregator     │
                                                └────────────────┘
```

The `run_worker` function is the top-level entry point. It loops over shard claims until the coordinator has no more work. Each iteration calls `run_filesystem_lease` for scan-commit-checkpoint execution, then `complete_shard` for coordinator cursor advancement.

## 2. The CoordinationFacade

The distributed worker is generic over `C: CoordinationFacade`. The facade provides three operations the loop needs:

- **`claim_next_available`** -- acquire the next shard lease. Returns an `AcquireResultView` containing the lease metadata and shard spec.
- **`complete`** -- mark a shard as done with a cursor position, fencing epoch, and deterministic operation ID.
- **`get_run_progress`** -- query how many shards remain active. Used to detect when the run is fully settled (zero active leases).

Production uses `EtcdCoordinator` from `gossip-coordination-etcd`. The worker does not know or care which backend it runs against -- the same loop works with in-memory test implementations.

## 3. The ShardLease

Each claim produces a `ShardLease`:

```rust
pub struct ShardLease {
    shard_id: Arc<str>,
    lease: Lease,
    range_start: Vec<u8>,
    scan_config: FsScanConfig,
    write_context: WriteContext,
    tenant_secret_key: TenantSecretKey,
}
```

The lease is constructed from the acquired coordination result by `build_lease_from_acquire`:

```rust
fn build_lease_from_acquire(
    acquired: AcquireResultView<'_>,
    identity: &WorkerIdentity,
) -> Result<ShardLease> {
    let spec = acquired.snapshot.spec();
    let write_context = WriteContext::new(
        acquired.lease.tenant(),
        identity.policy_hash,
        acquired.lease.run(),
        acquired.lease.shard(),
        acquired.lease.fence(),
    );

    Ok(ShardLease::new(
        Arc::from(acquired.lease.shard().to_string()),
        acquired.lease,
        spec.key_range_start().to_vec(),
        scan_config_from_spec(spec, &identity.scan_template)?,
        write_context,
        identity.tenant_secret_key,
    ))
}
```

The `WriteContext` bundles tenant, policy hash, run, shard, and fencing epoch. Every persistence row emitted under this lease carries this context for routing and deduplication.

## 4. The Claim Loop

The `claim_next_lease` function handles the retry logic:

```rust
fn claim_next_lease<C>(
    coordinator: &mut C,
    identity: &WorkerIdentity,
    scratch: &mut AcquireScratch,
) -> Result<Option<ShardLease>, DistributedRuntimeError>
where
    C: CoordinationFacade,
{
    loop {
        let now = wall_clock_now();
        match coordinator.claim_next_available(
            now, identity.tenant, identity.run, identity.worker, scratch,
        ) {
            Ok(acquired) => {
                return build_lease_from_acquire(acquired, identity)
                    .map(Some)
                    .map_err(DistributedRuntimeError::Coordinator);
            }
            Err(ClaimError::NoneAvailable { earliest_deadline }) => {
                let progress = coordinator.get_run_progress(
                    now, identity.tenant, identity.run,
                )?;
                if progress.active() == 0 {
                    return Ok(None);  // Run is fully settled.
                }
                std::thread::sleep(claim_retry_delay(now, earliest_deadline));
            }
            Err(ClaimError::Throttled { retry_after }) => {
                std::thread::sleep(claim_retry_delay(now, Some(retry_after)));
            }
            Err(error) => {
                return Err(DistributedRuntimeError::Coordinator(error.into()));
            }
        }
    }
}
```

Three outcomes: successful claim, "none available" (other workers hold all shards), or terminal error. On "none available," the worker checks whether any leases are still active. If zero remain, the run is done and the loop exits with `Ok(None)`. Otherwise, the worker sleeps until the earliest lease deadline and retries. The sleep duration is computed from the coordinator-provided deadline: `deadline - now`, clamped to at least 1 ms. Without a deadline, a fixed 25 ms delay is used.

## 5. The Per-Shard Execution: run_filesystem_lease

Each claimed shard is executed by `run_filesystem_lease`, which orchestrates the full scan-commit-checkpoint pipeline:

```rust
fn run_filesystem_lease<F, D>(
    recorder: Arc<dyn CoordinationEventRecorder>,
    persistence: &DistributedPersistence<F, D>,
    lease: &ShardLease,
    config: DistributedRuntimeConfig,
) -> Result<(ScanReport, Option<Cursor>), DistributedRuntimeError>
```

The function proceeds in four phases:

### 5.1 Setup

1. Validate scan budgets (`config.budgets.validate()`).
2. Build the scan configuration: clone the lease's `scan_config`, override `workers = 1` (single-threaded for monotonic sequence assignment), enable `persist_findings = true`.
3. Build the detection engine from the scan config's rules and transforms.
4. Create a `rule_fingerprint` closure from the engine for stable persistence-layer fingerprints.
5. Create a `CoordinationEventSink` for telemetry.
6. Create a `CommitPipeline` with bounded queues.
7. Split the pipeline into a `submitter` and `drainer`.
8. Create a `ReceiptCommitSink` wired to the submitter.

### 5.2 Concurrent Execution

The scan and commit draining run concurrently in a `std::thread::scope`:

```rust
let (outcome, submitted, stage_result) = std::thread::scope(|scope| {
    let stage_handle = scope.spawn(move || drain_commit_stage(drainer, write_context, max_buffered));

    let outcome = scan_fs_with_prebuilt_engine(&scan_config, engine, &sink, &commit, &cancel);
    let submitted = commit.finish();
    let stage_result = join_scoped(stage_handle, "receipt checkpoint drain thread");

    (outcome, submitted, stage_result)
});
```

The main thread runs the filesystem scan. The spawned thread drains commit-stage outcomes and feeds the checkpoint aggregator. When the scan completes, `commit.finish()` consumes the `ReceiptCommitSink` and verifies no items remain in-flight.

### 5.3 Post-Scan Verification

After both threads join:

1. `resolve_filesystem_lease_results` unwraps the three concurrent results, surfacing scan errors before durability errors (a broken scan often cascades into downstream drain errors).
2. `wait_for_submitted_commits` sorts both the submitted and committed sequence number lists and verifies they match pairwise. Any mismatch is a durability pipeline bug.

### 5.4 Checkpoint

If the shard committed at least one item:

1. `aggregator.prepare_checkpoint()` walks the reorder buffer and produces a `PendingPrefixCheckpoint`.
2. The function verifies that the prepared checkpoint covers exactly the expected number of units.
3. A `CheckpointCommitReceipt` is constructed with the checkpoint scope and a logical time derived from the last sequence number.
4. `aggregator.acknowledge_checkpoint(receipt)` drains the acknowledged prefix from the buffer.

The function returns `(ScanReport, Some(cursor))` for non-empty shards, or `(ScanReport, None)` for empty shards.

### Design Choice: Single Worker Thread

The scan runs with `workers = 1` so that `ReceiptCommitSink` sequence assignment remains monotonic without cross-thread synchronization. The commit pipeline provides the parallelism boundary: scan execution and durable commit proceed concurrently on separate threads. This is a deliberate trade-off: single-threaded scanning is slower for large directories, but it eliminates an entire class of concurrency bugs in the sequence numbering logic.

## 6. Shard Completion

After `run_filesystem_lease` returns, `complete_shard` records the result with the coordinator:

```rust
fn complete_shard<C>(
    coordinator: &mut C,
    identity: &WorkerIdentity,
    lease: &ShardLease,
    checkpoint: Option<&Cursor>,
) -> Result<(), DistributedRuntimeError>
```

The final cursor is chosen as:
- The receipt-driven checkpoint cursor when items were committed.
- The shard's `range_start` when zero items were committed but the range is non-empty.
- A synthetic sentinel key (`b"\x00"`) when the shard has no range start (empty lower bound).

Completion uses a deterministic `OpId` derived from `(shard_key, fence_epoch, OpKind::Complete)` so replayed calls are idempotent:

```rust
fn deterministic_op_id(key: ShardKey, fence: FenceEpoch, op_kind: OpKind) -> OpId {
    let mut hasher = domain_hasher("gossip.scanner_runtime.distributed.op_id");
    key.run().write_canonical(&mut hasher);
    key.shard().write_canonical(&mut hasher);
    fence.write_canonical(&mut hasher);
    op_kind.as_u8().write_canonical(&mut hasher);
    OpId::from_raw(finalize_64(&hasher))
}
```

If the coordinator reports the completion was already applied (idempotent replay), the function logs an info message and succeeds.

## 7. The run_worker Loop

The top-level function ties everything together:

```rust
pub fn run_worker<C, F, D>(
    coordinator: &mut C,
    identity: WorkerIdentity,
    persistence: DistributedPersistence<F, D>,
    config: DistributedRuntimeConfig,
) -> Result<DistributedRunReport, DistributedRuntimeError>
where
    C: CoordinationFacade,
    F: FindingsSink + Clone + Send + Sync + 'static,
    D: DoneLedger + Clone + Send + Sync + 'static,
```

The loop:

1. **Claims** the next shard via `claim_next_lease`. Returns `Ok(report)` when no shards remain.
2. **Executes** via `run_filesystem_lease`. Fail-fast on error.
3. **Completes** via `complete_shard`. Fail-fast on error.
4. Increments `shards_scanned` and repeats.

The `DistributedRunReport` tracks `leases_seen` and `shards_scanned`. The invariant `shards_scanned <= leases_seen` holds because claiming increments `leases_seen` before scanning increments `shards_scanned`.

## 8. Error Classification

The `DistributedRuntimeError` classifies failures by origin:

```rust
pub enum DistributedRuntimeError {
    Coordinator(AnyError),
    Runtime(ScanRuntimeError),
    Durability(AnyError),
}
```

**`Coordinator`** -- shard claiming, progress lookup, or completion failed. May be transient (network partition) or terminal (run not found).

**`Runtime`** -- the scan itself failed. Path validation, engine construction, or parallel scanner errors.

**`Durability`** -- the receipt-driven commit pipeline could not confirm durable progress. Commit stage failures, aggregation violations, or sequence number mismatches.

The worker loop terminates on the first error from any category. Uncompleted leases are not explicitly released; the coordination backend reclaims them when their deadlines expire.

## 9. The Four Invariants

The distributed module enforces four structural invariants documented at the module level:

1. **Receipt-only checkpoint advancement.** Checkpoint progress is derived exclusively from durable commit receipts, never from raw scan completion.

2. **Single-threaded scan execution.** Each shard runs with `workers = 1` so the `ReceiptCommitSink` sequence counter remains monotonic without cross-thread synchronization.

3. **At-least-once delivery.** The commit pipeline tolerates duplicate writes for the same `(write_context, item_key)` pair. Persistence backends must be idempotent.

4. **Fail-fast after claim.** Once a shard is claimed, any scan, commit, or completion error terminates the worker loop. Uncompleted leases expire via coordination-layer deadlines.

## What's Next

[Chapter 5](05-the-worker-binary.md) examines the `gossip-worker` binary: the env-var configuration system, the `ResolvedWorkerConfig` dispatch model, and the production composition root that wires the generic runtime to real etcd and PostgreSQL backends.
