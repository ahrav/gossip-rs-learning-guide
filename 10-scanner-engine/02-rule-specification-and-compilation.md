# The Rule Compiler -- From YAML to Vectorscan

*A security team deploys 847 detection rules from a single YAML file. One rule -- `generic-api-key` -- has a regex whose compiled DFA exceeds 32 MB, the default size limit for the Rust `regex` crate. The engine fails to compile the pattern and returns an opaque "CompiledTooBig" error. The operator raises the global limit to 512 MB to accommodate that one rule, but now every other pattern -- including a 14-byte literal match for Slack tokens -- pays for a 512 MB DFA cache reservation at construction time. Startup latency jumps from 1.2 seconds to 11 seconds. The next rule update adds a second heavy pattern. The operator raises the limit again. Memory usage at rest climbs past 2 GB, and the container OOM-kills during the first scan because each worker thread allocates its own DFA cache at that ceiling.*

*Meanwhile, 340 of the 847 rules specify an entropy gate. For each of those rules, the engine must compile keyword patterns into three encoding variants (raw, UTF-16LE, UTF-16BE), allocate a Shannon entropy evaluator, and possibly derive a character-class distribution gate -- all before the first byte of input is scanned. If each gate object is stored inline on the compiled rule struct, the per-rule size balloons from 88 bytes to over 400 bytes, and the hot scan loop pays cache misses on every rule iteration for gate data it touches only once per confirmed match. Without a compilation architecture that separates hot from cold, scales regex limits per-rule, and pools gate objects by type, the engine cannot serve both a hundred-rule lightweight deployment and a thousand-rule enterprise configuration from the same binary.*

---

Rule compilation is the bridge between the human-authored detection policy and the machine-executable scan loop. A `RuleSpec` describes *what* to find; the compilation pipeline transforms that description into a `RuleCompiled` that describes *how* to find it efficiently. This chapter traces the full pipeline: YAML deserialization, progressive regex compilation, gate pooling with the `NO_GATE` sentinel, anchor policy selection, and the two-tier `RuleCompiled` / `RuleCold` split that keeps the hot scan loop compact.

## 1. The YAML Schema

Detection rules are expressed as YAML documents with a single `rules` key whose value is a sequence of rule objects. The on-disk schema is defined by the `YamlRule` struct in `rules/yaml.rs`:

```rust
#[derive(Deserialize)]
pub(crate) struct YamlRule {
    /// Unique rule identifier used for diagnostics and finding attribution.
    pub name: String,
    /// Pattern compiled into a [`regex::bytes::Regex`] during parsing.
    pub regex: String,
    /// Literal byte strings fed to the Vectorscan pre-filter; at least one
    /// must appear in the input for a scan window to be opened.
    pub anchors: Vec<String>,
    /// Byte radius around each anchor hit that defines the scan window.
    pub radius: usize,
    #[serde(default)]
    pub must_contain: Option<String>,
    #[serde(default)]
    pub keywords_any: Option<Vec<String>>,
    #[serde(default)]
    pub value_suppressors_any: Option<Vec<String>>,
    #[serde(default)]
    pub entropy: Option<YamlEntropy>,
    #[serde(default)]
    pub char_class: Option<YamlCharClass>,
    #[serde(default)]
    pub two_phase: Option<YamlTwoPhase>,
    #[serde(default)]
    pub local_context: Option<YamlLocalContext>,
    #[serde(default)]
    pub offline_validation: Option<YamlOfflineValidation>,
    #[serde(default)]
    pub secret_group: Option<u16>,
    #[serde(default)]
    pub uuid_format_secret: bool,
    #[serde(default)]
    pub min_confidence: Option<i8>,
}
```

Four fields are required (`name`, `regex`, `anchors`, `radius`); all others default to absent. Unknown fields are silently ignored by serde so that newer YAML files can be read by older parser versions.

The `YamlEntropy` sub-struct carries the full entropy gate configuration:

```rust
pub(crate) struct YamlEntropy {
    pub min_bits_per_byte: f32,
    pub min_len: usize,
    pub max_len: usize,
    #[serde(default)]
    pub min_entropy_bits_per_byte: Option<f32>,
    #[serde(default)]
    pub digit_penalty: bool,
}
```

The `min_entropy_bits_per_byte` and `digit_penalty` fields are optional (`#[serde(default)]`) so that existing YAML rules without them continue to work. These are converted directly into the corresponding `EntropySpec` fields during Stage 3.

> **Note**: While serde silently ignores unknown fields at runtime, the companion test `default_rules_yaml_has_no_unknown_fields` guards against accidental typos in the built-in rule set by verifying that every field in `default_rules.yaml` maps to a known `YamlRule` field.

## 2. The Three-Stage Conversion Pipeline

The function `parse_yaml_rules` in `rules/yaml.rs` transforms YAML text into `Vec<RuleSpec>` through three stages applied per rule:

**Stage 1 -- Deserialize.** Serde maps YAML text into `YamlRule` structs using owned `String` / `Vec` types.

**Stage 2 -- Compile regex.** The pattern string is compiled into a `regex::bytes::Regex` via `build_regex`. This stage runs *outside* the interning lock to avoid holding the mutex during potentially expensive compilation.

**Stage 3 -- Intern and assemble.** String and byte fields are interned into the global `RuleAtomPool` (producing `&'static` references), numeric fields are converted directly, and a `RuleSpec` is emitted.

### 2.1 Progressive Regex Compilation

The opening failure scenario is not hypothetical. Complex patterns routinely exceed the default compiled-size limit. The engine handles this with a progressive retry strategy, defined in `rules/mod.rs`:

```rust
/// Progressive regex size limits (bytes) to tolerate large DFAs on complex rules.
///
/// We retry compilation on `CompiledTooBig` so a single oversized rule does not
/// require globally lifting size limits for all patterns.
const REGEX_SIZE_LIMITS: &[usize] = &[32 * 1024 * 1024, 128 * 1024 * 1024, 512 * 1024 * 1024];

pub(crate) fn build_regex(pattern: &str) -> Result<Regex, String> {
    for &limit in REGEX_SIZE_LIMITS {
        let mut builder = regex::bytes::RegexBuilder::new(pattern);
        builder.unicode(false);
        builder.size_limit(limit);
        builder.dfa_size_limit(limit);
        match builder.build() {
            Ok(re) => return Ok(re),
            Err(regex::Error::CompiledTooBig(_)) => {
                eprintln!("note: regex exceeded {limit} byte compile limit, retrying at next tier");
                continue;
            }
            Err(err) => return Err(err.to_string()),
        }
    }
    Err(format!(
        "regex compiled too big even at {} bytes",
        REGEX_SIZE_LIMITS[REGEX_SIZE_LIMITS.len() - 1]
    ))
}
```

The three tiers -- 32 MB, 128 MB, 512 MB -- form a geometric progression. Most rules compile at the first tier. A complex pattern like `generic-api-key` may need the second. Only pathological patterns reach the third, and if a pattern exceeds 512 MB, compilation fails with a structured error rather than consuming unbounded memory. The key property is that each rule pays only the cost of its own tier. A 14-byte Slack token pattern compiles at 32 MB and never sees the higher limits.

Only `CompiledTooBig` errors trigger a retry. Syntax errors and other failures return immediately -- there is no point retrying a malformed pattern at a larger budget.

### 2.2 Atom Interning

Rule literals -- names, anchors, keywords -- are interned into a process-global `RuleAtomPool` and returned as `&'static` references. The pool is protected by a `Mutex` with poison recovery:

```rust
static RULE_ATOMS: LazyLock<Mutex<RuleAtomPool>> =
    LazyLock::new(|| Mutex::new(RuleAtomPool::default()));
```

Interning serves two purposes. First, it makes `RuleSpec` fields `'static`, which eliminates lifetime entanglement between rule data and the engine. Second, it deduplicates shared atoms: if two rules use the same anchor string `"ghp_"`, only one heap allocation exists.

The pool uses `Box::leak` to produce `'static` references. This is intentional: rule literals live for the entire process, and the cost (roughly 2x the unique atom set) is acceptable because the rule corpus is small -- hundreds of rules, each with a few short literals.

### 2.3 Implicit Defaults -- Auto-Enabling Character-Class Gates

During Stage 3, the parser applies an implicit default. Rules with an entropy threshold at or above 3.0 bits/byte that lack an explicit `char_class` gate automatically receive one:

```rust
pub(crate) const AUTO_CHAR_CLASS_ENTROPY_THRESHOLD: f32 = 3.0;
pub(crate) const AUTO_CHAR_CLASS_MAX_LOWER_PCT: u8 = 95;
pub(crate) const AUTO_CHAR_CLASS_MIN_WINDOW_LEN: u16 = 32;
```

The rationale: rules at or above 3.0 bits/byte target high-entropy secrets (API keys, tokens) where a predominantly-lowercase scan window is a reliable signal for prose rather than a real secret. Rules below that threshold target passwords or low-entropy strings where the same gate would cause false negatives. The 3.0 threshold was chosen empirically to separate low-entropy rules (e.g., `nuget-config-password` at 1.0) from token-oriented rules.

## 3. Built-In Rules and Validation

The engine ships with a built-in rule set embedded via `include_str!` at compile time. From `rules/mod.rs`:

```rust
const BUILTIN_RULES_YAML: &str = include_str!("../../default_rules.yaml");

pub fn builtin_rules() -> Vec<RuleSpec> {
    static BUILTIN: OnceLock<Vec<RuleSpec>> = OnceLock::new();
    BUILTIN
        .get_or_init(|| {
            let rules = yaml::parse_yaml_rules(BUILTIN_RULES_YAML)
                .expect("built-in rules YAML must be valid");
            assert!(
                !rules.is_empty(),
                "built-in rules YAML must contain at least one rule"
            );
            rules
        })
        .clone()
}
```

The `OnceLock` ensures rules are parsed and compiled once, then cloned on subsequent calls. The `Box::leak` allocations in the YAML parser happen at most once. Invalid embedded YAML panics during initialization because that indicates a build-time packaging error, not runtime user input.

For external rule files, `load_rules_from_content` catches assertion panics from `assert_valid()` and converts them to structured `RulesError` values. The `catch_unwind` wrapper is sound because `assert_valid()` performs read-only invariant checks:

```rust
for rule in &rules {
    let name = rule.name.to_string();
    let result = std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| {
        rule.assert_valid();
    }));
    if let Err(payload) = result {
        let message = if let Some(s) = payload.downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = payload.downcast_ref::<String>() {
            s.clone()
        } else {
            std::panic::resume_unwind(payload);
        };
        return Err(RulesError::Validation {
            rule_name: name,
            message,
        });
    }
}
```

Only string-typed panics (from `assert!` / `panic!` macros) are treated as validation errors. Other panic types (OOM, stack overflow) are re-raised because they are not validation failures. This distinction matters: an operator who supplies a rule file with `entropy.min_len > entropy.max_len` gets a clear error message naming the rule and the violated invariant, not a process crash.

The default rules path is intentionally restricted to the executable-adjacent location. The current working directory is excluded because the scanner operates on untrusted repositories -- loading a `default_rules.yaml` planted inside a scanned repo would allow an attacker to suppress detections or crash the scanner.

## 4. The RuleSpec -- The Full Detection Policy

After conversion, each rule is represented as a `RuleSpec`. From `api.rs`:

```rust
pub struct RuleSpec {
    pub name: &'static str,
    pub anchors: &'static [&'static [u8]],
    pub radius: usize,
    pub validator: ValidatorKind,
    pub two_phase: Option<TwoPhaseSpec>,
    pub must_contain: Option<&'static [u8]>,
    pub keywords_any: Option<&'static [&'static [u8]]>,
    pub value_suppressors_any: Option<&'static [&'static [u8]]>,
    pub entropy: Option<EntropySpec>,
    pub char_class: Option<CharClassSpec>,
    pub local_context: Option<LocalContextSpec>,
    pub offline_validation: Option<OfflineValidationSpec>,
    pub uuid_format_secret: bool,
    pub secret_group: Option<u16>,
    pub min_confidence: Option<i8>,
    pub re: Regex,
}
```

Every `Option` field represents a gate that can suppress false positives. The gates form a cascade that runs cheapest-first: `must_contain` (a single `memmem` call) before `keywords_any` (a multi-pattern `memmem`), before `char_class` (a SIMD byte classification), before `entropy` (a histogram computation on extracted secret bytes), before `offline_validation` (a CRC or format check). Each gate is a filter that narrows the candidate set before the next, more expensive gate runs.

`RuleSpec::assert_valid` is called at engine build time on every rule. It validates cross-field invariants:

- `name` must be non-empty.
- `two_phase.seed_radius <= two_phase.full_radius`.
- `entropy.min_len <= entropy.max_len` and `min_bits_per_byte` is in `[0.0, 8.0]`.
- `secret_group` must reference a capture group that exists in the regex.
- `min_confidence` must be in `0..=10`.

Validation failures panic during engine construction, which is the correct failure mode: invalid rules are programming errors (for built-in rules) or authoring errors (for user-supplied YAML) and cannot be recovered at runtime.

### 4.1 EntropySpec -- Shannon and Min-Entropy Gates

The `EntropySpec` struct configures entropy-based filtering on extracted secret bytes. From `api.rs`:

```rust
pub struct EntropySpec {
    /// Minimum Shannon entropy threshold in bits/byte.
    pub min_bits_per_byte: f32,
    /// Matches shorter than this length pass without entropy checks.
    pub min_len: usize,
    /// Max number of bytes used for entropy calculation.
    pub max_len: usize,
    /// Lower bound on min-entropy in bits/byte (NIST SP 800-90B).
    pub min_entropy_bits_per_byte: Option<f32>,
    /// When `true`, apply a digit-only penalty before the Shannon check.
    pub digit_penalty: bool,
}
```

The first three fields define the core Shannon entropy gate: `min_bits_per_byte` sets the rejection threshold, `min_len` bypasses the check for short matches where entropy is noisy, and `max_len` caps compute cost on long matches.

The two additional fields provide finer-grained false positive control:

- **`min_entropy_bits_per_byte`** adds a second gate based on min-entropy (`H_inf = -log2(p_max)`) from NIST SP 800-90B. Unlike Shannon entropy, which averages over all symbols, min-entropy measures the probability of the single most common byte. This catches distributions where one byte dominates -- e.g., `AAAA...AABB` has moderate Shannon entropy but very low min-entropy. Candidates below this threshold are rejected. `None` disables the gate.

- **`digit_penalty`** penalizes all-digit sequences before the Shannon threshold check. When the evaluated entropy slice (after `max_len` capping) is composed entirely of ASCII digits, the engine subtracts `DIGIT_ONLY_PENALTY_NUMERATOR / log2(len)` from Shannon entropy (skipped for `len == 1` to avoid division by zero). This targets false positives from phone numbers, timestamps, and numeric IDs that can clear a 3.0 bits/byte Shannon threshold. The penalty can push effective Shannon below zero, rejecting candidates even with `min_bits_per_byte: 0.0`.

### 4.2 ValidatorKind -- Fast-Path Token Validation

The `RuleSpec::validator` field enables an optional fast path that can confirm token-like matches at anchor hits, bypassing the full window accumulation and regex evaluation pipeline. From `api.rs`:

```rust
pub enum ValidatorKind {
    /// Prefix + fixed-length tail + optional boundary/terminator checks.
    PrefixFixed {
        tail_len: u16,
        tail: TailCharset,
        require_word_boundary_before: bool,
        delim_after: DelimAfter,
    },
    /// Prefix + bounded-length tail + optional boundary/terminator checks.
    PrefixBounded {
        min_tail: u16,
        max_tail: u16,
        tail: TailCharset,
        require_word_boundary_before: bool,
        delim_after: DelimAfter,
    },
    /// Special-case validator for AWS access key IDs (A3T... / AKIA...).
    AwsAccessKey,
    /// No validator; always use the regex/window path.
    None,
}
```

When the engine encounters an anchor hit for a rule with a non-`None` validator, it runs the validator first. If the validator confirms the match, the finding is emitted immediately without building a window or running the regex. If the validator rejects, the engine falls back to the normal window + regex path. This is a pure performance optimization: validators assume the anchor match is match-start aligned in the raw representation.

The two structural variants work with `TailCharset` and `DelimAfter`:

- **`TailCharset`** defines the set of ASCII bytes accepted in the tail portion of a token. Variants include `UpperAlnum` (`[A-Z0-9]`), `Alnum` (`[A-Za-z0-9]`), `LowerAlnum` (`[a-z0-9]`), `AlnumDashUnderscore` (`[A-Za-z0-9_-]`), `Sendgrid66Set` (`[A-Za-z0-9=_\-.]`), `DatabricksSet` (`[a-hA-H0-9]`), and `Base64Std` (`[A-Za-z0-9+/]`). The validator scans bytes greedily from the anchor end and stops at the first byte outside the class.

- **`DelimAfter`** optionally checks the byte immediately following the tail. `DelimAfter::GitleaksTokenTerminator` requires `['"|\\s|;|\\x60]`, escaped newlines, or end-of-input. `DelimAfter::None` skips the check.

`PrefixBounded` is greedy -- it matches the longest tail within `[min_tail, max_tail]` and backtracks to find the longest tail immediately followed by a valid delimiter when `delim_after` is required.

The built-in gitleaks-derived rule set currently uses `ValidatorKind::None` for all rules; non-`None` variants are available for custom rule sets that can guarantee match-start-aligned anchors.

> **Note**: `ValidatorKind` is intentionally absent from the YAML schema (`YamlRule` does not include a `validator` field). Fast validators are tightly coupled to specific regex patterns and anchor layouts, so they are only set programmatically.

### 4.3 TransformConfig -- Controlling Decode Passes

Rules do not exist in isolation. The engine also compiles transform configurations that control how derived buffers (base64 decoded, URL-percent decoded) are produced and gated. From `api.rs`:

```rust
pub struct TransformConfig {
    pub id: TransformId,
    pub mode: TransformMode,
    pub gate: Gate,
    pub min_len: usize,
    pub max_spans_per_buffer: usize,
    pub max_encoded_len: usize,
    pub max_decoded_bytes: usize,
    pub plus_to_space: bool,
    pub base64_allow_space_ws: bool,
}
```

Each `TransformConfig` is validated at engine build time via `assert_valid()`, which enforces `max_encoded_len >= min_len` and ensures that enabled transforms have positive `max_spans_per_buffer` and `max_decoded_bytes`. The `gate` field controls a post-decode filter: `Gate::AnchorsInDecoded` stream-decodes and proceeds only if the decoded bytes contain any anchor variant. This gate runs *after* a span is decoded but *before* the decoded buffer is enqueued for recursive scanning, allowing early discard of unproductive decodes.

The `TransformMode` enum has three variants: `Disabled` (never apply), `Always` (always apply, subject to caps), and `IfNoFindingsInThisBuffer` (skip when the current buffer has already produced findings). The third mode is an explicit correctness trade-off: reducing redundant transform work at the cost of potentially missing findings in nested encodings.

## 5. Two-Tier Compilation -- RuleCompiled and RuleCold

The compilation data flow is documented in `engine/rule_repr.rs`:

```text
RuleSpec (api.rs)
  │
  ├─ compile_rule() ──► (RuleCompiled, CompiledGates)
  │                          │               │
  │                          │   Engine::new() pools each gate object into
  │                          │   a type-specific Vec on Engine and patches
  │                          │   the u32 index back onto RuleCompiled.
  │                          ▼
  │                     RuleCompiled   ── hot array iterated per buffer
  │                     RuleCold       ── parallel cold array (name, min confidence)
  │
  ├─ add_pat_raw/owned() ──► anchor map
  │                              ▼
  │                         map_to_patterns() ──► (patterns, targets, offsets)
  │                              ▼
  │                         Vectorscan prefilter DB
  │
  └─ compile_confirm_all() ──► ConfirmAllCompiled (pooled in second pass)
```

The split into two tiers is the central design decision. `compile_rule` transforms a validated `RuleSpec` into a compact runtime representation across two structs. From `rule_repr.rs`:

```rust
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
pub(super) struct RuleCold {
    pub(super) name: &'static str,
    pub(super) min_confidence: i8,
}
```

Size is enforced at compile time:

```rust
const _: () = assert!(std::mem::size_of::<RuleCompiled>() <= 88);
const _: () = assert!(std::mem::size_of::<RuleCold>() <= 24);
```

**Why two tiers?** The scan loop iterates `rules_hot` (the `Vec<RuleCompiled>`) for every merged window. That loop touches `re`, `must_contain`, and `rule_meta` on every candidate, then gate indices only when a gate is present. Cold metadata -- the human-readable rule name, the minimum confidence threshold -- is only needed when a finding survives all gates and is about to be emitted. Storing cold metadata inline would inflate `RuleCompiled` beyond a cache-line pair and waste cache capacity on data read once per emitted finding, not once per candidate window.

### 5.1 Gate Pooling with the NO_GATE Sentinel

Each gate field on `RuleCompiled` is a `u32` index into a type-specific pool on `Engine`. The sentinel `NO_GATE` (`u32::MAX`) means "this rule has no gate of this type":

```rust
pub(super) const NO_GATE: u32 = u32::MAX;
```

Using `u32::MAX` instead of `Option<u32>` saves 4 bytes per gate field. `Option<u32>` has no niche optimization for `u32::MAX`, so it requires an explicit discriminant plus padding. Across eight gate fields, the sentinel approach saves approximately 32 bytes per rule -- the difference between fitting in a cache-line pair and spilling into a third.

The pool resolution pattern is uniform across all gate types. From `engine/core.rs`:

```rust
#[inline(always)]
pub(super) fn confirm_all_gate(&self, idx: u32) -> Option<&ConfirmAllCompiled> {
    if idx == NO_GATE {
        None
    } else {
        Some(&self.confirm_all_gates[idx as usize])
    }
}
```

Valid pool indices never reach `u32::MAX` because each pool has at most one entry per rule, and the rule count is bounded well below `u32::MAX` by practical memory limits. Engine construction asserts this:

```rust
assert!(
    rules.len() <= (1usize << 24),
    "rule count {} exceeds 24-bit dedupe key budget",
    rules.len()
);
```

### 5.2 Packed Rule Metadata

The `rule_meta` field on `RuleCompiled` packs several per-rule flags and the optional `secret_group` index into a single `u32` to avoid padding:

```text
bits  0..=15  secret_group value (meaningful only when bit 17 is set)
bit   16      needs_assignment_shape_check
bit   17      has_secret_group_override
bit   18      uuid_format_secret
bits 19..=31  reserved (must be zero)
```

A dedicated "has" bit (`RULE_META_HAS_SECRET_GROUP`) disambiguates `secret_group: None` from `secret_group: Some(u16::MAX)`, since the value `0xFFFF` would otherwise collide with the all-ones mask when the group is absent. This micro-optimization matters because `rule_meta` is read on every candidate window, and the alternative -- a separate `Option<u16>` field -- would add 4 bytes of padding to `RuleCompiled`.

## 6. Gate Compilation -- Variant-Indexed Arrays

Gate objects that contain literal patterns must work across three encoding variants: raw bytes, UTF-16LE, and UTF-16BE. The `Variant` enum from `rule_repr.rs` provides stable array indices:

```rust
pub(super) enum Variant {
    Raw,
    Utf16Le,
    Utf16Be,
}

impl Variant {
    pub(super) fn idx(self) -> usize {
        match self {
            Variant::Raw => 0,
            Variant::Utf16Le => 1,
            Variant::Utf16Be => 2,
        }
    }
}
```

Compiled gates like `TwoPhaseCompiled` and `KeywordsCompiled` store pattern data in `[PackedPatterns; 3]` arrays indexed by `Variant::idx()`. This avoids runtime dispatch and encoding conversion in the scan loop -- the correct encoding is selected with a single array index.

Patterns are stored in a `PackedPatterns` struct that keeps all patterns contiguous for cache-friendly `memmem` checks:

```rust
pub(super) struct PackedPatterns {
    pub(super) bytes: Box<[u8]>,
    pub(super) offsets: Box<[u32]>,
}
```

Pattern `i` is `bytes[offsets[i]..offsets[i+1]]`. The `offsets` array is a prefix-sum table with length `patterns + 1`. This avoids a `Vec<Vec<u8>>` and the per-pattern heap allocations it would entail.

Value suppressors are the exception to the three-variant rule. They run on extracted secret bytes (always in decoded byte space, never raw UTF-16 window bytes), so they are compiled as raw-only `PackedPatterns`.

### 6.1 The PackedPatternsBuilder

Patterns are assembled through a builder that maintains the prefix-sum invariant during construction. From `rule_repr.rs`:

```rust
pub(super) struct PackedPatternsBuilder {
    bytes: Vec<u8>,
    offsets: Vec<u32>,
}
```

Each `push_raw`, `push_utf16le`, or `push_utf16be` call appends bytes and records a new offset. The builder freezes into an immutable `PackedPatterns` via `build()`, which converts `Vec` storage to `Box<[T]>` (no excess capacity). The typical lifecycle during `compile_rule`:

```rust
let keywords = spec.keywords_any.map(|kws| {
    let count = kws.len();
    let raw_bytes = kws.iter().map(|p| p.len()).sum::<usize>();
    let utf16_bytes = raw_bytes.saturating_mul(2);

    let mut raw = PackedPatternsBuilder::with_capacity(count, raw_bytes);
    let mut le = PackedPatternsBuilder::with_capacity(count, utf16_bytes);
    let mut be = PackedPatternsBuilder::with_capacity(count, utf16_bytes);

    for &p in kws {
        raw.push_raw(p);
        le.push_utf16le(p);
        be.push_utf16be(p);
    }

    KeywordsCompiled {
        any: [raw.build(), le.build(), be.build()],
    }
});
```

The UTF-16 expansion is ASCII-only: each byte `b` becomes `[b, 0x00]` (LE) or `[0x00, b]` (BE). This is not a general-purpose UTF-16 encoder -- it produces technically invalid UTF-16 for non-ASCII input, but the engine only uses it for literal gating on ASCII anchor patterns.

### 6.2 The ConfirmAllCompiled Gate

The `ConfirmAllCompiled` gate is a special case. Its literals are derived from regex analysis *after* `compile_rule` returns, so the caller builds it separately via `compile_confirm_all`. The gate uses a two-phase check strategy:

1. **Primary literal** (longest): checked first via a single `memmem` search. The longest mandatory literal provides the highest rejection rate and makes the common negative case fast.
2. **Rest literals**: checked with AND semantics (all must match) using `PackedPatterns` iteration.

From `rule_repr.rs`:

```rust
pub(super) fn compile_confirm_all(mut confirm_all: Vec<Vec<u8>>) -> Option<ConfirmAllCompiled> {
    if confirm_all.is_empty() {
        return None;
    }

    confirm_all.sort_unstable_by(|a, b| b.len().cmp(&a.len()).then_with(|| a.cmp(b)));
    let primary = confirm_all.remove(0);
    let primary_raw: Option<Box<[u8]>> = Some(primary.clone().into_boxed_slice());
    let primary_le: Option<Box<[u8]>> = Some(utf16le_bytes(&primary).into_boxed_slice());
    let primary_be: Option<Box<[u8]>> = Some(utf16be_bytes(&primary).into_boxed_slice());
    // ... build rest patterns for remaining literals ...
}
```

Sorting is longest-first with ties broken lexicographically. The tiebreaker ensures deterministic primary selection across compilations when multiple literals share the same length. UTF-16 variants are encoded the same way as anchors and keywords so the gate can reject windows on raw bytes without decoding.

### 6.3 The CompiledGates Transient Bag

`compile_rule` returns a `CompiledGates` bag alongside the `RuleCompiled`:

```rust
pub(super) struct CompiledGates {
    pub(super) two_phase: Option<TwoPhaseCompiled>,
    pub(super) keywords: Option<KeywordsCompiled>,
    pub(super) value_suppressors: Option<PackedPatterns>,
    pub(super) entropy: Option<EntropyCompiled>,
    pub(super) char_class: Option<CharClassCompiled>,
    pub(super) local_context: Option<LocalContextSpec>,
    pub(super) offline_validation: Option<OfflineValidationSpec>,
}
```

This bag is transient: `Engine::new` pools each `Some` variant into the corresponding type-specific `Vec`, obtains a `u32` pool index, and patches it onto `RuleCompiled`. After pooling, the bag is dropped. The `confirm_all` field is intentionally absent -- its literals come from regex analysis that runs between the two compilation passes.

## 7. Anchor Policy -- Choosing How to Prefilter

The `AnchorPolicy` enum controls how the engine selects anchor patterns during compilation. From `api.rs`:

```rust
pub enum AnchorPolicy {
    /// Prefer derived anchors, falling back to manual anchors if derivation fails.
    PreferDerived,
    /// Only use manual anchors; skip derivation.
    ManualOnly,
    /// Only use derived anchors; ignore manual anchors entirely.
    DerivedOnly,
}
```

**`PreferDerived`** is the default. The engine runs `compile_trigger_plan` on each rule's regex to extract literal prefixes and infixes. If extraction succeeds, the derived anchors replace manual ones, and any mandatory literal islands are compiled into a `ConfirmAllCompiled` gate. If extraction fails (unsupported regex features), the engine falls back to manual anchors.

**`ManualOnly`** skips derivation entirely. This is appropriate when regexes are complex and derivation produces low-selectivity anchors (e.g., single-byte literals that match too many positions).

**`DerivedOnly`** ignores manual anchors. Rules that cannot be gated are reported via `Engine::unfilterable_rules()` as `(rule_index, UnfilterableReason)` pairs. These rules require full-buffer validation on every scan.

The anchor derivation pass also decides per-rule whether to include the raw regex in the Vectorscan prefilter database:

```text
Include regex  -> Vectorscan can match directly, but DB is larger.
Omit regex     -> Rely on literal anchor patterns alone, cheaper but
                  only works if anchors are strong enough.
```

A rule omits its raw regex when all of: it has at least one anchor pattern, every anchor is at least 5 bytes, and the regex is not case-insensitive. The 5-byte threshold is empirically chosen -- below it, Vectorscan's Aho-Corasick automaton produces high false-positive rates.

### 7.1 Pattern Deduplication and Deterministic Ordering

Multiple rules may share the same anchor byte pattern. The engine deduplicates patterns through a shared `AHashMap<Vec<u8>, Vec<Target>>`, where each `Target` packs a (rule_id, variant) pair into a single `u32`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub(super) struct Target(u32);
```

The low 2 bits hold the variant tag; the upper 30 bits hold the rule ID. The `map_to_patterns` function flattens this map into three parallel arrays consumed by the Vectorscan compiler:

- `patterns[i]`: the i-th unique anchor pattern (sorted lexicographically).
- `flat_targets[offsets[i]..offsets[i+1]]`: the `Target` entries for pattern `i`.
- `offsets`: prefix-sum index with length `patterns.len() + 1`.

Because Vectorscan assigns pattern IDs positionally (pattern 0 gets ID 0), the lexicographic sort is essential: it guarantees the same anchor bytes always receive the same ID, regardless of `AHashMap` iteration order. Without stable pattern-ID assignment, a Vectorscan hit callback would route to the wrong (rule, variant) accumulator whenever the hash map reorders entries between compilations.

## 8. Confidence Threshold Derivation

Each rule has an effective minimum confidence threshold stored in `RuleCold::min_confidence`. The function `derive_min_confidence` in `rule_repr.rs` computes it:

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

The cascade has clear priority: explicit override wins, then keyword-plus-entropy (threshold 3), then assignment-shape (threshold 2), then zero. Offline validation is deliberately excluded from auto-derivation because the `OFFLINE_VALID` signal (+5) is only scored on root-semantic findings, so transform-derived findings can never earn it. A per-rule threshold of 5 would silently block all valid transform-derived findings.

## 9. Engine Construction -- Putting It All Together

`Engine::new_with_anchor_policy` in `engine/core.rs` orchestrates the full compilation. The construction proceeds in two passes:

**Pass 1 -- Compile rules and pool gates.** For each `RuleSpec`, `compile_rule` produces a `(RuleCompiled, CompiledGates)` pair. The engine pools each `Some` gate into the corresponding type-specific `Vec` and patches the resulting `u32` index back onto `RuleCompiled`:

```rust
for spec in rules.iter() {
    let (mut rule, gates) = compile_rule(spec);
    if let Some(tp) = gates.two_phase {
        assert!(two_phase_gates.len() < NO_GATE as usize);
        rule.two_phase = two_phase_gates.len() as u32;
        two_phase_gates.push(tp);
    }
    // ... same pattern for keywords, entropy, char_class, etc.
}
```

**Pass 2 -- Build prefilter databases.** The engine derives or selects anchor patterns per the policy, builds deduped pattern maps for raw and UTF-16 variants, and constructs up to five Vectorscan databases: the unified prefilter DB (always present), the UTF-16 block-mode DB, the UTF-16 stream DB, the decoded-stream DB, and the decoded-space gate DB.

Between the two passes, the engine warms regex caches to avoid lazy DFA allocation on the first real scan:

```rust
let warm = [0u8; 1];
for rule in &rules_compiled {
    let mut it = rule.re.find_iter(&warm);
    let _ = it.next();
}
```

This forces the internal DFA cache allocation during construction so the hot path pays zero allocation cost.

## 10. The Tuning Struct -- Bounding Worst-Case Cost

Every capacity knob in the engine is expressed through the `Tuning` struct. From `api.rs`:

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

Every cap exists to bound worst-case CPU or memory cost. When a cap is exceeded, work is dropped (not queued) -- the engine favors bounded latency over completeness. `merge_gap` controls how aggressively adjacent anchor-hit windows are merged into single validation windows: larger values reduce the number of regex evaluations but widen the window fed to the regex. `max_anchor_hits_per_rule_variant` is a safety valve: if a single (rule, anchor-variant) pair produces more raw anchor hits than this before merging, all hits are collapsed into one range spanning the first to last hit, preventing pathological O(n^2) merge behavior on adversarial input.

`Tuning::assert_valid` checks a subset of invariants at build time. Additional cross-field checks are performed by `Engine::new`:

```rust
assert!(
    tuning.max_transform_depth.saturating_add(1) <= MAX_DECODE_STEPS,
    "max_transform_depth exceeds MAX_DECODE_STEPS"
);
```

This ensures the decode-step chain has room for the root step plus the configured transform depth.

## Summary

The rule compilation pipeline transforms human-authored YAML into a machine-executable scan configuration through three stages: YAML deserialization, progressive regex compilation with per-rule size tiers, and gate pooling with the `NO_GATE` sentinel. The two-tier `RuleCompiled` / `RuleCold` split keeps the hot scan loop compact -- 88 bytes per rule for iteration, with cold metadata consulted only at finding-emission time. Gate objects live in type-specific pools on `Engine`, accessed via `u32` indices that fit within the cache-line budget.

In [Chapter 3](03-memory-architecture.md), we will see how the engine allocates and reuses per-scan memory -- the `ScanScratch` with its cache-line-split layout, the `ScratchVec` that never reallocates, and the `DecodeSlab` whose stability invariant makes raw-pointer aliasing in the work-queue loop sound.
