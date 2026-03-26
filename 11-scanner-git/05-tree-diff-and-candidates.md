# Changed Blobs Only -- Tree Diff and Candidate Extraction

*Commit `a3f9e01` modifies file `src/auth/tokens.rs` and adds directory `src/auth/providers/`. The parent commit `7bc4d82` has a root tree with 14,000 entries across 892 subtrees. The scanner diffs the two root trees. At depth 0, only the `src/` subtree OID differs -- the other 13,107 entries are identical and skipped without loading their subtrees. At depth 1, only `auth/` differs. At depth 2, the diff finds the modified blob `tokens.rs` (OID `e7d9112`, changed from `b3a1c04`) and the new subtree `providers/`. It recurses into `providers/`, finds blob `oauth.rs` (OID `d4e2f89`), and emits two candidates: a `Modify` for `tokens.rs` and an `Add` for `providers/oauth.rs`. Total tree bytes loaded: 4,218 (three subtrees). Total tree bytes in the repository: 3.7 MB. Without OID-only comparison, the scanner would load and parse all 3.7 MB. This is the tree diff optimization: unchanged subtrees are never loaded.*

---

Tree diffing is the mechanism that turns a commit plan into a set of candidate blobs. For each planned commit, the scanner compares the commit's root tree against its parent's root tree (or the empty tree for snapshot roots), emitting candidates for every blob that was added or modified. The diff operates on OIDs only -- no blob content is read during this stage.

## 1. The Tree Entry Format

Git tree objects contain entries in a specific binary format. From `tree_entry.rs`:

```rust
//! A tree object contains zero or more entries, each with format:
//! ```text
//! <mode> SP <name> NUL <oid>
//! ```
//!
//! Where:
//! - `<mode>`: ASCII octal digits (e.g., "100644", "40000")
//! - `SP`: Single space byte (0x20)
//! - `<name>`: Entry name bytes (non-empty, no slashes, no NUL)
//! - `NUL`: Single NUL byte (0x00)
//! - `<oid>`: Raw OID bytes (20 for SHA-1, 32 for SHA-256)
```

Entry types are classified by mode:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum EntryKind {
    /// Subdirectory (mode 040000).
    Tree,
    /// Regular file (mode 100644 or similar without execute bit).
    RegularFile,
    /// Executable file (mode 100755 or similar with execute bit).
    ExecutableFile,
    /// Symbolic link (mode 120000).
    Symlink,
    /// Gitlink/submodule (mode 160000).
    Gitlink,
    /// Unknown or invalid mode.
    Unknown,
}
```

The classification uses mask-based logic for robustness with historical non-canonical modes:

```rust
fn classify_mode(mode: u32) -> EntryKind {
    const S_IFMT: u32 = 0o170000;

    match mode & S_IFMT {
        0o040000 => EntryKind::Tree,
        0o120000 => EntryKind::Symlink,
        0o160000 => EntryKind::Gitlink,
        0o100000 => {
            if (mode & 0o100) != 0 {
                EntryKind::ExecutableFile
            } else {
                EntryKind::RegularFile
            }
        }
        _ => EntryKind::Unknown,
    }
}
```

Older Git versions and third-party tools created non-canonical modes like `100664` or `100600`. The mask-based approach handles them by examining type bits and the executable bit rather than requiring exact mode matches.

## 2. Git Tree Sort Ordering

Tree entries are stored in a specific order that differs from lexicographic: directories are sorted as if their name ends with `/`. This means file `foo` (terminated with NUL, 0x00) sorts before directory `foo/` (terminated with `/`, 0x2F). The diff algorithm must compare entries in this order:

```rust
//! - file "foo" terminates with NUL (0x00)
//! - dir "foo/" terminates with '/' (0x2F)
//! - So file < dir when names match exactly
```

This ordering is critical for the merge-walk algorithm. If the scanner used lexicographic comparison instead of Git tree ordering, it would misalign entries and produce incorrect diffs.

## 3. The TreeDiffWalker

The walker maintains state across calls for efficiency. From `tree_diff.rs`:

```rust
/// Tree diff walker configuration and state.
pub struct TreeDiffWalker {
    /// Maximum recursion depth.
    max_depth: u16,
    /// OID length (20 or 32).
    oid_len: u8,
    /// Path buffer (reused across entries).
    path_buf: Vec<u8>,
    /// Reused entry-name scratch to avoid per-diff allocations.
    name_scratch: Vec<u8>,
    /// Diff stack (reused across calls).
    stack: Vec<DiffFrame>,
    /// Statistics.
    stats: TreeDiffStats,
    /// In-flight tree bytes budget.
    tree_bytes_in_flight_limit: u64,
    /// Current in-flight tree bytes retained by stack frames.
    tree_bytes_in_flight: u64,
    /// Threshold for switching to streaming parsing.
    stream_threshold: usize,
}
```

The stack, path buffer, and name scratch are all reused across `diff_trees` calls. The stack replaces recursion with an explicit iterative loop, making depth limits easy to enforce.

## 4. The Merge-Walk Algorithm

The diff compares two trees entry by entry in Git tree order. For each pair:

```rust
//! - new < old: Entry added in new tree -> emit candidate (if blob-like)
//! - new > old: Entry deleted -> skip (nothing to scan)
//! - new == old:
//!   - If tree: recurse only if subtree OIDs differ
//!   - If blob-like: emit candidate only if OIDs differ (modify)
//!   - If gitlink: skip (submodule commits not scanned)
```

The action computation from `compute_action`:

```rust
fn compute_action(
    new_entry: Option<TreeEntry<'_>>,
    old_entry: Option<TreeEntry<'_>>,
    oid_len: u8,
) -> Result<Action, TreeDiffError> {
    match (new_entry, old_entry) {
        (None, None) => Ok(Action::Pop),

        (Some(new_ent), None) => {
            let oid = convert_oid(new_ent.oid_bytes, oid_len)?;
            Ok(Action::AddedEntry {
                oid,
                kind: new_ent.kind,
                mode: new_ent.mode,
            })
        }

        (None, Some(_)) => Ok(Action::DeletedEntry),

        (Some(new_ent), Some(old_ent)) => {
            let cmp = git_tree_name_cmp(
                new_ent.name,
                new_ent.kind.is_tree(),
                old_ent.name,
                old_ent.kind.is_tree(),
            );

            match cmp {
                Ordering::Less => {
                    let oid = convert_oid(new_ent.oid_bytes, oid_len)?;
                    Ok(Action::NewBeforeOld {
                        oid,
                        kind: new_ent.kind,
                        mode: new_ent.mode,
                    })
                }
                Ordering::Greater => Ok(Action::OldBeforeNew),
                Ordering::Equal => {
                    let new_oid = convert_oid(new_ent.oid_bytes, oid_len)?;
                    let old_oid = convert_oid(old_ent.oid_bytes, oid_len)?;
                    Ok(Action::MatchedEntries {
                        new_oid,
                        old_oid,
                        new_kind: new_ent.kind,
                        old_kind: old_ent.kind,
                        new_mode: new_ent.mode,
                    })
                }
            }
        }
    }
}
```

The matched-entries handler implements the four-way kind matrix:

```rust
    fn handle_matched_entries<S: TreeSource, C: CandidateSink>(
        &mut self,
        // ... parameters ...
    ) -> Result<(), TreeDiffError> {
        if new_oid == old_oid && new_kind.is_tree() && old_kind.is_tree() {
            perf_stats::sat_add_u64(&mut self.stats.subtrees_skipped, 1);
            return Ok(());
        }

        match (new_kind.is_tree(), old_kind.is_tree()) {
            (true, true) => {
                self.push_subtree_frame(source, new_oid, Some(old_oid), name)?;
            }
            (true, false) => {
                self.push_subtree_frame(source, new_oid, None, name)?;
            }
            (false, true) => {
                if new_kind.is_blob_like() {
                    self.emit_candidate(
                        candidates, new_oid, name, ChangeKind::Add,
                        new_mode, commit_id, parent_idx,
                    )?;
                }
            }
            (false, false) => {
                if new_kind.is_blob_like() {
                    if old_kind.is_blob_like() {
                        if new_oid != old_oid {
                            self.emit_candidate(
                                candidates, new_oid, name, ChangeKind::Modify,
                                new_mode, commit_id, parent_idx,
                            )?;
                        }
                    } else {
                        self.emit_candidate(
                            candidates, new_oid, name, ChangeKind::Add,
                            new_mode, commit_id, parent_idx,
                        )?;
                    }
                }
            }
        }

        Ok(())
    }
```

The critical optimization is the identical-subtree skip: when both entries are trees and their OIDs match, the entire subtree is skipped without loading it. For a 14,000-entry root tree where only one subtree changed, this eliminates loading and parsing of the other 13,999 entries and all their descendants.

## 5. In-Flight Tree Byte Budget

The walker tracks how many tree bytes are currently retained in stack frames and enforces a budget:

```rust
    fn load_tree_cursor<S: TreeSource>(
        &mut self,
        source: &mut S,
        oid: Option<&OidBytes>,
    ) -> Result<TreeCursor, TreeDiffError> {
        if let Some(oid) = oid {
            let bytes = source.load_tree(oid)?;
            let in_flight_len = bytes.in_flight_len() as u64;
            let new_in_flight = self.tree_bytes_in_flight.saturating_add(in_flight_len);

            if new_in_flight > self.tree_bytes_in_flight_limit {
                return Err(TreeDiffError::TreeBytesBudgetExceeded {
                    loaded: new_in_flight,
                    budget: self.tree_bytes_in_flight_limit,
                });
            }
            self.tree_bytes_in_flight = new_in_flight;

            Ok(TreeCursor::new(bytes, self.oid_len, self.stream_threshold))
        } else {
            Ok(TreeCursor::empty(self.oid_len))
        }
    }
```

Budget errors release all in-flight bytes on cleanup, enabling retries on the same walker:

```rust
    fn cleanup_after_diff_call(&mut self) {
        while let Some(frame) = self.stack.pop() {
            self.release_tree_bytes(frame.new_cursor.in_flight_len());
            self.release_tree_bytes(frame.old_cursor.in_flight_len());
        }

        self.path_buf.clear();
        self.tree_bytes_in_flight = 0;
    }
```

## 6. Buffered vs Streaming Cursors

Small trees are parsed from a single buffered slice. Large or spill-backed trees use streaming parsing to avoid loading the entire tree into memory:

```rust
enum TreeCursor {
    Buffered(BufferedCursor),
    Stream(TreeStream<TreeBytesReader>),
}

impl TreeCursor {
    fn new(bytes: TreeBytes, oid_len: u8, stream_threshold: usize) -> Self {
        let use_stream = bytes.len() >= stream_threshold || matches!(bytes, TreeBytes::Spilled(_));
        if use_stream {
            let reader = TreeBytesReader::new(bytes);
            let stream = TreeStream::new(reader, oid_len, TREE_STREAM_BUF_BYTES);
            Self::Stream(stream)
        } else {
            Self::Buffered(BufferedCursor::new(bytes, oid_len))
        }
    }
}
```

The streaming threshold defaults to `max_tree_cache_bytes` -- trees larger than the cache are streamed to keep the working set bounded. The stream buffer is 16 KB, which is large enough to hold many entries but small enough to avoid unnecessary memory pressure.

## 7. The BlobIntroducer (ODB-Blob Mode)

In ODB-blob mode, the `BlobIntroducer` replaces tree diffing. It traverses trees in topological commit order and emits each blob the first time it appears, using seen-set bitmaps keyed by MIDX index. From `blob_introducer.rs`:

```rust
/// Seen-set bitmaps for trees and blobs keyed by MIDX index.
///
/// Three independent bitsets are maintained:
/// - `trees` — tracks visited tree objects to skip entire subtrees.
/// - `blobs` — tracks emitted blob candidates (non-excluded).
/// - `blobs_excluded` — tracks blobs matched by path-exclusion policy.
#[derive(Debug)]
pub struct SeenSets {
    trees: DynamicBitSet,
    blobs: DynamicBitSet,
    blobs_excluded: DynamicBitSet,
}
```

The three-bitmap design prevents a subtle bug: if exclusion shared the `blobs` set, a blob first encountered under an excluded path would be marked "seen" and suppressed when later encountered under a non-excluded path. Separate tracking ensures the blob set is identical to the serial path regardless of encounter order.

In parallel mode, an `AtomicSeenSets` variant uses `fetch_or` for lock-free deduplication across worker threads. Each tree/blob is claimed by exactly one worker via the atomic test-and-set, matching a GC mark-phase architecture.

## 8. Candidate Context

Each emitted candidate carries context for attribution and deduplication:

```rust
    fn emit_candidate<C: CandidateSink>(
        &mut self,
        candidates: &mut C,
        oid: &OidBytes,
        name: &[u8],
        change_kind: ChangeKind,
        mode: u32,
        commit_id: u32,
        parent_idx: u8,
    ) -> Result<(), TreeDiffError> {
        let full_len = self.path_buf.len() + name.len();
        if full_len > MAX_PATH_LEN {
            return Err(TreeDiffError::PathTooLong {
                len: full_len,
                max: MAX_PATH_LEN,
            });
        }

        let path_start = self.path_buf.len();
        self.path_buf.extend_from_slice(name);

        let mode_u16 = mode as u16;
        let cand_flags = 0;
        candidates.emit(
            *oid,
            &self.path_buf,
            commit_id,
            parent_idx,
            change_kind,
            mode_u16,
            cand_flags,
        )?;

        self.path_buf.truncate(path_start);

        Ok(())
    }
```

The path is assembled in a reusable buffer and includes the full directory path (e.g., `src/auth/providers/oauth.rs`). The `CandidateSink` must copy the path if it needs to retain it, because the buffer is reused on the next emission.

## 9. Statistics

The walker tracks cumulative statistics:

```rust
#[derive(Clone, Debug, Default)]
pub struct TreeDiffStats {
    /// Number of trees loaded.
    pub trees_loaded: u64,
    /// Total bytes loaded from trees.
    pub tree_bytes_loaded: u64,
    /// Peak in-flight tree bytes retained by the walker.
    pub tree_bytes_in_flight_peak: u64,
    /// Number of candidates emitted.
    pub candidates_emitted: u64,
    /// Number of subtrees skipped (same OID).
    pub subtrees_skipped: u64,
    /// Maximum stack depth reached.
    pub max_depth_reached: u16,
}
```

The `subtrees_skipped` counter quantifies the OID-only optimization. For a typical incremental scan where one file changed, this number is large (thousands of skipped subtrees) while `trees_loaded` is small (a handful of changed subtrees).

## Summary / What's Next

Tree diffing extracts candidate blobs by comparing parent and child trees in Git tree order, skipping unchanged subtrees entirely. The ODB-blob alternative traverses all reachable trees and emits each unique blob once using seen-set bitmaps. Both paths produce candidate blobs with context (commit, path, change kind).

[Chapter 6](06-spill-and-dedup.md) takes these candidates and deduplicates them: the same blob OID may appear in multiple commits and multiple refs. The spill pipeline handles repositories with millions of candidates without exceeding memory limits.
