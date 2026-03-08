# "Proving the Contract" -- Conformance Harness

*A new S3 connector passes all unit tests. It is deployed to production scanning 8,400 buckets. After two hours, the coordinator notices that resuming shard `s3://acme-data/[objects_m..objects_p]` from cursor position `objects_n/report-2024.csv` produces items that were already emitted before the checkpoint. The done-ledger filters most duplicates, but 23 findings with version ID `0xb7a3...` slip through because the file was modified between the first and second enumeration, producing a different version hash. Duplicate findings -- "AWS access key found in report-2024.csv" -- are reported to the customer twice, eroding trust. The root cause: the connector's token-based resume was off-by-one, re-emitting the item at the cursor position instead of advancing past it. The connector's 47 unit tests all passed because none tested resume from a mid-run checkpoint with token perturbation. A conformance check would have caught this in CI in under two seconds.*

---

## Why a Conformance Harness Exists

Unit tests verify individual behaviors. A connector can pass hundreds of unit tests and still violate the enumeration contract in subtle ways -- off-by-one resume, duplicate keys across pages, tokens that silently drift from key-based positioning, or item references that accidentally leak credentials. These failures only surface when the connector interacts with the coordinator's checkpoint and resume machinery in production.

The conformance harness in `crates/gossip-contracts/src/connector/conformance.rs` is a higher-order test framework that validates any `EnumerationConnector` implementation against the full enumeration contract. The page validation from Chapter 4 runs on every page collected by the harness. Both the in-memory connector (Chapter 6) and the filesystem connector (Chapter 7) must pass this harness. The harness accepts closures for connector construction and page enumeration, making it connector-agnostic: any type that can enumerate pages through a closure can be tested.

---

## ConformanceConfig: Strict-by-Default Knobs

Here is the definition from `conformance.rs`:

```rust
/// Conformance harness knobs.
///
/// Defaults are intentionally strict so connectors must explicitly opt out of
/// checks they cannot satisfy rather than silently passing weaker coverage.
///
/// ## Default strategy
///
/// | Field | Default | Rationale |
/// |-------|---------|-----------|
/// | `max_pages` | 10 000 | Generous ceiling; most connectors finish well below |
/// | `max_total_items` | 1 000 000 | Permits large datasets without unbounded memory |
/// | `page_budgets` | 32 items, unlimited bytes, no deadline | Small pages exercise cursor logic frequently |
/// | `require_strict_key_order` | `true` | Catches duplicate-key bugs early |
/// | `require_cursor_eq_last_item` | `true` | Verifies cursor bookkeeping tracks actual data |
/// | `determinism` | `Deterministic` | Strictest; connectors over mutable stores opt out |
/// | `resume_checks` | both enabled | Both token perturbation paths tested |
/// | `secret_scan` | enabled, default patterns | Catches common credential leakage |
/// | `restart_points` | `Auto(4)` | Four evenly-spaced resume points |
#[derive(Clone, Debug)]
pub struct ConformanceConfig {
    /// Hard cap on page requests per run. The harness stops and reports
    /// [`ConformanceError::TooManyPages`] when exceeded.
    pub max_pages: NonZeroUsize,
    /// Hard cap on collected items per run. The harness stops and reports
    /// [`ConformanceError::TooManyItems`] when exceeded.
    pub max_total_items: NonZeroUsize,
    /// [`Budgets`] forwarded to each `enumerate_page` call. The harness
    /// enforces `max_items` directly, while `max_bytes` and `deadline` are
    /// advisory hints for the connector/runtime.
    pub page_budgets: Budgets,
    /// When `true`, the harness requires strictly increasing keys within
    /// each page (no duplicate keys allowed). When `false`, non-decreasing
    /// order (as enforced by [`super::page_validator::validate_page_range`]) is
    /// sufficient.
    pub require_strict_key_order: bool,
    /// When `true`, non-empty pages must have
    /// `next_cursor.last_key == last_item.item_key`. This catches
    /// connectors that advance the cursor past the data they returned.
    pub require_cursor_eq_last_item: bool,
    /// Cross-run determinism policy. See [`DeterminismExpectation`].
    pub determinism: DeterminismExpectation,
    /// Which token-perturbation scenarios to exercise during resume checks.
    pub resume_checks: ResumeChecks,
    /// Secret-scan policy for connector `ItemRef` values.
    pub secret_scan: SecretScanConfig,
    /// How to select baseline cursor checkpoints for resume comparisons.
    pub restart_points: RestartPoints,
}
```

Field-by-field breakdown:

- **`max_pages: NonZeroUsize`** -- Safety valve against connectors that never terminate. Default: 10,000 pages. If the harness collects this many pages without seeing an empty terminal page, it reports `TooManyPages`.

- **`max_total_items: NonZeroUsize`** -- Memory budget guard. Default: 1,000,000 items. At 80 bytes per `ItemObservation`, this limits the `items` vector to approximately 76 MiB. The limit is checked after each item is pushed, using `>` (not `>=`) so that exactly `max_total_items` items are accepted.

- **`page_budgets: Budgets`** -- Forwarded to every `enumerate_page` call. Default: 32 items per page, unlimited bytes, no deadline. Small pages are deliberate: they force the connector to exercise its cursor and resume logic more frequently, increasing the probability of catching pagination bugs.

- **`require_strict_key_order: bool`** -- Default: `true`. When enabled, the harness checks that consecutive keys within a page are strictly increasing (not merely non-decreasing). This catches connectors that emit duplicate keys, which would cause the done-ledger to process the same item twice.

- **`require_cursor_eq_last_item: bool`** -- Default: `true`. Non-empty pages must return a `next_cursor` whose `last_key` matches the last item's key. This catches connectors that advance the cursor past the data they returned, which would cause the next page to skip items.

- **`determinism: DeterminismExpectation`** -- Default: `Deterministic`. Controls whether the harness performs a second full enumeration and compares it item-by-item against the baseline. Connectors over mutable data sources (e.g., a live S3 bucket) should set this to `BestEffort` to avoid spurious failures from concurrent writes.

- **`resume_checks: ResumeChecks`** -- Default: both `drop_token` and `corrupt_token` enabled. Controls which token perturbation scenarios are exercised during resume testing.

- **`secret_scan: SecretScanConfig`** -- Default: enabled with `DEFAULT_FORBIDDEN_ITEMREF_PATTERNS`. Scans every `ItemRef` for byte substrings that look like credentials.

- **`restart_points: RestartPoints`** -- Default: `Auto(4)`. Selects four evenly-spaced cursor checkpoints from the baseline run for resume testing. `Auto(0)` disables resume checks entirely.

The defaults are intentionally strict. A connector must explicitly opt out of checks it cannot satisfy -- for example, setting `determinism: BestEffort` for a connector over a mutable store. This inversion of responsibility (strict by default, lenient by opt-out) ensures that new connectors get maximum coverage without the author having to remember to enable each check.

---

## DeterminismExpectation

Here is the definition from `conformance.rs`:

```rust
/// Whether repeated full enumeration runs are expected to produce identical
/// item sequences.
///
/// The harness uses this to decide whether to compare a second full run
/// against the baseline. Connectors backed by mutable or eventually-consistent
/// data sources should use [`BestEffort`](Self::BestEffort) to avoid spurious
/// failures from concurrent writes.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum DeterminismExpectation {
    /// Repeated full scans must produce identical [`ItemObservation`]
    /// sequences. The harness compares a second run element-by-element and
    /// reports [`ConformanceError::DeterminismMismatch`] on the first
    /// divergence.
    Deterministic,
    /// Determinism is not strictly required by caller policy.
    ///
    /// The harness skips the comparison rerun entirely (no cross-run
    /// difference checks). This is appropriate for connectors over live,
    /// mutable data sources where concurrent modifications may change
    /// enumeration results between runs.
    BestEffort,
}
```

The filesystem connector uses `Deterministic` in its conformance tests because the directory snapshot is fixed for the lifetime of a connector instance (indexing is one-shot). A hypothetical S3 connector over a live bucket would use `BestEffort` because objects can be created or deleted between runs.

---

## ResumeChecks: Token Perturbation Flags

Here is the definition from `conformance.rs`:

```rust
/// Resume-by-key checks toggled by the conformance harness.
///
/// The conformance harness tests a connector's ability to resume enumeration
/// using only the cursor `last_key` (without a valid pagination token). Each
/// flag independently enables a token perturbation scenario:
///
/// - **`drop_token`**: the harness removes the cursor's `token` field
///   entirely, leaving only `last_key`. The connector must resume from the
///   key position without the token.
/// - **`corrupt_token`**: the harness replaces the cursor's `token` with
///   random bytes. The connector must either fall back to key-based resume
///   or reject the corrupted token gracefully.
///
/// Keeping the controls separate allows callers to narrow failures to one
/// recovery path at a time.
///
/// Both flags default to `true` so connectors must explicitly opt out.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ResumeChecks {
    /// When `true`, test resume with the pagination token removed entirely.
    pub drop_token: bool,
    /// When `true`, test resume with the pagination token replaced by random bytes.
    pub corrupt_token: bool,
}
```

The two flags are independent because they test different failure modes. `drop_token` simulates a cursor that was persisted without its token (e.g., an older checkpoint format). `corrupt_token` simulates token corruption from storage bit-flips or cross-version incompatibility. A connector that passes `drop_token` but fails `corrupt_token` has a different bug than one that fails both. Keeping the flags separate isolates the diagnosis.

---

## SecretScanConfig and Forbidden Patterns

Here is the definition from `conformance.rs`:

```rust
/// Best-effort secret scan configuration for [`ItemRef`] bytes.
///
/// The harness performs a byte-substring search over every `ItemRef` emitted
/// during an enumeration run. A match triggers
/// [`ConformanceError::ItemRefAppearsToContainSecret`] with digest-only
/// diagnostics (no raw bytes are exposed).
///
/// This is a heuristic leak-detection surface. It is intentionally configurable
/// so tests can tune false-positive/false-negative trade-offs per connector.
/// Defaults enable scanning with [`DEFAULT_FORBIDDEN_ITEMREF_PATTERNS`] and
/// no additional canaries.
#[derive(Clone, Debug)]
pub struct SecretScanConfig {
    /// Global on/off switch for secret scanning. When `false`, the harness
    /// skips all substring and canary checks.
    pub enabled: bool,
    /// Byte substrings that should not appear in connector item references.
    /// Defaults to [`DEFAULT_FORBIDDEN_ITEMREF_PATTERNS`]. Callers can
    /// narrow or extend this list per connector to manage false positives.
    pub forbidden_substrings: Vec<&'static [u8]>,
    /// Full-byte canaries injected by tests for connector-specific leak
    /// checks. Unlike `forbidden_substrings` (which match common credential
    /// prefixes), canaries are exact secret values that a test plants in the
    /// connector's credential store and then verifies do not appear in
    /// enumerated handles.
    pub secret_canaries: Vec<Vec<u8>>,
}
```

The default forbidden patterns cover common credential formats:

Here is the constant from `conformance.rs`:

```rust
pub const DEFAULT_FORBIDDEN_ITEMREF_PATTERNS: &[&[u8]] = &[
    b"Authorization:",
    b"Bearer ",
    b"AKIA",
    b"ASIA",
    b"x-amz-security-token",
    b"X-Amz-Signature=",
    b"X-Amz-Credential=",
    b"GoogleAccessId=",
    b"-----BEGIN ",
    // GitHub token prefixes.
    b"ghp_",
    b"gho_",
    b"ghs_",
    b"ghr_",
    // JWT Base64-encoded header prefix.
    b"eyJ",
    // Narrower PEM key markers (supplement the generic `-----BEGIN `).
    b"-----BEGIN RSA PRIVATE",
    b"-----BEGIN OPENSSH",
    // Database connection-string schemes.
    b"postgres://",
    b"mongodb+srv://",
];
```

The patterns are heuristic. `AKIA` matches AWS access key ID prefixes. `eyJ` matches the Base64 encoding of `{"` -- the start of a JWT header. `ghp_` through `ghr_` match GitHub's token prefix scheme. `forbidden_substrings` catches common credential prefixes; `secret_canaries` lets tests plant exact secret values and verify they do not leak into enumerated handles.

---

## ItemObservation: Digest-Only Snapshots

Here is the definition from `conformance.rs`:

```rust
/// Digest-only snapshot of one item observed during an enumeration run.
///
/// Two observations are compared element-by-element during determinism and
/// resume checks. The `key` digest identifies position (which item), while
/// `fingerprint` tracks the observable handle bytes for that item. A mismatch
/// in either field between runs constitutes a conformance failure.
///
/// `ItemObservation` is `Copy` (80 bytes: two [`ToxicDigest`] values at
/// 40 bytes each) and safe to store in bulk inside [`EnumerationTrace`].
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ItemObservation {
    /// BLAKE3 digest of the item's [`ItemKey`] bytes.
    pub key: ToxicDigest,
    /// BLAKE3 digest of the item's [`ItemRef`] bytes (computed by
    /// `observe_item`). This detects handle-level drift across runs
    /// for a stable key.
    pub fingerprint: ToxicDigest,
}
```

The observation stores only digests, never raw bytes. This serves two purposes. First, memory budget: at 80 bytes per observation, 1,000,000 items consume approximately 76 MiB, compared to potentially unbounded storage for raw key and ref bytes. Second, redaction: the `ToxicDigest` from Chapter 4 is the building block for `ItemObservation`. All error messages, log output, and debug representations use `ToxicDigest::Display`, which prints only the first 8 bytes as hex. Raw connector bytes never appear in conformance diagnostics.

The `key` digest identifies which item (position in the enumeration). The `fingerprint` digest tracks the observable handle bytes (`ItemRef`). A determinism mismatch occurs when either field differs between two runs at the same index.

---

## EnumerationTrace: Passive Record of One Run

Here is the definition from `conformance.rs`:

```rust
/// Trace captured from one full enumeration run.
///
/// The harness populates this incrementally: after each `enumerate_page` call
/// it appends one [`ItemObservation`] per returned item to `items` and pushes
/// the page's `next_cursor` onto `cursors`.
///
/// This is a passive record type: it does not enforce internal consistency
/// between `items` and `cursors`. In particular, `items.len()` may be less
/// than `cursors.len() * page_size` when pages return variable-length results.
///
/// ## Field relationships
///
/// - `cursors.len()` equals the total number of page responses (one cursor
///   checkpoint per page).
/// - `items` contains every observed item across all pages, in encounter
///   order. Resume checks slice into this vector using cursor checkpoint
///   boundaries.
///
/// ## Memory
///
/// `ItemObservation` is 80 bytes (two `ToxicDigest` fields). At default
/// config limits (`max_total_items = 1_000_000`), the `items` vector alone
/// can reach ~76 MiB. Additional allocations during a full harness run:
///
/// - `seen_keys: HashSet<ToxicDigest>` — ~69 MiB at 1M items (40-byte keys
///   plus hash-map overhead). Allocated per `collect_trace` call and dropped
///   when the call returns.
/// - `key_to_index: HashMap<ToxicDigest, usize>` — ~76 MiB for resume-check
///   baseline lookups.
/// - Determinism reruns allocate a second `EnumerationTrace` (~76 MiB),
///   dropped before resume checks begin.
///
/// Because intermediate allocations are dropped between phases, peak working
/// set with all checks enabled at default 1M-item limit is roughly
/// 220–300 MiB (not the sum of all components). `cursors` grows with
/// `max_pages` but is typically modest.
///
/// Callers should tune [`ConformanceConfig::max_pages`] and
/// [`ConformanceConfig::max_total_items`] when targeting connectors with
/// large datasets.
#[derive(Clone, Debug)]
pub struct EnumerationTrace {
    /// Item observations in encounter order across all pages.
    pub items: Vec<ItemObservation>,
    /// Cursor checkpoint after each page, in page order. `cursors[i]` is
    /// the `next_cursor` returned by the `i`-th `enumerate_page` call.
    /// Resume checks select restart points from this vector. The total
    /// page count equals `cursors.len()`.
    pub cursors: Vec<Cursor>,
}
```

The trace is a passive record. `items` holds observations in encounter order; `cursors` holds one checkpoint per page. Resume checks work by selecting a cursor from `cursors`, computing the expected suffix from `items`, then comparing the suffix against a fresh enumeration starting from the perturbed cursor.

---

## Restart Point Selection

The harness selects which cursor checkpoints to use for resume testing via the `RestartPoints` enum and `select_restart_points` function.

Here is the definition from `conformance.rs`:

```rust
/// Restart-point selection strategy for resume checks.
///
/// During a baseline enumeration run the harness records a cursor checkpoint
/// after each page (stored in [`EnumerationTrace::cursors`]). Resume checks
/// then pick a subset of those checkpoints, perturb the token, and re-run
/// enumeration from each selected cursor. This enum controls how the subset
/// is chosen.
///
/// Index validation and out-of-range handling are owned by the harness;
/// invalid indices are silently clamped or skipped.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum RestartPoints {
    /// Have the harness automatically choose up to `N` points spread
    /// evenly across the baseline run's cursor checkpoints. A value of
    /// `0` disables resume checks even when [`ResumeChecks`] flags are set.
    Auto(usize),
    /// Restart from specific cursor checkpoint indices (zero-based
    /// positions into [`EnumerationTrace::cursors`]).
    Explicit(&'static [usize]),
}
```

`Auto(4)` (the default) spreads four points evenly across the baseline's cursor checkpoints, always including the first and last when `N >= 2`. `Explicit` allows tests to target specific restart points for focused debugging. The selection function normalizes both strategies into sorted, unique, in-range indices:

```rust
fn select_restart_points(restart_points: RestartPoints, len: usize) -> Vec<usize> {
    match restart_points {
        RestartPoints::Auto(requested) => {
            if requested == 0 || len == 0 {
                return Vec::new();
            }

            let count = requested.min(len);
            if count == 1 {
                return vec![0];
            }

            let max_index = len - 1;
            let mut out = Vec::with_capacity(count);
            for i in 0..count {
                out.push(i * max_index / (count - 1));
            }
            out.sort_unstable();
            out.dedup();
            out
        }
        RestartPoints::Explicit(indices) => {
            let mut out: Vec<_> = indices.iter().copied().filter(|&idx| idx < len).collect();
            out.sort_unstable();
            out.dedup();
            out
        }
    }
}
```

For `Auto(4)` with 10 cursor checkpoints, the selected indices are `[0, 3, 6, 9]` -- the first, two interior points, and the last. The sort-and-dedup step handles the edge case where the spread formula produces duplicate indices (e.g., `Auto(100)` with 3 checkpoints yields `[0, 1, 2]` after dedup).

---

## The Entry Point: `check_connector_conforms`

Here is the definition from `conformance.rs`:

```rust
/// Execute the connector conformance harness with strict-by-default checks.
///
/// The harness performs:
///
/// 1. Capability gate (`seek_by_key` required)
/// 2. Baseline full enumeration trace collection
/// 3. Optional deterministic rerun + item-by-item comparison
/// 4. Optional resume checks from selected cursor checkpoints
/// 5. Optional per-item secret scan during collection passes
///
/// Returns `Ok(())` only when all enabled checks pass.
pub fn check_connector_conforms<C, Make, Caps, Enum>(
    make: Make,
    caps: Caps,
    enumerate_page: Enum,
    start: &ItemKey,
    end: &ItemKey,
    cfg: ConformanceConfig,
) -> Result<(), ConformanceError>
where
    Make: Fn() -> C,
    Caps: Fn(&C) -> ConnectorCapabilities,
    Enum: Fn(&mut C, &Cursor, Budgets) -> Result<EnumerationPage, EnumerateError>,
{
    let mut baseline_connector = make();
    let connector_caps = caps(&baseline_connector);
    if !connector_caps.seek_by_key {
        return Err(ConformanceError::CapabilityMissingSeekByKey {
            caps: connector_caps,
        });
    }

    let baseline = collect_trace(
        &mut baseline_connector,
        &enumerate_page,
        start,
        end,
        &cfg,
        Cursor::initial(),
    )?;

    if matches!(cfg.determinism, DeterminismExpectation::Deterministic) {
        let mut rerun_connector = make();
        let rerun = collect_trace(
            &mut rerun_connector,
            &enumerate_page,
            start,
            end,
            &cfg,
            Cursor::initial(),
        )?;
        check_same_items(&baseline.items, &rerun.items)?;
    }

    run_resume_checks(&make, &enumerate_page, start, end, &cfg, &baseline)?;
    Ok(())
}
```

The function accepts four closures, not a trait object:

- **`make: Fn() -> C`** -- Factory that constructs a fresh connector instance. Called once for the baseline, once for the determinism rerun (if enabled), and once per resume check. Each invocation gets an independent connector, ensuring that mutable state from one run does not contaminate another.
- **`caps: Fn(&C) -> ConnectorCapabilities`** -- Returns the connector's advertised capabilities.
- **`enumerate_page: Fn(&mut C, &Cursor, Budgets) -> Result<EnumerationPage, EnumerateError>`** -- The enumeration function under test.
- **`start` / `end`** -- The shard key range to enumerate.

The closure-based design avoids requiring connectors to implement a specific trait just for testing. The filesystem connector's test wires it as:

```rust
check_connector_conforms(
    || make_conn(root.clone()),
    |c| c.caps(),
    |c, cursor, budgets| c.enumerate_page_range(&start, &end, cursor, budgets),
    &start,
    &end,
    config,
)
```

### Phased Execution

The harness executes five phases in sequence:

**Phase 1: Capability gate.** The harness requires `seek_by_key`. Without key-based seeking, cursor-based resume is impossible and the harness cannot verify the core contract.

**Phase 2: Baseline trace collection.** A fresh connector is created via `make()`. The harness calls `enumerate_page` in a loop, collecting `ItemObservation` digests and cursor checkpoints into an `EnumerationTrace`. Each page undergoes per-page validation (via `validate_page_range` from Chapter 4), strict ordering checks, cursor-last-item matching, duplicate-key detection, and secret scanning. The loop terminates when an empty page is returned.

**Phase 3: Determinism rerun (optional).** If `cfg.determinism` is `Deterministic`, a second connector is created and enumerated identically. The two traces are compared element-by-element via `check_same_items`. Length mismatches produce `DeterminismLengthMismatch`; element mismatches produce `DeterminismMismatch` at the first divergence.

**Phase 4: Resume checks (optional).** The harness selects restart points from the baseline's cursor checkpoints (via `select_restart_points`). For each restart point and each enabled `ResumeMode`, a fresh connector is created, the cursor is perturbed (token dropped or corrupted), and enumeration resumes from the perturbed cursor. The resulting items are compared against the expected suffix from the baseline trace. Length mismatches produce `ResumeLengthMismatch`; element mismatches produce `ResumeMismatch`.

**Phase 5: Secret scanning (inline).** Secret scanning happens during Phases 2 and 4, not as a separate phase. Each `ItemRef` is checked against `forbidden_substrings` and `secret_canaries` as it is collected. A match produces `ItemRefAppearsToContainSecret` with digest-only diagnostics.

---

## ConformanceError: The Failure Taxonomy

The `ConformanceError` enum has 15 variants, organized by the harness execution phase where they occur. Here is the definition from `conformance.rs`:

```rust
// Intentionally exhaustive — no `#[non_exhaustive]` per no-versioning policy.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ConformanceError {
    /// Connector does not advertise required `seek_by_key` capability.
    CapabilityMissingSeekByKey {
        caps: ConnectorCapabilities,
    },
    /// `enumerate_page` returned an operational error.
    EnumerateFailed {
        at_page: usize,
        class: ErrorClass,
        message_digest: ToxicDigest,
    },
    /// Connector returned more items than the configured page bound.
    ReturnedTooManyItems {
        at_page: usize,
        got: usize,
        max: usize,
    },
    /// Page-level structural violation detected by
    /// [`validate_page_range`].
    PageValidationFailed {
        at_page: usize,
        err: PageValidationError,
    },
    /// Secret-scan heuristic found a forbidden pattern inside an `ItemRef`.
    ItemRefAppearsToContainSecret {
        pattern: ToxicDigest,
        item_ref: ToxicDigest,
    },
    /// Adjacent keys within a page are equal (non-decreasing but not
    /// strictly increasing).
    NotStrictlyIncreasingWithinPage {
        at_page: usize,
        at_index: usize,
        prev_key: ToxicDigest,
        next_key: ToxicDigest,
    },
    /// Non-empty page returned a cursor key that differs from the last
    /// item key.
    CursorDoesNotMatchLastItem {
        at_page: usize,
        cursor_key: Option<ToxicDigest>,
        last_item_key: ToxicDigest,
    },
    /// Key digest appeared more than once in a single run.
    DuplicateKeyInRun {
        key: ToxicDigest,
    },
    /// Run exceeded configured `max_pages`.
    TooManyPages {
        max_pages: usize,
    },
    /// Run exceeded configured `max_total_items`.
    TooManyItems {
        max_total_items: usize,
    },
    /// Two full enumeration runs produced different [`ItemObservation`]s at
    /// the same index.
    DeterminismMismatch {
        at_index: usize,
        run1: ItemObservation,
        run2: ItemObservation,
    },
    /// Two full enumeration runs produced different numbers of items.
    DeterminismLengthMismatch {
        expected_len: usize,
        got_len: usize,
    },
    /// A resume run from a baseline cursor checkpoint produced a different
    /// item at the same suffix position.
    ResumeMismatch {
        restart_cursor_index: usize,
        mode: ResumeMode,
        at_index: usize,
        expected: ItemObservation,
        got: ItemObservation,
    },
    /// A resume run produced a different total number of items than the
    /// baseline suffix.
    ResumeLengthMismatch {
        restart_cursor_index: usize,
        mode: ResumeMode,
        expected_len: usize,
        got_len: usize,
    },
    /// A resume checkpoint's `last_key` does not match any item observed
    /// in the baseline run.
    CursorKeyNotInBaseline {
        cursor_key: ToxicDigest,
    },
}
```

The variants group by phase:

- **Capability gate** (1 variant): `CapabilityMissingSeekByKey`.
- **Per-page collection** (5 variants): `EnumerateFailed`, `ReturnedTooManyItems`, `PageValidationFailed`, `ItemRefAppearsToContainSecret`, `NotStrictlyIncreasingWithinPage`.
- **Cross-page checks** (2 variants): `CursorDoesNotMatchLastItem`, `DuplicateKeyInRun`.
- **Run-level caps** (2 variants): `TooManyPages`, `TooManyItems`.
- **Cross-run comparisons** (2 variants): `DeterminismMismatch`, `DeterminismLengthMismatch`.
- **Resume checks** (3 variants): `ResumeMismatch`, `ResumeLengthMismatch`, `CursorKeyNotInBaseline`.

Every variant carries enough context to diagnose the failure without re-running the harness. Page indices, item indices, mode indicators, and `ToxicDigest` values are all included. The enum is constrained to 256 bytes by a compile-time assertion, justifying the `#[allow(clippy::result_large_err)]` on the public entry point. Note that `ConformanceError` deliberately does **not** carry `#[non_exhaustive]` -- the source code comment at `conformance.rs:416` reads "Intentionally exhaustive — no `#[non_exhaustive]` per no-versioning policy."

All error diagnostics use `ToxicDigest` exclusively. The connector's raw bytes -- which may contain credential material, customer data, or binary content -- never appear in error messages, log output, or debug representations.

---

## Token Perturbation Helpers

Here are the definitions from `conformance.rs`:

```rust
fn drop_token(cursor: &Cursor) -> Cursor {
    match cursor.last_key() {
        Some(last_key) => Cursor::with_last_key(last_key.clone()),
        None => Cursor::initial(),
    }
}

fn corrupt_token(cursor: &Cursor, seed: u64) -> Cursor {
    let Some(last_key) = cursor.last_key() else {
        return Cursor::initial();
    };

    // Deterministic mutation keeps failures reproducible across runs.
    let hash = blake3::hash(&seed.to_le_bytes());
    let token = TokenBytes::try_from_slice(&hash.as_bytes()[..16])
        .expect("blake3 16-byte token is always valid for TokenBytes");
    Cursor::with_token(last_key.clone(), token)
}
```

`drop_token` strips the cursor's token field, leaving only the `last_key`. This tests the connector's ability to resume from key position alone.

`corrupt_token` replaces the token with 16 bytes derived from a BLAKE3 hash of the seed (the restart cursor index). The corruption is deterministic: the same seed always produces the same corrupted token, making failures reproducible across test runs. The connector must either detect the invalid token and fall back to key-based resume, or reject the corrupted token gracefully (returning an error that the harness would propagate as `EnumerateFailed`).

---

## Memory Budget

The doc comment on `EnumerationTrace` provides a detailed memory analysis. At default config limits (1,000,000 items, 10,000 pages):

- **`items` vector**: 80 bytes per `ItemObservation` x 1M items = ~76 MiB.
- **`seen_keys` HashSet**: 40-byte `ToxicDigest` keys + hash-map overhead = ~69 MiB. Allocated during `collect_trace` and dropped when the function returns.
- **`key_to_index` HashMap**: ~76 MiB for resume-check baseline lookups. Allocated during `run_resume_checks` and dropped when the function returns.
- **Determinism rerun**: A second `EnumerationTrace` = ~76 MiB. Dropped before resume checks begin.

Because intermediate allocations are dropped between phases, peak working set is roughly 220--300 MiB with all checks enabled at the 1M-item limit. Callers targeting connectors with large datasets should tune `max_pages` and `max_total_items` to fit within their memory budget.

---

## The Higher-Order Design

The harness accepts closures rather than trait objects for three reasons:

1. **No test-only trait required.** Connectors implement `EnumerationConnector` and `ReadConnector`. The harness does not force them to implement a third "conformance-testable" trait. Any type that can be wired through closures is testable.

2. **Fresh instances per run.** The `make` closure is called once per run (baseline, rerun, each resume check). This guarantees that mutable state from one run (e.g., the filesystem connector's cached index) does not contaminate another. The filesystem connector's test demonstrates this: `|| make_conn(root.clone())` constructs a fresh `FilesystemConnector` for each invocation.

3. **Flexible wiring.** The `enumerate_page` closure can adapt between the connector's method signature and the harness's expectations. The filesystem connector's `enumerate_page_range` takes explicit start/end keys, while the `ShardSpec`-based `enumerate_page` from the trait takes a shard reference. The closure captures the start/end keys and forwards them.

---

## Summary

The conformance harness in `conformance.rs` validates any enumeration connector against the full contract through phased execution: capability gate, baseline trace collection with per-page validation, optional deterministic rerun, optional resume checks with token perturbation, and inline secret scanning. `ConformanceConfig` provides strict-by-default knobs that connectors must explicitly relax. `ItemObservation` stores digest-only snapshots at 80 bytes each, and `EnumerationTrace` records one full run's observations and cursor checkpoints. The 15-variant `ConformanceError` enum covers every failure mode from capability gates through resume divergence, using `ToxicDigest` exclusively for safe diagnostics. Token perturbation helpers (`drop_token`, `corrupt_token`) test the connector's resilience to missing or corrupted pagination state. Chapter 9 examines the Git connector -- a third concrete implementation alongside the in-memory and filesystem connectors. Chapter 10 covers the scan-driver adapters that bridge all three connectors to the new `ScanDriver` execution model.
