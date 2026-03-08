# "The Overlapping Chunks" -- Bulk Splits and Manifest Validation

*A connector registers a manifest covering rows 0 through 10,000 in manifest 7. The builder chunks it into 100-row shards using `split_manifest_by_rows`. Shard 47 covers rows `[4,600, 4,700)`. Shard 48 covers rows `[4,699, 4,800)` -- an off-by-one in the boundary calculation gives it a start row one less than shard 47's end row. Row 4,699 now belongs to two shards. Individual shards pass per-entry validation at add time: each has a well-formed half-open interval with start strictly less than end. Only when `build_inputs` runs `validate_manifest` does the overlap surface -- after all 100 shards have consumed arena space. The registration fails, but the arena memory is not recovered until the builder is dropped. Two workers later acquire the overlapping shards from a different registration attempt. Both scan row 4,699. Both report a finding for the API key in that row. The deduplication layer downstream catches the duplicate, but the scan's resource consumption doubled for the overlapping region, and the incident report flags a manifest integrity violation that requires a full re-scan.*

---

The bulk split helpers in `PreallocShardBuilder` exist to generate contiguous, non-overlapping child shards from a single parent range. The opening scenario describes exactly the failure they prevent: an off-by-one in chunking arithmetic produces an overlap that per-entry validation cannot detect. Only manifest-level validation -- which checks all entries together -- catches cross-entry violations like overlaps, gaps, and duplicate IDs.

This chapter covers three mechanisms: `split_range_by_boundaries` for byte-range splits, `split_manifest_by_rows` for fixed-width row chunking, and `build_inputs` for final manifest validation. Together with the add methods from Chapter 5, these complete the builder's public API.

The distinction between these two split strategies reflects two fundamentally different data models. Byte-range splits divide a continuous key space at arbitrary boundaries -- the caller provides the exact split points (perhaps computed by `byte_midpoint` from Chapter 2). Manifest row splits divide a discrete row space at regular intervals -- the caller provides a chunk width and the builder computes boundaries arithmetically. Both strategies produce the same output: a sequence of staged `BuilderEntry` values with contiguous, non-overlapping key ranges.

## 1. `split_range_by_boundaries` -- Two-Pass Range Splitting

This method takes a parent byte range `[start, end)`, a set of interior split points, and produces one child per inter-boundary interval. For example, `start=a`, `split_points=[m, t]`, `end=z` yields three children: `[a,m)`, `[m,t)`, `[t,z)`. The split points must be strictly increasing and interior to the parent range -- the builder rejects duplicate, unsorted, or out-of-range points.

Here is the complete implementation from `builder.rs`:

```rust
    /// Add contiguous range children for a split parent in boundary order.
    ///
    /// `split_points` are interior boundaries between `start` and `end`.
    /// For example, `start=a`, `split_points=[m, t]`, `end=z` yields:
    /// `[a,m)`, `[m,t)`, `[t,z)`.
    ///
    /// Boundaries must be strictly increasing and interior to `[start, end)`.
    /// Duplicate, unsorted, or out-of-range points are rejected as
    /// [`PreallocShardBuilderError::RangeInvalid`].
    ///
    /// Capacity is checked up front for all children to prevent partial writes
    /// on entry-limit exhaustion.
    ///
    /// All children are staged with [`CursorUpdate::initial`].
    ///
    /// This method is not transactional for slab exhaustion: if a later child
    /// allocation returns [`PreallocShardBuilderError::SlabFull`], previously
    /// appended children remain staged.
    ///
    /// # Errors
    ///
    /// - [`PreallocShardBuilderError::FanOutExceeded`] if child count exceeds
    ///   [`MAX_SPLIT_CHILDREN`].
    /// - [`PreallocShardBuilderError::CapacityExceeded`] if all children would
    ///   not fit in the remaining entry budget.
    /// - [`PreallocShardBuilderError::RangeInvalid`] if any child range is
    ///   invalid.
    /// - [`PreallocShardBuilderError::SlabFull`] if arena storage is exhausted.
    pub fn split_range_by_boundaries(
        &mut self,
        start: &[u8],
        split_points: &[&[u8]],
        end: &[u8],
        connector_extra: &[u8],
    ) -> Result<(), PreallocShardBuilderError> {
        let additional = split_points.len().saturating_add(1);
        self.ensure_bulk_split_capacity(additional)?;

        // Pass 1 (validation-only): call `range_shard_ref` which validates
        // bounds without allocating arena storage. This catches inverted,
        // duplicate, or out-of-order boundaries before any entries are staged,
        // preserving the builder's state on boundary errors.
        let mut child_start = start;
        for &child_end in split_points {
            range_shard_ref(child_start, child_end, connector_extra, &mut self.scratch)
                .map_err(PreallocShardBuilderError::RangeInvalid)?;
            child_start = child_end;
        }
        range_shard_ref(child_start, end, connector_extra, &mut self.scratch)
            .map_err(PreallocShardBuilderError::RangeInvalid)?;

        // Pass 2 (allocate-and-stage): re-derive each child range and commit
        // it to the arena. SlabFull errors here leave a partial prefix staged.
        let mut child_start = start;
        for &child_end in split_points {
            let handle = range_shard_into(
                self.arena,
                child_start,
                child_end,
                connector_extra,
                &mut self.scratch,
            )
            .map_err(|err| match err {
                ShardIntoError::Build(err) => PreallocShardBuilderError::RangeInvalid(err),
                ShardIntoError::SlabFull(err) => PreallocShardBuilderError::SlabFull(err),
            })?;
            self.push_entry(handle, CursorUpdate::initial());
            child_start = child_end;
        }
        let handle = range_shard_into(
            self.arena,
            child_start,
            end,
            connector_extra,
            &mut self.scratch,
        )
        .map_err(|err| match err {
            ShardIntoError::Build(err) => PreallocShardBuilderError::RangeInvalid(err),
            ShardIntoError::SlabFull(err) => PreallocShardBuilderError::SlabFull(err),
        })?;
        self.push_entry(handle, CursorUpdate::initial());
        Ok(())
    }
```

### The Two-Pass Design

The two-pass approach is the key design decision. Understanding why it exists requires considering the failure modes of a naive single-pass implementation.

A single-pass approach would validate and allocate in one loop: for each child, validate the range, allocate arena storage, and stage the entry. If the third child of five has an inverted boundary, three children are already staged. The builder is left in a partially-populated state that the caller must clean up.

The two-pass approach separates validation from allocation:

```text
Pass 1 (read-only):
  for each child boundary pair:
    range_shard_ref(start, end, ...)   -- validate only, no arena allocation
    if invalid: return Err            -- builder state unchanged

Pass 2 (allocate-and-stage):
  for each child boundary pair:
    range_shard_into(arena, start, end, ...)  -- allocate and stage
    push_entry(handle, initial_cursor)
```

Pass 1 calls `range_shard_ref` -- the validation-only function from the hint module (discussed in Chapter 3) that checks key bounds without allocating arena storage. If any boundary pair is invalid (inverted, duplicate, or out-of-order), the function returns immediately with `RangeInvalid`. No entries are staged. No arena bytes are consumed. The builder's state is completely unchanged.

Pass 2 only runs after all boundaries pass validation. It calls `range_shard_into` to allocate and stage each child. At this point, the only remaining failure mode is `SlabFull` -- arena exhaustion. The doc comment is explicit about this asymmetry: "boundary errors leave the builder state completely unchanged" but "if arena slab allocation fails mid-loop, previously appended children remain staged."

The two-pass design is inspired by the same principle behind database two-phase commit: separate the "can we do this?" decision from the "do it" action. In database terms, Pass 1 is the prepare phase (vote to commit or abort) and Pass 2 is the commit phase (apply the changes). The key insight is that Pass 1's validation cost -- iterating through split points and calling `range_shard_ref` -- is trivially cheap compared to the arena allocation in Pass 2. The overhead of two passes is negligible, but the safety benefit (complete rollback on boundary errors) is significant.

Note the distinction between `range_shard_ref` (Pass 1) and `range_shard_into` (Pass 2). These are two functions from the hint module with identical validation logic but different side effects. `range_shard_ref` validates bounds and returns a borrowed `ShardSpecRef` without allocating arena storage. `range_shard_into` validates bounds, encodes the spec, and allocates a slab slot in the arena. Pass 1 uses the former because it needs validation without commitment; Pass 2 uses the latter because it needs both.

### Up-Front Capacity Checks

Before either pass runs, the method checks both fan-out and entry-budget limits:

```rust
let additional = split_points.len().saturating_add(1);
self.ensure_bulk_split_capacity(additional)?;
```

The child count is `split_points.len() + 1` (N split points produce N+1 intervals). The `saturating_add` prevents wrapping on adversarially large `split_points` slices. `ensure_bulk_split_capacity` first checks `MAX_SPLIT_CHILDREN` (the per-call fan-out cap of 256), then delegates to `ensure_entry_capacity` for the remaining entry budget. Both checks happen before any allocation, so a capacity-exceeded error leaves the builder state unchanged.

## 2. `split_manifest_by_rows` -- Fixed-Width Row Chunking

Manifest shards represent row ranges within a logical manifest (a database table, a file index, etc.). Splitting a manifest shard into fixed-width row chunks is a common operation at registration time: the connector knows the manifest has 10,000 rows and wants 100-row shards.

### The Split Count Helper

The chunk count is computed by a helper that handles ceiling division and overflow:

```rust
    /// Compute the number of fixed-width row chunks in `[start_row, end_row)`.
    ///
    /// Returns the ceiling division `(end_row - start_row) / rows_per_shard`,
    /// clamped to `usize::MAX` on conversion overflow. The clamped value is
    /// safe because callers pass it to [`Self::ensure_bulk_split_capacity`],
    /// which rejects anything above [`MAX_SPLIT_CHILDREN`].
    ///
    /// Rejects empty ranges (`start_row >= end_row`) and zero chunk width.
    fn manifest_split_count(
        start_row: u64,
        end_row: u64,
        rows_per_shard: u64,
    ) -> Result<usize, PreallocShardBuilderError> {
        if start_row >= end_row || rows_per_shard == 0 {
            return Err(Self::manifest_split_invalid_bounds());
        }
        let span = end_row
            .checked_sub(start_row)
            .ok_or_else(Self::manifest_split_invalid_bounds)?;
        let whole_chunks = span / rows_per_shard;
        let has_tail = span % rows_per_shard != 0;
        let shard_count = whole_chunks.saturating_add(u64::from(has_tail));
        Ok(usize::try_from(shard_count).unwrap_or(usize::MAX))
    }
```

The `unwrap_or(usize::MAX)` clamp handles the edge case where the chunk count exceeds `usize::MAX` on 32-bit targets. The clamped value flows into `ensure_bulk_split_capacity`, which rejects anything above `MAX_SPLIT_CHILDREN` (256). An astronomically large chunk count produces `FanOutExceeded`, not a panic.

A concrete example illustrates the chunking arithmetic:

```text
start_row = 0, end_row = 10, rows_per_shard = 4

span        = 10 - 0 = 10
whole_chunks = 10 / 4 = 2
has_tail    = 10 % 4 != 0 = true
shard_count = 2 + 1 = 3

Chunks: [0, 4), [4, 8), [8, 10)
```

### The Chunking Loop

```rust
    /// Add contiguous manifest-row children by chunking `[start_row, end_row)`.
    ///
    /// Splits the parent interval into fixed-width row chunks of
    /// `rows_per_shard`, with a final short tail chunk when needed.
    /// Capacity is checked up front for all derived children to prevent
    /// partial writes on entry-limit exhaustion.
    ///
    /// Chunk arithmetic uses saturating addition and end-clamping, so ranges
    /// near `u64::MAX` cannot wrap. All children are staged with
    /// [`CursorUpdate::initial`].
    ///
    /// This method is not transactional for slab exhaustion: if a later child
    /// allocation returns [`PreallocShardBuilderError::SlabFull`], previously
    /// appended children remain staged.
    pub fn split_manifest_by_rows(
        &mut self,
        manifest_id: u64,
        start_row: u64,
        end_row: u64,
        rows_per_shard: u64,
        connector_extra: &[u8],
    ) -> Result<(), PreallocShardBuilderError> {
        let additional = Self::manifest_split_count(start_row, end_row, rows_per_shard)?;
        self.ensure_bulk_split_capacity(additional)?;

        let mut chunk_start = start_row;
        while chunk_start < end_row {
            // Saturating add avoids wrapping on large row IDs, then clamps to
            // the parent end bound to preserve half-open tiling semantics.
            // Progress is guaranteed: rows_per_shard > 0 (checked by
            // manifest_split_count) and chunk_start < end_row (loop guard),
            // so chunk_end is always strictly greater than chunk_start.
            let chunk_end = chunk_start.saturating_add(rows_per_shard).min(end_row);
            debug_assert!(
                chunk_end > chunk_start,
                "split_manifest_by_rows: non-progressing chunk"
            );
            let handle = manifest_shard_into(
                self.arena,
                manifest_id,
                chunk_start,
                chunk_end,
                connector_extra,
                &mut self.scratch,
            )
            .map_err(|err| match err {
                ShardIntoError::Build(err) => PreallocShardBuilderError::ManifestCtorInvalid(err),
                ShardIntoError::SlabFull(err) => PreallocShardBuilderError::SlabFull(err),
            })?;
            self.push_entry(handle, CursorUpdate::initial());
            chunk_start = chunk_end;
        }

        Ok(())
    }
```

The arithmetic is carefully written to avoid wrapping near `u64::MAX`:

- `chunk_start.saturating_add(rows_per_shard)` prevents overflow when `chunk_start` is close to `u64::MAX`. Instead of wrapping to a small value, it clamps to `u64::MAX`.
- `.min(end_row)` then clamps the result to the parent's end bound, ensuring the last chunk does not extend beyond the parent.
- The `debug_assert!` checks the progress invariant: since `rows_per_shard > 0` (validated by `manifest_split_count`) and `chunk_start < end_row` (the loop guard), `chunk_end` is always strictly greater than `chunk_start`. The loop always terminates.

Unlike `split_range_by_boundaries`, this method does not use a two-pass approach. The reason is that manifest row boundaries are computed arithmetically (by addition and clamping), not validated against external input. The boundaries cannot be inverted or out-of-order because the chunking arithmetic guarantees monotonic progress. The only failure mode after the capacity check is `SlabFull`, which has the same partial-prefix semantics as the range split helper.

A visual example shows the tiling for a concrete case:

```text
manifest_id=7, start_row=0, end_row=10, rows_per_shard=4

Iteration 1: chunk_start=0, chunk_end=min(0+4, 10)=4   -> shard [0, 4)
Iteration 2: chunk_start=4, chunk_end=min(4+4, 10)=8   -> shard [4, 8)
Iteration 3: chunk_start=8, chunk_end=min(8+4, 10)=10  -> shard [8, 10)
Iteration 4: chunk_start=10, 10 < 10 is false           -> loop exits

Result: 3 shards covering [0,4), [4,8), [8,10)
         ^^^^^^^  ^^^^^^^  ^^^^^^
         full     full     tail (2 rows, not 4)
```

The tail chunk `[8, 10)` is shorter than `rows_per_shard` because the remaining span (2 rows) does not fill a complete chunk. The `.min(end_row)` clamp ensures the tail never extends past the parent's boundary, preventing the overlapping-chunk scenario from the opening failure.

The chunking also handles values near `u64::MAX` safely. Consider `start_row = u64::MAX - 5`, `end_row = u64::MAX`, `rows_per_shard = 4`. Without saturating arithmetic, `chunk_start + rows_per_shard` would overflow. With `saturating_add`, the computation produces `u64::MAX` (clamped), which is then reduced to `min(u64::MAX, u64::MAX) = u64::MAX`. The resulting two shards cover `[u64::MAX-5, u64::MAX-1)` and `[u64::MAX-1, u64::MAX)`.

## 3. `build_inputs` -- Final Manifest Validation

After all entries are staged, `build_inputs` materializes the final `InitialShardInput` rows and validates the complete manifest:

```rust
    /// Materialize borrowed manifest rows for registration.
    ///
    /// Re-validates that every stored handle is still live. Although the builder
    /// exclusively controls its borrowed arena (no external code can free handles
    /// between add and build), this check defends against future refactors that
    /// might introduce non-obvious invalidation paths.
    ///
    /// After handle validation, runs [`validate_manifest`] on the final slice
    /// so callers see registration-time manifest errors before touching
    /// coordinator state.
    ///
    /// This is a read-only view construction: it does not consume or mutate
    /// builder state. Repeated calls return equivalent rows while the builder
    /// remains unchanged.
    ///
    /// # Errors
    ///
    /// - [`PreallocShardBuilderError::InvalidSpecHandle`] when any staged
    ///   handle is stale/foreign.
    /// - [`PreallocShardBuilderError::ManifestInvalid`] when the staged rows
    ///   violate manifest constraints:
    ///   - empty manifest (no entries staged)
    ///   - duplicate shard IDs
    ///   - overlapping key ranges
    ///   - unbounded ranges (empty start or end key)
    ///   - invalid spec (fails [`ShardSpec::validate_ref`])
    ///   - cursor out of bounds (`last_key` outside spec range)
    ///   - cursor key too large (exceeds `MAX_KEY_SIZE`)
    pub fn build_inputs<'s>(
        &'s self,
    ) -> Result<InlineVec<InitialShardInput<'s>, CAP>, PreallocShardBuilderError>
    where
        'a: 's,
    {
        let mut out = InlineVec::new();
        for entry in self.entries.as_slice() {
            let spec = self
                .arena
                .try_view_spec(&entry.spec_handle)
                .ok_or(PreallocShardBuilderError::InvalidSpecHandle)?;
            out.push(InitialShardInput::new(entry.shard, spec, entry.cursor));
        }
        validate_manifest(out.as_slice()).map_err(PreallocShardBuilderError::ManifestInvalid)?;
        Ok(out)
    }
```

Three properties of `build_inputs` are worth noting:

**Handle liveness re-check.** The loop calls `try_view_spec` on every staged handle before constructing the `InitialShardInput`. The doc comment explains the rationale: the builder exclusively controls the arena, so handle invalidation between add and build is structurally impossible in the current design. The check defends against future refactors that introduce non-obvious invalidation paths -- a defensive programming choice that costs one arena lookup per entry.

**Manifest validation.** The call to `validate_manifest` checks the complete set of entries for cross-entry violations: overlapping key ranges, duplicate shard IDs, cursor out of bounds, and empty manifests. This is where the deferred validation from Chapter 5 pays off. Per-entry add-time checks catch malformed individual specs; `validate_manifest` catches the inter-entry invariants that only become visible when the full manifest is assembled.

**Idempotency.** `build_inputs` is read-only -- it does not consume or mutate builder state. Calling it twice returns equivalent rows. This allows callers to inspect the manifest (for logging, debugging, or dry-run validation) without committing to registration. The idempotency property also means that a caller can call `build_inputs` once to check validity, handle any errors, and then call it again to obtain the inputs for `register_shards` -- without needing to re-stage entries between the two calls.

The lifetime constraint `where 'a: 's` on the return type deserves attention. The returned `InlineVec<InitialShardInput<'s>, CAP>` borrows spec data from the arena (lifetime `'a`) and cursor data from the caller's original `CursorUpdate` references (also lifetime `'a`). The `'a: 's` bound ensures that the returned inputs cannot outlive the arena borrow or the cursor data. In practice, this means the caller must use the returned inputs before dropping the builder (which clears the arena) or before the cursor data goes out of scope.

## 4. Transactional Semantics

The module documentation summarizes the transactional guarantees across all builder operations:

```rust
//! # Transactional semantics
//!
//! A successful `add_*` call appends exactly one new entry and consumes one
//! shard ID. Failed `add_*` calls leave the staged manifest unchanged.
//!
//! Bulk split helpers preflight fan-out and entry budget before entering the
//! allocation loop. `split_range_by_boundaries` additionally validates all
//! child boundaries in a read-only pass before allocating, so **boundary
//! errors leave the builder state completely unchanged**. However, if arena
//! slab allocation fails mid-loop in either split helper, previously appended
//! children remain staged while later children are lost.
```

The guarantees form a hierarchy of atomicity:

```text
  Guarantee                     | add_*  | split_range (boundary err) | split_* (slab err)
  ------------------------------|--------|----------------------------|--------------------
  No entries staged on failure  |  Yes   |  Yes                       |  No (partial prefix)
  No shard IDs consumed         |  Yes   |  Yes                       |  No (consumed by prefix)
  Arena memory unchanged        |  Yes   |  Yes                       |  No (prefix allocated)
```

**Single adds** are fully atomic: a failed `add_range`, `add_prefix`, or `add_manifest` call leaves the builder state completely unchanged. The capacity check runs before arena allocation, and arena allocation runs before `push_entry`. If either fails, nothing is staged.

**Boundary errors in split_range_by_boundaries** are fully rolled back because Pass 1 validates all boundaries without allocating. If any boundary is invalid, the function returns before Pass 2 starts. No entries are staged, no shard IDs are consumed, no arena memory is allocated.

**Slab errors in split helpers** produce partial results. If the arena runs out of slots or bytes after staging three of five children, those three children remain staged with their shard IDs consumed. The caller must handle this case -- typically by dropping the builder (which clears the arena) and retrying with a larger arena.

This is a deliberate design trade-off. Making slab errors fully transactional would require tracking which handles were allocated during the current split call and freeing them on error -- a rollback mechanism that adds complexity for a failure mode that indicates a sizing miscalculation (the arena was not provisioned with enough capacity). The module documentation makes the partial-prefix behavior explicit so callers can account for it.

The practical impact of this trade-off is limited. Slab exhaustion during a split indicates that the caller underestimated the arena's required capacity. The correct response is to drop the builder (which clears the arena via `Drop`), create a larger arena, and retry. The partial-prefix entries from the failed split are cleared along with everything else. A fully transactional rollback would save the caller from having to retry, but at the cost of additional bookkeeping in every split call -- bookkeeping that is never needed when the arena is correctly sized.

## 5. The Internal Entry Struct

Each staged entry is a lightweight triple:

```rust
/// A staged shard registration: shard ID, arena-backed spec, and borrowed cursor.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
struct BuilderEntry<'a> {
    shard: ShardId,
    spec_handle: ShardSpecHandle,
    cursor: CursorUpdate<'a>,
}
```

`BuilderEntry` is `Copy` because all three fields are small value types: `ShardId` is a `u64` wrapper, `ShardSpecHandle` is an arena index, and `CursorUpdate` holds two `Option<&'a [u8]>` slices (borrowed, not owned). The entry does not own any heap memory -- the spec bytes live in the arena, and the cursor bytes are borrowed from the caller. This keeps the `InlineVec<BuilderEntry, CAP>` stack-friendly even at `CAP = 1024`.

The entry does not carry a `connector_extra` field because the extra bytes are encoded into the shard spec's metadata buffer during the add/split call. By the time the entry is staged, the connector extra is part of the spec bytes in the arena. The entry only needs to track the arena handle (which provides access to the spec bytes) and the cursor (which is borrowed from the caller). This keeps entries small and avoids redundant storage.

## 6. The End-to-End Workflow

Combining the add methods from Chapter 5 with the split helpers and validation from this chapter, a complete registration looks like this:

```text
  1. Caller creates a ShardArena with sufficient slot/byte capacity.
  2. Caller creates PreallocShardBuilder::new(arena, entry_limit).
     - Compile-time: CAP <= 1024
     - Runtime: entry_limit > 0, entry_limit <= CAP, entry_limit <= MAX_INITIAL_SHARDS

  3. Caller stages entries:
     - add_range / add_prefix / add_manifest for individual shards
     - split_range_by_boundaries for byte-range splits
     - split_manifest_by_rows for fixed-width row chunking
     - add_spec_ref / add_spec_handle for pre-existing specs

  4. Caller calls build_inputs():
     - Re-checks handle liveness
     - Runs validate_manifest (overlap, cursor bounds, uniqueness, empty check)
     - Returns InlineVec<InitialShardInput, CAP>

  5. Caller passes the InitialShardInput slice to register_shards.
     - If registration succeeds: shards enter the coordination protocol.
     - If registration fails: builder can be reset and retried.

  6. Builder is dropped, clearing the arena.
```

```text
         add_range()     split_range_by_boundaries()    build_inputs()
              |                    |                          |
              v                    v                          v
  +-------------------+   +-----------------+   +------------------------+
  | Per-entry checks: |   | Two-pass split: |   | Manifest-level checks: |
  | - capacity        |   | Pass 1: validate|   | - overlap detection    |
  | - range bounds    |   | Pass 2: allocate|   | - cursor bounds        |
  | - arena alloc     |   |                 |   | - uniqueness           |
  +-------------------+   +-----------------+   | - handle liveness      |
              |                    |             +------------------------+
              v                    v                          |
         push_entry()        push_entry() x N                v
              |                    |             InlineVec<InitialShardInput>
              +--------------------+                          |
                        |                                     v
                   entries buffer                    register_shards()
```

The two-phase workflow guarantees that the coordinator's `register_shards` receives a fully validated manifest. The opening scenario -- where 498 of 500 shards are partially committed -- cannot occur because the builder validates the complete manifest before any shard reaches the coordinator.

This pattern has a direct analogy in the coordination protocol itself. Recall from [Chapter 8](../04-boundary-2-coordination/08-split-replace.md) that `validate_split_coverage` checks children for contiguity and coverage before the coordinator commits any state changes. The builder's `build_inputs` serves the same role at registration time: it is the pre-commit validation gate that prevents malformed manifests from entering the system. The difference is that `validate_split_coverage` validates children against a parent's key range (a split invariant), while `build_inputs` validates a flat manifest for internal consistency (overlap, cursor bounds, uniqueness). Both embody the same principle: validate the complete set of changes before committing any of them.

### Connection to the Allocation Policy

The builder's design aligns with the project's tiered allocation policy. Registration is a COLD path -- it runs once at startup, not in the per-shard steady-state loop. The `InlineVec` entry buffer and the `ShardArena` slab allocator are pre-allocated structures, but the builder does not go to the extremes required on HOT paths (no pooled scratch buffers, no caller-owned reusable allocations). The `ShardSpecScratch` is reused across add calls within a single builder, which avoids per-call allocation for the key encoding buffers. But the builder freely allocates arena slots and does not attempt to minimize allocation count beyond what the slab provides. This is the correct trade-off for a COLD path: simplicity over allocation minimization.

## 7. Why Not a Single Generic Split Method?

A natural question arises: why does the builder provide two separate split methods (`split_range_by_boundaries` and `split_manifest_by_rows`) instead of a single generic method that accepts any splitting strategy?

The answer lies in the different validation requirements. Byte-range splits accept arbitrary caller-provided boundaries that must be validated against the parent range. The two-pass approach exists specifically to handle boundary validation errors transactionally. Manifest row splits compute boundaries internally through deterministic arithmetic -- no external boundaries to validate. A single generic method would either impose unnecessary validation overhead on manifest splits (a validation pass that cannot fail) or sacrifice the two-pass guarantee for byte-range splits.

The two methods also serve different connector types. Filesystem and object-store connectors typically work with byte-key ranges where split points come from external sources (load balancers, partition tables, manual configuration). Database connectors typically work with row ranges where chunk widths come from table statistics. Providing typed methods for each pattern makes the API self-documenting: calling `split_manifest_by_rows` makes the intent (row-based chunking) explicit in the code, whereas a generic `split(strategy)` call would require inspecting the strategy argument to understand the chunking behavior.

## 8. Summary

The bulk split helpers generate contiguous, non-overlapping child shards through two complementary strategies. `split_range_by_boundaries` uses a two-pass approach: validate all boundaries without allocation in Pass 1, then allocate and stage in Pass 2. `split_manifest_by_rows` uses arithmetic chunking with saturating addition and end-clamping to prevent wrapping near `u64::MAX`. Both methods check fan-out and entry-budget limits up front to prevent partial writes on capacity exhaustion.

`build_inputs` completes the workflow by running `validate_manifest` on the full set of staged entries, catching cross-entry violations (overlaps, cursor-out-of-bounds, duplicates) that per-entry add-time validation cannot detect. The result is a read-only, idempotent view of `InitialShardInput` rows ready for `register_shards`.

Chapter 7 reviews the test suite that verifies these algebraic properties.
