# The Needle Problem -- Detection Engine Architecture

*A scanning pipeline processes a 2.3 GB repository. At byte offset 1,048,576 of a `.env` file, the string `AKIAIOSFODNN7EXAMPLE` appears — a 20-byte AWS access key embedded in 14 million lines of code. The pipeline has 847 active detection rules, each with a compiled regex. Running all 847 regexes against every byte of every file yields 847 × 2,300,000,000 = 1.95 trillion regex-byte evaluations. At 200 MB/s per regex, a single file takes 11,500 CPU-seconds. The scanner crawls. Meanwhile, the AWS key sits untouched in production, leaking credentials for 14 hours before the scan completes. Without a prefilter stage that narrows 847 rules to the 3 that have anchor patterns matching this byte region, detection at scale is not viable.*

---

The scanner-engine crate is a standalone detection engine extracted from the scanner-rs project. It solves a specific problem: given a set of declarative rule specifications and an arbitrary byte buffer, find all secrets present in that buffer while keeping per-byte cost bounded regardless of rule count. The engine achieves this through a multi-stage pipeline where each stage narrows the work for the next, ensuring that expensive operations (regex evaluation, entropy computation, structural validation) run only on small candidate windows rather than the full input.

## 1. The Module Map

The engine is organized into three layers: a public API surface, a compilation layer, and a runtime scanning layer. From `lib.rs`:

```rust
pub mod b64_yara_gate;
pub mod content_policy;
pub mod lsm;
pub mod perf_counters;
pub mod pool;
pub mod regex2anchor;
pub mod scratch_memory;

mod api;
mod engine;
mod perf_stats;
mod rules;
```

> **Note**: Test-support and harness modules (`test_utils`, `tiger_harness`, `demo`) are gated behind `cfg(test)` or feature flags and are omitted here.

The `api` module defines all configuration types (`RuleSpec`, `TransformConfig`, `Tuning`) and result types (`Finding`, `FindingRec`). The `rules` module handles YAML parsing and regex compilation. The `engine` module contains the runtime — everything that executes after compilation.

The public modules in `lib.rs` serve supporting roles:

- **`content_policy`** — Content classification for binary-aware scanning. The primary entry point `classify_content` combines a NUL-byte heuristic (SIMD-accelerated via `memchr`, inspired by Git's `buffer_is_binary`) with extension matching to produce a `ContentVerdict`: `Text` (scan normally), `Binary` (skip unless `--scan-binary`), or `BinaryExtractable` (extract text from known formats like `.ipynb`, `.class`, `.jar`/`.war`, `.pyc`, `.env`). No heap allocation.

- **`lsm`** — Set-associative cache with CLOCK Nth-Chance eviction, used for LSM hot-data caching. Cache-line-friendly layout with configurable associativity (2/4/16 ways), 8- or 16-bit tags, and packed clock counters. Values are `Copy` and slot-reused without destructors.

- **`perf_counters`** — Optional hardware performance counter integration for Git scanning instrumentation (`perf-counters` feature). Covers pack inflate, delta apply, scan bytes, mapping bridge, cache hits, and tree-load operations. Uses relaxed atomics for low overhead; snapshots are best-effort. When disabled, all functions are no-ops and snapshots return zeroed values.

> **Note**: `vs_cache` (on-disk cache for serialized Vectorscan databases) is *not* a crate-level public module — it lives at `engine/vs_cache.rs`, private to the `engine` module. It reduces repeated startup compilation by caching `hs_database_t` to disk, keyed by a deterministic BLAKE3 hash of compile inputs (kind, mode, platform, Vectorscan version, patterns, flags, IDs). Integrity is verified with a 16-byte AEGIS-128L MAC. Controlled via `SCANNER_VS_DB_CACHE` and `SCANNER_VS_DB_CACHE_DIR` environment variables. It appears in the `engine/mod.rs` listing below.

Inside `engine/`, the module layout from `engine/mod.rs` separates concerns by scan-pipeline stage:

```rust
// Internal compiled representation modules
mod decode_state;
mod hit_pool;
mod rule_repr;
mod work_items;

// Core engine and scratch modules
mod core;
pub(crate) mod offline_validate;
mod scratch;

// Scanning implementation modules (Engine impl extensions)
mod buffer_scan;
mod stream_decode;
mod window_validate;

// Supporting modules
mod helpers;
mod safelist;
mod simd_classify;
mod transform;
mod vectorscan_prefilter;
mod vs_cache;
```

> **Note**: Test modules are omitted from this listing for clarity.

This decomposition is not organizational convenience. Each module corresponds to a distinct phase in the scan pipeline, and the dependencies between modules encode the data-flow order.

## 2. The Data Flow: RuleSpec to Finding

The engine has two distinct phases: build and scan. The build phase runs once; the scan phase runs per chunk.

```mermaid
flowchart TD
    A[RuleSpec + TransformConfig + Tuning] --> B[Engine::new]
    B --> C[RuleCompiled + Gate Pools]
    B --> D[Vectorscan Prefilter DBs]
    B --> E[Base64 YARA Gate]
    
    F[Input Buffer] --> G[scan_chunk_into]
    G --> H{Vectorscan Prefilter}
    H -->|Zero hits| I[Return empty]
    H -->|Hits| J[Per-rule window merge]
    J --> K[window_validate: 17-step pipeline]
    K --> L[FindingRec in ScanScratch]
    L --> M[Transform decode BFS]
    M --> N[drain_findings]
    N --> O["Vec<Finding>"]
```

### Build Phase

`Engine::new` consumes a `Vec<RuleSpec>`, a `Vec<TransformConfig>`, and a `Tuning` struct. From `core.rs`:

```rust
pub struct Engine {
    /// Hot rule representation used in scan-loop validation.
    pub(super) rules_hot: Vec<RuleCompiled>,
    /// Cold per-rule metadata indexed in parallel with `rules_hot`.
    pub(super) rules_cold: Vec<RuleCold>,
    /// Ordered transform configurations; indices into this vector are used as
    /// `transform_idx` throughout work items, decode steps, and finding provenance.
    pub(super) transforms: Vec<TransformConfig>,
    /// Scan-time budgets and capacity knobs.
    pub(crate) tuning: Tuning,
    // ... gate pools, Vectorscan DBs, safelist ...
}
```

The engine is immutable after construction. Every field is frozen. All mutable scan state lives in `ScanScratch`, a separate struct allocated per worker thread.

### Scan Phase

`scan_chunk_into` is the single entry point for scanning. It takes a byte buffer, a file ID, and a mutable `ScanScratch`. The scan proceeds as a breadth-first traversal of work items:

1. Run Vectorscan prefilter on the root buffer to populate touched `(rule, variant)` pairs.
2. Enqueue `WorkItem::scan_root()` into the work queue.
3. Process work items in FIFO order: `ScanBuf` items run the window validation pipeline; `DecodeSpan` items decode encoded spans and enqueue new `ScanBuf` items.
4. Drain findings from scratch into the output vector.

## 3. The Hot/Cold Split Philosophy

The engine separates data by access frequency. This is not premature optimization — it is a cache-line discipline that determines whether the inner scan loop fits in L1.

Per-rule data is split into two parallel arrays. From `rule_repr.rs`:

```rust
/// Hot compiled rule representation used during scanning.
#[derive(Clone, Debug)]
pub(super) struct RuleCompiled {
    pub(super) re: Regex,
    pub(super) must_contain: Option<&'static [u8]>,
    pub(super) rule_meta: u32,
    pub(super) confirm_all: u32,
    pub(super) keywords: u32,
    pub(super) value_suppressors: u32,
    pub(super) entropy: u32,
    pub(super) char_class: u32,
    pub(super) local_context: u32,
    pub(super) two_phase: u32,
    pub(super) offline_validation: u32,
}
```

```rust
/// Cold rule metadata consulted only at finding-emission time.
#[derive(Clone, Copy, Debug)]
pub(super) struct RuleCold {
    /// Human-readable rule name used in finding reports.
    pub(super) name: &'static str,
    /// BLAKE3 derive-key fingerprint of the rule name.
    ///
    /// Precomputed at engine construction via BLAKE3 derive-key mode over
    /// the `"gossip/rule/v1"` domain constant and `name`. Stored as raw
    /// bytes because `scanner-engine` does not depend on `gossip-contracts`
    /// (which owns `RuleFingerprint`). The `ScanEngine` trait adapter in
    /// `scanner-scheduler` wraps these bytes into `RuleFingerprint`.
    pub(super) fingerprint: [u8; 32],
    /// Effective minimum confidence threshold for this rule.
    pub(super) min_confidence: i8,
}
```

Compile-time size guards enforce this discipline:

```rust
const _: () = assert!(std::mem::size_of::<RuleCompiled>() <= 88);
const _: () = assert!(std::mem::size_of::<RuleCold>() <= 56);
```

**`RuleCompiled`** is touched for every candidate window. It contains the regex, the must-contain needle, and `u32` indices into gate pools. Gate objects (keyword tables, entropy parameters, two-phase configs) live in separate `Vec`s on `Engine` and are resolved through pool accessors only when the gate is present (`!= NO_GATE`). This indirection keeps `RuleCompiled` at 88 bytes — small enough to fit within a cache-line pair (128 bytes).

**`RuleCold`** is touched only when a finding survives all gates and is about to be emitted. It carries the rule name (a pointer + length), a 32-byte BLAKE3 derive-key fingerprint of the rule name (precomputed at engine construction for identity-chain derivation), and the minimum confidence threshold. The fingerprint is the dominant size contributor — 32 of the struct's 56 bytes. Storing these fields here instead of in `RuleCompiled` avoids polluting the hot array with data that is read once per emitted finding, not once per candidate window.

The same hot/cold split applies to `ScanScratch`. From `scratch.rs`:

```rust
/// # Layout — hot / cold split
///
/// `#[repr(C)]` preserves declared field order so the explicit 64-byte
/// boundary between hot and cold regions remains stable.
#[repr(C)]
pub struct ScanScratch {
    // ---------------- Hot scan-loop region ----------------
    pub(super) out: ScratchVec<FindingRec>,
    pub(super) norm_hash: ScratchVec<NormHash>,
    pub(super) drop_hint_end: ScratchVec<u64>,
    pub(super) max_findings: usize,
    pub(super) findings_dropped: usize,
    pub(super) work_q: ScratchVec<WorkItem>,
    pub(super) work_head: usize,
    // ... more hot fields ...

    // --------------- Cache-line boundary ----------------
    _cold_boundary: CachelineBoundary,

    // ---------------- Cold / conditional region ----------------
    pub(super) slab: DecodeSlab,
    pub(super) seen: FixedSet128,
    pub(super) seen_findings: FixedSet128,
    // ... decode ring, pending windows, stream state ...
}
```

The `CachelineBoundary` is a zero-sized type with `#[repr(align(64))]`:

```rust
#[repr(align(64))]
struct CachelineBoundary {
    _pad: [u8; 0],
}
```

Placed between hot and cold fields in a `#[repr(C)]` struct, this forces the first cold field (`slab`) to begin on a fresh 64-byte cache line. When transforms are inactive — the common case for many file types — cold fields are never touched, and the hot region's cache lines remain undisturbed.

## 4. The Gate Pool Architecture

Gates are heavyweight structures (keyword tables contain packed byte patterns; entropy gates carry precomputed log2 tables). Inlining them into `RuleCompiled` would bloat the hot array. Instead, the engine uses typed pool vectors with `u32` indices and a `NO_GATE` sentinel:

```rust
/// Sentinel value indicating no gate is assigned for a given slot.
pub(super) const NO_GATE: u32 = u32::MAX;
```

From `core.rs`, the engine holds one pool per gate type:

```rust
pub(super) confirm_all_gates: Vec<ConfirmAllCompiled>,
pub(super) keyword_gates: Vec<KeywordsCompiled>,
pub(super) value_suppressor_gates: Vec<PackedPatterns>,
pub(super) entropy_gates: Vec<EntropyCompiled>,
pub(super) two_phase_gates: Vec<TwoPhaseCompiled>,
pub(super) local_context_gates: Vec<LocalContextSpec>,
pub(super) offline_validation_gates: Vec<OfflineValidationSpec>,
pub(super) char_class_gates: Vec<CharClassCompiled>,
```

Resolution is through `#[inline(always)]` accessors:

```rust
#[inline(always)]
pub(super) fn entropy_gate(&self, idx: u32) -> Option<EntropyCompiled> {
    if idx == NO_GATE {
        None
    } else {
        Some(self.entropy_gates[idx as usize])
    }
}
```

Using `u32::MAX` as a sentinel instead of `Option<u32>` saves 4 bytes per gate field (no discriminant padding), shrinking `RuleCompiled` by approximately 32 bytes across its eight gate fields. Valid pool indices never reach `u32::MAX` because each pool has at most one entry per rule, and the rule count is bounded well below `u32::MAX` by practical memory limits.

## 5. The Allocation-Free Hot Path

The engine enforces a strict allocation discipline on the hot path. All per-scan buffers are pre-allocated in `ScanScratch::new` and reused across chunks:

```rust
pub(super) fn new(engine: &Engine) -> Self {
    let max_findings = engine.tuning.max_findings_per_chunk;
    // ...
    Self {
        out: ScratchVec::with_capacity(max_findings).expect("scratch out allocation failed"),
        norm_hash: ScratchVec::with_capacity(max_findings)
            .expect("scratch norm_hash allocation failed"),
        drop_hint_end: ScratchVec::with_capacity(max_findings)
            .expect("scratch drop_hint_end allocation failed"),
        // ...
    }
}
```

`ScratchVec` is a page-aligned, fixed-capacity vector that never reallocates. From `scratch_memory.rs`:

```rust
/// Fixed-capacity scratch vector backed by page-aligned storage.
///
/// This is a `Vec`-like API with a hard capacity. It never reallocates, so
/// once constructed it is safe to use in hot loops without risking
/// allocations.
pub struct ScratchVec<T> {
    ptr: NonNull<MaybeUninit<T>>,
    len: u32,
    cap: u32,
}
```

Page alignment (4 KiB minimum) keeps buffers SIMD-friendly. The `u32` length and capacity fields save 8 bytes versus `usize` on 64-bit targets while still accommodating the maximum buffer sizes used in practice (all offsets are `u32` by design).

Between scans, `reset_for_scan` clears per-scan state while preserving allocations:

```rust
pub(super) fn reset_for_scan(&mut self, engine: &Engine) {
    self.reset_common();
    self.hit_acc_pool
        .reset_touched(self.touched_pairs.as_slice());
    self.touched_pairs.clear();
    self.windows.clear();
    self.expanded.clear();
    self.spans.clear();
    self.ensure_capacity(engine);
}
```

The `ensure_capacity` call is idempotent after the first invocation — since `Engine` is immutable, capacity checks only matter on the first call. A `capacity_validated` flag skips the validation block on subsequent calls.

## 6. Encoding Variants

Every anchor, keyword, and confirm-all pattern is compiled into three encoding variants: Raw, UTF-16LE, and UTF-16BE. This tripling exists because real-world files contain UTF-16-encoded secrets (Windows registry exports, XML with BOM, PowerShell scripts). From `rule_repr.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub(super) enum Variant {
    Raw,
    Utf16Le,
    Utf16Be,
}

impl Variant {
    /// Stable index into per-variant `[_; 3]` arrays.
    pub(super) fn idx(self) -> usize {
        match self {
            Variant::Raw => 0,
            Variant::Utf16Le => 1,
            Variant::Utf16Be => 2,
        }
    }

    /// Scale a character radius into a byte radius for this variant.
    pub(super) fn scale(self) -> usize {
        match self {
            Variant::Raw => 1,
            Variant::Utf16Le | Variant::Utf16Be => 2,
        }
    }
}
```

The `(rule, variant)` pair is encoded as a flat integer `pair = rule_id * 3 + variant_idx`. This avoids a two-level map and keeps the hot loop cache-friendly. The `HitAccPool` (Chapter 3) uses this flat encoding to index into its per-pair accumulator storage.

> **Note**: The `rule_id * 3 + variant_idx` formula applies specifically to the `HitAccPool` pair index. The `Target` encoding used in the anchor map (Section 7.1 of Chapter 2) uses a different scheme: `(rule_id << 2) | variant_idx`, packing the variant tag into the low 2 bits.

## 7. Budget Enforcement

Every scan is bounded by the `Tuning` parameters. From `api.rs`:

```rust
pub struct Tuning {
    pub merge_gap: usize,
    pub max_windows_per_rule_variant: usize,
    pub pressure_gap_start: usize,
    pub max_anchor_hits_per_rule_variant: usize,
    pub max_utf16_decoded_bytes_per_window: usize,
    pub max_transform_depth: usize,
    pub max_total_decode_output_bytes: usize,
    pub max_work_items: usize,
    pub max_findings_per_chunk: usize,
    pub scan_utf16_variants: bool,
}
```

These are not suggestions. They are hard limits enforced at every stage. `max_findings_per_chunk` is enforced at finding insertion time; overflow increments `findings_dropped` rather than allocating. `max_work_items` caps the BFS work queue. `max_total_decode_output_bytes` limits total decoded output across all transforms. When any budget is exceeded, work is dropped — the engine favors bounded latency over completeness. This is the correct trade-off for a production scanner that must process adversarial input without memory exhaustion.

## Summary

The scanner-engine separates compilation from scanning, uses a hot/cold data split to maximize cache utilization, pools heavyweight gate objects behind `u32` indices with `NO_GATE` sentinels, and enforces per-scan budgets through `Tuning`. The engine is immutable after construction; all mutable state lives in `ScanScratch`, which is single-threaded and reused across chunks. Chapter 2 examines how `RuleSpec` values are compiled into this runtime representation — the bridge between declarative YAML rules and the optimized data structures described here.
