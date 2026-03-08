# Peeling Layers -- Transform Decoding and Deduplication

*An incident responder uploads a `.env` file containing `API_KEY=dG9rZW4tc2VjcmV0LTEyMzQ1`. That Base64 string encodes the literal `token-secret-12345`. The raw-buffer scan finds nothing -- the detector expects plaintext patterns, and the Base64 alphabet is just alphanumeric noise. The engine's transform pipeline decodes the span, and the detector fires on the decoded bytes at offset 0 of the decoded buffer. The finding's span says `0..20`. But the user needs to know where the secret lives in the original file, not in ephemeral decoded memory. Now layer a second encoding: the config value is URL-percent-encoded before Base64, so the raw buffer actually contains `API_KEY%3DdG9rZW4tc2VjcmV0LTEyMzQ1`. The `%3D` decodes to `=`, producing the Base64 payload, which in turn decodes to the secret. Two decode rounds, two coordinate systems, one finding that must point back to byte 8 of the original file. And when the file is scanned in overlapping 64 KiB chunks, the same encoded region appears in two consecutive chunks. Without deduplication, the same secret is reported twice. Without coordinate mapping, the user cannot locate it. This is the transform problem.*

---

Chapter 5 traced the seventeen-step validation cascade that a candidate window must survive. This chapter moves one level deeper: before validation can run at all, the scanner must find encoded payloads, decode them, track provenance through arbitrarily nested transforms, map decoded offsets back to the original file, and suppress the duplicate findings that chunked scanning inevitably produces. The machinery lives in three files: `transform.rs` (span detection and streaming decoders), `stream_decode.rs` (the streaming decode-and-scan loop), and `decode_state.rs` (provenance arena and decode slab).

## 1. Transform Pipeline Architecture

The transform subsystem follows a two-phase design: **find** then **decode**. The span finder is permissive -- it trades precision for cheap scanning, biasing toward recall. The decoder is strict -- it enforces encoding correctness and streams output in bounded chunks. This separation exists because most bytes in a typical buffer are not encoded at all. Running a full decoder over every byte would waste cycles; running a cheap span finder first narrows the input to regions worth decoding.

The engine supports two transforms: URL-percent encoding (RFC 3986 `%HH` escapes) and Base64 (standard and URL-safe alphabets). Each transform has four components:

1. **Span finder** (`find_url_spans_into`, `find_base64_spans_into`) -- single-pass scanner producing byte ranges.
2. **Streaming span detector** (`UrlSpanStream`, `Base64SpanStream`) -- stateful version for chunked input during streaming decode.
3. **Streaming decoder** (`stream_decode_url_percent`, `stream_decode_base64`) -- converts encoded bytes to decoded output in bounded chunks via a callback.
4. **Offset mapper** (`map_decoded_offset`) -- translates decoded-byte positions back to encoded-byte positions for coordinate reconstruction.

All four share a single 256-byte lookup table that classifies every byte value in one indexed load.

## 2. The BYTE_CLASS Lookup Table

Both URL and Base64 scanners need to classify bytes rapidly. A naive approach uses cascading `if`/`match` branches -- one for URL-ish characters, another for the Base64 alphabet, another for whitespace. Each branch is a pipeline stall waiting to happen. The codebase replaces all of them with a single `const`-constructed table.

From `transform.rs`:

```rust
const URLISH: u8 = 1 << 0;
const B64_CHAR: u8 = 1 << 1;
const B64_WS: u8 = 1 << 2;
const B64_WS_SPACE: u8 = 1 << 3;

static BYTE_CLASS: [u8; 256] = build_byte_class();
```

Each byte value maps to a bitmask that is the bitwise OR of all classes it belongs to. The letter `A` has both `URLISH` and `B64_CHAR` set. The byte `%` has `URLISH` only. A newline has `B64_WS` only. A scanner tests membership with a single load and mask:

```rust
let flags = BYTE_CLASS[b as usize];
let urlish = (flags & URLISH) != 0;
```

The table is 256 bytes -- four cache lines, fully resident in L1 after the first scan iteration. Because URL and Base64 scanners both index the same table, the data stays hot across interleaved scans. The `const fn build_byte_class()` evaluates at compile time, embedding the table in `.rodata` with no runtime initialization cost.

The Base64 decoder uses a complementary 256-byte table, `B64_DECODE_EX`, which maps each byte to its 6-bit decoded value (0-63), `B64_PAD` (64) for `=`, `B64_WS_SENTINEL` (`0xFE`) for whitespace, or `B64_INVALID` (`0xFF`) for everything else. This four-way classification in a single indexed load replaces what would otherwise be a branch-heavy chain of `if` checks in the decoder's inner loop.

## 3. URL-Percent Span Detection

The URL span finder scans for "URL-ish runs" -- contiguous sequences of RFC 3986 unreserved and reserved characters plus `%` and `+`. A run only produces a span if it contains at least one **trigger** byte (`%`, or `+` when `plus_to_space` is enabled). This prevents the scanner from decoding every plain word that happens to contain URL-safe characters.

From `transform.rs`, the streaming variant:

```rust
pub(super) struct UrlSpanStream {
    min_len: usize,
    max_len: usize,
    plus_to_space: bool,
    in_run: bool,
    start: u64,
    run_len: usize,
    triggers: usize,
    done: bool,
}
```

The scanner preserves state between chunks (`in_run`, `start`, `run_len`, `triggers`) so that a URL-ish run spanning a chunk boundary is not split into two separate spans. When a run exceeds `max_len`, it is split at the byte boundary to cap worst-case work. This split does not align to `%HH` triplet boundaries -- the trade-off is explicit in the module documentation. A `%HH` escape straddling the split point passes through the decoder unchanged, which is acceptable because the span finder biases toward recall and the decoder handles malformed escapes gracefully by passing them through verbatim.

The block-mode variant (`find_url_spans_into`) uses `memchr`/`memchr2` as a fast prefilter before entering the scan loop. If no `%` (or `+`) exists anywhere in the buffer, the function returns immediately without touching the per-byte classification table. This SIMD-accelerated check avoids the entire scan loop for buffers that contain no URL-encoded content -- the common case for most file types.

An additional fast-reject layer exists above the span finder: `transform_quick_trigger` returns `false` when the buffer cannot possibly contain an encoded payload, letting the engine skip the span scan entirely. For URL-percent, this is another `memchr` check; for Base64, it always returns `true` because the Base64 alphabet overlaps too heavily with normal text for a single-byte prefilter to be selective.

## 4. Base64 Span Detection and Alignment Shifts

Base64 detection is structurally similar but handles two additional complexities: whitespace tolerance and padding-terminated spans.

From `transform.rs`:

```rust
pub(super) struct Base64SpanStream {
    min_chars: usize,
    max_len: usize,
    allow_space_ws: bool,
    in_run: bool,
    pad_seen: bool,
    start: u64,
    run_len: usize,
    b64_chars: usize,
    have_b64: bool,
    last_b64: u64,
    done: bool,
}
```

**Whitespace tolerance.** Base64 content in the wild often contains line breaks (`\r\n` in PEM files) or spaces (in JSON values). The scanner allows `\n`, `\r`, `\t` unconditionally and `' '` when `allow_space_ws` is set. Whitespace contributes to `run_len` (for `max_len` enforcement) but not to `b64_chars` (for `min_chars` threshold). Spans are trimmed to the last Base64 alphabet byte so trailing whitespace is excluded from the span range.

**Padding state machine.** When `=` is seen, the scanner enters padding-tail mode (`pad_seen = true`). In this mode, additional `=` characters extend the span (handling `==` split across chunks), whitespace is tolerated but does not advance the span end, and any non-pad Base64 character finalizes the current span and begins a fresh run without consuming the byte. This prevents `QUJD==QUJD` from merging into one span -- it correctly produces two. A test in `transform.rs` guards this:

```rust
stream.feed(b"QUJDRA=", 0, &mut on_span);
stream.feed(b"=X", 7, &mut on_span);
stream.finish(9, &mut on_span);

assert!(
    spans.iter().any(|&(lo, hi)| lo == 0 && hi == 8),
    "expected span 0..8, got {spans:?}"
);
```

The span `0..8` covers `QUJDRA==` and excludes `X`, which begins a new run.

**Alignment shifts.** Base64 decodes in 4-character quanta. When a span is detected inside a decoded stream (a nested transform), the span boundary may not align to a quantum boundary. Decoding from a misaligned position produces garbage. The engine handles this by trying up to four alignment offsets (shifts 0 through 3), each skipping that many Base64 alphabet characters before attempting decode. `base64_skip_chars` uses a fast path that processes four bytes at a time when all are `B64_CHAR`, falling back to a per-byte loop only when whitespace or non-Base64 bytes appear. Each shift that produces a valid start position and meets the `min_chars` threshold is enqueued as a separate decode span.

## 5. Streaming Decode and the Nine-Phase Loop

The streaming decoder enforces encoding correctness and emits decoded output in bounded chunks without materializing the full decoded buffer. The URL-percent decoder uses `memchr`/`memchr2` to skip literal prefixes in bulk, then processes consecutive `%HH` escapes in a tight loop. Output accumulates in a 16 KiB stack buffer (`STREAM_DECODE_CHUNK_BYTES`) and is flushed via callback when headroom is exhausted. Invalid escapes pass through verbatim -- the decoder is infallible.

The Base64 decoder uses a three-tier strategy. The SIMD fast path (aarch64 NEON or x86_64 SSSE3) processes 16 input bytes into 12 output bytes per iteration: a `classify_and_decode` function validates all 16 bytes and produces 6-bit values in a single vector operation, then `pack_16_to_12` merges the 6-bit values into 8-bit output via multiply-add intrinsics and a shuffle. On x86_64, this uses `_mm_maddubs_epi16` for pair merging (each pair of 6-bit values becomes one 12-bit value) and `_mm_madd_epi16` for quad merging (two 12-bit values become one 24-bit value). A final byte-shuffle (`_mm_shuffle_epi8`) extracts the three meaningful bytes from each 32-bit lane, producing the 12-byte output. The scalar 4-byte batch handles the tail using `B64_DECODE_EX` to classify four bytes at once; only when all four values are below `B64_PAD` does the fast path fire. The per-byte slow path handles whitespace, padding, and the unpadded tail (2 or 3 leftover characters that produce 1 or 2 output bytes).

### Three Coordinate Spaces

Three coordinate spaces are active during the streaming decode loop:

```text
Root-buffer space:    |-------- original file bytes --------|
                      ^                                     ^
                      base_offset                           base_offset + chunk_len

Encoded-byte space:   |--- encoded span (e.g., Base64) ---|
                      ^                                    ^
                      root_start                           root_start + encoded_len

Decoded-byte space:   |--- decoded output (monotonic) ---|
                      0                         decoded_offset
```

**Root-buffer space** contains absolute offsets in the original file. **Encoded-byte space** contains offsets into the encoded span being decoded. **Decoded-byte space** contains monotonically increasing offsets produced by `stream_decode`, used by the ring buffer, timing wheel, and Vectorscan stream callbacks. Findings are reported in decoded-byte space and must be translated back to root-buffer space via `RootSpanMapCtx` before output.

### The Nine Phases

The decode loop in `decode_stream_and_scan` is the central coordination point. Each decoded chunk passes through nine phases:

1. **Budget enforcement** -- per-transform and global decode limits. If either would be exceeded, the chunk is truncated.
2. **UTF-16 slab buffering** -- conditional; feeds a block scanner later if a NUL byte signals possible UTF-16 content.
3. **Accounting** -- local output counter, global decode budget, incremental AEGIS-128L MAC update, ring buffer push.
4. **Vectorscan stream scan** -- raw anchor prefiltering on decoded chunks; callbacks append `VsStreamWindow` entries to a fixed-capacity staging buffer.
5. **Gate DB scan** -- decoded-space anchor gating for `AnchorsInDecoded` transforms, producing a `gate_hit` flag.
6. **UTF-16 stream activation** -- lazily started on the first NUL byte. The ring buffer is replayed (both segments) to catch up the Vectorscan UTF-16 automaton before feeding the current chunk.
7. **PendingWindow enqueue** -- stream match entries are materialized from the staging buffer and drained into a timing wheel keyed by `hi` (the window-end offset in decoded-byte space, granularity G=1). Per-rule hit caps are enforced; exceeding the cap sets `force_full`.
8. **Timing wheel drain** -- as `decoded_offset` advances, expired windows are batch-drained into a reusable `Vec` and processed sequentially. Each window is materialized from the ring (zero-copy when contiguous, copy when wrapped) or by re-decoding from the original encoded span (O(encoded.len()), the slow path).
9. **Span stream feeding** -- nested transform span detectors (`UrlSpanStream`, `Base64SpanStream`) consume decoded chunks, emitting child decode spans that are copied from the ring into the slab and enqueued as `PendingDecodeSpan` entries for recursive processing.

If any phase sets `force_full`, the entire streaming state is rolled back via `rollback_stream_and_fallback` -- slab truncated, budgets restored, staging buffers cleared -- and the engine falls through to `decode_span_fallback`, which decodes the full span into the slab and enqueues it for block scanning. Two rollback checkpoints exist: one after the streaming loop ends, and one after post-stream close-stream flushes, because closing the Vectorscan stream can emit additional matches that also trigger force-full.

Findings produced during streaming are staged in `tmp_findings`, not `scratch.out`, until the entire stream succeeds. This all-or-nothing commit protocol prevents partial results from leaking into output on rollback.

## 6. StepArena -- Compact Provenance Nodes

Every finding must carry its decode provenance: the chain of transforms that produced it. A naive approach stores a `Vec<DecodeStep>` per finding -- expensive when many findings share the same decode chain. The `StepArena` solves this with a parent-linked arena.

From `decode_state.rs`:

```rust
pub(super) struct StepNode {
    /// Parent step in the provenance chain, or [`STEP_ROOT`] for root-level steps.
    pub(super) parent: StepId,
    step: CompactDecodeStep,
}

const _: () = assert!(std::mem::size_of::<CompactDecodeStep>() == 12);
const _: () = assert!(std::mem::size_of::<StepNode>() == 16);
```

Each node is exactly 16 bytes: a 4-byte `StepId` parent pointer plus a 12-byte `CompactDecodeStep`. The compact step packs a tag bit, a payload index, and two `u32` span endpoints into 12 bytes -- less than half the 32-byte public `DecodeStep` that uses `usize` fields and `Range<usize>`. The bit layout of `tag_and_idx`:

```text
Bit 31: 0 = Transform, 1 = Utf16Window
Bits 30..0: transform_idx (Transform) or endianness ordinal (Utf16Window)
```

`StepId` values are indices into the arena's `ScratchVec<StepNode>`. The sentinel `STEP_ROOT` terminates the chain without requiring an `Option<StepId>` wrapper -- avoiding the 4-byte discriminant overhead that would push the node past 16 bytes. To reconstruct the full chain for output, `StepArena::materialize` walks parent pointers leaf-to-root, collects into a scratch buffer, and reverses in place:

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

The arena is append-only and reset between scans. Multiple findings from the same decoded buffer share the same parent chain -- only the leaf `StepId` differs. Push is O(1); materialization is O(depth), bounded by `max_transform_depth` (typically 2-3).

## 7. RootSpanMapCtx -- Decoded Offsets to Root Coordinates

When a detector fires on decoded bytes, the match span is in **decoded-byte space**. The user needs coordinates in **root-buffer space** -- the original file. `RootSpanMapCtx` bridges this gap.

From `scratch.rs`:

```rust
pub(super) struct RootSpanMapCtx {
    tc: *const TransformConfig,
    encoded_ptr: *const u8,
    encoded_len: usize,
    root_start: usize,
    overlap_backscan: usize,
}
```

The context captures the encoded span being decoded and its absolute position in the root buffer. Raw pointers avoid lifetime entanglement with the engine and buffer references; both point to data that outlives the scan (the engine is immutable after construction; the buffer outlives the scan call). An RAII guard (`RootSpanMapGuard`) clears the context on drop, even on panics, preventing stale pointers from surviving past the encoded buffer's lifetime.

The `map_span` method translates a decoded-byte range back to root coordinates by calling `map_decoded_offset` twice -- once for `start`, once for `end` -- and offsetting both by `root_start`. For URL-percent, `map_decoded_offset` walks encoded bytes one at a time: a `%HH` triplet consumes 3 encoded bytes for 1 decoded byte; everything else is 1:1. For Base64, it walks quantum-by-quantum, tracking cumulative decoded byte count, and returns as soon as the running total reaches the target.

For nested transforms (e.g., Base64 inside a URL-percent span), the `stream_decode.rs` code composes mappings explicitly:

```rust
let child_root_hint = if let Some(hint) = root_hint.as_ref() {
    let start = map_decoded_offset(tc, encoded, parent_span.start);
    let end = map_decoded_offset(tc, encoded, parent_span.end);
    Some(
        hint.start.saturating_add(start)
            ..hint.start.saturating_add(end),
    )
} else {
    Some(parent_span)
};
```

The inner decoded span is first mapped to the outer encoded span via `map_decoded_offset`, then offset by `hint.start` to produce absolute root-buffer coordinates.

The context also provides `has_trigger_before_or_in_match` and `drop_hint_end_for_match`, which are used by the dedup layer to widen drop boundaries for transform-derived findings (see Section 9).

## 8. Three-Level Deduplication

Chunked scanning with overlap, combined with recursive transforms, creates multiple opportunities for the same secret to be reported more than once. The engine applies deduplication at three levels.

### 8.1 Per-Scan: Decoded Buffer Identity

Within a single chunk scan, the same encoded region can appear as both a URL-percent span and a Base64 span, or at different alignment offsets. If two spans decode to identical bytes at the same root position, scanning both is wasted work. The `seen` field on `ScanScratch` is a `FixedSet128` keyed on 128-bit AEGIS-128L MACs of decoded content:

```rust
let h = mix_root_hint_hash(hash128(decoded), &root_hint);
if !scratch.seen.insert(h) {
    scratch.slab.buf.truncate(decoded_range.start);
    return;
}
```

`mix_root_hint_hash` XORs the content hash with a hash of the root-buffer span coordinates. Identical decoded bytes at different root positions produce distinct keys, because findings carry root coordinates for output.

For the streaming path, the content hash is computed incrementally via `Aegis128LMac::update()` on each decoded chunk, finalized after the stream completes. A fixed zero key is acceptable because the engine needs collision resistance, not authentication.

`FixedSet128` is an open-addressed hash table with epoch-based O(1) reset. Each slot is 32 bytes: a 128-bit key, a 32-bit generation counter, and padding. Reset advances the generation counter, making all prior entries logically invisible without clearing memory. On the rare `u32` wraparound (every ~4 billion resets), a single full clear restores the invariant. Linear probing uses the upper 64 bits of the key for initial slot selection, providing better distribution when lower bits exhibit patterns.

### 8.2 Per-File: Finding Dedup via DedupKey

Across chunks within the same file, the same finding can appear in overlapping regions. The `seen_findings` set stores 128-bit hashes of 32-byte `DedupKey` composites:

From `scratch.rs`:

```rust
#[repr(C)]
#[derive(Clone, Copy, bytemuck::Pod, bytemuck::Zeroable)]
struct DedupKey {
    file_id: u32,
    /// Lower 24 bits: `rule_id`. Upper 8 bits: `variant_disc`.
    rule_id_with_variant: u32,
    span_start: u32,
    span_end: u32,
    root_hint_start: u64,
    root_hint_end: u64,
}

const _: () = assert!(std::mem::size_of::<DedupKey>() == 32);
```

The 32-byte size is not accidental. AEGIS-128L processes data in 32-byte absorption blocks (two 128-bit AES blocks). A 32-byte key is absorbed in a single step with no trailing partial-block handling, which is measurably faster than the previous 33-byte layout that required an extra absorption round for one leftover byte. The key is cast to `&[u8]` via `bytemuck::bytes_of` and hashed directly -- no serialization overhead.

`rule_id_with_variant` packs a 24-bit rule ID with an 8-bit variant discriminator (`pack_rule_id_with_variant`) so that UTF-16 LE and BE findings with the same span produce distinct keys. The variant discriminator is derived from the leaf `StepNode` in the provenance arena, ensuring stability across chunks (it depends only on the decode step type, not arena indices).

For transform-derived findings, span coordinates are zeroed when a precise root-span mapping exists. The decoded offsets vary by chunk alignment, but the root hint window is stable -- using it alone as the key prevents the same secret from appearing twice just because two chunks aligned the decoded buffer differently. When mapping is unavailable (nested transforms with length-changing parents), the decoded span is included to prevent collapsing genuinely distinct matches.

Base64 transforms require special handling: padding rules cause the encoded-region length to vary by up to 3 bytes for identical decoded content. `normalize_root_hint_end_for_dedup` snaps `root_hint_end` to the padding-free minimum (`ceil(decoded_len * 4/3)`) when the actual encoded length exceeds it by 1-3 bytes. This normalization prevents false negatives where the same secret encoded with and without trailing `=` would hash to different keys.

Dedup is split into two layers: a per-file set (`seen_findings`) that suppresses cross-chunk repeats, and a per-scan set (`seen_findings_scan`) that enables within-scan replacement -- for example, preferring a transform-derived finding over a raw finding for the same secret, since the transform provides decoded content that aids triage. The replacement policy is most-informative-wins: transform findings replace raw findings; among same-type duplicates, the finding with the wider context window (larger `root_hint_end`) wins; at equal span, the higher confidence score wins.

### 8.3 Cross-Chunk: NormHash via BLAKE3

The third dedup layer targets the secret content itself, independent of location or encoding. `NormHash` is a 32-byte BLAKE3 digest of the raw extracted secret bytes:

```rust
pub type NormHash = [u8; 32];
```

Two findings with the same `NormHash` represent the same secret regardless of surrounding context, encoding transform, or chunk boundary. This is carried alongside each finding and used by downstream consumers (the scheduler, the output pipeline) to collapse findings across files and runs.

## 9. Chunk-Overlap Correctness: drop_prefix_findings

When the scanner processes a file in overlapping chunks, the overlap region appears in both the current and previous chunks. Without correction, findings in the overlap would be emitted twice. `drop_prefix_findings` removes findings that fall within the overlap prefix:

From `scratch.rs`:

```rust
pub fn drop_prefix_findings(&mut self, new_bytes_start: u64) {
    if new_bytes_start == 0 {
        return;
    }
    self.retain_findings_aligned(|_rec, drop_end| drop_end > new_bytes_start);
}
```

The predicate is simple: keep only findings whose drop boundary exceeds the start of new (non-overlap) bytes. The subtlety lies in how `drop_end` is computed for transform-derived findings.

For root-level findings, `drop_end` equals `root_hint_end`. For transform-derived findings, `RootSpanMapCtx::drop_hint_end_for_match` may extend the boundary. The comment in `scratch.rs` provides a worked example:

```text
encoded: "token=AAAA%3Dvalue"
                ^^^^         <- match (raw ASCII prefix)
                    ^        <- first trigger '%' at offset 14

Chunk 1 overlap: 4 bytes before match -- no trigger in overlap.
-> drop_hint_end = 15 (past the '%')

Chunk 2 starts at the '%', re-decodes, and could re-find "AAAA".
But drop_prefix_findings(15) discards it because 15 > match_end.
```

If a trigger already exists within the overlap backscan window (e.g., `x%2Btoken=AAAA%3D`), the prior chunk already saw the trigger, so no extension is needed. For Base64 transforms, the boundary extends to the end of the entire encoded region (`root_start + encoded_len`), because a Base64 span is only fully decodable when the complete encoded span including padding is visible in the chunk.

`retain_findings_aligned` uses a two-pass compaction optimized for the common case where nothing is dropped. The scan pass finds the first dropped row. If none is found, the function returns 0 without touching any sidecar arrays. When rows are dropped, the compact pass copies surviving rows in-place across three parallel arrays (`out`, `drop_hint_end`, `norm_hash`). The parallel-array invariant -- all three vectors stay length-aligned at all times -- is critical. Every push, truncation, and drain must maintain this lock-step relationship; violating it corrupts deduplication and materialization.

## 10. The DecodeSlab -- Allocation-Free Decoded Output

The `DecodeSlab` is a monotonic append-only buffer that stores all decoded bytes for a scan:

From `decode_state.rs`:

```rust
pub(super) struct DecodeSlab {
    pub(super) buf: Vec<u8>,
    pub(super) limit: usize,
}
```

Decoded spans append into the slab and receive a `Range<usize>` back. Work items carry these ranges instead of owning separate allocations. The slab is pre-allocated to the global decode budget (`max_total_decode_output_bytes`) and never reallocates during a scan, which is why the raw-pointer aliasing in the work-queue loop is sound -- all range references remain valid for the scan's lifetime.

The `append_stream_decode` method enforces three budgets on every decoded chunk: per-transform output cap (`max_decoded_bytes`), scan-level output cap, and slab capacity. On decode error, budget exhaustion, or zero output, the slab is truncated back to its pre-call length and the global byte counter is restored:

```rust
if res.is_err() || truncated || local_out == 0 || local_out > max_out {
    self.buf.truncate(start_len);
    *ctx_total_decode_output_bytes = start_ctx;
    return Err(());
}
```

This all-or-nothing rollback ensures downstream code never sees a partially decoded buffer. The pattern repeats at a larger scale in `decode_stream_and_scan`, where a `force_full` flag triggers `rollback_stream_and_fallback` -- truncating the slab, resetting all staging buffers (pending windows, stream matches, pending spans, temporary findings), and delegating to the full-span fallback decoder.

## Summary

The transform pipeline exists to find secrets hidden beneath encoding layers. Span finders trade precision for cheap scanning via a shared `BYTE_CLASS` table. Streaming decoders enforce correctness with bounded memory, using SIMD fast paths for Base64 and `memchr`-accelerated skip for URL-percent. The nine-phase streaming decode loop coordinates Vectorscan matching, timing-wheel window processing, and nested span detection in a single pass. The `StepArena` stores provenance in 16-byte parent-linked nodes. `RootSpanMapCtx` translates decoded offsets back to root coordinates through composed offset mappings. Three dedup levels -- per-scan content identity via AEGIS-128L, per-file `DedupKey` aligned to the 32-byte absorption rate, and cross-chunk `NormHash` via BLAKE3 -- suppress duplicates without false negatives. And `drop_prefix_findings` handles the overlap-boundary edge case with extended drop boundaries for transform-derived matches.

The next chapter examines how the engine coordinates these components at scale: the thread pool, work stealing, and scan scheduling that turn per-chunk scan results into a coherent stream of findings.
