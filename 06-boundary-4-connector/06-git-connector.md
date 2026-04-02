# "Untrusted Repositories" -- Git Connector

*A CI pipeline dispatches a secret scanner against 2,400 developer fork
repositories. Repository `evil-fork` contains a tracked file at
`config/../../etc/shadow` -- a path that `git ls-files -z` dutifully
reports as a legitimate tracked entry. The connector joins this path to the
repository root `/data/repos/evil-fork` and calls `fs::canonicalize`. The
resolved absolute path is `/data/etc/shadow`, which no longer starts with
`/data/repos/evil-fork`. Without the containment check, the connector's
`open()` call hands 1,847 bytes of system password hashes to the detection
engine. With the check, the path is silently skipped during indexing. The
entry never appears in the enumeration page. The scanner processes 2,399
repositories normally and produces zero findings from `/etc/shadow`. A
single `starts_with` check on a canonicalized path prevents the scanner
from becoming the breach vector.*

---

## Why a Git Connector Exists

The filesystem connector from Chapter 5 indexes all regular files under a
directory root. But a Git repository is not merely a directory of files --
it has a tracked-file manifest maintained by `git ls-files`. Scanning
untracked files (build artifacts, IDE caches, `.git` internal objects)
wastes time and produces false positives. A Git-specific connector restricts
enumeration to tracked files only, matching what a developer would see with
`git status`.

The `GitConnector` in `crates/gossip-connectors/src/git.rs` (~509 lines)
provides enumeration and read methods. It indexes
tracked files via `git ls-files -z`, produces paginated enumeration pages
from a sorted in-memory snapshot, and serves content reads through
standard filesystem I/O with path traversal defenses.

---

## Connector Identity

```rust
// crates/gossip-contracts/src/connector/mod.rs

/// Connector tag used to domain-separate stable item identity derivation.
pub const GIT_CONNECTOR_TAG: ConnectorTag = ConnectorTag::from_ascii(b"gitlocal");
```

The `"gitlocal"` tag ensures that a file named `README.md` tracked in a
Git repository produces a different `StableItemId` than the same filename
enumerated by the filesystem connector (`"fslocal"`) or any other connector.
Domain separation is enforced by the shared `derive_stable_item_id` helper
in `common.rs`, which hashes `(connector_tag, connector_instance_hash, key)`
via BLAKE3.

---

## The Indexed Entry Type

```rust
/// A single indexed file from `git ls-files`.
#[derive(Debug)]
struct GitEntry {
    /// Repository-relative path used as both the item key and the item ref.
    key: ItemKey,
    /// File size in bytes at index time, used as a size hint for budgeting.
    size_hint: u64,
}
```

Two fields -- the connector keeps the entry lightweight. Identity derivation
and version computation happen at page-assembly time, not at index time.

A key design decision: **item key and item ref share identical bytes.** The
repository-relative path serves as both the enumeration key (for cursor
progression and shard range checks) and the read handle (for `open` and
`read_range`). The `build_git_entry` function validates the same bytes
against both `ItemKey::try_from_slice` and `ItemRef::try_from_slice` at
index time, catching oversized or malformed paths before they enter the
enumeration page.

`GitEntry` implements `common::KeyedEntry`, enabling generic binary search
through the shared utilities in `common.rs`. Split-point selection uses the
shared `common::estimate_split_from_sorted` helper which feeds entries into
a `StreamingSplitEstimator` for byte-weighted median estimation.

---

## The IndexState Lifecycle

The `GitConnector` uses a lazy tri-state indexing lifecycle that latches
permanent failures:

```rust
enum IndexState {
    /// No indexing attempt has been made yet.
    NotIndexed,
    /// Snapshot built successfully; entries are populated and sorted.
    Indexed,
    /// A permanent error occurred during indexing; retries are suppressed.
    Failed(String),
}
```

Both `Indexed` and `Failed` are terminal states. Once the connector
successfully indexes, it never re-shells to `git`. Once a permanent error
is latched, subsequent calls fail fast with a stable diagnostic. Retryable
errors (deadline expiry) leave the state as `NotIndexed`, permitting
re-attempts with fresh budgets.

---

## The Connector Struct

```rust
pub struct GitConnector {
    /// Absolute path to the repository root (working directory).
    repo: PathBuf,
    /// Whether pagination cursors include positional tokens for O(1)
    /// resume. Defaults to `true`; set to `false` for key-only cursors.
    emit_tokens: bool,
    /// Optional upper bound on tracked files. When set, `ensure_indexed`
    /// rejects repositories that exceed this limit with a permanent error.
    max_tracked_files: Option<usize>,
    /// Lazy indexing state machine. See `IndexState`.
    index_state: IndexState,
    /// Sorted snapshot of tracked files, populated by `ensure_indexed`.
    entries: Vec<GitEntry>,
}
```

Five fields. Construction is infallible (`GitConnector::new(repo)`) -- no
I/O occurs. The repo path is asserted non-empty at construction time.
Builder methods `with_tokens(bool)` and `with_max_tracked_files(usize)`
configure behavior before the first enumeration call triggers indexing.

---

## Indexing: `ensure_indexed`

The `ensure_indexed` method is the gateway to all operations. It performs
five steps:

**Step 1: State check.** If already `Indexed`, return `Ok(())`. If
`Failed`, return the memoized error.

**Step 2: Deadline check.** If the deadline has expired before any I/O,
return a retryable error immediately.

**Step 3: Root canonicalization.** `fs::canonicalize(&self.repo)` resolves
the repository path to its real filesystem location. This prevents cwd
drift and ensures all subsequent path joins resolve against the canonical
root. If canonicalization fails, the error is classified: permanent errors
(not found, permission denied) latch `IndexState::Failed`; retryable errors
leave the state as `NotIndexed`.

**Step 4: Git ls-files.** The `list_git_tracked_paths` function shells out
to `git -C <repo> ls-files -z --`, using NUL-delimited output so paths
containing newlines or special characters are handled correctly. A non-zero
exit from `git` is classified as permanent (typically: not a git repository).

```rust
fn list_git_tracked_paths(repo: &Path) -> Result<Vec<Vec<u8>>, EnumerateError> {
    let output = Command::new("git")
        .arg("-C")
        .arg(repo)
        .arg("ls-files")
        .arg("-z")
        .arg("--")
        .output()
        .map_err(|error| classify_io_enumerate_error("git ls-files", repo, &error))?;

    if !output.status.success() {
        let digest = common::path_digest(repo);
        return Err(EnumerateError::permanent(format!(
            "git ls-files failed in ({digest}) (status={:?}): {}",
            output.status.code(),
            String::from_utf8_lossy(&output.stderr).trim()
        )));
    }

    let mut paths = Vec::new();
    for entry in output.stdout.split(|byte| *byte == 0) {
        if entry.is_empty() {
            continue;
        }
        paths.push(entry.to_vec());
    }
    Ok(paths)
}
```

**Step 5: Per-file validation and entry building.** For each tracked path,
the connector applies a three-layer defense against path traversal:

1. **Component filter.** Paths containing `..` components are rejected:

```rust
if rel_path
    .components()
    .any(|c| matches!(c, Component::ParentDir))
{
    continue;
}
```

2. **Canonicalize + containment.** The resolved absolute path must start
   with the canonicalized repo root:

```rust
let canonical_abs = match fs::canonicalize(&abs_path) {
    Ok(p) => p,
    Err(_) => continue,
};
if !canonical_abs.starts_with(&self.repo) {
    continue;
}
```

3. **Symlink rejection.** `symlink_metadata` (not `metadata`) checks the
   entry without following symlinks. Only regular files are indexed:

```rust
let metadata = match fs::symlink_metadata(&abs_path) {
    Ok(m) => m,
    Err(error) => {
        return Err(classify_io_enumerate_error("metadata", &abs_path, &error));
    }
};
if !metadata.is_file() {
    continue;
}
```

After all paths are processed, entries are sorted by key and the state
transitions to `Indexed`.

---

## Version Model

Git does not expose a strong per-file content hash through `ls-files`, so
the connector produces **weak** versions: a BLAKE3 digest over
`(path, file_size, mtime_nanos)`. Two snapshots of the same file will
yield identical versions only when both size and mtime agree. This is
sufficient for change-detection but does not guarantee content identity.

---

## Read Path: Security at Open Time

The `open_path_for_ref` method resolves an `ItemRef` to an absolute
filesystem path with two safety checks:

```rust
fn open_path_for_ref(&mut self, item_ref: &ItemRef) -> Result<PathBuf, ReadError> {
    self.ensure_indexed(None).map_err(enumerate_error_to_read)?;

    let ref_bytes = item_ref.as_bytes();
    self.entries
        .binary_search_by(|entry| entry.key.as_bytes().cmp(ref_bytes))
        .map_err(|_| ReadError::permanent("item_ref not found in index"))?;

    let abs_path = self.repo.join(path_buf_from_bytes(ref_bytes));
    let canonical = fs::canonicalize(&abs_path)
        .map_err(|error| classify_io_read_error("canonicalize", Some(&abs_path), &error))?;
    if !canonical.starts_with(&self.repo) {
        return Err(ReadError::permanent(
            "resolved path escapes repository boundary",
        ));
    }
    Ok(abs_path)
}
```

**Membership verification.** A binary search against the sorted index
confirms the requested ref exists in the snapshot. This prevents reading
files not in the index.

**Containment check.** Even after membership verification, the path is
re-canonicalized and checked against the repo boundary. This catches
read-time escapes where a symlink was created between indexing and reading.

---

## Capability Flags

```rust
impl GitConnector {
    pub fn caps(&self) -> ConnectorCapabilities {
        ConnectorCapabilities {
            seek_by_key: true,
            token_resume: self.emit_tokens,
            range_read: true,
            split_hints: true,
        }
    }
}
```

The full capability set, matching the in-memory and filesystem connectors.
`token_resume` tracks `emit_tokens` so callers can test key-only resume by
constructing with `with_tokens(false)`.

The read methods provide both `open` (returning a
`Box<dyn Read + Send>` wrapping `fs::File`) and `read_range` (using
`file.seek(SeekFrom::Start(offset))` followed by `file.read`). Both methods
delegate to `open_path_for_ref` for path resolution and containment checking.
The `read_range` method includes the standard overflow guard
(`offset.checked_add(dst.len() as u64)`) and budget clamping.

---

## Summary

The `GitConnector` provides a security-hardened, deterministic connector for
Git-tracked repository files. It indexes via `git ls-files -z` with a lazy
tri-state lifecycle (`NotIndexed`, `Indexed`, `Failed`). Path traversal
defense uses a three-layer approach: component filtering (reject `..`),
canonicalize-and-contain (resolved path must start with the canonical repo
root), and symlink rejection (`symlink_metadata`). The connector tag
`"gitlocal"` domain-separates identity from other connector types. Weak
versioning from `(path, size, mtime)` provides change detection without
content hashing. Read-time containment re-canonicalizes
paths before every open to catch symlink escapes created after indexing.

The scanner runtime section (Section 13) covers the connector dispatch
architecture and how these connectors integrate with the runtime.
