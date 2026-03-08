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
struct (`ConnectorCapabilities`) that underpin this contract. Every connector
is expected to provide the same four inherent methods -- `caps()`,
`choose_split_point()`, `open()`, `read_range()` -- that form a shared
surface for split-point selection and content access.

Chapter 2 defined the value types (`ScanItem`, `Budgets`, `Cursor`) that
flow through these methods. This chapter defines the methods themselves and
the error and capability types that govern their behavior.

---

## The Four-Method Convention

Every connector struct exposes the same set of inherent methods. These are
not formalized as a trait; each connector is a concrete type. Polymorphism
lives at the `ScanSourceFactory` / `ScanDriver` layer (covered in Chapter
8), where the runtime selects the appropriate factory based on the scan
assignment's `ConnectorKind`.

The shared surface consists of two groups: **planning methods** that support
shard management and **read methods** that access item content.

### Planning Methods

```rust
// From InMemoryDeterministicConnector (identical signatures on FilesystemConnector)

pub fn caps(&self) -> ConnectorCapabilities {
    ConnectorCapabilities {
        seek_by_key: true,
        token_resume: self.emit_tokens,
        range_read: true,
        split_hints: true,
    }
}

pub fn choose_split_point(
    &mut self,
    shard: &ShardSpec,
    cursor: &Cursor,
    _budgets: Budgets,
) -> Result<Option<ItemKey>, EnumerateError> {
    // ...
}
```

- **`caps(&self) -> ConnectorCapabilities`** -- Returns the static feature
  profile. Takes `&self` (not `&mut self`) because capability queries must
  not mutate connector state. Called during planning, before any scanning
  begins.

- **`choose_split_point(&mut self, ...) -> Result<Option<ItemKey>, EnumerateError>`**
  -- An optional method for connectors that can suggest natural partition
  boundaries. Connectors that do not support split hints return `Ok(None)`.
  Connectors that declare `split_hints: true` in their capabilities must
  provide a meaningful implementation.

### Read Methods

```rust
// From InMemoryDeterministicConnector (identical signatures on FilesystemConnector)

pub fn open(
    &mut self,
    item_ref: &ItemRef,
    _budgets: Budgets,
) -> Result<Box<dyn io::Read + Send>, ReadError> {
    // ...
}

pub fn read_range(
    &mut self,
    item_ref: &ItemRef,
    offset: u64,
    dst: &mut [u8],
    budgets: Budgets,
) -> Result<usize, ReadError> {
    // ...
}
```

- **`open(&mut self, item_ref, budgets) -> Result<Box<dyn Read + Send>, ReadError>`**
  -- Opens an item for sequential reading. The return type is a boxed trait
  object. This is an intentional design decision: boxing allows the caller to
  work with readers uniformly without knowing the concrete type, and the heap
  allocation is acceptable because `open` sits on the WARM path -- called
  once per item, not once per byte. The IO cost of the subsequent read
  dwarfs the allocation cost.

  The `'static` lifetime requirement on the reader is also intentional. The
  reader must own its resources (file handles, HTTP clients, buffers). It
  cannot borrow from the connector's `&self`, because the caller may hold
  the reader across multiple calls to the connector.

- **`read_range(&mut self, item_ref, offset, dst, budgets) -> Result<usize, ReadError>`**
  -- An optional random-access read path. Connectors without native range
  support return `ReadError::unsupported("range_read")`. The overflow safety
  requirement is explicit: implementations must use
  `offset.checked_add(dst.len() as u64)` before computing the end position.
  This prevents a u64 arithmetic overflow when `offset` is near `u64::MAX`
  and `dst` has nonzero length.

### Why Inherent Methods Instead of Traits

Connector polymorphism does not live at the individual connector level. Each
connector is a concrete struct -- `InMemoryDeterministicConnector`,
`FilesystemConnector`, `GitConnector` -- and the runtime never holds a `dyn
Connector` of any kind.

Instead, polymorphism lives one layer up. The `ScanSourceFactory` trait in
`gossip-scan-driver` maps `Assignment`s to source-specific `ScanDriver`s.
Each factory validates the assignment's `ConnectorKind`, constructs the
appropriate concrete connector internally, and returns a `Box<dyn
ScanDriver>`. The `ScanDriver::run` method receives the scanner engine,
execution config, event output sink, commit sink, and cancellation token --
a richer interface than a method-by-method delegation model could express.

The three factories are:

- **`FilesystemScanSourceFactory`** -- Bridges filesystem assignments to
  `parallel_scan_dir` from the scanner-scheduler crate.
- **`GitScanSourceFactory`** -- Bridges git assignments to `run_git_scan`
  from the scanner-git crate.
- **`InMemoryScanSourceFactory`** -- Bridges in-memory test datasets to a
  sequential commit-sink lifecycle driver.

The shared four-method convention is a structural pattern, not a compiler-
enforced contract. Connector authors follow it by convention; the
`ScanSourceFactory` adapter layer is the point where the compiler enforces
type correctness.

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

## The Execution Path: ScanSourceFactory and ScanDriver

The four inherent methods on each connector are not called directly by the
scan orchestration layer. The execution path goes through `ScanSourceFactory`
and `ScanDriver`, both defined in the `gossip-scan-driver` crate.

The `ScanSourceFactory` trait has a single method:

```rust
/// Factory that maps Assignments to source-specific ScanDrivers.
///
/// The runtime selects a factory based on Assignment::connector_kind,
/// then calls driver_for_assignment to obtain a boxed driver ready
/// to execute.
pub trait ScanSourceFactory: Send {
    fn driver_for_assignment(&self, assignment: &Assignment) -> Result<Box<dyn ScanDriver>>;
}
```

Each factory implementation validates the assignment's `ConnectorKind`,
constructs the appropriate concrete connector, and returns a `Box<dyn
ScanDriver>` that encapsulates the full scan lifecycle. The `ScanDriver`
receives the scanner engine, execution config, event output sink, commit
sink, and cancellation token -- a richer interface than direct method
delegation could express.

The concrete connector's `open()`, `read_range()`, and other methods are
called internally by the `ScanDriver` implementation. Callers of
`ScanDriver::run` never interact with the connector directly. This layering
means the connector surface can remain a structural convention (inherent
methods with matching signatures) while the `ScanSourceFactory` /
`ScanDriver` layer provides the compiler-enforced polymorphism boundary.

---

## Summary

The connector API module defines the runtime boundary between connector
implementations and the scan pipeline. `ErrorClass` forces a binary retry
decision at error construction time, eliminating the ambiguity that caused
the opening failure. `EnumerateError` and `ReadError` are structurally
identical but nominally distinct, preventing accidental cross-assignment
between the planning and read paths. `ConnectorCapabilities` advertises
connector features through four boolean flags with conservative defaults.
Every connector provides the same four inherent methods -- `caps()`,
`choose_split_point()`, `open()`, `read_range()` -- following a shared
convention. Polymorphism lives at the `ScanSourceFactory` / `ScanDriver`
layer, where the runtime selects the appropriate factory based on
`ConnectorKind` and the compiler enforces type correctness through the
`ScanDriver` trait.

Chapter 4 examines the circuit breaker design specification that governs
how error classification feeds into shard lifecycle decisions.
