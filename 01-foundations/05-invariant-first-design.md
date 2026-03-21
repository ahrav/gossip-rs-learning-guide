# Invariant-First Design

## What Are Invariants?

An **invariant** is a property that must always hold true at certain points in a program's execution. Invariants are the **contracts** that define correctness.

### Safety Invariants

**Safety invariants** define "what must never happen":

- A secret must never be reported as scanned if not fully scanned
- A TenantSecretKey must never be hashed into an identity derivation
- A finding must never have a SecretHash from a different tenant
- A hash must never collide across domain boundaries

Violating a safety invariant means the system is in an **incorrect state**.

### Liveness Invariants

**Liveness invariants** define "what must eventually happen":

- Every scan job must eventually complete or fail
- Every finding must eventually be reported or discarded
- Every page write must eventually be committed or rolled back

Violating a liveness invariant means the system is **stuck** (deadlocked, livelocked, or starved).

### Alpern & Schneider: Safety and Liveness

In their seminal 1985 paper "Defining Liveness", Alpern & Schneider proved that **every system property can be expressed as the intersection of a safety property and a liveness property**.

- **Safety**: "Nothing bad ever happens"
- **Liveness**: "Something good eventually happens"

This decomposition is fundamental to reasoning about distributed systems.

## Tiger Style: Invariant-Driven Development

The **Tiger Style** methodology, pioneered by TigerBeetle, advocates for **asserting invariants at every mutation site**, not just at boundaries.

Traditional approach:

```rust
// Validate at API boundary
pub fn create_finding(...) {
    validate_inputs(...);
    let finding = Finding { ... };
    self.store.insert(finding); // Hope the invariants hold
}
```

Tiger Style approach:

```rust
// Assert invariants at every mutation
pub fn create_finding(...) {
    validate_inputs(...);
    let finding = Finding { ... };

    // Invariant 1: Finding ID must match derived ID
    assert_eq!(finding.id, derive_finding_id(&FindingIdInputs { ... }));

    // Invariant 2: SecretHash must belong to tenant
    assert_eq!(finding.secret_hash.tenant_id(), finding.tenant_id);

    // Invariant 3: Finding must not already exist
    assert!(!self.store.contains_key(&finding.id));

    self.store.insert(finding);

    // Post-condition: Finding is now retrievable
    assert_eq!(self.store.get(&finding.id), Some(&finding));
}
```

**Benefits**:

1. **Early detection**: Invariant violations are caught immediately, not after propagating through the system
2. **Localized debugging**: The assertion points directly to the violation site
3. **Living documentation**: Assertions document the invariants in the code itself
4. **Regression prevention**: Once an invariant is asserted, it's checked on every execution

### TigerBeetle's Influence

TigerBeetle is a distributed financial ledger that prioritizes **correctness above all else**. Key practices:

- **Exhaustive assertions**: Every state transition asserts all relevant invariants
- **Deterministic simulation testing**: Run the same test millions of times with injected faults
- **Formal verification**: Prove invariants hold under all possible interleavings
- **No undefined behavior**: Use a memory-safe language (Zig) with explicit safety checks

Gossip-rs adapts these principles to Rust:

- Use Rust's type system to prevent invariant violations at compile time
- Assert runtime invariants at every mutation site
- Use property-based testing (proptest) to explore the input space
- Use golden vectors to detect regression

## The Invariant-to-Test Matrix

Gossip-rs B1 (Identity) maintains an **invariant-to-test matrix**: every invariant maps to one or more tests.

### Example: Finding ID Invariants

**Invariants**:

1. **Determinism**: `derive_finding_id(&inputs)` always produces the same output
2. **Collision-freedom**: Different inputs produce different outputs (with negligible collision probability)
3. **Domain separation**: FindingId and TenantId never collide, even with identical input
4. **Field sensitivity**: Changing any component (tenant, item, rule, secret) changes the output

**Tests**:

```rust
#[test]
fn finding_id_is_deterministic() {
    let inputs = FindingIdInputs { tenant, item, rule, secret };
    let id1 = derive_finding_id(&inputs);
    let id2 = derive_finding_id(&inputs);
    assert_eq!(id1, id2); // Invariant 1
}

#[test]
fn finding_id_is_collision_free() {
    let id1 = derive_finding_id(&FindingIdInputs { tenant: tenant1, item, rule, secret });
    let id2 = derive_finding_id(&FindingIdInputs { tenant: tenant2, item, rule, secret });
    assert_ne!(id1, id2); // Invariant 2
}

#[test]
fn finding_id_has_domain_separation() {
    let finding_id = derive_finding_id(&FindingIdInputs { tenant, item, rule, secret });
    let tenant_id = TenantId::from_bytes(bytes);
    assert_ne!(
        finding_id.as_bytes(),
        tenant_id.as_bytes()
    ); // Invariant 3
}

proptest! {
    #[test]
    fn finding_id_is_field_sensitive(
        tenant: TenantId,
        item: StableItemId,
        rule: RuleFingerprint,
        secret: SecretHash,
    ) {
        let inputs = FindingIdInputs { tenant, item, rule, secret };
        let id_original = derive_finding_id(&inputs);

        // Modify each field
        let id_tenant = derive_finding_id(&FindingIdInputs { tenant: tweak(tenant), ..inputs });
        let id_item = derive_finding_id(&FindingIdInputs { item: tweak(item), ..inputs });
        let id_rule = derive_finding_id(&FindingIdInputs { rule: tweak(rule), ..inputs });
        let id_secret = derive_finding_id(&FindingIdInputs { secret: tweak(secret), ..inputs });

        // All must be different (Invariant 4)
        assert_ne!(id_original, id_tenant);
        assert_ne!(id_original, id_item);
        assert_ne!(id_original, id_rule);
        assert_ne!(id_original, id_secret);
    }
}
```

This matrix ensures **no invariant is untested** and **no test is redundant**.

### Gossip-rs B1 Invariant Summary

Gossip-rs B1 (Identity) has approximately **37 tested invariants** across its source files (these counts are approximate and shift as the codebase evolves):

| Source File | Invariants | Test Types |
|------------|-----------|-----------|
| `types.rs` | 4 | Unit, Proptest |
| `finding.rs` | 6 | Unit, Proptest, Golden |
| `item.rs` | 4 | Unit, Proptest |
| `domain.rs` | 5 | Unit, Compile-time |
| `hashing.rs` | 2 | Unit, Golden |
| `canonical.rs` | 2 | Unit |
| `coordination.rs` | 3 | Unit, Golden |
| `golden.rs` | 3 | Golden |
| `macros.rs` | 2 | Compile-time |
| `policy.rs` | 3 | Unit, Golden |
| `mod.rs` | 3 | Unit |

These counts are approximate snapshots; the exact numbers grow as the identity module evolves.

## Property-Based Testing with Proptest

Traditional unit tests test **specific examples**:

```rust
#[test]
fn test_hash_determinism() {
    let input = b"specific input";
    let hash1 = hash(input);
    let hash2 = hash(input);
    assert_eq!(hash1, hash2);
}
```

This tests determinism for **one input**. But what about:
- Empty input?
- Very large input?
- Binary data?
- Extreme values?

**Property-based testing** tests **universal properties** by generating random inputs:

```rust
proptest! {
    #[test]
    fn test_hash_determinism_property(input: Vec<u8>) {
        let hash1 = hash(&input);
        let hash2 = hash(&input);
        assert_eq!(hash1, hash2); // Must hold for ALL inputs
    }
}
```

Proptest generates hundreds or thousands of test cases, including edge cases you wouldn't think to write manually.

### Example: Secret Hash Invariants

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn secret_hash_is_deterministic(
        tenant_key: TenantSecretKey,
        norm: NormHash,
    ) {
        let hash1 = key_secret_hash(&tenant_key, &norm);
        let hash2 = key_secret_hash(&tenant_key, &norm);
        prop_assert_eq!(hash1, hash2);
    }

    #[test]
    fn secret_hash_is_tenant_scoped(
        tenant_key1: TenantSecretKey,
        tenant_key2: TenantSecretKey,
        norm: NormHash,
    ) {
        prop_assume!(tenant_key1 != tenant_key2); // Different tenants

        let hash1 = key_secret_hash(&tenant_key1, &norm);
        let hash2 = key_secret_hash(&tenant_key2, &norm);

        prop_assert_ne!(hash1, hash2); // Same secret, different hashes
    }

    #[test]
    fn secret_hash_is_content_sensitive(
        tenant_key: TenantSecretKey,
        norm1: NormHash,
        norm2: NormHash,
    ) {
        prop_assume!(norm1 != norm2); // Different secrets

        let hash1 = key_secret_hash(&tenant_key, &norm1);
        let hash2 = key_secret_hash(&tenant_key, &norm2);

        prop_assert_ne!(hash1, hash2); // Different secrets, different hashes
    }
}
```

Proptest automatically generates:
- Empty secrets
- Single-byte secrets
- Large secrets (megabytes)
- Binary secrets (non-UTF8)
- Identical secrets (for collision testing)

Each property test runs 256 times by default (configurable).

### Shrinking: Minimal Failing Examples

When a proptest fails, it **shrinks** the input to find the **minimal failing case**:

```
thread 'test_secret_hash_is_content_sensitive' panicked at 'assertion failed'
shrinking to find minimal failing case...
input 1: [0, 0]
input 2: [0, 1]
```

This dramatically simplifies debugging: instead of a 10KB random input, you get a 2-byte minimal reproducer.

## The Golden Vector Pattern

A **golden vector** is a test that pins the exact byte output of a hash function. If the output changes, the test fails.

```rust
#[test]
fn golden_vector_finding_id() {
    let tenant_id = TenantId::from_bytes([0x11; 32]);
    let stable_item_id = StableItemId::from_bytes([0x22; 32]);
    let rule_hash = RuleFingerprint::from_bytes([0x33; 32]);
    let secret_hash = SecretHash::from_bytes_internal([0x44; 32]);

    let finding_id = derive_finding_id(&FindingIdInputs {
        tenant: tenant_id,
        item: stable_item_id,
        rule: rule_hash,
        secret: secret_hash,
    });

    // Actual golden vector from crates/gossip-contracts/src/identity/golden.rs.
    // These bytes are the real BLAKE3 output for the inputs above.
    #[rustfmt::skip]
    let expected: [u8; 32] = [
        0x81, 0xCE, 0x28, 0x85, 0xDF, 0x85, 0xE9, 0x63,
        0x0D, 0x46, 0xA0, 0x29, 0x59, 0xE9, 0x3A, 0xCF,
        0x8A, 0xCE, 0x8E, 0x79, 0x01, 0x6E, 0xE9, 0xD8,
        0x73, 0xC1, 0x09, 0xBB, 0x20, 0x76, 0x1F, 0xD8,
    ];

    assert_eq!(finding_id.as_bytes(), &expected);
}
```

**Purpose**:

1. **Regression detection**: If the hash function changes (bug, refactor, algorithm change), the golden vector fails
2. **Cross-language compatibility**: Golden vectors can be shared between implementations (Rust, Python, Go)
3. **API stability**: Changing a hash breaks all existing IDs, so golden vectors make this explicit

**When to update a golden vector**:

- **Never** without justification
- **Only** when intentionally changing the hash function
- **Requires** a version bump (e.g., `FINDING_ID_V1` → `FINDING_ID_V2`)

Golden vectors are **regression tests**: they ensure the past doesn't break.

### Example: Domain Constant Golden Vectors

```rust
#[test]
fn golden_vectors_for_domain_constants() {
    // These exact strings must never change without a version bump
    assert_eq!(FINDING_ID_V1, "gossip/finding/v1");
    assert_eq!(SECRET_HASH_V1, "gossip/secret-hash/v1");
    assert_eq!(ITEM_ID_V1, "gossip/item-id/v1");
    assert_eq!(SPLIT_ID_V1, "gossip/coord/v1/split-id");
    // ... (16 total)
}
```

If someone accidentally changes `"gossip/finding/v1"` to `"gossip/finding/v2"`, the golden vector fails immediately.

## Compile-Time Invariant Enforcement

Some invariants can be enforced at compile time using Rust's type system and const evaluation.

### Example: Domain Constant Count

```rust
pub const ALL: [&str; 16] = [
    SPLIT_ID_V1,
    OP_PAYLOAD_V1,
    // ... (16 total)
];
```

If you add a 17th domain constant but forget to add it to the array, **compilation fails**:

```
error: mismatched types
  expected array `[&str; 16]`
  found array `[&str; 17]`
```

No runtime check needed - the compiler enforces the invariant.

### Example: Typestate Protocol

The typestate pattern (from Chapter 4) enforces state machine protocols at compile time:

```rust
let intent = PageCommit::new(page_id, data);
let committed = intent.commit(&mut log); // Error: can't skip `write` step
```

The invariant "must write before committing" is enforced by the type system.

## Invariant Test Matrix Summary

Here's a summary of Gossip-rs B1's invariants (approximate counts; these grow as the codebase evolves):

### Identity Derivation (20 invariants)

| Invariant | Test Type | Files |
|-----------|-----------|-------|
| Determinism | Unit, Proptest | All ID types |
| Collision-freedom | Proptest | All ID types |
| Domain separation | Unit | All ID types |
| Field sensitivity | Proptest | FindingId, PolicyHash |
| Content sensitivity | Proptest | SecretHash, RuleFingerprint |

### Domain Registry (5 invariants)

| Invariant | Test Type | File |
|-----------|-----------|------|
| No duplicate values | Unit | `domain.rs` |
| No duplicate names | Unit | `domain.rs` |
| Naming convention | Unit | `domain.rs` |
| Complete coverage | Unit | `domain.rs` |
| Array length matches count | Compile-time | `domain.rs` |

### Secret Key Security (4 invariants)

| Invariant | Test Type | File |
|-----------|-----------|------|
| No ordering | Compile-time (no `Ord` impl) | `types.rs` |
| No hashing | Compile-time (no `Hash` impl) | `types.rs` |
| No ID derivation | Compile-time (no `CanonicalBytes` impl) | `types.rs` |
| Constant-time comparison | Unit | `types.rs` |

### Golden Vectors (8 invariants)

| Invariant | Test Type | Files |
|-----------|-----------|-------|
| Exact byte output | Unit (golden) | FindingId, SecretHash, RuleFingerprint, PolicyHash, hashing |
| Domain string stability | Unit (golden) | `domain.rs` |

These counts are approximate and evolve with the codebase.

## Lamport: Proving Correctness

In "Proving the Correctness of Multiprocess Programs" (1977), Leslie Lamport established the foundations of formal verification:

1. **Specify the invariants**: Define what must always be true
2. **Prove initialization**: Show that invariants hold initially
3. **Prove preservation**: Show that every operation preserves the invariants
4. **Conclude correctness**: If invariants always hold, the system is correct

Gossip-rs follows this methodology:

1. **Specify**: Document invariants in code comments and this guide
2. **Prove initialization**: Test constructors establish invariants
3. **Prove preservation**: Test every mutation preserves invariants
4. **Conclude correctness**: Comprehensive test cases verify all documented invariants

This is **lightweight formal verification**: we don't use a proof assistant, but we use the same reasoning principles.

## Key Takeaways

1. **Invariants** define correctness: safety ("nothing bad happens") and liveness ("something good happens")
2. **Tiger Style** asserts invariants at every mutation site, not just boundaries
3. **Invariant-to-test matrix** ensures every invariant is tested and every test verifies an invariant
4. **Property-based testing** (proptest) tests universal properties across the input space
5. **Golden vectors** pin exact byte outputs as regression tests
6. **Compile-time invariants** use Rust's type system to prevent violations at compile time
7. **Gossip-rs B1 has ~37 invariants** tested comprehensively across its source files (counts are approximate and grow with the codebase)
8. **Lamport's methodology** guides formal reasoning about correctness

## References

- [Alpern & Schneider, "Defining Liveness" (1985)](https://www.cs.cornell.edu/fbs/publications/DefLiveness.pdf)
- [Lamport, "Proving the Correctness of Multiprocess Programs" (1977)](https://lamport.azurewebsites.net/pubs/proving.pdf)
- [TigerBeetle Safety Documentation](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/DESIGN.md#safety)
- [Proptest Documentation](https://altsysrq.github.io/proptest-book/intro.html)
- Gossip-rs source: `crates/gossip-contracts/src/identity/` (all test files)

---

**Previous**: [Type-Driven Correctness](04-type-driven-correctness.md)
**Next**: [Boundary 1: Identity Spine](../02-boundary-1-identity-spine/) - How Gossip-rs enforces architectural boundaries.
