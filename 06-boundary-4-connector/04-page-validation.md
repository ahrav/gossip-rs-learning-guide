# "Ten Invariants, Zero Allocations" -- Page Validation

*A connector enumerating an S3 bucket emits a page containing three items.
The first item has key `[0x41, 0x42]` (lexicographic "AB"). The second has
key `[0x41, 0x43]` ("AC"). The third has key `[0x41, 0x41]` ("AA") -- an
object that was uploaded mid-enumeration and appeared in the listing out of
sort order. Without validation, the scan loop checkpoints the cursor at the
third item's key, `[0x41, 0x41]`. On the next page request, the connector
seeks past `[0x41, 0x41]` and begins listing from `[0x41, 0x42]`. Items
`[0x41, 0x42]` and `[0x41, 0x43]` are returned again -- duplicates. But
item `[0x41, 0x41]` is never re-emitted, because the cursor has already
advanced past it. The scan loop has no idea an item was skipped. It
completes the shard successfully. The item at key `[0x41, 0x41]` -- a
3.2 MB configuration file potentially containing database credentials --
is silently lost. The scanner will never inspect it for secrets.*

---

## Why Page Validation Exists

The failure above is a connector contract violation: items must arrive in
non-decreasing key order. But the scan loop cannot trust connectors to honor
this contract. Connectors wrap external systems -- object stores, version
control APIs, file systems -- that may have their own consistency quirks
during concurrent mutation. A single out-of-order item, an off-by-one cursor,
or a cursor that moves backward between pages can silently corrupt the
scanner's view of the data source.

The `gossip-contracts::connector::page_validator` module provides a pure
validation function that checks every page against ten ordered invariants
before the scan loop acts on it. The `ItemKey` ordering from Chapter 2 is
the foundation these invariants enforce. Chapter 3's
Connector `enumerate_page` methods produce the pages this validator
checks.

---

## ToxicDigest -- Redacted Hash Representation

Connector keys and cursors are untrusted external data. They may contain
file paths, repository names, user identifiers, or other customer-sensitive
content. Diagnostic output from the validator must never include raw byte
payloads. Instead, every byte value that appears in an error message is
first converted to a `ToxicDigest`: a fixed-size hash representation that
preserves enough information for log-line correlation without leaking
content.

Here is the definition from `page_validator.rs`:

```rust
/// A hash-only, fixed-size representation of toxic bytes.
///
/// Connector keys and cursors are untrusted external data ("toxic bytes")
/// that must never appear raw in logs, error messages, or metrics. Instead
/// of carrying the original payload, `ToxicDigest` stores just:
///
/// - `len`: the byte length of the original input, and
/// - `hash`: a full 32-byte BLAKE3 digest.
///
/// ## Display format
///
/// Both `Display` and `Debug` emit the same compact, single-line form:
///
/// ```text
/// len=42, hash=0a1b2c3d4e5f6a7b
/// ```
///
/// Only the first 8 bytes of the hash are printed (16 hex characters),
/// which is enough for log-line correlation without bloating output.
///
/// ## Equality semantics
///
/// `PartialEq` and `Eq` compare the **full** 32-byte hash (plus length),
/// not just the truncated display prefix. Two digests that display
/// identically are overwhelmingly likely to be equal, but equality is
/// always decided on the complete hash.
///
/// ## Copy semantics
///
/// `ToxicDigest` is `Copy` (40 bytes on 64-bit targets). Error types in
/// this module embed digests by value, avoiding indirection for what are
/// fundamentally small, immutable tokens.
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub struct ToxicDigest {
    /// Byte length of the original input. Preserved so diagnostics can
    /// distinguish zero-length sentinels from short keys without revealing
    /// content.
    len: usize,
    /// Full 32-byte BLAKE3 digest of the original input. Only the first
    /// 8 bytes are shown in display output; equality uses all 32.
    hash: [u8; 32],
}
```

Field-by-field:

- **`len: usize`** -- The byte length of the original input. This is
  preserved separately from the hash because two keys can have different
  lengths but collide on their hash prefix. The length also lets diagnostics
  distinguish a zero-length sentinel (the empty-start boundary) from a short
  key without revealing content.

- **`hash: [u8; 32]`** -- A full 32-byte BLAKE3 digest. Equality uses all
  32 bytes. Display uses only the first 8 bytes (16 hex characters). The
  asymmetry is deliberate: display needs to be compact for multi-field error
  lines, but equality needs full collision resistance.

The two constructors mirror the two ways callers access byte content:

Here is the definition from `page_validator.rs`:

```rust
impl ToxicDigest {
    /// Digest a raw byte slice into a redacted, fixed-size representation.
    ///
    /// This is the low-level entry point. Prefer [`of`](Self::of) when working
    /// with typed wrappers like [`ItemKey`] that implement `AsRef<[u8]>`.
    #[must_use]
    pub fn of_bytes(bytes: &[u8]) -> Self {
        let hash = blake3::hash(bytes);
        Self {
            len: bytes.len(),
            hash: *hash.as_bytes(),
        }
    }

    /// Digest any value that can be viewed as bytes.
    ///
    /// This is the ergonomic entry point for typed connector keys and cursors.
    /// The `?Sized` bound allows passing both owned wrappers (`&ItemKey`) and
    /// plain slices (`&[u8]`) without an extra `.as_ref()` at the call site.
    #[must_use]
    pub fn of<K: AsRef<[u8]> + ?Sized>(k: &K) -> Self {
        Self::of_bytes(k.as_ref())
    }
}
```

The `Display` and `Debug` implementations both produce the same compact
format:

Here is the definition from `page_validator.rs`:

```rust
impl fmt::Display for ToxicDigest {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "len={}, hash=", self.len)?;
        // First 8 bytes => 16 hex characters. Enough for correlation, short
        // enough that multi-field error lines stay readable.
        for b in &self.hash[..8] {
            write!(f, "{b:02x}")?;
        }
        Ok(())
    }
}

/// Delegates to `Display` so that `{:?}` formatting in error chains and
/// `Option<ToxicDigest>` debug output stays log-safe and compact.
impl fmt::Debug for ToxicDigest {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        fmt::Display::fmt(self, f)
    }
}
```

A concrete example: `ToxicDigest::of(b"hello")` produces
`len=5, hash=ea8f163db38682925e4491c5e58d4bb773c3451e8f2f548c3bf838b4` (truncated
to 16 hex characters in display). The full 32-byte hash is used for equality;
the display prefix is used for log correlation. The `Debug` delegation to
`Display` ensures that `{:?}` output in error chains remains log-safe --
no accidental raw byte dumps through debug formatting.

---

## PageItem -- The Generic Key Accessor

The validator needs to compare item keys for ordering and range membership,
but it should not depend on the concrete `ScanItem` type. Test fixtures use
lightweight wrappers like `Vec<u8>` instead of full `ScanItem` values. The
`PageItem` trait abstracts over both.

Here is the definition from `page_validator.rs`:

```rust
/// A view of a page item that exposes its ordered key.
///
/// Validation logic needs to compare item keys for ordering and range
/// membership, but should not depend on the concrete item carrier. This trait
/// abstracts over both production [`ScanItem`] values and lightweight test
/// fixtures (e.g., a newtype around `Vec<u8>`), keeping the validator
/// generic.
///
/// `K` is the key type used for ordering comparisons -- typically [`ItemKey`]
/// in production, but tests may substitute a simpler ordered byte wrapper.
pub trait PageItem<K: ?Sized> {
    /// Returns a reference to the item's ordering key.
    fn item_key(&self) -> &K;
}
```

`ScanItem` implements `PageItem` twice -- once for `ItemKey` (the production
path) and once for `[u8]` (the byte-level path used by `validate_page`):

Here is the definition from `page_validator.rs`:

```rust
/// Delegates to [`ScanItem::item_key`], the inherent accessor.
impl PageItem<ItemKey> for ScanItem {
    fn item_key(&self) -> &ItemKey {
        self.item_key()
    }
}

/// Provides byte-level key access for [`validate_page`], which projects
/// cursor keys through `ItemKey::as_ref()` to `&[u8]` in order to unify
/// the `K` parameter across cursors and items. Without this impl the
/// generic `validate_page_range::<[u8], _>` instantiation cannot treat
/// `ScanItem` as a [`PageItem`].
impl PageItem<[u8]> for ScanItem {
    fn item_key(&self) -> &[u8] {
        ScanItem::item_key(self).as_ref()
    }
}
```

The dual impl exists because the `validate_page` adapter (shown later)
instantiates `validate_page_range::<[u8], ScanItem>`. Cursor keys are
projected to `&[u8]` through `ItemKey::as_ref()`, and item keys must be
projected to the same type for unified comparison. Without the `PageItem<[u8]>`
impl, the generic instantiation would not compile.

---

## PageValidationViolation -- The Thin Discriminants

Each page invariant has a corresponding violation variant. The violation enum
is a thin, `Copy`-able discriminant designed for programmatic matching and
metrics labeling. It carries no payload -- all diagnostic context lives in
the companion `PageValidationDetails` type.

Here is the definition from `page_validator.rs`:

```rust
/// The page-validation rule that was violated.
///
/// This is a thin, `Copy`-able discriminant designed for programmatic
/// matching and metrics labeling. It intentionally carries no payload --
/// all diagnostic context lives in the companion [`PageValidationDetails`]
/// variant inside [`PageValidationError`].
///
/// Variants are grouped by concern (spec, cursor membership, item checks,
/// and cursor progression). Runtime validation order is defined by
/// [`validate_page_range`], not by enum declaration order.
///
/// Marked `#[non_exhaustive]` so new page-contract rules can be added
/// without a breaking change to downstream `match` arms.
#[non_exhaustive]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum PageValidationViolation {
    /// The declared page spec bounds are internally inconsistent.
    SpecRangeInvalid,
    /// The input cursor key is outside the allowed cursor range.
    InputCursorOutOfRange,
    /// The next cursor key is outside the allowed cursor range.
    NextCursorOutOfRange,
    /// An emitted item key is outside the allowed item range.
    ItemKeyOutOfRange,
    /// Returned items are not in non-decreasing key order.
    ItemsNotOrdered,
    /// The first returned item does not advance strictly past the input cursor.
    ItemsNotAfterCursor,
    /// Items were returned but `next_cursor.last_key` was missing.
    NextCursorMissing,
    /// `next_cursor.last_key` is behind the final returned item key.
    NextCursorBehindLastItem,
    /// The continuation cursor moved backwards relative to the input cursor.
    CursorRegressed,
    /// An empty page changed cursor state when it should have remained stable.
    EmptyPageCursorAdvanced,
}
```

Ten variants, each a single-word discriminant. The `#[non_exhaustive]`
attribute means downstream code must include a wildcard arm in `match`
expressions, future-proofing against new invariant checks. The enum
declaration groups variants by concern (spec, cursor, items, progression),
but the runtime validation order is determined by the `validate_page_range`
function, not by declaration order.

---

## PageValidationDetails -- The Rich Diagnostics

Each violation is paired with a details variant that carries the minimum
context needed to diagnose the failure. All byte content is redacted through
`ToxicDigest`.

Here is the definition from `page_validator.rs`:

```rust
/// Redacted diagnostic details for page-validation violations.
///
/// Each variant carries the minimum context needed to diagnose a specific
/// class of failure: redacted key digests ([`ToxicDigest`]), positional
/// indices, and range bounds. No raw connector bytes are preserved.
///
/// Some variants serve multiple [`PageValidationViolation`] kinds. For
/// example, [`CursorOutOfRange`](Self::CursorOutOfRange) is used by both
/// `InputCursorOutOfRange` and `NextCursorOutOfRange`, distinguished by
/// the embedded [`CursorWhich`] field. This keeps the variant count small
/// while preserving full diagnostic specificity.
///
/// The range bound conventions follow the page spec contract:
/// - **Cursor ranges** are inclusive on both ends: `[start, end]`.
/// - **Item ranges** are inclusive-exclusive: `[start, end)`.
///
/// These conventions are reflected in the `Display` output of
/// [`PageValidationError`], which uses `[start, end]` for cursor ranges
/// and `[start, end)` for item ranges.
///
/// Marked `#[non_exhaustive]` alongside [`PageValidationViolation`] so
/// new detail shapes can be introduced with new violation rules.
#[non_exhaustive]
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum PageValidationDetails {
    /// Details for [`PageValidationViolation::SpecRangeInvalid`].
    Spec {
        /// Inclusive lower bound from the page spec.
        start: ToxicDigest,
        /// Upper bound from the page spec (exclusive for items, inclusive for cursors).
        end: ToxicDigest,
    },
    /// Details for cursor-bound checks.
    CursorOutOfRange {
        /// Which cursor violated the bound.
        which: CursorWhich,
        /// Offending cursor key digest.
        key: ToxicDigest,
        /// Inclusive lower bound for cursor keys.
        start: ToxicDigest,
        /// Inclusive upper bound for cursor keys.
        end: ToxicDigest,
    },
    /// Details for item-bound checks.
    ItemOutOfRange {
        /// Index of the first offending item in the page payload.
        index: usize,
        /// Offending item key digest.
        key: ToxicDigest,
        /// Inclusive lower bound for item keys.
        start: ToxicDigest,
        /// Exclusive upper bound for item keys.
        end: ToxicDigest,
    },
    /// Details for local ordering failures between adjacent items.
    ItemsNotOrdered {
        /// Index of the later item in the offending pair.
        index: usize,
        /// Digest of the previous item key.
        prev: ToxicDigest,
        /// Digest of the current item key.
        next: ToxicDigest,
    },
    /// Details for the "items must advance past input cursor" rule.
    ItemsNotAfterCursor {
        /// Input cursor digest.
        cursor: ToxicDigest,
        /// Digest of the first returned item key.
        first_item: ToxicDigest,
    },
    /// Details for `next_cursor.last_key` lagging behind returned data.
    NextCursorBehindLastItem {
        /// Digest of the returned continuation cursor key.
        next_cursor: ToxicDigest,
        /// Digest of the last item key in the page.
        last_item: ToxicDigest,
    },
    /// Details for cursor monotonicity failures.
    CursorRegressed {
        /// Input cursor digest.
        input: ToxicDigest,
        /// Next cursor digest.
        next: ToxicDigest,
    },
    /// Details for cursor movement on empty pages.
    EmptyPageCursorAdvanced {
        /// Input cursor digest (if present).
        input: Option<ToxicDigest>,
        /// Next cursor digest (if present).
        next: Option<ToxicDigest>,
    },
    /// Details for missing continuation cursor on non-empty pages.
    NextCursorMissing,
}
```

Nine variants (not ten) because `CursorOutOfRange` is shared between
`InputCursorOutOfRange` and `NextCursorOutOfRange`, distinguished by the
embedded `CursorWhich` discriminant:

Here is the definition from `page_validator.rs`:

```rust
/// Identifies which cursor a violation refers to.
///
/// This discriminant exists because
/// [`PageValidationDetails::CursorOutOfRange`] is shared between the
/// `InputCursorOutOfRange` and `NextCursorOutOfRange` violations. Embedding
/// `CursorWhich` in the details variant avoids duplicating the
/// `{key, start, end}` field set while preserving full diagnostic specificity.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum CursorWhich {
    /// The caller-provided input cursor for this page request.
    Input,
    /// The connector-returned continuation cursor for the next request.
    Next,
}
```

---

## PageValidationError -- The Combined Error

The violation discriminant and the diagnostic details are combined into a
single error type with private fields.

Here is the definition from `page_validator.rs`:

```rust
/// Structured page-validation error combining a rule discriminant with
/// redacted diagnostic context.
///
/// ## Two-field design
///
/// - `violation`: a `Copy`-able [`PageValidationViolation`] discriminant for
///   programmatic dispatch, metrics counters, and alert routing.
/// - `details`: a [`PageValidationDetails`] variant with the redacted byte
///   digests and indices needed to produce a human-readable error message.
///
/// ## Display output
///
/// `Display` produces a single-line, log-safe message that names the violated
/// rule and includes all redacted fields from the details variant. If the
/// `(violation, details)` pair is inconsistent (which should not happen under
/// normal use), the `Display` impl falls back to a generic message using the
/// violation's `Debug` representation rather than panicking.
///
/// ## Error trait
///
/// Implements [`std::error::Error`] with no `source()` chain, since page
/// validation failures are leaf errors -- they do not wrap an underlying cause.
///
/// Fields are private; use [`violation()`](Self::violation) and
/// [`details()`](Self::details) for read access. The `(violation, details)`
/// pair is always consistent when produced by [`validate_page_range`] or
/// [`validate_page`].
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PageValidationError {
    /// The violated rule, suitable for `match`-based dispatch and metrics labels.
    violation: PageValidationViolation,
    /// Redacted diagnostic context for the violation.
    details: PageValidationDetails,
}
```

The two-field design serves different consumers. Metrics systems and circuit
breakers match on `violation()` -- a `Copy` discriminant that can be used as
a counter label without allocation. Diagnostic tooling and log aggregation
use `details()` to extract the redacted digests and positional indices for
human-readable error messages.

A compile-time assertion enforces that the error stays within a 256-byte size
budget:

Here is the definition from `page_validator.rs`:

```rust
/// Compile-time guard: `PageValidationError` must fit within 256 bytes to
/// justify the `#[allow(clippy::result_large_err)]` on the validation
/// functions. If this fires after adding fields, consider boxing the
/// largest `PageValidationDetails` variant.
const _: () = assert!(
    std::mem::size_of::<PageValidationError>() <= 256,
    "PageValidationError grew beyond expected size budget"
);
```

This assertion exists because the validation functions return
`Result<(), PageValidationError>` without boxing. A 256-byte error on the
stack is acceptable for a cold path (validation failures), but if the error
grew beyond that budget, boxing the largest variant would be the correct
response.

---

## validate_page_range -- The Core Validation Function

The heart of the module is a single generic function that checks ten
invariants in a fixed order, returning on the first failure.

Here is the definition from `page_validator.rs`:

```rust
/// Validate one connector page against ordering, membership, and cursor rules.
///
/// `start`/`end` define the key range, with empty slices treated as
/// unbounded boundaries. Validation runs in a fixed order and returns on the
/// first failure:
///
/// 1. range sanity (`start <= end` for bounded ranges),
/// 2. input cursor in range,
/// 3. next cursor in range,
/// 4. item membership and local ordering (single pass; membership checked
///    before ordering for each item),
/// 5. empty-page cursor stability,
/// 6. non-empty page requires `next_last_key`,
/// 7. first item must be strictly after `input_last_key`,
/// 8. next cursor must not regress behind input cursor,
/// 9. next cursor must be at least the last item key.
///
/// ## Range semantics
///
/// - Item keys are validated in `[start, end)`.
/// - Cursor keys are validated in `[start, end]`.
///
/// This asymmetry is deliberate: item membership follows shard half-open
/// semantics, while cursor checks stay slightly more permissive at the upper
/// boundary.
///
/// ## Comparison semantics
///
/// All ordering and membership comparisons operate on byte-level
/// representations (`key.as_ref()` slices), not on `K`'s `Ord` impl
/// directly. The `K: Ord` bound serves as a contract constraint -- it
/// prevents callers from accidentally instantiating the validator with an
/// unordered type -- but the actual comparison is always byte-lexicographic.
/// For the concrete types in this crate (`ItemKey`, `[u8]`, `Vec<u8>`),
/// byte-lexicographic order and `Ord` agree.
///
/// ## Performance
///
/// Validation is allocation-free on the success path. Toxic hashing is
/// performed only while constructing error payloads.
///
/// # Caller responsibilities
///
/// This function does not enforce a maximum item count. Callers must
/// bound page size (e.g., via [`Budgets`](super::Budgets)) before invoking
/// validation.
///
/// # Errors
///
/// Returns the first [`PageValidationError`] in the validation order listed
/// above.
#[allow(clippy::result_large_err)]
pub fn validate_page_range<K, I>(
    start: &K,
    end: &K,
    input_last_key: Option<&K>,
    items: &[I],
    next_last_key: Option<&K>,
) -> Result<(), PageValidationError>
where
    K: Ord + AsRef<[u8]> + ?Sized,
    I: PageItem<K>,
{
```

The function signature captures the design: generic over key type `K` and
item type `I`, bounded by `Ord + AsRef<[u8]>` for `K` and `PageItem<K>`
for `I`. The `?Sized` bound on `K` allows `[u8]` slices as keys directly.

### Check (a): Spec Range Sanity

```rust
    let start_bytes = start.as_ref();
    let end_bytes = end.as_ref();
    let start_unbounded = start_bytes.is_empty();
    let end_unbounded = end_bytes.is_empty();

    // (a) Spec sanity: start <= end for bounded ranges.
    if !start_unbounded && !end_unbounded && start_bytes > end_bytes {
        return Err(PageValidationError {
            violation: PageValidationViolation::SpecRangeInvalid,
            details: PageValidationDetails::Spec {
                start: ToxicDigest::of(start),
                end: ToxicDigest::of(end),
            },
        });
    }
```

Empty byte slices represent unbounded boundaries. If both bounds are present
and `start > end`, the range is inverted. The check is skipped when either
bound is empty (unbounded).

### Checks (b) and (c): Cursor Membership

```rust
    let cursor_in_range = |key: &K| {
        let key_bytes = key.as_ref();
        (start_unbounded || key_bytes >= start_bytes) && (end_unbounded || key_bytes <= end_bytes)
    };

    if let Some(input) = input_last_key
        && !cursor_in_range(input)
    {
        return Err(PageValidationError {
            violation: PageValidationViolation::InputCursorOutOfRange,
            details: PageValidationDetails::CursorOutOfRange {
                which: CursorWhich::Input,
                key: ToxicDigest::of(input),
                start: ToxicDigest::of(start),
                end: ToxicDigest::of(end),
            },
        });
    }

    if let Some(next) = next_last_key
        && !cursor_in_range(next)
    {
        return Err(PageValidationError {
            violation: PageValidationViolation::NextCursorOutOfRange,
            details: PageValidationDetails::CursorOutOfRange {
                which: CursorWhich::Next,
                key: ToxicDigest::of(next),
                start: ToxicDigest::of(start),
                end: ToxicDigest::of(end),
            },
        });
    }
```

Cursor keys are checked against a **closed** range `[start, end]`. The
`<=` comparison for the upper bound is deliberate: connectors may park a
continuation cursor at the exact upper boundary when a shard is exhausted.

### Check (c+d): Item Membership and Ordering (Single Pass)

```rust
    let mut prev: Option<&K> = None;
    for (index, item) in items.iter().enumerate() {
        let key = item.item_key();
        let key_bytes = key.as_ref();

        // Membership: item key must be in [start, end).
        let in_range = (start_unbounded || key_bytes >= start_bytes)
            && (end_unbounded || key_bytes < end_bytes);
        if !in_range {
            return Err(PageValidationError {
                violation: PageValidationViolation::ItemKeyOutOfRange,
                details: PageValidationDetails::ItemOutOfRange {
                    index,
                    key: ToxicDigest::of(key),
                    start: ToxicDigest::of(start),
                    end: ToxicDigest::of(end),
                },
            });
        }

        // Ordering: item keys must be non-decreasing.
        if let Some(prev_key) = prev
            && prev_key.as_ref() > key_bytes
        {
            return Err(PageValidationError {
                violation: PageValidationViolation::ItemsNotOrdered,
                details: PageValidationDetails::ItemsNotOrdered {
                    index,
                    prev: ToxicDigest::of(prev_key),
                    next: ToxicDigest::of(key),
                },
            });
        }
        prev = Some(key);
    }
```

Item keys are checked against a **half-open** range `[start, end)`. The
`<` comparison for the upper bound means an item key equal to `end` is
out of range. This follows shard half-open semantics: items at the boundary
belong to the next shard.

Membership is checked before ordering for each item. If an item is both out
of range and out of order, the out-of-range error takes priority. This
ordering produces more actionable diagnostics: an out-of-range item is a
more fundamental problem than a local ordering violation.

The membership and ordering checks run in a single pass over the items
slice. No second pass is needed.

### Check (e): Empty-Page Cursor Stability

```rust
    let input_bytes = input_last_key.map(|k| k.as_ref());
    let next_bytes = next_last_key.map(|k| k.as_ref());
    if items.is_empty() && input_bytes != next_bytes {
        return Err(PageValidationError {
            violation: PageValidationViolation::EmptyPageCursorAdvanced,
            details: PageValidationDetails::EmptyPageCursorAdvanced {
                input: input_last_key.map(ToxicDigest::of),
                next: next_last_key.map(ToxicDigest::of),
            },
        });
    }
```

An empty page must not advance the cursor. If no items were returned, the
cursor state must remain unchanged -- otherwise the scan would skip over
the gap between the old cursor position and the new one without examining
it. The comparison uses byte-level representations, not `K::PartialEq`,
to avoid false positives when `K` distinguishes values that are byte-equal
(the test suite verifies this with a `TaggedKey` type whose `PartialEq`
includes a metadata tag that `AsRef<[u8]>` ignores).

### Check (f): Non-Empty Page Requires next_last_key

```rust
    if !items.is_empty() && next_last_key.is_none() {
        return Err(PageValidationError {
            violation: PageValidationViolation::NextCursorMissing,
            details: PageValidationDetails::NextCursorMissing,
        });
    }
```

A page that returns items must provide a continuation cursor with a
`last_key`. Without it, the scan loop has no resumption point for the next
page request.

### Check (g): First Item Strictly After Input Cursor

```rust
    if let (Some(input), Some(first_item)) = (input_last_key, items.first()) {
        let first_key = first_item.item_key();
        if first_key.as_ref() <= input.as_ref() {
            return Err(PageValidationError {
                violation: PageValidationViolation::ItemsNotAfterCursor,
                details: PageValidationDetails::ItemsNotAfterCursor {
                    cursor: ToxicDigest::of(input),
                    first_item: ToxicDigest::of(first_key),
                },
            });
        }
    }
```

The first item on a page must be strictly greater than the input cursor key.
The cursor represents "the last key successfully processed." The next page
must start after that point. Equality is a violation: re-emitting the cursor
key would mean processing it twice.

### Check (h): Cursor Monotonicity

```rust
    if let (Some(input), Some(next)) = (input_last_key, next_last_key)
        && next.as_ref() < input.as_ref()
    {
        return Err(PageValidationError {
            violation: PageValidationViolation::CursorRegressed,
            details: PageValidationDetails::CursorRegressed {
                input: ToxicDigest::of(input),
                next: ToxicDigest::of(next),
            },
        });
    }
```

The continuation cursor must not move backward. A cursor that regresses
behind the input cursor would cause the scan to revisit items that were
already processed.

### Check (i): Next Cursor At Least Last Item

```rust
    if let (Some(next), Some(last_item)) = (next_last_key, items.last()) {
        let last_key = last_item.item_key();
        if next.as_ref() < last_key.as_ref() {
            return Err(PageValidationError {
                violation: PageValidationViolation::NextCursorBehindLastItem,
                details: PageValidationDetails::NextCursorBehindLastItem {
                    next_cursor: ToxicDigest::of(next),
                    last_item: ToxicDigest::of(last_key),
                },
            });
        }
    }

    Ok(())
}
```

The continuation cursor must be at or past the last item in the page.
A cursor behind the last item would cause the next page to re-emit items
from the current page.

---

## The Allocation-Free Success Path

On the happy path -- when all ten checks pass -- the function performs
zero heap allocations. No `ToxicDigest` values are computed, no error
structs are constructed, no strings are formatted. The function walks the
items slice once, comparing byte references, and returns `Ok(())`.

`ToxicDigest` computation (BLAKE3 hashing) happens only inside error
construction blocks. Each `return Err(...)` block calls `ToxicDigest::of()`
on the relevant keys, but these blocks are reached only on the cold error
path. The design ensures that validation cost on valid pages is proportional
to the number of items and nothing else.

---

## The Half-Open vs Closed Range Asymmetry

The validator intentionally uses two different range conventions:

- **Item keys** are checked against `[start, end)` -- half-open. An item
  key equal to `end` is out of range. This follows shard boundary semantics:
  items at the upper boundary belong to the next shard.

- **Cursor keys** are checked against `[start, end]` -- closed. A cursor
  key equal to `end` is in range. This is more permissive at the upper
  boundary, allowing connectors to park a continuation cursor at the exact
  shard boundary when enumeration is complete.

The module documentation notes this asymmetry explicitly and flags a
further subtlety: `ShardSpec` documents its cursor invariant as half-open
`[start, end)`. The validator's closed-range cursor check is deliberately
more permissive. Callers enforcing `ShardSpec`'s stricter invariant must
do so separately. This layering keeps the page validator focused on
connector contract violations without duplicating `ShardSpec` policy.

---

## validate_page -- The Production Adapter

The generic `validate_page_range` is wrapped by a concrete adapter for
production types:

Here is the definition from `page_validator.rs`:

```rust
/// Validate a concrete connector scan page against a [`ShardSpec`] and cursors.
///
/// This is a thin adapter over [`validate_page_range`] for the common connector
/// runtime types (`ShardSpec`, [`Cursor`], and [`ScanItem`]). It does not add
/// additional policy checks; callers get the same invariant set, range
/// conventions, and first-failure behavior as [`validate_page_range`].
///
/// # Caller responsibilities
///
/// This function does not enforce a maximum item count. Callers must
/// bound page size (e.g., via [`Budgets`](super::Budgets)) before invoking
/// validation.
///
/// # Errors
///
/// Forwards any [`PageValidationError`] produced by [`validate_page_range`].
#[allow(clippy::result_large_err)]
pub fn validate_page(
    spec: &ShardSpec,
    input_cursor: &Cursor,
    items: &[ScanItem],
    next_cursor: &Cursor,
) -> Result<(), PageValidationError> {
    validate_page_range(
        spec.key_range_start(),
        spec.key_range_end(),
        input_cursor.last_key().map(|key| key.as_ref()),
        items,
        next_cursor.last_key().map(|key| key.as_ref()),
    )
}
```

The adapter extracts byte slices from `ShardSpec` and `Cursor`, projecting
cursor keys through `ItemKey::as_ref()` to `&[u8]`. This instantiates
`validate_page_range::<[u8], ScanItem>`, which requires the
`PageItem<[u8]> for ScanItem` impl shown earlier.

The adapter adds no additional policy. Page size enforcement (maximum item
count) is the caller's responsibility, applied via `Budgets` before
invoking validation.

---

## Summary

The page validator module enforces ten ordered invariants against every
page returned by a connector's `enumerate_page` method. `ToxicDigest`
provides a `Copy`-able, 40-byte hash representation that redacts connector
keys and cursors from all diagnostic output. `PageValidationViolation` gives
metrics systems and circuit breakers a thin, `Copy`-able discriminant for
each rule. `PageValidationDetails` pairs with violations to provide redacted
diagnostic context. The core `validate_page_range` function checks spec
sanity, cursor membership, item membership with ordering, cursor stability,
cursor progression, and cursor consistency -- all in a single pass over
the items slice, allocation-free on the success path. The half-open item
range `[start, end)` and closed cursor range `[start, end]` asymmetry is a
deliberate design choice that matches shard boundary semantics while
remaining permissive for cursor parking.

Chapter 5 examines the circuit breaker design that handles validation
failures at the source level.
