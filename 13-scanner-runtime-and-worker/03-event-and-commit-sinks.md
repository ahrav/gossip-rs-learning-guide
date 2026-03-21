# The Output Channels -- Event Sinks and the Receipt-Driven Commit Pipeline

*A distributed worker scans shard `fs-0xe5` and discovers a leaked API key in `/data/repos/acme/services/auth/token.env` at byte range `[128, 172)`. The engine produces a `FindingRecord` with `norm_hash = 0xa3f7...c1` and `rule_id = 9`. The event sink streams the finding as a JSONL line: `{"path":"/data/repos/acme/services/auth/token.env","rule_name":"generic-api-key","start":128,"end":172,"source":"fs","confidence_score":8}`. The operator sees the line in the live feed within milliseconds. Good -- real-time visibility. But the downstream deduplication service queries the finding store for shard `fs-0xe5`. The store returns zero records. The event stream carries the path, the rule name, the byte range. It does not carry the `finding_id` -- the stable identity that is deterministic across scan runs. It does not carry the `occurrence_id` -- the instance-specific identity that distinguishes this match at this byte offset in this file version. The JSONL format was designed for human-readable telemetry, not for identity-bearing persistence. Without the `ReceiptCommitSink` to derive `norm_hash` to `secret_hash` to `finding_id` to `occurrence_id`, the finding is visible but untrackable. It cannot be deduplicated against previous scans. It cannot be linked to a key rotation event. It exists in the telemetry stream as an ephemeral event and vanishes when the stream closes.*

---

The runtime defines two parallel output channels with fundamentally different contracts. The event channel streams findings as formatted records for operators and monitoring systems -- it is fire-and-forget, best-effort, and format-flexible. The commit channel derives stable identities and persists them for deduplication and audit -- it is durable, identity-bearing, and error-propagating. This chapter examines both sides: the event formats for CLI output, the `CoordinationEventSink` for distributed telemetry, and the receipt-driven commit pipeline that replaces the event stream's ephemeral model with durable persistence.

## 1. The Four Event Formats

The runtime supports four output formats, selectable via the `EventFormat` enum from `lib.rs`:

```rust
pub enum EventFormat {
    #[default]
    Jsonl,
    Text,
    Json,
    Sarif,
}
```

Each format is implemented as a struct in `event_sink.rs` that implements the `EventOutput` trait from `scanner-scheduler`.

### 1.1 JSONL -- One Record Per Line

The `JsonlEventSink` is the default format. Each event is a single JSON object followed by a newline:

```rust
pub struct JsonlEventSink<W: Write + Send> {
    writer: Mutex<BufWriter<W>>,
}
```

The `Mutex<BufWriter<W>>` pattern allows safe concurrent access from scan worker threads while maintaining output buffering. The sink implements both `EventOutput` (for core events: findings, progress, summary, diagnostics) and `GitEventOutput` (for git-specific events: commit metadata, identity dictionary entries).

Finding records intentionally omit a `type` field for scanner-rs parity. `BrokenPipe` is treated as success to support piped CLI output (e.g., `gossip-worker fs /data | head -10`).

### 1.2 Text -- Human-Readable Output

The `TextEventSink` formats events as human-readable lines. In default mode, findings are compact: `path:start-end  rule_name  (source)`. In verbose mode (`--verbose`), findings get structured blocks with labeled fields. The text sink uses the same `Mutex<BufWriter<W>>` pattern as JSONL.

### 1.3 JSON and SARIF

The `JsonEventSink` produces a streaming JSON array. The `SarifEventSink` produces a SARIF 2.1.0 document. Both are wired through the same trait boundary.

## 2. The CoordinationEventSink -- Distributed Telemetry

In distributed mode, event output flows through the `CoordinationEventSink` rather than a CLI formatter. From `coordination_sink.rs`:

```rust
pub struct CoordinationEventSink {
    shard_id: Arc<str>,
    recorder: Arc<dyn CoordinationEventRecorder>,
    core_error_logged: AtomicBool,
    git_error_logged: AtomicBool,
}
```

The sink implements both `EventOutput` and `GitEventOutput`. Each event is converted into an owned representation and forwarded to the recorder:

```rust
impl EventOutput for CoordinationEventSink {
    fn emit_core(&self, event: CoreEvent<'_>) {
        let owned = OwnedCoreEvent::from_core(event);
        if let Err(error) = self.recorder.record_core_event(&self.shard_id, owned) {
            self.warn_recorder_failure(
                &self.core_error_logged,
                error,
                "recorder failed to persist core event; subsequent failures suppressed",
            );
        }
    }

    fn flush(&self) {}
}
```

Recorder failures are intentionally non-fatal. The first failure for each event kind (core / git) is logged; subsequent failures are suppressed to avoid flooding logs during sustained recorder outages. This design reflects a fundamental principle: **authoritative durability is enforced by the receipt-driven commit pipeline, not by event recording.** The event channel is best-effort telemetry.

### 2.1 The CoordinationEventRecorder Trait

The recorder trait defines three operations:

```rust
pub trait CoordinationEventRecorder: Send + Sync {
    fn record_core_event(&self, shard_id: &str, event: OwnedCoreEvent) -> Result<()>;
    fn record_git_event(&self, shard_id: &str, event: StoredGitEvent) -> Result<()>;
    fn record_commit_progress(&self, shard_id: &str, event: CommitProgressRecord) -> Result<()>;
}
```

The `record_commit_progress` method receives lifecycle markers (`Begin` / `Finish`) that track item processing for observability. These are best-effort breadcrumbs, not durability signals.

The production implementation (`ProductionCoordinationEventRecorder` in `gossip-worker/src/recorder.rs`) emits structured `tracing` events while enforcing a strict safe-field policy: raw item keys, paths, and git identity bytes are never emitted directly. Toxic fields are reduced to `len + short hash` digests to prevent accidental secret leakage into log aggregation systems.

### 2.2 CommitProgressRecord

The `CommitProgressRecord` enum carries lifecycle markers:

```rust
pub enum CommitProgressRecord {
    Begin {
        write_context: WriteContext,
        item_key: ItemKey,
        size_hint: Option<u64>,
    },
    Finish {
        write_context: WriteContext,
        item_key: ItemKey,
    },
}
```

These markers are emitted by the `ReceiptCommitSink` during item processing. They are observability-only -- the `Finish` marker means the item was submitted to the commit pipeline, not that the commit landed durably. Durability confirmation flows through the receipt/checkpoint path.

## 3. The Receipt-Driven Commit Pipeline

The commit pipeline is the durable counterpart to the event channel. While events stream findings for real-time visibility, the commit pipeline persists identity-bearing records through a receipt-driven model.

### 3.1 ReceiptCommitSink -- The Bridge

The `ReceiptCommitSink` (in `distributed.rs`) implements the `CommitSink` trait and bridges scan-loop callbacks into the commit pipeline. Its internal structure:

```rust
struct ReceiptCommitSink {
    shard_id: Arc<str>,
    recorder: Arc<dyn CoordinationEventRecorder>,
    write_context: WriteContext,
    tenant_secret_key: TenantSecretKey,
    rule_fingerprint: Arc<dyn Fn(u32) -> RuleFingerprint + Send + Sync>,
    submitter: CommitPipelineSender,
    next_sequence_no: AtomicU64,
    in_flight: Mutex<BTreeMap<ItemKey, InFlightItem>>,
    submitted: Mutex<Vec<u64>>,
    progress_error_logged: AtomicBool,
}
```

Key fields:

- **`rule_fingerprint`**: a closure that maps engine rule IDs to stable persistence-layer fingerprints. Called during `finish_item` translation.
- **`submitter`**: the sending half of the bounded commit pipeline channel.
- **`next_sequence_no`**: atomic counter for monotonic sequencing.
- **`in_flight`**: tracks items between `begin_item` and `finish_item`.

The sink is driven by a single-threaded drain loop. The interior `Mutex` fields satisfy `Send + Sync` without introducing real contention. `Ordering::Relaxed` is sufficient for `next_sequence_no` because there is no concurrent caller.

### 3.2 The Translation Step

When `finish_item` is called, `ReceiptCommitSink` translates the accumulated item state into persistence rows:

1. Derives logical timing from the sequence number: `(2n, 2n+1)`.
2. Falls back to a weak version when no explicit version was supplied.
3. Attaches a display-only `Location` for UTF-8 item keys.
4. Calls `translate_item_result` for deterministic row derivation.
5. Wraps the result into a `QueuedCommit` and submits to the pipeline.

The `CommitSink` surface provides only start/end byte offsets. Root-hint fields are unavailable through the bridge, so both `root_hint_start` and `root_hint_end` mirror `span_start` and `span_end`. This is safe because root-hint fields never participate in persistence identity derivation.

### 3.3 CommitPipeline -- The Bounded Queue

The `CommitPipeline` (in `commit_pipeline.rs`) is the structural choke point between scan execution and durable commit. It uses `std::sync::mpsc::sync_channel` for both the execution queue and the outcome queue:

```rust
pub struct CommitPipelineConfig {
    pub execution_queue_capacity: usize,
    pub outcome_queue_capacity: usize,
}
```

The pipeline starts a worker thread that:

1. Receives `QueuedCommit` from the execution queue.
2. Writes findings through `FindingsSink`, then the done-ledger row through `DoneLedger`.
3. Emits `CommitStageOutput::Committed` (with a `CheckpointAggregatorInput`) or `CommitStageOutput::Failed` to the outcome queue.

The bounded channels ensure that slow sinks pause scanning: when the commit worker is blocked in `ResultCommitter`, the execution queue fills, and `submit()` blocks. Memory usage is bounded by the queue capacity rather than by scan speed.

### 3.4 ResultCommitter -- The Write Path

The `ResultCommitter` (in `result_committer.rs`) implements the authoritative write sequence:

1. Validate the commit request (one done-ledger row, consistent tenant, valid spans).
2. Write findings, occurrences, and observations to `FindingsSink`.
3. Wait for findings durability.
4. Write the done-ledger row to `DoneLedger`.
5. Wait for done-ledger durability.
6. Return a `UnitCommitReceipt`.

The ordering is critical: findings must be durable before the done-ledger row. If the process crashes between steps 3 and 4, the missing done-ledger row causes re-scan on retry. If the order were reversed, a crash between the done-ledger write and the findings write would leave a "scanned clean" marker with no findings -- silent data loss.

### 3.5 PrefixCheckpointAggregator -- Receipt-Driven Progress

The `PrefixCheckpointAggregator` (in `checkpoint_aggregator.rs`) consumes durable receipts and tracks the contiguous committed prefix:

- Receipts arrive out of order and are buffered by sequence number.
- `prepare_checkpoint()` walks the buffer from the lowest sequence forward, collecting contiguous receipts into a `PendingPrefixCheckpoint`.
- `acknowledge_checkpoint()` drains the acknowledged prefix from the buffer.

The aggregator enforces ownership, per-unit receipt, monotonic key ordering (for ordered-content), and conflict-free duplicate detection. The checkpoint cursor produced by the aggregator is the authoritative progress signal for the coordination layer -- not the scan report, not the event stream.

## 4. How the Two Channels Interact

The event channel and commit channel operate in parallel on different threads:

```text
scan loop (single-threaded)
    |
    +-- EventOutput::emit_core()      --> CoordinationEventSink --> recorder
    |                                       (best-effort telemetry)
    |
    +-- CommitSink::finish_item()     --> ReceiptCommitSink --> CommitPipeline
                                            (durable persistence)
```

Both channels receive the same findings, but they serve different consumers. The event channel provides immediate visibility for operators. The commit channel provides durable, identity-bearing records for the persistence layer. Neither depends on the other: a recorder outage does not affect durability, and a pipeline stall does not affect event streaming (though it does pause scanning through back-pressure).

## What's Next

[Chapter 4](04-distributed-worker.md) examines the `run_worker` function that orchestrates the full claim-execute-complete cycle, showing how the sinks, pipeline, and aggregator are wired together for each shard lease.
