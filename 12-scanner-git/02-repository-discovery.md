# "Where Is the Repository?" -- Repository Discovery and Open

*A CI system mounts a shallow clone at `/workspace/repo`. The `.git` entry is a file, not a directory, containing `gitdir: /workspace/repo/.git/worktrees/feature-branch`. The scanner follows the pointer and finds a `commondir` file pointing to `../../..` -- the main worktree's `.git` directory at `/workspace/repo/.git`. It loads `info/alternates`, which references `/shared-objects/chromium.git/objects`. The alternates directory contains 312 pack files totaling 18.7 GB. The scanner attempts to read the config file to detect the object format, but the file is 94 KB -- beyond the 64 KB safety cap. Without bounded reads, the scanner would load the entire file into memory before parsing a single line. This is the repository discovery problem: before scanning a single object, the scanner must resolve an arbitrarily complex directory layout with explicit limits on every file read.*

---

Repository discovery is Stage 1 of the pipeline. It transforms a filesystem path into a `RepoJobState` that contains everything subsequent stages need without additional file opens (except pack files during execution). The design principle: resolve all paths, validate all directories, and load all metadata upfront so later stages operate on validated, in-memory state.

## 1. Repository Layout Detection

Git supports four repository layouts: normal worktrees, bare repositories, linked worktrees, and repositories with shared object stores (alternates). The `GitRepoPaths` struct from `repo.rs` captures the resolved state:

```rust
/// Resolved paths for a Git repository.
///
/// All paths are canonicalized and validated to exist at resolution time,
/// except `pack_dir` which may be missing in an empty repository.
#[derive(Clone, Debug)]
pub struct GitRepoPaths {
    /// Repository kind (worktree or bare).
    pub kind: RepoKind,
    /// Worktree root (where working files live).
    /// `None` for bare repositories.
    pub worktree_root: Option<PathBuf>,
    /// The `.git` directory (or bare repo root).
    /// For linked worktrees, this is the worktree-specific gitdir.
    pub git_dir: PathBuf,
    /// The common directory for shared data.
    /// For normal repos: same as `git_dir`.
    /// For linked worktrees: the main repo's `.git` directory.
    pub common_dir: PathBuf,
    /// The `objects` directory.
    /// Always under `common_dir`.
    pub objects_dir: PathBuf,
    /// The `objects/pack` directory.
    /// May not exist in empty repositories.
    pub pack_dir: PathBuf,
    /// Alternate object directories (from `info/alternates`).
    pub alternate_object_dirs: Vec<PathBuf>,
}
```

The resolution algorithm in `GitRepoPaths::resolve` handles three cases:

```rust
    pub fn resolve<E, L>(repo_root: &Path, limits: &L) -> Result<Self, E>
    where
        E: RepoError,
        L: RepoLimits,
    {
        assert!(
            !repo_root.as_os_str().is_empty(),
            "repo_root cannot be empty"
        );

        let dot_git = repo_root.join(".git");

        match fs::symlink_metadata(&dot_git) {
            Ok(meta) if meta.is_dir() => {
                // Case 1: .git is a directory (normal worktree)
                let git_dir = canonicalize_path(&dot_git)?;
                let worktree_root = Some(canonicalize_path(repo_root)?);
                return Self::from_git_dir(RepoKind::Worktree, worktree_root, git_dir, limits);
            }
            Ok(meta) if meta.is_file() => {
                // Case 2: .git is a file (linked worktree)
                let git_dir = parse_gitdir_file(&dot_git, repo_root, limits)?;
                let worktree_root = Some(canonicalize_path(repo_root)?);
                return Self::from_git_dir(RepoKind::Worktree, worktree_root, git_dir, limits);
            }
            _ => {}
        }

        // Case 3: Bare repository heuristic
        let head = repo_root.join("HEAD");
        let objects = repo_root.join("objects");
        let refs = repo_root.join("refs");
        let config = repo_root.join("config");

        let is_bare = is_file(&head) && is_dir(&objects) && (is_dir(&refs) || is_file(&config));

        if is_bare {
            let git_dir = canonicalize_path(repo_root)?;
            return Self::from_git_dir(RepoKind::Bare, None, git_dir, limits);
        }

        Err(E::not_a_repository())
    }
```

**Case 1** is the common path: `.git` is a directory, the git directory is that directory, and the worktree root is the parent. **Case 2** handles linked worktrees where `.git` is a file containing a `gitdir:` pointer. **Case 3** is the bare repository heuristic -- no `.git` entry, but the root itself contains `HEAD`, `objects`, and either `refs` or `config`.

The resolution uses `symlink_metadata` (not `metadata`) to distinguish files from directories without following symlinks at the `.git` level. Symlinks within the resolved path are followed by `canonicalize_path`.

## 2. Linked Worktrees and the Common Directory

Linked worktrees add indirection: the `.git` file points to a worktree-specific gitdir (e.g., `.git/worktrees/feature-branch`), and that directory contains a `commondir` file pointing back to the shared `.git` directory. The resolution chain:

```rust
fn resolve_common_dir<E, L>(git_dir: &Path, limits: &L) -> Result<PathBuf, E>
where
    E: RepoError,
    L: RepoLimits,
{
    let commondir_file = git_dir.join("commondir");

    if !is_file(&commondir_file) {
        return Ok(git_dir.to_path_buf());
    }

    let bytes = read_bounded_file::<E>(&commondir_file, limits.max_commondir_file_bytes())?;

    let path = parse_commondir_bytes(&bytes).ok_or(E::malformed_commondir_file())?;

    let resolved = resolve_path(git_dir, &path);
    let canonical = canonicalize_path::<E>(&resolved)?;

    if !is_dir(&canonical) {
        return Err(E::common_dir_not_dir());
    }

    Ok(canonical)
}
```

For normal repositories, `common_dir == git_dir`. For linked worktrees, `common_dir` is the main worktree's `.git` directory. The `is_linked_worktree` method makes this distinction explicit:

```rust
    /// Returns true if this is a linked worktree (git_dir != common_dir).
    #[inline]
    #[must_use]
    pub fn is_linked_worktree(&self) -> bool {
        self.git_dir != self.common_dir
    }
```

## 3. Alternates Resolution

The `info/alternates` file lists additional object directories. The parser enforces hard limits:

```rust
fn parse_alternates<E, L>(objects_dir: &Path, limits: &L) -> Result<Vec<PathBuf>, E>
where
    E: RepoError,
    L: RepoLimits,
{
    let alternates_file = objects_dir.join("info").join("alternates");

    if !is_file(&alternates_file) {
        return Ok(Vec::new());
    }

    let bytes = read_bounded_file::<E>(&alternates_file, limits.max_alternates_file_bytes())?;

    let max_count = limits.max_alternates_count() as usize;
    let mut alternates = Vec::with_capacity(max_count.min(8));

    let max_lines = max_count * 2; // Account for comments and blank lines
    let mut line_count = 0;

    for line in bytes.split(|&b| b == b'\n') {
        line_count += 1;
        if line_count > max_lines {
            break;
        }

        let trimmed = line.trim_ascii();
        if trimmed.is_empty() || trimmed.starts_with(b"#") {
            continue;
        }

        if alternates.len() >= max_count {
            break;
        }

        let path = bytes_to_path(trimmed);
        let resolved = resolve_path(objects_dir, &path);

        let canonical = match canonicalize_path::<E>(&resolved) {
            Ok(p) => p,
            Err(_) => return Err(E::alternate_not_dir()),
        };

        if !is_dir(&canonical) {
            return Err(E::alternate_not_dir());
        }

        alternates.push(canonical);
    }

    assert!(alternates.len() <= max_count);

    Ok(alternates)
}
```

The alternates parser enforces three limits: the file size (`max_alternates_file_bytes`), the number of entries (`max_alternates_count`), and the total number of lines processed (`max_count * 2` to account for comments and blanks). Recursive alternates are intentionally not expanded -- only immediate alternates are resolved.

## 4. Bounded File Reads

Every file read in the discovery phase is bounded:

```rust
fn read_bounded_file<E: RepoError>(path: &Path, max_bytes: u32) -> Result<Vec<u8>, E> {
    let file = File::open(path).map_err(E::io)?;
    let metadata = file.metadata().map_err(E::io)?;

    if metadata.len() > max_bytes as u64 {
        return Err(E::file_too_large(metadata.len(), max_bytes));
    }

    let size = metadata.len() as usize;
    let mut buffer = Vec::with_capacity(size);
    let mut take = file.take(max_bytes as u64);
    take.read_to_end(&mut buffer).map_err(E::io)?;

    Ok(buffer)
}
```

The size check via metadata happens first. The `take()` guard prevents reading beyond the limit even if the file grows between the metadata check and the read (TOCTOU defense). The limits trait defines the caps:

```rust
pub trait RepoLimits {
    /// Maximum bytes to read from `.git` file (gitdir pointer).
    fn max_dot_git_file_bytes(&self) -> u32;
    /// Maximum bytes to read from `commondir` file.
    fn max_commondir_file_bytes(&self) -> u32;
    /// Maximum bytes to read from `info/alternates` file.
    fn max_alternates_file_bytes(&self) -> u32;
    /// Maximum number of alternates to accept.
    fn max_alternates_count(&self) -> u8;
}
```

## 5. Object Format Detection

Git repositories may use SHA-1 (20-byte OIDs) or SHA-256 (32-byte OIDs). The format is detected from the Git config:

```rust
fn detect_object_format(
    paths: &GitRepoPaths,
    limits: &RepoOpenLimits,
) -> Result<ObjectFormat, RepoOpenError> {
    for config_path in paths.config_paths() {
        if !config_path.is_file() {
            continue;
        }

        let bytes = read_bounded_file(&config_path, limits.max_config_file_bytes)?;
        let text = std::str::from_utf8(&bytes).map_err(|_| RepoOpenError::InvalidUtf8Config)?;

        for line in text.lines() {
            let line = line.trim();
            if line.starts_with('#') || line.starts_with(';') || line.is_empty() {
                continue;
            }

            let lower = line.to_ascii_lowercase();
            if lower.contains("objectformat") && lower.contains("sha256") {
                return Ok(ObjectFormat::Sha256);
            }
        }

        return Ok(ObjectFormat::Sha1);
    }

    Ok(ObjectFormat::Sha1)
}
```

The detection is a lightweight heuristic scan, not a full Git config parser. It reads the first existing config file, skips comments, and searches for a line containing both `objectformat` and `sha256`. The default is SHA-1, which matches Git's default.

## 6. Start Set Resolution and Watermarks

The start set determines which refs the scanner processes. The `repo_open` function resolves refs via a caller-provided `StartSetResolver`, then loads watermarks from the persistence store:

```rust
pub trait StartSetResolver {
    fn resolve(&self, paths: &GitRepoPaths) -> Result<Vec<(Vec<u8>, OidBytes)>, RepoOpenError>;
}

pub trait RefWatermarkStore {
    fn load_watermarks(
        &self,
        repo_id: u64,
        policy_hash: [u8; 32],
        start_set_id: StartSetId,
        ref_names: &[&[u8]],
    ) -> Result<Vec<Option<OidBytes>>, RepoOpenError>;
}
```

The resolved refs are sorted lexicographically and interned into a `ByteArena`:

```rust
    refs.sort_by(|(a, _), (b, _)| a.as_slice().cmp(b.as_slice()));

    let mut ref_names = ByteArena::with_capacity(limits.max_refname_arena_bytes);
    let mut interned_refs = Vec::with_capacity(refs.len());

    for (name_bytes, tip) in &refs {
        if name_bytes.len() > limits.max_refname_bytes as usize {
            return Err(RepoOpenError::RefNameTooLong {
                len: name_bytes.len(),
                max: limits.max_refname_bytes as usize,
            });
        }

        let name_ref = ref_names
            .intern(name_bytes)
            .ok_or(RepoOpenError::ArenaOverflow)?;

        interned_refs.push((name_ref, *tip));
    }
```

Each ref is paired with its watermark (the last scanned tip OID for that ref, or `None` if never scanned):

```rust
/// A ref in the start set with its resolved tip and optional watermark.
#[derive(Clone, Debug)]
pub struct StartSetRef {
    /// Interned ref name (e.g., `refs/heads/main`).
    pub name: ByteRef,
    /// Resolved commit OID at tip.
    pub tip: OidBytes,
    /// Last scanned tip OID for this ref (if previously scanned).
    pub watermark: Option<OidBytes>,
}
```

The watermark is the key to incremental scanning. When present, the commit walk processes only the half-open range `(watermark, tip]` -- commits reachable from the tip but not reachable from the watermark. We will see in [Chapter 4](04-commit-planning.md) how the two-frontier range walk implements this.

## 7. RepoJobState -- The Complete Stage 1 Output

All discovery results are collected into `RepoJobState`:

```rust
#[derive(Debug)]
pub struct RepoJobState {
    /// Resolved repository paths.
    pub paths: GitRepoPaths,
    /// Object ID format (SHA-1 or SHA-256).
    pub object_format: ObjectFormat,
    /// Paths to artifact files (used for lock-file detection).
    pub artifact_paths: RepoArtifactPaths,
    /// Artifact bytes populated by `artifact_acquire`.
    pub mmaps: RepoArtifactMmaps,
    /// Artifact fingerprints (set by `acquire_midx` from pack/idx metadata).
    pub artifact_fingerprint: Option<RepoArtifactFingerprint>,
    /// Arena for ref name storage.
    pub ref_names: ByteArena,
    /// Start set refs, sorted deterministically by name.
    pub start_set: Vec<StartSetRef>,
}
```

**`artifact_fingerprint`** is `None` after `repo_open` and populated by `acquire_midx` in Stage 2. This is the baseline for concurrent maintenance detection: the scanner compares the current fingerprint against this baseline at two checkpoints during the scan.

**`mmaps.midx`** is also `None` after `repo_open`. The MIDX bytes are built in memory by `acquire_midx` and stored here for the duration of the scan.

The `artifacts_unchanged` method checks both the fingerprint and the presence of lock files:

```rust
    pub fn artifacts_unchanged(&self) -> Result<bool, RepoOpenError> {
        let Some(ref expected) = self.artifact_fingerprint else {
            return Ok(false);
        };

        if has_lock_files(&self.paths, &self.artifact_paths)? {
            return Ok(false);
        }

        let current = RepoArtifactFingerprint::from_pack_dirs(&self.paths)?;
        Ok(&current == expected)
    }
```

Lock files (`.lock` extensions in the pack directory or on artifact paths) indicate that `git gc` or `git maintenance` is actively modifying the repository. The scanner treats any lock file as evidence of concurrent modification.

## 8. Pack and Loose Directory Collection

Subsequent stages need to enumerate pack directories and loose object directories. These helpers from `repo_paths.rs` handle alternates deduplication:

```rust
/// Collect pack directories from the primary and alternate object dirs.
pub fn collect_pack_dirs(paths: &GitRepoPaths) -> Vec<PathBuf> {
    let mut dirs = Vec::with_capacity(1 + paths.alternate_object_dirs.len());
    dirs.push(paths.pack_dir.clone());
    for alternate in &paths.alternate_object_dirs {
        if alternate == &paths.objects_dir {
            continue;
        }
        dirs.push(alternate.join("pack"));
    }
    dirs
}

/// Collect loose object directories from the primary and alternate object dirs.
pub fn collect_loose_dirs(paths: &GitRepoPaths) -> Vec<PathBuf> {
    let mut dirs = Vec::with_capacity(1 + paths.alternate_object_dirs.len());
    dirs.push(paths.objects_dir.clone());
    for alternate in &paths.alternate_object_dirs {
        if alternate == &paths.objects_dir {
            continue;
        }
        dirs.push(alternate.clone());
    }
    dirs
}
```

Both functions skip alternates that equal the main objects directory to avoid duplicate scanning. The primary directory is always first, which matters for pack path resolution where search order determines precedence.

## Summary / What's Next

Repository discovery resolves arbitrarily complex Git layouts (worktrees, bare repos, linked worktrees, alternates) into a single `RepoJobState` with bounded file reads at every step. The state captures resolved paths, the object format, the start set with watermarks, and artifact paths for concurrent maintenance detection.

[Chapter 3](03-artifact-construction.md) covers Stage 2: building the MIDX from `.idx` files via k-way merge and constructing the `CommitGraphMem` from BFS commit loading -- both in memory, without trusting on-disk artifacts.
