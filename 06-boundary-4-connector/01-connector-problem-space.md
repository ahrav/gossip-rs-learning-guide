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
types, error taxonomy, and validation rules that every connector must satisfy.
Runtime connector implementations live elsewhere. The contracts crate owns the
interface; it does not own the implementations.

Recall from Chapter 5 (Boundary 1) that `StableItemId` and `ConnectorTag`
provide domain-separated identity. Those identity types flow through the
connector boundary -- connectors produce `StableItemId` values that
downstream components consume without knowing whether the ID came from S3 or
GitHub. The connector contract layer builds on that foundation by defining
what connectors produce (item references, cursors, split hints) and what
they consume (shard specs, budgets, resume cursors).

---

## The Two Method Groups

The connector contract organizes runtime behavior into two method groups:
enumeration methods for listing operations and read methods for content
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

By keeping the method groups separate, orchestration can compose them
independently. A connector might enumerate items on one thread and read items
on another. Error classification can park a shard on the read path (too many
retryable errors fetching content) without halting enumeration. A test harness
can mock enumeration while using a real read path, or vice versa.

Each concrete connector type (e.g., `FilesystemConnector`, `GitConnector`)
provides these methods as inherent `pub fn` methods. The
`OrderedContentSource` trait in `connector::ordered` provides a formal
contract for ordered-content connectors, while the Git family uses its own
trait surface in `connector::git`. The contract types in `api.rs`
(`ConnectorCapabilities`, `EnumerateError`, `ReadError`) define the shared
vocabulary that all connectors use.

Two methods define the planning surface:

- **`pub fn caps(&self) -> ConnectorCapabilities`** -- A static declaration
  of what the connector supports. Orchestration queries this at registration
  time to plan its enumeration strategy. The capabilities are declarative
  intent, not runtime guarantees -- callers must still handle errors from
  operations that the connector claims to support.

- **`pub fn choose_split_point(&mut self, ...) -> Result<Option<ItemKey>, EnumerateError>`** --
  An optional hook for connectors that know where natural partition boundaries
  exist (Git tree boundaries, S3 common prefixes). Connectors without
  meaningful split knowledge return `Ok(None)`.

Two methods define the read surface:

- **`pub fn open(&mut self, item_ref: &ItemRef, budgets: Budgets) -> Result<Box<dyn io::Read + Send>, ReadError>`** --
  Opens an item for sequential reading. The return type is a boxed trait
  object. The heap allocation is acceptable because `open` is a WARM-path
  operation (once per item, not once per byte). The `'static` constraint on
  the reader means implementations must own their resources -- no borrowing
  from `&self`.

- **`pub fn read_range(&mut self, ...) -> Result<usize, ReadError>`** -- An
  optional random-access fast path. Connectors without native range support
  return `Err(ReadError::unsupported("range_read"))`. Connectors that support
  range reads (S3 with byte-range headers, local filesystem with `seek`)
  provide a real implementation. The method specifies overflow safety
  requirements and EOF semantics (returning `Ok(0)` at end-of-item).

---

## The Six-Module Architecture

The connector module is split into six focused submodules, each with a
distinct responsibility. Two are internal organization units (`api`, `types`)
whose public items are re-exported at the `connector::` boundary. Four are
public namespaced modules (`common`, `conformance`, `ordered`, `git`) that
define family-specific contracts and reusable test infrastructure.

Here is the module structure from `mod.rs`:

```rust
mod api;
pub mod common;
pub mod conformance;
pub mod git;
pub mod ordered;
mod types;
```

The modules compose in a layered dependency order:

```text
  Layer 1: types.rs
  +-------------------------------------------------+
  | Validated value wrappers (ItemKey, ItemRef,      |
  | TokenBytes), Cursor, ScanItem, Budgets,          |
  | ConnectorInputError, VersionId, ToxicDigest      |
  +-------------------------------------------------+
                          |
                          v
  Layer 2: api.rs
  +-------------------------------------------------+
  | ErrorClass, EnumerateError, ReadError,            |
  | ConnectorCapabilities                             |
  +-------------------------------------------------+
                          |
                          v
  Layer 3: common.rs (pub)
  +-------------------------------------------------+
  | Shared paging vocabulary: PageBuf, PageState,     |
  | PagingCapabilities, KeyedPageItem,                |
  | validate_filled_page                              |
  +-------------------------------------------------+
                       /  |  \
                      v   |   v
  Layer 4a: ordered.rs  | Layer 4b: git.rs
  +---------------------+ | +------------------------+
  | OrderedContent       | | | Git family contract:   |
  |   Capabilities       | | |   RepoKey, RepoLocator,|
  | OrderedContent       | | |   GitRepoTarget,       |
  |   Source trait        | | |   GitRepoDiscovery     |
  | (fill_page,          | | |   Source, GitMirror     |
  |  choose_split_point, | | |   Manager, GitRepo     |
  |  open, read_range)   | | |   Executor             |
  +---------------------+ | +------------------------+
                           v
  Layer 4c: conformance.rs (pub)
  +-------------------------------------------------+
  | Reusable ordered-content conformance harness:     |
  | run_ordered_content_conformance,                  |
  | drain_ordered_source                              |
  +-------------------------------------------------+
```

**Layer 1 (types)** defines value types and validation errors. These are pure
data: no behavior beyond construction, access, and formatting. Every type
validates at construction time (non-empty, bounded, correctly paired). This
layer has no knowledge of traits or operations.

**Layer 2 (api)** defines the shared vocabulary for operation outcomes and
capabilities. It imports value types from Layer 1 and defines the error types
(`EnumerateError`, `ReadError`) that model operation outcomes, plus the
capability struct (`ConnectorCapabilities`) that advertises connector
features.

**Layer 3 (common)** defines shared paging vocabulary reused across connector
families: `PageBuf`, `PageState`, `PagingCapabilities`, `KeyedPageItem`, and
the `validate_filled_page` helper.

**Layer 4a (ordered)** defines the ordered-content family contract:
`OrderedContentCapabilities` and the `OrderedContentSource` trait with
`fill_page`, `open`, `read_range`, and `choose_split_point` methods. This
family models sources whose worker loop fills bounded ordered pages of
`ScanItem` values.

**Layer 4b (git)** defines the Git family contract: `RepoKey`, `RepoLocator`,
`GitRepoTarget`, `GitSelection`, `LocalMirror`, `GitExecutionLimits`,
`GitRunOutcome`, `GitRunError`, `GitDiscoveryCapabilities`,
`GitRepoDiscoverySource`, `GitMirrorManager`, and `GitRepoExecutor`. This
family models whole-repository execution rather than item-by-item enumeration.

**Layer 4c (conformance)** provides a reusable ordered-content conformance
harness (`run_ordered_content_conformance`, `drain_ordered_source`) that
validates connector implementations against the `OrderedContentSource` contract.

Family modules compose from the shared layers instead of inheriting a single
universal connector model: `ordered`, `git`, and `conformance` depend on
`common`, `types`, and `api` for paging, value wrappers, and error
classification.

The `mod.rs` re-exports flatten `api` and `types` into a single import
boundary at `gossip_contracts::connector`. The family contracts stay
namespaced under `connector::ordered` and `connector::git`, and the paging
vocabulary is accessible under `connector::common`. Runtime crates import
from `gossip_contracts::connector` and receive shared types without needing
to know which internal module defines each type.

---

## Responsibility Boundaries

The connector contract layer owns a precise slice of the system. Understanding
what it owns -- and what it explicitly does not own -- prevents the layer
from becoming a catch-all.

**What connectors own:**

- Value types for scan data (`ItemKey`, `ItemRef`, `TokenBytes`,
  `Cursor`, `ScanItem`).
- API vocabulary for split-point selection and reading (`ConnectorCapabilities`,
  `EnumerateError`, `ReadError`).
- Error classification for operation outcomes (`ErrorClass`,
  `EnumerateError`, `ReadError`).
- Capability negotiation (`ConnectorCapabilities`).

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
  (`retry_after_ms`). The actual retry scheduling, shard parking
  decisions, and global rate limiting are runtime concerns.

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
within `gossip-contracts` defines API vocabulary and value types. Actual
connector implementations (S3, GitHub, filesystem) live in separate crates at
Tier 1.

The dependency direction is strict and uni-directional:

```text
  Tier 0: gossip-contracts
  +----------------------------------------------+
  | connector::types     (value wrappers)         |
  | connector::api       (error & capability types)|
  | connector::common    (paging vocabulary)       |
  | connector::ordered   (OrderedContentSource)    |
  | connector::git       (git family contract)     |
  | coordination::*      (shard, cursor, ...)      |
  | identity::*          (StableItemId, ...)       |
  +----------------------------------------------+
                    ^
                    | depends on
                    |
  Tier 1: gossip-connectors (filesystem, git, in-memory), ...
  +----------------------------------------------+
  | FilesystemConnector (ordered-content family)  |
  | GitConnector        (inherent methods only)   |
  | InMemoryDeterministicConnector                |
  +----------------------------------------------+
```

There is no mutual dependency between Boundary 2 (coordination) and
Boundary 4 (connector). The connector module imports `ShardSpec` and
`CursorUpdate` from coordination, but coordination never imports connector
types. This one-way dependency means changing a connector's value types does
not recompile the coordination module, and changing coordination's internal
state machine does not recompile the connector contracts.

The `ordered.rs` module makes this dependency explicit in its imports:

```rust
use crate::coordination::ShardSpec;

use super::common::PageBuf;
use super::{Budgets, Cursor, EnumerateError, ItemKey, ItemRef, ReadError, ScanItem};
```

`ShardSpec` comes from the coordination module. Everything else comes from
the sibling modules via `super::`. The connector module reaches into
coordination for exactly one type -- the shard specification that defines what
range a connector enumerates within. It does not reach into coordination for
leases, fencing tokens, run status, or any other state machine concerns.
Note that `api.rs` itself has no imports from coordination or sibling
connector modules -- it imports only `std::fmt`. The cross-boundary dependency
surfaces in the family modules (`ordered.rs`, `git.rs`) that define trait
methods accepting `ShardSpec` parameters.

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
/// or to escalate (park the shard, etc.).
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
can apply different retry/parking policies per operation without
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
uniform interface over heterogeneous data sources. Two method groups --
planning (`caps`, `choose_split_point`) for shard management and
read (`open`, `read_range`) for content access -- model operations with
independent scaling and failure characteristics. Concrete connector types
provide these as inherent methods; polymorphism lives at the family-specific
trait layer (`OrderedContentSource` for ordered-content connectors, dedicated
Git traits for repository execution). Six submodules (types, api, common,
conformance, ordered, git) separate value validation from API vocabulary,
shared paging primitives, conformance testing infrastructure, and
family-specific contracts. The toxic-byte principle
ensures that `Debug` and `Display` never emit raw connector bytes,
preventing accidental secret leakage into logs. Error classification splits
cleanly into input validation (`ConnectorInputError`) and operation outcomes
(`EnumerateError`, `ReadError`), with a binary `ErrorClass` that
orchestration uses for retry decisions. Capability negotiation via
`ConnectorCapabilities` lets connectors declare features additively against a
conservative default.

Chapter 2 examines each toxic-byte wrapper in detail -- `ItemKey`, `ItemRef`,
`TokenBytes`, `Cursor`, `ScanItem`, `Budgets`, and `VersionId` -- showing
how the `define_toxic_bytes!` macro generates them and how named constructors
make invalid states unrepresentable.
