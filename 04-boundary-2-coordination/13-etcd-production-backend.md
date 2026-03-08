# When Memory Forgets -- The etcd Production Backend

*The cluster has been running for fourteen hours. Six workers coordinate 50,000 shards across 12 tenants, scanning an 11-petabyte corpus. The in-memory coordinator holds the entire truth: which shards are active, which workers hold leases, where each cursor last checkpointed. At 02:17 UTC the coordinator process receives SIGKILL — an OOM event triggered by a co-located batch job. The process restarts in 400 milliseconds. But when it comes back, the `InMemoryCoordinator` is empty. Every `RunRecord` is gone. Every `ShardRecord` is gone. The 50,000 shards that were mid-scan have no cursors, no lease owners, no fence epochs. Workers attempt to renew leases against shard keys that no longer exist and receive `ShardNotFound` errors. The entire scan must be re-discovered from external metadata, re-registered, and re-assigned from the beginning. Fourteen hours of incremental cursor progress — checkpointed but never persisted to disk — evaporates in a single process restart. The in-memory coordinator works perfectly for development and simulation testing. It does not survive production.*

---

The `InMemoryCoordinator` implements every coordination trait with full protocol correctness. It passes the simulation test suite. Its shard lifecycle, lease fencing, and idempotency logic are verified against the TLA+ specification. But it stores all state in process memory, which means a coordinator restart is a total-loss event. In a production deployment, the coordinator must survive restarts, rolling upgrades, and node failures without losing track of which shards are mid-scan, which workers hold leases, or where cursors last checkpointed.

etcd is a natural choice for this persistence layer. It provides linearizable key-value storage with transactional multi-key writes, lease-based TTLs, and prefix-range scans — exactly the primitives needed for coordination state. The `gossip-coordination-etcd` crate builds the bridge between the coordination protocol and a live etcd cluster.

The crate provides `EtcdCoordinator`, a backend that connects to a real etcd cluster over gRPC, constructs a deterministic keyspace for all coordination records, and defines a versioned binary codec for serializing `RunRecord` and `ShardRecord` values. It is structured in five modules: `config` for validated connection parameters, `keyspace` for deterministic key construction, `codec` for binary serialization, `backend` for the coordinator struct and trait implementations, and `error` for unified failure types.

The entire crate is marked `#![forbid(unsafe_code)]`. There is no unsafe Rust anywhere in the etcd backend. Every field encoding, bounds check, and allocation rollback operates through safe Rust abstractions.

## Validated Configuration

Before any network I/O occurs, the etcd backend validates its connection parameters through `EtcdCoordinatorConfig`. Configuration validation happens at construction time, not at first use — a pattern that ensures the system fails immediately with a clear error message rather than minutes later when the first etcd RPC fires. The struct captures two pieces of information: a list of etcd cluster endpoints and a namespace prefix that scopes all coordination keys.

From `config.rs`:

```rust
#[derive(Clone, PartialEq, Eq)]
pub struct EtcdCoordinatorConfig {
    endpoints: Vec<String>,
    namespace_prefix: String,
}
```

Construction goes through `new()`, which trims whitespace from every endpoint and the prefix before running validation. This normalization step matters in practice: endpoints read from environment variables or YAML config files often carry trailing whitespace or newlines that would otherwise cause silent connection failures. The `validate()` method enforces seven rules, each with a dedicated error variant.

From `config.rs`:

```rust
pub fn validate(&self) -> Result<(), EtcdCoordinatorConfigError> {
    if self.endpoints.is_empty() {
        return Err(EtcdCoordinatorConfigError::NoEndpoints);
    }

    for (index, endpoint) in self.endpoints.iter().enumerate() {
        if endpoint.is_empty() {
            return Err(EtcdCoordinatorConfigError::EmptyEndpoint { index });
        }
        if !(endpoint.starts_with("http://") || endpoint.starts_with("https://")) {
            return Err(EtcdCoordinatorConfigError::InvalidEndpointScheme { index });
        }
    }

    if self.namespace_prefix.is_empty() {
        return Err(EtcdCoordinatorConfigError::EmptyNamespacePrefix);
    }
    if !self.namespace_prefix.starts_with('/') {
        return Err(EtcdCoordinatorConfigError::NamespacePrefixMustStartWithSlash);
    }
    if self.namespace_prefix.ends_with('/') && self.namespace_prefix.len() > 1 {
        return Err(EtcdCoordinatorConfigError::NamespacePrefixMustNotEndWithSlash);
    }
    if self.namespace_prefix.contains("//") {
        return Err(EtcdCoordinatorConfigError::NamespacePrefixContainsDoubleSlash);
    }

    Ok(())
}
```

The seven validation error variants map one-to-one with these checks.

From `config.rs`:

```rust
pub enum EtcdCoordinatorConfigError {
    NoEndpoints,
    EmptyEndpoint { index: usize },
    InvalidEndpointScheme { index: usize },
    EmptyNamespacePrefix,
    NamespacePrefixMustStartWithSlash,
    NamespacePrefixMustNotEndWithSlash,
    NamespacePrefixContainsDoubleSlash,
}
```

Each variant carries just enough context to pinpoint the problem. `EmptyEndpoint` and `InvalidEndpointScheme` include the offending index so operators can identify which entry in a multi-endpoint list is wrong. The `NoEndpoints` variant catches the case where an empty list is passed — this is distinct from passing a list with blank entries. The namespace prefix rules prevent two classes of bugs: a prefix without a leading slash would produce keys that are not absolute etcd paths, and a prefix with a trailing slash would produce double slashes when child segments are appended. The `NamespacePrefixContainsDoubleSlash` variant guards against prefixes like `/gossip//v1` that would create invisible empty path segments in the etcd keyspace — segments that are difficult to debug with `etcdctl` because they appear identical to the intended path in most terminal output.

### Credential Redaction in Debug Output

Endpoint URIs can contain embedded credentials (e.g., `http://user:pass@etcd-0:2379`). The `Debug` implementation strips the userinfo portion before rendering, so credentials never leak into logs or diagnostic output.

From `config.rs`:

```rust
impl fmt::Debug for EtcdCoordinatorConfig {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        fn redact_endpoint(endpoint: &str) -> String {
            if let Some(scheme_end) = endpoint.find("://") {
                let authority_start = scheme_end + 3;
                let rest = &endpoint[authority_start..];
                let path_start = rest.find('/').unwrap_or(rest.len());
                if let Some(at_pos) = rest[..path_start].find('@') {
                    return format!("{}://***@{}", &endpoint[..scheme_end], &rest[at_pos + 1..]);
                }
            }
            endpoint.to_owned()
        }

        let redacted: Vec<String> = self.endpoints.iter().map(|e| redact_endpoint(e)).collect();
        f.debug_struct("EtcdCoordinatorConfig")
            .field("endpoints", &redacted)
            .field("namespace_prefix", &self.namespace_prefix)
            .finish()
    }
}
```

An endpoint `http://admin:secret@etcd-0:2379` renders as `http://***@etcd-0:2379`. Endpoints without userinfo pass through unchanged. This is security-by-default: a developer who writes `tracing::debug!(?config)` never accidentally exposes credentials.

Two convenience constructors cover common cases. `localhost()` targets `http://127.0.0.1:2379` with prefix `/gossip/v1` for local development. It routes through `new()` so the hard-coded values stay in lockstep with the validation rules — if the rules ever change, the `expect` inside `localhost()` will panic at compile-test time rather than silently producing an invalid config. `from_endpoints_csv()` splits a comma-separated string for environment-variable-driven configuration (`ETCD_ENDPOINTS=http://host1:2379,http://host2:2379`). It silently drops segments that are empty after trimming, so trailing commas in environment variables do not cause validation failures.

## The etcd Keyspace

Every coordination record maps to exactly one etcd key. The `EtcdKeyspace` struct owns this mapping, ensuring that persistence code never constructs raw key strings. This centralization matters: if two modules independently formatted keys, even a single-character discrepancy (e.g., uppercase vs. lowercase hex) would produce phantom keys that exist in etcd but are invisible to scans, or keys that collide when they should be distinct.

From `keyspace.rs`:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct EtcdKeyspace {
    prefix: String,
}
```

Construction validates the same prefix invariants as `EtcdCoordinatorConfig`: must start with `/`, must not end with `/` (unless it is exactly `"/"`), must not contain consecutive slashes, and must not be empty. The `EtcdKeyspace` has its own error type, `EtcdKeyspaceError`, with four variants mirroring these rules. The duplication is intentional: the config validates the user's input at the API boundary, and the keyspace validates the prefix at the storage boundary. If the config and keyspace are constructed independently (e.g., from different code paths), both boundaries remain protected.

### Hierarchical Key Layout

The key tree is designed for etcd prefix scans. Each category of record sits under a distinct path segment so a single `get_prefix` call enumerates all records of that kind without false positives.

```text
{prefix}/
  tenants/
    {tenant_hex}/                        # 64 lowercase hex chars (32-byte TenantId)
      runs/
        {run_hex}/                       # 16 zero-padded hex chars (u64 RunId)
          shards/
            {shard_hex}                  # 16 zero-padded hex chars (u64 ShardId)
            {shard_hex}/owner            # per-shard ownership key
          shards_active/
            {shard_hex}                  # active-shard index entry
      runs_active/
        {run_hex}                        # active-run index entry
```

The fixed-width hex encoding is deliberate. `TenantId` is a 32-byte value rendered as 64 lowercase hex characters. `RunId` and `ShardId` are `u64` values rendered as 16 zero-padded lowercase hex characters. Fixed width means lexicographic key order matches numeric order, which keeps etcd range scans predictable. All hex output uses `[0-9a-f]` exclusively — no uppercase letters — so byte-level equality works for key comparison.

A concrete example: for prefix `/gossip/v1`, tenant `0xABAB...AB` (32 bytes), run `0x42`, and shard `0xFF`:

- Run record: `/gossip/v1/tenants/abab...ab/runs/0000000000000042`
- Shard record: `/gossip/v1/tenants/abab...ab/runs/0000000000000042/shards/00000000000000ff`
- Shard owner: `/gossip/v1/tenants/abab...ab/runs/0000000000000042/shards/00000000000000ff/owner`
- Active run index: `/gossip/v1/tenants/abab...ab/runs_active/0000000000000042`
- Active shard index: `/gossip/v1/tenants/abab...ab/runs/0000000000000042/shards_active/00000000000000ff`

### Sibling Separation for Scan Isolation

The `runs/` and `runs_active/` directories are siblings under the tenant, not nested. The same holds for `shards/` and `shards_active/` under each run. The trailing slash on scan prefixes is critical.

From `keyspace.rs`:

```rust
pub fn run_records_scan_prefix_into(&self, buf: &mut String, tenant: TenantId) {
    self.runs_prefix_into(buf, tenant);
    buf.push('/');
}
```

Without the trailing slash, an etcd prefix scan on `.../runs` would also match `.../runs_active/...` because `"runs"` is a prefix of `"runs_active"`. The trailing slash restricts the scan to children of the `runs/` directory only. This is a subtle but critical invariant. Without it, a scan intended to enumerate run records would also return active-run index entries, producing spurious results that would be silently misinterpreted by code expecting only `RunRecord` blobs. The same trailing-slash convention applies to `shard_records_scan_prefix`, preventing cross-contamination between shard records and active-shard index entries.

The ownership key (`{shard_hex}/owner`) is stored as a child of the shard record key. This nesting is deliberate: a range-delete on the shard record key prefix also removes the ownership key in a single etcd operation, preventing orphaned ownership entries that could cause phantom lease holders.

### Buffer-Reuse API for Zero-Allocation Key Construction

Every key method has a corresponding `_into` variant that appends into a caller-owned `&mut String` without clearing it first. This allows hot-path callers to reuse a single buffer across multiple key constructions, avoiding per-call heap allocation.

From `keyspace.rs`:

```rust
pub fn shard_record_key_into(
    &self,
    buf: &mut String,
    tenant: TenantId,
    run: RunId,
    shard: ShardId,
) {
    self.run_shards_prefix_into(buf, tenant, run);
    buf.push('/');
    shard_hex_into(buf, shard);
}
```

The convenience methods that return `String` delegate to their `_into` counterparts internally. A persistence loop that builds keys for hundreds of shards can allocate a single `String` buffer, call `_into` for each key, submit the etcd request, call `buf.clear()`, and repeat — amortizing the allocation cost across the entire batch. The hex encoding itself uses a lookup table rather than `format!` to avoid per-byte format-string parsing.

From `keyspace.rs`:

```rust
fn encode_hex_into(buf: &mut String, bytes: &[u8]) {
    const LUT: &[u8; 16] = b"0123456789abcdef";
    buf.reserve(bytes.len() * 2);
    for &byte in bytes {
        buf.push(LUT[(byte >> 4) as usize] as char);
        buf.push(LUT[(byte & 0x0f) as usize] as char);
    }
}
```

## The Binary Codec

Coordination records must be persisted as etcd values with deterministic round-trip fidelity. A record encoded and then decoded must produce an identical record, byte for byte, field for field. The codec is hand-written rather than serde-derived so that every byte is explicit and the decode path can interleave validation with field reads — rejecting malformed blobs early without wasted allocation. A serde-based approach would deserialize first and validate second, meaning a corrupted blob could trigger arbitrary allocation before validation even begins. The hand-written codec rejects invalid data at the earliest possible byte.

### Wire Format

Every blob begins with a 3-byte header.

From `codec.rs`:

```rust
const VERSION_PREFIX_V1: &[u8; 2] = b"v1";

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
#[repr(u8)]
pub enum BlobKind {
    RunRecord = 1,
    ShardRecord = 2,
}
```

The header layout is `[version: 2 bytes ("v1")] [kind: 1 byte (BlobKind discriminant)]`. After the header, fields are encoded sequentially in little-endian byte order with no alignment padding. Variable-length fields (byte slices, vectors) are length-prefixed with a `u32` count. Optional fields use a 1-byte bool presence tag (`0` = absent, `1` = present). Enum discriminants are encoded as raw `u8` values matching each type's `as_u8()` representation.

The encoding primitives are symmetric pairs. Each `put_*` function on the encode side has a matching `read_*` method on the `Decoder`. For example, `put_u64` writes 8 little-endian bytes, and `read_u64` reads 8 bytes and reconstructs the `u64`. `put_bytes` writes a `u32` length followed by the raw bytes, and `read_vec` reads the length, bounds-checks it against `MAX_FIELD_SIZE` (1 MiB), then reads that many bytes. `put_opt_u64` writes a 1-byte presence tag followed by the value if present, and `read_opt_u64` reads the tag and conditionally reads the value. This symmetry makes the codec self-documenting: reading the encode function reveals the exact wire layout.

### Encoding

Encoding converts a domain `RunRecord` or `ShardRecord` into its owned intermediate, then writes the header followed by all fields in deterministic order.

From `codec.rs`:

```rust
pub fn encode_run_record(record: &RunRecord) -> Vec<u8> {
    let owned = OwnedRunRecord::from_record(record);
    let mut out = Vec::with_capacity(256);
    out.extend_from_slice(VERSION_PREFIX_V1);
    out.push(BlobKind::RunRecord.as_u8());
    owned.encode_into(&mut out);
    out
}
```

Shard encoding reads pooled fields from the caller's `ByteSlab` to extract byte content. The slab is borrowed immutably — no allocations occur during encoding. Unlike `encode_run_record`, shard encoding is fallible: it returns `Result` because the `OwnedShardRecord::encode_into` method validates the lease invariant (owner and deadline must be jointly present or absent) and returns `EtcdCodecError::InvariantViolation` on mismatch.

From `codec.rs`:

```rust
pub fn encode_shard_record(
    record: &ShardRecord,
    slab: &ByteSlab,
) -> Result<Vec<u8>, EtcdCodecError> {
    let owned = OwnedShardRecord::from_record(record, slab);
    let mut out = Vec::with_capacity(256);
    out.extend_from_slice(VERSION_PREFIX_V1);
    out.push(BlobKind::ShardRecord.as_u8());
    owned.encode_into(&mut out)?;
    Ok(out)
}
```

### Two-Phase Decode: Parse then Materialize

Decoding proceeds in two distinct phases. First, the blob is parsed into heap-owned intermediates (`OwnedRunRecord`, `OwnedShardRecord`) that hold all decoded values in plain `Vec<u8>` and `Vec<ShardId>` form. Structural validation happens immediately during this phase.

Second, materialization converts the owned intermediates into domain types. For `RunRecord` this is a relatively cheap operation: it constructs a `RunConfig` from flat fields and populates a `RingBuffer` op log. For `ShardRecord` it allocates pooled fields (`PooledShardSpec`, `PooledCursor`, `PooledSpawned`) into the caller's `ByteSlab`. This second phase is where the slab-backed allocation model creates complexity — and where the strong exception guarantee becomes essential.

From `codec.rs`:

```rust
pub fn decode_shard_record(
    blob: &[u8],
    slab: &mut ByteSlab,
) -> Result<ShardRecord, EtcdCodecError> {
    let mut decoder = Decoder::new(blob);
    decoder.expect_header(BlobKind::ShardRecord)?;
    let owned = OwnedShardRecord::decode(&mut decoder)?;
    decoder.finish()?;
    owned.into_record(slab)
}
```

The `Decoder` is a forward-only cursor over an immutable byte slice. All `read_*` methods advance the offset by exactly the number of bytes consumed. There is no seek or backtrack — the wire format is designed for single-pass sequential reads. On any read failure, the cursor position is unspecified, but the underlying slice is never mutated, so the caller can safely discard the decoder on error. The `finish()` call at the end asserts that every byte has been consumed — any trailing bytes produce `EtcdCodecError::TrailingBytes`, catching blobs that are longer than expected. This bidirectional length check (truncation on underflow, trailing bytes on overflow) ensures that any byte mismatch between encoder and decoder is detected rather than silently ignored.

The separation into two phases serves a deeper purpose: it keeps wire-format concerns out of the domain types. The `OwnedShardRecord` intermediate knows about byte slices and `Vec<ShardId>`. The domain `ShardRecord` knows about `PooledShardSpec` and `ByteSlab`. Neither crosses into the other's territory.

### Strong Exception Guarantee on Slab Rollback

The `ShardRecord` materialization path allocates incrementally into the slab: first the spec, then the cursor, then the spawned children. A `StagedShardAllocations` guard accumulates successful allocations. If any step fails, all previously staged allocations are rolled back in reverse order, restoring the slab to its pre-call state.

From `codec.rs`:

```rust
struct StagedShardAllocations {
    spec: Option<PooledShardSpec>,
    cursor: Option<PooledCursor>,
    spawned: Option<PooledSpawned>,
}

impl StagedShardAllocations {
    fn rollback(mut self, slab: &mut ByteSlab) {
        if let Some(mut s) = self.spawned.take() {
            s.release_fields(slab);
        }
        if let Some(mut c) = self.cursor.take() {
            c.release_fields(slab);
        }
        if let Some(mut s) = self.spec.take() {
            s.release_fields(slab);
        }
    }
}
```

The `into_record` method orchestrates this pattern:

From `codec.rs`:

```rust
fn into_record(self, slab: &mut ByteSlab) -> Result<ShardRecord, EtcdCodecError> {
    self.validate()?;
    let mut staged = StagedShardAllocations::new();
    match self.try_materialize(slab, &mut staged) {
        Ok(record) => Ok(record),
        Err(e) => {
            staged.rollback(slab);
            Err(e)
        }
    }
}
```

On success, all fields are moved out of the staged guard (setting each to `None`), so the guard's destructor becomes a no-op. On failure, the explicit `rollback` releases exactly the allocations that succeeded. The slab is never left in a half-allocated state. This matters because the slab is a shared resource: multiple shard records are decoded into the same slab during a prefix scan. A leak from one failed decode would reduce capacity for subsequent decodes, eventually producing spurious `SlabFull` errors that have no connection to actual memory pressure.

The `validate()` method is called twice intentionally — once after the initial decode and once at the start of `into_record`. The first call catches wire corruption early. The second is defense-in-depth against the owned intermediate reaching materialization in an inconsistent state, guarding against logic errors in the caller or in intermediate transformations.

### Codec Error Variants

The codec defines 12 error variants covering every failure mode from truncated input to structural invariant violations.

From `codec.rs`:

```rust
pub enum EtcdCodecError {
    Truncated { needed: usize, remaining: usize },
    InvalidVersionPrefix { actual: [u8; 2] },
    InvalidBlobKind { actual: u8 },
    UnexpectedBlobKind { expected: BlobKind, actual: BlobKind },
    InvalidBool { actual: u8 },
    InvalidEnum { ty: &'static str, actual: u8 },
    InvalidSpec { message: String },
    InvalidCursor { message: String },
    SlabFull(SlabFull),
    TrailingBytes { remaining: usize },
    InvariantViolation { kind: &'static str, detail: &'static str },
    FieldTooLarge { actual: usize, max: usize },
}
```

`Truncated` fires when the blob ends before the decoder reads the requested bytes — the most common symptom of a partial write or network corruption. `InvalidVersionPrefix` catches blobs from a different codec version or from a completely different system that happens to share the same etcd cluster (which the namespace prefix is supposed to prevent, but defense-in-depth applies). `UnexpectedBlobKind` catches attempts to decode a `ShardRecord` blob as a `RunRecord`, or vice versa — a scenario that arises if the caller passes the wrong key's value to the decoder. `InvalidEnum` catches unknown discriminants for any enum type, carrying the type name for diagnostics. `SlabFull` wraps the underlying allocation failure, preserving the error chain through the standard `Error::source()` method. `FieldTooLarge` guards against crafted length prefixes that would trigger multi-gigabyte allocations — the decoder caps any single variable-length field at 1 MiB (`MAX_FIELD_SIZE`). `InvariantViolation` is the most semantically rich variant: it carries a `kind` (which record type) and a `detail` (which specific invariant was violated), producing error messages like "decoded ShardRecord violates invariant: Parked shard must have park_reason".

### Defense Against Untrusted Input

The decoder applies structural bounds at every collection read. Before allocating a `Vec` for root shards, it checks the declared length against remaining wire bytes:

From `codec.rs`:

```rust
let root_len = decoder.read_len()?;
if root_len > decoder.remaining() / SHARD_ID_WIRE_SIZE {
    return Err(EtcdCodecError::InvariantViolation {
        kind: "RunRecord",
        detail: "root_shards length exceeds remaining wire bytes",
    });
}
```

This prevents a crafted `u32` length prefix from triggering a multi-GB `Vec::with_capacity` allocation. The same pattern appears for spawned children (`MAX_SPAWNED_PER_SHARD`), op log entries (`OP_LOG_CAP`), and byte vectors (`MAX_FIELD_SIZE = 1 MiB`).

## The EtcdCoordinator Struct

The backend struct ties configuration, keyspace, runtime, client, and protocol delegate together into a single unit that is constructed once and reused for the lifetime of the coordinator process.

From `backend.rs`:

```rust
pub struct EtcdCoordinator {
    config: EtcdCoordinatorConfig,
    keyspace: EtcdKeyspace,
    runtime: tokio::runtime::Runtime,
    client: etcd_client::Client,
    protocol_delegate: InMemoryCoordinator,
}
```

The `runtime` field is a current-thread Tokio runtime that bridges the synchronous coordination traits to the async etcd client. The coordination traits are deliberately synchronous (they return `Result<T, E>`, not futures) because the deterministic simulator cannot use an async runtime. The `client` field holds a live gRPC connection to the etcd cluster. The `protocol_delegate` is an `InMemoryCoordinator` that holds all shard and run state.

Five fields, each with a distinct responsibility: `config` remembers the validated connection parameters, `keyspace` builds keys, `runtime` bridges sync-to-async, `client` talks to etcd, and `protocol_delegate` implements protocol logic.

### Connection and Health Check

The `connect()` constructor performs a two-phase fail-fast initialization:

From `backend.rs`:

```rust
pub fn connect(config: EtcdCoordinatorConfig) -> Result<Self, EtcdCoordinatorError> {
    let runtime = tokio::runtime::Builder::new_current_thread()
        .enable_io()
        .enable_time()
        .build()
        .map_err(EtcdCoordinatorError::RuntimeBuild)?;

    let endpoints = config.endpoints().to_vec();
    let connect_opts =
        etcd_client::ConnectOptions::new().with_connect_timeout(DEFAULT_CONNECT_TIMEOUT);

    debug_assert!(
        tokio::runtime::Handle::try_current().is_err(),
        "connect() must not be called from within an active Tokio runtime"
    );

    let mut client = runtime
        .block_on(etcd_client::Client::connect(endpoints, Some(connect_opts)))
        .map_err(|source| EtcdCoordinatorError::Etcd {
            operation: EtcdOperation::Connect,
            source,
        })?;

    runtime
        .block_on(client.status())
        .map_err(|source| EtcdCoordinatorError::Etcd {
            operation: EtcdOperation::Status,
            source,
        })?;

    let keyspace = EtcdKeyspace::new(config.namespace_prefix())?;

    Ok(Self {
        config,
        keyspace,
        runtime,
        client,
        protocol_delegate: InMemoryCoordinator::new(DEFAULT_BOOTSTRAP_LEASE_DURATION),
    })
}
```

First, it establishes a gRPC channel with a 5-second connect timeout (`DEFAULT_CONNECT_TIMEOUT`). Five seconds is generous for LAN and localhost connections but fast enough to surface misconfigurations during startup — a firewall-dropped packet or wrong host will fail in 5 seconds rather than hanging indefinitely. Then it round-trips a maintenance `Status` RPC to confirm the cluster is reachable and responsive. This two-step sequence means that `connect()` returning `Ok` guarantees a validated config, a live network connection, and a responsive etcd cluster.

The `debug_assert!` catches accidental use from within an existing Tokio runtime — nested `block_on` panics at runtime, and this assertion surfaces the mistake during development. The struct-level documentation makes this constraint explicit: callers must invoke trait methods from synchronous code only.

The `InMemoryCoordinator` is bootstrapped with a 30-second lease duration (`DEFAULT_BOOTSTRAP_LEASE_DURATION`). This default applies to the in-memory delegate and controls how long a shard lease remains valid before requiring renewal.

### Debug Output

The `Debug` implementation omits the etcd client (which does not implement `Debug`) and the Tokio runtime. It shows the endpoint count rather than raw URIs to avoid credential leakage.

From `backend.rs`:

```rust
impl fmt::Debug for EtcdCoordinator {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("EtcdCoordinator")
            .field("endpoint_count", &self.config.endpoints().len())
            .field("namespace_prefix", &self.keyspace.prefix())
            .field(
                "coordination_storage_mode",
                &"delegated to InMemoryCoordinator",
            )
            .finish()
    }
}
```

## Why Delegate to InMemoryCoordinator

Every `CoordinationBackend`, `RunManagement`, and `ShardClaiming` trait method on `EtcdCoordinator` is a direct forwarding call to `self.protocol_delegate`.

From `backend.rs`:

```rust
impl CoordinationBackend for EtcdCoordinator {
    fn acquire_and_restore_into<'a>(
        &mut self,
        now: LogicalTime,
        tenant: TenantId,
        key: ShardKey,
        worker: WorkerId,
        out: &'a mut AcquireScratch,
    ) -> Result<AcquireResultView<'a>, AcquireError> {
        self.protocol_delegate
            .acquire_and_restore_into(now, tenant, key, worker, out)
    }

    // ... all other methods follow the same pattern
}
```

This delegation architecture exists for a specific reason: the `InMemoryCoordinator` is the simulation-tested, TLA+-verified reference implementation. Its shard lifecycle, lease fencing, idempotency dedup, cursor persistence, and split logic have been exhaustively validated through deterministic simulation at multiple fault levels. By delegating to it, the etcd backend inherits all of that correctness without re-implementing any protocol semantics.

This design also means that the etcd backend does not need its own protocol-level test suite. The coordination protocol is tested once in `gossip-coordination`, and every backend that delegates to `InMemoryCoordinator` gets that coverage for free. The etcd-specific tests focus on what the etcd layer adds: configuration validation, keyspace construction, and codec round-trip fidelity.

The etcd layer handles a different set of concerns: connection management, key encoding, binary serialization, and health checking. The keyspace and codec modules are fully built and tested. The remaining work is wiring etcd read/write operations into the trait methods so that each mutation becomes a transactional etcd write and each query reconstructs domain types from stored blobs — making state survive process restarts.

The `ShardClaiming` trait combines `CoordinationBackend` and `RunManagement` (they are its supertraits). Its single method, `claim_next_available`, queries run state for claimable shards and then acquires one — both steps delegated to the in-memory coordinator. Because all three traits are implemented through delegation, a caller holding an `&mut EtcdCoordinator` can use it anywhere a `&mut dyn ShardClaiming` is expected, with identical behavior to the in-memory backend.

From `backend.rs`:

```rust
impl ShardClaiming for EtcdCoordinator {
    fn claim_next_available<'a>(
        &mut self,
        now: LogicalTime,
        tenant: TenantId,
        run: RunId,
        worker: WorkerId,
        out: &'a mut AcquireScratch,
    ) -> Result<AcquireResultView<'a>, ClaimError> {
        self.protocol_delegate
            .claim_next_available(now, tenant, run, worker, out)
    }
}
```

## Error Taxonomy

The `EtcdCoordinatorError` enum captures failures at each stage of the connection and operation lifecycle.

From `error.rs`:

```rust
pub enum EtcdCoordinatorError {
    Config(EtcdCoordinatorConfigError),
    RuntimeBuild(std::io::Error),
    Etcd {
        operation: EtcdOperation,
        source: etcd_client::Error,
    },
    Keyspace(EtcdKeyspaceError),
}
```

The four variants form a progression that mirrors the `connect()` sequence:

1. **`Config`** — validation failed before any I/O. The inner `EtcdCoordinatorConfigError` identifies which constraint was violated.
2. **`RuntimeBuild`** — the Tokio current-thread runtime could not be created. This is a system-resource failure (fd limits, thread limits) that prevents the sync/async bridge from starting.
3. **`Etcd`** — a gRPC call failed. The `operation` field identifies which RPC (`Connect` or `Status`), and `source` carries the upstream `etcd_client::Error` with transport and cluster-level details.
4. **`Keyspace`** — the namespace prefix could not be converted into a valid keyspace builder.

From `error.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum EtcdOperation {
    Connect,
    Status,
}
```

`From` impls enable `?` propagation from config validation and keyspace construction inside `connect()`. This is a standard Rust pattern for error conversion, but it matters here because `connect()` calls both `EtcdCoordinatorConfig::validate()` (which returns `EtcdCoordinatorConfigError`) and `EtcdKeyspace::new()` (which returns `EtcdKeyspaceError`). The `From` impls let both error types convert to `EtcdCoordinatorError` through the `?` operator, keeping the `connect()` implementation clean:

From `error.rs`:

```rust
impl From<EtcdCoordinatorConfigError> for EtcdCoordinatorError {
    fn from(value: EtcdCoordinatorConfigError) -> Self {
        Self::Config(value)
    }
}

impl From<EtcdKeyspaceError> for EtcdCoordinatorError {
    fn from(value: EtcdKeyspaceError) -> Self {
        Self::Keyspace(value)
    }
}
```

The full `Error` trait implementation chains sources properly, so `anyhow` and `eyre` can walk the error chain for diagnostics. Every variant reports its inner error through `source()`, producing stack traces like "invalid etcd coordinator config: etcd endpoint at index 2 has invalid scheme (expected http:// or https://)" that pinpoint the exact failure without requiring a debugger.

## The Public API Surface

The crate's `lib.rs` re-exports a carefully curated set of types. The public API includes `EtcdCoordinator` (the backend), `EtcdCoordinatorConfig` and its error type (configuration), `EtcdKeyspace` and its error type (key construction), the codec's public functions and types (`encode_run_record`, `decode_run_record`, `encode_shard_record`, `decode_shard_record`, `BlobKind`, `EtcdCodecError`), and the coordinator error types (`EtcdCoordinatorError`, `EtcdOperation`). Internal details — the `Decoder` struct, the `OwnedRunRecord` and `OwnedShardRecord` intermediates, the encoding primitives, the `StagedShardAllocations` guard — remain private. This boundary ensures that callers depend on stable semantics (encode, decode, connect, use traits) without coupling to implementation details that may change as the etcd write path is wired up.

## Summary

The `gossip-coordination-etcd` crate builds the complete infrastructure for durable coordination state:

- **`EtcdCoordinatorConfig`** validates endpoints and namespace prefixes with 7 distinct error variants, redacts credentials in `Debug` output, and normalizes whitespace from environment-variable input.
- **`EtcdKeyspace`** maps identity tuples to deterministic ASCII keys in a hierarchical layout designed for prefix scans. Sibling separation (`runs/` vs `runs_active/`) prevents cross-category scan pollution. The `_into` buffer-reuse pattern eliminates per-call allocation on hot paths.
- **The binary codec** uses a 3-byte header (`"v1"` + `BlobKind` tag) followed by little-endian sequential fields. Two-phase decode (parse into owned intermediates, then materialize into domain types) separates wire concerns from domain construction. Slab allocation rollback provides a strong exception guarantee: the slab is unchanged on decode failure.
- **`EtcdCoordinator`** connects to etcd with a 5-second timeout, health-checks via a `Status` RPC, and delegates all protocol semantics to `InMemoryCoordinator` — inheriting the full simulation-tested, TLA+-verified coordination logic.
- **`EtcdCoordinatorError`** covers the full failure progression from config validation through runtime construction to gRPC errors.

## What's Next

The delegation model means the etcd backend has the correct trait surface but does not yet persist state across restarts. The next step is wiring the keyspace and codec into the trait methods: each mutation becomes an etcd transaction that atomically updates record keys and active indexes, and each read reconstructs domain types from stored blobs. When that wiring is complete, the coordinator survives restarts — and the 50,000 shards that were mid-scan resume from their last checkpointed cursors instead of starting over.
