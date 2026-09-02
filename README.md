# TEKWE3 — A Design for Transactionally Consistent Hybrid Search

**Status: design specification. Not implemented.**
**Author: Hami, 2026.**

This repository contains a complete engineering specification for a single-node storage engine in which key-value, full-text, and vector indexes live inside the same immutable segment — sharing one write-ahead log, one MVCC snapshot, and one compaction pipeline.

I designed it. I do not have the hardware to build it. It is published here in full, with a permissive license, in case someone who does wants to.

---

## The problem

Building a retrieval product today means running three systems:

| Need | Typical tool | Structural problem |
|---|---|---|
| Key-value / transactions | PostgreSQL, RocksDB | No search |
| Full-text | Elasticsearch, Lucene | No transactional KV |
| Vector | Qdrant, Milvus, pgvector | Separate index, lagging sync |

They are never simultaneously current. A user updates a record; the KV store reflects it, the vector index still returns the old embedding. Every RAG product in production carries this bug. It is usually described as a sync problem and treated as an operational nuisance.

It is not operational. It is structural, and it can be eliminated.

## The idea

Two observations make a unified engine possible:

1. **LSM compaction *is* Lucene segment merge.** Both merge sorted immutable runs into a larger sorted immutable run. One mechanism can serve both jobs.

2. **IVF partitions merge trivially; HNSW graphs do not.** Merging two IVF indexes concatenates per-centroid posting lists — nearly free. Merging HNSW graphs is expensive and degrades recall. Choosing IVF is what makes vector indexing compatible with LSM economics.

Given those, all three indexes can be built and maintained in a single compaction pass, and all three can be read at a single log sequence number.

The result is a hybrid query — predicate filter, BM25, and approximate nearest neighbour — served from one snapshot. **Cross-index skew becomes structurally zero**, not merely small.

## What is in this repository

Every document sits at the path it expects to be read from, so a repository
that starts building can adopt the tree as-is.

| File | Contents |
|---|---|
| `docs/ARCHITECTURE.md` | Full design: execution model, data paths, on-disk format, correctness architecture |
| `docs/ROADMAP.md` | Fifteen phases with concrete, falsifiable exit criteria |
| `AGENTS.md` | Operating manual for AI contributors on a long project (`CLAUDE.md` is a symlink to it) |
| `docs/CONTEXT_SYSTEM.md` | Documentation architecture: size caps, rotation, progressive disclosure |
| `docs/TESTING.md` | Test and log-preservation system; deterministic seed management |
| `docs/IDEA_INTAKE.md` | Change control: how a mid-project idea is captured without eroding invariants |
| `docs/AUDIT.md` | Self-audit of the document set, including its own gaps |
| `docs/KICKOFF.md` | Session-by-session start guide, and unresolved decisions |
| `docs/INDEX.md` | The reading router — which files a given task requires |
| `AUTHORSHIP.md` | Copyright, moral rights, trademark, attribution |
| `LICENSING.md` | What is licensed under what |
| `article.md` | The design's argument in essay form, and `publish/` its posting kit |

## Design highlights

- **Unified tri-modal segment** — one SSTable is simultaneously a KV segment, an inverted index, and an IVF vector index
- **Build-time index policy selection** — at compaction the key distribution is fully known, so the writer builds PGM, Eytzinger, and FST candidates, measures each, and records the winner in the footer. Learned-index benefits without learned-index risk.
- **Tiered index quality by LSM level** — brute force at L0 where data is small and volatile, IVF in the middle, a graph index at the top level where data is stable and the build amortizes
- **Self-driving compaction** — hotness tracked per key range; hot ranges get leveled, cold ranges get tiered
- **Deterministic simulation testing, built first** — the simulator exists before the engine, following FoundationDB and TigerBeetle. This is the single most important decision in the plan.

## What this design explicitly rejects

Recorded so nobody repeats the analysis:

- **eBPF kernel merge-sort for compaction** — the BPF verifier rejects unbounded loops, stacks over 512 bytes, and arbitrary memory access. Structurally impossible.
- **HNSW graph merging during compaction** — expensive and recall-destroying. IVF chosen instead.
- **A globally lock-free skiplist memtable** — unjustified bug surface once thread-per-core removes cross-thread access.
- **PGM-Index as the sole key index** — collapses on clustered and adversarial key distributions.
- **Write amplification below 1.2×** — not reachable once value-log garbage collection writes are counted. The honest target is 1.5–2.0×, measured from device SMART counters.

## Why I am not building it

The design assumes NVMe with `io_uring`, direct I/O, hole punching, and placement hints. My development machine is a 6th-generation i5 with 16 GB of RAM, a spinning disk, and Windows. Correctness could be proven under simulation on that hardware; performance — which is half the thesis — could not be measured at all.

Rather than build something whose central claims I cannot verify, I am publishing the design.

## If you want to build it

Everything needed is here. A few starting notes:

- **Start at Phase 0.** The deterministic simulator is built before the engine. If it is built last, the engine has to be rewritten to become deterministic.
- **Phase 8 is a legitimate stopping point.** An LSM engine with a deterministic simulator and SMART-measured write amplification is a complete, defensible system on its own — with zero text or vector features.
- **`docs/KICKOFF.md` lists unresolved decisions**, including a genuine contradiction in the invariants around SIMD and `unsafe`. They are marked as open, not hidden.
- **The signature experiment** is in `docs/ROADMAP.md` P11: under sustained write load, issue a million hybrid queries against this engine and against an Elasticsearch + Qdrant stack, and count the responses where the three views disagree. That number is the whole thesis.

I would genuinely like to hear about it if you do. Open an issue.

## Prior work

The design assembles published research. The contribution is the composition, not the components. Full credit in `docs/ARCHITECTURE.md`:

PGM-Index (Ferragina & Vinciguerra), BuRR (Dillinger et al.), RaBitQ (Gao & Long), SuRF (Zhang et al.), Monkey and Dostoevsky (Dayan et al.), FastCDC (Xia et al.), WiscKey (Lu et al.), Block-Max WAND (Ding & Suel), SIMD-BP128 (Lemire & Boytsov), Adaptive Radix Tree (Leis et al.), and the deterministic simulation approach pioneered by FoundationDB.

## License

**Documentation and specification:** [CC BY 4.0](LICENSE) — use, adapt, and build on it freely, including commercially. Attribution required.

**Any code added later:** [Apache-2.0](LICENSE-APACHE) OR [MIT](LICENSE-MIT), at your option.

**The TEKWE3 name is not licensed** under either. The full split is in [`LICENSING.md`](LICENSING.md); copyright, moral rights, and the trademark position in [`AUTHORSHIP.md`](AUTHORSHIP.md).

Found a hole in the design? [`SECURITY.md`](SECURITY.md) says what is most worth reporting. Sending a change? [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Citation

```
Hami. TEKWE3: A Design for Transactionally Consistent Hybrid Search
in a Unified Tri-Modal Storage Engine. 2026.
https://github.com/hami9/tekwe3
```

A machine-readable `CITATION.cff` is at the repository root.

---

```
TEKWE3 — created and authored by Hami.
Documentation licensed under CC BY 4.0; any code under Apache-2.0 OR MIT.
The TEKWE3 name and logo are trademarks of the author and are
not licensed under the above.
```

*Designed by Hami, 2026. If you build it, tell me.*
