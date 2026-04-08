# What Problem Are We Solving?

## The Secret Sprawl Problem

Modern software development depends on a long tail of external systems: cloud accounts, CI services, artifact registries, internal Git servers, package mirrors, and local deployment tooling. Every one of those systems introduces credentials: API keys, OAuth tokens, service-account blobs, SSH material, database passwords, and signed cookies.

Those secrets leak in predictable ways:

- **Hardcoded in source**: `.env` files, bootstrap scripts, sample configs, and test fixtures
- **Copied into repositories**: forks, mirrors, archives, and long-lived feature branches
- **Left in local or shared storage**: filesystem trees, build artifacts, backups, and exported bundles
- **Embedded in versioned history**: old commits can still carry active credentials even after the current file is clean

Once a secret leaks, the response window is short. The scanner has to find the leak quickly, classify it deterministically, and avoid turning one incident into dozens of duplicate alerts.

## The Scale Challenge

Even the current source tree hints at why this turns into a systems problem rather than a scripting problem: Gossip-rs already has separate runtime paths for filesystem scans, Git scanning, worker orchestration, coordination, and durable persistence.

A single machine eventually loses on three fronts:

- **Enumeration throughput**: large directory trees and Git histories take time to walk
- **Scan throughput**: every candidate item must pass through the detection engine
- **Durability and retry handling**: findings and completion markers must survive crashes and retries

That is why the codebase has both direct local execution and a distributed worker path. The guide spends most of its time on the distributed design because that is where correctness becomes hard.

## The Deduplication Challenge

The same secret can surface in several related but distinct ways:

```text
repo/main branch      -> secret appears in one tracked file
repo/old commit       -> same secret appears in history
repo/archive.tar      -> same bytes are packed into an artifact
repo/fork             -> same material is copied into another logical item
```

A useful scanner has to answer at least three different identity questions:

1. **Did we see the same normalized secret bytes again?**
2. **Is this the same logical finding in the same logical item?**
3. **Is this the same version-specific occurrence, or a new one?**

The current code answers those questions with a layered identity model:

- `StableItemId` identifies the logical item being scanned
- `SecretHash` identifies normalized secret material inside one tenant boundary
- `FindingId` identifies the stable finding `(tenant, item, rule, secret)`
- `OccurrenceId` identifies a version-specific location `(finding, version, offset, length)`

That design lets the system keep stable finding identity without pretending that every appearance of a secret is the same event.

## The Tenant Isolation Requirement

Gossip-rs is explicitly multi-tenant. The code enforces tenant isolation, but not by making every identifier tenant-specific.

The important distinction is:

- `StableItemId` is **tenant-independent**: the same logical item can produce the same stable item identity across tenants
- `SecretHash` is **tenant-scoped**: it is derived by keying a normalized secret hash with `TenantSecretKey`
- `FindingId`, `ObservationId`, done-ledger keys, and findings records all carry tenant scope

That means two tenants can scan the same logical item while still getting cryptographically isolated secret and finding identities:

```text
Tenant A: SecretHash = BLAKE3-keyed(TenantSecretKey_A, NormHash)
Tenant B: SecretHash = BLAKE3-keyed(TenantSecretKey_B, NormHash)

StableItemId_A == StableItemId_B is possible
SecretHash_A != SecretHash_B
FindingId_A != FindingId_B
```

Only `SecretHash` uses BLAKE3 keyed mode. The later identities use derive-key mode over explicit typed inputs such as `TenantId`, `StableItemId`, `RuleFingerprint`, `PolicyHash`, and `OccurrenceId`.

## The Exactly-Once Problem

Distributed scanning is not just "run more workers." Workers can crash after scanning but before checkpointing. Networks can time out after a write commits. A shard can be reclaimed after a lease expires.

The system still wants the practical version of exactly-once behavior:

- **Do not lose completed work**
- **Do not duplicate durable findings when retries happen**
- **Do not advance progress past data that was not durably committed**

Gossip-rs builds that out of two pieces:

- **Coordination-layer idempotency**: per-shard operation replay detection through `OpId` plus payload hashing
- **Persistence-layer durability**: done-ledger and findings sinks, with checkpoint advancement only after durable receipts

That is the recurring theme of the guide: at-least-once execution at the edges, paired with deterministic identity and idempotent durable writes in the middle.

## System Architecture

At a high level, the codebase currently implements this shape:

```mermaid
graph LR
    subgraph Inputs
        FS[Filesystem roots]
        GR[Git repositories / mirrors]
    end

    subgraph ControlPlane
        OR[gossip-orchestrator]
        CO[gossip-coordination]
    end

    subgraph Worker
        WR[gossip-worker]
        RT[gossip-scanner-runtime]
        CN[gossip-connectors<br/>(filesystem path)]
        EN[scanner-engine / scanner-git / scanner-scheduler]
    end

    subgraph Storage
        DL[DoneLedger]
        FI[FindingsSink]
    end

    FS --> OR
    GR --> OR
    OR --> CO
    CO --> WR
    WR --> RT
    RT --> CN
    RT --> OC
    RT --> GT
    RT --> DL
    RT --> FI
    RT --> CO
```

The connector box applies to the filesystem ordered-content path. Git repo-frontier work goes from `gossip-scanner-runtime` into `scanner-git` plus Git-specific shard payloads from `gossip-orchestrator`, not through a concrete Git connector implementation in `gossip-connectors`.

### Component Roles

**Orchestrator**:
- Normalizes filesystem and Git requests
- Plans initial shard geometry
- Encodes shard payload metadata
- Creates runs and registers startup shards through the coordination layer

**Coordination backend**:
- Assigns shard leases to workers
- Enforces fence epochs and bounded idempotency
- Tracks checkpoint, split, park, and completion transitions

**Worker runtime**:
- Claims leases
- Executes the filesystem ordered-content path through `gossip-connectors`, or the separate Git repo-frontier path through `scanner-git`
- Translates scan results into durable persistence writes
- Advances checkpoints only after durable receipts exist

**Persistence backends**:
- `DoneLedger` tracks whether a `(tenant, policy_hash, ovid_hash)` scope has already been durably processed
- `FindingsSink` stores stable findings, versioned occurrences, and policy/run-scoped observations

## Why This Is Hard

The hard part is not pattern matching in isolation. The hard part is getting all of these properties at once:

1. **Scale**: enough throughput to keep up with large trees and repository history
2. **Determinism**: the same bytes should produce the same identities and replay outcomes
3. **Isolation**: tenants must not be able to correlate each other's secrets or findings
4. **Recovery**: retries and reassignment must not create inconsistent durable state
5. **Progress safety**: cursors and checkpoints must only move after durable work

Those constraints are what drive the boundary-oriented design in the rest of the guide.

## What's Next

Now that we understand the problem, let's explore why distribution is necessary:

**[→ Next: 02-why-distributed.md](02-why-distributed.md)**

---

## References

- Akidau, Tyler et al. (2015). "The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing." *VLDB 2015*.
