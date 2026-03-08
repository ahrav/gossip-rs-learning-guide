# "The Perfect Test Double" -- In-Memory Deterministic Connector

*A property test generates 10,000 random file-path keys -- byte strings between 1 and 64 bytes drawn from the range `0x01..0x7F` -- and feeds them to a connector. The test asserts that enumerating with a page size of 1 and resuming 10,000 times produces the same set of items as a single full enumeration with `max_items` set to `u64::MAX`. On run 7,431 of the resume path, the test observes a different item key: the single-page enumeration returned `b"data/report"` at position 7,431, but the page-by-page resume returned `b"data/repors"`. The token-based resume path returned the wrong index because a duplicate key existed in the input set: two items both had key `b"data/report"`. The binary search via `upper_bound` landed on the second duplicate, skipping the first. The page-by-page enumeration produced 9,999 items; the full enumeration produced 10,000. Coverage silently diverged. The constructor should have rejected duplicates.*

---

## Why an In-Memory Connector Exists

The failure above is a test infrastructure bug, not a connector logic bug. The enumeration algorithm itself is correct -- it is the input that violated an assumption (unique keys). Catching this class of error requires a connector that is fully deterministic: same input, same output, every time. No network latency, no API rate limits, no eventual consistency, no filesystem race conditions.

The `InMemoryDeterministicConnector` in the `gossip-connectors` crate serves this purpose. It provides the full connector API surface -- enumeration, read, and split-point methods -- using sorted in-memory arrays. All BLAKE3 hashing is done once at construction. Subsequent page emissions use the pooled page assembly path: item keys, item refs, and optional token bytes are staged into a page-local `ByteSlab`, then materialized via `ItemKey::try_from_slot`, `ItemRef::try_from_slot`, and token construction from slots. This incurs one copy per staged field but keeps wrapper cloning allocation-free in the HOT page-emission loop. The page's cursor and items share the same slab owner (`PooledByteSlab`), so returning either one by value keeps pooled bytes valid until the last clone is dropped. The connector is designed for unit tests, conformance harnesses, and simulation workloads where reproducibility matters more than source realism.

The traits from Chapter 3 define the interface this connector implements. The `ItemKey`, `ScanItem`, and `Budgets` from Chapter 2 are the concrete types flowing through the enumeration.

---

## The Input Type: MemItem

Test callers construct their fixture data as a `Vec<MemItem>`. Each `MemItem` pairs a key with immutable content bytes.

Here is the definition from `in_memory.rs`:

```rust
/// One in-memory record served by [`InMemoryDeterministicConnector`].
#[derive(Clone, Debug)]
pub struct MemItem {
    /// Item key used for lexicographic ordering and cursor progression.
    pub key: ItemKey,
    /// Immutable payload returned by read operations.
    ///
    /// Wrapped in `Arc` to keep fixture duplication cheap.
    pub bytes: Arc<[u8]>,
}

impl MemItem {
    /// Build a memory-backed item with shared immutable bytes.
    ///
    /// Accepting `Into<Arc<[u8]>>` keeps fixture construction ergonomic while
    /// preserving zero-copy sharing across cloned fixture items.
    pub fn new(key: ItemKey, bytes: impl Into<Arc<[u8]>>) -> Self {
        Self {
            key,
            bytes: bytes.into(),
        }
    }
}
```

Field-by-field breakdown:

- **`key: ItemKey`** -- The item's key, used for lexicographic ordering and cursor progression. `ItemKey` is a bounded byte string from the connector contract layer; it is the same type that appears in `ScanItem` and `Cursor`.
- **`bytes: Arc<[u8]>`** -- The item's content payload. Wrapped in `Arc<[u8]>` (a reference-counted shared byte slice) rather than `Vec<u8>` because test fixtures frequently clone items when constructing multiple connectors from the same input set. With `Arc`, cloning an item bumps a reference count instead of copying the entire byte buffer. The `new()` constructor accepts `impl Into<Arc<[u8]>>`, so callers can pass `Vec<u8>`, `&[u8]`, or an existing `Arc<[u8]>`.

---

## The Internal Type: PreparedItem

At construction time, each `MemItem` is transformed into a `PreparedItem` with precomputed metadata. This is the single most important performance decision in the connector: all BLAKE3 hashing happens once, at construction, not on every page emission.

Here is the definition from `in_memory.rs`:

```rust
/// Precomputed per-item metadata, built once at construction time.
///
/// Every field except `key` and `bytes` is precomputed at construction time.
/// `stable_item_id` and `version_id` are derived from key/bytes plus the
/// connector tag; `item_ref` is the item's positional index in the sorted
/// array; `size_hint` is `bytes.len()`. All are deterministic for a fixed
/// input set. Computing them once eliminates redundant BLAKE3 hashing and
/// index-to-[`ItemRef`] conversion on every page emission. This mirrors the
/// `FileEntry` pattern used by `FilesystemConnector` (note: `FileEntry` is now
/// a transient staging type in the filesystem connector's streaming model,
/// not a persistent index entry; `GitEntry` in the git connector is the
/// closer analogue).
struct PreparedItem {
    /// Sorted item key for ordering and cursor progression.
    key: ItemKey,
    /// Immutable payload bytes for read operations.
    bytes: Arc<[u8]>,
    /// Big-endian `u64` index into the sorted item array.
    item_ref: ItemRef,
    /// Domain-separated BLAKE3 hash of `(connector_tag, key)`.
    stable_item_id: StableItemId,
    /// Domain-separated BLAKE3 hash of the content bytes.
    version_id: ObjectVersionId,
    /// Precomputed `bytes.len() as u64`.
    size_hint: u64,
}
```

Field-by-field breakdown:

- **`key: ItemKey`** -- Carried forward from the input `MemItem`. Used for binary search during shard-bound resolution and cursor advancement.
- **`bytes: Arc<[u8]>`** -- The shared content payload. Carried forward unchanged; the `open` method returns a reader over these bytes.
- **`item_ref: ItemRef`** -- A big-endian `u64` encoding of this item's index in the sorted array. Item 0 gets `ItemRef([0,0,0,0,0,0,0,0])`, item 1 gets `ItemRef([0,0,0,0,0,0,0,1])`, and so on. This makes `ItemRef` values compact, deterministic, and decodable back to an index in O(1).
- **`stable_item_id: StableItemId`** -- A domain-separated BLAKE3 hash of `(connector_tag, connector_instance_hash, key)`. Computed via the shared `derive_stable_item_id` helper in `common.rs`. The connector tag and connector instance hash ensure that the same key from different connectors or different connector instances produces distinct identity hashes.
- **`version_id: ObjectVersionId`** -- A domain-separated BLAKE3 hash of the content bytes. Computed via `ObjectVersionId::from_version_bytes`. This is the content-addressed version fingerprint: if the bytes change, the version ID changes.
- **`size_hint: u64`** -- Precomputed `bytes.len() as u64`. Used by the split-point algorithm to compute byte-weighted medians without re-measuring content length on every call.

`PreparedItem` also implements a trait abstraction from `common.rs` that enables generic binary search:

Here is the definition from `in_memory.rs`:

```rust
impl common::KeyedEntry for PreparedItem {
    fn key_bytes(&self) -> &[u8] {
        self.key.as_bytes()
    }
}
```

This implementation allows the `lower_bound` and `upper_bound` functions in
`common.rs` to operate generically over both `PreparedItem` (in-memory
connector) and `GitEntry` (git connector). Split-point selection uses the
shared `StreamingSplitEstimator` from `split_estimator.rs` rather than a
trait-based approach.

---

## The Connector Struct

Here is the definition from `in_memory.rs`:

```rust
/// Deterministic in-memory connector for shard-based enumeration and reads.
///
/// The connector is cheaply [`Clone`]able: internal prepared-item storage is
/// shared via [`Arc`], so cloning costs only an atomic reference count bump
/// plus one `Copy` field. Trait methods take `&mut self` because the trait
/// signatures require it, but no internal mutation occurs after construction.
///
/// # Determinism contract
///
/// Given the same input `Vec<MemItem>`:
///
/// - Items are sorted lexicographically by [`ItemKey`] at construction time.
/// - Duplicate keys are **rejected** (the constructor panics).
/// - [`ItemRef`] values are big-endian `u64` indices into that sorted vector,
///   so the Nth item always gets `ItemRef(N)`.
/// - [`StableItemId`] is derived via [`ItemIdentityKey::stable_id`](gossip_contracts::identity::ItemIdentityKey::stable_id)
///   (domain-separated BLAKE3 of connector-tag + connector-instance + key)
///   and [`ObjectVersionId`]
///   via [`ObjectVersionId::from_version_bytes`] (domain-separated BLAKE3 of
///   the item's content bytes), both deterministic and precomputed once at
///   construction. All items are emitted with [`VersionId::Strong`] because
///   the in-memory content is immutable.
///
/// Together these choices make enumeration order, identity fields, and read
/// handles reproducible across runs for identical inputs.
///
/// # Resume semantics
///
/// When tokens are enabled (the default), the cursor token carries the
/// next-index as a big-endian `u64`. On resume, the token is used as an
/// O(1) fast path with validation against `last_key`; on mismatch the
/// connector falls back to O(log n) binary search. See module docs for full
/// detail.
///
/// # Capabilities
///
/// The connector advertises `seek_by_key`, `token_resume` (when enabled),
/// `range_read`, and `split_hints` -- the full capability set.
#[derive(Clone)]
pub struct InMemoryDeterministicConnector {
    /// Sorted, precomputed item storage shared via [`Arc`] for cheap cloning.
    /// Built once at construction; never mutated afterward.
    items: Arc<[PreparedItem]>,
    /// When `true`, enumeration pages emit big-endian index tokens and the
    /// resume path uses them as an O(1) fast path with key-based fallback.
    /// When `false`, tokens are neither emitted nor consumed.
    emit_tokens: bool,
}
```

Field-by-field breakdown:

- **`items: Arc<[PreparedItem]>`** -- The sorted, precomputed item array shared via `Arc`. This is the central design decision for cheap cloning: multiple connector instances (e.g., one per test thread, or cloned into a simulation harness) share the same underlying data. Cloning the connector bumps an atomic reference count; it does not copy the item array. The `Arc<[PreparedItem]>` type (a reference-counted slice, not a `Vec`) means the allocation is a single contiguous block with no indirection.
- **`emit_tokens: bool`** -- Controls whether enumeration pages include pagination tokens. When `true` (the default), each page's continuation cursor carries a big-endian `u64` token encoding the next index. On resume, this token provides an O(1) fast path: the connector reads the index directly instead of performing a binary search. When `false`, tokens are neither emitted nor consumed, and resume always uses O(log n) key-based binary search. Disabling tokens is useful for testing key-only resume behavior or simulating connectors that cannot persist opaque token state.

---

## Construction: Sorting, Deduplication, and Precomputation

Here is the definition from `in_memory.rs`:

```rust
    /// Build a connector from an unordered collection of in-memory items.
    ///
    /// Items are sorted lexicographically by key (O(n log n)), verified to
    /// contain no duplicate keys, and then all per-item metadata is
    /// precomputed into internal records. Subsequent page emission pays
    /// only clone/copy costs.
    ///
    /// Token emission is enabled by default. Use [`with_tokens(false)`] to
    /// disable it.
    ///
    /// # Panics
    ///
    /// Panics if any two items share the same key. Duplicate keys are a
    /// test-setup bug and cannot be paginated correctly (see module docs).
    /// The panic message reports the offending key through [`ItemKey`] display
    /// formatting so diagnostics stay redacted under the toxic-byte policy.
    ///
    /// [`with_tokens(false)`]: Self::with_tokens
    pub fn new(connector_tag: ConnectorTag, connector_instance_id: impl AsRef<[u8]>, mut items: Vec<MemItem>) -> Self {
        items.sort_by(|left, right| left.key.cmp(&right.key));
        let connector_instance =
            ConnectorInstanceIdHash::from_instance_id_bytes(connector_instance_id.as_ref());

        // Enforce unique keys. Duplicate keys break cursor resume
        // (upper_bound skips remaining duplicates), StableItemId derivation
        // (collisions), and split-point selection (empty shards).
        if let Some(pos) = items.windows(2).position(|w| w[0].key == w[1].key) {
            panic!(
                "InMemoryDeterministicConnector requires unique item keys; \
                 duplicate key {} at sorted position {pos}",
                items[pos].key
            );
        }

        let prepared: Arc<[PreparedItem]> = items
            .into_iter()
            .enumerate()
            .map(|(idx, item)| {
                let idx_u64 = u64::try_from(idx).expect("item count must fit in u64");
                let item_ref = ItemRef::try_from_slice(&idx_u64.to_be_bytes())
                    .expect("8-byte big-endian index is always valid for ItemRef");
                let stable_item_id = derive_stable_item_id(connector_tag, connector_instance, &item.key);
                let version_id = ObjectVersionId::from_version_bytes(&item.bytes);
                let size_hint = item.bytes.len() as u64;
                PreparedItem {
                    key: item.key,
                    bytes: item.bytes,
                    item_ref,
                    stable_item_id,
                    version_id,
                    size_hint,
                }
            })
            .collect();

        Self {
            items: prepared,
            emit_tokens: true,
        }
    }
```

The constructor performs four steps in sequence:

**Step 1: Sort.** Items are sorted lexicographically by `ItemKey`. This is O(n log n) and happens once. After sorting, item positions are stable and can be used as `ItemRef` values.

**Step 2: Instance identity.** The `connector_instance_id` bytes are hashed into a `ConnectorInstanceIdHash` via `from_instance_id_bytes`. This scoped hash participates in `StableItemId` derivation so that the same key from two different connector instances produces distinct identities.

**Step 3: Duplicate rejection.** The `windows(2)` scan checks adjacent pairs in the sorted array for equal keys. If a duplicate is found, the constructor panics with a diagnostic message including the duplicate key's redacted display and its sorted position. The panic is deliberate: duplicate keys are a test-setup bug, not a runtime condition. The module documentation explains the three ways duplicates break correctness: cursor resume via `upper_bound` skips remaining duplicates, `StableItemId` derivation collides, and split-point selection can return a duplicate key producing empty shards.

**Step 4: Precomputation.** Each `MemItem` is transformed into a `PreparedItem` with all metadata computed eagerly. The `ItemRef` is the big-endian encoding of the item's sorted index. The `StableItemId` is a domain-separated BLAKE3 hash of the connector tag, connector instance hash, and key. The `ObjectVersionId` is a domain-separated BLAKE3 hash of the content bytes. The `size_hint` is `bytes.len()`. After this loop, no further hashing occurs -- page emission is pure copy.

The `with_tokens` builder method controls token behavior:

Here is the definition from `in_memory.rs`:

```rust
    /// Enable or disable pagination token emission/consumption.
    ///
    /// Disabling tokens is useful when callers want to validate key-only resume
    /// behavior or simulate connectors that cannot persist opaque token state.
    #[must_use]
    pub fn with_tokens(mut self, enabled: bool) -> Self {
        self.emit_tokens = enabled;
        self
    }
```

---

## The Core Enumeration Algorithm

The `enumerate_page_bounds` method is the heart of the connector. It implements a 5-phase algorithm that resolves shard boundaries, advances past the cursor, and extracts a page of precomputed metadata.

Here is a simplified conceptual version of the method from `in_memory.rs`. The
actual implementation delegates to shared helpers in `common.rs` —
`common::resolve_page_bounds` for range/deadline resolution,
`common::key_resume_start` for cursor advancement, and
`common::assemble_pooled_page` for slab-backed page assembly — rather than
inlining these steps. See `crates/gossip-connectors/src/in_memory.rs:323-405`
for the full pooled implementation.

```rust
    /// Core page enumeration for both shard-based and explicit-range callers.
    ///
    /// # Algorithm
    ///
    /// 1. **Budget gate** -- Return retryable error if deadline expired.
    /// 2. **Range validation** -- Reject `start > end` as a permanent error.
    /// 3. **Range resolution** -- Map keys to indices via binary search.
    /// 4. **Cursor advancement** -- Resume from `last_key` position.
    ///    When tokens are enabled, the token is used as an O(1) fast path:
    ///    validated by checking `items[token_idx - 1].key == last_key`. On
    ///    mismatch, falls back to O(log n) `upper_bound` binary search.
    /// 5. **Page extraction** -- Yield up to `max_items` consecutive items
    ///    from precomputed metadata (clone/copy only, no hashing).
    fn enumerate_page_bounds(
        &self,
        start: Option<&[u8]>,
        end: Option<&[u8]>,
        cursor: &Cursor,
        budgets: Budgets,
    ) -> Result<EnumerationPage, EnumerateError> {
        // Phase 1-3: resolve bounds and check deadline/range validity.
        let bounds = common::resolve_page_bounds(&self.items, start, end, budgets)?;

        // Phase 4: advance past the cursor's last-emitted key.
        let key_resume = common::key_resume_start(&self.items, cursor, bounds.range_start);
        let mut start_idx = key_resume;

        // O(1) token fast path: when tokens are enabled and the cursor carries
        // a well-formed token whose preceding item matches `last_key`, jump
        // directly to the token index. On mismatch, fall back to the
        // key-authoritative position computed above.
        if let Some(last_key) = cursor.last_key() {
            let token_idx = self
                .emit_tokens
                .then(|| common::cursor_token_index(cursor))
                .flatten();

            if let Some(token_idx) = token_idx
                && token_idx > 0
                && token_idx <= self.items.len()
                && self.items[token_idx - 1].key == *last_key
            {
                start_idx = start_idx.max(token_idx);

                #[cfg(debug_assertions)]
                debug_assert_eq!(
                    token_idx, key_resume,
                    "token index {token_idx} disagrees with \
                     key-derived resume position {key_resume}"
                );
            }
        }

        if start_idx >= bounds.range_end {
            return Ok(EnumerationPage::new(Vec::new(), cursor.clone()));
        }

        // Phase 5: assemble page via pooled slab path.
        let take = budgets.max_items().min(bounds.range_end - start_idx);
        let page_items = &self.items[start_idx..start_idx + take];

        // Stage item keys and refs into a page-local ByteSlab, then
        // materialize wrappers from those slots (allocation-free cloning).
        let staged = common::assemble_pooled_page(
            page_items
                .iter()
                .map(|item| (item.key.as_bytes(), item.item_ref.as_bytes())),
            self.emit_tokens,
            start_idx,
        )?;

        let common::StagedPage { wrappers, token } = staged;
        let mut out = Vec::with_capacity(wrappers.len());
        for ((item_key, item_ref), item) in wrappers.into_iter().zip(page_items) {
            out.push(
                ScanItem::new(
                    item_key,
                    item_ref,
                    item.stable_item_id,
                    VersionId::Strong(item.version_id),
                )
                .with_size_hint(item.size_hint),
            );
        }

        // Build continuation cursor from the last emitted key.
        let last_key = match out.last() {
            Some(item) => item.item_key().clone(),
            None => return Ok(EnumerationPage::new(Vec::new(), cursor.clone())),
        };
        let next_cursor = common::build_next_cursor_from_staged(
            last_key,
            start_idx,
            out.len(),
            self.emit_tokens,
            token,
        )?;

        Ok(EnumerationPage::new(out, next_cursor))
    }
```

Each phase serves a distinct purpose:

**Phase 1: Budget gate.** If the caller's deadline has already expired, the method returns `EnumerateError::retryable("budget deadline expired")` immediately. This is retryable (not permanent) because the same request with a fresh deadline would succeed. Returning an error rather than an empty page avoids ambiguity: an empty page means "no items in this range" (EOF), while a retryable error means "try again with more time."

**Phase 2: Range validation.** If both start and end bounds are provided and `start > end`, the range is inverted. This is a permanent error because no amount of retrying will fix a malformed range.

**Phase 3: Index resolution.** The `lower_bound` function from `common.rs` maps key boundaries to array indices via binary search (O(log n)). `lower_bound` returns the first index whose key is `>= bound`. For unbounded ranges (`None`), start defaults to 0 and end defaults to `items.len()`.

**Phase 4: Cursor advancement.** This is the most intricate phase. It uses a labeled block (`'resolve`) with two paths:

- **Token fast path (O(1)).** When tokens are enabled and the cursor carries a well-formed token, the token is parsed as a big-endian `u64` index. The connector validates that `items[token_idx - 1].key == last_key` -- this confirms the token has not gone stale. If validation passes, the token index is used directly. This check runs in all build profiles, not just debug.
- **Key fallback (O(log n)).** If the token is missing, malformed, or fails validation, the connector falls back to `upper_bound` binary search on `last_key`. The `upper_bound` function returns the first index whose key is strictly greater than `last_key`, ensuring the last-emitted key is never re-emitted. In debug builds, a `debug_assert_eq!` verifies that the token and key paths agree -- catching bugs during development.

**Phase 5: Page extraction.** The method takes up to `max_items` consecutive items from the precomputed `PreparedItem` array. Each item's metadata is cloned into a `ScanItem` with no hashing or computation. All items carry `VersionId::Strong` because the in-memory content is immutable. The continuation cursor carries the `last_key` and (when tokens are enabled) a big-endian token encoding the next index.

---

## Split-Point Selection

Here is the definition from `in_memory.rs`:

```rust
    /// Choose a byte-weighted split point in the optional range `[start, end)`.
    ///
    /// Delegates to [`common::estimate_split_from_sorted`] after resolving
    /// bounds and the cursor resume position. Returns `None` when fewer than
    /// two keys remain, the estimator produces no candidate, or the candidate
    /// fails validation.
    fn choose_split_point_bounds(
        &self,
        start: Option<&[u8]>,
        end: Option<&[u8]>,
        cursor: &Cursor,
    ) -> Result<Option<ItemKey>, EnumerateError> {
        let bounds = common::resolve_bounds(&self.items, start, end)?;
        let start_idx = common::key_resume_start(&self.items, cursor, bounds.range_start);
        if start_idx >= bounds.range_end {
            return Ok(None);
        }

        let range = &self.items[start_idx..bounds.range_end];
        common::estimate_split_from_sorted(
            range
                .iter()
                .map(|item| (item.key.as_bytes(), item.size_hint)),
            range.len(),
            cursor,
            end,
        )
    }
```

The split-point selection delegates range resolution to `common::resolve_bounds` (which validates the range and maps key boundaries to indices) and cursor advancement to `common::key_resume_start`. It then calls `common::estimate_split_from_sorted`, which feeds the entries into a `StreamingSplitEstimator` for byte-weighted median selection, and validates the candidate via `common::is_valid_split_candidate` (rejecting candidates that would not advance past the cursor or that fall at/beyond the upper bound).

The `StreamingSplitEstimator` (defined in `split_estimator.rs`) uses dual-axis stride-based sampling to maintain a fixed-size sample of `(key, size)` pairs, then estimates the byte-weighted median split point from that sample. Each incoming entry is tested against both a *rank stride* and a *byte stride*; the entry is recorded when either cadence fires. When the sample buffer exceeds its cap, stride-doubling compaction halves the buffer via nearest-neighbor selection on the byte axis, then doubles both strides so future observations are sampled at the coarser cadence. This approach produces balanced shards even when object sizes are highly skewed -- a common scenario where a repository contains one 100 MB binary alongside thousands of 1 KB source files. Count-based splitting would put half the items on each side, but one side would hold nearly all the bytes. Byte-weighted splitting puts half the bytes on each side.

---

## Read Methods

Here is the definition from `in_memory.rs`:

```rust
impl InMemoryDeterministicConnector {
    /// Returns a reader over the full item content.
    ///
    /// Budget enforcement is left to the runtime layer (which wraps the
    /// returned reader in a bounded adapter), consistent with the advisory
    /// budget contract. This matches [`read_range`](Self::read_range)
    /// semantics, which also does not reject items based on total size.
    pub fn open(
        &mut self,
        item_ref: &ItemRef,
        _budgets: Budgets,
    ) -> Result<Box<dyn io::Read + Send>, ReadError> {
        let bytes = self.open_ref_internal(item_ref)?.clone();
        Ok(Box::new(io::Cursor::new(bytes)))
    }

    /// Clamps the copy length to `min(dst.len(), max_bytes, available)` so
    /// the budget acts as an additional upper bound on bytes returned.
    pub fn read_range(
        &mut self,
        item_ref: &ItemRef,
        offset: u64,
        dst: &mut [u8],
        budgets: Budgets,
    ) -> Result<usize, ReadError> {
        if offset.checked_add(dst.len() as u64).is_none() {
            return Err(ReadError::permanent("offset + dst length overflow"));
        }

        let bytes = self.open_ref_internal(item_ref)?;
        let start = match usize::try_from(offset) {
            Ok(start) => start,
            Err(_) => return Ok(0),
        };
        if start >= bytes.len() {
            return Ok(0);
        }

        let max_bytes = usize::try_from(budgets.max_bytes()).unwrap_or(usize::MAX);
        let allowed = dst.len().min(max_bytes);
        if allowed == 0 {
            return Ok(0);
        }

        let to_copy = (bytes.len() - start).min(allowed);
        dst[..to_copy].copy_from_slice(&bytes[start..(start + to_copy)]);
        Ok(to_copy)
    }
}
```

The `open` method resolves the `ItemRef` to an index, clones the `Arc<[u8]>` content (a reference count bump, not a byte copy), and wraps it in an `io::Cursor`. The `Box<dyn io::Read + Send>` return type provides a type-erased reader so callers can handle content from any connector uniformly. The heap allocation is acceptable on this WARM path (called once per item, not per byte).

The `read_range` method provides random-access reads. Three safety guards protect against pathological inputs:

1. **Overflow guard.** `offset.checked_add(dst.len() as u64)` catches cases where the sum of offset and destination length would wrap a `u64`. This returns a permanent error because the caller constructed an impossible range.
2. **EOF handling.** If `offset >= bytes.len()`, the method returns `Ok(0)` -- consistent with `std::io::Read` semantics where reading past the end of content yields zero bytes.
3. **Budget clamping.** The copy length is clamped to `min(dst.len(), max_bytes, available)`, where `max_bytes` comes from the budget. This means the budget acts as an additional upper bound on bytes returned per call.

---

## Capability Flags

Here is the definition from `in_memory.rs`:

```rust
impl InMemoryDeterministicConnector {
    pub fn caps(&self) -> ConnectorCapabilities {
        ConnectorCapabilities {
            seek_by_key: true,
            token_resume: self.emit_tokens,
            range_read: true,
            split_hints: true,
        }
    }
```

All four capability flags are `true` (with `token_resume` tracking the `emit_tokens` setting). This makes the in-memory connector the most capable connector in the system -- it supports every feature the connector API defines:

- **`seek_by_key: true`** -- The connector can resume enumeration from an arbitrary key position via binary search.
- **`token_resume: self.emit_tokens`** -- When tokens are enabled, the connector emits and consumes big-endian index tokens for O(1) resume.
- **`range_read: true`** -- The connector supports byte-range reads via `read_range`.
- **`split_hints: true`** -- The connector emits split-point hints via byte-weighted median selection.

This full capability set is intentional for a test double: it exercises every code path in the orchestration layer that branches on connector capabilities. A connector that advertises only `seek_by_key` would leave the token-resume and range-read code paths untested.

---

## Why It Is Cheaply Clone-able

The `#[derive(Clone)]` on `InMemoryDeterministicConnector` produces a clone that bumps the `Arc<[PreparedItem]>` reference count (one atomic increment) and copies the `bool` field. The entire item array is shared, not copied. This matters in test harnesses that spawn multiple simulated workers, each needing their own mutable connector instance: cloning is effectively free regardless of dataset size.

The methods take `&mut self` for API uniformity with connectors that maintain internal state (connection pools, rate limiter tokens, cursor caches). The in-memory connector has no mutable state after construction, but the consistent signature keeps all connectors interchangeable at the call site.

---

## Summary

The `InMemoryDeterministicConnector` is a test double that provides the full connector API surface (enumeration, read, and split-point methods) using sorted in-memory arrays. Construction sorts items, rejects duplicates, and precomputes all metadata (BLAKE3 hashes, `ItemRef` indices, size hints) in a single O(n log n) pass. The connector takes a `connector_tag`, a `connector_instance_id`, and a `Vec<MemItem>`, deriving a `ConnectorInstanceIdHash` from the instance ID bytes for scoped identity derivation. The 5-phase enumeration algorithm in `enumerate_page_bounds` gates on budgets, validates ranges, resolves shard bounds via `lower_bound` binary search, advances past the cursor via an O(1) token fast path with O(log n) key fallback, and extracts pages from precomputed metadata with no further hashing. Split-point selection uses byte-weighted medians from `common::estimate_split_from_sorted` (backed by `StreamingSplitEstimator`) to balance shards by byte volume rather than item count. The connector is cheaply `Clone`-able via `Arc<[PreparedItem]>` sharing, advertises the full capability set, and guarantees bit-identical enumeration across runs for identical inputs.

Chapter 7 applies these same patterns to a real filesystem, where lazy indexing, symlink security, and TOCTOU gaps introduce constraints the in-memory connector avoids.
