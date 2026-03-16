# Appendix E: Source File Map

This appendix maps source files to the chapters that reference them, enabling quick navigation from documentation to code and vice versa.

## Source File → Chapter Map

### Identity System (Boundary 1)

`crates/gossip-contracts/src/identity/` (11 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-contracts/src/identity/mod.rs` | 02-01, 02-10 | Module structure, public API, re-exports |
| `crates/gossip-contracts/src/identity/types.rs` | 01-04, 02-04 | Newtype definitions: TenantId, etc. (32-byte types via `define_id_32!`) |
| `crates/gossip-contracts/src/identity/canonical.rs` | 01-02, 02-02 | CanonicalBytes trait, deterministic encoding |
| `crates/gossip-contracts/src/identity/hashing.rs` | 01-01, 02-02 | BLAKE3 wrappers, hasher initialization, domain separation |
| `crates/gossip-contracts/src/identity/domain.rs` | 01-03, 02-03 | Domain tag constants, derive-key helpers |
| `crates/gossip-contracts/src/identity/item.rs` | 02-05 | StableItemId, ItemKey, ObjectVersionId, ConnectorTag |
| `crates/gossip-contracts/src/identity/finding.rs` | 02-06 | FindingId, OccurrenceId, NormHash, SecretHash, RuleFingerprint |
| `crates/gossip-contracts/src/identity/policy.rs` | 02-07 | PolicyHash, RuleFingerprint |
| `crates/gossip-contracts/src/identity/macros.rs` | 01-04, 02-08 | `define_id_32!`, `define_id_32_restricted!`, `define_id_64!` macros |
| `crates/gossip-contracts/src/identity/golden.rs` | 02-09 | Golden vector tests, regression prevention |
| `crates/gossip-contracts/src/identity/coordination.rs` | 02-04, 04-01 | Coordination identity types: `RunId`, `ShardId`, `WorkerId`, `OpId`, `JobId` (all `u64` via `define_id_64!`), plus `FenceEpoch`, `LogicalTime`, `ShardKey` |

### Coordination Contracts (Boundary 2 — contract types)

`crates/gossip-contracts/src/coordination/` (8 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-contracts/src/coordination/mod.rs` | 04-01 | Module structure, public API, re-exports |
| `crates/gossip-contracts/src/coordination/shard_spec.rs` | 04-02, 04-10 | ShardSpec, from-spec-to-implementation |
| `crates/gossip-contracts/src/coordination/shard_spec_tests.rs` | 04-02 | ShardSpec unit tests |
| `crates/gossip-contracts/src/coordination/cursor.rs` | 04-04 | Cursor monotonicity, cursor types |
| `crates/gossip-contracts/src/coordination/pooled.rs` | 04-01, 04-11 | PooledShardSpec, PooledCursor, arena-pooled shard fields |
| `crates/gossip-contracts/src/coordination/split.rs` | 04-06 | Shard splitting, dynamic work distribution |
| `crates/gossip-contracts/src/coordination/manifest.rs` | 04-01 | RunManifest definition and metadata |
| `crates/gossip-contracts/src/coordination/limits.rs` | 04-01 | Coordination limit constants |

### gossip-coordination (Boundary 2 — runtime)

`crates/gossip-coordination/src/` (26 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-coordination/src/lib.rs` | 04-01 | Crate root, module declarations |
| `crates/gossip-coordination/src/traits.rs` | 04-08 | CoordinationBackend trait definition |
| `crates/gossip-coordination/src/record.rs` | 04-02 | ShardRecord, shard state machine |
| `crates/gossip-coordination/src/record_tests.rs` | 04-02 | ShardRecord unit tests |
| `crates/gossip-coordination/src/lease.rs` | 04-03, 04-14 | Lease acquisition, TTL, expiry, claim cooldown |
| `crates/gossip-coordination/src/run.rs` | 04-07, 04-16 | RunManifest, run lifecycle, run management |
| `crates/gossip-coordination/src/run_errors.rs` | 04-18 | Run-specific error types |
| `crates/gossip-coordination/src/run_tests.rs` | 04-07 | Run lifecycle tests |
| `crates/gossip-coordination/src/error.rs` | 04-18 | Error taxonomy for coordination |
| `crates/gossip-coordination/src/error_tests.rs` | 04-18 | Error taxonomy tests |
| `crates/gossip-coordination/src/session.rs` | 04-09 | WorkerSession, session management |
| `crates/gossip-coordination/src/session_tests.rs` | 04-09 | WorkerSession tests |
| `crates/gossip-coordination/src/facade.rs` | 04-13 | CoordinationFacade super-trait, ShardClaiming, high-level API |
| `crates/gossip-coordination/src/facade_tests.rs` | 04-13 | Facade tests |
| `crates/gossip-coordination/src/events.rs` | 04-15 | Events system, coordination event types |
| `crates/gossip-coordination/src/validation.rs` | 04-17 | Validation layer, 5-check preamble |
| `crates/gossip-coordination/src/in_memory.rs` | 04-11 | InMemoryCoordinator reference implementation |
| `crates/gossip-coordination/src/in_memory_tests.rs` | 04-11, 04-12 | In-memory backend tests |
| `crates/gossip-coordination/src/in_memory_filter_tests.rs` | 04-12 | In-memory filter tests |
| `crates/gossip-coordination/src/in_memory_run_tests.rs` | 04-12 | In-memory run tests |
| `crates/gossip-coordination/src/conformance.rs` | 04-12 | Conformance test infrastructure and reusable harness |
| `crates/gossip-coordination/src/conformance_tests.rs` | 04-12 | Conformance test suite |
| `crates/gossip-coordination/src/scenario_tests.rs` | 04-12 | Scenario-based integration tests |
| `crates/gossip-coordination/src/test_fixtures.rs` | 04-12 | Shared test fixtures and helpers |
| `crates/gossip-coordination/src/shard_limits.rs` | 04-01 | Shard limit constants and capacity constraints |
| `crates/gossip-coordination/src/split_execution.rs` | 04-06 | Split execution logic |

#### gossip-coordination Simulation Harness

`crates/gossip-coordination/src/sim/` (14 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-coordination/src/sim/mod.rs` | 08-05 | Simulation module structure, SimContext, FaultConfig |
| `crates/gossip-coordination/src/sim/harness.rs` | 08-05 | CoordinationSim, three-phase deterministic test driver |
| `crates/gossip-coordination/src/sim/harness_tests.rs` | 08-05 | CoordinationSim tests |
| `crates/gossip-coordination/src/sim/backend.rs` | 08-05 | SimulationBackend trait |
| `crates/gossip-coordination/src/sim/worker.rs` | 08-05 | SimWorker, per-worker bookkeeping |
| `crates/gossip-coordination/src/sim/fault_injector.rs` | 08-05 | FaultInjectingIntrospector, chaos testing |
| `crates/gossip-coordination/src/sim/invariants.rs` | 08-05 | InvariantChecker, invariants S1-S9 |
| `crates/gossip-coordination/src/sim/invariants_tests.rs` | 08-05 | Invariant checker tests |
| `crates/gossip-coordination/src/sim/test_util.rs` | 08-05 | Simulation test utilities |
| `crates/gossip-coordination/src/sim/mega_sim_tests.rs` | 08-05 | Large-scale simulation tests (100+ seed sweep) |
| `crates/gossip-coordination/src/sim/sim_behavioral_tests.rs` | 08-05 | Behavioral simulation tests, deterministic replay |
| `crates/gossip-coordination/src/sim/proptest_state_machine_tests.rs` | 08-05 | Proptest state-machine coordination tests |
| `crates/gossip-coordination/src/sim/overload.rs` | 08-05 | Overload simulation scenarios |
| `crates/gossip-coordination/src/sim/overload_tests.rs` | 08-05 | Overload simulation tests |

### gossip-coordination-etcd (Boundary 2 — etcd backend)

`crates/gossip-coordination-etcd/src/` (16 files: 12 top-level + 4 in `backend/`)

etcd-backed coordination backend implementing the full coordination trait surface (`CoordinationBackend`, `RunManagement`, `ShardClaiming`) against a real etcd cluster. Production-path alternative to the in-memory coordinator.

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-coordination-etcd/src/lib.rs` | 04-08 | Crate root, module declarations, `EtcdCoordinator` re-export |
| `crates/gossip-coordination-etcd/src/backend.rs` | 04-08, 04-11 | `EtcdCoordinator`: owns etcd client, health-check, trait method dispatch |
| `crates/gossip-coordination-etcd/src/backend/coordinator.rs` | 04-08 | Core coordinator implementation |
| `crates/gossip-coordination-etcd/src/backend/run_management.rs` | 04-08 | Run lifecycle operations against etcd |
| `crates/gossip-coordination-etcd/src/backend/shard_coordination.rs` | 04-08 | Shard lifecycle operations against etcd |
| `crates/gossip-coordination-etcd/src/backend/test_support.rs` | 04-12 | Test support utilities for the etcd backend |
| `crates/gossip-coordination-etcd/src/behavioral_conformance.rs` | 04-12 | Behavioral conformance tests for the etcd backend |
| `crates/gossip-coordination-etcd/src/config.rs` | 04-08 | Validated connection parameters (endpoints, namespace prefix) |
| `crates/gossip-coordination-etcd/src/keyspace.rs` | 04-08 | Deterministic ASCII etcd path construction for runs, shards, leases, indexes |
| `crates/gossip-coordination-etcd/src/codec.rs` | 04-08 | Binary encoding/decoding of coordination records to etcd values |
| `crates/gossip-coordination-etcd/src/codec_tests.rs` | 04-08 | Codec roundtrip and compatibility tests |
| `crates/gossip-coordination-etcd/src/error.rs` | 04-18 | etcd-specific error types and mapping to coordination errors |
| `crates/gossip-coordination-etcd/src/sim_coordinator.rs` | 08-05 | Simulation coordinator for deterministic testing of the etcd backend |
| `crates/gossip-coordination-etcd/src/sim_etcd_kv.rs` | 08-05 | Simulated etcd key-value store for deterministic testing |
| `crates/gossip-coordination-etcd/src/test_support.rs` | 04-12 | Shared test support utilities |
| `crates/gossip-coordination-etcd/src/tests.rs` | 04-12 | Integration tests for the etcd backend |

### gossip-frontier (Boundary 3: Shard Algebra)

`crates/gossip-frontier/src/` (7 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-frontier/src/lib.rs` | 05-01 | Crate root, `#![forbid(unsafe_code)]`, public re-exports |
| `crates/gossip-frontier/src/key_encoding.rs` | 05-02 | `KeyEncoding` trait, `PathKey`, `ManifestRowKey`, range arithmetic |
| `crates/gossip-frontier/src/key_encoding_tests.rs` | 05-02 | Property tests and edge cases |
| `crates/gossip-frontier/src/hint.rs` | 05-03, 05-04 | `ShardHint`/`ShardMetadata` wire framing, split propagation |
| `crates/gossip-frontier/src/hint_tests.rs` | 05-03, 05-04 | Roundtrip and propagation tests |
| `crates/gossip-frontier/src/builder.rs` | 05-05 | `PreallocShardBuilder` two-phase startup builder |
| `crates/gossip-frontier/src/builder_tests.rs` | 05-05 | Builder behavioral tests |

### Connector Contracts (Boundary 4)

`crates/gossip-contracts/src/connector/` (9 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-contracts/src/connector/mod.rs` | 06-01 | Module structure, public API, re-exports |
| `crates/gossip-contracts/src/connector/api.rs` | 06-03 | Error taxonomy, capability negotiation |
| `crates/gossip-contracts/src/connector/api_tests.rs` | 06-03 | Connector API tests |
| `crates/gossip-contracts/src/connector/common.rs` | 06-01 | Shared connector contract utilities |
| `crates/gossip-contracts/src/connector/common_tests.rs` | 06-01 | Common connector contract tests |
| `crates/gossip-contracts/src/connector/git.rs` | 06-07 | Git-specific connector contract types |
| `crates/gossip-contracts/src/connector/ordered.rs` | 06-01 | Ordered content source contract |
| `crates/gossip-contracts/src/connector/types.rs` | 06-01, 06-02 | ScanItem, ItemRef, Budgets, ConnectorCapabilities, ErrorClass, ItemKey, TokenBytes |
| `crates/gossip-contracts/src/connector/types_tests.rs` | 06-01, 06-02 | Connector type tests |

### Persistence Contracts (Boundary 5)

`crates/gossip-contracts/src/persistence/` (12 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-contracts/src/persistence/mod.rs` | 07-01 | Module structure, public API, re-exports |
| `crates/gossip-contracts/src/persistence/done_ledger.rs` | 07-02 | DoneLedger trait, DoneLedgerStatus lattice merge |
| `crates/gossip-contracts/src/persistence/done_ledger_tests.rs` | 07-02 | DoneLedger tests |
| `crates/gossip-contracts/src/persistence/findings.rs` | 07-03 | FindingsSink trait, CommitHandle durability contract |
| `crates/gossip-contracts/src/persistence/findings_tests.rs` | 07-03 | FindingsSink tests |
| `crates/gossip-contracts/src/persistence/lattice_property_tests.rs` | 07-02 | Lattice merge property-based tests |
| `crates/gossip-contracts/src/persistence/page_commit.rs` | 07-04 | PageCommit typestate builder |
| `crates/gossip-contracts/src/persistence/commit.rs` | 07-04 | Commit protocol types |
| `crates/gossip-contracts/src/persistence/error.rs` | 07-01 | Persistence error types |
| `crates/gossip-contracts/src/persistence/conformance.rs` | 07-05 | Persistence conformance test suite |
| `crates/gossip-contracts/src/persistence/ovid.rs` | 07-01 | Observation/version identity helpers |
| `crates/gossip-contracts/src/persistence/write_context.rs` | 07-01 | Shared write-side routing and fencing metadata |

### gossip-persistence-inmemory (Boundary 5 — in-memory backend)

`crates/gossip-persistence-inmemory/src/` (7 files)

In-memory reference persistence backends for tests and deterministic simulation. Implements the same monotonic lattice merge semantics and idempotent write contracts required of production backends, including explicit durability acknowledgement via `CommitHandle::wait()`.

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-persistence-inmemory/src/lib.rs` | 07-05 | Crate root, `InMemoryDoneLedger` and `InMemoryFindingsSink` re-exports |
| `crates/gossip-persistence-inmemory/src/done_ledger.rs` | 07-02, 07-05 | `InMemoryDoneLedger`: lattice-merge done-ledger with condvar-based commit handles |
| `crates/gossip-persistence-inmemory/src/findings.rs` | 07-03, 07-05 | `InMemoryFindingsSink`: idempotent findings writer with referential integrity checks |
| `crates/gossip-persistence-inmemory/src/store.rs` | 07-05 | Shared in-memory store state |
| `crates/gossip-persistence-inmemory/src/pending.rs` | 07-05 | Pending commit tracking and notification |
| `crates/gossip-persistence-inmemory/src/error.rs` | 07-05 | Persistence error types for in-memory backend |
| `crates/gossip-persistence-inmemory/src/tests.rs` | 07-05 | Integration and behavioral tests |

### gossip-done-ledger-postgres (Boundary 5 — Postgres done-ledger)

`crates/gossip-done-ledger-postgres/src/` (8 files)

Postgres-backed done-ledger implementing the `DoneLedger` trait with the same monotonic lattice merge semantics as the in-memory backend. Production persistence backend for tracking scanned-item status.

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-done-ledger-postgres/src/lib.rs` | 07-02, 07-05 | Crate root, `PostgresDoneLedger` re-export |
| `crates/gossip-done-ledger-postgres/src/backend.rs` | 07-02 | `PostgresDoneLedger`: Postgres-backed done-ledger implementation |
| `crates/gossip-done-ledger-postgres/src/error.rs` | 07-02 | Postgres done-ledger error types |
| `crates/gossip-done-ledger-postgres/src/migrations.rs` | 07-02 | Schema migration management |
| `crates/gossip-done-ledger-postgres/src/schema.rs` | 07-02 | SQL schema definitions |
| `crates/gossip-done-ledger-postgres/src/types.rs` | 07-02 | Type conversions for Postgres |
| `crates/gossip-done-ledger-postgres/src/tests.rs` | 07-05 | Integration tests |
| `crates/gossip-done-ledger-postgres/src/test_postgres.rs` | 07-05 | Postgres test utilities |

### gossip-findings-postgres (Boundary 5 — Postgres findings sink)

`crates/gossip-findings-postgres/src/` (9 files)

Postgres-backed findings sink implementing the `FindingsSink` trait. Production persistence backend for writing detection findings durably.

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-findings-postgres/src/lib.rs` | 07-03, 07-05 | Crate root, `PostgresFindingsSink` re-export |
| `crates/gossip-findings-postgres/src/backend.rs` | 07-03 | `PostgresFindingsSink`: Postgres-backed findings writer |
| `crates/gossip-findings-postgres/src/error.rs` | 07-03 | Postgres findings error types |
| `crates/gossip-findings-postgres/src/migrations.rs` | 07-03 | Schema migration management |
| `crates/gossip-findings-postgres/src/read_api.rs` | 07-03 | Read API for querying persisted findings |
| `crates/gossip-findings-postgres/src/schema.rs` | 07-03 | SQL schema definitions |
| `crates/gossip-findings-postgres/src/types.rs` | 07-03 | Type conversions for Postgres |
| `crates/gossip-findings-postgres/src/tests.rs` | 07-05 | Integration tests |
| `crates/gossip-findings-postgres/src/test_postgres.rs` | 07-05 | Postgres test utilities |

### gossip-pg-common (Boundary 5 — shared Postgres utilities)

`crates/gossip-pg-common/src/` (4 files)

Shared Postgres infrastructure (connection pooling, migration runner, test support) used by both `gossip-done-ledger-postgres` and `gossip-findings-postgres`.

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-pg-common/src/lib.rs` | 07-05 | Crate root, shared Postgres utilities |
| `crates/gossip-pg-common/src/migration.rs` | 07-05 | Migration runner |
| `crates/gossip-pg-common/src/types.rs` | 07-05 | Common Postgres type helpers |
| `crates/gossip-pg-common/src/test_support.rs` | 07-05 | Test support utilities for Postgres backends |

### gossip-contracts Top-Level

`crates/gossip-contracts/src/` (2 top-level files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-contracts/src/lib.rs` | 00-03 | Crate root, module declarations |
| `crates/gossip-contracts/src/test_util.rs` | 08-05 | Shared test utilities |

### gossip-stdx

`crates/gossip-stdx/src/` (23 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-stdx/src/lib.rs` | Appendix F | Crate root, shared data structures |
| `crates/gossip-stdx/src/ring_buffer.rs` | Appendix F | RingBuffer<T, N> fixed-capacity FIFO |
| `crates/gossip-stdx/src/ring_buffer_tests.rs` | Appendix F | Ring buffer tests |
| `crates/gossip-stdx/src/inline_vec.rs` | Appendix F | InlineVec<T, N> stack-first small-vector, 9 Kani proofs |
| `crates/gossip-stdx/src/inline_vec_tests.rs` | Appendix F | InlineVec tests |
| `crates/gossip-stdx/src/byte_slab.rs` | Appendix F | ByteSlab fixed-capacity byte allocation arena |
| `crates/gossip-stdx/src/byte_slab_tests.rs` | Appendix F | ByteSlab tests |
| `crates/gossip-stdx/src/byte_ring.rs` | -- | Byte-oriented ring buffer |
| `crates/gossip-stdx/src/atomic_bitset.rs` | -- | Atomic bitset for concurrent flag tracking |
| `crates/gossip-stdx/src/atomic_bitset_tests.rs` | -- | Atomic bitset tests |
| `crates/gossip-stdx/src/atomic_seen_sets.rs` | -- | Atomic seen-set for deduplication |
| `crates/gossip-stdx/src/atomic_seen_sets_tests.rs` | -- | Atomic seen-set tests |
| `crates/gossip-stdx/src/bitset.rs` | -- | Non-atomic bitset |
| `crates/gossip-stdx/src/bitset_tests.rs` | -- | Bitset tests |
| `crates/gossip-stdx/src/fastrange.rs` | -- | Fast range mapping (Lemire-style) |
| `crates/gossip-stdx/src/fixed_set.rs` | -- | Fixed-capacity set |
| `crates/gossip-stdx/src/fnv.rs` | -- | FNV-1a hash implementation |
| `crates/gossip-stdx/src/fnv_tests.rs` | -- | FNV hash tests |
| `crates/gossip-stdx/src/perf_stats.rs` | -- | Performance statistics helpers |
| `crates/gossip-stdx/src/spsc.rs` | -- | Single-producer single-consumer queue |
| `crates/gossip-stdx/src/test_support.rs` | -- | Test support utilities |
| `crates/gossip-stdx/src/timing_wheel.rs` | -- | Timing wheel for scheduled events |
| `crates/gossip-stdx/src/timing_wheel_tests.rs` | -- | Timing wheel tests |

### gossip-connectors

`crates/gossip-connectors/src/` (10 files)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/gossip-connectors/src/lib.rs` | 06-01 | Crate root, connector registrations |
| `crates/gossip-connectors/src/common.rs` | 06-06, 06-07 | Shared connector utilities |
| `crates/gossip-connectors/src/filesystem.rs` | 06-07 | FilesystemConnector, local directory tree enumeration |
| `crates/gossip-connectors/src/filesystem_tests.rs` | 06-07 | FilesystemConnector tests |
| `crates/gossip-connectors/src/git.rs` | 06-07 | Git connector, bridges scanner-git to connector boundary |
| `crates/gossip-connectors/src/git_tests.rs` | 06-07 | Git connector tests |
| `crates/gossip-connectors/src/in_memory.rs` | 06-06 | In-memory connector for testing |
| `crates/gossip-connectors/src/in_memory_tests.rs` | 06-06 | In-memory connector tests |
| `crates/gossip-connectors/src/split_estimator.rs` | 06-07 | Split estimator for dynamic shard splitting hints |
| `crates/gossip-connectors/src/split_estimator_tests.rs` | 06-07 | Split estimator tests |

### scanner-engine

`crates/scanner-engine/src/` (55 files: 11 top-level + 44 in 5 subdirectories)

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `crates/scanner-engine/src/lib.rs` | -- | Crate root, detection engine entry point |
| `crates/scanner-engine/src/api.rs` | -- | Public API types for the detection engine |
| `crates/scanner-engine/src/demo.rs` | -- | Demo/example detection engine usage |
| `crates/scanner-engine/src/b64_yara_gate.rs` | -- | Base64-aware YARA rule gating |
| `crates/scanner-engine/src/regex2anchor.rs` | -- | Regex-to-anchor optimization for fast pre-filtering |
| `crates/scanner-engine/src/regex2anchor_tests.rs` | -- | Regex-to-anchor tests |
| `crates/scanner-engine/src/scratch_memory.rs` | -- | Reusable scratch memory for scanning |
| `crates/scanner-engine/src/perf_counters.rs` | -- | Performance counter infrastructure |
| `crates/scanner-engine/src/perf_stats.rs` | -- | Performance statistics |
| `crates/scanner-engine/src/test_utils.rs` | -- | Test utilities for scanner-engine |
| `crates/scanner-engine/src/tiger_harness.rs` | -- | Tiger (simulation) test harness |
| Subdirectories: | | `content_policy/`, `engine/`, `lsm/`, `pool/`, `rules/` |

### scanner-git

`crates/scanner-git/src/` (106 files)

A large crate implementing the complete Git scanning pipeline. Key files:

| Source File | Topics |
|------------|---------|
| `crates/scanner-git/src/lib.rs` | Crate root |
| `crates/scanner-git/src/runner.rs` | Main scan runner |
| `crates/scanner-git/src/runner_exec.rs` | Runner execution logic |
| `crates/scanner-git/src/commit_walk.rs` | Git commit graph walking |
| `crates/scanner-git/src/commit_graph.rs` | Commit graph construction |
| `crates/scanner-git/src/blob_introducer.rs` | Blob introduction analysis |
| `crates/scanner-git/src/tree_diff.rs` | Tree diff computation |
| `crates/scanner-git/src/pack_reader.rs` | Git packfile reading |
| `crates/scanner-git/src/pack_decode.rs` | Git pack object decoding |
| `crates/scanner-git/src/pack_delta.rs` | Git delta decompression |
| `crates/scanner-git/src/engine_adapter.rs` | Bridge to scanner-engine |
| `crates/scanner-git/src/repo.rs` | Repository abstraction |
| `crates/scanner-git/src/preflight.rs` | Pre-scan validation |
| `crates/scanner-git/src/seen_store.rs` | Deduplication of seen blobs |
| `crates/scanner-git/src/policy_hash.rs` | Policy hash for git scanning |
| `crates/scanner-git/src/sim_git_scan/` | Simulation harness for git scanning |

### scanner-scheduler

`crates/scanner-scheduler/src/` (111 files: 15 top-level + 96 in 7 subdirectories)

| Source File | Topics |
|------------|---------|
| `crates/scanner-scheduler/src/lib.rs` | Crate root |
| `crates/scanner-scheduler/src/api.rs` | Scheduler public API |
| `crates/scanner-scheduler/src/engine.rs` | Engine integration |
| `crates/scanner-scheduler/src/pipeline.rs` | Scan pipeline orchestration |
| `crates/scanner-scheduler/src/runtime.rs` | Runtime management |
| `crates/scanner-scheduler/src/pool.rs` | Thread pool management |
| `crates/scanner-scheduler/src/events.rs` | Scheduler events |
| `crates/scanner-scheduler/src/finding_output.rs` | Finding output handling |
| `crates/scanner-scheduler/src/source_kind.rs` | Source type discrimination |
| `crates/scanner-scheduler/src/store.rs` | State store |
| `crates/scanner-scheduler/src/content_policy.rs` | Content policy enforcement |
| `crates/scanner-scheduler/src/scratch_memory.rs` | Reusable scratch memory |
| `crates/scanner-scheduler/src/json_write.rs` | JSON output writing |
| `crates/scanner-scheduler/src/perf_stats.rs` | Performance statistics |
| `crates/scanner-scheduler/src/test_utils.rs` | Test utilities |
| Subdirectories: | `archive/`, `git_scan/`, `scheduler/`, `sim/`, `sim_archive/`, `sim_scanner/`, `sim_scheduler/` |

### gossip-scanner-runtime

`crates/gossip-scanner-runtime/src/` (15 files)

| Source File | Topics |
|------------|---------|
| `crates/gossip-scanner-runtime/src/lib.rs` | Crate root, runtime orchestration |
| `crates/gossip-scanner-runtime/src/lib_tests.rs` | Runtime tests |
| `crates/gossip-scanner-runtime/src/cli.rs` | CLI argument parsing and wiring |
| `crates/gossip-scanner-runtime/src/cli_tests.rs` | CLI argument tests |
| `crates/gossip-scanner-runtime/src/checkpoint_aggregator.rs` | Receipt-driven committed-prefix checkpoint aggregation |
| `crates/gossip-scanner-runtime/src/commit_model.rs` | Commit model types for result persistence |
| `crates/gossip-scanner-runtime/src/commit_sink.rs` | CommitSink implementation |
| `crates/gossip-scanner-runtime/src/coordination_sink.rs` | Coordination-aware result sink |
| `crates/gossip-scanner-runtime/src/distributed.rs` | Distributed execution mode |
| `crates/gossip-scanner-runtime/src/event_sink.rs` | Event sink for observability |
| `crates/gossip-scanner-runtime/src/git_repo.rs` | Git repository executor for scanner runtime |
| `crates/gossip-scanner-runtime/src/ordered_content.rs` | Ordered content source for scanner runtime |
| `crates/gossip-scanner-runtime/src/parity.rs` | Parity checking between execution modes |
| `crates/gossip-scanner-runtime/src/result_committer.rs` | Per-item durable result committer (findings → done-ledger ordering) |
| `crates/gossip-scanner-runtime/src/result_translation.rs` | Translation between scan results and persistence types |

### gossip-worker

`crates/gossip-worker/src/` (1 file)

| Source File | Topics |
|------------|---------|
| `crates/gossip-worker/src/main.rs` | Distributed worker binary entry point |

### scanner-rs-cli

`crates/scanner-rs-cli/src/` (1 file)

| Source File | Topics |
|------------|---------|
| `crates/scanner-rs-cli/src/main.rs` | Standalone scanner CLI binary entry point |

### scanner-engine-integration-tests

`crates/scanner-engine-integration-tests/` (1 src file + test directories)

| Source File | Topics |
|------------|---------|
| `crates/scanner-engine-integration-tests/src/lib.rs` | Test crate root |
| `crates/scanner-engine-integration-tests/tests/` | Integration test suites |

Test directories include: `chunked_file_scans.rs`, `corpus/`, `diagnostic/`, `integration/`, `property/`, `regression/`, `simulation/`, `smoke/`.

### Documentation

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `docs/gossip-contracts/boundary-1-identity-spine.md` | 02-01 through 02-10 | Boundary 1 complete documentation |
| `docs/gossip-coordination/boundary-2-coordination.md` | 04-01 through 04-18 | Boundary 2 coordination protocol specification |
| `docs/gossip-coordination/coordination-testing.md` | 04-12, 08-05 | Test tier breakdown and cargo test commands |
| `docs/gossip-coordination/simulation-harness.md` | 08-05 | Simulation architecture, invariants S1-S9, fault levels |
| `diagrams/00-README.md` | 00-03 | Diagram index, how to read diagrams |
| `diagrams/01-system-overview.md` | 00-03, 08-01 | System overview diagram |
| `diagrams/02-boundary-dependency-graph.md` | 08-01 | Boundary dependency graph |
| `diagrams/03-id-derivation-dag.md` | 02-04, 02-05 | Identity derivation DAG |
| `diagrams/04-end-to-end-scan-flow.md` | 08-02 | End-to-end scan data flow |
| `diagrams/05-shard-and-run-state-machines.md` | 04-02, 04-07 | Shard and run state machine diagrams |
| `diagrams/06-fencing-protocol.md` | 04-03 | Fencing protocol sequence diagram |
| `diagrams/07-lease-lifecycle.md` | 04-03, 04-14 | Lease lifecycle diagram |
| `diagrams/08-pagecommit-typestate.md` | 07-04 | PageCommit typestate diagram |
| `diagrams/09-circuit-breaker.md` | 06-04 | Circuit breaker state machine diagram |
| `diagrams/10-failure-modes-and-recovery.md` | 08-03 | Failure modes and recovery paths |
| `diagrams/11-tenant-isolation.md` | 08-04 | Tenant isolation architecture |
| `diagrams/12-split-operations.md` | 04-06, 05-04 | Split operations diagram |
| `diagrams/13-shard-algebra-types.md` | 05-02, 05-03 | Shard algebra type relationships |
| `specs/coordination/ShardFencing.tla` | 04-03, Appendix G | TLA+ fencing specification |

### Project Artifacts

| Source File | Primary Chapters | Topics |
|------------|------------------|---------|
| `tmp/gossip-project-artifacts/phase2_spec.md` | 04-01 through 04-18 | Phase 2 coordination spec |
| `tmp/gossip-project-artifacts/distributed-secret-scanner-instructions.md` | 00-01, 00-02, REFERENCES | Original system design document |
| `tmp/gossip-project-artifacts/gossip-contracts-references.md` | REFERENCES, Deep Dive boxes | Contracts crate documentation |
| `tmp/gossip-project-artifacts/phase0-outline.md` | 00-01 | Phase 0 planning outline |

## Chapter → Source File Map

### Chapter 00: Prologue (4 files)

- **00-01: What Problem Are We Solving**
  - `tmp/gossip-project-artifacts/distributed-secret-scanner-instructions.md` (motivation)
  - `tmp/gossip-project-artifacts/phase0-outline.md` (planning)

- **00-02: Why Distributed**
  - `tmp/gossip-project-artifacts/distributed-secret-scanner-instructions.md` (design principles)

- **00-03: Architecture at a Glance**
  - `diagrams/00-README.md` (diagram index)
  - `diagrams/01-system-overview.md` (system overview)

- **00-04: How to Read This Guide**
  - This document (meta)

### Chapter 01: Foundations (5 files)

- **01-01: Cryptographic Hashing Primer**
  - `crates/gossip-contracts/src/identity/hashing.rs` (BLAKE3 wrappers)

- **01-02: Content-Addressed Identity**
  - `crates/gossip-contracts/src/identity/canonical.rs` (CanonicalBytes trait)
  - `crates/gossip-contracts/src/identity/hashing.rs` (hasher initialization)

- **01-03: Domain Separation**
  - `crates/gossip-contracts/src/identity/domain.rs` (domain tags)
  - `crates/gossip-contracts/src/identity/hashing.rs` (derive-key mode)

- **01-04: Type-Driven Correctness**
  - `crates/gossip-contracts/src/identity/types.rs` (newtype definitions)
  - `crates/gossip-contracts/src/identity/macros.rs` (`define_id_32!`, `define_id_64!`)

- **01-05: Invariant-First Design**
  - All identity source files (invariants demonstrated throughout)

### Chapter 02: Boundary 1 -- Identity Spine (10 files)

- **02-01: Identity Problem Space**
  - `crates/gossip-contracts/src/identity/mod.rs` (module structure)
  - `docs/gossip-contracts/boundary-1-identity-spine.md` (complete documentation)

- **02-02: Canonical Encoding**
  - `crates/gossip-contracts/src/identity/canonical.rs` (encoding before hashing)
  - `crates/gossip-contracts/src/identity/hashing.rs` (hasher initialization)

- **02-03: Domain Separation Registry**
  - `crates/gossip-contracts/src/identity/domain.rs` (tag constants)

- **02-04: ID Type Hierarchy**
  - `crates/gossip-contracts/src/identity/types.rs` (TenantId, etc.)
  - `crates/gossip-contracts/src/identity/coordination.rs` (RunId, ShardId, WorkerId — all `u64` via `define_id_64!`)
  - `diagrams/03-id-derivation-dag.md` (derivation DAG)

- **02-05: Item Identity**
  - `crates/gossip-contracts/src/identity/item.rs` (StableItemId, ItemKey, ObjectVersionId)
  - `diagrams/03-id-derivation-dag.md` (item derivation)

- **02-06: Secret and Finding Identity**
  - `crates/gossip-contracts/src/identity/finding.rs` (FindingId, OccurrenceId, NormHash, SecretHash)

- **02-07: Policy Hash**
  - `crates/gossip-contracts/src/identity/policy.rs` (PolicyHash, RuleFingerprint)

- **02-08: Macro System**
  - `crates/gossip-contracts/src/identity/macros.rs` (`define_id_32!`, `define_id_32_restricted!`, `define_id_64!`)

- **02-09: Golden Vectors and Testing**
  - `crates/gossip-contracts/src/identity/golden.rs` (test vectors)

- **02-10: Version Migration**
  - All `#[cfg(test)]` blocks in `crates/gossip-contracts/src/identity/*.rs`

### Chapter 03: Distributed Systems Theory (6 files)

- **03-01: Impossibility Results**
  - No direct source files (theory chapter)

- **03-02: Consistency Models**
  - No direct source files (theory chapter)

- **03-03: Leases and Fencing**
  - `crates/gossip-coordination/src/lease.rs` (lease implementation)
  - `specs/coordination/ShardFencing.tla` (TLA+ spec)
  - `diagrams/06-fencing-protocol.md` (fencing diagram)

- **03-04: Idempotency**
  - `crates/gossip-coordination/src/validation.rs` (idempotency checks)

- **03-05: Failure Detection**
  - `crates/gossip-coordination/src/lease.rs` (lease expiry as failure detection)

- **03-06: Exactly-Once Semantics**
  - `crates/gossip-contracts/src/persistence/mod.rs` (DoneLedger)
  - `crates/gossip-coordination/src/validation.rs` (validation preamble)

### Chapter 04: Boundary 2 -- Coordination (18 files)

- **04-01: Coordination Problem Space**
  - `crates/gossip-contracts/src/coordination/mod.rs` (contract types)
  - `crates/gossip-coordination/src/lib.rs` (runtime crate root)
  - `docs/gossip-coordination/boundary-2-coordination.md` (complete documentation)
  - `tmp/gossip-project-artifacts/phase2_spec.md` (design spec)

- **04-02: Shard State Machine**
  - `crates/gossip-coordination/src/record.rs` (ShardRecord, state transitions)
  - `crates/gossip-contracts/src/coordination/shard_spec.rs` (ShardSpec)
  - `diagrams/05-shard-and-run-state-machines.md` (state machine diagram)

- **04-03: Fencing Protocol Deep Dive**
  - `crates/gossip-coordination/src/lease.rs` (fencing tokens, lease TTL)
  - `specs/coordination/ShardFencing.tla` (TLA+ fencing spec)
  - `diagrams/06-fencing-protocol.md` (fencing diagram)
  - `diagrams/07-lease-lifecycle.md` (lease lifecycle)

- **04-04: Cursor Monotonicity**
  - `crates/gossip-contracts/src/coordination/cursor.rs` (cursor types and monotonicity)

- **04-05: Idempotency Protocol**
  - `crates/gossip-coordination/src/validation.rs` (idempotency checks)

- **04-06: Split Operations**
  - `crates/gossip-contracts/src/coordination/split.rs` (split plan types)
  - `crates/gossip-coordination/src/split_execution.rs` (split execution logic)
  - `diagrams/12-split-operations.md` (split diagram)

- **04-07: Run Lifecycle**
  - `crates/gossip-coordination/src/run.rs` (RunManifest, run lifecycle)
  - `diagrams/05-shard-and-run-state-machines.md` (run state machine)

- **04-08: Coordination Trait**
  - `crates/gossip-coordination/src/traits.rs` (CoordinationBackend trait)

- **04-09: Worker Session**
  - `crates/gossip-coordination/src/session.rs` (WorkerSession)

- **04-10: From Spec to Implementation**
  - `crates/gossip-contracts/src/coordination/shard_spec.rs` (ShardSpec)
  - `tmp/gossip-project-artifacts/phase2_spec.md` (original spec)

- **04-11: Reference Backend Implementation**
  - `crates/gossip-coordination/src/in_memory.rs` (InMemoryCoordinator)
  - `crates/gossip-coordination/src/in_memory_tests.rs` (backend tests)

- **04-12: Verification Roadmap**
  - `crates/gossip-coordination/src/conformance_tests.rs` (conformance suite)
  - `crates/gossip-coordination/src/scenario_tests.rs` (scenario tests)
  - `crates/gossip-coordination/src/test_fixtures.rs` (test helpers)
  - `docs/gossip-coordination/coordination-testing.md` (test tier breakdown)

- **04-13: Coordination Facade**
  - `crates/gossip-coordination/src/facade.rs` (CoordinationFacade super-trait)

- **04-14: Claim Cooldown**
  - `crates/gossip-coordination/src/lease.rs` (per-worker claim cooldown)
  - `diagrams/07-lease-lifecycle.md` (lease lifecycle)

- **04-15: Events System**
  - `crates/gossip-coordination/src/events.rs` (coordination events)

- **04-16: Run Management**
  - `crates/gossip-coordination/src/run.rs` (run management operations)

- **04-17: Validation Layer**
  - `crates/gossip-coordination/src/validation.rs` (5-check validation preamble)

- **04-18: Error Taxonomy**
  - `crates/gossip-coordination/src/error.rs` (coordination error types)
  - `crates/gossip-coordination/src/run_errors.rs` (run-specific errors)

### Chapter 05: Boundary 3 -- Shard Algebra (5 files)

- **05-01: Range Sharding Theory**
  - `crates/gossip-frontier/src/lib.rs` (crate root)

- **05-02: Key Encoding Schemas**
  - `crates/gossip-frontier/src/key_encoding.rs` (KeyEncoding trait, PathKey, ManifestRowKey)

- **05-03: Range Algebra**
  - `crates/gossip-frontier/src/hint.rs` (ShardHint, ShardMetadata)

- **05-04: Split Computation**
  - `crates/gossip-frontier/src/hint.rs` (split propagation)
  - `diagrams/12-split-operations.md` (split diagram)

- **05-05: Coverage Verification**
  - `crates/gossip-frontier/src/builder.rs` (PreallocShardBuilder)

### Chapter 06: Boundary 4 -- Connector (8 files)

- **06-01: Connector Problem Space**
  - `crates/gossip-contracts/src/connector/mod.rs` (connector contracts)
  - `crates/gossip-contracts/src/connector/types.rs` (ScanItem, ItemRef, Budgets, ConnectorCapabilities, ErrorClass)

- **06-02: Toxic Byte Value Wrappers**
  - `crates/gossip-contracts/src/connector/types.rs` (ItemKey, ItemRef, TokenBytes value wrappers)

- **06-03: Split, Read, and Capability Methods**
  - `crates/gossip-contracts/src/connector/api.rs` (Error taxonomy (ErrorClass, EnumerateError, ReadError) and capability negotiation (ConnectorCapabilities))
  - `crates/gossip-contracts/src/connector/api_tests.rs` (Connector API tests)

- **06-04: Circuit Breaker Design**
  - `crates/gossip-contracts/src/connector/mod.rs` (circuit breaker contract)
  - `diagrams/09-circuit-breaker.md` (circuit breaker diagram)

- **06-05: In-Memory Deterministic Connector**
  - `crates/gossip-connectors/src/in_memory.rs` (InMemoryDeterministicConnector for testing)
  - `crates/gossip-connectors/src/in_memory_tests.rs` (In-memory connector tests)

- **06-06: Filesystem Connector**
  - `crates/gossip-connectors/src/filesystem.rs` (FilesystemConnector)
  - `crates/gossip-connectors/src/filesystem_tests.rs` (FilesystemConnector tests)
  - `crates/gossip-connectors/src/split_estimator.rs` (Split estimator)
  - `crates/gossip-connectors/src/split_estimator_tests.rs` (Split estimator tests)
  - `crates/gossip-connectors/src/common.rs` (Shared connector utilities)

- **06-07: Git Connector**
  - `crates/gossip-connectors/src/git.rs` (Git connector)
  - `crates/gossip-connectors/src/git_tests.rs` (Git connector tests)

### Chapter 07: Boundary 5 -- Persistence (5 files)

- **07-01: Persistence Problem Space**
  - `crates/gossip-contracts/src/persistence/mod.rs` (persistence contracts)

- **07-02: Done Ledger**
  - `crates/gossip-contracts/src/persistence/done_ledger.rs` (DoneLedger trait, DoneLedgerStatus lattice merge)

- **07-03: Findings Sink**
  - `crates/gossip-contracts/src/persistence/findings.rs` (FindingsSink trait, CommitHandle durability contract)

- **07-04: Commit Protocol Typestate**
  - `crates/gossip-contracts/src/persistence/page_commit.rs` (PageCommit typestate builder)
  - `crates/gossip-contracts/src/persistence/commit.rs` (Commit protocol types)
  - `diagrams/08-pagecommit-typestate.md` (typestate diagram)

- **07-05: In-Memory Test Doubles**
  - `crates/gossip-persistence-inmemory/src/lib.rs` (InMemoryDoneLedger, InMemoryFindingsSink)
  - `crates/gossip-persistence-inmemory/src/done_ledger.rs` (In-memory done-ledger)
  - `crates/gossip-persistence-inmemory/src/findings.rs` (In-memory findings sink)
  - `crates/gossip-contracts/src/persistence/conformance.rs` (Persistence conformance test suite)

### Chapter 08: Cross-Cutting (5 files)

- **08-01: Boundary Dependency Graph**
  - All crate `Cargo.toml` files (dependencies)
  - `diagrams/01-system-overview.md` (system overview)
  - `diagrams/02-boundary-dependency-graph.md` (dependency graph)

- **08-02: Data Flow End-to-End**
  - All source files (trace through system)
  - `diagrams/04-end-to-end-scan-flow.md` (data flow diagram)

- **08-03: Failure Modes and Recovery**
  - `crates/gossip-coordination/src/lease.rs` (lease expiry)
  - `crates/gossip-coordination/src/session.rs` (session recovery)
  - `crates/gossip-contracts/src/connector/mod.rs` (circuit breaker contract)
  - `diagrams/10-failure-modes-and-recovery.md` (failure modes diagram)

- **08-04: Tenant Isolation**
  - `crates/gossip-contracts/src/identity/finding.rs` (SecretHash keying)
  - `crates/gossip-contracts/src/identity/types.rs` (TenantSecretKey)
  - `crates/gossip-coordination/src/validation.rs` (5-check preamble)
  - `diagrams/11-tenant-isolation.md` (tenant isolation diagram)

- **08-05: Deterministic Simulation Testing**
  - `crates/gossip-coordination/src/sim/harness.rs` (CoordinationSim)
  - `crates/gossip-coordination/src/sim/backend.rs` (SimulationBackend)
  - `crates/gossip-coordination/src/sim/worker.rs` (SimWorker)
  - `crates/gossip-coordination/src/sim/fault_injector.rs` (FaultInjectingIntrospector)
  - `crates/gossip-coordination/src/sim/invariants.rs` (invariants S1-S9)
  - `crates/gossip-coordination/src/sim/test_util.rs` (simulation utilities)
  - `crates/gossip-coordination/src/sim/mega_sim_tests.rs` (large-scale tests)
  - `crates/gossip-coordination/src/sim/sim_behavioral_tests.rs` (behavioral tests)
  - `docs/gossip-coordination/simulation-harness.md` (simulation architecture documentation)

### Chapter 09: Appendices (7 files: A through G)

- **Appendix A: Rust Patterns Used**
  - All source files (pattern examples throughout)

- **Appendix B: BLAKE3 Internals**
  - `crates/gossip-contracts/src/identity/hashing.rs` (BLAKE3 usage)

- **Appendix C: Property-Based Testing Guide**
  - All `#[cfg(test)]` blocks with `proptest!` (examples)

- **Appendix D: Glossary**
  - All source files (term definitions)

- **Appendix E: Source File Map**
  - This document (self-reference)

- **Appendix F: gossip-stdx Data Structures**
  - `crates/gossip-stdx/src/lib.rs` (crate root)
  - `crates/gossip-stdx/src/ring_buffer.rs` (ring buffer implementation)
  - `crates/gossip-stdx/src/inline_vec.rs` (InlineVec stack-first small vector)
  - `crates/gossip-stdx/src/byte_slab.rs` (ByteSlab arena allocator)

- **Appendix G: TLA+ Verification**
  - `specs/coordination/ShardFencing.tla` (TLA+ fencing specification)

## Finding Code from Concepts

### If you want to understand...

**Content-addressed identity**:
1. Read Chapter 01-02 (Content-Addressed Identity)
2. Read `crates/gossip-contracts/src/identity/hashing.rs`
3. Read `crates/gossip-contracts/src/identity/finding.rs`

**Domain separation**:
1. Read Chapter 01-03 (Domain Separation)
2. Read `crates/gossip-contracts/src/identity/domain.rs`
3. Read Appendix B (BLAKE3 Internals)

**Tenant isolation**:
1. Read Chapter 08-04 (Tenant Isolation)
2. Read `crates/gossip-contracts/src/identity/finding.rs` (SecretHash)
3. Read `crates/gossip-contracts/src/identity/types.rs` (TenantSecretKey)

**Lease management and fencing**:
1. Read Chapter 04-03 (Fencing Protocol Deep Dive)
2. Read `crates/gossip-coordination/src/lease.rs`
3. Read `specs/coordination/ShardFencing.tla`

**Claim cooldown**:
1. Read Chapter 04-14 (Claim Cooldown)
2. Read `crates/gossip-coordination/src/lease.rs`

**Exactly-once processing**:
1. Read Chapter 03-06 (Exactly-Once Semantics)
2. Read `crates/gossip-contracts/src/persistence/mod.rs` (DoneLedger)
3. Read Chapter 08-02 (Data Flow End-to-End)

**Failure recovery**:
1. Read Chapter 08-03 (Failure Modes and Recovery)
2. Read `crates/gossip-coordination/src/lease.rs` (lease expiry)
3. Read `crates/gossip-contracts/src/connector/mod.rs` (circuit breaker)

**Shard splitting**:
1. Read Chapter 05-04 (Split Computation)
2. Read Chapter 04-06 (Split Operations)
3. Read `crates/gossip-contracts/src/coordination/split.rs`
4. Read `crates/gossip-frontier/src/hint.rs`

**Coordination protocol**:
1. Read Chapter 04-08 (Coordination Trait)
2. Read `crates/gossip-coordination/src/traits.rs`
3. Read `crates/gossip-coordination/src/facade.rs`
4. Read `docs/gossip-coordination/boundary-2-coordination.md`

**Deterministic simulation testing**:
1. Read Chapter 08-05 (Deterministic Simulation Testing)
2. Read `crates/gossip-coordination/src/sim/harness.rs` (CoordinationSim)
3. Read `crates/gossip-coordination/src/sim/invariants.rs` (invariants S1-S9)
4. Read `docs/gossip-coordination/simulation-harness.md`

**Scanning pipeline**:
1. Read `crates/gossip-scanner-runtime/src/lib.rs` (runtime orchestration, CommitSink)
2. Read `crates/scanner-engine/src/lib.rs` (detection engine)
3. Read `crates/scanner-scheduler/src/lib.rs` (parallel scheduling)
4. Read `crates/scanner-git/src/lib.rs` (Git scanning)
5. Read `crates/gossip-connectors/src/git.rs` (connector bridge)

**Worker sessions**:
1. Read Chapter 04-09 (Worker Session)
2. Read `crates/gossip-coordination/src/session.rs`

**Events and observability**:
1. Read Chapter 04-15 (Events System)
2. Read `crates/gossip-coordination/src/events.rs`

**Error handling in coordination**:
1. Read Chapter 04-18 (Error Taxonomy)
2. Read `crates/gossip-coordination/src/error.rs`
3. Read `crates/gossip-coordination/src/run_errors.rs`

## Finding Documentation from Code

### If you're reading...

**`crates/gossip-contracts/src/identity/finding.rs`**:
- Overview: Chapter 02-06 (Secret and Finding Identity)
- Context: Chapter 01-02 (Content-Addressed Identity)
- Testing: Chapter 02-09 (Golden Vectors and Testing)

**`crates/gossip-coordination/src/lease.rs`**:
- Overview: Chapter 04-03 (Fencing Protocol Deep Dive)
- Cooldown: Chapter 04-14 (Claim Cooldown)
- Theory: Chapter 03-03 (Leases and Fencing)
- Failure handling: Chapter 08-03 (Failure Modes)
- Diagram: `diagrams/07-lease-lifecycle.md`

**`crates/gossip-coordination/src/traits.rs`**:
- Overview: Chapter 04-08 (Coordination Trait)
- Design doc: `docs/gossip-coordination/boundary-2-coordination.md`

**`crates/gossip-coordination/src/facade.rs`**:
- Overview: Chapter 04-13 (Coordination Facade)

**`crates/gossip-coordination/src/in_memory.rs`**:
- Overview: Chapter 04-11 (Reference Backend Implementation)
- Testing: Chapter 04-12 (Verification Roadmap)

**`crates/gossip-coordination/src/record.rs`**:
- Overview: Chapter 04-02 (Shard State Machine)
- Diagram: `diagrams/05-shard-and-run-state-machines.md`

**`crates/gossip-contracts/src/coordination/pooled.rs`**:
- Overview: Chapter 04-01 (Coordination Problem)
- Usage: Chapter 04-11 (Reference Implementation)
- Data structure: Appendix F (gossip-stdx Data Structures)

**`crates/gossip-coordination/src/session.rs`**:
- Overview: Chapter 04-09 (Worker Session)

**`crates/gossip-coordination/src/events.rs`**:
- Overview: Chapter 04-15 (Events System)

**`crates/gossip-coordination/src/validation.rs`**:
- Overview: Chapter 04-17 (Validation Layer)
- Context: Chapter 08-04 (Tenant Isolation)

**`crates/gossip-coordination/src/split_execution.rs`**:
- Overview: Chapter 04-06 (Split Operations)
- Diagram: `diagrams/12-split-operations.md`

**`crates/gossip-contracts/src/connector/mod.rs`**:
- Overview: Chapter 06-01 (Connector Problem Space)
- Circuit breaker: Chapter 06-05 (Circuit Breaker Design)

**`crates/gossip-contracts/src/connector/types.rs`**:
- Overview: Chapter 06-01 (Connector Problem Space)
- Toxic wrappers: Chapter 06-02 (Toxic Byte Value Wrappers)

**`crates/gossip-contracts/src/connector/api.rs`**:
- Error taxonomy and capabilities: Chapter 06-03 (Split, Read, and Capability Methods)

**`crates/gossip-connectors/src/in_memory.rs`**:
- Overview: Chapter 06-05 (In-Memory Deterministic Connector)

**`crates/gossip-connectors/src/filesystem.rs`**:
- Overview: Chapter 06-06 (Filesystem Connector)

**`crates/gossip-connectors/src/git.rs`**:
- Overview: Chapter 06-07 (Git Connector)

**`crates/gossip-contracts/src/persistence/mod.rs`**:
- Overview: Chapter 07-01 (Persistence Problem Space)

**`crates/gossip-contracts/src/persistence/done_ledger.rs`**:
- Overview: Chapter 07-02 (Done Ledger)

**`crates/gossip-contracts/src/persistence/findings.rs`**:
- Overview: Chapter 07-03 (Findings Sink)

**`crates/gossip-contracts/src/persistence/page_commit.rs`**:
- Typestate: Chapter 07-04 (Commit Protocol Typestate)

**`crates/gossip-persistence-inmemory/src/lib.rs`**:
- Overview: Chapter 07-05 (In-Memory Test Doubles)

**`crates/gossip-coordination-etcd/src/backend.rs`**:
- Overview: Chapter 04-08 (Coordination Trait), Chapter 04-11 (Reference Backend)

**`crates/gossip-frontier/src/key_encoding.rs`**:
- Overview: Chapter 05-02 (Key Encoding Schemas)

**`crates/gossip-frontier/src/hint.rs`**:
- Overview: Chapter 05-03 (Range Algebra), Chapter 05-04 (Split Computation)

**`crates/gossip-frontier/src/builder.rs`**:
- Overview: Chapter 05-05 (Coverage Verification)

**`crates/gossip-coordination/src/sim/harness.rs`**:
- Overview: Chapter 08-05 (Deterministic Simulation Testing)
- Design doc: `docs/gossip-coordination/simulation-harness.md`

**`crates/gossip-coordination/src/sim/invariants.rs`**:
- Overview: Chapter 08-05 (Deterministic Simulation Testing)
- Invariants S1-S9: `docs/gossip-coordination/simulation-harness.md`

**`crates/gossip-stdx/src/ring_buffer.rs`**:
- Overview: Appendix F (gossip-stdx Data Structures)

**`crates/gossip-stdx/src/inline_vec.rs`**:
- Overview: Appendix F (gossip-stdx Data Structures)

**`crates/gossip-stdx/src/byte_slab.rs`**:
- Overview: Appendix F (gossip-stdx Data Structures)
- Usage: Chapter 04-01 (Coordination Problem), Chapter 04-11 (Reference Implementation)

**`specs/coordination/ShardFencing.tla`**:
- Overview: Appendix G (TLA+ Verification)
- Context: Chapter 04-03 (Fencing Protocol Deep Dive)

## Deep Dive Boxes → Source Files

Many chapters include "Deep Dive" boxes that reference specific source locations:

| Deep Dive Topic | Source File | Chapter |
|-----------------|-------------|---------|
| BLAKE3 derive-key internals | `crates/gossip-contracts/src/identity/hashing.rs` | 01-01 |
| CanonicalBytes implementation | `crates/gossip-contracts/src/identity/canonical.rs` | 01-02 |
| Domain tag constants | `crates/gossip-contracts/src/identity/domain.rs` | 01-03 |
| define_id_32! / define_id_64! expansion | `crates/gossip-contracts/src/identity/macros.rs` | 01-04 |
| SecretHash keying | `crates/gossip-contracts/src/identity/finding.rs` | 02-06 |
| PolicyHash rule inclusion | `crates/gossip-contracts/src/identity/policy.rs` | 02-07 |
| Golden vector test structure | `crates/gossip-contracts/src/identity/golden.rs` | 02-09 |
| Coordination identity types | `crates/gossip-contracts/src/identity/coordination.rs` | 02-04 |
| Shard state machine transitions | `crates/gossip-coordination/src/record.rs` | 04-02 |
| Fencing protocol implementation | `crates/gossip-coordination/src/lease.rs` | 04-03 |
| Cursor monotonicity | `crates/gossip-contracts/src/coordination/cursor.rs` | 04-04 |
| Shard split implementation | `crates/gossip-coordination/src/split_execution.rs` | 04-06 |
| Run lifecycle management | `crates/gossip-coordination/src/run.rs` | 04-07 |
| CoordinationBackend trait | `crates/gossip-coordination/src/traits.rs` | 04-08 |
| WorkerSession management | `crates/gossip-coordination/src/session.rs` | 04-09 |
| InMemoryCoordinator | `crates/gossip-coordination/src/in_memory.rs` | 04-11 |
| CoordinationFacade | `crates/gossip-coordination/src/facade.rs` | 04-13 |
| Per-worker claim cooldown | `crates/gossip-coordination/src/lease.rs` | 04-14 |
| Coordination events | `crates/gossip-coordination/src/events.rs` | 04-15 |
| 5-check validation preamble | `crates/gossip-coordination/src/validation.rs` | 04-17 |
| Coordination error taxonomy | `crates/gossip-coordination/src/error.rs` | 04-18 |
| CoordinationSim architecture | `crates/gossip-coordination/src/sim/harness.rs` | 08-05 |
| Invariants S1-S9 | `crates/gossip-coordination/src/sim/invariants.rs` | 08-05 |
| Fault injection | `crates/gossip-coordination/src/sim/fault_injector.rs` | 08-05 |
| PageCommit type transitions | `crates/gossip-contracts/src/persistence/page_commit.rs` | 07-04 |
| Ring buffer implementation | `crates/gossip-stdx/src/ring_buffer.rs` | Appendix F |
| InlineVec stack-first vector | `crates/gossip-stdx/src/inline_vec.rs` | Appendix F |
| ByteSlab arena allocator | `crates/gossip-stdx/src/byte_slab.rs` | Appendix F |
| TLA+ shard fencing | `specs/coordination/ShardFencing.tla` | Appendix G |

## Test Files

Test code is organized in two ways:

**Inline tests** follow the convention `#[cfg(test)] mod tests` within source files:
- **Identity tests**: `#[cfg(test)]` blocks in `crates/gossip-contracts/src/identity/*.rs`
- **Coordination contract tests**: `#[cfg(test)]` blocks in `crates/gossip-contracts/src/coordination/*.rs`

**Dedicated test files** exist for coordination and simulation:
- `crates/gossip-coordination/src/record_tests.rs` -- ShardRecord tests
- `crates/gossip-coordination/src/in_memory_tests.rs` -- In-memory backend tests
- `crates/gossip-coordination/src/in_memory_filter_tests.rs` -- In-memory filter tests
- `crates/gossip-coordination/src/in_memory_run_tests.rs` -- In-memory run tests
- `crates/gossip-coordination/src/conformance_tests.rs` -- Conformance test suite
- `crates/gossip-coordination/src/scenario_tests.rs` -- Scenario-based integration tests
- `crates/gossip-coordination/src/test_fixtures.rs` -- Shared test fixtures
- `crates/gossip-coordination/src/sim/mega_sim_tests.rs` -- Large-scale simulation tests
- `crates/gossip-coordination/src/sim/sim_behavioral_tests.rs` -- Behavioral simulation tests
- `crates/gossip-coordination/src/sim/invariants_tests.rs` -- Invariant checker tests
- `crates/gossip-coordination/src/sim/test_util.rs` -- Simulation test utilities
- `crates/gossip-contracts/src/test_util.rs` -- Crate-level test utilities
- `crates/scanner-engine-integration-tests/tests/` -- Scanner-engine integration tests

See `docs/gossip-coordination/coordination-testing.md` for the test tier breakdown and cargo test commands.

## Documentation Files

| Document | Topics | Referenced In |
|----------|--------|---------------|
| `docs/gossip-contracts/boundary-1-identity-spine.md` | Complete Boundary 1 documentation | All of Chapter 02 |
| `docs/gossip-contracts/boundary-3-shard-algebra.md` | Boundary 3 shard algebra documentation | Chapter 05 |
| `docs/gossip-contracts/persistence-identity.md` | Persistence identity design | Chapter 07 |
| `docs/gossip-contracts/boundary-5-persistence.md` | Boundary 5 persistence documentation | Chapter 07 |
| `docs/gossip-coordination/boundary-2-coordination.md` | Coordination protocol specification | All of Chapter 04 |
| `docs/gossip-coordination/coordination-testing.md` | Test tier breakdown, cargo test commands | Chapter 04-12, 08-05 |
| `docs/gossip-coordination/coordination-error-model.md` | Coordination error taxonomy design | Chapter 04-18 |
| `docs/gossip-coordination/simulation-harness.md` | Simulation architecture, invariants S1-S9, fault levels | Chapter 08-05 |
| `diagrams/00-README.md` | Diagram index and reading guide | Chapter 00-03 |
| `specs/coordination/ShardFencing.tla` | TLA+ fencing specification | Chapters 04-03, Appendix G |
| `tmp/gossip-project-artifacts/phase2_spec.md` | Phase 2 coordination design spec | All of Chapter 04 |
| `tmp/gossip-project-artifacts/distributed-secret-scanner-instructions.md` | Original system design | Chapters 00, REFERENCES |
| `tmp/gossip-project-artifacts/gossip-contracts-references.md` | Contracts crate deep dives | REFERENCES, Deep Dive boxes |
| `tmp/gossip-project-artifacts/phase0-outline.md` | Phase 0 planning outline | Chapter 00-01 |

## Navigation Tips

### From Documentation to Code
1. Find the topic in the chapter table of contents
2. Look up the chapter in this appendix's "Chapter -> Source File Map"
3. Open the listed source files
4. Search for the specific function/type mentioned in the chapter

### From Code to Documentation
1. Note the file path you are reading
2. Look up the file in this appendix's "Source File -> Chapter Map"
3. Open the listed chapters
4. Search for the specific function/type in the chapter text

### Finding Related Code
1. Identify the boundary (B1-B5) from the file path:
   - `gossip-contracts/src/identity/` = Boundary 1
   - `gossip-contracts/src/coordination/` = Boundary 2 (contract types)
   - `gossip-coordination/src/` = Boundary 2 (runtime)
   - `gossip-coordination/src/sim/` = Cross-cutting (simulation)
   - `gossip-frontier/src/` = Boundary 3
   - `gossip-contracts/src/connector/` = Boundary 4 (contracts)
   - `gossip-connectors/src/` = Boundary 4 (implementations)
   - `gossip-contracts/src/persistence/` = Boundary 5 (contracts)
   - `gossip-persistence-inmemory/src/` = Boundary 5 (in-memory backend)
   - `gossip-done-ledger-postgres/src/` = Boundary 5 (Postgres done-ledger)
   - `gossip-findings-postgres/src/` = Boundary 5 (Postgres findings sink)
   - `gossip-pg-common/src/` = Boundary 5 (shared Postgres utilities)
   - `scanner-engine/src/` = Detection engine
   - `scanner-git/src/` = Git scanning pipeline
   - `scanner-scheduler/src/` = Scan scheduler
   - `gossip-scanner-runtime/src/` = Runtime orchestration
   - `gossip-coordination-etcd/src/` = Boundary 2 (etcd backend)
2. Look up all files in that boundary in this appendix
3. Read the "Topics" column to find related functionality

### Finding Tests
1. Open the source file you want to test
2. Scroll to the bottom (inline tests are usually at the end)
3. Look for `#[cfg(test)] mod tests`
4. Check for dedicated test files: `*_tests.rs` in the same directory
5. For simulation tests, look in `crates/gossip-coordination/src/sim/`
6. For scanner integration tests, look in `crates/scanner-engine-integration-tests/tests/`
7. Or search for `proptest!` for property-based tests

## Summary

This source file map provides bidirectional navigation between documentation and code:
- **Documentation -> Code**: Find implementation details for concepts
- **Code -> Documentation**: Find explanations for functions/types
- **Concept -> Files**: Locate all code related to a topic
- **File -> Concepts**: Understand what a file is responsible for

Source files are organized across 17 crates:

| Crate / Boundary | Directory | File Count |
|------------------|-----------|------------|
| B1: Identity | `gossip-contracts/src/identity/` | 11 files |
| B2: Coordination (contracts) | `gossip-contracts/src/coordination/` | 8 files |
| B2: Coordination (runtime) | `gossip-coordination/src/` | 25 files |
| B2: Simulation | `gossip-coordination/src/sim/` | 14 files |
| B2: Coordination (etcd) | `gossip-coordination-etcd/src/` | 8 files |
| B3: Shard Algebra | `gossip-frontier/src/` | 7 files |
| B4: Connector (contracts) | `gossip-contracts/src/connector/` | 9 files |
| B4: Connector (impls) | `gossip-connectors/src/` | 10 files |
| B5: Persistence (contracts) | `gossip-contracts/src/persistence/` | 11 files |
| B5: Persistence (in-memory) | `gossip-persistence-inmemory/src/` | 7 files |
| Shared data structures | `gossip-stdx/src/` | 23 files |
| Detection engine | `scanner-engine/src/` | 55 files (11 top + 5 subdirs) |
| Git scanning | `scanner-git/src/` | 106 files |
| Scan scheduler | `scanner-scheduler/src/` | 111 files (15 top + 7 subdirs) |
| Runtime orchestration | `gossip-scanner-runtime/src/` | 12 files |
| Worker binary | `gossip-worker/src/` | 1 file |
| CLI binary | `scanner-rs-cli/src/` | 1 file |
| Integration tests | `scanner-engine-integration-tests/` | 1 src + test dirs |
| B5: Done-ledger (Postgres) | `gossip-done-ledger-postgres/src/` | 8 files |
| B5: Findings (Postgres) | `gossip-findings-postgres/src/` | 9 files |
| B5: Postgres common | `gossip-pg-common/src/` | 4 files |

Use this appendix as a quick reference when navigating the Gossip-rs learning guide and codebase.
