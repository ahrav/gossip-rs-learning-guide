# Appendix A: Rust Patterns Used in Gossip-rs

This appendix catalogs the Rust patterns and idioms used throughout Gossip-rs, with brief explanations and pointers to where they appear in the codebase.

## 1. Newtype Pattern

**What it is**: Wrapping a primitive type in a struct to create a distinct type.

**Why use it**: Type safety. Prevents mixing incompatible types that have the same underlying representation.

**Example**:
```rust
pub struct TenantId([u8; 32]);
pub struct RunId(u64);
pub struct ShardId(u64);

// These are distinct types at compile time:
let tenant = TenantId([0x00; 32]);
let key = ShardKey::new(RunId::from_raw(1), ShardId::from_raw(1));

// Compile error: cannot use TenantId where ShardKey is expected
coordinator.acquire_and_restore_into(now, key, tenant, worker, &mut scratch);  // Error!
coordinator.acquire_and_restore_into(now, tenant, key, worker, &mut scratch);  // OK
```

**Where used in Gossip-rs**:
- All identity types: `TenantId`, `StableItemId`, `FindingId`, `OccurrenceId`, `PolicyHash`, `NormHash`, `SecretHash`
- Coordination types: `RunId`, `ShardId`, `WorkerId`, `FenceEpoch`
- Connector types: `ConnectorTag`, `ItemKey`, `ObjectVersionId`

**Benefits**:
- Impossible to pass wrong type to a function (compile error)
- Self-documenting: `FindingId` is more descriptive than `[u8; 32]`
- Encapsulation: Can add methods to newtype without polluting the primitive type

**Reference**: Chapter 02-04 (Identity Type Definitions)

## 2. Typestate Pattern

**What it is**: Encoding state transitions in the type system. Each state is a distinct type.

**Why use it**: Prevents invalid state transitions at compile time.

**Example** (design-stage — not yet implemented in crate, shown as the intended pattern):
```rust
pub struct PageCommit<S> {
    findings: Vec<Finding>,
    _state: PhantomData<S>,
}

pub struct Accumulating;
pub struct Sealed;
pub struct Committed;

impl PageCommit<Accumulating> {
    pub fn add_finding(&mut self, finding: Finding) {
        self.findings.push(finding);
    }

    pub fn seal(self, cursor: Cursor) -> PageCommit<Sealed> {
        // Transition: Accumulating → Sealed
        PageCommit {
            findings: self.findings,
            cursor: Some(cursor),
            _state: PhantomData,
        }
    }
}

impl PageCommit<Sealed> {
    // Cannot add findings (method not available)

    pub fn commit(self, backend: &Backend) -> PageCommit<Committed> {
        backend.write(self.findings);
        // Transition: Sealed → Committed
        PageCommit {
            findings: vec![],  // Clear findings
            _state: PhantomData,
        }
    }
}
```

**Where used in Gossip-rs**:
- Persistence: `PageCommit<S>` typestate is described in `persistence/mod.rs` doc comments but is **design-stage** (not yet implemented as a concrete struct)

**Benefits**:
- Cannot commit unsealed page (compile error)
- Cannot add findings after sealing (compile error)
- Self-documenting: type signature shows valid operations

**Reference**: Chapter 07 (Persistence and Page Commits)

## 3. Builder Pattern (Pedagogical)

**What it is**: Constructing complex objects step by step, validating as you go.

> **Note**: Gossip-rs currently uses validated constructors (e.g., `RunConfig::try_new`)
> rather than builder structs. This example illustrates the pattern conceptually;
> the types shown are pedagogical pseudocode, not real crate types.

**Example** (pedagogical pseudocode — not in codebase):
```rust
pub struct WidgetConfigBuilder {
    name: Option<String>,
    max_retries: Option<u32>,
    timeout: Option<Duration>,
}

impl WidgetConfigBuilder {
    pub fn new() -> Self { /* ... */ }

    pub fn name(mut self, name: String) -> Self {
        self.name = Some(name);
        self
    }

    pub fn max_retries(mut self, n: u32) -> Self {
        self.max_retries = Some(n);
        self
    }

    pub fn build(self) -> Result<WidgetConfig, ConfigError> {
        Ok(WidgetConfig {
            name: self.name.ok_or(ConfigError::MissingName)?,
            max_retries: self.max_retries.unwrap_or(3),
            // ...
        })
    }
}

// Usage:
let config = WidgetConfigBuilder::new()
    .name("scanner".into())
    .max_retries(5)
    .build()?;
```

**Current codebase approach**: `RunConfig::try_new(cursor_semantics, lease_duration, max_shard_retries)` validates inputs in a single constructor call rather than using a builder.

**Benefits**:
- Fluent API (readable)
- Validates at build time
- Can enforce required vs optional fields

## 4. Declarative Macros

**What it is**: Using `macro_rules!` to generate repetitive code.

**Why use it**: Reduce boilerplate, ensure consistency, enable nominal types.

**Example**:
```rust
macro_rules! define_id_32 {
    ($name:ident) => {
        #[derive(Clone, Copy, PartialEq, Eq, Hash)]
        pub struct $name([u8; 32]);

        impl $name {
            pub const fn from_bytes(bytes: [u8; 32]) -> Self {
                Self(bytes)
            }

            pub const fn as_bytes(&self) -> &[u8; 32] {
                &self.0
            }
        }

        impl CanonicalBytes for $name {
            fn write_canonical(&self, hasher: &mut Hasher) {
                hasher.update(&self.0);
            }
        }

        impl Debug for $name {
            fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
                write!(f, "{}({})", stringify!($name), hex::encode(&self.0[..8]))
            }
        }
    };
}

// Usage:
define_id_32!(TenantId);       // 32-byte content-addressed IDs
define_id_32!(FindingId);
define_id_32!(PolicyHash);

// For coordination IDs (u64-based):
macro_rules! define_id_64 {
    ($name:ident) => {
        #[derive(Clone, Copy, PartialEq, Eq, Hash)]
        pub struct $name(u64);
        // ... from_raw(), as_raw(), ZERO sentinel, Debug, CanonicalBytes
    };
}

define_id_64!(RunId);          // u64-based coordination IDs
define_id_64!(ShardId);
define_id_64!(WorkerId);
```

**Where used in Gossip-rs**:
- `define_id_32!` for all 32-byte content-addressed identity types (e.g., `TenantId`, `FindingId`, `PolicyHash`, `StableItemId`)
- `define_id_32_restricted!` for derivation-only types with restricted constructors (e.g., `NormHash`, `SecretHash`)
- `define_id_64!` for u64-based coordination types (e.g., `RunId`, `ShardId`, `WorkerId`, `OpId`, `JobId`)
- Generated traits: `Clone`, `PartialEq`, `Eq`, `Hash`, `CanonicalBytes`, `Debug`

**Why not generics?**: Generics create structural types (`Id<Tenant>`, `Id<Run>`). Macros create **nominal types** (`TenantId`, `RunId`). Nominal types prevent mixing (compile error if you pass `RunId` where `TenantId` expected).

**Reference**: Chapter 02-08 (Identity Macro Architecture)

## 5. Feature Flags

**What it is**: Conditional compilation based on feature flags.

**Why use it**: Include test infrastructure only in test builds, not production.

**Example**:
```rust
#[cfg(feature = "test-support")]
pub mod test_helpers {
    pub fn create_test_tenant() -> TenantId {
        TenantId([0xAA; 32])
    }
}

#[cfg(test)]
mod tests {
    use super::test_helpers::*;

    #[test]
    fn test_something() {
        let tenant = create_test_tenant();
        // ...
    }
}
```

**Where used in Gossip-rs**:
- `#[cfg(feature = "test-support")]` gates test doubles (DeterministicConnector, mock backends)
- `#[cfg(test)]` gates unit tests

**Benefits**:
- Production builds exclude test code → smaller binaries
- Test code can use patterns not allowed in production (e.g., `unsafe`, panics)

**Reference**: Throughout codebase, especially connector implementations

## 6. Proptest Strategies

**What it is**: Generating random test inputs with property-based testing.

**Why use it**: Test properties that must hold for ALL inputs, not just specific examples.

**Example**:
```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn finding_id_is_pure(
        tenant in any::<[u8; 32]>(),
        item in any::<[u8; 32]>(),
        rule in any::<[u8; 32]>(),
        secret in any::<[u8; 32]>(),
    ) {
        let inputs = FindingIdInputs {
            tenant: TenantId(tenant),
            item: StableItemId(item),
            rule: RuleFingerprint(rule),
            secret: SecretHash::from_bytes_internal(secret),
        };

        let id1 = derive_finding_id(&inputs);
        let id2 = derive_finding_id(&inputs);

        prop_assert_eq!(id1, id2);  // Purity: f(x) == f(x)
    }
}
```

**Where used in Gossip-rs**:
- Identity derivation tests: purity, collision-freedom, field sensitivity
- Coordination tests: lease acquisition, cursor monotonicity
- Shard algebra tests: range splitting, key encoding

**Benefits**:
- Tests universal properties (not just specific examples)
- Proptest automatically shrinks failing inputs to minimal case
- High confidence in correctness

**Reference**: Chapter 02-02 (Hash Derivation Testing), Appendix C (Property-Based Testing Guide)

## 7. LazyLock Statics

**What it is**: Thread-safe lazy initialization of static variables.

**Why use it**: Avoid expensive initialization in hot paths. Initialize once, reuse forever.

**Example** (from `hashing.rs`):
```rust
use std::sync::LazyLock;

pub(crate) static FINDING_HASHER: LazyLock<Hasher> =
    LazyLock::new(|| Hasher::new_derive_key(domain::FINDING_ID_V1));

pub(crate) fn derive_from_cached<T: CanonicalBytes>(base: &Hasher, inputs: &T) -> [u8; 32] {
    let mut hasher = base.clone();  // Clone the cached hasher
    inputs.write_canonical(&mut hasher);
    hasher.finalize().into()
}
```

**Where used in Gossip-rs**:
- All identity derivation functions cache pre-initialized BLAKE3 hashers
- Hasher is initialized once per process, cloned for each derivation

**Why clone is cheap**: `blake3::Hasher` implements `Clone` efficiently. Cloning copies the internal state (256 bytes), not re-running the key schedule.

**Benefits**:
- Faster derivation (no key schedule per call)
- Thread-safe (LazyLock handles synchronization)
- Zero runtime overhead after first initialization

**Reference**: Chapter 01-01 (Hash Derivation Internals), Appendix B (BLAKE3 Internals)

## 8. Constant-Time Comparison

**What it is**: Comparing secrets in constant time to prevent timing side-channels.

**Why use it**: Variable-time comparison leaks information through timing.

**Example**:
```rust
use subtle::{ConstantTimeEq, Choice};

impl ConstantTimeEq for TenantSecretKey {
    fn ct_eq(&self, other: &Self) -> Choice {
        self.0.ct_eq(&other.0)
    }
}

impl PartialEq for TenantSecretKey {
    fn eq(&self, other: &Self) -> bool {
        self.ct_eq(other).into()
    }
}
```

**Where used in Gossip-rs**:
- `TenantSecretKey` comparison (prevents timing leaks)

**How it works**: `ct_eq` always examines all 32 bytes, regardless of where the first difference occurs. Execution time is constant.

**Benefits**:
- Prevents timing side-channel attacks
- Security property enforced by type system (cannot accidentally use `==`)

**Reference**: Chapter 08-04 (Tenant Isolation)

## 9. Repr(u8) Enums

**What it is**: Enums with explicit discriminants for stable wire encoding.

**Why use it**: Ensure discriminant values don't change across compiler versions.

**Example**:
```rust
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum IdHashMode {
    Unkeyed = 0,
    KeyedV1 = 1,
}

impl IdHashMode {
    pub fn as_u8(self) -> u8 {
        self as u8
    }

    pub fn from_u8(v: u8) -> Option<Self> {
        match v {
            0 => Some(Self::Unkeyed),
            1 => Some(Self::KeyedV1),
            _ => None,
        }
    }
}
```

**Where used in Gossip-rs**:
- `IdHashMode` (Unkeyed vs KeyedV1)
- Shard state enums (Active, Done, Split, Parked)

**Benefits**:
- Stable wire format (0 always means Unkeyed, 1 always means KeyedV1)
- Explicit mapping (not relying on compiler's automatic discriminant assignment)
- Forward compatibility (can add new discriminants without breaking old data)

**Reference**: Chapter 02-02 (Hash Derivation Implementation)

## 10. Error Enums with thiserror

**What it is**: Enums for error types with derive-generated `Display` and `Error` impls.

**Why use it**: Type-safe error handling, human-readable messages with less boilerplate than hand-written impls.

**Example**:
```rust
#[derive(Debug, Clone, PartialEq, Eq, thiserror::Error)]
pub enum IdentityInputError {
    #[error("ConnectorTag must not be empty")]
    EmptyTag,
    #[error("ConnectorTag must be at most 8 bytes, got {0}")]
    TagTooLong(usize),
    #[error("ConnectorTag byte at index {index} is not ASCII graphic: 0x{byte:02X}")]
    NonGraphicByte {
        index: usize,
        byte: u8,
    },
    #[error("ItemIdentityKey locator must not be empty")]
    EmptyLocator,
    #[error("version bytes must not be empty")]
    EmptyVersionBytes,
}
```

The `thiserror::Error` derive generates `Display` and `Error` trait impls from the `#[error("...")]` attributes. Variants with `#[from]` get automatic `From` conversions for error propagation.

**Where used in Gossip-rs**:
- `IdentityInputError` (identity derivation errors)
- `GitScanError` (git scan pipeline errors with `#[from]` conversions)
- `CoordError` (coordination layer errors)
- `ConnectorError` (connector errors)

**Benefits**:
- Type-safe error handling (cannot confuse different error types)
- Human-readable error messages (`Display` impl)
- Composable (`?` operator works seamlessly)

**Reference**: Chapter 02-01 (Identity Module Overview)

## 11. PhantomData for Typestate

**What it is**: Zero-size marker type to carry type information without runtime cost.

**Why use it**: Implement typestate pattern without memory overhead.

**Example**:
```rust
use std::marker::PhantomData;

pub struct PageCommit<S> {
    findings: Vec<Finding>,
    _state: PhantomData<S>,  // Zero-size, compile-time only
}

// PhantomData<S> is zero bytes:
assert_eq!(std::mem::size_of::<PhantomData<Accumulating>>(), 0);
```

**Where used in Gossip-rs**:
- Typestate pattern in persistence layer

**Benefits**:
- Type-level state machine (compile-time checks)
- Zero runtime cost (optimized away)
- Self-documenting (type signature shows valid states)

**Reference**: Chapter 07 (Persistence and Page Commits)

## 12. Const Constructors

**What it is**: Functions that can be evaluated at compile time.

**Why use it**: Create constants without runtime overhead.

**Example**:
```rust
impl TenantId {
    pub const fn from_bytes(bytes: [u8; 32]) -> Self {
        Self(bytes)
    }
}

// Can be used in const context:
const TEST_TENANT: TenantId = TenantId::from_bytes([0xAA; 32]);
```

**Where used in Gossip-rs**:
- All identity type constructors are `const fn`
- Test helpers use `const` for test constants

**Benefits**:
- Zero runtime overhead (computed at compile time)
- Can use in `static` declarations
- Clearer intent (this is a pure constructor)

**Reference**: Chapter 02-04 (Identity Type Definitions)

## 13. Trait-Based Abstraction

**What it is**: Using traits to define contracts without specifying implementation.

**Why use it**: Enables dependency injection, testing, multiple backends.

**Example**:
```rust
// CoordinationFacade is a super-trait combining three sub-traits:
pub trait CoordinationFacade: CoordinationBackend + RunManagement + ShardClaiming {}

pub trait CoordinationBackend {
    fn acquire_and_restore_into<'a>(&mut self, now: LogicalTime, tenant: TenantId, key: ShardKey, worker: WorkerId, out: &'a mut AcquireScratch) -> Result<AcquireResultView<'a>, AcquireError>;
    fn checkpoint(&mut self, now: LogicalTime, tenant: TenantId, lease: &Lease, new_cursor: &CursorUpdate<'_>, op_id: OpId) -> Result<IdempotentOutcome<()>, CheckpointError>;
    // ... other methods
}

pub struct InMemoryCoordinator { /* ... */ }
impl CoordinationBackend for InMemoryCoordinator { /* ... */ }
// Blanket impl: any type implementing all three component traits gets CoordinationFacade for free.
impl<T: CoordinationBackend + RunManagement + ShardClaiming> CoordinationFacade for T {}
```

**Where used in Gossip-rs**:
- `CoordinationFacade` / `CoordinationBackend` (in-memory vs persistent)
- `CommitSink` (`CliNoOpCommitSink`) — per-item commit lifecycle, defined in `gossip-scanner-runtime`
- `DoneLedger` (design-stage — planned as a standalone trait; current persistence uses `CommitSink`)

**Benefits**:
- Test with in-memory backend, deploy with Postgres backend
- Swap implementations without changing caller code
- Clear contract (trait defines required operations)

**Reference**: Chapter 04 (Coordination), Chapter 06 (Connectors), Chapter 07 (Persistence)

## 14. derive(Clone, Copy) for Small Types

**What it is**: Automatically implementing `Clone` and `Copy` for types.

**Why use it**: Enable pass-by-value for small types (performance, ergonomics).

**Example**:
```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
pub struct TenantId([u8; 32]);

// Can copy implicitly:
fn foo(tenant: TenantId) { /* ... */ }
fn bar(tenant: TenantId) { /* ... */ }

let tenant = TenantId([0x00; 32]);
foo(tenant);  // Copied
bar(tenant);  // Copied again (still valid)
```

**Where used in Gossip-rs**:
- All identity types (32 bytes → cheap to copy)
- Coordination types (`RunId`, `ShardId`, `FenceEpoch`)

**Why safe**: Types are 32 bytes or smaller, no heap allocation, no interior mutability.

**Benefits**:
- Ergonomics (no `.clone()` calls)
- Performance (copy is just memcpy, often optimized to register moves)

**Reference**: Chapter 02-04 (Identity Type Definitions)

## 15. Redacted Debug

**What it is**: Custom `Debug` implementation that hides sensitive data.

**Why use it**: Prevent accidental leaks in logs, error messages, panic messages.

**Example**:
```rust
impl Debug for TenantSecretKey {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.write_str("TenantSecretKey([redacted])")
    }
}

// In logs:
println!("{:?}", tenant_key);
// Output: TenantSecretKey([redacted])
// NOT: TenantSecretKey([0x11, 0x22, 0x33, ...])
```

**Where used in Gossip-rs**:
- `TenantSecretKey` (always redacted)

**Benefits**:
- Prevents accidental secret leaks in logs
- Safe to use `{:?}` in error messages
- Explicit: anyone reading code knows this is secret

**Reference**: Chapter 08-04 (Tenant Isolation)

## Summary

Gossip-rs uses 15 Rust patterns strategically:

| Pattern | Primary Benefit | Example Types |
|---------|-----------------|---------------|
| Newtype | Type safety | `TenantId`, `FindingId`, all IDs |
| Typestate | Compile-time state machine | `PageCommit<S>` |
| Builder | Fluent construction | (pedagogical example) |
| Macros | Reduce boilerplate | `define_id_32!`, `define_id_64!` |
| Feature flags | Test/prod separation | `#[cfg(feature = "test-support")]` |
| Proptest | Universal properties | All identity tests |
| LazyLock | Cached initialization | BLAKE3 hashers |
| Constant-time comparison | Timing attack prevention | `TenantSecretKey` |
| Repr(u8) | Stable encoding | `IdHashMode` |
| Error enums | Type-safe errors | `IdentityInputError` |
| PhantomData | Zero-cost typestate | `PageCommit<S>` |
| Const constructors | Compile-time constants | `TenantId::from_bytes` |
| Trait abstraction | Swappable backends | `CoordinationFacade` |
| derive(Copy) | Ergonomic pass-by-value | All 32-byte types |
| Redacted Debug | Secret leak prevention | `TenantSecretKey` |

These patterns combine to create a codebase that is:
- **Type-safe**: Invalid operations are compile errors
- **Performant**: Zero-cost abstractions, cached initialization
- **Testable**: Trait-based backends, property-based testing
- **Secure**: Constant-time comparison, redacted debug, explicit types

**Next**: [Appendix B: BLAKE3 Internals](./B-blake3-internals.md) explains how BLAKE3 enables domain separation.
