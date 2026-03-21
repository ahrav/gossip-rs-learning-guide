# The Engine Factory -- Rule Loading, Transforms, and Tuning

*A scanner fleet processes 11 PB of data per day across 2,000 worker instances. Each worker handles approximately 500 assignments per hour. If every assignment constructs a fresh scanner engine -- loading rules from disk, compiling regex patterns, configuring transform decoders, computing anchor tables -- the fleet spends 2,000 instances times 500 assignments times 15 milliseconds per engine build: 4.2 hours of cumulative CPU time per hour, wasted on redundant initialization. The rules do not change between assignments on the same worker. The transform configuration does not change. The tuning parameters do not change. The only thing that changes is the input data. Rebuilding the engine for every assignment is paying a 15-millisecond tax on every scan for configuration that is static for the lifetime of the process. And 15 milliseconds is optimistic -- a production rule set with 800 patterns and 12 transform decoders may take 50-80 milliseconds to compile. At 500 assignments per hour per worker, that is 11 hours of cumulative CPU time per hour across the fleet. Without engine caching, the fleet wastes measurable fractions of its capacity on configuration parsing that could happen once at process startup.*

---

The scanner engine is the most expensive object to construct in the runtime. It contains compiled detection rules (regex automata compiled to bytecode), transform decoder configurations (base64, URL-percent, with span limits and depth caps), tuning parameters (merge gaps, pressure limits, finding caps), and anchor extraction tables (fast-path byte sequences that avoid running regex against every byte of input). The runtime must build this object before any scan can execute. This chapter examines the engine construction pipeline, the caching strategy that amortizes construction cost across assignments, and each configuration dimension.

## 1. The RuntimeEngineConfig

Engine construction is controlled by an internal configuration struct. This struct is `pub(crate)` -- it is not part of the public API because callers should not construct engines directly. They should use `FsScanConfig` or `GitScanConfig`, which the runtime translates into engine configuration:

```rust
#[derive(Clone, Debug, Default, PartialEq, Eq)]
struct RuntimeEngineConfig {
    anchor_mode: AnchorMode,
    decode_depth: Option<usize>,
    rules_file: Option<PathBuf>,
    transform_filter: TransformFilter,
}
```

Four fields, each controlling one dimension of engine behavior:

**`anchor_mode: AnchorMode`.** Selects how the engine extracts anchors from input data. The `AnchorMode` enum was introduced in [Chapter 1](01-runtime-architecture.md). `Manual` uses hand-curated anchor byte sequences defined in each rule spec (the `anchors` field on `RuleSpec`). `Derived` uses anchors automatically extracted from the rule's regex pattern by the engine's planner. Manual anchors are more precise (the rule author chose them based on domain knowledge); derived anchors are more complete (the planner extracts every literal prefix from the regex, catching patterns the author may have missed).

**`decode_depth: Option<usize>`.** Overrides the maximum depth for nested transform decoding. A `None` value uses the engine's default depth (3 levels). A `Some(1)` limits decoding to a single layer -- useful for scans where nested encoding is rare and decode overhead is undesirable. A `Some(0)` effectively disables transform decoding.

**`rules_file: Option<PathBuf>`.** Path to an external YAML rules file. When `None`, the runtime uses built-in default rules. When `Some`, the runtime reads and parses the file, replacing the default rules entirely.

**`transform_filter: TransformFilter`.** Controls which transform decoders are enabled, as described in [Chapter 1](01-runtime-architecture.md).

The struct derives `PartialEq` and `Eq`. This is critical for the caching strategy: the runtime compares each engine config against the default to decide whether to use the cached engine or build a fresh one. The comparison must be structural equality, not identity equality, so `PartialEq` is the right trait.

## 2. The OnceLock Caching Strategy

The runtime caches the default engine using `OnceLock`, a synchronization primitive that allows exactly-once initialization of a `static` value:

```rust
fn runtime_engine(
    config: &RuntimeEngineConfig,
) -> Result<Arc<scanner_engine::Engine>, ScanRuntimeError> {
    if config == &RuntimeEngineConfig::default() {
        static ENGINE: OnceLock<Arc<scanner_engine::Engine>> = OnceLock::new();
        let engine = ENGINE.get_or_init(|| {
            Arc::new(
                build_runtime_engine(&RuntimeEngineConfig::default())
                    .expect("default runtime engine must build"),
            )
        });
        Ok(Arc::clone(engine))
    } else {
        Ok(Arc::new(build_runtime_engine(config)?))
    }
}
```

The caching logic has two paths:

**Default config path.** When the engine config equals `RuntimeEngineConfig::default()` (manual anchors, no decode depth override, no custom rules file, all transforms enabled), the engine is built once and stored in a `static OnceLock`. The `OnceLock::get_or_init` method ensures that the initialization closure runs exactly once, even if multiple threads call `runtime_engine` concurrently. Subsequent calls return `Arc::clone(engine)` -- a reference count increment (a single atomic add), not a rebuild. This is the common case for distributed workers that process hundreds of assignments with identical engine configuration.

**Custom config path.** When any field differs from the default (custom rules file, non-default anchor mode, transform filter set to `None` or `Only`, decode depth override), a fresh engine is constructed for each call. Custom engines are not cached because the configuration space is unbounded -- there are infinitely many possible rules file paths, and caching them would require a concurrent hash map keyed by the full config, adding complexity without clear benefit. Custom configs are typically used in CLI invocations where the scan is a one-shot operation.

The `OnceLock` is initialized with `.expect("default runtime engine must build")`. This panic is intentional and documented: if the default engine cannot be constructed (corrupted built-in rules, regex compilation failure in the default rule), the process cannot function. The built-in rules are compile-time constants with a compile-time-validated regex; the only scenario where this panic fires is a binary corruption or a platform that does not support the regex engine's compiled bytecode. Failing fast with a clear panic message is preferable to propagating an error that every caller must handle for a condition that represents a fundamentally broken binary.

The `Arc` wrapper allows the engine to be shared between the runtime and the scan dispatch without cloning the engine's internal data structures (compiled regex automata, anchor lookup tables, transform configurations). The runtime passes `Arc<scanner_engine::Engine>` to the scan module, so the dispatch holds a reference-counted handle. When the scan completes, the reference count decrements. The engine is freed only when the last reference is dropped -- which, for the cached default engine, is at process exit.

## 3. The Engine Build Pipeline

The `build_runtime_engine` function assembles the engine from four components:

```rust
fn build_runtime_engine(
    config: &RuntimeEngineConfig,
) -> Result<scanner_engine::Engine, ScanRuntimeError> {
    let rules = load_runtime_rules(config.rules_file.as_deref())?;
    let mut tuning = default_runtime_tuning();
    if let Some(depth) = config.decode_depth {
        tuning.max_transform_depth = depth;
    }
    let transforms = apply_transform_filter(default_runtime_transforms(), &config.transform_filter);
    let policy = match config.anchor_mode {
        AnchorMode::Manual => AnchorPolicy::ManualOnly,
        AnchorMode::Derived => AnchorPolicy::DerivedOnly,
    };
    Ok(scanner_engine::Engine::new_with_anchor_policy(
        rules, transforms, tuning, policy,
    ))
}
```

Four steps, each producing one input to the engine constructor:

1. **Load rules.** From an external file or from the built-in defaults.
2. **Configure tuning.** Start with the default tuning parameters, override the decode depth if specified.
3. **Filter transforms.** Start with the default transform set, apply the filter to retain only the desired decoders.
4. **Construct engine.** Pass all four components to `Engine::new_with_anchor_policy`.

Each step is examined in detail below.

## 4. Rule Loading

Rules are loaded through a three-tier resolution strategy that balances flexibility (operator-provided rules), operational convenience (adjacent defaults), and guaranteed availability (compiled-in fallback):

```rust
fn load_runtime_rules(
    rules_file: Option<&Path>,
) -> Result<Vec<scanner_engine::RuleSpec>, ScanRuntimeError> {
    // 1. Explicit override.
    if let Some(path) = rules_file {
        let content = scanner_engine::read_rules_text(path).map_err(|error| {
            ScanRuntimeError::RulesConfig {
                path: Some(path.to_path_buf()),
                message: error.to_string(),
            }
        })?;
        let rules = scanner_engine::load_rules_from_content(&content).map_err(|error| {
            ScanRuntimeError::RulesConfig {
                path: Some(path.to_path_buf()),
                message: error.to_string(),
            }
        })?;
        let hash = scanner_engine::rules_content_hash64(content.as_bytes());
        eprintln!(
            "info: using rules from {} ({} rules, rule_hash: {hash:016x})",
            path.display(),
            rules.len(),
        );
        return Ok(rules);
    }

    // 2. Default candidate adjacent to binary.
    if let Some(default_path) = scanner_engine::default_rules_path() {
        if default_path.exists() && !default_path.is_file() {
            eprintln!(
                "warn: {} exists but is not a regular file; falling back to built-in rules",
                default_path.display()
            );
        } else if default_path.is_file() {
            let content = scanner_engine::read_rules_text(&default_path).map_err(|error| {
                ScanRuntimeError::RulesConfig {
                    path: Some(default_path.clone()),
                    message: error.to_string(),
                }
            })?;
            let rules = scanner_engine::load_rules_from_content(&content).map_err(|error| {
                ScanRuntimeError::RulesConfig {
                    path: Some(default_path.clone()),
                    message: error.to_string(),
                }
            })?;
            let hash = scanner_engine::rules_content_hash64(content.as_bytes());
            eprintln!(
                "info: using {} ({} rules, source: default_rules.yaml, rule_hash: {hash:016x})",
                default_path.display(),
                rules.len(),
            );
            return Ok(rules);
        }
    }

    // 3. Compile-time embedded fallback.
    let rules = scanner_engine::builtin_rules();
    let hash = scanner_engine::builtin_rules_hash64();
    eprintln!("info: no default_rules.yaml next to binary; using compiled-in rules");
    eprintln!(
        "info: using compiled-in rule set ({} rules, source: built-in, rule_hash: {hash:016x})",
        rules.len(),
    );
    Ok(rules)
}
```

The three tiers, in priority order:

**Tier 1: Explicit `--rules=<path>` override.** When the caller specifies a rules file path (via `FsScanConfig::rules_file` or `GitScanConfig::rules_file`), the function reads it, parses it, computes a 64-bit provenance hash via `scanner_engine::rules_content_hash64`, and logs the rule count and hash to stderr. This tier is for operators who maintain custom detection rule sets.

**Tier 2: `default_rules.yaml` adjacent to the binary.** When no explicit path is provided, the function checks for a file named `default_rules.yaml` in the same directory as the running binary (via `scanner_engine::default_rules_path()`). If the path exists but is not a regular file (e.g., a directory or symlink to a directory), a warning is logged and the function falls through to tier 3. If the file exists and is a regular file, it is loaded and parsed identically to tier 1. This tier enables deployment-time rule updates without recompiling the binary — operators drop a `default_rules.yaml` next to the scanner binary and it picks it up automatically.

**Tier 3: Compile-time embedded fallback via `scanner_engine::builtin_rules()`.** When neither an explicit file nor an adjacent default is available, the function falls back to rules embedded in the binary at compile time. The `scanner_engine` crate embeds `default_rules.yaml` via `include_str!` and parses it on first call (cached via `OnceLock`). The corresponding `scanner_engine::builtin_rules_hash64()` returns the hash of the embedded YAML content. This tier guarantees that the scanner always has a functional rule set, even in minimal container deployments that contain only the binary.

Every tier logs the rule count and a deterministic `rule_hash` fingerprint to stderr. This provenance information appears in scanner startup logs and enables operators to verify which rules are active on a given worker instance — a critical debugging capability when investigating finding discrepancies across a fleet.

## 5. Transform Configuration

The default transforms enable two decoders for common encoding schemes:

```rust
fn default_runtime_transforms() -> Vec<TransformConfig> {
    vec![
        TransformConfig {
            id: TransformId::UrlPercent,
            mode: TransformMode::Always,
            gate: Gate::AnchorsInDecoded,
            min_len: 16,
            max_spans_per_buffer: 8,
            max_encoded_len: 64 * 1024,
            max_decoded_bytes: 64 * 1024,
            plus_to_space: false,
            base64_allow_space_ws: false,
        },
        TransformConfig {
            id: TransformId::Base64,
            mode: TransformMode::Always,
            gate: Gate::AnchorsInDecoded,
            min_len: 32,
            max_spans_per_buffer: 8,
            max_encoded_len: 64 * 1024,
            max_decoded_bytes: 64 * 1024,
            plus_to_space: false,
            base64_allow_space_ws: false,
        },
    ]
}
```

**URL percent decoding** (`UrlPercent`) handles `%XX`-encoded sequences found in URLs, query strings, and percent-encoded configuration values. Secrets embedded in URLs (e.g., `https://user:p%40ssword@host/`) are decoded before rule matching.

**Base64 decoding** (`Base64`) handles standard base64-encoded content. Configuration files, environment variables, and Kubernetes secrets often contain base64-encoded credentials.

Both transforms share key configuration parameters:

**`mode: TransformMode::Always`.** The transform is always attempted, not conditionally triggered by input characteristics.

**`gate: Gate::AnchorsInDecoded`.** The engine only applies the regex after decoding if anchor byte sequences appear in the decoded output. This prevents the regex from running against decoded content that contains no secrets -- a critical optimization for transforms that decode large spans.

**`min_len`** sets the minimum span length. URL spans shorter than 16 bytes are not worth decoding (they cannot contain a meaningful secret). Base64 spans shorter than 32 bytes are similarly too short. These thresholds reduce decode overhead without missing real secrets.

**`max_spans_per_buffer: 8`.** At most 8 encoded spans are decoded per input buffer. This bounds worst-case decode cost for inputs with many short encoded spans.

**`max_encoded_len: 64 * 1024` and `max_decoded_bytes: 64 * 1024`.** Both capped at 64 KB to prevent memory amplification. A malicious input with a 10 MB base64 span would not cause a 10 MB allocation.

The filter is applied post-construction:

```rust
fn apply_transform_filter(
    transforms: Vec<TransformConfig>,
    filter: &TransformFilter,
) -> Vec<TransformConfig> {
    match filter {
        TransformFilter::All => transforms,
        TransformFilter::None => Vec::new(),
        TransformFilter::Only(ids) => transforms
            .into_iter()
            .filter(|transform| ids.contains(&transform.id))
            .collect(),
    }
}
```

## 6. Tuning Parameters

The runtime configures engine tuning with production-appropriate defaults:

```rust
fn default_runtime_tuning() -> scanner_engine::Tuning {
    scanner_engine::Tuning {
        merge_gap: 64,
        max_windows_per_rule_variant: 16,
        pressure_gap_start: 128,
        max_anchor_hits_per_rule_variant: 2048,
        max_utf16_decoded_bytes_per_window: 64 * 1024,
        max_transform_depth: 3,
        max_total_decode_output_bytes: 512 * 1024,
        max_work_items: 256,
        max_findings_per_chunk: 8192,
        scan_utf16_variants: true,
    }
}
```

Each parameter bounds a specific dimension of engine behavior to prevent worst-case blowup:

**`merge_gap: 64`.** Adjacent anchor hits within 64 bytes are merged into a single scan window. Without merging, a file with `password` appearing every 50 bytes would create thousands of overlapping windows. With a 64-byte merge gap, adjacent windows collapse into larger contiguous windows, reducing redundant regex invocations.

**`max_windows_per_rule_variant: 16`.** At most 16 scan windows per rule variant per input chunk. After 16 windows, the engine stops creating new windows for that rule and moves to the next rule. This bounds worst-case CPU cost for inputs that produce many anchor hits for a single rule.

**`pressure_gap_start: 128`.** The starting gap size for the engine's pressure-relief mechanism. When anchor hit density exceeds this threshold, the engine widens the gap between sampled windows to maintain bounded latency.

**`max_anchor_hits_per_rule_variant: 2048`.** A hard cap on anchor hits before the engine stops looking for more in the current chunk. This prevents quadratic behavior on adversarial inputs that contain thousands of anchor matches (e.g., a file consisting entirely of the word "password" repeated).

**`max_utf16_decoded_bytes_per_window: 64 * 1024`.** Maximum decoded bytes when processing UTF-16 content within a single scan window. UTF-16 doubles input size; this cap prevents memory amplification.

**`max_transform_depth: 3`.** Maximum nesting depth for transform decoding. Base64-inside-URL-encoding-inside-Base64 is depth 3. The `decode_depth` config override replaces this value when specified.

**`max_total_decode_output_bytes: 512 * 1024`.** Total decoded output cap across all transforms in one chunk. This prevents memory exhaustion from deeply nested or wide decoding.

**`max_work_items: 256`.** Maximum internal work items tracked simultaneously. This bounds the scheduler's memory usage during parallel scan coordination.

**`max_findings_per_chunk: 8192`.** Maximum findings emitted per input chunk. This bounds output size for pathological inputs that match thousands of times (e.g., a file where every line contains a token pattern).

**`scan_utf16_variants: true`.** Enables UTF-16 (LE and BE) scanning. Secrets in UTF-16-encoded files (Windows configuration files, .NET resource files, XML documents with BOM headers) are detected.

## 7. The Available Workers Calculation

Worker count defaults to the platform's available parallelism:

```rust
fn available_workers() -> usize {
    std::thread::available_parallelism()
        .map(|count| count.get())
        .unwrap_or(1)
}
```

`std::thread::available_parallelism()` returns a `NonZeroUsize` representing the number of hardware threads available to the process. On a 16-core machine with hyperthreading, this returns 32. On failure (rare -- typically only in heavily sandboxed environments where the information is not available), it falls back to 1. The `.get()` call extracts the `usize` from the `NonZeroUsize`.

## 8. Putting It Together

The engine construction pipeline flows as:

```text
  RuntimeEngineConfig
         |
         v
  +-------------------+
  | load_runtime_rules |---> Vec<RuleSpec>
  +-------------------+
         |
         v
  +------------------------+
  | default_runtime_tuning |---> Tuning (optionally override depth)
  +------------------------+
         |
         v
  +----------------------------+
  | default_runtime_transforms |---> Vec<TransformConfig>
  | apply_transform_filter     |
  +----------------------------+
         |
         v
  +------------------------------------+
  | Engine::new_with_anchor_policy(    |
  |   rules, transforms, tuning,      |
  |   policy                           |
  | )                                  |
  +------------------------------------+
         |
         v
  Arc<Engine> (cached in OnceLock if default config)
```

The cache check happens outside this pipeline. The `runtime_engine` function first compares the config against the default. If equal, it returns the cached `Arc`. If not, it runs the full pipeline and wraps the result in a fresh `Arc`. This two-tier strategy gives the common case (default config, hundreds of assignments per process lifetime) zero marginal engine construction cost, while still supporting custom configurations when needed.

## What's Next

[Chapter 3](03-event-and-commit-sinks.md) examines the four event output formats (JSONL, Text, JSON, SARIF), the `CoordinationEventSink` for distributed event recording, and the `ReceiptCommitSink` that derives the full identity chain from engine-level finding records.
