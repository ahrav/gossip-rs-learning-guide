# The Allocation Budget -- Memory Architecture for Zero-Copy Scanning

*A scanner worker processes a 3.8 MB JavaScript bundle containing 14 base64-encoded credential blocks. The engine decodes each block into a child buffer, scans the child for anchors, and enqueues the decoded bytes into the work queue. The first decode produces 2.1 MB of output. The work queue -- backed by a standard `Vec<WorkItem>` -- pushes the item and triggers a reallocation from 16 to 32 entries, copying 640 bytes of existing items to a new heap address. Twelve more decodes follow, each triggering its own reallocation as the queue grows. Meanwhile, the decode slab -- also a `Vec<u8>` -- reallocates from 4 MB to 8 MB mid-scan to accommodate the cumulative output. A raw pointer from a previously enqueued work item still references the old slab address. The scan loop dereferences the stale pointer. On Linux the old page has been reclaimed by a concurrent thread's allocation; the bytes it reads are a fragment of an HTTP response header from an unrelated scan. The engine feeds those bytes to the regex, which produces a spurious match on `ghp_7f3a2b`, and the scanner reports a GitHub personal access token inside a file that contains only minified React code.*

*The root cause is simple: the slab reallocated, but the work item's pointer did not update. Without a memory architecture that guarantees stability -- no reallocation during a scan, fixed capacities set before the first byte is read, explicit cache-line boundaries between hot and cold data -- every pointer held across a work-queue iteration is a use-after-free waiting to happen.*

---

The scanner engine processes millions of bytes per second per thread. At that throughput, a single heap allocation in the inner loop -- even one that completes in 50 nanoseconds -- adds measurable latency when multiplied across 100,000 chunks per file. The memory architecture addresses this with three design principles: pre-allocate all buffers before scanning begins, never reallocate during a scan, and separate hot-path data from cold-path data at cache-line boundaries.

This chapter examines the five core structures that implement these principles: `ScratchVec` (the page-aligned, fixed-capacity vector), `ScanScratch` (the per-scan mutable state with its `#[repr(C)]` hot/cold split), `DecodeSlab` (the monotonic append-only buffer whose stability invariant makes raw-pointer aliasing sound), `HitAccPool` (the raw-pointer hit accumulator), and `WorkItem` (the 40-byte flat entry that replaced a 104-byte enum).

## 1. ScratchVec -- Fixed Capacity, Page-Aligned, Never Reallocates

The fundamental building block is `ScratchVec<T>`, defined in `scratch_memory.rs`. It is a `Vec`-like container with one critical difference: its capacity is fixed at construction time and never grows.

```rust
pub struct ScratchVec<T> {
    ptr: NonNull<MaybeUninit<T>>,
    len: u32,
    cap: u32,
}
```

On 64-bit targets, the struct occupies exactly 16 bytes (one pointer plus two `u32` fields). This is 8 bytes smaller than `Vec<T>` (which stores `len` and `cap` as `usize`), and the `u32` fields are sufficient because no single scratch buffer holds more than 4 billion elements.

```rust
#[cfg(target_pointer_width = "64")]
const _: () = assert!(size_of::<ScratchVec<u8>>() == 16);
```

### 1.1 Page-Aligned Allocation

All `ScratchVec` allocations are page-aligned to a minimum of 4 KiB:

```rust
const PAGE_SIZE_MIN: usize = 4096;

pub fn with_capacity(cap: usize) -> Result<Self, ScratchMemoryError> {
    // ...
    let align = PAGE_SIZE_MIN.max(align_of::<T>());
    let layout =
        Layout::from_size_align(size, align).map_err(|_| ScratchMemoryError::InvalidLayout)?;

    let raw = unsafe { alloc(layout) };
    let ptr = NonNull::new(raw).ok_or(ScratchMemoryError::OutOfMemory)?;
    // ...
}
```

Page alignment serves two purposes. First, it keeps buffers SIMD-friendly -- the character-class distribution gate uses SIMD byte classification, and aligned loads avoid cross-page penalties. Second, it makes alignment predictable: every `ScratchVec` starts at the same alignment boundary regardless of element type, which simplifies reasoning about cache behavior.

### 1.2 Capacity Overruns Are Bugs

Unlike `Vec`, which silently reallocates on overflow, `ScratchVec` treats capacity overruns as logic errors:

```rust
pub fn push(&mut self, value: T) {
    debug_assert!(self.len < self.cap, "scratch vec capacity exceeded");
    unsafe {
        self.ptr
            .as_ptr()
            .add(self.len())
            .write(MaybeUninit::new(value));
    }
    self.len += 1;
}
```

The `debug_assert` fires in debug builds; in release builds, a push beyond capacity is undefined behavior. This contract is intentional. The engine pre-calculates exact capacities from the `Tuning` struct at construction time. If a capacity is exceeded, the tuning parameters are misconfigured -- that is a programming error, not a runtime condition to handle gracefully.

### 1.3 Ownership and Thread Safety

`ScratchVec` implements `Send` (when `T: Send`) but deliberately does not implement `Sync`:

```rust
unsafe impl<T: Send> Send for ScratchVec<T> {}
```

The type is designed for single-threaded scratch use within one scan. Omitting `Sync` encodes that intent at the type level -- there is no need for shared references across threads, and the absence of `Sync` prevents accidental sharing.

## 2. ScanScratch -- The Per-Scan Mutable State

`ScanScratch` is the primary allocation amortization vehicle for the engine. It owns every mutable buffer used during a scan: the findings output, the work queue, the hit accumulators, the decode slab, the ring buffer, and the Vectorscan scratch spaces. From `engine/scratch.rs`:

```rust
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
    pub(super) seen_findings_scan: FixedSet128,
    pub(super) total_decode_output_bytes: usize,
    pub(super) work_items_enqueued: usize,
    pub(super) capture_locs: Vec<Option<CaptureLocations>>,
    pub(super) stream_hit_counts: Vec<u32>,
    pub(super) stream_hit_touched: ScratchVec<u32>,
    pub(super) hit_acc_pool: HitAccPool,
    pub(super) touched_pairs: ScratchVec<u32>,
    pub(super) windows: ScratchVec<SpanU32>,
    pub(super) expanded: ScratchVec<SpanU32>,
    pub(super) spans: ScratchVec<SpanU32>,
    pub(super) step_arena: StepArena,
    pub(super) utf16_buf: ScratchVec<u8>,
    pub(super) steps_buf: ScratchVec<DecodeStep>,
    // --------------- Cache-line boundary ----------------
    _cold_boundary: CachelineBoundary,

    // ---------------- Cold / conditional region ----------------
    pub(super) slab: DecodeSlab,
    pub(super) seen: FixedSet128,
    pub(super) seen_findings: FixedSet128,
    pub(super) decode_ring: ByteRing,
    // ... (additional cold fields)
}
```

### 2.1 The Cache-Line Split

The `#[repr(C)]` attribute preserves declared field order. A zero-sized alignment marker forces a 64-byte cache-line boundary between the hot and cold regions:

```rust
#[repr(align(64))]
struct CachelineBoundary {
    _pad: [u8; 0],
}
```

The `[u8; 0]` body contributes no bytes to the struct size -- only the `#[repr(align(64))]` attribute matters. The compiler satisfies it by inserting padding *before* this field in the `#[repr(C)]` layout.

**Hot region** (before `_cold_boundary`): fields touched on every scan chunk -- output buffers, the work queue, hit accumulators, per-rule capture locations, and merged-window buffers. These dominate L1/L2 cache residency during the inner scan loop.

**Cold region** (after `_cold_boundary`): fields touched only when a transform fires (decode slab, ring buffer, pending windows), when emitting findings for cross-chunk deduplication, or for Vectorscan scratch management. Separating them avoids polluting the hot cache lines when transforms are inactive -- the common case for many file types.

**Per-chunk region** (after the Vectorscan scratch fields): fields written once per chunk or only under `perf-stats` / `debug_assertions`. This region includes safelist/offline suppression counters (`safelist_suppressed`, `offline_suppressed`, etc.), the one-shot prefilter flags (`root_prefilter_done`, `root_prefilter_saw_utf16`), overlap metadata (`chunk_overlap_backscan`), and the `capacity_validated` sentinel. Placing these after the cold region keeps them off cache lines touched by the inner scan loop and the transform decode path.

### 2.2 The Parallel-Array Invariant

Three arrays -- `out`, `norm_hash`, and `drop_hint_end` -- are kept length-aligned at all times:

```rust
/// **Parallel-array invariant**: `out`, `norm_hash`, and `drop_hint_end`
/// are kept length-aligned at all times. Every push, truncation, or drain
/// must maintain this lock-step relationship. Violating it corrupts finding
/// deduplication and materialization.
```

When a finding is emitted, all three arrays receive a push in the same code block. When findings are dropped (prefix deduplication, chunk-boundary trimming), all three are truncated to the same length. This lock-step relationship is enforced by convention, not by a wrapper type -- the engine's inner loop is too performance-sensitive for an abstraction layer.

### 2.3 Lifecycle

The `ScanScratch` lifecycle has four phases:

1. **Allocate** via `ScanScratch::new` -- pre-sizes all buffers to the engine's tuning parameters.
2. **Reset** per-chunk via `reset_for_scan` (or `reset_for_scan_after_prefilter` to preserve prefilter results) -- clears per-scan state while preserving allocations.
3. **Scan** -- the engine populates findings, work items, and decode steps.
4. **Drain** via `drain_findings` -- extracts results.

The `reset_for_scan` method demonstrates the sparse-reset optimization for stream hit counts:

```rust
// Sparse reset: zero only the stream-hit counters that were
// incremented (O(touched) instead of O(rules × 3)).
for idx in self.stream_hit_touched.drain() {
    let slot = idx as usize;
    if let Some(hit) = self.stream_hit_counts.get_mut(slot) {
        *hit = 0;
    }
}
```

With 847 rules and 3 variants each, a full reset would write 2,541 zero values. The sparse reset writes only the counters that were actually incremented -- typically a few dozen for a chunk with focused anchor hits.

### 2.4 Capacity Validation -- Once and Only Once

Since `Engine` is immutable after construction, capacity checks are idempotent:

```rust
pub(super) fn ensure_capacity(&mut self, engine: &Engine) {
    if self.capacity_validated {
        return;
    }
    // ... rebind Vectorscan scratch, check pool sizes, etc.
    self.capacity_validated = true;
}
```

The first `reset_for_scan` validates all capacities and potentially reallocates scratch buffers to match the engine's current tuning. Subsequent calls skip the validation block entirely, reducing per-chunk overhead to zero for the common case.

## 3. DecodeSlab -- The Stability Invariant

The `DecodeSlab` in `engine/decode_state.rs` is a contiguous byte buffer for decoded transform output:

```rust
pub(super) struct DecodeSlab {
    pub(super) buf: Vec<u8>,
    pub(super) limit: usize,
}
```

It is a monotonic append-only buffer with a hard capacity. Decoders append into the slab and receive a `Range<usize>` back. Work items carry those ranges instead of owning new allocations. The slab never reallocates during a scan -- its capacity equals the global decode budget, pre-allocated at construction time.

This non-reallocation property is the **stability invariant**. The scan loop in `core.rs` constructs raw `&[u8]` slices from slab ranges via `unsafe` slice construction. If the slab reallocated, those slices would point to freed memory. The invariant makes this `unsafe` code sound:

```rust
impl DecodeSlab {
    pub(super) fn with_limit(limit: usize) -> Self {
        let buf = Vec::with_capacity(limit);
        Self { buf, limit }
    }
}
```

The `Vec::with_capacity(limit)` call ensures the backing allocation is large enough for the entire decode budget. `buf.extend_from_slice(chunk)` within `append_stream_decode` never triggers reallocation because the total output is bounded by `limit`.

### 3.1 Rollback Semantics

`append_stream_decode` enforces three budgets per append: per-transform output, global decode output, and slab capacity. On any budget violation, the slab is truncated back to its pre-call length:

```rust
pub(super) fn append_stream_decode(
    &mut self,
    tc: &TransformConfig,
    input: &[u8],
    max_out: usize,
    ctx_total_decode_output_bytes: &mut usize,
    global_limit: usize,
) -> Result<Range<usize>, ()> {
    let start_len = self.buf.len();
    let start_ctx = *ctx_total_decode_output_bytes;
    // ...
    if res.is_err() || truncated || local_out == 0 || local_out > max_out {
        self.buf.truncate(start_len);
        *ctx_total_decode_output_bytes = start_ctx;
        return Err(());
    }
    Ok(start_len..(start_len + local_out))
}
```

This all-or-nothing approach ensures downstream code never sees a partially decoded buffer. Either the full decode succeeds and the returned `Range` points at valid output, or `Err(())` is returned and the slab is unchanged.

## 4. StepArena -- Compact Decode Provenance

Findings discovered in decoded buffers must carry a chain of decode steps so consumers can reconstruct the original encoded bytes. Storing a full `Vec<DecodeStep>` per finding would allocate and clone on every match. The `StepArena` in `engine/decode_state.rs` solves this with a parent-linked arena:

```rust
pub(super) struct StepArena {
    pub(super) nodes: ScratchVec<StepNode>,
}
```

Each node is 16 bytes -- a parent `StepId` plus a `CompactDecodeStep`:

```rust
struct CompactDecodeStep {
    tag_and_idx: u32,
    span_start: u32,
    span_end: u32,
}

pub(super) struct StepNode {
    pub(super) parent: StepId,
    step: CompactDecodeStep,
}

const _: () = assert!(std::mem::size_of::<CompactDecodeStep>() == 12);
const _: () = assert!(std::mem::size_of::<StepNode>() == 16);
```

The public `DecodeStep` uses `usize` spans and an enum with `Range<usize>` fields, costing 32 bytes on 64-bit targets. The compact form uses `u32` spans (safe because buffers are capped at `u32::MAX`) and packs the discriminant plus payload index into a single `u32`. The bit layout of `tag_and_idx`: bit 31 is 0 for Transform and 1 for Utf16Window; bits 30..0 hold the transform index or endianness ordinal.

Findings store a single `StepId` (a `u32` arena index). Materialization traces the parent chain and reverses it:

```rust
pub(super) fn materialize(&self, mut id: StepId, out: &mut ScratchVec<DecodeStep>) {
    out.clear();
    while id != STEP_ROOT {
        let cur = id;
        let node = &self.nodes[cur.0 as usize];
        out.push(node.step.to_decode_step());
        id = node.parent;
    }
    let len = out.len();
    for i in 0..len / 2 {
        out.as_mut_slice().swap(i, len - 1 - i);
    }
}
```

The arena is append-only during a scan and reset between chunks. `StepId` values are only valid while the arena is alive and not reset -- the sentinel `STEP_ROOT` (`StepId(u32::MAX)`) terminates the chain.

## 5. HitAccPool -- Raw-Pointer Hit Accumulation

The `HitAccPool` in `engine/hit_pool.rs` accumulates anchor hit windows across all (rule, variant) pairs. It is the most performance-critical data structure in the prefilter path:

```rust
pub(super) struct HitAccPool {
    max_hits: u32,
    pair_count: u32,
    touched_word_count: u32,
    _pad: u32,
    pair_meta: *mut PairMeta,
    windows: *mut SpanU32,
    coalesced: *mut SpanU32,
    touched_words: *mut u64,
}
```

All internal arrays are raw pointers. The struct stores them instead of `Vec` to eliminate bounds-check loads on the hot path. Per-pair metadata is collocated into a 4-byte `PairMeta`:

```rust
#[repr(C)]
struct PairMeta {
    len: u16,
    coalesced: u8,
    _pad: u8,
}

const _: () = assert!(std::mem::size_of::<PairMeta>() == 4);
```

Packing `len` and `coalesced` into 4 bytes means a single 32-bit load gives both fields. Sixteen consecutive pairs fit in one cache line.

### 5.1 The Coalesce Overflow Path

Storage is fixed-stride: `windows` is laid out as `pair * max_hits + idx`. Each pair starts as an append-only list. Once the hit count exceeds the cap, the pool switches to a single "coalesced" window covering the union of all hits:

```rust
#[cold]
#[inline(never)]
fn coalesce_overflow(&mut self, pair: usize, span: SpanU32) {
    let max_hits = self.max_hits as usize;
    let meta = unsafe { &mut *self.pair_meta.add(pair) };
    let len = meta.len as usize;
    let base = pair * max_hits;

    let mut lo = span.start;
    let mut hi = span.end;
    let mut min_anchor = span.anchor_hint;

    let windows = unsafe { std::slice::from_raw_parts(self.windows.add(base), len) };
    for s in windows {
        lo = lo.min(s.start);
        hi = hi.max(s.end);
        min_anchor = min_anchor.min(s.anchor_hint);
    }

    unsafe {
        *self.coalesced.add(pair) = SpanU32 {
            start: lo,
            end: hi,
            anchor_hint: min_anchor,
        };
    }
    meta.coalesced = 1;
    meta.len = 0;
}
```

The `#[cold]` and `#[inline(never)]` annotations are deliberate: the overflow path is extracted to keep the fast path compact and branch-predictor friendly. The branch predictor learns that the coalesce check almost never fires, and the compiler avoids inlining the cold code into the hot loop.

### 5.2 Sparse Reset

The pool uses a bitset (`touched_words`) to track which pairs were modified during a scan. Reset clears only the touched bits:

```rust
pub(super) fn reset_touched(&mut self, touched_pairs: &[u32]) {
    let words = self.touched_words;
    for &p in touched_pairs {
        let idx = p as usize;
        unsafe {
            let word = words.add(idx / 64);
            *word &= !(1u64 << (idx % 64));
        }
    }
}
```

This is O(touched), not O(pair_count). With 847 rules and 3 variants each (2,541 pairs), a full reset would write 40 cache lines of zeros. A scan that touches 30 pairs writes fewer than one cache line.

## 6. WorkItem -- The 40-Byte Flat Entry

The work queue drives breadth-first traversal of root and decoded buffers. Each entry is a `WorkItem`, defined in `engine/work_items.rs`:

```rust
#[derive(Clone, Copy)]
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

The previous representation used a nested enum (`ScanBuf`/`DecodeSpan` with nested `BufRef`/`EncRef` enums) that occupied 104 bytes. The flat representation packs everything into 40 bytes using sentinel values (`NONE_U32 = u32::MAX`, `NONE_U64 = u64::MAX`) for absent optional fields and a flags byte for variant discrimination:

```text
offset  field            sentinel meaning
------  --------------   -----------------------
  0     flags: u8        bit 0 = is_decode_span
                         bit 1 = buf_is_slab
                         bit 2 = has_enc_ref
                         bit 3 = enc_is_slab
                         bit 4 = has_root_hint
                         bit 5 = has_transform_idx
  1     depth: u8        max 7 (MAX_DECODE_STEPS-1)
  2     transform_idx: u16
  4     step_id: u32
  8     buf_lo: u32      NONE_U32 for Root buf
 12     buf_hi: u32      NONE_U32 for Root buf
 16     enc_lo: u32      NONE_U32 when absent
 20     enc_hi: u32      NONE_U32 when absent
 24     root_hint_lo: u64   NONE_U64 when absent
 32     root_hint_hi: u64   NONE_U64 when absent
```

Two logical variants exist, distinguished by bit 0 of `flags`:

- **ScanBuf** (`!is_decode_span`): scan a buffer (root or slab). `buf_lo`/`buf_hi` specify the slab range (or `NONE_U32` for root).
- **DecodeSpan** (`is_decode_span`): decode an encoded span, then scan the result.

The `WorkItem` is `Copy` and has no heap pointers and no `Drop`. The work-queue loop dequeues via bitwise copy without writing a default back to the slot.

### 6.1 The SpanU32 -- Compact Window Representation

Spans throughout the engine use `SpanU32` from `engine/hit_pool.rs`:

```rust
pub(super) struct SpanU32 {
    pub(super) start: u32,
    pub(super) end: u32,
    pub(super) anchor_hint: u32,
}
```

The `anchor_hint` field carries Vectorscan's `from` match offset, clamped to `[start, end]`. This hint allows the regex to start searching near the anchor position instead of at the window start, reducing redundant prefix scanning on windows with wide radii.

## 7. FindingRec -- The Compact Finding Record

Findings are recorded during scanning as `FindingRec` entries, defined in `api.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct FindingRec {
    pub file_id: FileId,
    pub rule_id: u32,
    pub span_start: u32,
    pub span_end: u32,
    pub root_hint_start: u64,
    pub root_hint_end: u64,
    pub dedupe_with_span: bool,
    pub step_id: StepId,
    pub confidence_score: i8,
}
```

A compile-time guard keeps the record compact:

```rust
const _: () = assert!(std::mem::size_of::<FindingRec>() <= 48);
```

48 bytes is three cache-line halves on typical x86-64 hardware. The record uses `u32` for span offsets (callers must chunk inputs to fit in `u32::MAX` bytes) and `u64` for root hint offsets (files can exceed 4 GB). The `step_id` is a lightweight arena reference -- 4 bytes instead of the variable-length `Vec<DecodeStep>` that would be needed for inline provenance.

`FindingRec` is later materialized into the full `Finding` struct by expanding the decode-step chain. The `confidence_score` field is an additive `i8` computed from per-finding evidence signals, with the Phase 1 range of 0--10 well within `i8` bounds. Findings below the rule's `min_confidence` threshold are suppressed before being written to the output buffer.

## 8. The Entropy Scratch -- Conditional Allocation

Not every engine needs entropy gating. Engines with no entropy gates skip the 1 KiB histogram allocation entirely:

```rust
pub(super) entropy_scratch: Option<Box<EntropyScratch>>,
```

```rust
pub(super) struct EntropyScratch {
    pub(super) counts: [u32; 256],
}
```

Allocation follows a two-phase strategy:

**Phase 1 -- Eager allocation at construction.** `ScanScratch::new` checks whether the engine has any entropy gates and pre-allocates the box if so:

```rust
entropy_scratch: if has_entropy_gates {
    Some(Box::new(EntropyScratch::new()))
} else {
    None
},
```

**Phase 2 -- Lazy fallback via `ensure_entropy_scratch`.** Code paths that reach an entropy gate when the box is `None` (e.g., a scratch instance originally built for an engine without entropy gates) allocate on first use:

```rust
pub(super) fn ensure_entropy_scratch(&mut self) -> &mut EntropyScratch {
    self.entropy_scratch
        .get_or_insert_with(|| Box::new(EntropyScratch::new()))
        .as_mut()
}
```

This two-phase approach means engines without entropy gates pay zero heap cost, while engines with entropy gates avoid a branch on every entropy check after construction. The lazy fallback covers the edge case where a `ScanScratch` outlives its original engine and is reused with one that has entropy gates.

The histogram uses a full `memset` reset (O(256) via `counts = [0u32; 256]`) after each entropy check rather than tracking which bins were touched. On modern ARM (stp loop), zeroing 1 KiB takes 2--3 nanoseconds. The constant cost eliminates the per-byte branch that a "touched list" approach would require in the histogram loop, which runs N times (N = input length) while the reset runs once.

## 9. Dedup Keys -- Aligned for AEGIS-128L

Finding deduplication uses a packed 32-byte key:

```rust
#[repr(C)]
#[derive(Clone, Copy, bytemuck::Pod, bytemuck::Zeroable)]
struct DedupKey {
    file_id: u32,
    rule_id_with_variant: u32,
    span_start: u32,
    span_end: u32,
    root_hint_start: u64,
    root_hint_end: u64,
}

const _: () = assert!(std::mem::size_of::<DedupKey>() == 32);
```

The 32-byte size is aligned to the AEGIS-128L absorption rate (2 x 128-bit AES blocks). The `hash128` function processes the key in a single absorption step with no trailing partial-block handling, which is measurably faster than the previous 33-byte packed layout. The rule ID and variant discriminator are packed into a single `u32`: lower 24 bits hold the rule ID, upper 8 bits hold the variant (non-UTF16=0, Utf16Le=1, Utf16Be=2).

The finding dedup budget is intentionally larger than the emission cap. From `scratch.rs`:

```rust
const FINDING_DEDUPE_MULTIPLIER: usize = 32;
```

Dedupe sets track more keys than the emit cap because many candidates can be observed (and intentionally dropped) before the chunk cap is reached. Keeping this budget separate from `max_findings` preserves cross-chunk duplicate suppression under high candidate pressure. The dedupe set itself is a `FixedSet128` -- a Bloom-style probabilistic filter sized as a power of two for efficient modular hashing.

## 10. Vectorscan Scratch Lifecycle

Vectorscan requires each scanning thread to have its own scratch memory, separate from the database. `ScanScratch` owns up to five Vectorscan scratch instances:

```rust
pub(super) vs_scratch: Option<VsScratch>,
pub(super) vs_utf16_scratch: Option<VsScratch>,
pub(super) vs_utf16_stream_scratch: Option<VsScratch>,
pub(super) vs_stream_scratch: Option<VsScratch>,
pub(super) vs_gate_scratch: Option<VsScratch>,
```

Each corresponds to one of the five Vectorscan databases that may exist on the `Engine`. When the engine is first used with a scratch instance, `ensure_capacity` rebinds each scratch to the current database pointer:

```rust
macro_rules! rebind_vs_scratch {
    ($db_field:expr, $scratch_field:expr, $label:literal) => {
        match $db_field.as_ref() {
            Some(db) => {
                let need_alloc = match $scratch_field.as_ref() {
                    Some(s) => s.bound_db_ptr() != db.db_ptr(),
                    None => true,
                };
                if need_alloc {
                    $scratch_field = Some(db.alloc_scratch().expect(concat!(
                        "vectorscan ",
                        $label,
                        " scratch allocation failed"
                    )));
                }
            }
            None => {
                $scratch_field = None;
            }
        }
    };
}
```

The `bound_db_ptr()` check detects whether the scratch is still bound to the correct database. Since `Engine` is immutable after construction, this check succeeds on every call after the first -- the rebind is a one-time cost, not per-scan.

## 11. RootSpanMapCtx -- Raw Pointers for Coordinate Mapping

During transform scans, findings are reported in decoded-byte coordinates. The `RootSpanMapCtx` in `scratch.rs` captures the encoded span being decoded so that decoded offsets can be translated back to root-buffer offsets:

```rust
pub(super) struct RootSpanMapCtx {
    tc: *const TransformConfig,
    encoded_ptr: *const u8,
    encoded_len: usize,
    root_start: usize,
    overlap_backscan: usize,
}
```

This type stores raw `*const` pointers to avoid lifetime entanglement with the engine and buffer references. Both pointers reference data owned by the `Engine` (immutable after construction) and the current scan buffer, which outlive the context. The scan loop clears `root_span_map_ctx` to `None` after each buffer scan completes, ensuring no pointer is live when the scratch is moved between threads.

The `unsafe impl Send` is sound for three reasons documented in the source: both pointers reference Engine-owned data that is immutable and outlives all scratch instances; the context is only populated during a scan and cleared after each buffer scan completes; and the same pattern is used by `RealEngineScratch` in the scheduler layer.

The mapping itself translates a decoded-byte span back to absolute root-buffer coordinates via `map_decoded_offset`, which accounts for the encoding ratio of the transform (e.g., URL-percent-encoded bytes decode at a 3:1 ratio for `%XX` sequences, 1:1 for passthrough bytes).

## 12. The Memory Budget Model

The engine's total memory footprint per worker thread is deterministic and calculable before the first scan. The budget breaks down into three categories:

**Compilation-time (shared).** The `Engine` struct is immutable and shared across all workers via `Arc<Engine>`. Its cost is proportional to the rule count and anchor pattern count: `rules_hot` (88 bytes per rule), `rules_cold` (56 bytes per rule), up to eight gate pools (variable, but bounded by rule count), five Vectorscan databases (sized by pattern complexity), and the base64 YARA pre-gate.

**Per-worker (fixed).** Each `ScanScratch` instance has a fixed memory envelope determined by the `Tuning` struct:

- `out`, `norm_hash`, `drop_hint_end`: `max_findings_per_chunk` entries each.
- `work_q`: `max_work_items + 1` entries (each 40 bytes).
- `slab`: `max_total_decode_output_bytes` bytes (the entire decode budget).
- `hit_acc_pool`: `rules * 3 * max_anchor_hits_per_rule_variant` windows.
- `seen_findings`: power-of-two hash set sized to `max_findings * 32`.
- Five Vectorscan scratch instances (one per database).

**Per-chunk (zero).** During steady-state scanning, no allocations occur. The `reset_for_scan` method clears logical lengths and pointers without calling `alloc` or `dealloc`. Capacity grows monotonically -- buffers may grow if the engine's tuning increases between scans, but they never shrink.

This three-tier model means an operator can predict the peak memory usage of a scanner deployment from the `Tuning` parameters and worker count alone, without profiling the actual input. A deployment with 8 workers, 847 rules, `max_findings_per_chunk = 1024`, and `max_total_decode_output_bytes = 16 MB` has a predictable ceiling: roughly 8 x (16 MB slab + ~2 MB scratch) plus one shared engine of ~50 MB.

## Summary

The scanner engine's memory architecture is built on five interlocking guarantees:

1. **ScratchVec** provides fixed-capacity, page-aligned storage that never reallocates. Capacity overruns are programming errors, not runtime conditions.
2. **ScanScratch** separates fields into three regions -- hot (inner scan loop), cold (transform decode, dedup), and per-chunk (suppression counters, prefilter flags, overlap metadata) -- at a 64-byte cache-line boundary using `#[repr(C)]` layout and a zero-sized alignment marker.
3. **DecodeSlab** pre-allocates the full decode budget and never grows. This stability invariant makes raw-pointer slice construction in the work-queue loop sound.
4. **HitAccPool** uses raw pointers and a collocated 4-byte `PairMeta` to eliminate bounds checks and achieve single-load access to per-pair metadata.
5. **WorkItem** packs two logical variants into a flat 40-byte `Copy` struct using sentinel values and a flags byte, down from 104 bytes in the enum representation.

Together, these structures ensure that the steady-state scan loop -- prefilter, window merge, regex validation, finding emission -- executes without touching the allocator. Allocations happen once at construction time, and the per-chunk cost is a `reset` that clears counters and pointers without freeing or acquiring memory.
