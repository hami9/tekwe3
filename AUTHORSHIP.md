# AUTHORSHIP.md — Creator's Rights & Intellectual Property

**Author and copyright holder:** Hami
**Project:** `TEKWE3`
**Created:** 2026
**Status:** Sole authorship. No contributions from third parties as of this writing.

> *This document is informational, not legal advice. For anything with real financial stakes — an acquisition, an employment dispute over ownership, or a licensing negotiation — consult a lawyer in the relevant jurisdiction.*

---

## 1. Why this file exists

Most people open-source a project and assume the license handles everything. It does not. An open-source license grants specific rights to the **code** and is silent or explicitly restrictive about everything else.

Three things stay with the author no matter what license is chosen:

1. **Copyright ownership** — the author owns the work; the license only grants others permission to use it
2. **Moral rights** — the right to be identified as author, and the right to object to distortion of the work
3. **Trademark** — the project's name and logo, which no standard open-source license grants away

This file states all three explicitly so that they are never assumed away.

---

## 2. Copyright

Copyright in this work belongs to **Hami**, the sole author, from the moment of creation. Registration is not required for copyright to exist under the Berne Convention, to which Iran, Canada, and the United States are all parties.

**What the open-source license does:** grants the public a permission to use, modify, and redistribute the code under stated conditions.

**What it does not do:** transfer ownership. The author remains the copyright holder and may additionally license the same work under different terms to anyone, at any time, including commercially. Open-sourcing is not exclusive and is not irreversible for future versions.

**Practical consequence:** the author can dual-license — Apache-2.0/MIT for the public, and a separate commercial license for a company that wants terms the public license does not offer. Only the copyright holder can do this.

**What the author actually intends, stated so nobody has to guess.** This
design is published to be used, not to be sold. Anyone who finds it useful may
build on it freely, commercially included, under the license terms — that is
what CC BY 4.0 is for and it is the intended outcome, not a fallback.

Ownership is kept for **attribution**, not for revenue: so that the work stays
credited to its author and cannot be passed off as someone else's. The
dual-licensing option above is a consequence of sole authorship, not a plan;
it stays open because closing it would cost something and keeping it costs
nothing.

If someone building this wants help, the author is available to collaborate on
a freelance basis. That is the only commercial interest here, and it is in the
work, not in the licence.

---

## 3. Moral rights (حقوق معنوی)

Under Iranian law (قانون حمایت حقوق مؤلفان، مصنفان و هنرمندان، ۱۳۴۸) and the Berne Convention, moral rights are:

- **Perpetual** — they do not expire with economic rights
- **Non-transferable** — they cannot be sold or assigned, even by agreement
- **Non-waivable** in most civil-law jurisdictions

They consist of:

| Right | Meaning here |
|---|---|
| **Attribution** (حق انتساب) | The author must be identified as the creator of this work wherever it is used or distributed |
| **Integrity** (حق حرمت اثر) | The author may object to modification or use that damages the work's reputation or misrepresents its intent |
| **Disclosure** (حق افشا) | The author decides when and how the work is first published |

**In practice this means:** even under a permissive license, someone who forks this project and removes the author's name from the copyright notices is violating the license terms *and* moral rights. Both Apache-2.0 §4(c) and MIT require the copyright notice to be preserved.

---

## 4. Trademark — the name is not licensed

**The project name, its abbreviation, and any logo are NOT covered by the code license.**

Apache-2.0 §6 is explicit: the license grants no trademark rights. MIT is silent, which produces the same result.

Reserved to the author:
- The project name in any form
- The CLI binary name and crate namespace
- Any logo or wordmark
- The domain name, if one is registered

Others may:
- Use the name **descriptively** — "compatible with `TEKWE3`", "a fork of `TEKWE3`"

Others may not:
- Name their fork or product the same thing
- Imply endorsement, affiliation, or official status
- Use the name for a competing commercial product or service

This is the same position taken by Rust, Linux, and PostgreSQL: the code is free, the name is not.

**What the reservation is for.** Not exclusivity — the design itself is free to
use and to build into a product (§2). The name is held back so that a fork
cannot present itself as the original, and so that a reader who finds something
called TEKWE3 can tell whether it came from here. It protects the attribution,
which is the one thing the author is actually keeping.

---

## 5. Original contributions — what is actually the author's

The design contains original engineering work. Recording it here establishes provenance with a date, which matters if authorship is ever disputed.

**Original to this project:**

| # | Contribution | Nature |
|---|---|---|
| N1 | Unified tri-modal segment — KV, full-text, and vector indexes in one immutable segment under one compaction and one snapshot | Novel architecture |
| N2 | Build-time index policy selection — choosing among PGM / Eytzinger / FST per file, measured at write time | Novel technique |
| N3 | Tiered index quality mapped onto LSM levels — brute force at L0, IVF mid, graph at L_max | Novel mapping |
| N4 | Per-key-range online compaction policy selection driven by hotness | Novel application of known theory |
| — | The deterministic-simulation-first process, documentation architecture, and agent operating manual | Original engineering process work |

**Not original, and credited to their authors** in `docs/CITATIONS.md`: PGM-Index, BuRR, RaBitQ, SuRF, Monkey, Dostoevsky, FastCDC, WiscKey, Block-Max WAND, SIMD-BP128, ART, CLOCK-Pro, and the deterministic simulation approach pioneered by FoundationDB.

Claiming another researcher's algorithm as one's own would destroy the credibility of the genuine contributions above. Correct attribution protects the author's claims, it does not weaken them.

**Establishing the date:** the git history is the primary record. Every commit is timestamped but deliberately not signed — see the §9 entry for why, and for the two conditions that would reverse it. The weight therefore rests on the archives, and they are in place: v1.0 was tagged and released on 2026-09-02 and is archived on Zenodo under **[10.5281/zenodo.22262651](https://doi.org/10.5281/zenodo.22262651)**, submitted to the Software Heritage archive, and included in GitHub's Archive Program. Each of those is a citable, timestamped record that does not depend on this repository continuing to exist.

---

## 6. Contributors — keeping ownership clean

While the project is solo, ownership is unambiguous. The moment a second person contributes, it stops being so unless a policy is in place before the first pull request.

**Policy: Developer Certificate of Origin (DCO).**

- Every commit requires `git commit -s`, adding `Signed-off-by: Name <email>`
- The contributor certifies they wrote the code and have the right to submit it under the project's license
- Contributors retain copyright in their own contributions; the project becomes jointly owned in parts

**Consequence to understand:** with a DCO, the author can no longer unilaterally relicense the whole project once outside contributions land, because those parts are not the author's to relicense.

**If preserving the ability to dual-license commercially matters**, use a Contributor License Agreement (CLA) instead of a DCO, in which contributors grant the author a license to their contribution broad enough to relicense. This is heavier and some contributors decline to sign it.

**Recommendation:** DCO for a portfolio and reputation project. CLA only if commercial dual-licensing is a real plan.

**Decided: DCO.** Commercial dual-licensing is not a plan here (§2), so the one
thing a CLA buys is worth nothing to this project, while its cost — asking a
contributor to sign a legal agreement before their first patch — is real. If a
contributor's work later makes part of the project unrelicensable by the author
alone, that is an acceptable consequence of a design meant to be used freely.

---

## 7. Independent contractor status and prior claims

**There is no employer.** The author works freelance, on personally owned
equipment, across three jurisdictions — Iran, Canada, and the United States
(Georgia). No employment contract exists, written or otherwise, and this
design was not produced for or delivered to any client.

That matters because the rule people worry about — the one that hands work to
an employer by default — is a rule about *employment*, and it does not reach an
independent contractor in any of the three:

| Jurisdiction | The rule for an independent contractor |
|---|---|
| **United States** | A work by an independent contractor is "made for hire" only if it falls into one of the statute's enumerated categories **and** a signed written agreement says so. Software normally satisfies neither. Absent that, the author owns it. |
| **Canada** | The employer-ownership rule applies to work made under a *contract of service* — employment. A contractor works under a *contract for services*, and keeps copyright absent a written assignment. |
| **Iran** | Under the 1379 software protection law, economic rights pass to an employer or commissioner only where creating that software was the purpose of the employment, or the subject of the contract. Neither applies here. |

The common thread: **no signed written assignment, no transfer.** None was
signed.

**Where the real exposure is, if anywhere:** not employment, but a *client
contract*. Some freelance agreements contain a clause assigning "all
intellectual property created during the term" rather than only the agreed
deliverables. Such a clause could, in principle, reach unrelated personal
work. The check worth doing once is to read the IP clause of each active
client agreement and confirm it is scoped to deliverables — and to note which
law each names as governing, since with work in three jurisdictions the answer
comes from the contract, not from a general rule.

**Moral rights are unaffected either way.** They cannot be assigned even by
agreement (§3), so attribution stays with the author regardless of how any
economic-rights question is resolved.

---

## 8. Required attribution notice

Every source file carries:

```
// SPDX-License-Identifier: Apache-2.0 OR MIT
// Copyright (c) 2026 Hami
```

`README.md`, `NOTICE`, and the whitepaper each carry:

```
TEKWE3 — created and authored by Hami.
Documentation licensed under CC BY 4.0; any code under Apache-2.0 OR MIT.
The TEKWE3 name and logo are trademarks of the author and are
not licensed under the above.
```

For academic or technical citation — this is the canonical title, and it is
the one recorded in `CITATION.cff`:

```
Hami. TEKWE3: A Design for Transactionally Consistent Hybrid Search
in a Unified Tri-Modal Storage Engine. 2026.
https://github.com/hami9/tekwe3
```

---

## 9. Checklist — complete before the repository goes public

```
[x] LICENSE — CC BY 4.0, covering the documentation this repository holds
[x] LICENSE-APACHE and LICENSE-MIT present at the repository root
[x] NOTICE file with the trademark reservation from §8
[x] LICENSING.md — the split, stated before code makes it ambiguous
[x] This AUTHORSHIP.md committed
[x] README credits the author by name, above the fold
[x] CONTRIBUTING.md states the DCO requirement
[x] SECURITY.md — disclosure policy, and how to report a design flaw
[x] CITATION.cff — machine-readable, canonical title
[ ] Copyright line with the author's name in every source file
      → nothing to do until code exists; SPDX header specified in LICENSING.md §3
[ ] docs/CITATIONS.md complete — every borrowed algorithm credited
      → P0 deliverable; docs/ARCHITECTURE.md carries the credit meanwhile
[—] Git commits signed — decided against, 2026-09-03, revisit if either
      condition below changes
      → What signing would prove — that a commit bearing the author's name
        was actually made by the author — is covered here by the archives:
        the Zenodo DOI, Software Heritage, and GitHub's Archive Program each
        hold a dated copy that does not depend on git metadata.
      → Most commits in this repository are made by an AI agent from a remote
        environment that does not hold the author's key, so signing would
        produce a partly-signed history rather than a signed one, and
        GitHub's vigilant mode would then mark every agent commit
        "Unverified" — worse-looking than the unsigned state it replaces.
      → Revisit when either changes: a second person contributes, or the
        repository starts carrying code. At that point impersonation has
        something worth stealing and the calculation flips.
      → This does not touch the DCO. `git commit -s` is still required of
        every contributor per §6 and `CONTRIBUTING.md`; a sign-off is a
        legal assertion in the message, not a cryptographic signature, and
        the two are unrelated despite the shared word.
[~] Employment IP position confirmed, if applicable
      → Not applicable: there is no employer. The author is an independent
        contractor working on personally owned equipment, and in all three
        jurisdictions involved a contractor keeps copyright absent a signed
        written assignment — none was signed. See §7.
      → One check remains, and it is about clients rather than employment:
        confirm the IP clause of each active client agreement is scoped to
        that engagement's deliverables, not to "all intellectual property
        created during the term".
[~] Project name checked for existing trademarks in software
      → informal search, 2026-09-02: no crate on crates.io, no npm package,
        and no software trademark found under TEKWE3 or TEKWE. Nearest hit
        is TEK WEH (US reg. 5522942), pharmaceuticals — a different class,
        different mark. This is a clearance search, not legal clearance:
        the register still has to be searched in the jurisdictions that
        matter before the name is relied on commercially.
[x] First release archived for a timestamped record (Software Heritage or Zenodo)
      → v1.0, released 2026-09-02. Zenodo: doi.org/10.5281/zenodo.22262651.
        Also submitted to the Software Heritage archive, and GitHub's own
        Archive Program is enabled for the repository.
```

Two items remain, and both wait for code to exist: the per-file copyright line
and `docs/CITATIONS.md`. Nothing is blocked on an action outside the
repository. Two lines carry a `[~]` because they are answered but not closed —
the name clearance search is not legal clearance, and the IP position leaves
one client-contract check worth doing once — and commit signing is `[—]`,
closed by a decision rather than an action, with the reasoning above so a
reader six months from now can judge whether it still holds.

---

## 10. What this protects against

| Scenario | What stops it |
|---|---|
| Someone forks and removes the author's name | License §4(c) / MIT notice clause + moral rights §3 |
| A company ships it commercially without credit | Attribution required by the license; trademark blocks the name |
| A fork calls itself the same name | Trademark reservation §4 |
| Someone claims the architecture as theirs | Dated git history + archived release §5 |
| A future employer or client claims ownership | No signed assignment and contractor status §7, plus the dated public record §5 |
| The author ever needs to license commercially — not a plan, but not foreclosed | Sole copyright preserved by DCO policy §6 |
