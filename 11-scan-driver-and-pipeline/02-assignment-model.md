# The Work Unit -- Assignments and Connector Kinds

*A distributed scheduler dispatches shard `git-0xb7` to Worker 3. The assignment says "scan this git repository at `/data/repos/acme-corp`." Worker 3 receives the message, but the assignment carries no connector kind field -- just a path string. The worker guesses "filesystem" based on the path format and launches a directory walker. The walker enumerates 47,000 files in the working tree, finds secrets in `.env` and `config.yaml`, and reports completion. The coordinator marks the shard done. But the shard was supposed to scan git history -- all 2,400 commits across 18 months. The 47,000 files are only the current snapshot. Every secret ever committed and later rotated -- `STRIPE_SECRET_KEY=sk_live_4eC39HqLyjWDarjtT1` in commit `a7f3e2d`, removed in commit `c91b4a0` -- is invisible. Shard `git-0xb7` is marked complete with zero historical coverage. The operator discovers the gap three weeks later during an audit. Without an explicit assignment model that names the connector kind, the source payload, and the coordination context, the system cannot distinguish between a filesystem scan and a git history scan of the same directory.*

---

The `Assignment` struct is the universal work unit in the scan-driver boundary. It carries everything a `ScanSourceFactory` needs to construct the correct driver: what kind of source to scan, where the source lives, which shard range the work covers, and where to resume from if the scan was previously interrupted. This chapter walks through each type in the assignment model, from the connector kind tag through the complete assignment struct.

## 1. ConnectorKind -- Naming the Source Family

The first question any factory must answer is: "what kind of source is this?" The `ConnectorKind` enum provides the answer. From `lib.rs`:

```rust
/// Connector/source family for an assignment.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ConnectorKind {
    Filesystem,
    Git,
    InMemory,
}
```

Three variants, each naming a source family:

**`Filesystem`.** The driver will walk a directory tree (or scan a single file). Source items are regular files. The driver enumerates them using filesystem APIs, reads their contents into buffers, and feeds each buffer to the scanner engine.

**`Git`.** The driver will iterate git objects -- commits, trees, and blobs -- through the repository's history. Source items are blobs extracted from commit diffs. The driver uses `libgit2` or a custom git walker to traverse the commit graph, diff adjacent commits, and feed changed blobs to the engine.

**`InMemory`.** A synthetic source that lives entirely in memory. Test harnesses use this to provide pre-constructed datasets without touching the filesystem or initializing a git repository. The `InMemory` variant exists exclusively for deterministic simulation testing.

The enum is `Copy` because it is a tag with no associated data -- three discriminant values, each fitting in a single byte. The factory uses it as a dispatch key to select the correct driver implementation. The runtime in `gossip-scanner-runtime` maps each variant to a concrete factory:

```rust
fn driver_for_assignment(
    assignment: &Assignment,
) -> Result<Box<dyn gossip_scan_driver::ScanDriver>, ScanRuntimeError> {
    let factory: &dyn ScanSourceFactory = match assignment.connector_kind {
        ConnectorKind::Filesystem => &FilesystemScanSourceFactory,
        ConnectorKind::Git => &GitScanSourceFactory,
        ConnectorKind::InMemory => {
            return Err(ScanRuntimeError::UnsupportedConnectorKind(
                assignment.connector_kind,
            ));
        }
    };

    factory
        .driver_for_assignment(assignment)
        .map_err(ScanRuntimeError::Driver)
}
```

The `InMemory` variant is explicitly unsupported in the production runtime. The function returns `ScanRuntimeError::UnsupportedConnectorKind` rather than constructing a driver. This is not a limitation -- it is a design choice. The in-memory backend exists for deterministic simulation testing, not for production scans. The runtime rejects it early with a typed error rather than silently constructing a driver that produces meaningless results.

## 2. AssignmentSource -- The Source-Specific Payload

`ConnectorKind` names the family. `AssignmentSource` carries the data specific to that family. From `lib.rs`:

```rust
/// Source-specific assignment payload.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum AssignmentSource {
    Filesystem { root: PathBuf },
    Git { repo_root: PathBuf },
    InMemory { dataset_id: String },
}
```

Each variant carries exactly the data its driver needs:

**`Filesystem { root: PathBuf }`.** The path to scan. This is either a directory (the walker enumerates all files recursively) or a single file (the scanner processes one item). The path is canonicalized by the runtime before being placed into the assignment -- symlinks are resolved, relative paths are made absolute, and the resulting path is guaranteed to exist at the time of construction. Canonicalization prevents a class of TOCTOU bugs where the path changes between validation and scanning.

**`Git { repo_root: PathBuf }`.** The repository root. The git driver uses this to open the repository, enumerate commits, and iterate blobs. The runtime validates that this path is the actual repository root (not a subdirectory) using `git rev-parse --show-toplevel`. This validation prevents a subtle failure mode where scanning a subdirectory misses commits that touch files outside that subtree, producing incomplete coverage without any error signal.

**`InMemory { dataset_id: String }`.** A string identifier for a synthetic dataset. Test harnesses use this to look up pre-constructed data in a registry. The string can encode any information the test needs: dataset size, finding density, error injection parameters.

The `ConnectorKind` and `AssignmentSource` are intentionally separate types rather than merged into a single enum. This separation serves two purposes. First, it allows the runtime to dispatch on `ConnectorKind` (a `Copy` tag that fits in a register) without pattern-matching into the payload (which may involve heap-allocated `PathBuf` or `String` values). Second, it makes the type system more explicit: a factory that handles `ConnectorKind::Filesystem` expects `AssignmentSource::Filesystem` in the assignment, and a mismatch is a programming error in the runtime, not an ambiguity in the wire protocol.

## 3. The Assignment Struct

The `Assignment` struct is the complete work unit. From `lib.rs`:

```rust
/// Work assignment translated by a [`ScanSourceFactory`] into a driver.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Assignment {
    pub job_id: String,
    pub connector_kind: ConnectorKind,
    pub connector_instance_id: String,
    pub policy_hash: PolicyHash,
    pub shard_spec: ShardSpec,
    pub cursor: Cursor,
    pub source: AssignmentSource,
}
```

Seven fields. Let us examine each field group.

**`job_id: String`.** A human-readable identifier for the scan job. In CLI mode, this is a synthetic ID derived from the connector instance ID (e.g., `"runtime-/data/repos/acme"`). In distributed mode, the coordinator assigns a job ID that correlates all shards belonging to the same scan operation. The job ID is not used for identity derivation -- it is a telemetry and debugging aid. If two operators both scan the same directory, they get different job IDs, which allows log correlation per invocation.

**`connector_kind: ConnectorKind` and `connector_instance_id: String`.** The kind identifies the source family. The instance ID identifies the specific source within that family. For a filesystem scan, the instance ID is typically the canonical path string (e.g., `"/data/repos/acme-corp"`). For a git scan, it is the repository root path. The pair `(connector_kind, connector_instance_id)` forms a natural key for the source being scanned. Two assignments with the same pair target the same source; their shard specs and cursors determine which portion of that source each assignment covers.

**`policy_hash: PolicyHash`.** Recall from the B1 Identity section that the `PolicyHash` represents the hash of the detection policy (rule set and configuration) in effect at scan time. This value flows into the identity chain: different policies scanning the same source produce different `finding_id` values. The assignment carries the policy hash so that downstream identity derivation (in the `DurableCommitSink`, covered in [Section 14, Chapter 3](../14-scanner-runtime-and-worker/03-event-and-commit-sinks.md)) can include it in the finding identity computation. In CLI mode, the policy hash is a placeholder constant (`[0x52; 32]`) because CLI scans do not track identity chains.

**`shard_spec: ShardSpec`.** The shard specification from the B2 Coordination section. A `ShardSpec` defines the key range `[start, end)` that this assignment covers. For a full scan (no sharding), the range is `([], [])` -- the empty byte slice for both bounds, representing the entire keyspace. For a sharded scan, the range restricts the driver to a subset of the source's items. The driver is expected to skip items whose keys fall outside the shard spec's range.

**`cursor: Cursor`.** The resume position. `Cursor::initial()` means "start from the beginning." A non-initial cursor means "resume from where a previous scan left off." The cursor is opaque bytes from the runtime's perspective; its semantics depend on the source kind. A filesystem cursor might encode the last-processed filename in lexicographic order. A git cursor might encode the last-processed commit OID.

**`source: AssignmentSource`.** The source-specific payload described in section 2.

## 4. How Assignments Are Built

The runtime constructs assignments through a helper function. From `gossip-scanner-runtime/src/lib.rs`:

```rust
pub(crate) fn build_assignment(
    connector_kind: ConnectorKind,
    connector_instance_id: String,
    source: AssignmentSource,
) -> Assignment {
    Assignment {
        job_id: format!("runtime-{connector_instance_id}"),
        connector_kind,
        connector_instance_id,
        policy_hash: PolicyHash::from_bytes([0x52; 32]),
        shard_spec: ShardSpec::with_range([], []),
        cursor: Cursor::initial(),
        source,
    }
}
```

For CLI and standalone runtime scans, the function produces:

- A synthetic job ID prefixed with `runtime-` followed by the connector instance ID.
- A placeholder policy hash (`[0x52; 32]` -- all bytes set to `0x52`, the ASCII code for `R`).
- An unbounded shard spec (`ShardSpec::with_range([], [])` -- the entire keyspace).
- An initial cursor (`Cursor::initial()` -- start from the beginning).

CLI mode does not need real coordination metadata because there is no coordinator. The assignment exists only to satisfy the `ScanSourceFactory` interface, which is the point: both CLI and distributed modes flow through the same factory, and the factory expects an `Assignment` regardless of the calling context.

In distributed mode, the `ShardLease` (covered in [Section 14, Chapter 4](../14-scanner-runtime-and-worker/04-distributed-worker.md)) carries a fully populated `Assignment` with a real shard spec, cursor, and policy hash assigned by the coordinator. The coordinator constructs the assignment as part of lease creation, populating every field with production values.

The filesystem scan path shows how the assignment is wired into the pipeline. From `gossip-scanner-runtime/src/lib.rs`:

```rust
pub(crate) fn scan_fs_with_runtime(
    config: &FsScanConfig,
    out: &dyn EventOutput,
    commit: &dyn CommitSink,
    cancel: &CancellationToken,
) -> Result<AssignmentOutcome, ScanRuntimeError> {
    let canonical_path = validate_fs_path(&config.path)?;
    let assignment = build_assignment(
        ConnectorKind::Filesystem,
        canonical_path.display().to_string(),
        AssignmentSource::Filesystem {
            root: canonical_path,
        },
    );
    let mut runtime = config
        .budgets
        .to_execution_config_with_workers(config.workers)?;
    runtime.filesystem.skip_archives = config.skip_archives;
    runtime.filesystem.skip_binary = !config.scan_binary;
    runtime.filesystem.emit_findings_to_commit_sink = config.persist_findings;
    let engine = RuntimeEngineConfig {
        anchor_mode: config.anchor_mode,
        decode_depth: config.decode_depth,
        rules_file: config.rules_file.clone(),
        transform_filter: config.transform_filter.clone(),
    };
    execute_assignment_with_config(&assignment, runtime, &engine, out, None, commit, cancel)
}
```

The path is validated and canonicalized first (`validate_fs_path` checks existence, file type, and resolves symlinks). Then an `Assignment` is built with the canonical path as both the connector instance ID and the filesystem root. Then the execution configuration is assembled from the user's budget and driver knobs. Finally, `execute_assignment_with_config` dispatches through the factory and calls `ScanDriver::run()`. Every scan -- filesystem or git, CLI or distributed -- passes through this same function.

## 5. The CursorUpdate -- Checkpoint Metadata

When a driver completes (or is asked for a mid-scan checkpoint), it may return a `CursorUpdate`. From `lib.rs`:

```rust
/// Source-provided checkpoint hint in assignment keyspace order.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct CursorUpdate {
    pub cursor: Cursor,
    pub committed_items: u64,
}
```

**`cursor: Cursor`.** The new resume position. If the scan is interrupted and restarted with this cursor, the driver will skip items already processed. The cursor is in "assignment keyspace order" -- the same ordering used by the shard spec's range boundaries. This ordering guarantees that a cursor produced by scanning shard `[A, M)` will not accidentally skip items in shard `[M, Z)`.

**`committed_items: u64`.** The count of items committed up to this cursor position. The coordinator uses this to track scan progress across checkpoint boundaries and to estimate completion time by comparing committed items against the total expected item count.

The `CursorUpdate` bridges the scan-driver boundary and the B2 coordination checkpoint protocol. When the distributed runtime completes a shard, it passes the `CursorUpdate` to `coordinator.complete_shard()`. The coordinator persists the cursor so that a future scan of the same shard can resume rather than re-scan from the beginning. For CLI scans, the cursor update is typically `None` (no coordination) or discarded after logging.

## 6. The AssignmentOutcome

The runtime wraps the driver's output into an `AssignmentOutcome`. From `gossip-scanner-runtime/src/lib.rs`:

```rust
/// Execution outcome for one assignment run.
#[derive(Clone, Debug)]
pub struct AssignmentOutcome {
    /// Scan-driver report for the assignment.
    pub report: ScanReport,
    /// Driver-provided checkpoint hint to hand back to coordinators.
    pub checkpoint_hint: Option<CursorUpdate>,
    /// Optional driver-specific debug output (e.g. git pack-decode stats).
    pub debug_output: Option<String>,
}
```

The outcome bundles the `ScanReport` (aggregate counters from the driver) with the optional `CursorUpdate` (resume position for the coordinator). The `Option` on the checkpoint hint is necessary because not all drivers support cursor-based resumption. The `SourceCapabilities` struct (covered in [Chapter 3](03-driver-and-factory-traits.md)) declares whether a driver produces meaningful checkpoint hints. When `supports_checkpoint_hints` is `false`, the runtime does not pass the checkpoint to the coordinator.

The `execute_assignment_with_config` function shows how the outcome is assembled:

```rust
fn execute_assignment_with_config(
    assignment: &Assignment,
    config: ScanExecutionConfig,
    engine_config: &RuntimeEngineConfig,
    out: &dyn EventOutput,
    git_out: Option<&dyn GitEventOutput>,
    commit: &dyn CommitSink,
    cancel: &CancellationToken,
) -> Result<AssignmentOutcome, ScanRuntimeError> {
    let mut driver = driver_for_assignment(assignment)?;
    let report = driver
        .run(runtime_engine(engine_config)?, &config, out, git_out, commit, cancel)
        .map_err(ScanRuntimeError::Driver)?;

    Ok(AssignmentOutcome {
        report,
        checkpoint_hint: driver.checkpoint_hint(),
        debug_output: driver.debug_output(),
    })
}
```

The function calls `driver_for_assignment` to obtain a driver from the factory, calls `run` with the engine and sinks, then retrieves the checkpoint hint. The driver is `&mut self` during `run` (it may accumulate internal state such as cursor position and item counts), and the checkpoint hint is queried after `run` completes. This ordering is intentional: the checkpoint represents the driver's final position after all items have been processed. A checkpoint queried during `run` would represent an intermediate position, which is useful for mid-scan checkpoints but not for the final result.

## What's Next

[Chapter 3](03-driver-and-factory-traits.md) examines the `ScanDriver` and `ScanSourceFactory` traits in detail, including the `SourceCapabilities` flags that tell the orchestration layer what each backend supports and the design rationale for trait-based dispatch over enum-based dispatch.
