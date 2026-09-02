# docs/CONTEXT_SYSTEM.md — Documentation Architecture

**Why this file exists.** TEKWE3 will run for months across hundreds of agent sessions, and the agents themselves will be replaced by newer models. Documentation is the only continuous memory. But documentation that grows without discipline becomes a cost that consumes the budget it was meant to save.

This file defines how the documentation stays cheap to read.

**Three principles:**

1. **Progressive disclosure.** An agent reads a router, then a task, then one spec file. Never the whole design.
2. **Bounded size.** Every document has a hard line cap. Exceeding it triggers a split, never an append.
3. **Compaction over accumulation.** History is summarized at phase boundaries and archived. Only summaries stay in the read path.

---

## 1. The complete documentation tree

```
tekwe3/
├─ AGENTS.md                     operating manual — read once per agent generation
├─ STATE.md                      ← REWRITTEN every session, cap 80 lines
├─ BACKLOG.md                    deferred work, cap 150 lines, triaged per phase
├─ CHANGELOG.md                  Keep a Changelog; only the top is ever read
├─ README.md                     for humans and recruiters, not for agents
│
├─ docs/
│  ├─ INDEX.md                   ← THE ROUTER. Read first, always. Cap 120.
│  ├─ INVARIANTS.md              ← numbered system rules. Cap 80.
│  ├─ REJECTED.md                ← dead approaches. Cap 100.
│  ├─ GLOSSARY.md                ← project vocabulary. Cap 120.
│  ├─ CONTEXT_SYSTEM.md          ← this file
│  ├─ ARCHITECTURE.md            ← index over spec/, cap 80 lines. Tier 3.
│  ├─ ROADMAP.md                 ← phases and exit criteria. Tier 3.
│  │
│  ├─ spec/                      ← normative design, one file per component
│  │   ├─ 00-overview.md              thesis, system shape
│  │   ├─ 01-execution-model.md       thread-per-core, sharding, mailboxes
│  │   ├─ 02-write-path.md
│  │   ├─ 03-read-path.md
│  │   ├─ 04-memtable.md
│  │   ├─ 05-wal.md
│  │   ├─ 06-sstable.md
│  │   ├─ 07-filters-index.md
│  │   ├─ 08-vlog.md
│  │   ├─ 09-compaction.md
│  │   ├─ 10-mvcc.md
│  │   ├─ 11-text.md
│  │   ├─ 12-vector.md
│  │   ├─ 13-query.md
│  │   ├─ 14-runtime-sim.md
│  │   ├─ 15-codec.md
│  │   ├─ 16-simd.md
│  │   ├─ 17-kernel.md
│  │   └─ 18-interfaces.md            CLI, server, public API
│  │
│  ├─ format/                    ← byte-level, versioned, conformance-tested
│  │   ├─ sstable-v3.md
│  │   ├─ wal-record-v1.md
│  │   ├─ manifest-v1.md
│  │   └─ vlog-chunk-v1.md
│  │
│  ├─ adr/
│  │   ├─ README.md              ← ONE LINE PER ADR. This is what agents read.
│  │   ├─ _TEMPLATE.md
│  │   └─ NNNN-<title>.md        cap 50 lines each
│  │
│  ├─ tasks/
│  │   ├─ _TEMPLATE.md
│  │   ├─ OPEN.md                ← one line per open task, the task board
│  │   ├─ P4-007-....md
│  │   └─ archive/P3/            closed tasks, moved at phase close
│  │
│  ├─ worklog/
│  │   ├─ 2026-03-14-47.md       cap 60 lines each
│  │   ├─ phase-3-summary.md     ← written at phase close
│  │   └─ archive/phase-3/       individual entries moved here
│  │
│  ├─ ideas/                     ← see docs/IDEA_INTAKE.md
│  │   ├─ INBOX.md              append-only verbatim captures, never edited
│  │   ├─ IDEA-NNNN.md          one evaluation each, cap 80 lines
│  │   └─ archive/phase-N/
│  │
│  ├─ TESTING.md                test + log preservation system
│  ├─ IDEA_INTAKE.md            idea capture + change control
│  ├─ PROCESS_CHANGELOG.md      changes to AGENTS.md itself, cap 60 lines
│  │
│  └─ benchmarks/               (see also logs/bench/)
│      └─ README.md             how to reproduce anything here
│
├─ logs/                        ← see docs/TESTING.md
│  ├─ gate/LATEST.md            most recent gate result, cap 30 lines
│  ├─ failures/INDEX.md         ← one line per failure. The read path.
│  ├─ seeds/regression.txt      ← every bug-finding seed. Kept forever.
│  └─ bench/INDEX.md            one line per recorded benchmark
│
└─ crates/<name>/CONTEXT.md      ← per-crate agent briefing. Cap 40 lines.
```

---

## 2. Size caps and enforcement

| File | Cap | Action on overflow |
|---|---|---|
| `docs/INDEX.md` | 120 lines | Never grows — it is a router. If it must, the tree is wrong. |
| `STATE.md` | 80 lines | Rewrite, never append. History belongs to git. |
| `docs/INVARIANTS.md` | 80 lines | Overflow means invariants are too granular — consolidate |
| `docs/REJECTED.md` | 100 lines | One line per rejection; detail lives in the ADR |
| `docs/GLOSSARY.md` | 120 lines | Split by domain if needed |
| `docs/ARCHITECTURE.md` | 80 lines | It is an index only — content belongs in `spec/` |
| `docs/spec/*.md` | 400 lines each | Split into sub-files with a local index |
| `docs/format/*.md` | 300 lines each | Split by section |
| Each ADR | 50 lines | If longer, it is a spec, not a decision |
| Each worklog | 60 lines | Longer means it is a narrative, not notes |
| `docs/tasks/*.md` | 80 lines | Longer means the task is too big — split it |
| `crates/*/CONTEXT.md` | 40 lines | Longer means it is duplicating the spec |
| `BACKLOG.md` | 150 lines | Triage: promote to task, or delete with a note |

**Enforcement.** Add to the `justfile`:

```
just doc-check
  → fails if any file exceeds its cap
  → fails if any file in docs/ is missing from docs/INDEX.md
  → fails if any ADR is missing from docs/adr/README.md
  → fails if any crate lacks a CONTEXT.md
  → fails if STATE.md was not modified in the last 5 commits
```

`just doc-check` is part of `just gate`. Documentation debt therefore cannot accumulate silently.

---

## 3. Rotation and compaction

**Worklogs.** At each phase close:
1. Write `docs/worklog/phase-N-summary.md` — max 100 lines, covering: what was built, what was decided, what surprised us, what remains weak
2. Move every individual entry into `docs/worklog/archive/phase-N/`
3. Agents read only the summary plus the two newest live entries

**Tasks.** At phase close, move closed tasks to `docs/tasks/archive/P<N>/`. `docs/tasks/OPEN.md` holds one line per open task and is the board.

**Backlog.** Triaged at every phase boundary. Every item is either promoted to a task, or deleted with a one-line reason appended to the phase summary. Nothing survives two phases untouched.

**STATE.md.** Rewritten from scratch each session. Never appended. If you want history, use `git log -p STATE.md`.

**ADRs.** Never deleted, never edited after acceptance. Superseded ADRs get `**Status:** Superseded by ADR-NNNN` on the first line, and the index line is updated. The index is the read path; the ADRs are the archive.

---

## 4. Splitting `ARCHITECTURE.md` — a P0 task

The current `ARCHITECTURE.md` is a single monolith. It must be split before P1 begins, or every agent will pay to read it.

**Task P0-011 — split the architecture spec**

1. Create `docs/spec/` with the 19 files listed in §1
2. Move each section of `ARCHITECTURE.md` into its component file, keeping content identical
3. Reduce `ARCHITECTURE.md` to an 80-line index: one paragraph of thesis, then a table mapping component → spec file
4. Extract every "hard rule", "invariant", and "never" from the prose into `docs/INVARIANTS.md` as numbered items
5. Extract §1 (corrections to v2) into `docs/REJECTED.md`
6. Write `crates/*/CONTEXT.md` for every crate that exists
7. Populate `docs/GLOSSARY.md` from terms used in the spec
8. Verify: every spec file under 400 lines, every file indexed, `just doc-check` green

**Acceptance:** an agent can complete a WAL task by reading `INDEX.md` + `STATE.md` + `INVARIANTS.md` + `spec/05-wal.md` + `tekwe3-wal/CONTEXT.md` + the task file — and nothing else.

---

## 5. Templates

### 5.1 `docs/INVARIANTS.md`

The highest value-per-token file in the repository. Every task file names the invariants it touches. Every self-review checks them.

```markdown
# System Invariants

Numbered, stable, never renumbered. Referenced as I-NN everywhere.

## Durability
- **I-01** An acknowledged write is never lost, under any crash schedule.
- **I-02** Recovery always yields a prefix-consistent state — never a partial batch.
- **I-03** One fsync per group-commit batch; never per record.

## Determinism
- **I-04** Engine code never calls wall-clock time, filesystem, or RNG directly.
  All of it goes through the Runtime trait.
- **I-05** HashMap iteration order never affects observable behavior.
- **I-06** A given TEKWE3_SEED reproduces a run byte-for-byte.

## On-disk format
- **I-07** Files are immutable after finalization. Never written in place.
- **I-08** Every section carries an independent checksum.
- **I-09** Corruption in one section never prevents reading another.
- **I-10** The footer sits at end-of-file with a repeated magic.
- **I-11** Every section is optional; a pure-KV SSTable is valid.
- **I-12** All multi-byte integers are little-endian.
- **I-13** All offsets are absolute from file start.
- **I-14** Parsing arbitrary bytes never panics and never invokes UB.
- **I-15** Reading a previous format version always works.

## Concurrency
- **I-16** A shard exclusively owns its data. No other thread touches it.
- **I-17** The only lock-free structures are the cross-shard mailboxes.
- **I-18** No unbounded queue exists anywhere; backpressure is mandatory.

## Consistency
- **I-19** KV, text, and vector reads in one query use one snapshot LSN.
- **I-20** Compaction never changes query results, only their cost.

## Code
- **I-21** unsafe exists only in tekwe3-sys, each block with a SAFETY comment.
- **I-22** No unwrap/expect/panic in library code.
- **I-23** No allocation is driven unbounded by untrusted input.
```

### 5.2 `docs/REJECTED.md`

Prevents each new agent generation from re-proposing the same dead ideas.

```markdown
# Rejected Approaches

Check this before proposing anything architectural.
Format: approach — why rejected — ADR.

- **eBPF kernel merge-sort for compaction** — BPF verifier rejects unbounded
  loops, >512B stack, arbitrary memory access. Structurally impossible. — ADR-0002
- **HNSW graph merging during compaction** — expensive, degrades recall.
  IVF chosen because partitions concatenate. — ADR-0003
- **Globally lock-free skiplist memtable with EBR** — bug surface not justified
  once thread-per-core removes cross-thread access. — ADR-0004
- **PGM-Index as sole key index** — collapses on clustered/adversarial keys.
  Replaced by build-time policy selection. — ADR-0005
- **Single global ZSTD dictionary** — breaks file self-containment (I-11). — ADR-0014
- **WAF target below 1.2×** — not reachable once vLog GC writes are counted.
  Target restated as 1.5–2.0×, measured from SMART. — ADR-0007
```

### 5.3 `crates/<name>/CONTEXT.md`

Read whenever a task touches that crate. Cap 40 lines. This is the briefing that replaces reading the crate.

```markdown
# tekwe3-wal — agent context

**Purpose.** Per-shard write-ahead log with group commit and parallel recovery.

**Spec:** docs/spec/05-wal.md · **Format:** docs/format/wal-record-v1.md
**Formal model:** tla/wal.tla — changes to the protocol require re-checking it.

**Invariants owned:** I-01, I-02, I-03
**Invariants relied upon:** I-04, I-08, I-16

**Constraining ADRs:** 0006 (group commit design), 0011 (no per-record fsync),
0019 (recovery parallelism)

**Depends on:** tekwe3-core, tekwe3-sys (io_uring), tekwe3-codec (crc32c)
**Depended on by:** tekwe3-engine

**Required test layers:** unit, property, deterministic sim with all 9 fault
types, dm-log-writes crash matrix, TLA+ model check

**Sharp edges**
- O_DIRECT requires 512-byte aligned buffers; use the registered buffer pool,
  never a plain Vec.
- The last record in a WAL file is expected to be partial after a crash.
  That is normal, not corruption. Do not error on it.
- Recovery is per-shard and parallel. Never introduce a shared counter.

**Do NOT**
- Add a per-record fsync "for safety" — it defeats the whole design (ADR-0011)
- Read wall-clock time here — violates I-04, breaks the simulator
```

### 5.4 `docs/adr/README.md`

The read path for decisions. One line each. Agents read this; they open a full ADR only when a line names their component.

```markdown
# ADR Index

| # | Title | Status | Touches |
|---|---|---|---|
| 0001 | Deterministic simulation built before the engine | Accepted | all |
| 0002 | Reject eBPF compaction; use io_uring_cmd + sched_ext | Accepted | sys, compaction |
| 0003 | IVF over HNSW for mergeable vector indexes | Accepted | vector |
| 0004 | Thread-per-core instead of a global lock-free memtable | Accepted | engine, memtable |
| 0005 | Build-time index policy selection | Accepted | index, sstable |
| 0006 | Group commit with one fsync per batch | Accepted | wal |
| 0007 | WAF target restated as 1.5–2.0×, SMART-measured | Accepted | vlog, bench |
| 0014 | Per-SSTable trained ZSTD dictionary | Accepted | sstable |
```

### 5.5 `docs/GLOSSARY.md`

Short definitions of terms as **this project** uses them, which is not always how the literature does.

```markdown
# Glossary

- **Shard** — a range of the keyspace exclusively owned by one core. Not a
  network partition; TEKWE3 is single-node.
- **LSN** — monotonic sequence number. Per-shard counter plus global epoch.
- **Gate** — the `just gate` verification suite. "Gate green" means it passed.
- **Policy (index)** — which of PGM / Eytzinger / FST a given SSTable uses,
  recorded as policy_id in the footer.
- **Policy (compaction)** — leveled vs tiered, chosen per key range.
- **Skew** — disagreement between the KV, text, and vector views of one record.
  TEKWE3's core claim is that skew is structurally zero.
- **Runtime** — the trait abstracting clock, disk, scheduler and RNG. Real in
  production, simulated in tests. Never bypassed.
- **Checkpoint** — a mid-task handoff block written when context runs low.
```

### 5.6 `docs/tasks/OPEN.md`

```markdown
# Open Tasks — P4

| ID | Title | Status | Blocked by |
|---|---|---|---|
| P4-007 | SSTable footer parsing | IN_PROGRESS | — |
| P4-008 | Block cache eviction (CLOCK-Pro) | READY | P4-007 |
| P4-009 | ZSTD dictionary training at write | READY | — |
| P4-010 | Format conformance test | DRAFT | P4-007, P4-009 |
| P4-011 | Split ARCHITECTURE.md into spec/ | READY | — |
```

---

## 6. Onboarding a new agent generation

When the human switches to a newer model, run this once:

```
You are joining an in-progress project. Onboard yourself in this order,
and read nothing else until you are done:

1. AGENTS.md — the whole file, once. This is your operating manual.
2. docs/INDEX.md — the router.
3. docs/INVARIANTS.md — the rules.
4. docs/REJECTED.md — do not re-propose these.
5. STATE.md — where we are.
6. docs/worklog/phase-<current-1>-summary.md — the last phase in one page.

Then report:
- Your understanding of the project thesis in 3 sentences
- The current phase and its remaining exit criteria
- Which invariants apply to the next task
- Any contradiction you found between these documents

Do not read spec files, source code, or ADRs yet. Do not write code.
I will confirm your understanding, then assign the first task.
```

If the new agent's three-sentence summary is wrong, the documentation is wrong. Fix the documentation, not the agent.

---

## 7. Documentation review at every phase close

Add to the phase closing checklist:

```
[ ] just doc-check green — all caps respected, everything indexed
[ ] Worklogs compacted into phase-N-summary.md, entries archived
[ ] Closed tasks archived; OPEN.md reflects reality
[ ] BACKLOG triaged: every item promoted or deleted with a reason
[ ] INVARIANTS.md updated with anything new the phase established
[ ] REJECTED.md updated with anything the phase ruled out
[ ] GLOSSARY.md covers every term the phase introduced
[ ] Every crate touched this phase has a current CONTEXT.md
[ ] ADR index matches the ADR files
[ ] Onboarding test: could a fresh agent start the next phase from
    INDEX + STATE + INVARIANTS + one spec file? If not, fix it now.
```

The final item is the real test. Run it honestly.
