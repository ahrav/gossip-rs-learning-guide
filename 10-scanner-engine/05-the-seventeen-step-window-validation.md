# Seventeen Gates -- Window Validation Pipeline

*A Vectorscan prefilter hit opens a 4,096-byte window around byte offset 8,192 in a `.yaml` configuration file. The window contains the string `password: correct-horse-battery-staple`. Rule 214 -- a generic password detector -- has a regex that matches `password\s*[:=]\s*(\S+)`. The regex fires. The captured secret is `correct-horse-battery-staple`. But the phrase is a well-known XKCD example, not a real credential. Without a value suppressor gate that checks the extracted secret against known placeholder patterns, this false positive propagates to the output, wastes analyst time, and erodes trust in the scanner. Without an entropy gate, a captured value of `aaaaaaaaaa` -- ten identical characters -- also survives. Without a local-context gate, `password: ${ENV_VAR}` triggers the regex because the template variable is syntactically valid. Each of these is a different class of false positive, and each requires a different kind of gate. The window validation pipeline exists to stack these gates in the correct order, cheapest first, so that false positives are rejected before expensive operations run and real secrets survive to the output.*

---

Chapter 4 traced the path from raw bytes to candidate windows. This chapter narrows the focus to what happens inside each window: the seventeen validation steps that a candidate must survive before it becomes a finding. The orchestrating function is `run_rule_on_window` in `window_validate.rs`, with `apply_emit_time_policy` handling the final suppression cascade. Every gate exists to reject a specific class of false positive, and the ordering is deliberate -- cheap byte-level checks run before expensive regex execution, and suppress-only gates run after evidence collection.

## 1. The Gate Sequence

The seventeen steps divide into four phases: pre-regex gates (steps 1-6), regex and extraction (steps 7-8), evidence and suppress gates (steps 9-14), and scoring and emission (steps 15-17). The complete sequence:

```text
  Candidate window
           |
           v
  +-----------------------------+
  | 1. must-contain (memmem)    |-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 2. confirm-all (AND memmem) |-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 3. keyword gate (OR memmem) |-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 4. assignment-shape precheck|-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 5. char-class SIMD gate     |-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 6. UTF-16 decode (if needed)|-----> reject (budget/error)
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 7. regex execution          |-----> no match: done
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 8. secret extraction        |
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 9. entropy gate             |-----> reject (Failed)
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 10. keyword evidence        |      (score signal)
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 11. value suppressor        |-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 12. local context           |-----> reject
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 13. root-context safelist   |-----> suppress
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 14. secret-bytes safelist   |-----> suppress
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 15. UUID-format reject      |-----> suppress
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 16. offline validation      |-----> suppress (Invalid)
  +-----------------------------+
           |
           v
  +-----------------------------+
  | 17. confidence scoring +    |-----> suppress (below threshold)
  |     min_confidence check    |
  +-----------------------------+
           |
           v
       Finding emitted
```

Steps 1-5 are pre-regex gates that run on the raw window bytes. They are ordered cheapest-first: single-pattern memmem, multi-pattern memmem (AND), multi-pattern memmem (OR), a single-pass byte scan, and a SIMD-accelerated byte classification. Step 6 is UTF-16 decoding (conditional). Steps 7-8 are the regex and secret extraction. Steps 9-12 are post-extraction gates that examine the secret or its local context. Steps 13-16 are emit-time suppression gates applied by `apply_emit_time_policy`. Step 17 is the additive confidence score computed from accumulated evidence.

> **Source code cross-reference**: The `window_validate.rs` module doc comment groups the first three byte-level gates (must-contain, confirm-all, keyword) as a single numbered step ("1. Cheap byte gates"), making steps 2-17 in the source correspond to steps 4-17 in this guide. Both arrive at 17 total individual checks; the difference is purely in grouping.

## 2. Pre-Regex Gates (Steps 1-5)

The pre-regex gates share a common design principle: they use O(n) or sub-linear byte operations to reject windows before paying the O(n * regex_complexity) cost of step 7. For a typical production rule set, these gates reject 70-90% of candidate windows, and the cost of all five gates combined is less than the cost of a single regex evaluation on the same window. The ordering within this phase is also cost-ordered: single-pattern memmem (step 1) is cheaper than multi-pattern memmem (steps 2-3), which is cheaper than a full byte scan (step 4), which is cheaper than SIMD classification (step 5).

### 2.1 Step 1: Must-Contain

The cheapest gate. A single memmem search for a mandatory literal substring. From `window_validate.rs`, inside `run_rule_on_window` for the `Variant::Raw` branch:

```rust
if let Some(needle) = rule.must_contain {
    if memmem::find(window, needle).is_none() {
        return;
    }
}
```

The `must_contain` field is set during rule compilation when the rule spec declares a mandatory anchor. If the window does not contain this exact byte sequence, no regex match is possible -- the entire window is rejected in O(window.len()) time. The `memmem` crate uses Two-Way or SIMD-accelerated algorithms internally, making this gate very cheap relative to the regex it guards.

For UTF-16 variants, must-contain runs on the **decoded** UTF-8 bytes, not the raw window. This is because the literal is specified in UTF-8 and would not match raw UTF-16 code units. This ordering difference -- raw for steps 2-3, decoded for step 1 -- is deliberate: steps 2-3 have variant-encoded patterns that work on raw bytes, while must-contain has only a single UTF-8 literal.

### 2.2 Step 2: Confirm-All (AND Memmem)

Multi-literal AND gate derived from regex analysis. The `confirm_all` gate contains mandatory literal sub-expressions that must *all* appear in any valid match. From `window_validate.rs`:

```rust
if let Some(confirm) = gates.confirm_all {
    let vidx = Variant::Raw.idx();
    if let Some(primary) = &confirm.primary[vidx] {
        if memmem::find(window, primary).is_none() {
            return;
        }
    }
    if !contains_all_memmem(window, &confirm.rest[vidx]) {
        return;
    }
}
```

The `primary` pattern is checked first (single memmem, cheapest). If it passes, the remaining patterns are checked via `contains_all_memmem` (multi-pattern packed search). The gate is indexed by variant (`vidx`), so UTF-16 variants check against UTF-16-encoded literals -- this runs on raw UTF-16 bytes *before* decode to avoid wasting decode budget on windows that cannot pass.

The confirm-all gate is derived during engine construction by the `compile_trigger_plan` function, which analyzes the regex AST to extract mandatory literal sub-expressions. The `primary` field holds the longest literal (checked first for maximum rejection power), and `rest` holds the remaining literals. These are compiled into packed memmem searchers at engine construction time, so the per-window cost is a sequence of fast packed searches rather than repeated regex evaluation.

### 2.3 Step 3: Keyword Gate (OR Memmem)

A pre-regex filter: if none of the rule's keywords appear in the window, the regex is irrelevant. Unlike confirm-all (which is AND logic), the keyword gate is OR logic -- any single keyword hit suffices:

```rust
if let Some(kws) = gates.keywords {
    if !contains_any_memmem(window, &kws.any[Variant::Raw.idx()]) {
        return;
    }
}
```

Like confirm-all, this gate runs on raw UTF-16 bytes for UTF-16 variants to avoid the decode cost.

### 2.4 Step 4: Assignment-Shape Precheck

For rules that match assignment-value patterns (e.g., `password\s*=\s*(\S+)`), a structural precheck rejects windows lacking the necessary shape:

```rust
if rule.needs_assignment_shape_check() && !has_assignment_value_shape(window) {
    return;
}
```

The `has_assignment_value_shape` function performs a single-pass byte scan. It searches for any assignment separator (`=`, `:`, `>`) using vectorized `memchr3`, then checks for a plausible token run of 10+ alphanumeric/underscore/hyphen/dot characters after the separator:

```rust
fn has_assignment_value_shape(window: &[u8]) -> bool {
    let sep_pos = match memchr::memchr3(b'=', b':', b'>', window) {
        Some(pos) => pos,
        None => return false,
    };
    let after_sep = &window[sep_pos + 1..];
    let token_start = after_sep
        .iter()
        .position(|&b| !matches!(b, b' ' | b'\t' | b'"' | b'\'' | b'`' | b'=' | b'>'))
        .unwrap_or(after_sep.len());
    if token_start >= after_sep.len() {
        return false;
    }
    let token_bytes = &after_sep[token_start..];
    let token_len = token_bytes
        .iter()
        .take_while(|&&b| b.is_ascii_alphanumeric() || b == b'_' || b == b'-' || b == b'.')
        .count();
    token_len >= 10
}
```

The 10-byte minimum is the shortest plausible secret length. Shorter tokens are overwhelmingly variable names or configuration values, not secrets. This gate rejects 70-80% of candidate windows for assignment-pattern rules on production workloads. The total cost is O(window.len()) with a small constant factor -- a single `memchr3` call (SIMD-vectorized on x86 and ARM) followed by a linear scan of the suffix.

The gate is conservative with **no false negatives**: a `false` return guarantees the regex will not match. This guarantee holds because the check is only enabled for rules whose patterns structurally require an assignment separator and a token run. False positives (windows that pass the shape check but fail the regex) are acceptable because the regex rejects them -- the goal is to skip the regex entirely on clearly irrelevant windows.

### 2.5 Step 5: Character-Class SIMD Gate

A SIMD-accelerated pre-filter that rejects windows dominated by lowercase ASCII -- prose, English variable names, and other text that cannot be high-entropy secrets:

```rust
if let Some(cc) = gates.char_class {
    if !char_class_gate_passes(window, cc) {
        return;
    }
}
```

The implementation in `simd_classify.rs` dispatches to NEON (aarch64) or SSE2 (x86_64) intrinsics. It classifies every byte into four categories -- lowercase, uppercase, digit, special -- and returns aggregate counts. The gate uses integer cross-multiplication to avoid float division on the hot path:

```rust
fn char_class_gate_passes(window: &[u8], spec: CharClassCompiled) -> bool {
    if window.len() < spec.min_window_len as usize {
        return true;
    }
    let profile = super::simd_classify::classify_window(window);
    (profile.lower as u64) * 100 <= (window.len() as u64) * (spec.max_lower_pct as u64)
}
```

The gate fails open for short windows (below `min_window_len`) to avoid false negatives. The generous default of `max_lower_pct: 95` accommodates dilution from surrounding context (e.g., `password=` preceding the secret).

This gate runs on the **full window**, not the extracted secret bytes. The entropy gate (step 9), by contrast, runs post-extraction on secret bytes only. Running char-class pre-regex avoids the regex cost entirely for prose windows, but surrounding context contributes to the classification. The tradeoff is that a window containing 90% prose with a 10% secret segment might pass the char-class gate but fail the entropy gate on the extracted secret. Both gates are necessary: char-class rejects cheaply at the window level, entropy rejects precisely at the secret level.

### 2.6 Step 6: UTF-16 Decode

For UTF-16 variants (`Utf16Le`, `Utf16Be`), the raw window bytes must be decoded to UTF-8 before regex matching. This step checks the per-window and total-output decode budgets, performs the decode, and runs the remaining gates on the decoded output:

```rust
let remaining = self
    .tuning
    .max_total_decode_output_bytes
    .saturating_sub(scratch.total_decode_output_bytes);
if remaining == 0 {
    return;
}
let max_out = self
    .tuning
    .max_utf16_decoded_bytes_per_window
    .min(remaining);
let decoded = match variant {
    Variant::Utf16Le => decode_utf16le_to_buf(raw_win, max_out, &mut scratch.utf16_buf),
    Variant::Utf16Be => decode_utf16be_to_buf(raw_win, max_out, &mut scratch.utf16_buf),
    _ => unreachable!(),
};
```

UTF-16 decoding tries both byte parities (even and odd offset within the window), hinted parity first. The anchor hint determines which alignment most likely contains the match. This is necessary because merged windows may contain anchors at both parities.

The `utf16_buf` in `ScanScratch` is reused across windows. After decoding, the code uses a raw-pointer reborrow to avoid holding an immutable borrow on `scratch.utf16_buf` while mutating other scratch fields (output vectors, dedup state). The safety argument relies on the fact that no code path between the pointer capture and the last use of the decoded slice writes to, resizes, or reallocates `utf16_buf`.

## 3. Regex Execution and Secret Extraction (Steps 7-8)

### 3.1 Step 7: Regex Execution

Regex matching uses a reusable `CaptureLocations` buffer to avoid per-match heap allocations:

```rust
fn for_each_capture_match(
    re: &regex::bytes::Regex,
    locs: &mut CaptureLocations,
    hay: &[u8],
    mut on_match: impl FnMut(&CaptureLocations, usize, usize),
) {
    let mut at = 0usize;
    while at <= hay.len() {
        let Some(m) = re.captures_read_at(locs, hay, at) else {
            break;
        };
        on_match(locs, m.start(), m.end());
        if m.end() == at {
            at = at.saturating_add(1);
        } else {
            at = m.end();
        }
    }
}
```

The regex does not search from the window start. Instead, the anchor hint narrows the search:

```rust
let hint_in_window = anchor_hint.saturating_sub(w.start);
let search_start = hint_in_window.saturating_sub(BACK_SCAN_MARGIN);
let search_window = &window[search_start..];
```

`BACK_SCAN_MARGIN` (64 bytes) accounts for the possibility that the regex match starts before the anchor literal. For example, the pattern `password\s*=\s*([A-Za-z0-9]+)` anchors on `password`, but the regex match conceptually begins at the `p`. The anchor hint points at the start of the literal `password`, so the regex search must start a few bytes before to capture any preceding context required by the pattern. 64 bytes exceeds the longest prefix observed in production rule sets while adding negligible cost relative to typical 4-16 KiB window sizes. Doubling this value showed no additional matches in benchmarks.

Gates still run on the full window for correctness -- a must-contain check that only examined the search window could miss a mandatory literal in the skipped prefix.

### 3.2 Step 8: Secret Extraction

The extracted secret span is not necessarily the full regex match. The extraction priority follows a three-level cascade handled by `extract_secret_span_locs_raw`:

1. Configured `secret_group` if present and non-empty.
2. First non-empty capture group (1..N). The Gitleaks convention places secrets in group 1, but rules with alternation may fire a higher group.
3. Full match (group 0) as fallback.

This distinction matters because entropy and value-suppressor gates run on the *extracted secret*, not the full match. The full match includes non-secret context (key names, assignment operators, quotes) that would dilute entropy measurements and interfere with suppressor pattern matching. A full match of `password = s3cr3t-k3y!` has low entropy because the prefix `password = ` is dominated by common English letters. The extracted secret `s3cr3t-k3y!` has high entropy -- this is the distinction that determines whether the entropy gate passes.

## 4. Evidence and Suppress Gates (Steps 9-12)

Steps 9-12 operate on the extracted secret bytes and its local context. They divide into two categories: *evidence gates* (steps 9-10) that collect signals for confidence scoring but do not reject, and *suppress gates* (steps 11-12) that reject outright. The entropy gate (step 9) straddles both categories: `Failed` rejects, `PassedMeasured` contributes positive evidence.

### 4.1 Step 9: Entropy Gate

The entropy gate evaluates Shannon entropy on the extracted secret bytes:

```rust
// post_match_entropy_outcome wraps the call, returning None when no
// entropy gate is configured for this rule.
let entropy_outcome = post_match_entropy_outcome(
    entropy,          // Option<EntropyCompiled>
    secret_bytes,
    scratch,          // &mut ScanScratch
    &self.entropy_log2,
);
if matches!(entropy_outcome, Some(EntropyGateOutcome::Failed)) {
    return;
}
```

The wrapper calls `scratch.ensure_entropy_scratch()` to obtain the `&mut EntropyScratch` histogram buffer, then delegates to the real gate function:

```rust
fn entropy_gate_outcome(
    spec: &EntropyCompiled,
    bytes: &[u8],
    scratch: &mut EntropyScratch,   // histogram buffer, not the full ScanScratch
    log2_table: &[f32],
) -> EntropyGateOutcome
```

Three outcomes are possible: `Failed` (hard reject -- entropy too low), `PassedMeasured` (entropy met the threshold), and `BypassedShortLen` (secret too short to measure reliably -- fail-open). The `Failed` outcome causes an immediate return. The other two are stored in `GateEvidence` for confidence scoring -- `PassedMeasured` contributes a positive signal.

The entropy calculation uses a pre-computed `ln(i)/ln(2)` lookup table sized to `max(entropy_gate.max_len)` across all rules, avoiding per-window `f32::log2` calls. The table is built once during engine construction and shared across all scans. An `EntropyScratch` histogram buffer in `ScanScratch` is reused across entropy evaluations to avoid allocating a fresh 256-entry frequency table per check.

The `entropy_gate_outcome` function in `engine/helpers/entropy.rs` evaluates checks in fail-fast order:

1. **Length bypass** (`< min_len`) — O(1), returns `BypassedShortLen`.
2. **Digit-only penalty** (if `digit_penalty` is enabled) — when the evaluated slice (capped at `max_len`) is composed entirely of ASCII digits, the engine subtracts `DIGIT_ONLY_PENALTY_NUMERATOR / log2(capped_len)` from Shannon entropy before thresholding. Skipped for `len == 1` to avoid division by zero. This targets false positives from phone numbers, timestamps, and numeric IDs that can clear a 3.0 bits/byte Shannon threshold. The penalty can push effective Shannon below zero.
3. **Shannon threshold** — rejects the vast majority of low-randomness input by comparing effective Shannon entropy against `min_bits_per_byte`.
4. **Min-entropy threshold** (optional, per NIST SP 800-90B) — only reached by inputs that pass Shannon. Min-entropy `H_inf = -log2(p_max)` measures worst-case predictability: the probability of the single most common byte. Unlike Shannon, which averages over all symbols, min-entropy catches distributions where one byte dominates even though the overall distribution looks diverse (e.g., `AAAA...AABB` has moderate Shannon entropy but very low min-entropy). Candidates below `min_entropy_bits_per_byte` are rejected.

### 4.2 Step 10: Keyword Evidence

Keyword evidence scans a bounded region around the full match for rule keywords. This is not a gate -- it never rejects -- but a positive signal for confidence scoring:

```rust
fn keyword_local_hit(
    hay: &[u8],
    match_start: usize,
    match_end: usize,
    keywords: Option<&PackedPatterns>,
) -> bool {
    let Some(keywords) = keywords else {
        return false;
    };
    let local_start = match_start.saturating_sub(KEYWORD_EVIDENCE_RADIUS);
    let local_end = match_end
        .saturating_add(KEYWORD_EVIDENCE_RADIUS)
        .min(hay.len());
    contains_any_memmem(&hay[local_start..local_end], keywords)
}
```

`KEYWORD_EVIDENCE_RADIUS` is 32 bytes -- large enough to catch `key = <secret>` patterns where the key name precedes the match, but small enough to avoid false confidence boosts from unrelated keywords elsewhere in the buffer.

### 4.3 Step 11: Value Suppressor

Checks the extracted secret bytes against known placeholder and example patterns:

```rust
if let Some(vs) = value_suppressors {
    if contains_any_memmem(secret_bytes, vs) {
        return;
    }
}
```

This is where `correct-horse-battery-staple`, `EXAMPLE`, and similar well-known test values are caught. The packed patterns use the same `memmem` infrastructure as keyword gates but apply to the secret value, not the window context.

### 4.4 Step 12: Local Context

The local-context gate evaluates up to three orthogonal sub-gates in short-circuit order:

```rust
fn local_context_passes(
    window: &[u8],
    secret_start: usize,
    secret_end: usize,
    spec: LocalContextSpec,
) -> bool {
    if spec.require_quoted {
        match is_quoted_at(window, secret_start, secret_end) {
            Some(true) => {}
            Some(false) => return false,
            None => {}
        }
    }
    if spec.require_same_line_assignment || spec.key_names_any.is_some() {
        let bounds = find_line_bounds(window, secret_start, spec.lookbehind, spec.lookahead);
        let (line_start, line_end) = match bounds {
            Some(bounds) => bounds,
            None => return true,
        };
        let line_before_secret = &window[line_start..line_end.min(secret_start)];
        if spec.require_same_line_assignment && !has_assignment_sep(line_before_secret) {
            return false;
        }
        if let Some(keys) = spec.key_names_any {
            if !contains_any_literal(line_before_secret, keys) {
                return false;
            }
        }
    }
    true
}
```

**`require_quoted`**: the secret must be wrapped in matching quote characters (`'`, `"`, or `` ` ``). Fails open when boundary bytes are out of bounds.

**`require_same_line_assignment`**: the line before the secret must contain `=`, `:`, or `>`.

**`key_names_any`**: the line before the secret must contain at least one configured key-name literal.

Sub-gates 2 and 3 share a single `find_line_bounds` call that searches backward and forward for newlines within bounded lookaround windows. When line boundaries are ambiguous (at window or chunk edges), the gate returns `true` -- fail-open -- because a false positive is cheaper than a missed finding.

## 5. Emit-Time Suppression (Steps 13-16)

Steps 13-16 are handled by `apply_emit_time_policy`, called from each emission site:

### 5.1 Step 13: Root-Context Safelist

The global safelist (`SafelistFilter`) uses a `RegexSet` with 18 patterns matching synthetic/demo/placeholder context. Only root-layer findings (`step_id == STEP_ROOT`) are checked -- decoded findings may have different context that does not reflect the original input. The safelist evaluates the full context buffer around the finding, not just the secret bytes. Patterns include context anchors like `[:=]\s*` and line-start anchors that only make sense in surrounding text.

A single-entry cache (`last_safelist_decision`) avoids redundant evaluations when consecutive regex captures in the same `for_each_capture_match` loop produce the same root-hint window. This is common when overlapping prefilter windows decode to the same region.

### 5.2 Step 14: Secret-Bytes Safelist

A second safelist tier matches the extracted secret value directly with 9 patterns. Unlike the context safelist, this applies to *all* findings including decoded/transform-derived ones, because placeholder values in decoded buffers are equally fake. Patterns for known placeholder values use `^...$` anchoring to prevent false suppression of composite secrets containing placeholder words as hyphen-separated segments (e.g., a real secret like `key-null-safety-9xK2mB` should not be suppressed by a pattern for the word "null"). Structural markers (redaction runs, template variables, base64 literals) remain as substring matches.

### 5.3 Step 15: UUID-Format Reject

A procedural byte check that recognizes the canonical 8-4-4-4-12 hyphenated hex UUID format (e.g., `550e8400-e29b-41d4-a716-446655440000`). This is structural-only -- no version or variant validation per RFC 9562. The check is gated per-rule by `uuid_format_secret()` so that rules intentionally capturing UUID-format secrets (e.g., Heroku and Snyk API keys that legitimately look like UUIDs) bypass suppression.

### 5.4 Step 16: Offline Structural Validation

For root-semantic findings, offline validation runs a deterministic, network-free structural check on the secret bytes. From `offline_validate.rs`:

```rust
pub(crate) fn validate(spec: OfflineValidationSpec, secret: &[u8]) -> OfflineVerdict {
    match spec {
        OfflineValidationSpec::Crc32Base62 {
            prefix_skip,
            payload_len,
            checksum_len,
        } => validate_crc32_base62(secret, prefix_skip, payload_len, checksum_len),
        OfflineValidationSpec::GithubFinegrainedPat => validate_github_fine_grained_pat(secret),
        OfflineValidationSpec::GrafanaServiceAccount => validate_grafana_service_account(secret),
        OfflineValidationSpec::AwsAccessKey => validate_aws_access_key(secret),
        OfflineValidationSpec::SentryOrgToken => validate_sentry_org_token(secret),
        OfflineValidationSpec::PyPiToken => validate_pypi_token(secret),
        OfflineValidationSpec::SlackToken => validate_slack_token(secret),
    }
}
```

Each validator returns a three-valued verdict: `Valid` (structural check passed -- likely real), `Invalid` (structurally broken -- safe to suppress), or `Indeterminate` (cannot tell -- do not suppress). The asymmetry is intentional: `Invalid` requires positive proof of structural failure; anything uncertain stays `Indeterminate`.

Validators use branchless decode strategies. All lookup tables (`BASE62_LUT`, `HEX_LUT`, `BASE64_LUT`, `BASE64URL_LUT`) share a sentinel convention where valid values never set bit 7 and invalid bytes map to `0xFF`. Decode loops OR-accumulate lookup results into a single `invalid` flag and defer the validity branch until after the loop. From `offline_validate.rs`, the base-62 decoder:

```rust
fn base62_decode_u32(bytes: &[u8]) -> Option<u32> {
    let mut acc: u64 = 0;
    let mut invalid: u8 = 0;
    for &b in bytes {
        let v = BASE62_LUT[b as usize];
        invalid |= v;
        acc = acc * 62 + v as u64;
    }
    if invalid & 0x80 != 0 {
        return None;
    }
    u32::try_from(acc).ok()
}
```

Valid LUT values (0-61) never set bit 7; the sentinel `0xFF` does. OR-accumulating into `invalid` captures any bad byte without short-circuiting the loop. Garbage accumulated from invalid digits is discarded by the final `invalid` check -- the accumulator is never read on the error path. On AArch64, the loop body compiles to `ldrb + orr + madd` (3 instructions, 0 branches per character), giving the out-of-order engine a straight-line body that pipelines without misprediction stalls. No heap allocation occurs -- decode buffers are stack-local arrays.

The three-valued verdict hierarchy (`Valid`, `Invalid`, `Indeterminate`) enforces an intentional asymmetry: `Invalid` requires positive proof of structural failure (e.g., a CRC mismatch). `Indeterminate` covers cases where the validator cannot tell -- too short, wrong prefix, ambiguous structure. The rule is: never suppress on uncertainty. A token that is too short to validate might still be real.

## 6. Confidence Scoring and Threshold (Step 17)

After all suppression gates pass, a confidence score is computed from the evidence collected in steps 9-10 plus rule-level and emit-time signals:

```rust
fn compute_confidence_score(
    evidence: &GateEvidence,
    rule: &RuleCompiled,
    outcome: &EmitPolicyOutcome,
) -> i8 {
    let mut s: i8 = 0;
    if matches!(
        evidence.entropy_outcome,
        Some(EntropyGateOutcome::PassedMeasured)
    ) {
        s = s.saturating_add(confidence::ENTROPY_PASS);
    }
    if evidence.keyword_local_hit {
        s = s.saturating_add(confidence::KEYWORD_PRESENT);
    }
    if rule.needs_assignment_shape_check() {
        s = s.saturating_add(confidence::ASSIGNMENT_SHAPE);
    }
    if outcome.offline_verdict_valid {
        s = s.saturating_add(confidence::OFFLINE_VALID);
    }
    s.clamp(0, 10)
}
```

Four signals contribute fixed weights:

| Signal | Weight | Condition |
|--------|--------|-----------|
| Entropy pass | `ENTROPY_PASS` (+1) | Measured Shannon entropy met threshold |
| Keyword present | `KEYWORD_PRESENT` (+2) | Rule keyword within 32 bytes of match |
| Assignment shape | `ASSIGNMENT_SHAPE` (+2) | Rule has assignment-value structure |
| Offline valid | `OFFLINE_VALID` (+5) | Structural validation returned `Valid` |

Phase 1 scores are clamped to `0..=10`. The theoretical maximum is 10 (all four signals). Suppress-only gates (must-contain, value-suppressors, safelists) contribute zero because they only filter -- they never add positive signal. The design separates evidence collection (steps 9-10) from scoring (step 17) so that suppress-only gates (steps 11-16) can veto a finding without computing a score.

The final check compares the score against the rule's `min_confidence` threshold:

```rust
if !gates.passes_threshold(confidence_score) {
    crate::perf_stats::sat_add_usize(&mut scratch.confidence_suppressed, 1);
    return;
}
```

Findings below the threshold are suppressed and counted. Surviving findings are committed to the scratch state via `push_finding_with_drop_hint`, recording the BLAKE3 hash of the secret bytes (for cross-chunk and cross-run deduplication), the drop-hint end offset (for overlap deduplication between adjacent chunks), and whether the decoded span participates in the dedup key. Root findings (`step_id == STEP_ROOT`) always include the span in the dedup key because their offsets are absolute file positions. Transform-derived findings with root-span mapping context exclude the span because decoded offsets can shift with chunk boundaries.

The `EmitPolicyOutcome` struct bundles these sidecar values so that `push_finding_with_drop_hint` receives them alongside the `FindingRec`:

```rust
struct EmitPolicyOutcome {
    drop_hint_end: u64,
    dedupe_with_span: bool,
    norm_hash: [u8; 32],
    offline_verdict_valid: bool,
}
```

The `FindingRec` itself is a compact at-most-48-byte struct with `u32` span offsets and `u64` root-hint coordinates. It stores the `confidence_score` alongside the finding data so that downstream consumers (finding materialization, dedup) can access it without re-evaluating gates.

## 7. Why This Ordering

The gate ordering is not arbitrary. Three principles govern the sequence:

**Cheapest first.** Steps 1-3 are O(window.len()) memmem scans. Step 4 is a single-pass byte scan. Step 5 is a SIMD-accelerated classification. These run before the regex (step 7), which is O(window.len() x regex_complexity). Rejecting a window before the regex saves the most expensive per-window operation.

**Extract before evaluate.** Steps 9-12 examine the *extracted secret*, not the full match. Running entropy on the full match would include key names and operators, diluting the measurement. Running value suppressors on the full match would miss patterns in the secret-only portion. Extraction (step 8) must therefore precede these gates.

**Suppress before score.** Suppression gates (steps 11-16) can veto a finding without computing a confidence score. Computing the score first would waste arithmetic on findings that are about to be discarded.

## Summary

The window validation pipeline is a seventeen-step funnel that transforms a candidate window into a scored, deduplicated finding -- or rejects it at the earliest possible step. Pre-regex gates (steps 1-5) handle byte-level rejection. The regex and extraction phase (steps 7-8) produces the secret span. Post-extraction gates (steps 9-12) evaluate entropy, keyword evidence, value suppressors, and local context. Emit-time gates (steps 13-16) apply global safelist suppression, UUID rejection, and offline structural validation. Confidence scoring (step 17) synthesizes accumulated evidence into a single threshold check. Each gate targets a distinct class of false positive, and the ordering ensures that cheap gates run first, extraction precedes evaluation, and suppression precedes scoring.
