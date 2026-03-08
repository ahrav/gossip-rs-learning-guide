# "Enumerate, Then Read" -- The Runtime Connector Traits

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
sharing the same GitHub App installation is now throttled. Shard enumeration
across 340 repositories stalls for 38 minutes while the rate-limit window
resets. A single miscategorized error -- `Retryable` instead of `Permanent`
-- cascades into a full-source outage.*

---

## Why Connector Traits Exist

The failure above has a root cause that no amount of retry logic can fix:
the error classification was wrong at the source. The connector told the
runtime "try again" when the correct signal was "stop trying." The system
needs a contract that forces connector authors to make this decision
explicitly at error-construction time, with no escape hatch for ambiguity.

The `gossip-contracts::connector::api` module defines this contract. It
provides two traits (`EnumerationConnector` and `ReadConnector`), an error
taxonomy (`ErrorClass`, `EnumerateError`, `ReadError`), a capability
declaration struct (`ConnectorCapabilities`), and a convenience supertrait
(`ConnectorInstance`). Together, these types form the runtime boundary
between connector implementations and the scan pipeline that drives them.

Chapter 2 defined the value types (`ScanItem`, `EnumerationPage`, `Budgets`,
`Cursor`) that flow through these traits. This chapter defines the traits
themselves -- the behavioral contracts that connector authors must satisfy.

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

Before the runtime calls any trait method, it needs to know what the
connector supports. `ConnectorCapabilities` is a plain struct of four
boolean flags, declared at registration time.

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
  implemented. When `false`, the default `read_range()` implementation
  returns `ReadError::unsupported("range_read")`. The runtime must fall back
  to reading entire items via `open()`.

- **`split_hints: bool`** -- Controls whether the connector can suggest
  split points for dynamic re-sharding. When `true`, the
  `choose_split_point()` method must be overridden (enforced by a
  `debug_assert!` in the default implementation).

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

## EnumerationConnector -- The Listing Contract

The first of the two runtime traits handles metadata traversal: listing
items within a shard range, page by page.

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

Three methods, one required, two with defaults:

- **`caps(&self) -> ConnectorCapabilities`** -- Returns the static feature
  profile. Takes `&self` (not `&mut self`) because capability queries must
  not mutate connector state. Called during planning, before any enumeration
  begins.

- **`enumerate_page(&mut self, ...) -> Result<EnumerationPage, EnumerateError>`**
  -- The core operation. Takes a `ShardSpec` (the key range to enumerate), a
  `Cursor` (the resumption point), and `Budgets` (advisory limits). Returns
  an `EnumerationPage` containing items and a continuation cursor, or an
  `EnumerateError`. The `&mut self` receiver allows connectors to maintain
  internal state (connection pools, pagination tokens, rate-limit trackers).

- **`choose_split_point(&mut self, ...) -> Result<Option<ItemKey>, EnumerateError>`**
  -- An optional method for connectors that can suggest natural partition
  boundaries. The default returns `Ok(None)`. The `debug_assert!` in the
  default body catches a specific misconfiguration: if a connector declares
  `split_hints: true` in its capabilities but forgets to override this
  method, the assertion fires in debug builds.

The `Send` bound on the trait enables connector instances to be transferred
between threads. The orchestration layer may move a connector to a worker
thread for parallel shard processing.

---

## ReadConnector -- The Content Access Contract

The second trait handles item content retrieval. It is intentionally separate
from `EnumerationConnector` because listing metadata and reading content
have fundamentally different cost profiles.

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

Two methods, one required:

- **`open(&mut self, item_ref, budgets) -> Result<Box<dyn Read + Send>, ReadError>`**
  -- Opens an item for sequential reading. The return type is a boxed trait
  object. This is an intentional design decision documented in the doc
  comment: boxing preserves object safety (callers can use `dyn
  ReadConnector` without knowing the concrete reader type), and the heap
  allocation is acceptable because `open` sits on the WARM path -- called
  once per item, not once per byte. The IO cost of the subsequent read
  dwarfs the allocation cost.

  The `'static` lifetime requirement on the reader is also intentional. The
  reader must own its resources (file handles, HTTP clients, buffers). It
  cannot borrow from the connector's `&self`, because the caller may hold
  the reader across multiple calls to the connector.

- **`read_range(&mut self, item_ref, offset, dst, budgets) -> Result<usize, ReadError>`**
  -- An optional random-access read path. The default returns
  `ReadError::unsupported("range_read")`. The doc comment explicitly calls
  out an overflow safety requirement: implementations must use
  `offset.checked_add(dst.len() as u64)` before computing the end position.
  This prevents a u64 arithmetic overflow when `offset` is near `u64::MAX`
  and `dst` has nonzero length.

---

## ConnectorInstance -- The Pure Bound Alias

When the runtime needs both enumeration and read capabilities from the same
connector, it uses the `ConnectorInstance` supertrait.

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

The blanket impl means any type that implements both `EnumerationConnector`
and `ReadConnector` automatically satisfies `ConnectorInstance`. Connector
authors never write `impl ConnectorInstance for MyConnector` -- it is
derived for free.

The `?Sized` bound on the blanket impl is important: it enables `dyn
ConnectorInstance` to work. Without `?Sized`, the blanket impl would only
cover `Sized` types, and trait objects (which are unsized) would not satisfy
the bound.

The doc comment explains why `ConnectorInstance` must remain empty: adding
a required method would break the blanket impl, since the blanket cannot
provide a body for arbitrary `T`. New capabilities belong on
`EnumerationConnector` or `ReadConnector` directly, where each connector can
provide its own implementation.

The test suite verifies all three trait objects are object-safe and `Send`:

```rust
#[test]
fn traits_are_object_safe_and_send() {
    fn assert_obj_safe_send<T: Send + ?Sized>() {}
    assert_obj_safe_send::<dyn EnumerationConnector>();
    assert_obj_safe_send::<dyn ReadConnector>();
    assert_obj_safe_send::<dyn ConnectorInstance>();
}
```

---

## Deprecation Note and the ScanDriver Execution Model

The actual source code at `api.rs:344-354` marks both `EnumerationConnector`
and `ReadConnector` with a deprecation notice:

> Deprecated surface: new unified execution paths should use
> `gossip_scan_driver::ScanSourceFactory` and `gossip_scan_driver::ScanDriver`.
> This trait remains supported for existing Phase III connector callers.

The new execution model works through adapter factories in
`crates/gossip-connectors/src/scan_driver.rs`. Each concrete connector type
has a corresponding `ScanSourceFactory` implementation:

- **`FilesystemScanSourceFactory`** -- Bridges filesystem assignments to
  `parallel_scan_dir` from the scanner-scheduler crate.
- **`GitScanSourceFactory`** -- Bridges git assignments to `run_git_scan`
  from the scanner-git crate.
- **`InMemoryScanSourceFactory`** -- Bridges in-memory test datasets to a
  sequential commit-sink lifecycle driver.

Each factory implements `ScanSourceFactory::driver_for_assignment`, which
validates the assignment's `ConnectorKind` and source payload, then returns
a `Box<dyn ScanDriver>`. The `ScanDriver::run` method receives the scanner
engine, execution config, event output sink, commit sink, and cancellation
token -- a richer interface than the page-by-page enumeration model of
`EnumerationConnector`.

The `EnumerationConnector` and `ReadConnector` traits remain the foundation
for the connector contract: they define enumeration pages, cursor
progression, and content access. The `ScanDriver` adapters consume these
traits internally but present a higher-level interface to the scan
orchestration layer. Chapter 10 covers the scan-driver adapters in detail.

---

## Summary

The connector API module defines the runtime boundary between connector
implementations and the scan pipeline. `ErrorClass` forces a binary retry
decision at error construction time, eliminating the ambiguity that caused
the opening failure. `EnumerateError` and `ReadError` are structurally
identical but nominally distinct, preventing accidental cross-assignment
between the enumeration and read paths. `ConnectorCapabilities` advertises
connector features through four boolean flags with conservative defaults.
`EnumerationConnector` and `ReadConnector` define the behavioral contracts,
with default implementations and `debug_assert!` guards for capability
consistency. `ConnectorInstance` is a pure bound alias that composes both
traits through a blanket impl.

Chapter 4 introduces the page validator that checks the structural
correctness of every page these traits produce.
