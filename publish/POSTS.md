# Publication kit

Ready-to-paste versions of `../article.md` for each platform. The article is
the canonical text; everything here is a framing of it.

**One rule across all of them:** the honest hook is *"I designed this and I am
not building it, here is why."* It is unusual, it is checkable, and it is the
reason the post is worth reading rather than the thousandth RAG take. Do not
bury it, and do not dress the design up as something running in production.

---

## 1. Title options

| # | Title | Where it fits |
|---|---|---|
| 1 | **Every RAG system has the same bug, and it is architectural** | The article's own title. Strongest on dev.to, LinkedIn, X. Reads as a claim, which is exactly what it is. |
| 2 | **A storage engine where KV, full-text, and vector share one segment** | Best for Hacker News. Describes the object instead of the grievance; HN rewards that. |
| 3 | **LSM compaction is Lucene segment merge, and that makes hybrid search consistent** | Leads with the technical observation. Good for r/databases, Lobsters, systems-heavy audiences. |
| 4 | **I designed a tri-modal storage engine and I am not going to build it** | Leads with the unusual part. Highest curiosity, lowest technical signal — use as a fallback, not a default. |

Avoid: anything with "revolutionary", "the future of", or a number in front of
a list. The design's credibility rests on its restraint.

---

## 2. Hacker News

**Not a Show HN.** Show HN is for something the reader can run. This is a
specification, and mislabelling it will draw the one comment you cannot
recover from.

**Submit:** title option 2, URL `https://github.com/hami9/tekwe3`.

Then post this immediately as the first comment — on HN the author's first
comment does more work than the title:

```
Author here. This is a design document, not software: there is no code in
the repository and I am not planning to write any. Explaining why is
probably the most useful thing I can do in this comment.

The problem it addresses is the one every retrieval stack has. You run a
KV store, a full-text index, and a vector index. They are never
simultaneously current, so a hybrid query assembles its answer from three
different points in time. That is normally treated as a sync problem and
tuned. I think it is structural, and removable.

Two observations make removal possible. First, LSM compaction and Lucene
segment merge are the same operation - merge sorted immutable runs into a
larger sorted immutable run - so one pipeline can maintain both a KV store
and an inverted index. Second, IVF vector indexes merge cheaply (concatenate
per-centroid posting lists) where HNSW graphs do not. IVF costs recall
against HNSW at equal memory; that is the price, paid back with RaBitQ
one-bit codes and a full-precision rerank. The reason to choose it is not
that it is better, it is that it is compatible with LSM economics.

Given both, one SSTable can be a KV segment, an inverted index, and a vector
index at once, written by one compaction at one log sequence number. Skew
between the three views becomes zero by construction rather than small by
tuning. That is testable: put both stacks under sustained write load, issue
a million hybrid queries, count the responses where the three views disagree
about the same record. One number is nonzero, the other should be zero. If
it is not, the design is wrong.

Why I am not building it: the design assumes NVMe with io_uring, direct I/O,
hole punching and placement hints. I have a 6th-gen i5, 16GB, a spinning
disk, and Windows. Correctness I could prove there - the plan builds a
deterministic simulator before the engine, FoundationDB-style. Performance I
could not measure at all, and performance is half the thesis. Publishing the
design seemed better than building something whose central claims I cannot
check.

So it is all there: architecture, on-disk format, a fifteen-phase roadmap
with falsifiable exit criteria, the rejected approaches with their reasons
(including a kernel-space eBPF compaction worker that the BPF verifier makes
impossible), and an audit of the document set's own gaps - including one
unresolved contradiction in the invariants that is marked rather than
papered over. CC BY. If you have the hardware, take it.

Happy to argue about any of it, especially the IVF-over-HNSW call and the
claim that skew goes to exactly zero.
```

**Timing:** weekday, 08:00–10:00 US Eastern. Stay at the keyboard for the
first two hours; on HN the discussion is the post.

**Expect, and prepare for:**

- *"Just use pgvector / Postgres with the right extensions."* True for most
  products, and worth conceding immediately. The design's claim is about
  structural skew under sustained write load, not about whether a simpler
  stack is usually the right call.
- *"Single-node only?"* Yes. Distribution is out of scope and stated as such.
- *"IVF recall is worse than HNSW."* Yes — say so before they do; that is why
  the top LSM level gets a graph index and why rerank is structurally
  required, not an optimization.
- *"Vaporware / AI-written spec."* The strongest answer is the audit: the
  document set criticises itself, names its gaps, and records what it ruled
  out and why. Point at `docs/AUDIT.md` and the rejected-approaches list.

---

## 3. dev.to

Front matter, then the body of `../article.md` unchanged:

```yaml
---
title: "Every RAG System Has the Same Bug, and It Is Architectural"
published: true
description: "Three systems own three copies of the truth, so a hybrid query answers from three points in time. A design that removes the skew structurally - published unbuilt, with the reason."
tags: databases, rag, architecture, vectorsearch
canonical_url: https://github.com/hami9/tekwe3
---
```

Notes:

- dev.to allows four tags; the four above are the ones with real traffic here.
- Set `canonical_url` to the repository so search engines credit it, not the
  mirror.
- The article's own H1 is dropped by dev.to in favour of the front-matter
  title — start the pasted body at *"If you have shipped anything with
  retrieval in it…"*.
- Same body works on Medium and Hashnode; on Medium set the canonical link in
  story settings, or the repository loses the ranking.

---

## 4. X / Twitter thread

Seven posts. Post 1 is the whole argument in miniature — most readers will see
nothing else.

```
1/ Every RAG stack has the same bug.

You run a KV store, a text index, and a vector index. They are never
simultaneously current. So a hybrid query answers from three different
points in time.

It is treated as a sync problem. It is structural, and it can be removed.

2/ Observation one:

LSM compaction and Lucene segment merge are the same operation. Both merge
sorted immutable runs into a larger sorted immutable run.

RocksDB and Lucene built the same machine independently, because the same
problem has the same shape.

3/ So one compaction pipeline can maintain a KV store *and* an inverted
index. Merging two SSTables also concatenates and re-encodes their posting
lists, in the same pass.

The text index stops being a separate system. It is a section of the file
you were already writing.

4/ Observation two: the vector side is where this usually dies.

HNSW is the default, and merging two HNSW graphs is expensive and wrecks
recall. If your vector index cannot merge cheaply, it cannot live in an LSM,
because merging is all an LSM does.

IVF can. It is a concatenation.

5/ IVF has worse recall than HNSW at equal memory. That is the real price.

Paid back with RaBitQ one-bit codes scanned by SIMD popcount, plus a
full-precision rerank.

The point is not that IVF is better. It is that IVF is compatible with LSM
economics and HNSW is not.

6/ Now one SSTable is all three: KV segment, inverted index, IVF index. One
WAL, one compaction, one manifest.

A hybrid query pins one log sequence number and reads all three from it.

Cross-index skew becomes zero by construction. Not small. Zero.

7/ Which is falsifiable, and that is the part I care about.

Sustained write load, a million hybrid queries, count the responses where
the three views disagree. One stack's number is nonzero. Mine should be
zero, or the design is wrong.

I do not have the hardware to run it. So it is published instead:
github.com/hami9/tekwe3
```

Keep the link in the last post only. Reply to your own thread with the
hardware explanation if anyone asks why it is unbuilt — it converts skeptics
better as an answer than as a claim.

---

## 5. LinkedIn

```
I spent a long time designing a storage engine, and then decided not to
build it. The full specification is now public.

The problem: every retrieval product runs a KV store, a full-text index, and
a vector index. They are never simultaneously current, so a hybrid query
answers from three different points in time. The industry treats this as a
sync problem to be tuned. I think it is structural.

The design puts all three indexes inside the same immutable segment, built
by one compaction pass and read at one log sequence number - which makes the
skew between them zero by construction rather than small by configuration.
It turns on two observations: that LSM compaction and Lucene segment merge
are the same operation, and that IVF vector partitions merge cheaply where
HNSW graphs do not.

Why publish instead of build: the design assumes NVMe hardware I do not
have. Correctness I could prove in simulation; performance I could not
measure, and performance is half the claim. Publishing a design I can defend
seemed better than shipping numbers I cannot verify.

Architecture, on-disk format, a fifteen-phase roadmap with falsifiable exit
criteria, the approaches I ruled out and why, and an audit of the document
set's own gaps. CC BY - if you have the hardware, take it.

github.com/hami9/tekwe3
```

---

## 6. After posting

- Answer the first ten comments on every platform, including the dismissive
  ones. A design published without a defence reads as abandoned.
- Record any refutation that lands as an issue in the repository, in the
  reporter's words. `docs/IDEA_INTAKE.md` §8 is explicit about capturing an
  idea verbatim before deciding anything about it.
- If someone starts building it, link their repository from the README.
