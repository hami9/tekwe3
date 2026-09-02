# TEKWE3 v3 — Architecture Specification

**Status:** Normative. This document is the single source of truth for design.
**Rule:** Code that contradicts this document is a bug. Design that contradicts this document requires an ADR that supersedes the relevant section.

---

## 0. Thesis

> A single-node, thread-per-core, crash-consistent storage engine in which **key-value**, **full-text (BM25)**, and **vector (ANN)** indexes live inside the same immutable segment, share one write-ahead log, one MVCC snapshot, and one compaction pipeline — delivering **transactionally consistent hybrid search**, which no single-node system offers today.

### 0.1 Why this thesis is defensible

Building a retrieval product today requires gluing three systems together:

| Need | Typical tool | Structural problem |
|---|---|---|
| Key-value / transactions | PostgreSQL, RocksDB | No search |
| Full-text | Elasticsearch, Lucene | No transactional KV |
| Vector | Qdrant, Milvus, pgvector | Separate index, lagging sync |

The result is **eventual consistency between three systems**. A user updates a record; the KV store is current, the vector index is stale. Every RAG product in production has this bug, and nobody has structurally eliminated it.

TEKWE3 eliminates it by construction, based on two observations:

1. **LSM compaction *is* Lucene segment merge.** Both are "merge sorted immutable runs into a larger sorted immutable run." One mechanism can serve both jobs.
2. **IVF partitions merge trivially; HNSW graphs do not.** Merging two IVF indexes is concatenating per-centroid posting lists — nearly free. Merging two HNSW graphs is expensive and degrades recall. Choosing IVF makes vector indexing compatible with LSM economics.

Given those, all three indexes can be built and maintained in a single compaction pass, and all three can be read at a single LSN.

This is the **only** claim the project needs to make. Everything else is engineering quality.

---

## 1. Corrections to the v2 proposal

Recorded so they are not re-litigated. Each has a corresponding ADR.

| v2 claim | Verdict | v3 replacement |
|---|---|---|
| eBPF kernel merge-sort worker | **Impossible.** The BPF verifier rejects unbounded loops, >512B stack, and arbitrary memory access. Compaction cannot run in eBPF. | `io_uring_cmd` NVMe passthrough + `sched_ext` BPF scheduler + eBPF for observability only |
| WAF below 1.2× | **Overstated.** vLog garbage collection performs writes of its own. | Target **1.5–2.0×**, measured from device SMART counters (vs. ~15–30× for RocksDB) |
| Globally lock-free skiplist with EBR | **Unjustified bug surface** for a small team. | **Thread-per-core, shared-nothing.** Each shard is single-threaded → a plain ART memtable, zero locks on the hot path |
| PGM-Index as a total B-tree replacement | **Fails on adversarial/clustered key distributions.** | **Build-time index policy selection** among PGM / Eytzinger / FST |
| HNSW graphs merged during compaction | **Expensive and recall-destroying.** | **IVF + RaBitQ**; merge = posting-list concatenation |
| "Zero bugs" | **Not achievable by review.** | **Deterministic Simulation Testing**, the only approach known to produce this outcome in practice |

---

## 2. Innovations

Ordered by strength of contribution.

### N1 — Unified tri-modal segment *(the thesis)*
Every SSTable is simultaneously a KV segment, an inverted-index segment, and an IVF vector segment. One compaction maintains all three.
**Deliverable:** hybrid queries (predicate + BM25 + ANN) served from a single snapshot at a single LSN. Zero cross-index skew, structurally.

### N2 — Build-time index policy selection
At compaction time the complete key distribution is known. The writer builds all three candidate indexes, measures each, and records the winner in the footer:

- **PGM-Index** — smallest for near-linear distributions
- **Eytzinger array** over block first-keys — hard worst-case guarantee
- **FST / front-coded trie** — string keys with shared prefixes

Learned-index benefits without learned-index risk. Worth a section in the whitepaper on its own.

### N3 — Tiered index quality by LSM level
Map LSM economics onto index quality:

| Level | Data character | Vector index | Text index |
|---|---|---|---|
| L0–L1 | Hot, volatile, small | SIMD brute force (recall = 1.0) | Plain postings |
| L2–L4 | Warm, medium | IVF + RaBitQ 1-bit | Postings + Block-Max metadata |
| L_max | Cold, stable, ~90% of bytes | IVF + DiskANN graph (full rebuild) | Postings + BMW + skip lists |

Low levels are small, so brute force is cheap. The top level is stable, so an expensive graph build amortizes. This mapping has not been published.

### N4 — Self-driving compaction
A count-min sketch with exponential decay tracks hotness per key range. Policy is chosen per range:

- Hot range → **Leveled** (low read amplification)
- Cold range → **Tiered** (low write amplification)

Theoretical basis: Monkey (SIGMOD'17), Dostoevsky (SIGMOD'18). The contribution is making it *online and per-range* rather than a global static choice.

### N5 — Monkey-optimal filter allocation
Instead of a uniform "10 bits/key everywhere," allocate filter bits across levels to minimize total false positives per byte of RAM. Uses **BuRR (Bumped Ribbon Retrieval)** rather than Bloom — roughly 30% more compact at equal FPR.

### N6 — Range filters
Bloom and Ribbon filters do not help range scans. **SuRF** does. For an engine claiming search capability, this is the difference between a toy and a serious system.

### N7 — Placement-hinted vLog
Each vLog chunk carries a lifetime hint derived from the LSM level of its keys:

- `fcntl(F_SET_RW_HINT)` on commodity hardware
- **NVMe Flexible Data Placement (FDP, TP4146)** on modern SSDs
- ZNS as an optional path

Hot and cold data land in separate NAND blocks, driving device-internal GC toward zero. **Measured directly from the drive's own SMART counters**, which makes the benchmark impossible to fake.

### N8 — Adaptive value routing + content-defined dedup
Replaces WiscKey's fixed threshold:

- `< 128 B` → inlined in the SSTable (no extra read)
- `128 B – 4 KB` → vLog
- `≥ 4 KB` → blob store with **FastCDC** content-defined chunking and XXH3 deduplication

Large savings on versioned documents — precisely the RAG workload — and nearly free, since the hashing pipeline already exists.

### N9 — Compaction-trained compression dictionaries
A ZSTD dictionary is trained per SSTable over a sample of its own blocks and stored inside the file. Typically **2–5× better ratio** on small records. Cheap, deterministic, measurable.

### N10 — Zero-copy branching and time travel
The manifest is an append-only log of versions, so:

- `read AS OF LSN 91422`
- `tekwe3 branch create eval-v2` → a new manifest root pointing at existing SSTables plus a fresh memtable

Cost: zero bytes. Killer application for AI workloads: **A/B two embedding strategies over the same dataset without copying it.** No vector database offers this.

### N11 — Realistic kernel fast path

| Technique | Kernel | Expected benefit |
|---|---|---|
| `IORING_SETUP_SINGLE_ISSUER` + `DEFER_TASKRUN` | 6.1+ | Meaningful p99 reduction |
| Registered buffers and files | 5.x | Removes per-submission lookup |
| **NVMe passthrough via `io_uring_cmd` on `/dev/ngXnY`** | 5.19+ | 10–15% latency reduction (bypasses block layer) |
| **`sched_ext` BPF scheduler** | 6.12+ | Guarantees compaction cannot starve foreground reads |
| eBPF observability on `blk_*` tracepoints | 5.x | Real block-layer latency histograms |
| XRP-style BPF request resubmission | Research | **Stretch goal, Phase 12, may be abandoned** |

### N12 — Optional WAL-less durability
If the device reports `AWUPF` ≥ block size (read via NVMe Identify Controller), the double write can be skipped. Automatic fallback when unsupported.

---

## 3. Execution model

Thread-per-core, shared-nothing.

```
┌────────────────────────────────────────────────────────────┐
│                       TEKWE3 Engine                        │
│                                                            │
│   Shard 0        Shard 1        Shard 2       Shard N-1    │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐   ┌─────────┐   │
│  │ Memtable│    │ Memtable│    │ Memtable│   │ Memtable│   │
│  │  (ART)  │    │  (ART)  │    │  (ART)  │   │  (ART)  │   │
│  ├─────────┤    ├─────────┤    ├─────────┤   ├─────────┤   │
│  │  WAL    │    │  WAL    │    │  WAL    │   │  WAL    │   │
│  ├─────────┤    ├─────────┤    ├─────────┤   ├─────────┤   │
│  │ LSM     │    │ LSM     │    │ LSM     │   │ LSM     │   │
│  │ L0..Ln  │    │ L0..Ln  │    │ L0..Ln  │   │ L0..Ln  │   │
│  ├─────────┤    ├─────────┤    ├─────────┤   ├─────────┤   │
│  │io_uring │    │io_uring │    │io_uring │   │io_uring │   │
│  └────┬────┘    └────┬────┘    └────┬────┘   └────┬────┘   │
│       │              │              │             │        │
│       └──── SPSC mailboxes (cross-shard only) ────┘        │
└────────────────────────────────────────────────────────────┘
                          │
                 ┌────────┴─────────┐
                 │  vLog (per-shard)│
                 │  Manifest log    │  single writer
                 └──────────────────┘
```

**Hard rules**

1. A shard exclusively owns its data. No other thread touches it.
2. Sharding is **range-based** (not hash) to preserve ordered scans; ranges split and merge dynamically under load.
3. Cross-shard communication goes through SPSC ring buffers. No shared mutexes.
4. CPU pinning; optional `SCHED_FIFO` for foreground threads.
5. The only lock-free data structures in the system are the cross-shard mailboxes. Everything else is single-threaded by construction.

---

## 4. Data paths

### 4.1 Write path

```
Write(key, value, document?, vector?)
  │
  ├─ 1. Route to shard (range lookup, read-mostly map)
  ├─ 2. Assign LSN (per-shard counter + global epoch)
  ├─ 3. Adaptive value routing
  │       < 128 B  → inline
  │       < 4 KB   → vLog append (chunk-aligned, lifetime-hinted)
  │       ≥ 4 KB   → FastCDC chunks → XXH3 dedup → blob store
  ├─ 4. Append to WAL group-commit batch
  │       io_uring WRITE_FIXED, O_DIRECT, one fsync/FUA per batch
  ├─ 5. Insert into ART memtable:
  │       key → { lsn, value_ptr, doc_ptr?, vec_ptr? }
  └─ 6. Acknowledge (durable), or early-ack in relaxed mode
```

### 4.2 Point read path

```
Get(key, snapshot_lsn)
  │
  ├─ 1. Active memtable, then frozen memtables (MVCC-visible scan)
  ├─ 2. For each level, in order:
  │       a. Key-range check from manifest      ← cheapest
  │       b. BuRR filter probe (SIMD, interleaved)  ← RAM
  │       c. Key index lookup per footer policy:
  │            PGM (error ≤ ε) | Eytzinger | FST
  │       d. Async block read via io_uring
  │            (NVMe passthrough when enabled)
  ├─ 3. Block cache (sharded, CLOCK-Pro)
  ├─ 4. ZSTD decompress with the file's embedded dictionary
  └─ 5. Resolve value pointer → vLog chunk cache or blob store
```

### 4.3 Hybrid query path *(the thesis in operation)*

```
HybridQuery { predicate, text, vector, k } @ snapshot_lsn
  │
  ├─ Pin one snapshot LSN for all three paths   ◄── consistency guarantee
  │
  ├─ [A] Predicate → Roaring bitmap (SIMD AND/OR)
  │
  ├─ [B] Text: Block-Max WAND
  │        Per-SSTable score upper bounds allow skipping entire files,
  │        the top-k analogue of a Bloom filter skip.
  │
  ├─ [C] Vector: IVF probe → RaBitQ 1-bit Hamming scan (SIMD popcount)
  │        → top-4k candidates → rerank with full-precision vectors
  │
  ├─ [D] Fusion: Reciprocal Rank Fusion or tuned linear weighting
  │
  └─ [E] Final rerank → top-k
```

---

## 4.4 Resource contention — the multi-modal tax

Three problems exist only because one segment serves three modalities. A conventional LSM has none of them. Each belongs in a spec file, and each needs an ADR before the phase that implements it.

### 4.4.1 Compaction here is CPU-bound, not I/O-bound

In a conventional LSM, compaction is limited by disk bandwidth. In TEKWE3 a single compaction must additionally merge and re-encode posting lists (SIMD-BP128), recompute BM25 block-max bounds and term statistics, quantize vectors (RaBitQ), and train a ZSTD dictionary. **The bottleneck inverts to CPU.**

Thread-per-core sharpens this: if compaction runs on the shard's own core, a vector-quantization burst directly stalls that shard's foreground reads. The scheduler must budget CPU cycles, not only I/O.

**Decision required — ADR, phase P7:**

| Option | Mechanism | Cost |
|---|---|---|
| **A — on-shard with a cycle budget** | Compaction runs on the owning core, yielding after a bounded quantum | Preserves shared-nothing. Foreground latency bounded by quantum size. |
| **B — dedicated compaction cores** | Separate pool; data handed over by ownership transfer | Protects foreground latency, but breaks strict shared-nothing and adds cross-core traffic |

**Recommendation: A with an explicit measured budget.** Escalate to B only if p99.9 under compaction fails the P7 target.

**Rules**
- The compaction scheduler budgets **CPU quanta and I/O together**, never I/O alone.
- Vector quantization and dictionary training are the two most expensive stages. Both are interruptible and yield at block boundaries.
- P12's `sched_ext` scheduler enforces isolation at kernel level, but the userspace budget must work correctly without it. Kernel help is an optimization, never a prerequisite.

### 4.4.2 The block cache is type-aware, not uniform

One cache serving three access patterns will thrash. A sequential scan of a long posting list would evict the IVF centroids that every vector query needs.

**Rules**
- The cache is **segmented by content type**: KV data blocks, postings, vector centroids and codes, filters, key indexes.
- Each segment has a floor and a ceiling as a share of total cache. Centroids and filters get a **pinned floor** — they are small, hot, and expensive to miss.
- Scan-heavy access (posting-list traversal, brute-force vector scan at L0) is admitted with **scan resistance**: single-touch blocks do not promote.
- CLOCK-Pro operates **within** a segment, never across segments.

### 4.4.3 Codebook versioning and vector drift

An IVF codebook trained on early data degrades as the vector distribution shifts. But retraining invalidates every RaBitQ code written under the previous codebook — this is the hard part, and it is why the codebook is versioned rather than mutated.

**Rules**
- Every SSTable footer names the **codebook version** it was written against. Codebooks live in the manifest, are immutable, and are only superseded.
- A query may therefore span several codebook versions. The planner probes per version and merges candidates before rerank. Because rerank uses full-precision vectors from the vLog, recall survives crossing versions — **this is why two-stage retrieval is mandatory here, not an optimization.**
- Retraining is triggered by **measured drift, never by a timer**: track centroid assignment imbalance and mean residual per level. When either crosses threshold, schedule a retrain.
- A retrain is a compaction that **rewrites vector sections only**, at the highest level, where data is coldest and the rewrite amortizes. It never blocks writes.
- An old codebook is retained until no live SSTable references it.

---

## 5. On-disk format — SSTable v3

```
┌────────────────────────────────────────────────────────────┐
│ Magic  "TEKWE303"                                  [8 B]   │
│ Format version, flags, shard_id, level            [24 B]   │
├────────────────────────────────────────────────────────────┤
│ ZSTD dictionary (trained on this file's own blocks)        │
├────────────────────────────────────────────────────────────┤
│ Data blocks (4 KB aligned, prefix-coded, restart intervals)│
│   Block 0: [entries][restart offsets][crc32c]              │
│   ...                                                      │
│   Block N                                                  │
├────────────────────────────────────────────────────────────┤
│ Inverted index section                                     │
│   ├─ Term dictionary (FST)                                 │
│   ├─ Postings (SIMD-BP128 delta + bitpacking)              │
│   ├─ Block-Max metadata (per-block score upper bounds)     │
│   └─ Term statistics (df, ttf) for global IDF              │
├────────────────────────────────────────────────────────────┤
│ Vector index section                                       │
│   ├─ IVF centroid assignments                              │
│   ├─ RaBitQ codes (1-bit, SIMD-aligned)                    │
│   ├─ Optional DiskANN graph (L_max only)                   │
│   └─ Codebook version pointer (shared, in manifest)        │
├────────────────────────────────────────────────────────────┤
│ Filter section                                             │
│   ├─ BuRR ribbon filter (interleaved, Monkey-allocated)    │
│   └─ SuRF range filter (optional, per-level policy)        │
├────────────────────────────────────────────────────────────┤
│ Key index section                                          │
│   └─ EXACTLY ONE OF: PGM params | Eytzinger array | FST    │
│      (policy_id in the footer selects which)               │
├────────────────────────────────────────────────────────────┤
│ Footer  [fixed 96 B, at end of file]                       │
│   section offsets (8 × u64) | policy_id | codec ids        │
│   min_lsn | max_lsn | entry_count | xxh3_128(footer)       │
│   Magic (repeated) ← detects truncated files               │
└────────────────────────────────────────────────────────────┘
```

**Format invariants — never violate**

1. Every section carries an independent checksum. Corruption in the vector section must not break KV reads.
2. The footer sits at the end with a repeated magic, so truncation is detected immediately.
3. Files are immutable. Never written in place after finalization.
4. Every section is optional. A pure-KV SSTable must be valid.
5. Any format change increments the version and adds a backward-compatible read path.
6. All multi-byte integers are little-endian. All offsets are absolute from file start.

---

## 6. Correctness architecture

The goal of "no bugs" is achieved by construction, not by review.

### 6.1 Deterministic Simulation Testing — built first, not last

The entire engine runs against a swappable `Runtime` trait:

- **Simulated clock** — no wall-clock time exists in engine code
- **Simulated disk** — read/write/fsync/fallocate with fault injection: torn writes, reordering, misdirected writes, bit rot, `ENOSPC`, `fsync` failure, unbounded latency
- **Deterministic scheduler** — single-threaded, thread interleaving chosen from the seed
- **Simulated network** — for a future distributed phase

Every run reproduces exactly under `TEKWE3_SEED=<n>`. Bugs stop being "sometimes in production" and become "seed 8842, operation 1937."

If the simulator is built last, the engine must be rewritten to become deterministic. FoundationDB and TigerBeetle both built it first. So do we.

### 6.2 Defense layers

| Layer | Tool | What it catches |
|---|---|---|
| Model-based | Differential vs. `BTreeMap` reference | Any semantic divergence |
| Property-based | `proptest` | `parse(serialize(x)) == x` for every on-disk struct |
| Fuzzing | `cargo-fuzz` on every parser | Panic or UB on arbitrary bytes |
| UB detection | **Miri** over all `unsafe` | Undefined behavior |
| Concurrency | **Loom** on mailboxes | Memory-ordering bugs |
| Bounded proof | **Kani** on core invariants | Verified within bounds |
| Formal spec | **TLA+** for WAL, manifest, vLog GC | Design flaws before code exists |
| Real crash | `dm-log-writes` + CrashMonkey | Recovery from any interruption point |
| Transactional | **Elle** (Jepsen) | Isolation anomalies |
| Runtime | ASAN / TSAN / UBSAN in CI | Residual issues |

### 6.3 Code invariants

- `#![forbid(unsafe_op_in_unsafe_fn)]` in every crate
- `unsafe` is permitted **only** in `tekwe3-sys`; every block carries a `// SAFETY:` justification
- `unwrap()`, `expect()`, and `panic!` are **forbidden** in library code; allowed only in tests and binary entry points
- `clippy -D warnings` is a merge gate
- No `async` in the core engine; the thread-per-core runtime is explicit

---

## 7. Workspace layout

**Language: Rust.** Miri, Loom, Kani, proptest, and cargo-fuzz do not coexist in any other ecosystem. For a "no bugs" objective this is decisive. C++ has no equivalent toolchain; Zig's verification ecosystem is not yet mature.

```
tekwe3/
├─ Cargo.toml                # workspace, pinned MSRV, shared lints
├─ AGENTS.md                 # operating manual for AI contributors
├─ STATE.md                  # current position (updated every session)
├─ BACKLOG.md                # deferred items
├─ CHANGELOG.md              # Keep a Changelog format
├─ justfile                  # `just gate` runs the full verification gate
├─ crates/
│  ├─ tekwe3-sys/            # ⚠️ the only crate permitted to use unsafe
│  │                         # io_uring, NVMe passthrough, fallocate,
│  │                         # RW_HINT, FDP, AWUPF capability probes
│  ├─ tekwe3-core/           # Key, Lsn, Error, Config, Metrics
│  ├─ tekwe3-codec/          # varint, SIMD-BP128, delta, ZSTD, crc32c, xxh3
│  ├─ tekwe3-simd/           # SIMD kernels + runtime dispatch
│  ├─ tekwe3-memtable/       # ART with SIMD node scanning
│  ├─ tekwe3-wal/            # group commit, parallel recovery
│  ├─ tekwe3-sstable/        # writer, reader, footer, block cache
│  ├─ tekwe3-filter/         # BuRR + SuRF + Monkey allocator
│  ├─ tekwe3-index/          # PGM + Eytzinger + FST + PolicySelector
│  ├─ tekwe3-vlog/           # value log, GC, hole punching, FastCDC dedup
│  ├─ tekwe3-text/           # tokenizer, postings, BM25, Block-Max WAND
│  ├─ tekwe3-vector/         # IVF, RaBitQ, rerank, DiskANN
│  ├─ tekwe3-compaction/     # policies, scheduler, cost model, hotness
│  ├─ tekwe3-mvcc/           # snapshots, transactions, branching
│  ├─ tekwe3-query/          # hybrid planner, RRF fusion
│  ├─ tekwe3-engine/         # shard runtime, public API
│  ├─ tekwe3-sim/            # 🔬 deterministic simulator + fault injection
│  ├─ tekwe3-bench/          # YCSB, BEIR, ann-benchmarks, SMART-based WAF
│  ├─ tekwe3-cli/            # admin CLI + telemetry TUI
│  └─ tekwe3-server/         # optional gRPC/HTTP
├─ fuzz/                     # cargo-fuzz targets
├─ tla/                      # TLA+ specifications
├─ docs/
│  ├─ ARCHITECTURE.md        # this document
│  ├─ ROADMAP.md
│  ├─ format/                # byte-level format specs
│  ├─ adr/                   # architecture decision records
│  ├─ tasks/                 # task tickets
│  ├─ worklog/               # per-session engineering journal
│  └─ benchmarks/            # results + reproduction scripts
└─ .github/workflows/        # kernel matrix: 6.1, 6.6, 6.12, latest
```

---

## 8. Evaluation

Benchmarks must be impossible to fake and reproducible by a stranger.

| Axis | Workload | Baseline | Metric |
|---|---|---|---|
| KV throughput | YCSB A–F | RocksDB, Badger, Sled | ops/s, p50/p99/p99.9/p99.99 |
| Write amplification | YCSB-A, 24 h | RocksDB | **Device SMART counters**, not our own logs |
| Space amplification | Fixed dataset | RocksDB | on-disk bytes / logical bytes |
| Text relevance | BEIR, ≥ 3 datasets | Lucene / Tantivy | nDCG@10, target ±1% |
| Vector | ann-benchmarks (SIFT, GIST, DEEP) | FAISS, HNSWlib, Qdrant | recall@10 vs. QPS |
| Hybrid | Custom benchmark | Elasticsearch + Qdrant | latency **and skew count** |
| Consistency | Elle | — | zero anomalies |
| Crash safety | dm-log-writes, 10k interruption points | — | always prefix-consistent recovery |

### 8.1 The skew test — the project's signature result

Under sustained write load, issue hybrid queries against both TEKWE3 and an Elasticsearch + Qdrant stack. Count responses where the KV, text, and vector views disagree.

The competing stack will produce a nonzero count. TEKWE3 will produce zero, structurally.

**That single number is the headline of the whitepaper, the README, and the launch post.**

---

## 9. Scope realism

The complete vision is 18–36 months of one person's work, even with AI assistance. Define three exit points:

| Version | Phases | Estimate | Value |
|---|---|---|---|
| **Defensible MVP** | P0–P8 | 4–6 months | A complete LSM engine with deterministic simulation testing. Already above 95% of GitHub systems projects. |
| **Full thesis** | P0–P11 | 9–14 months | Transactional hybrid search. The actual differentiator. |
| **Showcase** | P0–P14 | 18–36 months | Kernel fast path, branching, tiered storage |

**Recommendation:** commit to P8 and ship it. An LSM engine with a deterministic simulator and SMART-measured write amplification — with zero vector or text features — is already a career-making artifact. Add P9–P11 as v2.
