# The Engine Factory -- Rule Loading, Transforms, and Tuning

*The scanner engine is the most expensive object to construct in the runtime. It contains compiled detection rules (regex automata compiled to bytecode), transform decoder configurations (base64, URL-percent, with span limits and depth caps), tuning parameters (merge gaps, pressure limits, finding caps), and anchor extraction tables (fast-path byte sequences that avoid running regex against every byte of input). A production rule set with 800 patterns and 12 transform decoders may take 50-80 milliseconds to compile. The runtime must build this object before any scan can execute.*

---

The scanner engine is the most expensive object to construct in the runtime. It contains compiled detection rules (regex automata compiled to bytecode), transform decoder configurations (base64, URL-percent, with span limits and depth caps), tuning parameters (merge gaps, pressure limits, finding caps), and anchor extraction tables (fast-path byte sequences that avoid running regex against every byte of input). The runtime must build this object before any scan can execute. This chapter examines the engine construction pipeline: how rules are loaded, how transforms are filtered, how tuning is applied, and how the engine is constructed with `catch_unwind` to convert rule-compilation panics into recoverable errors.

## 1. Engine Configuration Parameters

Engine construction is driven by four individual parameters extracted from the caller's `FsScanConfig` or `GitScanConfig`. There is no intermediate config struct -- `build_runtime_engine` takes the parameters directly:

**`rules_file: Option<&Path>`.** Path to an external YAML rules file. When `None`, the runtime uses built-in default rules. When `Some`, the runtime reads and parses the file, replacing the default rules entirely.

**`transform_filter: &TransformFilter`.** Controls which transform decoders are enabled, as described in [Chapter 1](01-runtime-architecture.md).

**`decode_depth: Option<usize>`.** Overrides the maximum depth for nested transform decoding. A `None` value uses the engine's default depth (3 levels). A `Some(1)` limits decoding to a single layer -- useful for scans where nested encoding is rare and decode overhead is undesirable. A `Some(0)` effectively disables transform decoding.

**`anchor_mode: AnchorMode`.** Selects how the engine extracts anchors from input data. The `AnchorMode` enum was introduced in [Chapter 1](01-runtime-architecture.md). `Manual` uses hand-curated anchor byte sequences defined in each rule spec (the `anchors` field on `RuleSpec`). `Derived` uses anchors automatically extracted from the rule's regex pattern by the engine's planner. Manual anchors are more precise (the rule author chose them based on domain knowledge); derived anchors are more complete (the planner extracts every literal prefix from the regex, catching patterns the author may have missed).

Each call to `build_runtime_engine` constructs a fresh engine. The `Arc` wrapper allows the engine to be shared between the runtime and the scan dispatch without cloning the engine's internal data structures (compiled regex automata, anchor lookup tables, transform configurations). The runtime passes `Arc<scanner_engine::Engine>` to the scan module, so the dispatch holds a reference-counted handle. When the scan completes, the reference count decrements.

## 2. The Engine Build Pipeline

The `build_runtime_engine` function takes four individual parameters and assembles the engine:

```rust
pub(crate) fn build_runtime_engine(
    rules_file: Option<&Path>,
    transform_filter: &TransformFilter,
    decode_depth: Option<usize>,
    anchor_mode: AnchorMode,
) -> Result<Arc<scanner_engine::Engine>, ScanRuntimeError> {
    let rules_path = rules_file.map(Path::to_path_buf);
    let rules = load_runtime_rules(rules_file)?;
    let transforms = filtered_runtime_transforms(transform_filter);
    let mut tuning = default_runtime_tuning();
    if let Some(depth) = decode_depth {
        tuning.max_transform_depth = depth;
    }

    let engine = std::panic::catch_unwind(AssertUnwindSafe(|| {
        scanner_engine::Engine::new_with_anchor_policy(
            rules, transforms, tuning, anchor_policy(anchor_mode),
        )
    }))
    .map_err(|payload| ScanRuntimeError::RulesConfig {
        path: rules_path,
        message: panic_payload_message(payload),
    })?;

    Ok(Arc::new(engine))
}
```

Five steps, each producing one input to the engine constructor:

1. **Load rules.** From an external file or from the built-in defaults.
2. **Filter transforms.** Start with the default transform set, apply the filter to retain only the desired decoders.
3. **Configure tuning.** Start with the default tuning parameters, override the decode depth if specified.
4. **Construct engine inside `catch_unwind`.** The engine constructor runs inside `std::panic::catch_unwind` to convert panics in rule compilation (e.g., a malformed external regex) into recoverable `ScanRuntimeError::RulesConfig` errors rather than crashing the process.
5. **Wrap in `Arc`.** The engine is returned as `Arc<scanner_engine::Engine>` for shared ownership.

Each step is examined in detail below.

## 3. Rule Loading

Rules are loaded through a simple two-branch resolution:

```rust
fn load_runtime_rules(
    rules_file: Option<&Path>,
) -> Result<Vec<scanner_engine::RuleSpec>, ScanRuntimeError> {
    match rules_file {
        Some(path) => {
            let content = scanner_engine::read_rules_text(path).map_err(|error| {
                ScanRuntimeError::RulesConfig {
                    path: Some(path.to_path_buf()),
                    message: error.to_string(),
                }
            })?;
            scanner_engine::load_rules_from_content(&content).map_err(|error| {
                ScanRuntimeError::RulesConfig {
                    path: Some(path.to_path_buf()),
                    message: error.to_string(),
                }
            })
        }
        None => Ok(scanner_engine::builtin_rules()),
    }
}
```

Two branches:

**Explicit `--rules=<path>` override.** When the caller specifies a rules file path (via `FsScanConfig::rules_file` or `GitScanConfig::rules_file`), the function reads it with `scanner_engine::read_rules_text`, then parses it with `scanner_engine::load_rules_from_content`. Both steps map errors into `ScanRuntimeError::RulesConfig` with the original path for diagnostic context.

**Built-in fallback via `scanner_engine::builtin_rules()`.** When no file path is provided, the function returns the compile-time embedded rules. The `scanner_engine` crate embeds `default_rules.yaml` via `include_str!` and parses it on first call (cached via `OnceLock`). This guarantees that the scanner always has a functional rule set, even in minimal container deployments that contain only the binary.

## 4. Transform Configuration

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

The filter is applied post-construction via `filtered_runtime_transforms`, which takes only the filter parameter:

```rust
fn filtered_runtime_transforms(filter: &TransformFilter) -> Vec<TransformConfig> {
    let mut transforms = default_runtime_transforms();
    match filter {
        TransformFilter::All => transforms,
        TransformFilter::None => Vec::new(),
        TransformFilter::Only(ids) => {
            transforms.retain(|transform| ids.contains(&transform.id));
            transforms
        }
    }
}
```

The function starts from the default transform set and applies the filter: `All` returns the full set, `None` returns an empty vec, and `Only` uses `retain` to keep only matching IDs in place.

## 5. Tuning Parameters

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

## 6. The Available Workers Calculation

Worker count defaults to the platform's available parallelism:

```rust
pub(crate) fn available_workers() -> usize {
    std::thread::available_parallelism()
        .map(|count| count.get())
        .unwrap_or_else(|error| {
            tracing::warn!(%error, "failed to query available parallelism, defaulting to 1");
            1
        })
}
```

`std::thread::available_parallelism()` returns a `NonZeroUsize` representing the number of hardware threads available to the process. On a 16-core machine with hyperthreading, this returns 32. On failure (rare -- typically only in heavily sandboxed environments where the information is not available), it logs a warning via `tracing::warn!` with the error details and falls back to 1. The `.get()` call extracts the `usize` from the `NonZeroUsize`.

## 7. Putting It Together

The engine construction pipeline flows as:

```text
  build_runtime_engine(rules_file, transform_filter, decode_depth, anchor_mode)
         |
         v
  +-------------------+
  | load_runtime_rules |---> Vec<RuleSpec>
  +-------------------+
         |
         v
  +------------------------------+
  | filtered_runtime_transforms  |---> Vec<TransformConfig>
  +------------------------------+
         |
         v
  +------------------------+
  | default_runtime_tuning |---> Tuning (optionally override depth)
  +------------------------+
         |
         v
  +------------------------------------+
  | catch_unwind {                     |
  |   Engine::new_with_anchor_policy(  |
  |     rules, transforms, tuning,     |
  |     policy                         |
  |   )                                |
  | }                                  |
  +------------------------------------+
         |
         v
  Arc<Engine>
```

Each call to `build_runtime_engine` runs the full pipeline and wraps the result in a fresh `Arc`. The `catch_unwind` wrapper ensures that panics during rule compilation (e.g., from malformed external regex patterns) are captured and converted into recoverable `ScanRuntimeError::RulesConfig` errors rather than crashing the process.

## What's Next

[Chapter 3](03-event-and-commit-sinks.md) examines the four event output formats (JSONL, Text, JSON, SARIF), the `CoordinationEventSink` for distributed event recording, and the `ReceiptCommitSink` that derives the full identity chain from engine-level finding records.
