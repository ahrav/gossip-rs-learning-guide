# Prefilter, Then Prove -- The Scan Pipeline

*A 16 KB chunk from a `.env` file arrives at the scanner. The engine has 847 compiled rules. A naive approach runs all 847 regexes against all 16,384 bytes -- 13.9 million regex-byte evaluations for a single chunk. The scan takes 4.2 ms. At 500,000 chunks per second across a 14 TB repository, that is 2,100 CPU-seconds per second of wall-clock time. The pipeline stalls. But the Vectorscan prefilter, sweeping the same 16 KB in 12 microseconds, reports that only 3 anchor patterns matched, touching 3 of the 2,541 possible (rule, variant) pairs. The engine validates 3 windows totaling 1,200 bytes of regex work instead of 13.9 million. Without the prefilter stage narrowing rules to the handful that matter for this chunk, detection at scale collapses under its own weight.*

---

The scan pipeline is the runtime heart of the scanner-engine. Chapter 1 introduced the architecture; this chapter traces the full path a single chunk takes through `scan_chunk_into` -- from Vectorscan prefilter through work-queue traversal to finding emission. Every decision in this pipeline exists to enforce a single principle: *work must be proportional to hits, not to the product of rules and bytes*.

## 1. The Entry Point: `scan_chunk_into`

The top-level scan method lives in `core.rs`. It accepts a root buffer, a file identifier, a base offset for coordinate mapping, and a reusable scratch state:

```rust
pub fn scan_chunk_into(
    &self,
    root_buf: &[u8],
    file_id: FileId,
    base_offset: u64,
    scratch: &mut ScanScratch,
) {
```

The method orchestrates seven steps, labeled A through F in the source plus the work-queue loop. Each step either narrows work or enforces a budget.

### 1.1 Step A-C: Capacity, Reset, and Overlap

Before any pattern matching begins, three housekeeping operations execute. `ensure_capacity` lazily allocates Vectorscan scratch on the first call -- subsequent calls are no-ops. The hit accumulator pool is reset for any pairs touched by a prior scan. Overlap tracking records the chunk's file ID and base offset so that `drop_prefix_findings` can later suppress duplicate findings in overlapping regions between adjacent chunks.

These three steps are cheap (no heap traffic after the first call) and establish invariants the rest of the pipeline depends on: accumulator pools start zeroed, and the scratch knows its position in the file stream.

A precondition enforced via debug assertion: `root_buf.len() <= u32::MAX`. All span offsets in `FindingRec` are `u32`-addressed, and the pair encoding uses `u32` indices. Buffers exceeding 4 GiB would cause silent truncation in span offsets. The caller is responsible for chunking input so each buffer fits within this bound.

### 1.2 Step D: The Vectorscan Prefilter

Step D runs the Vectorscan multi-pattern matcher against the raw root buffer:

```rust
let vs = self
    .vs
    .as_ref()
    .expect("vectorscan prefilter database unavailable (fallback disabled)");
let mut vs_scratch_owned = scratch
    .vs_scratch
    .take()
    .expect("vectorscan scratch missing");
let (result, _vs_nanos) =
    perf::time(|| vs.scan_raw(root_buf, scratch, &mut vs_scratch_owned));
perf::record_scan_vs_prefilter(_vs_nanos);
scratch.vs_scratch = Some(vs_scratch_owned);
```

The Vectorscan database (`VsPrefilterDb`) was compiled during engine construction from anchor patterns and (for rules with weak anchors) raw regex patterns. When Vectorscan fires a callback, it records the hit into a per-(rule, variant) accumulator in the scratch state. The accumulator uses a flat encoding: `pair = rule_id * 3 + variant_idx`, where `Raw = 0`, `Utf16Le = 1`, `Utf16Be = 2`. This avoids a two-level map and keeps the touched-pairs vector as a cache-friendly `u32` list.

The prefilter also returns a boolean indicating whether any UTF-16 pattern matched, which controls whether UTF-16 windows need sorting downstream.

Construction of the prefilter is fail-fast: if the Vectorscan database cannot be compiled, the engine panics at build time rather than silently falling back to full-buffer scans. This is a deliberate design choice -- a silent fallback would hide a catastrophic performance regression where every chunk triggers full-rule evaluation, and the operator would see only increased latency with no diagnostic signal pointing to the root cause.

The Vectorscan scratch buffer follows a take-put ownership pattern: the caller takes the scratch out of the `Option`, passes it by mutable reference to the scan call, then puts it back. This avoids requiring `ScanScratch` to implement `Clone` or allocating a fresh scratch per scan. The `Option` wrapper exists because Vectorscan scratch is lazily allocated on first use -- before the first scan, the slot is `None` and `ensure_capacity` fills it.

### 1.3 The One-Shot Prefilter Flag

A subtle coordination detail: `scan_chunk_into` runs the prefilter on the root buffer at Step D, but the actual rule validation happens inside `scan_rules_on_buffer` (called from the work-queue loop). Without coordination, `scan_rules_on_buffer` would re-run the prefilter on the root buffer -- doubling the cost. A one-shot flag solves this:

```rust
scratch.root_prefilter_saw_utf16 = saw_utf16;
scratch.root_prefilter_done = true;
```

Inside `scan_rules_on_buffer`, the flag is consumed:

```rust
let (mut used_vectorscan_utf16, skip_prefilter) = if scratch.root_prefilter_done {
    scratch.root_prefilter_done = false;
    (scratch.root_prefilter_saw_utf16, true)
} else {
    (false, false)
};
```

The first call (root buffer) skips the prefilter because the caller already ran it. Subsequent calls (transform-decoded buffers) see `false` and run the full prefilter. This is a one-shot signal: consumed on read, never reset to `true` again during the same scan.

## 2. The Zero-Hit Bypass

Step E handles the common case: nothing matched.

```rust
if scratch.touched_pairs.is_empty() {
    perf::record_scan_zero_hit_chunk();

    let needs_transform_scan = self.has_active_transforms
        && (self
            .scanbuf_transform_idxs_active_non_base64
            .iter()
            .any(|&tidx| {
                let tc = &self.transforms[tidx];
                root_buf.len() >= tc.min_len
                    && transform_quick_trigger(tc, root_buf)
                    && self.base64_buffer_gate(tc, root_buf)
                    && (tc.id != TransformId::UrlPercent
                        || self.url_percent_buffer_gate(tc, root_buf))
            })
            || self
                .scanbuf_transform_idxs_active_base64
                .iter()
                .any(|&tidx| {
                    let tc = &self.transforms[tidx];
                    root_buf.len() >= tc.min_len
                        && transform_quick_trigger(tc, root_buf)
                        && self.base64_buffer_gate(tc, root_buf)
                }));

    if !needs_transform_scan {
        perf::record_scan_prefilter_bypass();
        scratch.out.clear();
        scratch.norm_hash.clear();
        scratch.drop_hint_end.clear();
        return;
    }
}
```

When no Vectorscan pattern fired, no regex validation is needed. But the pipeline cannot return immediately -- encoded content (Base64, URL-percent) may hide secrets whose anchor patterns are invisible in raw bytes. Transform discovery must still run unless every active transform's buffer-level gate rejects this chunk.

The gate cascade for each transform checks four conditions in short-circuit order: minimum length, quick-trigger byte presence, Base64 YARA pre-gate (for Base64 transforms), and URL-percent anchor-byte-set gate (for URL-percent transforms). The URL-percent gate uses a 256-bit bitmap (`anchor_byte_set`) built during engine construction. The bitmap is indexed as `set[byte >> 6] & (1 << (byte & 63))`. A `%XX` triplet only matters if its decoded byte appears in at least one anchor pattern. Buffers with `%d` and `%s` format specifiers but no anchor-relevant percent-encodings are rejected without any decoding work. The gate implementation from `core.rs`:

```rust
pub(super) fn url_percent_gate_check(
    anchor_byte_set: &[u64; 4],
    plus_to_space: bool,
    buf: &[u8],
) -> bool {
    if *anchor_byte_set == [0u64; 4] {
        return true;
    }
    if plus_to_space
        && memchr::memchr(b'+', buf).is_some()
        && (anchor_byte_set[(b' ' >> 6) as usize] & (1u64 << (b' ' & 63)) != 0)
    {
        return true;
    }
    let bytes = buf;
    let mut i = 0;
    while i < bytes.len() {
        if bytes[i] == b'%' {
            if i + 2 < bytes.len() {
                if let Some(decoded) = decode_hex_pair(bytes[i + 1], bytes[i + 2]) {
                    if anchor_byte_set[(decoded >> 6) as usize] & (1u64 << (decoded & 63)) != 0 {
                        return true;
                    }
                    i += 3;
                    continue;
                }
            } else {
                return true;
            }
        }
        i += 1;
    }
    false
}
```

The gate handles three edge cases: an empty anchor set (conservative pass), `plus_to_space` mode where `+` decodes to space without `%XX` encoding, and truncated `%` triplets at end-of-buffer (conservative pass). The function returns `false` only when it can prove no decoded byte would match any anchor -- the common case for format-specifier-heavy buffers.

When every gate rejects every transform, `scan_chunk_into` clears the output vectors and returns. This is the fast path for the majority of chunks in most repositories -- configuration files, documentation, and source code without secrets or encoded content. The prefilter bypass perf counter tracks how often this path fires.

## 3. Pair Encoding and the Touched-Pairs Model

The prefilter populates `scratch.touched_pairs` -- a list of `u32` values, each encoding a (rule, variant) pair as `rule_id * 3 + variant_idx`. The buffer-scan loop in `scan_rules_on_buffer` iterates only these touched pairs:

```rust
const VARIANTS: [Variant; 3] = [Variant::Raw, Variant::Utf16Le, Variant::Utf16Be];
let touched_len = scratch.touched_pairs.len();
for i in 0..touched_len {
    let pair = scratch.touched_pairs[i] as usize;
    let rid = pair / 3;
    let vidx = pair % 3;
    let variant = VARIANTS[vidx];
    let rule = &self.rules_hot[rid];
    let gates = self.resolve_gates(rid as u32, rule);

    scratch.hit_acc_pool.take_into(pair, &mut scratch.windows);
    if scratch.windows.is_empty() {
        continue;
    }
```

This encoding ensures that work scales with the number of touched pairs, not with the total rule count. For a 847-rule engine where Vectorscan touches 3 pairs, the loop iterates 3 times -- not 2,541 (847 rules times 3 variants). On a typical scan where most chunks match zero or a handful of rules, this reduces the per-chunk work from O(rules) to O(hits).

The `take_into` call moves accumulated hit windows from the pool into `scratch.windows` for the current pair. Each hit window is a `SpanU32` -- a compact 12-byte struct holding `start`, `end`, and `anchor_hint` as `u32` values:

```rust
pub(super) struct SpanU32 {
    pub(super) start: u32,
    pub(super) end: u32,
    pub(super) anchor_hint: u32,
}
```

The `anchor_hint` records where Vectorscan reported the match start. This value is crucial for two downstream operations: the regex uses it to focus its search (via `BACK_SCAN_MARGIN`), and the merge algorithm preserves the earliest hint when collapsing adjacent windows.

Gate resolution happens once per pair via `resolve_gates`, which hoists all per-rule gate-pool lookups into a stack-local `ResolvedGates` struct. From `window_validate.rs`:

```rust
pub(super) struct ResolvedGates<'e> {
    pub(super) confirm_all: Option<&'e ConfirmAllCompiled>,
    pub(super) keywords: Option<&'e KeywordsCompiled>,
    pub(super) entropy: Option<EntropyCompiled>,
    pub(super) char_class: Option<CharClassCompiled>,
    pub(super) value_suppressors: Option<&'e PackedPatterns>,
    pub(super) local_context: Option<LocalContextSpec>,
    pub(super) min_confidence: i8,
}
```

Without this struct, LLVM may conservatively re-load gate pool references after `&mut ScanScratch` mutations inside the per-window inner loop. Rust's aliasing rules guarantee `&self` and `&mut ScanScratch` do not overlap, but the optimizer does not always exploit `noalias` through complex call chains. Hoisting the lookups into local variables keeps them in registers across the inner loop, eliminating redundant `NO_GATE` sentinel checks and repeated pool indexing.

## 4. Window Merge, Coalesce, and Pressure

Vectorscan callbacks fire in match-end order, not start order. Multiple patterns may interleave. The raw hit windows for each (rule, variant) pair arrive unsorted. From `buffer_scan.rs`:

```rust
if (variant == Variant::Raw && scratch.windows.len() > 1)
    || (used_vectorscan_utf16
        && matches!(variant, Variant::Utf16Le | Variant::Utf16Be)
        && scratch.windows.len() > 1)
{
    scratch
        .windows
        .as_mut_slice()
        .sort_unstable_by_key(|s| s.start);
}

merge_ranges_with_gap_sorted(&mut scratch.windows, merge_gap);
coalesce_under_pressure_sorted(
    &mut scratch.windows,
    hay_len,
    pressure_gap_start,
    self.tuning.max_windows_per_rule_variant,
);
```

Three operations run in sequence:

**Sort.** Required for raw variants (always) and UTF-16 variants (only when the Vectorscan UTF-16 DB was used). When UTF-16 hits come from literal-anchor matching alone, they arrive pre-sorted -- skipping the sort saves measurable time on buffers with many UTF-16 windows. The sort uses `sort_unstable_by_key` keyed on `start`, which avoids the allocation overhead of stable sorting and is well-suited to the nearly-sorted inputs that Vectorscan often produces (match-end order is close to start order for patterns of similar length).

**Gap-tolerant merge.** From `helpers/window.rs`, this O(n) in-place pass collapses adjacent or overlapping windows separated by at most `merge_gap` bytes into a single wider window:

```rust
pub(crate) fn merge_ranges_with_gap_sorted(ranges: &mut ScratchVec<SpanU32>, gap: u32) {
    let len = ranges.len();
    if len <= 1 {
        return;
    }
    let s = ranges.as_mut_slice();
    let mut write = 0usize;
    let mut cur = s[0];
    for i in 1..len {
        let r = unsafe { *s.get_unchecked(i) };
        debug_assert!(r.start >= cur.start);
        if r.start <= cur.end.saturating_add(gap) {
            cur.end = cur.end.max(r.end);
            cur.anchor_hint = cur.anchor_hint.min(r.anchor_hint);
        } else {
            unsafe { *s.get_unchecked_mut(write) = cur };
            write += 1;
            cur = r;
        }
    }
    unsafe { *s.get_unchecked_mut(write) = cur };
    write += 1;
    ranges.truncate(write);
}
```

When merging, the earliest anchor hint is preserved -- this is essential for correctness because the anchor hint tells the regex where to start searching.

**Pressure coalesce.** When the window count still exceeds `max_windows_per_rule_variant` after merging, `coalesce_under_pressure_sorted` doubles the merge gap exponentially and re-merges:

```rust
pub(crate) fn coalesce_under_pressure_sorted(
    ranges: &mut ScratchVec<SpanU32>,
    hay_len: u32,
    mut gap: u32,
    max_windows: usize,
) {
    if ranges.len() <= max_windows {
        return;
    }
    while ranges.len() > max_windows && gap < hay_len {
        merge_ranges_with_gap_sorted(ranges, gap);
        gap = gap.saturating_mul(2);
    }
    if ranges.len() > max_windows && !ranges.is_empty() {
        let start = ranges[0].start;
        let end = ranges[ranges.len() - 1].end;
        let mut min_anchor = start;
        for i in 0..ranges.len() {
            min_anchor = min_anchor.min(ranges[i].anchor_hint);
        }
        ranges.clear();
        ranges.push(SpanU32 {
            start: start.min(hay_len),
            end: end.min(hay_len),
            anchor_hint: min_anchor.min(hay_len),
        });
    }
}
```

The exponential gap doubling is O(n * log(hay_len / gap)) -- at most a handful of passes. If that still leaves too many windows, a hard fallback collapses everything into a single window spanning the first to last range. This bounds the number of regex validation passes per (rule, variant) pair deterministically, preventing pathological inputs (e.g., a buffer where every byte triggers an anchor match) from causing unbounded work. The tradeoff is wider windows -- more bytes for the regex to scan -- but the regex cost is bounded by a constant multiplier rather than growing with hit count.

## 5. Two-Phase Confirmation

Some rules have noisy prefilter patterns that trigger frequently but whose regexes are expensive. Two-phase confirmation filters these cheaply. From `buffer_scan.rs`:

```rust
if let Some(tp) = self.two_phase_gate(rule.two_phase) {
    let seed_radius_bytes = tp.seed_radius.saturating_mul(variant.scale());
    let full_radius_bytes = tp.full_radius.saturating_mul(variant.scale());
    let extra = full_radius_bytes.saturating_sub(seed_radius_bytes);

    scratch.expanded.clear();
    let windows_len = scratch.windows.len();
    for i in 0..windows_len {
        let seed = scratch.windows[i];
        let seed_range = seed.to_range();
        let win = &buf[seed_range.clone()];
        if !contains_any_memmem(win, &tp.confirm[vidx]) {
            continue;
        }
        let lo = seed_range.start.saturating_sub(extra);
        let hi = (seed_range.end + extra).min(buf.len());
        scratch
            .expanded
            .push(SpanU32::new(lo, hi, seed.anchor_hint as usize));
    }
```

The seed pass runs a cheap `memmem` check at a narrow radius. Confirm patterns are mandatory literal sub-expressions extracted from the regex AST -- their absence in the seed window proves no match exists in any superset. This rejects 80-95% of noisy prefilter hits before paying the O(regex) cost.

Surviving windows are expanded from `seed_radius` to `full_radius`, then re-merged and re-coalesced with the same algorithms from Section 4. Only the expanded, confirmed windows proceed to regex validation via `run_rule_on_window`.

The soundness argument for two-phase confirmation is straightforward: confirm patterns are mandatory literal sub-expressions extracted from the regex AST during engine construction. Because these literals *must* appear in any valid regex match, their absence in the seed window proves no match exists in any superset of that window. Widening after a miss would be wasted work. The confirm set is indexed by variant (`vidx`) so the memmem check uses the correct byte encoding (raw ASCII for `Variant::Raw`, UTF-16-encoded literals for `Variant::Utf16Le` and `Variant::Utf16Be`).

## 6. The Work-Queue Loop: BFS Over Decode Layers

After `scan_rules_on_buffer` processes the root buffer, the pipeline enters a breadth-first work-queue loop. This loop handles transform discovery (Base64, URL-percent) and recursive scanning of decoded output. The design choice to use a work queue rather than recursive function calls is motivated by two concerns: stack depth (nested transforms could overflow the call stack for deeply encoded content) and budget enforcement (a loop with explicit counters is trivial to bound, while recursive calls require passing budgets through every call frame). From `core.rs`:

```rust
scratch.work_q.push(WorkItem::scan_root());

let max_decode_bytes = self.tuning.max_total_decode_output_bytes;
let max_work = self.tuning.max_work_items;
let max_depth = self.tuning.max_transform_depth;
while scratch.work_head < scratch.work_q.len() {
    if scratch.total_decode_output_bytes >= max_decode_bytes {
        break;
    }
    let item = unsafe { *scratch.work_q.get_unchecked(scratch.work_head) };
    scratch.work_head += 1;
```

The work-queue loop dequeues items via cursor advance rather than `Vec::remove(0)`. The `work_head` index increments monotonically; the `WorkItem` is read by copy from the current slot without writing a default back. This avoids O(n) shifting per dequeue and keeps the loop body free of allocation or memory movement overhead. The work queue only grows (via `push`) during the loop body, never shrinks, so `work_head < work_q.len()` is a stable loop invariant.

The `WorkItem` type is a flat 40-byte `Copy` struct that replaces recursion with a FIFO queue. Two logical variants exist, distinguished by a flag bit:

- **ScanBuf**: validate regexes in prefilter windows for a buffer (root or decoded), then discover transform spans and enqueue `DecodeSpan` items.
- **DecodeSpan**: decode an encoded span (Base64 or URL-percent), write the output to the decode slab, then enqueue a `ScanBuf` for the decoded output.

This breadth-first traversal makes budget enforcement trivial. Three budgets are checked per iteration:

| Budget | Tuning field | Purpose |
|--------|-------------|---------|
| Decode bytes | `max_total_decode_output_bytes` | Caps total decoded output across all transforms |
| Work items | `max_work_items` | Caps the number of enqueued decode/scan items |
| Depth | `max_transform_depth` | Caps nesting (e.g., Base64 inside URL-percent) |

Tuning values are hoisted into local variables before the loop. Although `self` is `&Engine` (immutable), LLVM cannot prove that `&mut ScanScratch` does not alias `self.tuning` fields through complex call chains. Locals let the register allocator hold these values in registers across iterations.

### 6.1 The ScanBuf Arm

The ScanBuf arm resolves the buffer reference -- either the caller-owned root buffer or a slab-backed decoded buffer -- using raw pointers to avoid holding borrows on `scratch.slab` while passing `scratch` mutably into `scan_rules_on_buffer`. The aliasing safety argument is:

1. The slab is pre-allocated and never reallocated during a scan, so the pointer remains valid.
2. `scan_rules_on_buffer` writes only to scratch output buffers (findings, hit accumulators), never to the slab region backing the current buffer.
3. The root buffer is caller-owned and immutable for the scan duration.

After rule validation, the arm discovers transform spans. The engine selects which transforms to run based on whether findings have already been found in this buffer:

```rust
let (non_base64_tidxs, base64_tidxs) =
    self.scanbuf_transform_buckets(found_any_in_this_buf);
```

When no findings exist yet, all active transforms run (`Active` + `Always` modes). When findings have been found, only `TransformMode::Always` transforms run -- `IfNoFindingsInThisBuffer` transforms are skipped because the raw prefilter already discovered secrets. The rationale: if secrets are visible in raw bytes, investing CPU cycles in decoding transforms adds marginal value. The `IfNoFindingsInThisBuffer` mode captures the common case where a Base64-encoded secret is the *only* secret in a chunk -- when nothing was found raw, it is worth decoding to look for encoded secrets.

The transform buckets are pre-computed during engine construction. Four vectors partition transform indices by `(mode, id)`:

```rust
scanbuf_transform_idxs_active_non_base64: Vec<usize>,
scanbuf_transform_idxs_active_base64: Vec<usize>,
scanbuf_transform_idxs_always_non_base64: Vec<usize>,
scanbuf_transform_idxs_always_base64: Vec<usize>,
```

The Base64/non-Base64 split exists so the caller can apply the encoded-space pre-gate (`b64_gate`) only to Base64 spans without branching per-transform in the span loop. `Disabled` transforms are excluded at construction time -- they never appear in any bucket.

For each qualifying transform, `find_spans_into` discovers encoded spans in the buffer. Each span is bounded by `max_spans_per_buffer` to prevent a single transform from monopolizing the work budget. Base64 spans receive additional handling: the YARA pre-gate checks whether the encoded bytes could possibly produce any anchor after decoding, and four alignment shifts (0-3) cover every possible quantum boundary. Base64 encodes in 4-character quanta, and a span extracted from raw bytes may not start on a quantum boundary relative to the original encoding. By trying all four shifts, the pipeline covers every alignment without knowing which is correct. Duplicate start offsets (which can occur when whitespace collapses shifts) are deduplicated.

### 6.2 The DecodeSpan Arm

The DecodeSpan arm resolves the encoded span, checks the decode-bytes budget, and either streams the decode through `decode_stream_and_scan` (when a Vectorscan stream DB is available) or falls back to `decode_span_fallback` (full-decode, then scan). The stream path interleaves decoding with pattern matching, avoiding the need to buffer the entire decoded output before scanning.

Buffer references use the same raw-pointer technique as the ScanBuf arm. The encoded span may reference either the root buffer (`is_slab == false`) or the decode slab (`is_slab == true`). The slab case arises during nested transforms -- for example, a URL-percent span discovered inside a Base64-decoded buffer. The `EncRef` type encodes this distinction:

```rust
pub(super) struct EncRef {
    pub(super) lo: u32,
    pub(super) hi: u32,
    pub(super) is_slab: bool,
}
```

A `root_hint_maps_encoded` flag tracks whether the root hint exactly covers the encoded bytes being decoded. When true, decoded matches can be mapped back to precise root-buffer positions for deduplication. When false (nested transforms or mismatched hints), the pipeline keeps coarse root-hint windows to avoid fabricating exact offsets from partial context.

The root-span mapping context (`root_span_map_ctx`) is built in the ScanBuf arm for each work item that has a transform and an encoded-span reference. It translates decoded-space offsets back to root-buffer coordinates, ensuring that findings from nested transforms (e.g., a secret found inside a URL-percent span that was itself inside a Base64 span) report offsets into the original input, not intermediate decoded buffers. The context is only valid when the encoded span maps 1:1 with the root hint -- either directly from root, or through a slab where lengths match.

### 6.3 WorkItem Layout

The `WorkItem` type is a 40-byte flat struct designed for cache efficiency in the hot work-queue loop. The layout from `work_items.rs`:

```rust
pub(super) struct WorkItem {
    flags: u8,
    depth: u8,
    transform_idx: u16,
    step_id: StepId,
    buf_lo: u32,
    buf_hi: u32,
    enc_lo: u32,
    enc_hi: u32,
    root_hint_lo: u64,
    root_hint_hi: u64,
}

const _: () = assert!(std::mem::size_of::<WorkItem>() <= 40);
```

Sentinel values (`u32::MAX`, `u64::MAX`) replace `Option` wrappers and enum discriminants. A flags byte encodes six boolean properties via individual bits: `is_decode_span`, `buf_is_slab`, `has_enc_ref`, `enc_is_slab`, `has_root_hint`, and `has_transform_idx`. This replaces a former 104-byte enum+nested-enum layout with a flat, `Copy` representation. The work-queue loop dequeues via bitwise copy without writing a default back to the slot -- saving a 40-byte store per iteration.

## 7. The Pipeline as a Funnel

The full pipeline forms a narrowing funnel. Each stage reduces the bytes that the next stage must examine:

```text
  Root buffer (16,384 bytes)
           |
           v
  Vectorscan prefilter
  [touched pairs: 3 of 2,541]
           |
           v
  Sort + merge + coalesce
  [windows: 5 merged to 3]
           |
           v
  Two-phase confirm (optional)
  [windows: 3 confirmed to 2]
           |
           v
  Regex validation (1,200 bytes)
           |
           v
  Transform discovery + decode
           |
           v
  Recursive scan of decoded output
```

At each stage, the engine trades a cheap operation (bitmap check, memmem scan, sort) for an exponential reduction in downstream work. The prefilter reduces 2,541 pairs to 3. Merge reduces 5 windows to 3. Two-phase confirm reduces 3 to 2. The regex runs on 1,200 bytes instead of 13.9 million. This multiplicative narrowing is what makes scanning 847 rules against terabytes of data viable on commodity hardware.

The narrowing ratios vary by workload. Configuration files with many key-value pairs may touch more rules. Binary files with embedded text may trigger more UTF-16 variants. The pipeline handles both extremes: the zero-hit bypass exits in microseconds for chunks with no matches, while the pressure-coalesce mechanism bounds pathological cases to a constant number of regex passes regardless of hit density.

### 7.1 Chunk Overlap and Deduplication

One detail spans the boundary between the scan pipeline and the caller: chunk overlap. Matches that straddle a chunk boundary must appear entirely within at least one chunk. The engine computes the required overlap from rule radii and prefilter pattern widths:

```rust
pub fn required_overlap(&self) -> usize {
    self.max_window_diameter_bytes
        .saturating_add(self.max_prefilter_width.saturating_sub(1))
}
```

`max_window_diameter` is the largest validation window (2 x radius x variant scale) across all rules. `max_prefilter_width - 1` accounts for the widest prefilter pattern straddling the boundary. When scanning overlapping chunks, the caller invokes `drop_prefix_findings` with the new start offset to suppress duplicates from the overlap prefix. The `drop_hint_end` field on each finding records the absolute byte offset past which the finding should be kept, enabling precise deduplication at chunk boundaries.

## Summary

`scan_chunk_into` orchestrates a seven-step pipeline that transforms an unbounded scanning problem into a bounded one. The Vectorscan prefilter selects relevant (rule, variant) pairs. Window merge and coalesce bound the number of regex passes. Two-phase confirmation rejects noisy prefilter hits cheaply. The work-queue loop replaces recursion with breadth-first traversal, enforcing decode-byte, work-item, and depth budgets per iteration. Transform discovery is gated by finding-aware mode selection and encoded-space pre-gates. The one-shot prefilter flag avoids redundant Vectorscan scans, and the chunk overlap mechanism ensures correctness at chunk boundaries.

Chapter 5 narrows the focus to the single most complex function in the pipeline: `run_rule_on_window`. Where this chapter traced the path from raw bytes to candidate windows, the next chapter traces the seventeen validation gates that a candidate window must survive before it becomes a finding.
