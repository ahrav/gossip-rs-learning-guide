# The Orchestration Pipeline -- Runtime Architecture

*An operator deploys a new scanner version. The first scan targets `/data/corpus`, a 2 TB filesystem with 4.7 million files. The operator launches the scan with `--workers=16` and `--max-items=256`. The runtime validates the path, discovers it exists and is a directory, canonicalizes it to `/data/corpus`, builds an `FsScanConfig` with the validated path and `ScanBudgets` set to 256 items, loads the default rule set, builds the scanner engine, and dispatches through the `ordered_content` module. The scan completes in 14 minutes with 12,847 findings. The next day, the operator runs the same scan but accidentally passes `--max-items=0`. The runtime rejects the configuration before dispatching: `ConnectorInputError::ZeroBudget { field: "max_items" }`. Zero items per checkpoint means zero progress -- the scan would run forever without advancing the cursor. A third operator on a different team passes `--max-bytes=0`. Same rejection. The budget validation catches configuration errors that would have silently produced an infinite-checkpoint loop or a zero-progress scan. Without early validation in the runtime, the error would surface deep in the dispatch as undefined behavior -- or worse, not surface at all.*

---

The `gossip-scanner-runtime` crate is the orchestration layer. It sits above the connector and scanner-engine crates and below the binary entrypoints (the CLI scanner and the worker binary). Its job is to translate high-level configuration into the precise types the scanning boundary expects: validated paths, configured engines, and wired sinks. The runtime contains no scanning logic itself — every substantive operation is delegated to the `ordered_content` or `git_repo` dispatch modules. What the runtime adds is configuration validation, engine caching, sink construction, and error wrapping.

This chapter maps the end-to-end pipeline from configuration to scan completion, examining each stage in detail.

## 1. The Pipeline Overview

The runtime pipeline has five stages:

```mermaid
flowchart LR
    A[Config] --> B[Validate & Canonicalize]
    B --> C[Build Assignment]
    C --> D[Build Engine]
    D --> E[Runtime Dispatch]
    E --> F[AssignmentOutcome]
```

1. **Config**: The caller provides a `FsScanConfig` or `GitScanConfig` with paths, budgets, worker counts, and engine options.
2. **Validate & Canonicalize**: The runtime checks that paths exist, resolves symlinks, and (for git) verifies that the path is a repository root.
3. **Build Assignment**: The runtime constructs an `Assignment` with the validated path, connector kind, and default coordination metadata.
4. **Build Engine**: The runtime loads rules, configures transforms, applies tuning, and constructs or retrieves the cached scanner engine.
5. **Runtime Dispatch**: The runtime selects the appropriate driver for the assignment and executes the scan.

Each stage is a potential failure point, and each failure is reported through a structured error type. The runtime never lets a configuration error propagate to the driver level -- every validation happens at the top of the pipeline, before any resources are allocated or any scanning begins.

## 2. ExecutionMode -- Two Distinct Paths

The runtime defines two execution modes that route to genuinely different code paths:

```rust
/// How the runtime acquires source items.
///
/// `Direct` dispatches via the local scan implementation. `Connector`
/// selects the source-family boundary: filesystem scans run one ordered
/// connector page acquisition/validation step, while Git connector mode
/// uses the direct path.
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub enum ExecutionMode {
    /// Scan source items directly from local state.
    #[default]
    Direct,
    /// Scan through the connector/runtime family path.
    Connector,
}
```

The top-level filesystem dispatcher routes based on the mode:

```rust
/// Top-level filesystem scan dispatcher.
///
/// Routes to `scan_fs_direct` or `scan_fs_connector` based on
/// [`FsScanConfig::execution_mode`]. Direct mode runs the local scan;
/// connector mode exercises the ordered-content page-fill boundary.
pub fn scan_fs(config: &FsScanConfig) -> Result<ScanReport, ScanRuntimeError> {
    match config.execution_mode {
        ExecutionMode::Direct => scan_fs_direct(config),
        ExecutionMode::Connector => scan_fs_connector(config),
    }
}
```

**Direct mode** uses null sinks and lightweight execution. No connector is constructed, no budgets are validated, no page acquisition occurs. The scan runs through the scheduler-driven local path:

```rust
/// Filesystem scan using null sinks (no event or commit output).
///
/// Suitable for headless / batch scans where only the aggregate report matters.
pub fn scan_fs_direct(config: &FsScanConfig) -> Result<ScanReport, ScanRuntimeError> {
    let out = NullEventOutput;
    let commit = commit_sink::CliNoOpCommitSink;
    let cancel = CancellationToken::new();
    scan_fs_with_runtime(config, &out, &commit, &cancel).map(|outcome| outcome.report)
}
```

**Connector mode** validates the path, constructs a `FilesystemConnector`, creates `Budgets`, builds a `RestoredShardState`, and runs through `OrderedContentRuntime`. This exercises the full ordered-content page-fill boundary -- the same path that a distributed worker would use:

```rust
/// Connector-mode filesystem scan (single-page validation boundary).
///
/// Validates the target path, constructs a [`FilesystemConnector`], and
/// executes one ordered page acquisition through
/// [`ordered_content::OrderedContentRuntime`]. Content reads, rule execution,
/// and durability are handled by the direct filesystem runtime path.
pub fn scan_fs_connector(config: &FsScanConfig) -> Result<ScanReport, ScanRuntimeError> {
    let canonical_path = validate_fs_path(&config.path)?;
    let budgets = Budgets::try_new(config.budgets.max_items, config.budgets.max_bytes, None)?;
    let state = RestoredShardState::new(
        ShardSpec::unbounded(),
        Cursor::initial(),
        CursorSemantics::Completed,
    );
    let runtime_input = ordered_content::OrderedContentRuntimeInput::new(state, budgets);
    let mut source = FilesystemConnector::new(canonical_path);

    match ordered_content::OrderedContentRuntime::execute_source(&mut source, &runtime_input)? {
        ordered_content::OrderedContentExecutionOutcome::Finished => Ok(ScanReport::default()),
        ordered_content::OrderedContentExecutionOutcome::Page(page) => {
            if page.page().state().next_cursor().is_some() {
                let items = page.report().items_scanned;
                tracing::warn!(
                    items_scanned = items,
                    "connector page indicates more items available; \
                     scan result is partial"
                );
            }
            Ok(page.report())
        }
        ordered_content::OrderedContentExecutionOutcome::Stopped(stop) => {
            Err(ScanRuntimeError::Driver(anyhow::anyhow!("{stop}")))
        }
    }
}
```

The connector-mode path constructs an unbounded `ShardSpec` and an initial `Cursor`, then runs a single page acquisition. If the page indicates more items are available (via a non-`None` next cursor), a warning is logged -- the CLI entry point processes only one page, so the scan result may be partial for large directory trees.

The Git dispatcher follows a different pattern: `scan_git_connector` delegates directly to `scan_git_direct`, because the Git source family does not yet use the ordered-content page-fill boundary.

The `ExecutionMode` is parsed from strings with standard case-insensitive matching:

```rust
impl std::str::FromStr for ExecutionMode {
    type Err = ParseExecutionModeError;

    fn from_str(value: &str) -> Result<Self, Self::Err> {
        match value.trim().to_ascii_lowercase().as_str() {
            "direct" => Ok(Self::Direct),
            "connector" => Ok(Self::Connector),
            _ => Err(ParseExecutionModeError {
                raw: value.to_owned(),
            }),
        }
    }
}
```

The `.trim()` call handles whitespace-padded input from configuration files. The `.to_ascii_lowercase()` call handles case variations. The error variant captures the raw input string for diagnostic display.

## 3. ScanBudgets -- Budget Validation

Scan budgets control checkpoint frequency and byte limits. These are the operational knobs that the runtime enforces before any driver is constructed:

```rust
/// Runtime budgets for source scans.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ScanBudgets {
    /// Maximum items processed between checkpoints.
    pub max_items: usize,
    /// Total bytes the executor may read from the source per page pass.
    pub max_bytes: u64,
}

impl Default for ScanBudgets {
    fn default() -> Self {
        Self {
            max_items: 4_096,
            max_bytes: 64 * 1024 * 1024, // 64 MiB
        }
    }
}
```

**`max_items: usize`.** The checkpoint frequency in items. After processing this many items, the orchestration layer can query `checkpoint_hint()` and persist a cursor update to the coordinator. The default is 4,096 items -- frequent enough to provide reasonable resumption granularity, but infrequent enough to avoid checkpoint overhead dominating scan time. The `ScanBudgets::validate()` method enforces that this value is non-zero before any scan dispatch begins.

**`max_bytes: u64`.** Total bytes the executor may read from the source per page pass. The default is 64 MiB. At 1-2 GB/s engine throughput, this yields approximately 32-64 ms of scan work per page. Memory pressure stays bounded because only one 256 KiB scan buffer is live at a time; the byte budget controls how many bytes are read from the source, not how much RAM is resident. The runtime enforces that this value is non-zero.

The budget validation rejects zero values for both fields:

```rust
impl ScanBudgets {
    pub fn validate(self) -> Result<(), ScanRuntimeError> {
        if self.max_items == 0 {
            return Err(ScanRuntimeError::ConnectorInput(
                ConnectorInputError::ZeroBudget { field: "max_items" },
            ));
        }
        if self.max_bytes == 0 {
            return Err(ScanRuntimeError::ConnectorInput(
                ConnectorInputError::ZeroBudget { field: "max_bytes" },
            ));
        }
        Ok(())
    }
}
```

The validation occurs before any assignment is built or scan is dispatched, following the fail-fast principle: configuration errors are caught at the top of the pipeline, not deep inside the dispatch where the error context is lost. The `ConnectorInputError::ZeroBudget` variant carries the field name (`"max_items"` or `"max_bytes"`), making the error message unambiguous.

## 4. FsScanConfig -- Filesystem Configuration

The filesystem scan configuration is the most feature-rich config type. It uses a builder pattern for ergonomic construction:

```rust
/// Filesystem scan config.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct FsScanConfig {
    /// Filesystem root or file path to scan.
    pub path: PathBuf,
    /// Number of worker threads to use.
    pub workers: usize,
    /// Optional transform decode depth override.
    pub decode_depth: Option<usize>,
    /// When true, archive expansion is disabled.
    pub skip_archives: bool,
    /// When true, binary files are scanned.
    pub scan_binary: bool,
    /// When true, findings are persisted via the commit sink bridge.
    pub persist_findings: bool,
    /// Anchor extraction policy for rule matching.
    pub anchor_mode: AnchorMode,
    /// Optional external rules file path.
    pub rules_file: Option<PathBuf>,
    /// Transform decoder filter.
    pub transform_filter: TransformFilter,
    /// Execution mode selector.
    pub execution_mode: ExecutionMode,
    /// Scan execution budget controls.
    pub budgets: ScanBudgets,
}
```

Each field maps to a specific runtime decision:

**`path: PathBuf`** is the scan target. The runtime validates and canonicalizes it before use. It may be a directory (the walker enumerates all files recursively) or a single file (the scanner processes one item).

**`workers: usize`** defaults to the number of available CPUs via `std::thread::available_parallelism()`. The builder method `with_workers` clamps the value to at least 1. This parallelism is used by the driver to spawn worker threads for concurrent file processing.

**`decode_depth: Option<usize>`** overrides the engine's maximum transform nesting depth. A `None` uses the engine default (3 levels). A `Some(1)` limits decoding to a single layer. Base64-inside-URL-encoding-inside-Base64 chains are bounded by this value to prevent exponential decode blowup on adversarial inputs.

**`skip_archives: bool` and `scan_binary: bool`** control content filtering. The defaults scan archives (they may contain secrets in compressed config files) but skip binary files (they produce a high rate of false positives). These map directly to the `FilesystemExecutionConfig` fields in the scan-driver boundary.

**`persist_findings: bool`** enables the commit sink bridge. In CLI mode, this is `false` -- findings go through the event output only. In distributed mode, the `ReceiptCommitSink` needs this set to `true` so the driver calls `begin_item`, `upsert_findings`, and `finish_item`.

**`anchor_mode: AnchorMode`** selects the anchor extraction policy. The `AnchorMode` enum has two variants:

```rust
/// Anchor extraction mode for rule planning.
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub enum AnchorMode {
    #[default]
    Manual,
    Derived,
}
```

`Manual` uses hand-curated anchor byte sequences defined in each rule spec. `Derived` uses anchors automatically extracted from the rule's regex pattern. The engine construction pipeline (covered in [Chapter 2](02-engine-construction.md)) maps these variants to `AnchorPolicy::ManualOnly` and `AnchorPolicy::DerivedOnly`.

**`rules_file: Option<PathBuf>`** and **`transform_filter: TransformFilter`** provide external rule and transform overrides. The transform filter is a three-variant enum:

```rust
/// Controls which transform decoders are enabled in the runtime engine.
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub enum TransformFilter {
    #[default]
    All,
    None,
    Only(Vec<TransformId>),
}
```

`All` enables every registered transform decoder. `None` disables all transforms (raw-only scanning). `Only(vec)` enables a specific subset. This filter is applied after the default transforms are constructed, not during construction -- the runtime builds the full transform set and then filters it, ensuring that the default set is always the starting point.

The constructor provides sensible defaults:

```rust
impl FsScanConfig {
    #[must_use]
    pub fn new(path: impl Into<PathBuf>) -> Self {
        Self {
            path: path.into(),
            workers: available_workers(),
            decode_depth: None,
            skip_archives: false,
            scan_binary: false,
            persist_findings: false,
            anchor_mode: AnchorMode::Manual,
            rules_file: None,
            transform_filter: TransformFilter::All,
            execution_mode: ExecutionMode::Direct,
            budgets: ScanBudgets::default(),
        }
    }
}
```

## 5. GitScanConfig -- Git Configuration

The git configuration carries a comparable set of fields to `FsScanConfig`, reflecting the fact that the git driver also needs engine tuning, content-level policy, and git-specific operational knobs:

```rust
/// Git scan config.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct GitScanConfig {
    /// Repository root path to scan.
    pub repo: PathBuf,
    /// Number of pack-exec worker threads.
    pub workers: usize,
    /// Optional transform decode depth override.
    pub decode_depth: Option<usize>,
    /// When true, binary blobs are scanned.
    pub scan_binary: bool,
    /// Git debug output level.
    pub debug_level: GitDebugLevel,
    /// When true, enrich commit metadata with identity dictionary IDs.
    pub enrich_identities: bool,
    /// Anchor extraction policy for rule matching.
    pub anchor_mode: AnchorMode,
    /// Optional external rules file path.
    pub rules_file: Option<PathBuf>,
    /// Transform decoder filter.
    pub transform_filter: TransformFilter,
    /// Stable repository identifier used in persistence keys.
    pub repo_id: u64,
    /// Git scan mode (diff-history vs ODB-blob fast path).
    pub scan_mode: GitScanMode,
    /// Merge-diff strategy for merge commits.
    pub merge_mode: MergeDiffMode,
    /// Ref-selection policy that determines which refs form the scan start set.
    pub ref_selection: GitRefSelection,
    /// Optional tree delta cache size override in MiB.
    pub tree_delta_cache_mb: Option<u32>,
    /// Optional engine chunk size override in MiB.
    pub engine_chunk_mb: Option<u32>,
    /// Execution mode selector.
    pub execution_mode: ExecutionMode,
    /// Scan execution budget controls.
    pub budgets: ScanBudgets,
}
```

Like `FsScanConfig`, `GitScanConfig` exposes `scan_binary`, `anchor_mode`, `rules_file`, `transform_filter`, and `decode_depth` -- the same engine-level knobs that control content policy and transform depth. On top of those shared knobs, `GitScanConfig` adds git-specific fields: `debug_level` controls git diagnostic output, `enrich_identities` enables identity dictionary enrichment on commit metadata, `repo_id` provides a stable key for persistence, `scan_mode` selects between diff-history and ODB-blob fast-path traversal, `merge_mode` chooses the merge-diff strategy, `ref_selection` determines which refs form the scan start set (the default branch, all branches, or an explicit ref list), and `tree_delta_cache_mb` / `engine_chunk_mb` tune memory usage for the git walker. The only filesystem-specific field absent from the git config is `skip_archives` (git blobs are not archive-expanded) and `persist_findings` (controlled at the distributed runtime level instead).

## 6. Path Validation

The runtime validates paths before constructing assignments. This validation is the first defense against configuration errors that would otherwise surface as cryptic driver failures.

For filesystem paths, the validation is straightforward:

```rust
fn validate_fs_path(path: &Path) -> Result<PathBuf, ScanRuntimeError> {
    if !path.exists() {
        return Err(ScanRuntimeError::InvalidPath {
            origin: "filesystem",
            path: path.to_path_buf(),
            message: "path does not exist".to_owned(),
        });
    }
    if !path.is_dir() && !path.is_file() {
        return Err(ScanRuntimeError::InvalidPath {
            origin: "filesystem",
            path: path.to_path_buf(),
            message: "path must be a directory or regular file".to_owned(),
        });
    }
    fs::canonicalize(path).map_err(|error| ScanRuntimeError::Io {
        op: "canonicalize",
        path: Some(path.to_path_buf()),
        error,
    })
}
```

Three checks: existence, file type (directory or regular file -- rejecting symlinks to non-existent targets, device nodes, named pipes), and canonicalization (resolving symlinks, converting to absolute path). The canonicalized path is returned for use in the assignment.

Git path validation is more rigorous. It must verify not just that a directory exists, but that it is the root of a git repository -- not a subdirectory inside one:

```rust
fn validate_git_repo_path(path: &Path) -> Result<PathBuf, ScanRuntimeError> {
    if !path.exists() {
        return Err(ScanRuntimeError::InvalidPath {
            origin: "git",
            path: path.to_path_buf(),
            message: "repository path does not exist".to_owned(),
        });
    }
    if !path.is_dir() {
        return Err(ScanRuntimeError::InvalidPath {
            origin: "git",
            path: path.to_path_buf(),
            message: "repository path must be a directory".to_owned(),
        });
    }

    let output = Command::new("git")
        .arg("-C")
        .arg(path)
        .args(["rev-parse", "--show-toplevel"])
        .output()
        .map_err(|error| ScanRuntimeError::Io {
            op: "git rev-parse",
            path: Some(path.to_path_buf()),
            error,
        })?;
    if !output.status.success() {
        return Err(ScanRuntimeError::GitCommandFailed {
            repo: path.to_path_buf(),
            status_code: output.status.code(),
            stderr: String::from_utf8_lossy(&output.stderr).into_owned(),
        });
    }

    let toplevel = PathBuf::from(std::str::from_utf8(&output.stdout).unwrap_or("").trim_end());
    let canonical_input = fs::canonicalize(path).map_err(|error| ScanRuntimeError::Io {
        op: "canonicalize",
        path: Some(path.to_path_buf()),
        error,
    })?;
    let canonical_toplevel = fs::canonicalize(&toplevel).map_err(|error| ScanRuntimeError::Io {
        op: "canonicalize",
        path: Some(toplevel.clone()),
        error,
    })?;

    if canonical_input != canonical_toplevel {
        return Err(ScanRuntimeError::InvalidPath {
            origin: "git",
            path: path.to_path_buf(),
            message: format!(
                "path is inside a git repository but is not the repository root (root is '{}')",
                canonical_toplevel.display()
            ),
        });
    }

    Ok(canonical_input)
}
```

The function uses `git rev-parse --show-toplevel` to find the actual repository root, then canonicalizes both the input path and the discovered root, and compares them. If the user passes a subdirectory inside a repository (e.g., `/data/repos/acme/src/`), the function rejects it with a descriptive error that names the actual root. This prevents a subtle but dangerous failure mode: scanning a subdirectory as a git repository would miss commits that touch files outside that subdirectory, producing incomplete coverage without any error signal.

## 7. ScanRuntimeError -- Structured Error Reporting

The runtime defines a structured error enum that captures context for every failure mode:

```rust
/// Runtime wiring errors for scan execution.
///
/// Covers the full lifecycle from path validation through engine construction
/// and scan dispatch. Each variant carries enough context for a human-readable
/// error message without requiring access to the original inputs.
#[derive(Debug, thiserror::Error)]
pub enum ScanRuntimeError {
    /// A scan target path failed validation.
    #[error("{origin} path '{}' invalid: {message}", path.display())]
    InvalidPath {
        /// Which subsystem originated the path.
        origin: &'static str,
        /// The path that failed validation.
        path: PathBuf,
        /// Human-readable reason for the failure.
        message: String,
    },
    /// A `git` subprocess exited with a non-zero status.
    #[error("git command failed for '{}' (status={status_code:?}): {stderr}", repo.display())]
    GitCommandFailed {
        /// Repository path the command was invoked against.
        repo: PathBuf,
        /// Process exit code, if available.
        status_code: Option<i32>,
        /// Captured stderr output from the git process.
        stderr: String,
    },
    /// An I/O operation failed during runtime setup.
    #[error("{}", fmt_io_error(.op, .path.as_ref(), .error))]
    Io {
        /// Short description of the operation.
        op: &'static str,
        /// Associated file path, when applicable.
        path: Option<PathBuf>,
        /// Underlying I/O error.
        #[source]
        error: std::io::Error,
    },
    /// The external rules configuration file could not be loaded or parsed.
    #[error("{}", fmt_rules_config_error(.path.as_ref(), .message))]
    RulesConfig {
        /// Path to the rules file, if one was specified.
        path: Option<PathBuf>,
        /// Human-readable parse or load error.
        message: String,
    },
    /// A connector input parameter was invalid.
    #[error("{0}")]
    ConnectorInput(#[source] ConnectorInputError),
    /// The family runtime returned an execution error.
    #[error("runtime execution failed: {0}")]
    Driver(#[source] anyhow::Error),
}
```

`Display` and `Error` are derived automatically by `thiserror::Error`. The `#[error("...")]` attributes use helper functions (`fmt_io_error`, `fmt_rules_config_error`) for complex formatting.

Five structured variants plus one escape hatch:

**`InvalidPath`** includes the source kind (`"filesystem"` or `"git"`), the offending path, and a human-readable message. An operator seeing this error knows immediately which path failed and why.

**`GitCommandFailed`** includes the repo path, the git exit code (`Option<i32>` because the process may be killed by a signal), and the stderr output. This is the information an operator needs to diagnose git infrastructure problems.

**`Io`** includes the operation name (`"canonicalize"`, `"git rev-parse"`), an optional path, and the underlying `std::io::Error`. The operation name provides context that the raw I/O error lacks.

**`RulesConfig`** includes the optional rules file path and a message from the rule parser. This covers both missing files and parse errors within valid files.

**`ConnectorInput`** wraps `ConnectorInputError` (from `gossip-contracts`), which includes the `ZeroBudget` variant used by budget validation.

**`Driver`** wraps `anyhow::Error` as the escape hatch for driver-internal errors that do not fit the structured categories.

## 8. The Complete Wiring

The full execution path for a filesystem scan, assembling all the pieces:

```rust
pub(crate) fn scan_fs_with_runtime(
    config: &FsScanConfig,
    out: &dyn EventOutput,
    commit: &dyn CommitSink,
    cancel: &CancellationToken,
) -> Result<AssignmentOutcome, ScanRuntimeError> {
    let canonical_path = validate_fs_path(&config.path)?;
    ordered_content::scan_local_filesystem(config, canonical_path, out, commit, cancel)
}
```

This function is the internal dispatch point for filesystem scan execution in the runtime. CLI scans, distributed worker scans, and test harness scans all flow through `scan_fs_with_runtime` (or `scan_git_with_runtime` for git). The parameters differ (different sinks, different configs), but the dispatch pattern is identical: validate the path, then delegate to the appropriate module (`ordered_content` for filesystem, `git_repo` for git).

## 9. Additional Modules

The runtime crate contains several supporting modules beyond the core orchestration pipeline:

**`parity.rs` -- JSONL parity helpers for cross-scanner validation.** This module provides canonical finding comparison utilities for validating gossip-rs output against the reference scanner-rs implementation. It normalizes both finding shapes (`type="finding"` + `rule` from scanner-rs and `rule_name` from gossip-rs), requires matching `commit_meta` events for git findings with `commit_id`, and sorts output findings for deterministic comparison. The parity module enables regression testing that ensures gossip-rs and scanner-rs produce equivalent results for the same input.

**`cli_tests.rs` -- Integration tests for CLI-mode scan paths.** This module exercises the `scan_fs_direct`, `scan_fs_connector`, `scan_git_direct`, and `scan_git_connector` entry points with fixture data, verifying that configuration validation rejects invalid inputs and that both execution modes produce expected results.

**`commit_sink.rs` and `coordination_sink.rs`** implement the persistence-side sink contracts. These are covered in detail in [Chapter 3](03-event-and-commit-sinks.md).

**`event_sink.rs`** implements the telemetry-side output formats (JSONL, text, JSON, SARIF). Also covered in [Chapter 3](03-event-and-commit-sinks.md).

**`distributed.rs`** bridges the runtime to the coordination layer for distributed worker deployments.

## 10. Built-in Rules and Provenance

The `scanner-engine` crate provides two functions that the runtime uses for its compile-time rule fallback:

**`scanner_engine::builtin_rules()`** returns the full production rule set, parsed from `default_rules.yaml` which is embedded in the binary via `include_str!` at compile time. The parsed rules are cached in a `OnceLock` so the YAML parsing and `Box::leak` allocations happen at most once. The function panics if the embedded YAML is invalid or empty — this indicates a build-time packaging error, not a runtime user input problem.

**`scanner_engine::builtin_rules_hash64()`** returns a deterministic 64-bit fingerprint of the embedded rule YAML content. The hash uses a fixed-seed `ahash::RandomState` for reproducibility across builds. The runtime logs this hash at startup so operators can verify rule provenance: two workers with the same `rule_hash` are guaranteed to be running the same detection rules.

## What's Next

[Chapter 2](02-engine-construction.md) examines how the scanner engine is built: rule loading from files and built-in defaults, transform configuration with URL-percent and Base64 decoders, tuning parameters that bound worst-case behavior, and anchor policies that control the fast-path filter.
