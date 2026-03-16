# "Every Component Verified" -- Filesystem Connector

*A secret scanner is deployed at `/data/repos` to scan 14,000 git checkouts for leaked credentials. Repository `acme-infra` contains a symlink at `config/database.yml` pointing to `/etc/shadow`. The indexing walk follows the symlink, classifying `/etc/shadow` as a scannable file with key `acme-infra/config/database.yml`. The connector's `open()` call reads `/etc/shadow` through the symlink, handing 1,847 bytes of password hashes to the detection engine. The scanner -- designed to find secrets -- has just exfiltrated the system's password database through its own read path. Every finding from that file is a false positive that contains real credential material. If those findings are shipped to the customer's dashboard, the scanner has become the breach.*

---

## Why the Filesystem Connector Exists

The in-memory connector from Chapter 5 proves that the enumeration contract works for items held entirely in process memory. Production scanners need a connector that reads files from disk. This introduces three problems the in-memory connector does not face: symlinks can escape the scan root, filesystem state can change between enumeration and reading (TOCTOU), and the directory walk itself can be expensive enough to blow deadlines.

The `FilesystemConnector` in `crates/gossip-connectors/src/filesystem.rs` solves all three. It provides read and split-point methods over a rooted directory tree, applying symlink-aware traversal, component-by-component `openat` with `O_NOFOLLOW` at every step, and a streaming DFS walk with deadline enforcement. Chapter 5's in-memory connector shares the binary-search and byte-weighted-split utilities from `common.rs`; this chapter shows how the filesystem connector replaces those shared index-then-search primitives with a fundamentally different streaming architecture that trades snapshot isolation for bounded memory.

> **Platform availability:** `FilesystemConnector` is only available on Unix
> (`#[cfg(unix)]`). The module relies on Unix-specific APIs: `openat` with
> `O_NOFOLLOW` and `O_DIRECTORY` flags, raw file descriptor manipulation via
> `OwnedFd` and `libc`, and `std::os::unix::fs::MetadataExt` for inode and
> device fields. Non-Unix platforms are not supported.

### Pooled Page Emission

Like the in-memory connector, `FilesystemConnector` uses the pooled toxic-byte page assembly path for enumeration. Filesystem `ItemRef` bytes are identical to `ItemKey` bytes (both are the `/`-separated relative path), so each item's bytes are staged once into a page-local `ByteSlab` and both wrappers are materialized from that same slot via `assemble_pooled_page_shared_key_ref` in `common.rs`. This shared-slot optimization cuts staging copies versus the generic two-slot path while keeping wrapper cloning allocation-free in the HOT emit loop.

---

## Why Streaming Instead of Batch Indexing?

The previous design materialized the entire directory tree into a sorted `Vec<FileEntry>` before serving the first page. This had a simple appeal: binary search made cursor resume O(log n), and a prefix-sum array gave O(log n) byte-weighted split selection. But it also meant the connector's memory usage was proportional to the total number of files under the root -- a 2-million-file repository consumed ~200 MB of index entries before a single page was emitted.

The streaming DFS model replaces this with memory proportional to the *active DFS frontier*:

- `O(Σ entries_per_active_dir)` for buffered per-directory entry frames, plus
- `O(visited_dirs)` for cycle-detection identities collected during descent.

For a tree with 10 levels of nesting and 100 files per directory, the old model held all files in memory. The new model holds at most 10 sorted directory buffers (one per DFS depth) plus the visited-directory set. Split-point hints are supported via an integrated `StreamingSplitEstimator` that is fed as each in-range file is emitted during pagination. The estimator uses byte-weighted dual-axis sampling in bounded memory, replacing the batch predecessor's prefix-sum approach without requiring a full in-memory index. See [Streaming Split Estimation](#streaming-split-estimation-streamingsplitestimator) below.

---

## WalkWarning: Non-Fatal Issue Records

Before examining the walk machinery, it is worth understanding how the connector handles non-fatal issues. Not every problem encountered during traversal is fatal -- symlinks, unreadable subdirectories, encoding failures, and special files (FIFOs, sockets, device nodes) are all skipped rather than aborting the walk. Each skipped entry produces a warning.

Here is the definition from `filesystem.rs:130-135`:

```rust
#[derive(Debug)]
pub struct WalkWarning {
    /// Redacted digest of the path that triggered the warning.
    pub path_digest: ToxicDigest,
    /// Human-readable description of why the entry was skipped.
    pub message: String,
}
```

The `path_digest` field stores a `ToxicDigest` rather than a raw `PathBuf` so that filesystem paths never leak into log-visible diagnostics. The digest is computed via `ToxicDigest::of_bytes(path.as_os_str().as_bytes())` at warning creation time. Convenience constructors `WalkWarning::io(path, op, err)` and `WalkWarning::skipped(path, reason)` handle this conversion internally.

Warnings are capped at `max_warnings` (default 1024). Beyond the cap, only the overflow count is incremented via the `push_walk_warning` helper at `filesystem.rs:1149-1160`:

```rust
fn push_walk_warning(
    warnings: &mut Vec<WalkWarning>,
    max_warnings: usize,
    overflow_count: &mut usize,
    warning: WalkWarning,
) {
    if warnings.len() < max_warnings {
        warnings.push(warning);
    } else {
        *overflow_count += 1;
    }
}
```

This prevents pathological directories (e.g., a directory with 100,000 symlinks) from consuming unbounded memory for warning storage. Callers can inspect `walk_warnings()` after enumeration to audit what was skipped, and `overflow_warning_count()` to detect whether warnings were dropped.

---

## The Streaming Walk Architecture

### Core Types

The streaming DFS walk is built from four types that work together to yield globally sorted files one at a time. Understanding these types is essential before reading the connector itself.

**`BufferedDirEntry`** (`filesystem.rs:180-183`) is the simplest -- one entry read from a directory, before any path encoding or metadata lookup:

```rust
struct BufferedDirEntry {
    name: OsString,
    file_type: fs::FileType,
}
```

**`WalkFrame`** (`filesystem.rs:190-199`) represents one directory on the DFS stack. Its `entries` deque is pre-sorted (using `cmp_dir_entry`) when the directory is first opened, so popping from the front preserves per-directory order:

```rust
struct WalkFrame {
    /// Directory component relative to parent (`None` for root frame).
    component: Option<OsString>,
    depth: usize,
    /// Yielded-entry count since the last deadline check.
    entries_since_check: usize,
    /// Index of the next child to poll in this frame's sorted entry list.
    next_child_index: u32,
    /// Sorted remaining entries for this directory.
    entries: VecDeque<BufferedDirEntry>,
}
```

The `next_child_index` field tracks how many entries have been consumed from this frame. This value is serialized into walk tokens for cursor resume (discussed later). The `entries_since_check` counter drives periodic deadline checks at a cadence of `DEADLINE_CHECK_INTERVAL` (512) entries, balancing responsiveness against `Instant::now()` syscall overhead.

**`WalkQuery`** (`filesystem.rs:208-216`) bundles the parameters that stay constant across a single `next_file` call:

```rust
struct WalkQuery<'a> {
    root: &'a Path,
    connector_instance: ConnectorInstanceIdHash,
    max_depth: usize,
    start: Option<&'a [u8]>,
    end: Option<&'a [u8]>,
    deadline: Option<Instant>,
    max_warnings: usize,
}
```

**`WalkState`** (`filesystem.rs:258-274`) is the resumable DFS walker itself. This is the heart of the streaming model:

```rust
struct WalkState {
    stack: Vec<WalkFrame>,
    current_path: PathBuf,
    pending: Option<FileEntry>,
    last_emitted_key: Option<ItemKey>,
    emitted_count: usize,
    exhausted: bool,
    /// `(dev, ino)` pairs of directories already descended into.
    visited_dirs: HashSet<(u64, u64)>,
}
```

The key invariants, documented in the source:

- `stack` + `current_path` represent the active DFS frontier.
- `pending` stores exactly one already-discovered file that must be emitted before continuing traversal (used for upper-bound stop and cursor seek).
- `last_emitted_key` tracks connector-visible resume position.
- `exhausted` is sticky once a full walk pass returns `None`.
- `visited_dirs` tracks `(dev, ino)` pairs to break cycles from bind mounts or directory hardlinks.

### Why Sorted Per-Directory Yields Global Order

The insight that makes streaming work without a global sort is: *per-directory sorting plus depth-first traversal produces globally sorted keys*. When the walker enters a directory, it reads all entries, sorts them using `cmp_with_trailing_sep` (which treats directory names as if they had a virtual trailing `/`), and then processes them front-to-back. Directories are descended into immediately (depth-first), so all descendants of entry N are emitted before entry N+1. Since the entries within each directory are sorted, and descendants inherit their parent's prefix, the global key sequence is sorted.

The `cmp_with_trailing_sep` function at `filesystem.rs:1194-1203` implements this ordering:

```rust
fn cmp_with_trailing_sep(
    left: &[u8],
    left_is_dir: bool,
    right: &[u8],
    right_is_dir: bool,
) -> std::cmp::Ordering {
    let l = left.iter().copied().chain(left_is_dir.then_some(b'/'));
    let r = right.iter().copied().chain(right_is_dir.then_some(b'/'));
    l.cmp(r)
}
```

This mirrors Git tree ordering (`name` vs `name/`) and is the key comparator that makes local per-directory sorting equivalent to sorting fully encoded relative paths globally.

---

## The FilesystemConnector Struct

Here is the definition from `filesystem.rs:283-301`:

```rust
pub struct FilesystemConnector {
    root: PathBuf,
    connector_instance: ConnectorInstanceIdHash,
    emit_tokens: bool,
    max_walk_depth: usize,
    max_warnings: usize,
    walk_key_range_start: Option<Box<[u8]>>,
    walk_key_range_end: Option<Box<[u8]>>,
    walk_state: Option<WalkState>,
    walk_warnings: Vec<WalkWarning>,
    overflow_warning_count: usize,
    /// Directory fd for the canonical root, opened lazily.
    root_fd: Option<OwnedFd>,
    /// Single-entry FD cache for sequential `read_range` calls.
    cached_file: Option<CachedFile>,
    /// Streaming split-point estimator fed during pagination walks and reset
    /// whenever walk state is rebuilt from root.
    split_estimator: StreamingSplitEstimator,
}
```

Field-by-field breakdown:

- **`root: PathBuf`** -- The directory root passed at construction. On the first enumeration call, this is canonicalized via `fs::canonicalize` to resolve relative paths and root-level symlinks. Canonicalization prevents cwd drift: if the process working directory changes after construction, the connector still opens files from the correct absolute path.

- **`connector_instance: ConnectorInstanceIdHash`** -- A hash derived from the root path bytes at construction time via `ConnectorInstanceIdHash::from_instance_id_bytes`. This scopes stable item IDs to the connector instance, so identical relative paths under different roots produce distinct `StableItemId` values. The hash participates in `derive_stable_item_id` and is threaded through `WalkQuery` to each `next_file` call.

- **`emit_tokens: bool`** -- Controls whether walk-state tokens are included in cursors. Defaults to `false`. When `true`, each page's next cursor carries a serialized `WalkToken` that encodes the DFS stack position, enabling O(1) resume without rewinding from root. Key-based resume remains available regardless of this setting.

- **`max_walk_depth: usize`** -- Hard limit on directory recursion depth (default: 512). Directories deeper than this limit are skipped with a warning, preventing pathological directory trees from exhausting the DFS stack.

- **`max_warnings: usize`** -- Cap on stored `WalkWarning` entries (default: 1024). Warnings beyond this cap are counted in `overflow_warning_count` but not stored.

- **`walk_key_range_start` / `walk_key_range_end: Option<Box<[u8]>>`** -- Connector-level key-range bounds, set via `with_key_range()` or `with_shard_bounds()`. These are intersected with per-request shard bounds before traversal, and enable subtree pruning during the walk (see [Key-Range Pruning](#key-range-pruning-should_skip_subtree) below).

- **`walk_state: Option<WalkState>`** -- The live DFS traversal state. `None` before the first enumeration; `Some(...)` while walking. Unlike the old `IndexState` tri-state, this is either absent or active -- there is no "Failed" latch. The streaming model intentionally retries on every call when the root is unavailable because root-directory conditions can change between calls (e.g., a mount appearing, permissions being adjusted).

- **`walk_warnings: Vec<WalkWarning>`** -- Accumulated non-fatal walk diagnostics. Warnings persist across pages and cursor re-alignments until the connector is dropped.

- **`overflow_warning_count: usize`** -- Count of warnings dropped because the buffer was full.

- **`root_fd: Option<OwnedFd>`** -- A directory file descriptor for the canonicalized root, opened lazily by `ensure_root_fd`. All read-path operations use `openat` relative to this fd, confining file access to the subtree under the root.

- **`cached_file: Option<CachedFile>`** -- A single-entry FD cache for sequential `read_range` calls, using a named struct (`filesystem.rs:174-177`):

```rust
struct CachedFile {
    item_ref: Box<[u8]>,
    file: fs::File,
}
```

The cache is keyed on `item_ref` bytes (the encoded relative path). When a `read_range` targets a different file, the old fd is dropped and a new one is opened.

- **`split_estimator: StreamingSplitEstimator`** -- A bounded-memory streaming estimator that tracks file keys and byte weights as they are emitted during pagination. Each in-range file's key and size are fed to the estimator via `observe` during page assembly. The estimator is reset in `reset_walk_state` whenever the walk is rebuilt from root, so split hints always reflect the current walk's observation history rather than mixing data from a prior traversal. See [Streaming Split Estimation](#streaming-split-estimation-streamingsplitestimator) for a full description.

### Builder Methods

The connector uses a builder pattern for configuration. All builder methods return `Self` and are marked `#[must_use]`:

- **`with_tokens(enabled: bool)`** -- Enable or disable opaque resume tokens.
- **`with_key_range(start, end)`** -- Restrict traversal to keys inside the half-open range `[start, end)`. `None` on either side means unbounded.
- **`with_shard_bounds(start, end)`** -- Convenience wrapper matching `ShardSpec` semantics: empty slices are treated as unbounded.
- **`with_max_depth(max_depth)`** -- Set a hard ceiling on DFS depth.
- **`with_max_warnings(max_warnings)`** -- Set warning retention cap.

---

## FileEntry: One Walk Output

Here is the definition from `filesystem.rs:162-167`:

```rust
struct FileEntry {
    key: ItemKey,
    stable_item_id: StableItemId,
    version: VersionId,
    size: u64,
}
```

Unlike the old design where `FileEntry` was one row in a persistent in-memory index, this is now an internal staging type between `WalkState::next_file` and page assembly. It does not represent a persisted or full in-memory index -- it is produced, consumed into a `ScanItem`, and discarded within a single page.

- **`key: ItemKey`** -- The encoded relative path. Both the sort key and the value returned as `ItemKey` in enumeration pages.
- **`stable_item_id: StableItemId`** -- Derived via `derive_stable_item_id(FILESYSTEM_CONNECTOR_TAG, query.connector_instance, &key)`. The `FILESYSTEM_CONNECTOR_TAG` is `ConnectorTag::from_ascii(b"fslocal")`, and `query.connector_instance` is the `ConnectorInstanceIdHash` derived from the root path. Together they ensure filesystem-sourced items have identity hashes disjoint from other connector types and other connector instances scanning different roots.
- **`version: VersionId`** -- Always `VersionId::Weak(...)`, computed from file metadata. Weak versions are cheaper than content hashing but can diverge between enumeration and read if the file changes.
- **`size: u64`** -- File byte size at walk time, used for size hints in enumeration pages.

---

## Lazy Initialization: `ensure_root_fd` and `start_walk_if_needed`

Construction of a `FilesystemConnector` is cheap: no I/O occurs. The first call that needs the filesystem triggers two lazy initialization steps.

### `ensure_root_fd` (filesystem.rs:422-445)

This method canonicalizes the root path and opens a directory fd, but -- critically -- it does *not* latch failures. The streaming walk model intentionally retries on every call when the root is unavailable because root-directory conditions can change between calls (3-4 syscalls per attempt is cheap). This is a deliberate departure from the old `IndexState::Failed` memoization pattern.

```rust
fn ensure_root_fd(&mut self) -> Result<(), EnumerateError> {
    if self.root_fd.is_some() {
        return Ok(());
    }

    // Canonicalize root to prevent cwd drift and resolve root symlinks.
    self.root = fs::canonicalize(&self.root)
        .map_err(|err| classify_io_enumerate_error("canonicalize", &self.root, &err))?;

    // Capture path identity BEFORE open so the authoritative check is
    // fstat on the fd (which cannot be swapped out from under us).
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

The root identity verification step (`verify_root_identity` at `filesystem.rs:2186-2194`) narrows the TOCTOU window. The connector captures the canonical path's `(dev, ino)` *before* opening, then compares it to the opened fd's `fstat` result. The race window is stat→open (same direction) rather than open→stat where the fd could refer to a different directory. If the identity mismatches, the error is classified as retryable (a directory swap is a transient environmental condition, not a permanent attribute of the path).

### `start_walk_if_needed` (filesystem.rs:448-463)

This method creates the initial `WalkState` on the first enumeration request:

```rust
fn start_walk_if_needed(&mut self, deadline: Option<Instant>) -> Result<(), EnumerateError> {
    self.ensure_root_fd()?;
    if self.walk_state.is_some() {
        return Ok(());
    }

    let state = WalkState::new(
        &self.root,
        deadline,
        self.max_warnings,
        &mut self.walk_warnings,
        &mut self.overflow_warning_count,
    )?;
    self.walk_state = Some(state);
    Ok(())
}
```

`WalkState::new` (`filesystem.rs:1512-1559`) reads the root directory, sorts its entries into the first `WalkFrame`, seeds the `visited_dirs` set with the root's `(dev, ino)`, and pre-allocates a path buffer with 4 KiB of headroom for typical path component growth. The walker is now ready to yield files.

---

## The Directory Walk: `WalkState::next_file`

The `next_file` method (`filesystem.rs:1724-2008`) is the engine of the streaming model. Each call returns the next regular file in globally sorted DFS order, or `None` when the walk is exhausted. The walker is resilient to filesystem churn: unreadable entries, symlinks, special files, and encoding failures become warnings and are skipped.

The control flow within `next_file`:

1. **Drain pending.** If a previous call stashed a file in `self.pending` (from an upper-bound stop or cursor seek), return it immediately.

2. **Check exhaustion.** If the walk previously returned `None` and set `self.exhausted = true`, return `None` immediately.

3. **Poll the DFS stack.** Pop the next entry from the top frame's sorted deque. If the frame is empty, pop it from the stack and continue to the parent frame.

4. **Handle symlinks.** Skip with a warning.

5. **Handle directories.** This is where several new features activate:
   - **Key-range subtree pruning:** If bounds are present, compute the directory's encoded prefix and call `should_skip_subtree` to test whether the subtree can possibly overlap the requested key range. If not, skip without descent.
   - **Cycle detection:** Call `fs::metadata` to get the directory's `(dev, ino)` and check the `visited_dirs` set. Duplicate identities (from bind mounts or directory hardlinks) are skipped with a warning. Note that cycle detection is placed *after* pruning so that a pruned directory does not poison the visited set -- a later in-range path to the same inode can still descend.
   - **Depth limit:** If `depth >= max_depth`, skip with a warning.
   - **Descent:** Read the subdirectory, sort its entries via `read_dir_sorted_entries`, and push a new `WalkFrame` onto the stack.

6. **Handle non-regular files.** Skip with a warning.

7. **Stat the file.** Call `fs::symlink_metadata` (which does not follow symlinks) and verify the result is still a regular file (guards against TOCTOU type changes between `readdir` and `stat`).

8. **Encode the key.** Strip the root prefix, encode via `encode_rel_path`, and apply in-range filtering against the query's start/end bounds. Since keys are globally sorted, exceeding the upper bound means every subsequent key will too -- the walker returns `None` without setting `exhausted` (the walk state may be reused with different bounds).

9. **Derive identity and version.** Compute `StableItemId` via domain-separated BLAKE3 and `VersionId::Weak` from metadata.

10. **Return the `FileEntry`.** Pop the entry's component from `current_path` and return.

When the DFS stack empties, `self.exhausted` is set to `true` and subsequent calls return `None`.

### Bounded Directory Size: `MAX_ENTRIES_PER_DIR`

The `read_dir_sorted_entries` function (`filesystem.rs:1431-1485`) enforces a hard cap of 500,000 entries per directory (`MAX_ENTRIES_PER_DIR` at `filesystem.rs:1145`). Pathological directories (e.g., `/proc`-like mounts or intentional DoS) could otherwise exhaust memory during the sort step. 500K entries × ~40 bytes ≈ 20 MB, which is large but bounded. Exceeding this limit returns a retryable error rather than a warning, because the directory's sorted state cannot be produced.

---

## Key-Range Pruning: `should_skip_subtree`

When key-range bounds are present (from `with_key_range()`, `with_shard_bounds()`, or per-request shard bounds), the walker can skip entire directory subtrees without descent. This is the function from `filesystem.rs:1401-1424`:

```rust
pub(crate) fn should_skip_subtree(
    dir_prefix: &[u8],
    shard_start: Option<&[u8]>,
    shard_end: Option<&[u8]>,
) -> bool {
    if dir_prefix.is_empty() {
        return false;
    }

    // Subtree entirely below range:
    let mut subtree_key = InlineVec::<u8, 256>::new();
    subtree_key.extend_from_slice(dir_prefix);
    subtree_key.push(b'/');
    if let Some(successor) = prefix_successor(subtree_key.as_slice())
        && shard_start.is_some_and(|start| successor.as_slice() <= start)
    {
        return true;
    }

    // Subtree entirely above range:
    shard_end.is_some_and(|end| end <= subtree_key.as_slice())
}
```

The logic works on the encoded key prefix of the directory (e.g., `b"src/lib"`). Keys under the subtree are bounded by:
- Inclusive lower bound: `dir_prefix + b'/'`
- Exclusive upper bound: the byte-level successor of that lower bound (via `prefix_successor`)

Two pruning conditions:
1. **Subtree entirely below range:** `prefix_successor(dir_prefix + b'/') <= shard_start`. Every possible key under this directory sorts before the shard start.
2. **Subtree entirely above range:** `shard_end <= dir_prefix + b'/'`. Every possible key under this directory sorts at or after the shard end.

The implementation is conservative: `true` means safe-to-skip; `false` may still be out of range but is kept to avoid false-positive pruning. Root prefixes (empty) are never pruned. Prefixes whose last byte is `0xFF` (no finite successor) are never pruned on the below-range side. Correctness is enforced at the leaf level -- the pruning is an optimization, not the authority.

The `prefix_successor` helper (`filesystem.rs:1381-1388`) uses `InlineVec<u8, 256>` to avoid heap allocation for typical path-length prefixes:

```rust
pub(crate) fn prefix_successor(prefix: &[u8]) -> Option<InlineVec<u8, 256>> {
    let last = *prefix.last()?;
    if last == u8::MAX {
        return None;
    }
    let mut out = InlineVec::<u8, 256>::new();
    out.extend_from_slice(&prefix[..prefix.len() - 1]);
    out.push(last + 1);
    Some(out)
}
```

---

## Cursor Alignment and Resume

The streaming walk introduces a more complex resume protocol than the old binary-search model. The connector must reconcile its live `WalkState` with the caller-provided `Cursor` on every page request.

### `align_walk_to_cursor` (filesystem.rs:593-671)

This is the entry point for cursor reconciliation. The logic:

1. **Walk state exists and matches cursor.** The `cursor_matches_state` helper (`filesystem.rs:496-502`) compares `WalkState::last_emitted_key` against `cursor.last_key()`. If they agree, the walk is already positioned correctly -- continue from where we left off.

2. **Cursor has no last key.** This is an initial cursor. Reset the walk state from root.

3. **Cursor has a last key and tokens are enabled.** Attempt to decode a `WalkToken` from the cursor, then call `WalkState::from_token` to rebuild the DFS stack at the saved position. If token restore succeeds, seek forward to the first key strictly greater than `last_key`.

4. **Token decode/restore fails (or tokens are disabled).** Fall back to rebuilding from root and seeking forward via `seek_after_last_key`.

The `cursor_matches_state` function has a subtle guard: an exhausted walk that never emitted any key (`last_emitted_key == None`) must *not* match an initial cursor (`last_key == None`). Without this, reusing a connector across ranges would silently drop results.

### `seek_after_last_key` (filesystem.rs:508-555)

After a walk reset (either from token restore or root rebuild), this method advances the walker through the sorted key space until it finds the first file strictly greater than `last_key`, staging that file into `WalkState::pending` for the next page:

```rust
fn seek_after_last_key(
    &mut self,
    last_key: &ItemKey,
    walk_query: WalkQuery<'_>,
) -> Result<Option<ItemKey>, EnumerateError> {
    let mut skipped = 0usize;
    // Clear any stale pending entry
    {
        let state = self.walk_state.as_mut()
            .expect("walk state must exist while seeking to cursor");
        state.pending = None;
    }

    loop {
        let next = { /* ... call state.next_file(...) ... */ };
        let Some(file) = next else { break; };
        if file.key.as_bytes() > last_key.as_bytes() {
            let state = self.walk_state.as_mut()
                .expect("walk state must exist while seeking to cursor");
            state.pending = Some(file);
            break;
        }
        skipped += 1;
    }
    // ...
}
```

This is O(n) in the worst case (full rewind from root). Token-based resume amortizes this by restoring the DFS stack to a position near the target, reducing the seek distance to approximately one directory's worth of entries.

### Walk Tokens: `WalkToken` and `WalkTokenFrame`

Walk tokens provide a serialized DFS checkpoint that can be embedded in cursors. The types (`filesystem.rs:230-244`):

```rust
struct WalkToken {
    frames: Vec<WalkTokenFrame>,
}

struct WalkTokenFrame {
    component: Vec<u8>,
    next_child_index: u32,
}
```

The wire format is compact: `[version: u8][frame_count: u16][frames...]`, where each frame is `[component_len: u16][component_bytes][next_child_index: u32]`. Total size is bounded by `MAX_TOKEN_SIZE`.

**Encoding** (`WalkToken::encode_from_state`, `filesystem.rs:1321-1377`) serializes the current DFS stack in a single buffer. If the stack exceeds the token size budget, frames are truncated from the bottom. The truncated leaf frame has its `next_child_index` rewound by one to force re-entry of the directory whose descendants were dropped.

**Decoding** (`WalkToken::decode_bytes`, `filesystem.rs:1270-1319`) validates the wire format with several safety checks:
- Version byte must match `WALK_TOKEN_VERSION` (0x01).
- Root frame must have an empty component.
- Non-root frames must have non-empty components with no `/`, NUL, `.`, or `..` (preventing path traversal).
- Pre-allocation is capped at 64 frames regardless of the declared count (defense against forged tokens).
- Trailing bytes are rejected.

**Restoring** (`WalkState::from_token`, `filesystem.rs:1571-1702`) walks the token's frame sequence against the live filesystem: for each frame, it reads the corresponding directory, sorts entries, fast-forwards the deque by `next_child_index`, and verifies no symlinks or cycles are introduced by the token-supplied components. Any failure returns `Ok(None)`, falling back to key-only resume.

In test builds, the connector runs a cross-check: after token-based resume, it performs an independent key-only walk from root to verify both resume methods agree on the next key. On mismatch, the token result is discarded and key-only position is used.

---

## How Scanning is Driven

The filesystem connector does not expose a page-by-page enumeration method
to external callers. Instead, scanning is orchestrated through
`gossip-scanner-runtime`, which uses the `ordered_content` module to drive
filesystem scanning. The walk state, cursor alignment, and split estimation
mechanisms described above are used internally to traverse the filesystem
within the assigned shard bounds.

### Capabilities

The `caps` method at `filesystem.rs` reports:

```rust
pub fn caps(&self) -> ConnectorCapabilities {
    ConnectorCapabilities {
        seek_by_key: true,
        token_resume: self.emit_tokens,
        range_read: true,
        split_hints: true,
    }
}
```

Split hints are enabled. `choose_split_point_bounds` (`filesystem.rs:897-932`) validates deadline and range bounds (for consistent error classification), then delegates to the integrated `StreamingSplitEstimator`. The estimator is fed during pagination walks on the same connector instance, so no separate full-range walk is required. A fresh connector or one that has just rebuilt its walk state starts from an empty estimator and returns `Ok(None)` until sufficient pages have been served. The returned split key is further guarded by `is_valid_split_candidate` to prevent degenerate splits that fall before the cursor or outside the effective shard range.

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

The estimator is instance-scoped to the `FilesystemConnector`. Three integration points tie it to the connector's lifecycle:

**Feeding during scanning.** During the walk, every in-range file emitted is observed:

```rust
// Only caller-visible, in-range files contribute to the estimator.
self.split_estimator.observe(file.key.as_bytes(), file.size);
```

Files outside the effective key range (pruned by subtree or bound checks) are not fed, so the estimator's sketch reflects only the key space the connector is actively serving.

**Reset on walk rebuild.** `reset_walk_state` (`filesystem.rs:476-486`) reconstructs the `WalkState` from root and replaces the estimator with a fresh instance:

```rust
fn reset_walk_state(&mut self, deadline: Option<Instant>) -> Result<(), EnumerateError> {
    let state = WalkState::new(
        &self.root,
        deadline,
        self.max_warnings,
        &mut self.walk_warnings,
        &mut self.overflow_warning_count,
    )?;
    self.cached_file = None;
    self.walk_state = Some(state);
    self.split_estimator = StreamingSplitEstimator::new(Self::SPLIT_ESTIMATOR_SAMPLE_CAP);
    Ok(())
}
```

This prevents stale observations from a prior walk from contaminating split hints for a new traversal. Cursor realignment, key-range changes, and any other event that rebuilds the walk from root also reset the estimator.

**Split-point selection.** `choose_split_point_bounds` (`filesystem.rs:897-932`) calls `self.split_estimator.estimate_split_key()` to obtain a candidate, then validates it against the effective shard range and cursor position via `is_valid_split_candidate`. If the estimator has insufficient data (fewer than two observations), or if the candidate fails validation, the method returns `Ok(None)` -- the caller retries after more pages have been served.

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

This is the function that prevents the symlink-following attack from the opening scenario. Here is the definition from `filesystem.rs:946-1022`:

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

**O_NONBLOCK on open, then fcntl to clear.** The leaf file is opened with `O_NONBLOCK` to prevent the `open` syscall from blocking indefinitely on FIFOs or device nodes. In a TOCTOU race, a regular file could be replaced with a FIFO between walk and read. Without `O_NONBLOCK`, the `open` on the FIFO would block forever. After validation, `clear_nonblock` (`filesystem.rs:2114-2135`) removes the flag via `fcntl(F_SETFL)` so that subsequent reads use normal blocking semantics.

**fstat validation.** After opening the leaf, `metadata()` (backed by `fstat`) checks that the opened descriptor refers to a regular file. This catches the case where a file was replaced with a directory, socket, or device node between `openat` and the first read.

**Stack-allocated C string buffer.** Each path component is copied into a 256-byte stack buffer (`c_buf`) for null-termination, avoiding heap allocation in the HOT read path. Components exceeding `NAME_MAX` (255) or containing embedded NUL bytes are rejected as permanent errors.

Note that the streaming model no longer has a `verify_membership` step that binary-searches the index before reads. The old batch model could cheaply verify that an `ItemRef` existed in the sorted index; the streaming model does not retain a full index. Defense-in-depth is provided by `encode_rel_path` (which rejects `.`, `..`, and root prefixes) and the `openat`+`O_NOFOLLOW` traversal itself.

---

## Read Path: `open` and `read_range`

The read methods at `filesystem.rs:1091-1129`:

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

The FD cache is cleared during `reset_walk_state` (`filesystem.rs:476-486`), because cursor-style jumps can move reads to unrelated paths where keeping stale per-file cache entries provides no locality benefit.

---

## Path Encoding: `encode_rel_path`

Here is the definition from `filesystem.rs:2026-2048`:

```rust
fn encode_rel_path(rel: &Path) -> Result<Vec<u8>, EnumerateError> {
    let mut out = Vec::new();
    for component in rel.components() {
        let normal = match component {
            std::path::Component::Normal(segment) => segment,
            _ => {
                return Err(EnumerateError::permanent(format!(
                    "path contains non-normal component ({})",
                    ToxicDigest::of_bytes(rel.as_os_str().as_bytes())
                )));
            }
        };
        if !out.is_empty() {
            out.push(b'/');
        }
        out.extend_from_slice(normal.as_bytes());
    }
    if out.is_empty() {
        return Err(EnumerateError::permanent(
            "relative file path encoded to an empty key",
        ));
    }
    Ok(out)
}
```

Only `Component::Normal` segments are accepted -- `.`, `..`, and root prefixes are rejected with error messages using `ToxicDigest` to redact raw paths. Segments are joined with `/` (byte `0x2F`), producing a key whose lexicographic byte order matches the logical path order. The encoding is reversible by splitting on `/`, since Unix filenames cannot contain `/`. Empty paths are rejected because they would produce a zero-length `ItemKey`.

---

## Weak Version IDs: `derive_fs_version_id`

Here is the definition from `filesystem.rs:2068-2076`:

```rust
fn derive_fs_version_id(metadata: &fs::Metadata) -> ObjectVersionId {
    let mut encoded = [0u8; 40];
    encoded[0..8].copy_from_slice(&metadata.mtime().to_be_bytes());
    encoded[8..16].copy_from_slice(&metadata.mtime_nsec().to_be_bytes());
    encoded[16..24].copy_from_slice(&metadata.len().to_be_bytes());
    encoded[24..32].copy_from_slice(&metadata.ino().to_be_bytes());
    encoded[32..40].copy_from_slice(&metadata.dev().to_be_bytes());
    ObjectVersionId::from_version_bytes(&encoded)
}
```

The version ID is a BLAKE3 hash of 40 bytes of file metadata: modification time (seconds and nanoseconds), file length, inode number, and device ID. Including `dev()` ensures distinct version IDs across filesystem mount boundaries where inode numbers can collide. This version is wrapped in `VersionId::Weak` because it does not hash the file's content. The trade-off is deliberate: content hashing during enumeration would require reading every file, destroying the O(metadata) cost of traversal.

---

## Error Classification

I/O error classification has moved to `common.rs:620-630`, where it is shared between the filesystem and git connectors:

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

The `is_symlink_loop` helper (`common.rs:634-636`) checks for `ELOOP` via `raw_os_error()` on Unix, with a non-Unix stub that returns `false`.

`NotFound` (file deleted), `PermissionDenied` (access revoked), `ELOOP` (symlink detected by `O_NOFOLLOW`), `NotADirectory` (intermediate path component replaced), and `IsADirectory` (leaf component replaced with a directory) are all permanent. Transient errors like `EAGAIN` or `EWOULDBLOCK` remain retryable.

The classification wrappers `classify_io_enumerate_error` (`common.rs:648-659`) and `classify_io_read_error` (`common.rs:666-676`) redact paths through `ToxicDigest` so raw filesystem paths never appear in error messages.

The `ELOOP` classification deserves special attention. When `openat` is called with `O_NOFOLLOW` on a symlink, the kernel returns `ELOOP`. This is permanent because the symlink is a structural property of the filesystem -- retrying the same path will produce the same result.

---

## Consistency Model

Enumeration is streaming, not snapshot-isolated. Directory entries are buffered per frame when that directory is opened, and file metadata is read later when entries are emitted. Concurrent filesystem mutation can therefore make specific items appear/disappear between pages. When races are detected (e.g., a file that was a regular file at `readdir` time becomes a directory at `stat` time), the walk records a `WalkWarning` and continues.

This is an inherent limitation of the streaming model, not a bug. The `O_NOFOLLOW` traversal prevents symlink-based attacks during the read path. Content-level consistency (detecting that a file changed between enumeration and read) is delegated to the orchestration runtime, which can compare the weak version ID from enumeration against a freshly computed one after reading.

The FD cache widens the TOCTOU window slightly: a cached descriptor may outlive a concurrent file replacement. Callers requiring snapshot consistency should use version checks at a higher layer.

---

## Summary

The `FilesystemConnector` provides a security-hardened, deadline-aware, streaming filesystem connector. `WalkState` drives a resumable sorted DFS walk that keeps memory proportional to the active directory frontier rather than the total file count. `WalkFrame` buffers and sorts one directory's entries for globally ordered key emission via `cmp_with_trailing_sep`. `ensure_root_fd` lazily opens and identity-verifies the root directory without latching failures. `should_skip_subtree` prunes entire directory subtrees that cannot overlap the requested key range, and `visited_dirs` breaks traversal cycles from bind mounts or directory hardlinks. `WalkToken` serializes DFS stack positions for O(1) cursor resume, with key-only fallback as the ground truth. `open_beneath_root` applies `O_NOFOLLOW` at every component of the `openat` traversal, opens with `O_NONBLOCK` to prevent FIFO blocking, and validates via `fstat`. Split hints are provided by an integrated `StreamingSplitEstimator` that is fed during scanning and produces byte-weighted split keys in bounded memory. `is_permanent_io_error` in `common.rs` is shared between filesystem and git connectors. Scanning is orchestrated through `gossip-scanner-runtime`, which uses the `ordered_content` module to drive filesystem scanning.
