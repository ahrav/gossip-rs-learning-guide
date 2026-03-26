# "The Misinterpreted Metadata" -- Hint Metadata and Wire Framing

*A coordinator splits a manifest shard. The children inherit the parent's raw metadata bytes but have no way to know those bytes represent a manifest row range. The worker that acquires one of the children treats the metadata as a file-path prefix and enumerates the wrong items entirely. Shard `0x0007` covers manifest rows 0-500, but the worker interprets the 16-byte key boundaries as a path prefix `\x00\x00\x00\x00\x00\x00\x00\x07...` and scans a directory that does not exist. The scan completes with zero items but reports success. 500 manifest rows are never scanned. No error is raised. The coordinator marks the shard as complete. The data source has a 500-row blind spot that persists until someone notices the gap manually -- if they ever do.*

---

## 1. Why Typed Hints Exist

The failure above stems from a fundamental ambiguity: raw metadata bytes carry no indication of their intended interpretation. A 16-byte blob could be two big-endian `u64` manifest row bounds, or it could be a filesystem path prefix, or it could be an opaque connector token. Without a type tag in the wire format, every consumer must guess, and guesses are silent when wrong.

The hint system solves this by embedding a typed routing tag at the front of every metadata envelope. The tag tells the consumer whether the shard represents a generic byte range, a prefix scan, or a manifest row interval -- before any connector-specific bytes are touched. This chapter walks through the complete wire framing: the `ShardHint` enum and its three variants, the `ShardMetadata` envelope that wraps hints alongside connector-opaque data, the reusable scratch buffers that keep the encode/decode path allocation-free, and the convenience constructors that build validated `ShardSpecRef` values from typed inputs.

## 2. Module-Level Design

The hint module in `hint.rs` is organized around three design principles that shape every public API.

### 2.1 Allocation Discipline

Recall from Chapter 1 that `KeyEncoding` implementations write into caller-owned `KeyBuf` scratch buffers to avoid per-encode heap allocation. The hint module extends this discipline to metadata encoding. The module-level documentation in `hint.rs` states the contract directly:

> The public encode/decode APIs are designed for zero heap allocation in steady-state paths:
> - Decode returns borrowed views into caller-provided input bytes.
> - Encode writes into a caller-owned reusable `MetadataBuf`.
> - Convenience constructors (`range_shard_ref`, `prefix_shard_ref`, `manifest_shard_ref`) return borrowed `ShardSpecRef` values.

Decode outputs are lifetime-bound to the provided input slice. Encode is two-step: a preflight sizing check followed by a write into caller scratch. This pattern -- size, then write -- appears at both the hint level and the metadata envelope level.

### 2.2 No-Versioning Policy

There is no format version byte in the wire encoding. The module documentation states:

> Hint encoding is intentionally versionless: there is no format version byte and no compatibility shim. A hint is encoded as a tag plus fixed variant fields. Unknown tags are rejected immediately. Format evolution is additive by introducing new tags, not by in-band version negotiation.

This means a new hint type (say, a hypothetical `TableRange`) would receive tag `0x03`. Existing decoders see an unknown tag and return `ShardHintDecodeError::UnknownTag(0x03)` immediately. There is no negotiation, no fallback, no silent degradation.

### 2.3 Metadata Envelope Structure

The wire format for complete shard metadata is:

```text
[hint_len:u32 BE][hint_bytes][connector_extra]
```

The first four bytes declare how many of the following bytes belong to the hint frame. Everything after the hint frame is connector-owned opaque data that this module copies through untouched. The coordinator decodes only the hint frame; connectors decode the trailing `connector_extra` bytes using their own formats.

## 3. Wire Tag Constants

Here are the wire tag constants from `hint.rs`:

```rust
/// Wire tag for [`ShardHint::Range`].
const TAG_RANGE: u8 = 0x00;
/// Wire tag for [`ShardHint::Prefix`].
const TAG_PREFIX: u8 = 0x01;
/// Wire tag for [`ShardHint::Manifest`].
const TAG_MANIFEST: u8 = 0x02;
```

Three tags, three hint types. The tag is always the first byte of a hint frame. The constants are module-private -- external code interacts with the typed `ShardHint` enum, never with raw tag bytes. Tag assignment starts at `0x00` and increments sequentially. Because the no-versioning policy mandates additive evolution, a future fourth variant would receive `0x03`, a fifth would receive `0x04`, and so on. Tags `0x03` through `0xFF` are currently unassigned; any decoder encountering them returns `UnknownTag` immediately.

Supporting size constants define the fixed overhead for each variant:

```rust
/// Prefix hint header: 1-byte tag + 4-byte big-endian length.
const PREFIX_HEADER_LEN: usize = 1 + 4;
/// Manifest hint total size: 1-byte tag + three 8-byte big-endian u64 fields.
const MANIFEST_LEN: usize = 1 + 8 + 8 + 8;
/// Metadata envelope hint-length prefix size (4-byte big-endian u32).
const METADATA_HINT_LEN_PREFIX: usize = 4;
```

These constants serve dual purposes: they define the minimum expected input length during decode (used in truncation checks), and they define the exact output length during encode (used to slice the scratch buffer). The constant `U32_MAX_USIZE` (defined as `u32::MAX as usize`) is also present in the module; it establishes the ceiling for the prefix length field, which is framed as a `u32` in the wire format.

## 4. The `ShardHint` Enum

Here is the definition from `hint.rs`:

```rust
/// Routing hint embedded in shard metadata.
///
/// Hints let downstream components infer the intended key domain (range,
/// prefix, or manifest rows) without inspecting connector-specific payload
/// bytes.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ShardHint<'a> {
    /// Generic byte-range shard with no extra structured narrowing.
    ///
    /// Wire form: `[0x00]`.
    Range,
    /// Prefix-bounded shard. `prefix` bytes are connector-defined.
    ///
    /// Wire form: `[0x01][prefix_len:u32 BE][prefix_bytes]`.
    Prefix { prefix: &'a [u8] },
    /// Intended half-open row range within one manifest: `[start_row, end_row)`.
    ///
    /// Wire form:
    /// `[0x02][manifest_id:u64 BE][start_row:u64 BE][end_row:u64 BE]`.
    ///
    /// Both encode and decode enforce `start_row < end_row`; invalid bounds
    /// are rejected. Direct enum construction does not enforce this invariant.
    Manifest {
        /// Connector-defined manifest identifier.
        manifest_id: u64,
        /// Intended inclusive start row.
        start_row: u64,
        /// Intended exclusive end row.
        end_row: u64,
    },
}
```

The enum is `Copy` because every variant is either unit-like (`Range`), borrows a slice (`Prefix`), or holds three `u64` values (`Manifest`). The lifetime `'a` ties the `Prefix` variant's borrowed bytes to the input buffer, enforcing the zero-allocation decode contract at the type level. A `ShardHint` decoded from a byte slice cannot outlive that slice -- the compiler enforces this statically.

Note the doc comment on `Manifest`: "Both encode and decode enforce `start_row < end_row`; invalid bounds are rejected. Direct enum construction does not enforce this invariant." This is an intentional design asymmetry. Constructing a `ShardHint::Manifest { manifest_id: 7, start_row: 100, end_row: 50 }` is legal at the Rust type level, but attempting to encode it or decode it from wire bytes will fail. The encode/decode boundary is the validation boundary, not the constructor.

### Wire Format by Variant

Each variant encodes to a distinct wire form:

```text
Range:
  +------+
  | 0x00 |    (1 byte total)
  +------+

Prefix:
  +------+-------------------+---------------------+
  | 0x01 | prefix_len:u32 BE | prefix_bytes        |
  +------+-------------------+---------------------+
  1 byte    4 bytes             prefix_len bytes

Manifest:
  +------+-------------------+-------------------+-------------------+
  | 0x02 | manifest_id:u64 BE| start_row:u64 BE  | end_row:u64 BE   |
  +------+-------------------+-------------------+-------------------+
  1 byte    8 bytes             8 bytes             8 bytes
                                                        (25 bytes total)
```

**Range** is 1 byte. It carries no structured information beyond the tag itself. A Range hint says "this shard covers a byte range; consult the shard spec's start/end keys for the actual bounds."

**Prefix** is variable-length. The 4-byte big-endian length prefix allows prefixes up to `u32::MAX` bytes (in practice, bounded by `MAX_METADATA_SIZE`). The prefix bytes are connector-defined -- for a filesystem connector, they represent a directory path; for an API connector, a resource ID prefix.

**Manifest** is fixed at 25 bytes. The three `u64` fields identify which manifest and which half-open row interval `[start_row, end_row)` the shard covers. Both encode and decode enforce `start_row < end_row`. The fixed-width layout means Manifest hints require exactly 25 bytes regardless of the actual row numbers -- manifest 1 covering rows 0-10 takes the same space as manifest 999 covering rows 0-1,000,000,000. This predictability simplifies capacity planning for metadata envelopes.

## 5. The `ShardMetadata` Envelope

Here is the definition from `hint.rs`:

```rust
/// Structured shard metadata envelope.
///
/// `hint` is coordination-visible and validated by this module.
/// `connector_extra` is opaque and copied through untouched.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ShardMetadata<'a> {
    /// Coordination-visible routing hint.
    pub hint: ShardHint<'a>,
    /// Connector-private bytes, preserved opaquely by this module.
    pub connector_extra: &'a [u8],
}
```

**`hint`** is the typed routing information that the coordinator and split propagation logic inspect. It determines how a shard is interpreted when acquired.

**`connector_extra`** is an opaque byte slice that the hint module never interprets. Connectors use this for domain-specific data: bucket identifiers, authentication tokens, checkpoint cursors, or any other state that travels with the shard but is invisible to coordination logic.

The envelope wire format wraps these two components:

```text
Metadata Envelope:
  +-------------------+---------------------+---------------------+
  | hint_len:u32 BE   | hint_bytes          | connector_extra     |
  +-------------------+---------------------+---------------------+
  4 bytes               hint_len bytes        remaining bytes
```

Empty metadata (zero bytes) is a valid encoding that decodes as `ShardHint::Range` with empty `connector_extra`. This default makes Range the implicit hint for any shard that does not carry explicit metadata. The design choice is deliberate: shards created by coordination code that predates the hint system -- or by test code that does not care about hints -- automatically carry the least-specific hint type. No existing shard becomes invalid when the hint system is introduced.

The separation between `hint` and `connector_extra` creates a clean layering boundary. The coordination layer reads the hint to make routing and split-propagation decisions. The connector layer reads connector_extra to recover domain-specific context (which S3 bucket, which database cursor, which API pagination token). Neither layer needs to understand the other's data. When metadata passes through the coordinator during a split, the hint is transformed by the propagation logic (covered in Chapter 4) while the connector_extra is dropped -- child shards start fresh with whatever connector context their connector provides at registration.

## 6. Reusable Scratch: `MetadataBuf` and `ShardSpecScratch`

### MetadataBuf

Here is the definition from `hint.rs`:

```rust
/// Reusable fixed-capacity metadata scratch buffer.
///
/// Sized to hold any valid metadata envelope ([`MAX_METADATA_SIZE`] bytes).
/// Allocate one at startup or per-thread and reuse it across repeated
/// `encode_into` calls to avoid per-encode heap allocation.
///
/// The returned slice from `encode_into` borrows this buffer. Any
/// subsequent write overwrites the previous content; callers must
/// consume or copy the slice before the next encode.
pub struct MetadataBuf {
    buf: [u8; Self::CAPACITY],
    len: usize,
}
```

`MetadataBuf` is a fixed-capacity array on the stack, sized to `MAX_METADATA_SIZE`. The `len` field tracks how many bytes of the array contain valid encoded data. The `as_bytes()` method returns `&self.buf[..self.len]` -- a borrowed slice into the active region.

The key insight is the reuse pattern: allocate one `MetadataBuf` at startup, pass it to every `encode_into` call, and consume the returned slice before the next encode. Each encode overwrites the previous content. This eliminates per-encode heap allocation on the hot path.

The implementation provides `new()`, `as_bytes()`, `len()`, `is_empty()`, and `clear()` methods. `clear()` sets `len` to zero without touching the backing storage -- the array content is left intact, and only the active-length marker is reset. This makes `clear()` an O(1) operation regardless of buffer size. The `Default` trait implementation delegates to `new()`, and the `Debug` implementation shows both the active length and the active bytes, omitting the unused tail of the backing array.

### ShardSpecScratch

Here is the definition from `hint.rs`:

```rust
/// Caller-owned scratch for allocation-free shard-spec construction helpers.
///
/// Bundles a [`KeyBuf`] for the start bound, a [`KeyBuf`] for the end bound,
/// and a [`MetadataBuf`] for the encoded metadata envelope. One scratch can
/// be reused across many helper calls (`range_shard_ref`, `prefix_shard_ref`,
/// `manifest_shard_ref`).
///
/// Returned borrowed `ShardSpecRef` values alias this scratch and become
/// stale once the scratch is mutated again. Callers must allocate the spec
/// into an arena or copy it before reusing the scratch.
#[derive(Debug, Default)]
pub struct ShardSpecScratch {
    start: KeyBuf,
    end: KeyBuf,
    metadata: MetadataBuf,
}
```

`ShardSpecScratch` bundles the three scratch buffers that the convenience constructors need: a `KeyBuf` for the start key, a `KeyBuf` for the end key, and a `MetadataBuf` for the encoded metadata. One scratch instance serves many consecutive helper calls. Recall from Chapter 1 that `KeyBuf` is a fixed-capacity stack buffer for shard-key arithmetic -- `ShardSpecScratch` extends this principle to encompass the full shard specification.

The doc comment highlights a critical lifetime constraint: "Returned borrowed `ShardSpecRef` values alias this scratch and become stale once the scratch is mutated again." In practice, this means a connector registering 100 shards must allocate each `ShardSpecRef` into a `ShardArena` (or copy it) before calling the next convenience constructor. The `_into` variants (`range_shard_into`, `prefix_shard_into`, `manifest_shard_into`) handle this pattern automatically -- they call the `_ref` variant, then immediately allocate the result into an arena.

## 7. The Encode Path

Encoding is a two-step process: first compute the encoded size, then write the bytes.

### Hint Sizing

Here is `ShardHint::encoded_len` from `hint.rs`:

```rust
#[inline]
pub fn encoded_len(&self) -> Result<usize, ShardEncodeError> {
    match self {
        Self::Range => Ok(1),
        Self::Prefix { prefix } => {
            if prefix.len() > U32_MAX_USIZE {
                return Err(ShardEncodeError::PrefixTooLarge {
                    size: prefix.len(),
                    max: U32_MAX_USIZE,
                });
            }
            let size = PREFIX_HEADER_LEN.saturating_add(prefix.len());
            if size > MAX_METADATA_SIZE {
                return Err(ShardEncodeError::HintTooLarge {
                    size,
                    max: MAX_METADATA_SIZE,
                });
            }
            Ok(size)
        }
        Self::Manifest {
            start_row, end_row, ..
        } => {
            if *start_row >= *end_row {
                return Err(ShardEncodeError::InvertedManifestRows {
                    start_row: *start_row,
                    end_row: *end_row,
                });
            }
            Ok(MANIFEST_LEN)
        }
    }
}
```

Range is always 1 byte. Prefix validates that the prefix length fits in a `u32` framing field and that the total (header + payload) does not exceed `MAX_METADATA_SIZE`. Manifest validates row ordering and returns the fixed 25-byte size. The sizing step catches all encoding errors before any bytes are written -- no partial writes, no buffer corruption on error.

The two-stage prefix validation is worth noting. First, `prefix.len() > U32_MAX_USIZE` checks whether the prefix can be represented in the 4-byte length field at all. Second, `size > MAX_METADATA_SIZE` checks whether the framed hint fits within the system-wide metadata capacity. These produce different error variants (`PrefixTooLarge` vs. `HintTooLarge`) so callers can distinguish "prefix is too big for the wire format" from "hint is valid but exceeds the metadata budget."

### Hint Writing

Here is `ShardHint::encode_into` from `hint.rs`:

```rust
#[inline]
pub fn encode_into<'b>(&self, buf: &'b mut MetadataBuf) -> Result<&'b [u8], ShardEncodeError> {
    let hint_len = self.encoded_len()?;
    encode_hint_into_slice(*self, &mut buf.buf[..hint_len]);
    buf.len = hint_len;
    Ok(buf.as_bytes())
}
```

The method calls `encoded_len` for preflight validation, then delegates to the private `encode_hint_into_slice` to write the actual bytes. The returned slice borrows from `buf` and is valid until the next write. The `*self` dereference in the `encode_hint_into_slice` call is possible because `ShardHint` is `Copy`.

The private `encode_hint_into_slice` function handles the actual byte-level writes. For Range, it writes `TAG_RANGE` at offset 0. For Prefix, it writes `TAG_PREFIX` at offset 0, the big-endian `u32` prefix length at offsets 1-4, and the prefix bytes starting at offset 5. For Manifest, it writes `TAG_MANIFEST` at offset 0, followed by three big-endian `u64` values (manifest_id, start_row, end_row) at offsets 1-8, 9-16, and 17-24. A `debug_assert` verifies the output slice is large enough -- this assertion fires in debug builds if the sizing preflight and the actual write disagree on the required length.

### Metadata Envelope Sizing and Writing

Here is `ShardMetadata::encoded_len` from `hint.rs`:

```rust
#[inline]
pub fn encoded_len(&self) -> Result<usize, ShardEncodeError> {
    let hint_len = self.hint.encoded_len()?;
    let total_size = METADATA_HINT_LEN_PREFIX
        .checked_add(hint_len)
        .and_then(|s| s.checked_add(self.connector_extra.len()))
        .unwrap_or(usize::MAX);

    if total_size > MAX_METADATA_SIZE {
        return Err(ShardEncodeError::MetadataTooLarge {
            size: total_size,
            max: MAX_METADATA_SIZE,
        });
    }

    Ok(total_size)
}
```

The total size is `4 + hint_len + connector_extra.len()`. The `checked_add` chain prevents overflow -- if the arithmetic overflows, the result saturates to `usize::MAX`, which will always exceed `MAX_METADATA_SIZE`.

Here is `ShardMetadata::encode_into` from `hint.rs`:

```rust
#[inline]
pub fn encode_into<'b>(&self, buf: &'b mut MetadataBuf) -> Result<&'b [u8], ShardEncodeError> {
    let hint_len = self.hint.encoded_len()?;
    let total_size = METADATA_HINT_LEN_PREFIX
        .checked_add(hint_len)
        .and_then(|s| s.checked_add(self.connector_extra.len()))
        .unwrap_or(usize::MAX);

    if total_size > MAX_METADATA_SIZE {
        return Err(ShardEncodeError::MetadataTooLarge {
            size: total_size,
            max: MAX_METADATA_SIZE,
        });
    }

    let hint_len_u32 =
        u32::try_from(hint_len).expect("hint_len is validated to fit in u32 framing");
    let out = &mut buf.buf[..total_size];

    out[..METADATA_HINT_LEN_PREFIX].copy_from_slice(&hint_len_u32.to_be_bytes());

    let hint_start = METADATA_HINT_LEN_PREFIX;
    let hint_end = hint_start + hint_len;
    encode_hint_into_slice(self.hint, &mut out[hint_start..hint_end]);

    out[hint_end..].copy_from_slice(self.connector_extra);
    buf.len = total_size;
    Ok(buf.as_bytes())
}
```

The write sequence is: (1) write the 4-byte hint length prefix, (2) write the hint bytes, (3) copy the connector_extra bytes. The `expect` on `u32::try_from` is safe because `encoded_len` already validated that the hint length fits.

## 8. The Decode Path

### Hint Decode

Here is `ShardHint::decode` from `hint.rs`:

```rust
#[inline]
pub fn decode(data: &'a [u8]) -> Result<(ShardHint<'a>, usize), ShardHintDecodeError> {
    let Some(&tag) = data.first() else {
        return Err(ShardHintDecodeError::EmptyData);
    };

    match tag {
        TAG_RANGE => Ok((ShardHint::Range, 1)),
        TAG_PREFIX => decode_prefix(data),
        TAG_MANIFEST => decode_manifest(data),
        other => Err(ShardHintDecodeError::UnknownTag(other)),
    }
}
```

The function reads the first byte, dispatches on the tag, and returns both the decoded hint and the number of bytes consumed. Trailing bytes after the consumed region are intentionally ignored -- the caller is responsible for enforcing exact consumption at the outer envelope boundary. For `Prefix`, the decoded `prefix` slice borrows directly from `data`, maintaining the zero-allocation contract.

### Metadata Envelope Decode

Here is `ShardMetadata::decode` from `hint.rs`:

```rust
#[inline]
pub fn decode(metadata: &'a [u8]) -> Result<Self, ShardMetadataDecodeError> {
    if metadata.is_empty() {
        return Ok(Self::from_hint(ShardHint::Range));
    }

    let Some(hint_len_bytes) = metadata.get(..METADATA_HINT_LEN_PREFIX) else {
        return Err(ShardMetadataDecodeError { hint_error: None });
    };
    let hint_len = u32::from_be_bytes(hint_len_bytes.try_into().unwrap()) as usize;

    let hint_frame_end = METADATA_HINT_LEN_PREFIX
        .checked_add(hint_len)
        .ok_or(ShardMetadataDecodeError { hint_error: None })?;
    if hint_frame_end > metadata.len() {
        return Err(ShardMetadataDecodeError { hint_error: None });
    }

    let hint_frame = &metadata[METADATA_HINT_LEN_PREFIX..hint_frame_end];
    let (hint, consumed) =
        ShardHint::decode(hint_frame).map_err(|e| ShardMetadataDecodeError {
            hint_error: Some(e),
        })?;
    if consumed != hint_frame.len() {
        return Err(ShardMetadataDecodeError { hint_error: None });
    }

    Ok(Self {
        hint,
        connector_extra: &metadata[hint_frame_end..],
    })
}
```

The decode sequence is: (1) empty input maps to Range with empty connector_extra, (2) read the 4-byte hint length, (3) validate that hint_len bytes are available, (4) extract the hint frame, (5) decode the hint within that frame, (6) verify exact consumption, (7) everything after the hint frame is connector_extra.

The exact-consumption check (`consumed != hint_frame.len()`) catches malformed hints that decode successfully but leave unexpected trailing bytes within the declared hint region. Consider a hint_len of 8 but a hint that only consumes 5 bytes -- this indicates either a corrupt length prefix or a hint variant that does not fill the declared frame. The check rejects both cases.

The `checked_add` on `hint_frame_end` guards against arithmetic overflow. If `hint_len` is close to `usize::MAX`, adding it to `METADATA_HINT_LEN_PREFIX` would wrap around and produce a small value, potentially passing the bounds check against `metadata.len()`. The `checked_add` converts this to a `None`, which maps to the envelope-structural error.

## 9. Error Types

Three error types cover the failure modes across encode, hint-decode, and metadata-decode.

Here is `ShardHintDecodeError` from `hint.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, thiserror::Error)]
pub enum ShardHintDecodeError {
    /// Input is empty where a hint tag is required.
    EmptyData,
    /// Tag byte is not recognized.
    UnknownTag(u8),
    /// Prefix frame is incomplete.
    TruncatedPrefix { expected_min: usize, actual: usize },
    /// Manifest frame is incomplete.
    TruncatedManifest { expected_min: usize, actual: usize },
    /// Manifest row bounds are inverted or degenerate (`start_row >= end_row`).
    InvertedManifestRows { start_row: u64, end_row: u64 },
}
```

**`EmptyData`** fires when `decode` receives a zero-length slice. **`UnknownTag`** fires for any tag byte that is not `0x00`, `0x01`, or `0x02` -- this is how the no-versioning policy rejects future tags in old decoders. **`TruncatedPrefix`** and **`TruncatedManifest`** fire when the input is too short for the declared variant. Both carry the expected and actual sizes for diagnostics. **`InvertedManifestRows`** fires when the manifest's `start_row >= end_row`, even if the wire bytes are structurally valid.

Here is `ShardMetadataDecodeError` from `hint.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ShardMetadataDecodeError {
    /// The hint-level error when the envelope parsed but the hint did not.
    /// `None` for envelope-structural failures (short length prefix,
    /// length exceeds input, or consumed bytes != declared hint length).
    pub hint_error: Option<ShardHintDecodeError>,
}
```

This is a struct, not an enum. When `hint_error` is `Some`, the envelope parsed correctly but the inner hint frame was malformed. When `hint_error` is `None`, the envelope itself was structurally invalid (short length prefix, length exceeds available bytes, or hint consumed fewer bytes than declared).

Here is `ShardEncodeError` from `hint.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, thiserror::Error)]
#[non_exhaustive]
pub enum ShardEncodeError {
    /// Prefix bytes exceed `u32` framing width.
    PrefixTooLarge { size: usize, max: usize },
    /// Encoded hint exceeds metadata capacity.
    HintTooLarge { size: usize, max: usize },
    /// Encoded metadata exceeds [`MAX_METADATA_SIZE`].
    MetadataTooLarge { size: usize, max: usize },
    /// Manifest row bounds are inverted or degenerate (`start_row >= end_row`).
    InvertedManifestRows { start_row: u64, end_row: u64 },
}
```

**`PrefixTooLarge`** catches prefixes that exceed the `u32` framing width. This is a framing-level constraint: the wire format uses a 4-byte field for the prefix length, so prefixes longer than `u32::MAX` bytes cannot be represented at all.

**`HintTooLarge`** catches hints whose encoded form exceeds `MAX_METADATA_SIZE` on its own (before adding the envelope overhead). In practice, this fires for prefix hints with very long prefix bytes -- Range (1 byte) and Manifest (25 bytes) are always small enough.

**`MetadataTooLarge`** catches the total envelope (4-byte length prefix + hint bytes + connector_extra) exceeding `MAX_METADATA_SIZE`. A hint that fits on its own can still push the total past the limit when combined with large connector_extra bytes.

**`InvertedManifestRows`** catches the row-ordering violation `start_row >= end_row` at encode time. This is the same invariant that `ShardHintDecodeError::InvertedManifestRows` enforces at decode time. The dual enforcement means that even if hand-crafted wire bytes bypass the encoder, the decoder will independently reject inverted rows.

The `#[non_exhaustive]` attribute allows adding new encode error variants (for example, if a future hint type introduces a new failure mode) without a breaking change to callers' `match` arms.

## 10. Convenience Constructors

Three convenience constructors build validated `ShardSpecRef` values from typed inputs without requiring callers to manually encode hints and assemble metadata envelopes.

Here is `range_shard_ref` from `hint.rs`:

```rust
pub fn range_shard_ref<'a>(
    start: &'a [u8],
    end: &'a [u8],
    connector_extra: &[u8],
    scratch: &'a mut ShardSpecScratch,
) -> Result<ShardSpecRef<'a>, ShardSpecInputError> {
    let metadata = ShardMetadata::new(ShardHint::Range, connector_extra)
        .encode_into(scratch.metadata_buf())
        .map_err(map_encode_to_spec_input_error)?;
    let spec = ShardSpecRef::new(start, end, metadata);
    ShardSpec::validate_ref(spec)?;
    Ok(spec)
}
```

Here is `prefix_shard_ref` from `hint.rs`:

```rust
pub fn prefix_shard_ref<'a>(
    prefix: &'a [u8],
    connector_extra: &[u8],
    scratch: &'a mut ShardSpecScratch,
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
    let (end_buf, metadata_buf) = (&mut scratch.end, &mut scratch.metadata);
    let end = prefix_successor(prefix, end_buf).ok_or(PrefixShardError::NoSuccessor)?;
    let metadata = ShardMetadata::new(ShardHint::Prefix { prefix }, connector_extra)
        .encode_into(metadata_buf)
        .map_err(map_encode_to_prefix_error)?;
    let spec = ShardSpecRef::new(prefix, end, metadata);
    ShardSpec::validate_ref(spec).map_err(PrefixShardError::from)?;
    Ok(spec)
}
```

Recall from Chapter 2 that `prefix_successor` computes the exclusive upper bound for a prefix scan. `prefix_shard_ref` uses it to derive the end key automatically: a prefix shard covering `"src/"` gets end key `"src0"` (the byte after `'/'`). The prefix is validated for emptiness, size, and successor existence before metadata encoding begins. This ordering ensures callers always see `PrefixTooLarge` for oversized prefixes, not a generic `MetadataTooLarge` from the hint encoder.

Here is `manifest_shard_ref` from `hint.rs`:

```rust
pub fn manifest_shard_ref<'a>(
    manifest_id: u64,
    start_row: u64,
    end_row: u64,
    connector_extra: &[u8],
    scratch: &'a mut ShardSpecScratch,
) -> Result<ShardSpecRef<'a>, ShardSpecInputError> {
    ManifestRowKey::new(manifest_id, start_row).encode_into(&mut scratch.start);
    ManifestRowKey::new(manifest_id, end_row).encode_into(&mut scratch.end);
    let (start_buf, end_buf, metadata_buf) = (&scratch.start, &scratch.end, &mut scratch.metadata);
    let metadata = ShardMetadata::new(
        ShardHint::Manifest {
            manifest_id,
            start_row,
            end_row,
        },
        connector_extra,
    )
    .encode_into(metadata_buf)
    .map_err(map_encode_to_spec_input_error)?;
    let spec = ShardSpecRef::new(start_buf.as_bytes(), end_buf.as_bytes(), metadata);
    ShardSpec::validate_ref(spec)?;
    Ok(spec)
}
```

`manifest_shard_ref` encodes both `ManifestRowKey` boundaries into the scratch `KeyBuf` fields, then encodes the `Manifest` hint into the scratch `MetadataBuf`, and finally assembles and validates the `ShardSpecRef`. The `ManifestRowKey::encode_into` calls produce the fixed-width 16-byte big-endian encodings described in Chapter 1. The resulting shard's key range boundaries and its metadata hint carry the same row interval -- the boundaries define where the shard sits in the global keyspace, and the hint tells consumers what that position means semantically.

Each convenience constructor follows the same three-step pattern: (1) encode metadata into the scratch buffer, (2) assemble a `ShardSpecRef` from the raw byte slices, (3) validate the assembled spec via `ShardSpec::validate_ref`. If validation fails, the error is mapped through a dedicated translation function (`map_encode_to_spec_input_error` for range/manifest, `map_encode_to_prefix_error` for prefix) that converts `ShardEncodeError` into the error type expected by the caller. This mapping ensures that callers of `prefix_shard_ref` always receive `PrefixShardError` variants and never see raw encode errors.

### Decode Helpers

Three corresponding decode helpers extract typed information from an existing shard spec. Here are the definitions from `hint.rs`:

```rust
pub fn decode_metadata<'a, S>(spec: S) -> Result<ShardMetadata<'a>, ShardMetadataDecodeError>
where
    S: IntoShardSpecRef<'a>,
{
    let spec = spec.into_spec_ref();
    ShardMetadata::decode(spec.metadata())
}

pub fn decode_hint<'a, S>(spec: S) -> Result<ShardHint<'a>, ShardMetadataDecodeError>
where
    S: IntoShardSpecRef<'a>,
{
    decode_metadata(spec).map(|metadata| metadata.hint)
}

pub fn decode_connector_extra<'a, S>(spec: S) -> Result<&'a [u8], ShardMetadataDecodeError>
where
    S: IntoShardSpecRef<'a>,
{
    decode_metadata(spec).map(|metadata| metadata.connector_extra)
}
```

`decode_metadata` extracts the full envelope. `decode_hint` extracts only the hint, discarding connector_extra. `decode_connector_extra` extracts only the opaque trailing bytes. All three validate the full envelope first -- `decode_connector_extra` is not a raw slice accessor that returns unvalidated bytes. Even when a caller only needs the connector_extra, the hint frame is decoded and validated, ensuring that malformed metadata is always surfaced as an error.

The generic `S: IntoShardSpecRef<'a>` bound accepts both owned `ShardSpec` references and borrowed `ShardSpecRef` values. This flexibility means callers can pass `&ShardSpec` (an owned shard from the coordinator's shard map) or a `ShardSpecRef` (a zero-copy borrowed view from an arena) without manual conversion. The returned hint or connector_extra borrows from the spec's metadata bytes, so the lifetime `'a` ties the output to the input spec's storage.

## 11. Summary

The hint wire framing establishes a typed metadata layer between the coordinator's raw byte view and the connector's domain-specific interpretation. `ShardHint` tags each shard with its intended domain (Range, Prefix, or Manifest). `ShardMetadata` wraps the hint in an envelope that separates coordination-visible routing from connector-private data. The encode/decode path is allocation-free in steady state, using caller-owned `MetadataBuf` and `ShardSpecScratch` buffers. Three convenience constructors -- `range_shard_ref`, `prefix_shard_ref`, and `manifest_shard_ref` -- build validated shard specs from typed inputs, and three decode helpers extract typed information back out.

With typed hints embedded in shard metadata, the next question is: what happens to those hints when a shard splits? [Chapter 4](04-inheriting-intent.md) introduces hint propagation -- the rules that determine how a parent's hint transforms into child hints during a split-replace operation.
