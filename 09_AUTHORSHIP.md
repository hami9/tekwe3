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

Copyright in this work belongs to **Hami**, the sole author, from the moment of creation. Registration is not required for copyright to exist under the Berne Convention, to which both Iran and Canada are parties.

**What the open-source license does:** grants the public a permission to use, modify, and redistribute the code under stated conditions.

**What it does not do:** transfer ownership. The author remains the copyright holder and may additionally license the same work under different terms to anyone, at any time, including commercially. Open-sourcing is not exclusive and is not irreversible for future versions.

**Practical consequence:** the author can dual-license — Apache-2.0/MIT for the public, and a separate commercial license for a company that wants terms the public license does not offer. Only the copyright holder can do this. It is one of the main reasons sole authorship is worth preserving.

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

**Establishing the date:** the git history is the primary record. Every commit is timestamped and signed. For stronger evidence, tag the first public release and archive it — a Software Heritage archive or a Zenodo DOI both produce a citable, timestamped record independent of GitHub.

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

---

## 7. Employment and prior claims

Worth confirming once, in writing, and then never worrying about again:

- Work produced on personal time, on personal equipment, and outside an employer's field of business is normally the author's. Some employment contracts and some jurisdictions define this more broadly than expected.
- If there is any current or past employment contract with an IP assignment clause, read it before the first public commit.
- If any doubt exists, a short written acknowledgment from the employer that this project is personal removes the ambiguity permanently. Obtaining it before publication is easy; obtaining it after the project becomes valuable is not.

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
Licensed under Apache-2.0 OR MIT.
The TEKWE3 name and logo are trademarks of the author and are
not licensed under the above.
```

For academic or technical citation:

```
Hami. TEKWE3: A Unified Tri-Modal Storage Engine with
Transactionally Consistent Hybrid Search. 2026.
https://github.com/<user>/<repo>
```

---

## 9. Checklist — complete before the repository goes public

```
[ ] Copyright line with the author's name in every source file
[ ] LICENSE-APACHE and LICENSE-MIT present at the repository root
[ ] NOTICE file with the trademark reservation from §8
[ ] This AUTHORSHIP.md committed
[ ] docs/CITATIONS.md complete — every borrowed algorithm credited
[ ] README credits the author by name, above the fold
[ ] CONTRIBUTING.md states the DCO requirement
[ ] Git commits signed (`git config commit.gpgsign true`)
[ ] Employment IP position confirmed, if applicable
[ ] Project name checked for existing trademarks in software
[ ] First release archived for a timestamped record (Software Heritage or Zenodo)
```

---

## 10. What this protects against

| Scenario | What stops it |
|---|---|
| Someone forks and removes the author's name | License §4(c) / MIT notice clause + moral rights §3 |
| A company ships it commercially without credit | Attribution required by the license; trademark blocks the name |
| A fork calls itself the same name | Trademark reservation §4 |
| Someone claims the architecture as theirs | Dated git history + archived release §5 |
| A future employer claims ownership | Prior public dated publication §7 |
| The author wants to sell a commercial license later | Sole copyright preserved by DCO policy §6 |
