# Gossip-rs: A From-First-Principles Guide to Building a Distributed Secret Scanner

## Purpose

This is a learning guide that explains the *why* behind every design decision in the Gossip-rs distributed secret scanner. Rather than just documenting what the code does, this guide explores the fundamental problems of distributed systems and shows how Gossip-rs addresses them through concrete, production-ready patterns.

## Target Audience

Engineers who want to understand distributed systems design through a concrete, real-world system. You should be familiar with Rust, and while basic distributed systems knowledge is helpful, it is not required—we build from first principles.

## Prerequisites

- **Rust familiarity**: You should be comfortable reading Rust code including traits, lifetimes, and Result types
- **Basic distributed systems knowledge**: Helpful but not required—we explain concepts from the ground up
- **Curiosity**: This guide digs deep into the theory and practice of distributed coordination

## Project Status

| Boundary | Name | Status | Implementation Details |
|----------|------|--------|------------------------|
| B1 | Identity & Hashing Spine | ✅ Fully Implemented | 11 files, 48 invariants, 9 golden vectors, zero unsafe code |
| B2 | Coordination | ✅ Fully Implemented | 22 source files + 8 contract files in gossip-contracts/coordination, S1-S9 invariants, reference backend, simulation harness (14 sim modules), TLA+ spec, ~35K lines |
| B3 | Shard Algebra | ✅ Fully Implemented | 4 source files in gossip-frontier crate, key encoding, hint metadata, builder |
| B4 | Connector | ✅ Fully Implemented | 5 contract files, 3 concrete connectors (in-memory, filesystem, git) with streaming split estimation, scan-driver adapter, 8 guide chapters |
| B5 | Persistence | 📝 Contract Stubs | Design stage, architecture decision in docs/boundary-5-persistence.md |
| — | Scanner Engine | ✅ Extracted | Detection engine: rule loading, content policy, scan engine, reusable scratch |
| — | Scanner Git | ✅ Extracted | Git scanning pipeline: repo open, commit walk, tree diff, pack decode, blob spill |
| — | Scan Driver | ✅ Implemented | Unified scan-driver boundary for source-specific execution backends |
| — | Scanner Scheduler | ✅ Extracted | Parallel execution runtime: executor, local FS scanning, archive expansion |
| — | Scanner Runtime | ✅ Implemented | Unified CLI and distributed entrypoints backed by ScanDriver |
| — | Worker | ✅ Implemented | Worker binary entrypoint routing scans through scanner-runtime |

## Sequential Reading Path

This guide is designed to be read in order. Each chapter builds on concepts introduced in previous chapters:

1. **00-prologue** — What problem are we solving, and why distributed?
2. **01-foundations** — Core concepts: hashing, content addressing, state machines
3. **02-boundary-1** — Identity & Hashing Spine (B1)
4. **03-distributed-theory** — Coordination primitives: leases, fencing, idempotency
5. **04-boundary-2** — Coordination (B2), split protocol (12 chapters)
6. **05-boundary-3** — Shard Algebra (B3), key encoding, hint metadata, builder (7 chapters)
7. **06-boundary-4** — Connector (B4), toxic byte wrappers, split/read/capability methods, circuit breaker design, in-memory and filesystem connectors, git connector, scan-driver adapters (8 chapters)
8. **07-boundary-5** — Persistence (B5)
9. **08-cross-cutting** — Testing, failure modes, operational concerns
10. **09-appendices** — References, glossary, TLA+ specifications
11. **10-scanner-engine** — Detection engine: rule loading, content policy, scan pipeline
12. **11-scan-driver-and-pipeline** — Unified execution seam for source-specific backends
13. **12-scanner-git** — Git scanning pipeline: repo open, commit walk, tree diff, pack decode
14. **13-scanner-scheduler** — Parallel execution runtime: executor, archive expansion
15. **14-scanner-runtime-and-worker** — Unified CLI and distributed entrypoints

## Topic Jump Table

| Chapter | Topics |
|---------|--------|
| **00-prologue** | Problem statement, scale requirements, architecture overview |
| **01-foundations** | Content addressing, collision-free encoding, type-driven design |
| **02-boundary-1** | TenantId, PolicyHash, StableItemId, FindingId, OccurrenceId, ObjectVersionId, ConnectorInstanceIdHash, ObservationId, deterministic derivation |
| **03-distributed-theory** | CAP theorem, linearizability, consensus, failure detection |
| **04-boundary-2** | The coordination problem, leases and fencing, runs and claiming, acquiring and scanning, idempotency and op-log, completion and parking, shard algebra (half-open ranges, split-replace, split-residual), validation layer, reference implementation, simulation and TLA+ (12 chapters) |
| **05-boundary-3** | Key encoding, range arithmetic, hint metadata, split propagation, startup builder, property tests (7 chapters) |
| **06-boundary-4** | Connector problem space, toxic byte value wrappers, split/read/capability methods, circuit breaker design, in-memory deterministic connector, filesystem connector with streaming split estimation, git connector, scan-driver adapters (8 chapters) |
| **07-boundary-5** | Done ledger, exactly-once semantics, commit protocol, WAL design |
| **08-cross-cutting** | Deterministic simulation testing, chaos engineering, observability |
| **09-appendices** | 30+ academic papers, 70+ Mermaid diagrams, TLA+ specs |
| **10-scanner-engine** | Rule loading, content policy, scan engine, scratch memory, pool management |
| **11-scan-driver** | ScanSourceFactory, ScanDriver, assignment-to-driver seam |
| **12-scanner-git** | Repo open, commit walk, tree diff, pack decode, blob spill, seen store, engine adapter |
| **13-scanner-scheduler** | Executor, local FS scanning, archive expansion, scheduling primitives |
| **14-scanner-runtime** | CLI and distributed mode, config-to-assignment mapping, parity invariant |

## What Makes This Guide Different

- **First Principles**: We derive every design decision from fundamental constraints
- **Theory Meets Practice**: Academic papers cited inline with production code references
- **Complete Implementation**: B1 is fully implemented with 48 invariants, 9 golden vectors, and exhaustive property tests
- **Spec-to-Implementation Journey**: B2 is fully implemented with the normative spec → code journey documented
- **Complete Simulation Harness**: Deterministic simulation testing documented with actual code walkthroughs covering S1-S9 invariants
- **Real Production Constraints**: Multi-tenant isolation, exactly-once semantics, zero data loss

## How to Use This Guide

### For Learners

Read sequentially from Chapter 0 through Chapter 9. Work through the exercises in each chapter. Study the Mermaid diagrams and trace through the state machines.

### For Implementers

Start with the prologue to understand the problem space, then jump to the boundary chapters (02, 04, 05, 06) for implementation contracts and invariants.

### For Reviewers

Focus on Chapter 03 (distributed theory) and Chapter 07 (testing strategy) to understand the correctness arguments.

## Documentation Stats

- **106 guide chapters** across 15 sections (prologue through scanner runtime)
- **30+ academic papers** referenced with inline citations
- **70+ Mermaid diagrams** showing data flow, state machines, and system interactions
- **Direct code references** to actual implementation across `crates/gossip-contracts/`, `crates/gossip-coordination/`, `crates/gossip-connectors/`, `crates/gossip-frontier/`, `crates/scanner-engine/`, `crates/gossip-scan-driver/`, `crates/scanner-git/`, `crates/scanner-scheduler/`, `crates/gossip-scanner-runtime/`, `crates/gossip-worker/`, and `crates/scanner-rs-cli/`
- **Safety and liveness invariants** with formal specifications across all implemented boundaries

## Navigation

- Start with [00-prologue/01-what-problem-are-we-solving.md](00-prologue/01-what-problem-are-we-solving.md)
- See [REFERENCES.md](REFERENCES.md) for complete bibliography
- See [09-appendices/D-glossary.md](09-appendices/D-glossary.md) for terminology

---

**License**: This guide and the Gossip-rs project are open source. See LICENSE for details.

**Contributing**: Found an error or have a question? Open an issue in the Gossip-rs repository.
