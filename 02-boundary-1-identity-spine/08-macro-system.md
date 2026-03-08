# The Macro System

## Why Macros Instead of Generics?

**The alternative (generic struct):**
```rust
pub struct Id<Tag>([u8; 32], PhantomData<Tag>);

// Usage:
type TenantId = Id<TenantTag>;
type FindingId = Id<FindingTag>;
```

**Problem:** A single `Id<Tag>` generic shares a single type constructor. Accidentally converting between ID domains becomes trivial:

```rust
let tenant_id: TenantId = Id::from_bytes([0x42; 32]);
let finding_id: FindingId = tenant_id;  // ❌ Compiles! Type mismatch not caught.
```

**The macro approach:**

Each macro invocation generates a **separate concrete struct**:
```rust
pub struct TenantId([u8; 32]);
pub struct FindingId([u8; 32]);
```

These are **nominally distinct** — the compiler rejects `TenantId` where `FindingId` is expected:

```rust
let tenant_id = TenantId::from_bytes([0x42; 32]);
let finding_id: FindingId = tenant_id;  // ✅ Compile error: expected FindingId, found TenantId
```

**Zero runtime cost:** No `PhantomData`, no vtables, no indirection. The type safety is entirely compile-time.

## define_id_32! — Public Construction

**Source:** `macros.rs:74-133`

Generates a `[u8; 32]` newtype with:
- `#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]`
- `Debug` printing `TypeName(aabbccdd..)` (first 4 bytes, hex)
- `Display` impl that delegates to `Debug` (so `tracing` span fields using `%` or `?` are interchangeable)
- `ZERO` constant (all-zeros sentinel)
- `from_bytes` (pub const) — public constructor
- `as_bytes` (pub const) — borrow inner array
- `CanonicalBytes` impl (fixed-width, no length prefix)

**Usage:**
```rust
crate::define_id_32! {
    /// A test identity type.
    TestId
}

let id = TestId::from_bytes([0xAB; 32]);
assert_eq!(id.as_bytes(), &[0xAB; 32]);
assert_eq!(format!("{:?}", id), "TestId(abababab..)");
assert_eq!(TestId::ZERO.as_bytes(), &[0u8; 32]);
```

**Traits derived:**
- `Clone`, `Copy` — Cheap copying (32 bytes is small)
- `PartialEq`, `Eq` — Equality comparison
- `PartialOrd`, `Ord` — Ordering (for B-trees, sorted collections)
- `Hash` — Hash maps, hash sets
- `Display` — Delegates to `Debug` (so `tracing` `%` and `?` fields are interchangeable)
- `CanonicalBytes` — Content-addressed derivation

**Debug implementation** (lines 85-93):
```rust
impl ::core::fmt::Debug for $name {
    fn fmt(&self, f: &mut ::core::fmt::Formatter<'_>) -> ::core::fmt::Result {
        write!(
            f,
            concat!(stringify!($name), "({:02x}{:02x}{:02x}{:02x}..)"),
            self.0[0], self.0[1], self.0[2], self.0[3]
        )
    }
}
```

Shows only the first 4 bytes to keep log output compact while still giving enough entropy to distinguish instances.

**CanonicalBytes implementation** (lines 124-132):
```rust
impl $crate::identity::CanonicalBytes for $name {
    // Fixed-width: 32 bytes are written directly with no length prefix.
    #[inline]
    fn write_canonical(&self, h: &mut $crate::blake3::Hasher) {
        h.update(&self.0);
    }
}
```

## define_canonical_input! — Input Struct Generation

**Source:** `macros.rs:226-256`

Generates a fixed-field input struct whose `CanonicalBytes` encoding follows the declared field order. Used for identity/policy input structs such as `FindingIdInputs`, `OccurrenceIdInputs`, `ObservationIdInputs`, and `PolicyHashInputs`.

**Produces:**
- `#[derive(Clone, Copy, Debug, PartialEq, Eq)]`
- `CanonicalBytes` impl that writes fields in **struct declaration order** (reordering fields is a breaking change)

**Usage:**
```rust
super::macros::define_canonical_input! {
    pub struct FindingIdInputs {
        pub tenant: TenantId,
        pub item: StableItemId,
        pub rule: RuleFingerprint,
        pub secret: SecretHash,
    }
}
```

**Key property:** The `CanonicalBytes` impl is generated from the struct declaration, so hash order follows declaration order automatically. This is an internal helper (`pub(crate)` use) that avoids the need for a proc-macro crate while keeping the struct definition and `CanonicalBytes` impl in one place.

## define_id_32_restricted! — Private Construction

**Source:** `macros.rs:163-210`

Generates the same API as `define_id_32!` except:
- Inner field is private (not `pub`)
- Constructor is `from_bytes_internal` with `pub(crate)` visibility
- No `ZERO` constant (restricted types must come from derivation functions)
- `Debug` prints the caller-supplied string (typically `"TypeName([redacted])"`)

**Purpose:** Enforce construction-safety. Values that *must* be produced by a specific derivation function (e.g., a keyed hash) cannot be forged by supplying arbitrary bytes.

**Usage:**
```rust
crate::define_id_32_restricted! {
    /// A restricted test type.
    TestRestricted,
    debug_display = "TestRestricted([redacted])"
}

// Only crate-internal code can construct:
let id = TestRestricted::from_bytes_internal([0xFF; 32]);

// Debug is redacted:
assert_eq!(format!("{:?}", id), "TestRestricted([redacted])");
```

**Debug implementation** (lines 176-180):
```rust
impl ::core::fmt::Debug for $name {
    fn fmt(&self, f: &mut ::core::fmt::Formatter<'_>) -> ::core::fmt::Result {
        write!(f, $debug)
    }
}
```

Emits the caller-supplied static string to avoid leaking security-sensitive bytes (e.g., hash outputs) to logs.

**Real-world examples:**
- `NormHash` (restricted) — Secret digest, must come from detection engine
- `SecretHash` (restricted) — Must come from `key_secret_hash` function

## define_id_64! — 64-Bit Coordination IDs

**Source:** `macros.rs:300-351`

Generates a `u64` newtype with:
- `#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]`
- `Debug` printing `TypeName(decimal)` (decimal is more useful for sequential IDs)
- `Display` impl that delegates to `Debug` (so `tracing` span fields using `%` or `?` are interchangeable)
- `ZERO` constant (zero sentinel)
- `from_raw` (pub const) — public constructor
- `as_raw` (pub const) — return inner u64
- `CanonicalBytes` impl (little-endian 8-byte encoding, no length prefix)

**Purpose:** Coordination types use `u64` rather than the `[u8; 32]` width used by content-addressed types. These are counters, assigned identifiers, or lookup keys with bounded cardinality per run. The 64-bit width fits in a single register, halves HashMap key size, and packs 8 values per cache line.

**Usage:**
```rust
crate::define_id_64! {
    /// A test 64-bit identity type.
    TestId64
}

let id = TestId64::from_raw(42);
assert_eq!(id.as_raw(), 42);
assert_eq!(format!("{:?}", id), "TestId64(42)");
assert_eq!(TestId64::ZERO.as_raw(), 0);
```

**Real-world examples:** `RunId`, `ShardId`, `WorkerId`, `OpId`, `JobId` (all in `coordination.rs`)

## Fully-Qualified Paths

All macros use fully-qualified paths so they work from any invocation site:

```rust
// Instead of:
use blake3::Hasher;
use crate::identity::CanonicalBytes;

// The macros use:
$crate::blake3::Hasher
$crate::identity::CanonicalBytes
::core::fmt::Debug
::core::mem::size_of
```

**Why:** Macros expand at the call site. If a consumer crate (e.g., `gossip-engine`) invokes `define_id_32!`, the macro body must resolve paths relative to `gossip-contracts`, not the consumer crate. `$crate` expands to the crate that defines the macro.

## Test-Support Macros

Three additional macros exist for **consumer-crate integration tests**:

### smoke_test_id_32! — Import Canary for Public IDs

**Source:** `macros.rs:246-275`

**Purpose:** Act as an import canary. If a type is removed from the public API or its layout changes, the downstream crate's build breaks immediately.

**What it generates:**
1. A `const _: ()` block with compile-time `size_of` assertions per type (fires during `cargo check`)
2. `#[test] fn id_types_are_32_bytes()` — Runtime confirmation
3. `#[test] fn zero_sentinels_are_zeroed()` — Runtime check of `T::ZERO`

**Constraint:** **Invoke at most once per scope.** The generated test function names are fixed (`id_types_are_32_bytes`, `zero_sentinels_are_zeroed`).

**Usage:**
```rust
// In gossip-engine/tests/identity_smoke.rs:
gossip_contracts::smoke_test_id_32!(FindingId, OccurrenceId, RuleFingerprint);
```

**Generated code:**
```rust
// Compile-time guard (fires during `cargo check`):
const _: () = {
    ::core::assert!(::core::mem::size_of::<FindingId>() == 32);
    ::core::assert!(::core::mem::size_of::<OccurrenceId>() == 32);
    ::core::assert!(::core::mem::size_of::<RuleFingerprint>() == 32);
};

#[test]
fn id_types_are_32_bytes() {
    assert_eq!(::core::mem::size_of::<FindingId>(), 32, "FindingId is not 32 bytes");
    // ... (repeated for all types)
}

#[test]
fn zero_sentinels_are_zeroed() {
    let z = FindingId::ZERO;
    assert_eq!(*z.as_bytes(), [0u8; 32], "FindingId::ZERO is not all-zero");
    // ... (repeated for all types)
}
```

### smoke_test_id_64! — Import Canary for 64-Bit IDs

**Source:** `macros.rs:382-411`

**Purpose:** The 64-bit counterpart of `smoke_test_id_32!`. Consumer crates invoke this in their `tests/identity_smoke.rs` to catch layout or API changes in downstream builds.

**What it generates:**
1. A `const _: ()` block with compile-time `size_of == 8` assertions per type
2. `#[test] fn id_64_types_are_8_bytes()` — Runtime confirmation
3. `#[test] fn id_64_zero_sentinels_are_zeroed()` — Runtime check of `T::ZERO.as_raw() == 0`

**Constraint:** **Invoke at most once per scope.** The generated test function names are fixed.

**Usage:**
```rust
gossip_contracts::smoke_test_id_64!(RunId, ShardId, WorkerId, OpId, JobId);
```

### smoke_test_id_size! — Compile-Time Size Assertion

**Source:** `macros.rs:434-441`

**Purpose:** Compile-time size assertion for types that lack a `ZERO` sentinel (restricted types, non-32-byte types).

**What it generates:**
- A `const _: ()` block with a `size_of` assertion

**Multiple invocations safe:** Each produces an independent `const _` block.

**Usage:**
```rust
// In gossip-engine/tests/identity_smoke.rs:
gossip_contracts::smoke_test_id_size!(ConnectorTag, 8);
gossip_contracts::smoke_test_id_size!(NormHash, 32);
gossip_contracts::smoke_test_id_size!(SecretHash, 32);
```

**Generated code:**
```rust
const _: () = {
    ::core::assert!(::core::mem::size_of::<ConnectorTag>() == 8);
};
const _: () = {
    ::core::assert!(::core::mem::size_of::<NormHash>() == 32);
};
const _: () = {
    ::core::assert!(::core::mem::size_of::<SecretHash>() == 32);
};
```

## Property Tests for Macros

**Source:** `macros.rs:443-703`

The macro system has its own test suite (using test-only types `PubTestId` and `RestrictedTestId`).

**Test coverage:**

| Test Category | Coverage |
|---------------|----------|
| Construction | `from_bytes` roundtrip, `from_bytes_internal` roundtrip |
| Debug | Hex prefix, no full-value leak, redaction for restricted types |
| Traits | Compile-time trait assertions, equality, ordering, hashing |
| CanonicalBytes | Fixed-width (no prefix), stability (purity), collision-freedom |

**Example test** (`macros.rs:465-470`):
```rust
#[test]
fn pub_from_bytes_roundtrip() {
    let bytes = [0xAB; 32];
    let id = PubTestId::from_bytes(bytes);
    assert_eq!(*id.as_bytes(), bytes);
}
```

**Property test** (`macros.rs:651-661`):
```rust
proptest! {
    #[test]
    fn pub_canonical_bytes_stable(bytes in uniform32(any::<u8>())) {
        let id = PubTestId::from_bytes(bytes);
        let mut h1 = Hasher::new();
        let mut h2 = Hasher::new();
        id.write_canonical(&mut h1);
        id.write_canonical(&mut h2);
        prop_assert_eq!(h1.finalize(), h2.finalize());
    }
}
```

## Macro Comparison Table

| Feature | `define_id_32!` | `define_id_32_restricted!` | `define_id_64!` |
|---------|----------------|---------------------------|-----------------|
| Width | `[u8; 32]` | `[u8; 32]` | `u64` |
| Constructor visibility | `pub` | `pub(crate)` | `pub` |
| Constructor name | `from_bytes` | `from_bytes_internal` | `from_raw` |
| `ZERO` constant | Yes | No | Yes |
| Debug output | Hex prefix (first 4 bytes) | Caller-supplied redacted string | Decimal value |
| Display impl | Delegates to Debug | None | Delegates to Debug |
| Use case | Content-addressed IDs | Derivation-only IDs | Coordination IDs |
| Examples | `TenantId`, `FindingId`, `RuleFingerprint` | `NormHash`, `SecretHash` | `RunId`, `ShardId`, `WorkerId` |

## Design Rationale

**Why separate concrete structs win over a generic:**

| Criterion | Generic `Id<Tag>` | Separate structs (macros) |
|-----------|-------------------|---------------------------|
| **Type safety** | ❌ Easy to accidentally convert between domains | ✅ Compiler enforces domain separation |
| **Error messages** | ❌ "expected `Id<TenantTag>`, found `Id<FindingTag>`" (confusing) | ✅ "expected `TenantId`, found `FindingId`" (clear) |
| **API ergonomics** | ❌ `Id::<TenantTag>::from_bytes` (turbofish required) | ✅ `TenantId::from_bytes` (no turbofish) |
| **Code size** | ✅ Single generic impl | ❌ One impl per invocation (mitigated by inlining) |
| **Compilation time** | ✅ One monomorphization | ❌ One monomorphization per invocation |

**Verdict:** Type safety and API clarity outweigh the compilation-time cost. The macros are invoked 16 times across the identity and coordination modules (9 public 32-byte, 2 restricted 32-byte, 5 coordination 64-bit); the compile-time impact is negligible.

## Summary

| Macro | Purpose | Generated Traits | Constructor Visibility |
|-------|---------|------------------|------------------------|
| `define_id_32!` | Public 32-byte IDs | `Clone`, `Copy`, `Eq`, `Ord`, `Hash`, `Display`, `CanonicalBytes` | `pub` |
| `define_id_32_restricted!` | Derivation-only 32-byte IDs | `Clone`, `Copy`, `Eq`, `Ord`, `Hash`, `CanonicalBytes` | `pub(crate)` |
| `define_id_64!` | Public 64-bit coordination IDs | `Clone`, `Copy`, `Eq`, `Ord`, `Hash`, `Display`, `CanonicalBytes` | `pub` |
| `define_canonical_input!` | Fixed-field input structs | `Clone`, `Copy`, `Debug`, `Eq`, `CanonicalBytes` | N/A (struct fields) |
| `smoke_test_id_32!` | Consumer-crate import canary (32-byte) | N/A | N/A |
| `smoke_test_id_64!` | Consumer-crate import canary (64-bit) | N/A | N/A |
| `smoke_test_id_size!` | Compile-time size assertion | N/A | N/A |

**Key takeaway:** Macros eliminate boilerplate while enforcing nominal type safety. Each ID type is a separate struct, preventing accidental cross-domain conversion.

**Next chapter:** Golden Vectors and Testing (`golden.rs`) -- the 57-invariant test matrix.
