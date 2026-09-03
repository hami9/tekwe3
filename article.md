# Every RAG System Has the Same Bug, and It Is Architectural

*A design I am not going to build, published in full.*

---

If you have shipped anything with retrieval in it, you have run this stack: Postgres or RocksDB for the records, Elasticsearch or Lucene for text, Qdrant or pgvector for embeddings. Three systems, three write paths, three indexes.

And they are never simultaneously current.

A user edits a document. The KV store has the new text immediately. The full-text index catches up in a second or two. The vector index catches up whenever re-embedding finishes — seconds, sometimes minutes. In that window your hybrid query returns a result set assembled from three different points in time. The filter matches the new record, the BM25 score reflects the old text, the nearest-neighbour result reflects an embedding of a paragraph that no longer exists.

Everyone treats this as a sync problem. Add a queue, tune the refresh interval, accept eventual consistency. It gets filed under operations.

It is not an operations problem. It is what happens when three systems each own their own copy of the truth, and it does not go away with more queues. But it can be eliminated — not reduced, eliminated — and the way to do it turns on two observations that I have not seen written down together.

## One: LSM compaction *is* segment merge

An LSM tree writes immutable sorted runs and periodically merges them into larger sorted runs. Lucene writes immutable segments and periodically merges them into larger segments.

These are the same operation. Both take sorted immutable inputs and produce a sorted immutable output, discarding deleted entries along the way. RocksDB and Lucene independently built the same machine because the same problem produces the same shape.

So one compaction pipeline can maintain both a key-value store and an inverted index. When you merge two SSTables, you concatenate and re-encode their posting lists in the same pass. The text index is not a separate system with its own write path. It is a section of the file you were already writing.

## Two: IVF merges, HNSW does not

The vector side is where this usually dies, and it is why nobody has done it.

HNSW is the default choice for approximate nearest neighbour search, and it is excellent for a static index. But merging two HNSW graphs is a genuinely hard problem — expensive, and the naive approaches degrade recall. If your vector index cannot be merged cheaply, it cannot live inside an LSM, because merging is the only thing an LSM does.

IVF can. An IVF index is a set of centroids and, for each centroid, a posting list of vectors assigned to it. Merging two IVF indexes over a shared codebook is concatenating per-centroid posting lists. It is close to free.

IVF has lower recall than HNSW at equal memory. That is a real cost, and it is the price of the whole idea. But combined with modern quantization — RaBitQ gives one-bit codes with provable error bounds, scanned with SIMD popcount — and a full-precision rerank stage, it lands close enough to be competitive.

The reason this choice matters is not that IVF is better. It is that **IVF is compatible with LSM economics and HNSW is not.**

## What follows

Once both indexes merge, one SSTable can be all three things at once: a key-value segment, an inverted index segment, and an IVF vector segment. One write-ahead log. One compaction. One manifest.

And then the consistency property falls out for free. All three indexes are written by the same compaction, in the same file, referencing the same log sequence number. A hybrid query pins one LSN and reads all three from it.

Cross-index skew becomes structurally zero. Not small. Not tuned. There is no mechanism by which the three views could disagree, because there are not three views.

That produces a falsifiable experiment, which is the part I care about most. Put both systems under sustained write load. Issue a million hybrid queries against this engine and against an Elasticsearch + Qdrant stack. Count the responses where the filter, text, and vector views disagree about the same record.

One of those numbers is nonzero. The other is zero by construction. If it is not, the design is wrong and the experiment says so.

## Some things that follow from the shape

A few consequences are worth stating because they were not obvious to me at the start.

**Compaction becomes CPU-bound.** In a normal LSM, compaction is limited by disk bandwidth. Here it also re-encodes posting lists, recomputes BM25 block-max bounds, quantizes vectors, and trains a compression dictionary. The bottleneck inverts. In a thread-per-core design this is sharp: a quantization burst on a shard's core stalls that shard's reads. The scheduler has to budget CPU cycles, not just I/O.

**Index quality can be tiered by level.** LSM levels differ in size and stability, so they should not get the same index. Level 0 is small and volatile, so brute-force SIMD scanning is cheap and gives perfect recall. The top level holds most of the bytes and rarely changes, so an expensive graph build amortizes. The middle gets IVF. Index quality follows the same economics as everything else in an LSM.

**Learned indexes work if you stop treating them as a replacement.** PGM-Index is excellent on near-linear key distributions and collapses on clustered ones. But at compaction time you know the exact key distribution. So build all three candidates — PGM, an Eytzinger array over block first-keys, an FST — measure them, and record the winner in the footer. You get the learned-index win where it exists and a hard guarantee where it does not.

**The block cache has to be type-aware.** One cache serving small random KV blocks, long sequential posting lists, and hot vector centroids will thrash. A posting-list scan will evict the centroids every vector query needs. Segment the cache by content type, pin a floor for centroids and filters, and make scans non-promoting.

**Codebook versioning is the hard part of the vector story.** Retraining centroids as the embedding distribution drifts invalidates every quantized code written under the old codebook. So codebooks are versioned in the manifest and never mutated; each SSTable records which version it was written against; a query may span versions and merge candidates before rerank. Two-stage retrieval stops being an optimization and becomes structurally required — the full-precision rerank is what makes crossing a version boundary safe.

## Things I ruled out, so you do not have to

The first draft had a kernel-space compaction worker in eBPF. It is not possible. The BPF verifier rejects unbounded loops, stacks over 512 bytes, and arbitrary memory access. A merge sort cannot pass it. What you can actually do at the kernel boundary is narrower and less glamorous: NVMe passthrough via `io_uring_cmd` to skip the block layer, a `sched_ext` BPF scheduler so compaction cannot starve foreground reads, and eBPF for observability.

The first draft also claimed write amplification below 1.2× from key-value separation. That number ignores the writes the value-log garbage collector performs. The honest target is 1.5–2.0×, and it should be measured from the drive's own SMART counters rather than from the engine's accounting, because a benchmark you can fake is not a benchmark.

## Why I am publishing instead of building

The design assumes NVMe with `io_uring`, direct I/O, hole punching, and placement hints. I have a 6th-generation i5, 16 GB of RAM, a spinning disk, and Windows.

Correctness could be proven on that machine — the plan builds a deterministic simulator before the engine, so crash consistency is tested against a simulated disk with injected torn writes, reordering, and fsync failures. None of that needs good hardware.

Performance could not be measured at all. And performance is half the thesis. Building something whose central claims I cannot verify seemed worse than not building it.

So the full specification is public: architecture, on-disk format, a fifteen-phase roadmap with falsifiable exit criteria, the correctness strategy, and an honest audit of its own gaps — including one unresolved contradiction in the invariants that I left marked rather than papered over.

If you have the hardware and the interest, take it. It is CC BY, and I would genuinely like to know how it goes.

**[github.com/hami9/tekwe3](https://github.com/hami9/tekwe3)**

---

*Cite it as:* Hami. *TEKWE3: A Design for Transactionally Consistent Hybrid Search in a Unified Tri-Modal Storage Engine.* 2026. [doi.org/10.5281/zenodo.22262651](https://doi.org/10.5281/zenodo.22262651)

*TEKWE3 — created and authored by Hami. Documentation licensed under CC BY 4.0; any code under Apache-2.0 OR MIT. The TEKWE3 name is a trademark of the author and is not licensed under either.*

*The design assembles published research — PGM-Index, BuRR, RaBitQ, SuRF, Monkey, FastCDC, WiscKey, Block-Max WAND, and the deterministic simulation approach from FoundationDB. The contribution is the composition, not the components. Full credit in the repository.*
