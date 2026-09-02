# LICENSING.md — what is licensed under what

**Author and copyright holder:** Hami · **Year:** 2026

This repository contains a design specification. There is no code in it yet.
The split below is stated now so that it does not have to be reconstructed
later, when code and documentation live side by side.

---

## 1. The split

| Content | License | File |
|---|---|---|
| Documentation, specification, diagrams, prose — everything in this repository today | **CC BY 4.0** | `LICENSE` |
| Any code added later — source, tests, build scripts, benchmarks | **Apache-2.0 OR MIT**, at the user's option | `LICENSE-APACHE`, `LICENSE-MIT` |
| The name `TEKWE3`, its abbreviation, the crate namespace, the CLI binary name, any logo or wordmark | **Not licensed by either** | `AUTHORSHIP.md` §4 |

`LICENSE` is the license of the repository as it stands, which is why it holds
the CC BY 4.0 text. `LICENSE-APACHE` and `LICENSE-MIT` are placed now so the
code license is settled before the first line of code is written, not after.

## 2. What CC BY 4.0 means here

Use, adapt, translate, quote, and build on the design — commercially included.
The one condition is attribution: credit Hami, link to this repository, and
say if you changed anything. Implementing the design in code is not a
derivative of the documentation, and the resulting code is yours; the
attribution request in `README.md` is a request, not a license term.

## 3. What Apache-2.0 OR MIT means for future code

`OR` is the user's choice, not the author's — either license alone suffices.
This is the Rust ecosystem default: MIT for maximum compatibility, Apache-2.0
for its explicit patent grant.

Every source file will carry, as its first lines:

```
// SPDX-License-Identifier: Apache-2.0 OR MIT
// Copyright (c) 2026 Hami
```

## 4. Third-party material

The design assembles published research and credits it in
`docs/ARCHITECTURE.md`; `docs/CITATIONS.md` (a P0 deliverable) will record,
per algorithm: the paper, the reference implementation, that implementation's
license, and **whether its code was read**. Algorithms are not copyrightable;
implementations are. Implementing from the paper alone is the default, and
reading a GPL or AGPL reference implementation is what would create exposure.

Once code exists, dependency licenses are enforced at commit time by
`cargo-deny` in the gate, per `docs/AUDIT.md` §2 gap 1 — not checked at
release, when a copyleft dependency costs a rewrite.

## 5. Relicensing

The author is the sole copyright holder and may license the same work under
other terms, including commercially, at any time. Publishing under CC BY 4.0
does not exhaust that right, and does not revoke the CC BY grant already made
for the published versions. Outside contributions change this — see
`CONTRIBUTING.md` for the DCO policy and `AUTHORSHIP.md` §6 for its
consequence.

## 6. No warranty

Both code licenses disclaim warranty explicitly. So does this: the design is
published as-is, unbuilt and unmeasured. Its performance claims are arguments,
not results, and are labelled as such throughout.
