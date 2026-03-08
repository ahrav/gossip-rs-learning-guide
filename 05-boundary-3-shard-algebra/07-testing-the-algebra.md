# Testing the Algebra -- Property Tests and Invariant Coverage

This chapter walks through the test suite that verifies the shard algebra's correctness guarantees. The `gossip-frontier` crate has 1,842 lines of tests across three files, organized by module: key encoding (`key_encoding_tests.rs`, 510 lines), hint framing and propagation (`hint_tests.rs`, 790 lines), and the startup builder (`builder_tests.rs`, 542 lines). The tests use three complementary techniques: deterministic unit tests for specific edge cases, `rstest` parameterized tests for systematic boundary exploration, and `proptest` property tests for algebraic invariants that hold across all inputs.

---

The shard algebra's correctness rests on a small set of algebraic properties. Key encoding must preserve logical ordering when projected to bytes. Prefix successors must be strictly greater than their inputs and serve as tight upper bounds. Hint metadata must roundtrip through encode/decode without information loss. Split propagation must narrow correctly and reject out-of-bounds children. The builder must stage entries atomically, assign IDs sequentially, and detect manifest violations at build time. Each property has dedicated test coverage.

The test organization follows a deliberate layering strategy. Key encoding tests verify the lowest-level byte operations. Hint tests build on key encoding (using `ManifestRowKey` and `KeyBuf` as test infrastructure). Builder tests build on both (using hint constructors to create valid shard specs). This layering means that a failure in a lower layer produces failures in higher layers, creating a diagnostic cascade that points to the root cause.

## 1. Key Encoding Tests

The `key_encoding_tests.rs` file tests the foundational layer: the `KeyEncoding` trait, `PathKey`, `ManifestRowKey`, `prefix_successor`, `key_successor`, and `byte_midpoint`. These functions translate typed keys into comparable byte sequences (Chapter 1) and compute split boundaries (Chapter 2).

### The BytesKey Test Helper

The file defines a minimal `KeyEncoding` implementation for raw byte vectors:

```rust
#[derive(Clone, Debug, PartialEq, Eq, PartialOrd, Ord)]
struct BytesKey(Vec<u8>);

impl KeyEncoding for BytesKey {
    fn encode_into(&self, buf: &mut KeyBuf) {
        buf.copy_from_slice(&self.0);
    }
}
```

This helper exists solely for testing. Production code uses `PathKey` (UTF-8 file paths) and `ManifestRowKey` (fixed-width big-endian tuples). `BytesKey` provides a transparent pass-through that makes test assertions readable: the encoded bytes are exactly the input bytes.

### Deterministic Unit Tests

The unit tests pin specific behaviors that the property tests cannot easily target:

**Identity encoding.** `path_key_uses_identity_utf8_encoding` verifies that `PathKey` encodes as raw UTF-8 bytes. This is the fundamental property from Chapter 1 -- file path ordering in byte space must match logical path ordering.

```rust
#[test]
fn path_key_uses_identity_utf8_encoding() {
    let key = PathKey::new("src/lib.rs");
    let mut buf = KeyBuf::new();
    key.encode_into(&mut buf);
    assert_eq!(buf.as_bytes(), b"src/lib.rs");
}
```

**Fixed-width manifest keys.** `manifest_row_key_is_fixed_width_and_decodes` verifies that `ManifestRowKey` produces exactly `ENCODED_LEN` bytes (16: two big-endian `u64` fields) and that decoding recovers the original manifest ID and row number.

```rust
#[test]
fn manifest_row_key_is_fixed_width_and_decodes() {
    let key = ManifestRowKey::new(42, 99);
    let mut buf = KeyBuf::new();
    key.encode_into(&mut buf);
    assert_eq!(buf.len(), ManifestRowKey::ENCODED_LEN);
    let decoded = decode_manifest_row_key(buf.as_bytes()).expect("valid encoding");
    assert_eq!(decoded.manifest_id(), 42);
    assert_eq!(decoded.row(), 99);
}
```

**Rejection edge cases.** Empty paths, oversized paths, and non-fixed-width inputs are rejected:

```rust
#[test]
#[should_panic(expected = "PathKey path must not be empty")]
fn path_key_new_rejects_empty_path() {
    let _ = PathKey::new("");
}
```

**Prefix successor mechanics.** `prefix_successor_basic` pins the byte-level behavior:

```rust
#[test]
fn prefix_successor_basic() {
    let mut buf = KeyBuf::new();
    assert_eq!(prefix_successor(b"abc", &mut buf), Some(b"abd".as_slice()));
    assert_eq!(
        prefix_successor(b"ab\xff", &mut buf),
        Some(b"ac".as_slice())
    );
    assert_eq!(prefix_successor(b"\xff\xff", &mut buf), None);
    assert_eq!(prefix_successor(b"", &mut buf), None);
}
```

The last byte is incremented. If it is `0xFF`, it is dropped and the previous byte is incremented (the carry chain). If all bytes are `0xFF`, no successor exists. These are the edge cases from Chapter 2 that property tests exercise systematically.

**Key successor strategy.** `key_successor_prefers_append_when_capacity_available` verifies the append-zero strategy for keys below `MAX_KEY_SIZE`:

```rust
#[test]
fn key_successor_prefers_append_when_capacity_available() {
    let mut buf = KeyBuf::new();
    assert_eq!(key_successor(b"abc", &mut buf), Some(b"abc\0".as_slice()));
}
```

Appending `0x00` produces the smallest possible key strictly greater than the input. This is the tightest possible successor: no key exists between `"abc"` and `"abc\0"` in lexicographic byte order. At `MAX_KEY_SIZE`, appending is not possible (the key is already at maximum length), so `key_successor` falls back to `prefix_successor`, which increments the last non-`0xFF` byte and truncates the trailing `0xFF` bytes.

### Rstest Parameterized Tests

Two rstest-parameterized test functions systematically cover boundary conditions:

**`byte_midpoint_edge_cases`** tests eight input combinations for the midpoint function, including empty inputs, shared leading-zero prefixes, and different-length inputs:

```rust
#[rstest]
#[case::both_empty(&[], &[], None)]
#[case::empty_vs_nonempty(&[], &[0x02], Some(vec![0x01]))]
#[case::full_byte_range(&[0x00], &[0xFF], Some(vec![0x7F]))]
#[case::shared_leading_zero_prefix(&[0x00, 0x00], &[0x00, 0x02], Some(vec![0x00, 0x01]))]
#[case::second_byte_gap(&[0x01, 0x00], &[0x01, 0x04], Some(vec![0x01, 0x02]))]
#[case::different_lengths_close(&[0x01], &[0x01, 0x00], None)]
#[case::high_bytes_different_lengths(&[0xFF], &[0xFF, 0x02], Some(vec![0xFF, 0x01]))]
#[case::wide_gap_different_lengths(&[0x10], &[0x10, 0x80], Some(vec![0x10, 0x40]))]
fn byte_midpoint_edge_cases(#[case] a: &[u8], #[case] b: &[u8], #[case] expected: Option<Vec<u8>>) {
    let mut buf = KeyBuf::new();
    assert_eq!(byte_midpoint(a, b, &mut buf), expected.as_deref());
}
```

Each case name documents the edge condition being tested. The `different_lengths_close` case is particularly important: when `a = [0x01]` and `b = [0x01, 0x00]`, there is no byte sequence strictly between them (the only candidate would be `a` itself, which is not strictly greater). The midpoint function correctly returns `None`.

**`prefix_shard_error_display_contains`** tests the `Display` implementation for `PrefixShardError`, ensuring error messages contain expected substrings for diagnostics.

### Property Tests

The property tests verify algebraic invariants that must hold across all inputs. Each test is annotated with `miri_proptest_config()` to reduce case counts under Miri (the memory safety checker), ensuring the test suite runs under both normal and sanitized configurations.

**Prefix successor ordering.** Two properties together establish that `prefix_successor` computes a tight upper bound:

```rust
#[test]
fn prefix_successor_is_strictly_greater(prefix in proptest::collection::vec(any::<u8>(), 0..=32)) {
    let mut buf = KeyBuf::new();
    if let Some(succ) = prefix_successor(&prefix, &mut buf) {
        prop_assert!(succ > prefix.as_slice());
    } else {
        let no_successor = prefix.is_empty() || prefix.iter().all(|&byte| byte == u8::MAX);
        prop_assert!(no_successor);
    }
}

#[test]
fn prefix_successor_is_upper_bound_for_prefixed_keys(
    prefix in proptest::collection::vec(any::<u8>(), 1..=32),
    suffix in proptest::collection::vec(any::<u8>(), 0..=32),
) {
    prop_assume!(prefix.iter().any(|&byte| byte != u8::MAX));
    let mut buf = KeyBuf::new();
    let succ = prefix_successor(&prefix, &mut buf).expect("non-all-ff prefix has successor");

    let mut key = prefix.clone();
    key.extend_from_slice(&suffix);

    prop_assert!(key.as_slice() < succ);
}
```

The first property says: for any prefix, the successor is strictly greater (or does not exist, in which case the prefix is empty or all-`0xFF`). The second says: for any key that starts with the prefix, the key is strictly less than the successor. Together, these prove that the half-open interval `[prefix, successor)` contains exactly the keys that start with the prefix -- the correctness foundation for prefix shards (Chapter 2).

**Order-preserving encoding.** Two properties verify that encoding preserves logical ordering:

```rust
#[test]
fn path_key_encoding_preserves_logical_order(
    a in proptest::collection::vec(0u8..=127, 1..=64),
    b in proptest::collection::vec(0u8..=127, 1..=64),
) {
    let a = String::from_utf8(a).expect("ASCII bytes should always be valid UTF-8");
    let b = String::from_utf8(b).expect("ASCII bytes should always be valid UTF-8");
    prop_assume!(a < b);

    let mut a_buf = KeyBuf::new();
    let mut b_buf = KeyBuf::new();
    PathKey::new(&a).encode_into(&mut a_buf);
    PathKey::new(&b).encode_into(&mut b_buf);

    prop_assert!(a_buf.as_bytes() < b_buf.as_bytes());
}
```

If logical path `a < b`, then encoded `a < b` in byte order. The analogous property for `ManifestRowKey` verifies that tuple ordering `(manifest_id, row)` is preserved by the fixed-width big-endian encoding.

**Roundtrip canonicality.** `manifest_row_key_encode_decode_roundtrip` verifies that for all `(manifest_id, row)` pairs, encoding and then decoding recovers the original values. This is the canonicality property: every valid key has exactly one encoded representation.

**Midpoint strictness.** `byte_midpoint_is_strictly_between_when_present` verifies the core midpoint invariant from Chapter 2:

```rust
#[test]
fn byte_midpoint_is_strictly_between_when_present(
    a in proptest::collection::vec(any::<u8>(), 0..=32),
    b in proptest::collection::vec(any::<u8>(), 0..=32),
) {
    let mut buf = KeyBuf::new();
    if let Some(mid) = byte_midpoint(&a, &b, &mut buf) {
        prop_assert!(a.as_slice() < mid);
        prop_assert!(mid < b.as_slice());
    }
}
```

When a midpoint exists, it is strictly between the two inputs. When it does not exist (the inputs are adjacent or equal), the function returns `None`. This property guarantees that `byte_midpoint` produces valid split boundaries -- a midpoint that is not strictly interior to the range would create a degenerate child shard with zero keys.

Additional property tests cover `key_successor` comprehensively:

- **`key_successor_is_strictly_greater`** -- The successor is always greater than the input when it exists.
- **`key_successor_none_only_when_expected`** -- `None` is returned only for oversized keys or all-`0xFF` keys at exactly `MAX_KEY_SIZE`.
- **`key_successor_appends_zero_when_below_max`** -- Below `MAX_KEY_SIZE`, the successor is always `key ++ 0x00`.
- **`key_successor_delegates_to_prefix_successor_at_max`** -- At exactly `MAX_KEY_SIZE`, the result matches `prefix_successor`.

These four properties together characterize the complete behavior of `key_successor`: its strategy (append vs. increment), its domain (when it returns `Some` vs. `None`), and its relationship to `prefix_successor` at the size boundary.

The `shard_spec_from_manifest_range_rejects_inverted_or_degenerate` property verifies the inverse: for all `(manifest_id, a, b)` where `a >= b`, the manifest range constructor returns `InvertedRange`. This is the negative complement of the roundtrip property -- it ensures that invalid inputs are rejected rather than silently producing malformed specs.

## 2. Hint Tests

The `hint_tests.rs` file tests the metadata framing layer: `ShardHint` encode/decode, `ShardMetadata` envelope, convenience constructors, and `propagate_hint_on_split`. These functions ensure that shard type tags survive serialization and propagate correctly through splits (Chapters 3-4).

### Unit Roundtrips

Each hint variant has a dedicated roundtrip test:

```rust
#[test]
fn shard_hint_range_roundtrip() {
    let hint = ShardHint::Range;
    let encoded = encode_hint(hint);
    assert_eq!(encoded, vec![0x00]);

    let (decoded, consumed) = ShardHint::decode(&encoded).expect("range hint should decode");
    assert_eq!(decoded, hint);
    assert_eq!(consumed, encoded.len());
}
```

The range hint encodes as a single byte `0x00`. The prefix hint encodes as tag `0x01` followed by a 4-byte big-endian length and the prefix bytes. The manifest hint encodes as tag `0x02` followed by three big-endian `u64` fields (manifest ID, start row, end row). Each roundtrip test checks both the decoded value and the consumed byte count, ensuring the decoder advances exactly to the end of the frame.

### Decode Error Coverage

The tests systematically cover every decode rejection path:

```rust
#[test]
fn shard_hint_decode_rejects_empty_data() {
    assert_eq!(ShardHint::decode(&[]), Err(ShardHintDecodeError::EmptyData),);
}

#[test]
fn shard_hint_decode_rejects_unknown_tag() {
    assert_eq!(
        ShardHint::decode(&[0x7F]),
        Err(ShardHintDecodeError::UnknownTag(0x7F)),
    );
}

#[test]
fn shard_hint_decode_rejects_truncated_prefix_header() {
    assert_eq!(
        ShardHint::decode(&[0x01, 0x00, 0x00]),
        Err(ShardHintDecodeError::TruncatedPrefix {
            expected_min: 5,
            actual: 3,
        }),
    );
}
```

Empty input, unknown tags, truncated prefix headers, truncated prefix payloads, truncated manifest frames, inverted manifest rows, and equal manifest rows are all tested. Each test verifies the specific error variant and its diagnostic fields.

The encode/decode asymmetry is explicitly tested. `shard_hint_encode_rejects_inverted_manifest_rows` and `shard_hint_encode_rejects_equal_manifest_rows` verify that the encoder refuses to produce frames for invalid manifest hints. `shard_hint_decode_rejects_raw_inverted_manifest` verifies that the decoder independently rejects hand-crafted wire bytes with inverted rows, even though the encoder would never produce them. This double-sided testing ensures that neither the encoder nor the decoder relies on the other for correctness -- each independently validates its invariants, a defense-in-depth approach that protects against future refactors that change one side without updating the other.

### Metadata Envelope Tests

The `ShardMetadata` envelope wraps a `ShardHint` with connector-specific extra bytes (Chapter 3). The tests verify:

- Empty metadata defaults to `ShardHint::Range` with empty `connector_extra`. This default allows shards to omit metadata entirely without producing a decode error.
- Non-empty, non-conforming metadata is rejected (five specific malformed byte sequences covering truncated headers, incorrect length prefixes, and mismatched payload sizes).
- Encode/decode roundtrip preserves both the hint and the connector extra, verifying that the 4-byte length prefix and the hint frame are correctly composed.
- Oversized payloads are rejected at encode time with `MetadataTooLarge`, preventing callers from constructing metadata that would exceed the `MAX_METADATA_SIZE` limit of the shard spec.

### Split Propagation Tests

The `propagate_hint_on_split` tests verify the rules from Chapter 4. Each shard kind has distinct propagation semantics:

**Range stays range.** Splitting a range shard produces range children. There is no narrower type to propagate.

```rust
#[test]
fn propagate_hint_on_split_range_stays_range() {
    let propagated = propagate_hint_on_split(&ShardHint::Range, b"a", b"b")
        .expect("range propagation should always succeed");
    assert_eq!(propagated, ShardHint::Range);
}
```

**Prefix demotes to range.** Splitting a prefix shard produces range children, because a child of a prefix shard covers a sub-range of the prefix's key space -- it is no longer a full prefix match.

```rust
#[test]
fn propagate_hint_on_split_prefix_demotes_to_range() {
    let parent = ShardHint::Prefix { prefix: b"src/" };
    let propagated = propagate_hint_on_split(&parent, b"src/", b"src/z")
        .expect("valid prefix child bounds should propagate");
    assert_eq!(propagated, ShardHint::Range);
}
```

**Prefix rejects out-of-bounds children.** Two tests verify that child boundaries outside the prefix's key space are rejected:

```rust
#[test]
fn propagate_hint_on_split_prefix_rejects_out_of_range_start() {
    let parent = ShardHint::Prefix { prefix: b"src/" };
    let err = propagate_hint_on_split(&parent, b"aaa", b"src/z")
        .expect_err("out-of-range start should fail");
    assert_eq!(
        err,
        HintPropagationError::InvalidPrefixBoundary {
            boundary: SplitBoundary::Start
        }
    );
}
```

**Manifest narrows row range.** Splitting a manifest shard produces a manifest child with a narrower `[start_row, end_row)` interval. The test verifies that the child's row bounds are correctly extracted from the encoded key boundaries:

```rust
#[test]
fn propagate_hint_on_split_manifest_adjusts_child_rows() {
    let parent = ShardHint::Manifest {
        manifest_id: 7,
        start_row: 0,
        end_row: 100,
    };
    let child_start = encode_manifest_row_key(7, 25);
    let child_end = encode_manifest_row_key(7, 75);

    let propagated = propagate_hint_on_split(&parent, &child_start, &child_end)
        .expect("valid manifest child bounds should propagate");
    assert_eq!(
        propagated,
        ShardHint::Manifest {
            manifest_id: 7,
            start_row: 25,
            end_row: 75,
        }
    );
}
```

**Manifest rejects ID mismatch, out-of-bounds rows, empty ranges, and non-manifest boundaries.** Ten tests cover the complete error surface for manifest propagation. The tests systematically vary which validation check fails:

- `rejects_non_manifest_boundaries` -- the child's start key is not a valid `ManifestRowKey` encoding (raw bytes instead of 16-byte fixed-width).
- `rejects_manifest_id_mismatch` -- the child's manifest ID differs from the parent's (child has manifest 8, parent has manifest 7).
- `rejects_out_of_parent_bounds` -- the child's start row is below the parent's start row.
- `rejects_empty_child_range` -- `start_row == end_row`, producing a zero-width interval.
- `rejects_non_manifest_end_boundary` -- only the end key is non-decodable; the start key is valid.
- `rejects_end_manifest_id_mismatch` -- the end key has a different manifest ID than the parent.
- `rejects_end_row_beyond_parent` -- the child's end row exceeds the parent's end row.

Each test targets a single validation check, ensuring that every rejection path in `propagate_hint_on_split` is exercised. This one-test-per-check discipline makes it possible to verify that the function's error reporting is precise: a manifest ID mismatch on the start boundary produces `ManifestIdMismatch { parent: 7, child: 8 }`, not a generic "invalid boundary" message.

### Hint Property Tests

Three property tests verify algebraic invariants across random inputs:

**`shard_hint_proptest_roundtrip`** -- For all hint variants (generated by `arb_owned_hint()`), encoding and decoding recovers the original hint.

**`shard_metadata_proptest_roundtrip`** -- For all hint-and-connector-extra combinations, the full metadata envelope roundtrips.

**`prefix_shard_contains_all_keys_with_prefix`** -- For any non-all-`0xFF` prefix and any suffix, the key `prefix || suffix` falls within the shard's `[start, end)` range:

```rust
#[test]
fn prefix_shard_contains_all_keys_with_prefix(
    prefix in proptest::collection::vec(any::<u8>(), 1..=32),
    suffix in proptest::collection::vec(any::<u8>(), 0..=32),
) {
    prop_assume!(prefix.iter().any(|&byte| byte != u8::MAX));
    prop_assume!(prefix.len() + suffix.len() <= MAX_KEY_SIZE);

    let mut scratch = ShardSpecScratch::new();
    let spec = prefix_shard_ref(prefix.as_slice(), b"ctx", &mut scratch)
        .expect("prefix shard should be constructible for bounded prefix");
    let mut key = prefix.clone();
    key.extend_from_slice(&suffix);

    prop_assert!(key.starts_with(&prefix));
    prop_assert!(spec.contains_key(&key));
}
```

This property closes the loop between key encoding and hint construction: the prefix successor from Chapter 2 correctly bounds the key space, and the shard spec's `contains_key` method agrees with the prefix membership test. If this property failed, prefix shards would miss items that should be in their range.

## 3. Builder Tests

The `builder_tests.rs` file tests the `PreallocShardBuilder` from Chapters 5 and 6. The tests focus on public guarantees: add-path validation, staging/build-phase separation, split helper semantics, and lifecycle behavior.

### Mixed-Type Registration

```rust
#[test]
fn mixed_range_prefix_manifest_builds_valid_manifest() {
    let mut arena = ShardArena::with_capacity(8, 4_096);
    let mut builder = PreallocShardBuilder::<8>::new(&mut arena, 8).unwrap();
    builder.add_range(b"a", b"f", b"").unwrap();
    builder.add_prefix(b"m/", b"").unwrap();
    builder.add_manifest(7, 0, 10, b"").unwrap();

    let inputs = builder.build_inputs().unwrap();
    assert_eq!(inputs.len(), 3);
    validate_manifest(inputs.as_slice()).unwrap();
}
```

This test verifies that range, prefix, and manifest shards coexist in a single manifest. The `validate_manifest` call at the end re-runs the same validation that `build_inputs` performs internally -- a belt-and-suspenders check that the test's manifest is independently valid.

### Sequential ID Assignment

```rust
#[test]
fn ids_are_assigned_in_insertion_order() {
    let mut arena = ShardArena::with_capacity(8, 4_096);
    let mut builder = PreallocShardBuilder::<8>::new(&mut arena, 8).unwrap();
    builder.add_range(b"a", b"f", b"").unwrap();
    builder.add_prefix(b"m/", b"").unwrap();
    builder.add_manifest(7, 0, 10, b"").unwrap();

    let inputs = builder.build_inputs().unwrap();
    let ids: Vec<u64> = inputs.iter().map(|entry| entry.shard().as_raw()).collect();
    assert_eq!(ids, vec![0, 1, 2]);
}
```

Shard IDs are assigned monotonically from 0 in insertion order. This determinism is critical for reproducibility: given the same add sequence, the same shard IDs are produced.

### Phase Separation: Add-Time vs. Build-Time Errors

The tests demonstrate the two-phase validation design:

```rust
#[test]
fn overlapping_ranges_fail_on_build_validation() {
    let mut arena = ShardArena::with_capacity(4, 2_048);
    let mut builder = PreallocShardBuilder::<4>::new(&mut arena, 4).unwrap();
    builder.add_range(b"a", b"m", b"").unwrap();
    builder.add_range(b"f", b"z", b"").unwrap();

    let err = builder.build_inputs().unwrap_err();
    assert!(matches!(
        err,
        PreallocShardBuilderError::ManifestInvalid(
            ManifestValidationError::OverlappingRanges { .. }
        )
    ));
}
```

Both `add_range` calls succeed -- each range is individually valid (`a < m` and `f < z`). The overlap between `[a, m)` and `[f, z)` is only detectable when the full manifest is assembled. `build_inputs` catches it.

Similarly, `cursor_outside_range_fails_manifest_validation` shows that an out-of-bounds cursor is accepted at add time but rejected at build time:

```rust
#[test]
fn cursor_outside_range_fails_manifest_validation() {
    let mut arena = ShardArena::with_capacity(4, 2_048);
    let mut builder = PreallocShardBuilder::<4>::new(&mut arena, 4).unwrap();
    builder
        .add_range_with_cursor(b"a", b"f", b"", CursorUpdate::new(b"z"))
        .unwrap();
    let err = builder.build_inputs().unwrap_err();
    assert!(matches!(
        err,
        PreallocShardBuilderError::ManifestInvalid(
            ManifestValidationError::CursorOutOfBounds { .. }
        )
    ));
}
```

### Split Helper Guarantees

The split tests verify the transactional semantics from Chapter 6:

**No partial writes on capacity exhaustion.** When the entry budget is insufficient for all children, the builder rejects the entire split and remains empty:

```rust
#[test]
fn split_range_capacity_exceeded_has_no_partial_writes() {
    let mut arena = ShardArena::with_capacity(4, 4_096);
    let mut builder = PreallocShardBuilder::<512>::new(&mut arena, 3).unwrap();
    let split_points: Vec<&[u8]> = vec![b"f", b"m", b"t"];
    let err = builder
        .split_range_by_boundaries(b"a", &split_points, b"z", b"")
        .unwrap_err();

    assert!(matches!(
        err,
        PreallocShardBuilderError::CapacityExceeded {
            limit: 3,
            current: 0,
            additional: 4,
        }
    ));
    assert!(builder.is_empty());
}
```

Four children (3 split points + 1) exceed the entry limit of 3. The builder is still empty after the error.

**No partial writes on fan-out exhaustion.** The same guarantee holds for `MAX_SPLIT_CHILDREN`:

```rust
#[test]
fn split_range_fan_out_exceeded_has_no_partial_writes() {
    let mut arena = ShardArena::with_capacity(512, 4_096);
    let mut builder = PreallocShardBuilder::<512>::new(&mut arena, 512).unwrap();
    let split_points: Vec<&[u8]> = vec![b"m"; MAX_SPLIT_CHILDREN];
    let err = builder
        .split_range_by_boundaries(b"a", &split_points, b"z", b"")
        .unwrap_err();

    assert!(matches!(
        err,
        PreallocShardBuilderError::FanOutExceeded {
            limit,
            requested,
        } if limit == MAX_SPLIT_CHILDREN && requested == MAX_SPLIT_CHILDREN + 1
    ));
    assert!(builder.is_empty());
}
```

**Manifest row tiling.** `split_manifest_by_rows_exact_capacity_tiles_manifest_rows` verifies the chunking arithmetic from Chapter 6:

```rust
#[test]
fn split_manifest_by_rows_exact_capacity_tiles_manifest_rows() {
    let mut arena = ShardArena::with_capacity(3, 4_096);
    let mut builder = PreallocShardBuilder::<3>::new(&mut arena, 3).unwrap();
    builder.split_manifest_by_rows(7, 0, 10, 4, b"").unwrap();

    let inputs = builder.build_inputs().unwrap();
    assert_eq!(inputs.len(), 3);
    let rows: Vec<(u64, u64, u64)> = inputs
        .iter()
        .map(|entry| {
            let start = decode_manifest_row_key(entry.spec().key_range_start()).unwrap();
            let end = decode_manifest_row_key(entry.spec().key_range_end()).unwrap();
            (start.manifest_id(), start.row(), end.row())
        })
        .collect();
    assert_eq!(rows, vec![(7, 0, 4), (7, 4, 8), (7, 8, 10)]);
}
```

The manifest ID is preserved across all chunks. The rows tile perfectly: `[0,4)`, `[4,8)`, `[8,10)`. The last chunk is shorter (2 rows instead of 4) because the total span (10) is not evenly divisible by the chunk width (4).

**`u64` boundary safety.** A dedicated test verifies that row values near `u64::MAX` do not wrap:

```rust
#[test]
fn split_manifest_by_rows_handles_u64_boundary_without_wrap() {
    let mut arena = ShardArena::with_capacity(4, 4_096);
    let mut builder = PreallocShardBuilder::<4>::new(&mut arena, 4).unwrap();
    let start_row = u64::MAX - 5;
    let end_row = u64::MAX;
    builder
        .split_manifest_by_rows(99, start_row, end_row, 4, b"")
        .unwrap();

    let inputs = builder.build_inputs().unwrap();
    assert_eq!(inputs.len(), 2);
    let first_start =
        decode_manifest_row_key(inputs.as_slice()[0].spec().key_range_start()).unwrap();
    let first_end = decode_manifest_row_key(inputs.as_slice()[0].spec().key_range_end()).unwrap();
    let second_start =
        decode_manifest_row_key(inputs.as_slice()[1].spec().key_range_start()).unwrap();
    let second_end = decode_manifest_row_key(inputs.as_slice()[1].spec().key_range_end()).unwrap();

    assert_eq!(first_start.row(), start_row);
    assert_eq!(first_end.row(), start_row + 4);
    assert_eq!(second_start.row(), start_row + 4);
    assert_eq!(second_end.row(), end_row);
}
```

The saturating addition and end-clamping from Chapter 6 ensure correct behavior: `(u64::MAX - 5).saturating_add(4) = u64::MAX - 1`, clamped to `min(u64::MAX - 1, u64::MAX) = u64::MAX - 1`. The second chunk starts at `u64::MAX - 1` and ends at `u64::MAX`.

### Rstest Parameterized Builder Tests

The builder has one rstest-parameterized test function covering manifest row split fan-out:

```rust
#[rstest]
#[case::over_max_split_children(
    512, 512,
    u64::try_from(MAX_SPLIT_CHILDREN).unwrap() + 1,
    1, 4_096,
    MAX_SPLIT_CHILDREN + 1,
)]
#[case::rounding_overflow(
    512, 512, u64::MAX,
    {
        let target = u64::try_from(MAX_SPLIT_CHILDREN + 1).unwrap();
        (u64::MAX / target).saturating_add(1)
    },
    1_000_000,
    MAX_SPLIT_CHILDREN + 1,
)]
fn split_manifest_by_rows_fan_out_exceeded_has_no_partial_writes(
    #[case] arena_slots: usize,
    #[case] entry_limit: usize,
    #[case] end_row: u64,
    #[case] rows_per_shard: u64,
    #[case] arena_bytes: usize,
    #[case] exp_requested: usize,
) {
    let mut arena = ShardArena::with_capacity(arena_slots, arena_bytes);
    let mut builder = PreallocShardBuilder::<512>::new(&mut arena, entry_limit).unwrap();
    let err = builder
        .split_manifest_by_rows(7, 0, end_row, rows_per_shard, b"")
        .unwrap_err();

    assert!(matches!(
        err,
        PreallocShardBuilderError::FanOutExceeded {
            limit,
            requested,
        } if limit == MAX_SPLIT_CHILDREN && requested == exp_requested
    ));
    assert!(builder.is_empty());
}
```

The `rounding_overflow` case targets the ceiling division in `manifest_split_count`: when `end_row` is `u64::MAX` and `rows_per_shard` is chosen to produce exactly `MAX_SPLIT_CHILDREN + 1` chunks, the fan-out check must still reject the split. This catches off-by-one errors in the chunk count calculation.

### Configuration Validation

Two tests verify the constructor's rejection paths:

```rust
#[test]
fn config_rejects_zero_entry_limit() {
    let mut arena = ShardArena::with_capacity(4, 4_096);
    assert!(matches!(
        PreallocShardBuilder::<4>::new(&mut arena, 0),
        Err(PreallocShardBuilderError::EntryLimitZero)
    ));
}

#[test]
fn config_rejects_entry_limit_exceeding_cap() {
    let mut arena = ShardArena::with_capacity(4, 4_096);
    assert!(matches!(
        PreallocShardBuilder::<2>::new(&mut arena, 4),
        Err(PreallocShardBuilderError::CapMismatch {
            entry_limit: 4,
            cap: 2
        })
    ));
}
```

These tests pin the error variants and their payloads. The `CapMismatch` test uses `CAP = 2` with `entry_limit = 4`, demonstrating that the generic parameter is checked at runtime (the compile-time assertion only enforces `CAP <= 1024`).

### Error-Mapping Paths

The error-mapping tests verify that each shard kind maps its constructor errors to the correct builder error variant:

- `add_range_with_inverted_bounds_returns_range_invalid` -- inverted byte range produces `RangeInvalid`.
- `add_prefix_with_empty_prefix_returns_prefix_invalid` -- empty prefix produces `PrefixInvalid`.
- `add_manifest_with_inverted_rows_returns_manifest_ctor_invalid` -- inverted row range produces `ManifestCtorInvalid`.

These tests are simple but critical: they verify the `map_err` closures in each add method correctly translate inner errors to the appropriate builder error variant. A mapping error (e.g., accidentally wrapping a prefix error as `RangeInvalid`) would produce confusing diagnostics.

### Lifecycle and Idempotency

Two tests verify reset and idempotency behavior:

**`reset_allows_fresh_reuse_and_resets_ids`** verifies that after `reset()`, the builder accepts new entries, restarts shard IDs at 0, and produces a valid manifest. The test stages two range shards, resets, stages one prefix shard, and verifies that the new shard has ID 0 (not 2). This pins the ID-reset guarantee from Chapter 5.

**`build_inputs_is_idempotent`** verifies that calling `build_inputs` twice produces equivalent results without mutating builder state. The test compares shard IDs and key-range starts between the two calls. This pins the read-only guarantee: `build_inputs` is a view construction, not a consuming operation.

### Empty Builder Validation

```rust
#[test]
fn build_inputs_on_empty_builder_returns_manifest_empty() {
    let mut arena = ShardArena::with_capacity(4, 2_048);
    let builder = PreallocShardBuilder::<4>::new(&mut arena, 4).unwrap();
    let err = builder.build_inputs().unwrap_err();
    assert!(matches!(
        err,
        PreallocShardBuilderError::ManifestInvalid(ManifestValidationError::Empty)
    ));
}
```

An empty builder produces `ManifestInvalid(Empty)` at build time, not at construction time. This is consistent with the two-phase design: construction creates a valid builder; staging is optional; `build_inputs` validates the result. A zero-entry manifest is rejected because it would produce a run with no shards to scan -- a logical error that should be caught before reaching the coordinator.

## 4. Coverage Summary

| Module | Unit Tests | Parameterized (rstest) | Property Tests (proptest) | Total Functions |
|--------|-----------|----------------------|--------------------------|----------------|
| key_encoding | 21 | 2 (11 cases) | 13 | 36 |
| hint | 40 | 0 | 3 | 43 |
| builder | 31 | 1 (2 cases) | 0 | 32 |
| **Total** | **92** | **3 (13 cases)** | **16** | **111** |

The three test techniques serve complementary roles. Unit tests pin specific edge cases and error paths. Rstest parameterized tests systematically cover boundary conditions with named cases. Property tests verify algebraic invariants across random inputs that unit tests cannot enumerate.

The property tests are concentrated in the key encoding and hint modules because these modules define algebraic properties (ordering preservation, roundtrip canonicality, midpoint strictness, prefix containment) that are naturally expressed as universal quantifiers. The builder module's behavior is more stateful -- sequential add/build workflows with specific capacity limits and error sequences -- which lends itself to deterministic scenario tests rather than random-input properties.

All proptest-based tests use `miri_proptest_config()`, which reduces the number of generated cases when running under Miri (Rust's memory safety interpreter). Under normal test execution, proptest generates its default case count (256 cases per test). Under Miri, the count drops to a smaller number (typically 16-32) because Miri's interpretation overhead makes full case counts impractically slow. The reduced count is acceptable because Miri's value is in checking memory safety invariants (use-after-free, out-of-bounds access, uninitialized reads), not in statistical coverage -- a single Miri pass over a few cases catches safety violations that thousands of normal runs would miss.

### What the Tests Do Not Cover

The test suite does not include property tests for the builder module. This is a conscious gap, not an oversight. Property-based testing of stateful APIs (like the builder's add-stage-build workflow) requires a state-machine model that generates random operation sequences and checks invariants after each step. The B2 coordination section achieves this through the `CoordinationSim` harness ([Chapter 12](../04-boundary-2-coordination/12-proving-correctness.md)), which generates random acquire/checkpoint/split sequences and checks the S1-S7 invariants. A similar harness for the builder would generate random add/split/reset/build sequences and check that IDs are sequential, entries are non-overlapping, and capacity limits are respected. The existing deterministic tests cover the critical error paths comprehensively, so a full property-based state machine is deferred rather than omitted.

The test suite also does not include benchmarks. The builder operates on COLD paths (startup registration), where absolute performance is less critical than correctness. The project's allocation policy (from `CLAUDE.md`) explicitly permits straightforward allocation on COLD paths. Benchmark-gating applies to HOT paths (per-shard steady-state loops), which the builder does not participate in.

## 5. Completing the Shard Algebra

This completes the shard algebra boundary. Key encoding translates typed keys into comparable bytes (Chapter 1). Range arithmetic computes split boundaries and midpoints (Chapter 2). Hint metadata tags shards with their intended key domain and propagates correctly through splits (Chapters 3-4). The startup builder stages bulk registrations with transactional semantics (Chapters 5-6). Property tests verify the algebraic invariants across all modules (this chapter).

The coordination protocol in B2 (Chapters 7-9 of the [coordination section](../04-boundary-2-coordination/07-why-splits-exist.md)) uses these primitives when executing splits: `SplitReplacePlan` and `SplitResidualPlan` pass child boundaries through `propagate_hint_on_split`, and `register_shards` accepts inputs produced by `PreallocShardBuilder::build_inputs`. The shard algebra provides the mathematical foundation; the coordination protocol provides the distributed execution.
