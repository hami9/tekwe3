# Contributing to TEKWE3

This repository is a **design specification**, not an implementation. That
shapes what a useful contribution looks like: the most valuable thing you can
send is an argument that some part of the design is wrong, with the reasoning
attached.

---

## What is most wanted

1. **A refutation.** A case the design does not handle, an invariant that
   cannot hold, a performance claim that the hardware will not support. The
   thesis — that cross-index skew can be made structurally zero — is meant to
   be falsifiable. See `SECURITY.md` for the correctness-flaw list.
2. **A measurement.** Anyone with an NVMe machine can test claims this design
   asserts but has never run. Numbers, with the hardware manifest that makes
   them reproducible, are worth more than any prose here.
3. **A missing citation.** If prior art already does something the design
   presents as new, that needs correcting — `AUTHORSHIP.md` §5 lists what is
   claimed as original, and each of those claims is contestable.
4. **An implementation.** Fork it and build it. You do not need permission,
   and you do not need to contribute it back. Open an issue if you start; the
   author would like to know.

Typo and clarity fixes are welcome too, as ordinary pull requests.

## Before proposing a change to the design

The design has a change-control system, and it applies to outside proposals
exactly as it applies to the author:

- **Read `docs/IDEA_INTAKE.md` first.** It defines the tiers: an idea that
  requires editing a Tier A file — `AGENTS.md`, the invariants, the roadmap's
  shape, the thesis — is escalated and decided explicitly, never absorbed
  quietly into a diff.
- **Check `docs/AUDIT.md` §2 and the "What this design explicitly rejects"
  section of `README.md`.** Several approaches have already been analysed and
  ruled out with reasons. Re-proposing one is fine, but engage with the
  recorded reason.
- **Respect the size caps** in `docs/CONTEXT_SYSTEM.md` (`docs/INDEX.md` 120
  lines, `spec/*` 400, and so on). A document over its cap is split, not
  appended to. A pull request that pushes a file over its cap will be asked to
  split it.
- **One idea per pull request.** A change to the design and a change to the
  process are two pull requests.

## Developer Certificate of Origin

Every commit must be signed off:

```bash
git commit -s -m "your message"
```

which appends:

```
Signed-off-by: Your Name <your.email@example.com>
```

By signing off you certify the [Developer Certificate of Origin
1.1](https://developercertificate.org/): that you wrote the contribution or
otherwise have the right to submit it under the project's license.

Use your real name and a working email address. Sign-off is required on every
commit in the pull request, not only the last one.

**What this means for ownership,** stated plainly because it is easy to get
wrong later: you keep the copyright in what you write. The project becomes
jointly owned in parts, and the author consequently cannot relicense those
parts alone. This is the deliberate choice recorded in `AUTHORSHIP.md` §6 — a
DCO rather than a CLA — and it is not going to be swapped for a CLA after the
fact.

## License of contributions

- **Documentation and specification changes** are contributed under
  **CC BY 4.0**, the license of the material they modify.
- **Code**, when there is any, is contributed under **Apache-2.0 OR MIT**, with
  the SPDX header on every file.

See `LICENSING.md`. Do not paste text or code from a source you do not have the
right to relicense — including a reference implementation of a paper the design
cites. `docs/CITATIONS.md` will record, per algorithm, whether the reference
code was read; if you read one before writing a section, say so in the pull
request.

## Style

Match what is there. Specifically:

- Plain declarative English. No marketing register, no exclamation marks.
- Claims are labelled by kind. A measured number, an estimate, and a guess are
  three different things and are never written as if they were one.
- Rejections are recorded with their reason, so nobody repeats the analysis.
- Tables where the content is tabular; prose where it is an argument.

## Getting an answer

Open an issue. There is one maintainer and no support commitment, but genuine
technical questions about the design are the reason this repository exists.
