# "The Degenerate Bisection" -- Range Arithmetic for Split Planning

*A split planner bisects shard `[0x01, 0x02)`. It averages the boundary bytes:
`(0x01 + 0x02) / 2 = 0x01`. The midpoint equals the start. The planner
creates child shard `[0x01, 0x01)` -- an empty range -- and child shard
`[0x01, 0x02)` -- the original range, unchanged. The split accomplished
nothing. The planner retries, gets the same result: the arithmetic midpoint
of two adjacent single-byte values, when truncated to a single byte, always
lands on the lower boundary. The shard remains a monolith. Worker A's scan
takes 47 minutes for this single shard while 15 other workers sit idle,
waiting for work. The run that should have completed in 3 minutes blocks on
one shard that cannot be subdivided. The split planner has a correct formula
for averaging -- and a fundamentally wrong model of what "between" means in
a variable-length byte keyspace.*

---

## Why Range Arithmetic Is Hard

Recall from Chapter 1 that the coordinator operates on half-open byte
intervals `[start, end)`. Split planning requires computing a point strictly
between `start` and `end` to divide a shard into two children. In a
fixed-width integer keyspace, this is trivial: `(start + end) / 2`. But byte
keys are variable-length strings, not fixed-width integers. The space between
`[0x01]` and `[0x02]` is not one value apart -- it contains every key that
starts with `0x01` followed by any suffix: `[0x01, 0x00]`, `[0x01, 0x00, 0x00]`,
`[0x01, 0xFF, 0xFF]`, and so on. The midpoint of this space is not a
single-byte value. It lives in a longer key.

The `key_encoding` module provides three pure functions that handle this
arithmetic correctly:

1. **`prefix_successor`** -- computes the exclusive upper bound for a prefix
   scan. Given `b"abc"`, it returns `b"abd"`. This is used to construct
   prefix-bounded shards.

2. **`key_successor`** -- computes the minimal strict successor of an
   arbitrary key. Given `b"abc"` it returns `b"abc\x00"`. This is the
   fallback when arithmetic midpoint computation lands on a boundary.

3. **`byte_midpoint`** -- computes an approximate bisection point between two
   keys. Given `[0x40]` and `[0xC0]`, it returns `[0x80]`. Given `[0x01]`
   and `[0x02]`, it returns `[0x01, 0x00]` -- extending into a longer key to
   find a value strictly between the inputs.

All three return `Option` -- `None` signals that the requested successor or
midpoint does not exist within representable bounds. All three write their
output into a caller-provided `KeyBuf` and return a borrowed slice, following
the zero-allocation convention established in Chapter 1.

---

## prefix_successor -- The Upper Bound of a Prefix

The `prefix_successor` function answers: given a byte prefix, what is the
smallest byte string that is strictly greater than every key starting with
that prefix? This is the exclusive upper bound for a prefix scan.

Here is the definition from `key_encoding.rs`:

```rust
/// Compute the lexicographic successor of a byte prefix.
///
/// Returns the smallest byte string strictly greater than `prefix` that also
/// acts as an exclusive upper bound for *every* key that starts with `prefix`.
/// Formally, if `Some(succ)` is returned then for every key `k`:
///
/// ```text
/// k.starts_with(prefix)  =>  k < succ
/// ```
///
/// # Algorithm
///
/// 1. Find the rightmost byte that is not `0xFF`.
/// 2. Truncate the prefix to include that byte, discarding the trailing
///    `0xFF` suffix.
/// 3. Increment the last remaining byte by one.
///
/// Truncation is essential: incrementing the full key without truncating
/// trailing `0xFF` bytes first would produce a longer string via carry
/// propagation that is not a tight upper bound (it would leave a gap).
/// Truncation + increment produces the shortest possible successor.
///
/// # Returns `None`
///
/// - Empty prefix: no bytes to increment.
/// - All-`0xFF` prefix: the prefix already covers the top of the keyspace;
///   there is no byte string that can serve as an exclusive upper bound.
/// - Prefix length exceeds [`KeyBuf::CAPACITY`].
///
/// # Output buffer semantics
///
/// The returned slice aliases `buf`; reusing `buf` for another operation
/// overwrites that output.
///
/// # Complexity
///
/// `O(prefix.len())` time, no heap allocation.
///
/// # Examples
///
/// ```text
/// let mut buf = KeyBuf::new();
/// prefix_successor(b"abc", &mut buf)     => Some(b"abd")
/// prefix_successor(b"ab\xff", &mut buf)  => Some(b"ac")      // trailing 0xFF stripped
/// prefix_successor(b"\xff", &mut buf)    => None              // top of keyspace
/// prefix_successor(b"", &mut buf)        => None              // nothing to increment
/// ```
pub fn prefix_successor<'a>(prefix: &[u8], buf: &'a mut KeyBuf) -> Option<&'a [u8]> {
    let result_len = {
        let scratch: &mut [u8; MAX_KEY_SIZE] = (&mut buf.buf[..MAX_KEY_SIZE]).try_into().unwrap();
        prefix_successor_into(prefix, scratch)?.len()
    };
    buf.len = result_len;
    Some(buf.as_bytes())
}
```

The implementation delegates to `prefix_successor_into` from
`gossip-contracts::coordination`, which contains the canonical algorithm. The
wrapper adapts it to the `KeyBuf` interface. The algorithm itself is three
steps:

**Step 1: Find the rightmost non-0xFF byte.** Scan from the end of the
prefix toward the beginning, looking for a byte that is not `0xFF`. If every
byte is `0xFF`, the prefix covers the top of the keyspace and there is no
successor. Return `None`.

**Step 2: Truncate.** Discard everything after (and including) the trailing
`0xFF` bytes. This shortens the result to end at the rightmost non-`0xFF`
byte.

**Step 3: Increment.** Add 1 to the last remaining byte.

The truncation step is what distinguishes this from naive increment. Consider
the prefix `b"ab\xff"`:

```text
Naive approach (wrong):
  b"ab\xff" + 1 = b"ab\xff" with carry = b"ac\x00"
  But b"ac\x00" is NOT the tightest upper bound.
  The key b"ac" (without the \x00) is smaller and still greater than
  every key starting with b"ab\xff".

Truncate-then-increment (correct):
  1. Rightmost non-0xFF byte: 'b' at index 1
  2. Truncate to b"ab"
  3. Increment last byte: b"ac"
  Result: b"ac"
```

The difference matters: `b"ac\x00"` leaves a gap. Keys like `b"ac"` itself
(with no suffix) fall between `b"ab\xff..."` and `b"ac\x00"`. The truncated
successor `b"ac"` is the tightest possible upper bound -- there is no byte
string between `b"ab\xff\xff\xff..."` and `b"ac"`.

Here are the four cases from the doc comment, worked step by step:

```text
prefix_successor(b"abc"):
  Rightmost non-0xFF: 'c' (0x63) at index 2
  Truncate to b"abc" (no trailing 0xFF to strip)
  Increment: 0x63 + 1 = 0x64 = 'd'
  Result: b"abd"

prefix_successor(b"ab\xff"):
  Rightmost non-0xFF: 'b' (0x62) at index 1
  Truncate to b"ab" (strip trailing \xff)
  Increment: 0x62 + 1 = 0x63 = 'c'
  Result: b"ac"

prefix_successor(b"\xff"):
  No non-0xFF byte found
  Result: None

prefix_successor(b""):
  Empty prefix, nothing to scan
  Result: None
```

This algorithm is not original to Gossip-rs. It is identical to
FoundationDB's `strinc` operation and CockroachDB's `Key.PrefixEnd` method.
The doc comment explicitly notes this lineage. The algorithm is a natural
consequence of lexicographic ordering on variable-length byte strings: the
tightest upper bound for a prefix is always the shortest string that sorts
immediately after all extensions of that prefix.

---

## key_successor -- The Minimal Strict Successor

Where `prefix_successor` computes the upper bound for all extensions of a
prefix, `key_successor` computes the smallest key strictly greater than a
single given key. The distinction matters: `prefix_successor(b"abc")` returns
`b"abd"` (greater than `b"abc..."` for any suffix), while
`key_successor(b"abc")` returns `b"abc\x00"` (the next key after `b"abc"`
itself).

Here is the definition from `key_encoding.rs`:

```rust
/// Compute the smallest representable key that is strictly greater than `key`.
///
/// This primitive derives an exclusive
/// end-bound from an inclusive key, subject to the system-wide key size
/// limit ([`MAX_KEY_SIZE`] = 4 KiB).
///
/// # Strategy
///
/// | Condition | Action | Rationale |
/// |---|---|---|
/// | `key.len() < MAX_KEY_SIZE` | Append `0x00` | Minimal extension: `key` is a proper prefix of `key \|\| [0x00]`, so it sorts immediately after `key` but before any other extension sharing the same prefix. |
/// | `key.len() == MAX_KEY_SIZE` | Delegate to [`prefix_successor`] | Cannot grow; must increment in place. |
/// | `key.len() > MAX_KEY_SIZE` | Return `None` | Key already violates the size ceiling. |
///
/// Appending `0x00` is preferred over incrementing because it never fails
/// (any byte string can be extended with a zero byte), whereas incrementing
/// fails for the all-`0xFF` case.
///
/// # Returns `None`
///
/// - Key exceeds `MAX_KEY_SIZE`.
/// - Key is exactly `MAX_KEY_SIZE` bytes and all bytes are `0xFF` (no
///   successor exists within the representable keyspace).
///
/// # Output buffer semantics
///
/// The returned slice aliases `buf`; reusing `buf` overwrites that output.
///
/// # Complexity
///
/// `O(key.len())` time, no heap allocation.
pub fn key_successor<'a>(key: &[u8], buf: &'a mut KeyBuf) -> Option<&'a [u8]> {
    let result_len = {
        let scratch: &mut [u8; MAX_KEY_SIZE] = (&mut buf.buf[..MAX_KEY_SIZE]).try_into().unwrap();
        key_successor_into(key, scratch)?.len()
    };
    buf.len = result_len;
    Some(buf.as_bytes())
}
```

Like `prefix_successor`, this function delegates to a canonical
implementation in `gossip-contracts` and adapts it to the `KeyBuf` interface.

The strategy table from the doc comment defines three cases:

**Case 1: `key.len() < MAX_KEY_SIZE`.** Append `0x00`. The result is
`key || [0x00]`, which is the minimal strict successor. This works because
in lexicographic ordering, `"abc"` < `"abc\x00"` < `"abc\x00\x00"` < `"abd"`.
The appended `0x00` byte is the smallest possible extension, so no key can
exist between `"abc"` and `"abc\x00"`.

**Case 2: `key.len() == MAX_KEY_SIZE`.** The key is already at maximum
length and cannot be extended. Fall back to `prefix_successor`, which
increments the rightmost non-`0xFF` byte (with truncation). This may fail
if all bytes are `0xFF`.

**Case 3: `key.len() > MAX_KEY_SIZE`.** The key exceeds the system ceiling.
Return `None`. This key should not exist in the system, and computing a
successor for it is meaningless.

Appending `0x00` is preferred over incrementing for a specific reason: append
never fails (any byte sequence can be extended), while increment fails when
the last byte is `0xFF` and requires the truncation logic of
`prefix_successor`. The strategy table ensures that the simpler, always-
successful operation is used whenever possible.

The relationship between `key_successor` and `prefix_successor` is precise:

```text
key_successor(b"abc")     = b"abc\x00"        (append 0x00)
prefix_successor(b"abc")  = b"abd"            (truncate + increment)

key_successor skips over:     nothing (immediate next key)
prefix_successor skips over:  b"abc\x00", b"abc\x01", ..., b"abc\xff",
                              b"abc\x00\x00", ... (all extensions of b"abc")
```

`key_successor` produces the immediate next key. `prefix_successor` produces
the key after all possible extensions. For split planning, both are useful:
`byte_midpoint` uses `key_successor` as a fallback when arithmetic bisection
fails.

---

## byte_midpoint -- Bisecting a Key Range

The core split-planning function computes a key strictly between two
boundaries, dividing a shard's range into two approximately equal halves.

Here is the definition from `key_encoding.rs`:

```rust
/// Compute a midpoint key in the open interval `(a, b)`.
///
/// Designed for split planning to bisect a shard's key range. The caller can
/// then create two child shards covering `[parent_start, mid)` and
/// `[mid, parent_end)`.
///
/// # Algorithm
///
/// The shorter input is right-padded with `0x00` to
/// `max_len = max(a.len(), b.len())`, then the function executes this exact
/// sequence:
///
/// 1. **Add** padded `a + b` from LSB to MSB with carry, yielding
///    `max_len` or `max_len + 1` bytes.
/// 2. **Halve** that sum by long division from MSB to LSB.
/// 3. **Try overflow-normalized candidate**: when the quotient is
///    `max_len + 1` bytes and begins with `0x00`, drop exactly that one
///    leading byte and return it if `a < candidate < b`.
/// 4. **Try fixed-width candidate**: test the unmodified quotient bytes and
///    return them if they do not exceed [`MAX_KEY_SIZE`] and `a < candidate < b`.
/// 5. **Fallback successor**: if both arithmetic candidates fail, compute
///    `key_successor(a)` and return it only if it remains `< b`.
///
/// No other canonicalization is applied (notably, no general leading-zero
/// trimming).
///
/// The successor fallback handles dense lexicographic ranges where the
/// arithmetic midpoint lands on a boundary, e.g. `[0x01]..[0x02]` returns
/// `[0x01, 0x00]`.
///
/// # Returns `None`
///
/// - `a >= b` (precondition violated; includes both-empty inputs).
/// - Either input exceeds [`MAX_KEY_SIZE`].
/// - Neither arithmetic candidate is strictly inside `(a, b)`, and
///   `key_successor(a)` is missing or `>= b`.
///
/// # Output buffer semantics
///
/// The returned midpoint aliases `out`; subsequent writes to `out` replace it.
///
/// # Complexity
///
/// `O(max(a.len(), b.len()))` time using stack-resident scratch buffers
/// (bounded by [`KeyBuf::CAPACITY`]), with no heap allocation.
pub fn byte_midpoint<'a>(a: &[u8], b: &[u8], out: &'a mut KeyBuf) -> Option<&'a [u8]> {
    if a >= b {
        return None;
    }

    let max_len = a.len().max(b.len());
    if max_len == 0 || max_len > MAX_KEY_SIZE {
        return None;
    }

    // Phase 1: Big-endian addition from LSB to MSB with direct positional writes.
    let mut sum = KeyBuf::new();
    let mut carry: u16 = 0;
    for idx in (0..max_len).rev() {
        let a_byte = if idx < a.len() { u16::from(a[idx]) } else { 0 };
        let b_byte = if idx < b.len() { u16::from(b[idx]) } else { 0 };
        let total = a_byte + b_byte + carry;
        sum.buf[idx + 1] = (total & 0xFF) as u8;
        carry = total >> 8;
    }
    sum.buf[0] = carry as u8;
    sum.len = max_len + 1;

    // Phase 2: Divide by 2 from MSB to LSB.
    let mut remainder: u16 = 0;
    for idx in 0..sum.len {
        let value = (remainder << 8) | u16::from(sum.buf[idx]);
        sum.buf[idx] = (value / 2) as u8;
        remainder = value % 2;
    }

    // Phase 3: Validate arithmetic candidates.
    if sum.len == max_len + 1 && sum.buf[0] == 0 {
        let normalized = &sum.buf[1..sum.len];
        if normalized > a && normalized < b {
            out.copy_from_slice(normalized);
            return Some(out.as_bytes());
        }
    }

    let arithmetic = &sum.buf[..sum.len];
    if arithmetic.len() <= MAX_KEY_SIZE && arithmetic > a && arithmetic < b {
        out.copy_from_slice(arithmetic);
        return Some(out.as_bytes());
    }

    // Phase 4: If neither arithmetic candidate is interior, use the minimal
    // strict successor of `a`.
    let mut successor_buf = KeyBuf::new();
    let successor = key_successor(a, &mut successor_buf)?;
    if successor < b {
        out.copy_from_slice(successor);
        return Some(out.as_bytes());
    }

    None
}
```

This function is 56 lines of arithmetic. Each phase deserves careful
attention.

### Phase 1: Big-Endian Addition

The function treats both keys as big-endian integers. The shorter key is
implicitly right-padded with `0x00` bytes. Addition proceeds from the least
significant byte (rightmost) to the most significant byte (leftmost),
propagating carry.

The result is stored in a `KeyBuf` of length `max_len + 1` -- one byte
longer than the inputs to accommodate a possible carry from the most
significant position. The carry byte is written to `sum.buf[0]`, and the
remaining bytes occupy `sum.buf[1..max_len+1]`.

### Phase 2: Long Division by 2

The sum is divided by 2 using schoolbook long division from MSB to LSB. At
each position, the algorithm combines the remainder from the previous
position (shifted left by 8 bits) with the current byte, divides by 2, and
propagates the remainder.

After this phase, `sum` contains the arithmetic average `(a + b) / 2`,
represented as a `max_len + 1` byte big-endian value.

### Phase 3: Validate Arithmetic Candidates

The code labels both candidate checks as a single "Phase 3" (see
`key_encoding.rs`, line 597). Two candidates are tried in order:

**Overflow-normalized candidate.** If the sum produced a carry (the result is
`max_len + 1` bytes) and that carry divided to `0x00`, the leading zero can
be dropped. The normalized candidate is `sum.buf[1..sum.len]` -- the
original-width representation. If this value is strictly between `a` and `b`,
it is returned.

**Fixed-width candidate.** If the overflow-normalized candidate did not work
(or there was no overflow), the function tries the full-width quotient
`sum.buf[..sum.len]`. If this value does not exceed `MAX_KEY_SIZE` and falls
strictly between `a` and `b`, it is returned.

### Phase 4: Fallback to key_successor

If neither arithmetic candidate produces a value strictly inside `(a, b)`,
the function falls back to `key_successor(a)` (see `key_encoding.rs`,
line 612). This handles the degenerate case from the opening scenario: when
`a` and `b` are adjacent single-byte values, the arithmetic midpoint lands on
`a` itself. The successor fallback extends `a` by appending `0x00`, producing
a value inside the range.

---

## Worked Example: The Easy Case

Computing `byte_midpoint([0x40], [0xC0])`:

```text
Inputs: a = [0x40], b = [0xC0]
max_len = 1

Phase 1: Addition (right to left)
  idx=0: a_byte=0x40 (64), b_byte=0xC0 (192), carry=0
         total = 64 + 192 + 0 = 256 = 0x100
         sum.buf[1] = 0x00, carry = 1
  Final: sum.buf[0] = 0x01 (carry)
  sum = [0x01, 0x00], len = 2

Phase 2: Division by 2 (left to right)
  idx=0: value = (0 << 8) | 0x01 = 1
         sum.buf[0] = 1/2 = 0, remainder = 1
  idx=1: value = (1 << 8) | 0x00 = 256
         sum.buf[1] = 256/2 = 128 = 0x80, remainder = 0
  sum = [0x00, 0x80], len = 2

Phase 3: Overflow-normalized candidate
  sum.len (2) == max_len + 1 (2) and sum.buf[0] == 0x00
  normalized = [0x80]
  Check: [0x80] > [0x40] and [0x80] < [0xC0]  =>  true
  Return [0x80]
```

The result `[0x80]` (128) is exactly the arithmetic average of `[0x40]` (64)
and `[0xC0]` (192). This is the straightforward case where fixed-width
integer arithmetic works directly.

---

## Worked Example: The Degenerate Case

Computing `byte_midpoint([0x01], [0x02])` -- the scenario from the opening:

```text
Inputs: a = [0x01], b = [0x02]
max_len = 1

Phase 1: Addition (right to left)
  idx=0: a_byte=0x01, b_byte=0x02, carry=0
         total = 1 + 2 + 0 = 3
         sum.buf[1] = 0x03, carry = 0
  Final: sum.buf[0] = 0x00
  sum = [0x00, 0x03], len = 2

Phase 2: Division by 2 (left to right)
  idx=0: value = (0 << 8) | 0x00 = 0
         sum.buf[0] = 0, remainder = 0
  idx=1: value = (0 << 8) | 0x03 = 3
         sum.buf[1] = 3/2 = 1 = 0x01, remainder = 1
  sum = [0x00, 0x01], len = 2

Phase 3: Overflow-normalized candidate
  sum.len (2) == max_len + 1 (2) and sum.buf[0] == 0x00
  normalized = [0x01]
  Check: [0x01] > [0x01]?  =>  false (equal, not strictly greater)
  Skip.

Phase 3 (cont.): Fixed-width candidate
  arithmetic = [0x00, 0x01]
  Check: [0x00, 0x01] > [0x01]?  =>  false (0x00 < 0x01)
  Skip.

Phase 4: Fallback to key_successor
  key_successor([0x01]) = [0x01, 0x00]  (append 0x00)
  Check: [0x01, 0x00] < [0x02]?  =>  true
  Return [0x01, 0x00]
```

The arithmetic midpoint of 1 and 2 is 1.5, which truncates to 1 -- equal to
the start. Neither arithmetic candidate works. The `key_successor` fallback
extends `[0x01]` to `[0x01, 0x00]`, which is strictly between `[0x01]` and
`[0x02]` in lexicographic order. The split planner can now create two children:
child 1 with `start=[0x01], end=[0x01, 0x00]` and child 2 with
`start=[0x01, 0x00], end=[0x02]` (both half-open intervals).

This fallback is the answer to the opening failure. A naive bisection that
stays within fixed-width arithmetic cannot find a midpoint between adjacent
values. By extending into a longer key, the successor fallback exploits the
variable-length nature of the byte keyspace to always find a split point --
as long as the range contains more than one key.

---

## Worked Example: Different-Length Inputs

Computing `byte_midpoint([0x00, 0x00], [0x00, 0x02])`:

```text
Inputs: a = [0x00, 0x00], b = [0x00, 0x02]
max_len = 2

Phase 1: Addition (right to left)
  idx=1: a=0x00, b=0x02, carry=0 => total=2, sum.buf[2]=0x02, carry=0
  idx=0: a=0x00, b=0x00, carry=0 => total=0, sum.buf[1]=0x00, carry=0
  Final: sum.buf[0] = 0x00
  sum = [0x00, 0x00, 0x02], len = 3

Phase 2: Division by 2
  idx=0: value=0, quotient=0, remainder=0
  idx=1: value=0, quotient=0, remainder=0
  idx=2: value=2, quotient=1, remainder=0
  sum = [0x00, 0x00, 0x01], len = 3

Phase 3: Overflow-normalized candidate
  sum.len (3) == max_len + 1 (3) and sum.buf[0] == 0x00
  normalized = [0x00, 0x01]
  Check: [0x00, 0x01] > [0x00, 0x00] and [0x00, 0x01] < [0x00, 0x02]
  => true and true
  Return [0x00, 0x01]
```

The result `[0x00, 0x01]` is the natural midpoint between `[0x00, 0x00]` and
`[0x00, 0x02]`. The overflow normalization correctly strips the leading zero
that arose from the addition, returning a value with the same width as the
inputs.

---

## Precondition Checks

Before any arithmetic begins, `byte_midpoint` validates its inputs:

```rust
    if a >= b {
        return None;
    }

    let max_len = a.len().max(b.len());
    if max_len == 0 || max_len > MAX_KEY_SIZE {
        return None;
    }
```

Four conditions cause early `None`:

1. **`a >= b`** -- The range is empty or inverted. There is no interior point.
   This also covers both-empty inputs (`[] >= []` is true).

2. **`max_len == 0`** -- Both inputs are empty. Covered by `a >= b` but
   checked explicitly for clarity.

3. **`max_len > MAX_KEY_SIZE`** -- At least one input exceeds the system key
   size ceiling. The arithmetic would produce a result that cannot be stored
   in a `ShardSpec`.

---

## PrefixShardError -- Failure Modes for Prefix Conversion

When converting a user-supplied prefix into a bounded key range, several
things can go wrong. The `PrefixShardError` enum captures all of them,
ordered from cheapest to most expensive to detect.

Here is the definition from `key_encoding.rs`:

```rust
/// Error for invalid prefix-based shard operations.
///
/// Models failures when converting a user-supplied byte prefix into a valid
/// `[prefix, prefix_successor)` key range for shard construction. Each variant
/// represents a distinct failure mode, ordered roughly from cheapest to most
/// expensive to detect:
///
/// 1. [`EmptyPrefix`](Self::EmptyPrefix) -- trivially invalid.
/// 2. [`PrefixTooLarge`](Self::PrefixTooLarge) -- violates the system key-size ceiling.
/// 3. [`NoSuccessor`](Self::NoSuccessor) -- the prefix is valid but has no
///    lexicographic successor (all `0xFF`), so the half-open range cannot be
///    bounded.
/// 4. [`InvalidShardSpec`](Self::InvalidShardSpec) -- the derived range failed
///    downstream validation in [`ShardSpec`].
#[derive(Debug, Clone, PartialEq, Eq)]
#[non_exhaustive]
pub enum PrefixShardError {
    /// Prefix cannot be empty for prefix-only shard operations.
    EmptyPrefix,
    /// Prefix exceeds the shard-key size ceiling ([`MAX_KEY_SIZE`]).
    PrefixTooLarge {
        /// Actual prefix size in bytes.
        size: usize,
        /// Maximum allowed key size in bytes.
        max: usize,
    },
    /// Prefix has no lexicographic successor (all bytes are `0xFF`), so the
    /// exclusive end-bound of the half-open range cannot be computed.
    NoSuccessor,
    /// The derived key range passed local validation but failed downstream
    /// [`ShardSpec`] construction.
    InvalidShardSpec(ShardSpecInputError),
}

impl fmt::Display for PrefixShardError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::EmptyPrefix => write!(f, "prefix shard requires a non-empty prefix"),
            Self::PrefixTooLarge { size, max } => {
                write!(f, "prefix too large ({size} bytes, max {max})")
            }
            Self::NoSuccessor => {
                write!(
                    f,
                    "prefix has no lexicographic successor (all bytes are 0xFF)"
                )
            }
            Self::InvalidShardSpec(err) => write!(f, "{err}"),
        }
    }
}

impl std::error::Error for PrefixShardError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::InvalidShardSpec(err) => Some(err),
            _ => None,
        }
    }
}

impl From<ShardSpecInputError> for PrefixShardError {
    fn from(value: ShardSpecInputError) -> Self {
        Self::InvalidShardSpec(value)
    }
}
```

The four variants form a detection pipeline, ordered by cost:

**1. `EmptyPrefix`** -- A zero-length prefix has no bytes to form a range
start. Detected immediately by checking `prefix.is_empty()`. Cost: O(1).

**2. `PrefixTooLarge { size, max }`** -- The prefix exceeds `MAX_KEY_SIZE`
(4096 bytes). Detected by comparing `prefix.len()` to the ceiling. The
variant stores both the actual size and the maximum, making error messages
self-describing. Cost: O(1).

**3. `NoSuccessor`** -- The prefix is valid in length but consists entirely
of `0xFF` bytes. The `prefix_successor` function returns `None` because there
is no byte string that can serve as the exclusive upper bound -- the prefix
already covers the top of the keyspace. Detected by calling
`prefix_successor` and checking for `None`. Cost: O(prefix.len()).

**4. `InvalidShardSpec(ShardSpecInputError)`** -- The prefix and its
successor passed all local checks, but the resulting `[prefix,
prefix_successor)` range failed `ShardSpec::validate_ref`. This catches
edge cases where the derived range violates invariants that only the
`ShardSpec` validation layer knows about. Cost: O(prefix.len()) plus
whatever `validate_ref` does.

The ordering matters: cheap checks run first. If the prefix is empty, there
is no point computing its successor. If it exceeds `MAX_KEY_SIZE`, there is
no point checking whether its bytes are all `0xFF`. Each variant short-
circuits before reaching more expensive logic.

The `#[non_exhaustive]` attribute allows new variants to be added in the
future without breaking downstream matches. The `From<ShardSpecInputError>`
impl enables the `?` operator in `shard_spec_from_prefix_ref` to convert
downstream validation errors automatically.

---

## Comparison with Other Systems

The range arithmetic in `key_encoding` follows patterns established by
several distributed databases that use half-open byte-range partitioning:

| System | Successor Operation | Reference |
|--------|-------------------|-----------|
| FoundationDB | `strinc` (identical truncate+increment) | Apple, 2013 |
| CockroachDB | `Key.PrefixEnd` (same algorithm) | CockroachDB docs |
| Bigtable | Row key splitting | Chang et al., OSDI 2006 |
| Spanner | Half-open key-range splits | Corbett et al., OSDI 2012 |

The `prefix_successor` algorithm is identical to FoundationDB's `strinc`:
find the rightmost non-`0xFF` byte, truncate, increment. CockroachDB's
`Key.PrefixEnd` implements the same logic. This is not coincidence -- it is
the only correct algorithm for computing a tight exclusive upper bound over a
prefix in a lexicographic byte keyspace.

The `byte_midpoint` function is more novel. Bigtable and Spanner split
ranges, but their split-point selection uses statistics (data size, access
frequency) rather than pure key arithmetic. Gossip-rs uses `byte_midpoint`
as a deterministic fallback when no statistical hint is available: given only
the range boundaries, compute the approximate center.

The key insight shared across all these systems is that byte keys are not
integers. They are variable-length strings in an ordered alphabet. Operations
that work on fixed-width integers (averaging, incrementing) must be adapted
to handle carries, length changes, and the asymmetry between prefix extension
(always possible below `MAX_KEY_SIZE`) and in-place increment (fails at
`0xFF` boundaries).

---

## The Relationship Between the Three Functions

The three range arithmetic functions form a dependency chain used by the
split planner:

```text
                    +-------------------+
                    |   Split Planner   |
                    +-------------------+
                            |
                   needs a midpoint
                            |
                            v
                    +-------------------+
                    |   byte_midpoint   |
                    +-------------------+
                     /              \
             Phase 3:             Phase 4:
             arithmetic          fallback
                |                   |
                v                   v
        (self-contained      +-------------------+
         add + divide)       |  key_successor    |
                             +-------------------+
                                    |
                             at MAX_KEY_SIZE:
                                    |
                                    v
                             +-------------------+
                             | prefix_successor  |
                             +-------------------+
```

`byte_midpoint` is the top-level entry point. It tries arithmetic bisection
first (phases 1-3). When arithmetic fails -- typically for adjacent
single-byte values -- it falls back to `key_successor`. `key_successor` in
turn falls back to `prefix_successor` when the key is at maximum length.

This layering ensures that a split point can always be found if one exists.
The only case where `byte_midpoint` returns `None` is when `a >= b` (empty
or inverted range), the inputs exceed `MAX_KEY_SIZE`, or `a` and `b` are
truly adjacent with no representable key between them -- a condition that
requires `a` to be at `MAX_KEY_SIZE` length with all `0xFF` bytes, which
`key_successor` also cannot handle.

---

## Summary

Range arithmetic in `key_encoding` solves the problem that byte keys are not
fixed-width integers. `prefix_successor` computes tight exclusive upper bounds
for prefix scans using the truncate-then-increment algorithm shared with
FoundationDB and CockroachDB. `key_successor` computes minimal strict
successors by appending `0x00` when possible. `byte_midpoint` bisects a
key range using big-endian addition and long division, falling back to
`key_successor` when arithmetic lands on a boundary.

Together, these three functions give the split planner the ability to
subdivide any non-degenerate shard range -- including the `[0x01, 0x02)`
case that defeats naive averaging.

With key encoding and range arithmetic in place, the next layer up is hint
metadata: how does a shard carry information about its intended key domain
through splits? Chapter 3 covers `ShardHint`, `ShardMetadata`, and the
`propagate_hint_on_split` function that preserves connector-specific
information when a shard divides.
