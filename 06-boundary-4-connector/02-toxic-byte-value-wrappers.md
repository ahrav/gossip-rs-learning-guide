# "Redacted at the Boundary" -- Toxic-Byte Value Wrappers

*A connector is configured for an S3 bucket. An `ItemRef` in the scan pipeline
encodes the object's ARN and access key ID -- the connector needs both to
construct a pre-signed URL for the read path. During development, someone
auto-derives `Debug` on the struct wrapping the raw bytes. A `Cursor` is
built with `token = Some(b"eyJjb250aW51YXRpb...")`
-- the S3 continuation token -- but accidentally passes `None` for `last_key`.
The coordinator stores the cursor opaquely. Scanning proceeds to the next
shard. Two hours later, the coordinator restarts from a checkpoint. It loads
the cursor: token `eyJjb250aW51YXRpb...` is present, but `last_key` is
`None`. The connector receives the cursor on resume. It has a token, but no
key context to determine position in the keyspace. The S3 API rejects the
stale token (the `ListObjectsV2` continuation token expired after 5 minutes).
The connector falls back to re-enumerating from the shard's start boundary.
Every item already processed -- all 47,000 objects enumerated in the first
pass -- is enumerated again. Downstream deduplication catches some, but items
whose content changed between passes generate duplicate findings. The scan
that was 85% complete restarts from zero.*

---

## Why Toxic-Byte Wrappers Exist

The failure above has a root cause that no amount of runtime retry logic can
fix: the `Cursor` type allowed an invalid state -- token without key -- to
exist. The connector produced it, the coordinator stored it, and the system
discovered the problem only on resume, two hours later.

The connector value types in `crates/gossip-contracts/src/connector/types.rs`
exist to make these invalid states unrepresentable at construction time.
Every byte that crosses the connector boundary -- item keys, item references,
pagination tokens, cursor state -- passes through a validated wrapper. The
wrappers enforce three invariants: non-emptiness (no zero-length byte
slices), bounded size (hard caps prevent unbounded growth), and redacted
formatting (no raw bytes in `Debug` or `Display` output).

Chapter 1 established the toxic-byte principle: all connector-originated
bytes are secrets-adjacent. This chapter examines the concrete types that
enforce that principle.

---

## The `define_toxic_bytes!` Macro

Three of the connector boundary types -- `ItemKey`, `ItemRef`, and
`TokenBytes` -- share identical structure: a newtype wrapping
`ToxicBytesStorage` with validated constructors and redacted formatting.
Rather than duplicating the implementation, a macro generates all three
from a compact declaration.

### Internal Storage: `ToxicBytesStorage`

Before examining the macro itself, it is important to understand the backing
storage. Toxic-byte wrappers support two representations through the
`ToxicBytesStorage` enum:

Here is the definition from `types.rs`:

```rust
/// Backing storage for toxic-byte wrappers.
///
/// `Owned` is the baseline COLD/WARM representation; `Pooled` is used by
/// HOT-path page emission, where wrappers borrow bytes from a page-local slab.
#[derive(Clone)]
enum ToxicBytesStorage {
    Owned(Box<[u8]>),
    Pooled(PooledToxicBytes),
}

impl ToxicBytesStorage {
    #[inline]
    fn as_bytes(&self) -> &[u8] {
        match self {
            Self::Owned(bytes) => bytes,
            Self::Pooled(pooled) => pooled.as_bytes(),
        }
    }

    #[inline]
    fn is_pooled(&self) -> bool {
        matches!(self, Self::Pooled(_))
    }
}
```

Two variants serve the tiered allocation policy:

- **`Owned(Box<[u8]>)`** -- The baseline COLD/WARM representation.
  Constructors `try_from_vec` and `try_from_slice` produce owned wrappers.
  Suitable for input boundaries, test fixtures, and any path where
  simplicity matters more than allocation cost.

- **`Pooled(PooledToxicBytes)`** -- The HOT-path representation. Wrappers
  borrow bytes from a page-local `ByteSlab` via a `ByteSlot` handle,
  keeping the backing `Arc<PooledByteSlab>` alive. Cloning a pooled wrapper
  bumps the `Arc` reference count -- no byte copying. This is how
  enumeration connectors avoid per-item heap allocation during page emission.

### The Macro Definition

Here is the definition from `types.rs`:

```rust
/// Generates a toxic-byte wrapper struct with validated constructors and
/// redacted formatting.
///
/// Two public arms control whether ordering traits are implemented:
/// - `ordered $name, max = $max` — includes `PartialOrd` + `Ord` (cursor keys).
/// - `$name, max = $max` — unordered (opaque handles and tokens).
///
/// Both arms produce: struct, `try_from_vec`, `try_from_slice`, `as_bytes`,
/// `len`, `AsRef<[u8]>`, `is_pooled`, and identical
/// `Debug`/`Display` (redacted to `TypeName(len=N, hash=XXXXXXXX..)`).
macro_rules! define_toxic_bytes {
    (
        $(#[$meta:meta])*
        ordered $name:ident, max = $max:expr
    ) => {
        define_toxic_bytes!(@base $(#[$meta])* $name, $max);

        impl ::core::cmp::PartialOrd for $name {
            fn partial_cmp(&self, other: &Self) -> Option<::core::cmp::Ordering> {
                Some(self.cmp(other))
            }
        }

        impl ::core::cmp::Ord for $name {
            fn cmp(&self, other: &Self) -> ::core::cmp::Ordering {
                self.as_bytes().cmp(other.as_bytes())
            }
        }
    };
    (
        $(#[$meta:meta])*
        $name:ident, max = $max:expr
    ) => {
        define_toxic_bytes!(@base $(#[$meta])* $name, $max);
    };
    (@base $(#[$meta:meta])* $name:ident, $max:expr) => {
        $(#[$meta])*
        #[derive(Clone)]
        #[allow(clippy::len_without_is_empty)]
        pub struct $name(ToxicBytesStorage);

        impl $name {
            pub fn try_from_vec(bytes: Vec<u8>) -> Result<Self, ConnectorInputError> {
                validate_toxic_bytes(stringify!($name), &bytes, $max)?;
                Ok(Self(ToxicBytesStorage::Owned(bytes.into_boxed_slice())))
            }

            pub fn try_from_slice(bytes: &[u8]) -> Result<Self, ConnectorInputError> {
                validate_toxic_bytes(stringify!($name), bytes, $max)?;
                Ok(Self(ToxicBytesStorage::Owned(bytes.to_vec().into_boxed_slice())))
            }

            /// Fallible constructor from a slab slot.
            ///
            /// This is used by HOT enumeration paths that build page-local
            /// slab-backed wrappers to avoid per-item heap allocation.
            pub fn try_from_slot(
                slot: ByteSlot,
                slab: Arc<PooledByteSlab>,
            ) -> Result<Self, ConnectorInputError> {
                let pooled = PooledToxicBytes::new(slot, slab);
                validate_toxic_bytes(stringify!($name), pooled.as_bytes(), $max)?;
                Ok(Self(ToxicBytesStorage::Pooled(pooled)))
            }

            #[inline]
            #[must_use]
            pub fn as_bytes(&self) -> &[u8] {
                self.0.as_bytes()
            }

            #[inline]
            #[must_use]
            pub fn len(&self) -> usize {
                self.as_bytes().len()
            }

            #[doc(hidden)]
            #[inline]
            #[must_use]
            pub fn is_pooled(&self) -> bool {
                self.0.is_pooled()
            }
        }

        impl AsRef<[u8]> for $name {
            #[inline]
            fn as_ref(&self) -> &[u8] {
                self.as_bytes()
            }
        }

        impl ::core::cmp::PartialEq for $name {
            fn eq(&self, other: &Self) -> bool {
                self.as_bytes() == other.as_bytes()
            }
        }

        impl ::core::cmp::Eq for $name {}

        impl ::core::hash::Hash for $name {
            fn hash<H: ::core::hash::Hasher>(&self, state: &mut H) {
                ::core::hash::Hash::hash(self.as_bytes(), state);
            }
        }

        impl ::core::fmt::Debug for $name {
            fn fmt(&self, f: &mut ::core::fmt::Formatter<'_>) -> ::core::fmt::Result {
                fmt_toxic_bytes(f, stringify!($name), self.as_bytes())
            }
        }

        impl ::core::fmt::Display for $name {
            fn fmt(&self, f: &mut ::core::fmt::Formatter<'_>) -> ::core::fmt::Result {
                fmt_toxic_bytes(f, stringify!($name), self.as_bytes())
            }
        }
    };
}
```

The macro has three arms:

- **`ordered $name, max = $max`** -- Implements `PartialOrd` and `Ord` via
  bytewise comparison through `as_bytes()`. Used for types that participate
  in lexicographic comparison (cursor keys, shard range boundaries). The
  ordered arm delegates to `@base` and then adds the ordering impls
  separately.

- **`$name, max = $max`** -- The unordered variant. Delegates directly to
  `@base` with no ordering traits. Used for opaque handles and tokens that
  are looked up, not sorted.

- **`@base`** -- The implementation arm that both public arms delegate to.
  It generates the struct (a single-field newtype over `ToxicBytesStorage`),
  three fallible constructors (`try_from_vec`, `try_from_slice`, and
  `try_from_slot`), two accessors (`as_bytes`, `len`), an `is_pooled`
  diagnostic method, an `AsRef<[u8]>` implementation, and hand-written
  `PartialEq`, `Eq`, `Hash`, `Debug`, and `Display` implementations.

**Key difference from a naive derive approach:** `PartialEq`, `Eq`, and
`Hash` are implemented manually rather than derived. The manual
implementations delegate to `as_bytes()`, ensuring that comparison and
hashing operate on the byte content regardless of whether the wrapper is
backed by owned or pooled storage. A derived `PartialEq` would compare the
`ToxicBytesStorage` enum discriminant first, causing an `Owned` wrapper and
a `Pooled` wrapper with identical bytes to compare as unequal.

The `#[allow(clippy::len_without_is_empty)]` attribute suppresses a Clippy
lint. Normally, a type with `len()` should also have `is_empty()`. Here,
`is_empty()` would unconditionally return `false` because constructors reject
empty input -- a method that always returns the same value adds no
information.

Both heap-allocating constructors delegate to the shared validation function:

Here is the definition from `types.rs`:

```rust
/// Validates the two shared constructor invariants for toxic-byte wrappers:
///
/// 1. `bytes` must not be empty (rejects zero-length slices).
/// 2. `bytes.len()` must not exceed `max`.
///
/// All wrapper constructors delegate here so the emptiness and size checks are
/// defined in exactly one place.
#[inline]
fn validate_toxic_bytes(
    field: &'static str,
    bytes: &[u8],
    max: usize,
) -> Result<(), ConnectorInputError> {
    if bytes.is_empty() {
        return Err(ConnectorInputError::Empty { field });
    }
    if bytes.len() > max {
        return Err(ConnectorInputError::TooLarge {
            field,
            size: bytes.len(),
            max,
        });
    }
    Ok(())
}
```

Two checks, one function, used by every toxic-byte wrapper. The `field`
parameter is a `&'static str` produced by `stringify!($name)` in the macro,
so error messages include the type name without runtime allocation.

The formatting function that both `Debug` and `Display` delegate to:

Here is the definition from `types.rs`:

```rust
/// Shared `Debug`/`Display` formatter for all toxic-byte wrappers.
///
/// Output format: `TypeName(len=N, hash=XXXXXXXX..)` where the hash is the
/// first 4 bytes of the BLAKE3 digest in lowercase hex. Both `Debug` and
/// `Display` intentionally produce identical output so callers cannot
/// accidentally bypass the toxic-byte redaction policy by switching format
/// traits.
#[inline]
fn fmt_toxic_bytes(
    f: &mut fmt::Formatter<'_>,
    type_name: &'static str,
    bytes: &[u8],
) -> fmt::Result {
    let [a, b, c, d] = hash_prefix_4(bytes);
    write!(
        f,
        "{type_name}(len={}, hash={:02x}{:02x}{:02x}{:02x}..)",
        bytes.len(),
        a,
        b,
        c,
        d
    )
}
```

The output format is deterministic: same bytes always produce the same hash
prefix. The 4-byte BLAKE3 prefix (8 hex characters) provides enough entropy
for log correlation without being a cryptographic commitment. `Debug` and
`Display` produce identical output. There is no format-trait backdoor to raw
bytes.

---

## `PooledByteSlab` and Zero-Alloc Page Emission

The `ToxicBytesStorage::Pooled` variant exists to serve the HOT enumeration
path. When a connector emits a page of items, it needs to construct
`ItemKey`, `ItemRef`, and `TokenBytes` wrappers for each item. With owned
storage, each wrapper allocates a separate `Box<[u8]>` on the heap --
acceptable for COLD/WARM paths, but costly when emitting hundreds of items
per page in a tight loop.

The pooled path eliminates these per-item allocations. The connector
allocates a single `ByteSlab` (a contiguous byte buffer from `gossip-stdx`)
sized to hold all wrapper bytes for the entire page. Individual fields are
staged into the slab via `PooledByteSlab::allocate`, which returns a
`ByteSlot` handle. The slab is then wrapped in `Arc<PooledByteSlab>` for
shared read-only access, and each wrapper is constructed via
`try_from_slot(slot, slab.clone())`.

Here is the definition from `types.rs`:

```rust
/// Shared page-local slab owner for pooled toxic-byte wrappers.
///
/// Connectors use the two-phase API:
/// 1. **Mutable phase:** call [`allocate`](Self::allocate) to stage byte fields.
/// 2. **Shared phase:** wrap in `Arc` for shared read-only access
///    via pooled toxic-byte wrappers.
pub struct PooledByteSlab {
    slab: ByteSlab,
}

impl PooledByteSlab {
    /// Wrap a page-local slab for staged allocation and later shared read access.
    #[inline]
    #[must_use]
    pub fn new(slab: ByteSlab) -> Self {
        Self { slab }
    }

    /// Stage bytes into the slab during page assembly.
    #[inline]
    pub fn allocate(&mut self, bytes: &[u8]) -> Result<ByteSlot, gossip_stdx::SlabFull> {
        self.slab.allocate(bytes)
    }
}
```

The `Drop` implementation on `PooledByteSlab` provides defense-in-depth
for toxic bytes: it calls `zeroize_used()` to overwrite all staged bytes
with zeros before the slab memory is freed, preventing sensitive residue
from lingering in heap memory. It then calls `clear()` to reset the slab's
internal state, satisfying `ByteSlab`'s debug assertion that no live slots
remain at drop time.

```rust
impl Drop for PooledByteSlab {
    fn drop(&mut self) {
        self.slab.zeroize_used();
        self.slab.clear();
    }
}
```

The pooled backing handle itself is straightforward:

```rust
/// Slab-backed toxic-byte storage handle.
///
/// `slot` indexes bytes inside `slab`; cloning this handle is allocation-free.
#[derive(Clone)]
struct PooledToxicBytes {
    slot: ByteSlot,
    slab: Arc<PooledByteSlab>,
}
```

Cloning a `PooledToxicBytes` increments the `Arc` reference count -- no byte
copying occurs. All wrappers from the same page share one `Arc<PooledByteSlab>`,
so the slab stays alive until the last wrapper is dropped.

The in-memory connector module documentation (`in_memory.rs:26-36`)
describes the concrete usage pattern: "Enumeration stages item keys, item
refs, and optional token bytes into a page-local `ByteSlab`, then
materializes wrappers with `ItemKey::try_from_slot`, `ItemRef::try_from_slot`,
and token construction from slots. This incurs one copy per staged field but
keeps wrapper cloning allocation-free in the HOT page-emission loop."

---

## `ItemKey` -- Ordered Enumeration Position

`ItemKey` is the ordered variant. It represents a position in the
connector's enumeration keyspace -- a file path, an S3 object key, a manifest
row encoding. The coordination layer stores these as cursor `last_key` fields,
and shard range boundaries are compared using their lexicographic order.

Here is the definition from `types.rs`:

```rust
define_toxic_bytes! {
    /// Ordered enumeration key used for sharding, paging, and cursor progression.
    ///
    /// ## Ordering
    ///
    /// `Ord` uses bytewise lexicographic comparison. This matches the cursor
    /// progression contract: advancing means moving to a strictly greater key.
    ///
    /// ## Toxic-byte policy
    ///
    /// `Debug` and `Display` are redacted (length + hash prefix). Raw bytes can
    /// be borrowed through [`as_bytes`](Self::as_bytes) / [`AsRef<[u8]>`](AsRef),
    /// or borrowed via [`AsRef<[u8]>`](AsRef).
    ordered ItemKey, max = MAX_ITEM_KEY_SIZE
}
```

The `ordered` keyword triggers the macro arm that implements `PartialOrd` and
`Ord` via bytewise comparison through `as_bytes()`. Lexicographic byte
comparison matches the cursor progression contract: advancing means moving to
a strictly greater key. The size limit is `MAX_ITEM_KEY_SIZE`:

Here is the definition from `types.rs`:

```rust
/// Maximum size of an [`ItemKey`] in bytes (4 KiB).
///
/// Kept in lock-step with the coordination cursor `last_key` limit
/// (`coordination::cursor::MAX_KEY_SIZE`) so that every valid connector key
/// can be stored directly in a cursor without truncation. Alignment is
/// verified by the `constants_align_with_coordination_limits` test.
pub const MAX_ITEM_KEY_SIZE: usize = 4_096;
```

The 4 KiB limit is not arbitrary. It matches the coordination cursor's
`MAX_KEY_SIZE` so that any key produced by a connector fits in cursor state
without truncation. The alignment is enforced by a test that compiles in the
same crate:

```rust
#[test]
fn constants_align_with_coordination_limits() {
    use crate::coordination::{CursorMaxTokenSize, MAX_KEY_SIZE};
    assert_eq!(MAX_ITEM_KEY_SIZE, MAX_KEY_SIZE);
    assert_eq!(MAX_TOKEN_SIZE, CursorMaxTokenSize);
}
```

If someone changes the coordination cursor limit without updating the
connector constant (or vice versa), this test fails.

---

## `ItemRef` -- Opaque Read Handle

`ItemRef` is the unordered variant. It is the handle a connector returns
alongside each enumerated item so the coordinator can later ask the connector
to open or read that item. The bytes are opaque to the coordination layer.

Here is the definition from `types.rs`:

```rust
define_toxic_bytes! {
    /// Opaque, credential-free connector handle for read/open operations.
    ///
    /// ## Differences from [`ItemKey`]
    ///
    /// - **Not ordered.** `ItemRef` does not derive `Ord` because the coordination
    ///   layer never compares or sorts references; they are looked up, not ranged.
    /// - **Independent size limit.** [`MAX_ITEM_REF_SIZE`] is not tied to cursor
    ///   field limits because references never enter cursor state.
    ///
    /// ## Toxic-byte policy
    ///
    /// `Debug` and `Display` are redacted (length + hash prefix). Raw bytes can
    /// be borrowed through [`as_bytes`](Self::as_bytes) / [`AsRef<[u8]>`](AsRef),
    /// or borrowed via [`AsRef<[u8]>`](AsRef).
    ItemRef, max = MAX_ITEM_REF_SIZE
}
```

Two differences from `ItemKey`:

- **Not ordered.** The macro invocation omits the `ordered` keyword, so
  `PartialOrd` and `Ord` are not implemented. References are looked up by
  identity, not compared by position. The coordination layer never sorts or
  ranges over references.

- **Independent size limit.** `MAX_ITEM_REF_SIZE` is 16 KiB, four times
  larger than `MAX_ITEM_KEY_SIZE`. This limit is not derived from cursor
  field constraints because references never enter cursor state -- they flow
  from enumeration to the read path without being persisted in the
  coordinator's checkpoint.

---

## `TokenBytes` -- Opaque Pagination Token

`TokenBytes` wraps connector-opaque pagination or resume tokens. The
coordinator persists these in cursor state and hands them back on the next
page request.

Here is the definition from `types.rs`:

```rust
define_toxic_bytes! {
    /// Connector-opaque pagination or resume token.
    ///
    /// ## Cursor coupling
    ///
    /// [`MAX_TOKEN_SIZE`] is derived from the coordination cursor `token` limit,
    /// so any token that passes validation here is guaranteed to fit in cursor
    /// state. A token only has meaning when paired with an [`ItemKey`] (the
    /// cursor `last_key`).
    ///
    /// ## Toxic-byte policy
    ///
    /// `Debug` and `Display` are redacted (length + hash prefix). Raw bytes can
    /// be borrowed through [`as_bytes`](Self::as_bytes) / [`AsRef<[u8]>`](AsRef),
    /// or borrowed via [`AsRef<[u8]>`](AsRef).
    TokenBytes, max = MAX_TOKEN_SIZE
}
```

Like `ItemRef`, `TokenBytes` is unordered. Like `ItemKey`, its size limit
is coupled to coordination: `MAX_TOKEN_SIZE` (16 KiB) matches the
coordination cursor's `CursorMaxTokenSize`. This coupling ensures that any
token passing connector validation fits in cursor state without truncation.
The `constants_align_with_coordination_limits` test enforces the alignment
at compile time.

The doc comment states a critical invariant: "A token only has meaning when
paired with an `ItemKey`." This is the invariant that the `Cursor` type
enforces structurally.

---

## `Cursor` -- The Two-Layer Paging Model

The `Cursor` type owns connector paging state and bridges to the
coordination layer's borrowed `CursorUpdate`. It enforces the invariant that
a token never exists without a key.

Here is the definition from `types.rs`:

```rust
/// Owned cursor used across connector boundaries.
///
/// ## Invariants
///
/// - `token` is only meaningful when paired with a `last_key`.
/// - Constructors make the invalid `(None, Some(token))` state
///   unrepresentable.
/// - [`try_from_update`](Self::try_from_update) validates external cursor
///   updates and rejects token-without-key states via
///   [`ConnectorInputError::TokenWithoutLastKey`].
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Cursor {
    last_key: Option<ItemKey>,
    token: Option<TokenBytes>,
}
```

Two private fields:

- **`last_key: Option<ItemKey>`** -- The last fully processed key. `None`
  means no progress has been made (initial state). `Some` means the connector
  has advanced to this position.

- **`token: Option<TokenBytes>`** -- A connector-opaque continuation token.
  `None` means no token is needed for the next page. `Some` means the
  connector requires this token to resume. The invariant: `token.is_some()`
  implies `last_key.is_some()`.

The fields are private. Three named constructors control the valid states:

Here is the definition from `types.rs`:

```rust
impl Cursor {
    /// Initial cursor: no progress key and no resume token.
    #[inline]
    #[must_use]
    pub fn initial() -> Self {
        Self {
            last_key: None,
            token: None,
        }
    }

    /// Cursor with a `last_key` and no token.
    #[inline]
    #[must_use]
    pub fn with_last_key(last_key: ItemKey) -> Self {
        Self {
            last_key: Some(last_key),
            token: None,
        }
    }

    /// Cursor with both `last_key` and token.
    #[inline]
    #[must_use]
    pub fn with_token(last_key: ItemKey, token: TokenBytes) -> Self {
        Self {
            last_key: Some(last_key),
            token: Some(token),
        }
    }
}
```

Three constructors, three valid states:

| Constructor | `last_key` | `token` | Meaning |
|-------------|-----------|---------|---------|
| `initial()` | `None` | `None` | No progress; start from beginning |
| `with_last_key(k)` | `Some(k)` | `None` | Progressed to `k`; no token needed |
| `with_token(k, t)` | `Some(k)` | `Some(t)` | Progressed to `k` with continuation token |

The fourth combination -- `(None, Some(t))` -- is unrepresentable. No
constructor produces it. The opening failure scenario is prevented by
construction.

The coordination bridge method makes the invariant enforcement explicit:

Here is the definition from `types.rs`:

```rust
    pub fn as_update(&self) -> CursorUpdate<'_> {
        match (&self.last_key, &self.token) {
            (None, None) => CursorUpdate::initial(),
            (Some(last_key), None) => CursorUpdate::with_last_key(last_key.as_bytes()),
            (Some(last_key), Some(token)) => {
                CursorUpdate::with_token(last_key.as_bytes(), token.as_bytes())
            }
            (None, Some(_)) => unreachable!(
                "Cursor invariant violated: token present without last_key. \
                 All constructors prevent this state."
            ),
        }
    }
```

The `(None, Some(_))` arm uses `unreachable!` rather than silently recovering.
If a future code change somehow violates the invariant, the system panics
immediately rather than silently producing a corrupted cursor.

The reverse direction -- constructing an owned `Cursor` from a borrowed
coordination `CursorUpdate` -- validates the invariant explicitly:

Here is the definition from `types.rs`:

```rust
    pub fn try_from_update(update: CursorUpdate<'_>) -> Result<Self, ConnectorInputError> {
        let last_key = match update.last_key() {
            None => None,
            Some(last_key) => Some(ItemKey::try_from_slice(last_key)?),
        };
        let token = match update.token() {
            None => None,
            Some([]) => None,
            Some(token) => Some(TokenBytes::try_from_slice(token)?),
        };
        if last_key.is_none() && token.is_some() {
            return Err(ConnectorInputError::TokenWithoutLastKey);
        }
        Ok(Self { last_key, token })
    }
```

Two normalization behaviors make this function robust:

1. **Empty token normalization.** `Some([])` is treated as `None`. This
   prevents a semantic difference between "no token" represented as `None`
   and "no token" represented as `Some(empty_bytes)`.

2. **Token-without-key rejection.** After both fields are parsed, the
   function checks the invariant explicitly and returns
   `ConnectorInputError::TokenWithoutLastKey` if violated.

---

## `ScanItem` -- Composite Metadata Bundle

Each enumerated item carries identity fields and optional metadata. The
`ScanItem` type bundles them with a builder pattern for the optional fields.

Here is the definition from `types.rs`:

```rust
/// Connector-owned item metadata produced by an enumeration pass.
///
/// `ScanItem` keeps core identity fields (`item_key`, `item_ref`,
/// `stable_item_id`, `version`) alongside optional bounded metadata used for
/// presentation and scheduling (`size_hint`, `content_hints`, `location`).
///
/// Optional fields default to `None` and can be set with builder-style methods
/// after constructing the required identity.
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ScanItem {
    item_key: ItemKey,
    item_ref: ItemRef,
    stable_item_id: StableItemId,
    version: VersionId,
    size_hint: Option<u64>,
    content_hints: Option<ContentHints>,
    location: Option<Location>,
}
```

Seven fields divide into two groups:

**Required identity fields** (set at construction):

- **`item_key: ItemKey`** -- The ordered position for paging and cursor
  progression. This is the field that advances the cursor.
- **`item_ref: ItemRef`** -- The opaque handle for later read/open operations.
  Only the originating connector interprets these bytes.
- **`stable_item_id: StableItemId`** -- The tenant-independent identity from
  Boundary 1. Downstream deduplication and occurrence tracking use this.
- **`version: VersionId`** -- Whether the content identity is strong or weak
  (covered below).

**Optional metadata fields** (set via builder methods):

- **`size_hint: Option<u64>`** -- A byte-size estimate for scheduling and
  budget accounting. Advisory only.
- **`content_hints: Option<ContentHints>`** -- MIME type and encoding hints.
  Bounded but not grammar-validated.
- **`location: Option<Location>`** -- Human-readable display location for
  logs and UI. Bounded and required to be non-empty.

The constructor requires all four identity fields and initializes optional
fields to `None`:

Here is the definition from `types.rs`:

```rust
impl ScanItem {
    #[must_use]
    pub fn new(
        item_key: ItemKey,
        item_ref: ItemRef,
        stable_item_id: StableItemId,
        version: VersionId,
    ) -> Self {
        Self {
            item_key,
            item_ref,
            stable_item_id,
            version,
            size_hint: None,
            content_hints: None,
            location: None,
        }
    }

    #[must_use]
    pub fn with_size_hint(self, size_hint: u64) -> Self {
        Self {
            size_hint: Some(size_hint),
            ..self
        }
    }

    #[must_use]
    pub fn with_content_hints(self, content_hints: ContentHints) -> Self {
        Self {
            content_hints: Some(content_hints),
            ..self
        }
    }

    #[must_use]
    pub fn with_location(self, location: Location) -> Self {
        Self {
            location: Some(location),
            ..self
        }
    }
}
```

The builder methods consume `self` and return `Self`, enabling chaining. The
`#[must_use]` attribute on each ensures the caller does not discard the
modified value.

The destructuring method returns all fields as owned values:

Here is the definition from `types.rs`:

```rust
    #[must_use]
    pub fn into_parts(
        self,
    ) -> (
        ItemKey,
        ItemRef,
        StableItemId,
        VersionId,
        Option<u64>,
        Option<ContentHints>,
        Option<Location>,
    ) {
        (
            self.item_key,
            self.item_ref,
            self.stable_item_id,
            self.version,
            self.size_hint,
            self.content_hints,
            self.location,
        )
    }
```

---

## `Budgets` -- Making Zero Budgets Unrepresentable

The `Budgets` type carries scan-level stop conditions. Its design uses
`NonZeroUsize` and `NonZeroU64` to make vacuous zero budgets a type error.

Here is the definition from `types.rs`:

```rust
/// Page/scan budget controls used by connector enumeration APIs.
///
/// `max_items` and `max_bytes` are stored as [`NonZeroUsize`] and
/// [`NonZeroU64`] respectively so that a vacuous zero budget is
/// unrepresentable. Use [`try_new`](Self::try_new) to construct.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Budgets {
    max_items: NonZeroUsize,
    max_bytes: NonZeroU64,
    deadline: Option<Instant>,
}
```

Three fields:

- **`max_items: NonZeroUsize`** -- Upper bound on the number of returned
  items. Stored as `NonZeroUsize` so zero is unrepresentable at the type
  level. A zero-item budget would make every page vacuously complete.
- **`max_bytes: NonZeroU64`** -- Upper bound on cumulative bytes consumed.
  Same `NonZero` rationale.
- **`deadline: Option<Instant>`** -- An optional monotonic deadline. `None`
  means no deadline. The `is_expired_at` method accepts the current instant
  as a parameter:

Here is the definition from `types.rs`:

```rust
    #[inline]
    #[must_use]
    pub fn is_expired_at(&self, now: Instant) -> bool {
        self.deadline.is_some_and(|deadline| now >= deadline)
    }
```

The `now` parameter is not a convenience -- it is a simulation-testing
requirement. A function that calls `Instant::now()` internally is
non-deterministic: its result depends on wall-clock time, which cannot be
controlled in a deterministic simulation. By accepting `now` as input, the
function becomes pure. The simulation harness controls time; the production
runtime passes the real instant.

---

## `VersionId` -- Strong vs. Weak Semantics

Connectors often have a content identifier for each item, but the
trustworthiness of that identifier varies. An S3 `ETag` for a non-multipart
upload is a strong version identity: the same ETag guarantees the same
content. A Git blob hash is similarly strong. But an HTTP `Last-Modified`
header is weak: the same timestamp does not guarantee identical content.

Here is the definition from `types.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum VersionId {
    /// Strong version identity that should reliably reference immutable content.
    Strong(ObjectVersionId),
    /// Weak version identity where content may change without version changes.
    Weak(ObjectVersionId),
}

impl VersionId {
    #[inline]
    #[must_use]
    pub const fn object_version_id(self) -> ObjectVersionId {
        match self {
            Self::Strong(version_id) | Self::Weak(version_id) => version_id,
        }
    }

    #[inline]
    #[must_use]
    pub const fn is_strong(self) -> bool {
        matches!(self, Self::Strong(_))
    }
}
```

Both variants carry the same `ObjectVersionId` payload. The enum tag
communicates confidence, not data. Downstream code that derives occurrence
identities uses `object_version_id()` without branching on strength.
Trust-sensitive decisions -- "should we skip re-scanning this item because
its version has not changed?" -- use `is_strong()` to decide whether the
version signal is reliable.

---

## `ConnectorInputError` -- Unified Input Validation

All input validation failures across the connector boundary surface through
a single error enum.

Here is the definition from `types.rs`:

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ConnectorInputError {
    /// A required field was empty (zero-length byte slice).
    Empty { field: &'static str },
    /// A field exceeded its hard size limit.
    TooLarge {
        field: &'static str,
        /// Actual byte length of the rejected input.
        size: usize,
        /// Upper bound for this field (one of the `MAX_*` constants).
        max: usize,
    },
    /// Cursor update had `token` without `last_key`.
    TokenWithoutLastKey,
    /// A budget field was zero, which would make the budget vacuous.
    ZeroBudget { field: &'static str },
}
```

Four variants cover the full validation surface:

- **`Empty { field }`** -- A required byte field was zero-length. The `field`
  string identifies which type was rejected (e.g., `"ItemKey"`, `"TokenBytes"`).
- **`TooLarge { field, size, max }`** -- A field exceeded its hard limit. The
  error carries the actual size and the maximum, making diagnostic messages
  self-contained.
- **`TokenWithoutLastKey`** -- The cursor paging invariant was violated. No
  additional data is needed because the invariant is unambiguous.
- **`ZeroBudget { field }`** -- A budget field was zero. The `field` string
  identifies which budget (e.g., `"Budgets.max_items"`, `"Budgets.max_bytes"`).

The `&'static str` fields are produced by `stringify!` in macro-generated
code or by string literals in hand-written constructors. No runtime
allocation is needed for error construction.

---

## Summary

The connector value types in `types.rs` form a layered defense against
invalid state at the connector boundary. The `define_toxic_bytes!` macro
generates `ItemKey` (ordered, 4 KiB max), `ItemRef` (unordered, 16 KiB max),
and `TokenBytes` (unordered, 16 KiB max) with identical validation and
redacted formatting. The internal representation is `ToxicBytesStorage`,
an enum with `Owned(Box<[u8]>)` and `Pooled(PooledToxicBytes)` variants.
`PartialEq`, `Eq`, and `Hash` are implemented manually (not derived) to
compare by byte content regardless of storage variant. The `try_from_slot`
constructor creates pooled wrappers backed by a page-local `ByteSlab` for
zero-alloc HOT-path emission via `PooledByteSlab`. `Cursor` uses
named constructors to make the token-without-key state unrepresentable.
`ScanItem` bundles required identity fields with optional metadata via a
builder pattern. `Budgets` uses `NonZeroUsize` and `NonZeroU64` to prevent
vacuous zero budgets, and its `is_expired_at` method accepts the current
instant as a parameter for deterministic simulation testing. `VersionId`
tags content identifiers with `Strong` or `Weak` confidence semantics.
`ConnectorInputError` provides a unified validation error surface across
all these types.

Chapter 3 shows how these types flow through connector `choose_split_point`,
`open`, and `read_range` methods, and how the `ScanDriver` adapter layer
bridges connectors to the scan pipeline.
