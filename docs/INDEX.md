# docs/INDEX.md — Read this first, every session

**Purpose:** this file is a router, not a document. It tells you which files a given piece of work requires, so you never read more than you need. It is capped at 120 lines and must never grow beyond that.

**If you are a new agent:** read `AGENTS.md` §0 (context budget) and §2 (constitution) once, then come back here. Do not read anything else until this file tells you to.

**What exists today.** This repository holds the specification only: `AGENTS.md`, `AUTHORSHIP.md`, and `docs/{ARCHITECTURE,ROADMAP,CONTEXT_SYSTEM,TESTING,IDEA_INTAKE,AUDIT,KICKOFF}.md`. Every other file named below — `STATE.md`, `docs/INVARIANTS.md`, `docs/spec/*`, `docs/adr/*`, `docs/worklog/*`, `crates/*/CONTEXT.md`, `logs/*` — is a P0 deliverable that does not exist yet. The tables below are the target state: a missing file is work to create, not a broken link.

---

## Always read (Tier 0) — ~1.5k tokens

| File | What it gives you |
|---|---|
| `STATE.md` | Where the project is right now, this minute |
| `docs/INVARIANTS.md` | The numbered rules no code may break |
| this file | What to read next |

---

## Read by task type (Tier 1)

Find your row. Read only what it lists.

| If you are working on… | Read | Crate context |
|---|---|---|
| Simulator, fault injection, determinism | `spec/14-runtime-sim.md` | `tekwe3-sim/CONTEXT.md` |
| Varint, bitpacking, compression, checksums | `spec/15-codec.md` | `tekwe3-codec/CONTEXT.md` |
| SIMD kernels, runtime dispatch | `spec/16-simd.md` | `tekwe3-simd/CONTEXT.md` |
| Memtable, ART, MVCC key encoding | `spec/04-memtable.md` | `tekwe3-memtable/CONTEXT.md` |
| WAL, group commit, recovery | `spec/05-wal.md`, `tla/wal.tla` | `tekwe3-wal/CONTEXT.md` |
| SSTable read/write, block cache | `spec/06-sstable.md`, `format/sstable-v3.md` | `tekwe3-sstable/CONTEXT.md` |
| Bloom/Ribbon/SuRF filters | `spec/07-filters-index.md` | `tekwe3-filter/CONTEXT.md` |
| PGM, Eytzinger, FST, PolicySelector | `spec/07-filters-index.md` | `tekwe3-index/CONTEXT.md` |
| Value log, GC, dedup, placement hints | `spec/08-vlog.md` | `tekwe3-vlog/CONTEXT.md` |
| Compaction, policy, hotness, scheduling | `spec/09-compaction.md` | `tekwe3-compaction/CONTEXT.md` |
| MVCC, transactions, snapshots, branching | `spec/10-mvcc.md` | `tekwe3-mvcc/CONTEXT.md` |
| Tokenizer, postings, BM25, Block-Max WAND | `spec/11-text.md` | `tekwe3-text/CONTEXT.md` |
| IVF, RaBitQ, rerank, DiskANN | `spec/12-vector.md` | `tekwe3-vector/CONTEXT.md` |
| Hybrid planner, fusion | `spec/13-query.md` | `tekwe3-query/CONTEXT.md` |
| io_uring, NVMe passthrough, sched_ext, syscalls | `spec/17-kernel.md` | `tekwe3-sys/CONTEXT.md` |
| Shard runtime, public API | `spec/01-execution-model.md` | `tekwe3-engine/CONTEXT.md` |
| Benchmarks | `docs/benchmarks/README.md` | `tekwe3-bench/CONTEXT.md` |
| CLI / server | `spec/18-interfaces.md` | respective `CONTEXT.md` |

**Cross-cutting, read only when the task touches them:**

| Concern | File |
|---|---|
| The write path end to end | `spec/02-write-path.md` |
| The read path end to end | `spec/03-read-path.md` |
| Anything crossing shard boundaries | `spec/01-execution-model.md` |

---

## Read on demand (Tier 2)

| Question you have | File |
|---|---|
| "Was this decided before?" | `docs/adr/README.md` — the index. Open a full ADR only if a line names your component. |
| "Has this approach already been rejected?" | `docs/REJECTED.md` — check before proposing anything architectural |
| "What does this term mean here?" | `docs/GLOSSARY.md` |
| "What happened last session?" | `docs/worklog/` — the newest entry only |
| "What happened in an earlier phase?" | `docs/worklog/phase-N-summary.md` — never the individual entries |
| "What was deferred?" | `BACKLOG.md` |
| "Has this bug been seen before?" | `logs/failures/INDEX.md` — one line each. Open a REPORT only if the line matches. |
| "What is the current gate status?" | `logs/gate/LATEST.md` |
| "What benchmark numbers do we have?" | `logs/bench/INDEX.md` |
| "The human just had an idea" | `docs/IDEA_INTAKE.md` §8 — capture verbatim, do not implement |
| "I am running tests or triaging" | `docs/TESTING.md` |
| "Did the process change?" | `docs/PROCESS_CHANGELOG.md` — check the AGENTS.md version header first |

---

## Read rarely — announce why before reading (Tier 3)

| File | Read only when |
|---|---|
| `docs/ARCHITECTURE.md` | Starting a new phase, or resolving a spec conflict. It is an index over `docs/spec/` — you almost always want a single spec file instead. |
| `docs/ROADMAP.md` | Phase kickoff or phase close |
| `AGENTS.md` in full | First session as a new agent, or when the human says the process changed |
| Another crate's source | Never for context. Use `rg` to find one symbol, read that region only. |

---

## Where to write

| You produced… | It goes in |
|---|---|
| A decision | `docs/adr/NNNN-<title>.md` + one line in `docs/adr/README.md` |
| A rejected approach | one line in `docs/REJECTED.md` (+ ADR if non-obvious) |
| A format change | `docs/format/*.md` in the same commit as the code |
| A behavior change | the relevant `docs/spec/NN-*.md` |
| Session notes | `docs/worklog/YYYY-MM-DD-<n>.md` |
| Deferred work | `BACKLOG.md`, with enough context to act on it cold |
| A benchmark | `docs/benchmarks/<name>.md` + reproduction script + hardware manifest |
| A new term | `docs/GLOSSARY.md` |
| A new document of any kind | **also a line in this file, same commit** |

---

## Size caps — enforced at every phase close

If a file exceeds its cap, it is split, not appended to. Full rules in `docs/CONTEXT_SYSTEM.md`.

`INDEX.md` 120 · `STATE.md` 80 · `INVARIANTS.md` 80 · `REJECTED.md` 100 ·
`GLOSSARY.md` 120 · `spec/*.md` 400 each · `CONTEXT.md` 40 each ·
each ADR 50 · each worklog 60 · `BACKLOG.md` 150
