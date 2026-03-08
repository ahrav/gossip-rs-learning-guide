# Appendix B: BLAKE3 Internals

BLAKE3 is the cryptographic hash function at the heart of Gossip-rs's identity system. This appendix explains how BLAKE3 works internally and why it enables powerful domain separation.

## BLAKE3 Overview

**Key properties**:
- **Fast**: ~1 GiB/s on modern CPUs (SIMD-accelerated)
- **Secure**: No known vulnerabilities (based on BLAKE2, which descends from the BLAKE SHA-3 competition finalist; Keccak won)
- **Merkle tree structure**: Enables parallelism, incremental hashing
- **Three modes**: Hash, Keyed, Derive-Key

**Why Gossip-rs uses BLAKE3**:
1. **Derive-key mode**: Domain separation without performance cost
2. **Performance**: Identity derivation is on the hot path
3. **Portability**: Rust implementation is well-maintained, no C dependencies

## BLAKE3 Merkle Tree Structure

BLAKE3 splits input into 1 KiB chunks, processes each chunk independently, then combines them in a binary tree.

```mermaid
graph TB
    Input[Input: 4 KiB] --> C1[Chunk 1<br/>0-1 KiB]
    Input --> C2[Chunk 2<br/>1-2 KiB]
    Input --> C3[Chunk 3<br/>2-3 KiB]
    Input --> C4[Chunk 4<br/>3-4 KiB]

    C1 --> H1[Hash 1]
    C2 --> H2[Hash 2]
    C3 --> H3[Hash 3]
    C4 --> H4[Hash 4]

    H1 --> P1[Parent 1]
    H2 --> P1
    H3 --> P2[Parent 2]
    H4 --> P2

    P1 --> Root[Root Hash]
    P2 --> Root

    style Root fill:#e1f5ff
```

**Why this matters**:
- **Parallelism**: Each chunk can be hashed on a separate CPU core
- **Incremental hashing**: Can append data without re-hashing from the beginning
- **Streaming**: Can hash data larger than RAM

**For small inputs** (< 1 KiB, which is most identity derivations in Gossip-rs):
- Single chunk → single compression function call → very fast

## The Three Modes

### Mode 1: Hash Mode (Standard)

**Usage**: General-purpose hashing.

**Construction**:
```rust
let mut hasher = blake3::Hasher::new();
hasher.update(b"input data");
let hash: [u8; 32] = hasher.finalize().into();
```

**Internal**: Key schedule uses fixed initialization vector (IV):
```rust
const IV: [u32; 8] = [
    0x6A09E667, 0xBB67AE85, 0x3C6EF372, 0xA54FF53A,
    0x510E527F, 0x9B05688C, 0x1F83D9AB, 0x5BE0CD19,
];
```

**Gossip-rs does NOT use this mode** (uses derive-key mode for domain separation).

### Mode 2: Keyed Mode (MAC-Like)

**Usage**: Message Authentication Code (MAC). Keyed hash with a secret.

**Construction**:
```rust
let key: [u8; 32] = /* secret key */;
let mut hasher = blake3::Hasher::new_keyed(&key);
hasher.update(b"input data");
let mac: [u8; 32] = hasher.finalize().into();
```

**Internal**: Key schedule uses the provided key as the initialization vector:
```rust
// Pseudocode
let iv: [u32; 8] = u8_array_to_u32_words(key);
```

**Security property**: Without the key, cannot produce a valid MAC. Even if attacker knows the input and output, cannot forge a MAC for a different input.

**Gossip-rs uses this for `SecretHash` derivation**:
```rust
let mut hasher = blake3::Hasher::new_keyed(tenant_secret_key.as_bytes());
hasher.update(domain::SECRET_HASH_V1.as_bytes());  // "gossip/secret-hash/v1"
hasher.update(norm_hash.as_bytes());
let secret_hash: [u8; 32] = hasher.finalize().into();
```

**Why keyed mode for SecretHash**: Tenant isolation. Different tenant key → mathematically independent hash function.

### Mode 3: Derive-Key Mode (Domain Separation)

**Usage**: Derive domain-specific hash functions from a context string.

**Construction**:
```rust
let mut hasher = blake3::Hasher::new_derive_key("DOMAIN_TAG");
hasher.update(b"input data");
let hash: [u8; 32] = hasher.finalize().into();
```

**Internal**:
1. Hash the context string to derive a key
2. Use that derived key as the initialization vector (like keyed mode)

```rust
// Pseudocode
fn derive_key_hasher(context: &str) -> Hasher {
    let derived_key = blake3::derive_key(context, /* output_len */ 32);
    Hasher::new_keyed(&derived_key)
}
```

**Why this provides domain separation**:
- Each context string produces a unique initialization vector
- Mathematically, two hashers with different contexts are **independent hash functions**
- Stronger than prefix-based separation (`hash("DOMAIN_A" || input)`)

**Gossip-rs uses this for ALL identity derivations** (except `SecretHash`):
```rust
static FINDING_HASHER: LazyLock<blake3::Hasher> =
    LazyLock::new(|| blake3::Hasher::new_derive_key(domain::FINDING_ID_V1));

static OCCURRENCE_HASHER: LazyLock<blake3::Hasher> =
    LazyLock::new(|| blake3::Hasher::new_derive_key(domain::OCCURRENCE_ID_V1));

static POLICY_HASH_HASHER: LazyLock<blake3::Hasher> =
    LazyLock::new(|| blake3::Hasher::new_derive_key(domain::POLICY_HASH_V2));
```

## Domain Separation Deep Dive

### Why Domain Separation Matters

**Without domain separation**:
```rust
// BAD: Same hash function for all purposes
let hash1 = blake3::hash(b"alice@example.com");
let hash2 = blake3::hash(b"alice@example.com");

assert_eq!(hash1, hash2);
// Problem: If hash1 is a TenantId and hash2 is a FindingId,
// they collide even though they represent different entities!
```

**With domain separation**:
```rust
// GOOD: Different hash functions for different purposes
let mut hasher1 = blake3::Hasher::new_derive_key("TENANT_ID");
hasher1.update(b"alice@example.com");
let tenant_id = hasher1.finalize();

let mut hasher2 = blake3::Hasher::new_derive_key("FINDING_ID");
hasher2.update(b"alice@example.com");
let finding_id = hasher2.finalize();

assert_ne!(tenant_id, finding_id);
// No collision: different domains → different hash functions
```

### Derive-Key vs Prefix-Based Separation

**Prefix-based** (common but weaker):
```rust
let hash1 = blake3::hash(b"TENANT_ID:alice@example.com");
let hash2 = blake3::hash(b"FINDING_ID:alice@example.com");
```

**Problems**:
1. **Delimiter confusion**: What if input contains `:`? Need escaping.
2. **Prefix length variance**: `"TENANT_ID:"` vs `"FINDING_ID:"` have different lengths → different chunk boundaries → performance variance.
3. **Not mathematically independent**: Both use the same hash function, just different inputs.

**Derive-key** (stronger):
```rust
let hash1 = derive_key_hasher("TENANT_ID").hash(b"alice@example.com");
let hash2 = derive_key_hasher("FINDING_ID").hash(b"alice@example.com");
```

**Benefits**:
1. **No delimiter issues**: Context and input are separate. Input can contain any bytes.
2. **Constant performance**: All domain hashers process input the same way (no prefix overhead).
3. **Mathematically independent**: Different context → different IV → different compression function → different hash function.

### Formal Security Property

**Claim**: For distinct contexts `C1` and `C2`, the functions `H_C1(x) = derive_key_hash(C1, x)` and `H_C2(x) = derive_key_hash(C2, x)` are independent PRFs (pseudorandom functions).

**What this means**:
- Knowing `H_C1(x)` reveals no information about `H_C2(x)`
- Cannot find collisions across domains (cannot find `x` and `y` such that `H_C1(x) = H_C2(y)`)
- Each domain has its own 256-bit output space (no shared collision risk)

**Proof sketch**: BLAKE3's derive-key mode derives a unique key per context. Keyed BLAKE3 is a PRF (proven in BLAKE3 paper). Independent keys → independent PRFs.

## Key Schedule Caching

### The Performance Problem

```rust
// Naive implementation (SLOW)
fn derive_finding_id(input: &[u8]) -> [u8; 32] {
    let mut hasher = blake3::Hasher::new_derive_key("gossip/finding/v1");
    // ^ This runs the key derivation every time! (expensive)
    hasher.update(input);
    hasher.finalize().into()
}
```

**Cost of `new_derive_key`**:
1. Hash the context string (`"gossip/finding/v1"`) to derive a key
2. Initialize the hasher state with the derived key

For `"gossip/finding/v1"` (17 bytes), this is ~100 CPU cycles.

**If called 1 million times**: 100M cycles wasted on re-deriving the same key.

### The Solution: LazyLock Caching

```rust
use std::sync::LazyLock;

// Initialize once per process
static FINDING_HASHER: LazyLock<blake3::Hasher> =
    LazyLock::new(|| blake3::Hasher::new_derive_key(domain::FINDING_ID_V1));

fn derive_finding_id(input: &[u8]) -> [u8; 32] {
    let mut hasher = FINDING_HASHER.clone();
    // ^ Clone the cached hasher (cheap: just copy internal state)
    hasher.update(input);
    hasher.finalize().into()
}
```

**Cost of `FINDING_HASHER.clone()`**:
- Copy ~256 bytes of internal state (hasher state + buffer)
- ~10 CPU cycles (optimized to memcpy)

**Speedup**: 10× faster than re-deriving key every time.

### Why Clone Is Cheap

```rust
pub struct Hasher {
    cv_stack: [u32; 54],       // Chaining values (216 bytes)
    block: [u8; 64],           // Current block (64 bytes)
    block_len: u8,             // Block length (1 byte)
    blocks_compressed: u64,    // Counter (8 bytes)
    flags: u32,                // Flags (4 bytes)
    key: [u32; 8],             // Key (32 bytes)
    // Total: ~320 bytes
}

impl Clone for Hasher {
    fn clone(&self) -> Self {
        // Just memcpy the struct
        Self {
            cv_stack: self.cv_stack,
            block: self.block,
            block_len: self.block_len,
            blocks_compressed: self.blocks_compressed,
            flags: self.flags,
            key: self.key,
        }
    }
}
```

**Key insight**: The expensive part (key derivation) is already done. Cloning just copies the post-derivation state.

## BLAKE3 Performance

### Throughput Benchmarks

On a modern CPU (Apple M1):
- **Hashing throughput**: ~1 GiB/s (single core)
- **Parallel throughput**: ~8 GiB/s (8 cores)

For Gossip-rs workloads (small inputs, ~100 bytes average):
- **FindingId derivation**: ~500 ns (2M derivations/sec)
- **SecretHash derivation**: ~600 ns (1.6M derivations/sec, includes keyed hash)
- **NormHash derivation**: ~300 ns (3.3M derivations/sec, smaller input)

### SIMD Acceleration

BLAKE3 uses SIMD instructions when available:
- **AVX2** (Intel/AMD): 256-bit vectors → process 4 chunks in parallel
- **AVX-512** (Intel/AMD): 512-bit vectors → process 8 chunks in parallel
- **NEON** (ARM): 128-bit vectors → process 2 chunks in parallel

**Gossip-rs benefits**: On x86_64 with AVX2, identity derivation is ~3× faster than without SIMD.

### Cache-Friendly Design

BLAKE3's internal state (~320 bytes) fits in L1 cache. For small inputs (< 1 KiB):
- **Hot path**: L1 cache only (no memory access)
- **Latency**: ~100 cycles (3× faster than memory access)

**Gossip-rs benefits**: Identity derivation is CPU-bound, not memory-bound.

## Comparison to Other Hash Functions

| Hash Function | Speed (1 GiB/s) | Security | Parallel | Derive-Key Mode |
|---------------|-----------------|----------|----------|-----------------|
| MD5           | ~3.0            | ❌ Broken | ❌ No    | ❌ No           |
| SHA-1         | ~1.0            | ❌ Broken | ❌ No    | ❌ No           |
| SHA-256       | ~0.3            | ✅ Secure | ❌ No    | ❌ No           |
| SHA-512       | ~0.5            | ✅ Secure | ❌ No    | ❌ No           |
| BLAKE2b       | ~0.8            | ✅ Secure | ❌ No    | ⚠️ Manual       |
| BLAKE3        | ~1.0            | ✅ Secure | ✅ Yes   | ✅ Built-in     |

**BLAKE3 wins on**:
- **Speed**: Faster than SHA-2, competitive with BLAKE2
- **Parallelism**: Merkle tree structure enables multi-core hashing
- **API**: Derive-key mode is built-in, no manual domain tagging

## Why Not HMAC?

**HMAC** (Hash-based Message Authentication Code) is a common way to key a hash:
```rust
// HMAC-SHA256
let mac = hmac::HMAC::new(sha256, key);
mac.update(input);
let result = mac.finalize();
```

**HMAC construction**:
```
HMAC(key, input) = Hash((key ⊕ opad) || Hash((key ⊕ ipad) || input))
```

**Two hash calls** → ~2× slower than keyed BLAKE3.

**Gossip-rs uses keyed BLAKE3 instead**:
```rust
let mut hasher = blake3::Hasher::new_keyed(&key);
hasher.update(input);
let mac = hasher.finalize();
```

**One hash call** → faster, simpler, equally secure.

**Security**: BLAKE3's keyed mode provides the same security properties as HMAC (PRF, MAC).

## BLAKE3 in Gossip-rs Identity Derivation

### Summary Table

| Identity Type | BLAKE3 Mode | Domain Constant | Context String Value | Keyed? |
|---------------|-------------|-----------------|----------------------|--------|
| SplitId (coordination) | Derive-Key | `domain::SPLIT_ID_V1` | `"gossip/coord/v1/split-id"` | No |
| OpPayload (coordination) | Derive-Key | `domain::OP_PAYLOAD_V1` | `"gossip/coord/v1/op-payload"` | No |
| FindingId | Derive-Key | `domain::FINDING_ID_V1` | `"gossip/finding/v1"` | No |
| OccurrenceId | Derive-Key | `domain::OCCURRENCE_ID_V1` | `"gossip/occurrence/v1"` | No |
| ObservationId | Derive-Key | `domain::OBSERVATION_ID_V1` | `"gossip/observation/v1"` | No |
| SecretHash | Keyed | `domain::SECRET_HASH_V1` | `"gossip/secret-hash/v1"` | Yes (TenantSecretKey) |
| StableItemId | Derive-Key | `domain::ITEM_ID_V1` | `"gossip/item-id/v1"` | No |
| ConnectorInstanceIdHash | Derive-Key | `domain::CONNECTOR_INSTANCE_ID_V1` | `"gossip/connector-instance-id/v1"` | No |
| ObjectVersionId | Derive-Key | `domain::OBJECT_VERSION_V1` | `"gossip/object-version/v1"` | No |
| RuleFingerprint | Derive-Key | `domain::RULE_FINGERPRINT_V1` | `"gossip/rule/v1"` | No |
| PolicyHash | Derive-Key | `domain::POLICY_HASH_V2` | `"gossip/policy-hash/v2"` | No |
| RulesDigest | Derive-Key | `domain::RULES_DIGEST_V1` | `"gossip/rules-digest/v1"` | No |
| OVID (persistence) | Derive-Key | `domain::OVID_V1` | `"gossip/persistence/v1/ovid"` | No |
| DoneLedgerKey (persistence) | Derive-Key | `domain::DONE_LEDGER_KEY_V1` | `"gossip/persistence/v1/done-key"` | No |
| TriageGroupKey (planned) | Derive-Key | `domain::TRIAGE_GROUP_KEY_V1` | `"gossip/persistence/v1/triage-group"` | No |

> **Note on NormHash**: `NormHash` is externally computed by the detection engine and received via `from_digest()`. It is not derived within the contracts crate and does not use a domain tag from the registry.

**Key insight**: Only `SecretHash` uses keyed mode (for tenant isolation). All others use derive-key mode (for domain separation). All domain tags follow the `"gossip/<subsystem>/v<N>"` naming convention defined in `domain.rs`.

### Example: FindingId Derivation

```rust
// 1. Initialize hasher (cached, done once per process)
static FINDING_HASHER: LazyLock<blake3::Hasher> =
    LazyLock::new(|| blake3::Hasher::new_derive_key(domain::FINDING_ID_V1));
// This runs: derive_key = BLAKE3("gossip/finding/v1")
//            IV = u8_to_u32(derive_key)
//            hasher = Hasher::with_iv(IV)

// 2. Clone cached hasher (cheap: ~10 cycles)
let mut hasher = FINDING_HASHER.clone();

// 3. Update with inputs (fast: SIMD-accelerated)
tenant.write_canonical(&mut hasher);                // 32 bytes
item.write_canonical(&mut hasher);                  // 32 bytes
rule.write_canonical(&mut hasher);                  // 32 bytes
secret.write_canonical(&mut hasher);                // 32 bytes
// Total input: 128 bytes (single chunk)

// 4. Finalize (fast: single compression)
let hash: [u8; 32] = hasher.finalize().into();
let finding_id = FindingId(hash);
```

**Performance**: ~500 ns total (~2M derivations/sec).

## Security Considerations

### Preimage Resistance

**Property**: Given hash `H(x) = y`, infeasible to find `x`.

**BLAKE3 provides**: 256-bit preimage resistance (attacker needs ~2^256 operations).

**Gossip-rs relies on this for**: Cannot reverse-engineer `NormHash` from `SecretHash`.

### Collision Resistance

**Property**: Infeasible to find `x ≠ y` such that `H(x) = H(y)`.

**BLAKE3 provides**: 256-bit collision resistance (attacker needs ~2^128 operations, birthday bound).

**Gossip-rs relies on this for**: Different secrets have different `NormHash` values (no collisions).

### Second-Preimage Resistance

**Property**: Given `x` and `H(x) = y`, infeasible to find `x' ≠ x` such that `H(x') = y`.

**BLAKE3 provides**: 256-bit second-preimage resistance.

**Gossip-rs relies on this for**: Cannot forge a different policy with the same `PolicyHash`.

## Future: BLAKE3 in Persistence

**Planned use cases**:
1. **PageCommit HMAC**: Authenticate page commits with tenant key to prevent tampering.
2. **Cursor integrity**: Hash cursor state to detect corruption.
3. **Audit log**: Hash-chain of operations for tamper-evident logging.

**Why BLAKE3 is ideal**:
- Fast enough for hot path
- Built-in keyed mode for authentication
- Portable (Rust implementation, no C dependencies)

## Summary

BLAKE3 is the cryptographic foundation of Gossip-rs's identity system. Key takeaways:

1. **Three modes**: Hash (standard), Keyed (MAC), Derive-Key (domain separation)
2. **Gossip-rs uses**:
   - Derive-Key mode for all identities (domain separation)
   - Keyed mode for `SecretHash` (tenant isolation)
3. **Performance optimizations**:
   - LazyLock caching of key schedules (~10× speedup)
   - SIMD acceleration (~3× speedup on AVX2)
   - Cache-friendly design (~100 cycles per derivation)
4. **Security properties**:
   - 256-bit preimage resistance
   - 256-bit collision resistance
   - Domain independence (derive-key mode)
5. **Why not alternatives**:
   - HMAC: 2× slower (two hash calls)
   - SHA-256: 3× slower (no SIMD)
   - BLAKE2: No built-in derive-key mode

**Result**: Fast, secure, portable identity derivation with mathematical domain separation.

**Next**: [Appendix C: Property-Based Testing Guide](./C-property-based-testing-guide.md) explains how Gossip-rs uses proptest to verify identity properties.
