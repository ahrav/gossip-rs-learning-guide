# Proving Correctness: Simulation and TLA+

## Why Unit Tests Are Not Enough

The coordination protocol manages N workers contending over M shards through K operations under F independent fault modes. A single shard passes through states Active, Done, Split, and Parked. Each state transition is gated by a fence epoch check, a lease expiry check, a tenant isolation check, an owner identity check, and cursor monotonicity validation. When you multiply the possible orderings of three workers operating on five shards through ten operation types under three fault injection levels, the state space grows combinatorially in a way that no finite set of hand-written scenarios can cover.

Consider a specific failure scenario. Worker W1 acquires shard S at epoch 2 and checkpoints at cursor "f". A GC pause freezes W1 for 200 ticks. During the pause, W1's lease expires. Worker W2 acquires shard S at epoch 3 and checkpoints at cursor "h". W1 resumes and attempts to checkpoint at cursor "g". The coordinator must reject W1's checkpoint with `StaleFence` because W1 holds epoch 2 while the shard is now at epoch 3 -- and the cursor must remain at "h", not regress to "g".

A unit test can verify that `checkpoint(stale_lease) -> Err(StaleFence)` in isolation. But the correctness of the full scenario depends on the *composition* of multiple operations across multiple workers with specific timing. The fence epoch must increment on every acquisition. The cursor must never regress. The rejected operation must not corrupt any coordinator state. Testing these properties in combination, across all possible orderings, requires two techniques that go beyond unit testing: deterministic simulation and formal specification.

This chapter examines both techniques as they are implemented in Gossip-rs, and shows how they connect back to the failure scenarios explored throughout this guide.

## The S1-S9 Invariants

Every safety property in the coordination protocol is captured by one of nine invariants, labeled S1 through S9. Each invariant maps directly to a failure scenario from earlier chapters -- and each was motivated by a concrete way the protocol could fail.

### S1: Mutual Exclusion

**Rule**: At most one worker holds a non-expired lease per shard at any logical time.

**Failure scenario**: Two workers simultaneously believe they own the same shard and write conflicting data. This is the fundamental safety property that the entire fencing mechanism exists to prevent (Chapter 2).

### S2: Fence Monotonicity

**Rule**: A shard's `fence_epoch` never decreases for a given `(RunId, ShardId)`.

**Failure scenario**: If the fence epoch could decrease, a zombie worker from an earlier epoch could appear to hold a valid lease. The fence epoch is the shard's authoritative timestamp of ownership -- it must only increase (Chapter 2).

### S3: Terminal Irreversibility

**Rule**: Once a shard reaches Done, Split, or Parked, no protocol operation changes its status. The exception is the administrative Parked-to-Active unpark transition, which requires a fence epoch bump.

**Failure scenario**: A completed shard that reverts to Active could be scanned twice, producing duplicate results. A split shard that reverts could invalidate its children's key ranges (Chapter 1, Chapter 6).

### S4: Record Invariant

**Rule**: `ShardRecord::validate_invariants()` returns `Ok` for every shard record at every step. This is a structural invariant: Parked shards must have a `park_reason`, terminal shards must not hold leases, fence epochs must be at least `INITIAL`, the op-log must be bounded, and split shards must have non-empty spawned lists.

**Failure scenario**: A Parked shard without a park reason would confuse the operator trying to decide whether to unpark it. A terminal shard with an active lease would block future administrative operations.

### S5: Cursor Monotonicity

**Rule**: `cursor.last_key()` never decreases (lexicographically) within a shard's lifetime.

**Failure scenario**: If the cursor regresses, items that were already scanned could be scanned again, producing duplicate results. Cursor monotonicity is the progress guarantee that prevents the scan from going backward (Chapter 4).

### S6: Cursor Bounds

**Rule**: Non-initial cursors remain within the shard's `[spec.start, spec.end)` key range.

**Failure scenario**: A cursor outside the shard's assigned range indicates a routing bug -- the worker is scanning keys it was never assigned. After a split-residual operation narrows the parent's range, cursors must respect the new, tighter bounds (Chapter 4, Chapter 8).

### S7: Split Coverage

**Rule**: A split-parent's spawned children exist in the coordinator and reference the correct parent via their `parent` field.

**Failure scenario**: If a child shard references the wrong parent (or does not exist at all), the key range partitioning is broken -- items in the child's range might never be scanned, or might be scanned by the wrong shard (Chapter 8).

### S8: Run-Terminal Irreversibility

**Rule**: Terminal run states (`Done`, `Failed`, `Cancelled`) never revert to a non-terminal state. Unlike S3, runs have no "unpark" exception -- all terminal-to-non-terminal transitions are violations.

**Failure scenario**: A completed run that reverts to Active could accept new shard registrations or re-process shards that were already marked done, creating orphaned work and inconsistent audit records.

### S9: Claim Cooldown

**Rule**: The same worker must not claim (acquire) a shard twice within `cooldown_interval` logical ticks.

**Failure scenario**: Without cooldown enforcement, a fast worker could repeatedly acquire and release shards in a tight loop, starving other workers. The cooldown creates a fairness window that prevents claim-hogging.

### The Connection Table

Each failure scenario from the learning guide motivated one or more of these invariants. The invariants are not abstract safety properties invented in a vacuum -- they are formalized defenses against specific, concrete ways the protocol can fail:

| Chapter | Failure Scenario | Invariant(s) |
|---------|-----------------|--------------|
| Ch 1: Problem Space | Completed shard re-activated | S3 (Terminal Irreversibility) |
| Ch 2: Leases and Fencing | Zombie worker writes after lease expiry | S1 (Mutual Exclusion), S2 (Fence Monotonicity) |
| Ch 3: Starting a Scan | Stale epoch bypasses fence check | S2 (Fence Monotonicity) |
| Ch 4: Acquiring and Scanning | Cursor regresses after ownership transfer | S5 (Cursor Monotonicity) |
| Ch 4: Acquiring and Scanning | Cursor outside shard key range after split | S6 (Cursor Bounds) |
| Ch 5: Idempotency and Op-Log | Replay changes terminal state | S3 (Terminal Irreversibility) |
| Ch 6: Completing the Scan | Children missing or orphaned after split | S7 (Split Coverage) |
| Ch 7: Why Splits Exist | Run completes with non-terminal shards | S3, S4 (Record Invariant) |
| Ch 8: Split-Replace | Parked shard reverts without fence bump | S3 sub-property (Unpark Without Fence Bump) |
| Ch 9: Split-Residual | Session lifecycle corrupts record | S4 (Record Invariant) |
| Run Management | Run reverts from terminal state | S8 (Run-Terminal Irreversibility) |
| Worker Fairness | Worker starves others via rapid claiming | S9 (Claim Cooldown) |

## CoordinationSim: The Simulation Harness

The simulation harness (`CoordinationSim`) is a deterministic testing framework inspired by three systems: FoundationDB's simulation testing (Zhou et al., SIGMOD 2021), TigerBeetle's VOPR, and sled's simulation harness. Its core principle is borrowed from FoundationDB: **never trust the system's own validation for correctness verification**. The coordinator already runs `ShardRecord::assert_invariants()` after every mutation, but those internal checks cannot detect cross-shard violations, temporal regressions that span multiple steps, or referential integrity failures across parent/child records. An external observer with its own accumulated history is required.

### Architecture: Four Layers

The simulation is built in four composable layers:

**Layer 1: SimContext** -- Owns a single `ChaCha8Rng` PRNG and a monotonic `LogicalTime` clock. Every random decision in a simulation run flows through this one RNG instance. The PRNG is ChaCha8 (RFC 8439), which is algorithm-specified and value-stable across platforms -- unlike `StdRng`, which is explicitly non-portable. Fault rates use integer parts-per-million (PPM) instead of floating-point to eliminate IEEE 754 rounding variance.

**Layer 2: SimWorker** -- Per-worker bookkeeping that tracks lease claims, op-ID generation, pause state, and cursor progress. Critically, this is the worker's *local belief* about what it holds. It can diverge from the coordinator's ground truth after a silent lease expiry -- and that divergence is exactly what creates interesting fault scenarios.

**Layer 3: InvariantChecker** -- The external observer. Maintains per-shard history (previous fence epochs, previous cursor positions, previous terminal states) across simulation steps to detect temporal violations. Validates all nine invariants (S1-S9) in a single pass over coordinator state after *every* operation.

**Layer 4: CoordinationSim** -- The top-level driver. Runs a three-stage simulation with weighted random operation generation, fault injection, and full invariant checking.

### The Three-Phase Execution Model

A simulation run consists of three sequential stages:

**Phase 0: Zombie Preamble.** A scripted, deterministic sequence that exercises the bookkeeping-cleanup path by expiring a lease and re-acquiring on a different worker. This runs unconditionally before random operations, ensuring the most common distributed failure pattern is tested on every seed.

**Phase 1: Safety Phase.** Runs `safety_ops` weighted random operations under fault injection. The first few operations (the warmup window, `WARMUP_OPS = 5`) suppress faults to let workers establish leases before time-jumps can expire them. After warmup, every operation type is exercised at weighted probabilities: acquire, renew, checkpoint, complete, park, split-replace, split-residual, replay, conflict, zombie checkpoint, claim, time advance, pause, resume, session lifecycle, and unpark.

After every single operation -- whether it succeeds or fails -- the `InvariantChecker` runs a full sweep of all nine invariants (S1-S9). This is the core safety guarantee: no operation can leave the coordinator in a state that violates any invariant without immediate detection.

**Phase 2: Liveness Phase.** Runs `liveness_ops` operations biased toward acquire and complete (60% combined), verifying that the system converges to terminal states. This tests the bounded liveness property: given enough operations, every shard should eventually reach Done, Split, or Parked.

### The External Invariant Checker

The `InvariantChecker` is the heart of the simulation's safety verification. It is defined in `crates/gossip-coordination/src/sim/invariants.rs` and implements a stateful observer pattern:

```rust
pub struct InvariantChecker {
    /// Last-seen fence epoch per (tenant, run, shard), for S2 monotonicity.
    prev_epochs: BTreeMap<HistoryKey, FenceEpoch>,
    /// Last-seen shard status per (tenant, run, shard), for S3 terminal
    /// irreversibility.
    prev_terminal: BTreeMap<HistoryKey, ShardStatus>,
    /// Last-seen cursor `last_key` per (tenant, run, shard), for S5 cursor
    /// monotonicity. `None` means the cursor was in its initial state.
    prev_cursors: BTreeMap<HistoryKey, Option<Box<[u8]>>>,
    /// Reusable buffer for S1 mutual-exclusion post-pass check.
    active_holders: HashMap<ShardKey, Vec<WorkerId>>,
    /// Reusable scratch buffer for S7 split-coverage: missing children.
    scratch_missing: Vec<ShardId>,
    /// Reusable scratch buffer for S7 split-coverage: wrong-parent children.
    scratch_wrong_parent: Vec<ShardId>,
    /// Reusable scratch buffer for post-pass pruning of terminal shards.
    scratch_prune: Vec<HistoryKey>,
    /// Last-seen run status per (tenant, run), for S8 run-terminal
    /// irreversibility.
    prev_run_terminal: BTreeMap<(TenantId, RunId), RunStatus>,
    /// Last successful claim timestamp per worker, for S9 cooldown spacing.
    last_claim_by_worker: BTreeMap<WorkerId, LogicalTime>,
    /// Minimum tick spacing between successive claims by the same worker.
    cooldown_interval: u64,
    /// Buffered S9 violations pushed by `record_claim_success` and drained
    /// by `check_all`.
    cooldown_violations: Vec<InvariantViolation>,
}
```

Eleven fields supporting nine invariants. The `prev_*` maps grow as new shards appear (e.g., from split operations that create children). After each pass, permanently terminal shards (`Done`, `Split`) have their epoch and cursor history pruned to bound memory growth in long-running simulations. `Parked` shards are never pruned because the Parked-to-Active unpark transition requires continued monitoring. `prev_terminal` is retained for all terminal shards so S3 can still detect illegal reversions.

The `scratch_*` buffers and `active_holders` map are retained across calls and `.clear()`'d rather than reallocated, reducing per-call allocation pressure on the simulation hot path. `cooldown_violations` buffers S9 violations that are pushed by `record_claim_success` (called when `ClaimNext` succeeds) and drained during the next `check_all`.

The `check_all` method performs a single pass over all shard records:

```rust
pub fn check_all(
    &mut self,
    coordinator: &impl SimIntrospection,
    tenant: TenantId,
    now: LogicalTime,
) -> Vec<InvariantViolation> {
    let mut violations = Vec::new();
    self.active_holders.clear();

    for ((tid, key), record) in coordinator.shards() {
        if tid != tenant { continue; }
        let id = (tenant, record.run, record.shard);

        self.accumulate_active_holder(key, record, now);       // S1
        let prev_epoch = self.check_fence_monotonicity(         // S2
            id, record.fence_epoch, &mut violations);
        self.check_terminal_irreversibility(                     // S3
            id, record.status, prev_epoch, record.fence_epoch,
            &mut violations);
        Self::check_record_invariant(coordinator, record,        // S4
            &mut violations);
        let current_last_key = coordinator.cursor_last_key(record);
        let (spec_start, spec_end) = coordinator.spec_bounds(record);
        self.check_cursor_monotonicity(id, current_last_key,     // S5
            &mut violations);
        Self::check_cursor_in_bounds(record, current_last_key,   // S6
            spec_start, spec_end, &mut violations);
        self.check_split_coverage(                               // S7
            record, tenant, coordinator, &mut violations);
    }

    self.check_mutual_exclusion(&mut violations);                // S1 post-pass
    self.check_run_terminal_irreversibility(                     // S8
        coordinator, tenant, &mut violations);
    violations.append(&mut self.cooldown_violations);            // S9
    // ... prune permanently terminal shards ...
    violations
}
```

S1 (Mutual Exclusion) is a cross-record property -- two records for the same shard key with overlapping active leases -- so it cannot be checked inline per record. Active lease holders are accumulated during the pass and scanned for duplicates afterward. S8 (Run-Terminal Irreversibility) iterates run records separately via `coordinator.runs()`. S9 (Cooldown) violations are buffered by `record_claim_success` and appended here.

The S1 accumulation uses a deliberately conservative boundary convention:

```rust
fn accumulate_active_holder(
    &mut self, key: ShardKey, record: &ShardRecord, now: LogicalTime
) {
    if let Some(holder) = record.lease()
        && holder.deadline() >= now  // >= not >, conservative
    {
        self.active_holders.entry(key).or_default().push(holder.owner());
    }
}
```

The `>=` comparison (rather than the coordinator's strict `<`) means a lease at its exact deadline is treated as active for mutual-exclusion checking. This accepts false positives (flagging a non-violation at boundary ticks) over false negatives (missing a real dual-holder scenario). A false positive is harmless; a false negative would undermine the safety guarantee.

### Invariant Violations as Structured Types

When a violation is detected, the checker produces a typed `InvariantViolation` enum rather than a string. This enables pattern matching in test assertions:

```rust
pub enum InvariantViolation {
    MutualExclusion { key: ShardKey, workers: [WorkerId; 2] },
    FenceMonotonicity { run: RunId, shard: ShardId,
                        prev: FenceEpoch, current: FenceEpoch },
    TerminalIrreversibility { run: RunId, shard: ShardId,
                              was: ShardStatus, now: ShardStatus },
    RecordInvariant { run: RunId, shard: ShardId, message: String },
    CursorMonotonicity { run: RunId, shard: ShardId,
                         prev: Option<Box<[u8]>>,
                         current: Option<Box<[u8]>> },
    CursorOutOfBounds { run: RunId, shard: ShardId },
    SplitCoverage { run: RunId, shard: ShardId,
                    detail: SplitCoverageDetail },
    UnparkWithoutFenceBump { run: RunId, shard: ShardId,
                             fence_at_park: FenceEpoch,
                             fence_at_unpark: FenceEpoch },
    RunTerminalIrreversibility { run: RunId,
                                 was: RunStatus, now: RunStatus },
    CooldownViolation { worker: WorkerId,
                        this_claim: LogicalTime,
                        prev_claim: LogicalTime,
                        min_interval: u64 },
}
```

Every violation carries enough context to diagnose the failure: which shard, which run, what the previous value was, and what the current (violating) value is. The `UnparkWithoutFenceBump` variant is a sub-property of S3 that specifically detects the case where a Parked-to-Active transition occurs without incrementing the fence epoch. The `RunTerminalIrreversibility` and `CooldownViolation` variants correspond to S8 and S9 respectively.

## FaultConfig: Three Difficulty Levels

Fault injection uses three severity levels, each designed to test a different aspect of the protocol:

| Level | Lease Expiry | Worker Pause | Time Jump | Design Intent |
|-------|-------------|-------------|-----------|---------------|
| SunnyDay | 0% | 0% | 0% | Validate happy-path correctness |
| Stormy | 10% (100,000 PPM) | 5% (50,000 PPM) | 10% (100,000 PPM) | Simulate production-like failure rates |
| Radioactive | 20% (200,000 PPM) | 10% (100,000 PPM) | 20% (200,000 PPM) | Stress-test corner cases |

The Radioactive level is inspired by TigerBeetle's VOPR (Viewstamped Operation Replication), which uses progressively aggressive fault injection to surface bugs that only manifest under extreme conditions. At 20% lease expiry rate, roughly one in five operations involves a stale lease -- a scenario that would be vanishingly rare in production but reveals protocol weaknesses that would otherwise hide for years.

Fault rates use PPM (parts-per-million) rather than floating-point to ensure bitwise determinism across platforms:

```rust
pub struct FaultConfig {
    lease_expiry_ppm: u32,    // 0..=1_000_000
    pause_ppm: u32,
    pause_range: RangeInclusive<u64>,
    time_jump_ppm: u32,
    time_jump_range: RangeInclusive<u64>,
}
```

The `should_inject` function draws uniformly over `[0, 1_000_000)` and compares against the PPM threshold. Zero never fires, one million always fires, and everything in between is proportional:

```rust
fn should_inject(rng: &mut ChaCha8Rng, ppm: u32) -> bool {
    if ppm == 0 { return false; }
    assert!(ppm <= PPM_MAX);
    rng.random_range(0u32..PPM_MAX) < ppm
}
```

The time-jump ranges differ by level: Stormy jumps 50-200 ticks (enough to expire a single lease with the default 100-tick duration), while Radioactive jumps 100-500 ticks (enough to expire multiple consecutive leases).

## Worker Simulation

Each simulated worker (`SimWorker`) maintains its own local bookkeeping:

```rust
pub struct SimWorker {
    id: WorkerId,
    held_shards: BTreeMap<(u64, u64), (ShardKey, Lease)>,
    next_op: u64,
    op_ceiling: u64,
    paused: bool,
    last_cursors: BTreeMap<(RunId, ShardId), Vec<u8>>,
}
```

### Op-ID Partitioning

Workers generate op-IDs from non-overlapping partitions: worker N uses IDs in `[N * 1_000_000, (N+1) * 1_000_000)`. This guarantees cross-worker uniqueness without any coordination between workers -- each worker can generate up to one million unique op-IDs without consulting any shared state.

### Forward-Only Cursor Generation

The harness tracks per-worker cursor progress in `last_cursors` and only generates cursors that advance beyond the previous checkpoint. Without this, random cursors would frequently regress, flooding the run with expected `CursorRegression` rejections that mask real bugs. The cursor generation stays within the shard's key range bounds to avoid expected `CursorOutOfBounds` rejections.

### Stale Lease Tracking

When worker B acquires a shard previously held by worker A, the harness saves A's superseded lease. These stale leases feed the `ZombieCheckpoint` operation, which bypasses bookkeeping cleanup to exercise the coordinator's fence-based `StaleFence` rejection path directly. This is the mechanism that verifies the zombie worker scenario from Chapter 2 across hundreds of random orderings.

### The Operation Vocabulary

The simulation generates operations from a vocabulary of 17 distinct types:

```rust
pub enum SimOp {
    Acquire { worker, key },         // Lease acquisition
    Renew { worker, key },           // Lease extension
    Checkpoint { worker, key },      // Cursor progress
    Complete { worker, key },        // Terminal: shard done
    Park { worker, key },            // Terminal: shard parked
    SplitReplace { worker, key },    // Terminal: parent -> children
    SplitResidual { worker, key },   // Non-terminal: parent shrinks
    ReplayCheckpoint { worker, key },    // Idempotent replay
    ConflictCheckpoint { worker, key },  // OpId conflict
    ZombieCheckpoint,                    // Stale-lease injection
    ClaimNext { worker },            // Claim-next-available facade
    AdvanceTime { ticks },           // Clock manipulation
    PauseWorker { worker },          // Simulate stall
    ResumeWorker { worker },         // Resume from stall
    SessionLifecycle { worker, key },// Full session lifecycle
    Unpark { key },                  // Admin: Parked -> Active
    TerminateRun { run, kind },      // Admin: run terminal transition
}
```

Each operation is selected by weighted random choice. The weights ensure that every coordinator code path is exercised while maintaining a realistic distribution (checkpoints more frequent than splits, time advances interspersed to create lease expiry opportunities).

## The Mega-Sim: Large-Scale Seed Sweeping

The mega simulation test (`mega_sim_10k_steps`) is the primary CI gate. It sweeps across multiple PRNG seeds in parallel, running each seed through the full three-phase simulation:

```rust
#[test]
#[ignore]  // too slow for default cargo test
fn mega_sim_10k_steps() {
    let seed_count = parse_seed_count(); // default: 100 seeds
    let fault_level = parse_fault_level(); // default: Stormy

    // Divide seeds across OS threads
    std::thread::scope(|s| {
        let handles: Vec<_> = seeds.chunks(chunk_size).map(|chunk| {
            s.spawn(move || {
                for seed in chunk {
                    let report = CoordinationSim::new(seed, fault_level)
                        .with_workers_and_shards(4, 15)
                        .run(10_000, 2_000);
                    // Collect failures with reproduction commands
                }
            })
        }).collect();
    });
}
```

Each seed runs 4 workers contending over 15 shards through 10,000 safety operations (random operations under fault injection) followed by 2,000 liveness operations (biased toward acquire + complete). After every single operation, the `InvariantChecker` runs all nine checks (S1-S9).

The test makes two assertions:

1. **Zero violations**: Any seed producing an invariant violation fails the test with a reproduction command: `GOSSIP_SIM_SEED=<seed> cargo test -p gossip-coordination mega_sim -- --ignored --nocapture`.

2. **Event-kind coverage**: The aggregate across all seeds must contain five core event kinds (AcquireOk, CheckpointOk, CompleteOk, Rejected, TimeAdvanced). This catches harness regressions that silently suppress entire coordinator code paths.

### Stress Tests

Beyond the standard mega-sim sweep, dedicated stress tests exercise extreme configurations:

- **200 shards under Stormy faults** (`stress_200_shards_stormy`): 8 workers competing for 200 shards, testing the invariant checker's pruning logic and memory bounds at scale.

- **Radioactive split cascades** (`stress_split_cascade`): 4 workers, 20 shards under Radioactive faults, designed to trigger multi-level split cascades where parent shards split and their children get split again. Verifies that S7 (Split Coverage) holds even when the shard set grows dynamically.

### Convergence Testing

The mega-sim sweep checks *safety* (S1-S9) but never asserts *convergence* -- that every shard eventually reaches a terminal state. Dedicated convergence proptests close this gap:

```rust
fn assert_convergence(seed: u64, level: FaultLevel,
                      safety: usize, liveness: usize) {
    let report = CoordinationSim::new(seed, level)
        .with_workers_and_shards(4, 15)
        .run(safety, liveness);
    assert!(report.violations.is_empty());
    assert!(report.converged);
}
```

These run 200 proptest cases each for SunnyDay (50 safety + 2,000 liveness ops) and Stormy (500 safety + 15,000 liveness ops). The Stormy budget is 7.5x larger than SunnyDay to compensate for the ~10% fault-induced rejection rate. Radioactive is intentionally omitted from convergence testing because its 20% fault pressure makes bounded convergence unreliable within practical op budgets.

## The TLA+ Specification

While simulation testing exercises the implementation across many random operation sequences, it cannot explore *all possible* interleavings. A simulation with 4 workers and 15 shards running 10,000 operations visits a tiny fraction of the reachable state space. The TLA+ specification (`specs/coordination/ShardFencing.tla`) addresses this gap through exhaustive model checking.

### The Formal Model

The TLA+ specification models the coordination protocol as a state machine with seven state variables per shard, plus per-worker cached epochs and a global logical clock:

```tla
VARIABLES
    status,         \* [AllShards -> StatusSet]
    fence_epoch,    \* [AllShards -> 0..MaxEpoch]
    owner,          \* [AllShards -> Workers \cup {none}]
    deadline,       \* [AllShards -> 0..MaxTime]
    cursor,         \* [AllShards -> 0..MaxCursor]
    spawned,        \* [AllShards -> SUBSET AllShards]
    prev_cursor,    \* [AllShards -> 0..MaxCursor] -- ghost variable
    worker_epoch,   \* [Workers -> [AllShards -> 0..MaxEpoch]]
    clock           \* Nat -- logical time
```

The `prev_cursor` variable is a *ghost variable* -- it does not affect the protocol's behavior but exists solely so the model checker can verify cursor monotonicity (S5) as a state invariant.

### Actions Map to Protocol Operations

Each TLA+ action corresponds to a coordinator operation:

**Acquire(w, s)**: Worker `w` acquires shard `s`. Requires the shard to be Active and not currently leased. Increments `fence_epoch`, sets `owner` and `deadline`, and caches the new epoch in `worker_epoch[w][s]`.

**Checkpoint(w, s)**: Worker `w` advances the cursor on shard `s`. Requires `FenceGuard(w, s)` -- the worker must have a matching epoch, unexpired lease, and be the current owner. Increments `cursor` and saves the old value in `prev_cursor`.

**Complete(w, s)** and **Park(w, s)**: Terminal transitions. Require `FenceGuard`, set status to Done or Parked, and clear the lease (owner and deadline).

**SplitReplace(w, s)**: Atomic split. Requires `FenceGuard` and `s = parent`. Transitions parent to Split, creates children as Active with `fence_epoch = 1` (INITIAL), and clears the parent's lease. The atomicity is critical: `status'` updates the parent and both children in a single step.

**Unpark(s)**: Administrative operation (not lease-gated). Transitions Parked to Active with a fence bump. This is the only operation that can revert a terminal status.

**Tick**: Non-deterministic clock advance. Models adversarial timing -- the clock can advance at any time, potentially expiring leases.

### The FenceGuard Predicate

The `FenceGuard` predicate encodes the validation pipeline from `validation.rs`:

```tla
FenceGuard(w, s) ==
    /\ status[s] = "Active"              \* Check 2: terminal status
    /\ worker_epoch[w][s] = fence_epoch[s] \* Check 3: fence epoch
    /\ clock < deadline[s]                \* Check 4: lease expiry
    /\ owner[s] = w                       \* Check 5: owner identity
```

Check 1 (tenant isolation) is omitted because the TLA+ model is single-tenant. The check ordering matches `validate_lease` in the implementation.

### Safety Properties

The specification checks safety properties matching the simulation's invariants:

```tla
\* P1: Mutual exclusion
MutualExclusion ==
    \A s \in AllShards :
        Cardinality({w \in Workers :
            /\ owner[s] = w
            /\ worker_epoch[w][s] = fence_epoch[s]
            /\ clock < deadline[s]}) <= 1

\* P2: Zombie rejection
ZombieRejection ==
    \A w \in Workers, s \in AllShards :
        /\ owner[s] = w /\ clock < deadline[s]
        => worker_epoch[w][s] = fence_epoch[s]

\* Fence monotonicity (action property)
FenceMonotonicity ==
    \A s \in AllShards : fence_epoch'[s] >= fence_epoch[s]

\* Terminal irreversibility (action property)
TerminalIrreversibility ==
    \A s \in AllShards :
        /\ (status[s] = "Done" => status'[s] = "Done")
        /\ (status[s] = "Split" => status'[s] = "Split")

\* Cursor monotonicity (ghost variable)
CursorMonotonicity ==
    \A s \in AllShards :
        status[s] /= "NotCreated" => cursor[s] >= prev_cursor[s]

\* Split atomicity
SplitAtomicity ==
    \A s \in AllShards :
        status[s] = "Split" =>
            /\ spawned[s] /= {}
            /\ \A c \in spawned[s] : status[c] /= "NotCreated"
```

Note that `FenceMonotonicity` and `TerminalIrreversibility` are *action properties* (checked on every state transition), not state invariants. They are wrapped in temporal formulas for TLC: `AlwaysFenceMonotonicity == [][FenceMonotonicity]_vars`.

### TLC Model Configuration

The safety configuration (`ShardFencing.cfg`) uses 3 workers, 3 shards (parent + 2 children), with small bounds:

```
CONSTANT Workers = {w1, w2, w3}
CONSTANT AllShards = {parent, child1, child2}
CONSTANT MaxEpoch = 3
CONSTANT MaxCursor = 2
CONSTANT MaxTime = 6
CONSTANT LeaseDuration = 2
SYMMETRY WorkerSymmetry
```

Workers are interchangeable (symmetry set reduces the state space by 6x). Shards are *not* symmetric because the parent has a distinguished role in `SplitReplace`. With these bounds, TLC exhaustively explores all reachable states -- every possible interleaving of 3 workers acquiring, checkpointing, completing, parking, splitting, and unparking 3 shards, with the clock ticking non-deterministically.

### The Liveness Specification

A separate configuration (`ShardFencing_liveness.cfg`) checks the liveness property: under weak fairness, an Active unleased shard is eventually leased.

```tla
Liveness ==
    \A s \in AllShards :
        (  status[s] = "Active"
        /\ ~IsLeased(s)
        /\ fence_epoch[s] < MaxEpoch
        /\ clock + LeaseDuration <= MaxTime)
        ~> IsLeased(s)
```

The `~>` (leads-to) operator asserts that if the left-hand side is true at some point, the right-hand side eventually becomes true. The property is conditioned on model bounds (epoch and time must have room to produce a valid lease), which is standard practice for liveness in bounded models.

### What TLA+ Catches That Simulation Cannot

The TLA+ specification and the simulation harness are complementary verification tools. Each catches bugs the other cannot:

**TLA+ advantages:**
- Exhaustive exploration of all interleavings for a given model size. If a bug exists within the model bounds, TLC *will* find it.
- Action properties (like fence monotonicity) checked on every state transition, not just at observation points.
- Formal proof that safety properties are invariants of the protocol, not just absent from sampled runs.
- Detection of design-level bugs before any code is written.

**Simulation advantages:**
- Tests the actual Rust implementation, catching implementation bugs that match a correct spec.
- Exercises realistic configurations (15+ shards, 4+ workers, 10K+ operations) far beyond TLC's bounded model.
- Tests fault injection patterns (GC pauses, clock skew, crash-restart) that are difficult to model in TLA+.
- Tests the full validation pipeline including tenant isolation, op-log eviction, and error priority ordering.
- Verifies operational concerns like allocation pressure, memory bounds, and history pruning.

The two techniques overlap in their coverage of the core safety properties (mutual exclusion, fence monotonicity, terminal irreversibility, cursor monotonicity, split coverage). The overlap is intentional -- it provides defense in depth. A bug that escapes TLA+'s bounded model might be caught by simulation's larger configurations, and a bug that escapes simulation's random sampling might be caught by TLA+'s exhaustive search.

## Connecting Everything

The following table maps the full chain from failure scenario through invariant through both verification mechanisms:

| Failure Scenario | Invariant | Simulation Check | TLA+ Property |
|-----------------|-----------|-----------------|---------------|
| Zombie worker writes after lease expiry | S1: Mutual Exclusion | `check_mutual_exclusion` post-pass | `MutualExclusion` state invariant |
| Fence epoch decreases | S2: Fence Monotonicity | `check_fence_monotonicity` temporal | `AlwaysFenceMonotonicity` action property |
| Terminal shard reverts to Active | S3: Terminal Irreversibility | `check_terminal_irreversibility` temporal | `AlwaysTerminalIrreversibility` action property |
| Unpark without fence bump | S3 sub-property | `UnparkWithoutFenceBump` detection | N/A (Unpark action bumps fence by construction) |
| Parked shard missing park_reason | S4: Record Invariant | `check_record_invariant` structural | `TypeOK` type invariant |
| Cursor regresses after ownership transfer | S5: Cursor Monotonicity | `check_cursor_monotonicity` temporal | `CursorMonotonicity` (ghost variable) |
| Cursor outside shard key range | S6: Cursor Bounds | `check_cursor_in_bounds` structural | `AlwaysCursorNonRegression` action property |
| Split children missing or orphaned | S7: Split Coverage | `check_split_coverage` referential | `SplitAtomicity`, `ChildImpliesParentSplit` |
| Run reverts from terminal state | S8: Run-Terminal Irreversibility | `check_run_terminal_irreversibility` temporal | N/A (not modeled in TLA+) |
| Worker claims too fast | S9: Claim Cooldown | `record_claim_success` + `check_all` drain | N/A (not modeled in TLA+) |
| Stale-epoch worker holds valid lease | S1 + S2 | `ZombieCheckpoint` op exercises path | `ZombieRejection` state invariant |
| Non-terminal shard with active lease and fence epoch below INITIAL | S4 | Record-level `validate_invariants` | `TerminalUnleased`, `FenceEpochSanity` |
| Active unleased shard never acquired | Liveness | `converged` flag in `SimReport` | `Liveness` leads-to property |

### The Fault-Injecting Introspector

One remaining challenge: the `InMemoryCoordinator` stores shards in a `HashMap<(TenantId, ShardKey), ShardRecord>`, which structurally prevents S1 violations -- at most one record exists per key per tenant. The checker's S1 detection logic cannot be tested against the real backend.

The `FaultInjectingIntrospector` solves this by wrapping the real backend and injecting synthetic shard records into the observation stream:

```rust
pub struct FaultInjectingIntrospector<B: SimIntrospection> {
    inner: B,
    synthetic_records: Vec<(TenantId, ShardKey, ShardRecord)>,
}
```

`shards()` returns real records followed by synthetic records. `shard_lookup()` delegates to the inner backend only -- synthetic records are invisible to point lookups. This asymmetry models a class of real-world distributed-system bugs where a full scan reveals inconsistencies that targeted reads miss (e.g., stale replicas visible in range scans but masked by read-repair on point lookups).

Dedicated tests inject dual-holder scenarios, fence regressions, terminal reversions, cursor regressions, out-of-bounds cursors, and missing split children -- confirming that the `InvariantChecker` would detect each violation if a future backend produced it.

## The Verification Pyramid

The full verification strategy forms a pyramid with increasing abstraction:

| Level | Technique | What It Catches | Where in Codebase |
|-------|-----------|----------------|-------------------|
| 1 | Unit tests | Single-operation correctness | `in_memory_tests.rs` |
| 2 | Conformance tests | Cross-invariant interactions | `conformance_tests.rs` |
| 3 | Scenario tests | Multi-step workflow correctness | `scenario_tests.rs` |
| 4 | Property-based tests | Random-input structural invariants | `proptest_state_machine_tests.rs` |
| 5 | Deterministic simulation | Safety under fault injection | `crates/gossip-coordination/src/sim/harness.rs`, `crates/gossip-coordination/src/sim/mega_sim_tests.rs` |
| 6 | TLA+ model checking | Protocol design correctness | `specs/coordination/ShardFencing.tla` |

Each level catches bugs the levels below it cannot. Unit tests verify individual operations. Conformance tests verify that two invariants do not conflict. Scenario tests verify realistic multi-step workflows. Property-based tests verify structural invariants under random inputs. Simulation testing verifies safety under faults across thousands of random operation sequences. TLA+ verifies that the protocol design itself is correct, exhaustively within bounded model sizes.

The combination of deterministic simulation (testing the implementation under faults, at scale, across many seeds) and TLA+ model checking (verifying the protocol design exhaustively within bounds) provides a level of confidence in the coordination protocol that neither technique could achieve alone. The simulation cannot explore all interleavings, and TLA+ cannot test the implementation. Together, they cover the space from protocol design through implementation correctness.

> **References**: The simulation approach follows FoundationDB's simulation testing framework (Zhou et al., "FoundationDB: A Distributed Unbundled Transactional Key Value Store", SIGMOD 2021), which found bugs that were "astronomically unlikely" to trigger in production testing. The progressive difficulty levels (SunnyDay, Stormy, Radioactive) are inspired by TigerBeetle's VOPR, which uses similar escalation to surface corner-case bugs. The Jepsen methodology (Kingsbury, 2013-present) establishes the principle that distributed systems must be tested under partition and failure conditions to have any confidence in their safety claims. The TLA+ specification follows the patterns described by Lamport in "Specifying Systems" (2002) and the practices reported by Newcombe et al. ("How Amazon Web Services Uses Formal Methods", CACM 2015).
