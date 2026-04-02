# "Every Component Verified" -- Filesystem Connector

*A secret scanner is deployed at `/data/repos` to scan 14,000 git checkouts for leaked credentials. Repository `acme-infra` contains a symlink at `config/database.yml` pointing to `/etc/shadow`. The indexing walk follows the symlink, classifying `/etc/shadow` as a scannable file with key `acme-infra/config/database.yml`. The connector's `open()` call reads `/etc/shadow` through the symlink, handing 1,847 bytes of password hashes to the detection engine. The scanner -- designed to find secrets -- has just exfiltrated the system's password database through its own read path. Every finding from that file is a false positive that contains real credential material. If those findings are shipped to the customer's dashboard, the scanner has become the breach.*

---

## Why the Filesystem Connector Exists

The in-memory connector from Chapter 4 proves that the enumeration contract works for items held entirely in process memory. Production scanners need a connector that reads files from disk. This introduces three problems the in-memory connector does not face: symlinks can escape the scan root, filesystem state can change between enumeration and reading (TOCTOU), and the directory walk itself can be expensive enough to blow deadlines.

The `FilesystemConnector` in `crates/gossip-connectors/src/filesystem.rs` (~1,233 lines) solves all three. It provides read and split-point methods over a rooted directory tree, applying component-by-component `openat` with `O_NOFOLLOW` at every step for symlink-resistant read confinement. The connector integrates a `StreamingSplitEstimator` for byte-weighted split hints, though the capability is advertised as `split_hints: false` because the enumeration path does not feed observations into the estimator. The split-point API is available for external callers via `choose_split_point`.

> **Platform availability:** `FilesystemConnector` is only available on Unix
> (`#[cfg(unix)]`). The module relies on Unix-specific APIs: `openat` with
> `O_NOFOLLOW` and `O_DIRECTORY` flags, raw file descriptor manipulation via
> `OwnedFd` and `libc`, and `std::os::unix::fs::MetadataExt` for inode and
> device fields. Non-Unix platforms are not supported.

### Pooled Page Emission

Like the in-memory connector, `FilesystemConnector` uses the pooled toxic-byte page assembly path for enumeration. Filesystem `ItemRef` bytes are identical to `ItemKey` bytes (both are the `/`-separated relative path), so each item's bytes are staged once into a page-local `ByteSlab` and both wrappers are materialized from that same slot via `assemble_pooled_page_shared_key_ref` in `common.rs`. This shared-slot optimization cuts staging copies versus the generic two-slot path while keeping wrapper cloning allocation-free in the HOT emit loop.

---

## Read-Path Confinement

The filesystem connector's primary security mechanism is component-by-component
`openat` traversal from a canonical root directory fd, using `O_NOFOLLOW` at
each step. This prevents symlink escapes and intermediate directory
substitution attacks.

Files are opened with `O_NONBLOCK`, validated as regular files via metadata,
then `O_NONBLOCK` is cleared before reads. This two-step approach avoids
blocking on FIFOs or device nodes that might slip past the regular-file check
in a TOCTOU race.

The connector also integrates a `StreamingSplitEstimator` for split-point
hints. A fresh connector starts with an empty estimator and returns `None`
until sufficient observations accumulate. Connector-level key-range bounds
participate in split-point selection: split candidates outside the effective
`[start, end)` range are rejected.

---

## The FilesystemConnector Struct

Here is the definition from `filesystem.rs`:

```rust
pub struct FilesystemConnector {
    root: PathBuf,
    walk_key_range_start: Option<Box<[u8]>>,
    walk_key_range_end: Option<Box<[u8]>>,
    root_mode: Option<RootMode>,
    connector_instance: Option<ConnectorInstanceIdHash>,
    /// Directory fd for canonical directory roots, opened lazily.
    root_fd: Option<OwnedFd>,
    /// Single-entry FD cache for sequential `read_range` calls.
    cached_file: Option<CachedFile>,
    /// Streaming split-point estimator fed by external observation and reset
    /// whenever connector state is rebuilt.
    split_estimator: StreamingSplitEstimator,
}
```

Eight fields, each with a clear role:

- **`root: PathBuf`** -- The directory root passed at construction. On the first operation that needs the filesystem, this is canonicalized via `fs::canonicalize` to resolve relative paths and root-level symlinks. Canonicalization prevents cwd drift: if the process working directory changes after construction, the connector still opens files from the correct absolute path.

- **`walk_key_range_start` / `walk_key_range_end: Option<Box<[u8]>>`** -- Connector-level key-range bounds, set via `with_key_range()`. These are intersected with per-request shard bounds before split-point selection.

- **`root_mode: Option<RootMode>`** -- Tracks whether the root is a directory or a single file. Lazily determined on first use. When the root is a single file, enumeration produces exactly one item; when it is a directory, the connector walks the subtree.

- **`connector_instance: Option<ConnectorInstanceIdHash>`** -- Precomputed hash for `StableItemId` derivation. Combines the connector tag and the canonical root path into a single identity component. Lazily computed alongside root canonicalization so that items from distinct connector roots produce disjoint identity hashes.

- **`root_fd: Option<OwnedFd>`** -- A directory file descriptor for the canonicalized root, opened lazily by `ensure_root_fd`. All read-path operations use `openat` relative to this fd, confining file access to the subtree under the root.

- **`cached_file: Option<CachedFile>`** -- A single-entry FD cache for sequential `read_range` calls:

```rust
struct CachedFile {
    item_ref: Box<[u8]>,
    file: fs::File,
}
```

The cache is keyed on `item_ref` bytes (the encoded relative path). When a `read_range` targets a different file, the old fd is dropped and a new one is opened. Cache size is intentionally one entry to match dominant sequential range reads without retaining unbounded descriptors.

- **`split_estimator: StreamingSplitEstimator`** -- A bounded-memory streaming estimator that tracks file keys and byte weights as they are fed externally. The estimator uses byte-weighted sampling to produce split-point hints without requiring a full in-memory index.

### Construction and Builder Methods

Construction is infallible and performs no I/O:

```rust
pub fn new(root: impl Into<PathBuf>) -> Self {
    // Asserts the root path is non-empty, then initializes with
    // None for optional fields and a fresh StreamingSplitEstimator.
}
```

The builder method `with_key_range(start, end)` restricts split-point selection to keys inside the half-open range `[start, end)`. `None` on either side means unbounded. The range is intersected with per-request shard bounds.

---

## Lazy Initialization: `ensure_root_fd`

Construction of a `FilesystemConnector` is cheap: no I/O occurs. The first
operation that needs the filesystem triggers lazy initialization.

The `ensure_root_fd` method canonicalizes the root path and opens a
directory fd. It does *not* latch failures -- the connector intentionally
retries on every call when the root is unavailable because root-directory
conditions can change between calls (e.g., a mount appearing, permissions
being adjusted). The per-attempt cost is low (3-4 syscalls).

```rust
fn ensure_root_fd(&mut self) -> Result<(), EnumerateError> {
    if self.root_fd.is_some() {
        return Ok(());
    }

    self.root = fs::canonicalize(&self.root)
        .map_err(|err| classify_io_enumerate_error("canonicalize", &self.root, &err))?;

    let path_meta = fs::metadata(&self.root)
        .map_err(|err| classify_io_enumerate_error("stat_root", &self.root, &err))?;
    let expected_id = (path_meta.dev(), path_meta.ino());

    let root_fd = open_dir_fd(&self.root)
        .map_err(|err| classify_io_enumerate_error("open_root_dir", &self.root, &err))?;

    verify_root_identity(&root_fd, expected_id)
        .map_err(|err| classify_io_enumerate_error("verify_root_identity", &self.root, &err))?;

    self.root_fd = Some(root_fd);
    Ok(())
}
```

Root identity is verified by capturing the canonical path's `(dev, ino)`
*before* opening, then comparing it to the opened fd's `fstat` result.
This narrows the TOCTOU window: the race is stat-then-open (same direction)
rather than open-then-stat where the fd could refer to a different directory
than the one we stat'd. If the identity mismatches, the error is classified
as retryable (a directory swap is a transient environmental condition, not
a permanent attribute of the path).

---

## Capabilities and Operations

The `caps` method reports:

```rust
pub fn caps(&self) -> ConnectorCapabilities {
    ConnectorCapabilities {
        seek_by_key: true,
        token_resume: false,
        range_read: true,
        split_hints: false,
    }
}
```

The connector supports key-based seek and byte-range reads. Split hints
are derived from the integrated `StreamingSplitEstimator`, which is fed
externally. A fresh connector starts with an empty estimator and returns
`Ok(None)` until sufficient observations accumulate. The returned split
key is guarded by `is_valid_split_candidate` to prevent degenerate splits
that fall before the cursor or outside the effective shard range.

### Split-Point Selection

The `choose_split_point` method resolves shard bounds, intersects them
with the connector-level key-range bounds, and delegates to the
`StreamingSplitEstimator`:

```rust
pub fn choose_split_point(
    &mut self,
    shard: &ShardSpec,
    cursor: &Cursor,
    budgets: Budgets,
) -> Result<Option<ItemKey>, EnumerateError> {
    let start = borrowed_shard_bound(shard.key_range_start(), "start")?;
    let end = borrowed_shard_bound(shard.key_range_end(), "end")?;
    self.choose_split_point_bounds(start, end, cursor, budgets.deadline())
}
```

The internal `choose_split_point_bounds` method intersects the request
bounds with the connector's configured key range, checks the deadline,
queries the estimator, and validates the candidate against cursor position
and effective shard bounds.

### Key-Range Intersection

The `intersect_key_bounds` function computes the tighter of each pair of
request and connector-level bounds. It returns `None` when the intersection
collapses to an empty interval (effective start >= effective end), signaling
that no split is possible.

---

## Streaming Split Estimation: `StreamingSplitEstimator`

The batch predecessor (`choose_split_index` in `common.rs`) required all items in memory to compute a byte-weighted split key via a prefix-sum array. The streaming walk model eliminated that in-memory index, which left a gap: no data structure from which to derive split points. The `StreamingSplitEstimator` in `split_estimator.rs` fills this gap by maintaining a bounded-memory sketch of the key stream as it flows through pagination.

### Why Byte-Weighted Over Count-Balanced

Storage systems with skewed file sizes benefit from byte-balanced shards rather than item-balanced ones. A shard containing 10,000 small config files and one 4 GB binary is not "half done" after enumerating 5,001 files. Byte-weighted splitting divides the shard so that each half processes roughly equal total bytes, balancing downstream read and detection work.

### The Algorithm

The estimator implements a three-phase pipeline: dual-axis sampling, stride-doubling compaction, and midpoint estimation.

**Phase 1: Dual-axis sampling.** Each incoming `(key, file_size)` pair is tested against two independent cadences -- a *rank stride* (every N-th item by ordinal position) and a *byte stride* (every M bytes of observed byte weight). The key is recorded if either cadence fires. A byte-triggered sample records the exact byte mark it straddles, giving tighter byte-space fidelity than rank alone. At most one sample is recorded per `observe` call, an intentional O(1) amortised-cost trade-off.

**Phase 2: Stride-doubling compaction.** When the sample buffer exceeds its cap, it is compacted to half capacity via nearest-neighbor selection on the recorded-byte-position axis (falling back to rank when all byte positions are identical). Both strides are then doubled so future observations are sampled at the coarser cadence. Because the buffer halves and the strides double together, the total number of compaction events is logarithmic in stream length, and non-sampled steady-state observations require no heap work.

Compaction uses a `SampleAxis` enum (`split_estimator.rs:103-113`) to select between `Rank` and `Bytes` axes. The `compaction_axis` function checks whether the first and last samples have distinct recorded byte positions; if so, it selects `SampleAxis::Bytes` for nearest-neighbor spacing, otherwise it falls back to `SampleAxis::Rank`. The same axis enum is used by `nearest_sample` during split estimation and by `selected_sample_indices` during compaction.

After nearest-neighbor selection, a **plateau redistribution** pass (`redistribute_plateau_picks` at `split_estimator.rs:354-430`) detects runs of retained picks that share the same axis value. When many samples sit on a "plateau" -- for example, a large run of zero-size files following a single heavy file all share the same recorded byte position -- the forward-progress constraint in nearest-neighbor selection clusters picks at the plateau's leading edge. Plateau redistribution spreads those picks evenly across the plateau's rank extent, preserving rank-axis diversity. Without this step, the rank-fallback split would drift far from the true ordinal midpoint on streams with large zero-byte tails.

**Phase 3: Midpoint estimation.** `estimate_split_key` binary-searches the retained samples for the key whose recorded byte position is closest to `total_bytes / 2`. If all items are zero-size, it falls back to the rank midpoint (`count / 2`). Two boundary guards prevent degenerate splits: the first-key guard catches front-loaded weight where the byte-weighted median lands on item 0 (producing a zero-item left shard), and a symmetric last-key guard catches back-loaded weight where the candidate lands on the last key (producing an empty right shard). Both guards fall back to the rank-based midpoint instead.

### Core Type

Here is the definition from `split_estimator.rs:539-561`:

```rust
pub(crate) struct StreamingSplitEstimator {
    sample_cap: usize,
    /// Sparse checkpoints retained from the stream. Each sample carries both
    /// ordinal rank and the byte-space search position associated with the
    /// same observed key.
    samples: Vec<Sample>,
    /// Rank sampling stride in observed items.
    rank_stride: u64,
    /// Byte sampling stride in byte-position space.
    byte_stride: u64,
    /// Next rank at which a sample should be recorded.
    next_rank_sample: u64,
    /// Next byte-position mark whose straddling file should be sampled.
    next_byte_mark: u64,
    /// Total observed byte weight (saturating on overflow).
    total_bytes: u64,
    /// Total observed item count.
    count: u64,
    /// The very first key observed in stream order. Tracked explicitly so the
    /// "avoid degenerate first-item split" guard remains correct even if
    /// downsampling evicts the original first sample.
    first_observed_key: Option<Box<[u8]>>,
}
```

Each retained `Sample` (`split_estimator.rs:123-133`) carries the key together with its position on both axes:

```rust
struct Sample {
    /// 0-based ordinal index of this key in stream order.
    rank: u64,
    /// Recorded byte position at which this sample was stored.
    recorded_byte_position: u64,
    key: Box<[u8]>,
}
```

Carrying both axes lets the estimator use byte-space interpolation for accuracy while falling back to rank when byte positions collapse (e.g., an all-zero-size stream).

### Key Methods

**`observe`** (`split_estimator.rs:648-718`) records one `(key, file_size)` pair from the streaming walk:

```rust
pub(crate) fn observe(&mut self, key: &[u8], file_size: u64) {
    debug_assert!(
        self.samples
            .last()
            .is_none_or(|last| key >= last.key.as_ref()),
        "observe() requires keys in non-decreasing order; received key that precedes the last sampled key"
    );

    if self.first_observed_key.is_none() {
        self.first_observed_key = Some(Box::<[u8]>::from(key));
    }

    let rank = self.count;
    let recorded_byte_position = self.total_bytes;
    let end_bytes = self.total_bytes.saturating_add(file_size);

    // Dual-axis trigger: record if the rank cadence fires OR if this
    // file's byte interval [recorded_byte_position, end_bytes) straddles the
    // next byte mark.
    let sample_rank = rank >= self.next_rank_sample;
    let sample_bytes = file_size > 0
        && recorded_byte_position <= self.next_byte_mark
        && self.next_byte_mark < end_bytes;

    if sample_rank || sample_bytes {
        // Prefer the byte-stride mark as the recorded position when the
        // byte trigger fired; this gives tighter byte-space fidelity.
        let byte_position = if sample_bytes {
            self.next_byte_mark
        } else {
            recorded_byte_position
        };
        self.samples.push(Sample::new(rank, byte_position, key));
    }

    self.count = self.count.saturating_add(1);
    self.total_bytes = end_bytes;

    if self.samples.len() > self.sample_cap {
        self.grow_strides_and_compact();
        return;
    }

    if sample_rank {
        self.next_rank_sample = rank.saturating_add(self.rank_stride);
    }
    if sample_bytes {
        let next_byte_mark = self.next_byte_mark.saturating_add(self.byte_stride);
        self.next_byte_mark = if next_byte_mark < end_bytes {
            align_to_stride(self.total_bytes, self.byte_stride)
        } else {
            next_byte_mark
        };
    }

    if self.count == u64::MAX || self.total_bytes == u64::MAX {
        self.realign_sample_marks();
    }
}
```

The control flow after sample insertion has three important branches:

1. **Compaction path**: If the sample buffer exceeds its cap, `grow_strides_and_compact()` fires (which internally calls `realign_sample_marks()`), and the method returns early. No per-axis mark advancement is needed because compaction realigns both marks onto the new doubled stride grid.

2. **Normal path**: Each axis advances its next-sample mark independently only when its trigger fired. The rank mark advances by one `rank_stride`. The byte mark advances by one `byte_stride`, but if the file was so large that `end_bytes` already exceeds the next mark, `align_to_stride` snaps forward to the next grid-aligned position beyond the consumed bytes. This prevents a single large file from causing the byte mark to fall behind the actual byte counter.

3. **Counter saturation**: At `u64::MAX` for either counter, `realign_sample_marks()` is called to prevent stale marks from blocking further sampling. This is a defensive edge case for streams exceeding 2^64 items or bytes.

**`estimate_split_key`** (`split_estimator.rs:741-782`) returns the retained key closest to the byte-weighted midpoint:

```rust
pub(crate) fn estimate_split_key(&self) -> Option<&[u8]> {
    if self.count < 2 {
        return None;
    }

    let max_rank = self.count - 1;
    let midpoint_rank = self.count / 2;
    let rank_target = midpoint_rank.clamp(1, max_rank);

    if self.total_bytes == 0 {
        return Self::nearest_sample(&self.samples, SampleAxis::Rank, rank_target);
    }

    let target_weight = self.total_bytes / 2;
    let candidate = Self::nearest_sample(&self.samples, SampleAxis::Bytes, target_weight)
        .or_else(|| Self::nearest_sample(&self.samples, SampleAxis::Rank, rank_target))?;

    // Keep split semantics aligned with existing connectors: avoid
    // degenerate "first item" splits when weight is front-loaded.
    if self
        .first_observed_key
        .as_ref()
        .is_some_and(|first| candidate == first.as_ref())
    {
        return Self::nearest_sample(&self.samples, SampleAxis::Rank, rank_target);
    }

    // Symmetric guard: avoid degenerate "last item" splits when weight is
    // back-loaded, which would produce an empty right shard.
    if self
        .samples
        .last()
        .is_some_and(|last| candidate == last.key.as_ref())
    {
        return Self::nearest_sample(&self.samples, SampleAxis::Rank, rank_target);
    }

    Some(candidate)
}
```

Two boundary guards prevent degenerate splits:

- **First-key guard**: when weight is front-loaded (e.g. one huge file followed by many small ones), the byte-weighted median can land on item 0, producing a zero-item left shard. The first observed key is tracked explicitly in `first_observed_key` so this guard survives downsampling that might evict the original first sample.
- **Last-key guard**: the symmetric case -- when weight is back-loaded, the candidate can land on the last key, producing an empty right shard. The check uses `samples.last()` rather than a separate tracked field because the estimator only returns sampled keys (key-fidelity invariant) and compaction always preserves the last sample.

Both guards fall back to the rank-based midpoint instead.

### Integration with the Filesystem Connector

The estimator is instance-scoped to the `FilesystemConnector`. It is fed
externally during enumeration and queried during split-point selection. When
the connector state is rebuilt, the estimator is reset so split hints always
reflect the current observation history.

**Split-point selection.** `choose_split_point_bounds` calls
`self.split_estimator.estimate_split_key()` to obtain a candidate, then
validates it against the effective shard range and cursor position via
`is_valid_split_candidate`. If the estimator has insufficient data (fewer
than two observations), or if the candidate fails validation, the method
returns `Ok(None)` -- the caller retries after more observations accumulate.

### Complexity and Memory

- **`observe`**: O(1) amortised. Compaction fires at most O(log(stream_length / sample_cap)) times, each costing O(sample_cap).
- **`estimate_split_key`**: O(log sample_cap) binary search.
- **Memory**: O(sample_cap) samples, each carrying a heap-allocated key. The default cap (`DEFAULT_SAMPLE_CAP = 1024` at `split_estimator.rs:586`) is sufficient for less than 1% byte-weighted error on the crate's 20,000-key descending-size regression workload.

### Design Choices

**Sparse sample set over full sketch.** Retaining concrete keys avoids the need to re-derive a split key from a quantile summary. The estimator only returns *actually observed* keys; it never synthesizes or interpolates key bytes. This eliminates an entire class of encoding bugs where a synthesized key might not correspond to a real file path.

**u128 arithmetic in interpolation.** The `interpolated_position` helper uses `u128` intermediate arithmetic to avoid floating-point rounding issues at byte offsets above 2^53. Recorded byte positions for large filesystems can easily exceed that threshold.

**Single byte-mark per `observe` call.** Each call records at most one sample, even if the file's byte range spans multiple stride marks. This bounds per-observation cost to O(1). For Zipf-like workloads dominated by a few very large files, the byte axis may have gaps in that region, but the rank sample for the same file still captures it. A dedicated regression test confirms less than 1% byte-weighted error on a 20,000-key Zipf-like stream.

---

## The Security Model: `open_beneath_root`

This is the function that prevents the symlink-following attack from the opening scenario. Here is the definition from `filesystem.rs:648`:

```rust
fn open_beneath_root(&self, ref_bytes: &[u8]) -> Result<(fs::File, fs::Metadata), ReadError> {
    let root_fd = self
        .root_fd
        .as_ref()
        .ok_or_else(|| ReadError::permanent("root directory handle is unavailable"))?;

    // ...component-by-component openat traversal...
}
```

This function implements four layers of defense:

**O_NOFOLLOW at every component.** The `openat` call applies `O_NOFOLLOW` to both intermediate directories and the leaf file. If an attacker replaces an intermediate directory with a symlink (e.g., `sub/` → `/etc/`), the `openat` on `sub` returns `ELOOP`, classified as permanent. Applying `O_NOFOLLOW` only at the final component would leave intermediate symlink replacements undetected.

**O_NONBLOCK on open, then fcntl to clear.** The leaf file is opened with `O_NONBLOCK` to prevent the `open` syscall from blocking indefinitely on FIFOs or device nodes. In a TOCTOU race, a regular file could be replaced with a FIFO between walk and read. Without `O_NONBLOCK`, the `open` on the FIFO would block forever. After validation, `clear_nonblock` (`filesystem.rs:1181`) removes the flag via `fcntl(F_SETFL)` so that subsequent reads use normal blocking semantics.

**fstat validation.** After opening the leaf, `metadata()` (backed by `fstat`) checks that the opened descriptor refers to a regular file. This catches the case where a file was replaced with a directory, socket, or device node between `openat` and the first read.

**Stack-allocated C string buffer.** Each path component is copied into a 256-byte stack buffer (`c_buf`) for null-termination, avoiding heap allocation in the HOT read path. Components exceeding `NAME_MAX` (255) or containing embedded NUL bytes are rejected as permanent errors.

Defense-in-depth is provided by the path component rejection (which rejects `.`, `..`, empty, and NUL-containing components) and the `openat`+`O_NOFOLLOW` traversal itself. The connector does not maintain an in-memory index for membership verification -- the `openat` traversal is the authority.

---

## Read Path: `open` and `read_range`

The read methods at `filesystem.rs:840-856`:

**`open`** opens a fresh file handle via `open_beneath_root`, clears `O_NONBLOCK`, and returns a `Box<dyn io::Read + Send>`. Budget enforcement is left to the runtime layer.

**`read_range`** uses the single-entry FD cache:

```rust
fn get_or_open_cached(&mut self, item_ref: &ItemRef) -> Result<&fs::File, ReadError> {
    let ref_bytes = item_ref.as_bytes();
    let need_open = self
        .cached_file
        .as_ref()
        .is_none_or(|cached| cached.item_ref.as_ref() != ref_bytes);
    if need_open {
        self.ensure_root_fd().map_err(enumerate_error_to_read)?;
        let (file, _metadata) = self.open_beneath_root(ref_bytes)?;
        clear_nonblock(&file)?;
        self.cached_file = Some(CachedFile {
            item_ref: ref_bytes.into(),
            file,
        });
    }
    Ok(&self.cached_file.as_ref().expect("cache must exist").file)
}
```

The cache holds at most one fd, bounding resource usage. When a `read_range` targets a different file, the old fd is dropped and a new one is opened. The `read_range` call itself uses `read_at` (positioned read) so the file offset is not shared state.

The FD cache entry is naturally invalidated when a `read_range` targets a different file than the cached one.

---

## Weak Version IDs: `derive_filesystem_version`

Here is the definition from `filesystem.rs:950-961`:

```rust
fn derive_filesystem_version(rel_path: &[u8], metadata: &fs::Metadata) -> ObjectVersionId {
    let mut version_bytes = Vec::with_capacity(rel_path.len() + 48);
    version_bytes.extend_from_slice(rel_path);
    version_bytes.push(0);
    version_bytes.extend_from_slice(&metadata.dev().to_le_bytes());
    version_bytes.extend_from_slice(&metadata.ino().to_le_bytes());
    version_bytes.extend_from_slice(&metadata.len().to_le_bytes());
    version_bytes.extend_from_slice(&metadata.mtime().to_le_bytes());
    version_bytes.extend_from_slice(&metadata.mtime_nsec().to_le_bytes());
    version_bytes.extend_from_slice(&metadata.mode().to_le_bytes());
    ObjectVersionId::from_version_bytes(&version_bytes)
}
```

The version ID is a BLAKE3 hash of the relative path bytes (NUL-terminated) concatenated with 48 bytes of file metadata: device ID, inode number, file length, modification time (seconds and nanoseconds), and file mode. Including the relative path ensures that renaming a file (without changing content or metadata) produces a distinct version ID. Including `dev()` ensures distinct version IDs across filesystem mount boundaries where inode numbers can collide. This version is wrapped in `VersionId::Weak` because it does not hash the file's content. The trade-off is deliberate: content hashing during enumeration would require reading every file, destroying the O(metadata) cost of traversal.

---

## Error Classification

I/O error classification has moved to `common.rs:210`, where it is shared between the filesystem and git connectors:

```rust
pub(crate) fn is_permanent_io_error(err: &io::Error) -> bool {
    matches!(
        err.kind(),
        io::ErrorKind::NotFound
            | io::ErrorKind::PermissionDenied
            | io::ErrorKind::InvalidInput
            | io::ErrorKind::InvalidFilename
            | io::ErrorKind::NotADirectory
            | io::ErrorKind::IsADirectory
    ) || is_symlink_loop(err)
}
```

The `is_symlink_loop` helper (`common.rs:224`) checks for `ELOOP` via `raw_os_error()` on Unix, with a non-Unix stub that returns `false`.

`NotFound` (file deleted), `PermissionDenied` (access revoked), `ELOOP` (symlink detected by `O_NOFOLLOW`), `NotADirectory` (intermediate path component replaced), and `IsADirectory` (leaf component replaced with a directory) are all permanent. Transient errors like `EAGAIN` or `EWOULDBLOCK` remain retryable.

The classification wrappers `classify_io_enumerate_error` (`common.rs:238`) and `classify_io_read_error` (`common.rs:256`) redact paths through `ToxicDigest` so raw filesystem paths never appear in error messages.

The `ELOOP` classification deserves special attention. When `openat` is called with `O_NOFOLLOW` on a symlink, the kernel returns `ELOOP`. This is permanent because the symlink is a structural property of the filesystem -- retrying the same path will produce the same result.

---

## Consistency Model

The `O_NOFOLLOW` traversal prevents symlink-based attacks during the read path. Content-level consistency (detecting that a file changed between enumeration and read) is delegated to the orchestration runtime, which can compare the weak version ID from enumeration against a freshly computed one after reading.

The FD cache widens the TOCTOU window slightly: a cached descriptor may outlive a concurrent file replacement. Callers requiring snapshot consistency should use version checks at a higher layer.

---

## Summary

The `FilesystemConnector` (~1,233 lines) provides a security-hardened filesystem connector for Unix platforms. `ensure_root_fd` lazily opens and identity-verifies the root directory without latching failures. `open_beneath_root` applies `O_NOFOLLOW` at every component of the `openat` traversal, opens with `O_NONBLOCK` to prevent FIFO blocking, and validates via `fstat`. The single-entry FD cache optimizes sequential `read_range` calls. Split hints are provided by an integrated `StreamingSplitEstimator` that produces byte-weighted split keys in bounded memory. Key-range intersection narrows split-point selection to the effective `[start, end)` range. `is_permanent_io_error` in `common.rs` is shared between filesystem and git connectors. Chapter 6 covers the git connector.
