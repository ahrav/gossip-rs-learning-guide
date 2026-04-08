# Why Distributed?

## The Central Question

Could Gossip-rs stay a single-process scanner?

For direct local scans, yes. The source tree still supports local filesystem and Git entrypoints. But the moment we want durable multi-worker progress, lease handoff, and bounded replay across failures, a single-process design stops being enough.

The pressure comes from four constraints.

## Constraint 1: Scale Drives Distribution

A single process can scan bytes. It cannot indefinitely keep up with:

- large filesystem trees,
- deep Git history,
- repeated re-scans after crashes or retries,
- and durable writes that must happen without blocking future work forever.

The scanner crates in the repo already reflect that split:

- `scanner-engine` owns the detection engine
- `scanner-scheduler` owns high-throughput filesystem execution
- `scanner-git` owns the Git pipeline
- `gossip-scanner-runtime` composes those pieces into direct and distributed worker paths

That separation exists because throughput and durability are no longer one concern.

**Distribution is what lets the system scale scan execution and failure handling independently.**

## Constraint 2: Failure Isolation

A monolith has a large blast radius. A distributed system can stop one bad shard, one bad repo, or one bad connector path without stopping everything else.

In the current code, the failure boundaries are concrete:

- ordered-content shards can fail independently of Git repo-frontier shards,
- connector errors carry a retry posture through `ErrorClass`,
- coordination can park a shard or let a lease expire without corrupting the rest of the run,
- durable writes are isolated behind persistence interfaces instead of being fused into the scan loop.

That means one failure does not have to become a global stop-the-world event.

## Constraint 3: Work Partitioning

The current system does **not** partition all work with one universal shard shape.

Instead, it has two partitioning families:

- **Filesystem ordered-content shards** use half-open key ranges stored in `ShardSpec`, durable progress in `Cursor`, and `gossip-frontier` encodings such as `KeyEncoding`, `PathKey`, and `ManifestRowKey`
- **Git repo-frontier shards** start as exact singleton shards: one normalized repo target per startup shard, later claimed and executed by `run_git_repo_worker`

For filesystem work, a run eventually looks like ordered keyspace ownership:

```text
Shard A: [apps/, infra/)
Shard B: [infra/, services/)
Shard C: [services/, end)
```

The current startup planner does **not** pre-split filesystem requests into manifest-row ranges. It emits one full-range shard per normalized filesystem request, and later split flows carve that space into smaller ordered ranges when needed.

For Git work, startup sharding is exact-key ownership:

```text
Shard A: [repo-key-a, repo-key-a\0)
Shard B: [repo-key-b, repo-key-b\0)
```

This matters because a filesystem worker asks "is this ordered key still inside my shard range, and where should my cursor resume?", while a Git worker asks "which exact repo target does this singleton shard own, and has its durable finalize step completed?"

### Why Range-Based Sharding?

Range-based shards are a good fit for the filesystem path that exists today:

1. **Ordered enumeration**: connectors already expose ordered pages and resumable cursors
2. **Cheap splitting**: `gossip-frontier` can compute successor keys and split boundaries without rehashing the world
3. **Coverage checks**: half-open ranges make it possible to reason about gaps, overlaps, and split correctness

That is why `gossip-frontier` exists as its own boundary instead of burying shard math inside one connector. Git uses the same coordination substrate, but its startup planner currently emits exact singleton repo-frontier shards rather than resumable ordered-content ranges.

## Constraint 4: Exactly-Once Processing

Distributed work becomes complicated when a worker can crash in the middle of a shard.

The practical failure story looks like this:

1. A worker claims a shard lease
2. It scans some items
3. It submits findings and completion state to durable backends
4. It crashes before the next checkpoint is safely recorded
5. Another worker later reclaims the shard

To make retries safe, Gossip-rs uses two layers of protection.

### Coordination-layer idempotency

Each shard keeps a bounded FIFO op-log. The replay key is not just "same operation name"; it is the pair of:

```text
(OpId, payload_hash)
```

That lets the coordinator distinguish:

- a true replay of the same operation,
- a buggy reuse of the same `OpId` with different parameters,
- and a genuinely new operation.

### Persistence-layer durability

The done ledger and findings sink absorb duplicate attempts through deterministic identities and idempotent writes.

The done-ledger key is not "one row per item." It is a tenant- and policy-scoped object-version identity:

```text
(TenantId, PolicyHash, OvidHash)
```

where `OvidHash` is derived from `(StableItemId, VersionId)`.

The worker runtime then advances checkpoints only after durable receipts come back from the persistence path. That is the bridge from at-least-once execution to exactly-once-effective progress.

**Distribution requires replay-safe coordination and receipt-driven durable progress.**

## Putting It Together

Distribution is not a flourish. It is the mechanism that makes the current architecture viable.

| Constraint | Why Distribution | Gossip-rs Solution |
|------------|------------------|-------------------|
| **Scale** | Scan execution, Git history traversal, and durability must progress independently | Separate runtime crates and worker paths |
| **Failure Isolation** | One bad shard or connector path should not stop the rest of the run | Lease-based coordination, shard parking, and isolated worker execution |
| **Work Partitioning** | Workers need resumable ownership of filesystem key ranges and exact ownership of Git repo targets | Filesystem: `ShardSpec` + `Cursor` + `gossip-frontier`; Git: singleton repo-frontier shards |
| **Exactly-Once** | Crashes and retries are inevitable | Op-log idempotency + idempotent persistence + receipt-driven checkpointing |

## The Coordination Layer

To make all of that work, the system needs a coordination layer that:

1. Assigns shards to workers through time-bounded leases
2. Detects stale owners with fence epochs
3. Supports checkpoint, split, park, and completion transitions
4. Lets workers retry safely without duplicating state transitions

That is **Boundary 2 (Coordination)**, covered in Section 04 of the guide.

## What's Next

Now that we understand why distribution is necessary, let's look at the overall architecture:

**[→ Next: 03-architecture-at-a-glance.md](03-architecture-at-a-glance.md)**

---

## References

- Corbett, James C. et al. (2012). "Spanner: Google's Globally-Distributed Database." *OSDI 2012*.
- Akidau, Tyler et al. (2015). "The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing." *VLDB 2015*.
