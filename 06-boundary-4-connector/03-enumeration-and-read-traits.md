# The Shared Connector Surface -- Split, Read, and Capability Methods

*A GitHub connector encounters a 401 Unauthorized response from the GitHub
API. The connector author classifies the error as `ErrorClass::Retryable` --
a reasonable guess, since the API occasionally returns spurious 401s during
key rotations. But this 401 is permanent: the integration token was revoked
by an organization admin ten minutes ago. The scan loop dutifully retries.
Attempt 1. Attempt 2. Each attempt consumes one request against GitHub's
5,000-request-per-hour rate limit. By attempt 47, the connector has burned
nearly 1% of the hourly budget on a request that will never succeed. By
attempt 100, the scan loop exhausts its transient-retry budget and finally
parks the shard. But the damage is done: every other connector instance
sharing the same GitHub App installation is now throttled. Scanning across
340 repositories stalls for 38 minutes while the rate-limit window resets.
A single miscategorized error -- `Retryable` instead of `Permanent` --
cascades into a full-source outage.*

---

## Why a Shared Connector Surface Exists

The failure above has a root cause that no amount of retry logic can fix:
the error classification was wrong at the source. The connector told the
runtime "try again" when the correct signal was "stop trying." The system
needs a contract that forces connector authors to make this decision
explicitly at error-construction time, with no escape hatch for ambiguity.

The `gossip-contracts::connector::api` module defines the error taxonomy
(`ErrorClass`, `EnumerateError`, `ReadError`) and the capability declaration
struct (`ConnectorCapabilities`) that underpin this contract. The connector
module organizes polymorphism through *family-specific traits*: the
`ordered::OrderedContentSource` trait for item-at-a-time sources (filesystem,
in-memory) and the `git::GitRepoExecutor` / `git::GitRepoDiscoverySource` /
`git::GitMirrorManager` traits for repo-native Git execution.

Chapter 2 defined the value types (`ScanItem`, `Budgets`, `Cursor`) that
flow through these traits. This chapter defines the traits themselves and
the error and capability types that govern their behavior.

---

## The Family-Based Trait Architecture

Connector polymorphism lives at the trait level, organized into families that
match each source type's natural execution model. Each family composes from
shared infrastructure (`common.rs` for paging, `types.rs` for value wrappers,
`api.rs` for error classification) and adds the trait surface specific to
its execution style.

### Shared Paging Vocabulary (common.rs)

All connector families share a paging vocabulary defined in `common.rs`.
The key types are:

- **`PageBuf<T>`** -- A non-empty typed page container. Carries a `Vec<T>`
  of items plus a `PageState` indicating whether the page is terminal or
  resumable.

- **`PageState`** -- Either `HasMore { cursor }` (the caller must resume
  with the supplied cursor) or `Complete` (the page is terminal for the
  current scope).

- **`KeyedPageItem`** -- A trait for items that participate in ordered page
  emission. Requires `item_key()` (the ordered key) and `size_hint()`.
  Both `ScanItem` (for ordered sources) and `GitRepoTarget` (for Git
  discovery) implement this trait.

- **`validate_filled_page`** -- Validates ordering, uniqueness, and
  shard-bound rules generically for any `KeyedPageItem` page.

### The Ordered-Content Family (ordered.rs)

The `OrderedContentSource` trait models sources whose worker loop is
naturally: fill a bounded page of `ScanItem` values, scan or skip committed
items, open or range-read item bytes, and checkpoint progress. Filesystem
and in-memory connectors implement this trait.

```rust
pub trait OrderedContentSource: Send {
    /// Returns the ordered-content features this source supports.
    fn capabilities(&self) -> OrderedContentCapabilities;

    /// Fill one bounded page of ordered content items within `shard`.
    fn fill_page(
        &mut self,
        shard: &ShardSpec,
        cursor: &Cursor,
        budgets: Budgets,
    ) -> Result<Option<PageBuf<ScanItem>>, EnumerateError>;

    /// Suggest a split point strictly inside the remaining shard suffix.
    fn choose_split_point(
        &mut self,
        _shard: &ShardSpec,
        _cursor: &Cursor,
        _budgets: Budgets,
    ) -> Result<Option<ItemKey>, EnumerateError> {
        Ok(None)
    }

    /// Open the full content for an item.
    fn open(
        &mut self,
        item_ref: &ItemRef,
        budgets: Budgets,
    ) -> Result<Box<dyn io::Read + Send>, ReadError>;

    /// Read a byte range from item content.
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

Five methods, not four. The key addition is `fill_page`, which replaces the
old ad-hoc enumeration pattern with a structured page-returning method.
`fill_page` returns `Ok(Some(page))` with a non-empty `PageBuf<ScanItem>`
or `Ok(None)` to signal terminal completion (the empty terminal call in the
two-call shard completion pattern).

`OrderedContentCapabilities` is a separate capability struct from the
shared `ConnectorCapabilities`. It declares three flags specific to
ordered-content sources:

```rust
pub struct OrderedContentCapabilities {
    pub range_read: bool,
    pub split_hints: bool,
    pub token_resume: bool,
}
```

`choose_split_point` and `read_range` have default implementations (return
`Ok(None)` and `Err(ReadError::unsupported("range_read"))` respectively),
so connectors that lack these features only need to implement
`capabilities`, `fill_page`, and `open`.

### The Git Family (git.rs)

Git execution is modeled separately because it operates on whole
repositories: commit walks, tree diffs, pack-order blob scans, and watermark
persistence. There is no `open()` or `read_range()` -- the executor controls
the full scan lifecycle internally.

The Git family splits execution into three traits, each representing a
distinct phase:

**`GitRepoDiscoverySource`** -- Pages over repository work units in
deterministic frontier order:

```rust
pub trait GitRepoDiscoverySource: Send {
    fn capabilities(&self) -> GitDiscoveryCapabilities;

    fn discover_page(
        &mut self,
        shard: &ShardSpec,
        cursor: &Cursor,
        budgets: Budgets,
    ) -> Result<Option<PageBuf<GitRepoTarget>>, EnumerateError>;

    fn choose_split_point(
        &mut self,
        _shard: &ShardSpec,
        _cursor: &Cursor,
        _budgets: Budgets,
    ) -> Result<Option<ItemKey>, EnumerateError> {
        Ok(None)
    }
}
```

Each `GitRepoTarget` carries a `RepoKey` (a semantic newtype over
`ItemKey`) and a `RepoLocator` (where the repository can be found). The
discovery page uses the same `PageBuf<T>` container as ordered sources,
because `GitRepoTarget` implements `KeyedPageItem`.

**`GitMirrorManager`** -- Acquires or refreshes a local mirror suitable
for execution:

```rust
pub trait GitMirrorManager: Send {
    fn sync_mirror(&mut self, locator: &RepoLocator) -> Result<LocalMirror, GitRunError>;
}
```

**`GitRepoExecutor`** -- Runs the repo-native Git scan against a prepared
mirror:

```rust
pub trait GitRepoExecutor: Send {
    fn run_repo(
        &mut self,
        mirror: &LocalMirror,
        selection: &GitSelection,
        limits: GitExecutionLimits,
    ) -> Result<GitRunOutcome, GitRunError>;
}
```

`GitSelection` combines ref selection policy (`GitRefSelection`), scan mode
(`GitScanMode`: diff-history or ODB-blob), and merge strategy
(`GitMergeStrategy`) into one contract-level value. `GitExecutionLimits`
carries resource knobs (worker counts, cache sizes, binary scanning flag)
without depending on runtime Git types.

The Git family has its own error type, `GitRunError`, generated by the same
`define_connector_error!` macro used for `EnumerateError` and `ReadError`.
It adds one domain-specific constructor: `concurrent_maintenance()` for
the retryable case where a concurrent `git gc` invalidates repository state.

### Why Traits Instead of Inherent Methods

Polymorphism lives at the trait level. The runtime holds `dyn
OrderedContentSource` or `dyn GitRepoExecutor` values -- no factories
or adapters are needed to achieve dispatch. Each family trait is object-safe
and `Send`-bounded, so implementations can be passed across thread
boundaries and stored behind trait objects.

This is a deliberate departure from a "structural convention" model where
connectors provide matching inherent methods without a shared trait. The
trait-based approach gives the compiler full enforcement: if a filesystem
connector forgets to implement `fill_page`, it simply does not compile as
an `OrderedContentSource`. The Git family's three-trait decomposition
(discovery, mirroring, execution) further ensures that each phase can be
tested, mocked, and composed independently.

---

## ErrorClass -- The Binary Retry Decision

The most consequential decision a connector makes is whether an error is
retryable. The `ErrorClass` enum forces this to be a binary choice with no
middle ground.

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

impl ErrorClass {
    /// Returns `true` when the classification is [`Retryable`](Self::Retryable).
    #[inline]
    #[must_use]
    pub const fn is_retryable(self) -> bool {
        matches!(self, Self::Retryable)
    }
}

impl fmt::Display for ErrorClass {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::Retryable => f.write_str("retryable"),
            Self::Permanent => f.write_str("permanent"),
        }
    }
}
```

Two variants. No `Unknown`. No `MaybeRetryable`. No `TransientButUnsure`.

This is a deliberate design choice. An "unknown" variant would push the
retry decision to the orchestration layer, which has no domain knowledge
about whether a specific API error is transient. The connector author is the
only person who knows whether a given HTTP status code, gRPC error, or
filesystem condition is recoverable. By restricting the enum to two variants,
the type system forces the connector author to make that determination at
construction time.

The `#[non_exhaustive]` attribute allows future variants (if the retry model
ever needs refinement), but the current design is intentionally minimal.
Every downstream consumer -- the scan loop, circuit breakers, metrics
counters -- branches on `is_retryable()` and treats the world as binary.

---

## EnumerateError and ReadError -- Structurally Identical, Nominally Distinct

Enumeration and reading are separate operations with different performance
characteristics. Enumeration is metadata-bound (listing items). Reading is
bandwidth-bound (fetching content). They scale differently, fail
differently, and the runtime applies independent retry and circuit-breaker
policies to each. The error types must be distinct so that the compiler
prevents accidental cross-assignment.

Rather than duplicating the struct definition, the codebase uses a macro to
generate both types from a single template.

Here is the `define_connector_error!` macro from `api.rs`:

```rust
/// Generates a connector error type with private fields, named constructors,
/// accessors, and trait impls (`Display`, `Error`).
///
/// Each invocation produces a distinct nominal type with identical structure,
/// preserving type safety between different connector operations (see module
/// docs for rationale). The generated struct has three private fields:
///
/// - `class: ErrorClass` — binary retry posture
/// - `message: String` — connector-originated diagnostic text
/// - `retry_after_ms: Option<u64>` — optional advisory backoff hint
///
/// Generated API surface per type:
/// - Accessors: `class()`, `message()`, `retry_after_ms()`, `is_retryable()`, `into_message()`
/// - Constructors: `retryable()`, `rate_limited()`, `permanent()`
/// - Traits: `Clone`, `Debug`, `PartialEq`, `Eq`, `Display`, `Error`
macro_rules! define_connector_error {
    (
        $(#[$meta:meta])*
        $name:ident
    ) => {
        $(#[$meta])*
        #[derive(Clone, Debug, PartialEq, Eq)]
        pub struct $name {
            class: ErrorClass,
            message: String,
            retry_after_ms: Option<u64>,
        }

        impl $name {
            /// Returns the retryability classification for this failure.
            #[inline]
            #[must_use]
            pub fn class(&self) -> ErrorClass {
                self.class
            }

            /// Returns the connector-originated diagnostic text.
            ///
            /// Not sanitized by this type -- callers must treat it as untrusted
            /// when logging or displaying.
            #[inline]
            #[must_use]
            pub fn message(&self) -> &str {
                &self.message
            }

            /// Returns the optional connector-provided retry hint in milliseconds.
            ///
            /// Advisory only: the runtime may impose stricter or global backoff
            /// policies. A value of `0` is passed through without normalization.
            #[inline]
            #[must_use]
            pub fn retry_after_ms(&self) -> Option<u64> {
                self.retry_after_ms
            }

            /// Returns `true` when the error is classified as
            /// [`Retryable`](ErrorClass::Retryable).
            ///
            /// Convenience shorthand for `self.class().is_retryable()`.
            #[inline]
            #[must_use]
            pub fn is_retryable(&self) -> bool {
                self.class.is_retryable()
            }

            /// Consumes the error and returns the owned diagnostic message.
            #[inline]
            #[must_use]
            pub fn into_message(self) -> String {
                self.message
            }

            /// Construct a retryable error without a backoff hint.
            ///
            /// Use for transient failures where the connector has no opinion on
            /// when to retry (e.g., a generic network timeout).
            #[must_use]
            pub fn retryable(message: impl Into<String>) -> Self {
                Self {
                    class: ErrorClass::Retryable,
                    message: message.into(),
                    retry_after_ms: None,
                }
            }

            /// Construct a retryable error with a connector-supplied backoff hint.
            ///
            /// Typical source: an HTTP `Retry-After` header or equivalent API signal.
            /// The `retry_after_ms` value is stored as-is (including `0`) with no
            /// clamping or normalization -- that is the runtime's responsibility.
            #[must_use]
            pub fn rate_limited(message: impl Into<String>, retry_after_ms: u64) -> Self {
                Self {
                    class: ErrorClass::Retryable,
                    message: message.into(),
                    retry_after_ms: Some(retry_after_ms),
                }
            }

            /// Construct a permanent error.
            ///
            /// Use when the failure will persist until something external changes
            /// (credentials, permissions, resource existence). The runtime should
            /// not retry blindly; typical follow-up is to park the shard or skip
            /// the item with a diagnostic reason.
            #[must_use]
            pub fn permanent(message: impl Into<String>) -> Self {
                Self {
                    class: ErrorClass::Permanent,
                    message: message.into(),
                    retry_after_ms: None,
                }
            }
        }

        impl fmt::Display for $name {
            fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
                write!(f, "{}: ", self.class)?;
                fmt_sanitized_message(f, &self.message)?;
                if let Some(retry_after_ms) = self.retry_after_ms {
                    write!(f, " (retry_after_ms={})", retry_after_ms)?;
                }
                Ok(())
            }
        }

        /// Boundary-terminal: the underlying cause (IO, HTTP, parse errors) is
        /// flattened into `message` at construction time. Connectors should log
        /// the original error before constructing this type.
        impl std::error::Error for $name {}
    };
}
```

Each invocation of this macro generates a complete type. The two invocations
are:

```rust
define_connector_error! {
    /// Error returned by connector enumeration/list operations.
    ///
    /// Bundles orchestration-facing policy metadata ([`ErrorClass`] and an
    /// optional backoff hint) with a connector-originated diagnostic message.
    /// Constructed via named constructors ([`retryable`](Self::retryable),
    /// [`rate_limited`](Self::rate_limited), [`permanent`](Self::permanent))
    /// that enforce consistent class/retry combinations. Fields are private
    /// to preserve these invariants; use [`class`](Self::class),
    /// [`message`](Self::message), and [`retry_after_ms`](Self::retry_after_ms)
    /// to inspect.
    ///
    /// See also [`ReadError`], the structurally parallel type for the read/open
    /// path. The two are kept separate intentionally -- see the module docs for
    /// rationale.
    EnumerateError
}

define_connector_error! {
    /// Error returned by connector read/open operations.
    ///
    /// Structurally identical to [`EnumerateError`] (same three fields, same
    /// named-constructor pattern), but kept as a distinct type so the compiler
    /// enforces which operation path produced the error. This also allows
    /// orchestration to apply independent retry and circuit-breaker policies
    /// for reads vs enumerations.
    ///
    /// Fields are private to preserve constructor invariants; use
    /// [`class`](Self::class), [`message`](Self::message), and
    /// [`retry_after_ms`](Self::retry_after_ms) to inspect.
    ///
    /// In addition to the shared constructors, `ReadError` provides
    /// [`unsupported`](Self::unsupported) for capability mismatches discovered
    /// at call time.
    ReadError
}
```

### Named Constructors Enforce Invariants

The struct fields are private. There is no way to construct an `EnumerateError`
or `ReadError` except through the three named constructors:

- **`retryable(message)`** -- Creates a `Retryable` error with no backoff hint.
  The connector knows the failure is transient but has no opinion on when to
  retry.

- **`rate_limited(message, retry_after_ms)`** -- Creates a `Retryable` error
  with an advisory backoff duration. The `retry_after_ms` value is stored
  as-is, including `0`. No clamping, no normalization -- that responsibility
  belongs to the runtime. This constructor is the only way to attach a backoff
  hint, and the hint always pairs with `Retryable` class. A permanent error
  with a backoff hint is unrepresentable.

- **`permanent(message)`** -- Creates a `Permanent` error with no backoff hint.
  The combination of permanent class with a retry hint is structurally
  impossible.

These constructors eliminate an entire category of bugs: a connector cannot
accidentally create a permanent error that carries a retry hint, or a
retryable error that the runtime treats as permanent.

### ReadError::unsupported -- The Capability Mismatch Constructor

`ReadError` has one additional constructor beyond the macro-generated set.

Here is the definition from `api.rs`:

```rust
impl ReadError {
    /// Construct a permanent read error for an unsupported operation.
    ///
    /// Convenience for capability mismatches discovered at call time -- for
    /// example, a range-read attempt against a connector that does not
    /// advertise [`ConnectorCapabilities::range_read`]. The message is
    /// formatted as `"{feature} not supported"`.
    #[must_use]
    pub fn unsupported(feature: &'static str) -> Self {
        Self::permanent(format!("{feature} not supported"))
    }
}
```

This delegates to `permanent()` -- an unsupported operation is inherently
non-retryable. No amount of retrying will make a connector suddenly support
range reads.

### Display Sanitization

The `Display` impl generated by the macro calls `fmt_sanitized_message`,
which replaces control characters with U+FFFD (the Unicode replacement
character). This prevents log injection: a malicious or buggy connector
message containing ANSI escape sequences (`\x1b[31m`) or null bytes (`\x00`)
is rendered safely.

Here is the sanitization function from `api.rs`:

```rust
/// Writes `message` to `f`, replacing control characters with U+FFFD
/// (REPLACEMENT CHARACTER) to prevent log injection.
///
/// Replaced ranges: C0 (U+0000..U+001F) except HT/LF/CR, DEL (U+007F),
/// and C1 (U+0080..U+009F). Together with HT/LF/CR (which are preserved),
/// these ranges form the full set for which [`char::is_control()`] returns
/// `true`.
#[inline]
fn fmt_sanitized_message(f: &mut fmt::Formatter<'_>, message: &str) -> fmt::Result {
    for ch in message.chars() {
        if ch.is_control() && !matches!(ch, '\t' | '\n' | '\r') {
            f.write_str("\u{FFFD}")?;
        } else {
            fmt::Write::write_char(f, ch)?;
        }
    }
    Ok(())
}
```

Tabs, newlines, and carriage returns are preserved because they are
legitimate whitespace in multi-line diagnostic messages. Everything else in
the C0, DEL, and C1 control ranges is replaced. The raw `message()` accessor
returns the original unsanitized value -- the sanitization boundary is at
`Display`, not at storage.

---

## ConnectorCapabilities -- The Static Feature Declaration

Before the runtime calls any connector method, it needs to know what the
connector supports. `ConnectorCapabilities` is a plain struct of four
boolean flags, declared at construction time.

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

Field-by-field:

- **`seek_by_key: bool`** -- Controls whether the runtime can pass a
  `last_key` cursor to resume enumeration at an arbitrary key position. APIs
  that support seek (like listing files in a sorted directory) set this to
  `true`. APIs that only support forward pagination through opaque tokens
  (like S3 `ListObjectsV2`) set this to `false`.

- **`token_resume: bool`** -- Controls whether the runtime round-trips
  opaque continuation tokens between pages. This is the natural model for
  cloud APIs that return a "next page token." A connector may support both
  `seek_by_key` and `token_resume` if its underlying API offers both
  mechanisms.

- **`range_read: bool`** -- Controls whether the `read_range()` method is
  implemented. When `false`, `read_range()` returns
  `ReadError::unsupported("range_read")`. The runtime must fall back to
  reading entire items via `open()`.

- **`split_hints: bool`** -- Controls whether the connector can suggest
  split points for dynamic re-sharding. When `true`, the
  `choose_split_point()` method provides meaningful split candidates.

The `Default` implementation sets all four flags to `false`. This is the
conservative choice: a connector that declares no capabilities still works
correctly. The runtime falls back to sequential reads, no seek, no token
resume, and no split hints. The test suite verifies this:

```rust
#[test]
fn default_caps_are_conservative() {
    let capabilities = ConnectorCapabilities::default();
    assert!(!capabilities.seek_by_key);
    assert!(!capabilities.token_resume);
    assert!(!capabilities.range_read);
    assert!(!capabilities.split_hints);
}
```

---

## Runtime Connector Implementations

The family traits defined in `gossip-contracts` are implemented by concrete
connector types in the `gossip-connectors` crate:

- **`FilesystemConnector`** (`gossip-connectors/src/filesystem.rs`) --
  Implements `OrderedContentSource`. Enumerates files in sorted directory
  order, opens them for sequential reading, and supports byte-range reads
  via `seek`.

- **`InMemoryConnector`** (`gossip-connectors/src/in_memory.rs`) --
  Implements `OrderedContentSource`. Backed by an in-memory dataset for
  deterministic testing. Supports the full ordered-content surface including
  split hints and token-based pagination.

- **`GitConnector`** (`gossip-connectors/src/git.rs`) -- Implements the
  Git family traits (`GitRepoDiscoverySource`, `GitMirrorManager`,
  `GitRepoExecutor`). Discovers repositories, manages local mirrors, and
  executes repo-native Git scans.

The concrete connectors' `fill_page()`, `open()`, `discover_page()`, and
other trait methods are called directly by the scan orchestration layer
through trait objects. Callers hold `Box<dyn OrderedContentSource>` or
`Box<dyn GitRepoExecutor>` -- the trait boundary provides the
compiler-enforced polymorphism.

---

## Summary

The connector API module defines the runtime boundary between connector
implementations and the scan pipeline. `ErrorClass` forces a binary retry
decision at error construction time, eliminating the ambiguity that caused
the opening failure. `EnumerateError` and `ReadError` are structurally
identical but nominally distinct, preventing accidental cross-assignment
between enumeration and read paths. `ConnectorCapabilities` advertises
connector features through four boolean flags with conservative defaults.
Polymorphism lives at the family trait level: `OrderedContentSource` for
item-at-a-time connectors (with `capabilities()`, `fill_page()`,
`choose_split_point()`, `open()`, `read_range()`) and the Git family's
three-trait decomposition (`GitRepoDiscoverySource`, `GitMirrorManager`,
`GitRepoExecutor`) for repo-native execution. All traits are object-safe
and `Send`-bounded, giving the compiler full enforcement of the connector
contract.

Chapter 4 examines the circuit breaker design specification that governs
how error classification feeds into shard lifecycle decisions.
