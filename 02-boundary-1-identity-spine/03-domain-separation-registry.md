# Domain Separation Registry

## All 17 Domain Constants

**Source:** `crates/gossip-contracts/src/identity/domain.rs`

| Constant | Value | Subsystem | Hash Mode | Purpose |
|----------|-------|-----------|-----------|---------|
| `SPLIT_ID_V1` | `"gossip/coord/v1/split-id"` | Coordination | derive-key | Shard-ID derivation during split operations |
| `OP_PAYLOAD_V1` | `"gossip/coord/v1/op-payload"` | Coordination | derive-key | Op-log payload hashing for idempotency |
| `FINDING_ID_V1` | `"gossip/finding/v1"` | Identity | derive-key | FindingId derivation |
| `OCCURRENCE_ID_V1` | `"gossip/occurrence/v1"` | Identity | derive-key | OccurrenceId derivation |
| `OBSERVATION_ID_V1` | `"gossip/observation/v1"` | Identity | derive-key | ObservationId derivation |
| `SECRET_HASH_V1` | `"gossip/secret-hash/v1"` | Identity | **keyed** | SecretHash keying (only keyed-mode constant) |
| `ITEM_ID_V1` | `"gossip/item-id/v1"` | Identity | derive-key | StableItemId derivation |
| `CONNECTOR_INSTANCE_ID_V1` | `"gossip/connector-instance-id/v1"` | Identity | derive-key | ConnectorInstanceIdHash derivation |
| `OBJECT_VERSION_V1` | `"gossip/object-version/v1"` | Identity | derive-key | ObjectVersionId derivation |
| `RULE_FINGERPRINT_V1` | `"gossip/rule/v1"` | Policy | derive-key | RuleFingerprint derivation |
| `POLICY_HASH_V2` | `"gossip/policy-hash/v2"` | Policy | derive-key | PolicyHash derivation (v2: v1 never shipped) |
| `RULES_DIGEST_V1` | `"gossip/rules-digest/v1"` | Policy | derive-key | Rules-digest derivation |
| `OVID_V1` | `"gossip/persistence/v1/ovid"` | Persistence | derive-key | OVID (Object-Version Identity) hash |
| `DONE_LEDGER_KEY_V1` | `"gossip/persistence/v1/done-key"` | Persistence | derive-key | Done-ledger key derivation (reserved) |
| `TRIAGE_GROUP_KEY_V1` | `"gossip/persistence/v1/triage-group"` | Persistence | derive-key | TriageGroupKey derivation |
| `COORDINATION_TELEMETRY_V1` | `"gossip/worker/v1/coordination-telemetry"` | Worker | derive-key | Coordination telemetry redaction digest |
| `GIT_REPO_ID_V1` | `"gossip/git/v1/repo-id"` | Git | derive-key | Stable 64-bit repository-namespace derivation for repo-native Git scans |

## Naming Convention

**Pattern:** `"gossip/<subsystem>/v<N>[/<operation>]"`

**Components:**
1. `gossip` — Fixed prefix (all domain constants start with this)
2. `<subsystem>` — Logical owner: `coord`, `finding`, `occurrence`, `observation`, `secret-hash`, `item-id`, `connector-instance-id`, `object-version`, `rule`, `policy-hash`, `rules-digest`, `persistence`, `worker`
3. `v<N>` — Monotonically increasing scheme version (starts at 1, never 0)
4. `<operation>` — Present when the subsystem owns multiple derivations (e.g., `coord` has `split-id` and `op-payload`)

**Examples:**
- `"gossip/finding/v1"` — Subsystem `finding`, no operation suffix (only one derivation)
- `"gossip/coord/v1/split-id"` — Subsystem `coord`, operation `split-id` (one of multiple)
- `"gossip/policy-hash/v2"` — Version bump (v1 was redesigned before shipping)

**Validation test** (`domain.rs:199-232`):
```rust
#[test]
fn all_constants_follow_naming_convention() {
    for (name, s) in all_domain_constants() {
        // Must start with "gossip/"
        assert!(s.starts_with("gossip/"));

        // Subsystem segment must be non-empty
        let after_prefix = &s["gossip/".len()..];
        let next_slash = after_prefix.find('/').unwrap_or(after_prefix.len());
        assert!(next_slash > 0);

        // Version marker: "/v" followed by digit(s), then end-of-string or '/'.
        let has_valid_version = s.match_indices("/v").any(|(pos, _)| {
            let after_v = &s[pos + 2..];
            let digit_count = after_v.chars().take_while(|c| c.is_ascii_digit()).count();
            digit_count > 0
                && (digit_count == after_v.len() || after_v.as_bytes()[digit_count] == b'/')
        });
        assert!(has_valid_version);

        assert!(!s.ends_with('/'));
    }
}
```

## The 5-Layer Uniqueness Enforcement

Domain constants must be globally unique. A collision would break domain separation, allowing cross-derivation attacks. Five independent mechanisms enforce uniqueness:

### Layer 1: Compile-Time Array Length

**Mechanism:** The `ALL` array has a fixed length (17). Adding a constant without updating the array length is a compile error.

**Source** (`domain.rs:149-166`):
```rust
pub const ALL: [&str; 17] = [
    SPLIT_ID_V1,
    OP_PAYLOAD_V1,
    FINDING_ID_V1,
    OCCURRENCE_ID_V1,
    OBSERVATION_ID_V1,
    SECRET_HASH_V1,
    ITEM_ID_V1,
    CONNECTOR_INSTANCE_ID_V1,
    OBJECT_VERSION_V1,
    RULE_FINGERPRINT_V1,
    POLICY_HASH_V2,
    RULES_DIGEST_V1,
    OVID_V1,
    DONE_LEDGER_KEY_V1,
    TRIAGE_GROUP_KEY_V1,
    COORDINATION_TELEMETRY_V1,
    GIT_REPO_ID_V1,
];
```

**Error message if length is wrong:**
```
error[E0308]: mismatched types
  --> crates/gossip-contracts/src/identity/domain.rs:149:26
   |
149 | pub const ALL: [&str; 17] = [
   |                          ^^ expected an array with a fixed size of 17 elements, found one with 18 elements
```

### Layer 2: `no_duplicate_values` Test

**Mechanism:** Insert all values into a `HashSet`. If insertion fails, a duplicate exists.

**Source** (`domain.rs:250-260`):
```rust
#[test]
fn no_duplicate_values() {
    let all = all_domain_constants();
    let mut seen = HashSet::new();
    for (name, value) in &all {
        assert!(
            seen.insert(*value),
            "duplicate domain constant value for {name}: {value:?}"
        );
    }
}
```

### Layer 3: `no_duplicate_names` Test

**Mechanism:** Insert all constant names into a `HashSet`. Catches copy-paste errors where the same name is reused.

**Source** (`domain.rs:262-269`):
```rust
#[test]
fn no_duplicate_names() {
    let all = all_domain_constants();
    let mut seen = HashSet::new();
    for (name, _) in &all {
        assert!(seen.insert(*name), "duplicate domain constant name: {name}");
    }
}
```

### Layer 4: `fixture_covers_all_constants` Test

**Mechanism:** Cross-check the `all_domain_constants()` fixture against the `ALL` array. Ensures no constant is defined but excluded from the registry.

**Source** (`domain.rs:271-292`):
```rust
#[test]
fn fixture_covers_all_constants() {
    let fixture = all_domain_constants();
    assert_eq!(
        fixture.len(),
        ALL.len(),
        "all_domain_constants() has {} entries but ALL has {}. \
         Did you add a constant without updating both?",
        fixture.len(),
        ALL.len(),
    );
    // Every value in the fixture must appear in ALL.
    for (name, value) in &fixture {
        assert!(
            ALL.contains(value),
            "fixture entry {name} ({value:?}) missing from ALL array"
        );
    }
}
```

### Layer 5: Format Tests

**Mechanism:** Three tests verify that all constants:
1. Are printable ASCII (lowercase letters, digits, `/`, `-` only)
2. Follow the naming convention (`gossip/<subsystem>/v<N>[/<operation>]`)
3. Have reasonable length (11..64 bytes)

**Source** (`domain.rs:183-248`):
```rust
#[test]
fn all_constants_are_printable_ascii() {
    for (name, value) in all_domain_constants() {
        for (i, &byte) in value.as_bytes().iter().enumerate() {
            assert!(
                byte.is_ascii_lowercase()
                    || byte.is_ascii_digit()
                    || byte == b'/'
                    || byte == b'-',
                "domain constant {name} has disallowed byte 0x{byte:02X} at position {i}",
            );
        }
    }
}

#[test]
fn all_constants_have_reasonable_length() {
    for (name, value) in all_domain_constants() {
        assert!(value.len() >= 11); // Shortest: "gossip/x/v1" = 11 bytes
        assert!(value.len() <= 64); // Max: prevents excessively long tags
    }
}
```

## The `SECRET_HASH_V1` Exception

**All constants except `SECRET_HASH_V1` use BLAKE3 derive-key mode** (`Hasher::new_derive_key(context)`). The domain tag is the context string passed to BLAKE3's key schedule.

**`SECRET_HASH_V1` is the only constant used with BLAKE3 keyed mode** (`Hasher::new_keyed(key)`). The domain tag is fed as **data** inside the keyed hasher, not as the derive-key context.

**Why:** The hasher is already in keyed mode with the `TenantSecretKey` as the MAC key. The domain tag acts as a **versioning prefix** to separate this derivation from any other use of the same tenant key.

**Source** (`finding.rs:327-335`):
```rust
pub fn key_secret_hash(key: &TenantSecretKey, norm: &NormHash) -> SecretHash {
    let mut h = Hasher::new_keyed(key.as_bytes());
    // Domain tag is fed as *data* (not a derive-key context) because the
    // hasher is already in keyed mode. This acts as a context prefix that
    // separates this derivation from any other use of the same tenant key.
    h.update(domain::SECRET_HASH_V1.as_bytes());
    h.update(norm.as_bytes());
    SecretHash::from_bytes_internal(finalize_32(&h))
}
```

**Comparison:**

| Mode | Constant | Usage | Domain Tag Role |
|------|----------|-------|----------------|
| derive-key | `FINDING_ID_V1` | `Hasher::new_derive_key("gossip/finding/v1")` | Context for BLAKE3 key schedule |
| keyed | `SECRET_HASH_V1` | `Hasher::new_keyed(tenant_key); h.update("gossip/secret-hash/v1")` | Data prefix inside keyed hasher |

**Rationale for keyed mode:** `SecretHash` must provide cross-tenant isolation. Two tenants scanning the same secret should produce different `SecretHash` values. Keying with `TenantSecretKey` ensures this: an attacker with tenant A's `SecretHash` values cannot determine if tenant B found the same secret. Rainbow tables are useless without the tenant key.

## Domain Separation Guarantee

**BLAKE3's derive-key mode** (from the [BLAKE3 spec](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf), section 5.3):

> The derive_key function computes a context-specific subkey from a root key and a context string. The output is a 256-bit key that is cryptographically independent of keys derived with other context strings.

**Practical interpretation:** Two hashers with different domain tags are treated as **independent hash functions**. Cross-domain collisions remain cryptographically negligible (bounded by BLAKE3's 256-bit security), but **not mathematically impossible** (collisions between different hash functions are not ruled out by the pigeonhole principle).

**Example:**
```rust
let mut h1 = Hasher::new_derive_key("gossip/finding/v1");
h1.update(&[0x42; 32]);
let digest1 = h1.finalize();

let mut h2 = Hasher::new_derive_key("gossip/occurrence/v1");
h2.update(&[0x42; 32]);
let digest2 = h2.finalize();

// digest1 != digest2 (with overwhelming probability)
assert_ne!(digest1, digest2);
```

**Test** (`hashing.rs:211-218`):
```rust
proptest! {
    #[test]
    fn domain_separation_for_random_payload(data in vec(any::<u8>(), 1..256)) {
        let d1 = hash_payload("gossip/left/v1", data.as_slice());
        let d2 = hash_payload("gossip/right/v1", data.as_slice());
        prop_assert_ne!(d1, d2);
    }
}
```

## Summary Table

| Layer | Mechanism | Catches | Fires At |
|-------|-----------|---------|----------|
| 1. Compile-time length | `ALL: [&str; 17]` | Added constant, forgot to update array | `cargo check` |
| 2. Value uniqueness | `HashSet::insert(value)` | Two constants with same string value | `cargo test` |
| 3. Name uniqueness | `HashSet::insert(name)` | Copy-paste error (same name reused) | `cargo test` |
| 4. Fixture completeness | Cross-check fixture vs `ALL` | Constant in fixture but not in `ALL` | `cargo test` |
| 5. Format validation | ASCII, naming, length | Invalid characters, wrong pattern, too long/short | `cargo test` |

**Next chapter:** The ID Type Hierarchy (`types.rs`) — the three root types and the full derivation chain.
