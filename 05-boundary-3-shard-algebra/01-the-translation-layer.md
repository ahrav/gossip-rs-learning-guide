# "Two Connectors, One Keyspace" -- The Translation Layer

*A filesystem connector enumerates paths under `/data/repo/`. It produces keys
like `"repo/alpha"`, `"repo/manifests/v2"`, and `"repo/z"`. A manifest
connector indexes the same repository by `(manifest_id, row)` pairs -- numeric
tuples encoded as big-endian `u64`s. The coordinator receives both streams.
It stores shard boundaries as raw `&[u8]` slices and determines range
membership by lexicographic byte comparison. The filesystem key `"repo/z"`
encodes to `[0x72, 0x65, 0x70, 0x6F, 0x2F, 0x7A]`. The manifest key
`(0, 42)` encodes to `[0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x2A]`. Because `0x72 > 0x00`,
the path key sorts after the manifest key. A shard whose range spans both
domains now contains items from two incompatible key schemas. Range-membership
checks succeed for both -- the bytes compare -- but the resulting shard is
semantically meaningless. Items fall into shards that no connector knows how
to interpret. Coverage breaks silently. No error is raised.*

---

## Why a Translation Layer Exists

The failure above stems from a missing contract. The coordinator operates on
opaque byte slices. It does not know what a "path" is, or what a "manifest row"
is. It compares bytes, computes midpoints, and splits ranges. Connectors operate
on typed domain objects -- filesystem paths, database row identifiers, API
pagination tokens. Between these two worlds sits the **translation layer**: a set
of traits, key types, and bridge helpers that convert typed keys into
byte-comparable encodings while preserving logical ordering.

The translation layer is the subject of the `gossip-frontier` crate. It
enforces a single invariant that makes everything downstream work: if key `a`
is logically less than key `b` in the connector's domain, then `encode(a)` must
be lexicographically less than `encode(b)` in the byte domain. Violate this, and
shard boundaries become meaningless. Uphold it, and the coordinator can compare,
split, and bisect keys from any connector without understanding their semantics.

---

## The gossip-frontier Crate

The `gossip-frontier` crate is the bridge between connectors and coordination.
It sits at a precise point in the dependency graph: it may depend on
`gossip-contracts` (identity types, shard data model), but it must never
reference `gossip-coordination`, any connector implementation, or the
persistence layer. This boundary is intentional -- it keeps the encoding logic
testable in isolation and prevents circular dependencies.

Here is the crate-level documentation from `lib.rs`:

```rust
//! Shard algebra: key encoding, range arithmetic, hint metadata, and builder.
//!
//! This crate is the bridge between connectors (which think in typed keys
//! like file paths or manifest rows) and the coordination layer (which
//! operates on opaque `&[u8]` range boundaries).
//!
//! # Modules
//!
//! - [`key_encoding`] — byte-order-preserving key encoders
//!   ([`KeyEncoding`], [`PathKey`], [`ManifestRowKey`]), range arithmetic
//!   ([`prefix_successor`], [`key_successor`], [`byte_midpoint`]), and
//!   `ShardSpec` constructors from typed key inputs.
//!
//! - [`hint`] — versionless shard-hint wire framing ([`ShardHint`],
//!   [`ShardMetadata`]), allocation-free typed shard-spec builders
//!   ([`range_shard_ref`], [`prefix_shard_ref`], [`manifest_shard_ref`]),
//!   decode helpers, and split-propagation logic
//!   ([`propagate_hint_on_split`]).
//!
//! - [`builder`] — startup-preallocated shard builder with
//!   borrowed-first add paths and bounded-capacity manifests.
//!
//! # Zero-allocation design
//!
//! All submodules follow the same caller-owned-buffer pattern: callers
//! provide a reusable scratch buffer ([`KeyBuf`], [`MetadataBuf`], or
//! [`ShardSpecScratch`]) and receive borrowed output slices. This
//! eliminates per-operation heap allocation on the coordination hot path
//! (acquire, checkpoint, split).
//!
//! # Dependency direction
//!
//! May depend on `gossip-contracts` (identity + shard data model).
//! Must not reference `gossip-coordination`, `connector`, or `persistence`.
```

Note `#![forbid(unsafe_code)]` at the top of the crate root. The entire
translation layer -- key encoding, range arithmetic, shard construction --
operates without a single `unsafe` block. This is feasible because the hot
path uses stack-resident fixed-capacity buffers rather than pointer
arithmetic.

Three modules compose the crate. This chapter covers the first:
`key_encoding`, which provides the ordering-preserving trait, two concrete
key types, and the bridge helpers that construct `ShardSpec` values from
typed inputs. Chapter 2 covers the range arithmetic functions
(`prefix_successor`, `key_successor`, `byte_midpoint`) that also live in
`key_encoding`. The `hint` and `builder` modules are covered in later
chapters.

---

## The KeyEncoding Trait

The central abstraction is a single-method trait. Every connector key type
that wants to participate in shard algebra must implement it.

Here is the definition from `key_encoding.rs`:

```rust
/// Trait for key types that encode into lexicographically ordered bytes.
///
/// # Ordering contract
///
/// ```text
/// a < b  (logical ordering)  =>  encode(a) < encode(b)  (byte lex ordering)
/// ```
///
/// This invariant is load-bearing: cursor monotonicity, shard range-membership
/// checks, and deterministic split planning all rely on it. Violating it
/// silently corrupts range boundaries.
///
/// # Canonicality
///
/// Implementations must be canonical and deterministic:
/// - Logically equal keys must encode to identical byte strings.
/// - A single key must not encode differently across calls.
///
/// These properties ensure that key comparisons are stable regardless of when
/// or where they happen (local planner, coordinator, or across restarts).
///
/// # Usage
///
/// Connectors implement this trait for their domain-specific key types (e.g.,
/// file paths, manifest row IDs). The coordinator never calls `encode_into`
/// directly -- it operates on raw `&[u8]` boundaries produced by the shard
/// builder or split planner.
///
/// # Buffer contract
///
/// Implementations must write a complete canonical encoding into `buf` whose
/// length does not exceed [`KeyBuf::CAPACITY`]. Calling
/// [`KeyBuf::copy_from_slice`] satisfies this contract.
pub trait KeyEncoding: Sized {
    /// Encode this key into bytes that preserve logical ordering under
    /// lexicographic byte comparison.
    fn encode_into(&self, buf: &mut KeyBuf);
}
```

Three properties are required of every implementation:

1. **Order preservation.** If `a < b` in the logical domain, then `encode(a)`
   must sort before `encode(b)` under lexicographic byte comparison. This is
   the ordering contract stated in the doc comment. It is not checked at
   compile time -- Rust's type system cannot express "this encoding preserves
   an ordering" -- so violations are caught only by property tests (the test
   suite uses `proptest` to verify this for both `PathKey` and
   `ManifestRowKey`).

2. **Canonicality.** Equal keys must produce identical byte strings. A key
   must not encode differently on Tuesday than on Thursday. This matters
   because the coordinator persists shard boundaries across restarts. If the
   same logical key encodes to different bytes before and after a restart,
   the shard's range shifts silently.

3. **Buffer contract.** The encoding must fit within `KeyBuf::CAPACITY` bytes.
   Implementations satisfy this by calling `KeyBuf::copy_from_slice`, which
   panics if the input exceeds capacity. This makes buffer overflows a loud
   failure rather than a silent corruption.

The coordinator never calls `encode_into` directly. It receives pre-encoded
`&[u8]` boundaries from the shard builder or split planner. The trait exists
solely so that connectors can produce those boundaries from typed keys in a
way that preserves ordering.

---

## KeyBuf -- The Stack-Resident Scratch Buffer

Every encoding operation writes into a `KeyBuf`. This is a fixed-capacity
stack buffer that eliminates heap allocation on the encoding path.

Here is the definition from `key_encoding.rs`:

```rust
/// Reusable stack buffer for shard-key arithmetic.
///
/// The buffer owns fixed-capacity storage and tracks the active key prefix via
/// `len`. Bytes after `len` are scratch space and not part of the logical key.
///
/// Capacity is `MAX_KEY_SIZE + 1` to allow `byte_midpoint`'s internal
/// carry-expanded arithmetic sum.
#[derive(Clone)]
pub struct KeyBuf {
    buf: [u8; Self::CAPACITY],
    len: usize,
}

impl KeyBuf {
    /// Maximum number of bytes this buffer can hold.
    pub const CAPACITY: usize = MAX_KEY_SIZE + 1;

    /// Create an empty key buffer.
    pub fn new() -> Self {
        Self {
            buf: [0u8; Self::CAPACITY],
            len: 0,
        }
    }

    /// View the active key bytes.
    ///
    /// The returned slice borrows this buffer and is invalidated by the next
    /// mutating write to the same `KeyBuf`.
    pub fn as_bytes(&self) -> &[u8] {
        &self.buf[..self.len]
    }

    /// Active byte length.
    pub fn len(&self) -> usize {
        self.len
    }

    /// Whether no bytes are active.
    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    /// Copy `src` into this buffer and set the active length.
    ///
    /// This overwrites the active prefix `[0..src.len())`. Bytes beyond the new
    /// length are left unchanged and remain scratch space.
    ///
    /// # Panics
    ///
    /// Panics if `src.len() > Self::CAPACITY`.
    pub fn copy_from_slice(&mut self, src: &[u8]) {
        assert!(
            src.len() <= Self::CAPACITY,
            "key bytes exceed KeyBuf capacity: {} > {}",
            src.len(),
            Self::CAPACITY
        );
        self.buf[..src.len()].copy_from_slice(src);
        self.len = src.len();
    }
}

impl Default for KeyBuf {
    fn default() -> Self {
        Self::new()
    }
}

impl fmt::Debug for KeyBuf {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("KeyBuf")
            .field("len", &self.len)
            .field("bytes", &self.as_bytes())
            .finish()
    }
}
```

A field-by-field breakdown:

- **`buf: [u8; Self::CAPACITY]`** -- A fixed-size byte array on the stack.
  `CAPACITY` is `MAX_KEY_SIZE + 1`, which is 4097 bytes. The extra byte
  beyond `MAX_KEY_SIZE` (4096) exists because `byte_midpoint` performs
  big-endian addition of two keys, which can produce a carry that extends the
  sum by one byte. Without the extra byte, the addition would overflow the
  buffer.

- **`len: usize`** -- Tracks how many bytes of `buf` are "active." Bytes
  from index `len` onward are scratch space. The `as_bytes()` method returns
  only `&buf[..len]`, so downstream consumers never see stale data from a
  previous encoding.

The `copy_from_slice` method is the only write path. It panics if the source
exceeds `CAPACITY`, which means every `KeyEncoding` implementation that uses
it gets bounds checking for free. There is no way to silently write past the
end of the buffer.

The returned slice from `as_bytes()` borrows the `KeyBuf`. This is the
caller-owned-buffer pattern described in the crate documentation: the caller
creates a `KeyBuf` once (typically on the stack), passes it to encoding and
arithmetic functions, and reads the result via the borrowed slice. No heap
allocation occurs per operation.

---

## PathKey -- Identity Encoding for Filesystem Paths

The simplest concrete key type encodes UTF-8 path strings as their raw bytes
with no transformation.

Here is the definition from `key_encoding.rs`:

```rust
/// UTF-8 path key encoded as identity bytes.
///
/// # Invariants
///
/// Encoding is exactly `path.as_bytes()`: no separator rewriting, Unicode
/// normalization, or case folding.
///
/// # Trade-off
///
/// This keeps encoding deterministic and allocation-free, but logically
/// equivalent filesystem paths with different textual forms compare as
/// different keys. Callers that need canonical path semantics must normalize
/// before constructing [`PathKey`].
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct PathKey<'a> {
    path: &'a str,
}

impl<'a> PathKey<'a> {
    /// Create a path key from UTF-8 path text.
    ///
    /// # Panics
    ///
    /// Panics if `path` is empty or if `path.len()` exceeds [`MAX_KEY_SIZE`].
    pub const fn new(path: &'a str) -> Self {
        assert!(!path.is_empty(), "PathKey path must not be empty");
        assert!(
            path.len() <= MAX_KEY_SIZE,
            "PathKey path exceeds MAX_KEY_SIZE"
        );
        Self { path }
    }

    /// Return a reference to the underlying UTF-8 path.
    pub const fn as_str(self) -> &'a str {
        self.path
    }
}

impl KeyEncoding for PathKey<'_> {
    fn encode_into(&self, buf: &mut KeyBuf) {
        buf.copy_from_slice(self.path.as_bytes());
    }
}
```

The struct contains a single field:

- **`path: &'a str`** -- A borrowed UTF-8 string slice. The lifetime `'a`
  ties the key to the source string, avoiding a copy. The constructor enforces
  two invariants at creation time: the path must not be empty (an empty key
  has no meaningful position in the keyspace), and it must not exceed
  `MAX_KEY_SIZE` (4096 bytes).

The `KeyEncoding` implementation is a single line: copy the path's raw UTF-8
bytes into the buffer. This is the **identity encoding** -- no separator
rewriting (`/` stays as `0x2F`), no Unicode normalization (NFC and NFD forms
of the same character encode differently), no case folding (`"Repo"` and
`"repo"` are distinct keys).

This trade-off is explicit in the doc comment. Identity encoding is
deterministic and allocation-free, but it pushes normalization responsibility
to the caller. A filesystem connector that wants `"./foo/../bar"` and
`"bar"` to be the same key must canonicalize before constructing the
`PathKey`. The encoding layer does not second-guess the input.

For ASCII-only paths (the common case in repository scanning), identity
encoding has a useful property: the lexicographic order of the byte encoding
matches the lexicographic order of the string itself. The path `"alpha"`
encodes to bytes that sort before `"beta"`, just as `"alpha" < "beta"` in
string comparison.

---

## ManifestRowKey -- Fixed-Width Numeric Encoding

The second concrete key type encodes a `(manifest_id, row)` pair as two
big-endian `u64` values, producing exactly 16 bytes.

Here is the definition from `key_encoding.rs`:

```rust
/// Fixed-width manifest row key encoded as `(manifest_id, row)` in big-endian
/// `u64`s.
///
/// Lexicographic byte ordering matches tuple ordering: compare by
/// `manifest_id` first, then by `row`. The fixed 16-byte layout avoids
/// delimiters/varints and keeps decode cost constant.
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct ManifestRowKey {
    manifest_id: u64,
    row: u64,
}

impl ManifestRowKey {
    /// Fixed encoded width in bytes: 8-byte manifest ID + 8-byte row.
    pub const ENCODED_LEN: usize = 16;

    /// Create a manifest row key.
    pub const fn new(manifest_id: u64, row: u64) -> Self {
        Self { manifest_id, row }
    }

    /// Manifest identifier component.
    pub const fn manifest_id(self) -> u64 {
        self.manifest_id
    }

    /// Row component.
    pub const fn row(self) -> u64 {
        self.row
    }
}

impl KeyEncoding for ManifestRowKey {
    fn encode_into(&self, buf: &mut KeyBuf) {
        let mut tmp = [0u8; Self::ENCODED_LEN];
        tmp[..8].copy_from_slice(&self.manifest_id.to_be_bytes());
        tmp[8..].copy_from_slice(&self.row.to_be_bytes());
        buf.copy_from_slice(&tmp);
    }
}
```

A field-by-field breakdown:

- **`manifest_id: u64`** -- Identifies which manifest (a logical grouping
  of rows) this key belongs to.

- **`row: u64`** -- The row number within that manifest.

The constant `ENCODED_LEN = 16` documents the wire width: 8 bytes for
`manifest_id` + 8 bytes for `row`. This is a fixed-width encoding -- every
`ManifestRowKey` produces exactly 16 bytes regardless of the numeric values.

The `KeyEncoding` implementation writes both fields as big-endian bytes into a
local 16-byte array, then copies that array into the `KeyBuf`. Big-endian
encoding is the critical choice: for unsigned integers, big-endian byte
representation preserves numeric ordering under lexicographic comparison.
The value `42` encodes to `[0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x2A]`,
and `43` encodes to `[0x00, ..., 0x2B]`. Comparing these byte sequences
lexicographically yields the same result as comparing the integers.

Because `manifest_id` occupies the first 8 bytes, lexicographic comparison
checks `manifest_id` first and breaks ties by `row`. This matches Rust's
derived `Ord` for tuples: `(a1, a2) < (b1, b2)` iff `a1 < b1`, or
`a1 == b1` and `a2 < b2`.

Fixed-width encoding avoids two alternatives that introduce complexity:

- **Delimiters** (e.g., separating fields with `0x00`) require escaping if
  the field values can contain the delimiter byte. For `u64` values that span
  the full range, every byte value from `0x00` to `0xFF` is possible.

- **Variable-length integers** (varints) compress small values but break
  lexicographic ordering -- a 1-byte varint for `42` sorts before a 2-byte
  varint for `200`, but the relationship is not guaranteed by byte comparison
  alone without length-prefixing.

The corresponding decode function reconstructs the key from its byte form.

Here is the definition from `key_encoding.rs`:

```rust
/// Decode a fixed-width [`ManifestRowKey`] byte encoding.
///
/// Returns a [`ManifestRowKey`] when `key` is exactly
/// [`ManifestRowKey::ENCODED_LEN`] bytes containing two big-endian `u64`
/// fields. Returns `None` for any other length.
///
/// This function validates framing only; semantic checks (for example,
/// monotonic row ranges) are performed by higher-level helpers.
pub fn decode_manifest_row_key(key: &[u8]) -> Option<ManifestRowKey> {
    if key.len() != ManifestRowKey::ENCODED_LEN {
        return None;
    }
    let manifest_id = u64::from_be_bytes(key[..8].try_into().unwrap());
    let row = u64::from_be_bytes(key[8..].try_into().unwrap());
    Some(ManifestRowKey::new(manifest_id, row))
}
```

The function is strict about length: if the input is not exactly 16 bytes, it
returns `None`. The `unwrap()` calls on `try_into()` are safe because the
length check guarantees that both 8-byte slices exist. Semantic validation
(e.g., ensuring that a start row is less than an end row) is deferred to
higher-level helpers.

---

## Comparing the Two Key Schemas

The following table summarizes how the two key types differ in encoding
strategy, and why mixing them in a single shard's range is invalid:

```text
+------------------+---------------------------+-------------------------------+
| Property         | PathKey                   | ManifestRowKey                |
+------------------+---------------------------+-------------------------------+
| Domain           | UTF-8 filesystem paths    | (manifest_id, row) tuples     |
| Encoding         | Identity (raw bytes)      | Big-endian u64 pair           |
| Width            | Variable (1..4096 bytes)  | Fixed (16 bytes)              |
| First byte range | 0x01..0x7F (typical ASCII)| 0x00..0xFF (full range)       |
| Ordering         | String lexicographic      | Numeric tuple ordering        |
| Normalization    | None (caller's job)       | None needed (canonical form)  |
| Decode cost      | Zero (bytes are the key)  | Constant (two u64 reads)      |
+------------------+---------------------------+-------------------------------+
```

A shard range `[start, end)` is meaningful only when both boundaries use the
same encoding schema. The coordinator cannot tell that
`[0x72, 0x65, 0x70, 0x6F]` is a path and
`[0x00, ..., 0x2A]` is a manifest row -- it sees only bytes. The translation
layer prevents cross-schema ranges by construction: each bridge helper
(covered next) encodes both start and end keys using the same `KeyEncoding`
implementation.

---

## The Encoding Flow

The full path from a typed connector key to a coordinator-visible `ShardSpec`
follows this pipeline:

```text
 Connector                Translation Layer               Coordinator
 --------                ==================               -----------

 PathKey("repo/z")  ---->  encode_into(&mut KeyBuf)  ---->  &[u8] boundary
                           |                                    |
                           v                                    v
                     KeyBuf { buf: [0x72,0x65,...], len: 6 }   ShardSpec {
                                                                 key_range_start: ...,
 ManifestRowKey      ---->  encode_into(&mut KeyBuf)  ---->      key_range_end: ...,
  (42, 100)                |                                     metadata: ...
                           v                                   }
                     KeyBuf { buf: [0x00,...,0x2A,
                                    0x00,...,0x64], len: 16 }
```

The connector creates a typed key. The translation layer encodes it into a
`KeyBuf`. The bridge helpers (below) take the encoded bytes and construct a
`ShardSpec` that the coordinator can store and compare.

---

## ShardSpec Bridge Helpers

Three functions translate typed key inputs into `ShardSpec` values. Each comes
in a `_ref` variant (returns a borrowed `ShardSpecRef`) and an `_into` variant
(allocates into a `ShardArena`). The `_ref` variant is the foundation; the
`_into` variant wraps it with arena allocation.

### shard_spec_from_keys_ref

The most general bridge helper accepts any two `KeyEncoding` implementations
as start and end boundaries.

Here is the definition from `key_encoding.rs`:

```rust
/// Construct a borrowed [`ShardSpecRef`] from typed start/end boundaries.
///
/// `start` and `end` are encoded into caller-provided scratch buffers, and
/// validation is delegated to [`ShardSpec::validate_ref`].
/// Returned range bytes borrow from `start_buf`/`end_buf`; reusing either
/// buffer invalidates those views.
///
/// # Errors
///
/// Returns [`ShardSpecInputError`] when the encoded range fails validation:
/// inverted bounds, keys exceeding [`MAX_KEY_SIZE`], or oversized metadata.
#[must_use = "returns a Result that must be checked for validation errors"]
pub fn shard_spec_from_keys_ref<'a, Start: KeyEncoding, End: KeyEncoding>(
    start: &Start,
    end: &End,
    metadata: &'a [u8],
    start_buf: &'a mut KeyBuf,
    end_buf: &'a mut KeyBuf,
) -> Result<ShardSpecRef<'a>, ShardSpecInputError> {
    start.encode_into(start_buf);
    end.encode_into(end_buf);
    let spec = ShardSpecRef::new(start_buf.as_bytes(), end_buf.as_bytes(), metadata);
    ShardSpec::validate_ref(spec)?;
    Ok(spec)
}
```

The function encodes both keys into caller-provided buffers, constructs a
borrowed `ShardSpecRef` from the encoded bytes, and validates the result.
Validation catches inverted ranges (`start >= end`), keys exceeding
`MAX_KEY_SIZE`, and oversized metadata. The `#[must_use]` attribute ensures
callers cannot ignore the `Result`.

The lifetime `'a` ties the returned `ShardSpecRef` to both scratch buffers
and the metadata slice. Reusing either `KeyBuf` for another encoding
invalidates the view. This is the zero-allocation pattern in action: no heap
allocation occurs, but the caller must manage buffer lifetimes.

### shard_spec_from_prefix_ref

For prefix-based shards, the end boundary is computed automatically as the
prefix's lexicographic successor.

Here is the definition from `key_encoding.rs`:

```rust
/// Construct a prefix-bounded [`ShardSpecRef`] as `[prefix, prefix_successor)`.
///
/// The returned end bound borrows from `successor_buf`.
///
/// # Errors
///
/// Returns [`PrefixShardError`] when the prefix is empty, exceeds
/// [`MAX_KEY_SIZE`], has no lexicographic successor (all `0xFF`), or when
/// the derived spec fails downstream validation.
#[must_use = "returns a Result that must be checked for validation errors"]
pub fn shard_spec_from_prefix_ref<'a>(
    prefix: &'a [u8],
    metadata: &'a [u8],
    successor_buf: &'a mut KeyBuf,
) -> Result<ShardSpecRef<'a>, PrefixShardError> {
    if prefix.is_empty() {
        return Err(PrefixShardError::EmptyPrefix);
    }
    if prefix.len() > MAX_KEY_SIZE {
        return Err(PrefixShardError::PrefixTooLarge {
            size: prefix.len(),
            max: MAX_KEY_SIZE,
        });
    }

    let end = prefix_successor(prefix, successor_buf).ok_or(PrefixShardError::NoSuccessor)?;
    let spec = ShardSpecRef::new(prefix, end, metadata);
    ShardSpec::validate_ref(spec).map_err(PrefixShardError::from)?;
    Ok(spec)
}
```

This function performs three checks before constructing the spec: the prefix
must not be empty, must not exceed `MAX_KEY_SIZE`, and must have a
lexicographic successor (all-`0xFF` prefixes do not). The `prefix_successor`
function -- covered in detail in Chapter 2 -- computes the exclusive upper
bound. The resulting shard covers exactly the set of keys that begin with the
given prefix.

### shard_spec_from_manifest_range_ref

For manifest-based shards, both boundaries come from the same manifest ID
with different row numbers.

Here is the definition from `key_encoding.rs`:

```rust
/// Construct a same-manifest [`ShardSpecRef`] from a half-open row range.
///
/// Encodes `(manifest_id, start_row)` and `(manifest_id, end_row)` as
/// [`ManifestRowKey`] boundaries. Returned start/end bounds borrow from
/// `start_buf` and `end_buf`.
///
/// # Errors
///
/// Returns [`ShardSpecInputError`] when `start_row >= end_row` (inverted
/// range) or the encoded keys fail downstream spec validation.
#[must_use = "returns a Result that must be checked for validation errors"]
pub fn shard_spec_from_manifest_range_ref<'a>(
    manifest_id: u64,
    start_row: u64,
    end_row: u64,
    metadata: &'a [u8],
    start_buf: &'a mut KeyBuf,
    end_buf: &'a mut KeyBuf,
) -> Result<ShardSpecRef<'a>, ShardSpecInputError> {
    let start = ManifestRowKey::new(manifest_id, start_row);
    let end = ManifestRowKey::new(manifest_id, end_row);
    shard_spec_from_keys_ref(&start, &end, metadata, start_buf, end_buf)
}
```

This is a thin wrapper around `shard_spec_from_keys_ref`. It constructs two
`ManifestRowKey` values from the same `manifest_id` and delegates all
encoding and validation. Because both keys share the same manifest ID, the
shard covers a contiguous row range within a single manifest. Cross-manifest
shards require the general `shard_spec_from_keys_ref` with different
`manifest_id` values in the start and end keys.

### The _ref / _into Dual API Pattern

Each bridge helper has a companion `_into` variant that allocates the
validated spec into a `ShardArena`. The `_into` variant wraps the `_ref`
variant and maps errors through `ShardIntoError`. This two-phase pattern
separates validation (which is cheap) from allocation (which can fail due to
arena capacity). Callers that only need to inspect the spec without
persisting it use `_ref`. Callers that need to store the spec in the
coordinator's shard map use `_into`.

---

## ShardIntoError -- Two-Phase Error Reporting

The `_into` variants can fail in two distinct phases: validation and
allocation. The error type captures both.

Here is the definition from `key_encoding.rs`:

```rust
/// Error returned by `*_into` helpers when build or arena allocation fails.
///
/// The two-phase `*_into` helpers first build a borrowed `ShardSpecRef`,
/// then allocate it in a [`ShardArena`]. This enum distinguishes between
/// build-time validation failures (`Build`) and arena capacity exhaustion
/// (`SlabFull`). The generic parameter `E` is the build error type, which
/// varies by constructor (e.g., [`ShardSpecInputError`] for range/manifest,
/// [`PrefixShardError`] for prefix).
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ShardIntoError<E> {
    /// The spec failed validation before allocation was attempted.
    Build(E),
    /// The arena has no remaining capacity for this spec.
    SlabFull(SlabFull),
}

impl<E: fmt::Display> fmt::Display for ShardIntoError<E> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::Build(err) => write!(f, "{err}"),
            Self::SlabFull(err) => write!(f, "{err}"),
        }
    }
}

impl<E: std::error::Error + 'static> std::error::Error for ShardIntoError<E> {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::Build(err) => Some(err),
            Self::SlabFull(err) => Some(err),
        }
    }
}
```

The generic parameter `E` varies by constructor:

- For `shard_spec_from_keys_into` and `shard_spec_from_manifest_range_into`,
  `E` is `ShardSpecInputError`.
- For `shard_spec_from_prefix_into`, `E` is `PrefixShardError`.

The `Build` variant means the encoded keys failed validation -- inverted
range, oversized key, empty prefix, or no successor. The `SlabFull` variant
means the keys were valid but the arena has no remaining capacity. This
distinction matters for error recovery: a `Build` error is a logic bug in the
caller (bad input), while a `SlabFull` error is a resource exhaustion
condition that might be resolved by growing the arena or evicting old specs.

---

## Putting It Together

The full encoding pipeline for a filesystem connector looks like this:

```text
  1. Connector creates PathKey("data/repo/alpha")
  2. shard_spec_from_prefix_ref(b"data/repo/", metadata, &mut buf)
     a. Validates prefix is non-empty and within MAX_KEY_SIZE
     b. Computes prefix_successor(b"data/repo/") => b"data/repo0"
        (0x2F + 1 = 0x30, which is ASCII '0')
     c. Constructs ShardSpecRef { start: b"data/repo/", end: b"data/repo0" }
     d. Validates start < end (lexicographic)
  3. Coordinator stores the ShardSpec and compares boundaries as raw bytes
```

For a manifest connector:

```text
  1. Connector creates ManifestRowKey(42, 0) and ManifestRowKey(42, 1000)
  2. shard_spec_from_manifest_range_ref(42, 0, 1000, metadata, &mut s, &mut e)
     a. Encodes start: [0x00..0x2A | 0x00..0x00] (16 bytes)
     b. Encodes end:   [0x00..0x2A | 0x00..0x03E8] (16 bytes)
     c. Constructs ShardSpecRef with these boundaries
     d. Validates start < end
  3. Coordinator stores the ShardSpec and compares boundaries as raw bytes
```

In both cases, the coordinator sees only byte slices. The typed key domain is
erased at the translation boundary. This erasure is the entire point: the
coordinator's shard comparison, split, and bisection logic works identically
regardless of whether the underlying keys are filesystem paths, manifest
rows, or some future key type that has not been invented yet.

---

## Summary

The `gossip-frontier` crate's `key_encoding` module provides the translation
layer between typed connector keys and the coordinator's opaque byte
boundaries. The `KeyEncoding` trait enforces order preservation and
canonicality. `PathKey` implements identity encoding for filesystem paths.
`ManifestRowKey` implements fixed-width big-endian encoding for numeric
tuples. Three bridge helpers -- `shard_spec_from_keys_ref`,
`shard_spec_from_prefix_ref`, and `shard_spec_from_manifest_range_ref` --
construct validated `ShardSpec` values from typed inputs, each with a
zero-allocation `_ref` variant and an arena-allocating `_into` variant.

Now that keys can encode to comparable bytes, the next question is: given a
prefix or range, how does the system compute the boundaries for splitting?
Chapter 2 covers the range arithmetic functions -- `prefix_successor`,
`key_successor`, and `byte_midpoint` -- that answer this question.
