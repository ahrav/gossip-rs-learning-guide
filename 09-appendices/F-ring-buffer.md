# Appendix F: gossip-stdx Data Structures

This appendix covers the three custom data structures in the `gossip-stdx` crate:
`RingBuffer`, `InlineVec`, and `ByteSlab`. Each exists because the standard
library does not provide a suitable alternative with the required compile-time
guarantees.

---

## RingBuffer

### Purpose

The coordination layer maintains a per-shard idempotency log so that
retried operations return their cached result instead of executing twice.
Each `ShardRecord` carries a 16-entry FIFO op-log implemented as
`RingBuffer<OpLogEntry, 16>`. The standard library does not provide a
fixed-capacity, stack-allocated ring buffer with power-of-2 overflow
semantics, so `gossip-stdx` ships its own.

Key requirements the standard library cannot satisfy:

- **Fixed capacity known at compile time** -- backpressure must be deterministic.
- **Zero heap allocation** -- the hot path cannot tolerate allocator latency.
- **Power-of-2 index arithmetic** -- bitwise AND replaces division on every access.
- **Overwrite-oldest semantics** -- `push_back_overwrite` evicts the front element in O(1) when full.

Source: `crates/gossip-stdx/src/ring_buffer.rs`

## Design

The struct uses three fields:

```rust
pub struct RingBuffer<T, const N: usize> {
    buf: [MaybeUninit<T>; N],
    head: u32,
    len: u32,
}
```

`buf` is a stack-allocated array of `MaybeUninit<T>` -- no constructors run
at creation, and no heap allocation occurs. `head` tracks the index of the
oldest element. `len` tracks how many slots are initialized. The element at
logical index `i` lives at physical index `(head + i) & MASK`.

The power-of-2 constraint on `N` enables a single-cycle bitmask for all
index calculations:

```rust
const MASK: u32 = Self::CAPACITY - 1;

// In push_back_assume_capacity:
let tail = (self.head + self.len) & Self::MASK;

// In pop_front:
self.head = (self.head + 1) & Self::MASK;
```

This compiles to a single AND instruction instead of an expensive
division/modulo sequence on the hot path.

## API Overview

| Method | Behavior |
|---|---|
| `push_back(value)` | Append; returns `Err(value)` if full |
| `push_back_overwrite(value)` | Append; evicts and returns oldest if full |
| `push_back_assume_capacity(value)` | Append; **panics** if full (crate-internal) |
| `pop_front()` | Remove and return oldest element |
| `get(index)` | Logical index access (0 = oldest) |
| `len()` / `is_empty()` / `is_full()` | Capacity queries |
| `capacity()` | Returns `N` |
| `clear()` | Drop all elements in FIFO order |
| `iter()` | Borrowing iterator in FIFO order |
| `into_iter()` | Consuming iterator in FIFO order |

> **Note**: There are no dedicated `front()` or `back()` peek methods. The oldest
> element can be accessed via `get(0)` and the newest via `get(len() - 1)`.

## Compile-Time Validation

The capacity constraint is enforced entirely at compile time through const
evaluation. Attempting to instantiate `RingBuffer<T, 5>` (not a power of 2)
or `RingBuffer<T, 0>` produces a compile error:

```rust
const CAPACITY: u32 = {
    assert!(N > 0, "RingBuffer capacity must be > 0");
    assert!(N & (N - 1) == 0, "RingBuffer capacity must be power of 2");
    assert!(
        N <= u32::MAX as usize / 2,
        "N must fit in u32 and not risk overflow"
    );
    N as u32
};
```

The `new()` constructor forces evaluation of `CAPACITY` with `let _ = Self::CAPACITY;`,
ensuring the assertions run for every concrete instantiation.

## Unsafe Internals, Safe API

The implementation uses `MaybeUninit<T>` and raw pointer operations internally,
but the public API is entirely safe. Three invariants make this sound:

1. **Slot initialization tracking.** `head` and `len` together define the
   initialized window: physical slots `[head, head+len)` (wrapping via mask)
   contain valid `T` values. All other slots are uninitialized.

2. **Bounded indexing.** Every physical index is computed as
   `(head + offset) & MASK`, which is always less than `CAPACITY`. The mask
   guarantees that `get_unchecked` never reads out of bounds.

3. **Drop correctness.** The `Drop` impl delegates to `clear()`, which calls
   `pop_front()` in a loop. Each `pop_front` updates `head` and `len` *before*
   the element is dropped. If `T::drop()` panics during unwinding, the
   remaining bookkeeping is still consistent -- no double-drop occurs.

## Miri Testing

The crate runs its full test suite under Miri (`cargo +nightly miri test`),
which checks for undefined behavior at runtime: use-after-free, double-free,
out-of-bounds reads, and uninitialized memory access. The test suite includes
a `DropTracker` helper that uses `Arc<AtomicUsize>` to count drop calls,
verifying that every element is dropped exactly once across contiguous layouts,
wrapped layouts, overwrite cycles, and clone independence.

Property tests (proptest) are also Miri-compatible with reduced case counts:

```rust
fn miri_proptest_config() -> proptest::test_runner::Config {
    if cfg!(miri) {
        proptest::test_runner::Config {
            failure_persistence: None,
            cases: 32,
            ..Default::default()
        }
    } else {
        proptest::test_runner::Config::default()
    }
}
```

## Iterator Implementations

Two iterator types are provided:

**`Iter<'a, T, N>`** -- borrowing iterator that yields `&T` in FIFO order.
Implements `DoubleEndedIterator` for reverse scans (used by `op_log_lookup`
to find recent entries first), `ExactSizeIterator`, and `FusedIterator`.

**`IntoIter<T, N>`** -- consuming iterator that yields owned `T` values by
calling `pop_front()` internally. When dropped before exhaustion, the
remaining elements are cleaned up by the inner `RingBuffer::drop`.

Both iterators support `size_hint` with exact bounds.

## Usage in Coordination

Each `ShardRecord` embeds the ring buffer as its bounded idempotency log:

```rust
// crates/gossip-coordination/src/record.rs

pub op_log: RingBuffer<OpLogEntry, { ShardRecord::OP_LOG_CAP }>,

// ...

impl ShardRecord {
    pub const OP_LOG_CAP: usize = 16;
}
```

When a coordinator processes an operation, it first checks for a cached result
via reverse-iteration lookup:

```rust
pub fn op_log_lookup(&self, op: OpId) -> Option<&OpLogEntry> {
    self.op_log.iter().rev().find(|e| e.op_id() == op)
}
```

If the `OpId` is found, the cached `OpResult` is returned without re-executing.
If not found, the operation executes and the result is recorded:

```rust
pub fn op_log_push(&mut self, entry: OpLogEntry) {
    // ...
    self.op_log.push_back_overwrite(entry);
}
```

`push_back_overwrite` is the critical method here. When all 16 slots are
occupied, it evicts the oldest entry and inserts the new one -- both in O(1).
This is the "bounded" in "bounded idempotency": the log remembers only the
most recent 16 operations. Evicted entries are treated as new operations on
retry, which is safe because of convergent state transitions and fence-epoch
zombie fencing.

A compile-time assertion binds the constant to the expected value:

```rust
// crates/gossip-coordination/src/record.rs
const _: () = assert!(ShardRecord::OP_LOG_CAP == 16);
```

## Summary

- `RingBuffer<T, N>` is a fixed-capacity, stack-allocated FIFO with
  `MaybeUninit` storage and power-of-2 bitwise index arithmetic.
- The public API is entirely safe; `unsafe` is confined to internal slot
  access with invariants enforced by `head`/`len` bookkeeping.
- The coordination layer uses `RingBuffer<OpLogEntry, 16>` per shard for
  bounded idempotency -- `push_back_overwrite` evicts the oldest entry in
  O(1) when the buffer is full.
- Miri and property-based testing (with a `VecDeque` oracle) validate
  memory safety and FIFO correctness across all buffer layouts.

---

## ByteSlab

### Purpose

The coordination layer stores 3-5 variable-length byte fields per shard record
(key range start/end, metadata, cursor last-key, cursor token). Without pooling,
each field is a separate `Box<[u8]>` heap allocation, making per-field allocation
the dominant cost on the `checkpoint` and `acquire_and_restore_into` hot paths.

`ByteSlab` is a fixed-capacity arena allocator that pre-allocates a single
contiguous buffer and carves out variable-size regions via a two-phase allocation
strategy: free-list reuse first, bump-pointer second. This turns N heap
allocations per shard into N slab operations with no system allocator involvement.

Source: `crates/gossip-stdx/src/byte_slab.rs`

### Design

```text
┌─────────────────────────────────────────────────┐
│                  buf: Vec<u8>                   │
│ ┌───────┬──────┬───────┬────────────────────┐   │
│ │ alloc │ free │ alloc │  bump region ────►  │   │
│ └───────┴──────┴───────┴────────────────────┘   │
│         ▲                ▲                       │
│         │                │                       │
│    free_list[16]       bump                      │
└─────────────────────────────────────────────────┘
```

The slab has four key internal fields:

- **`buf: Vec<u8>`** -- the contiguous byte buffer, allocated once at
  construction.
- **`bump: u32`** -- the next byte position for bump-pointer allocation.
  Only advances forward.
- **`free_list: Vec<(u32, u32)>`** -- sorted by offset. Each entry is
  `(offset, size)`. Returned regions are coalesced with adjacent free slots.
- **`live_count: u32`** -- number of currently-live allocations. Used for leak
  detection in `Drop`.

### Two-Phase Allocation

When `allocate(data)` is called:

1. **Power-of-2 rounding.** The requested size is rounded up to the next power
   of 2 (minimum 16 bytes). This bounds internal fragmentation to 2× and
   simplifies free-list matching.

2. **Free-list probe.** The free list is searched (linear scan for best-fit) for
   the smallest slot that fits. If found, the slot is removed and reused.
   With F typically 0--5 entries, this is a trivial loop over contiguous
   memory.

3. **Bump fallback.** If no free slot fits, `bump` is advanced by the
   rounded size. If this would exceed the backing buffer's capacity, `SlabFull`
   is returned.

4. **Data copy.** The caller's byte slice is copied into the allocated region,
   and a `ByteSlot` handle is returned.

Deallocation returns the slot's region to the free list and attempts to coalesce
with adjacent free regions.

### ByteSlot Handle

`ByteSlot` is a 16-byte value type containing four `u32` fields:

```rust
#[derive(Clone, Copy, Debug)]
pub struct ByteSlot {
    offset: u32,    // Byte offset into the slab's backing buffer.
    len: u32,       // Actual data length.
    alloc_size: u32,  // Allocated capacity (power-of-2 rounded).
    owner_id: u32,  // Provenance token matching the slab's current generation.
}
```

The `owner_id` field provides provenance tracking: each `ByteSlab` maintains a
generation counter that increments on `clear()`. A slot's `owner_id` must match
the slab's current generation for `get()` to succeed. This prevents use-after-
clear bugs where a handle from a previous generation is used to read stale data.

`ByteSlot::EMPTY` is a sentinel with `offset = 0, len = 0, alloc_size = 0,
owner_id = 0`. `slab.get(EMPTY)` returns `&[]` and `slab.deallocate(EMPTY)` is
a no-op. This allows pooled types to use `EMPTY` for absent optional fields
without special-casing.

### Invariants

The slab maintains five invariants:

| ID | Invariant | Enforcement |
|----|-----------|-------------|
| SLAB-1 | **Provenance**: non-EMPTY slots have `owner_id` matching slab generation | `get()` panics on mismatch |
| SLAB-2 | **No aliasing**: no two live slots overlap in the backing buffer | Structural (allocation never overlaps) |
| SLAB-3 | **Balance**: `deallocate` decreases `live_count` by exactly 1 | Runtime assertion |
| SLAB-4 | **Conservation**: `live_count` equals total non-empty slots across all records | Harness-level assertion |
| SLAB-5 | **Leak freedom**: `live_count == 0` when coordinator drops | `ByteSlab::Drop` assertion |

### Usage in Coordination

Each `InMemoryCoordinator` owns a single `ByteSlab`:

```rust
pub struct InMemoryCoordinator {
    shards: AHashMap<TenantId, AHashMap<ShardKey, ShardRecord>>,
    // ...
    slab: ByteSlab,
}
```

Each `ShardRecord` stores its spec and cursor as pooled handles:

```rust
pub struct ShardRecord {
    // ...
    spec: PooledShardSpec,    // 3 ByteSlot handles
    cursor: PooledCursor,     // 0-2 ByteSlot handles
    // ...
}
```

Accessors borrow from the slab, not from the record:

```rust
// Returns a slice borrowed from the slab's backing buffer.
fn key_range_start<'a>(&self, slab: &'a ByteSlab) -> &'a [u8] {
    slab.get(self.key_range_start)
}
```

The `to_spec()` and `to_cursor()` methods materialize owned `ShardSpec` and
`Cursor` values when crossing API boundaries (e.g., `ShardRecord::snapshot()`),
but internal operations use the borrowed accessors exclusively to avoid heap
allocation.

### Cleanup Methods

Pooled types provide two cleanup methods:

- **`deallocate(self)`** -- consumes the wrapper. Use when rolling back a failed
  record creation (e.g., `PooledCursor::from_cursor` fails after spec was
  allocated).
- **`release_fields(&mut self)`** -- resets fields to EMPTY in-place. Use when
  the wrapper will be reused (e.g., `PooledCursor::update` releases old fields
  before writing new ones). Safe to call multiple times.

### SlabFull Semantics

`SlabFull` is returned when the slab's bump region is exhausted and no free-list
slot can satisfy the request. It propagates through `ShardRecord::new_active()`,
`ShardRecord::new_split_child()`, and cursor update paths. The coordinator
treats it as a capacity planning failure: the `slab_capacity` in
`CoordinatorRuntimeConfig` needs to be increased.

### Summary

- `ByteSlab` is a fixed-capacity arena allocator with bump-pointer + free-list
  allocation and power-of-2 rounding.
- `ByteSlot` handles provide cheap, `Copy`-able references into the slab's
  backing buffer with provenance tracking.
- The coordination layer uses `ByteSlab` to pool all variable-length byte fields
  in `ShardRecord`, eliminating per-field heap allocation on hot paths.
- Five invariants (SLAB-1 through SLAB-5) ensure memory safety, leak freedom,
  and aliasing prevention.

---

## InlineVec

### Purpose

The coordination layer needs a small, growable collection for tracking spawned
shard IDs (child shards created during split operations). Most shards produce
2-4 children; long split chains are rare. `Vec<T>` always heap-allocates, even
for 1-2 elements. `ArrayVec` is fixed-capacity and cannot handle rare overflow.
`SmallVec` requires a third-party dependency with different safety trade-offs.

`InlineVec<T, N>` stores up to `N` elements inline on the stack using
`MaybeUninit`; only when capacity is exceeded does it spill to heap allocation.
This gives the common case (≤ N elements) zero heap allocation while handling
the rare overflow case gracefully.

Source: `crates/gossip-stdx/src/inline_vec.rs`

### Design

```rust
pub struct InlineVec<T, const N: usize> {
    repr: Repr<T, N>,
}

enum Repr<T, const N: usize> {
    Inline { buf: [MaybeUninit<T>; N], len: u32 },
    Heap(Vec<T>),
}
```

When the collection holds `N` or fewer elements, `repr` is `Repr::Inline` and
elements live in the stack-allocated `buf` array. When the `(N+1)`-th element
is pushed, all inline elements are moved into a heap-allocated `Vec<T>` and
`repr` transitions to `Repr::Heap`, which delegates all subsequent operations
to the inner `Vec`.

### Key Properties

- **Stack-first**: Common case (≤ N elements) is allocation-free.
- **Spill-to-heap**: Rare case (> N elements) gracefully degrades to `Vec<T>`.
- **`MaybeUninit` storage**: No `Default` bound on `T`; uninitialized slots
  carry no overhead.
- **Kani proofs**: 9 Kani model-checking proofs verify correctness of unsafe
  operations (push, pop, index, drop, spill).

### Usage in Coordination

`SpawnedList` is a type alias for `InlineVec<ShardId, MAX_SPAWNED_PER_SHARD>`
where `MAX_SPAWNED_PER_SHARD = 1024`, used in test helpers to track child
shards created during split operations:

```rust
// crates/gossip-coordination/src/record.rs
#[cfg(test)]
pub type SpawnedList = InlineVec<ShardId, MAX_SPAWNED_PER_SHARD>;
```

Runtime shard records use `PooledSpawned` (slab-backed storage) instead.
The `SpawnedList` alias exists only for test ergonomics. Most splits produce
2 children, so the 1024-slot capacity is never the binding constraint.

### Summary

- `InlineVec<T, N>` is a stack-first small vector with heap spill for overflow.
- `MaybeUninit` storage avoids `Default` bounds and initialization overhead.
- 9 Kani proofs verify memory safety of unsafe operations.
- The coordination layer uses `InlineVec<ShardId, 1024>` (`SpawnedList`,
  `#[cfg(test)]` only) for allocation-free shard tracking in tests; runtime
  records use `PooledSpawned` (slab-backed storage).
