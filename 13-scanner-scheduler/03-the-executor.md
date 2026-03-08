# "The Lost Task" -- The Work-Stealing Executor

*Worker 2 has 847 tasks in its local deque. Workers 0, 1, and 3 are parked -- their deques are empty, the global injector is empty, and their steal attempts returned `Steal::Empty`. Meanwhile, an external I/O completion thread calls `handle.spawn(task)`. It loads the state word: `accepting == true`, `in_flight == 847`. It begins the CAS to increment the count. Between the load and the CAS, the main thread calls `executor.join()`, which atomically clears the accepting bit via `fetch_and(!1)`. The spawner's CAS succeeds -- it incremented a state word where accepting was still set. The task enters the injector. But `join()` has already started draining. If `join()` had observed `in_flight == 0` at the moment it closed the gate, it would signal termination. The workers would exit. The spawned task would sit in the injector forever, never executed. This is the TOCTOU race between spawn and join, and the combined atomic state machine exists to prevent it.*

---

The executor is the scheduler's CPU engine: a work-stealing thread pool where N workers pop tasks from local Chase-Lev deques, steal from siblings when idle, and receive external work through a global injector queue. This chapter traces the executor from configuration through the worker lifecycle to termination detection.

## 1. ExecutorConfig

Every executor starts with a configuration struct. From `executor.rs`:

```rust
/// Executor configuration.
///
/// All defaults are conservative. Profile with your workload before tuning.
#[derive(Clone, Copy, Debug)]
pub struct ExecutorConfig {
    /// Number of worker threads.
    pub workers: usize,

    /// Seed for deterministic victim selection.
    ///
    /// Same seed + same task spawn order = reproducible steal pattern (modulo timing).
    pub seed: u64,

    /// Steal attempts before giving up per idle cycle.
    pub steal_tries: u32,

    /// Spin iterations before yielding/parking.
    pub spin_iters: u32,

    /// Park timeout after spinning/yielding.
    pub park_timeout: Duration,

    /// Try to pin each worker to a core (Linux only).
    pub pin_threads: bool,
}

impl Default for ExecutorConfig {
    fn default() -> Self {
        Self {
            workers: 1,
            seed: 0x853c49e6748fea9b,
            steal_tries: 4,
            spin_iters: 200,
            park_timeout: Duration::from_micros(200),
            pin_threads: super::affinity::default_pin_threads(),
        }
    }
}
```

The `seed` field controls all randomized behavior. Combined with a single-worker configuration, the executor produces deterministic execution order -- identical across runs, platforms, and machines.

## 2. The Combined Atomic State Machine

The executor uses a single `AtomicUsize` to encode both the in-flight task count and the accepting flag. The state layout from `executor_core.rs`:

```rust
/// LSB in the combined state: 1 when accepting external spawns.
pub(crate) const ACCEPTING_BIT: usize = 1;
/// Count unit for the combined state (count stored in bits 1+).
pub(crate) const COUNT_UNIT: usize = 2;
```

```text
State = (in_flight_count << 1) | accepting_bit

       63                              1   0
      ┌─────────────────────────────┬─────┐
      │      in_flight_count        │ A   │
      └─────────────────────────────┴─────┘
                                      │
                                      └── accepting bit (1=open, 0=closed)
```

The helper functions are minimal:

```rust
/// Extract the in-flight count from the combined state word.
#[inline(always)]
pub(crate) fn in_flight(state: usize) -> usize {
    state >> 1
}

/// Check whether the executor is accepting external spawns.
#[inline(always)]
pub(crate) fn is_accepting(state: usize) -> bool {
    (state & ACCEPTING_BIT) != 0
}

/// Clear the accepting bit and return the previous state word.
#[inline(always)]
pub(crate) fn close_gate(state: &AtomicUsize) -> usize {
    state.fetch_and(!ACCEPTING_BIT, Ordering::AcqRel)
}
```

The combined encoding eliminates the TOCTOU race. An external spawn uses a CAS loop that atomically checks accepting AND increments the count:

```rust
#[inline]
pub fn spawn(&self, task: T) -> Result<(), T> {
    // CAS loop: atomically check accepting AND increment count
    let mut s = self.shared.state.load(Ordering::Acquire);
    loop {
        if !is_accepting(s) {
            return Err(task);
        }
        // Try to increment count (add 2 because count is in bits 1+)
        match self.shared.state.compare_exchange_weak(
            s,
            s.wrapping_add(COUNT_UNIT),
            Ordering::AcqRel,
            Ordering::Acquire,
        ) {
            Ok(_) => break,
            Err(actual) => s = actual,
        }
    }

    self.shared.injector.push(task);
    self.shared.unpark_one();
    Ok(())
}
```

If `join()` clears the accepting bit between the load and the CAS, the CAS fails because the expected value no longer matches. The spawner reloads, sees `!is_accepting`, and returns `Err(task)`.

## 3. XorShift64 -- Deterministic Victim Selection

Work stealing requires choosing a victim worker. The scheduler uses a per-worker XorShift64 RNG. From `rng.rs`:

```rust
/// Deterministic RNG for scheduling decisions.
///
/// # No Copy
///
/// Intentionally does not implement `Copy` to prevent accidental
/// stream duplication. Use `clone()` explicitly if needed.
#[derive(Clone, Debug)]
pub struct XorShift64 {
    state: u64,
}

impl XorShift64 {
    /// Create a new RNG with the given seed.
    ///
    /// Seed 0 is mapped to a non-zero value to avoid the all-zero lockup state.
    #[inline]
    pub fn new(seed: u64) -> Self {
        let seed = if seed == 0 { 0x9E3779B97F4A7C15 } else { seed };
        Self { state: seed }
    }

    /// Generate the next u64 value using XorShift64.
    ///
    /// The shift constants (13, 7, 17) are from Marsaglia's "Xorshift RNGs" paper
    /// and produce a full-period generator (2^64 - 1 values before repeating).
    #[inline]
    pub fn next_u64(&mut self) -> u64 {
        let mut x = self.state;
        x ^= x << 13;
        x ^= x >> 7;
        x ^= x << 17;
        self.state = x;
        x
    }
```

Bounded sampling uses Lemire's fast-range method instead of modulo division:

```rust
/// Generate a random usize in `[0, upper)`.
///
/// Uses Lemire's fast range method (multiplication instead of division).
/// Power-of-two bounds use a bitmask fast path.
#[inline]
pub fn next_usize(&mut self, upper: usize) -> usize {
    debug_assert!(upper > 0, "upper bound must be > 0");

    // Power-of-two fast path: bitmask instead of any arithmetic
    if upper.is_power_of_two() {
        return (self.next_u64() as usize) & (upper - 1);
    }

    // Lemire's method: multiply-high, no division
    self.bounded_u64(upper as u64) as usize
}
```

The performance difference matters: modulo compiles to `udiv` on ARM (15-40 cycles), while Lemire's method uses multiplication (3-4 cycles) with one unconditional threshold division for rejection sampling. For 4-16 worker configurations, the power-of-two fast path (1 cycle bitmask) is the common case.

Per-worker RNGs are forked from the master seed using splitmix64 as a mixer:

```rust
let rng_seed =
    thread_cfg.seed ^ (worker_id as u64).wrapping_mul(0x9E3779B97F4A7C15);
```

## 4. WorkerCtx -- Per-Worker Context

Every task receives a `WorkerCtx` that provides access to the local deque, scratch space, RNG, and metrics. From `executor.rs`:

```rust
/// Per-worker context passed to every task execution.
pub struct WorkerCtx<T, S> {
    /// Worker ID (0..workers).
    pub worker_id: usize,
    /// User-defined per-worker scratch space.
    pub scratch: S,

    /// Per-worker RNG for randomized stealing.
    pub rng: XorShift64,
    /// Per-worker metrics (no cross-thread contention).
    pub metrics: WorkerMetricsLocal,

    local: Worker<T>,
    parker: Parker,
    shared: Arc<Shared<T>>,

    /// Counter for wake-on-hoard heuristic.
    local_spawns_since_wake: u32,
}
```

Two spawn methods expose different performance profiles:

```rust
/// Spawn a task locally (preferred).
///
/// Task goes to this worker's local queue. Other workers can steal it,
/// but the local worker tries it first. Maximizes cache locality.
#[inline]
pub fn spawn_local(&mut self, task: T) {
    increment_count(&self.shared.state);
    perf_stats::sat_add_u64(&mut self.metrics.tasks_enqueued, 1);
    self.local.push(task);

    // Wake-on-hoard: if we've spawned many tasks locally, wake a sibling
    self.local_spawns_since_wake += 1;
    if self.local_spawns_since_wake >= WAKE_ON_HOARD_THRESHOLD {
        self.local_spawns_since_wake = 0;
        self.shared.unpark_one();
    }
}
```

**`spawn_local` is the fast path.** No CAS needed for the accepting check (workers only run while the executor is live), just a single `fetch_add` on the combined state. The wake-on-hoard heuristic prevents one worker from accumulating thousands of tasks while siblings sleep.

The hoard threshold is 32, documented in `executor_core.rs`:

```rust
/// Threshold for local spawns before waking a sibling.
///
/// # Why 32?
///
/// | Threshold | Wakeup Rate | Overhead | Tail Latency |
/// |-----------|-------------|----------|--------------|
/// | 8 | High | ~12.5% of spawns trigger syscall | Low |
/// | 32 | Medium | ~3% of spawns trigger syscall | Medium |
/// | 128 | Low | ~0.8% of spawns trigger syscall | Higher |
///
/// 32 balances responsiveness with syscall overhead.
pub(crate) const WAKE_ON_HOARD_THRESHOLD: u32 = 32;
```

## 5. Task Popping Priority

The `pop_task` function in `executor_core.rs` defines a three-level priority:

```rust
/// Try to pop a task: local first, then injector, then steal.
#[inline(always)]
pub(crate) fn pop_task<T, S, C>(
    steal_tries: u32,
    ctx: &mut C,
) -> Option<(T, PopSource, Option<usize>)>
where
    C: WorkerCtxLike<T, S>,
{
    // 1) Local fast path (LIFO, best cache locality)
    if let Some(t) = ctx.pop_local() {
        return Some((t, PopSource::Local, None));
    }

    // 2) Global injector (batch steal reduces contention)
    if let Some(t) = ctx.steal_from_injector() {
        return Some((t, PopSource::Injector, None));
    }

    // 3) Randomized victim stealing
    let n = ctx.worker_count();
    if n <= 1 {
        return None;
    }

    for _ in 0..steal_tries {
        // Pick random victim, excluding self
        let mut victim = ctx.rng_next_usize(n - 1);
        if victim >= ctx.worker_id() {
            victim += 1;
        }

        if let Some(t) = ctx.steal_from_victim(victim) {
            ctx.record_steal_success();
            return Some((t, PopSource::Steal, Some(victim)));
        }

        ctx.record_steal_attempt();
    }

    None
}
```

**Local pop (LIFO)** gives the best cache locality -- the most recently pushed task likely operates on data still in L1/L2. **Injector batch steal** amortizes global queue access by moving multiple tasks to the local deque. **Randomized victim selection** avoids correlated contention: round-robin stealing causes "thundering herd" when all workers are idle and all try to steal from Worker 0 simultaneously.

The victim formula `victim = rng.next_usize(n-1); if victim >= self { victim += 1 }` ensures uniform distribution over the remaining N-1 workers without ever selecting self.

## 6. The Worker Loop

The worker loop delegates to `worker_step` for policy, keeping production and simulation in lockstep:

```rust
fn worker_loop<T, S, RunnerFn>(
    cfg: ExecutorConfig,
    runner: &Arc<RunnerFn>,
    ctx: &mut WorkerCtx<T, S>,
) where
    T: Send + 'static,
    RunnerFn: Fn(T, &mut WorkerCtx<T, S>) + Send + Sync + 'static,
{
    let mut idle = TieredIdle::new();
    let mut trace = NoopTrace;
    let mut run = |task: T, ctx: &mut WorkerCtx<T, S>| (runner)(task, ctx);

    loop {
        match worker_step(&cfg, &mut run, ctx, &mut idle, &mut trace) {
            WorkerStepResult::RanTask { .. } => {
                perf_stats::sat_add_u64(&mut ctx.metrics.tasks_executed, 1);
            }
            WorkerStepResult::NoWork => {
                ctx.metrics.idle_spins = ctx.metrics.idle_spins.saturating_add(1);
            }
            WorkerStepResult::ShouldPark { timeout } => {
                ctx.metrics.park_count = ctx.metrics.park_count.saturating_add(1);
                ctx.parker.park_timeout(timeout);
            }
            WorkerStepResult::ExitDone | WorkerStepResult::ExitPanicked => break,
        }
    }
}
```

The tiered idle strategy transitions from spinning to parking:

```rust
impl IdleHooks for TieredIdle {
    fn on_work(&mut self) {
        self.idle_rounds = 0;
    }

    fn on_idle(&mut self, cfg: &ExecutorConfig) -> IdleAction {
        self.idle_rounds = self.idle_rounds.saturating_add(1);

        if self.idle_rounds <= cfg.spin_iters {
            std::hint::spin_loop();
            return IdleAction::Continue;
        }

        // Yield every 16th iteration past the spin threshold.
        if (self.idle_rounds & 0xF) == 0 {
            self.yield_count = self.yield_count.saturating_add(1);
            thread::yield_now();
        }

        IdleAction::Park {
            timeout: cfg.park_timeout,
        }
    }
}
```

**Spin** (first 200 iterations): best for bursty workloads where work arrives within nanoseconds. **Yield** (every 16th iteration past threshold): gives other threads a chance without a full context switch. **Park** (after spin threshold): `park_timeout(200µs)` sleeps until unparked or timeout, minimizing CPU waste.

## 7. Termination Detection

The `worker_step` function in `executor_core.rs` handles the critical termination path:

```rust
let prev_state = ctx.shared_state_fetch_sub(COUNT_UNIT);
let prev_count = in_flight(prev_state);

#[cfg(any(test, feature = "scheduler-sim"))]
assert!(prev_count > 0, "in_flight underflow");

if prev_count == 1 && !is_accepting(prev_state) {
    ctx.initiate_done();
}
```

The last task to complete (count goes from 1 to 0) while the gate is closed triggers termination. `initiate_done` sets the `done` flag and wakes all workers:

```rust
/// Signal all workers to stop.
fn initiate_done(&self) {
    self.done.store(true, Ordering::Release);
    self.unpark_all();
}
```

Panics are caught and isolated:

```rust
let res = panic::catch_unwind(panic::AssertUnwindSafe(|| {
    (runner)(task, ctx);
}));

if let Err(p) = res {
    ctx.shared_state_fetch_sub(COUNT_UNIT);
    ctx.record_panic(p);
    return WorkerStepResult::ExitPanicked;
}
```

The in-flight count is decremented even on panic, and only the first panic is captured -- subsequent panics are discarded to ensure `join()` propagates a deterministic error.

## 8. Loom Verification

The state machine's correctness is verified under all possible interleavings using Loom. From `executor_core.rs`:

```rust
/// Concurrent spawn vs join: either spawn succeeds (before close) or
/// fails (after close). The combined state must remain consistent.
#[test]
fn spawn_vs_join_gate_atomicity() {
    loom::model(|| {
        let state = Arc::new(AtomicUsize::new(ACCEPTING_BIT));
        let s2 = Arc::clone(&state);

        let h = thread::spawn(move || try_spawn(&s2));

        // Main: close gate (join)
        let prev = close_gate(&state);
        let main_saw_count = in_flight(prev);

        let spawn_ok = h.join().unwrap();

        let final_state = state.load(Ordering::Acquire);
        let final_count = in_flight(final_state);

        assert!(!is_accepting(final_state), "gate must be closed after join");

        if spawn_ok {
            assert_eq!(final_count, 1, "spawn succeeded → count must be 1");
        } else {
            assert_eq!(final_count, main_saw_count, "spawn failed → count unchanged");
        }
    });
}
```

Loom explores every interleaving of the two threads. The assertion proves that the combined atomic always reaches a consistent terminal state: either the spawn succeeded and the count reflects it, or the spawn failed and the count is unchanged.

## What's Next

[Chapter 4](04-runtime-and-chunks.md) examines the runtime infrastructure built on top of the executor: the columnar `FileTable`, the fixed-capacity `BufferPool` with per-worker local queues, and the chunking system with overlap-based deduplication that ensures no secret is missed at boundaries.
