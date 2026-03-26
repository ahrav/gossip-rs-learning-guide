# Type-Driven Correctness

## The Problem with Primitive Types

Consider a naive identity system using raw byte arrays:

```rust
fn store_finding(
    tenant_id: [u8; 32],
    finding_id: [u8; 32],
    secret: [u8; 32],
) {
    // Which is which? The compiler can't help you.
    storage.insert(finding_id, tenant_id); // BUG: Swapped!
}
```

Even with careful naming, mistakes are easy:

```rust
let tenant = derive_tenant_id(config);
let finding = derive_finding_id(data);

// Accidental mix-up
if tenant == finding { // No compiler error!
    // Logic error: comparing apples to oranges
}
```

The compiler sees all three as `[u8; 32]` and happily accepts any permutation. This is a **type safety hole**: logically distinct concepts are represented by the same type.

## The Newtype Pattern

The **newtype pattern** wraps a primitive type in a nominally distinct struct:

```rust
struct TenantId([u8; 32]);
struct FindingId([u8; 32]);
struct SecretHash([u8; 32]);
```

Now the compiler enforces semantic distinctions:

```rust
fn store_finding(
    tenant_id: TenantId,
    finding_id: FindingId,
    secret_hash: SecretHash,
) {
    storage.insert(finding_id, tenant_id); // Still wrong semantically,
                                           // but types are correct
    storage.insert(tenant_id, finding_id); // Compile error!
    //             ^^^^^^^^^ expected FindingId, found TenantId
}
```

Even though `TenantId` and `FindingId` are both `[u8; 32]` internally, they are **nominally distinct types**. The compiler rejects mixing them.

### Why Not Type Aliases?

A type alias doesn't provide type safety:

```rust
type TenantId = [u8; 32];
type FindingId = [u8; 32];

let tenant: TenantId = [0u8; 32];
let finding: FindingId = tenant; // No error! Type aliases are transparent.
```

Type aliases are purely for documentation. The compiler treats `TenantId` and `FindingId` as interchangeable.

Newtypes, by contrast, are **opaque**:

```rust
struct TenantId([u8; 32]);
struct FindingId([u8; 32]);

let tenant = TenantId([0u8; 32]);
let finding: FindingId = tenant; // Error: mismatched types
```

The compiler sees `TenantId` and `FindingId` as fundamentally different types.

## Gossip-rs Identity Types

Gossip-rs defines 22 identity types using the newtype pattern across three macro families and six hand-rolled types:

**32-byte types via `define_id_32!`** (public constructor):

```rust
pub struct TenantId([u8; 32]);
pub struct PolicyHash([u8; 32]);
pub struct RuleFingerprint([u8; 32]);
pub struct FindingId([u8; 32]);
pub struct OccurrenceId([u8; 32]);
pub struct ObservationId([u8; 32]);
pub struct ConnectorInstanceIdHash([u8; 32]);
pub struct StableItemId([u8; 32]);
pub struct ObjectVersionId([u8; 32]);
```

**32-byte types via `define_id_32_restricted!`** (`pub(crate)` constructor):

```rust
pub struct NormHash([u8; 32]);     // Normalized secret digest
pub struct SecretHash([u8; 32]);   // Tenant-keyed secret identity
```

**64-bit types via `define_id_64!`** (public constructor):

```rust
pub struct RunId(u64);
pub struct ShardId(u64);
pub struct WorkerId(u64);
pub struct OpId(u64);
pub struct JobId(u64);
```

**Hand-rolled types** (domain-specific invariants the macros cannot express):

```rust
pub struct TenantSecretKey([u8; 32]);  // Omits Ord, Hash, CanonicalBytes
pub struct FenceEpoch(u64);            // Panicking increment, INITIAL sentinel
pub struct LogicalTime(u64);           // Checked-add overflow prevention
pub struct ShardKey { run: RunId, shard: ShardId }  // 16-byte compound lookup key
pub struct ConnectorTag([u8; 8]);      // Source-system discriminator
pub struct ItemIdentityKey { connector, connector_instance, locator }  // Variable-width item identity
```

Each type is **structurally identical** to its underlying primitive but **nominally distinct**.

### Example: Preventing Misuse

```rust
// Correct usage
let finding_id = derive_finding_id(&FindingIdInputs {
    tenant,  // &TenantId
    item,    // &StableItemId
    rule,    // &RuleFingerprint
    secret,  // &SecretHash
});

// Incorrect usage (wrong field type)
let finding_id = derive_finding_id(&FindingIdInputs {
    tenant: &stable_item_id, // Error: expected &TenantId, found &StableItemId
    item: &tenant_id,        // Error: expected &StableItemId, found &TenantId
    rule,
    secret,
});
```

The compiler catches the mistake at compile time, not at runtime.

## Restricted Constructors

Some identity types have **restricted constructors** to prevent forgery.

### Public Constructors

Types like `TenantId` have public constructors because external code legitimately needs to create them:

```rust
impl TenantId {
    pub const fn from_bytes(bytes: [u8; 32]) -> Self {
        Self(bytes)
    }
}
```

Anyone with a raw 32-byte value can construct a `TenantId`. In production, this is typically derived from an external tenant identifier (e.g., `blake3(external_tenant_uuid)` to normalize width).

### Restricted Constructors

Types like `SecretHash` have `pub(crate)` constructors because only the designated derivation function should create them:

```rust
pub fn key_secret_hash(key: &TenantSecretKey, norm: &NormHash) -> SecretHash {
    let mut h = Hasher::new_keyed(key.as_bytes());
    h.update(domain::SECRET_HASH_V1.as_bytes());
    h.update(norm.as_bytes());
    SecretHash::from_bytes_internal(finalize_32(&h))
}
```

**Why restricted?**

`SecretHash` requires a `TenantSecretKey` and a `NormHash`. External code shouldn't be able to forge secret hashes by bypassing the keyed hashing mechanism:

```rust
// BAD: If constructor were public
let forged_hash = SecretHash::from_bytes([0u8; 32]); // Arbitrary hash!

// GOOD: Constructor is private, must go through proper derivation
let secret_hash = key_secret_hash(&tenant_key, &norm_hash); // Enforced
```

The `define_id_32_restricted!` macro generates types with `pub(crate)` constructors:

```rust
crate::define_id_32_restricted! {
    /// Secret hash (redacted in debug output).
    SecretHash,
    debug_display = "SecretHash([redacted])"
}
```

This produces:

```rust
pub struct SecretHash([u8; 32]);

impl SecretHash {
    pub(crate) fn from_bytes_internal(bytes: [u8; 32]) -> Self {
        SecretHash(bytes)
    }

    // Public accessors
    pub fn as_bytes(&self) -> &[u8; 32] { &self.0 }
}
```

> **Escape hatch:** `NormHash` uses `define_id_32_restricted!` for its redacted `Debug` impl (preventing accidental logging of security-sensitive material), but adds a separate `pub fn from_digest(bytes: [u8; 32])` constructor. This is intentional — the engine crate is the sole legitimate producer of `NormHash` values and must be able to build them from raw digests.

## Selective Trait Implementation

Not all types should implement all traits. Gossip-rs carefully controls trait implementations to prevent misuse.

### Example: TenantSecretKey

`TenantSecretKey` deliberately **omits** several common traits:

```rust
// Implemented traits
impl Debug for TenantSecretKey { /* redacted output */ }
impl Clone for TenantSecretKey { /* ... */ }
impl PartialEq for TenantSecretKey { /* constant-time comparison */ }

// Deliberately NOT implemented
// impl Ord for TenantSecretKey
// impl Hash for TenantSecretKey
// impl CanonicalBytes for TenantSecretKey
```

**Why the omissions?**

#### No `Ord`

Secret keys must never be ordered:

```rust
// If Ord were implemented, this would be possible
let mut keys = vec![key1, key2, key3];
keys.sort(); // Leaks information via sorting side-channels!
```

Ordering operations can leak information through:
- Timing side-channels (comparison timing)
- Cache side-channels (memory access patterns)
- Sorting algorithms (number of comparisons)

By omitting `Ord`, the compiler prevents ordering operations:

```rust
keys.sort(); // Error: TenantSecretKey doesn't implement Ord
```

#### No `Hash`

Secret keys must never be used as map keys:

```rust
// If Hash were implemented, this would be possible
let mut map = HashMap::new();
map.insert(secret_key, value); // Leaks key via hash function!
```

Hashing a secret key:
- Leaks the key to the hash function's internal state
- May enable hash collision attacks
- Violates the principle of least privilege (maps shouldn't see keys)

By omitting `Hash`, the compiler prevents map insertion:

```rust
map.insert(secret_key, value); // Error: TenantSecretKey doesn't implement Hash
```

#### No `CanonicalBytes`

The `CanonicalBytes` trait is used for hashing into identity derivations. Secret keys must never be hashed into IDs:

```rust
// If CanonicalBytes were implemented, this would be possible
let mut hasher = domain_hasher(FINDING_ID_V1);
tenant_id.write_canonical(&mut hasher);
secret_key.write_canonical(&mut hasher); // BUG: Secret key in finding ID!
```

By omitting `CanonicalBytes`, the compiler prevents accidental inclusion in identity derivations:

```rust
let mut hasher = domain_hasher(FINDING_ID_V1);
tenant_id.write_canonical(&mut hasher);
secret_key.write_canonical(&mut hasher); // Error: no method `write_canonical`
```

### Trait Implementation Matrix

| Type | `Debug` | `Clone` | `PartialEq` | `Ord` | `Hash` | `CanonicalBytes` |
|------|---------|---------|-------------|-------|--------|-----------------|
| `TenantId` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `FindingId` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `SecretHash` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `TenantSecretKey` | ✅ (redacted) | ✅ | ✅ (const-time) | ❌ | ❌ | ❌ |

The asymmetry is intentional: secret keys are special and must be handled with care.

## Typestate Pattern (Preview)

The typestate pattern uses the type system to enforce **state machine protocols**.

In Gossip-rs B5 (Persistence), the `PageCommit<S>` type uses typestate to enforce commit ordering across three durability stages:

```rust
// State 1: AwaitingFindings (no durability confirmed yet)
struct PageCommit<AwaitingFindings> {
    scope: Arc<CommitScope>,
    state: AwaitingFindings,
}

// State 2: FindingsDurable (findings receipt recorded)
struct PageCommit<FindingsDurable> {
    scope: Arc<CommitScope>,
    state: FindingsDurable,  // carries FindingsCommitReceipt
}

// State 3: ItemDurable (findings + done-ledger durable)
struct PageCommit<ItemDurable> {
    scope: Arc<CommitScope>,
    state: ItemDurable,  // carries ItemCommitReceipt
}

// State 4: CheckpointDurable (terminal -- all stages durable)
struct PageCommit<CheckpointDurable> {
    scope: Arc<CommitScope>,
    state: CheckpointDurable,  // carries PageCommitReceipt
}
```

Each state has different available methods:

```rust
impl PageCommit<AwaitingFindings> {
    pub fn record_findings(self, receipt: FindingsCommitReceipt)
        -> PageCommit<FindingsDurable>
    { /* Transition: AwaitingFindings → FindingsDurable */ }
}

impl PageCommit<FindingsDurable> {
    pub fn record_done_ledger(self, receipt: DoneLedgerCommitReceipt)
        -> Result<PageCommit<ItemDurable>, PageCommitValidationError>
    { /* Transition: FindingsDurable → ItemDurable */ }
}

impl PageCommit<ItemDurable> {
    pub fn record_checkpoint(self, receipt: CheckpointCommitReceipt)
        -> Result<PageCommit<CheckpointDurable>, PageCommitValidationError>
    { /* Transition: ItemDurable → CheckpointDurable */ }
}
```

The compiler enforces the protocol:

```rust
let page = PageCommit::new(scope);
let findings_done = page.record_findings(findings_receipt);
let item_done = findings_done.record_done_ledger(dl_receipt)?;
let checkpoint_done = item_done.record_checkpoint(cp_receipt)?;

// Can't skip steps
let page = PageCommit::new(scope);
page.record_checkpoint(cp_receipt); // Error: no method `record_checkpoint` on PageCommit<AwaitingFindings>
```

This is **compile-time enforcement** of a runtime protocol. Invalid state transitions are impossible.

Typestate is covered in depth in Chapter 07-04 (Commit Protocol Typestate). The key insight: **the type system can encode more than just data types--it can encode state machines**.

## Why Macros, Not Generics?

A natural question: why not use generics?

```rust
// Hypothetical generic version
struct Id<Tag>([u8; 32], PhantomData<Tag>);

struct TenantTag;
type TenantId = Id<TenantTag>;

struct FindingTag;
type FindingId = Id<FindingTag>;
```

This provides some type safety but **weakens encapsulation**:

1. **Shared constructor**: All `Id<Tag>` types share the same constructor:

```rust
impl<Tag> Id<Tag> {
    pub fn from_bytes(bytes: [u8; 32]) -> Self {
        Id(bytes, PhantomData)
    }
}

// Now anyone can construct any ID type
let tenant_id = Id::<TenantTag>::from_bytes([0u8; 32]);
let finding_id = Id::<FindingTag>::from_bytes([0u8; 32]);
```

2. **Cross-domain conversion**: Generic parameters can be trivially changed:

```rust
// Convert TenantId to FindingId (BAD!)
let tenant_id: Id<TenantTag> = /* ... */;
let finding_id: Id<FindingTag> = Id(tenant_id.0, PhantomData);
```

3. **Restricted constructors are difficult**: You can't make the constructor `pub(crate)` for some tags but public for others.

### Macros Provide Full Isolation

The `define_id_32!`, `define_id_32_restricted!`, and `define_id_64!` macros generate **separate concrete types**:

```rust
// Each invocation generates a distinct struct
crate::define_id_32! {
    /// Tenant identity
    TenantId
}
crate::define_id_32_restricted! {
    /// Secret hash
    SecretHash,
    debug_display = "SecretHash([redacted])"
}
crate::define_id_64! {
    /// Unique run identifier scoping all shards within a single pipeline execution.
    RunId
}

// Expands to:
pub struct TenantId([u8; 32]);
impl TenantId {
    pub const fn from_bytes(bytes: [u8; 32]) -> Self { TenantId(bytes) }
}

pub struct SecretHash([u8; 32]);
impl SecretHash {
    pub(crate) const fn from_bytes_internal(bytes: [u8; 32]) -> Self { SecretHash(bytes) }
}

pub struct RunId(u64);
impl RunId {
    pub const fn from_raw(raw: u64) -> Self { RunId(raw) }
}
```

Each type has its own constructor with its own visibility. Cross-type conversion requires explicit unsafe code (which triggers review).

**Tradeoff**: More boilerplate, but stronger encapsulation and clearer intent.

## Implementation in Gossip-rs

The identity macros live in `crates/gossip-contracts/src/identity/macros.rs`:

```rust
/// Defines a 32-byte identity type with public constructor
macro_rules! define_id_32 {
    ($(#[$meta:meta])* $name:ident) => {
        $(#[$meta])*
        #[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
        pub struct $name([u8; 32]);

        // Debug shows first 4 bytes in hex: "TypeName(aabbccdd..)"
        impl core::fmt::Debug for $name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                write!(f, concat!(stringify!($name), "({:02x}{:02x}{:02x}{:02x}..)"),
                    self.0[0], self.0[1], self.0[2], self.0[3])
            }
        }

        // Display produces the same output as Debug (interchangeable in tracing spans)
        impl core::fmt::Display for $name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                core::fmt::Debug::fmt(self, f)
            }
        }

        impl $name {
            /// All-zeros sentinel (placeholder for uninitialized records).
            pub const ZERO: Self = Self([0u8; 32]);

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
    };
}

/// Defines a 32-byte identity type with restricted constructor
macro_rules! define_id_32_restricted {
    ($(#[$meta:meta])* $name:ident, debug_display = $debug:expr) => {
        $(#[$meta])*
        #[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
        pub struct $name([u8; 32]);

        // Prints the caller-supplied static string to avoid leaking
        // security-sensitive bytes to logs.
        impl core::fmt::Debug for $name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                write!(f, $debug)
            }
        }

        impl $name {
            // No ZERO constant — restricted types must come from derivation functions.
            pub(crate) const fn from_bytes_internal(bytes: [u8; 32]) -> Self {
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
    };
}

/// Defines a 64-bit identity type with public constructor
macro_rules! define_id_64 {
    ($(#[$meta:meta])* $name:ident) => {
        $(#[$meta])*
        #[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
        pub struct $name(u64);

        // Decimal format: "TypeName(42)"
        impl core::fmt::Debug for $name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                write!(f, concat!(stringify!($name), "({})"), self.0)
            }
        }

        impl core::fmt::Display for $name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                core::fmt::Debug::fmt(self, f)
            }
        }

        impl $name {
            pub const ZERO: Self = Self(0);
            pub const fn from_raw(raw: u64) -> Self { Self(raw) }
            pub const fn as_raw(&self) -> u64 { self.0 }
        }

        impl CanonicalBytes for $name {
            fn write_canonical(&self, h: &mut Hasher) {
                self.0.write_canonical(h);
            }
        }
    };
}
```

Usage:

```rust
crate::define_id_32! {
    /// Identity of a tenant in the system.
    TenantId
}
crate::define_id_32! {
    /// Identity of a detected secret.
    FindingId
}
crate::define_id_32_restricted! {
    /// Content hash of a secret.
    SecretHash,
    debug_display = "SecretHash([redacted])"
}
crate::define_id_64! {
    /// Unique run identifier scoping all shards within a single pipeline execution.
    RunId
}
crate::define_id_64! {
    /// Shard identity within a run.
    ShardId
}
```

The macros provide:
- Consistent implementation across all identity types
- Controlled trait implementation
- Restricted visibility where needed
- Generated documentation

### Additional Identity Infrastructure

Beyond the macro-generated types, the identity subsystem includes several important components not yet covered in detail:

- **`IdentityInputError`**: Fallible constructors (e.g., `ConnectorTag::from_ascii`) return this error when inputs are invalid (empty, too long, non-ASCII). See `crates/gossip-contracts/src/identity/item.rs`.
- **`finalize_64`**: Truncates a BLAKE3 hash to 64 bits (used for coordination IDs like `ShardId` derivation during splits). See `crates/gossip-contracts/src/identity/hashing.rs`.
- **`IdHashMode`**: Enum (`KeyedV1`) that tags the hashing strategy used for identity derivation, enabling forward-compatible policy versioning. See `crates/gossip-contracts/src/identity/policy.rs`.
- **`PolicyHashInputs` / `compute_policy_hash`**: Derives a `PolicyHash` from rule configuration inputs (policy version, hash mode, evidence version, rules digest). See `crates/gossip-contracts/src/identity/policy.rs`.
- **`ShardId::is_derived()`**: Checks bit 63 to distinguish root shards (externally assigned) from split-derived shards. See `crates/gossip-contracts/src/identity/coordination.rs`.
- **`PooledShardSpec` / `PooledCursor` / `PooledSpawned`**: Slab-backed wrappers that store shard spec fields, cursor data, and lineage records in pre-allocated `ByteSlab` arenas — eliminating per-operation heap allocation on the coordination hot path. See `crates/gossip-contracts/src/coordination/pooled.rs`.

These are covered in depth in later boundary chapters (B1 Identity, B2 Coordination).

## Key Takeaways

1. **The newtype pattern** wraps primitives in nominally distinct types for type safety
2. **Gossip-rs defines 22 identity types** across three macro families (`define_id_32!`, `define_id_32_restricted!`, `define_id_64!`) plus six hand-rolled types
3. **Restricted constructors** (`pub(crate)`) prevent external code from forging hashes
4. **Selective trait implementation** prevents misuse (e.g., `TenantSecretKey` omits `Ord`, `Hash`, `CanonicalBytes`)
5. **Macros over generics** provide stronger encapsulation and per-type constructor visibility
6. **Typestate (preview)** uses the type system to enforce state machine protocols at compile time

## References

- [Rust Design Patterns - Newtype](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html)
- [Typestate Pattern in Rust](http://cliffle.com/blog/rust-typestate/)
- Gossip-rs source: `crates/gossip-contracts/src/identity/macros.rs`
- Gossip-rs source: `crates/gossip-contracts/src/identity/types.rs`

---

**Previous**: [Domain Separation](03-domain-separation.md)
**Next**: [Invariant-First Design](05-invariant-first-design.md) - How Gossip-rs ensures correctness through exhaustive invariant testing.
