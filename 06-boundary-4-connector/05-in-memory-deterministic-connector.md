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

At construction time, each `MemItem` is transformed into a `PreparedItem` with precomputed metadata. This is the single most important performance decision in the connector: size hints are computed once at construction, not on every split-point query.

Here is the definition from `in_memory.rs`:

```rust
/// Precomputed per-item metadata, built once at construction time.
///
/// `size_hint` is precomputed as `bytes.len()` at construction time.
/// Computing it once avoids repeated length queries during split-point
/// estimation.
struct PreparedItem {
    /// Sorted item key for ordering and cursor progression.
    key: ItemKey,
    /// Immutable payload bytes for read operations.
    bytes: Arc<[u8]>,
    /// Precomputed `bytes.len() as u64`.
    size_hint: u64,
}
```

Field-by-field breakdown:

- **`key: ItemKey`** -- Carried forward from the input `MemItem`. Used for binary search during shard-bound resolution and cursor advancement.
- **`bytes: Arc<[u8]>`** -- The shared content payload. Carried forward unchanged; the `open` method returns a reader over these bytes.
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
///
/// Together these choices make ordering and read
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
    pub fn new(mut items: Vec<MemItem>) -> Self {
        items.sort_by(|left, right| left.key.cmp(&right.key));

        // Enforce unique keys. Duplicate keys break cursor resume
        // (upper_bound skips remaining duplicates) and split-point selection
        // (empty shards). Format the
        // duplicate through ItemKey itself so panic diagnostics stay redacted.
        if let Some(pos) = items.windows(2).position(|w| w[0].key == w[1].key) {
            panic!(
                "InMemoryDeterministicConnector requires unique item keys; \
                 duplicate key {} at sorted position {pos}",
                items[pos].key
            );
        }

        let prepared: Arc<[PreparedItem]> = items
            .into_iter()
            .map(|item| {
                let size_hint = item.bytes.len() as u64;
                PreparedItem {
                    key: item.key,
                    bytes: item.bytes,
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

The constructor performs three steps in sequence:

**Step 1: Sort.** Items are sorted lexicographically by `ItemKey`. This is O(n log n) and happens once. After sorting, item positions are stable and can be used as `ItemRef` values.

**Step 2: Duplicate rejection.** The `windows(2)` scan checks adjacent pairs in the sorted array for equal keys. If a duplicate is found, the constructor panics with a diagnostic message including the duplicate key's redacted display and its sorted position. The panic is deliberate: duplicate keys are a test-setup bug, not a runtime condition. The module documentation explains the two ways duplicates break correctness: cursor resume via `upper_bound` skips remaining duplicates, and split-point selection can return a duplicate key producing empty shards.

**Step 3: Precomputation.** Each `MemItem` is transformed into a `PreparedItem` with its `size_hint` precomputed as `bytes.len()`. After this loop, split-point estimation can operate without re-measuring content sizes.

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

## How Items Are Scanned

The in-memory connector does not expose an enumeration method. Instead, the
`InMemoryScanSourceFactory` (covered in Chapter 8) constructs the connector
from an `Assignment` and drives scanning through `ScanDriver::run()`. The
driver iterates the sorted item array sequentially, calling
`commit.begin_item()` / `commit.finish_item()` per item through the
`CommitSink` protocol. The connector's `open()` and `read_range()` methods
provide content access when the driver needs to feed item bytes to the
scanner engine.

The shared helpers in `common.rs` -- `resolve_bounds`, `key_resume_start`,
`lower_bound`, `upper_bound` -- still handle shard-boundary resolution and
cursor advancement for split-point selection, which needs to navigate the
sorted item array by key range.

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

The `InMemoryDeterministicConnector` is a test double that provides the connector API surface (read and split-point methods) using sorted in-memory arrays. Construction sorts items, rejects duplicates, and precomputes size hints in a single O(n log n) pass. The connector takes a `Vec<MemItem>` and stores sorted `PreparedItem` records. Split-point selection uses byte-weighted medians from `common::estimate_split_from_sorted` (backed by `StreamingSplitEstimator`) to balance shards by byte volume rather than item count. The connector is cheaply `Clone`-able via `Arc<[PreparedItem]>` sharing, advertises the full capability set, and guarantees bit-identical ordering across runs for identical inputs. Scanning is driven by the `InMemoryScanSourceFactory` through the `ScanDriver::run()` protocol rather than page-by-page enumeration.

Chapter 6 applies these same patterns to a real filesystem, where lazy indexing, symlink security, and TOCTOU gaps introduce constraints the in-memory connector avoids.
