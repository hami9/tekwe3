# docs/IDEA_INTAKE.md — Idea Capture & Change Control

**The problem this solves.** Mid-project, the human has an idea and types it into the chat. Three things then go wrong by default:

1. The idea is **lost** — it lives in a chat transcript nobody reads again.
2. The agent **implements it immediately**, in the middle of an unrelated task, producing a mixed diff and unplanned scope.
3. The agent **quietly bends the rules** to accommodate it — softening an invariant, contradicting a spec, or overwriting a decision that had good reasons behind it.

This document makes all three impossible.

**Core principle:** *capture instantly, evaluate deliberately, integrate only with approval.* An idea is data. Data is recorded before it is judged, and judged before it is acted on.

---

## 1. Document authority — what an agent may and may not change

This is the section that prevents a good idea from breaking the system.

| Tier | Files | Agent's permission |
|---|---|---|
| **A — Constitutional** | `AGENTS.md`, `docs/INVARIANTS.md`, `docs/ARCHITECTURE.md` §0 thesis, `docs/ROADMAP.md` phase structure, `docs/IDEA_INTAKE.md`, `docs/TESTING.md` | **Propose only.** Present a diff and wait for explicit human approval. Never edit unilaterally, never as a side effect of another task. |
| **B — Normative** | `docs/spec/*`, `docs/format/*`, `docs/CONTEXT_SYSTEM.md`, additions to `INVARIANTS.md` | Edit **only** with an accompanying ADR, in a dedicated commit. Never bundled with implementation. |
| **C — Operational** | `STATE.md`, `docs/worklog/*`, `docs/tasks/*`, `crates/*/CONTEXT.md`, `BACKLOG.md`, `CHANGELOG.md`, `docs/INDEX.md`, `docs/GLOSSARY.md`, `logs/**` | Edit freely as part of normal work. This is expected. |

**Hard rules**

- If an idea requires changing a Tier A file, **stop and escalate.** Do not proceed to implementation, and do not partially accommodate it.
- If an idea contradicts a numbered invariant, say so **loudly and by number**: *"This violates I-16. Adopting it means either abandoning thread-per-core or amending I-16. Which?"* Never route around an invariant silently.
- An idea is never grounds for editing a Tier A file "because the human clearly wants it." Wanting it is the input; approving the specific diff is a separate act.
- `AGENTS.md` carries a version header. Any approved change increments it, and the change is logged in `docs/adr/`. Agents check the version at bootstrap so they know if the process changed under them.

---

## 2. The intake pipeline

```
   Human types an idea, at any time, in any session
                    │
        ┌───────────▼────────────┐
        │  1. CAPTURE  (instant) │  verbatim, unedited, timestamped
        └───────────┬────────────┘  → docs/ideas/INBOX.md
                    │
                    │   ◄── the current task continues UNCHANGED
                    │
        ┌───────────▼────────────┐
        │  2. TRIAGE   (fast)    │  30 seconds of checking
        └───────────┬────────────┘
                    │
        ┌───────────┴────────────────────────────┐
        │                                        │
   already in REJECTED.md?              conflicts with INVARIANTS?
   duplicates BACKLOG?                  changes the thesis?
        │                                        │
   show the ADR,                          🚨 ESCALATE
   ask if new info                        stop, do not evaluate further
   changes it                             until the human decides
        │                                        │
        └───────────┬────────────────────────────┘
                    │  novel and non-constitutional
        ┌───────────▼────────────┐
        │  3. EVALUATE           │  written analysis, no code
        └───────────┬────────────┘
                    │
        ┌───────────▼────────────┐
        │  4. BLAST RADIUS       │  what does this invalidate?
        └───────────┬────────────┘
                    │
        ┌───────────▼────────────┐
        │  5. CLASSIFY + PROPOSE │  A / B / C / D / E
        └───────────┬────────────┘
                    │
              ⏸ HUMAN DECIDES
                    │
        ┌───────────▼────────────┐
        │  6. INTEGRATE          │  ADR, spec, tasks, index
        └────────────────────────┘
```

**The single most important rule: never implement an idea in the session it arrives.** Capture, evaluate, schedule. An idea acted on immediately corrupts the current task's diff, skips the ADR, and bypasses the blast-radius analysis. Ideas are scheduled work, not interrupts.

---

## 3. Step 1 — Capture

Happens immediately, costs almost nothing, and never blocks the current task.

`docs/ideas/INBOX.md` — append-only, never edited, never reordered:

```markdown
## IDEA-0031 — 2026-03-14 14:22 — session 47 — status: NEW

**Verbatim (human's own words, unedited):**
> what if instead of training the zstd dict per file we trained it once
> per level and versioned it in the manifest, cold levels barely change
> so the dict would be way better and we'd stop paying the training cost
> on every single sstable

**Context when raised:** mid P4-007 (footer parsing)
**Agent action taken:** captured only. Current task unaffected.
```

**Capture rules**

1. **Verbatim.** Do not clean up grammar, do not translate, do not summarize. The human's exact words are the record. Your interpretation goes in the evaluation, clearly separated.
2. **Immediate.** Captured in the same response the idea arrives in, in one line of acknowledgment. No analysis yet.
3. **Non-blocking.** The current task continues untouched. Say so explicitly: *"Captured as IDEA-0031. Continuing P4-007."*
4. **Never dropped.** Every idea gets an ID and, eventually, a disposition. There is no such thing as an idea that was quietly ignored.
5. **Never edited after capture.** Status changes go in the evaluation file, not by rewriting the capture.

---

## 4. Steps 2–3 — Triage and evaluation

Triage is fast and mechanical:

```
[ ] Is it in docs/REJECTED.md?          → show the ADR, ask if new info applies
[ ] Does it duplicate a BACKLOG item?   → merge, note the ID
[ ] Does it contradict docs/INVARIANTS? → 🚨 name the invariant, ESCALATE
[ ] Does it change the thesis (§0 of ARCHITECTURE)? → 🚨 ESCALATE
[ ] Does it require a Tier A edit?      → 🚨 ESCALATE
[ ] Does it change a format already written to disk? → HIGH severity
```

Evaluation is written, never coded. Output goes in `docs/ideas/IDEA-0031.md`:

```markdown
# IDEA-0031 — Per-level ZSTD dictionaries versioned in the manifest

**Raised:** 2026-03-14, session 47 · **Status:** EVALUATED, awaiting decision
**Capture:** docs/ideas/INBOX.md#IDEA-0031

## Restated
Train one ZSTD dictionary per LSM level instead of per SSTable, store it
in the manifest with a version, and have SSTables reference it by version
rather than embedding it.

## Merit
- Cold levels are stable, so a level dictionary would be trained on far
  more data → likely better ratio than the per-file sample
- Removes ~8% writer cost currently spent on per-file training (ADR-0014)
- Removes up to 110 KB of per-file dictionary overhead

## Conflicts
- ⚠️ **Violates I-11** (a file is self-contained and independently readable).
  An SSTable would no longer be readable without the manifest.
- Contradicts ADR-0014, which rejected a *global* dictionary for exactly
  this reason. Per-level is a middle ground ADR-0014 did not consider.
- Complicates recovery: manifest corruption would make data unreadable,
  turning a metadata failure into a data-loss failure.

## Blast radius
| Affected | Impact |
|---|---|
| I-11 | Must be amended or the idea rejected — Tier A, human decision |
| docs/format/sstable-v3.md | Dictionary section becomes a version reference |
| docs/format/manifest-v1.md | New dictionary registry |
| P4-009 (ZSTD training) | Rewritten, ~2 days of work invalidated |
| P4 completed work | Footer layout unaffected ✅ |
| P7 compaction | Level dictionary retraining becomes a compaction duty |
| Recovery | New failure mode requiring new sim scenarios |

## Cost
~1 week including new sim fault scenarios. Not compatible with any
already-written data — would need a format version bump to v4.

## Alternative not yet considered
Keep per-file dictionaries but seed training from the parent level's
dictionary. Captures most of the ratio benefit while preserving I-11.
This may be the better answer to the same problem.

## Recommendation
**Class E — escalate.** The merit is real but it trades away I-11, which
is a constitutional property. I recommend the alternative above instead:
it addresses the actual complaint (poor per-file sample quality) without
touching an invariant. Human decision required either way.
```

---

## 5. Step 5 — Classification

| Class | Meaning | Action |
|---|---|---|
| **A — Adopt now** | Small, fits the current phase, no spec change | Create a task in the current phase. Still a separate task, never folded into the running one. |
| **B — Adopt later** | Sound, but belongs to a future phase | Create the task, assign it to that phase, note it in `ROADMAP.md` |
| **C — Backlog** | Good, unscheduled | `BACKLOG.md` with enough context to act on cold |
| **D — Reject** | Does not survive evaluation | One line in `docs/REJECTED.md` + an ADR if the reasoning is non-obvious |
| **E — Escalate** | Touches Tier A, an invariant, or the thesis | Stop. Present options. Wait. |

**A "good idea" that reaches class D is a success, not a failure.** A recorded rejection with reasoning saves every future agent from re-proposing it.

---

## 6. Step 6 — Integration

Only after explicit human approval. Executed as its own commit sequence, never mixed with feature work.

```
[ ] ADR written and added to docs/adr/README.md
[ ] docs/spec/* updated (Tier B — ADR must exist first)
[ ] docs/format/* updated + version bumped if on-disk layout changed
[ ] docs/INVARIANTS.md amended — ONLY with explicit human approval
[ ] Tasks created in docs/tasks/, added to OPEN.md
[ ] Tasks INVALIDATED by this change explicitly marked SUPERSEDED,
    not silently deleted
[ ] docs/ROADMAP.md updated if a phase changed shape (Tier A → approval)
[ ] docs/INDEX.md updated if any new document appeared
[ ] docs/GLOSSARY.md updated if new vocabulary appeared
[ ] crates/*/CONTEXT.md updated for affected crates
[ ] docs/ideas/IDEA-NNNN.md status set to ADOPTED / REJECTED / DEFERRED,
    with the ADR or task IDs recorded
[ ] INBOX.md status line updated (the capture itself stays untouched)
[ ] CHANGELOG entry if user-visible
```

**Superseded work is marked, never deleted.** If IDEA-0031 invalidates task P4-009, that task file gets `**Status:** SUPERSEDED by IDEA-0031 / ADR-0022` at the top and stays in place. A future agent reading it must be able to see why it stopped mattering.

---

## 7. Self-modification — changing the process itself

Sometimes the idea is about `AGENTS.md` itself. That is legitimate; processes must evolve. It is also Tier A, so it follows a stricter path:

1. Agent proposes as an explicit diff, with the problem it solves stated concretely and, where possible, the session where the problem actually bit.
2. Human approves the exact wording. Not the intent — the wording.
3. `AGENTS.md` version header increments: `<!-- AGENTS.md v7 — 2026-03-14 -->`
4. An ADR records what changed and why.
5. A line is added to `docs/PROCESS_CHANGELOG.md`, capped at 60 lines.

Agents read the version header at bootstrap. If it differs from what the previous worklog recorded, the agent re-reads `PROCESS_CHANGELOG.md` — not all of `AGENTS.md`.

**Never** modify `AGENTS.md` as a side effect of doing something else. Process changes are their own commit, their own PR, their own decision.

---

## 8. System prompt — idea intake

Paste this alongside `AGENTS.md`. It governs every session, because ideas arrive unannounced.

```
IDEA INTAKE — TEKWE3

At any moment the human may raise an idea, mid-task, unannounced.
When that happens, follow docs/IDEA_INTAKE.md exactly.

IMMEDIATE (same response, one line, no analysis):
  1. Append it VERBATIM to docs/ideas/INBOX.md with a new IDEA-NNNN id.
     Do not clean it up, translate it, or summarize it.
  2. Say: "Captured as IDEA-NNNN. Continuing <current task>."
  3. CONTINUE THE CURRENT TASK UNCHANGED.

NEVER, under any framing:
  - Implement an idea in the session it arrives
  - Fold it into the diff of the running task
  - Edit a Tier A file to accommodate it (AGENTS.md, INVARIANTS.md,
    the thesis, the phase structure) — propose only, then wait
  - Route around an invariant quietly. If it conflicts with I-NN,
    say the number out loud and stop.
  - Drop it, however small or however bad it seems

WHEN THE HUMAN ASKS FOR EVALUATION (a separate act):
  Triage first:
    - already in REJECTED.md? show the ADR, ask if new info changes it
    - duplicates BACKLOG? merge, cite the id
    - conflicts with INVARIANTS, the thesis, or a Tier A file?
      🚨 ESCALATE. Stop there. Do not design around it.

  Then write docs/ideas/IDEA-NNNN.md containing:
    - Restated in your own words (so misunderstanding surfaces early)
    - Merit, stated honestly and at its strongest
    - Conflicts, by invariant number and ADR number
    - BLAST RADIUS: which docs, tasks, phases, and completed work this
      invalidates. Be brutal. "This invalidates 2 days of P4-009" is the
      most useful sentence you can write.
    - Cost estimate
    - At least one alternative that achieves the same goal more cheaply
    - Classification A/B/C/D/E with a recommendation

  Write no code. Evaluation is prose.

INTEGRATION happens only after explicit approval, as its own commits,
following the checklist in docs/IDEA_INTAKE.md §6. Tasks invalidated by
the change are marked SUPERSEDED, never deleted.

Your job is to protect the project from its own enthusiasm — including
the human's, and especially your own. A well-reasoned "this conflicts
with I-16, here is a cheaper alternative" is worth more than a fast yes.
```

---

## 9. System prompt — idea review session

Run this at every phase boundary to clear the inbox.

```
IDEA REVIEW — end of phase <N>

Read docs/ideas/INBOX.md and every IDEA file with status NEW or
EVALUATED. Read docs/REJECTED.md and docs/INVARIANTS.md.

For each open idea produce:
  - One-line restatement
  - Current classification and whether it should change now that the
    phase is complete and we know more
  - Whether the next phase is the right home for it

Then produce:
  - Ideas to adopt in phase <N+1>, ordered by value per unit of work
  - Ideas to reject now, with the REJECTED.md lines written out
  - Ideas that are still genuinely open, and what information would
    settle them
  - Any idea that has been open across three phase boundaries — these
    should be rejected or adopted, never left drifting

Update every idea's status. The inbox must contain zero NEW items when
this session ends.
```

---

## 10. File inventory added by this system

```
docs/ideas/
├─ INBOX.md              append-only verbatim captures. Never edited.
├─ IDEA-NNNN.md          one evaluation per idea. Cap 80 lines.
└─ archive/phase-N/      closed ideas, moved at phase close

docs/PROCESS_CHANGELOG.md   changes to AGENTS.md itself. Cap 60 lines.
```

Add to `just doc-check`:

```
→ fails if INBOX.md contains a NEW item older than one phase
→ fails if any IDEA-NNNN.md lacks a final status
→ fails if AGENTS.md changed without a PROCESS_CHANGELOG entry
→ fails if any task marked SUPERSEDED lacks the superseding ID
```
