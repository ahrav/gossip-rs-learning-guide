# Idempotency

## Overview

Network failures create an unavoidable uncertainty: when a request times out, you don't know if it succeeded or failed. The server might have processed the request but the response was lost. Retrying blindly risks duplicate operations—charging a customer twice, scanning the same page twice, recording the same finding twice.

**Idempotency** solves this: design operations so that applying them N times produces the same result as applying them once. Retry safely, knowing that duplicates are harmless.

Gossip-rs uses idempotency throughout:
- **Shard operations**: Op-log prevents duplicate state mutations
- **Finding ingestion**: Content-addressed identity deduplicates findings
- **Page scanning**: Cursor validation prevents re-enumeration

This chapter explains the idempotency pattern, its implementation in Gossip-rs, and subtle edge cases (like payload hash validation) that prevent bugs.

## The Problem: Lost Responses

Consider a naive "complete" operation:

```
Client: Checkpoint(shard=42, cursor=page_17_cursor)
Server: [processes request, writes to DB]
Network: [response packet dropped]
Client: [timeout after 5s]
Client: Checkpoint(shard=42, cursor=page_17_cursor)  ← RETRY
Server: [processes again, writes duplicate record]
```

The duplicate record is harmless for checkpoint (writing the same cursor twice is fine due to idempotency), but consider a more dangerous case:

```
Client: Checkpoint(shard=42, cursor=100)
Server: [writes cursor=100]
Network: [response lost]
Client: [timeout, retry]
Client: Checkpoint(shard=42, cursor=150)  ← Caller moved forward
Server: [writes cursor=150]
Network: [response lost again]
Client: [timeout, retry]
Client: Checkpoint(shard=42, cursor=100)  ← BUG: Retries with OLD cursor!
Server: [writes cursor=100]
```

The cursor moved backward! Pages 101-150 will be scanned again, producing duplicate findings. This is a safety violation.

**The challenge**: Distinguish "first attempt" from "retry of completed operation" from "retry with different payload (a new operation)".

## Definition of Idempotency

**Formal definition**:

> An operation f is idempotent if, for all inputs x:
> ```
> f(f(x)) = f(x)
> ```

**In distributed systems**:

> An operation is idempotent if applying it multiple times with the same inputs produces the same result (and same observable effects) as applying it once.

**Examples**:
- **Idempotent**: `SET x = 5`, `DELETE user WHERE id = 42`, HTTP PUT
- **Not idempotent**: `INCREMENT x`, `INSERT INTO users`, HTTP POST

**Why natural idempotency is rare**: Most interesting operations are not naturally idempotent. `checkpoint` is only idempotent if you always set the same cursor value. But in practice, the cursor advances, so retries with the same operation ID must detect "this is a retry of cursor=100, not a new request for cursor=150".

## The Stripe Idempotency Pattern

Stripe's API pioneered a practical pattern for making operations idempotent (Brandur Leach, 2017). Gossip-rs adapts this pattern for shard operations.

**Core idea**: The client generates a unique **operation ID (OpId)** and includes it in every request. The server maintains an **operation log (op-log)** mapping OpId → result. On retry:
1. Check if OpId is in the op-log
2. If found: return the stored result (don't re-execute)
3. If not found: execute the operation, store result in op-log, return result

**Stripe's additional requirement**: If the OpId is found but the request payload differs, return an error (conflict). This prevents accidental reuse of an OpId for a different operation.

### Gossip-rs Adaptation

Gossip-rs uses this pattern for all shard mutations:

1. **OpId generation**: Caller-generated CSPRNG value (a raw `u64`):
   ```rust
   // OpId is NOT a content-addressed hash — it's a random nonce.
   // Must be generated via a cryptographically secure PRNG (e.g., OsRng)
   // or a coordinated counter to prevent accidental collisions.
   let op_id = OpId::from_raw(rng.gen::<u64>());
   ```
   `OpId` is a 64-bit value that serves as an idempotency key. The caller generates a unique `OpId` per logical operation attempt. Two independent workers using the same `OpId` for different operations will trigger `OpIdConflict`, which is indistinguishable from a legitimate retry with a corrupted payload.

2. **Payload hash**: Separate hash of the operation inputs (using domain-separated hashing):
   ```rust
   PayloadHash = Blake3::new_derive_key("gossip/coord/v1/op-payload").update(serialize(operation)).finalize()
   ```
   The payload hash uses BLAKE3 derive-key mode with the domain tag `"gossip/coord/v1/op-payload"` for domain separation. It is truncated to `u64` for storage efficiency.

3. **Op-log storage**: Bounded log of recent operations (16 entries per shard). Each entry:
   ```rust
   pub struct OpLogEntry {
       op_id: OpId,
       kind: OpKind,
       result: OpResult,
       payload_hash: u64,
       executed_at: LogicalTime,
   }
   ```
   The `kind` field (`OpKind`) distinguishes operation types (e.g., `Checkpoint` vs `Complete` vs `Park`), enabling the server to apply type-specific validation on replay. The `payload_hash` is a plain `u64` (truncated Blake3), and `executed_at` uses `LogicalTime` rather than wall-clock time to avoid clock-skew issues in the op-log itself.

4. **Three-way decision on request**:
   ```rust
   match op_log.get(op_id) {
       None => {
           // New operation: execute and store result
           let result = execute_operation(req);
           op_log.insert(op_id, payload_hash, result);
           Ok(result)
       }
       Some(record) if record.payload_hash == payload_hash => {
           // Duplicate: return stored result
           Ok(record.result.clone())
       }
       Some(record) => {
           // Conflict: same OpId, different payload
           Err(OpIdConflict { op_id, expected_hash: record.payload_hash, actual_hash: payload_hash })
       }
   }
   ```

### Why Payload Hash?

**The subtle bug it prevents**:

Suppose a caller reuses the same `OpId` for two different checkpoint operations (a bug, since `OpId` should be generated fresh via CSPRNG for each logical operation):
```
Request 1: Checkpoint(shard=42, cursor=100, op_id=0xABC)
→ Executes, stores result

Request 2: Checkpoint(shard=42, cursor=150, op_id=0xABC)  ← Same OpId!
→ Op-log finds 0xABC, returns stored result (cursor=100)
→ Caller thinks cursor is 150, but it's actually 100
```

The OpId collision caused silent data corruption. The cursor stayed at 100, but the caller believes it advanced to 150. Subsequent operations (like `complete`) will reference the wrong cursor.

**Solution**: The payload hash catches this. Since the OpId is a CSPRNG nonce (not derived from content), the server relies on the payload hash to detect mismatched retries:
```
Request 1: Checkpoint(cursor=100, payload_hash=H1)
→ Executes, stores (op_id, H1, result)

Request 2: Checkpoint(cursor=150, payload_hash=H2)
→ Op-log finds op_id, but H2 ≠ H1
→ Return OpIdConflict { op_id, expected_hash: H1, actual_hash: H2 }
```

The payload hash is a **defensive mechanism**: even if the caller reuses an OpId (due to a bug), the server detects the mismatch and fails safe.

## Op-Log Design

### Bounded Size

Unbounded op-logs grow without limit, eventually exhausting memory. Gossip-rs uses a **bounded op-log**: 16 entries per shard (configurable).

**Why 16?**
- Typical retry policies use exponential backoff: 1s, 2s, 4s, 8s, 16s = ~30s total. A 16-entry log covers ~16 operations. If operations are 1-2s apart, the log spans ~30s of history.
- Memory usage: 16 entries × 128 bytes/entry = 2 KB per shard. For 100K shards: ~200 MB total (acceptable).

**Eviction policy**: When the log is full, evict the **oldest entry** (FIFO). This means idempotency is only guaranteed for recent operations. After eviction, a retry of an old operation will be treated as a new operation.

**Implication**: Clients should not retry operations indefinitely. After a reasonable timeout (e.g., 1 minute), give up and reconcile state manually.

### Persistence vs In-Memory

**Gossip-rs choice**: Op-log is **in-memory only** (not persisted to the coordination backend).

**Trade-offs**:
- **Pros**: Low latency (no disk I/O), simple implementation
- **Cons**: Op-log is lost on coordinator restart, meaning retries across restarts are not idempotent

**Why this is safe**: The op-log is a **performance optimization**, not a correctness requirement. If the op-log is lost:
- A retried operation will be re-executed
- For naturally idempotent operations (e.g., `complete` on an already-Done shard), this is harmless
- For cursor-advancing operations (e.g., `checkpoint`), the 5-check validation preamble (see [03-leases-and-fencing.md](./03-leases-and-fencing.md)) provides correctness: the cursor must advance monotonically, so a duplicate checkpoint is rejected by the `CursorRegression` check.

**Future enhancement**: Persist the op-log to provide idempotency across coordinator restarts. This requires careful design to avoid performance regressions (e.g., batch writes, async persistence).

## Idempotency in Finding Ingestion

Findings use **content-addressed identity** for deduplication:

```rust
FindingId = BLAKE3::derive_key("gossip/finding/v1", tenant_id || stable_item_id || rule_fingerprint || secret_hash)
```

The `FindingId` uses BLAKE3 **derive-key mode** with the domain tag `"gossip/finding/v1"` for domain separation. It is derived from exactly four inputs: the tenant, the version-stable item identifier (e.g., a repo or bucket), the detection rule fingerprint, and the tenant-keyed secret hash. Notably, position information like byte offset and length is *not* part of `FindingId`---those fields belong to `OccurrenceId`, which ties a finding to a specific location within a specific object version:

```rust
OccurrenceId = BLAKE3::derive_key("gossip/occurrence/v1", finding_id || object_version_id || byte_offset || byte_length)
```

This separation means that the same secret detected by the same rule in the same item always produces the same `FindingId`, regardless of where it appears or which version is being scanned.

Two workers scanning the same secret will compute identical `FindingId`. When ingesting findings:
```rust
if store.contains(finding_id) {
    // Already recorded, skip
    return Ok(Duplicate);
}
store.insert(finding_id, finding_data);
```

This is **perfect idempotency**: deterministic IDs mean duplicate detection is O(1) and has no false positives.

**Contrast with UUID-based IDs**: If findings used randomly generated UUIDs, duplicate detection would require fuzzy matching (same repo + path + value?), which is expensive and error-prone.

## Idempotency in Page Scanning

Cursors prevent re-enumeration:

1. **Before scanning page N**: Record `scan_started(page=N, cursor=C)` in the shard state
2. **Scan page N**: Enumerate items, emit findings
3. **After scanning page N**: Record `scan_completed(page=N, cursor=C′)` in the done ledger

If the worker crashes between steps 2 and 3, another worker will retry. The retry is safe because:
- **Finding deduplication**: Content-addressed IDs prevent duplicate findings
- **Cursor validation**: The coordination layer rejects checkpoints whose `last_key` byte slice regresses behind the previously recorded `last_key`, preventing backward movement

**Edge case**: What if page N is scanned twice concurrently (due to lease reassignment)? Both workers scan, emit findings, and write done-records. The outcome:
- Findings are deduplicated (same `FindingId`)
- Done-records are both written (harmless duplication)
- No corruption: content-addressed identity ensures correctness

## When Idempotency Is Not Enough

Idempotency prevents duplicate operations, but it doesn't solve:

**1. Out-of-order delivery**: If operations A and B are sent, but B arrives before A, idempotency doesn't restore order. Solution: use **sequence numbers** or **vector clocks** to enforce causal order.

**2. Lost operations**: If an operation is never delivered (not just delayed), idempotency doesn't help. Solution: use **at-least-once delivery** (retry until acknowledged) + idempotency.

**3. Byzantine failures**: Idempotency assumes honest participants. If a client maliciously sends different payloads with the same OpId, the payload hash will detect the conflict, but the system may need additional authentication (HMAC, signatures) to prevent forgery.

## Testing Idempotency

**Property-based test**: Generate random operation sequences, inject duplicates, verify that outcomes are identical:

```rust
#[test]
fn test_checkpoint_idempotency() {
    let operations = vec![
        Checkpoint { cursor: 100 },
        Checkpoint { cursor: 100 },  // Duplicate
        Checkpoint { cursor: 150 },
        Checkpoint { cursor: 150 },  // Duplicate
    ];

    let mut state = ShardState::new();
    for op in operations {
        apply_operation(&mut state, op);
    }

    // Should be identical to applying without duplicates:
    let expected = ShardState::with_cursor(150);
    assert_eq!(state, expected);
}
```

**Chaos engineering**: Use fault injection (e.g., Toxiproxy) to simulate network failures, force retries, verify no corruption.

**Jepsen tests**: Distributed test framework that simulates partitions, clock skew, crashes. Analyzes operation histories to detect violations of idempotency invariants.

## References and Further Reading

- **Leach (2017)**: ["Implementing Stripe-like Idempotency Keys in Postgres"](https://brandur.org/idempotency-keys), blog post
- **IETF Draft**: ["The Idempotency-Key HTTP Header Field"](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header), standardizing idempotency keys for HTTP APIs
- **Gray & Reuter (1993)**: *Transaction Processing: Concepts and Techniques* (Chapter 4: Atomicity and idempotency)
- **Kleppmann (2017)**: *Designing Data-Intensive Applications*, Chapter 11 (stream processing, exactly-once semantics)
- **Akidau et al. (2015)**: ["The Dataflow Model"](https://research.google/pubs/pub43864/), VLDB (exactly-once semantics in stream processing)

---

**Next**: [05-failure-detection.md](./05-failure-detection.md) explains how Gossip-rs detects worker failures using lease expiry and error classification, enabling safe recovery without false positives.
