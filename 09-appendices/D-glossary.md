# Appendix D: Glossary

This glossary defines 50+ domain terms used throughout Gossip-rs, with brief explanations and chapter references.

## A

**Accumulating**: (Design-stage — not yet implemented.) Type state for `PageCommit` during which findings can be added. Transitions to `Sealed` state when page is sealed. (Chapter 07)

**Atomicity**: Property of an operation that either completes entirely or not at all (no partial state). Example: Page commit is atomic (findings + done-ledger + cursor advance). (Chapter 07, 08-03)

**Audit Trail**: Record of all state transitions in the coordination layer. Enables debugging and compliance. (Chapter 04)

## B

**Boundary**: Architectural layer with well-defined responsibilities and dependencies. Gossip-rs has 5 boundaries. (Chapter 00, 08-01)

**BLAKE3**: Cryptographic hash function used for identity derivation. Supports derive-key mode for domain separation. (Appendix B)

**Blast Radius**: Scope of impact when a component fails. Gossip-rs minimizes blast radius (one connector failure doesn't affect others). (Chapter 08-03)

**Blocking**: Property of a task that cannot proceed until another task completes. Shards can be blocked by lease expiry. (Chapter 04)

**Budgets**: Per-operation resource limits: `max_items`, `max_bytes`, `deadline`. Passed to connector enumeration and read operations to bound resource consumption. (Chapter 06)

**byte_midpoint**: Approximate bisection point between two keys for split planning. Five-phase algorithm: pad, add, halve, try overflow-normalized, fallback successor. Used by shard algebra to compute split boundaries. (Chapter 05)

**ByteSlab**: Fixed-capacity arena allocator in `gossip-stdx` that provides bump-pointer + free-list allocation for variable-length byte fields. Pre-allocates a single contiguous buffer at startup; used by `InMemoryCoordinator` to pool `ShardRecord` spec and cursor byte fields, eliminating per-field heap allocation on hot paths. (Appendix F, Chapter 04-11)

**ByteSlot**: 16-byte handle into a `ByteSlab` containing offset, length, alloc_size, and owner_id fields (all `u32`). Deliberately `Copy` internally but wrapped in non-`Clone` pooled types (`PooledShardSpec`, `PooledCursor`) to prevent aliased handles. `ByteSlot::EMPTY` is a sentinel for absent fields. (Appendix F)

## C

**CanonicalBytes**: Trait for deterministic serialization to bytes. Used by all identity types for encoding. (Chapter 02-02)

**Circuit Breaker**: Pattern for preventing cascading failures. Opens after N failures, prevents further requests, closes after cooldown. (Chapter 06, 08-03)

**Collision**: When two distinct inputs produce the same hash. BLAKE3 provides 256-bit collision resistance (~2^128 operations to find). (Appendix B)

**Committed**: (Design-stage — not yet implemented.) Type state for `PageCommit` after findings have been durably written. Terminal state. (Chapter 07)

**Connector**: Component that bridges Gossip-rs to external systems (GitHub, S3, etc.). Implements enumeration and content reading. (Chapter 06)

**ConnectorCapabilities**: Feature-flag struct advertising what a connector supports: `seek_by_key`, `token_resume`, `range_read`, `split_hints`. Returned by connectors to let the coordination layer adapt its strategy. (Chapter 06)

**ConnectorInstance**: Convenience supertrait combining `EnumerationConnector + ReadConnector + Send`. Used as a single bound when the full connector interface is required. (Chapter 06)

**ConnectorInstanceIdHash**: 32-byte BLAKE3 derive-key hash of a connector instance identifier. Variable-length instance IDs (e.g., `"github-installation-1"`) are hashed once into a fixed-width value so that `ItemIdentityKey` framing remains simple and identity collisions are prevented when two instances scan the same locator under the same connector tag. Derived via `domain::CONNECTOR_INSTANCE_ID_V1` (`"gossip/connector-instance-id/v1"`). Defined in `item.rs`. (Chapter 02-05, Appendix B)

**ConnectorTag**: Identity type for the source system. Example: `ConnectorTag::GitHub`. (Chapter 02-05)

**Content-Addressed**: Identity derived from content, not assigned externally. Same content → same identity. (Chapter 01-01, 02-01)

**Coordination**: Boundary responsible for work distribution, lease management, progress tracking. (Chapter 04)

**CoordinationBackend**: Trait for shard lifecycle operations (acquire, checkpoint, complete, park, split) defined in `gossip-coordination`. The `CoordinationFacade` super-trait combines `CoordinationBackend`, `RunManagement`, and `ShardClaiming` into a single bound. Implementation: `InMemoryCoordinator`, persistent backends. (Chapter 04-02)

**CoordinationFacade**: Super-trait that unifies `CoordinationBackend` (shard lifecycle), `RunManagement` (run creation/completion), and `ShardClaiming` (claim-next-available). Defined in `gossip-coordination/src/facade.rs`. Used as the primary trait bound when the full coordination API is required. (Chapter 04-13)

**CommitSink**: Trait defined in `gossip-scan-driver` for persisting scan results from the detection pipeline. Bridges between the scanner and the coordination/persistence layer. (Chapter 07)

**Cursor**: Opaque token representing position in enumeration. Enables resumption after crash. (Chapter 06)

## D

**`define_canonical_input!`**: Declarative macro in `macros.rs` that generates a struct with an automatic `CanonicalBytes` implementation. Fields are written to the BLAKE3 hasher in struct declaration order, making field reordering a breaking change (it changes derived hashes). Used for `FindingIdInputs`, `OccurrenceIdInputs`, `ObservationIdInputs`, and `PolicyHashInputs`. (Chapter 02-08)

**Derive-Key Mode**: BLAKE3 mode that derives a domain-specific key from a context string. Enables domain separation. (Appendix B)

**Deterministic**: Always produces the same output for the same input. All identity derivation is deterministic. (Chapter 01-01, 02-02)

**Deterministic Simulation Testing (DST)**: Testing technique that runs the entire distributed system in a single process with controlled scheduling. (Chapter 08-05)

**Domain Separation**: Ensuring identities from different domains (FindingId vs OccurrenceId) cannot collide. BLAKE3's derive-key mode provides this. (Chapter 01-03, Appendix B)

**Done Ledger**: Persistent set of items that have been scanned. Enables exactly-once processing across runs. (Chapter 07-02)

**DoneLedgerKey**: Identity for a done-ledger entry. Includes `StableItemId` and `ObjectVersionId`. (Chapter 07-02)

## E

**EnumerateError**: Connector enumeration failure carrying an `ErrorClass` discriminant (`Retryable` or `Permanent`) plus a diagnostic message. Returned from `EnumerationConnector::enumerate_page`. (Chapter 06)

**Enumeration**: Process of listing items from a source. Connectors implement ordered enumeration within shard ranges. (Chapter 06)

**EnumerationConnector**: Trait for paginated item enumeration from external sources. Primary method: `enumerate_page` returns an `EnumerationPage` of `ScanItem`s plus a next cursor. (Chapter 06)

**EnumerationPage**: Return type from `enumerate_page`: a vec of `ScanItem`s plus an optional next cursor for pagination. (Chapter 06)

**ErrorClass**: Retryable vs Permanent connector error classification. Retryable errors (timeouts, rate limits) trigger backoff; Permanent errors (not found, access denied) abort the shard. (Chapter 06)

**Exactly-Once**: Guarantee that each item is scanned exactly once, even across crashes and reruns. Enabled by done-ledger. (Chapter 07-02, 08-02)

## F

**Fail-Safe**: Principle of rejecting invalid operations rather than silently proceeding. Example: fencing token mismatch → reject commit. (Chapter 08-03)

**Fencing Token**: Monotonically increasing counter that prevents split-brain. Only the worker with the latest token can commit. (Chapter 04-04, 08-03)

**FilesystemConnector**: Concrete connector implementation for local directory tree enumeration. Uses stack-based walk (no recursion), skips symlinks, produces deterministic ordering. Defined in `gossip-connectors`. (Chapter 06)

**FindingId**: Content-addressed identity for a finding (groups occurrences of the same secret). Derived from tenant, item, rule, and secret hash. (Chapter 02-06)

**FindingIdInputs**: Struct holding inputs for FindingId derivation: `tenant`, `item`, `rule`, `secret`. (Chapter 02-06)

**FindingsSink**: Design-stage persistence interface for writing findings (not yet implemented as a standalone trait). The production interface is `CommitSink` (defined in `gossip-scan-driver`), which handles per-item commit lifecycle including findings persistence. See `CommitSink`. (Chapter 07-03)

## G

**Golden Vector**: Known-good input/output pair for testing. Ensures identity derivation remains stable across versions. (Chapter 02-09)

**gossip-scan-driver**: Crate defining the `ScanDriver`, `ScanSourceFactory`, and `CommitSink` traits that bridge connectors to the detection engine. Lives at `crates/gossip-scan-driver/`.

**gossip-scanner-runtime**: Crate providing runtime orchestration APIs — CLI argument wiring, coordination sink, event sink, parity checking between direct and connector execution modes. Lives at `crates/gossip-scanner-runtime/`.

## H

**Half-Open Interval**: Range notation `[start, end)` includes start, excludes end. Used for shard ranges. (Chapter 03, 05-02)

**Hasher**: BLAKE3 state machine that accumulates input and produces a hash. Can be cloned to reuse key schedule. (Appendix B)

**HintPropagationError**: Error taxonomy for split-time hint propagation failures in `gossip-frontier`. Covers cases where shard metadata cannot be correctly split or forwarded to child shards. (Chapter 05)

**HMAC**: Hash-based Message Authentication Code. Gossip-rs uses keyed BLAKE3 instead (faster, simpler). (Appendix B)

## I

**IdHashMode**: Enum for hash mode: `Unkeyed` (derive-key, no tenant keying) or `KeyedV1` (with TenantSecretKey). Variants have stable `#[repr(u8)]` discriminants (0 and 1). Conversion via `from_u8(v)` / `as_u8(self)`. (Chapter 02-02)

**Identity Spine**: Boundary 1, responsible for all content-addressed identity derivation. (Chapter 02)

**Idempotency**: Property that an operation can be repeated safely (same effect as doing it once). All coordination operations are idempotent. (Chapter 04-03, 08-03)

**InlineVec<T, N>**: Stack-first small vector that stores up to N elements inline using `MaybeUninit`; spills to heap only when capacity is exceeded. Used as the backing store for `SpawnedList`. Defined in `gossip-stdx`. (Appendix F)

**InMemoryCoordinator**: Coordination backend defined in `gossip-coordination` that stores state in memory (for dev/test). (Chapter 04-02)

**Invariant**: Property that must always hold. Examples: cursor monotonicity, no double-leasing. (Chapter 04, 08-05)

**ItemKey**: Connector-specific identity for a resource. Example: GitHub URL, S3 path. (Chapter 02-05, 05-02)

**ItemRef**: Opaque handle for read operations, produced during enumeration. Passed to `ReadConnector::open` to retrieve item content. (Chapter 06)

## K

**KeyBuf**: Reusable stack buffer for shard-key arithmetic (capacity: `MAX_KEY_SIZE + 1`). Used by `gossip-frontier` key encoding operations to avoid heap allocation during split and successor computations. (Chapter 05)

**Keyed Hash**: Hash function parameterized by a secret key. Used for SecretHash to ensure tenant isolation. (Chapter 01-01, Appendix B)

**KeyEncoding**: Trait for key types that encode into lexicographically ordered bytes. Defines the ordering contract and canonicality requirements. Implemented by `PathKey` and `ManifestRowKey` in `gossip-frontier`. (Chapter 05)

**key_successor**: Minimal strict successor of an arbitrary key within `MAX_KEY_SIZE` bounds. Used to compute exclusive upper bounds for range scans. Defined in `gossip-frontier`. (Chapter 05)

**Key Schedule**: Process of deriving internal hash state from a key or context string. BLAKE3 key schedules are cached in `LazyLock`. (Appendix B)

## L

**LazyLock**: Rust type for thread-safe lazy initialization. Used to cache BLAKE3 hashers. (Appendix A, B)

**Lease**: Time-limited ownership of a shard by a worker. Expires after TTL if not renewed. (Chapter 04-03)

**Liveness**: Property that the system makes progress (doesn't deadlock). Lease expiry ensures liveness. (Chapter 04-03, 08-03)

**LogicalTime**: Abstract time type that test harnesses can control. Enables deterministic simulation testing. (Chapter 04, 08-05)

## M

**Manifest**: Metadata for a run: tenant, policy, connector, initial shards, creation time. (Chapter 04-01)

**ManifestRowKey**: Fixed-width 16-byte key encoding as `(manifest_id, row)` in big-endian `u64`s. Implements `KeyEncoding` in `gossip-frontier`. Used for manifest-based shard ranges. (Chapter 05)

**MetadataBuf**: Reusable fixed-capacity buffer for metadata encode/decode (capacity: `MAX_METADATA_SIZE`). Used by `ShardMetadata` serialization in `gossip-frontier` to avoid heap allocation. (Chapter 05)

**Monotonicity**: Property that a value only increases, never decreases. Cursors and fencing tokens are monotonic. (Chapter 04-04, 05-02)

## N

**Newtype**: Rust pattern of wrapping a primitive in a struct for type safety. All identity types are newtypes. (Appendix A)

**NormHash**: Content-addressed hash of a normalized secret match. Used to derive SecretHash. (Chapter 02-06)

**Nominal Type**: Type distinguished by name, not structure. Example: `TenantId([u8; 32])` vs `RunId(u64)` — distinct types even though both are simple wrappers. (Appendix A)

## O

**ObservationId**: Content-addressed 32-byte identity for a policy-scoped detection event. Derived from `(tenant, policy, occurrence)` using BLAKE3 derive-key mode with `domain::OBSERVATION_ID_V1` (`"gossip/observation/v1"`). Distinguishes "policy A detected occurrence O" from "policy B detected the same occurrence O" without changing the underlying `FindingId` or `OccurrenceId`. Defined in `finding.rs`. (Chapter 02-06, Appendix B)

**ObservationIdInputs**: Struct holding inputs for `ObservationId` derivation: `tenant: TenantId`, `policy: PolicyHash`, `occurrence: OccurrenceId`. All fields are fixed-width (3 × 32 = 96 bytes). Generated by `define_canonical_input!`. (Chapter 02-06)

**ObjectVersionId**: Optional identity for a version of a mutable object. Enables re-scanning when content changes. (Chapter 02-05)

**OccurrenceId**: Content-addressed identity for a specific occurrence of a secret (unique finding + location). (Chapter 02-06)

**OccurrenceIdInputs**: Struct holding inputs for OccurrenceId derivation: `finding: FindingId`, `version: ObjectVersionId`, `byte_offset: u64`, `byte_length: u64`. (Chapter 02-06)

**OP_LOG_CAP**: Compile-time constant (`const OP_LOG_CAP: usize = 16`) defining the maximum number of retained op-log entries per shard. Determines the bounded idempotency window. Verified at compile time via `const _: () = assert!(ShardRecord::OP_LOG_CAP == 16);`. (Chapter 04-03)

**OpId**: Idempotency token derived from fencing token. Used to detect duplicate operations. (Chapter 04-03)

**Op-Log**: Log of operations with their OpIds. Enables idempotency checks. (Chapter 04-03)

**OvidHash**: Hash of `ObjectVersionId` for use in done-ledger keys. (Chapter 02-05, 07-02)

## P

**PageCommit**: (Design-stage — not yet implemented.) Typestate-encoded accumulator for findings in a page. States: Accumulating, Sealed, Committed. (Chapter 07-04)

**PageValidationError**: Violation plus diagnostic details for page-level validation failures. Returned by the page validator when connector output violates ordering, uniqueness, or budget constraints. (Chapter 06)

**ParkReason**: Enum explaining why a shard was parked: `SourceUnreachable`, `RateLimited`, etc. (Chapter 04-06, 08-03)

**PathKey**: UTF-8 path key encoded as identity bytes with no normalization. Implements `KeyEncoding` in `gossip-frontier`. Used for filesystem-style shard ranges. (Chapter 05)

**PolicyHash**: Content-addressed identity for a detection policy. Ensures run uses the intended policy version. (Chapter 02-07)

**PolicyHashInputs**: Struct holding inputs for PolicyHash derivation: `policy_json`, `rule_fingerprints`. (Chapter 02-07)

**PooledCursor**: Arena-pooled mirror of `Cursor` backed by `Option<ByteSlot>` handles into a `ByteSlab`. Holds 0-2 slots for `last_key` and `token`. Uses `Option<ByteSlot>` (not bare `ByteSlot::EMPTY`) to preserve the semantic distinction between "no progress" (`None`) and "present but empty." (Chapter 04-01, Appendix F)

**PooledShardSpec**: Arena-pooled mirror of `ShardSpec` backed by `ByteSlot` handles into a `ByteSlab`. Holds exactly 3 slots for `key_range_start`, `key_range_end`, and `metadata`. Intentionally not `Copy` or `Clone` to prevent aliased handles (SLAB-2). (Chapter 04-01, Appendix F)

**PreallocShardBuilder**: Startup-preallocated shard builder in `gossip-frontier` with two-phase workflow: stage shard specs, then finalize into immutable shard set. Avoids runtime allocation during shard construction. (Chapter 05)

**prefix_successor**: Exclusive upper bound for a prefix scan (analogous to FoundationDB `strinc`). Given a byte prefix, returns the minimal key strictly greater than all keys sharing that prefix. Defined in `gossip-frontier`. (Chapter 05)

**Preimage Resistance**: Property that given hash `H(x) = y`, it's infeasible to find `x`. BLAKE3 provides 256-bit preimage resistance. (Appendix B)

**Proptest**: Rust library for property-based testing. Used extensively in identity tests. (Appendix C)

**PRF**: Pseudorandom Function. Keyed hash functions (like BLAKE3-keyed) are PRFs. (Appendix B)

## R

**Range Sharding**: Partitioning work by key range. Enables parallelism and resumability. (Chapter 03, 05-02)

**ReadConnector**: Trait for opening item content as a streaming reader. Given an `ItemRef` produced during enumeration, returns an `AsyncRead` handle for the item's bytes. (Chapter 06)

**Redacted Debug**: Custom `Debug` impl that hides sensitive data. Used for `TenantSecretKey`. (Appendix A)

**RingBuffer<T, N>**: Fixed-capacity, stack-allocated FIFO ring buffer backed by `MaybeUninit` with power-of-2 bitwise index arithmetic. Used as `RingBuffer<OpLogEntry, 16>` for bounded shard op-logs. `push_back_overwrite` provides O(1) FIFO eviction. Defined in `gossip-stdx`. (Appendix F)

**Roundtrip**: Property that `decode(encode(x)) == x`. All identity types support lossless serialization. (Appendix C)

**RuleFingerprint**: Content-addressed 32-byte identity for a detection rule, defined in `finding.rs` via `define_id_32!`. The engine/policy layer computes rule fingerprints externally and passes them into finding derivation. The contracts crate treats the value as opaque. Used as an input to `FindingIdInputs` for `derive_finding_id`, and referenced by `domain::RULE_FINGERPRINT_V1` (`"gossip/rule/v1"`). Invariant: the same rule definition always produces the same `RuleFingerprint`; if detection semantics change, the fingerprint changes. (Chapter 02-06, 02-07)

**RunId**: Identity for a scan run. A `u64`-based type generated by `define_id_64!` (not content-addressed, not UUID). (Chapter 04-01)

**RunManifest**: Metadata for a run. Includes tenant, policy, connector, initial shards. (Chapter 04-01)

## S

**Safety**: Property that nothing bad happens (no data corruption, no split-brain). Fencing tokens ensure safety. (Chapter 04-04, 08-03)

**ScanDriver**: Trait defined in `gossip-scan-driver` for orchestrating a scan pass over a source. Implementations wire together enumeration, detection, and result persistence.

**scanner-engine**: Crate containing the standalone detection engine — YARA rule compilation, regex-to-anchor optimization, content scanning, and match extraction. Lives at `crates/scanner-engine/`.

**scanner-git**: Crate implementing the Git scanning pipeline — pack file decoding, commit graph walking, blob introduction analysis, diff-based history scanning. Lives at `crates/scanner-git/`.

**scanner-scheduler**: Crate implementing the parallel scan scheduler — thread pool management, work scheduling, archive extraction, pipeline coordination, and simulation harnesses. Lives at `crates/scanner-scheduler/`.

**ScanSourceFactory**: Trait defined in `gossip-scan-driver` for creating scan sources. Produces items to feed into the detection pipeline.

**ScanItem**: Enumerated item bundling key, ref, stable ID, version, and optional metadata. Produced by `EnumerationConnector::enumerate_page` and consumed by the scan pipeline. (Chapter 06)

**Sealed**: (Design-stage — not yet implemented.) Type state for `PageCommit` after sealing (no more findings can be added). Ready for commit. (Chapter 07-04)

**SecretHash**: Keyed hash of NormHash using TenantSecretKey. Ensures tenant isolation for FindingId. (Chapter 02-06, 08-04)

**Shard**: Unit of work (a key range) assigned to a worker. Workers enumerate items within shard ranges. (Chapter 03, 04)

**ShardHint**: Routing hint enum in `gossip-frontier`: Range, Prefix, or Manifest row range. Carried inside `ShardMetadata` to guide connector enumeration strategy. (Chapter 05)

**ShardMetadata**: Envelope wrapping a `ShardHint` plus opaque connector-extra bytes. Wire-framed with length-prefixed encoding for zero-copy decode. Defined in `gossip-frontier`. (Chapter 05)

**SlabFull**: Error returned when a `ByteSlab` cannot accommodate a requested allocation. Propagated as `Result<_, SlabFull>` from `ShardRecord` constructors and cursor update methods. Indicates the coordinator's arena needs a larger `slab_capacity` in `CoordinatorRuntimeConfig`. (Appendix F, Chapter 04-11)

**ShardId**: Identity for a shard. A `u64`-based type generated by `define_id_64!`; derived (split) shards have bit 63 set. (Chapter 04-02)

**ShardSpecScratch**: Caller-owned scratch buffer for allocation-free shard-spec construction. Avoids heap allocation on hot paths when building `ShardSpec` values. (Chapter 04, 05)

**SIMD**: Single Instruction Multiple Data. CPU instruction set for parallel processing. BLAKE3 uses SIMD for speed. (Appendix B)

**Split**: Operation that divides a shard into two smaller shards for load balancing. (Chapter 03, 04-05)

**SplitBoundary**: Discriminant for split-boundary validation errors: `Start` or `End`. Indicates which boundary of a proposed split range failed validation. Used in `gossip-frontier` split computations. (Chapter 05)

**SpawnedList**: Type alias `InlineVec<ShardId, MAX_SPAWNED_PER_SHARD>` where `MAX_SPAWNED_PER_SHARD = 1024`, used for tracking shards created via split operations. Defined in `gossip-coordination/src/record.rs` (test-only alias; runtime records use `PooledSpawned` for slab-backed storage). (Chapter 04-06)

**StableItemId**: Content-addressed identity for an item that remains stable across runs. (Chapter 02-05)

## T

**TenantId**: Identity for a tenant (organization). Used everywhere for multi-tenancy. (Chapter 02-04, 08-04)

**TenantSecretKey**: 32-byte secret key for tenant isolation. Used in keyed hash for SecretHash. (Chapter 02-06, 08-04)

**TokenBytes**: Opaque pagination token wrapper with toxic-byte redaction. Wraps connector-produced resume tokens and sanitizes them for safe logging. (Chapter 06)

**Typestate**: Rust pattern of encoding state in the type system. `PageCommit<S>` uses typestate (design-stage — not yet implemented). (Appendix A)

**TTL**: Time-To-Live. Leases expire after TTL if not renewed. (Chapter 04-03)

## U

**UUID**: Universally Unique IDentifier. Note: coordination types (`RunId`, `ShardId`, `WorkerId`) in Gossip-rs are `u64`-based (`define_id_64!`), not UUIDs. (Chapter 04)

## W

**WorkerSession**: Context for a worker's shard assignment: shard ID, range, fencing token, lease expiry. (Chapter 04-03)

**WorkerId**: Identity for a worker process. A `u64`-based type generated by `define_id_64!`. (Chapter 04)

## Z

**Zero-Cost Abstraction**: Rust abstraction with no runtime overhead. Examples: newtypes, PhantomData, const functions. (Appendix A)

## Acronyms

- **API**: Application Programming Interface
- **B1-B5**: Boundaries 1 through 5
- **DB**: Database
- **DST**: Deterministic Simulation Testing
- **I/O**: Input/Output
- **MAC**: Message Authentication Code
- **OOM**: Out Of Memory
- **PBT**: Property-Based Testing
- **PRF**: Pseudorandom Function
- **PRNG**: Pseudorandom Number Generator
- **RPC**: Remote Procedure Call
- **SHA**: Secure Hash Algorithm
- **TUI**: Terminal User Interface

## Chapter Cross-Reference

| Term | Primary Chapters |
|------|------------------|
| Boundary | 00, 08-01 |
| Content-Addressed | 01-01, 02-01 |
| BLAKE3 | 01-01, Appendix B |
| Domain Separation | 01-03, 02-03, Appendix B |
| Canonical Encoding | 01-02, 02-02 |
| TenantId | 02-04, 08-04 |
| StableItemId | 02-05 |
| FindingId | 02-06 |
| OccurrenceId | 02-06 |
| ObservationId | 02-06, Appendix B |
| PolicyHash | 02-07 |
| Golden Vector | 02-09 |
| Range Sharding | 03 |
| Coordination | 04 |
| Arena Pooling | 04-01, 04-11, Appendix F |
| Lease | 04-03 |
| Fencing Token | 04-04 |
| Idempotency | 04-03, 08-03 |
| Connector | 06 |
| Circuit Breaker | 06, 08-03 |
| Cursor | 06 |
| Persistence | 07 |
| PageCommit | 07-04 (design-stage) |
| Done Ledger | 07-02 |
| Tenant Isolation | 08-04 |
| Failure Recovery | 08-03 |
| DST | 08-05 |
| Proptest | Appendix C |

## See Also

For in-depth explanations:
- **Boundary definitions**: Chapter 00 (Architecture Overview)
- **Identity derivation**: Chapters 02-01 through 02-10
- **Coordination protocols**: Chapter 04
- **Failure recovery**: Chapter 08-03
- **Rust patterns**: Appendix A
- **BLAKE3 internals**: Appendix B
- **Property-based testing**: Appendix C

**Next**: [Appendix E: Source File Map](./E-source-file-map.md) maps source files to chapters.
