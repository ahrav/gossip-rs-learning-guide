# When Memory Forgets -- The etcd Production Backend Infrastructure

*The cluster has been running for fourteen hours. Six workers coordinate 50,000 shards across 12 tenants, scanning an 11-petabyte corpus. The in-memory coordinator holds the entire truth: which shards are active, which workers hold leases, where each cursor last checkpointed. At 02:17 UTC the coordinator process receives SIGKILL — an OOM event triggered by a co-located batch job. The process restarts in 400 milliseconds. But when it comes back, the `InMemoryCoordinator` is empty. Every `RunRecord` is gone. Every `ShardRecord` is gone. The 50,000 shards that were mid-scan have no cursors, no lease owners, no fence epochs. Workers attempt to renew leases against shard keys that no longer exist and receive `ShardNotFound` errors. The entire scan must be re-discovered from external metadata, re-registered, and re-assigned from the beginning. Fourteen hours of incremental cursor progress — checkpointed but never persisted to disk — evaporates in a single process restart. The in-memory coordinator works perfectly for development and simulation testing. It does not survive production.*

---

The `InMemoryCoordinator` implements every coordination trait with full protocol correctness. It passes the simulation test suite. Its shard lifecycle, lease fencing, and idempotency logic are verified against the TLA+ specification. But it stores all state in process memory, which means a coordinator restart is a total-loss event. In a production deployment, the coordinator must survive restarts, rolling upgrades, and node failures without losing track of which shards are mid-scan, which workers hold leases, or where cursors last checkpointed.

etcd is a natural choice for this persistence layer. It provides linearizable key-value storage with transactional multi-key writes, lease-based TTLs, and prefix-range scans — exactly the primitives needed for coordination state. The `gossip-coordination-etcd` crate builds the bridge between the coordination protocol and a live etcd cluster. Unlike an approach that would delegate protocol logic to `InMemoryCoordinator` and persist state as a side-effect, the `EtcdCoordinator` implements `CoordinationBackend`, `RunManagement`, and `ShardClaiming` directly against etcd. Every mutation is a CAS transaction against etcd keys. Every query reconstructs domain types from stored blobs. State survives process restarts because it never existed only in memory.

The crate is structured around seven core modules: `config` for validated connection parameters and persistence-tuning knobs, `keyspace` for deterministic key construction, `codec` for binary serialization of records and owner-key bindings, `backend` for the coordinator structs and direct trait implementations (with a `backend/` subdirectory for internal sub-modules), `error` for unified failure types, `sim_etcd_kv` for an in-memory model of the etcd KV subset (point reads, prefix scans, CAS transactions, lease semantics) used in deterministic simulation, and `sim_coordinator` for a deterministic coordinator adapter that runs the sync etcd coordination logic against `SimulatedEtcdKV`. The `sim_*` modules are gated behind `cfg(test)` or `feature = "test-support"`. Additional support files include `behavioral_conformance.rs` for protocol conformance testing and `test_support.rs` for shared test infrastructure. The entire crate is marked `#![forbid(unsafe_code)]`. There is no unsafe Rust anywhere in the etcd backend. Every field encoding, bounds check, and allocation rollback operates through safe Rust abstractions.

From `lib.rs`:

```rust
#![forbid(unsafe_code)]

mod backend;
mod codec;
mod config;
mod error;
mod keyspace;
#[cfg(any(test, feature = "test-support"))]
pub mod sim_coordinator;
#[cfg(any(test, feature = "test-support"))]
pub mod sim_etcd_kv;
```

The crate exposes three entrypoints:

- **`EtcdCoordinator`** — sync wrapper that owns a single-threaded Tokio runtime and drives all etcd RPCs via `block_on`. Implements `CoordinationBackend`, `RunManagement`, and `ShardClaiming`. Use this when no async runtime is available.
- **`AsyncEtcdCoordinator`** — async core that implements `AsyncCoordinationBackend` and `AsyncRunManagement`. Use this when an async runtime is already available (e.g., inside a Tokio task).
- **`SimEtcdCoordinator`** *(compiled under `cfg(test)` or `feature = "test-support"`)* — sync coordinator backed by `SimulatedEtcdKV` for deterministic simulation and invariant checking without a live etcd server.

All three share the same coordination validation and transaction shapes, differing only in transport and execution model.

This chapter covers the infrastructure layers — config, keyspace, codec, and error — that form the foundation beneath the live backend. Chapter 14 covers how `EtcdCoordinator` uses these primitives to implement the coordination protocol directly against etcd with CAS transactions, owner key fencing, and lease management.

## Validated Configuration

Before any network I/O occurs, the etcd backend validates its connection parameters through `EtcdCoordinatorConfig`. Configuration validation happens at construction time, not at first use — a pattern that ensures the system fails immediately with a clear error message rather than minutes later when the first etcd RPC fires. The struct captures nine fields: a list of etcd cluster endpoints, a namespace prefix that scopes all coordination keys, a wall-clock TTL for owner leases, a retry budget for optimistic CAS transactions, per-tenant and global shard count ceilings, a maximum children-per-split-op limit, optional authentication credentials, and optional TLS configuration.

From `config.rs`:

```rust
#[derive(Clone)]
pub struct EtcdCoordinatorConfig {
    endpoints: Vec<String>,
    namespace_prefix: String,
    owner_lease_ttl_secs: i64,
    optimistic_txn_retries: usize,
    max_shards_per_tenant: usize,
    max_total_shards: usize,
    max_children_per_op: usize,
    /// Optional username/password pair for etcd authentication.
    auth: Option<(String, String)>,
    /// Optional TLS configuration for encrypted etcd connections.
    #[cfg(feature = "tls")]
    tls: Option<etcd_client::TlsOptions>,
}
```

The first two fields — `endpoints` and `namespace_prefix` — carry connection identity. The next two — `owner_lease_ttl_secs` and `optimistic_txn_retries` — are persistence-tuning knobs that control how the backend interacts with etcd during live operation. `owner_lease_ttl_secs` controls the wall-clock etcd lease attached to shard owner keys: if a worker dies without releasing its lease, the owner key is automatically deleted after this many seconds. `optimistic_txn_retries` bounds the number of read-modify-write retry loops around a fenced etcd transaction before the backend gives up and returns an error. These knobs exist because the backend executes CAS transactions directly against etcd — there is no in-memory delegate that absorbs the retry logic.

The next three fields — `max_shards_per_tenant`, `max_total_shards`, and `max_children_per_op` — are shard-creation guards. `max_shards_per_tenant` caps persisted shard growth per tenant (default: 100,000, matching the in-memory backend). `max_total_shards` caps shards across the entire backend (default: 1,000,000). `max_children_per_op` bounds the fanout of a single atomic `split_replace` publication (default: 8), keeping the total etcd transaction operation count within the cluster's `--max-txn-ops` limit.

Three constants define the defaults:

From `config.rs`:

```rust
pub(crate) const DEFAULT_CONNECT_TIMEOUT: Duration = Duration::from_secs(5);
pub(crate) const DEFAULT_OWNER_LEASE_TTL_SECS: i64 = 60;
pub(crate) const DEFAULT_OPTIMISTIC_TXN_RETRIES: usize = 8;
```

Five seconds is generous for LAN and localhost connections but fast enough to surface misconfigurations during startup. Sixty seconds for the owner lease TTL balances two concerns: long enough that a brief network partition does not falsely expire ownership, short enough that a hard-killed worker releases its shards within a minute. Eight retries for optimistic transactions provides enough headroom for typical contention levels without spinning indefinitely on a hot key.

### Construction Paths

The primary constructor `new()` accepts endpoints and a namespace prefix, applying default tuning knobs. For callers that need explicit control, `new_with_tuning()` accepts all four parameters.

From `config.rs`:

```rust
pub fn new<I, S>(
    endpoints: I,
    namespace_prefix: impl Into<String>,
) -> Result<Self, EtcdCoordinatorConfigError>
where
    I: IntoIterator<Item = S>,
    S: Into<String>,
{
    Self::new_with_tuning(
        endpoints,
        namespace_prefix,
        DEFAULT_OWNER_LEASE_TTL_SECS,
        DEFAULT_OPTIMISTIC_TXN_RETRIES,
    )
}
```

Both constructors trim whitespace from every endpoint and the prefix before running validation. This normalization step matters in practice: endpoints read from environment variables or YAML config files often carry trailing whitespace or newlines that would otherwise cause silent connection failures.

`from_endpoints_csv()` and `from_endpoints_csv_with_tuning()` split a comma-separated string for environment-variable-driven configuration (`ETCD_ENDPOINTS=http://host1:2379,http://host2:2379`). They silently drop segments that are empty after trimming, so trailing commas in environment variables do not cause validation failures.

`localhost()` targets `http://127.0.0.1:2379` with prefix `/gossip/v1` and default tuning knobs for local development. It routes through `new()` so the hard-coded values stay in lockstep with the validation rules — if the rules ever change, the `expect` inside `localhost()` will panic at compile-test time rather than silently producing an invalid config. The `Default` implementation delegates to `localhost()`.

From `config.rs`:

```rust
#[must_use]
pub fn localhost() -> Self {
    Self::new(["http://127.0.0.1:2379"], "/gossip/v1")
        .expect("hard-coded localhost config must remain valid")
}

// ...

impl Default for EtcdCoordinatorConfig {
    fn default() -> Self {
        Self::localhost()
    }
}
```

### Builder Methods for Auth and TLS

Two builder methods layer optional concerns onto a validated config. `with_auth()` sets a username/password pair for etcd authentication. `with_tls()` (gated behind the `"tls"` feature) sets TLS options for encrypted connections. Both return `Self` with the `#[must_use]` attribute, enabling fluent chaining.

From `config.rs`:

```rust
#[must_use]
pub fn with_auth(mut self, user: impl Into<String>, password: impl Into<String>) -> Self {
    self.auth = Some((user.into(), password.into()));
    self
}

#[cfg(feature = "tls")]
#[must_use]
pub fn with_tls(mut self, tls: etcd_client::TlsOptions) -> Self {
    self.tls = Some(tls);
    self
}
```

### Validation Error Variants

The `validate()` method enforces fourteen rules, each with a dedicated error variant. Seven cover endpoint and prefix checks, two cover the persistence-tuning knobs (lease TTL and CAS retries), and five cover the shard-creation guards (retry ceiling, shard count limits, and split fanout bounds).

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
    if self.owner_lease_ttl_secs <= 0 {
        return Err(EtcdCoordinatorConfigError::NonPositiveOwnerLeaseTtl);
    }
    if self.optimistic_txn_retries == 0 {
        return Err(EtcdCoordinatorConfigError::ZeroOptimisticTxnRetries);
    }
    if self.optimistic_txn_retries > MAX_OPTIMISTIC_TXN_RETRIES {
        return Err(EtcdCoordinatorConfigError::ExcessiveOptimisticTxnRetries { .. });
    }
    if self.max_shards_per_tenant == 0 {
        return Err(EtcdCoordinatorConfigError::ZeroMaxShardsPerTenant);
    }
    if self.max_total_shards == 0 {
        return Err(EtcdCoordinatorConfigError::ZeroMaxTotalShards);
    }
    if self.max_children_per_op == 0 {
        return Err(EtcdCoordinatorConfigError::ZeroMaxChildrenPerOp);
    }
    if self.max_children_per_op > MAX_CHILDREN_PER_SPLIT_TXN {
        return Err(EtcdCoordinatorConfigError::MaxChildrenPerOpExceedsEtcdTxnBudget { .. });
    }
    // Also checks against the global MAX_SPLIT_CHILDREN contract limit.

    Ok(())
}
```

The full error enum:

From `config.rs`:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
#[non_exhaustive]
pub enum EtcdCoordinatorConfigError {
    NoEndpoints,
    EmptyEndpoint { index: usize },
    InvalidEndpointScheme { index: usize },
    EmptyNamespacePrefix,
    NamespacePrefixMustStartWithSlash,
    NamespacePrefixMustNotEndWithSlash,
    NamespacePrefixContainsDoubleSlash,
    NonPositiveOwnerLeaseTtl,
    ZeroOptimisticTxnRetries,
    ExcessiveOptimisticTxnRetries { requested: usize, max: usize },
    ZeroMaxShardsPerTenant,
    ZeroMaxTotalShards,
    ZeroMaxChildrenPerOp,
    MaxChildrenPerOpExceedsEtcdTxnBudget { requested: usize, max: usize },
    MaxChildrenPerOpExceedsGlobalLimit { requested: usize, global_max: usize },
}
```

`EmptyEndpoint` and `InvalidEndpointScheme` include the offending index so operators can identify which entry in a multi-endpoint list is wrong. The namespace prefix rules prevent two classes of bugs: a prefix without a leading slash would produce keys that are not absolute etcd paths, and a prefix with a trailing slash would produce double slashes when child segments are appended. `NamespacePrefixContainsDoubleSlash` guards against prefixes like `/gossip//v1` that would create invisible empty path segments in the etcd keyspace. `NonPositiveOwnerLeaseTtl` catches zero or negative values — the TTL is an `i64` because the etcd client API uses signed seconds, but a non-positive TTL is meaningless. `ZeroOptimisticTxnRetries` ensures at least one attempt is made for every CAS transaction. `ExcessiveOptimisticTxnRetries` caps the retry budget to prevent CAS operations from blocking for minutes under contention. The shard-limit variants (`ZeroMaxShardsPerTenant`, `ZeroMaxTotalShards`) ensure at least one shard can be created. `MaxChildrenPerOpExceedsEtcdTxnBudget` prevents the `split_replace` transaction from exceeding etcd's default `--max-txn-ops` limit of 128 (each child adds 3 txn entries to a fixed overhead of 9). `MaxChildrenPerOpExceedsGlobalLimit` ensures the per-op cap does not exceed the protocol-level `MAX_SPLIT_CHILDREN` constant.

### Credential Redaction in Debug Output

Endpoint URIs can contain embedded credentials (e.g., `http://user:pass@etcd-0:2379`). The `Debug` implementation strips the userinfo portion before rendering, so credentials never leak into logs or diagnostic output. Authentication credentials and TLS configuration are also redacted.

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
        let mut s = f.debug_struct("EtcdCoordinatorConfig");
        s.field("endpoints", &redacted)
            .field("namespace_prefix", &self.namespace_prefix)
            .field("owner_lease_ttl_secs", &self.owner_lease_ttl_secs)
            .field("optimistic_txn_retries", &self.optimistic_txn_retries)
            .field("max_shards_per_tenant", &self.max_shards_per_tenant)
            .field("max_total_shards", &self.max_total_shards)
            .field("max_children_per_op", &self.max_children_per_op);
        if let Some((user, _)) = &self.auth {
            s.field("auth_user", user);
            s.field("auth_password", &"[REDACTED]");
        }
        #[cfg(feature = "tls")]
        s.field("tls", &self.tls.as_ref().map(|_| "[configured]"));
        s.finish()
    }
}
```

An endpoint `http://admin:secret@etcd-0:2379` renders as `http://***@etcd-0:2379`. The auth password is always `[REDACTED]`. TLS presence renders as `[configured]` without exposing certificate material. This is security-by-default: a developer who writes `tracing::debug!(?config)` never accidentally exposes credentials.

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

From `keyspace.rs`:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
#[non_exhaustive]
pub enum EtcdKeyspaceError {
    EmptyPrefix,
    PrefixMustStartWithSlash,
    PrefixMustNotEndWithSlash,
    PrefixContainsDoubleSlash,
}
```

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

The `join_namespace_into` helper handles the root prefix (`"/"`) as a special case: joining `"/"` with `"tenants"` produces `"/tenants"`, not `"//tenants"`.

From `keyspace.rs`:

```rust
fn join_namespace_into(&self, buf: &mut String, suffix: &str) {
    debug_assert!(!suffix.starts_with('/'));
    if self.prefix == "/" {
        buf.push('/');
    } else {
        buf.push_str(&self.prefix);
        buf.push('/');
    }
    buf.push_str(suffix);
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
    ShardOwner = 3,
}
```

The header layout is `[version: 2 bytes ("v1")] [kind: 1 byte (BlobKind discriminant)]`. Three blob kinds exist: `RunRecord` for serialized run records, `ShardRecord` for serialized shard records, and `ShardOwner` for the owner-key binding that lives under each shard's `/owner` key in the etcd keyspace. After the header, fields are encoded sequentially in little-endian byte order with no alignment padding. Variable-length fields (byte slices, vectors) are length-prefixed with a `u32` count. Optional fields use a 1-byte bool presence tag (`0` = absent, `1` = present). Enum discriminants are encoded as raw `u8` values matching each type's `as_u8()` representation.

The encoding primitives are symmetric pairs. Each `put_*` function on the encode side has a matching `read_*` method on the `Decoder`. For example, `put_u64` writes 8 little-endian bytes, and `read_u64` reads 8 bytes and reconstructs the `u64`. `put_bytes` writes a `u32` length followed by the raw bytes, and `read_vec` reads the length, bounds-checks it against `MAX_FIELD_SIZE` (1 MiB), then reads that many bytes. `put_opt_u64` writes a 1-byte presence tag followed by the value if present, and `read_opt_u64` reads the tag and conditionally reads the value. This symmetry makes the codec self-documenting: reading the encode function reveals the exact wire layout.

### Owner Key Codec

The `ShardOwner` blob kind is specific to the shard ownership mechanism. Each shard's `/owner` key in the etcd keyspace stores a binding of worker identity to fence epoch. The `OwnerLeaseValue` struct captures this pair:

From `codec.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct OwnerLeaseValue {
    pub worker: WorkerId,
    pub fence: FenceEpoch,
}
```

The encode and decode functions for owner values are compact — just the 3-byte header followed by two `u64` fields:

From `codec.rs`:

```rust
pub fn encode_owner_value(worker: WorkerId, fence: FenceEpoch) -> Vec<u8> {
    let mut out = Vec::with_capacity(3 + 16);
    out.extend_from_slice(VERSION_PREFIX_V1);
    out.push(BlobKind::ShardOwner.as_u8());
    put_u64(&mut out, worker.as_raw());
    put_u64(&mut out, fence.as_raw());
    out
}

pub fn decode_owner_value(bytes: &[u8]) -> Result<OwnerLeaseValue, EtcdCodecError> {
    let mut decoder = Decoder::new(bytes);
    decoder.expect_header(BlobKind::ShardOwner)?;
    let worker = WorkerId::from_raw(decoder.read_u64()?);
    let fence = FenceEpoch::from_raw(decoder.read_u64()?);
    if fence < FenceEpoch::INITIAL {
        return Err(EtcdCodecError::InvariantViolation {
            kind: "OwnerLeaseValue",
            detail: "fence must be >= INITIAL",
        });
    }
    decoder.finish()?;
    Ok(OwnerLeaseValue { worker, fence })
}
```

The decode path validates that the fence epoch is at least `FenceEpoch::INITIAL` — a sub-initial fence would indicate a corrupted or fabricated owner key. The `finish()` call rejects any trailing bytes, guarding against appended garbage. This owner value is the payload that the backend's CAS transactions compare and swap when claiming or fencing shard ownership. Chapter 14 covers how these transactions work in practice.

### Run Record and Shard Record Encoding

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

Shard encoding reads pooled fields from the caller's `ByteSlab` to extract byte content. The slab is borrowed immutably — no allocations occur during encoding. Unlike `encode_run_record`, shard encoding is fallible: it returns `Result` because `OwnedShardRecord::encode_into` validates the lease invariant (owner and deadline must be jointly present or absent) and returns `EtcdCodecError::InvariantViolation` on mismatch.

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

Decoding proceeds in two distinct phases. First, the blob is parsed into heap-owned intermediates (`OwnedRunRecord`, `OwnedShardRecord`) that hold all decoded values in plain `Vec<u8>` and `Vec<ShardId>` form. Structural validation happens immediately during this phase. Second, materialization converts the owned intermediates into domain types. For `RunRecord` this is a relatively cheap operation: it constructs a `RunConfig` from flat fields and populates a `RingBuffer` op log. For `ShardRecord` it allocates pooled fields (`PooledShardSpec`, `PooledCursor`, `PooledSpawned`) into the caller's `ByteSlab`. This second phase is where the slab-backed allocation model creates complexity — and where the strong exception guarantee becomes essential.

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
#[derive(Debug, Clone, PartialEq, Eq)]
#[non_exhaustive]
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

## Error Taxonomy

The `error` module defines two types that cover all infrastructure failure modes. `EtcdOperation` labels which RPC or codec stage failed. `EtcdCoordinatorError` captures the top-level error with its source chain.

### Operation Discriminant

From `error.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
#[non_exhaustive]
pub enum EtcdOperation {
    Connect,
    Status,
    Get,
    Put,
    Delete,
    Txn,
    LeaseGrant,
    LeaseKeepAlive,
    LeaseRevoke,
}
```

Nine variants cover the full surface of etcd interactions. `Connect` and `Status` are used during startup. `Get`, `Put`, `Delete`, and `Txn` cover the key-value operations that the backend executes during coordination. `LeaseGrant`, `LeaseKeepAlive`, and `LeaseRevoke` cover the lease lifecycle that backs shard owner key TTLs. Each variant has a display string that appears in error messages (e.g., "etcd lease_grant operation failed: ..."), providing structured context for diagnostics without embedding the full error chain in the variant name.

### Top-Level Coordinator Errors

From `error.rs`:

```rust
#[derive(Debug)]
#[non_exhaustive]
pub enum EtcdCoordinatorError {
    Config(EtcdCoordinatorConfigError),
    Keyspace(EtcdKeyspaceError),
    RuntimeBuild(std::io::Error),
    Codec {
        operation: EtcdOperation,
        source: EtcdCodecError,
    },
    Etcd {
        operation: EtcdOperation,
        source: etcd_client::Error,
    },
}
```

Five variants form a progression that mirrors the backend's lifecycle:

1. **`Config`** — validation failed before any I/O. The inner `EtcdCoordinatorConfigError` identifies which of the nine constraints was violated.
2. **`Keyspace`** — the namespace prefix could not be converted into a valid keyspace builder.
3. **`RuntimeBuild`** — the Tokio current-thread runtime could not be created. This is a system-resource failure (fd limits, thread limits) that prevents the sync/async bridge from starting.
4. **`Codec`** — a v1 blob failed to encode or decode during the given operation. This variant pairs the `EtcdOperation` that triggered the codec call with the `EtcdCodecError` source, preserving both "what were we doing" and "what went wrong with the bytes" in a single error.
5. **`Etcd`** — a gRPC call failed. The `operation` field identifies which RPC (any of the nine `EtcdOperation` variants), and `source` carries the upstream `etcd_client::Error` with transport and cluster-level details.

The `Codec` variant is a notable addition: because the backend implements coordination directly against etcd, codec failures can occur during any operation that reads or writes record blobs. The operation tag identifies whether the codec error happened during a `Get` (decode failure on a read), a `Put` (encode failure before a write), or a `Txn` (decode or encode failure during a compare-and-swap).

`From` impls enable `?` propagation from config validation and keyspace construction:

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

The full `Error` trait implementation chains sources properly, so `anyhow` and `eyre` can walk the error chain for diagnostics. Every variant reports its inner error through `source()`, producing stack traces like "etcd lease_grant operation failed: connection refused" that pinpoint the exact failure without requiring a debugger.

## The Public API Surface

The crate's `lib.rs` re-exports a carefully curated set of types. The public API includes `EtcdCoordinator` (the backend), `EtcdCoordinatorConfig` and its error type (configuration), `EtcdKeyspace` and its error type (key construction), the codec's public functions and types (`encode_run_record`, `decode_run_record`, `encode_shard_record`, `decode_shard_record`, `encode_owner_value`, `decode_owner_value`, `BlobKind`, `OwnerLeaseValue`, `EtcdCodecError`), and the coordinator error types (`EtcdCoordinatorError`, `EtcdOperation`).

From `lib.rs`:

```rust
pub use backend::EtcdCoordinator;
pub use codec::{
    BlobKind, EtcdCodecError, OwnerLeaseValue, decode_owner_value, decode_run_record,
    decode_shard_record, encode_owner_value, encode_run_record, encode_shard_record,
};
pub use config::{EtcdCoordinatorConfig, EtcdCoordinatorConfigError};
pub use error::{EtcdCoordinatorError, EtcdOperation};
pub use keyspace::{EtcdKeyspace, EtcdKeyspaceError};
```

Internal details — the `Decoder` struct, the `OwnedRunRecord` and `OwnedShardRecord` intermediates, the encoding primitives, the `StagedShardAllocations` guard — remain private. This boundary ensures that callers depend on stable semantics (encode, decode, connect, use traits) without coupling to implementation details.

## Summary

The `gossip-coordination-etcd` crate builds the complete infrastructure for durable coordination state:

- **`EtcdCoordinatorConfig`** validates endpoints, namespace prefixes, owner lease TTL, and optimistic retry budget with 9 distinct error variants. It redacts credentials in `Debug` output, normalizes whitespace from environment-variable input, and provides builder methods for auth and TLS.
- **`EtcdKeyspace`** maps identity tuples to deterministic ASCII keys in a hierarchical layout designed for prefix scans. Sibling separation (`runs/` vs `runs_active/`, `shards/` vs `shards_active/`) prevents cross-category scan pollution. The `_into` buffer-reuse pattern eliminates per-call allocation on hot paths.
- **The binary codec** uses a 3-byte header (`"v1"` + `BlobKind` tag) followed by little-endian sequential fields. Three blob kinds cover run records, shard records, and owner-key bindings. Two-phase decode (parse into owned intermediates, then materialize into domain types) separates wire concerns from domain construction. Slab allocation rollback provides a strong exception guarantee: the slab is unchanged on decode failure. The `OwnerLeaseValue` codec gives the backend a compact, validated representation of shard ownership for CAS transactions.
- **`EtcdCoordinatorError`** covers the full failure progression from config validation through runtime construction to codec failures and gRPC errors, with 9 `EtcdOperation` variants labeling every possible failure site.

## What's Next

The infrastructure layers are in place: validated configuration with tuning knobs for CAS retries and lease TTLs, a deterministic keyspace that maps every coordination entity to an etcd key, a versioned binary codec that serializes records and owner-key bindings with round-trip fidelity, and a structured error taxonomy covering every failure mode. Chapter 14 covers how `EtcdCoordinator` uses these primitives to implement the coordination protocol directly against etcd — CAS transactions for fenced mutations, owner key fencing with `OwnerLeaseValue` payloads, lease-backed TTLs for automatic ownership expiry, and optimistic retry loops bounded by the configured retry budget. The backend does not delegate to `InMemoryCoordinator`. Every coordination operation reads from etcd, mutates through transactions, and writes back to etcd. When a coordinator process restarts, the 50,000 shards that were mid-scan resume from their last checkpointed cursors instead of starting over.
