# Trust but Verify -- Confidence Scoring and Structural Validation

*The scanner flags `AKIAIOSFODNN7EXAMPLE` as an AWS access key. The regex matches the `AKIA` prefix followed by sixteen uppercase-alphanumeric characters. Shannon entropy measures 4.2 bits per byte -- well above the 3.0-bit threshold. The keyword gate found `aws_access_key_id` twelve bytes before the match. Every pre-extraction gate passes. But the sixteen-character suffix encodes a base-32 payload that embeds a 40-bit AWS account number in bits 1 through 40. Decoded, the account ID is 526,587,584,255 -- a plausible twelve-digit number. The example key from the AWS documentation, however, fails: its decoded account ID exceeds 999,999,999,999 because the suffix `IOSFODNN7EXAMPLE` was chosen for readability, not for structural validity. Without offline validation -- a network-free check that decodes the base-32 suffix and verifies the account-ID range -- this example key becomes finding number 47,291 in the report, indistinguishable from the 47,290 real credentials that preceded it.*

---

Chapter 5 introduced the seventeen-step window validation pipeline and named each gate. Steps 16 and 17 -- offline structural validation and confidence scoring -- were described in outline but not in depth. This chapter provides that depth. Offline validation is the strongest false-positive discriminator in the pipeline: a CRC match or check-digit verification is near-cryptographic proof that a token is structurally well-formed. Confidence scoring synthesizes all per-finding evidence -- entropy, keywords, assignment shape, and offline validity -- into a single number that determines whether a finding survives the `min_confidence` threshold. Together, these two mechanisms close the gap between "regex matched" and "this is probably a real secret."

## 1. The Additive Confidence Model

Every finding accumulates evidence as it passes through the pipeline. The evidence is not a boolean (pass/fail) but a set of weighted signals. Four signals are defined in `api.rs`, inside the `confidence` module:

```rust
pub mod confidence {
    /// Entropy gate passed — secret has sufficient randomness.
    pub const ENTROPY_PASS: i8 = 1;
    /// Local keyword evidence — rule keyword found within 32 bytes of the match span.
    pub const KEYWORD_PRESENT: i8 = 2;
    /// Assignment-shape check passed — secret follows `key = value` pattern.
    pub const ASSIGNMENT_SHAPE: i8 = 2;
    /// Offline structural validation returned Valid (CRC, charset, etc.).
    pub const OFFLINE_VALID: i8 = 5;
}
```

The weights reflect relative strength as false-positive discriminators. Entropy (+1) is a weak baseline -- many non-secrets have high entropy (compressed data, base64 images, random filenames). Keyword and assignment-shape evidence (+2 each) are moderate: finding `password=` near a match is meaningful but not conclusive. Offline validity (+5) is strong: a CRC-32 that matches, a base-32 account ID in range, or a macaroon header that decodes correctly is structural proof that the token was machine-generated in the expected format.

Phase 1 scores range from 0 to 10. The theoretical maximum is 10 (all four signals fire). Compile-time assertions enforce this ceiling:

```rust
const _: () = assert!(confidence::OFFLINE_VALID <= 10);
const _: () = assert!((confidence::KEYWORD_PRESENT + confidence::ENTROPY_PASS) <= 10);
const _: () = assert!(confidence::ASSIGNMENT_SHAPE <= 10);
```

### 1.1 What Does Not Contribute

Suppress-only gates -- must-contain, value suppressors, safelists, UUID-format reject -- contribute zero to the confidence score. They remove false positives but add no positive evidence. A finding that passes must-contain is not more likely to be real; it merely was not eliminated. This asymmetry is deliberate: confidence measures the strength of confirming evidence, not the number of filters survived.

### 1.2 The Scoring Function

Evidence collection happens at steps 9-10 of the pipeline (entropy gate and keyword probe). The score is computed at step 17, after all suppress gates have had the opportunity to veto. From `window_validate.rs`, the `compute_confidence_score` function:

```rust
#[inline(always)]
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

The function uses `saturating_add` to prevent overflow, though the Phase 1 maximum of 10 is well within `i8` bounds. The `clamp(0, 10)` is a belt-and-suspenders guard verified by debug assertions. Each signal is independent: a finding can earn entropy + keyword (+3) or offline-valid alone (+5) or assignment-shape + entropy (+3). No current rule earns all four signals simultaneously, but the model accommodates future rules that do.

### 1.3 Evidence Separation

The per-finding evidence bundle is captured in `GateEvidence`:

```rust
struct GateEvidence {
    /// Entropy-gate decision for this finding's extracted secret bytes.
    entropy_outcome: Option<EntropyGateOutcome>,
    /// Whether any rule keyword appears within KEYWORD_EVIDENCE_RADIUS bytes
    /// of the full regex match in the current buffer.
    keyword_local_hit: bool,
}
```

Separating evidence collection from scoring has a structural purpose. Four emission sites exist in `window_validate.rs` -- raw direct, UTF-16 direct, raw staging, and UTF-16 staging. Each site builds a `GateEvidence` independently. All four sites call the same `compute_confidence_score` function. Without the evidence struct, each emission site would need its own inline scoring logic, and divergence between the four sites would be inevitable.

## 2. The `derive_min_confidence` Cascade

Rules declare a minimum confidence threshold in `RuleSpec::min_confidence`. When a finding's computed score falls below this threshold, it is suppressed. The threshold is precomputed during engine construction and stored in `RuleCold::min_confidence`. From `rule_repr.rs`:

```rust
pub(super) fn derive_min_confidence(spec: &RuleSpec) -> i8 {
    use crate::api::confidence;

    if let Some(v) = spec.min_confidence {
        return v;
    }
    if spec.keywords_any.is_some() && spec.entropy.is_some() {
        return confidence::KEYWORD_PRESENT + confidence::ENTROPY_PASS;
    }
    if needs_assignment_shape_check(spec) {
        return confidence::ASSIGNMENT_SHAPE;
    }
    0
}
```

The cascade has four tiers. The first matching tier wins:

1. **Explicit override** (`Some(v)`). The rule author set a specific threshold. This takes absolute precedence.
2. **Keyword + entropy** (score 3). When both gates are configured, the auto-derived threshold requires a finding to earn both signals: a local keyword hit (+2) and a measured entropy pass (+1). A finding with keywords but low entropy, or high entropy but no keyword, is suppressed.
3. **Assignment-shape** (score 2). Currently applies only to `generic-api-key`, which matches token patterns in arbitrary contexts and benefits most from structural filtering. The threshold requires the assignment-shape signal to fire.
4. **Default** (score 0). No threshold filtering. Every finding that survives the suppress gates is emitted.

### 2.1 Why Offline Validation Is Excluded from Auto-Derivation

Offline validation contributes `OFFLINE_VALID` (+5) to the score, but `derive_min_confidence` never uses it as an auto-derived threshold. The reason is architectural: offline validation runs only on root-semantic findings (`parent_step_id == STEP_ROOT`). Transform-derived findings -- secrets discovered inside base64-decoded or URL-decoded spans -- are never checked because their bytes come from a decoded buffer, not the original input. A per-rule threshold of 5 would silently block all valid transform-derived findings. Rules that want an offline-tier threshold must set `min_confidence` explicitly to `Some(5)`.

### 2.2 Threshold Application at Emission Time

The threshold check runs at the very end of the pipeline, after confidence scoring. From the `ResolvedGates` struct in `window_validate.rs`:

```rust
impl ResolvedGates<'_> {
    #[inline(always)]
    fn passes_threshold(&self, score: i8) -> bool {
        score >= self.min_confidence
    }
}
```

When a finding fails the threshold, a counter is incremented for instrumentation and the finding is discarded:

```rust
if !gates.passes_threshold(confidence_score) {
    crate::perf_stats::sat_add_usize(&mut scratch.confidence_suppressed, 1);
    return;
}
```

The `confidence_suppressed` counter is available when the `perf-stats` feature is enabled, allowing operators to observe how many findings each rule suppresses by threshold. A rule that suppresses most findings by threshold either has a miscalibrated threshold or genuinely noisy anchors that the pre-regex gates cannot filter.

## 3. Offline Structural Validators

Offline validation is step 16 of the seventeen-step pipeline. It runs inside `apply_emit_time_policy`, after safelists and UUID-format rejection have had their chance to suppress. The entry point is the `validate` function in `offline_validate.rs`, which dispatches on `OfflineValidationSpec`:

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

Every validator returns one of three verdicts:

- **`Valid`** -- structural check passed. Contributes `OFFLINE_VALID` (+5) to confidence.
- **`Invalid`** -- structural check failed (bad CRC, out-of-range check digit, wrong header). Safe to suppress.
- **`Indeterminate`** -- cannot determine (token too short, ambiguous prefix, wrong length for CRC extraction). Finding is not suppressed. Contributes 0 to confidence.

The asymmetry is intentional. `Invalid` requires positive proof of structural failure. Anything uncertain stays `Indeterminate` to avoid suppressing a legitimate finding that the validator could not parse.

### 3.1 CRC-32 / Base-62 (General Check-Digit)

The parameterized `Crc32Base62` variant handles any token that embeds a CRC-32 checksum encoded as base-62 digits. The token layout is `[prefix][payload][checksum]`:

```rust
fn validate_crc32_base62(
    secret: &[u8],
    prefix_skip: u8,
    payload_len: u8,
    checksum_len: u8,
) -> OfflineVerdict {
    let skip = prefix_skip as usize;
    let plen = payload_len as usize;
    let clen = checksum_len as usize;
    let required = skip + plen + clen;

    if secret.len() < required {
        return OfflineVerdict::Indeterminate;
    }

    let payload = &secret[skip..skip + plen];
    let checksum_bytes = &secret[skip + plen..skip + plen + clen];

    let actual_crc = crc32fast::hash(payload);

    let decoded_crc = match base62_decode_u32(checksum_bytes) {
        Some(v) => v,
        None => return OfflineVerdict::Indeterminate,
    };

    if actual_crc == decoded_crc {
        OfflineVerdict::Valid
    } else {
        OfflineVerdict::Invalid
    }
}
```

The rule spec provides `prefix_skip`, `payload_len`, and `checksum_len` as compile-time constants. Tokens like npm, NuGet, and other package-manager tokens that use this format can all use the same validator variant with different geometry parameters. The `checksum_len` is capped at 6 during engine build validation, because `62^6 = 56,800,235,584` which exceeds `u32::MAX` -- 6 base-62 digits can encode any CRC-32 value.

### 3.2 GitHub Fine-Grained PAT

GitHub fine-grained personal access tokens have the format `github_pat_<76 body chars><6 base-62 CRC>` (93 bytes total). The CRC-32 is computed over the first 87 bytes -- including the `github_pat_` prefix, unlike the general `Crc32Base62` variant where the prefix is skipped:

```rust
fn validate_github_fine_grained_pat(secret: &[u8]) -> OfflineVerdict {
    if secret.len() < GH_PAT_TOTAL_LEN {
        return OfflineVerdict::Indeterminate;
    }
    if !secret.starts_with(GH_PAT_PREFIX) {
        return OfflineVerdict::Indeterminate;
    }

    let data = &secret[..GH_PAT_TOTAL_LEN - GH_PAT_CHECKSUM_LEN];
    let checksum_bytes = &secret[GH_PAT_TOTAL_LEN - GH_PAT_CHECKSUM_LEN..GH_PAT_TOTAL_LEN];

    let actual_crc = crc32fast::hash(data);
    let decoded_crc = match base62_decode_u32(checksum_bytes) {
        Some(v) => v,
        None => return OfflineVerdict::Indeterminate,
    };

    if actual_crc == decoded_crc {
        OfflineVerdict::Valid
    } else {
        OfflineVerdict::Invalid
    }
}
```

The distinction matters: the general variant hashes only the payload (after prefix skip), but GitHub hashes prefix + payload. A single parameterized variant cannot express both behaviors, which is why GitHub gets a dedicated validator.

### 3.3 AWS Access Key (Check Digit)

AWS access key IDs encode an account number in a base-32 scheme. The validator checks length, prefix, charset, and account-ID range. From `offline_validate.rs`:

```rust
fn validate_aws_access_key(secret: &[u8]) -> OfflineVerdict {
    if secret.len() < AWS_KEY_LEN {
        return OfflineVerdict::Indeterminate;
    }

    let key = &secret[..AWS_KEY_LEN];

    let prefix_word = u32::from_le_bytes([key[0], key[1], key[2], key[3]]);
    const PREFIX_AKIA: u32 = u32::from_le_bytes(*b"AKIA");
    const PREFIX_ASIA: u32 = u32::from_le_bytes(*b"ASIA");
    const PREFIX_ABIA: u32 = u32::from_le_bytes(*b"ABIA");
    const PREFIX_ACCA: u32 = u32::from_le_bytes(*b"ACCA");

    let valid_prefix = matches!(
        prefix_word,
        PREFIX_AKIA | PREFIX_ASIA | PREFIX_ABIA | PREFIX_ACCA
    ) || (key[0] == b'A'
        && key[1] == b'3'
        && key[2] == b'T'
        && (key[3].is_ascii_uppercase() || key[3].is_ascii_digit()));

    if !valid_prefix {
        return OfflineVerdict::Indeterminate;
    }

    let suffix = &key[4..];
    match decode_aws_account_id(suffix) {
        Some(id) if id <= 999_999_999_999 => OfflineVerdict::Valid,
        Some(_) => OfflineVerdict::Invalid,
        None => OfflineVerdict::Invalid,
    }
}
```

The prefix is validated by loading four bytes as a single `u32` and comparing against known constants -- a micro-optimization that avoids four separate `starts_with` calls. The sixteen-character suffix is decoded from AWS's base-32 alphabet (`[A-Z2-7]`) into 80 bits (10 bytes). The account ID occupies bits 1-40 (skipping a flag bit at position 0):

```text
Byte:   [  0  ] [  1  ] [  2  ] [  3  ] [  4  ] [  5  ] ...
Bits:   F AAAAAAA AAAAAAAA AAAAAAAA AAAAAAAA AAAAAAAA A-------
        ^                                              ^
        bit 0 (flag, skipped)                          bit 41+
```

A valid AWS account ID is at most 999,999,999,999 (twelve decimal digits). The example key `AKIAIOSFODNN7EXAMPLE` has a suffix that decodes to an account ID exceeding this bound, which is why the validator correctly returns `Invalid`.

### 3.4 Grafana Service-Account Token (Hex CRC)

Grafana service-account tokens use a different encoding for their checksum: 8 lowercase hex characters instead of base-62. The format is `glsa_<32 alphanumeric>_<8 hex CRC-32>` (46 bytes total). The CRC-32 is computed over the first 37 bytes (`glsa_` prefix + 32 random characters), and the result is hex-encoded in the trailing 8 characters after the separator. From `offline_validate.rs`:

```rust
fn validate_grafana_service_account(secret: &[u8]) -> OfflineVerdict {
    if secret.len() < GLSA_MIN_LEN {
        return OfflineVerdict::Indeterminate;
    }
    if !secret.starts_with(GLSA_PREFIX) {
        return OfflineVerdict::Indeterminate;
    }

    let sep_pos = GLSA_PREFIX.len() + GLSA_RANDOM_LEN;
    if secret[sep_pos] != b'_' {
        return OfflineVerdict::Indeterminate;
    }

    let data = &secret[..sep_pos];
    let hex_bytes = &secret[sep_pos + 1..sep_pos + 1 + GLSA_CHECKSUM_HEX_LEN];

    let actual_crc = crc32fast::hash(data);
    let decoded_crc = match hex_decode_u32(hex_bytes) {
        Some(v) => v,
        None => return OfflineVerdict::Indeterminate,
    };

    if actual_crc == decoded_crc {
        OfflineVerdict::Valid
    } else {
        OfflineVerdict::Invalid
    }
}
```

The hex decoder (`hex_decode_u32`) uses the same branchless sentinel pattern as the base-62 decoder, but with `HEX_LUT` (valid values 0-15, sentinel `0xFF`) and a shift-accumulator instead of multiply-accumulate: `acc = (acc << 4) | nibble as u32`. 8 hex digits map 1:1 to 32 bits, so overflow is impossible -- unlike base-62 where `u32::try_from` catches values that exceed `u32::MAX`.

### 3.5 Sentry DSN (Base64 Structure)

Sentry org-auth-tokens follow the format `sntrys_<base64-payload>_<43 base64 signature>`. The validator performs four checks in sequence:

1. The prefix `sntrys_`.
2. A `_` separator (found via `rposition`) between the payload and the 43-character signature.
3. The signature consists of valid base64 data characters (not padding, not invalid).
4. The base64 payload decodes and its first bytes match `{"iat":` (a JSON `iat` field).

The signature validation uses branchless OR-accumulation over 43 bytes, checking `BASE64_LUT[b] & 0xC0` for each byte. Valid base64 values (0-63) have bits 6-7 clear; both `0xFF` (invalid) and `0xFE` (padding) have at least one of those bits set. A non-zero result means at least one byte was not a data character.

The payload check uses a two-phase design optimized for instruction-level parallelism. Phase 1 is a branchless validity scan over all payload bytes using the `v & (v >> 7)` trick described in Section 4. Phase 2 decodes only the first 12 base64 characters -- `ceil(7/3) * 4 = 12` -- which produces 9 decoded bytes, more than enough to cover the 7-byte `{"iat":` prefix. This avoids decoding the entire payload (which can be hundreds of bytes) when only the header determines the verdict. From `offline_validate.rs`, the `base64_decoded_starts_with` function handles both phases:

```rust
fn base64_decoded_starts_with(
    input: &[u8],
    prefix: &[u8],
    max_decoded_bytes: usize,
) -> Result<bool, Base64DecodeError> {
    let max_decoded = input.len() * 3 / 4;
    if max_decoded > max_decoded_bytes {
        return Err(Base64DecodeError::OutputTooSmall);
    }

    let mut any_invalid: u8 = 0;
    for &b in input {
        let v = BASE64_LUT[b as usize];
        any_invalid |= v & (v >> 7);
    }
    if any_invalid != 0 {
        return Err(Base64DecodeError::InvalidChar);
    }

    let decode_input_len = prefix.len().div_ceil(3) * 4;
    let decode_slice = if decode_input_len < input.len() {
        &input[..decode_input_len]
    } else {
        input
    };

    // ... partial decode and prefix comparison ...
}
```

The error type distinguishes `InvalidChar` (structurally broken, maps to `OfflineVerdict::Invalid`) from `OutputTooSmall` (oversized payload, maps to `OfflineVerdict::Indeterminate`). The 512-byte decoded cap returns `Indeterminate` for oversized payloads rather than processing arbitrarily large inputs.

### 3.6 PyPI Token (Macaroon Format)

PyPI upload tokens are base64url-encoded macaroon V2 binaries prefixed with `pypi-`. The minimum valid token is 21 bytes (`pypi-` + 16 base64url header characters). The validator decodes the first 16 base64url characters into 12 bytes and compares against a known header:

```rust
const PYPI_HEADER: [u8; 12] = [
    0x02, 0x01, 0x08, b'p', b'y', b'p', b'i', b'.', b'o', b'r', b'g', 0x02,
];
```

The header encodes: macaroon V2 version (`0x02`), location field tag (`0x01`), location length (`0x08`), the literal string `pypi.org`, and the identifier field tag (`0x02`). Any token that does not decode to this exact 12-byte sequence is `Invalid`.

The decoder uses `BASE64URL_LUT` -- the URL-safe variant where `-` maps to 62 and `_` maps to 63 (instead of `+` and `/` in standard base64). No padding sentinel exists because `pymacaroons` always strips trailing `=` characters. The decode loop accumulates 6-bit values into a `u32` buffer and emits bytes when the buffer reaches 8 bits, using the standard deferred-invalidity pattern (`invalid |= v; ... if invalid & 0x80 != 0`). The decoded output is a stack-local `[u8; 12]` -- exactly the header size, zero allocation.

### 3.7 Slack Token (Segment Validation)

Slack tokens are the most complex validator because the Slack API uses seven distinct token formats under a common `xox`-prefix family. The dispatch table from `offline_validate.rs`:

| Prefix | Sub-validator | Format |
|--------|--------------|--------|
| `xoxe.xox[bp]-` | config access | `{1 digit}-{163-166 upper+digit}` |
| `xoxb-` | bot token | `{d10-13}-{d10-13}-{alnum tail}` or legacy |
| `xoxp-` | user token | `{d10-13}-{d10-13}-{d10-13}-{alnum 28-34}` |
| `xoxe-` | config refresh or user | disambiguate by first segment |
| `xapp-` | app token | `{1 digit}-{upper+digit}-{digits}-{lower+digit}` |
| `xox[os]-` | legacy | `{d+}-{d+}-{d+}-{hex+}` |
| `xox[ar]-` | legacy workspace | `(?:{digit}-)?{alnum 8-48}` |

Compound prefixes (`xoxe.xoxb-`, `xoxe.xoxp-`) are checked before simple prefixes to avoid misrouting. Unknown prefixes return `Indeterminate` for forward compatibility with future Slack token formats.

The `xoxb-` bot token sub-validator illustrates the dual-format problem: current bot tokens use three segments (`{d10-13}-{d10-13}-{alnum tail}`), but legacy bot tokens use two (`{d8-14}-{an18-26}`). The validator tries the current format first, then falls back to legacy. If neither matches, it returns `Indeterminate` rather than `Invalid` -- because the token may belong to a future format variant:

```rust
fn validate_slack_xoxb(after_prefix: &[u8]) -> OfflineVerdict {
    let (segs, seg_count) = splitn_stack::<4>(after_prefix, b'-');

    if seg_count >= 3 {
        let s1 = segs[0];
        let s2 = segs[1];
        if (10..=13).contains(&s1.len())
            && all_bytes(s1, |b| b.is_ascii_digit())
            && (10..=13).contains(&s2.len())
            && all_bytes(s2, |b| b.is_ascii_digit())
        {
            let tail_start = s1.len() + 1 + s2.len() + 1;
            let tail = &after_prefix[tail_start..];
            if !tail.is_empty() && all_bytes(tail, is_alnum_or_hyphen) {
                return OfflineVerdict::Valid;
            }
        }
    }

    if seg_count >= 2 {
        let s1 = segs[0];
        if (8..=14).contains(&s1.len()) && all_bytes(s1, |b| b.is_ascii_digit()) {
            let tail_start = s1.len() + 1;
            let tail = &after_prefix[tail_start..];
            if (18..=26).contains(&tail.len()) && all_bytes(tail, |b| b.is_ascii_alphanumeric()) {
                return OfflineVerdict::Valid;
            }
        }
    }

    OfflineVerdict::Indeterminate
}
```

The `xoxe-` prefix is ambiguous: it covers both config-refresh tokens (single digit + 146 uppercase characters) and user/enterprise tokens (three numeric segments + alphanumeric tail). The disambiguator is the length of the first segment -- a single digit routes to the config-refresh validator, while 10-13 digits delegate to `validate_slack_user_token`.

Each sub-validator splits on `-` using `splitn_stack`, a zero-allocation helper that writes segments into a stack-local array:

```rust
fn splitn_stack<const N: usize>(data: &[u8], sep: u8) -> ([&[u8]; N], usize) {
    let mut segs: [&[u8]; N] = [&[]; N];
    let mut count = 0;
    let mut start = 0;
    for (i, &b) in data.iter().enumerate() {
        if b == sep && count + 1 < N {
            segs[count] = &data[start..i];
            count += 1;
            start = i + 1;
        }
    }
    segs[count] = &data[start..];
    count += 1;
    (segs, count)
}
```

Segment validation uses `all_bytes`, which OR-accumulates a mismatch flag to minimize branches:

```rust
fn all_bytes(seg: &[u8], pred: fn(u8) -> bool) -> bool {
    let mut bad: u8 = 0;
    for &b in seg {
        bad |= !pred(b) as u8;
    }
    bad == 0
}
```

## 4. Branchless Decode Strategy

All four lookup tables -- `BASE62_LUT`, `HEX_LUT`, `BASE64_LUT`, and `BASE64URL_LUT` -- share a common sentinel convention:

- Valid values occupy the low bits (0-61 for base-62, 0-15 for hex, 0-63 for base64) and never set bit 7.
- Invalid bytes map to `0xFF` (bit 7 set).
- Base64 padding (`=`) maps to `0xFE` (bit 7 set, bit 0 clear).
- `BASE64URL_LUT` omits the padding sentinel entirely because PyPI tokens never contain `=` padding.

Decode loops exploit this by OR-accumulating lookup results into a single `invalid` flag and deferring the validity branch until after the loop. From the base-62 decoder:

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

The `invalid |= v` line is the key: valid values (0-61) never set bit 7, so OR-accumulating them leaves the flag clear. The sentinel `0xFF` always sets bit 7. Garbage accumulated from invalid digits propagates into `acc`, but that value is never read on the error path -- the `invalid & 0x80` check discards it first.

This pattern eliminates per-character branches. On AArch64, the loop body compiles to `ldrb + orr + madd` -- three instructions, zero branches per character. The CPU's out-of-order engine can pipeline the loop body without misprediction stalls, regardless of the input data. The hex decoder uses the same strategy with `(acc << 4) | nibble` instead of multiply-add. The base64 decoder uses a subtler variant: the `v & (v >> 7)` trick distinguishes padding (`0xFE`) from invalid (`0xFF`):

```text
0xFF → 0xFF & (0xFF >> 7) = 0xFF & 1 = 1  (invalid)
0xFE → 0xFE & (0xFE >> 7) = 0xFE & 1 = 0  (padding, acceptable)
0–63 → v & 0 = 0                            (valid)
```

## 5. The Verdict Hierarchy and Emission Integration

Offline validation runs inside `apply_emit_time_policy`, the centralized emit-time gate in `window_validate.rs`. This function evaluates four suppression checks in sequence and, when all pass, returns an `EmitPolicyOutcome` that carries the sidecar data needed for dedup, drop-prefix bookkeeping, and confidence scoring:

```rust
struct EmitPolicyOutcome {
    /// Absolute byte offset past which drop_prefix_findings should keep this finding.
    drop_hint_end: u64,
    /// Whether the decoded span should participate in the dedup key.
    dedupe_with_span: bool,
    /// BLAKE3 digest of the raw secret bytes.
    norm_hash: [u8; 32],
    /// Whether offline structural validation returned Valid.
    offline_verdict_valid: bool,
}
```

The `offline_verdict_valid` field is the bridge between offline validation and confidence scoring. It is `true` when (and only when) the validator returned `OfflineVerdict::Valid`. The `Indeterminate` verdict sets it to `false` -- structurally ambiguous tokens earn no confidence bonus. This prevents a token that is too short for CRC extraction from receiving the same +5 score as a token whose CRC was verified.

The full gate sequence inside `apply_emit_time_policy`:

1. Context-window safelist (step 13) -- root findings only.
2. Secret-bytes safelist (step 14) -- all findings, including decoded.
3. UUID-format quick-reject (step 15) -- skipped when the rule sets `uuid_format_secret`.
4. Compute offline verdict via `compute_offline_verdict`.
5. If `Invalid` and `suppresses_on_invalid()` is true, suppress.
6. Otherwise, record the verdict in `EmitPolicyOutcome::offline_verdict_valid`.

From `window_validate.rs`, the offline integration point:

```rust
let offline_verdict = self.compute_offline_verdict(rule, secret_bytes, parent_step_id);
if matches!(offline_verdict, Some(crate::api::OfflineVerdict::Invalid))
    && self
        .offline_validation_gate(rule.offline_validation)
        .is_some_and(|spec| spec.suppresses_on_invalid())
{
    crate::perf_stats::sat_add_usize(&mut scratch.offline_suppressed, 1);
    return None;
}
```

The `suppresses_on_invalid()` method on `OfflineValidationSpec` is a per-spec policy flag. Currently all seven variants return `true` -- an `Invalid` verdict always suppresses. The match arm is kept explicit so that adding a new variant forces a compile-time decision about suppression policy. A future variant that detects structural failure but wants to keep the finding for manual review would return `false`.

`compute_offline_verdict` itself only runs for root-semantic findings:

```rust
fn compute_offline_verdict(
    &self,
    rule: &RuleCompiled,
    secret_bytes: &[u8],
    parent_step_id: StepId,
) -> Option<crate::api::OfflineVerdict> {
    if parent_step_id != STEP_ROOT {
        return None;
    }
    let spec = self.offline_validation_gate(rule.offline_validation)?;
    Some(super::offline_validate::validate(spec, secret_bytes))
}
```

The `parent_step_id` check is the gate. Transform-derived findings have a parent step that points into the decode-step arena, not `STEP_ROOT`. UTF-16 root findings require care: their own `step_id` is a `Utf16Window` decode step, but their parent is `STEP_ROOT`. The callers pass the parent step ID for offline validation, not the emitted step ID, so UTF-16 root findings are correctly validated.

### 5.1 The `FindingRec` Confidence Field

The computed score is stored in `FindingRec::confidence_score`, an `i8` field:

```rust
pub struct FindingRec {
    // ... span, rule_id, file_id, root hints ...
    /// Additive confidence score computed from per-finding evidence signals.
    pub confidence_score: i8,
}
```

The field does not participate in dedup keys -- two findings at the same span with different scores deduplicate normally. A compile-time assertion enforces that `FindingRec` fits in 48 bytes (three cache-line halves):

```rust
const _: () = assert!(std::mem::size_of::<FindingRec>() <= 48);
```

The score flows out of the scanner through finding materialization (where `FindingRec` is expanded into `Finding`) and into the scheduler and unified event layers as metadata. Downstream consumers can use the score for prioritization, aggregation, triage ordering, or secondary thresholding beyond the scanner's own `min_confidence` gate. The scanner itself uses the score only for the single threshold comparison at step 17; it does not sort, rank, or reorder findings by confidence.

## 6. Feature Gates for Testing and Benchmarking

The offline validation module exposes bench hooks behind the `bench` feature:

```rust
#[cfg(feature = "bench")]
#[inline(always)]
pub fn bench_offline_validate_aws_access_key(secret: &[u8]) -> bool {
    matches!(validate_aws_access_key(secret), OfflineVerdict::Valid)
}
```

Four bench hooks exist: AWS access key, Sentry org token, PyPI token, and Slack token. Each wraps the internal validator in a `pub` function that Criterion benchmarks can call without exposing the full `validate` dispatch. The `bench` feature also gates the SIMD classification benchmarks and the transform decode benchmarks (see `Cargo.toml`), isolating performance measurement infrastructure from production builds.

The `sim-harness` and `test-support` features gate constructors used by simulation and test harnesses. `StepId::from_raw` allows harnesses to reconstruct `StepId` values across crate boundaries:

```rust
#[cfg(any(test, feature = "sim-harness", feature = "test-support"))]
pub fn from_raw(raw: u32) -> Self {
    Self(raw)
}
```

The `tiger-harness` feature enables deterministic simulation hooks. The `perf-stats` feature, implied by `sim-harness`, enables the `confidence_suppressed`, `safelist_suppressed`, `offline_suppressed`, and related counters that track how many findings each suppression gate discards. The `perf-counters` feature extends `perf-stats` with hardware performance counter integration. The feature dependency graph from `Cargo.toml`:

```text
sim-harness ──→ perf-stats
perf-counters ─→ perf-stats
stats ──────→ perf-stats
b64-stats ──→ stats ──→ perf-stats
```

This layered design means that enabling `sim-harness` for deterministic simulation testing automatically enables the performance counters needed to observe suppression rates, without requiring explicit opt-in to `perf-stats`.

## 7. Design Constraints

Three constraints govern offline validation:

**No heap allocation.** Decode buffers are stack-local (`[u8; 10]` for AWS, `[u8; 12]` for PyPI). The lookup tables are 256-byte `const` arrays compiled into the binary's read-only segment. No `Vec`, no `String`, no `Box`.

**Conservative verdicts.** When a token is too short for CRC extraction, or has a wrong prefix, or produces non-base-62 characters where a checksum is expected, the verdict is `Indeterminate` -- not `Invalid`. This fail-open behavior prevents suppressing legitimate findings that the validator could not structurally parse. The regex may have captured a wider span than the actual token, or the token format may have evolved.

**No regex or I/O.** Validators operate on the already-extracted `&[u8]` slice. They compile no regexes, open no files, and make no network calls. This is an offline check in the strictest sense: the only inputs are the secret bytes and the spec geometry.

## 8. Summary

Confidence scoring and offline validation are the final gates before a finding is emitted. The additive confidence model sums four weighted signals -- entropy (+1), keyword (+2), assignment shape (+2), offline valid (+5) -- into a score clamped to 0-10. The `derive_min_confidence` cascade auto-selects a threshold based on which gates a rule configures: keyword+entropy requires 3, assignment-shape requires 2, rules without either require 0. Offline validation is excluded from auto-derivation because it only fires on root-semantic findings.

Seven offline validators handle the common token formats: CRC-32/base-62 (generic), GitHub PAT, Grafana service-account, AWS access key, Sentry org-auth-token, PyPI upload token, and Slack API tokens. All use branchless decode strategies with sentinel lookup tables. All return a three-valued verdict (`Valid`, `Invalid`, `Indeterminate`) where `Invalid` requires positive structural proof.

The `FindingRec` carries the computed `confidence_score` as an `i8` field. It does not participate in dedup keys -- two findings at the same span with different scores still deduplicate normally. The score flows through the scheduler and unified event layers as a metadata signal that downstream consumers can use for prioritization, aggregation, or secondary thresholding beyond the scanner's own `min_confidence` gate.
