# The Literal Sieve -- Prefiltering and Anchor Derivation

*A detection rule ships with the regex `[A-Za-z0-9+/]{40}`. The pattern matches any 40-character run of base64 alphabet characters. It has no fixed literal substring -- no `"api_key"`, no `"ghp_"`, no `"AKIA"`. Every byte in the pattern is drawn from a 64-element character class. When the scanner's anchor-derivation pass runs over this regex, `analyze` returns `Info { exact: None, pf: Prefilter::All }`. The pattern is unanchorable. Without anchors, the prefilter stage produces zero hits for this rule, and the engine falls back to a run-length residue gate. But consider the alternative: if the engine had no prefilter at all and simply ran the regex against each position in a 16 KB buffer, it would attempt 16,384 match starts per rule. With 847 rules of similar shape, that is 13.9 million regex evaluations per chunk. At the 500,000 chunks-per-second throughput target, the CPU budget is exceeded by three orders of magnitude. The prefilter exists to transform this O(positions x rules) cost into O(anchor_hits) -- a quantity that, for a well-anchored rule set, is measured in single digits per chunk.*

---

Chapter 4 traced the scan pipeline from entry point to finding emission. Chapter 5 detailed the seventeen validation steps inside each candidate window. Both chapters assumed that candidate windows arrive pre-narrowed: a handful of byte ranges per chunk, each tied to a specific rule. This chapter explains how those windows are generated. The answer is a three-layer prefilter stack: Vectorscan multi-pattern databases for literal-match scanning, a regex-to-anchor derivation pass that extracts literal substrings from regex HIR, and specialized gates (SIMD byte classification, base64 Aho-Corasick, URL-percent bitmap) for rules that resist anchoring. Each layer is progressively more expensive and less selective, and the engine applies them in order: anchors first, residue gates second, full-buffer scan last.

## 1. Why Prefiltering Matters

The scan pipeline processes chunks of raw bytes. Each chunk may contain secrets matching any subset of the rule set. The naive approach -- iterate over every rule, run its regex against the full chunk -- has cost proportional to `rules x bytes`. The prefilter approach inverts this: scan the chunk once with a multi-pattern engine, collect the positions where anchor literals appear, then run only the matching rules against narrow windows around those positions. Cost becomes proportional to `anchor_hits`, a quantity bounded by the actual secret density of the input, not the size of the rule set.

The key data structure is the **anchor**: a byte substring that must appear in any match of a rule's regex. The soundness invariant, stated in `regex2anchor.rs`:

```text
For any pattern and haystack:
if regex_matches(pattern, haystack),
then haystack must contain at least one derived anchor.
```

This invariant is non-negotiable. If it is violated, the prefilter can suppress a true match, producing a false negative. The anchor derivation algorithm is therefore conservative: it returns errors when it cannot safely produce anchors, and the engine falls back to progressively weaker gates rather than risk missed detections.

## 2. Vectorscan Multi-Pattern Matching

The prefilter stage delegates literal scanning to Vectorscan (the open-source continuation of Intel Hyperscan). Vectorscan compiles hundreds of literal patterns into a single finite automaton and scans the input buffer in one pass, reporting all match positions. The engine builds five distinct database types, each serving a different coordinate system and scanning mode.

### 2.1 The Five Database Types

From `vectorscan_prefilter.rs`, the database types and their roles:

```rust
pub(crate) struct VsPrefilterDb {
    /// Compiled Vectorscan block-mode database.
    db: *mut vs::hs_database_t,
    /// Number of raw rule patterns in the database.
    raw_rule_count: u32,
    /// Per-raw-pattern metadata (rule id + width + seed radius).
    raw_meta: Vec<RawPatternMeta>,
    /// Rule ids that failed individual compilation (fallback path).
    raw_missing_rules: Vec<u32>,
    /// Pattern id where anchor literals begin (equals `raw_rule_count`).
    anchor_id_base: u32,
    /// Number of anchor literal patterns.
    anchor_pat_count: u32,
    /// Rule/variant targets for anchor patterns.
    anchor_targets: Vec<VsAnchorTarget>,
    /// Prefix-sum offsets into `anchor_targets`.
    anchor_pat_offsets: Vec<u32>,
    /// Byte length of each anchor pattern.
    anchor_pat_lens: Vec<u32>,
    /// Max bounded width across all rules.
    max_width: u32,
    /// True if any rule reports an unbounded width.
    unbounded: bool,
}
```

**`VsPrefilterDb`** is the primary raw-byte database. It operates in block mode (the entire chunk is scanned at once). It contains two pattern classes in a single compiled database: raw rule regexes compiled with `HS_FLAG_PREFILTER` (conservative hits for window seeding) and anchor literal patterns appended after the raw patterns. The `anchor_id_base` field marks where literal patterns begin in the pattern-id space, allowing the match callback to dispatch between the two classes.

**`VsAnchorDb`** handles UTF-16 anchor prefiltering in block mode. Its patterns are literal byte sequences (encoded as `\xNN` regexes) that represent anchor strings in both UTF-16LE and UTF-16BE forms. Matches fan out to per-rule, per-variant windows using the prefix-sum target mapping.

**`VsStreamDb`** serves decoded-stream prefiltering. When the engine applies transforms (base64 decode, URL-percent decode), it produces a decoded byte stream. The stream database processes this output incrementally, emitting candidate windows in decoded-byte coordinates. Patterns are compiled with `HS_FLAG_PREFILTER`.

**`VsGateDb`** is a lightweight stream-mode database for decoded-space gating. It uses `HS_FLAG_SINGLEMATCH` to avoid repeated callbacks -- a single hit is sufficient to signal that at least one anchor variant appears in the decoded stream. This gates whether downstream scanning is worth performing.

**`VsUtf16StreamDb`** is the stream-mode counterpart to `VsAnchorDb`. It processes decoded UTF-16 byte streams in chunks, translating stream-local match offsets into absolute decoded positions using a `base_offset` field.

### 2.2 Pattern Id Layout and Match Dispatch

The `VsPrefilterDb` packs raw-regex and anchor-literal patterns into a single compiled database. Pattern ids below `raw_rule_count` denote raw regex hits; ids at or above `anchor_id_base` denote anchor literal hits. The match callback in `vs_on_match` dispatches on this boundary:

```rust
unsafe extern "C" fn vs_on_match(
    id: c_uint,
    from: u64,
    to: u64,
    _flags: c_uint,
    ctx: *mut c_void,
) -> c_int {
    let c = unsafe { &mut *(ctx as *mut VsMatchCtx) };
    if id < c.raw_rule_count {
        let raw_idx = id as usize;
        let meta = unsafe { *c.raw_meta.add(raw_idx) };
        let rid = meta.rule_id as usize;
        // ... seed window from match_width and seed_radius ...
        return 0;
    }

    if c.anchor_pat_count == 0 || id < c.anchor_id_base {
        return 0;
    }

    let pid = (id - c.anchor_id_base) as usize;
    // ... expand anchor hit to per-target windows ...
    0
}
```

Raw-regex hits index into `raw_meta` (a flat `#[repr(C)]` array of `RawPatternMeta` structs -- 12 bytes each, no padding) to retrieve `rule_id`, `match_width`, and `seed_radius`. Anchor hits index into the prefix-sum target table to fan out to all `(rule, variant)` pairs tied to that literal pattern.

### 2.3 Window Seeding Math

For a raw-regex match ending at byte offset `to`:

```text
start = to - match_width   (saturating)
lo    = start - seed_radius (saturating)
hi    = to + seed_radius    (clamped to hay_len)
```

The `match_width` is derived from `hs_expression_info`, which reports the maximum number of bytes a regex can consume. For truly unbounded patterns (`.*`), Vectorscan reports `UINT_MAX`; the engine maps this to `u32::MAX` and falls back to a whole-buffer window. The `seed_radius` expands the window to account for context that the regex expects around its core match. All arithmetic is saturating to prevent underflow on small offsets, and clamped to `hay_len` to prevent over-read.

## 3. Regex-to-Anchor Derivation

The anchor derivation pass is the compile-time heart of the prefilter system. It runs once per rule during engine construction and produces a `TriggerPlan` that determines how the prefilter handles that rule at scan time.

### 3.1 Walking the HIR

The algorithm parses each rule's regex into the `regex-syntax` HIR (high-level intermediate representation) and performs structural induction over the tree. The HIR accounts for flags, case folding, and syntactic sugar -- for example, `(?i:ab)` is already expanded into `[aA][bB]` by the parser.

The core function is `analyze`, which returns an `Info` for each HIR node:

```rust
struct Info {
    /// If Some, the exact set of strings this node can match.
    /// None means the set is too large or unbounded.
    exact: Option<Vec<Vec<u8>>>,
    /// The prefilter derived from this node.
    pf: Prefilter,
}
```

The `exact` field carries a finite enumeration of all possible match strings for the node. The `pf` field carries a weaker summary -- a substring or set of substrings that must appear in any match. Information only ever weakens as nodes are combined: concatenation forms cross products, alternation forms unions, and any node that can match empty drops required substrings entirely.

### 3.2 Repetition and the Empty-Match Trap

Repetition nodes introduce the most subtle correctness risk in anchor derivation. The key rule: if a repetition can match empty (`min == 0`), then its subexpression's anchors cannot be required -- the regex can match without producing any of them. The `analyze_repetition` function handles this:

```rust
fn analyze_repetition(rep: &Repetition, cfg: &AnchorDeriveConfig) -> Info {
    let min = rep.min;
    let max = rep.max;
    let sub_info = analyze(&rep.sub, cfg);

    if min == 0 {
        if max == Some(0) {
            return Info::exact(vec![vec![]], cfg).unwrap_or_else(Info::all);
        }
        if let Some(exact) = &sub_info.exact {
            if max == Some(1) {
                let mut set = vec![vec![]];
                set.extend(exact.iter().cloned());
                return Info::exact(set, cfg).unwrap_or_else(Info::all);
            }
        }
        return Info::all();
    }
    // ... handle min > 0 ...
}
```

The `?` quantifier (`max == Some(1), min == 0`) is the one zero-min case that preserves a finite exact set: it yields `{""} ∪ sub`, which is enumerable. All other zero-min repetitions (`*`, `{0,n}`) produce unbounded or empty-possible sets, so they degrade to `Info::all()` -- effectively erasing any anchor from that subtree. When `min > 0`, the mandatory copies form a required prefix: the algorithm cross-products the subexpression's exact set `min` times to derive the mandatory substring.

### 3.3 The Concatenation Window

Concatenation is the richest case. When the full cross product exceeds size caps, the algorithm searches for the best contiguous exact window -- a run of consecutive children that all have finite exact sets. The window-search loop in `best_exact_window_prefilter` slides across children, greedily extending the window while the cross product stays within bounds.

The scoring heuristic balances length against set size:

```rust
fn anchor_set_score(min_len: usize, set_size: usize) -> i64 {
    let penalty = ceil_log2(set_size.max(1)) as i64;
    (min_len as i64) * SCORE_BITS_PER_BYTE - penalty
}
```

Longer anchors score higher (8 points per byte of minimum length). Larger sets are penalized by `log2(set_size)` to avoid preferring a huge set of short anchors over a small set of longer ones. For the pattern `api[_-]key=[0-9]+`, this yields the window `["api_key=", "api-key="]` with score 63, rather than the single-child fallback `"api"` with score 24.

### 3.4 Alternation Soundness

Alternation requires every branch to be anchorable. If any branch can match the empty string or has no extractable anchor, the entire alternation is unanchorable. This is a direct consequence of the soundness invariant: the prefilter must return `true` for every input that matches the regex. If even one alternation branch is anchorless, a match through that branch would be invisible to the prefilter.

```rust
fn analyze_alternation(alts: &[Hir], cfg: &AnchorDeriveConfig) -> Info {
    // ...
    for child in &children {
        if let Some(exact) = &child.exact {
            if exact.iter().any(|s| s.is_empty()) {
                return Info::all();
            }
            all_anchors.extend(exact.iter().cloned());
        } else {
            match &child.pf {
                Prefilter::All => return Info::all(),
                Prefilter::Substring(s) => all_anchors.push(s.clone()),
                Prefilter::AnyOf(set) => all_anchors.extend(set.iter().cloned()),
            }
        }
    }
    // ...
}
```

A single empty-string branch makes the whole alternation return `Prefilter::All`. The algorithm never guesses, never filters out weak alternatives, and never strengthens information -- it only weakens it.

### 3.5 Residue Gates

When anchors fail (pattern too broad, only weak anchors, character classes dominating), the algorithm attempts a residue gate. The `RunLengthGate` is the primary fallback. It captures patterns that reduce to a single consuming character class repeated `min..max` times, optionally wrapped by ASCII word boundaries. The gate is specified by a 256-bit byte mask (identical encoding to the anchor byte-set bitmap), a minimum run length, an optional maximum, and a boundary flag.

From `regex2anchor.rs`:

```rust
pub struct RunLengthGate {
    /// 256-bit membership mask for bytes allowed in the run.
    pub byte_mask: [u64; 4],
    /// Min number of units in the run (bytes).
    pub min_len: u32,
    /// Optional max. `None` means unbounded.
    pub max_len: Option<u32>,
    /// ASCII boundary handling applied around the run.
    pub boundary: Boundary,
    /// Optional minimum entropy threshold for the run span.
    pub min_entropy: Option<f32>,
    /// Whether to also scan UTF-16LE/BE ASCII forms.
    pub scan_utf16_variants: bool,
}
```

The byte mask is derived from `class_to_ascii_mask`, which maps a regex character class to a bitset. Only ASCII-only classes are eligible; non-ASCII classes return `None` and the gate derivation fails. The boundary is honored only when both leading and trailing word boundaries are present -- a single-sided boundary is ignored to prevent false negatives from partial boundary matches.

For the opening example pattern `[A-Za-z0-9+/]{40}`, the run-length gate captures the 64-element character class as a byte mask and requires a minimum run of 40 matching bytes. At scan time, the engine scans for a contiguous run of bytes in the mask of at least the minimum length -- far cheaper than running the regex at every position, though less selective than a literal anchor.

### 3.6 The Trigger Plan Hierarchy

The top-level function `compile_trigger_plan` tries gates in priority order:

1. **Anchors** (fastest, most selective). If `choose_anchors` succeeds, the plan includes an `anchors` OR-set and an optional `confirm_all` AND-set of mandatory literal islands.
2. **Residue gates** (run-length or k-gram). If anchors fail, `derive_residue_gate_plan` attempts a byte-class run-length gate or, when the `kgram-gate` feature is enabled, a bounded k-gram gate.
3. **Unfilterable**. If all gates fail, the rule is marked unfilterable with a reason (`MatchesEmptyString`, `OnlyWeakAnchors`, `NoSoundGate`). The engine runs the full regex against every buffer for unfilterable rules.

```rust
pub fn compile_trigger_plan(
    pattern: &str,
    cfg: &AnchorDeriveConfig,
) -> Result<TriggerPlan, AnchorDeriveError> {
    // ...
    if hir_matches_empty(&hir) {
        return Ok(TriggerPlan::Unfilterable {
            reason: UnfilterableReason::MatchesEmptyString,
        });
    }

    let info = analyze(&hir, cfg);
    match choose_anchors(&info, cfg) {
        Ok(anchors) => {
            let mut confirm_all = collect_confirm_all_literals(&hir, cfg);
            confirm_all.retain(|c| !anchors.contains(c));
            return Ok(TriggerPlan::Anchored { anchors, confirm_all });
        }
        Err(_anchor_err) => {}
    }

    if let Some(gate) = derive_residue_gate_plan(&hir) {
        return Ok(TriggerPlan::Residue { gate });
    }
    // ...
}
```

The `confirm_all` field deserves attention. It collects mandatory literal islands from the top-level concatenation -- substrings that must appear in any match but are not the primary anchor. At scan time, the engine uses these for a cheap AND filter after the anchor OR-hit, reducing false candidates without changing soundness.

## 4. The Anchor Byte-Set Bitmap

When the engine applies URL-percent decoding as a transform, it faces a gating question: does this buffer contain any `%XX` triplet that decodes to a byte relevant to any anchor pattern? Scanning every `%XX` triplet against all anchor patterns would be expensive. The `anchor_byte_set` bitmap provides a constant-time answer.

From `core.rs`, the construction:

```rust
let mut anchor_byte_set = [0u64; 4];
for pat in &anchor_patterns_all {
    for &b in pat.as_slice() {
        anchor_byte_set[(b >> 6) as usize] |= 1u64 << (b & 63);
    }
}
```

This is a 256-bit bitmap split across four `u64` words. Byte `b` maps to bit `b & 63` in word `b >> 6`. Every byte that appears in any anchor pattern sets its bit. At scan time, `url_percent_gate_check` scans the buffer for `%XX` triplets, decodes each one, and checks the bitmap:

```rust
pub(super) fn url_percent_gate_check(
    anchor_byte_set: &[u64; 4],
    plus_to_space: bool,
    buf: &[u8],
) -> bool {
    if *anchor_byte_set == [0u64; 4] {
        return true;
    }
    // ...
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

The gate is conservative: an empty bitmap (no anchors compiled) returns `true` (allow scan). A truncated `%XX` at end-of-buffer returns `true`. The gate returns `false` only when it can prove no decoded byte would match any anchor -- the common case for format-specifier-heavy buffers (`%d`, `%s`, `%x`) that dominate source code but never encode anchor-relevant bytes.

## 5. SIMD Character Classification

The character-class distribution gate (step 5 in the window validation pipeline from Chapter 5) uses SIMD-accelerated byte classification to count lowercase, uppercase, digit, and special characters in a window. The classification result feeds the char-class gate, which rejects windows whose character distribution is incompatible with the rule's expected secret shape.

From `simd_classify.rs`, the dispatch:

```rust
pub(crate) fn classify_window(window: &[u8]) -> CharClassProfile {
    #[cfg(all(target_arch = "aarch64", not(miri)))]
    {
        return unsafe { neon::classify_window_neon(window) };
    }
    #[cfg(all(target_arch = "x86_64", not(miri)))]
    {
        return unsafe { sse2::classify_window_sse2(window) };
    }
    #[allow(unreachable_code)]
    classify_window_scalar(window)
}
```

### 5.1 NEON: Two-Op Wrapping Subtract

The NEON implementation uses a two-operation range check per character class. To test whether a byte `v` falls in `[base, base + width)`:

```rust
let is_lower = vcltq_u8(vsubq_u8(v, splat_a), width_26);
```

`vsubq_u8` performs wrapping (not saturating) subtraction. If `v >= base`, the result is `v - base`, which is in `[0, width)` for in-range bytes. If `v < base`, the subtraction wraps around to a large unsigned value (256 + v - base), which exceeds `width` and fails the unsigned less-than comparison. This is cheaper than saturating subtract, which would clamp bytes below the base to zero -- incorrectly passing the `< width` check for any byte below the range.

Accumulation uses a counter trick: NEON comparison masks are `0xFF` (all-ones) on match. Subtracting `0xFF` from an unsigned accumulator wraps to an increment by 1. Lanes are flushed to `u32` totals every 255 iterations to prevent `u8` overflow:

```rust
lower_acc = vsubq_u8(lower_acc, is_lower);
// ...
if chunks_in_epoch == 255 {
    lower_total += vaddlvq_u8(lower_acc) as u32;
    lower_acc = zero;
    chunks_in_epoch = 0;
}
```

### 5.2 SSE2: Three-Op Signed Comparison

SSE2 lacks unsigned byte comparison. The implementation uses signed comparison pairs instead -- three operations per range:

```rust
let is_lower = _mm_and_si128(
    _mm_cmpgt_epi8(v, lower_lo),   // v > ('a' - 1)
    _mm_cmpgt_epi8(lower_hi, v),   // ('z' + 1) > v
);
```

This works correctly for all 256 byte values because ASCII ranges are positive in signed `i8` representation, and bytes >= 0x80 are negative -- they fail all `> positive_constant` comparisons, so non-ASCII bytes are never classified as letters or digits. The accumulation and flush strategy is identical to the NEON path.

### 5.3 Scalar Fallback

A scalar implementation serves as the proptest/Miri oracle and handles the 0-15 byte tail after the SIMD loop:

```rust
fn classify_window_scalar(window: &[u8]) -> CharClassProfile {
    let (mut lower, mut upper, mut digit) = (0u32, 0u32, 0u32);
    for &b in window {
        match b {
            b'a'..=b'z' => lower += 1,
            b'A'..=b'Z' => upper += 1,
            b'0'..=b'9' => digit += 1,
            _ => {}
        }
    }
    CharClassProfile {
        lower,
        upper,
        digit,
        special: window.len() as u32 - lower - upper - digit,
    }
}
```

The invariant `lower + upper + digit + special == window.len()` is verified by property tests across all input lengths and byte values.

## 6. Base64 YARA Gate

Secrets often appear base64-encoded in configuration files, environment variables, and API responses. A raw anchor like `"AKIA"` (an AWS access key prefix) will not appear in the base64 encoding of the same string. The base64 YARA gate bridges this gap by pre-computing the base64-encoded forms of each anchor and scanning for those in base64-ish spans.

### 6.1 The Three-Offset Permutation

Base64 groups input bytes into 3-byte blocks. A raw anchor can start at byte offset 0, 1, or 2 within a block, yielding three distinct base64 encodings. YARA documents this for the string `"This program cannot"`. The implementation in `b64_yara_gate.rs` follows the same trimming rules:

```rust
fn yara_base64_perm(anchor: &[u8], offset: usize, min_pat_len: usize) -> Option<Vec<u8>> {
    // Build [0; offset] + anchor to simulate block alignment.
    let mut prefixed = Vec::with_capacity(offset + anchor.len());
    prefixed.resize(offset, 0u8);
    prefixed.extend_from_slice(anchor);

    let enc = base64_encode_std(&prefixed);

    // Unstable prefix: depends on unknown preceding bytes.
    let left = match offset {
        0 => 0usize,
        1 => 2usize,
        2 => 3usize,
        _ => unreachable!(),
    };

    // Unstable suffix: depends on unknown following bytes.
    let rem = prefixed.len() % 3;
    let right = match rem {
        0 => 0usize,
        1 => 3usize,
        2 => 2usize,
        _ => unreachable!(),
    };

    if enc.len() <= left + right {
        return None;
    }
    let pat = enc[left..(enc.len() - right)].to_vec();
    if pat.len() < min_pat_len { None } else { Some(pat) }
}
```

Leading base64 characters that depend on unknown preceding bytes are trimmed (0, 2, or 3 characters depending on offset). Trailing characters that depend on unknown following bytes are trimmed based on the total length modulo 3. The surviving middle segment is stable regardless of surrounding bytes.

### 6.2 Dense Aho-Corasick Over 64 Symbols

The deduplicated base64 patterns are compiled into a dense Aho-Corasick automaton over the 64-symbol base64 alphabet. The automaton is explicitly not over 256 byte values -- input bytes are first mapped through a lookup table that converts base64 characters to symbols 0-63, maps whitespace to a skip tag, maps `'='` to a padding tag, maps URL-safe variants (`'-'` to `'+'`, `'_'` to `'/'`), and tags everything else as invalid:

```rust
const fn build_lut(allow_ascii_ws: bool) -> [u8; 256] {
    let mut lut = [TAG_INVALID; 256];
    let mut i = 0usize;
    while i < 256 {
        let b = i as u8;
        if b == b'=' {
            lut[i] = TAG_PAD;
        } else if /* whitespace check */ {
            lut[i] = TAG_WS;
        } else {
            let sym = base64_sym(b);
            if sym != TAG_INVALID {
                lut[i] = sym;
            }
        }
        i += 1;
    }
    lut
}
```

The automaton uses a dense next-state table (`state * 64 + sym -> next_state`), making scan time strictly O(1) per byte with no branches in the hot loop:

```rust
#[inline]
fn next_state(&self, state: usize, sym: usize) -> usize {
    self.next[state * 64 + sym] as usize
}
```

Memory cost is `O(states x 64)` -- a deliberate trade of space for predictable latency. The `PaddingPolicy` controls whether `'='` halts scanning (for per-span precision) or resets state and continues (for whole-buffer prefiltering). Invalid bytes always reset to the root state, so matches never span across non-base64 boundaries.

### 6.3 Streaming and Canonicalization

The gate supports incremental scanning across chunk boundaries via `GateState`, a 5-byte struct tracking the automaton state and a padding-halt flag. The scan loop processes one byte at a time through the input LUT:

```rust
pub fn scan_with_state(&self, encoded: &[u8], st: &mut GateState) -> bool {
    if self.padding_policy == PaddingPolicy::StopAndHalt && st.halted {
        return false;
    }
    let mut state = st.state as usize;
    let lut = input_lut(self.whitespace_policy);

    for &b in encoded {
        let v = lut[b as usize];
        if v < 64 {
            state = self.ac.next_state(state, v as usize);
            if self.ac.is_match(state) {
                st.state = state as u32;
                return true;
            }
            continue;
        }
        if v == TAG_WS { continue; }
        if v == TAG_PAD {
            match self.padding_policy {
                PaddingPolicy::StopAndHalt => { st.halted = true; return false; }
                PaddingPolicy::ResetAndContinue => { state = 0; continue; }
            }
        }
        state = 0;
    }
    st.state = state as u32;
    false
}
```

Canonicalization happens implicitly through the LUT: URL-safe `'-'` maps to the same symbol as `'+'` (symbol 62), and `'_'` maps to the same symbol as `'/'` (symbol 63). Whitespace is skipped entirely. This means the gate produces identical results regardless of whether the input uses standard, URL-safe, or line-wrapped base64 encoding -- a property verified by proptest canonicalization-invariance tests.

## 7. UTF-16 NUL Invariant Fast Path

Many Windows-native formats store text as UTF-16LE, where ASCII characters are encoded as `[byte, 0x00]`. The engine exploits a structural invariant: in UTF-16LE-encoded ASCII, every odd-indexed byte is NUL. A buffer that claims to be UTF-16LE but has non-NUL bytes at odd positions is not valid UTF-16LE ASCII text.

The `VsAnchorDb` handles UTF-16 scanning in block mode. Anchor patterns for UTF-16 are pre-encoded as raw byte sequences -- `"api"` becomes `[0x61, 0x00, 0x70, 0x00, 0x69, 0x00]` for UTF-16LE. Vectorscan matches these byte patterns directly against the raw buffer, and the match callback fans out to per-variant windows. The `saw_utf16` flag tracks whether any UTF-16 anchor hit was observed, allowing the engine to skip UTF-16-specific validation when no hits occur.

The NUL-byte interleaving pattern also serves as a fast rejection test: if the engine is considering whether a buffer region contains UTF-16LE text, it checks for the characteristic NUL pattern before committing to a full decode pass. This avoids the cost of UTF-16 decoding on buffers that contain only raw bytes or UTF-8 text -- the vast majority of scanned content.

## Summary

The prefilter stack converts the O(rules x bytes) scanning problem into O(anchor_hits) work. Regex-to-anchor derivation extracts literal substrings from regex HIR via structural induction, with a soundness invariant that guarantees no false negatives. Vectorscan compiles these anchors (along with raw regex prefilter patterns) into five database types covering raw bytes, UTF-16, decoded streams, and gating signals. The anchor byte-set bitmap provides constant-time rejection for URL-percent transforms. SIMD character classification accelerates the char-class distribution gate with architecture-specific NEON and SSE2 paths. The base64 YARA gate pre-computes three-offset permutations and scans with a dense 64-symbol Aho-Corasick automaton. Together, these layers ensure that the engine spends time proportional to the density of real secrets in the input, not to the size of the rule set or the input buffer.
