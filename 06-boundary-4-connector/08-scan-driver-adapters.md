# "Bridging Two Worlds" -- Scan-Driver Adapters

*A scan orchestrator dispatches 500 filesystem assignments and 200 git
assignments across a worker pool. The filesystem assignments use
`parallel_scan_dir` from the scanner-scheduler crate, which spawns scoped
threads internally and communicates via channel-based event sinks. The git
assignments use `run_git_scan` from the scanner-git crate, which drives a
commit-graph walker with ref watermark tracking. Both scan engines produce
findings, progress events, and completion reports -- but through completely
different interfaces. Without an adapter layer, the orchestrator would need
connector-specific dispatch logic for every scan engine variant: different
thread models, different event formats, different cancellation protocols.
Adding a new connector type would require modifying the orchestrator itself.
The adapter layer eliminates this coupling: each connector provides a factory
that produces a uniform `ScanDriver`, and the orchestrator dispatches
assignments without knowing which engine runs behind the trait.*

---

## Why Scan-Driver Adapters Exist

The connector capability, split-point, and read methods from Chapters 1-3
define a per-item access model: query capabilities, suggest split points,
open and read content. This model is clean for contract definition, but
production scan engines operate at a higher level of abstraction. A
filesystem scanner spawns parallel workers across directory trees. A git
scanner walks commit graphs and ref hierarchies. These engines need more
than per-item method calls -- they need cancellation tokens, commit sinks
for finding persistence, and event output channels for progress reporting.

The `gossip_scan_driver` crate defines two traits for this higher-level model:

- **`ScanSourceFactory`** -- A factory that validates an `Assignment` and
  produces a boxed `ScanDriver` for that assignment's connector kind and
  source payload.
- **`ScanDriver`** -- The execution trait. Its `run` method receives the
  scanner engine, execution config, event output sink, commit sink, and
  cancellation token.

The adapter implementations live in
`crates/gossip-connectors/src/scan_driver.rs` (~1,200 lines). This module
bridges the gap between the connector contracts and the scan execution model.

---

## FilesystemScanSourceFactory

```rust
/// Factory for filesystem assignments.
#[derive(Clone, Copy, Debug, Default)]
pub struct FilesystemScanSourceFactory;

/// Build a filesystem driver from an assignment and preserve its shard bounds.
///
/// The bounds are stored on the driver even though `parallel_scan_dir` cannot
/// consume them yet, so assignment metadata is not lost as execution paths
/// evolve toward connector-backed filesystem enumeration.
fn filesystem_driver_from_assignment(assignment: &Assignment) -> Result<FsScanDriver> {
    if assignment.connector_kind != ConnectorKind::Filesystem {
        bail!(
            "assignment kind mismatch: expected filesystem, got {:?}",
            assignment.connector_kind
        );
    }

    let AssignmentSource::Filesystem { root } = &assignment.source else {
        bail!("filesystem assignment missing filesystem source payload");
    };

    Ok(FsScanDriver {
        root: root.clone(),
        checkpoint_hint: None,
        shard_start: assignment.shard_spec.key_range_start().into(),
        shard_end: assignment.shard_spec.key_range_end().into(),
    })
}

impl ScanSourceFactory for FilesystemScanSourceFactory {
    fn driver_for_assignment(&self, assignment: &Assignment) -> Result<Box<dyn ScanDriver>> {
        Ok(Box::new(filesystem_driver_from_assignment(assignment)?))
    }

    fn capabilities(&self) -> SourceCapabilities {
        SourceCapabilities {
            supports_checkpoint_hints: true,
            supports_cooperative_cancel: false,
        }
    }
}
```

The factory delegates construction to the standalone
`filesystem_driver_from_assignment` helper. This helper validates two
preconditions: the assignment's `ConnectorKind` must be `Filesystem`, and
the source payload must be the `Filesystem` variant carrying a `root` path.
It returns an `FsScanDriver` that will later call `parallel_scan_dir` from
the scanner-scheduler crate.

The `FsScanDriver` struct carries four fields:

```rust
struct FsScanDriver {
    root: PathBuf,
    checkpoint_hint: Option<CursorUpdate>,
    /// Inclusive assignment lower bound (`[]` means unbounded).
    shard_start: Box<[u8]>,
    /// Exclusive assignment upper bound (`[]` means unbounded).
    shard_end: Box<[u8]>,
}
```

The `shard_start` and `shard_end` fields capture the assignment's key-range
boundaries. These bounds are stored on the driver even though
`parallel_scan_dir` cannot consume them yet -- assignment metadata is
preserved so it is not lost as execution paths evolve toward
connector-backed filesystem enumeration. The `shard_bounds()` method
converts them to `Option` slices (empty vec maps to `None`).

**Capability declaration:** `supports_checkpoint_hints: true` because the
driver can track its scan position. `supports_cooperative_cancel: false`
because `parallel_scan_dir` does not accept a cancellation token and has no
mid-scan cancellation mechanism -- the driver only checks the token before
starting.

### FsScanDriver Execution

The `FsScanDriver::run` method uses `std::thread::scope` to spawn two
forwarding threads: one for events and one for commit messages. These bridge
the scanner-scheduler's channel-based `EventOutput` and `StoreProducer`
interfaces to the scan-driver's trait-based sinks.

```rust
impl ScanDriver for FsScanDriver {
    fn run(
        &mut self,
        engine: Arc<scanner_engine::Engine>,
        cfg: &ScanExecutionConfig,
        out: &dyn EventOutput,
        git_out: Option<&dyn GitEventOutput>,
        commit: &dyn CommitSink,
        cancel: &gossip_scan_driver::CancellationToken,
    ) -> Result<ScanReport> {
        std::thread::scope(|scope| -> Result<ScanReport> {
            let (event_tx, event_rx) = unbounded();
            let event_forwarder = scope.spawn(move || forward_events(out, None, event_rx));

            let (commit_tx, commit_rx) = unbounded();
            let commit_forwarder = scope.spawn(move || forward_commits(commit, commit_rx));

            // Configure and run the parallel scan...
            let report = parallel_scan_dir(&self.root, engine, scan_cfg)?;

            drop(event_tx);
            drop(commit_tx);

            join_scoped(event_forwarder, "event forwarder thread")?;
            join_scoped(commit_forwarder, "commit forwarder thread")??;

            Ok(ScanReport { /* ... */ })
        })
    }
}
```

The `ChannelStoreProducer` adapter deserves attention: it normalizes the
scanner-scheduler's absolute OS-encoded paths back to the connector's
relative `/`-separated key encoding. The `normalize_scheduler_path` function
strips the scan root prefix and re-encodes as `/`-separated bytes, matching
the filesystem connector's `encode_rel_path` contract. This ensures that
commit-sink keys align with connector cursors regardless of how the
scheduler internally represents paths.

---

## GitScanSourceFactory

```rust
/// Factory for git assignments.
#[derive(Clone, Copy, Debug, Default)]
pub struct GitScanSourceFactory;

impl ScanSourceFactory for GitScanSourceFactory {
    fn driver_for_assignment(&self, assignment: &Assignment) -> Result<Box<dyn ScanDriver>> {
        if assignment.connector_kind != ConnectorKind::Git {
            bail!(
                "assignment kind mismatch: expected git, got {:?}",
                assignment.connector_kind
            );
        }

        let AssignmentSource::Git { repo_root } = &assignment.source else {
            bail!("git assignment missing git source payload");
        };

        Ok(Box::new(GitScanDriver {
            repo_root: repo_root.clone(),
            debug_output: None,
        }))
    }

    fn capabilities(&self) -> SourceCapabilities {
        SourceCapabilities {
            supports_checkpoint_hints: false,
            supports_cooperative_cancel: false,
        }
    }
}
```

Git assignments carry a `repo_root` path. The factory produces a
`GitScanDriver` that calls `run_git_scan` from the scanner-git crate. The
driver's `debug_output` field is initially `None` and may be populated after
a scan with formatted debug/perf output (see `build_git_scan_config` and
`format_git_debug_output` below).

**Capabilities:** Both checkpoint hints and cooperative cancel are `false`.
Git scans operate on a commit-graph model (ref watermarks, seen-blob
deduplication) that does not map to the per-item cursor checkpoint model.

### GitScanDriver Execution

The `GitScanDriver::run` method uses a `NativeRefResolver` (configured with
`StartSetConfig::DefaultBranchOnly`) to resolve refs natively and two
separate no-op stores for full-scan semantics: an `EmptyWatermarkStore` to
treat every ref as having no prior watermark, and a `NeverSeenStore` to
treat every blob as unseen for dedup purposes. This combination forces a
complete scan on each run:

```rust
impl ScanDriver for GitScanDriver {
    fn run(
        &mut self,
        engine: Arc<scanner_engine::Engine>,
        cfg: &ScanExecutionConfig,
        out: &dyn EventOutput,
        git_out: Option<&dyn GitEventOutput>,
        _commit: &dyn CommitSink,
        cancel: &gossip_scan_driver::CancellationToken,
    ) -> Result<ScanReport> {
        self.debug_output = None;
        if cancel.is_cancelled() {
            return Ok(ScanReport::default());
        }

        std::thread::scope(|scope| -> Result<ScanReport> {
            let (event_tx, event_rx) = unbounded();
            let event_forwarder = scope.spawn(move || forward_events(out, git_out, event_rx));

            let git_sink: Arc<dyn GitEventSink> =
                Arc::new(ChannelGitEventOutput::new(event_tx.clone()));

            let git_cfg = build_git_scan_config(cfg)?;

            let resolver = NativeRefResolver::new(StartSetConfig::DefaultBranchOnly);
            let watermarks = EmptyWatermarkStore;
            let seen = NeverSeenStore;
            let result = run_git_scan(
                &self.repo_root, engine, &resolver, &seen, &watermarks,
                None, &git_cfg, git_sink,
            )?;

            self.debug_output = format_git_debug_output(&result.0, cfg.git.debug_level);
            drop(event_tx);
            join_scoped(event_forwarder, "event forwarder thread")?;

            Ok(git_report_to_scan_report(result, scan_elapsed))
        })
    }

    fn debug_output(&self) -> Option<String> {
        self.debug_output.clone()
    }
}
```

The commit sink is intentionally unused. The source comment explains: "Git
scans operate on a commit-graph model (ref watermarks, seen-blob
deduplication) that does not map to the per-item begin/finish lifecycle of
the commit sink."

### Runtime Config Mapping: `build_git_scan_config`

Before calling `run_git_scan`, the driver translates the runtime-level
`ScanExecutionConfig` into the low-level `GitScanConfig` expected by the
git scanner. The `build_git_scan_config` function (`scan_driver.rs:697-742`)
handles this mapping:

```rust
fn build_git_scan_config(cfg: &ScanExecutionConfig) -> Result<GitScanConfig> {
    const MIB: u32 = 1024 * 1024;

    fn mebibytes_to_u32_bytes(value_mb: u32, label: &str) -> Result<u32> {
        if value_mb == 0 {
            bail!("{label} must be >= 1 MiB");
        }
        value_mb
            .checked_mul(MIB)
            .ok_or_else(|| anyhow!("{label} exceeds supported size"))
    }

    let mut git_cfg = GitScanConfig {
        repo_id: cfg.git.repo_id,
        scan_mode: cfg.git.scan_mode,
        merge_diff_mode: cfg.git.merge_diff_mode,
        pack_exec_workers: cfg.git.pack_exec_workers
            .unwrap_or_else(|| cfg.workers.max(1)).max(1),
        enrich_identities: cfg.git.enrich_identities,
        ..GitScanConfig::default()
    };
    git_cfg.engine_adapter.scan_binary = cfg.git.scan_binary;

    if let Some(value_mb) = cfg.git.tree_delta_cache_mb {
        git_cfg.tree_diff.max_tree_delta_cache_bytes =
            mebibytes_to_u32_bytes(value_mb, "x-tree-delta-cache-mb")?;
    }
    if let Some(value_mb) = cfg.git.engine_chunk_mb {
        git_cfg.engine_adapter.chunk_bytes =
            mebibytes_to_usize_bytes(value_mb, "x-engine-chunk-mb")?;
    }

    Ok(git_cfg)
}
```

Runtime options use MiB units for CLI ergonomics; the function performs
checked MiB-to-byte conversion (rejecting zero values and overflow) and
applies sane worker fallbacks. Optional fields like `tree_delta_cache_mb`
and `engine_chunk_mb` are only applied when explicitly set, otherwise the
`GitScanConfig::default()` values are preserved.

### Debug Output

After a scan completes, the driver calls `format_git_debug_output` to
optionally produce formatted debug/perf text from the `GitScanReport`.
The result is stored in `self.debug_output` and exposed via the
`ScanDriver::debug_output()` trait method. The `GitDebugLevel` enum
controls verbosity -- `Off` produces `None`, while higher levels format
scan statistics for stderr output (useful for `--debug` / `--debug=perf`
CLI flags).

---

## InMemoryScanSourceFactory

```rust
/// Factory for in-memory dataset assignments used in tests/harnesses.
#[derive(Clone, Debug)]
pub struct InMemoryScanSourceFactory {
    dataset_id: String,
    items: Arc<[MemItem]>,
}

impl InMemoryScanSourceFactory {
    #[must_use]
    pub fn new(dataset_id: impl Into<String>, mut items: Vec<MemItem>) -> Self {
        items.sort_by(|left, right| left.key.cmp(&right.key));
        Self {
            dataset_id: dataset_id.into(),
            items: items.into(),
        }
    }
}
```

The in-memory factory sorts items at construction and stores them in an
`Arc<[MemItem]>` for cheap sharing across driver instances. The
`driver_for_assignment` method validates that the assignment's `dataset_id`
matches the factory's, preventing cross-dataset confusion in test harnesses.

**Capabilities:** Both `supports_checkpoint_hints` and
`supports_cooperative_cancel` are `true`. The in-memory driver is simple
enough to support both features.

### InMemoryScanDriver Execution

The `InMemoryScanDriver` iterates pre-loaded items sequentially, driving the
commit-sink lifecycle (`begin_item` / `finish_item`) and emitting checkpoint
hints at configured intervals:

```rust
impl ScanDriver for InMemoryScanDriver {
    fn run(
        &mut self,
        _engine: Arc<scanner_engine::Engine>,
        cfg: &ScanExecutionConfig,
        out: &dyn EventOutput,
        _git_out: Option<&dyn GitEventOutput>,
        commit: &dyn CommitSink,
        cancel: &gossip_scan_driver::CancellationToken,
    ) -> Result<ScanReport> {
        let mut report = ScanReport::default();
        let checkpoint_every = cfg.checkpoint_every_items.max(1);

        for item in self.items.iter() {
            if cancel.is_cancelled() {
                break;
            }

            commit.begin_item(&item.key, &meta)?;
            commit.finish_item(&item.key)?;

            report.items_scanned = report.items_scanned.saturating_add(1);

            if report.items_scanned % checkpoint_every == 0 {
                self.checkpoint_hint = Some(CursorUpdate {
                    cursor: Cursor::with_last_key(item.key.clone()),
                    committed_items: report.items_scanned,
                });
            }
        }

        Ok(report)
    }
}
```

The engine parameter is intentionally unused -- this driver exercises the
commit-sink lifecycle and checkpoint machinery without running the scanner
engine. Tests that need engine integration use the filesystem or git drivers.

---

## The Event Forwarding Pattern

All three drivers use the same pattern to bridge scan-engine events across
thread boundaries. Scanner events borrow strings and slices from the
emitting thread (`CoreEvent<'_>`), so they cannot outlive that thread. The
`OwnedCoreEvent` enum clones borrowed fields into owned storage so events
can be forwarded through a crossbeam channel:

```rust
fn forward_events(out: &dyn EventOutput, git_out: Option<&dyn GitEventOutput>, rx: Receiver<OwnedDriverEvent>) {
    while let Ok(event) = rx.recv() {
        match event {
            OwnedDriverEvent::Core(core) => core.emit_into(out),
            OwnedDriverEvent::Git(git) => {
                if let Some(sink) = git_out {
                    git.emit_into(sink);
                }
            }
        }
    }
    out.flush();
}
```

The commit forwarding path is more nuanced: `forward_commits` captures the
first error but continues draining the channel. Breaking out on first error
would leave messages in the channel, preventing sender threads from
completing and causing the scoped thread join to deadlock.

---

## How the Adapters Relate to the Connector Methods

The adapter layer does not replace the connector API -- it builds on
it. The connector capability and read methods define the
contract vocabulary: what capabilities exist, how split points are chosen,
what errors mean. The `ScanDriver` adapters consume this vocabulary while
providing a higher-level execution interface:

| Concern | Connector Methods | ScanDriver Adapters |
|---------|-----------------|---------------------|
| Planning | `caps` / `choose_split_point` | `capabilities` declaration |
| Content access | `open` / `read_range` per item | Engine handles reads internally |
| Error handling | `EnumerateError` / `ReadError` | `anyhow::Result` wrapping engine errors |
| Cancellation | Not modeled | `CancellationToken` parameter |
| Finding persistence | Not modeled | `CommitSink` parameter |
| Progress reporting | Not modeled | `EventOutput` parameter |

The connector methods remain the right abstraction for split-point queries
and per-item content access. The scan-driver adapters are the right
abstraction for production orchestration that dispatches heterogeneous
assignments across a worker pool.

---

## Summary

The scan-driver adapter module in `scan_driver.rs` bridges three concrete
connector types to the `ScanDriver` execution model.
`FilesystemScanSourceFactory` validates filesystem assignments and produces
drivers that call `parallel_scan_dir` with channel-based event and commit
forwarding. `GitScanSourceFactory` validates git assignments and produces
drivers that call `run_git_scan` with ref resolution and full-scan
semantics. `InMemoryScanSourceFactory` validates in-memory test assignments
and produces drivers that exercise the commit-sink lifecycle sequentially.
Each factory declares its capabilities (checkpoint hints, cooperative
cancel) so the orchestrator can adapt its dispatch strategy. The
`OwnedCoreEvent` and `forward_commits` patterns handle the lifetime and
error-handling challenges of cross-thread event forwarding. Together with
the connector API from earlier chapters, these adapters form the
complete interface between external data sources and the Gossip-rs scanning
pipeline.
