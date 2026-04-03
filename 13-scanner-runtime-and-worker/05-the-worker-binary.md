# The Worker Binary -- Environment Configuration and Production Backends

*An operator deploys `gossip-worker` to a fleet of 200 machines. The first deployment attempt fails on 12 machines. The error message: `invalid worker configuration: done-ledger PostgreSQL DSN must not be empty`. Three seconds to diagnose: the `GOSSIP_DONE_LEDGER_POSTGRES_DSN` environment variable was not set in the deployment manifest for those 12 machines. A second operator on a different team encounters: `invalid worker configuration: run_id: must be > 0 (0 is the unassigned sentinel)`. Five seconds to diagnose: the orchestrator passed `GOSSIP_RUN_ID=0` instead of a real run ID. A third operator sees: `worker startup failed: failed to connect etcd coordination backend`. The error message is intentionally opaque about the endpoint -- connection errors suppress the inner source to prevent DSN fragments (hostname, port, username) from leaking into log aggregation systems. A binary entrypoint has one job: translate operator intent (environment variables) into validated runtime calls, and translate runtime results (including errors) into human-readable output with actionable context.*

---

The `gossip-worker` crate is a binary entrypoint backed by a library that exposes the typed configuration surface and production composition root. The binary itself does three things: initializes tracing, resolves configuration from environment variables and CLI arguments, and delegates to the runtime. It contains no scanning logic, no engine construction, no event formatting, no identity derivation, no coordination protocol. Every substantive operation is performed by `gossip-scanner-runtime`. The worker is the outermost shell; the runtime is the reusable library.

## 1. The Architecture

The crate has three modules:

```rust
pub mod config;      // Typed configuration surface
pub mod production;  // Production composition root
pub mod recorder;    // Production telemetry recorder
```

The dependency chain is intentionally narrow:

```text
gossip-worker (binary)
    |
    +-- gossip-scanner-runtime   (scan dispatch, distributed loop)
    +-- gossip-coordination-etcd (EtcdCoordinator)
    +-- gossip-done-ledger-postgres (DoneLedgerPg)
    +-- gossip-findings-postgres (FindingsSinkPg)
    +-- tracing-subscriber       (structured logging)
```

The worker depends on the runtime for scanning and on the concrete backend crates for production infrastructure. The runtime itself is generic over `CoordinationFacade`, `FindingsSink`, and `DoneLedger`; the worker is where the generic parameters are resolved to real types.

## 2. Environment Variable Configuration

The worker configuration is resolved entirely from environment variables plus optional CLI argument overrides. From `config.rs`:

```rust
pub const ENV_WORKER_MODE: &str = "GOSSIP_WORKER_MODE";
pub const ENV_WORKER_BACKEND: &str = "GOSSIP_WORKER_BACKEND";
pub const ENV_WORKER_SOURCE: &str = "GOSSIP_WORKER_SOURCE";
pub const ENV_WORKER_PATH: &str = "GOSSIP_WORKER_PATH";
pub const ENV_ETCD_ENDPOINTS: &str = "GOSSIP_ETCD_ENDPOINTS";
pub const ENV_ETCD_NAMESPACE: &str = "GOSSIP_ETCD_NAMESPACE";
pub const ENV_DONE_LEDGER_POSTGRES_DSN: &str = "GOSSIP_DONE_LEDGER_POSTGRES_DSN";
pub const ENV_FINDINGS_POSTGRES_DSN: &str = "GOSSIP_FINDINGS_POSTGRES_DSN";
pub const ENV_TENANT_ID: &str = "GOSSIP_TENANT_ID";
pub const ENV_RUN_ID: &str = "GOSSIP_RUN_ID";
pub const ENV_WORKER_ID: &str = "GOSSIP_WORKER_ID";
pub const ENV_POLICY_HASH: &str = "GOSSIP_POLICY_HASH";
pub const ENV_TENANT_SECRET_KEY: &str = "GOSSIP_TENANT_SECRET_KEY";
pub const ENV_STARTUP_SCHEMA_MODE: &str = "GOSSIP_STARTUP_SCHEMA_MODE";
pub const ENV_MAX_ITEMS: &str = "GOSSIP_MAX_ITEMS";
pub const ENV_MAX_BYTES: &str = "GOSSIP_MAX_BYTES";
pub const ENV_COMMIT_QUEUE_CAPACITY: &str = "GOSSIP_COMMIT_QUEUE_CAPACITY";
pub const ENV_WORKER_RULES_FILE: &str = "GOSSIP_WORKER_RULES_FILE";
pub const ENV_WORKER_DECODE_DEPTH: &str = "GOSSIP_WORKER_DECODE_DEPTH";
pub const ENV_WORKER_SCAN_BINARY: &str = "GOSSIP_WORKER_SCAN_BINARY";
pub const ENV_FS_SKIP_ARCHIVES: &str = "GOSSIP_FS_SKIP_ARCHIVES";
pub const ENV_WORKER_ANCHOR_MODE: &str = "GOSSIP_WORKER_ANCHOR_MODE";
```

The `resolve_worker_config_from_env_and_args` function parses these variables (plus optional CLI overrides like the positional source and path arguments) into a typed `ResolvedWorkerConfig`.

## 3. The Two Launch Paths

The resolved configuration is one of two variants:

```rust
pub enum ResolvedWorkerConfig {
    Local(LocalWorkerConfig),
    Distributed(Box<DistributedWorkerConfig>),
}
```

**`LocalWorkerConfig`** -- a direct scan with no coordinator. The execution mode is structurally fixed to `ExecutionMode::Direct`. The source can be filesystem or git:

```rust
pub struct LocalWorkerConfig {
    source: WorkerSourceSettings,
    budgets: ScanBudgets,
}

pub enum WorkerSourceSettings {
    Fs(FsSourceSettings),
    Git(GitSourceSettings),
}
```

**`DistributedWorkerConfig`** -- a production distributed scan with etcd coordination and PostgreSQL persistence:

```rust
pub struct DistributedWorkerConfig {
    backends: ProductionBackendConfig,
    startup: ProductionStartupSettings,
    identity: WorkerIdentityConfig,
    source: FsSourceSettings,
    runtime: DistributedWorkerRuntimeSettings,
}
```

The `DistributedWorkerConfig` is type-constrained to `FsSourceSettings` -- distributed mode currently supports only filesystem scans. The backend selection is structurally fixed to `BackendSelection::Production`. The `startup` field controls startup-time schema readiness validation: `StartupSchemaMode::Validate` (the default) requires both PostgreSQL schemas to already exist and match the embedded migration history; `StartupSchemaMode::DevAutoMigrate` applies embedded migrations before validation, intended for local development workflows.

### 3.1 BackendSelection

The backend selection enum has two variants:

```rust
pub enum BackendSelection {
    Local,
    Production,
}
```

`Local` maps to `LocalWorkerConfig` (direct scans with no external backends). `Production` maps to `DistributedWorkerConfig` (etcd + PostgreSQL). There is no "fake" or "test" backend selection -- the binary always uses real backends or no backends.

### 3.2 WorkerIdentityConfig

Distributed mode requires a full worker identity:

```rust
pub struct WorkerIdentityConfig {
    tenant: TenantId,
    run: RunId,
    worker: WorkerId,
    policy_hash: PolicyHash,
    tenant_secret_key: TenantSecretKey,
}
```

Construction validates: `run` and `worker` must not be the zero sentinel, and `tenant_secret_key` must not be all zeros. The `Debug` implementation redacts `tenant_secret_key` to prevent leakage into logs.

### 3.3 ProductionBackendConfig

The backend configuration bundles etcd and PostgreSQL connection parameters:

```rust
pub struct ProductionBackendConfig {
    etcd: EtcdCoordinatorConfig,
    done_ledger_postgres_dsn: String,
    findings_postgres_dsn: String,
}
```

DSNs are trimmed before storage so accidental whitespace from environment variables does not survive into the startup path. Empty DSNs after trimming are rejected with `ProductionBackendConfigError::EmptyDoneLedgerPostgresDsn` or `EmptyFindingsPostgresDsn`.

The `Debug` implementation redacts both DSNs:

```rust
impl fmt::Debug for ProductionBackendConfig {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("ProductionBackendConfig")
            .field("etcd", &self.etcd)
            .field("done_ledger_postgres_dsn", &"[redacted]")
            .field("findings_postgres_dsn", &"[redacted]")
            .finish()
    }
}
```

## 4. Budget Validation and Ceilings

Distributed mode validates scan budgets against both zero-checks and ceiling checks:

```rust
const MAX_ITEMS_CEILING: usize = 10_000_000;
const MAX_BYTES_CEILING: u64 = 64 * 1024 * 1024 * 1024; // 64 GiB
const MAX_COMMIT_QUEUE_CEILING: usize = 65_536;
```

These ceilings prevent resource exhaustion from misconfiguration. Passing `GOSSIP_MAX_ITEMS=999999999` would allocate enormous checkpoint structures; the ceiling rejects it before any work begins.

## 5. The Production Composition Root

The `production.rs` module is the thin outer layer that binds the generic distributed runtime to real infrastructure backends:

```rust
pub struct ProductionRuntimeBackends {
    coordinator: EtcdCoordinator,
    persistence: DistributedPersistence<FindingsSinkPg, DoneLedgerPg>,
}
```

The module provides two constructors:

**`build_production_backends`** -- builds backends from a `ProductionBackendConfig` and `ProductionStartupSettings`, creating PostgreSQL connections from DSN strings using `postgres::NoTls`:

```rust
pub fn build_production_backends(
    config: &ProductionBackendConfig,
    startup: ProductionStartupSettings,
) -> Result<ProductionRuntimeBackends, ProductionBootstrapError>
```

**`build_production_backends_from_clients`** -- builds backends from pre-connected PostgreSQL clients, allowing callers that need TLS or custom connection handling to supply their own:

The bootstrap process:
1. Connect both PostgreSQL backends in parallel (overlapping the two ~5s connection timeouts).
2. Connect the etcd coordinator.
3. Apply the selected startup schema policy: `Validate` requires existing schemas and migration history; `DevAutoMigrate` runs embedded migrations first, then validates.
4. Return the bundled backends.

Typed startup failures distinguish which step failed:

```rust
pub enum ProductionBootstrapError {
    DoneLedgerConnect(postgres::Error),
    FindingsConnect(postgres::Error),
    EtcdConnect(EtcdCoordinatorError),
    DoneLedgerSchemaReadiness(ProductionSchemaReadinessError),
    FindingsSchemaReadiness(ProductionSchemaReadinessError),
    DoneLedgerAutoMigrate(DoneLedgerPgMigrationError),
    FindingsAutoMigrate(FindingsPgMigrationError),
    ThreadPanicked { backend: &'static str },
}
```

The `DoneLedgerSchemaReadiness` and `FindingsSchemaReadiness` variants wrap a `ProductionSchemaReadinessError` that classifies the specific readiness failure (missing table, missing migration history row, corrupted checksum, checksum mismatch). The `DoneLedgerAutoMigrate` and `FindingsAutoMigrate` variants fire only under `DevAutoMigrate` mode when migration application itself fails. The `ThreadPanicked` variant catches connection threads that panic instead of returning a result.

Connection-error variants intentionally suppress the inner `postgres::Error` from `Display` and `Error::source()` to prevent DSN fragments (hostname, port, username) from leaking through error chain formatters (anyhow, eyre, tracing-error).

## 6. The run_production_worker Function

The convenience entry point for the binary:

```rust
pub fn run_production_worker(
    config: &ProductionBackendConfig,
    startup: ProductionStartupSettings,
    identity: WorkerIdentity,
    runtime: DistributedRuntimeConfig,
) -> Result<DistributedRunReport, ProductionWorkerError>
```

This function builds the backends with the supplied startup policy, then calls `run_worker` with the concrete `EtcdCoordinator`, `FindingsSinkPg`, and `DoneLedgerPg` types. The `ProductionWorkerError` distinguishes startup failures from runtime failures:

```rust
pub enum ProductionWorkerError {
    Startup(ProductionBootstrapError),
    Runtime(DistributedRuntimeError),
}
```

## 7. The Production Recorder

The `recorder.rs` module provides `ProductionCoordinationEventRecorder`, which implements the `CoordinationEventRecorder` trait with structured `tracing` events and a strict safe-field policy:

- Raw item keys, paths, git identity bytes, and diagnostic text are never emitted directly.
- Toxic fields are reduced to `len + short hash` digests using a BLAKE3 derive-key hasher (`COORDINATION_TELEMETRY_V1` domain).
- The hasher is cached per-thread via `thread_local!` with `reset()` between calls, avoiding the ~1.9 KiB memcpy of cloning the full hasher state.
- Recorder sink failures are non-fatal and logged once per category (core / git / progress).

## 8. The main() Function

The binary entrypoint is straightforward:

```rust
fn main() {
    init_tracing();

    let resolved = match resolve_worker_config_from_env_and_args(std::env::args().skip(1)) {
        Ok(cfg) => cfg,
        Err(error) => {
            tracing::error!(error = %error, "invalid worker configuration");
            std::process::exit(2);
        }
    };

    match execute_resolved_worker_with(&resolved, run_distributed_worker) {
        Ok(report) => log_worker_report(&resolved, report),
        Err(error) => {
            tracing::error!(error = %error, "worker scan failed");
            std::process::exit(1);
        }
    }
}
```

Three steps: initialize tracing, resolve configuration, dispatch. Exit code 2 for configuration errors, exit code 1 for runtime errors, exit code 0 for success. The `execute_resolved_worker_with` function takes a closure for the distributed runner, allowing tests to inject an in-memory coordinator and persistence bundle while the binary uses the real etcd/PostgreSQL path.

The dispatch function:

```rust
fn execute_resolved_worker_with<F>(
    resolved: &ResolvedWorkerConfig,
    distributed_runner: F,
) -> Result<WorkerRunReport, WorkerError>
where
    F: FnOnce(&DistributedWorkerConfig) -> Result<DistributedRunReport, ProductionWorkerError>,
{
    match resolved {
        ResolvedWorkerConfig::Local(cfg) => run_local_worker(cfg).map(WorkerRunReport::Local),
        ResolvedWorkerConfig::Distributed(cfg) => distributed_runner(cfg)
            .map(WorkerRunReport::Distributed)
            .map_err(WorkerError::from),
    }
}
```

Local mode calls `scan_fs` or `scan_git` directly. Distributed mode calls the injected runner. Both paths produce a `WorkerRunReport` that the binary logs with structured tracing fields (backend, source, mode, path, item counts).

## 9. Tracing Initialization

The tracing setup uses `tracing-subscriber` with `EnvFilter`:

```rust
fn init_tracing() {
    let filter = match EnvFilter::builder()
        .with_default_directive(LevelFilter::INFO.into())
        .from_env()
    {
        Ok(f) => f,
        Err(err) => {
            eprintln!("WARNING: malformed RUST_LOG directive ({err}); falling back to default");
            EnvFilter::builder()
                .with_default_directive(LevelFilter::INFO.into())
                .from_env_lossy()
        }
    };
    tracing_subscriber::fmt()
        .with_writer(std::io::stderr)
        .with_env_filter(filter)
        .with_target(false)
        .compact()
        .init();
}
```

`RUST_LOG` is honored if present. The default is `info`. If the directive is malformed, a warning is printed to stderr and the filter falls back to lossy parsing so the process still starts. The `.with_writer(std::io::stderr)` sends tracing output to stderr, keeping stdout clean for structured finding output. The `.with_target(false)` suppresses the module path from log lines (redundant when the binary is the only process). The `.compact()` format produces single-line log entries suitable for structured log aggregation.
