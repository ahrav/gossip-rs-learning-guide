# "The Heterogeneous Zoo" -- Connector Problem Space

*An S3 connector enumerates objects in a production bucket. Each `ScanItem`
carries an `ItemRef` that encodes the object's ARN and access key ID -- the
connector needs both to construct a pre-signed URL for the read path. During
development, someone auto-derives `Debug` on the struct wrapping the raw
bytes. A log line fires at `tracing::debug!("processing item: {:?}", scan_item)`.
The `ItemRef` formats as `ItemRef([65, 75, 49, 41, ...])`
-- the raw bytes of `AKIAIOSFODNN7EXAMPLE`, a live AWS access key. The structured
log record ships to Datadog. The Datadog index is searchable by the security
team, the SRE on-call group, and a third-party audit integration. The secret
key -- the very thing the scanner exists to find -- now sits in plaintext in
three secondary systems that the scanner was supposed to protect. No alert fires.
No error is raised. The log level was `debug`, not `error`, so nobody reviews
it. The key persists in the log aggregator for 90 days, indexed and
full-text-searchable.*

---

## Why Connectors Exist

The failure above is a formatting accident, but the conditions that make it
possible are structural. A secret scanner must connect to heterogeneous data
sources: GitHub repositories, S3 buckets, local filesystems, SaaS APIs with
OAuth flows, databases with cursor-based pagination. Each source has different
authentication, different pagination semantics (page tokens vs. key-seek vs.
offset), different rate-limit behavior (HTTP 429 with `Retry-After` headers
vs. TCP backpressure vs. no signaling at all), and different failure modes
(transient network errors vs. permanent 403s vs. gone-forever 404s).

Without a uniform interface, every downstream component -- the coordinator,
the shard planner, the scan loop, the persistence layer -- must understand
every connector's quirks. The code becomes a matrix of special cases:
"if S3, use token pagination; if GitHub, use cursor pagination; if filesystem,
scan until EOF." Each new connector multiplies the integration surface.

The `gossip-contracts` crate solves this with a connector contract layer that
lives in `crates/gossip-contracts/src/connector/`. This module defines the
types, traits, and validation rules that every connector must satisfy. Runtime
connector implementations live elsewhere. The contracts crate owns the
interface; it does not own the implementations.

Recall from Chapter 5 (Boundary 1) that `StableItemId` and `ConnectorTag`
provide domain-separated identity. Those identity types flow through the
connector boundary -- connectors produce `StableItemId` values that
downstream components consume without knowing whether the ID came from S3 or
GitHub. The connector contract layer builds on that foundation by defining
what connectors produce (enumeration pages, item references, cursors) and
what they consume (shard specs, budgets, resume cursors).

---

## The Two-Trait Design

The connector contract splits runtime behavior across two traits:
`EnumerationConnector` for listing operations and `ReadConnector` for content
access. This separation is not accidental.

Enumeration and reading have fundamentally different resource profiles.
Enumeration is metadata-bound: it lists object keys, file paths, or manifest
rows. The bottleneck is API call count, not bandwidth. A single enumeration
page from S3 `ListObjectsV2` returns up to 1,000 keys in a few kilobytes of
JSON. Reading is bandwidth-bound: it fetches full object content. A single
S3 `GetObject` call can stream gigabytes. The two operations hit different
rate-limit ceilings, different cost tiers (S3 charges differently for LIST
vs. GET), and different failure modes (a rate-limited LIST returns HTTP 503;
a missing object returns HTTP 404).

By splitting the traits, orchestration can compose them independently. A
connector might enumerate items on one thread and read items on another. A
circuit breaker can trip on the read path (too many 5xx errors fetching
content) without halting enumeration. A test harness can mock enumeration
while using a real read path, or vice versa.

Here is the definition from `api.rs`:

```rust
/// Runtime contract for connector enumeration/listing operations.
///
/// This trait is intentionally separate from [`ReadConnector`]: some backends
/// can list metadata efficiently but have different cost/failure behavior for
/// content reads. Keeping the surfaces split lets orchestration compose them
/// independently, while [`ConnectorInstance`] provides a shorthand bound when
/// both are required.
pub trait EnumerationConnector: Send {
    /// Advertise connector capabilities used by orchestration planning.
    ///
    /// This is a declarative, static profile for the configured
    /// connector instance. It should align with method behavior, but callers
    /// must still handle runtime `unsupported`/policy errors.
    fn caps(&self) -> ConnectorCapabilities;

    /// Enumerate one page of items for `shard` from `cursor` under `budgets`.
    ///
    /// Returns scan items plus the continuation cursor for the next request
    /// (see [`EnumerationPage`]). Empty pages are valid and carry meaning
    /// through the returned cursor state.
    ///
    /// # Budget enforcement
    ///
    /// [`Budgets`] values are **advisory at this trait layer**. Connector
    /// implementations **should** honor [`Budgets::max_items`] as an upper
    /// bound on the returned page length, but callers **must not** assume
    /// compliance. The runtime orchestration layer is responsible for
    /// enforcement: it **may** truncate excess items, apply backpressure,
    /// or terminate a misbehaving connector.
    fn enumerate_page(
        &mut self,
        shard: &ShardSpec,
        cursor: &Cursor,
        budgets: Budgets,
    ) -> Result<EnumerationPage, EnumerateError>;

    /// Optional split-point hint for dynamic shard subdivision.
    ///
    /// Default behavior is `Ok(None)`, indicating "no hint available". Keep
    /// this default unless the connector can provide useful, low-risk split
    /// candidates. Returned hints are advisory and may be ignored by callers.
    /// Runtime callers **should** validate that returned keys fall within
    /// the shard's key range before acting on them.
    ///
    /// # Capability consistency
    ///
    /// The default implementation includes a `debug_assert!` that fires when
    /// [`caps().split_hints`](ConnectorCapabilities::split_hints) is `true`
    /// but this method has not been overridden. This catches a misconfiguration
    /// where the connector advertises split-hint support without providing an
    /// implementation. Connectors that override this method **must** also
    /// return `split_hints: true` from [`caps`](EnumerationConnector::caps).
    fn choose_split_point(
        &mut self,
        _shard: &ShardSpec,
        _cursor: &Cursor,
        _budgets: Budgets,
    ) -> Result<Option<ItemKey>, EnumerateError> {
        debug_assert!(
            !self.caps().split_hints,
            "connector advertises split_hints but does not override choose_split_point"
        );
        Ok(None)
    }
}
```

Three methods define the enumeration surface:

- **`caps(&self) -> ConnectorCapabilities`** -- A static declaration of what
  the connector supports. Orchestration queries this at registration time to
  plan its enumeration strategy. The capabilities are declarative intent, not
  runtime guarantees -- callers must still handle errors from operations that
  the connector claims to support.

- **`enumerate_page(&mut self, ...) -> Result<EnumerationPage, EnumerateError>`** --
  The core listing operation. It takes a `ShardSpec` (the key range to
  enumerate within), a `Cursor` (where to resume), and `Budgets` (stop
  conditions). The `ShardSpec` from Boundary 3 defines the key range each
  connector enumerates within. Budget enforcement is explicitly advisory at
  this layer: the runtime wraps connectors in enforcement adapters rather than
  trusting implementations.

- **`choose_split_point(&mut self, ...) -> Result<Option<ItemKey>, EnumerateError>`** --
  An optional hook for connectors that know where natural partition boundaries
  exist (Git tree boundaries, S3 common prefixes). The default returns
  `Ok(None)`. The `debug_assert!` catches a specific misconfiguration: a
  connector that advertises `split_hints: true` in `caps()` but forgets to
  override this method.

Here is the definition from `api.rs`:

```rust
/// Runtime contract for connector item reads.
///
/// This surface is split from [`EnumerationConnector`] so orchestration can
/// reason separately about metadata traversal and payload IO costs.
pub trait ReadConnector: Send {
    /// Open an item for sequential read access.
    ///
    /// # Return type: `Box<dyn Read + Send>`
    ///
    /// The boxed trait object is intentional: it preserves object safety so
    /// callers can work with `dyn ReadConnector` at runtime (connector
    /// polymorphism). The heap allocation is acceptable because `open` sits
    /// on the **WARM** read path — called once per item, not per byte —
    /// and the IO cost of the subsequent read dominates.
    ///
    /// The returned reader must be self-contained (`'static`): implementations
    /// cannot borrow from `&self` or other local references. Readers must own
    /// their resources (e.g., an open file handle, a cloned HTTP client, or an
    /// `Arc`-backed buffer).
    ///
    /// # Budget enforcement
    ///
    /// [`Budgets`] values are **advisory at this trait layer**. Connector
    /// implementations **should** respect budget hints (e.g., for chunked
    /// fetching or deadline-aware reads), but callers **must not** assume
    /// compliance. The runtime orchestration layer is responsible for
    /// enforcement: it **must** wrap the returned reader in a size-bounded,
    /// deadline-aware adapter before passing it to downstream consumers.
    ///
    /// Callers may drop the reader at any point; implementations must ensure
    /// resource cleanup via [`Drop`], not via read-to-completion.
    ///
    /// # Item reference affinity
    ///
    /// `item_ref` **must** originate from an [`EnumerationPage`] produced by
    /// this connector instance (or a compatible instance backed by the same
    /// data source). Passing an [`ItemRef`] obtained from a different
    /// connector type is a caller error and may produce garbage reads or
    /// opaque failures.
    fn open(
        &mut self,
        item_ref: &ItemRef,
        budgets: Budgets,
    ) -> Result<Box<dyn io::Read + Send>, ReadError>;

    /// Optional range-read fast path.
    ///
    /// Writes up to `dst.len()` bytes from `item_ref` starting at `offset`
    /// into `dst`, returning the number of bytes written. Implementations
    /// **must** return a value `<= dst.len()`. Runtime callers **should**
    /// defensively clamp before using the value as a slice bound.
    ///
    /// # Overflow safety
    ///
    /// Implementors **must** guard against `offset + dst.len()` overflow
    /// before performing arithmetic on the combined value. Use
    /// `offset.checked_add(dst.len() as u64)` and return an appropriate
    /// [`ReadError`] if the addition wraps.
    ///
    /// **EOF behavior:** when `offset` is at or past the end of the item,
    /// implementations must return `Ok(0)` (consistent with
    /// [`std::io::Read`] semantics). A short read (returned count less than
    /// `dst.len()`) does not necessarily indicate EOF -- only `Ok(0)` is
    /// the definitive end-of-item signal.
    ///
    /// Connectors without native range support should keep this default,
    /// which returns a permanent [`ReadError::unsupported`] classification.
    fn read_range(
        &mut self,
        _item_ref: &ItemRef,
        _offset: u64,
        _dst: &mut [u8],
        _budgets: Budgets,
    ) -> Result<usize, ReadError> {
        Err(ReadError::unsupported("range_read"))
    }
}
```

Two methods define the read surface:

- **`open(&mut self, item_ref: &ItemRef, budgets: Budgets) -> Result<Box<dyn io::Read + Send>, ReadError>`** --
  Opens an item for sequential reading. The return type is a boxed trait
  object. The doc comment explains why: object safety requires it for `dyn
  ReadConnector` polymorphism, and the heap allocation is acceptable because
  `open` is a WARM-path operation (once per item, not once per byte). The
  `'static` constraint on the reader means implementations must own their
  resources -- no borrowing from `&self`.

- **`read_range(&mut self, ...) -> Result<usize, ReadError>`** -- An optional
  random-access fast path. The default returns `Err(ReadError::unsupported("range_read"))`,
  making it safe to leave unimplemented. Connectors that support range reads
  (S3 with byte-range headers, local filesystem with `seek`) override this.
  The doc comment specifies overflow safety requirements and EOF semantics.

---

## The Convenience Supertrait

When a call path requires both enumeration and read capabilities, it uses a
single bound instead of writing `EnumerationConnector + ReadConnector`
everywhere.

Here is the definition from `api.rs`:

```rust
/// Convenience supertrait for connector types.
///
/// Use this as a bound when a call path requires both enumeration and read
/// capabilities. The blanket implementation means connector types do not
/// need an explicit `impl ConnectorInstance`.
///
/// # Design constraint
///
/// Because there is a blanket `impl<T: EnumerationConnector + ReadConnector
/// + ?Sized> ConnectorInstance for T` (the `?Sized` bound enables `dyn
///   ConnectorInstance`), adding required methods to this trait would break the
/// blanket impl — the empty body cannot supply an implementation for
/// arbitrary `T`. This is intentional: `ConnectorInstance` is a **pure bound
/// alias**, not an extension point. New connector surface area belongs on
/// [`EnumerationConnector`] or [`ReadConnector`] directly.
pub trait ConnectorInstance: EnumerationConnector + ReadConnector {}

impl<T: EnumerationConnector + ReadConnector + ?Sized> ConnectorInstance for T {}
```

`ConnectorInstance` is a pure bound alias. The blanket `impl` means any type
that implements both `EnumerationConnector` and `ReadConnector` automatically
implements `ConnectorInstance`. The `?Sized` bound is essential: it enables
`dyn ConnectorInstance`, which is how the runtime holds connector objects
polymorphically. The doc comment makes the design constraint explicit: adding
methods to `ConnectorInstance` would break the blanket impl, so new surface
area always goes on the individual traits.

---

## The Four-Layer Architecture

The connector module is split into four focused layers, each with a distinct
responsibility. The module-level documentation in `mod.rs` names them
explicitly.

Here is the module structure from `mod.rs`:

```rust
mod api;
pub mod conformance;
pub mod page_validator;
mod types;
```

The layers build on each other in a strict dependency order:

```text
  Layer 1: types.rs
  +-------------------------------------------------+
  | Validated value wrappers (ItemKey, ItemRef,      |
  | TokenBytes), Cursor, ScanItem, EnumerationPage,  |
  | Budgets, ConnectorInputError                     |
  +-------------------------------------------------+
                          |
                          v
  Layer 2: api.rs
  +-------------------------------------------------+
  | ErrorClass, EnumerateError, ReadError,            |
  | ConnectorCapabilities, EnumerationConnector,      |
  | ReadConnector, ConnectorInstance                  |
  +-------------------------------------------------+
                          |
                          v
  Layer 3: page_validator.rs
  +-------------------------------------------------+
  | PageValidationError, PageValidationViolation,     |
  | validate_page, validate_page_range               |
  +-------------------------------------------------+
                          |
                          v
  Layer 4: conformance.rs
  +-------------------------------------------------+
  | ConformanceConfig, EnumerationTrace,              |
  | ItemObservation, ConformanceError                 |
  +-------------------------------------------------+
```

**Layer 1 (types)** defines value types and validation errors. These are pure
data: no behavior beyond construction, access, and formatting. Every type
validates at construction time (non-empty, bounded, correctly paired). This
layer has no knowledge of traits or operations.

**Layer 2 (api)** defines operation contracts. It imports value types from
Layer 1 and defines traits that connectors implement. The error types
(`EnumerateError`, `ReadError`) model operation outcomes. The capability
struct (`ConnectorCapabilities`) advertises connector features. This layer
knows about `ShardSpec` from Boundary 2's coordination module but does not
depend on any runtime connector.

**Layer 3 (page_validator)** defines page-level diagnostic types and
validation functions. It checks that connector-produced pages satisfy
ordering, range-membership, cursor-monotonicity, and progression invariants.
The validator uses types from Layers 1 and 2 but adds no new traits.

**Layer 4 (conformance)** defines a test harness that validates connector
behavior end-to-end. It performs baseline enumeration, determinism checks,
resume-point verification, and secret scanning. This layer builds on all
three lower layers and is typically used in integration tests and bring-up
workflows.

The `mod.rs` re-exports flatten all four layers into a single import
boundary. Runtime crates import from `gossip_contracts::connector` and
receive types from all layers without needing to know which internal module
defines each type.

---

## Responsibility Boundaries

The connector contract layer owns a precise slice of the system. Understanding
what it owns -- and what it explicitly does not own -- prevents the layer
from becoming a catch-all.

**What connectors own:**

- Value types for enumeration data (`ItemKey`, `ItemRef`, `TokenBytes`,
  `Cursor`, `ScanItem`, `EnumerationPage`).
- Trait contracts for listing and reading (`EnumerationConnector`,
  `ReadConnector`).
- Error classification for operation outcomes (`ErrorClass`,
  `EnumerateError`, `ReadError`).
- Capability negotiation (`ConnectorCapabilities`).
- Page-level validation diagnostics (`PageValidationError`, `validate_page`).
- Conformance testing vocabulary (`ConformanceConfig`, `ConformanceError`).

**What connectors do not own:**

- **Identity (Boundary 1).** `StableItemId` and `ObjectVersionId` are defined
  in the identity module. Connectors produce them but do not define them.
- **Coordination (Boundary 2).** Shard assignment, lease management, fencing,
  and cursor persistence live in the coordination module. The connector layer
  bridges to coordination via `Cursor::as_update()` and
  `Cursor::try_from_update()`, but the coordination state machine is not the
  connector's concern.
- **Shard Algebra (Boundary 3).** Key encoding, range arithmetic, and shard
  construction live in `gossip-frontier`. Connectors receive `ShardSpec`
  values from the coordinator and enumerate within them; they do not construct
  or split shards.
- **Persistence (Boundary 5).** Where scan results are stored, how they are
  indexed, and how they are recovered after a crash -- all persistence
  concerns live elsewhere.
- **Retry and backoff policy.** The connector contract defines error
  classification (`Retryable` vs. `Permanent`) and advisory backoff hints
  (`retry_after_ms`). The actual retry scheduling, circuit-breaker
  transitions, and global rate limiting are runtime concerns.

---

## The Toxic-Byte Principle

Every byte that crosses the connector boundary is secrets-adjacent. An
`ItemRef` might encode an access key. An `ItemKey` might contain a file path
that reveals repository structure. A `TokenBytes` might contain a session
token. Even a `Location` display string might include a pre-signed URL with
embedded credentials.

The connector contract enforces a single formatting rule: `Debug` and
`Display` implementations on toxic-byte wrappers must never emit raw content.
Instead, they produce a redacted form: `TypeName(len=N, hash=XXXXXXXX..)`,
where the hash is the first 4 bytes of the BLAKE3 digest in lowercase hex.
Both `Debug` and `Display` produce identical output. There is no format trait
that bypasses redaction. Callers who need raw bytes must explicitly call
`as_bytes()` or `into_bytes()` -- a deliberate, auditable action.

This is not defense against malicious code. It is defense against the S3
connector scenario in the opening: a developer writes `{:?}` in a log line,
and the type's formatting implementation decides what appears. If `Debug`
auto-derives to show raw bytes, the secret leaks. If `Debug` is hand-written
to show `len + hash`, the log line is operationally useful (the hash
correlates entries across log lines) without being a liability.

Chapter 2 examines each toxic-byte wrapper in detail -- how the
`define_toxic_bytes!` macro generates them, what size bounds they enforce,
and how the ordered vs. unordered distinction shapes the type's API surface.

---

## Compilation Tiers and Dependency Direction

The `gossip-contracts` crate sits at Tier 0 in the compilation graph: it
defines interfaces that all other crates depend on. The connector module
within `gossip-contracts` defines trait contracts and value types. Actual
connector implementations (S3, GitHub, filesystem) live in separate crates at
Tier 1.

The dependency direction is strict and uni-directional:

```text
  Tier 0: gossip-contracts
  +------------------------------------------+
  | connector::types     (value wrappers)     |
  | connector::api       (trait contracts)    |
  | connector::page_validator (diagnostics)   |
  | connector::conformance (test harness)     |
  | coordination::*      (shard, cursor, ...)  |
  | identity::*          (StableItemId, ...)   |
  +------------------------------------------+
                    ^
                    | depends on
                    |
  Tier 1: gossip-connector-s3, gossip-connector-github, ...
  +------------------------------------------+
  | impl EnumerationConnector for S3Connector |
  | impl ReadConnector for S3Connector        |
  +------------------------------------------+
```

There is no mutual dependency between Boundary 2 (coordination) and
Boundary 4 (connector). The connector module imports `ShardSpec` and
`CursorUpdate` from coordination, but coordination never imports connector
types. This one-way dependency means changing a connector's value types does
not recompile the coordination module, and changing coordination's internal
state machine does not recompile the connector contracts.

The `api.rs` module makes this dependency explicit in its imports:

```rust
use crate::coordination::ShardSpec;

use super::{Budgets, Cursor, EnumerationPage, ItemKey, ItemRef};
```

`ShardSpec` comes from the coordination module. Everything else comes from
the sibling `types` module via `super::`. The connector module reaches into
coordination for exactly one type -- the shard specification that defines what
range a connector enumerates within. It does not reach into coordination for
leases, fencing tokens, run status, or any other state machine concerns.

---

## Error Classification

Connector operations can fail in two structurally distinct phases:
input validation (before the operation runs) and operation execution (during
the remote call). The connector contract assigns each phase its own error
surface.

**Input validation** errors use `ConnectorInputError`, defined in `types.rs`.
These cover byte-level invariant violations: empty keys, oversized tokens,
cursors with tokens but no keys, zero budgets. These errors are deterministic
-- the same input always produces the same error -- and they fire before any
network call.

**Operation execution** errors use `EnumerateError` and `ReadError`, defined
in `api.rs`. These carry three fields: an `ErrorClass` (retryable vs.
permanent), a connector-originated diagnostic message, and an optional
backoff hint. The two types are structurally identical but nominally distinct.

Here is the definition from `api.rs`:

```rust
/// Binary retry posture for connector operation failures.
///
/// Orchestration layers use this to decide whether to re-attempt an operation
/// or to escalate (park the shard, trigger a circuit breaker transition, etc.).
/// The classification is set by the connector at error-construction time and is
/// immutable thereafter.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
#[non_exhaustive]
pub enum ErrorClass {
    /// Transient or capacity-related failure. The same request may succeed on
    /// retry without any change to inputs or configuration. Typical causes:
    /// network timeouts, HTTP 429/503, temporary service unavailability.
    Retryable,
    /// The same request will not succeed until something external changes --
    /// credentials, permissions, resource existence, or connector configuration.
    /// Typical causes: HTTP 401/403/404, malformed resource identifiers.
    Permanent,
}
```

`ErrorClass` is a binary signal: either the operation is worth retrying, or it
is not. The `#[non_exhaustive]` attribute allows future variants (e.g., a
`Degraded` class for partial successes) without a breaking change.

The module documentation explains why `EnumerateError` and `ReadError` are
separate types despite having identical fields: "enumeration and reading are
modeled as separate operations with independent scaling characteristics
(metadata-bound vs bandwidth-bound). Keeping their error types distinct means
public signatures are unambiguous about which operation failed, orchestration
can apply different retry/circuit-breaker policies per operation without
downcasting or tag-matching, and the compiler prevents accidental
cross-assignment between the two paths."

---

## Capability Negotiation

Connectors declare their features at registration time through a static
capability struct.

Here is the definition from `api.rs`:

```rust
/// Feature flags that a connector advertises at registration time.
///
/// Orchestration and planning layers use these to choose enumeration strategy
/// (key-seek vs token-resume), decide whether range reads are available, and
/// determine if the connector can emit split hints for dynamic re-sharding.
///
/// `Default` produces a conservative "no features" profile (all fields
/// `false`), which is safe for connectors that have not been updated to
/// declare capabilities. Callers should still handle [`ReadError::unsupported`]
/// at call time, because a capability flag is a *static declaration of
/// intent*, not a guarantee that every call will succeed.
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct ConnectorCapabilities {
    /// The connector can resume enumeration from an arbitrary key position.
    ///
    /// When `true`, the runtime may supply a `last_key` cursor field to skip
    /// ahead in sorted enumeration order. Connectors without this must
    /// enumerate from the beginning of each shard range.
    pub seek_by_key: bool,
    /// The connector supports opaque token-based pagination.
    ///
    /// When `true`, the runtime will round-trip an opaque [`TokenBytes`]
    /// continuation token between pages. This is the natural model for APIs
    /// that return a "next page token" (e.g., S3 `ListObjectsV2`).
    ///
    /// [`TokenBytes`]: crate::connector::TokenBytes
    pub token_resume: bool,
    /// The connector can serve byte-range reads of item content.
    ///
    /// When `false`, the runtime must read entire items. Attempted range
    /// reads against a non-capable connector produce
    /// [`ReadError::unsupported`].
    pub range_read: bool,
    /// The connector emits split hints alongside enumeration pages.
    ///
    /// Split hints inform the sharding layer where natural partition
    /// boundaries exist (e.g., Git tree object boundaries, S3 key prefixes).
    /// The runtime may use these to dynamically subdivide large shards.
    pub split_hints: bool,
}
```

Four boolean flags control orchestration behavior:

- **`seek_by_key`** -- Whether the connector can resume enumeration from an
  arbitrary key position. A filesystem connector that lists sorted directory
  entries can seek by key. An API that only supports opaque page tokens
  cannot.

- **`token_resume`** -- Whether the connector uses opaque continuation tokens.
  S3's `ListObjectsV2` returns a `NextContinuationToken`; the connector
  wraps it in `TokenBytes` and the runtime round-trips it through the cursor.

- **`range_read`** -- Whether the connector supports byte-range reads. S3
  supports `Range` headers; a streaming API that only serves full responses
  does not.

- **`split_hints`** -- Whether the connector can suggest natural partition
  boundaries. This feeds the shard algebra layer with domain-specific
  knowledge that pure byte-midpoint splitting cannot provide.

The `Default` implementation sets all flags to `false`. This is the
conservative profile: a connector that declares no capabilities works
correctly (it enumerates from the beginning, uses no tokens, reads full
items, provides no split hints). Capability flags are additive. A connector
opts in to features; it never needs to opt out.

---

## Summary

The connector contract layer in `gossip-contracts::connector` provides a
uniform interface over heterogeneous data sources. Two traits --
`EnumerationConnector` for listing and `ReadConnector` for content access --
model operations with independent scaling and failure characteristics. A
convenience supertrait `ConnectorInstance` provides a single bound when both
are needed. Four internal layers (types, api, page_validator, conformance)
separate value validation from trait contracts from page diagnostics from
end-to-end harness testing. The toxic-byte principle ensures that `Debug` and
`Display` never emit raw connector bytes, preventing accidental secret
leakage into logs. Error classification splits cleanly into input validation
(`ConnectorInputError`) and operation outcomes (`EnumerateError`,
`ReadError`), with a binary `ErrorClass` that orchestration uses for retry
decisions. Capability negotiation via `ConnectorCapabilities` lets connectors
declare features additively against a conservative default.

Chapter 2 examines each toxic-byte wrapper in detail -- `ItemKey`, `ItemRef`,
`TokenBytes`, `Cursor`, `ScanItem`, `EnumerationPage`, `Budgets`, and
`VersionId` -- showing how the `define_toxic_bytes!` macro generates them and
how named constructors make invalid states unrepresentable.

> **Deprecation note:** The `EnumerationConnector` and `ReadConnector` trait
> definitions at `api.rs:344-354` carry a deprecation notice: "Deprecated
> surface: new unified execution paths should use
> `gossip_scan_driver::ScanSourceFactory` and
> `gossip_scan_driver::ScanDriver`." The page-by-page traits remain supported
> for existing Phase III connector callers, but new orchestration paths use
> scan-driver adapter factories (covered in Chapter 10).
