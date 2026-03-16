# Appendix C: Property-Based Testing Guide

Property-based testing (PBT) is a technique for testing universal properties that must hold for ALL inputs, not just specific examples. This appendix explains how Gossip-rs uses proptest to verify correctness.

## What Is Property-Based Testing?

### Traditional Example-Based Testing

```rust
#[test]
fn finding_id_is_pure_example() {
    let inputs = FindingIdInputs {
        tenant: TenantId([0xAA; 32]),
        item: StableItemId([0xBB; 32]),
        rule: RuleFingerprint([0xCC; 32]),
        secret: SecretHash::from_bytes_internal([0xDD; 32]),
    };

    let id1 = derive_finding_id(&inputs);
    let id2 = derive_finding_id(&inputs);

    assert_eq!(id1, id2);  // Passes for THIS input
}
```

**Problem**: Passes for one specific input. What about:
- All zeros: `TenantId([0x00; 32])`?
- All 0xFF: `TenantId([0xFF; 32])`?
- Random bytes: `TenantId([0x1A, 0x2B, 0x3C, ...])`?

**Traditional solution**: Write 100 test cases. Still only covers 100 out of 2^768 possible inputs (768 bits = 32+32+32 bytes).

### Property-Based Testing

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

        prop_assert_eq!(id1, id2);  // Checks 256 random inputs (default)
    }
}
```

**Benefits**:
- Tests 256 random inputs (configurable)
- Explores edge cases (all zeros, all 0xFF, mixed patterns)
- If it fails, proptest **shrinks** the input to the minimal failing case

**Example shrinking**:
```
Failed: [0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF]
Shrinking...
Minimal failing input: [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x01]
```

## Common Properties Tested in Gossip-rs

### 1. Purity (Determinism)

**Property**: `f(x) == f(x)` — calling the function twice with the same input produces the same output.

**Why it matters**: Identity derivation must be deterministic (same secret → same FindingId across runs).

**Example**:
```rust
proptest! {
    #[test]
    fn norm_hash_roundtrip_is_stable(
        digest in any::<[u8; 32]>(),
    ) {
        // NormHash is externally computed by the detection engine;
        // the contracts crate receives it via from_digest().
        let hash1 = NormHash::from_digest(digest);
        let hash2 = NormHash::from_digest(digest);

        prop_assert_eq!(hash1, hash2);
    }
}
```

**Note**: `NormHash` has no `derive` method or `NormHashInputs` struct. It is constructed via `NormHash::from_digest(bytes: [u8; 32])`, receiving a pre-computed 32-byte digest from the detection engine. The contracts crate treats it as an opaque digest.

**Failure scenario**: If `from_digest` did not deterministically wrap the same bytes, downstream `SecretHash` derivation would be non-deterministic.

### 2. Collision-Freedom

**Property**: `x ≠ y → f(x) ≠ f(y)` — distinct inputs produce distinct outputs (no collisions).

**Why it matters**: Different secrets must produce different FindingIds (otherwise, triage grouping breaks).

**Example**:
```rust
proptest! {
    #[test]
    fn different_secrets_different_finding_ids(
        tenant in any::<[u8; 32]>(),
        item in any::<[u8; 32]>(),
        rule in any::<[u8; 32]>(),
        secret1 in any::<[u8; 32]>(),
        secret2 in any::<[u8; 32]>(),
    ) {
        prop_assume!(secret1 != secret2);  // Require distinct inputs

        let inputs1 = FindingIdInputs {
            tenant: TenantId(tenant),
            item: StableItemId(item),
            rule: RuleFingerprint(rule),
            secret: SecretHash::from_bytes_internal(secret1),
        };
        let inputs2 = FindingIdInputs {
            tenant: TenantId(tenant),
            item: StableItemId(item),
            rule: RuleFingerprint(rule),
            secret: SecretHash::from_bytes_internal(secret2),
        };

        let id1 = derive_finding_id(&inputs1);
        let id2 = derive_finding_id(&inputs2);

        prop_assert_ne!(id1, id2);
    }
}
```

**Caveat**: This tests *probabilistic* collision-freedom. True collision-freedom requires `2^256` tests (infeasible). We test 256 random pairs and rely on cryptographic hash properties for the rest.

### 3. Field Sensitivity

**Property**: Changing any single input field changes the output.

**Why it matters**: Every field must contribute to the derived identity (otherwise, field is ignored, which is a bug).

**Example**:
```rust
proptest! {
    #[test]
    fn finding_id_sensitive_to_tenant(
        tenant1 in any::<[u8; 32]>(),
        tenant2 in any::<[u8; 32]>(),
        item in any::<[u8; 32]>(),
        rule in any::<[u8; 32]>(),
        secret in any::<[u8; 32]>(),
    ) {
        prop_assume!(tenant1 != tenant2);  // Require different tenants

        let inputs1 = FindingIdInputs {
            tenant: TenantId(tenant1),
            item: StableItemId(item),
            rule: RuleFingerprint(rule),
            secret: SecretHash::from_bytes_internal(secret),
        };
        let inputs2 = FindingIdInputs {
            tenant: TenantId(tenant2),  // Only tenant differs
            item: StableItemId(item),
            rule: RuleFingerprint(rule),
            secret: SecretHash::from_bytes_internal(secret),
        };

        let id1 = derive_finding_id(&inputs1);
        let id2 = derive_finding_id(&inputs2);

        prop_assert_ne!(id1, id2);  // Output must differ
    }
}
```

**What this catches**: If derivation accidentally used a hardcoded tenant ID instead of the input parameter, this test would fail.

### 4. Roundtrip (Serialization/Deserialization)

**Property**: `decode(encode(x)) == x` — serialization is lossless.

**Why it matters**: Identity types must serialize to bytes for storage and deserialize back without loss.

**Example**:
```rust
proptest! {
    #[test]
    fn finding_id_canonical_bytes_stable(bytes in any::<[u8; 32]>()) {
        let original = FindingId(bytes);

        let mut h1 = blake3::Hasher::new();
        original.write_canonical(&mut h1);
        let hash1 = h1.finalize();

        let mut h2 = blake3::Hasher::new();
        original.write_canonical(&mut h2);
        let hash2 = h2.finalize();

        prop_assert_eq!(hash1, hash2);
    }
}
```

**What this catches**: If `write_canonical` wrote different bytes on repeated calls, the hash would differ.

### 5. Field-Order Independence (For Commutative Inputs)

**Property**: For inputs where order doesn't matter semantically, order shouldn't affect encoding.

**Example** (hypothetical for a set of tags):
```rust
proptest! {
    #[test]
    fn tag_set_order_independent(
        tag1 in any::<String>(),
        tag2 in any::<String>(),
    ) {
        let set1 = TagSet::from_vec(vec![tag1.clone(), tag2.clone()]);
        let set2 = TagSet::from_vec(vec![tag2, tag1]);

        // Canonical encoding should be identical (sorted order)
        let mut h1 = blake3::Hasher::new();
        set1.write_canonical(&mut h1);
        let mut h2 = blake3::Hasher::new();
        set2.write_canonical(&mut h2);
        prop_assert_eq!(h1.finalize(), h2.finalize());
    }
}
```

**What this catches**: If `TagSet` naively encoded tags in insertion order, this test would fail.

### 6. Monotonicity

**Property**: For ordered types, `x < y → f(x) < f(y)`.

**Example** (for shard ranges):
```rust
proptest! {
    #[test]
    fn shard_ranges_monotonic(
        start in any::<[u8; 32]>(),
        mid in any::<[u8; 32]>(),
        end in any::<[u8; 32]>(),
    ) {
        prop_assume!(start < mid && mid < end);

        let range1 = ShardRange::new(start, mid);
        let range2 = ShardRange::new(mid, end);

        // Split at midpoint should produce two non-overlapping ranges
        prop_assert!(range1.end() <= range2.start());
    }
}
```

**What this catches**: Off-by-one errors in range splitting (overlapping ranges).

## Proptest Strategies for Gossip-rs Types

### Strategy 1: Fixed-Size Arrays

```rust
use proptest::array::uniform32;

// Generate random 32-byte arrays
let strategy = uniform32(any::<u8>());

proptest! {
    #[test]
    fn test_with_32_bytes(bytes in uniform32(any::<u8>())) {
        let tenant = TenantId(bytes);
        // ...
    }
}
```

**Why not `any::<[u8; 32]>()`?**: `any` is a shorthand, but `uniform32` is more explicit and readable.

### Strategy 2: Variable-Length Vectors

```rust
use proptest::collection::vec;

// Generate vectors of 1-256 bytes
let strategy = vec(any::<u8>(), 1..256);

proptest! {
    #[test]
    fn test_with_variable_data(data in vec(any::<u8>(), 1..256)) {
        // NormHash wraps a pre-computed 32-byte digest from the engine.
        // Variable-length input testing applies to the engine's hashing,
        // not to NormHash construction in the contracts crate.
        let norm_hash = NormHash::from_digest([0x00; 32]);
        // ...
    }
}
```

**Why 1..256?**: Minimum 1 byte (empty input is often an edge case), maximum 256 (keep test runtime reasonable).

### Strategy 3: Constrained Enums

```rust
// Generate valid IdHashMode values
let strategy = prop_oneof![
    Just(IdHashMode::Unkeyed),
    Just(IdHashMode::KeyedV1),
];

proptest! {
    #[test]
    fn test_hash_mode(mode in prop_oneof![Just(IdHashMode::Unkeyed), Just(IdHashMode::KeyedV1)]) {
        // mode is guaranteed to be a valid IdHashMode
        let byte = mode.as_u8();
        prop_assert!(byte == 0 || byte == 1);
    }
}
```

**Alternative for enums with `#[derive(Arbitrary)]`**:
```rust
#[derive(Debug, Clone, Copy, Arbitrary)]
pub enum IdHashMode {
    Unkeyed = 0,
    KeyedV1 = 1,
}

proptest! {
    #[test]
    fn test_hash_mode(mode: IdHashMode) {
        // proptest automatically generates valid modes
    }
}
```

### Strategy 4: Domain Types

```rust
// Custom strategy for valid FindingIdInputs
fn finding_id_inputs_strategy() -> impl Strategy<Value = FindingIdInputs> {
    (
        uniform32(any::<u8>()),  // tenant
        uniform32(any::<u8>()),  // item
        uniform32(any::<u8>()),  // rule
        uniform32(any::<u8>()),  // secret
    ).prop_map(|(tenant, item, rule, secret)| {
        FindingIdInputs {
            tenant: TenantId::from_bytes(tenant),
            item: StableItemId::from_bytes(item),
            rule: RuleFingerprint::from_bytes(rule),
            secret: SecretHash::from_bytes_internal(secret),
        }
    })
}

proptest! {
    #[test]
    fn test_finding_id_with_strategy(inputs in finding_id_inputs_strategy()) {
        let id = derive_finding_id(&inputs);
        // ...
    }
}
```

**Note**: `NormHash` has no `derive` method or `NormHashInputs` struct. `NormHash` is constructed via `NormHash::from_digest(bytes)` from a pre-computed engine digest. The strategy example above uses `FindingIdInputs` instead, which is the primary derivation input struct.

### Strategy 5: Dependent Strategies

```rust
// Generate two distinct tenants
fn distinct_tenants_strategy() -> impl Strategy<Value = (TenantId, TenantId)> {
    uniform32(any::<u8>())
        .prop_flat_map(|tenant1| {
            // Generate second tenant that is different from first
            uniform32(any::<u8>())
                .prop_filter("tenants must differ", move |tenant2| tenant1 != *tenant2)
                .prop_map(move |tenant2| (TenantId(tenant1), TenantId(tenant2)))
        })
}

proptest! {
    #[test]
    fn test_tenant_isolation((tenant1, tenant2) in distinct_tenants_strategy()) {
        // tenant1 != tenant2 is guaranteed
        prop_assert_ne!(tenant1, tenant2);
    }
}
```

## Miri Compatibility

**Miri** is Rust's memory sanitizer. It detects undefined behavior but runs very slowly (~100× slower than native).

**Problem**: Running 256 proptest cases under Miri takes minutes.

**Solution**: Use `miri_proptest_config()` to reduce case count under Miri.

```rust
#[cfg(miri)]
fn proptest_config() -> ProptestConfig {
    ProptestConfig {
        cases: 10,  // Only 10 cases under Miri
        ..Default::default()
    }
}

#[cfg(not(miri))]
fn proptest_config() -> ProptestConfig {
    ProptestConfig {
        cases: 256,  // Full 256 cases normally
        ..Default::default()
    }
}

proptest! {
    #![proptest_config(proptest_config())]

    #[test]
    fn finding_id_is_pure(/* ... */) {
        // Runs 10 cases under Miri, 256 cases normally
    }
}
```

**Result**: Miri tests complete in ~10 seconds instead of ~10 minutes.

## Shrinking

When a property fails, proptest automatically finds the **minimal failing input**.

### Example: Shrinking in Action

```rust
proptest! {
    #[test]
    fn example_shrinking(x in 0u32..1000) {
        // Bug: fails for x >= 100
        prop_assert!(x < 100);
    }
}
```

**Execution**:
1. Proptest generates `x = 753` (random)
2. Test fails: `753 < 100` is false
3. Proptest shrinks:
   - Try `x = 376` → fails
   - Try `x = 188` → fails
   - Try `x = 94` → passes
   - Try `x = 141` → fails
   - Try `x = 117` → fails
   - Try `x = 105` → fails
   - Try `x = 99` → passes
   - Try `x = 102` → fails
   - Try `x = 100` → fails
4. Minimal failing input: `x = 100`

**Output**:
```
Test failed: x = 100
thread 'example_shrinking' panicked at 'assertion failed: `(left < right)`
  left: `100`,
 right: `100`'
```

**Benefit**: Debugging is easier with `x = 100` than with `x = 753`.

### Shrinking Strategies

- **Integers**: Shrink toward zero
- **Vectors**: Shrink length, shrink elements
- **Strings**: Shrink length, shrink characters toward 'a'
- **Tuples**: Shrink each element independently

**Custom shrinking**:
```rust
fn custom_strategy() -> impl Strategy<Value = MyType> {
    (0u32..1000, vec(any::<u8>(), 0..100))
        .prop_map(|(x, data)| MyType { x, data })
        .prop_shrink(|my_type| {
            // Custom shrinking logic
            vec![
                MyType { x: my_type.x / 2, data: my_type.data.clone() },
                MyType { x: my_type.x, data: my_type.data[..my_type.data.len() / 2].to_vec() },
            ]
        })
}
```

## Integration with Gossip-rs Tests

### Where PBT Is Used

1. **Identity derivation** (`crates/gossip-contracts/src/identity/*.rs`):
   - Purity: `f(x) == f(x)`
   - Collision-freedom: `x ≠ y → f(x) ≠ f(y)`
   - Field sensitivity: changing one field changes output
   - Roundtrip: `from_bytes(to_bytes(x)) == x`

2. **Canonical encoding** (`crates/gossip-contracts/src/identity/canonical.rs`):
   - Field-order independence: order matters for arrays, not for sets
   - Determinism: same struct → same bytes

3. **Shard algebra** (`crates/gossip-frontier/src/*.rs`):
   - Range splitting: no overlaps, no gaps
   - Key encoding: monotonicity preserved
   - Coverage: every item falls in exactly one shard

4. **Coordination** (`crates/gossip-coordination/src/*.rs`):
   - Lease acquisition: no double-leasing
   - Cursor monotonicity: cursor always advances
   - Fencing token gaps: no missing tokens

### Example: Field Sensitivity Test

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn finding_id_changes_if_tenant_changes(
            tenant1 in uniform32(any::<u8>()),
            tenant2 in uniform32(any::<u8>()),
            item in uniform32(any::<u8>()),
            rule in uniform32(any::<u8>()),
            secret in uniform32(any::<u8>()),
        ) {
            prop_assume!(tenant1 != tenant2);

            let inputs1 = FindingIdInputs {
                tenant: TenantId(tenant1),
                item: StableItemId(item),
                rule: RuleFingerprint(rule),
                secret: SecretHash::from_bytes_internal(secret),
            };
            let inputs2 = FindingIdInputs {
                tenant: TenantId(tenant2),
                item: StableItemId(item),
                rule: RuleFingerprint(rule),
                secret: SecretHash::from_bytes_internal(secret),
            };

            let id1 = derive_finding_id(&inputs1);
            let id2 = derive_finding_id(&inputs2);

            prop_assert_ne!(id1, id2, "FindingId must change when tenant changes");
        }

        #[test]
        fn finding_id_changes_if_rule_changes(
            tenant in uniform32(any::<u8>()),
            item in uniform32(any::<u8>()),
            rule1 in uniform32(any::<u8>()),
            rule2 in uniform32(any::<u8>()),
            secret in uniform32(any::<u8>()),
        ) {
            prop_assume!(rule1 != rule2);

            let inputs1 = FindingIdInputs {
                tenant: TenantId(tenant),
                item: StableItemId(item),
                rule: RuleFingerprint(rule1),
                secret: SecretHash::from_bytes_internal(secret),
            };
            let inputs2 = FindingIdInputs {
                tenant: TenantId(tenant),
                item: StableItemId(item),
                rule: RuleFingerprint(rule2),
                secret: SecretHash::from_bytes_internal(secret),
            };

            let id1 = derive_finding_id(&inputs1);
            let id2 = derive_finding_id(&inputs2);

            prop_assert_ne!(id1, id2, "FindingId must change when rule changes");
        }

        #[test]
        fn finding_id_changes_if_secret_changes(
            tenant in uniform32(any::<u8>()),
            item in uniform32(any::<u8>()),
            rule in uniform32(any::<u8>()),
            secret1 in uniform32(any::<u8>()),
            secret2 in uniform32(any::<u8>()),
        ) {
            prop_assume!(secret1 != secret2);

            let inputs1 = FindingIdInputs {
                tenant: TenantId(tenant),
                item: StableItemId(item),
                rule: RuleFingerprint(rule),
                secret: SecretHash::from_bytes_internal(secret1),
            };
            let inputs2 = FindingIdInputs {
                tenant: TenantId(tenant),
                item: StableItemId(item),
                rule: RuleFingerprint(rule),
                secret: SecretHash::from_bytes_internal(secret2),
            };

            let id1 = derive_finding_id(&inputs1);
            let id2 = derive_finding_id(&inputs2);

            prop_assert_ne!(id1, id2, "FindingId must change when secret changes");
        }
    }
}
```

**Coverage**: Tests that every input field matters. If derivation accidentally ignored one field, at least one of these tests would fail.

## Best Practices

### 1. Use `prop_assume!` Sparingly

```rust
// BAD: Rejects 99.99% of inputs
proptest! {
    #[test]
    fn test_rare_case(x in any::<u32>()) {
        prop_assume!(x == 42);  // Only 1 in 4 billion inputs pass
        // ...
    }
}
```

**Problem**: Proptest will reject 255 inputs, generate 1 valid one, run very slowly.

**Solution**: Use a targeted strategy.
```rust
// GOOD: Only generates valid inputs
proptest! {
    #[test]
    fn test_rare_case(x in Just(42u32)) {
        // ...
    }
}
```

### 2. Test Properties, Not Implementations

```rust
// BAD: Tests implementation details
proptest! {
    #[test]
    fn finding_id_uses_blake3(inputs: FindingIdInputs) {
        let id = derive_finding_id(&inputs);
        // Check that id was derived using BLAKE3 (how?)
        // This is fragile (breaks if we switch hash functions)
    }
}
```

```rust
// GOOD: Tests properties
proptest! {
    #[test]
    fn finding_id_is_deterministic(inputs: FindingIdInputs) {
        let id1 = derive_finding_id(&inputs);
        let id2 = derive_finding_id(&inputs);
        prop_assert_eq!(id1, id2);  // Property: determinism
    }
}
```

### 3. Combine PBT with Golden Tests

```rust
// Property: Determinism
proptest! {
    #[test]
    fn finding_id_is_deterministic(inputs: FindingIdInputs) {
        let id1 = derive_finding_id(&inputs);
        let id2 = derive_finding_id(&inputs);
        prop_assert_eq!(id1, id2);
    }
}

// Golden test: Exact output for known input
#[test]
fn finding_id_golden_vector() {
    let inputs = FindingIdInputs {
        tenant: TenantId([0x00; 32]),
        item: StableItemId([0x00; 32]),
        rule: RuleFingerprint([0x00; 32]),
        secret: SecretHash::from_bytes_internal([0x00; 32]),
    };
    let id = derive_finding_id(&inputs);
    assert_eq!(
        id.as_bytes(),
        &hex::decode("a1b2c3d4e5f6...").unwrap()[..],
    );
}
```

**Benefit**: PBT tests universal properties. Golden tests catch regressions in exact output.

## Summary

Property-based testing is a powerful technique for verifying correctness:

- **Tests universal properties**: Not just specific examples, but properties that must hold for ALL inputs
- **Finds edge cases**: Explores input space more thoroughly than manual test cases
- **Shrinks failures**: Automatically finds minimal failing input for easier debugging
- **High confidence**: 256 random inputs >> 5 manual examples

**Common properties in Gossip-rs**:
1. Purity: `f(x) == f(x)`
2. Collision-freedom: `x ≠ y → f(x) ≠ f(y)`
3. Field sensitivity: Changing one field changes output
4. Roundtrip: `decode(encode(x)) == x`
5. Monotonicity: `x < y → f(x) < f(y)`

**Proptest strategies**:
- Fixed-size arrays: `uniform32(any::<u8>())`
- Variable-length vectors: `vec(any::<u8>(), 1..256)`
- Enums: `prop_oneof!` or `#[derive(Arbitrary)]`
- Custom types: `prop_map`, `prop_filter`, `prop_flat_map`

**Best practices**:
- Use `prop_assume!` sparingly (prefer targeted strategies)
- Test properties, not implementations
- Combine PBT with golden tests for regression prevention
- Use `miri_proptest_config()` for Miri compatibility

**Result**: High confidence in correctness, fewer bugs in production.

**Next**: [Appendix D: Glossary](./D-glossary.md) defines all domain terms used in Gossip-rs.
