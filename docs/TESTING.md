# docs/TESTING.md — Testing & Log Preservation System

**Purpose.** A failure that is not recorded will be rediscovered. A seed that found a bug and was then forgotten will let the bug return. This document defines how test evidence is captured, indexed, and kept cheap to read.

**Core principle:** *test output is a permanent project asset, not terminal scrollback.* Every gate run, every failure, every benchmark, every fuzz find leaves a durable, indexed artifact.

---

## 1. The log tree

```
logs/
├─ README.md                    how to read this tree. Cap 40 lines.
│
├─ gate/
│   ├─ LATEST.md                ← result of the most recent gate run. Cap 30 lines.
│   └─ 2026-03-14-1432-PASS.log full output, gzipped after 7 days
│
├─ failures/
│   ├─ INDEX.md                 ← ONE LINE PER FAILURE. This is the read path.
│   └─ F-0042-footer-len-overflow/
│       ├─ REPORT.md            triage report. Cap 60 lines.
│       ├─ repro.sh             minimal reproduction, must be runnable
│       ├─ raw.log              full original output
│       └─ artifacts/           crashing input, sim trace, core dump
│
├─ seeds/
│   ├─ regression.txt           ← every seed that EVER found a bug. Run forever.
│   ├─ nightly.txt              current wide-sweep range
│   └─ corpus/                  fuzz corpora, one directory per target
│
├─ bench/
│   ├─ INDEX.md                 ← one line per recorded result
│   ├─ HARDWARE.md              every machine used, with kernel and drive model
│   └─ 2026-03-14-p4-sstable.json
│
└─ archive/
    └─ phase-3/                 everything from closed phases, moved at phase close
```

**Read path rules.** Agents read `failures/INDEX.md` and `bench/INDEX.md`. They open a `REPORT.md` only when the index line is relevant to the current task. They read raw logs only when triaging that specific failure. Raw logs are never in the bootstrap path.

---

## 2. What must be logged, always

| Event | Artifact | Never |
|---|---|---|
| Any `just gate` run | `logs/gate/<ts>-<PASS\|FAIL>.log` + update `LATEST.md` | Report a gate result without writing the log |
| Any test failure | A new `logs/failures/F-NNNN-*/` directory | Fix a failure without recording it |
| Any sim failure | Failure dir + **the seed appended to `regression.txt`** | Discard a failing seed |
| Any fuzz crash | Failure dir + the input committed to the corpus | Delete a crashing input after fixing |
| Any benchmark | `logs/bench/<ts>-<name>.json` + `INDEX.md` line | Quote a number that has no artifact |
| Any flaky observation | Failure dir, classified `FLAKY` | Re-run until green and move on |

**The absolute rule:** if you observed it, it is recorded. An agent that fixes a bug and leaves no failure record has destroyed evidence that the next agent needs.

---

## 3. Seed management — the project's immune system

The deterministic simulator's value compounds only if failing seeds are kept.

```
logs/seeds/regression.txt
# seed      found              what it caught                  fixed in
8842        2026-03-02  F-0031 WAL partial record misparse     a91c3e2
19307       2026-03-09  F-0038 GC dropped live value           7d2b110
44120       2026-03-14  F-0042 footer length underflow         pending
```

**Rules**

1. A seed that ever produced a failure is appended to `regression.txt` **permanently**. It is never removed, even years later.
2. `just gate` runs every seed in `regression.txt` on every invocation. This set only grows.
3. Nightly CI runs `regression.txt` plus a fresh random sweep from `nightly.txt`.
4. When a new seed fails, add it **before** fixing the bug, so the fix is provably verified against it.
5. Never "fix" a failing seed by changing the seed. That is fraud against your future self.

---

## 4. Failure lifecycle

Every failure follows this path. No shortcuts.

```
1. OBSERVE      Capture the full output immediately, before any change.
                Create logs/failures/F-NNNN-<slug>/, save raw.log.

2. REPRODUCE    Reproduce it deterministically. Write repro.sh.
                If it will not reproduce → classify FLAKY, do NOT proceed
                to a fix. A flaky test is itself a bug (§6).

3. MINIMIZE     Reduce to the smallest input or shortest operation
                sequence that still fails. Record the minimized case.

4. CLASSIFY     BUG_CODE | BUG_TEST | BUG_SPEC | FLAKY | ENVIRONMENT
                BUG_SPEC means the design is wrong → escalate, ADR needed.

5. RECORD       Write REPORT.md. Add one line to failures/INDEX.md.
                If a sim seed was involved, append it to regression.txt NOW.

6. REGRESS      Write a test that fails because of this bug, and watch
                it fail. This test is permanent.

7. FIX          Smallest change that makes the regression test pass.

8. VERIFY       just gate. Paste output. Confirm the regression test
                and the recorded seed both pass.

9. CLOSE        Update INDEX.md line to FIXED with the commit SHA.
                Commit message references F-NNNN.
```

**Commit trailer for a fix:**

```
fix(sstable): reject footer with length below minimum

Failure: F-0042
Seed: 44120
Task: P4-007
Gate: PASS
```

---

## 5. Failure report template

`logs/failures/F-0042-footer-len-overflow/REPORT.md`, cap 60 lines.

```markdown
# F-0042 — footer length underflow on truncated file

**Found:** 2026-03-14, fuzz target `fuzz_footer_parse`
**Class:** BUG_CODE
**Severity:** HIGH — violates I-14 (parsing must never panic)
**Status:** FIXED in 9c41e0a
**Seed / input:** artifacts/crash-0a3f.bin, sim seed 44120

## Symptom
Subtraction overflow panic in footer.rs:88 when the file is exactly
95 bytes — one less than the fixed footer size.

## Root cause
`len - FOOTER_SIZE` computed before checking `len >= FOOTER_SIZE`.
The check existed but ran after the arithmetic.

## Invariant violated
I-14 — parsing arbitrary bytes never panics.

## Why tests missed it
The property test generated only valid footers. Truncation was tested
at block boundaries, never at sub-footer lengths.

## Fix
checked_sub with an explicit FooterTooShort error, plus a length guard
before any arithmetic.

## Permanent guards added
- Regression test `footer_shorter_than_footer_size`
- Property test extended to generate lengths 0..=FOOTER_SIZE
- Seed 44120 added to regression.txt
- Crashing input committed to the fuzz corpus

## Reproduce
    ./logs/failures/F-0042-footer-len-overflow/repro.sh
```

**Index line in `logs/failures/INDEX.md`:**

```
| F-0042 | footer length underflow | BUG_CODE | HIGH | sstable | I-14 | FIXED 9c41e0a |
```

---

## 6. Flaky test policy — zero tolerance

A test that passes sometimes is worse than a test that always fails, because it trains you to ignore red.

1. A flaky test is **never** fixed by re-running or by adding a retry.
2. Flakiness is a defect with two possible causes: the test depends on something nondeterministic (fix the test), or the code is nondeterministic (fix the code — and this likely violates **I-04/I-06**).
3. Any flaky observation gets a failure directory, class `FLAKY`, and is treated as a HIGH severity bug against the determinism invariants.
4. If a flaky test cannot be made deterministic within one session, it is **deleted** and a task is filed. A deleted test is honest; a flaky test is a lie.

---

## 7. Benchmark protocol

A number without provenance does not exist.

Every benchmark record contains:

```json
{
  "name": "p4-sstable-point-read",
  "date": "2026-03-14T14:32:11Z",
  "commit": "9c41e0a",
  "hardware_id": "dev-box-1",
  "kernel": "6.12.4",
  "command": "just bench-sstable",
  "dataset": "ycsb-c-100M-16B-keys",
  "repetitions": 10,
  "warmup": 3,
  "results": { "p50_us": 41.2, "p99_us": 118.9, "p999_us": 402.1 },
  "variance": { "p50_stddev": 0.8, "p99_stddev": 4.1 },
  "baseline": { "rocksdb_9.0": { "p50_us": 58.7, "p99_us": 201.4 } }
}
```

**Rules**

- Baseline comparisons run on the **same machine, same kernel, same dataset, same session**. Never compare against a number from a paper or from a different box.
- Minimum 10 repetitions with variance reported. A result without variance is not a result.
- Every machine used is described in `logs/bench/HARDWARE.md` and referenced by `hardware_id`.
- Any number that reaches the README traces to a benchmark JSON and a reproduction script.
- Write amplification comes from device SMART counters, never from our own accounting.

---

## 8. Test organization conventions

```
crates/<crate>/
├─ src/**             #[cfg(test)] unit tests next to the code
└─ tests/
    ├─ property.rs    proptest suites
    ├─ differential.rs  optimized vs reference implementation
    ├─ sim.rs         deterministic simulation scenarios
    └─ regression.rs  ← one test per fixed failure, named test_f0042_*
fuzz/fuzz_targets/
    └─ fuzz_<parser>.rs
```

**Naming:** `test_<unit>_<condition>_<expected>` — e.g. `test_footer_parse_truncated_returns_too_short`. Regression tests are named after their failure ID so the link is never lost.

---

## 9. Retention and rotation

| Artifact | Kept | Archived |
|---|---|---|
| `regression.txt` | **Forever** | Never |
| Fuzz corpus | **Forever** | Never |
| Failure directories | Forever | Moved to `archive/phase-N/` at phase close, index line stays live |
| Benchmark JSON | Forever | Index line stays live |
| Gate logs, PASS | 30 days | Deleted |
| Gate logs, FAIL | Forever | Attached to the failure directory |

`logs/failures/INDEX.md` and `logs/bench/INDEX.md` **never** move to archive. They are the permanent read path.

---

## 10. System prompt — testing session

Paste this when the session's purpose is testing, triage, or verification.

```
TESTING SESSION — TEKWE3

Read AGENTS.md §0 (context budget) and docs/TESTING.md before starting.
Read logs/failures/INDEX.md and logs/gate/LATEST.md. Do not read raw
logs unless triaging a specific failure.

Your job this session is evidence, not features. Rules:

1. RECORD BEFORE YOU FIX. Capture full output into a failure directory
   before changing a single line. Evidence first, always.

2. NEVER DISCARD A FAILING SEED. Append it to logs/seeds/regression.txt
   immediately, before the fix exists. It runs forever from that moment.

3. REPRODUCE BEFORE DIAGNOSING. If it will not reproduce, classify it
   FLAKY and treat it as a HIGH severity determinism bug (I-04, I-06).
   Do not guess at a fix for something you cannot trigger.

4. WRITE THE REGRESSION TEST FIRST. Watch it fail. Paste the failure.
   A regression test you never saw fail proves nothing.

5. EVERY FAILURE GETS A REPORT. logs/failures/F-NNNN-<slug>/REPORT.md
   plus one line in INDEX.md. No exceptions, however small the bug.

6. NAME THE INVARIANT. Every failure states which numbered invariant
   from docs/INVARIANTS.md it violated. If none applies, that is itself
   a finding — the invariant list is incomplete. Say so.

7. NO NUMBER WITHOUT PROVENANCE. Any benchmark you report has a JSON
   artifact, a hardware_id, at least 10 repetitions, and variance.

8. NEVER SAY TESTS PASS WITHOUT PASTING OUTPUT.

Report in this shape:
  ## Baseline        gate status before you touched anything
  ## Failures found  one block per failure: symptom, class, invariant
  ## Triage          for each: reproduce, minimize, root cause
  ## Evidence        which failure directories and seeds you created
  ## Fixes           regression test first, then the fix, then gate output
  ## Residual risk   what is still untested, honestly
  ## Log updates     which INDEX files you touched
```

---

## 11. System prompt — failure triage only

For when a specific failure needs deep investigation.

```
TRIAGE — failure F-NNNN

Read only: logs/failures/F-NNNN/, the spec file for the affected
component, docs/INVARIANTS.md, and the relevant crate CONTEXT.md.

Work through the lifecycle in docs/TESTING.md §4 and stop after step 5
(RECORD). Do not write the fix in this session.

Produce:
1. A deterministic reproduction (repro.sh) — or an explicit statement
   that it does not reproduce, and what you tried
2. The minimized failing case
3. Root cause, stated as a specific line and a specific wrong assumption
4. Classification: BUG_CODE | BUG_TEST | BUG_SPEC | FLAKY | ENVIRONMENT
5. The invariant violated
6. Why existing tests missed it  ← the most valuable output of triage
7. What permanent guards would prevent the whole class, not just this case
8. A completed REPORT.md and INDEX.md line

If the class is BUG_SPEC, stop and escalate. The design is wrong and a
human must decide before any code changes.
```
