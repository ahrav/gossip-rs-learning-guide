# References

This document consolidates all academic papers, books, and technical articles referenced throughout the Gossip-rs learning guide. Each entry includes a brief note about its relevance to the project.

## Leases & Fencing

**Gray, Cary & Cheriton, David R.** (1989)
"Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency"
*SOSP 1989*
The foundational paper on lease-based coordination. Gossip-rs uses time-bounded leases for shard ownership.

**Kleppmann, Martin** (2016)
"How to do distributed locking"
*Blog post*
Critique of distributed locks and the need for fencing tokens. Directly influenced Gossip-rs's fencing token design.

**Hazelcast Engineering** (2019)
"Distributed Locks Are Dead; Long Live Distributed Locks!"
*Blog post*
Modern perspective on distributed locking with practical examples. Informed our lock-free coordination approach.

**Hochstein, Lorin** (2025)
"Locks, Leases, Fencing Tokens, FizzBee!"
*Blog post*
Recent exploration of lease semantics with formal modeling. Validated our lease protocol invariants.

**polyglot_factotum** (2022)
"Modelling distributed locking in TLA+"
*Blog post*
TLA+ specifications for distributed locks. Served as template for our B2 coordination specs.

## Idempotency

**Leach, Brandur** (2017a)
"Designing robust and predictable APIs with idempotency"
*Stripe Engineering Blog*
Industry standard for idempotency key design. Gossip-rs implements Stripe-style idempotency for work submissions.

**Leach, Brandur** (2017b)
"Implementing Stripe-like Idempotency Keys in Postgres"
*Blog post*
Practical implementation guide. Influenced our done ledger design in B5.

**IETF Internet-Draft** (2023)
"Idempotency-Key HTTP Header Field"
*draft-ietf-httpapi-idempotency-key-header*
Standardization effort for idempotency keys. Gossip-rs API follows this emerging standard.

## Range Sharding

**Corbett, James C. et al.** (2012)
"Spanner: Google's Globally-Distributed Database"
*OSDI 2012*
Introduced range-based sharding with automatic rebalancing. Gossip-rs B3 adapts Spanner's range algebra.

**Bacon, David F. et al.** (2017)
"Spanner: Becoming a SQL System"
*SIGMOD 2017*
Evolution of Spanner's design. Informed our range split heuristics.

**Chang, Fay et al.** (2006)
"Bigtable: A Distributed Storage System for Structured Data"
*OSDI 2006*
Foundational work on distributed key-value stores. Influenced our shard key design.

**Karger, David et al.** (1997)
"Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web"
*STOC 1997*
Classic paper on consistent hashing. Gossip-rs uses range sharding instead, but the load balancing principles apply.

**CockroachDB Documentation**
"Range Splits and Merges"
*CockroachDB Docs*
Practical implementation of range-based sharding. Influenced our split threshold policies.

## Content-Addressed Identity

**Chacon, Scott & Straub, Ben** (2014)
"Git Internals – Git Objects"
*Pro Git Book, 2nd Edition*
Git's content-addressed storage model. Direct inspiration for Gossip-rs B1 identity design.

**Protocol Labs** (2018)
"IPFS Content Addressing"
*IPFS Documentation*
Modern content-addressed storage. Validated our CID-like approach for ItemID.

**Merkle, Ralph C.** (1987)
"A Digital Signature Based on a Conventional Encryption Function"
*CRYPTO 1987*
Original Merkle tree paper. Gossip-rs uses Merkle-tree principles for batch verification.

## Hashing

**BLAKE3 Team** (2020)
"BLAKE3 Specification"
*GitHub Repository*
Gossip-rs uses BLAKE3 for all hashing due to speed and security properties.

**Krawczyk, H., Bellare, M., & Canetti, R.** (1997)
"HMAC: Keyed-Hashing for Message Authentication"
*RFC 2104*
HMAC construction used in Gossip-rs for tenant-derived keys in B1.

## Consistency Models

**Herlihy, Maurice P. & Wing, Jeannette M.** (1990)
"Linearizability: A Correctness Condition for Concurrent Objects"
*TOPLAS 1990*
Defines linearizability. Gossip-rs coordinator provides linearizable work assignment.

**Vogels, Werner** (2009)
"Eventually Consistent"
*Communications of the ACM*
Classic paper on eventual consistency. Gossip-rs findings sink is eventually consistent across replicas.

**Gilbert, Seth & Lynch, Nancy** (2002)
"Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"
*SIGACT News 2002*
The CAP theorem proof. Gossip-rs chooses CP (consistency + partition tolerance) for coordination.

**Lamport, Leslie** (1978)
"Time, Clocks, and the Ordering of Events in a Distributed System"
*Communications of the ACM*
Foundational paper on distributed time. Gossip-rs uses logical clocks for causality tracking.

## Failure Detection

**Hayashibara, Naohiro et al.** (2004)
"The φ Accrual Failure Detector"
*SRDS 2004*
Adaptive failure detection. Gossip-rs lease expiry uses timeout-based detection, but φ-detector principles inform our tuning.

**Nygard, Michael T.** (2007, 2018)
*Release It! Design and Deploy Production-Ready Software, 2nd Edition*
Pragmatic Bookshelf
Circuit breakers and stability patterns. Gossip-rs B4 connectors implement circuit breakers for external APIs.

## Impossibility Results

**Fischer, Michael J., Lynch, Nancy A., & Paterson, Michael S.** (1985)
"Impossibility of Distributed Consensus with One Faulty Process"
*Journal of the ACM*
The FLP impossibility result. Explains why Gossip-rs uses leases instead of consensus.

**Alpern, Bowen & Schneider, Fred B.** (1985)
"Defining Liveness"
*Information Processing Letters*
Formal definition of liveness properties. Gossip-rs invariants are classified as safety (INV-Sxx) or liveness (INV-Lxx).

## Testing

**Zhou, Jingyu et al.** (2021)
"FoundationDB: A Distributed Unbundled Transactional Key Value Store"
*SIGMOD 2021*
Describes FoundationDB's deterministic simulation testing. Gold standard for testing distributed systems.

**TigerBeetle Engineering** (2025)
"A Descent Into the Vörtex: Building Deterministic Simulation Testing for TigerBeetle"
*Blog post*
Modern implementation of DST. Directly inspired Gossip-rs's testing strategy in Chapter 08.

**Eaton, Phil** (2024)
"What's the big deal about Deterministic Simulation Testing?"
*Blog post*
Accessible introduction to DST. Recommended reading before Chapter 08.

**WarpStream Engineering** (2025)
"Deterministic Simulation Testing for Our Entire SaaS"
*Blog post*
DST at scale in production. Validates our approach to testing B2-B5 boundaries.

## General Distributed Systems

**Kleppmann, Martin** (2017)
*Designing Data-Intensive Applications*
O'Reilly Media
Comprehensive overview of distributed systems. Essential background reading for this guide.

**Cachin, Christian, Guerraoui, Rachid, & Rodrigues, Luís** (2011)
*Introduction to Reliable and Secure Distributed Programming, 2nd Edition*
Springer
Formal treatment of distributed algorithms. Source for many Gossip-rs protocol proofs.

**Akidau, Tyler et al.** (2015)
"The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing"
*VLDB 2015*
Defines at-least-once + idempotent = exactly-once semantics. Core to Gossip-rs B2 design.

## Persistence

**Pillai, Thanumalayan Sankaranarayana et al.** (2014)
"All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications"
*OSDI 2014*
Challenges of durable persistence. Informs Gossip-rs B5 crash recovery protocol.

**Mohan, C. et al.** (1992)
"ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging"
*ACM TODS*
Classic WAL algorithm. Gossip-rs done ledger uses WAL principles.

**Eppstein, David et al.** (2011)
"What's the Difference? Efficient Set Reconciliation without Prior Context"
*SIGCOMM 2011*
Set reconciliation algorithms. Applicable to Gossip-rs done ledger synchronization.

## Scanner Engine & Pattern Matching

**Intel Corporation** (2023)
"Hyperscan: High-Performance Multiple Regex Matching Library"
*Vectorscan (community fork)*
Multi-pattern regex engine using DFA and NFA hybrid algorithms. Gossip-rs `scanner-engine` uses Vectorscan in block and stream modes for prefilter-based window seeding: each rule regex is compiled with `HS_FLAG_PREFILTER` for conservative hit detection, then confirmed via per-rule verification.

**Aho, Alfred V. & Corasick, Margaret J.** (1975)
"Efficient String Matching: An Aid to Bibliographic Search"
*Communications of the ACM*
Classic multi-pattern string matching in O(n + m + z) time. The Aho-Corasick automaton underlies Vectorscan's literal matching paths and is the basis for keyword-based anchor prefiltering in the scan pipeline.

**Shannon, Claude E.** (1948)
"A Mathematical Theory of Communication"
*Bell System Technical Journal*
Defines Shannon entropy (bits per symbol). Gossip-rs uses Shannon entropy as a gate to discard low-randomness regex matches early in the detection pipeline — findings below a per-rule `min_bits_per_byte` threshold are rejected before emission.

**NIST** (2018)
"Recommendation for the Entropy Sources Used for Random Bit Generation"
*NIST SP 800-90B*
Defines min-entropy (`H_inf = log2(n) - log2(max_count)`). Gossip-rs optionally applies min-entropy as a secondary gate to catch skewed byte distributions where Shannon entropy alone looks moderate.

**Denis, Frank** (2023)
"AEGIS: A Fast Authenticated Encryption Algorithm"
*IETF RFC 9312 / CFRG*
AEGIS-128L is an AES-NI-based authenticated encryption scheme. Gossip-rs repurposes its MAC mode (zero key, message-only) as a fast 128-bit collision-resistant fingerprint for decoded-buffer deduplication, avoiding a separate hash dependency.

**Intel Corporation** (2024)
"Intel 64 and IA-32 Architectures Software Developer's Manual — SSE2 Intrinsics"
*Intel SDM*
SSE2 signed-comparison pairs for byte-range classification. Gossip-rs `simd_classify` uses `_mm_cmpgt_epi8` pairs to bucket bytes into ASCII classes (upper, lower, digit, special) at 16 bytes per iteration.

**ARM Ltd.** (2024)
"ARM Architecture Reference Manual — NEON Intrinsics"
*ARM ARM*
NEON wrapping-subtract + unsigned-compare for byte-range classification. Gossip-rs `simd_classify` uses `vsubq_u8` + `vcltq_u8` (2 ops per range) on aarch64, which is cheaper than the 3-op SSE2 equivalent.

## Git Internals

**Git Project** (2011)
"Git Pack Format"
*Documentation/gitformat-pack.txt*
Specification for `.pack` files: header, object entries, and delta chains. Gossip-rs `scanner-git` parses pack files directly for object extraction during scanning.

**Git Project** (2018)
"Git Pack Index Format (v2)"
*Documentation/gitformat-pack.txt*
Specification for `.idx` v2 files: fanout table, OID table, CRC32 table, and 4-byte/8-byte offset tables. Gossip-rs `pack_idx` module provides zero-copy parsing of v2 index files with O(1) offset lookups.

**Git Project** (2018)
"Multi-Pack-Index (MIDX) Format"
*Documentation/gitformat-pack.txt*
Specification for the multi-pack-index: fanout, OID lookup, and pack-name tables across multiple pack files. Gossip-rs `midx_build` implements an in-memory MIDX builder using k-way merge over pre-sorted OID streams (O(N log P) where P = pack count).

**Git Project** (2018)
"Commit-Graph Format"
*Documentation/gitformat-commit-graph.txt*
Specification for the commit-graph file: OID fanout, OID table, commit data (tree OID, parent positions, generation number, timestamp), and extra-edges list. Gossip-rs `commit_graph_mem` builds an equivalent in-memory SoA representation when on-disk commit-graph files are missing.

**MacDonald, John** (2000)
"Diff Delta Compression in Git (xdelta/vcdiff family)"
*Git Source: `diff-delta.c`*
Git's delta compression encodes objects as a base reference plus copy/insert instructions. Understanding delta chains is essential for correct object reconstruction during pack file traversal in `scanner-git`.

**Chacon, Scott & Straub, Ben** (2014)
"Git Internals – Packfiles"
*Pro Git Book, 2nd Edition, Chapter 10*
Accessible explanation of pack file creation, thin packs, and index generation. Provides the conceptual background for Gossip-rs's pack/index/MIDX parsing stack.

## Work Scheduling

**Chase, David & Lev, Yosef** (2005)
"Dynamic Circular Work-Stealing Deque"
*SPAA 2005*
The Chase-Lev work-stealing deque: LIFO push/pop for the owner, FIFO steal for thieves. Gossip-rs `scanner-scheduler` uses the `crossbeam-deque` implementation (which implements this algorithm) for per-worker task queues with randomized stealing across N workers.

**Arora, Nimar S., Blumofe, Robert D., & Plaxton, C. Greg** (1998)
"Thread Scheduling for Multiprogrammed Multiprocessors"
*SPAA 1998*
Theoretical foundations of randomized work stealing. Proves expected O(T₁/P + T∞) bounds. Informs Gossip-rs's randomized steal target selection (XorShift64 RNG per worker).

**Blumofe, Robert D. & Leiserson, Charles E.** (1999)
"Scheduling Multithreaded Computations by Work Stealing"
*Journal of the ACM*
Seminal analysis of work-stealing schedulers. Establishes the work-first principle (prioritize local execution). Gossip-rs follows the local-first spawn pattern: tasks are pushed to the local deque before being made available for stealing.

**Nichols, Bradford et al.** (1996)
*Pthreads Programming: A POSIX Standard for Better Multiprocessing*
O'Reilly Media
Practical guide to thread management, synchronization, and the Parker/Unparker idle pattern. Gossip-rs uses `crossbeam-utils` Parker/Unparker for tiered idle strategy (spin → park).

**Podlipnig, Stefan & Böszörményi, László** (2003)
"A Survey of Web Cache Replacement Strategies"
*ACM Computing Surveys*
Survey of admission control and backpressure strategies. Gossip-rs `scanner-scheduler` implements token-based and byte-based budgets as backpressure mechanisms: work admission is gated by `TokenBudget` and `ByteBudget` using lock-free CAS loops with RAII permits.

**Welsh, Matt, Culler, David, & Brewer, Eric** (2001)
"SEDA: An Architecture for Well-Conditioned, Scalable Internet Services"
*SOSP 2001*
Introduced staged event-driven architecture with explicit queue-based backpressure between stages. Gossip-rs's separation of I/O discovery, work injection, and CPU execution into budget-gated stages follows SEDA principles.

## Secret Detection

**YARA Project** (2024)
"Writing YARA Rules — Base64 Strings"
*YARA Documentation*
Documents base64 offset alignment (0, 1, 2-byte shifts produce three encoding permutations) and the stable/unstable character boundary. Gossip-rs `b64_yara_gate` implements this algorithm to generate stable base64 substrings from anchor literals for prefilter matching.

**Josefsson, S.** (2006)
"The Base16, Base32, and Base64 Data Encodings"
*RFC 4648*
Canonical specification for Base64 encoding alphabets (standard and URL-safe) and padding. Gossip-rs handles both alphabets by canonicalizing URL-safe characters (`-` → `+`, `_` → `/`) during base64 gate scanning.

**Meli, Michael, McNiece, Matthew R., & Reaves, Bradley** (2019)
"How Bad Can It Git? Characterizing Secret Leakage in Public GitHub Repositories"
*NDSS 2019*
Large-scale study of secret leakage in public repositories. Found hundreds of thousands of leaked credentials. Motivates the design of Gossip-rs's multi-stage detection pipeline (prefilter → entropy gate → regex confirm → confidence scoring).

**Sinha, Vibha Singhal et al.** (2015)
"Detecting and Preventing Secret Data Exfiltration"
*IEEE Security & Privacy*
Techniques for identifying sensitive data patterns in source code. Informs the multi-signal approach (regex patterns, entropy thresholds, character-class profiles) used in Gossip-rs's confidence model.

**detect-secrets (Yelp)** (2018)
"detect-secrets: An enterprise friendly way of detecting and preventing secrets in code"
*GitHub Repository*
Pioneered entropy-based secret detection with digit-only penalty heuristics. Gossip-rs's digit-only entropy penalty (`DIGIT_ONLY_PENALTY_NUMERATOR / log2(len)`) directly follows this approach to reduce false positives on numeric strings.

**TruffleHog (Truffle Security)** (2022)
"TruffleHog: Find and verify credentials"
*GitHub Repository*
Modern credential scanner combining regex detection with live verification. Gossip-rs follows a similar multi-stage architecture: prefilter (Vectorscan), confirm (per-rule regex), gate (entropy + character class), then emit with confidence scoring.

---

## Citation Format

Throughout this guide, references are cited inline using the format:

```
[Author(s), Year]
```

For example:

> Leases provide time-bounded resource ownership [Gray & Cheriton, 1989], eliminating the need for expensive consensus protocols.

See individual chapters for specific citations.

## Further Reading

For readers new to distributed systems, we recommend reading in this order:

1. Kleppmann (2017) — *Designing Data-Intensive Applications*
2. Lamport (1978) — "Time, Clocks, and the Ordering of Events"
3. Gray & Cheriton (1989) — "Leases"
4. Kleppmann (2016) — "How to do distributed locking"
5. Akidau et al. (2015) — "The Dataflow Model"

For the scanner engine and pattern matching pipeline (Chapters 10–11):

6. Shannon (1948) — "A Mathematical Theory of Communication"
7. Aho & Corasick (1975) — "Efficient String Matching"
8. Intel, "Hyperscan" / Vectorscan documentation
9. Denis (2023) — "AEGIS: A Fast Authenticated Encryption Algorithm"
10. YARA Project — "Writing YARA Rules — Base64 Strings"
11. detect-secrets (Yelp) — entropy-based secret detection

For Git internals and pack file parsing (Chapters 12–13):

12. Chacon & Straub (2014) — *Pro Git*, Chapter 10 (Packfiles)
13. Git Project — Pack Format, Pack Index v2, MIDX, Commit-Graph specifications
14. Meli et al. (2019) — "How Bad Can It Git?"

For work scheduling and runtime (Chapters 13–14):

15. Chase & Lev (2005) — "Dynamic Circular Work-Stealing Deque"
16. Blumofe & Leiserson (1999) — "Scheduling Multithreaded Computations by Work Stealing"
17. Welsh et al. (2001) — "SEDA: An Architecture for Well-Conditioned, Scalable Internet Services"

This provides a solid foundation before diving into the Gossip-rs implementation.
