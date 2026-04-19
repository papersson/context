# Map-Only Workloads at Scale

*Data validation as the canonical case.*

This document is a reference for a specific shape of computation: evaluating a bounded set of per-row rules over a large columnar corpus, with no reduce phase and no cross-row dependency. The motivating example throughout is **data validation** — take a batch of input records, evaluate a set of declarative rules against each row, and emit per-row annotations (valid / invalid / which rules failed) for downstream consumers. The same architectural story applies to any map-only per-row transform with sparse or compact output.

For the broader 3×3 matrix (three architectures × three workload shapes) see `docs/project.md`. For what *streaming* means at different levels of the stack — distributed systems, incremental consumption, memory hierarchy — see `docs/reference.md`. Experiment narrative, measured numbers, and open questions live in `notes.md`.

---

## 1. Scope

A *map-only workload* is a single-pass transformation where each output record depends only on its corresponding input record. There is no shuffle, no barrier, no cross-node state, no reduce. In MapReduce vocabulary it is a map stage with no reduce stage; in Spark it is a job with no shuffle boundary; in SQL it is `SELECT f(x) FROM t` with no `GROUP BY`.

*Data validation* is the most common real-world instance. Given a corpus and a rule set, evaluate each rule per row and emit an annotation: a boolean valid/invalid flag, a bitmask of which rules failed, or a structured error record. Downstream pipelines consume that annotation to route valid rows into production and invalid rows into a quarantine, an investigation queue, or a deletion vector.

What's in scope:

- Row-local predicates (`fare_amount > 100`, `trip_distance > 10`, `ST_Area(geom) > 0`).
- Row-vs-broadcast predicates, where each row is checked against a small shared reference set (e.g. "which NYC borough?" against five polygons, "within 50 m of a known station" against a few hundred points). Still narrow, still EP once the reference set is broadcast.
- Per-row transformations that produce annotations (a validation status column, a bitmap of failed rules, an error-detail struct).

What's explicitly out of scope, because it breaks the map-only shape:

- Aggregations with non-trivial reduce (`GROUP BY hour`, `COUNT(DISTINCT user)`). These are the B and C columns of the 3×3 matrix.
- Joins beyond broadcast, including spatial self-joins, nearest-neighbour queries, any rule whose evaluation on row `i` depends on row `j`.
- Iterative / multi-pass algorithms (ML training epochs, fixed-point computations).
- Random point lookups. Map-only assumes a sequential scan.

One consequence worth naming up front: because there is no reduce, the output cardinality equals the input cardinality. The pipeline has *two* storage edges — a read edge and a write edge — whereas an analytics pipeline typically has one (input 1 TiB → output 8 bytes). Section 5 deals with what that implies.

---

## 2. Why the architecture works: stateless compute + decoupled storage

The modern shape for this workload is decoupled storage (S3, GCS, Azure Blob) and stateless elastic compute (spot VMs, Lambda, Fargate, Databricks jobs clusters). It is worth naming why this works, because the current consensus inverted a prior one.

The original Hadoop/HDFS design pinned compute to the machine holding the data. Networks were scarce at ~1 Gbps; disks were fast at ~100 MB/s per spindle × many spindles per node; moving bytes across the network was slower than running code next to them. "Data locality" was the primary scheduling objective, and the data node *was* part of the cluster's state.

That assumption flipped. Datacenter networks are now 10–100+ Gbps per node, aggregate cross-rack bandwidth exceeds the aggregate read bandwidth of a comparably-sized disk cluster, and object storage scales horizontally on GETs-per-second in a way no local filesystem does. Once storage bandwidth stopped being the scarce resource, pinning compute to data lost its justification, and an architecture emerged where:

- Bytes live in object storage, durable at ~$0.02/GB/month, eleven-nines, lifecycle-managed.
- Compute workers hold *no durable state*. They fetch inputs from object storage, process, write outputs back. Kill any worker mid-job and start another one with the same inputs; nothing is lost.
- Storage and compute scale on independent duty cycles. Storage is 24/7; compute is bursty and often idle.

Two concepts are easy to conflate:

**Embarrassingly parallel** is a property of the *problem*. It says tasks don't need to talk to each other. Map-only workloads are the cleanest case.

**Stateless compute** is a property of the *workers*. It says workers hold no durable local state — everything they need comes from outside, and everything they produce goes back outside. A Spark job with shuffles is *not* embarrassingly parallel, yet its executors are still stateless: shuffle data, checkpoints, and outputs live in external storage or can be recomputed from lineage. The *machines* are disposable even when the *computation* has dependencies.

Map-only workloads are the best possible fit for stateless elastic compute: the problem has no inter-task dependency *and* the workers need no durable state. Everything else is a harder case of the same architecture.

---

## 3. The producer–consumer convergence

The key piece of physics that makes the whole thing viable: **for most real analytical scans, streaming from object storage and having the data already in RAM land at roughly the same wall time.** Not because the network is as fast as DRAM — it isn't, by 10–100× — but because the CPU is doing enough work per byte that the raw bandwidth gap stops mattering.

### The compute chain

Every byte read from object storage passes through a fixed chain of CPU work before a rule can be evaluated:

```
  [ network ] --compressed bytes--> [ decompress ] --> [ decode ] --> [ rule kernel ] --> [ emit annotation ]
                                    LZ4/Snappy/ZSTD     RLE, dict,        per-row
                                                        Parquet encoding  predicates
```

Each step of the chain does CPU work proportional to the bytes flowing through it. Cores stay pegged near 100 % utilisation whenever the pipeline is fed, because every byte that arrives needs to be decompressed, decoded, and evaluated — there is no idle CPU waiting for bytes. The only way to avoid the chain is to answer the query entirely from metadata (e.g. `COUNT(*)` from row-group statistics).

### Busy vs. binding

"CPU is busy" and "CPU is the bottleneck" are two different statements. The first is about utilisation: cores are running instructions. The second is about whether adding more CPU reduces wall time. These come apart:

- **Network-narrower regime**: network delivers bytes slower than the compute chain could process them. Cores are still at ~100 % util (decompressing and decoding whatever arrives), but faster CPU wouldn't help — the NIC is the narrow pipe.
- **Compute-narrower regime**: the compute chain is slower than the network could deliver. Cores at ~100 % util, and faster CPU *would* reduce wall time.
- **Balanced regime**: the two are within a factor of two of each other. Small changes in rule complexity or network tuning flip which binds.

On our 8 × c7g.4xlarge fleet, Workload A sat in the balanced regime: the single-AND variant skewed network-narrower (14.85 s wall, 12.4 GiB/s wire aggregate = ~83 % of NIC burst budget), while the 1000-predicate variant skewed compute-narrower (55.55 s wall, kernel time dominating). Same fleet, same data, different rule complexity, different binding edge.

### Why the RAM-vs-S3 gap closes

RAM is an order of magnitude faster than the fastest network, two orders faster than typical. Yet the convergence holds for most real analytical work, because four effects close the raw bandwidth gap:

1. **CPU work per byte.** The compute chain runs at 1–5 GiB/s of logical data per core on modern hardware for non-trivial rule kernels. Aggregate across 16 cores and network and compute meet within a factor of a few — not the 10–100× raw bandwidth gap.
2. **Compression.** Parquet typically compresses 3–10×. A 3 GB/s network delivery becomes an effective 10–30 GB/s of logical data after decompress.
3. **Column pruning.** If the rules need three of fifty columns, only ~6 % of the bytes are fetched.
4. **Predicate pushdown and file skipping.** Parquet row-group min/max statistics let the reader skip entire row groups whose ranges exclude all predicates. Iceberg and Delta add table-level file skipping.

A RAM-cached input (Spark `.cache()`, Arrow in-memory tables, DuckDB temp tables) skips the decompress + decode steps of the compute chain; the rule kernel still runs. So the RAM-vs-S3 delta isn't bounded by the network-vs-DRAM bandwidth ratio — it's bounded by **how much of total wall time decompress + decode accounted for**.

- If the rule kernel dominates the compute chain (our 1000-predicate case — filter kernel is ~60 s of CPU work across 128 cores), RAM ≈ S3. Both pay the kernel cost; the decompress + decode savings are marginal in absolute terms.
- If decompress + decode dominate (our single-AND case — filter kernel is ~0.1 s total), RAM wins by the decode-to-total ratio. Not 100× — maybe 3–5× — because most of the single-AND wall time was network, and RAM doesn't help with network if the network was already the narrow pipe.
- If the rule is answerable from metadata (`COUNT(*)`), neither compute chain nor network is touched meaningfully. RAM still wins by whatever the metadata-fetch latency was, but the scan architecture isn't really the right thing to be reasoning about.

### When the convergence breaks

The argument above assumes a steady-state scan of a corpus with non-trivial per-row rule work. It fails in three specific situations:

- **Small datasets.** First-byte latency (20–200 ms per GET) dominates steady-state throughput. You never reach the regime where the compute-chain/network balance matters.
- **Multi-pass workloads.** ML training epochs pay the network cost on every pass. This is why worker-NVMe caches and `.cache()` exist — to amortise the first-pass network cost across later passes.
- **Random access.** Point lookups at small block sizes never reach steady state. The compute chain is underfed and the network is wasted on per-request overhead.

For a map-only validation pipeline on a well-sized fleet, none of those usually applies. Wall time is determined by the compute chain and the network, their relative rates, and which part of the compute chain dominates.

### The write edge

Map-only workloads have two storage edges, not one. The argument above concerns the *read* edge. On the write edge:

- Output bytes per input row is usually small (a bit, a byte, a short error struct).
- The CPU work to *produce* the annotation is bounded by the rule kernel itself.
- When output is sparse (see §5), aggregate write bytes are a small fraction of read bytes.

So the write edge rarely binds, but it's a real second layer in the pipeline and enters the binding-worker search for dense-output workloads.

---

## 4. Binding workers and ceilings

A map-only pipeline has this shape:

```
  [ object storage ] --GET--> [ decompress ] --> [ decode ] --> [ kernel ] --> [ write-back ] --> [ object storage ]
                              |                               |
                              +---- page cache / DRAM ---- ...+
```

Throughput equals `min` across layers. The **binding worker** is the layer that currently caps throughput; tuning moves the bottleneck from one layer to the next, not free-fall improvement across the board. A well-designed system makes the binding worker observable, so you know which layer to feed next.

The full catalog of parallelism axes — ILP, SIMD, multi-thread, pipeline, I/O concurrency, distributed data — is in `docs/project.md` and not re-derived here. What follows is the distilled ceilings lookup, in the order they appear as scale grows.

### Four scale regimes

Each regime introduces one new ceiling on top of all prior ones. A regime is *closed* when its ceiling is identified, explained, and either hit or bounded by a named external factor.

**Single core.** Ceilings: SIMD width, IPC, decode throughput. On Graviton3 (Neoverse V1), the filter kernel for Workload A lowers to SVE-256 `fcmgt` + `uaddv` with 2× unroll; measured steady-state IPC is 2.796 with ~95.8 % core utilisation at 2.49 GHz effective clock, which places it in the "silicon-bound" band with ~10 % soft headroom (V1 architectural max is ~8 IPC; "moderate" code runs at ~2, "stalled" at ≤ 1.5). Per-core insns/sec: ~111 G.

**Single node.** New ceilings: DRAM bandwidth, NIC bandwidth, per-prefix object-storage rate, per-node I/O concurrency. On `c7g.4xlarge` (16 vCPU Graviton3, 15 Gbps NIC baseline), the single-node cap for a read-heavy scan is the NIC, at ~1.87 GiB/s sustained. Hit this by running enough concurrent range requests to keep the pipe full; under-concurrency wastes it. A single GET tops out at ~80–100 MB/s, so saturating 15 Gbps takes roughly 30–50 concurrent requests; 100 Gbps NICs need 150+. AWS S3 serves ~9 GiB/s from a single dense prefix on a single client — that's a per-prefix ceiling, not a client ceiling, so sharding objects across prefixes raises it further.

**Small cluster (≤ 8 nodes).** New consideration: coordination cost of the trivial final combine. Measured: 93–97 % of linear scaling across 1 → 4 → 8 nodes, with no cross-node traffic during the compute phase. 8-node capstone on Workload A proper (single `AND` of two): 1 TiB in 14.85 s = **69.1 GiB/s**. With the 1000-predicate stress kernel: 55.55 s = 18.48 GiB/s, compute-bound. Orchestration is a per-node SSH launch with a synchronised `T_start` and a scalar-sized final combine on the laptop.

**Fleet (N × 10 nodes).** New candidate ceilings: aggregate S3 GET/s across prefixes (5,500/s per prefix, shardable across many prefixes), regional instance availability (the soft quota on vCPU count), NIC bandwidth at the AZ level, and budget. Not hit in this project — we were quota-capped at 128 vCPU = 8 × c7g.4xlarge. At a fleet of 100+ nodes the binding worker starts to shift back toward the request-rate side of storage, which is why real-world large-scale platforms shard one "dataset" across many prefixes or buckets.

### Ceilings reference table

| regime | layer | ceiling on our setup | tactic that unblocks |
|---|---|---|---|
| single core | decode | per-core parquet decode throughput | column projection, vectorised reader |
| single core | kernel | SIMD width × IPC (111 G insns/core-sec on Graviton3 SVE-256) | batch-major kernel layout, batch-level pruning |
| single node | NIC | ~1.87 GiB/s at 15 Gbps baseline | concurrent range requests (30–50 in flight) |
| single node | per-prefix S3 | ~9 GiB/s on a dense prefix | shard across prefixes |
| single node | per-request overhead | 20–100 ms first-byte latency per GET | coalesce adjacent ranges, pipeline requests |
| cluster | disjoint-work coordination | near-zero; the combine is scalar-sized | partition inputs by file, sync `T_start` |
| fleet | aggregate S3 GET/s | 5,500/s per prefix | multi-prefix corpus layout |
| fleet | regional vCPU quota | soft limit, request raises | budget and request |

---

## 5. The output side — what validation adds

An analytics pipeline has input at 1 TiB and output at 8 bytes. A validation pipeline has input at 1 TiB and output at `N_rows × bytes_per_annotation`, which is within one or two orders of magnitude of the input. The write edge becomes a real layer in the pipeline, and the storage layout for the output becomes a design decision with downstream consequences.

Three shapes for validation output:

**Exception table.** A separate dataset — usually much smaller than the input — listing only the rows that failed a rule, keyed by a stable row identifier plus the details of which rule(s) failed. When the invalid rate is low (most well-formed corpora have selectivity below 1 %), this is the cheapest output by a large margin. Downstream valid-only consumers simply read the original corpus.

**Delete vector / deletion bitmap.** A dense bitmap indexed by row position, one bit per row, marking invalid rows. Iceberg v2 and Delta Lake both support this as a first-class format (Iceberg calls it a "deletion file"). A single bit per row is 125 MB per TiB of input — tiny on the write side, and downstream readers apply the vector at scan time to filter invalid rows. Good fit when rules are boolean and the invalid set is not small enough to make an exception table clearly cheaper.

**Inline annotation column.** Add a `validation_status` column (or a bitmask-of-failed-rules column) to every row and rewrite the Parquet files. The most expensive output — you're effectively paying the input write cost twice — but it keeps everything in one dataset and makes downstream reads a simple `WHERE validation_status = 'valid'`. Appropriate when annotations are rich (multi-rule detail, error messages) and the corpus is rewritten anyway.

### The sparsity assumption

Most real-world validation rules are built on the assumption that inputs are *mostly* valid. A typical rule set pushes invalid rates below 1 % on a well-governed corpus. When sparsity holds, the output-side write cost is negligible compared to the input read cost, and the producer–consumer argument from §3 effectively applies to reads only.

When sparsity *doesn't* hold — every row fails at least one rule, or rules are structured to attach rich multi-field annotations to every row — write bandwidth re-enters the binding-worker search. The pipeline shape becomes symmetric: `read_bandwidth ↔ CPU ↔ write_bandwidth`, and either edge can bind depending on instance and network shape. Instances with balanced read/write NIC allocation matter more; some cloud NICs are asymmetric.

### A layout question for downstream consumers

Validation output creates a question the input-only pipeline doesn't have: **how do valid-only downstream readers skip the invalid rows?** The answer is a storage-layout decision made at write time, and it determines the shape of every subsequent read. This is structurally the same kind of decision as "partition input by H3 cell for spatial queries" or "Z-order by `user_id` for high-card aggregation" — a write-time choice whose effect compounds across all downstream reads.

---

## 6. Hyper-optimisation, and the cost of generality

There's a persistent illusion that tuning is about config flags. For map-only workloads on modern hardware, the wins live in a short and specific list:

- **Kernel engineering.** SIMD width (explicit vectorisation or a loop shape LLVM auto-vectorises), batch-major vs row-major data flow, batch-level pruning (compute a batch-wide `max` once, skip per-predicate work when the batch can't possibly satisfy it).
- **I/O concurrency.** Coalesce adjacent column-chunk range requests into a single GET; keep tens of GETs in flight per node; use HTTP keep-alive connections and enough of them to saturate the NIC. Default SDK settings are almost always too conservative.
- **Memory layout.** jemalloc as global allocator; arena reuse for hot batch buffers; avoid copies between network buffer, decompression buffer, and Arrow batch when the format allows zero-copy paths.

Where optimisations *don't* live, on this workload shape:

- Scheduling cleverness. There is nothing to schedule — disjoint inputs, no shuffle, no stragglers.
- Runtime features. Fault tolerance, speculative execution, dynamic partitioning. All solve problems this workload doesn't have.
- Micro-batching tricks. A well-tuned pipeline is already running at IPC ≈ 3; there's no slack to extract.

### The cost of generality

A hand-rolled pipeline in this project ran Workload A + 1000-predicate stress in 55.55 s at 8-node scale. A hyper-tuned Apache Spark 3.5.3 on the same fleet with the same workload and input ran it in 349 s. The ratio is ~6.3×, decomposable as:

- **~5× from batch-level optimisations the framework can't replicate.** Our Rust kernel computes `batch_max_fare` once per batch and skips predicates whose literal exceeds it. Catalyst — Spark's query optimiser — can't push per-batch min/max into user-supplied kernels because user kernels are opaque. The only way to get 1000 per-row predicates through Spark at all is a Scala UDF over a broadcast `Array[Double]`, which crosses the Catalyst ↔ user-code boundary once per row and forgoes any batch-level pruning.
- **~1.2× from SIMD width and boundary overhead.** The hand-rolled kernel lowers to SVE-256 with 2× unroll and a vectorised horizontal sum. The UDF's body is JIT-compiled by C2 to scalar `fcmp` + conditional increment — no vectorisation, because the data-dependent branch resists it. Plus the per-row UDF dispatch cost.

This 5–6× number is a distilled data point, not a universal law. It's the cost for *this specific workload shape*: map-only, CPU-heavy, with rule-specific optimisations that live below the framework's abstraction. On workloads where the framework actually earns its keep — shuffles, fault tolerance, SQL surface, ecosystem interop — the ratio shifts or inverts. Map-only validation needs none of the things the framework sells, so the full tax lands on the wall time.

For experimental detail and per-run measurements, see `notes.md`.

---

## 7. When to reach for this architecture

The architecture — stateless elastic compute, decoupled object storage, per-row map kernel, optional sparse output — is the right fit when all of the following hold:

- **The corpus is bounded, columnar, and lives in object storage.** Parquet, ORC, or similar. Incremental consumption from object storage is the I/O model the whole pipeline is built around.
- **Rules are declarative and per-row, or per-row + small-broadcast.** Row-local predicates, row-vs-shared-reference-set predicates. Anything requiring access to other rows in the same dataset breaks the shape.
- **CPU work per row is meaningful.** Cheap scans (`COUNT(*)`, single-field filters on already-thin data) are network-bound rather than CPU-bound, which means RAM wins by a factor and object storage stops looking equivalent. Validation usually lands well inside the CPU-bound regime because rules do real work per row.
- **Output is sparse or compact.** Per-row annotations fit in a bit, a byte, or a short struct. The invalid set is small relative to the input. An exception table or delete vector is a reasonable output shape.
- **The run is single-pass.** No iterative recomputation. If you need to re-evaluate rules repeatedly on the same data, budget for a local cache of the decoded corpus.

When to *not* reach for this architecture:

- **Tiny datasets** where first-byte object-storage latency dominates steady-state throughput. Single-node batch processing on local disk or memory is faster.
- **Random point lookups.** A key-value store or an indexed columnar engine is the right shape.
- **Multi-pass iterative algorithms** — ML training, fixed-point graph queries. Budget for a local cache of decoded data and a different architecture.
- **Cross-row-dependent rules.** Spatial self-joins, nearest-neighbour, density tests, deduplication. These are wide transformations; see the C column of `docs/project.md`.
- **Dense per-row outputs** where the write edge is within a factor of the read edge. The design becomes symmetric and the "network storage ≈ RAM" convergence applies to reads and writes independently.

The frame generalises beyond validation. Any map-only per-row transform with sparse or compact output fits: classification scoring at inference time, feature extraction, schema migration, PII redaction, format conversion with transformation. Validation is where the shape shows up most often, because the rule set is usually declarative and the output is usually sparse — two properties that make the match particularly clean.
