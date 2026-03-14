# The Contract Enforcer -- Persistence Conformance Testing

*A team ships a new PostgreSQL persistence backend. Unit tests pass. Integration tests pass. The backend correctly inserts done-ledger records, retrieves them by key, and returns proper receipts. Three weeks into production, a monitoring alert fires: 847 items that were successfully scanned now show status `FailedRetryable`. Investigation reveals a race condition in the upsert path. Two workers scan the same item concurrently. Worker A finishes first and writes `ScannedWithFindings` with `bytes_scanned = 2,048` and `findings_count = 1`. Worker B, which failed on a network timeout, writes `FailedRetryable` with `bytes_scanned = 4,096` and `findings_count = 1` a few milliseconds later. The PostgreSQL backend's upsert uses `ON CONFLICT DO UPDATE SET status = EXCLUDED.status` -- it keeps the latest write, not the highest-rank status. The `ScannedWithFindings` entry is silently overwritten by `FailedRetryable`. The done-ledger's join-semilattice invariant requires that scanned states always dominate failed states, regardless of arrival order. No unit test caught this because every unit test wrote records in a single order and never tested the reverse. A single run of the conformance harness -- specifically check 3, `scan-then-fail dominance` -- would have caught the bug before the backend left the developer's laptop.*

---

Persistence backends have a contract. The done-ledger status forms a join-semilattice: scanned states dominate failed states. Findings writes are idempotent: replaying the same batch must not create duplicate durable rows. Occurrences must reference existing findings; observations must reference existing occurrences. A backend that violates any of these invariants will corrupt production data in ways that are invisible until a race condition or retry reveals the inconsistency.

Unit tests written by the backend author test what the author *thought* the contract required. They may miss edge cases that the author never considered. The conformance harness tests what the contract *actually* requires, using checks derived from the trait specifications themselves. Every backend -- in-memory, SQLite, PostgreSQL -- must pass the same suite.

The harness lives in a single file: `gossip-contracts/src/persistence/conformance.rs`. It depends only on the persistence traits and identity types. It has no knowledge of any specific backend implementation. It is the contract, executable.

The design follows a pattern common in database engine testing: define the behavioral contract independently of any implementation, construct synthetic fixtures that exercise edge cases, and verify observable outcomes against the specification. The harness is strict about what it can actually observe through the current persistence API. It does not test internal implementation details like index structures or caching behavior. It tests externally visible invariants: idempotency, lattice merge, referential integrity, atomicity of failure, and secret redaction.

## The Entry Point: run_conformance

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

pub fn run_conformance<L, F>(
    done_ledger: &L,
    findings: &F,
) -> Result<PersistenceConformanceReport, PersistenceConformanceError>
where
    L: DoneLedger,
    F: FindingsSink + FindingsConformanceProbe,
{
    let done_ledger_checks = run_done_ledger_conformance(done_ledger)?;
    let findings_checks = run_findings_conformance(findings)?;
    let redaction_checks = run_redaction_conformance()?;
    Ok(PersistenceConformanceReport {
        done_ledger_checks,
        findings_checks,
        redaction_checks,
    })
}
```

The function is generic over `L: DoneLedger` and `F: FindingsSink + FindingsConformanceProbe`. Any backend that implements these traits can be tested. The three sub-suites run sequentially, and execution stops at the first failure. A successful report means all eleven checks passed.

The sequential execution with early termination is a deliberate choice. In a conformance suite, the first failure is the most important: it identifies the contract violation that must be fixed before other checks become meaningful. Running all checks and collecting failures would produce a larger report, but the additional failures are often cascading consequences of the first one. The harness optimizes for actionability over completeness.

The findings backend must also implement `FindingsConformanceProbe` -- a narrow read-side trait that the harness uses to verify idempotency. The production `FindingsSink` API is write-only by design. Without a probe, a backend that silently duplicates rows on replay would still return `Ok(())` both times, and the harness would have no way to detect the violation.

## Fixture Isolation

Every test case constructs its own `SampleFixture` from a distinct seed byte:

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

struct SampleFixture {
    tenant_id: TenantId,
    policy_hash: PolicyHash,
    ovid_hash: OvidHash,
    finding: FindingRecord,
    occurrence: OccurrenceRecord,
    observation: ObservationRecord,
    scanned_record: DoneLedgerRecord,
    failed_record: DoneLedgerRecord,
}
```

Each identity field is derived from `fill32(seed + offset)` with a unique offset, producing non-overlapping key material across fixtures. This means test cases are independent of each other and of any pre-existing state in the backend. They can run against a shared backend instance without cross-contamination.

The fixture includes both a `scanned_record` and a `failed_record` sharing the same `DoneLedgerKey`. The `failed_record` intentionally uses a larger `bytes_scanned` (4,096) than the `scanned_record` (2,048). This is realistic -- a failure during a large-file scan followed by a successful scan of a smaller file -- and ensures the per-field max check in the dominance assertions is non-vacuous. A backend that simply returns the dominant record wholesale, instead of computing per-field maximums, will fail the check.

The `sample_fixture` function also constructs a complete three-layer findings hierarchy: a `FindingRecord`, an `OccurrenceRecord` that references it, and an `ObservationRecord` that references the occurrence. This referential closure is needed for the findings conformance checks, which submit complete batches and verify that the backend correctly validates parent-child relationships.

## Done-Ledger Conformance: Four Checks

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

pub fn run_done_ledger_conformance<L>(done_ledger: &L) -> Result<u32, PersistenceConformanceError>
where
    L: DoneLedger,
{
    check_done_ledger_idempotent(done_ledger)?;
    check_done_ledger_fail_then_scan(done_ledger)?;
    check_done_ledger_scan_then_fail(done_ledger)?;
    check_done_ledger_batch_get_positional(done_ledger)?;
    Ok(4)
}
```

### Check 1: Idempotent Upsert

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_done_ledger_idempotent<L>(done_ledger: &L) -> Result<(), PersistenceConformanceError>
where
    L: DoneLedger,
{
    let fixture = sample_fixture(0x11)?;
    let receipt = submit_done_ledger(
        done_ledger,
        "done-ledger/idempotent:first",
        std::slice::from_ref(&fixture.scanned_record),
    )?;
    if receipt.record_count() != 1
        || receipt.scanned_count() != 1
        || receipt.findings_count() != 1
    {
        return Err(PersistenceConformanceError::DoneLedgerInvariant {
            case: "done-ledger/idempotent:first",
            message: format!(
                "unexpected receipt counts: record_count={}, scanned_count={}, findings_count={}",
                receipt.record_count(),
                receipt.scanned_count(),
                receipt.findings_count(),
            ),
        });
    }
    let _ = submit_done_ledger(
        done_ledger,
        "done-ledger/idempotent:second",
        std::slice::from_ref(&fixture.scanned_record),
    )?;
    let fetched = get_single_done_ledger(
        done_ledger, "done-ledger/idempotent:fetch", &fixture
    )?;
    if fetched != fixture.scanned_record {
        return Err(PersistenceConformanceError::DoneLedgerInvariant {
            case: "done-ledger/idempotent:fetch",
            message: format!(
                "replaying the same record changed durable state: expected {:?}, got {:?}",
                fixture.scanned_record, fetched
            ),
        });
    }
    Ok(())
}
```

The harness writes a `ScannedWithFindings` record, verifies the receipt counts, replays the same record, reads it back, and asserts byte-for-byte equality. A backend that uses insert-or-replace instead of insert-or-merge would pass this check (the data is identical both times), but the check still validates receipt semantics and read-after-write consistency. The receipt verification is important: it confirms that the backend correctly counts the number of records, scanned entries, and findings in each batch, information that callers use for progress tracking and monitoring.

### Check 2: Fail-Then-Scan Dominance

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_done_ledger_fail_then_scan<L>(done_ledger: &L) -> Result<(), PersistenceConformanceError>
where
    L: DoneLedger,
{
    let fixture = sample_fixture(0x22)?;
    let _ = submit_done_ledger(
        done_ledger,
        "done-ledger/fail-then-scan:failed",
        std::slice::from_ref(&fixture.failed_record),
    )?;
    let _ = submit_done_ledger(
        done_ledger,
        "done-ledger/fail-then-scan:scanned",
        std::slice::from_ref(&fixture.scanned_record),
    )?;
    let fetched = get_single_done_ledger(
        done_ledger, "done-ledger/fail-then-scan:fetch", &fixture
    )?;
    assert_scanned_dominates(
        "done-ledger/fail-then-scan:fetch",
        &fixture.failed_record,
        &fixture.scanned_record,
        &fetched,
    )
}
```

This check writes a `FailedRetryable` record first, then a `ScannedWithFindings` record for the same key. The read-back must show the scanned status. The `assert_scanned_dominates` helper verifies five properties of the durable record: the key is unchanged, the status is a scanned variant, `bytes_scanned` has not regressed below the max of both inputs, `findings_count` has not regressed, and the status rank is at least as high as the scanned record's rank.

The five-property check is more thorough than a simple status comparison. A backend that returns the scanned record wholesale (ignoring the failed record's `bytes_scanned` of 4,096) will fail the `bytes_scanned` regression check, because the scanned record only has `bytes_scanned` of 2,048. The merge must compute per-field maximums.

### Check 3: Scan-Then-Fail Dominance

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_done_ledger_scan_then_fail<L>(done_ledger: &L) -> Result<(), PersistenceConformanceError>
where
    L: DoneLedger,
{
    let fixture = sample_fixture(0x33)?;
    let _ = submit_done_ledger(
        done_ledger,
        "done-ledger/scan-then-fail:scanned",
        std::slice::from_ref(&fixture.scanned_record),
    )?;
    let _ = submit_done_ledger(
        done_ledger,
        "done-ledger/scan-then-fail:failed",
        std::slice::from_ref(&fixture.failed_record),
    )?;
    let fetched = get_single_done_ledger(
        done_ledger, "done-ledger/scan-then-fail:fetch", &fixture
    )?;
    assert_scanned_dominates(
        "done-ledger/scan-then-fail:fetch",
        &fixture.failed_record,
        &fixture.scanned_record,
        &fetched,
    )
}
```

The reverse arrival order. A `ScannedWithFindings` record is written first, then a `FailedRetryable` record for the same key. The durable state must still show the scanned status. This is the check that catches the `ON CONFLICT DO UPDATE SET status = EXCLUDED.status` bug described in the opening scenario. A backend that simply keeps the latest write will fail here, because the latest write is the failure.

This check and Check 2 together prove the lattice property holds regardless of arrival order. The join of `FailedRetryable` and `ScannedWithFindings` is `ScannedWithFindings`, period. The two checks together are the minimum necessary to verify commutativity of the lattice join. A single-direction test would miss backends that implement the merge correctly in one direction but use last-write-wins in the other.

### Check 4: Batch-Get Positional Semantics

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_done_ledger_batch_get_positional<L>(
    done_ledger: &L,
) -> Result<(), PersistenceConformanceError>
where
    L: DoneLedger,
{
    let fixture = sample_fixture(0xAA)?;
    let _ = submit_done_ledger(
        done_ledger,
        "done-ledger/batch-get-positional:write",
        std::slice::from_ref(&fixture.scanned_record),
    )?;
    let absent_ovid_hash = derive_ovid_hash(&OvidHashInputs {
        stable_item_id: StableItemId::from_bytes(fill32(0xAA_u8.wrapping_add(20))),
        version: VersionId::Strong(ObjectVersionId::from_version_bytes(
            b"conformance-absent-version",
        )),
    });
    let rows = done_ledger
        .batch_get(
            fixture.tenant_id,
            fixture.policy_hash,
            &[fixture.ovid_hash, absent_ovid_hash],
        )
        .map_err(|err| PersistenceConformanceError::DoneLedgerGet {
            case: "done-ledger/batch-get-positional:fetch",
            source: Box::new(err),
        })?;
    if rows.len() != 2 { /* ... invariant error ... */ }
    if rows[0].is_none() { /* ... expected Some for written key ... */ }
    if rows[1].is_some() { /* ... expected None for absent key ... */ }
    Ok(())
}
```

`batch_get` with a two-element `ovid_hashes` slice must return a result vector of length two, where `result[0]` is `Some` for the written key and `result[1]` is `None` for an absent key. The positional alignment must hold regardless of which keys are present. This catches backends that return results in hash order instead of query order, or that filter out absent keys instead of returning `None` placeholders.

Positional semantics matter because callers use the result vector index to correlate responses with requests. A batch query for keys `[A, B, C]` must return results `[result_A, result_B, result_C]` in exactly that order. A backend that returns `[result_B, result_A, result_C]` or `[result_A, result_C]` (filtering out the absent `B`) would cause callers to misinterpret which keys exist and which do not. This check prevents that class of bug.

## Findings Conformance: Four Checks

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

pub fn run_findings_conformance<F>(findings: &F) -> Result<u32, PersistenceConformanceError>
where
    F: FindingsSink + FindingsConformanceProbe,
{
    check_findings_idempotent(findings)?;
    check_findings_orphan_occurrence(findings)?;
    check_findings_orphan_observation(findings)?;
    check_findings_observation_merge(findings)?;
    Ok(4)
}
```

### Check 1: Idempotent Upsert with Durable Count Verification

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs (abbreviated)

fn check_findings_idempotent<F>(findings: &F) -> Result<(), PersistenceConformanceError>
where
    F: FindingsSink + FindingsConformanceProbe,
{
    let fixture = sample_fixture(0x44)?;
    let before = probe_counts(findings, "findings/idempotent:before")?;
    let batch = fixture.findings_batch();
    let receipt = submit_findings(findings, "findings/idempotent:first", batch)?;
    if receipt.finding_count() != 1
        || receipt.occurrence_count() != 1
        || receipt.observation_count() != 1
    { /* ... invariant error ... */ }
    let after_first = probe_counts(findings, "findings/idempotent:after-first")?;
    let delta = after_first.saturating_sub(before);
    if delta != (DurableFindingsCounts { findings: 1, occurrences: 1, observations: 1 })
    { /* ... invariant error ... */ }

    let _ = submit_findings(
        findings, "findings/idempotent:replay", fixture.findings_batch()
    )?;
    let after_replay = probe_counts(findings, "findings/idempotent:after-replay")?;
    if after_replay != after_first {
        return Err(PersistenceConformanceError::FindingsInvariant {
            case: "findings/idempotent:after-replay",
            message: format!(
                "replaying the same batch changed durable counts: \
                 before replay {:?}, after replay {:?}",
                after_first, after_replay
            ),
        });
    }
    Ok(())
}
```

This is the most thorough idempotency check in the suite. It writes a complete three-layer batch (finding + occurrence + observation), verifies receipt counts, snapshots durable counts, confirms the delta is exactly `(1, 1, 1)`, replays the identical batch, and verifies that durable counts are unchanged after the replay. The probe-based before/after comparison is essential: without it, a backend that silently duplicates rows would appear correct because `upsert_batch` returns `Ok(())` both times.

### Check 2: Orphan Occurrence Rejection

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_findings_orphan_occurrence<F>(findings: &F) -> Result<(), PersistenceConformanceError>
where
    F: FindingsSink + FindingsConformanceProbe,
{
    let fixture = sample_fixture(0x55)?;
    let occurrences = [fixture.occurrence.clone()];
    assert_orphan_rejected(
        findings,
        "findings/missing-finding",
        FindingsUpsertBatch::new(&[], &occurrences, &[]),
    )?;
    Ok(())
}
```

Submitting an occurrence without its parent finding must fail. The `assert_orphan_rejected` helper snapshots durable counts before and after the expected failure, then asserts they are identical -- proving the rejected write left no partial rows behind:

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn assert_orphan_rejected<F>(
    findings: &F,
    case: &'static str,
    batch: FindingsUpsertBatch<'_>,
) -> Result<(), PersistenceConformanceError>
where
    F: FindingsSink + FindingsConformanceProbe,
{
    let before = probe_counts(findings, case)?;
    expect_findings_failure(findings, case, batch)?;
    let after = probe_counts(findings, case)?;
    if after != before {
        return Err(PersistenceConformanceError::FindingsInvariant {
            case,
            message: format!(
                "failed write changed durable counts: before {:?}, after {:?}",
                before, after
            ),
        });
    }
    Ok(())
}
```

The `expect_findings_failure` helper accepts failure at either phase: submission (`upsert_batch` returns `Err`) or durability (`handle.wait()` returns `Err`). Both are acceptable -- the harness only reports a violation if the write durably succeeds. This flexibility accommodates backends that validate referential integrity at different points in their pipeline.

### Check 3: Orphan Observation Rejection

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_findings_orphan_observation<F>(findings: &F) -> Result<(), PersistenceConformanceError>
where
    F: FindingsSink + FindingsConformanceProbe,
{
    let fixture = sample_fixture(0x66)?;
    let observations = [fixture.observation.clone()];
    assert_orphan_rejected(
        findings,
        "findings/missing-occurrence",
        FindingsUpsertBatch::new(&[], &[], &observations),
    )?;
    Ok(())
}
```

Same principle as orphan occurrence rejection, one level deeper in the hierarchy. An observation without its parent occurrence must be rejected, and durable counts must remain unchanged. The check reuses `assert_orphan_rejected` -- the only difference is the batch composition.

### Check 4: Observation Upsert Merge

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn check_findings_observation_merge<F>(findings: &F) -> Result<(), PersistenceConformanceError>
where
    F: FindingsSink + FindingsConformanceProbe,
{
    let fixture = sample_fixture(0x88)?;
    let batch = fixture.findings_batch();
    let _ = submit_findings(findings, "findings/observation-merge:first", batch)?;
    let after_first = probe_counts(findings, "findings/observation-merge:after-first")?;

    let replayed_observation = ObservationRecord::new(
        fixture.observation.tenant_id(),
        fixture.observation.occurrence_id(),
        fixture.observation.policy_hash(),
        fixture.observation.ovid_hash(),
        fixture.observation.run_id(),
        fixture.observation.shard_id(),
        fixture.observation.fence_epoch(),
        LogicalTime::from_raw(fixture.observation.seen_at().as_raw() + 1000),
    );
    let findings_slice = [fixture.finding.clone()];
    let occurrences_slice = [fixture.occurrence.clone()];
    let observations_slice = [replayed_observation];
    let replay_batch = FindingsUpsertBatch::new(
        &findings_slice, &occurrences_slice, &observations_slice
    );
    let _ = submit_findings(findings, "findings/observation-merge:replay", replay_batch)?;
    let after_replay = probe_counts(findings, "findings/observation-merge:after-replay")?;

    if after_replay.observations != after_first.observations {
        return Err(PersistenceConformanceError::FindingsInvariant {
            case: "findings/observation-merge:after-replay",
            message: format!(
                "replaying an observation with the same observation_id but different seen_at \
                 created a duplicate row: observations before replay {}, after replay {}",
                after_first.observations, after_replay.observations
            ),
        });
    }
    Ok(())
}
```

This check writes a complete batch, then replays the observation with the same `observation_id` but a `seen_at` timestamp that is 1,000 logical ticks later. The finding and occurrence are replayed too (they must pass referential integrity), but the key assertion is on the observation count: it must not increase. The backend must merge the observation (updating provenance) rather than inserting a duplicate row.

This catches a common SQL mistake: using `INSERT` with a unique constraint violation handler that silently drops the second write instead of updating provenance fields. The observation count would be correct (no duplicate), but the `seen_at` timestamp would not advance. The current harness checks counts only -- a future enhancement could verify that the merged timestamp reflects the later value.

The check includes the parent finding and occurrence in the replay batch. This is necessary because the backend must validate referential integrity on every submission, and an observation-only batch would fail the orphan-occurrence check. The harness bundles parents with the replayed observation to isolate the test to observation merge behavior specifically, not referential integrity (which is tested separately in checks 2 and 3).

## Redaction Conformance: Three Checks

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

pub fn run_redaction_conformance() -> Result<u32, PersistenceConformanceError> {
    let mut checks = 0u32;
    let fixture = sample_fixture(0x77)?;
    let norm = NormHash::from_digest([0xE7; 32]);
    let norm_debug = format!("{norm:?}");
    assert_no_raw_encoding("redaction/norm-hash-debug", &norm_debug, norm.as_bytes())?;
    checks += 1;

    let secret_hash = fixture.finding.secret_hash();
    let secret_debug = format!("{secret_hash:?}");
    assert_no_raw_encoding(
        "redaction/secret-hash-debug",
        &secret_debug,
        secret_hash.as_bytes(),
    )?;
    checks += 1;

    let finding_debug = format!("{:?}", fixture.finding);
    assert_no_raw_encoding(
        "redaction/finding-record-debug",
        &finding_debug,
        secret_hash.as_bytes(),
    )?;
    checks += 1;

    Ok(checks)
}
```

The redaction suite verifies that `Debug` implementations on sensitive persistence types do not leak raw secret bytes. This is a defense-in-depth check: secret-derived types should use redacted or truncated debug output so that structured logging and error messages do not accidentally expose key material.

Three types are checked: `NormHash` (the normalized secret hash), `SecretHash` (the keyed, tenant-scoped secret hash), and `FindingRecord` (the aggregate record containing a `SecretHash` field). The `assert_no_raw_encoding` helper checks three encodings:

1. **Full-length hex** -- the complete lowercase hex encoding of the byte sequence.
2. **Prefix hex** -- the first 8 bytes (16 hex characters). This catches the `define_id_32!` macro's 4-byte Debug format (`TypeName(e7e7e7e7..)`) with margin, guarding against accidental use of an unrestricted macro on a sensitive type.
3. **Base64** -- standard base64 encoding of the full byte sequence.

The redaction suite does not require a backend instance. It runs entirely in memory against the type system's `Debug` implementations. This makes it fast and deterministic, but it also means the checks verify type-level redaction only. A backend that logs raw bytes through a different code path (e.g., SQL query logging) would not be caught by this suite. Defense in depth requires additional measures at the infrastructure layer.

## The FindingsConformanceProbe Trait

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

pub trait FindingsConformanceProbe: Send + Sync {
    /// Backend-specific probe error.
    type Error: Error + Send + Sync + 'static;

    /// Return durable row counts currently visible through the backend.
    fn durable_counts(&self) -> Result<DurableFindingsCounts, Self::Error>;
}
```

The production `FindingsSink` API is write-only by design. That is correct for production, but it means pure write-only testing cannot observe whether replay created duplicate durable rows. A backend could accept the same batch twice and silently duplicate storage while still returning `Ok(())` both times.

The probe trait adds a narrow read surface -- just enough for the conformance harness to snapshot row counts before and after writes. Backend crates typically implement this on their in-memory test double or directly on the integration-test wrapper around the real store. Production code paths never reference this trait.

The `DurableFindingsCounts` type mirrors the three-layer data model:

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct DurableFindingsCounts {
    pub findings: u64,
    pub occurrences: u64,
    pub observations: u64,
}

impl DurableFindingsCounts {
    pub const fn saturating_sub(self, base: Self) -> Self {
        Self {
            findings: self.findings.saturating_sub(base.findings),
            occurrences: self.occurrences.saturating_sub(base.occurrences),
            observations: self.observations.saturating_sub(base.observations),
        }
    }
}
```

The `saturating_sub` method computes per-layer deltas between before and after snapshots. Saturating arithmetic avoids panic if a backend resets counts between snapshots -- the delta clamps to zero rather than wrapping. This defensive choice matters for backends with eventual consistency or snapshot isolation, where a count query between two writes might see an intermediate state.

The "currently visible" contract in `durable_counts` means the counts should reflect all writes whose `CommitHandle::wait` has already returned successfully. Backends that buffer writes asynchronously must ensure the probe sees flushed state. For the in-memory backends, this is trivially satisfied because all mutations happen under the same lock that protects the count query.

## The Report Structure

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct PersistenceConformanceReport {
    pub done_ledger_checks: u32,
    pub findings_checks: u32,
    pub redaction_checks: u32,
}

impl PersistenceConformanceReport {
    pub const fn total_checks(self) -> u32 {
        self.done_ledger_checks + self.findings_checks + self.redaction_checks
    }
}

impl fmt::Display for PersistenceConformanceReport {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "persistence conformance: {} done-ledger, {} findings, {} redaction ({} total)",
            self.done_ledger_checks,
            self.findings_checks,
            self.redaction_checks,
            self.total_checks(),
        )
    }
}
```

Each field counts the number of checks that passed in that sub-suite. The harness returns `Err` on the first failure, so a successful report means all checks passed. Current totals: 4 done-ledger, 4 findings, 3 redaction, 11 total.

## The Error Taxonomy

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

pub enum PersistenceConformanceError {
    FixtureConstruction { case: &'static str, source: BoxError },
    DoneLedgerSubmit { case: &'static str, source: BoxError },
    DoneLedgerWait { case: &'static str, source: BoxError },
    DoneLedgerGet { case: &'static str, source: BoxError },
    DoneLedgerInvariant { case: &'static str, message: String },
    FindingsSubmit { case: &'static str, source: BoxError },
    FindingsWait { case: &'static str, source: BoxError },
    FindingsInvariant { case: &'static str, message: String },
    Probe { case: &'static str, source: BoxError },
    RedactionLeak { case: &'static str, debug_output: String, leaked_fragment: String },
}
```

Every variant carries a `case` label formatted as `"sub-suite/test:phase"` (e.g., `"done-ledger/idempotent:fetch"`) so failures can be traced back to the exact check and phase that produced them.

Variants split into two categories. **Operational failures** (`*Submit`, `*Wait`, `*Get`, `Probe`, `FixtureConstruction`) indicate the backend returned an error where the harness expected success. These carry a boxed source error for diagnostics, preserving the original error chain so developers can trace the root cause through the backend's internal error types. **Invariant violations** (`*Invariant`, `RedactionLeak`) indicate the backend succeeded but the observable result violated a persistence contract. These carry a human-readable message describing the violation, including the specific values that were expected and observed.

The `RedactionLeak` variant includes both the full debug output and the leaked fragment, but the `Display` implementation truncates the fragment to 8 characters to avoid leaking the very secret it detected:

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

Self::RedactionLeak { case, leaked_fragment, .. } => {
    let truncated = if leaked_fragment.len() > 8 {
        format!("{}...", &leaked_fragment[..8])
    } else {
        leaked_fragment.clone()
    };
    write!(
        f,
        "sensitive Debug output leaked raw bytes in {case}: \
         leaked fragment `{truncated}` (full output redacted)"
    )
}
```

## Internal Helpers

The harness uses several internal helper functions to reduce repetition and ensure consistent error handling.

### submit_done_ledger and submit_findings

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn submit_done_ledger<L>(
    done_ledger: &L,
    case: &'static str,
    records: &[DoneLedgerRecord],
) -> Result<DoneLedgerCommitReceipt, PersistenceConformanceError>
where
    L: DoneLedger,
{
    let handle = done_ledger.batch_upsert(records).map_err(|err| {
        PersistenceConformanceError::DoneLedgerSubmit {
            case,
            source: Box::new(err),
        }
    })?;
    handle.wait().map_err(|err| {
        PersistenceConformanceError::DoneLedgerWait {
            case,
            source: Box::new(err),
        }
    })
}
```

Both helpers wrap the two-phase `batch_upsert` / `upsert_batch` then `handle.wait()` flow, mapping submission errors and durability errors into distinct `PersistenceConformanceError` variants with the provided `case` label. The case label follows the `"sub-suite/test:phase"` convention, making it possible to identify the exact failure point in a multi-phase check.

### assert_scanned_dominates

The most complex assertion helper, used by both dominance checks:

```rust
// From crates/gossip-contracts/src/persistence/conformance.rs

fn assert_scanned_dominates(
    case: &'static str,
    failed: &DoneLedgerRecord,
    scanned: &DoneLedgerRecord,
    actual: &DoneLedgerRecord,
) -> Result<(), PersistenceConformanceError> {
    if actual.key() != scanned.key() { /* key changed */ }
    if !actual.status().is_scanned() { /* not scanned */ }
    if actual.bytes_scanned() < failed.bytes_scanned().max(scanned.bytes_scanned()) {
        /* bytes_scanned regressed */
    }
    if actual.findings_count() < failed.findings_count().max(scanned.findings_count()) {
        /* findings_count regressed */
    }
    if actual.status().rank() < scanned.status().rank() {
        /* status rank regressed */
    }
    Ok(())
}
```

Five properties are verified: key stability, scanned status, `bytes_scanned` monotonicity, `findings_count` monotonicity, and status rank monotonicity. Each property has its own error message identifying which specific metric regressed and by how much. The five-property decomposition is deliberate: it catches backends that get some properties right while violating others. A backend that correctly merges status but ignores `bytes_scanned` will pass the status check but fail the `bytes_scanned` check, producing a diagnostic message that pinpoints the exact regression.

## Summary

| Sub-suite | Check | What it catches |
|---|---|---|
| Done-ledger | Idempotent upsert | Non-idempotent upsert, incorrect receipts |
| Done-ledger | Fail-then-scan dominance | Last-write-wins instead of lattice merge |
| Done-ledger | Scan-then-fail dominance | Status downgrade on late failure write |
| Done-ledger | Batch-get positional | Wrong result ordering, missing `None` placeholders |
| Findings | Idempotent upsert | Duplicate row creation on replay |
| Findings | Orphan occurrence | Missing referential integrity for findings |
| Findings | Orphan observation | Missing referential integrity for occurrences |
| Findings | Observation merge | Duplicate observation rows on provenance update |
| Redaction | NormHash debug | Raw secret bytes in `Debug` output |
| Redaction | SecretHash debug | Raw secret bytes in `Debug` output |
| Redaction | FindingRecord debug | Raw secret bytes leaked through container type |

The conformance harness encodes eleven invariants that every persistence backend must satisfy. The done-ledger checks enforce the status join-semilattice and positional batch-get semantics. The findings checks enforce idempotency, referential integrity, and observation merge semantics. The redaction checks enforce secret-safe `Debug` output. A backend that passes all eleven checks implements the persistence contract correctly -- not because it was tested against a specific scenario, but because it was tested against the contract itself.

## What's Next

The in-memory backends covered in the previous chapter are the first implementations to pass this conformance suite. The PostgreSQL backends (`gossip-done-ledger-postgres` and `gossip-findings-postgres`) also pass all eleven checks, validating that the lattice merge, idempotent replay, referential integrity, and redaction contracts hold against a real database with SQL-based upsert logic -- not just in-memory data structures. When additional backends are added (SQLite or otherwise), they run the same eleven checks. The conformance harness is the single source of truth for what "correct persistence" means in this system. No backend ships without passing it.
