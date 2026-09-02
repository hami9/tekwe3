# Security Policy

**There is no code in this repository.** TEKWE3 is a design specification, so
there is nothing here that can be exploited at runtime. That makes the
interesting reports the ones about the design itself.

## Reporting a design flaw

If you have found a hole in the correctness argument — a case where the
described protocol loses data, returns a wrong result, or breaks an invariant —
**open a public issue.** A specification bug found before implementation costs
nothing to fix and helps everyone reading it, so there is no reason to keep it
private.

Particularly wanted:

- A crash sequence the WAL and recovery design does not survive
- An MVCC or snapshot case where a hybrid query can observe two different log
  sequence numbers across the three indexes — the thesis is that this is
  structurally impossible, and a counterexample would refute it
- A compaction interleaving that drops a live key or resurrects a deleted one
- A codebook-version boundary where a query can silently return wrong
  neighbours instead of merging candidates and reranking
- Anything in `docs/ARCHITECTURE.md` that assumes a kernel, hardware, or crate
  guarantee that does not actually hold

## Reporting a vulnerability, once code exists

When an implementation lands, this section governs:

- **Use GitHub's private vulnerability reporting** (Security → Report a
  vulnerability) rather than a public issue.
- Include a reproducing case. Under this project's testing rules that means a
  **seed**: every failure is reproducible from one, per `docs/TESTING.md`.
- Expect an acknowledgement within 7 days and an assessment within 30. There
  is one maintainer; this is a best-effort timeline, not a corporate SLA.
- Coordinated disclosure: 90 days, or sooner once a fix ships. Credit is given
  unless you ask otherwise.
- There is no bug bounty.

**Supported versions:** none yet. Before 1.0, only the latest commit on the
default branch is supported.

## Severity ladder

Until `docs/INCIDENT.md` exists (a P0 deliverable, specified in
`docs/AUDIT.md` §2b), this is the ladder in force:

| Severity | Definition | Response |
|---|---|---|
| **S0** | Acknowledged data loss, or silent corruption | Stop all feature work. Reproduce **before** fixing. Preserve the reporter's artifacts unmodified. Add the reproducing seed to `regression.txt` before a fix exists. Publish a written postmortem, including what the tests should have caught and now will. |
| **S1** | Recoverable data loss, or wrong query results | Fix takes priority over feature work; regression test required |
| **S2** | Availability, performance, or resource regression | Normal triage |

A storage engine that loses data has failed at its one job, so S0 is treated as
a stop-the-line event rather than a bug of unusual size.

## Scope

In scope: this specification, and any code later added to this repository.

Out of scope: the third-party systems the design compares itself against, the
papers it cites, and anything about the repository's GitHub configuration.
