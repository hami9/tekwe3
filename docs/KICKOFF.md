# Kickoff — TEKWE3

Everything needed to start the agent. Read this, then paste Session 1.

---

## 1. Findings from the last pass — resolve these first

### 🔴 F1 — A contradiction inside our own documents

`INVARIANTS.md` I-21 says *"unsafe exists only in `tekwe3-sys`."* But `tekwe3-simd` needs SIMD intrinsics, and `core::arch` intrinsics are `unsafe`. The two rules cannot both hold.

This must be decided by you, not silently by the agent. **ADR-0004, session 1:**

| Option | Toolchain | Safety | Cost |
|---|---|---|---|
| **A** — `std::simd` | **Nightly required** | Safe, portable, no unsafe at all | Nightly toolchain forever; API can shift |
| **B** — `wide` / `pulp` crates | Stable | Safe wrappers over intrinsics | Less control over exact instruction selection |
| **C** — `core::arch` directly | Stable | Requires amending I-21 to permit unsafe in `tekwe3-simd` | Maximum control, more unsafe to audit |

**Recommendation: B, escalating to C only where a benchmark proves B leaves performance on the table.** That keeps the toolchain on stable and preserves I-21 for now. But it is your call, and it is constitutional — amending an invariant is Tier A.

### 🟡 F2 — `AGENTS.md` violates its own rule

It is 869 lines, roughly 11k tokens. It is loaded every session, which contradicts the 8k bootstrap budget in its own §0. Split it:

```
AGENTS.md          ~250 lines — ALWAYS loaded
                   §0 context budget · §1 identity · §2 constitution
                   §3 session protocol · §10 escalation · §11 response format
                   §14 failure modes

docs/PROCESS.md    ~600 lines — read on demand
                   §4 tasks · §5 git · §6 documentation · §7 anti-hallucination
                   §8 testing gates · §9 templates · §12–13 phase prompts
```

This is task **P0-002**. Do it in session 1, before anything else costs a session's context.

### 🟡 F3 — Toolchain decisions are unmade

The agent will otherwise pick these silently. Settle them in session 1 as ADRs:

- Rust edition and MSRV, pinned in `rust-toolchain.toml`
- Stable vs nightly (decided by F1)
- Allocator: system vs `mimalloc` vs `jemalloc` — matters for a thread-per-core design
- Error crate: `thiserror` confirmed; `anyhow` **forbidden** in library crates

### 🟢 F4 — Session 1 is the only session with no bootstrap

The repository does not exist yet, so there is no `STATE.md` to read. Tell the agent explicitly, or it will hunt for files that are not there. The Session 1 prompt below already says so.

### 🟢 F5 — `docs/AUDIT.md` is not a repository document

It is the P0 backlog. Convert its findings into task files, then archive it at `docs/worklog/phase-0-inputs.md`. Do not leave it in a build repository's `docs/` as a permanent file.

**Deviation, deliberate:** in *this* repository — a published design specification with no code — the audit and this kickoff stay visible at `docs/AUDIT.md` and `docs/KICKOFF.md`. A reader evaluating the design should be able to find its self-criticism without digging through a worklog archive. F5 applies the moment an implementation repository exists.

---

## 2. File → repository path

The documents already sit at these paths in this repository, so a build repository can copy the tree unchanged.

| Document | Path | What still has to happen to it |
|---|---|---|
| Router | `docs/INDEX.md` | Nothing — read it first, every session |
| Architecture | `docs/ARCHITECTURE.md` | Split into `docs/spec/` (P0-011) |
| Roadmap | `docs/ROADMAP.md` | Nothing |
| Operating manual | `AGENTS.md` (+ `CLAUDE.md` symlink) | Split per F2 (P0-002) |
| Context system | `docs/CONTEXT_SYSTEM.md` | Nothing |
| Testing | `docs/TESTING.md` | Nothing |
| Idea intake | `docs/IDEA_INTAKE.md` | Nothing |
| Audit | `docs/AUDIT.md` | Convert to tasks → `docs/worklog/phase-0-inputs.md` (see F5) |
| Kickoff | `docs/KICKOFF.md` | This file → `docs/worklog/phase-0-inputs.md` too (see F5) |
| Authorship | `AUTHORSHIP.md` | Nothing — root, alongside `LICENSING.md` and `NOTICE` |

**Tooling note.** For Claude Code, `AGENTS.md` at the repository root is picked up automatically; also symlink it as `CLAUDE.md`. For Cursor, `.cursorrules`. For anything else, paste it as the system prompt.

---

## 3. Capture your hardware before session 1

`logs/bench/HARDWARE.md` must exist before any benchmark. Run these and keep the output:

```bash
uname -r
lscpu | head -25
lsb_release -a 2>/dev/null || cat /etc/os-release
free -h
nvme list
sudo nvme id-ctrl /dev/nvme0 | grep -iE "awupf|awun|mn |fr "
cat /sys/block/nvme0n1/queue/logical_block_size
cat /sys/block/nvme0n1/queue/physical_block_size
ls /dev/ng* 2>/dev/null && echo "NVMe passthrough available"
grep -q sched_ext /proc/kallsyms 2>/dev/null && echo "sched_ext present"
```

`AWUPF` decides whether the WAL-less path (N12) is even possible. Find out now.

---

## 4. Session 1 prompt — paste this

```
SESSION 1 — TEKWE3, project bootstrap

You are a senior systems engineer starting a new project. This is the
only session with no bootstrap ritual: the repository does not exist yet,
so there is no STATE.md to read. Do not look for one.

I am providing 7 documents. Read them in this order:
  1. AGENTS.md               → your operating manual. Read fully, once.
  2. docs/ARCHITECTURE.md    → the design
  3. docs/ROADMAP.md         → phases and exit criteria; you are in P0
  4. docs/INDEX.md           → the router
  5. docs/CONTEXT_SYSTEM.md  → documentation architecture and size caps
  6. docs/TESTING.md         → test and log preservation
  7. docs/IDEA_INTAKE.md     → idea capture and change control

YOUR JOB THIS SESSION: scaffolding only. Write NO engine code. Not one
line. If you feel the urge to implement something, that is the failure
mode this rule exists to prevent.

Deliverables:

A. Repository skeleton
   - Cargo workspace named tekwe3, crates prefixed tekwe3-
   - All 20 crate directories from ARCHITECTURE §7, each with lib.rs
     containing only the crate-level lints and a doc comment
   - rust-toolchain.toml with a pinned version
   - Shared workspace lints: forbid(unsafe_op_in_unsafe_fn), clippy config

B. Documentation tree
   - Place all 7 documents at the paths in the mapping I gave you
   - Write docs/INVARIANTS.md — the numbered rules I-01..I-23, taken from
     ARCHITECTURE and CONTEXT_SYSTEM §5.1
   - Write docs/REJECTED.md from ARCHITECTURE §1
   - Write docs/GLOSSARY.md
   - Write STATE.md, BACKLOG.md, CHANGELOG.md
   - Create empty docs/adr/README.md, docs/tasks/OPEN.md, docs/ideas/INBOX.md
   - Create the logs/ tree from TESTING §1 with empty INDEX files
   - Create all templates: ADR, task, worklog, PR

C. Legal foundation (ROADMAP P0)
   - LICENSE-APACHE, LICENSE-MIT
   - deny.toml rejecting copyleft, wired into the gate
   - SECURITY.md, CONTRIBUTING.md with DCO, NOTICE
   - docs/LICENSING.md, docs/CITATIONS.md (skeleton with every algorithm
     named in ARCHITECTURE §2), docs/COMPATIBILITY.md,
     docs/DEPENDENCY_POLICY.md

D. Automation
   - justfile with `gate` and `doc-check` recipes per AGENTS §8.1
   - .github/workflows/ci.yml with the kernel matrix 6.1/6.6/6.12/latest

E. Decisions I must approve — present as ADR drafts, do NOT decide alone
   - ADR-0002 licensing (dual Apache-2.0/MIT — confirm)
   - ADR-0003 name availability: check crates.io and GitHub for
     tekwe3. Report what you find.
   - ADR-0004 SIMD approach. There is a CONTRADICTION in the docs:
     I-21 says unsafe only in tekwe3-sys, but core::arch intrinsics are
     unsafe and tekwe3-simd needs them. Present options A/B/C from the
     kickoff document with your recommendation. Do not resolve it
     yourself — amending an invariant is Tier A.
   - ADR-0005 toolchain: edition, MSRV, allocator, stable vs nightly

F. First tasks
   - Write docs/tasks/ files for P0-002 (split AGENTS.md per F2),
     P0-011 (split ARCHITECTURE into docs/spec/), and the next 5 P0 items
   - Populate docs/tasks/OPEN.md

Then STOP and report:
   - What you created, as a tree
   - The 4 ADR drafts awaiting my decision
   - Any other contradiction or ambiguity you found in the 7 documents
     — I want these, do not smooth them over
   - just gate output (it should pass trivially on an empty workspace)
   - STATE.md as you wrote it

Do not start P0-002 or any other task. I will approve the ADRs first.
```

---

## 5. Session 2 and 3

**Session 2 — documentation split.** Normal bootstrap now applies.

```
SESSION 2 — task P0-002 and P0-011

Bootstrap normally per AGENTS.md §3.1.

Then execute, in order:
  1. P0-002 — split AGENTS.md into AGENTS.md (~250 lines, always loaded)
     + docs/PROCESS.md (~600 lines, on demand). Boundaries are in the
     kickoff document F2. Update docs/INDEX.md to route to both.
  2. P0-011 — split docs/ARCHITECTURE.md into the 19 files under
     docs/spec/. Content stays identical; only the container changes.
     Reduce ARCHITECTURE.md to an 80-line index.
  3. Write crates/*/CONTEXT.md for all 20 crates, using the template in
     CONTEXT_SYSTEM §5.3. Each is capped at 40 lines.

Then run the ONBOARDING TEST on yourself and report honestly:
  Given ONLY docs/INDEX.md + STATE.md + docs/INVARIANTS.md +
  docs/spec/05-wal.md + crates/tekwe3-wal/CONTEXT.md, could a fresh agent
  state the project thesis, the current phase, and the applicable
  invariants — under 8k tokens of reading?

  If no, the split is wrong. Fix it now. This is the last cheap moment
  to fix it.
```

**Session 3 — the simulator.** The first code in the project.

```
SESSION 3 — P0: the deterministic simulator

Bootstrap normally.

Implement tekwe3-sim: the Runtime trait (Clock, Disk, Scheduler, Rng),
RealRuntime, and SimRuntime with all 9 fault types from ROADMAP P0.

Test-first, as always. The defining exit criterion:

  TEKWE3_SEED=42 cargo test -p tekwe3-sim, run twice → identical trace hash

Every fault type needs a test proving it can fire and is observed.

No other crate gets code this session. The simulator exists before the
engine, and that ordering is the single most important decision in this
project.
```

---

## 6. Watch for these in the first month

| Signal | What it means | Do |
|---|---|---|
| Agent says "tests pass" with no output | Constitution §2.2 broken | Reject the work, quote the rule |
| A diff exceeds 400 lines | Task was too big | Split it, do not merge |
| `STATE.md` untouched for 3 sessions | Handoff is decaying | Stop features, fix the docs |
| Agent proposes eBPF compaction again | It has not read `REJECTED.md` | Fix the router, not the agent |
| A failing seed appears in a log but not `regression.txt` | The immune system is leaking | Highest priority fix |
| An idea gets implemented in the session it arrives | `IDEA_INTAKE` is being skipped | Revert it, capture properly |
| Bootstrap reading exceeds 8k tokens | Documents outgrew their caps | Run `doc-check`, split |

---

## 7. Start here

1. Decide F1 (the SIMD/unsafe contradiction) — or let the agent present it and decide next session
2. Run the hardware capture in §3, keep the output
3. Paste Session 1
4. Approve the ADRs
5. Session 2, then session 3

The simulator is the first code. Everything else waits for it.
