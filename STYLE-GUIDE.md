# Style Guide — Gossip-rs Learning Guide

This document defines the conventions for writing new sections of the
Gossip-rs learning guide. It is the authoritative reference for LLM agents
producing new boundary sections and for human contributors reviewing or
extending existing content.

The conventions below were extracted from the B2 Coordination section
(12 chapters, ~52K words) and represent the established standard. All
future sections must conform to these rules.

---

## 1. Purpose and Audience

The guide targets **engineers learning distributed systems through a
concrete, production-ready codebase**. The reader is assumed to:

- Know Rust (traits, lifetimes, `Result`, `Option`, pattern matching)
- Not know distributed systems (leases, fencing, consensus, linearizability)
- Expect every design decision to be derived from first principles

The guide is not an API reference. It is a narrative that follows a system's
lifecycle from problem statement through implementation to verification,
explaining *why* each design choice exists.

---

## 2. Chapter Structure Template

Every chapter follows this skeleton:

### 2.1 Title

Format: `# [Evocative Short Title] -- [Technical Subtitle]`

Use an em-dash (`--`) to separate the narrative title from the technical
subtitle. The evocative title names the failure or concept in human terms.
The subtitle names the technical mechanism.

Examples from B2:

| Chapter | Title |
|---------|-------|
| 01 | `Two Workers, One Shard -- The Coordination Problem` |
| 02 | `The Zombie Worker -- Leases and Fencing` |
| 05 | `Safe Retries -- Idempotency and the Op-Log` |
| 07 | `"The Unbounded Shard" -- Why Splits Exist and How Ranges Work` |
| 08 | `"Split-Replace" -- Parent Retirement with Coverage Proof` |

Chapters that walk through existing code (reference implementation,
validation layer) may use a descriptive title without a failure-evoking
prefix:

| Chapter | Title |
|---------|-------|
| 10 | `The Validation Layer -- How Safety Is Enforced` |
| 11 | `The Reference Implementation -- InMemoryCoordinator Walk-Through` |

Optional: quote marks around the evocative title when it names a concept
being introduced for the first time (e.g., `"The Unbounded Shard"`).

A `Chapter N:` prefix is acceptable but not required. Some B2 chapters
include it, some do not. Pick one convention per boundary section and
apply it consistently.

### 2.2 Opening Failure Scenario

The chapter opens with 1-2 italicized paragraphs describing a **concrete
failure** that motivates the chapter. This is not hypothetical. It is a
narrative written in present tense with specific details.

Rules:

- **Italicized** (wrapped in `*...*`)
- **Present tense, narrative style**: "Worker A sends a checkpoint
  request. The network hiccups..."
- **Specific values**: shard IDs, cursor positions, timestamps, key
  names, error codes, byte counts, durations
- **Must describe a failure that actually happens** without the concept
  being introduced in this chapter
- **Ends with a one-sentence problem statement**: "This is the
  coordination problem." / "Without idempotency, retries are
  indistinguishable from new operations."
- **Length**: 100-250 words
- **Rhetorical questions are acceptable** in the scenario itself (e.g.,
  "How does Worker A know its lease expired?")

The scenario is followed by a **horizontal rule** (`---`) before the
first content section begins.

**Exception**: Walkthrough and verification chapters (e.g., Ch. 10
Validation Layer, Ch. 11 Reference Implementation, Ch. 12 Proving
Correctness) may open with a direct introduction instead of a failure
scenario. These chapters synthesize concepts from earlier chapters rather
than introducing a new failure mode.

### 2.3 Motivation Section

After the horizontal rule, 1-2 paragraphs connect the failure scenario
to the concept being introduced. This section answers "why does this
mechanism exist?" and names the core abstraction.

### 2.4 Core Content Sections

Numbered or named sections (e.g., `## 1. The Split Motivation` or
`## ShardStatus -- The Four-State Lifecycle`). Each introduces one
concept backed by real code.

Subsections use decimal numbering: `### 2.1`, `### 2.2`, etc.

### 2.5 Summary / What's Next

A brief wrap-up connecting the chapter to the next one. This section is
short — a few sentences, not a page. Name the specific chapter that
follows and the concept it introduces.

---

## 3. Failure Scenario Rules (Detailed)

The opening failure scenario is the single most important structural
element. It grounds every chapter in a concrete failure that the reader
can visualize.

### Writing Checklist

1. Choose a failure that the concept in this chapter *prevents*.
2. Name the actors: Worker A, Worker B, the coordinator, the network.
3. Include at least three specific values (shard ID, cursor position,
   timing, key name, error code).
4. Write in present tense ("Worker A acquires...", "The network
   hiccups...").
5. Show the consequence: duplicates, data loss, silent corruption,
   infinite loops.
6. End with a sentence that names the problem.
7. Keep it between 100 and 250 words.

### Examples of Specific Values

Good: "Worker A has scanned commits `a0000`..`f3a21`"
Bad:  "Worker A has scanned some commits"

Good: "The lease expires at second 30. The GC pause completes at second 38."
Bad:  "The lease expires while the worker is paused."

Good: "repos 1-1200 and 1100-2400 overlap by 100"
Bad:  "the ranges overlap"

---

## 4. Code Inclusion Rules

### 4.1 Real Code Only

Every code block must come from the actual source at
`crates/gossip-contracts/src/`. Never write pseudocode when real code
exists. Never invent simplified versions of real functions.

### 4.2 Source Attribution

Precede code blocks with a reference to the source file:

- "Here is the definition from `record.rs`:" (before the block)
- "From `traits.rs`, the `CoordinationBackend` trait defines:" (inline)
- `crates/gossip-coordination/src/in_memory.rs` (standalone
  path in a code block or blockquote when referencing the whole file)

### 4.3 Complete Implementations Preferred

Show the full function or struct when it is ≤ 60 lines. For longer
items, show the complete type signature and key sections, with a note
about what was elided.

### 4.4 Field-by-Field Walkthrough

After showing a struct, walk through each field or field group explaining
what it is for. Use bold text for the field or group name:

> **`#[repr(u8)]` with explicit discriminants.** The `u8` values are
> persisted to durable storage. The codebase enforces this with
> compile-time assertions:

### 4.5 Doc Comments Preserved

Keep `///` doc comments in code blocks. They are part of the teaching.
Do not strip them to save space.

### 4.6 Compile-Time Assertions

Include `const _: ()` assertion blocks when present in the source. They
demonstrate invariant discipline and are a teaching point in themselves.

### 4.7 Test Code Selectively

Include test code only when it demonstrates a truth table, a property,
or a non-obvious invariant. Do not include test boilerplate
(setup/teardown, helper construction).

### 4.8 No Invented Annotations

Do not add comments to code blocks that are not in the source. No
`// <-- this is important!` or `// NOTE: ...` annotations. The
surrounding prose handles explanation.

---

## 5. Diagram Rules

### 5.1 Format

Use **Mermaid** for all diagrams. Inline them directly in the markdown
(no separate files).

### 5.2 Diagram Types by Use Case

| Use Case | Diagram Type |
|----------|-------------|
| Lifecycle types (`ShardStatus`, `RunStatus`) | `stateDiagram-v2` |
| Multi-actor flows (worker ↔ coordinator) | `sequenceDiagram` |
| Decision trees (validation checks) | `flowchart` / `graph` |
| Comparison layouts (before/after split) | `graph LR` with subgraphs |
| Simple structural layouts (intervals, ranges) | ASCII art in `text` blocks |

### 5.3 ASCII Art

Use plain `text` code blocks for simple structural diagrams where Mermaid
adds complexity without value:

```text
+-----------------------------------------------------+
|  start (inclusive)          end (exclusive)           |
|  |--------------------------|                        |
+-----------------------------------------------------+
```

### 5.4 Validation Flowcharts

Multi-step validation sequences use ASCII-art flowcharts in `text` blocks
(not Mermaid), matching the style used in Ch. 10:

```text
  Incoming operation
           |
           v
  +---------------------------+
  | 1. check_op_idempotency   |
  +---------------------------+
           |
           v
  +---------------------------+
  | 2. validate_lease         |
  +---------------------------+
```

---

## 6. Tone and Voice

### 6.1 Declarative, Not Chatty

Good: "The coordinator validates five checks in priority order."
Bad:  "Let's take a look at what happens when..."
Bad:  "Now we're going to explore..."

### 6.2 Technical Precision

Use correct terminology. Define terms on first use.

Good: "Half-open interval `[start, end)`"
Bad:  "range from start to end"

Good: "lexicographic byte order"
Bad:  "alphabetical order"

### 6.3 No Hedging

Good: "This panics."
Bad:  "This might panic."
Bad:  "This could potentially panic."

Good: "The fence epoch prevents stale writes."
Bad:  "The fence epoch should help prevent stale writes."

### 6.4 Explain the Why

Every design choice gets a rationale. Do not merely describe what the
code does — explain why it was designed that way and what failure it
prevents.

Good: "Tenant is checked first to prevent timing side-channels that
      could leak information about whether a shard exists in another
      tenant's namespace."

Bad:  "Tenant is checked first."

### 6.5 Academic Citations

Reference papers at point of use. Format:
`Author, "Title" (Venue Year)`

Example: `Kleppmann, "How to do distributed locking" (2016)`

For system references, use the same inline style:
`Chang et al., OSDI 2006` or `Zhou et al., SIGMOD 2021`

Use a table when comparing multiple systems:

| System | Property | Reference |
|--------|----------|-----------|
| **Bigtable** | `[startRow, endRow)` tablets | Chang et al., OSDI 2006 |
| **Spanner** | Half-open key-range splits | Corbett et al., OSDI 2012 |

### 6.6 Prohibited Patterns

- No emojis
- No exclamation marks (except inside failure scenarios)
- No rhetorical questions in body text (acceptable in failure scenarios)
- No first person ("I") in body text
- **"We" exception**: "we" is acceptable in two contexts:
  (1) cross-reference constructions ("We will see in Chapter N how...")
  and (2) walkthrough passages where the author guides the reader
  through code ("Let us examine each field group"). Avoid "we" in
  declarative statements about system behavior ("The coordinator
  validates..." not "We validate...")
- No "let's" contractions (use "let us" when the walkthrough form is
  needed)
- No hedging words ("might", "could potentially", "should probably")
  when describing system behavior. "Might" is acceptable when
  describing hypothetical failure scenarios or conditional future
  behavior

---

## 7. Chapter Sizing

Target word counts (including code blocks):

| Category | Target | Maximum |
|----------|--------|---------|
| Standard chapter | 4,000-4,500 words | 5,000 words |
| Code-heavy chapter (validation, walkthrough) | 4,500-5,000 words | 5,500 words |
| Capstone chapter (simulation, verification) | 4,500-5,000 words | 5,500 words |

These ranges are derived from the actual B2 chapters:

- Chapters 1-9: 4,051-4,408 words (mean: 4,172)
- Chapters 10-12: 4,959-5,058 words (mean: 4,996)

If a chapter exceeds the maximum, split it into two chapters.

---

## 8. Part Structure

Chapters within each boundary section are grouped into Parts that follow
a narrative arc. The arc is not arbitrary — it follows the order in which
a reader needs to understand concepts to make sense of the next concept.

### The Four-Part Framework

**Part I: Foundations** — Data types, state machines, safety mechanisms.
Establishes the vocabulary before any operations are introduced. A reader
finishing Part I should be able to look at any struct definition in the
boundary and understand what each field is for.

**Part II: The Lifecycle** — Follow a single operation from start to
finish. Each concept is introduced at the moment the operation needs it.
For B2 Coordination, this is the scan lifecycle: create a run → acquire
a shard → checkpoint progress → complete or park. For other boundaries,
identify the primary operation sequence from the source code.

**Part III: Advanced Mechanics** — Concepts that build on the lifecycle
but are not part of the happy path. Splits, error recovery, advanced
error handling, edge cases. These chapters assume the reader has
internalized the lifecycle from Part II.

**Part IV: Verification** — The validation layer, reference
implementation walkthrough, simulation harness, and formal methods.
These chapters synthesize everything from Parts I-III and show how
correctness is enforced and verified.

### B2 Part Mapping (Reference)

| Part | Chapters | Topic |
|------|----------|-------|
| I | 1-2 | The coordination problem, leases and fencing |
| II | 3-6 | Run creation, acquire, checkpoint, complete/park |
| III | 7-9 | Why splits exist, split-replace, split-residual |
| IV | 10-12 | Validation layer, reference impl, simulation/TLA+ |

---

## 9. Cross-Reference Conventions

### 9.1 Forward References

Use when a concept is mentioned before its dedicated chapter:

"We will see in Chapter N how [concept] enforces this."

### 9.2 Back References

Use when building on a previously introduced concept:

"Recall from Chapter N that [concept]."

Or inline: "the five-check cascade from Chapter 3"

### 9.3 Source File References

Before a code block: `"Here is the definition from `record.rs`:"` or
`"From `traits.rs`:"`

In running text: `filename.rs` when naming the file, or
`filename.rs:123` when discussing a specific line.

### 9.4 Inter-Section Links

Use relative markdown links for cross-boundary references:

`[text](../04-boundary-2-coordination/07-why-splits-exist.md)`

For intra-boundary references (same directory):

`[Chapter 3](03-starting-a-scan.md)`

### 9.5 Invariant References

Reference named invariants by label: "INV-2", "S1 through S7". Define
the label on first use and use it consistently thereafter.

---

## 10. Section and File Naming

### 10.1 Top-Level Directories

Format: `NN-boundary-N-short-name/`

| Directory | Boundary |
|-----------|----------|
| `02-boundary-1-identity-spine/` | B1 Identity |
| `04-boundary-2-coordination/` | B2 Coordination |
| `05-boundary-3-shard-algebra/` | B3 Shard Algebra |
| `06-boundary-4-connector/` | B4 Connector |
| `07-boundary-5-persistence/` | B5 Persistence |

Note: The directory numbering (`04`, `05`, `06`) reflects reading order,
not boundary numbering. The `boundary-N` portion in the directory name
uses the original boundary number from the architecture.

### 10.2 Chapter Files

Format: `NN-kebab-case-title.md`

The number prefix matches reading order within the section. Examples:

- `01-the-coordination-problem.md`
- `07-why-splits-exist.md`
- `12-proving-correctness.md`

### 10.3 Non-Boundary Directories

| Directory | Content |
|-----------|---------|
| `00-prologue/` | Problem statement, architecture overview, reading guide |
| `01-foundations/` | Core concepts (hashing, content addressing, type-driven design) |
| `03-distributed-systems-theory/` | Theory primers (leases, fencing, idempotency, failure detection) |
| `08-cross-cutting/` | Testing strategy, operational concerns |
| `09-appendices/` | References, glossary, TLA+ specs |

---

## 11. What NOT to Do

These rules exist because the conventions were established in B2 and
deviations would create inconsistency.

1. **Do not write pseudocode** when real code exists in the source tree.
2. **Do not add "exercises" or "quiz" sections.** The guide teaches
   through narrative and code, not through assignments.
3. **Do not use first person** ("I", "we") except in failure scenarios
   where a character perspective is needed.
4. **Do not create separate files for diagrams.** Inline all Mermaid in
   the chapter markdown.
5. **Do not reference old chapter structures or versioned APIs.** There
   is one version: the current one.
6. **Do not add comments to code** that are not in the actual source.
   No `// <-- this is important!` annotations.
7. **Do not create backward-compatibility shims** between old and new
   chapter numbers.
8. **Do not add emojis, exclamation marks, or "fun facts."**
9. **Do not explain Rust syntax.** The reader knows Rust.
10. **Do not add a "Prerequisites" section per chapter.** The sequential
    reading path handles prerequisite ordering.
11. **Do not add external tracking IDs** (beads IDs, finding numbers,
    priority tags) in any content.

---

## 12. Boundary-Specific Adaptation Notes

The style guide provides the structural framework. The narrative arc for
each boundary must be discovered from the source code. The framework is
constant; the content is domain-specific.

### Process for Writing a New Boundary Section

1. **Read ALL source files** in the boundary's crate directory.
2. **Identify the lifecycle arc** — the primary operation sequence that
   a single unit of work follows from creation to completion. This
   becomes the Part II narrative spine.
3. **Identify failure modes at each lifecycle step.** Each failure mode
   becomes a chapter opener (the italicized failure scenario).
4. **Identify invariants and their enforcement code.** These become
   the depth content within chapters and the material for the
   verification chapter.
5. **Identify the reference implementation** (usually an `in_memory.rs`
   or test double). This becomes a walkthrough chapter.
6. **Identify simulation or property testing.** This becomes the
   verification chapter.

### Known Boundary Arcs

**B2 Coordination** (`crates/gossip-contracts/src/coordination/`):
Scan lifecycle — create run → acquire shard → checkpoint → complete/park → split.

**B3 Shard Algebra** (`crates/gossip-frontier/src/`):
Range lifecycle — key encoding → range splitting → coverage verification →
hint propagation.

**B4 Connector** (`crates/gossip-contracts/src/connector/`):
Enumeration lifecycle — connect → enumerate → page → validate →
backpressure → circuit-break.

**B5 Persistence** (`crates/gossip-contracts/src/persistence/`):
Write lifecycle — buffer → commit → acknowledge → recover.

### Adaptation Guidance

- The Part structure (I-IV) applies to every boundary, but the chapter
  count within each Part varies based on the domain's complexity.
- Not every boundary will have split operations or shard algebra. Omit
  Part III if the boundary has no advanced mechanics beyond the lifecycle.
- Every boundary should have a verification chapter (Part IV). If the
  boundary has no simulation harness yet, the chapter covers the
  property tests and invariant assertions that exist.

---

## 13. README Update Checklist

When adding a new boundary section, update these locations:

- [ ] `README.md` — Project Status table (boundary name, status, stats)
- [ ] `README.md` — Sequential Reading Path
- [ ] `README.md` — Topic Jump Table
- [ ] `README.md` — Documentation Stats (paper count, diagram count)
- [ ] `00-prologue/03-architecture-at-a-glance.md` — boundary descriptions
- [ ] `00-prologue/04-how-to-read-this-guide.md` — reading order
- [ ] Cross-references in other sections that forward-reference the new
  boundary (search for the boundary name across all `.md` files)
- [ ] Section numbers if directories were renumbered

---

## 14. Quick Reference Card

For LLM agents starting a new boundary section, follow this sequence:

```text
1. Read all source files in the boundary's crate directory
2. Identify: lifecycle arc, failure modes, invariants, ref impl, tests
3. Draft the chapter list (map to Parts I-IV)
4. For each chapter:
   a. Write the failure scenario (100-250 words, italicized, specific)
   b. Write the motivation section (1-2 paragraphs)
   c. Write the core content with real code blocks
   d. Add Mermaid diagrams for state machines and flows
   e. Add cross-references (forward and back)
   f. Write the summary / what's next
   g. Verify word count (4,000-5,000 target)
5. Verify all code blocks match actual source
6. Update README and prologue files (Section 13 checklist)
```

Style invariants to verify before submitting:

- [ ] Every code block comes from actual source files
- [ ] No pseudocode, no invented comments in code blocks
- [ ] No first person in body text
- [ ] No emojis, exclamation marks, or hedging
- [ ] Every design choice has a stated rationale
- [ ] Failure scenarios use specific values, not abstractions
- [ ] Cross-references use correct chapter numbers
- [ ] Word count within bounds
