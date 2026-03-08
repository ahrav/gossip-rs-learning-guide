# The Half-Open Walk -- Commit Planning and Incremental Scanning

*A repository has three branches: `main` at commit `a3f9e01` (generation 847), `staging` at `b2c8d44` (generation 845), and `hotfix/auth` at `c1e5f67` (generation 843). The previous scan advanced watermarks to `main=7bc4d82` (generation 802), `staging=4fe3a11` (generation 799), and left `hotfix/auth` with no watermark (new branch). The scanner must walk the half-open ranges `(7bc4d82, a3f9e01]`, `(4fe3a11, b2c8d44]`, and `(_, c1e5f67]` -- but `hotfix/auth` branched from `staging` at commit `4fe3a11`, so its entire history up to generation 799 was already scanned. Without ancestry detection, the scanner re-scans 799 commits. With ancestry detection, it recognizes that `c1e5f67` is a descendant of watermark `4fe3a11` and skips the redundant history. This is the incremental scanning problem: extracting exactly the unscanned commits from an overlapping set of refs.*

---

The commit plan determines which commits enter the tree diff stage. Every commit in the plan will have its tree compared against parent trees, producing candidate blobs. The plan must be complete (no unscanned commits are omitted), minimal (no previously-scanned commits are included), and deterministic (identical repository state produces identical output).

## 1. PlannedCommit -- The Output Unit

Each commit in the plan is represented by a `PlannedCommit`. From `commit_walk.rs`:

```rust
/// A planned commit to be processed by tree diffing.
///
/// - If `snapshot_root == true`, this commit should be diffed against the
///   empty tree (snapshot enumeration).
/// - Otherwise, diff against parent trees per merge diff mode.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct PlannedCommit {
    /// Position in the commit-graph.
    pub pos: Position,
    /// If true, diff against the empty tree (snapshot semantics).
    pub snapshot_root: bool,
}
```

The `pos` field is a commit-graph `Position` -- an index into the `CommitGraphMem` arrays. All subsequent operations (tree OID lookup, parent traversal, generation comparison) use this position for O(1) access.

## 2. The CommitGraph Trait

The commit walk operates through a trait that abstracts over both file-backed and in-memory commit graphs:

```rust
/// Commit-graph access required by traversal and ordering.
pub trait CommitGraph {
    /// Total commits in the commit-graph.
    fn num_commits(&self) -> u32;
    /// Lookup a commit OID, returning its position if found.
    fn lookup(&self, oid: &OidBytes) -> Result<Option<Position>, CommitPlanError>;
    /// Returns the generation number for a commit.
    fn generation(&self, pos: Position) -> u32;
    /// Collects parent positions into `scratch`.
    fn collect_parents(
        &self,
        pos: Position,
        max_parents: u32,
        scratch: &mut ParentScratch,
    ) -> Result<(), CommitPlanError>;
    /// Returns the root tree OID for the commit at `pos`.
    fn root_tree_oid(&self, pos: Position) -> Result<OidBytes, CommitPlanError>;
    /// Returns the commit OID for the commit at `pos`.
    fn commit_oid(&self, pos: Position) -> Result<OidBytes, CommitPlanError>;
    /// Returns the committer timestamp (seconds since epoch).
    fn committer_timestamp(&self, pos: Position) -> u64;
}
```

The trait requires deterministic parent iteration order. Parent order affects traversal order, which affects output stability.

## 3. The ParentScratch Buffer

Parent collection uses a scratch buffer that avoids heap allocation for the common case of 1-2 parents:

```rust
/// Scratch buffer for collecting parent positions.
///
/// # Performance
/// - <=16 parents: no heap allocation.
/// - >16 parents: a single spill into `Vec` reused across calls.
#[derive(Debug)]
pub struct ParentScratch {
    inline: [Position; MAX_PARENTS_INLINE],
    inline_len: usize,
    spill: Vec<Position>,
    spilled: bool,
}
```

The inline buffer holds 16 parents (128 bytes). Octopus merges with more than 16 parents spill to a heap `Vec` that is reused across calls. The push method enforces a per-commit parent limit:

```rust
    pub fn push(&mut self, pos: Position, max_parents: u32) -> Result<(), CommitPlanError> {
        if !self.spilled {
            if self.inline_len < MAX_PARENTS_INLINE {
                self.inline[self.inline_len] = pos;
                self.inline_len += 1;
            } else {
                self.spill.clear();
                self.spill
                    .extend_from_slice(&self.inline[..self.inline_len]);
                self.spill.push(pos);
                self.spilled = true;
            }
        } else {
            self.spill.push(pos);
        }

        if self.len() > max_parents as usize {
            return Err(CommitPlanError::TooManyParents {
                count: self.len(),
                max: max_parents as usize,
            });
        }

        Ok(())
    }
```

## 4. The Two-Frontier Range Walk

The core algorithm mirrors `git rev-list <tip> ^<watermark>` using two generation-ordered heaps:

```rust
//! The introduced-by walk mirrors `git rev-list <tip> ^<watermark>` using two
//! generation-ordered heaps: an interesting frontier (commits reachable from
//! `tip`) and an uninteresting frontier (commits reachable from `watermark`).
//! Before emitting the highest-generation interesting commit, the algorithm
//! advances the uninteresting heap down to that generation so any commit
//! reachable from the watermark is marked and excluded.
```

The walker state from `commit_walk.rs`:

```rust
/// Two-frontier range walk state.
struct RefRangeWalker {
    heap_interesting: BinaryHeap<HeapItem>,
    heap_uninteresting: BinaryHeap<HeapItem>,
}
```

Heap items are ordered by generation (max-heap), with position as tiebreaker:

```rust
#[derive(Clone, Copy)]
struct HeapItem {
    gen: u32,
    pos: Position,
}

impl Ord for HeapItem {
    fn cmp(&self, other: &Self) -> Ordering {
        self.gen
            .cmp(&other.gen)
            .then_with(|| self.pos.0.cmp(&other.pos.0))
    }
}
```

The walk advances the uninteresting frontier before each emission to ensure exclusion is complete at the current generation:

```rust
    fn advance_uninteresting(&mut self, target_gen: u32) -> Result<(), CommitPlanError> {
        while let Some(&top_u) = self.walker.heap_uninteresting.peek() {
            if top_u.gen < target_gen {
                break;
            }

            let u = self.walker.heap_uninteresting.pop().unwrap();
            let pos = u.pos;
            self.scratch.set_bit(pos, MARK_U);

            self.cg.collect_parents(
                pos,
                self.limits.max_parents_per_commit,
                &mut self.parent_scratch,
            )?;
            for &p in self.parent_scratch.as_slice() {
                if !self.scratch.has(p, SEEN_U) {
                    self.scratch.set_bit(p, SEEN_U);
                    let gen = self.cg.generation(p);
                    self.walker
                        .heap_uninteresting
                        .push(HeapItem { gen, pos: p });
                }
            }

            if self.walker.current_heap_size() > self.limits.max_heap_entries {
                return Err(CommitPlanError::HeapLimitExceeded {
                    entries: self.walker.current_heap_size(),
                    max: self.limits.max_heap_entries,
                });
            }
        }
        Ok(())
    }
```

The per-ref traversal scratch uses a byte-per-commit array with a touched list for efficient clearing between refs:

```rust
const SEEN_I: u8 = 1 << 0;  // Seen in interesting walk
const SEEN_U: u8 = 1 << 1;  // Seen in uninteresting walk
const MARK_U: u8 = 1 << 2;  // Reachable from watermark (excluded)

struct RefScratch {
    state: Vec<u8>,
    touched: Vec<u32>,
}
```

Clearing only the touched positions between refs is O(touched) instead of O(N), which matters when processing many refs against a large commit graph.

## 5. Cross-Ref Emission Dedup

The `VisitedCommitBitset` deduplicates emission across refs:

```rust
/// Bitset indexed by commit-graph `Position` for cross-ref emission dedup.
pub struct VisitedCommitBitset {
    words: Vec<u64>,
    capacity: u32,
}
```

A commit is emitted at most once across all refs. The `test_and_set` method returns true only if the bit was newly set:

```rust
    pub fn test_and_set(&mut self, pos: Position) -> bool {
        let i = pos.0 as usize;
        let word = i >> 6;
        let bit = i & 63;
        let mask = 1u64 << bit;
        let was_unset = (self.words[word] & mask) == 0;
        self.words[word] |= mask;
        was_unset
    }
```

## 6. New-Ref Skip Optimization

When a ref has no watermark (new branch), the entire history is a candidate. But if that branch was forked from a ref that does have a watermark, the shared history was already scanned. The walker detects this via ancestry checks:

```rust
        if r.watermark.is_none() {
            let mut checks = 0u32;
            for other in self.refs.iter() {
                if checks >= self.limits.max_new_ref_skip_checks {
                    break;
                }
                let Some(wm_oid) = other.watermark else {
                    continue;
                };
                let Some(wm_pos) = self.cg.lookup(&wm_oid)? else {
                    continue;
                };
                checks += 1;

                if is_ancestor(
                    self.cg,
                    tip_pos,
                    wm_pos,
                    &mut self.scratch,
                    &mut self.ancestor_stack,
                    &mut self.parent_scratch,
                    self.limits.max_parents_per_commit,
                )? {
                    self.cur_tip = None;
                    self.cur_wm = None;
                    return self.init_next_ref();
                }
            }
        }
```

The check count is bounded to avoid O(refs^2) behavior. If the tip of the new ref is an ancestor of another ref's watermark, the entire history was already scanned, and the ref is skipped entirely.

## 7. Watermark Ancestry Validation

A watermark that is not an ancestor of the current tip indicates history rewriting (force push). The walker handles this by ignoring the invalid watermark and falling back to a full-history walk:

```rust
        let mut wm_pos_opt: Option<Position> = None;
        if let Some(wm_oid) = r.watermark {
            if let Some(wm_pos) = self.cg.lookup(&wm_oid)? {
                if is_ancestor(
                    self.cg,
                    wm_pos,
                    tip_pos,
                    &mut self.scratch,
                    &mut self.ancestor_stack,
                    &mut self.parent_scratch,
                    self.limits.max_parents_per_commit,
                )? {
                    wm_pos_opt = Some(wm_pos);
                }
            }
        }
```

This is correct behavior: if someone force-pushed to a branch, the previously-scanned commits may no longer be reachable. The scanner falls back to scanning all reachable commits rather than potentially missing content.

## 8. Topological Ordering

After collecting all interesting commits, the plan is sorted into topological order using Kahn's algorithm:

```rust
/// Collects introduced-by commits and returns them in topological order.
pub fn introduced_by_plan<CG: CommitGraph>(
    repo: &RepoJobState,
    cg: &CG,
    limits: CommitWalkLimits,
) -> Result<Vec<PlannedCommit>, CommitPlanError> {
    let iter = CommitPlanIter::new(repo, cg, limits)?;
    let mut positions = Vec::new();
    for item in iter {
        positions.push(item?.pos);
    }

    let ordered = topo_order_positions(cg, &positions, limits)?;
    Ok(ordered
        .into_iter()
        .map(|pos| PlannedCommit {
            pos,
            snapshot_root: false,
        })
        .collect())
}
```

Topological ordering guarantees ancestors appear before descendants. This enables "first-introduction" semantics: a blob is attributed to the earliest commit that introduces it, because that commit is processed first. The ordering uses dense arrays and a ring buffer queue:

```rust
/// Fixed-capacity ring buffer queue.
struct PosQueue {
    buf: Vec<Position>,
    head: usize,
    len: usize,
}
```

The algorithm runs in O(V + E) where V is the number of commits in the plan and E is the number of parent edges between them. Cycle detection is built in: if `ordered.len() != total`, a cycle exists and the function returns `TopoSortCycle`.

## 9. Memory Model

The commit walk allocates three structures per scan:

- `VisitedCommitBitset`: 1 bit per commit in the graph (for a 4.2M commit repo, 512 KB)
- `RefScratch`: 1 byte per commit plus a touched list (4.2 MB + variable)
- Heap frontiers: bounded by `CommitWalkLimits::max_heap_entries`

The `ParentScratch` is reused across all calls without per-call allocation for typical commits. Only octopus merges with more than 16 parents trigger a heap spill.

## Summary / What's Next

The commit plan extracts exactly the unscanned commits from an overlapping set of refs using a two-frontier generation-ordered walk. Watermarks enable incremental scanning; ancestry checks prevent redundant re-scanning of shared history; topological ordering ensures first-introduction semantics.

[Chapter 5](05-tree-diff-and-candidates.md) takes this plan and walks each commit's trees, comparing parent and child trees to extract the specific blobs that changed -- the candidates that will eventually be decoded and scanned.
