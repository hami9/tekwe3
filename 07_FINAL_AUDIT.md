# Final Audit — TEKWE3 Documentation System

**Date:** 2026-08-31 · **Scope:** all seven documents · **Verdict:** ready to start P0, with 6 gaps closed below

---

## 1. Naming — applied

The project is named **TEKWE3**, chosen by the author. All identifiers below derive from it mechanically.

| Item | Value |
|---|---|
| Project | TEKWE3 |
| Repository | `tekwe3` |
| Crate prefix | `tekwe3-` (`tekwe3-core`, `tekwe3-wal`, `tekwe3-sstable`, …) |
| Binary / CLI | `tekwe3` |
| SSTable magic | `TEKWE303` (8 bytes) |
| Simulator seed env | `TEKWE3_SEED` |
| Phase tags | `v0.8.0-P8` |

All seven documents are consistent; zero stale references remain.


---

## 2. Audit findings

### ✅ Verified sound

| Area | Assessment |
|---|---|
| Thesis | Single defensible claim, structurally supported by two real technical observations. Not a list of buzzwords. |
| Technical honesty | Every unbuildable v2 claim corrected and recorded in `REJECTED.md` so it stays corrected |
| Context economy | Four-tier reading model, 8k bootstrap cap, router-first. The core risk of a long AI project is addressed. |
| Correctness strategy | DST built first, not last. This is the only approach with a track record. |
| Traceability | Task → invariant → spec → ADR → commit → failure ID → seed. The chain is unbroken in both directions. |
| Failure memory | `regression.txt` grows forever. The project's immune system accumulates rather than resets. |
| Change control | Tier A/B/C authority prevents a mid-project idea from quietly eroding an invariant. |
| Scope realism | Three explicit exit points, with P8 named as a legitimate stopping place. |

### 🔧 Gaps found and closed in this pass

| # | Gap | Fix |
|---|---|---|
| 1 | No licensing anywhere | P0 legal foundation: dual Apache-2.0/MIT, SPDX headers, `cargo-deny` in the gate, `LICENSING.md`, ADR-0002 |
| 2 | Algorithms implemented from papers with no attribution or license trail | `docs/CITATIONS.md` — per algorithm: paper, reference implementation, its license, **and whether we read that code**. Implementing from the paper alone is the default. |
| 3 | No format compatibility promise | `docs/COMPATIBILITY.md` — a stronger commitment than semver, since users' data depends on it |
| 4 | No dependency admission criteria | `docs/DEPENDENCY_POLICY.md` — license, maintenance, MSRV, transitive weight, build-vs-audit |
| 5 | No security or disclosure policy | `SECURITY.md` in P0, before the repo is public |
| 6 | Benchmark and dataset publication rights unexamined | P14 legal review: DeWitt clauses, BEIR/SIFT/YCSB terms, `docs/benchmarks/DATASETS.md` |

### ⚠️ Remaining gaps — create these during P0

Small, but each will bite later.

**a. Resume-after-a-long-gap protocol.** Life interrupts projects. Add to `AGENTS.md` §3:

```
If the newest worklog is more than 30 days old, do not resume the
open task. Instead:
  1. Read STATE.md, INVARIANTS.md, and the last phase summary
  2. Run `just gate` — dependencies and kernels drift; assume nothing
  3. Run `cargo update --dry-run` and report what moved
  4. Re-read the open task file and state, in your own words, what it
     was for and whether it still makes sense
  5. Report drift and wait for confirmation before writing code
Cold-resuming a half-finished task is how silent corruption enters.
```

**b. Data-loss incident protocol.** Distinct from failure triage. Once the engine is public, a report of lost data is not a normal bug. Add `docs/INCIDENT.md`:

```
Severity S0 — acknowledged data lost, or silently corrupted
  → Stop all feature work. Reproduce first, always before fixing.
  → Preserve the reporter's artifacts unmodified.
  → Add the reproducing seed to regression.txt before any fix exists.
  → Publish a written postmortem, including what the tests should have
    caught and now will.
S1 — recoverable data loss, or wrong query results
S2 — availability, performance, or resource regression
```

An S0 postmortem published openly is worth more to your reputation than the bug cost you.

**c. `docs/benchmarks/` vs `logs/bench/` overlap.** Make the split explicit and enforce it in `doc-check`:

```
logs/bench/       raw artifacts: JSON, INDEX.md, HARDWARE.md — machine-written
docs/benchmarks/  narrative writeups that cite those artifacts — human-read
```

**d. Environment manifest as a P0 deliverable.** `logs/bench/HARDWARE.md` must exist and describe the dev box before P1, or early numbers become unattributable and worthless.

**e. `AGENTS.md` version header.** `docs/IDEA_INTAKE.md` §7 requires it, but `AGENTS.md` does not yet carry one. Add `<!-- AGENTS.md v1 — 2026-08-31 -->` on line 1 and create `docs/PROCESS_CHANGELOG.md`.

---

## 3. Optimizations applied

| Change | Rationale |
|---|---|
| Test layer detail lives in `TESTING.md`; `AGENTS.md` §8.2 now points to it | One authority per topic. Duplicated rules drift apart. |
| `logs/failures/INDEX.md` is the read path, never the raw logs | Keeps failure history out of the bootstrap budget while keeping it fully available |
| `docs/INDEX.md` routes to the testing and idea systems | A router that omits half the system is not a router |
| Legal checks are gate-enforced (`cargo-deny`), not launch-checked | A copyleft dependency discovered at launch costs a rewrite; discovered at commit it costs one minute |
| Citations record *whether we read the reference code* | This is the actual legal exposure. Algorithms are free; implementations are not. |

---

## 4. The document set

| # | File | Repository path | Read frequency |
|---|---|---|---|
| 00 | INDEX | `docs/INDEX.md` | Every session, first |
| 01 | Architecture | `docs/ARCHITECTURE.md` → split into `docs/spec/` in P0-011 | Rare (Tier 3) |
| 02 | Roadmap | `docs/ROADMAP.md` | Phase boundaries |
| 03 | System prompt | `AGENTS.md` | Once per agent generation |
| 04 | Context system | `docs/CONTEXT_SYSTEM.md` | When writing docs |
| 05 | Testing | `docs/TESTING.md` | Testing and triage sessions |
| 06 | Idea intake | `docs/IDEA_INTAKE.md` | Whenever an idea arrives |

---

## 5. First three sessions

**Session 1 — scaffolding.** Create the repository, the workspace, the full documentation tree, all licensing files, `cargo-deny`, `justfile` with `gate` and `doc-check`, and CI. Write `INVARIANTS.md` (I-01…I-23), `REJECTED.md`, `GLOSSARY.md`, `INDEX.md`, `STATE.md`. Write no engine code.

**Session 2 — P0-011.** Split `ARCHITECTURE.md` into 19 spec files. Write every `CONTEXT.md`. Run the onboarding test: can a fresh agent state the thesis from `INDEX + STATE + INVARIANTS` alone, under 8k tokens? If not, the split is wrong — fix it before writing code.

**Session 3 — the simulator.** `tekwe3-sim`: the `Runtime` trait, `SimRuntime`, all nine fault types, seed reproducibility. Nothing else exists yet, and that is correct.

---

## 6. Final assessment

The plan is technically honest, the documentation is economical to read, the correctness strategy has a track record, and the process survives agent replacement. That combination is rare, and it is what will make the repository credible to a reviewer.

The remaining risk is not technical. It is scope. Every phase past P8 is optional; P8 alone — an LSM engine with a deterministic simulator and SMART-measured write amplification — is already the strongest thing most systems engineers will build in a year.

Ship P8. Then decide about the rest.
