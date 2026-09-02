# TEKWE3 v3 — Roadmap

**Golden rule:** A phase is not complete until every Exit Criterion is proven by pasted output from an actual command. Advancing without a green gate compounds technical debt and is the primary way "no bugs" fails.

**Legend:** 🔴 critical · 🟡 important · 🟢 optional / future · 🔬 research, may be abandoned

---

## Overview

| Phase | Title | Output | Estimate |
|---|---|---|---|
| **P0** 🔴 | Foundation & determinism harness | Simulator before any engine code | 2–3 weeks |
| **P1** 🔴 | Codec, SIMD, checksums | Fuzz-clean encoding layer | 2 weeks |
| **P2** 🔴 | Memtable (ART) + MVCC keys | Single-threaded memtable equivalent to BTreeMap | 3 weeks |
| **P3** 🔴 | WAL + parallel recovery | Proven crash resilience | 3–4 weeks |
| **P4** 🔴 | SSTable v3 + block cache | Frozen, documented on-disk format | 4 weeks |
| **P5** 🔴 | Filters + index policy selector | BuRR + PGM/Eytzinger/FST | 3–4 weeks |
| **P6** 🔴 | vLog + KV separation + GC | Measured WAF < 2.0× | 4 weeks |
| **P7** 🔴 | Compaction + self-driving policy | Complete, stable LSM | 5–6 weeks |
| **P8** 🔴 | MVCC, transactions, snapshots | **◄ Shippable MVP** | 4 weeks |
| **P9** 🟡 | Text index (BM25 + Block-Max WAND) | nDCG@10 within ±1% of Lucene | 6 weeks |
| **P10** 🟡 | Vector index (IVF + RaBitQ) | recall/QPS competitive with FAISS | 6 weeks |
| **P11** 🟡 | Hybrid query engine | **◄ Thesis realized** | 4 weeks |
| **P12** 🟢 | Kernel fast path | Passthrough + sched_ext | 4 weeks |
| **P13** 🟢 | Branching + time travel | Showcase feature | 3 weeks |
| **P14** 🟢 | Launch: whitepaper, benchmarks, README | Release package | 3 weeks |

---

## P0 — Foundation & determinism harness 🔴

**Why first:** if the simulator is built last, the engine must be rewritten to become deterministic. This phase is non-negotiable and cannot be reordered.

### Work items
- Workspace with pinned MSRV, shared lints, `#![forbid(unsafe_op_in_unsafe_fn)]`
- `tekwe3-core`: `Key`, `Lsn`, `Error` (thiserror, no `Box<dyn Error>`), validated `Config`
- **`tekwe3-sim`**: a `Runtime` trait through which the engine reaches the outside world
  - `Clock` — simulated time
  - `Disk` — read/write/fsync/fallocate with fault injection
  - `Scheduler` — deterministic thread interleaving from seed
  - Two implementations: `RealRuntime` and `SimRuntime`
- Fault injection catalogue: torn write, write reordering, misdirected write, bit flip, `ENOSPC`, `fsync` failure, unbounded latency, partial read, spurious `EINTR`
- `tekwe3-sys` skeleton: capability probes (io_uring opcode support, `AWUPF`, FDP / `RW_HINT` availability)
- `justfile` with the `gate` and `doc-check` recipes
- CI: fmt, clippy, test, kernel matrix 6.1 / 6.6 / 6.12 / latest

### Documentation system — as important as the code
Because this project spans hundreds of agent sessions and several agent generations, the documentation system is built in P0, not later. See `docs/CONTEXT_SYSTEM.md`.

- `AGENTS.md`, `STATE.md`, `BACKLOG.md`, `CHANGELOG.md`
- **`docs/INDEX.md`** — the router every session reads first
- **`docs/INVARIANTS.md`** — numbered rules I-01…I-nn, referenced by every task
- **`docs/REJECTED.md`** — dead approaches, so they are never re-proposed
- **`docs/GLOSSARY.md`** — project vocabulary
- **P0-011: split `ARCHITECTURE.md` into `docs/spec/` (19 files, ≤400 lines each)**
- `crates/*/CONTEXT.md` for every crate
- Templates: ADR, task, worklog, PR
- **`just doc-check`** enforcing size caps, index completeness, ADR index sync

### Legal and licensing foundation — settled in P0, never retrofitted
Licensing decided at the end of a project is a licensing crisis. Decide now, enforce automatically.

- **`LICENSE-APACHE` + `LICENSE-MIT`** — dual Apache-2.0 / MIT, the Rust ecosystem norm. Apache-2.0 carries an express patent grant, which matters for a system implementing published algorithms. Record the choice in `docs/adr/0002-licensing.md` with the rejected alternatives (AGPL, SSPL, BSL) and why.
- **`docs/LICENSING.md`** — the project's licensing policy in one page: our license, what we accept from dependencies, what is forbidden.
- **SPDX headers** on every source file: `// SPDX-License-Identifier: Apache-2.0 OR MIT`
- **`cargo-deny` in the gate** — `deny.toml` with an allowlist of acceptable dependency licenses. **Copyleft dependencies (GPL, AGPL, SSPL) are rejected automatically.** This runs on every commit, not at launch.
- **`NOTICE`** — third-party attributions required by upstream licenses
- **`docs/CITATIONS.md`** — the design implements many published algorithms. Algorithms are not copyrightable; **reference implementations are.** For each of PGM-Index, BuRR, RaBitQ, SuRF, FastCDC, WiscKey, Monkey, Block-Max WAND, SIMD-BP128, ART, record: the paper, the reference implementation, its license, and **whether we read its code or implemented from the paper alone.** Implementing from the paper is strongly preferred and must be the default.
- **`SECURITY.md`** — vulnerability disclosure policy and contact. A storage engine holding user data needs one before it is public.
- **`CONTRIBUTING.md`** with a DCO sign-off requirement (`git commit -s`)
- **Name availability check** — `tekwe3` on crates.io, GitHub org, npm, and a trademark search. Record the result. Renaming after launch is expensive.
- **`docs/COMPATIBILITY.md`** — the on-disk format compatibility promise, which is a stronger commitment than semver: *which format versions will always remain readable, and what a version bump obliges us to do.*
- **`docs/DEPENDENCY_POLICY.md`** — when a dependency may be added: license, maintenance, MSRV, transitive weight, and whether writing it ourselves is cheaper than auditing it.

### Exit criteria
```
[ ] TEKWE3_SEED=42 cargo test -p tekwe3-sim, run twice → identical trace hash
[ ] cargo clippy --workspace --all-targets -- -D warnings → zero warnings
[ ] Each of the 9 fault types has a test proving it can fire and is observed
[ ] tekwe3-sys capability probe runs on real hardware and prints results
[ ] `just gate` exists and passes on a clean checkout
[ ] docs/adr/0001-deterministic-simulation-first.md written
[ ] `just doc-check` green: all size caps respected, every doc indexed
[ ] ARCHITECTURE.md split into docs/spec/, reduced to an 80-line index
[ ] ONBOARDING TEST: a fresh agent given only INDEX.md + STATE.md +
    INVARIANTS.md + one spec file can correctly state the thesis, the
    current phase, and the applicable invariants — under 8k tokens read
[ ] LICENSE-APACHE and LICENSE-MIT present; SPDX header on every file
[ ] cargo-deny green and wired into `just gate`; deny.toml rejects copyleft
[ ] docs/CITATIONS.md lists every algorithm with paper, reference impl,
    its license, and whether we read that code
[ ] SECURITY.md, CONTRIBUTING.md (DCO), NOTICE present
[ ] Name availability verified and recorded in docs/adr/0003
[ ] docs/COMPATIBILITY.md states the on-disk format promise
```

---

## P1 — Codec, SIMD, checksums 🔴

### Work items
- Varint (LEB128 + zigzag), delta encoding, **SIMD-BP128** bitpacking
- `tekwe3-simd`: runtime dispatch across scalar / SSE4.2 / AVX2 / AVX-512 / NEON. **Every SIMD kernel has a scalar reference implementation.**
- crc32c (hardware-accelerated) for the hot path; XXH3-128 for large blocks
- ZSTD / LZ4 wrappers with dictionary support

### Exit criteria
```
[ ] Property test: every codec round-trips 100k random inputs
[ ] Differential test: every SIMD kernel == scalar reference on 1M random vectors
[ ] cargo fuzz on every decoder, 4 h, zero crashes
[ ] cargo miri test -p tekwe3-simd → clean
[ ] Benchmark table recorded in docs/benchmarks/p1-codec.md
```

---

## P2 — Memtable (ART) + MVCC keys 🔴

### Work items
- **Adaptive Radix Tree** (Node4 / Node16 / Node48 / Node256) with path compression and SIMD scanning on Node16
- MVCC key encoding: `user_key || !lsn` big-endian, so the newest version sorts first
- Frozen memtable via pointer swap, no locks
- Snapshot-consistent iterator

### Exit criteria
```
[ ] Differential test vs BTreeMap: 100M random operations, identical results
[ ] MVCC test: each snapshot observes exactly the correct version
[ ] cargo miri test -p tekwe3-memtable → clean
[ ] Benchmark: > 5M inserts/s single core, 16-byte keys
[ ] Memory overhead per entry measured and recorded
```

---

## P3 — WAL + parallel recovery 🔴

### Work items
- Per-shard WAL → zero cross-core contention
- Group commit: batching with one `fsync`/FUA per batch
- io_uring `WRITE_FIXED` + registered buffers + `O_DIRECT`
- Record format `[len][crc32c][lsn][payload]` with trailing partial-record detection
- Parallel recovery: each shard recovers independently
- **TLA+ specification of the WAL and recovery protocol, written before implementation**
- 🟢 WAL-less path when `AWUPF` is sufficient, behind a feature flag

### Exit criteria
```
[ ] TLA+ model checker on the spec: zero invariant violations
[ ] DST: interruption at every byte offset of the WAL → recovery always
    yields a prefix-consistent state
[ ] dm-log-writes: 10,000 real interruption points → zero acknowledged
    writes lost
[ ] Recovering 1 GB of WAL across N cores takes ≈ single-core time / N
[ ] Differential test against the model: no acknowledged record is ever lost
[ ] docs/adr/ entry for the group-commit design
```

---

## P4 — SSTable v3 + block cache 🔴

### Work items
- Complete writer and reader per `docs/format/sstable-v3.md`
- 4 KB aligned blocks, prefix compression with restart intervals
- Footer with repeated magic and per-section checksums
- **ZSTD dictionary trained on the file's own block sample** (N9)
- Block cache: sharded, CLOCK-Pro eviction, no global lock
- Async reads via io_uring; NVMe passthrough behind a feature flag

### Exit criteria
```
[ ] docs/format/sstable-v3.md written and verified byte-for-byte against
    the implementation by a conformance test
[ ] cargo fuzz on the parser, 24 h, zero panics or UB on arbitrary bytes
[ ] Property: write(entries) → read() == entries, over 10k random files
[ ] File truncated at any offset → clean error, never a panic
[ ] Corrupting any one section leaves all other sections readable
[ ] Dictionary compression ratio improvement measured and recorded
[ ] Block cache is SEGMENTED BY TYPE (spec §4.4.2): a full posting-list
    scan must not evict pinned centroids or filters — prove it with a
    hit-rate test per segment under a mixed workload
```

---

## P5 — Filters + index policy selector 🔴

### Work items
- **BuRR** with interleaved layout for SIMD probing
- **Monkey allocator**: optimal filter bits per level
- **SuRF** range filter, optional per level
- **PolicySelector** — the N2 innovation:
  1. Build a PGM index; measure size and maximum error
  2. Build an Eytzinger array over block first-keys; measure size
  3. Build an FST; measure size
  4. Select the smallest candidate meeting the latency budget; record `policy_id` in the footer

### Exit criteria
```
[ ] Measured FPR within 1% of theoretical, across 5 key distributions
[ ] Monkey allocation reduces total FPR vs uniform allocation — numeric proof
[ ] PolicySelector tested on 4 adversarial distributions:
      sequential | zipfian | clustered | adversarial-gaps
    → never worse than baseline binary search on any of them
[ ] Index RAM footprint vs a B-tree baseline recorded (a real number,
    replacing the unverified "95% reduction" claim)
[ ] Fuzz on every index parser → clean
```

---

## P6 — vLog + KV separation + GC 🔴

### Work items
- Append-only, chunk-aligned, per-shard vLog
- **Adaptive value routing** (N8): < 128 B inline, < 4 KB vLog, ≥ 4 KB blob
- **FastCDC + XXH3 dedup** for blobs
- GC via `FALLOC_FL_PUNCH_HOLE` with liveness counted from the manifest
- **Placement hints** (N7): `F_SET_RW_HINT`, plus NVMe FDP when available
- 🟢 ZNS path

### Exit criteria
```
[ ] WAF measured from device SMART counters (data_units_written), NOT from
    our own logs. YCSB-A, 24 h. Target < 2.0×
[ ] WAF with placement hints vs without — delta recorded
[ ] DST: crash during GC → no live value is ever lost, across all seeds
[ ] Dedup ratio on a versioned-document dataset recorded
[ ] Space amplification after 10 GC cycles measured
[ ] docs/adr/ entry justifying the routing thresholds
```

---

## P7 — Compaction + self-driving policy 🔴

### Work items
- Both leveled and tiered compaction
- **Hotness tracking**: count-min sketch with exponential decay per key range
- **Per-range policy selection** (N4): hot → leveled, cold → tiered
- Backpressure and admission control — writers must never be able to swamp the engine
- Compaction scheduler with an I/O budget and priorities

### Exit criteria
```
[ ] 72 h continuous mixed YCSB run with no stalls and no unbounded growth
[ ] Pareto plot (WAF vs read amplification) against RocksDB on identical
    hardware and identical axes
[ ] Self-driving: under a Zipfian workload, WAF drops measurably vs fixed
    leveled while p99 read latency does not regress — record the real number
[ ] DST: crash at every compaction stage → manifest always consistent
[ ] p99.9 read latency under heavy compaction measured (baseline for P12)
[ ] ADR on compaction CPU budget (spec §4.4.1, option A vs B) — decided
    with a measurement, not a preference
[ ] Compaction scheduler budgets CPU QUANTA as well as I/O; prove that a
    saturated compaction cannot starve foreground reads on the same core
    WITHOUT sched_ext (kernel help is P12, not a prerequisite)
```

---

## P8 — MVCC, transactions, snapshots 🔴 ◄ MVP

### Work items
- Snapshot isolation on a global LSN
- Read-write transactions with optimistic conflict detection
- Manifest as an append-only version log
- Stable public API with documentation
- CLI with telemetry TUI

### Exit criteria
```
[ ] Elle (Jepsen) over 1M transactions: zero snapshot-isolation anomalies
[ ] Public API documented, semver locked, runnable examples as doc-tests
[ ] README complete: architecture diagram, real benchmarks, install steps
[ ] Full test suite clean under ASAN / TSAN / UBSAN
[ ] Tagged v0.8.0-P8
[ ] 🚀 Publishable. Stopping here is a legitimate and respectable outcome.
```

---

## P9 — Text index (BM25 + Block-Max WAND) 🟡

### Work items
- Unicode-aware, extensible tokenizer with Persian/Arabic normalization
- Postings with SIMD-BP128; term dictionary as an FST
- BM25 using global term statistics from the manifest
- **Block-Max WAND across LSM levels** (N1) — skip entire SSTables for top-k
- Posting-list merging during compaction

### Exit criteria
```
[ ] nDCG@10 on ≥ 3 BEIR datasets within ±1% of Lucene/Tantivy
[ ] Global IDF proven correct after compaction (differential test)
[ ] BMW pruning: percentage of SSTables skipped, recorded
[ ] Top-10 latency over 10M documents measured
```

---

## P10 — Vector index (IVF + RaBitQ) 🟡

### Work items
- IVF partitioning with a versioned global codebook in the manifest
- **RaBitQ** 1-bit quantization with SIMD popcount Hamming scanning
- Two-stage rerank using full-precision vectors from the vLog
- **Tiered index quality** (N3): L0–L1 brute force, mid IVF, L_max DiskANN
- IVF merging during compaction = posting-list concatenation

### Exit criteria
```
[ ] recall@10 vs QPS curves on SIFT1M, GIST1M, DEEP-1M against FAISS-IVF
    and HNSWlib on identical hardware
[ ] Merging two SSTables does not reduce recall (before/after differential)
[ ] Codebook version recorded in every SSTable footer; a query spanning
    TWO codebook versions returns correct results (spec §4.4.3)
[ ] Drift detection implemented: centroid imbalance + mean residual,
    threshold-triggered, never timer-triggered
[ ] Vector-section-only retrain compaction implemented and proven not to
    block writes
[ ] Codebook drift: measure recall after 10× data growth and after a
    deliberate distribution shift (swap embedding model mid-dataset)
[ ] Memory per vector recorded vs FAISS
```

---

## P11 — Hybrid query engine 🟡 ◄ thesis

### Work items
- Planner fusing predicate → BM25 → ANN in a single pass
- Roaring bitmaps with SIMD set operations for predicates
- Reciprocal Rank Fusion plus tunable linear weighting
- **All three paths read from one `snapshot_lsn`** — structural guarantee

### Exit criteria
```
[ ] 🎯 SKEW TEST: under 50k writes/s, issue 1M hybrid queries
      → zero inconsistencies between KV, text, and vector views
      → run the identical test against Elasticsearch + Qdrant and count
        their inconsistencies
    This number is the project's headline result.
[ ] Hybrid p99 latency vs Elasticsearch + Qdrant on the same dataset
[ ] Fusion quality (nDCG) on a hybrid benchmark
[ ] Tagged v0.11.0-P11
```

---

## P12 — Kernel fast path 🟢

### Work items
- `IORING_SETUP_SINGLE_ISSUER` + `DEFER_TASKRUN` (6.1+)
- **NVMe passthrough** via `io_uring_cmd` on `/dev/ngXnY` (5.19+)
- **`sched_ext` BPF scheduler** (6.12+) isolating compaction from foreground reads
- eBPF observability: block-layer latency histograms
- 🔬 XRP-style BPF resubmission — research, may fail, that is acceptable

### Exit criteria
```
[ ] Every optimization behind a feature flag with automatic fallback
[ ] Measured delta recorded separately for each technique
[ ] sched_ext: p99.9 read latency under compaction vs the P7 baseline
    ← this is the tweetable number
[ ] Works, or cleanly disables itself, on all four CI kernels
[ ] If XRP fails, docs/adr/ records why — a negative result is still a result
```

---

## P13 — Branching + time travel 🟢

### Exit criteria
```
[ ] `tekwe3 branch create X` adds zero bytes on disk (proven with du)
[ ] Reads AS OF LSN correct across 100 historical versions
[ ] Version GC never breaks a live branch
[ ] Demo: two embedding strategies evaluated on one dataset, no copy
```

---

## P14 — Launch 🟢

### Work items
- `docs/whitepaper.pdf` — the spec, cleaned up, with experimental results
- README with SVG diagrams, live benchmark table, latency percentile plots
- Benchmark site with reproducible results (scripts plus hardware manifest)
- ADRs, especially for **rejected** decisions (eBPF merge-sort, HNSW merging)
- Engineering blog post: "Why I built the deterministic simulator first"

### Legal review before going public
- **SBOM** generated (`cargo-sbom` / CycloneDX) and published with the release
- **`cargo-about`** generates the full third-party license page; verify it matches `NOTICE`
- **Benchmark publication rights** — verify every system you benchmark against permits published comparisons. Open-source engines (RocksDB, Badger, Sled, Lucene, FAISS, Qdrant, PostgreSQL) do. **Commercial databases frequently carry DeWitt clauses forbidding published benchmarks** — if any commercial system is in the comparison table, confirm its EULA allows it or remove it.
- **Dataset licenses** — BEIR component datasets, SIFT/GIST/DEEP, and YCSB each carry their own terms, some non-commercial. Verify redistribution and publication rights for every dataset whose results appear in the README. Record in `docs/benchmarks/DATASETS.md`.
- **Trademark** — final check that no conflicting mark exists in the database or search-engine space before the name is publicized
- **Contributor terms** — DCO enforcement live in CI before the repo accepts outside PRs
- **`docs/CITATIONS.md` final pass** — every implemented algorithm credited in both the whitepaper and the README. Academic credit is not optional, and reviewers notice its absence.
- **Export control** — no cryptography is shipped (checksums only). If encryption is ever added, revisit before release.

### Exit criteria
```
[ ] Every number in the README is reproducible by a single script
[ ] A stranger can build and benchmark in under 10 minutes from the README
[ ] An ADR exists for every major decision, including rejected alternatives
[ ] cargo-deny and cargo-about green; SBOM published with the release
[ ] Every benchmarked system's license permits published comparison
[ ] Every dataset used in published results has documented rights
[ ] docs/CITATIONS.md complete; every algorithm credited in the whitepaper
```

---

## Risk register

| Risk | Likelihood | Contingency |
|---|---|---|
| Codebook drift degrades vector recall | High | Versioned codebooks + drift-triggered vector-only retrain (spec §4.4.3) |
| Compaction saturates CPU and stalls foreground reads | **High** | CPU-quanta budget, interruptible quantization (spec §4.4.1). This is the resource model inverting vs a normal LSM — do not assume I/O-bound. |
| Cache thrashing between the three modalities | Medium | Type-segmented cache with pinned floors (spec §4.4.2) |
| PGM does not win on realistic keys | Medium | PolicySelector already covers this ✅ |
| XRP does not work on upstream kernels | High | Drop it; write the ADR explaining why |
| WAF does not reach below 2× | Medium | Publish the real number. Honesty outranks the claim. |
| `sched_ext` unavailable on target kernel | Medium | cgroup v2 I/O weights as fallback |
| **Scope expands beyond capacity** | **High** | Stop at P8 and ship. Seriously. |
