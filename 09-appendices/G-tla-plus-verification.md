# Appendix G: TLA+ Formal Verification

## Why TLA+ for B2 Coordination

The Boundary 2 coordination protocol defines 7 operations (Acquire, Checkpoint,
Complete, Park, SplitReplace, Unpark, Tick/time-advance) across 5 shard states
(NotCreated, Active, Done, Split, Parked). Each operation has multi-branch
decision trees governed by fence epoch validation, lease expiry, terminal status
checks, and cursor bounds. The resulting interaction surface is too large for
informal reasoning to guarantee correctness.

TLA+ addresses this by exhaustively model-checking every reachable state. The
model checker (TLC) explores all possible interleavings of worker actions and
clock ticks, verifying that safety invariants hold in every state and liveness
properties hold across all fair behaviors. This is the same approach used at
Amazon Web Services, where formal methods found subtle bugs in production
distributed systems that testing alone could not catch (Newcombe et al., "How
Amazon Web Services Uses Formal Methods," CACM 2015).

The TLA+ spec lives in a single file that captures the protocol's essential
logic without implementation details like byte-key ranges or tenant isolation.
This abstraction is deliberate: TLA+ verifies the *protocol design*, while
simulation testing (Tier 4) verifies the *Rust implementation*.

**Source file:** `/Users/ahrav/Projects/Gossip-rs/specs/coordination/ShardFencing.tla`

---

## ShardFencing.tla Overview

### Constants

The specification declares a fixed topology of one parent shard and two children,
a set of workers (used as a symmetry set for state-space reduction), and bounds
that make the model finite for TLC:

```tla
CONSTANTS
    Workers,        \* Set of worker IDs (symmetry set for TLC)
    AllShards,      \* Set of shard IDs: {parent, child1, child2}
    MaxEpoch,       \* Upper bound for fence epochs (for TLC finiteness)
    MaxCursor,      \* Upper bound for abstract cursor position
    MaxTime,        \* Upper bound for logical clock
    LeaseDuration   \* Duration added to clock for lease deadline
```

The production safety config (`ShardFencing.cfg`) instantiates these as 3
workers, MaxEpoch=3, MaxCursor=2, MaxTime=6, LeaseDuration=2.

### Variables

Each TLA+ variable maps to a field in the Rust `ShardRecord`:

| TLA+ variable  | Type                              | Rust equivalent                         |
|-----------------|-----------------------------------|-----------------------------------------|
| `status`        | `[AllShards -> StatusSet]`        | `ShardRecord.status` (`ShardStatus`)    |
| `fence_epoch`   | `[AllShards -> 0..MaxEpoch]`      | `ShardRecord.fence_epoch` (`FenceEpoch`)|
| `owner`         | `[AllShards -> Workers U {none}]` | `ShardRecord.lease.map(\|l\| l.owner)`  |
| `deadline`      | `[AllShards -> 0..MaxTime]`       | `ShardRecord.lease.map(\|l\| l.deadline)` |
| `cursor`        | `[AllShards -> 0..MaxCursor]`     | `ShardRecord.cursor` (`Cursor`)         |
| `spawned`       | `[AllShards -> SUBSET AllShards]` | `ShardRecord.spawned` (`SpawnedList (InlineVec<ShardId, 1024>)`)  |
| `prev_cursor`   | `[AllShards -> 0..MaxCursor]`     | Ghost variable (no Rust equivalent)     |
| `worker_epoch`  | `[Workers -> [AllShards -> 0..MaxEpoch]]` | `Lease.fence` (cached by worker) |
| `clock`         | `1..MaxTime`                      | `LogicalTime` parameter                 |

### Actions

The specification models 7 actions. `FenceGuard(w, s)` encodes the
`validate_lease` checks from `validation.rs` (terminal status rejection, fence
epoch comparison, lease expiry):

```tla
FenceGuard(w, s) ==
    /\ status[s] = "Active"
    /\ worker_epoch[w][s] = fence_epoch[s]
    /\ clock < deadline[s]
    /\ owner[s] = w
```

| TLA+ action    | Rust function          | Lease-gated? |
|----------------|------------------------|:------------:|
| `Acquire`      | `acquire_and_restore_into` | No           |
| `Checkpoint`   | `checkpoint`           | Yes          |
| `Complete`     | `complete`             | Yes          |
| `Park`         | `park_shard`           | Yes          |
| `SplitReplace` | `split_replace`        | Yes          |
| `Unpark`       | `unpark_shard`         | No           |
| `Tick`         | (environment action)   | N/A          |

### What is NOT modeled

The TLA+ spec deliberately omits several aspects that are orthogonal to the
fencing protocol or are tested by other means:

- **Op-log idempotency** -- orthogonal to fencing; tested by Tier 1-3 tests
- **Tenant isolation** -- single-tenant model; tested by unit and conformance tests
- **Byte-key ranges** -- abstract integer cursors suffice for monotonicity
- **Lease renewal** (`renew`) -- no fence bump; does not affect safety properties
- **Split residual** (`split_residual`) -- parent stays Active; same fencing as Checkpoint

---

## Safety Properties

### P1: Mutual Exclusion -> S1

**TLA+ definition:**

```tla
MutualExclusion ==
    \A s \in AllShards :
        Cardinality({w \in Workers :
            /\ owner[s] = w
            /\ worker_epoch[w][s] = fence_epoch[s]
            /\ clock < deadline[s]}) <= 1
```

In the TLA+ model, `owner` is a function returning exactly one value per shard,
so `MutualExclusion` is tautological by construction. The operative safety
guarantee is `ZombieRejection` (P2). Both are retained for documentation and
consistency with the simulation's S1 check.

**Rust equivalent:** `InvariantChecker::check_mutual_exclusion` in
`crates/gossip-coordination/src/sim/invariants.rs`. The Rust checker
accumulates active lease holders into a `HashMap<ShardKey, Vec<WorkerId>>` during
a pass over all shard records, then scans for duplicates. Unlike the TLA+ model,
the Rust implementation uses a `HashMap` that can hold multiple records per shard
key (defensive for future backends), making S1 a non-trivial check.

### P2: Zombie Rejection -> StaleFence error

**TLA+ definition:**

```tla
ZombieRejection ==
    \A w \in Workers, s \in AllShards :
        /\ owner[s] = w
        /\ clock < deadline[s]
        => worker_epoch[w][s] = fence_epoch[s]
```

This property ensures that a worker holding a valid lease always has an epoch
matching the shard's current fence epoch. A stale worker (one that missed an
epoch bump due to re-acquisition by another worker) cannot hold a valid lease.

**Rust equivalent:** The `validate_lease` function in `validation.rs` performs
check 3 (fence epoch comparison): if `lease.fence != record.fence_epoch`, the
operation is rejected with a `StaleFence` error. This is the runtime enforcement
of P2.

### P3: Split Atomicity -> S7

**TLA+ definition:**

```tla
SplitAtomicity ==
    \A s \in AllShards :
        status[s] = "Split" =>
            /\ spawned[s] /= {}
            /\ \A c \in spawned[s] : status[c] /= "NotCreated"

ChildImpliesParentSplit ==
    \A c \in {child1, child2} :
        status[c] /= "NotCreated" => status[parent] = "Split"
```

These two invariants together ensure that the parent-to-Split and
children-to-Active transitions happen atomically. There is no state where
children exist without the parent being Split, and no state where the parent is
Split without all children being Active (or beyond).

**Rust equivalent:** `InvariantChecker::check_split_coverage` in
`crates/gossip-coordination/src/sim/invariants.rs`. The Rust S7 checker
verifies three sub-cases: (a) the split shard has a non-empty `spawned` list,
(b) every referenced child exists in the coordinator, and (c) every child's
`parent` field references the correct parent.

### Additional Safety Properties

**Fence monotonicity (S2):** `[][fence_epoch'[s] >= fence_epoch[s]]_vars` --
fence epochs never decrease. Rust equivalent: `InvariantChecker::check_fence_monotonicity`.

**Terminal irreversibility (S3):** `[][Done => Done' /\ Split => Split']_vars` --
Done and Split states never revert. Parked-to-Active is allowed only via Unpark
with a fence bump. Rust equivalent: `InvariantChecker::check_terminal_irreversibility`.

**Cursor monotonicity (S5):** `cursor[s] >= prev_cursor[s]` using a ghost
variable. Rust equivalent: `InvariantChecker::check_cursor_monotonicity`.

---

## Liveness Properties

The specification includes one liveness property:

```tla
Liveness ==
    \A s \in AllShards :
        (  status[s] = "Active"
        /\ ~IsLeased(s)
        /\ fence_epoch[s] < MaxEpoch
        /\ clock + LeaseDuration <= MaxTime)
        ~> IsLeased(s)
```

This states that under weak fairness on `Acquire`, an Active unleased shard with
room in the model bounds is eventually leased. The liveness spec (`LiveSpec`)
excludes `Tick` from the next-state relation because `Tick` is an environment
action; per Lamport's convention, no fairness is applied to environment actions.
In a bounded model, unconstrained `Tick` can advance the clock past `MaxTime`
before `Acquire` fires, starving acquisition. Removing `Tick` freezes the clock
at 1, verifying the protocol's acquisition liveness under favorable timing. The
full expire-and-re-acquire cycle is verified by the safety spec (which explores
all timing interleavings) and the simulation tests.

The liveness config (`ShardFencing_liveness.cfg`) uses:
- 2 workers (smaller state space)
- **No SYMMETRY** (incompatible with liveness checking)
- **No CONSTRAINT** (unsound with liveness checking)
- `-lncheck final` flag for sound liveness verification

---

## Cross-Reference Table

| TLA+ Property               | Kind     | Sim Label | Rust Function                                     | Source File                                              |
|------------------------------|----------|:---------:|---------------------------------------------------|----------------------------------------------------------|
| `MutualExclusion`            | Safety   | S1        | `InvariantChecker::check_mutual_exclusion`         | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `ZombieRejection`            | Safety   | --        | `validate_lease` (check 3: fence epoch)            | `crates/gossip-coordination/src/validation.rs`           |
| `SplitAtomicity`             | Safety   | S7        | `InvariantChecker::check_split_coverage`           | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `ChildImpliesParentSplit`    | Safety   | S7        | `InvariantChecker::check_split_coverage`           | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `TerminalUnleased`           | Safety   | S4        | `ShardRecord::validate_invariants` (INV-3)         | `crates/gossip-coordination/src/record.rs`               |
| `FenceEpochSanity`           | Safety   | S2        | `ShardRecord::validate_invariants` (INV-4)         | `crates/gossip-coordination/src/record.rs`               |
| `CursorMonotonicity`         | Safety   | S5        | `InvariantChecker::check_cursor_monotonicity`      | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `AlwaysFenceMonotonicity`    | Temporal | S2        | `InvariantChecker::check_fence_monotonicity`       | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `AlwaysTerminalIrreversibility` | Temporal | S3     | `InvariantChecker::check_terminal_irreversibility` | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `AlwaysCursorNonRegression`  | Temporal | S5        | `InvariantChecker::check_cursor_monotonicity`      | `crates/gossip-coordination/src/sim/invariants.rs`       |
| `Liveness`                   | Liveness | INV-L01   | (no direct Rust equivalent; tested by simulation)  | --                                                       |

---

## Mutation Testing

Mutation testing validates that each invariant is *necessary* -- that weakening
or removing a single guard in the specification causes TLC to find a
counterexample. Without mutation testing, an invariant could be vacuously true
(never exercised) or redundant.

The suite (`specs/coordination/run_mutations.sh`) runs 14 mutations:

| #  | Mutation                                      | Expected Violation              |
|:--:|-----------------------------------------------|---------------------------------|
| 1  | Remove `worker_epoch` cache from Acquire      | `ZombieRejection`               |
| 2  | Remove `status = Active` check from Acquire   | `TerminalUnleased`              |
| 3  | Don't clear owner in Complete                 | `TerminalUnleased`              |
| 4  | Don't activate children in SplitReplace       | `SplitAtomicity`                |
| 5  | Allow `Done -> Active` in Unpark              | `AlwaysTerminalIrreversibility` |
| 6  | Swap cursor/prev_cursor in Checkpoint         | `CursorMonotonicity`            |
| 7  | Non-vacuity: `EventuallyAcquired`             | (should pass)                   |
| 8  | Non-vacuity: `Liveness` (INV-L01)             | (should pass)                   |
| 9  | Keep parent Active in SplitReplace            | `ChildImpliesParentSplit`       |
| 10 | Set children `fence_epoch = 0` in SplitReplace| `FenceEpochSanity`              |
| 11 | Reset epoch to 0 in Unpark                    | `AlwaysFenceMonotonicity`       |
| 12 | Park resets cursor to 0                       | `AlwaysCursorNonRegression`     |
| 13 | Non-vacuity: `NeverSplit` (negation)          | `NeverSplit` violated           |
| 14 | Non-vacuity: `NeverDone` (negation)           | `NeverDone` violated            |

Each safety mutation (1-6, 9-12) creates a copy of the spec with one deliberate
defect, runs TLC, and asserts that the *specific* expected invariant is violated
(not just any error). Non-vacuity checks (7-8, 13-14) confirm that temporal
properties are satisfiable and key states are reachable.

---

## Running TLC

### Prerequisites

- Java 11+ on `PATH`
- `tla2tools.jar` at `specs/tla2tools.jar` (checked into the repository)

### Commands

All commands are run from the repository root.

**Development (fast feedback, ~seconds):**

```bash
java -XX:+UseParallelGC -cp specs/tla2tools.jar tlc2.TLC \
  -workers auto -deadlock \
  -config specs/coordination/ShardFencing_dev.cfg \
  specs/coordination/ShardFencing.tla
```

**Production safety (exhaustive, 3 workers):**

```bash
java -XX:+UseParallelGC -cp specs/tla2tools.jar tlc2.TLC \
  -workers auto -deadlock \
  -config specs/coordination/ShardFencing.cfg \
  specs/coordination/ShardFencing.tla
```

**Liveness verification:**

```bash
java -XX:+UseParallelGC -cp specs/tla2tools.jar tlc2.TLC \
  -workers auto -deadlock -lncheck final \
  -config specs/coordination/ShardFencing_liveness.cfg \
  specs/coordination/ShardFencing.tla
```

**Mutation test suite (all 14 mutations):**

```bash
bash specs/coordination/run_mutations.sh
```

Key flags: `-deadlock` disables TLC's deadlock detection (the spec intentionally
reaches terminal states). `-lncheck final` is required for liveness checking.
SYMMETRY is used for safety configs (workers are interchangeable) but never for
liveness (unsound).

---

## Limitations: TLA+ vs. Simulation

TLA+ and simulation testing are complementary verification layers that catch
different classes of defects.

```
  TLA+ Model Checking             Simulation (Tier 4)
  ─────────────────────           ───────────────────────────
  Verifies: protocol design       Verifies: Rust implementation
  (abstract model)                under fault injection

  Exhaustive within its           Randomized but exercises
  abstraction boundary            real Rust code paths

  Catches: design bugs            Catches: implementation bugs
  (missing guards, wrong          (off-by-one, wrong enum variant,
  state transitions,              hash collisions, edge cases
  impossible liveness)            in split/cursor logic)

  Limitation: abstract model      Limitation: probabilistic,
  may diverge from the            not exhaustive; only as good
  implementation                  as seed coverage
```

What TLA+ cannot check:
- Rust-specific issues (integer overflow, hash collisions, borrow checker edge cases)
- Byte-key lexicographic ordering (abstracted to integer cursors)
- Multi-tenant isolation (single-tenant model)
- Op-log idempotency (orthogonal to fencing protocol)
- Performance characteristics (TLA+ is about correctness, not speed)

What simulation cannot check:
- Exhaustive coverage of all interleavings (simulation is randomized)
- Protocol-level design flaws (simulation exercises Rust code, not the abstract design)
- Liveness under all fair schedulings (simulation uses finite seeds)

A third layer -- runtime assertions (`ShardRecord::validate_invariants`) -- serves
as the last line of defense in production, panicking before state corruption can
propagate.

---

## Summary

- **ShardFencing.tla** exhaustively verifies 8 safety invariants, 3 action
  properties, and 1 liveness property across all reachable states of the
  epoch-based shard fencing protocol.
- Every TLA+ property maps to a specific Rust function: simulation invariant
  checks (S1-S9 in `InvariantChecker`) enforce the same properties at runtime
  against the real implementation.
- A 14-mutation test suite validates that each invariant is necessary -- removing
  any single guard produces a TLC counterexample targeting the specific expected
  property.
- TLA+ verifies the protocol *design*; simulation verifies the Rust
  *implementation*; runtime assertions are the last line of defense. The three
  layers are complementary, not redundant.
