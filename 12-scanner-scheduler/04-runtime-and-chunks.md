# "The Split Secret" -- Runtime and Chunks

*A 2.1 MiB JavaScript bundle contains an AWS access key starting at byte offset 262,127 and ending at byte offset 262,167 -- a 40-byte secret that spans the boundary between two 256 KiB payload chunks. Chunk 0 covers bytes 0..262,144. Chunk 1 covers bytes 262,144..524,288. Without overlap, the detection engine scans chunk 0 and sees only `AKIA3E7XPZRW` (the first 17 bytes), which is not a complete key. It scans chunk 1 and sees `RLNZMFUG4OXRZ7YCVWNQ` starting at offset 0, which looks like random base64, not a key prefix. The secret passes through the scanner undetected. With an engine contract declaring `required_overlap_bytes = Some(256)`, chunk 1 carries a 256-byte prefix from chunk 0's tail. Now chunk 1 contains the complete key at a relative offset that maps back to the original absolute position. The overlap deduplication predicate -- `root_hint_end > new_bytes_start` -- ensures the finding is attributed to exactly one chunk. This is how the chunking system prevents both false negatives and duplicate findings.*

---

Between the executor (Chapter 3) and the I/O backends (Chapter 5) sits the runtime layer: a columnar file table for metadata, a thread-safe buffer pool that eliminates allocation churn, and the chunking iterator that produces overlap-preserving chunk metadata. This chapter traces data from discovery through chunk iteration to the deduplication predicate.

## 1. FileTable -- Columnar SoA Layout

The `FileTable` in `runtime.rs` stores per-file metadata in a structure-of-arrays layout. On Unix, paths live in a fixed-capacity byte arena to avoid per-file heap allocation:

```rust
/// Columnar file metadata store used by the pipeline.
///
/// Uses parallel vectors (SoA) to keep memory usage simple and allow stable
/// indexing via [`FileId`]. On Unix, paths are stored in a fixed-capacity
/// byte arena referenced by [`PathSpan`] to avoid per-file heap allocation
/// after startup.
pub struct FileTable {
    #[cfg(unix)]
    path_spans: Vec<PathSpan>,
    #[cfg(unix)]
    path_bytes: ScratchVec<u8>,
    #[cfg(not(unix))]
    paths: Vec<PathBuf>,
    sizes: Vec<u64>,
    dev_inodes: Vec<(u64, u64)>,
    flags: Vec<u32>,
}
```

The SoA layout means scanning all file sizes (for example, to compute total bytes) touches a contiguous `Vec<u64>` -- no pointer chasing through heterogeneous structs. The `FileId` is a stable index:

```rust
pub fn push(&mut self, path: PathBuf, size: u64, dev_inode: (u64, u64), flags: u32) -> FileId {
    #[cfg(unix)]
    {
        use std::os::unix::ffi::OsStrExt;
        let bytes = path.as_os_str().as_bytes();
        self.push_path_bytes(bytes, size, dev_inode, flags)
    }
    #[cfg(not(unix))]
    {
        assert!(
            self.sizes.len() < self.sizes.capacity(),
            "file table capacity exceeded"
        );
        assert!(self.sizes.len() < u32::MAX as usize);
        let id = FileId(self.sizes.len() as u32);
        self.paths.push(path);
        self.sizes.push(size);
        self.dev_inodes.push(dev_inode);
        self.flags.push(flags);
        id
    }
}
```

The arena approach has a deliberate trade-off: overflow is a hard error (`panic`) rather than a silent reallocation. This keeps allocations predictable at the cost of requiring the caller to size the arena correctly at startup.

## 2. TsBufferPool -- Thread-Safe Buffer Recycling

The `TsBufferPool` provides fixed-capacity, allocation-free buffer management with per-worker local queues. From `ts_buffer_pool.rs`:

```rust
/// Configuration for the thread-safe buffer pool.
#[derive(Clone, Copy, Debug)]
pub struct TsBufferPoolConfig {
    /// Size of each buffer in bytes.
    pub buffer_len: usize,
    /// Total number of buffers in the pool.
    pub total_buffers: usize,
    /// Number of workers (for per-worker local queues).
    pub workers: usize,
    /// Capacity of each per-worker local queue.
    pub local_queue_cap: usize,
}
```

The routing strategy uses a three-level hierarchy:

```text
acquire():
  1. Check per-worker local queue (TLS lookup via worker_id)
  2. Fall back to global queue
  3. Steal from other workers' local queues (prevents starvation)

release():
  1. Try per-worker local queue (bounded, may fail if full)
  2. Fall back to global queue (always succeeds within capacity)
```

From the implementation:

```rust
#[inline]
pub fn try_acquire(&self) -> Option<TsBufferHandle> {
    // Fast path: per-worker local queue
    if let Some(wid) = worker_id::current_worker_id() {
        debug_assert!(
            wid < self.inner.locals.len(),
            "worker_id {} out of range for pool with {} workers",
            wid,
            self.inner.locals.len()
        );
        if wid < self.inner.locals.len() {
            if let Some(buf) = self.inner.locals[wid].pop() {
                return Some(TsBufferHandle {
                    pool: self.clone(),
                    buf: Some(buf),
                });
            }
        }
    }

    // Fallback: global queue
    if let Some(buf) = self.inner.global.pop() {
        return Some(TsBufferHandle {
            pool: self.clone(),
            buf: Some(buf),
        });
    }

    // Steal from other workers' local queues.
    for local_queue in &self.inner.locals {
        if let Some(buf) = local_queue.pop() {
            return Some(TsBufferHandle {
                pool: self.clone(),
                buf: Some(buf),
            });
        }
    }

    None
}
```

### 2.1 Pre-Seeding to Reduce Cold-Start Contention

All buffers are allocated at construction time. The pool pre-distributes buffers to local queues to avoid cold-start contention on the global queue:

```rust
// Pre-allocate all buffers into global queue first
for _ in 0..cfg.total_buffers {
    let buf = vec![0u8; cfg.buffer_len].into_boxed_slice();
    global
        .push(buf)
        .expect("global queue capacity mismatch (internal error)");
}

// Pre-seed local queues to reduce cold-start contention.
for local in locals.iter().take(cfg.workers) {
    for _ in 0..cfg.local_queue_cap {
        if let Some(buf) = global.pop() {
            if local.push(buf).is_err() {
                unreachable!("local queue full during pre-seeding");
            }
        } else {
            break;
        }
    }
}
```

### 2.2 False Sharing Prevention

Per-worker local queues are wrapped in `CachePadded` to prevent cache-line bouncing:

```rust
/// Per-worker local queues indexed by worker_id.
///
/// Wrapped in `CachePadded` to prevent false sharing between workers.
/// Without this, adjacent workers updating their queue indices would
/// cause cache-line bouncing and destroy scalability.
locals: Vec<CachePadded<ArrayQueue<Box<[u8]>>>>,
```

### 2.3 RAII Buffer Handle

The `TsBufferHandle` guarantees leak-free buffer return:

```rust
/// RAII handle to a pooled buffer.
///
/// The buffer is automatically returned to the pool when this handle is dropped.
pub struct TsBufferHandle {
    pool: TsBufferPool,
    buf: Option<Box<[u8]>>,
}

impl Drop for TsBufferHandle {
    fn drop(&mut self) {
        if let Some(buf) = self.buf.take() {
            self.pool.release_box(buf);
        }
    }
}
```

The `Option` wrapper supports `take()` in the `Drop` implementation -- the buffer is moved out of the handle and returned to the pool. The `release_box` path mirrors the acquire path: try the per-worker local queue first, fall back to global.

## 3. ChunkParams and the Engine Contract

Chunk parameters derive from the engine's overlap requirement. From `chunking.rs`:

```rust
/// Chunking parameters that match the scanner's reader semantics.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ChunkParams {
    /// New bytes to read per chunk (excluding overlap).
    pub payload_bytes: u32,
    /// Overlap bytes to prepend from previous chunk's tail.
    pub overlap_bytes: u32,
}
```

The `params_from_contract` function bridges the engine contract to chunking:

```rust
pub fn params_from_contract(
    payload_bytes: u32,
    contract: EngineContract,
) -> Result<ChunkParams, &'static str> {
    if payload_bytes == 0 {
        return Err("payload_bytes must be > 0");
    }
    match contract.required_overlap_bytes {
        Some(overlap_bytes) => Ok(ChunkParams {
            payload_bytes,
            overlap_bytes,
        }),
        None => Err("engine contract has unbounded overlap requirement; \
             overlap-only chunking is not safe"),
    }
}
```

If the engine declares unbounded overlap (`None`), the function returns an error -- the scheduler refuses to chunk with overlap-only semantics, forcing the caller to use streaming-state scanning.

## 4. ChunkMeta -- Metadata That Travels with Bytes

Every chunk carries metadata that enables deduplication. From `chunking.rs`:

```rust
/// Metadata that must travel with chunk bytes into the scan stage.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct ChunkMeta {
    /// Object this chunk belongs to.
    pub object_id: ObjectId,
    /// Absolute offset in the object for `chunk.data()[0]`.
    pub base_offset: u64,
    /// Number of bytes in `chunk.data()` (including overlap prefix).
    pub len: u32,
    /// Number of overlap bytes at the front of `chunk.data()`.
    pub prefix_len: u32,
    /// Start offset into the backing buffer.
    pub buf_offset: u32,
    /// View identity (raw, base64-decoded, etc).
    pub view: ViewId,
}
```

The critical deduplication predicate:

```rust
/// Keep predicate using a relative end offset inside the chunk buffer.
/// **Use this in hot paths - it avoids u64 math.**
///
/// Equivalent to `keep_finding_abs_end` because `base_offset` cancels out:
/// - `rel_end` is offset within `chunk.data()`
/// - Keep iff `rel_end > prefix_len`
#[inline]
pub fn keep_finding_rel_end(&self, rel_end: u32) -> bool {
    rel_end > self.prefix_len
}
```

This single comparison -- `rel_end > prefix_len` -- determines whether a finding belongs to this chunk or should be dropped as a duplicate from the overlap region. The relative version avoids u64 arithmetic in the findings loop, where it executes for every candidate finding.

## 5. ChunkIter -- Overlap-Preserving Iterator

The `ChunkIter` produces `ChunkMeta` records for an object of known length:

```rust
/// Iterator over chunk metadata for an object of known length.
pub struct ChunkIter {
    object_id: ObjectId,
    obj_len: u64,
    payload_bytes: u32,
    overlap_bytes: u32,
    /// Bytes consumed from the object (excluding overlap).
    offset: u64,
    /// Next chunk's prefix_len (from previous chunk's tail).
    tail_len: u32,
    view: ViewId,
}
```

The `next()` implementation:

```rust
impl Iterator for ChunkIter {
    type Item = ChunkMeta;

    #[inline]
    fn next(&mut self) -> Option<Self::Item> {
        let offset = self.offset;
        if offset >= self.obj_len {
            return None;
        }

        let remaining = self.obj_len - offset;
        let read = remaining.min(self.payload_bytes as u64) as u32;

        let prefix_len = self.tail_len;
        let len = prefix_len + read;
        let base_offset = offset - prefix_len as u64;

        // Branchless: when overlap_bytes == 0, min(0, len) == 0
        let next_tail_len = self.overlap_bytes.min(len);

        // Advance state
        self.offset = offset + read as u64;
        self.tail_len = next_tail_len;

        Some(ChunkMeta {
            object_id: self.object_id,
            base_offset,
            len,
            prefix_len,
            buf_offset: 0,
            view: self.view,
        })
    }
}
```

For a 250-byte object with payload=100 and overlap=20:

| Chunk | offset | read | prefix_len | len | base_offset |
|-------|--------|------|------------|-----|-------------|
| 0 | 0 | 100 | 0 | 100 | 0 |
| 1 | 100 | 100 | 20 | 120 | 80 |
| 2 | 200 | 50 | 20 | 70 | 180 |

Chunk 1 includes bytes 80..200 (the last 20 bytes of chunk 0 as overlap, plus 100 new bytes). Any finding that ends at byte 100 or earlier in chunk 1's buffer falls in the overlap region and is dropped by `keep_finding_rel_end`.

## 6. Proof: No False Negatives with Bounded Span

The chunking tests include a proof-by-construction that overlap-based deduplication has no false negatives. From `chunking.rs`:

```rust
#[test]
fn overlap_dedupe_no_false_negatives_for_bounded_span_model() {
    let data = b"aaaaaSECRETbbbbbbbbbbbbSECRETcccccccccccccccc";
    let needle = b"SECRET";

    // Bounded-span model: required overlap is len-1.
    let contract = EngineContract::bounded((needle.len() - 1) as u32);
    let params = params_from_contract(16, contract).unwrap();
    let view = ViewId(0);

    let expected = find_all(data, needle);

    let object_id = make_object_id();

    let mut got = Vec::new();
    for meta in ChunkIter::new(object_id, data.len() as u64, params, view) {
        let start = meta.base_offset as usize;
        let end = start + meta.len as usize;
        let chunk = &data[start..end];

        for (rs, re) in find_all(chunk, needle) {
            let root_hint_end = meta.base_offset + re;
            if meta.keep_finding_abs_end(root_hint_end) {
                got.push((meta.base_offset + rs, meta.base_offset + re));
            }
        }
    }

    got.sort();
    got.dedup();
    assert_eq!(got, expected, "should find all secrets without duplicates");
}
```

And the negative proof -- insufficient overlap misses boundary-spanning matches:

```rust
#[test]
fn insufficient_overlap_can_miss_boundary_match() {
    let data: Vec<u8> = b"xxxxSECR"
        .iter()
        .chain(b"ETyyyyyy".iter())
        .copied()
        .collect();
    let needle = b"SECRET";

    let expected = find_all(&data, needle);
    assert_eq!(expected.len(), 1, "sanity: whole-buffer scan finds SECRET");

    // Zero overlap - will miss the boundary match
    let params = ChunkParams::new(8, 0);
    // ...
    assert!(got.is_empty(), "should miss the boundary-spanning match");
}
```

## 7. The Overlap Carry Pattern

The local filesystem scanner (covered in detail in [Chapter 5](05-filesystem-scanning.md)) uses an I/O pattern called "overlap carry" instead of seeking back for each chunk's overlap. From `local_fs_owner.rs`:

```text
Iteration 1:                    Iteration 2:
┌─────────────────────────┐     ┌─────────────────────────┐
│      payload bytes      │     │overlap│  new payload    │
│      (from read)        │     │(copy) │  (from read)    │
└─────────────────────────┘     └─────────────────────────┘
                          │            ▲
                          └────────────┘
                        copy_within(tail → head)
```

One buffer per file, sequential reads, `copy_within` to carry overlap bytes forward. This eliminates seeks, re-reading overlap from the kernel, and per-chunk buffer pool churn.

## 8. Metrics and Observability

Per-worker metrics are plain integers with no cross-thread contention. From `metrics.rs`:

```rust
/// Log2 histogram for cheap p95/p99-ish tracking.
///
/// Buckets represent power-of-2 ranges:
/// - Bucket 0: [0, 2)
/// - Bucket 1: [2, 4)
/// - Bucket k: [2^k, 2^(k+1))
///
/// Maximum trackable value is 2^63.
#[derive(Clone, Debug)]
pub struct Log2Hist {
    /// Count per bucket.
    pub buckets: [u64; 64],
    /// Total count of recorded values.
    pub count: u64,
    /// Sum of all recorded values (for mean calculation).
    pub sum: u64,
}
```

Recording is O(1) with a single `leading_zeros` call to determine the bucket. Aggregation happens only on `join()`, after all workers have completed. This design means metrics collection adds zero contention to the scanning hot path.

## What's Next

[Chapter 5](05-filesystem-scanning.md) traces the complete lifecycle of a filesystem scan: the `LocalFsOwner` scheduler, gitignore-aware directory walking via the `ignore` crate, chunk reading with the overlap carry pattern, and the io_uring backend for Linux.
