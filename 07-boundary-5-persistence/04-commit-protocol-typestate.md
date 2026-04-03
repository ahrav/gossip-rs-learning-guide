# The Ghost Cursor -- Commit Ordering and the PageCommit Typestate Machine

*A distributed scan worker processes a page of 128 items for shard 3 under tenant `0x7a3f…`. The scan engine detects 14 secrets across those items. The persistence backend durably writes all 14 findings. It then durably writes 128 done-ledger rows. The worker issues a checkpoint to advance the shard cursor from offset `item_key=0xab01` to `item_key=0xef99`. The backend accepts the write — but the underlying fsync fails silently. The process restarts. The coordination layer reads the last durable cursor: still `0xab01`. It re-dispatches the page. The scan engine rediscovers the same 14 secrets, but now the done-ledger already contains entries for all 128 items from the previous attempt. The deduplication logic sees "already scanned" for items that were never checkpointed. Depending on the done-ledger conflict resolution policy, the system either skips items it believes are complete (missing findings from items whose done-ledger entries survived but whose findings did not) or double-reports findings with different observation timestamps. The cursor never advanced, but the done-ledger says the work is finished. The system's view of progress and its view of completeness have diverged. There is no single write to roll back — the corruption is spread across three independent persistence surfaces.*

---

The failure above is not a theoretical concern. Any system that persists data across multiple storage surfaces — findings rows, deduplication records, cursor checkpoints — must commit those surfaces in a strict order, and must prove that each surface is durable before advancing to the next. If the ordering is violated, or if a later stage claims success without the earlier stages having been confirmed, crash recovery enters an inconsistent state that no single retry can repair.

The core insight is that a page's durability is not a single event but a *sequence* of events, each gated on the previous one. Findings must be durable before done-ledger rows are written, because done-ledger rows signal "this item is complete" and any reader (including crash recovery) that sees a done-ledger entry will skip the item. Done-ledger rows must be durable before the checkpoint advances, because the checkpoint tells the coordination layer "this page is finished, move on." If the checkpoint advances before the done-ledger is confirmed, a crash leaves the coordinator believing the page is complete while the done-ledger has no record of the items — the next scan for this shard starts from the new cursor and the skipped items are never re-examined.

Gossip-rs encodes this ordering constraint at the type level. The `PageCommit<S>` typestate machine makes it a *compile error* to advance the checkpoint before the done-ledger is confirmed, or to confirm the done-ledger before findings are durable. The compiler enforces the invariant that runtime bugs cannot. Where a traditional implementation might assert ordering at runtime and panic on violation, the typestate approach shifts the check to compile time: code that would violate the ordering does not compile, so the violation can never reach production.

## CommitScope: Identifying a Page's Commit Work

Every page commit is scoped to exactly one combination of tenant, run, shard, fence epoch, item count, and cursor boundary. The `CommitScope` struct freezes these values at construction time and serves as the reference identity for every subsequent validation check:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:89-97
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct CommitScope {
    tenant_id: TenantId,
    policy_hash: PolicyHash,
    run_id: RunId,
    shard_id: ShardId,
    fence_epoch: FenceEpoch,
    committed_units: NonZeroU64,
    checkpoint_boundary: CheckpointBoundary,
}
```

Seven fields, each serving a distinct purpose:

- **`tenant_id`** — Tenant isolation boundary. A receipt from tenant A must never be applied to tenant B's commit.
- **`policy_hash`** — Detection-policy version under which the scope was produced. Carried for scope-identity completeness — checkpoint receipt validation compares the full scope — even though checkpoint routing itself is policy-independent.
- **`run_id`** — Identifies the scan run that produced the page. Prevents stale receipts from a previous run from being mistakenly applied to a new one.
- **`shard_id`** — The shard that emitted the page. In a sharded system, concurrent pages from different shards must not cross-contaminate.
- **`fence_epoch`** — The coordination fence epoch under which processing occurred. If the epoch has advanced (because another worker took over the shard), receipts from the old epoch are invalid.
- **`committed_units: NonZeroU64`** — The number of units the page represents. This is the cross-step invariant: the done-ledger receipt must confirm exactly this many records. `NonZeroU64` enforces that a commit scope always represents at least one unit of work.
- **`checkpoint_boundary: CheckpointBoundary`** — The checkpoint boundary the commit must durably acknowledge. This is the "destination" — the position the shard will advance to once all three stages are confirmed.

Once constructed, the scope is immutable. Every receipt-validation check compares against these frozen values. If the scope could be mutated mid-protocol, the typestate guarantees would be meaningless.

The constructor and accessors are straightforward:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:99-161
impl CommitScope {
    #[must_use]
    pub fn new(
        tenant_id: TenantId,
        policy_hash: PolicyHash,
        run_id: RunId,
        shard_id: ShardId,
        fence_epoch: FenceEpoch,
        committed_units: NonZeroU64,
        checkpoint_boundary: CheckpointBoundary,
    ) -> Self {
        Self {
            tenant_id,
            policy_hash,
            run_id,
            shard_id,
            fence_epoch,
            committed_units,
            checkpoint_boundary,
        }
    }

    pub const fn tenant_id(&self) -> TenantId { self.tenant_id }
    pub const fn policy_hash(&self) -> PolicyHash { self.policy_hash }
    pub const fn run_id(&self) -> RunId { self.run_id }
    pub const fn shard_id(&self) -> ShardId { self.shard_id }
    pub const fn fence_epoch(&self) -> FenceEpoch { self.fence_epoch }
    pub const fn committed_units(&self) -> NonZeroU64 { self.committed_units }
    pub fn checkpoint_boundary(&self) -> &CheckpointBoundary { &self.checkpoint_boundary }
}
```

All accessors are `const fn` where possible and `#[must_use]` — the compiler warns if a caller retrieves a scope field and discards the result. The `checkpoint_boundary` accessor returns a reference because `Cursor` is not `Copy` (it contains variable-length key data).

## The Four Typestate Markers

The `PageCommit<S>` machine moves through four states, each represented by a zero-overhead marker struct. The type parameter `S` is one of these four types, and each `impl` block exposes only the transition methods valid for that particular state.

### AwaitingFindings: The Entry Point

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:231-232
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct AwaitingFindings;
```

The initial state. No durable acknowledgement exists yet. The only available transition is to provide a findings receipt.

### FindingsDurable: Findings Confirmed

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:238-241
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct FindingsDurable {
    findings: FindingsCommitReceipt,
}
```

Findings have been durably written. The marker carries the `FindingsCommitReceipt` so it can be bundled into the composite `ItemCommitReceipt` when the done-ledger stage completes. The only available transition is to provide a done-ledger receipt.

### ItemDurable: Findings and Done-Ledger Confirmed

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:249-252
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ItemDurable {
    item_commit: ItemCommitReceipt,
}
```

Both findings and done-ledger rows are durable. The marker carries the composite `ItemCommitReceipt`, which callers can extract for progress reporting even before the checkpoint round-trip completes. The only available transition is to provide a checkpoint receipt.

### CheckpointDurable: Terminal State

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:258-261
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct CheckpointDurable {
    page_commit: PageCommitReceipt,
}
```

All three stages are durable. The `PageCommitReceipt` inside is sufficient proof that the cursor can be safely advanced. No further transitions exist.

Notice the progression of carried data: `AwaitingFindings` carries nothing (it is a unit struct), `FindingsDurable` carries one receipt, `ItemDurable` carries a composite of two receipts, and `CheckpointDurable` carries the full composite of all three. Each state accumulates proof from the previous stages, building toward the terminal `PageCommitReceipt`. The typestate markers are not just access-control tokens — they are the receipts themselves, threaded through the type parameter.

## Design Trade-Off: Typestate vs. Runtime Enum

The source code explicitly documents the alternative that was rejected. A runtime state enum with `match` arms and panics on invalid transitions would produce simpler generic signatures — no `<S>` parameter, no separate `impl` blocks per state. But incorrect ordering in a runtime enum is a *panic*, discoverable only through testing or production crashes. In a typestate machine, incorrect ordering is a *compilation failure*, discoverable the moment the code is written.

The trade-off is real. The typestate approach makes `PageCommit<S>` generic, which means callers that hold a `PageCommit` in an intermediate state must also be generic or use an `enum` wrapper. Functions that accept a `PageCommit` at any state must be written as `fn foo<S>(page: &PageCommit<S>)`. Storage in containers requires either a fixed state or a type-erased wrapper.

Given that incorrect ordering is a data-loss bug — the ghost cursor scenario from the opening — the compile-time cost is justified. The source module docs state this directly: "typestate makes invalid orderings a compile error at the cost of being generic over `S`."

## The PageCommit\<S\> Struct and Its Transitions

The core struct is generic over the state parameter:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:332-337
#[must_use = "page commits must be driven to a durable receipt or explicitly dropped"]
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct PageCommit<S> {
    scope: Arc<CommitScope>,
    state: S,
}
```

The `scope` field uses `Arc<CommitScope>` because the scope is shared across all typestate transitions without cloning the inner data. The `#[must_use]` annotation ensures callers cannot silently drop a `PageCommit` mid-protocol — the compiler warns if a page commit is constructed but never driven to completion or explicitly discarded.

A shared accessor is available in every state:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:339-346
impl<S> PageCommit<S> {
    #[inline]
    #[must_use]
    pub fn scope(&self) -> &CommitScope {
        &self.scope
    }
}
```

### Transition 1: AwaitingFindings → FindingsDurable

Construction starts the protocol:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:348-356
impl PageCommit<AwaitingFindings> {
    #[inline]
    pub fn new(scope: CommitScope) -> Self {
        Self {
            scope: Arc::new(scope),
            state: AwaitingFindings,
        }
    }
```

The `record_findings` method accepts a receipt the caller already holds:

```rust
    // From crates/gossip-contracts/src/persistence/page_commit.rs:365-369
    #[inline]
    pub fn record_findings(self, receipt: FindingsCommitReceipt) -> PageCommit<FindingsDurable> {
        PageCommit {
            scope: self.scope,
            state: FindingsDurable { findings: receipt },
        }
    }
```

The `wait_findings` method accepts a `CommitHandle` and blocks until the receipt is available:

```rust
    // From crates/gossip-contracts/src/persistence/page_commit.rs:377-383
    pub fn wait_findings<H>(self, handle: H) -> Result<PageCommit<FindingsDurable>, H::Error>
    where
        H: CommitHandle<Receipt = FindingsCommitReceipt>,
    {
        let receipt = handle.wait()?;
        Ok(self.record_findings(receipt))
    }
}
```

No validation is performed on the findings receipt. The findings sink is driven by the same in-process pipeline that constructed the scope, so cross-page mix-ups are structurally prevented. The receipt carries only aggregate counts (finding, occurrence, and observation row counts), not scope-identifying fields.

### Transition 2: FindingsDurable → ItemDurable

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:613-624
    pub fn record_done_ledger(
        self,
        receipt: DoneLedgerCommitReceipt,
    ) -> Result<PageCommit<ItemDurable>, PageCommitValidationError> {
        let expected = self.scope.committed_units().get();
        let actual = receipt.record_count();
        if expected != actual {
            return Err(PageCommitValidationError::LedgerUnitCountMismatch { expected, actual });
        }

        let item_commit = ItemCommitReceipt::new(self.scope.clone(), self.state.findings, receipt);
        Ok(PageCommit {
            scope: self.scope,
            state: ItemDurable { item_commit },
        })
    }
```

This transition performs the first validation: the done-ledger receipt's `record_count` must exactly equal the scope's `committed_units`. If the page scope says 128 items were committed, the done-ledger receipt must confirm exactly 128 rows. A mismatch indicates a partial flush or a receipt mix-up between concurrent pages.

On success, the method assembles an `ItemCommitReceipt` — a composite proof that both findings and done-ledger rows are durable for this page.

The corresponding `wait_done_ledger` combines the handle wait with validation:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:424-434
    pub fn wait_done_ledger<H>(
        self,
        handle: H,
    ) -> Result<PageCommit<ItemDurable>, CommitAdvanceError<H::Error>>
    where
        H: CommitHandle<Receipt = DoneLedgerCommitReceipt>,
    {
        let receipt = handle.wait().map_err(CommitAdvanceError::Wait)?;
        self.record_done_ledger(receipt)
            .map_err(CommitAdvanceError::Validation)
    }
```

### Transition 3: ItemDurable → CheckpointDurable

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:674-689
    pub fn record_checkpoint(
        self,
        receipt: CheckpointCommitReceipt,
    ) -> Result<PageCommit<CheckpointDurable>, PageCommitValidationError> {
        if receipt.scope() != self.scope() {
            return Err(PageCommitValidationError::CheckpointScopeMismatch {
                expected: Box::new(self.scope().clone()),
                actual: Box::new(receipt.scope().clone()),
            });
        }

        let page_commit = PageCommitReceipt::new(self.state.item_commit, receipt);
        Ok(PageCommit {
            scope: self.scope,
            state: CheckpointDurable { page_commit },
        })
    }
```

The checkpoint transition performs the strictest validation: the receipt's embedded `CommitScope` must match this page's scope in every field — tenant, run, shard, fence epoch, item count, and cursor. This is the "full scope equality" check. Checkpoint receipts go through a backend round-trip (potentially across a network boundary to a coordination store), where receipt mix-ups between concurrent pages are most likely. The single equality check on the embedded scope catches all of them.

On success, the method assembles the terminal `PageCommitReceipt` — proof that all three stages are durable.

The `wait_checkpoint` follows the same pattern:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:482-492
    pub fn wait_checkpoint<H>(
        self,
        handle: H,
    ) -> Result<PageCommit<CheckpointDurable>, CommitAdvanceError<H::Error>>
    where
        H: CommitHandle<Receipt = CheckpointCommitReceipt>,
    {
        let receipt = handle.wait().map_err(CommitAdvanceError::Wait)?;
        self.record_checkpoint(receipt)
            .map_err(CommitAdvanceError::Validation)
    }
```

### Terminal State: Extracting the Receipt

Once in the `CheckpointDurable` state, the caller extracts the final proof:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:495-509
impl PageCommit<CheckpointDurable> {
    #[inline]
    #[must_use]
    pub fn page_commit_receipt(&self) -> &PageCommitReceipt {
        &self.state.page_commit
    }

    #[inline]
    #[must_use]
    pub fn into_page_commit_receipt(self) -> PageCommitReceipt {
        self.state.page_commit
    }
}
```

Two extraction methods: a borrow for inspecting the receipt without consuming the state machine, and a consuming move for callers that are done with the `PageCommit` entirely.

## Why Two Methods Per Transition

Every transition stage offers a pair of methods: `record_X` and `wait_X`. This is not redundancy — it serves two distinct use patterns.

**`record_X`** is for callers that have already obtained the receipt through their own means. Perhaps the caller issued a `handle.wait()` separately, or received the receipt from a pre-resolved path. The caller has the receipt in hand and simply needs to advance the state machine. `record_X` methods are synchronous and infallible (for findings) or fallible only on validation (for done-ledger and checkpoint).

**`wait_X`** is for callers that have a `CommitHandle` — the backend's promise of eventual durability — and want the state machine to drive the wait. The method calls `handle.wait()` internally, then delegates to `record_X` for validation. `wait_X` methods can fail with either a backend wait error (transient) or a validation error (deterministic).

This dual-method design has a critical implication for error recovery. The `wait_X` methods consume the `PageCommit` via `self`. If the backend wait fails, the state machine is gone. The caller must reconstruct a new `PageCommit` from scratch and retry the full protocol from `AwaitingFindings`. This is intentional: after a wait failure, the page's I/O state is unknown (the backend may or may not have committed), so the only safe action is to treat the page as abandoned.

Callers that need finer retry control use `record_X` methods with a separately managed `handle.wait()` call, keeping the `PageCommit` alive until a receipt is definitively in hand.

## Validation Errors

### PageCommitValidationError

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs
#[derive(Debug, Clone, PartialEq, Eq, thiserror::Error)]
pub enum PageCommitValidationError {
    #[error("done-ledger unit count mismatch: expected {expected}, got {actual}")]
    LedgerUnitCountMismatch { expected: u64, actual: u64 },
    #[error("checkpoint scope mismatch")]
    CheckpointScopeMismatch {
        expected: Box<CommitScope>,
        actual: Box<CommitScope>,
    },
}
```

Two variants, each guarding a different invariant:

- **`LedgerUnitCountMismatch`** — The done-ledger receipt acknowledged a different number of records than the scope's `committed_units`. This catches partial flushes (the backend wrote 120 of 128 rows before confirming) and receipt mix-ups (a receipt from a 64-item page applied to a 128-item page).

- **`CheckpointScopeMismatch`** — The checkpoint receipt's embedded scope does not match the page scope on at least one field. Carries both the expected and actual `CommitScope` (boxed for diagnostics). This catches every category of cross-page confusion: wrong tenant, wrong shard, wrong epoch, wrong cursor boundary.

Both are deterministic — retrying with the same inputs produces the same error. These indicate programming errors or backend mis-routing bugs, not transient failures.

An important asymmetry exists in validation depth. The findings stage performs *no* validation because the findings receipt carries only aggregate counts — it cannot identify which page it belongs to. The done-ledger stage performs *partial* validation (item count only), because the done-ledger receipt also lacks full scope identity but does carry a record count that must be consistent. The checkpoint stage performs *full* validation (all seven scope fields), because the checkpoint receipt embeds a complete `CommitScope` after traversing a backend round-trip where cross-page confusion is most likely. This graduated validation strategy avoids redundant checks at stages where the data physically cannot be misrouted, while applying maximum scrutiny at the stage with the highest risk.

### CommitAdvanceError\<E\>

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:198-204
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CommitAdvanceError<E> {
    Wait(E),
    Validation(PageCommitValidationError),
}
```

The `wait_X` methods return this composite error. The type parameter `E` is the backend's wait error type — it could be an I/O error, a replication timeout, or any other transient failure. The `Validation` variant wraps the deterministic `PageCommitValidationError`. Callers can pattern-match to decide: retry the whole page (on `Wait`) or report a bug (on `Validation`).

The `Display` and `Error` implementations preserve the source chain for both variants:

```rust
// From crates/gossip-contracts/src/persistence/page_commit.rs:218-228
impl<E> Error for CommitAdvanceError<E>
where
    E: Error + Send + Sync + 'static,
{
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Self::Wait(err) => Some(err),
            Self::Validation(err) => Some(err),
        }
    }
}
```

This enables standard error-chain tooling (`anyhow`, `eyre`, tracing's `err` field) to display the full causal chain from the user-facing error down to the root backend failure or validation mismatch.

## The CommitHandle and CommitReceipt Trait System

The typestate machine does not directly interact with any persistence backend. Instead, it operates on two abstraction traits that backends implement.

### CommitReceipt: Proof of Durability

```rust
// From crates/gossip-contracts/src/persistence/commit.rs:52
pub trait CommitReceipt: Clone + fmt::Debug + Send + Sync + 'static {}
```

A marker trait with supertrait bounds ensuring receipts can be cloned into composite receipts, debug-formatted in diagnostics, and sent across threads. Every receipt type in the system — `FindingsCommitReceipt`, `DoneLedgerCommitReceipt`, `CheckpointCommitReceipt`, `ItemCommitReceipt`, `PageCommitReceipt` — implements this trait.

### CommitHandle: Deferred Durability

```rust
// From crates/gossip-contracts/src/persistence/commit.rs:63-76
#[must_use = "durability is not established until wait() returns a receipt"]
pub trait CommitHandle: Send + 'static {
    type Receipt: CommitReceipt;
    type Error: Error + Send + Sync + 'static;

    fn wait(self) -> Result<Self::Receipt, Self::Error>;
}
```

A `CommitHandle` represents the backend's acceptance of a write — but acceptance is not durability. Only `wait()` returning `Ok(receipt)` proves the write is durable. The `wait` method consumes `self`, preventing double-waits. The `#[must_use]` annotation prevents silently dropping an unwaited handle.

### ReadyCommitHandle: The Synchronous Adapter

```rust
// From crates/gossip-contracts/src/persistence/commit.rs:84-120
#[must_use = "the contained result must be observed via wait()"]
#[derive(Debug)]
pub struct ReadyCommitHandle<R, E>(Result<R, E>);

impl<R, E> ReadyCommitHandle<R, E> {
    #[inline]
    pub fn ok(receipt: R) -> Self {
        Self(Ok(receipt))
    }

    #[inline]
    pub fn err(error: E) -> Self {
        Self(Err(error))
    }

    #[inline]
    pub fn from_result(result: Result<R, E>) -> Self {
        Self(result)
    }
}

impl<R, E> CommitHandle for ReadyCommitHandle<R, E>
where
    R: CommitReceipt,
    E: Error + Send + Sync + 'static,
{
    type Receipt = R;
    type Error = E;

    #[inline]
    fn wait(self) -> Result<Self::Receipt, Self::Error> {
        self.0
    }
}
```

`ReadyCommitHandle` wraps an already-computed `Result`. Its `wait()` returns immediately. This serves two purposes: synchronous backends that perform inline WAL + fsync can wrap the result without inventing an async mechanism, and test doubles can inject known success or failure outcomes. The happy-path test in the source uses `ReadyCommitHandle` to drive the entire typestate machine through all four states without any real I/O.

### The Receipt Hierarchy

Receipts compose hierarchically, mirroring the commit protocol stages:

| Receipt | Contents | Assembled By |
|---------|----------|--------------|
| `FindingsCommitReceipt` | finding, occurrence, observation counts | Backend findings sink |
| `DoneLedgerCommitReceipt` | record count, scanned count, findings count | Backend done-ledger sink |
| `CheckpointCommitReceipt` | full `CommitScope` + `LogicalTime` | Backend checkpoint writer |
| `ItemCommitReceipt` | `Arc<CommitScope>` + findings + done-ledger receipts | `record_done_ledger` transition |
| `PageCommitReceipt` | `ItemCommitReceipt` + `CheckpointCommitReceipt` | `record_checkpoint` transition |

The bottom three (`FindingsCommitReceipt`, `DoneLedgerCommitReceipt`, `CheckpointCommitReceipt`) are produced by persistence backends. The top two (`ItemCommitReceipt`, `PageCommitReceipt`) are assembled by the typestate machine itself — they exist only because the machine validated the ordering and the cross-step invariants. Holding a `PageCommitReceipt` is proof that all three stages completed in the correct order with consistent scope.

## State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> AwaitingFindings: PageCommit::new(scope)

    AwaitingFindings --> FindingsDurable: record_findings(receipt)
    AwaitingFindings --> FindingsDurable: wait_findings(handle)

    FindingsDurable --> ItemDurable: record_done_ledger(receipt)
    FindingsDurable --> ItemDurable: wait_done_ledger(handle)

    ItemDurable --> CheckpointDurable: record_checkpoint(receipt)
    ItemDurable --> CheckpointDurable: wait_checkpoint(handle)

    CheckpointDurable --> [*]: into_page_commit_receipt()

    note right of FindingsDurable
        Validates: done-ledger record_count
        must equal scope.committed_units
    end note

    note right of ItemDurable
        Validates: checkpoint receipt scope
        must equal page scope (all 7 fields)
    end note
```

The diagram shows the linear progression through four states. There are no backward transitions, no skip transitions, and no branches. Every path through the machine visits the same three validation gates in the same order. The compiler enforces this — calling `record_checkpoint` on a `PageCommit<AwaitingFindings>` is a type error, not a runtime panic.

## Compile-Time Enforcement in Practice

The source includes `compile_fail` doc-tests that prove the compiler rejects invalid orderings. Skipping findings and jumping straight to the done-ledger:

```rust
// This does not compile — record_done_ledger is only available on FindingsDurable
let page = PageCommit::new(scope);
let _ = page.record_done_ledger(receipt); // ERROR: no method named `record_done_ledger`
                                          // found for `PageCommit<AwaitingFindings>`
```

Skipping the done-ledger and jumping from findings to checkpoint:

```rust
// This does not compile — record_checkpoint is only available on ItemDurable
let page = PageCommit::new(scope).record_findings(findings_receipt);
let _ = page.record_checkpoint(checkpoint_receipt); // ERROR: no method named `record_checkpoint`
                                                    // found for `PageCommit<FindingsDurable>`
```

These are not aspirational goals — they are tested in CI via Rust's `compile_fail` test infrastructure. The typestate machine converts a data-loss bug class into a compilation failure.

The source includes a more subtle test as well: a done-ledger receipt from a completely different page (different tenant, different run, different shard) is *accepted* by `record_done_ledger` as long as the `record_count` matches — because that transition intentionally validates only the item count, not full scope identity. The full scope check is deferred to `record_checkpoint`, where the backend round-trip makes cross-page confusion a real risk. This deliberate validation asymmetry is tested explicitly to confirm it is a design choice, not an oversight.

## Relationship to CommitSink

The codebase contains two commit-related abstractions that operate at different granularities:

**`CommitSink`** is the per-item interface used by scan drivers. It exposes three methods — `begin_item`, `upsert_findings`, `finish_item` — that the driver calls for each scanned item within a page:

```rust
// From crates/gossip-scanner-runtime/src/commit_sink.rs:418-430
pub trait CommitSink: Send + Sync {
    fn begin_item(&self, item_key: &ItemKey, meta: &ItemMeta) -> Result<()>;
    fn upsert_findings(&self, item_key: &ItemKey, batch: &FindingsBatch) -> Result<()>;
    fn finish_item(&self, item_key: &ItemKey) -> Result<()>;
}
```

**`PageCommit<S>`** is the per-page commit ordering protocol used by the runtime. It operates one level above `CommitSink`, orchestrating the durability of an entire page's worth of items.

The relationship: the runtime uses `CommitSink` to process individual items within a page, accumulating findings and done-ledger entries. Once all items in the page are processed, the runtime batches the accumulated results and drives them through the `PageCommit` typestate: first flushing all findings to get a `FindingsCommitReceipt`, then flushing all done-ledger rows to get a `DoneLedgerCommitReceipt`, then issuing the checkpoint to get a `CheckpointCommitReceipt`. The typestate machine guarantees the ordering; the `CommitSink` trait handles the individual-item granularity within each batch.

`CommitSink` uses runtime method-ordering conventions (`begin_item` before `upsert_findings` before `finish_item`) because it operates behind a `dyn` trait object and processes many items concurrently. `PageCommit<S>` uses compile-time typestate enforcement because it processes exactly one page at a time and the ordering violation is a data-loss bug. The two abstractions are complementary: `CommitSink` for the high-frequency per-item path, `PageCommit` for the critical per-page durability gate.

The layering can be visualized as:

```
┌─────────────────────────────────────────────────────┐
│  PageCommit<S> (per-page, compile-time ordering)    │
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  CommitSink (per-item, runtime ordering)      │  │
│  │                                               │  │
│  │  begin_item → upsert_findings → finish_item   │  │
│  │  begin_item → upsert_findings → finish_item   │  │
│  │  ...repeated for each item in the page...     │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  record_findings → record_done_ledger →             │
│    record_checkpoint → PageCommitReceipt            │
└─────────────────────────────────────────────────────┘
```

The inner `CommitSink` loop handles individual items. The outer `PageCommit` machine handles the page-level durability gates. Neither replaces the other — they enforce ordering at different granularities and with different mechanisms appropriate to their respective constraints.

## Summary

| Concept | Role |
|---------|------|
| `CommitScope` | Frozen identity for one page commit: tenant, policy, run, shard, epoch, committed units, checkpoint boundary |
| `AwaitingFindings` | Entry state — no durability confirmed |
| `FindingsDurable` | Findings confirmed durable; carries `FindingsCommitReceipt` |
| `ItemDurable` | Findings + done-ledger confirmed; carries composite `ItemCommitReceipt` |
| `CheckpointDurable` | Terminal state; carries `PageCommitReceipt` proving full page durability |
| `record_X` / `wait_X` | Dual transition methods: pre-resolved receipt vs. deferred `CommitHandle` |
| `PageCommitValidationError` | Cross-step invariant violations: item count mismatch, scope mismatch |
| `CommitAdvanceError<E>` | Composite error for `wait_X`: transient backend failure or deterministic validation failure |
| `CommitHandle` / `CommitReceipt` | Backend-neutral abstraction for deferred durability acknowledgement |
| `ReadyCommitHandle` | Pre-resolved adapter for synchronous backends and test doubles |

The typestate machine transforms a protocol that could fail silently at runtime into one that cannot be misused at compile time. The three-stage ordering — findings, done-ledger, checkpoint — is not a convention documented in a comment. It is a type-level constraint enforced by the Rust compiler. Dropping a stage, reordering stages, or applying a receipt from the wrong page are all compilation errors.

## What's Next

Chapter 5 examines the in-memory persistence backends that implement the `CommitHandle` and `CommitReceipt` traits. These test doubles — including `ReadyCommitHandle` and `CliNoOpCommitSink` — make it possible to drive the entire `PageCommit` typestate machine through all four states in unit tests without any disk I/O, verifying the ordering invariants and validation logic in isolation from real storage backends. That chapter also covers how `CliNoOpCommitSink` serves as the `CommitSink` implementation for CLI-mode scans, where per-item persistence is unnecessary and the three-method lifecycle degenerates into no-ops.
