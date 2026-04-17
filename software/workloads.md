# Embarrassingly Parallel Predicate Evaluation on Large Parquet Datasets

A reference for reasoning about the workload shape: many independent predicate evaluations against a shared dataset, at volumes from ~100 GB to multi-TB.

## 1. Workload definition

Many independent predicate evaluations against a shared dataset, where each record (or small group) can be evaluated in isolation. No shuffle required. Typical examples: rules-engine evaluation, compliance and fraud screening, data quality checks, ad-targeting match computation, cheap per-record scoring, feature extraction for downstream training.

The defining characteristics:

- **Embarrassingly parallel.** Output for any record depends only on that record's contents.
- **Read-heavy, scan-shaped.** The dominant work is reading the dataset and applying per-row logic.
- **Possibly many predicates.** The same dataset may be evaluated against dozens or hundreds of independent conditions.
- **Parquet on object storage or local NVMe.** The typical physical layout in modern data platforms.

## 2. The central model

The single most useful framing:

```
wall_clock ≈ max(
    fetch_overhead,
    bytes_touched / effective_scan_rate,
    predicate_cpu_cost,
    output_write_cost
)
```

Where `bytes_touched` is the physical bytes actually read from storage after:

- partition pruning
- row-group min/max pruning
- page index pruning
- column projection (only the columns the predicates reference)

The key insight: **logical dataset size is not the relevant variable**. A 5 TB dataset with strong pruning and tight column projection might touch only 100 GB physically; a nominally small 100 GB dataset read with a wide column set and no pruning touches all 100 GB. The spread is 50x on the same logical workload.

Everything interesting about optimizing this workload reduces to one of three moves:

1. Reduce `bytes_touched` — better Parquet layout, predicate pushdown, projection discipline.
2. Increase `effective_scan_rate` — storage choice, client tuning, engine choice.
3. Change the dominant term — move to a database, or accept CPU-bound and scale cores.

## 3. Scan rate reality

The `effective_scan_rate` number in the model is not a property of the engine — it's a property of the whole storage + network + client stack. Realistic ranges for well-tuned Parquet reads with modern engines (DuckDB, DataFusion, polars, Arrow-backed):

| Storage | Effective scan rate (per instance) | Notes |
|---|---|---|
| Local NVMe (striped) | 3–6 GB/s | Often CPU/decode-bound at the high end |
| Local NVMe (single) | 2–4 GB/s | Typical for a modern instance |
| Local SATA SSD | 500 MB–1 GB/s | Storage-bound |
| EBS gp3 / Azure Premium | 500 MB–1.5 GB/s | Sized for OLTP, disappointing for scans |
| S3/GCS/ADLS, tuned client | 1–3 GB/s | Requires aggressive concurrency, large instance |
| S3/GCS, default client | 200–600 MB/s | Default concurrency is usually too low |
| NFS/EFS | Variable, often poor | Long-tail latency wrecks parallel workloads |

The ~30x spread across storage options, for the same engine and same data, dominates engine choice. Before arguing about DuckDB vs. Spark, settle the storage path.

**Parquet scans are not sequential.** A Parquet read is: fetch footer → decide row groups → fetch column chunks (scattered byte ranges) → optionally fetch page indexes → decode. On object storage this means hundreds to thousands of small range requests per file. Client-side request concurrency, HTTP connection pooling, and prefetch behavior dominate the achievable rate.

**Cold start matters.** For 100 GB workloads, cold-to-warm execution can differ by 3–10x. Most benchmarks compare warm-to-warm; production jobs usually run cold.

**Tail latency compounds under sharding.** Wall time across N workers is the slowest worker, not the average. S3 p99 read latency can be 10x p50. At enough workers, every run hits it. Production systems add hedged reads, retries with varied prefixes, or randomized file ordering to mitigate this.

## 4. Bottleneck regimes

For this workload shape, three regimes:

**I/O-bound.** The default for simple numeric/date/boolean predicates on columnar data with few referenced columns. Most time is spent reading and decoding bytes. This is where the `bytes_touched / scan_rate` term dominates.

**Memory-bandwidth / decode-bound.** When data is cached or pruning is so strong that storage is no longer the limit, but predicates are still cheap. The bottleneck is scanning Arrow/Parquet buffers through CPU caches.

**CPU-bound.** When predicates themselves are expensive: regex, complex string matching, JSON parsing, geospatial ops, ML inference, Python UDFs. Here adding cores helps linearly; reducing bytes touched may not.

A useful heuristic: **many cheap predicates do not make a workload CPU-bound.** Vectorized execution over columnar data makes simple comparisons nearly free relative to scan cost. A 50-predicate evaluation with shared subexpression reuse is often still I/O-bound. CPU-bound means per-row work is intrinsically expensive, not that there's a lot of it.

## 5. Why Spark is usually the wrong tool here

Spark's competitive advantage is handling shuffles well — adaptive query execution, skew handling, sophisticated join optimization, lineage-based fault tolerance. For embarrassingly parallel workloads, none of that machinery applies.

What you pay for Spark on this workload:

- Driver startup and executor allocation (tens of seconds, even warm)
- Per-task coordination overhead (significant at 800+ tasks for 100 GB at default 128 MB partitions)
- JVM warm-up and GC
- Python UDF serialization cost if predicates are Python
- Orchestration complexity that assumes a shuffle will happen

What you gain: stage-level fault tolerance on very long jobs, a consistent platform story, operational familiarity.

At 100 GB, the fixed overhead of Spark can exceed the actual scan time of a single-node engine. At multi-TB, Spark is defensible but not optimal unless the predicate is genuinely CPU-expensive and needs horizontal scale-out.

**The key conceptual split:** *data distribution* and *compute distribution* are orthogonal. You can distribute compute without using a distributed compute framework. For embarrassingly parallel predicates, a scheduler fanning out N single-node jobs (Kubernetes Jobs, AWS Batch, Argo Workflows) is often strictly better than Spark:

- Each worker runs at full single-node efficiency (no JVM, no task scheduling, native vectorized execution).
- Zero shuffle infrastructure.
- Zero distributed-engine coordination cost.
- Failure isolation is per-shard, which is the right granularity for this workload.
- You pay only for what you need, when you need it.

## 6. Wall-clock estimates

For well-laid-out Parquet with column projection and predicate pushdown, using native vectorized engines (DuckDB, DataFusion, Polars):

### 100 GB logical shard

| Physical bytes touched | Local NVMe | Remote (tuned S3) |
|---|---|---|
| 5–20 GB (strong pruning) | 2–10 s | 5–25 s |
| 20–50 GB (columns only) | 8–25 s | 15–60 s |
| ~100 GB (no effective pruning) | 30–60 s | 70–150 s |

### 1 TB, sharded as 10 × 100 GB workers

Wall time ≈ one shard's wall time, assuming storage supplies aggregate bandwidth:

- Strong pruning: 5–25 s
- Typical columns-only scan: 10–60 s
- Near-full physical scan: 30 s – 2.5 min

Bottleneck becomes aggregate storage bandwidth, not compute coordination.

### Multi-TB (e.g., 10 TB as 100 shards)

Roughly the same as one shard, plus tail effects (metadata, stragglers, uneven shards):

- Strong pruning: tens of seconds
- Columns-only scan: under 1–2 min
- Near-full physical scan: 1–5 min

If storage can't deliver aggregate throughput, time scales toward `total_bytes / aggregate_bandwidth`.

### Spark overhead

Spark on the same data is typically:

- At 100 GB: seconds to tens of seconds slower (significant as a percentage of total).
- At multi-TB: modestly slower in percentage terms, not order-of-magnitude, unless the job is badly written.

## 7. Data fetching and caching

Data fetching is not trivial. "Just cache the hot data" is simple to say and messy to implement.

### Why naive caching breaks down

**What to cache.** The full dataset is often too large for local NVMe. A projected subset requires knowing predicates in advance, which breaks the "many independent predicates" model. A column subset breaks when a new predicate touches an uncached column.

**Cache invalidation.** Upstream Parquet changes (new partitions, compaction, schema evolution) invalidate caches. Options: TTL (wasteful), event-driven (complex, requires integration), or version-keyed via a table format like Delta/Iceberg/Hudi (commits you to that format).

**Cache coherence across workers.** Independent local caches on N workers require stable sharding. A shared cache tier (Alluxio, JuiceFS, a dedicated NVMe service) adds a distributed system with its own failure modes.

**Eviction policy.** LRU is default and often wrong for analytical workloads — a one-time cold scan evicts hot data about to be reused. Production-grade systems (Snowflake result cache, Databricks Delta Cache, Trino file cache) invest substantial engineering in this because naive caching backfires.

**Pre-warming.** Requires knowing the access pattern, scheduling the prefetch, and handling the case where it isn't done when the job starts. Usually grows into a separate data pipeline.

**Cost.** NVMe-equipped instances (e.g., i4i family) are ~3x the compute-equivalent. Running these 24/7 for jobs that execute an hour a day is wasteful compared to reading from object storage each time.

### Strategies that actually work

| Strategy | When it fits | Cost |
|---|---|---|
| No cache, tune object-store client aggressively | Most workloads up to multi-TB | Low — engine tuning only |
| Tiered storage with explicit pre-staging | Stable hot set, bursty-interactive access | Medium — pipeline for staging |
| Caching layer (Alluxio, JuiceFS, Delta Cache) | 10+ workers, TB-scale hot data, hours of compute/day | High — operational burden |
| Load into columnar database (DuckDB, ClickHouse, StarRocks) | Same dataset queried repeatedly with many predicates | Low–medium — one-time ETL |
| Ephemeral cold S3 with spot instances | Background jobs, latency-insensitive | Low |

**The quietly best answer for the "many predicates against stable dataset" pattern is often option 4: load into a columnar database.** A DuckDB file on local NVMe, or a ClickHouse/StarRocks instance, handles caching, layout, and indexing as part of its model. Query latency drops 5–20x vs. repeated file scans. Underused because teams default to "Parquet on S3 + engine" thinking.

### Extended scan rate is a whole-stack property

The honest version of the scan rate is a function of:

- Storage medium (local NVMe vs. S3 vs. NFS)
- Client concurrency and connection pooling
- Parquet layout (row group size, column ordering, file count, sort order)
- Cache state (cold vs. warm OS page cache)
- Tail latency characteristics of the storage layer
- Network path (same-AZ, cross-AZ, cross-region)

"Perfectly optimized" quietly assumes all of these are tuned. In practice, Parquet layout issues (small files, tiny row groups, missing statistics, no sort on predicate columns) blow up effective scan rate by 5–20x more often than engine choice does.

## 8. Key optimizations

Ordered roughly by impact:

**Scan once, evaluate all predicates in one pass.** If a job has 50 predicates and runs 50 scans, wall time and cost are multiplied by 50. Correct design: one scan per shard, compute shared atomic conditions once, combine masks in memory. Moves the work from `O(N_predicates × scan_time)` toward `O(scan_time + mask_combine_time)`. This is the single largest structural win for this workload.

**Fix Parquet layout.** Row groups sized for the engine (DuckDB likes 100k–1M rows; Spark defaults to 128 MB splits). Column order that groups predicate columns. Sort on the most selective predicate column to maximize min/max pruning. Avoid the 50,000-tiny-files antipattern. Compaction is boring work with outsized returns.

**Projection discipline.** Read only referenced columns. Obvious, but easy to break when predicates are composed dynamically or when an engine materializes more than it needs. Verify by inspecting `bytes_touched` in engine telemetry.

**Tune the object-store client.** DuckDB `threads`, DataFusion concurrency, Spark `spark.hadoop.fs.s3a.connection.maximum`. Defaults are almost always too conservative. Target 2–3 GB/s per instance from S3 before considering a cache layer.

**Shard at the right granularity.** Shard size should keep workers busy for at least tens of seconds to amortize startup cost, but small enough that tail latency doesn't dominate. For 100 GB–1 TB, 10–100 shards is typically the sweet spot.

**Use a scheduler, not a distributed engine.** Kubernetes Jobs, AWS Batch, Argo — fan out N single-node jobs. Cheaper, faster, simpler than Spark for this workload shape.

**Move to a columnar database if the dataset is re-scanned often.** DuckDB file, ClickHouse, StarRocks. Stops treating it as a file-scan problem.

## 9. Decision framework

Starting questions, in order:

1. **Is the predicate CPU-expensive?** If yes (regex-heavy, UDF-heavy, ML inference), the bottleneck is cores. Use any framework that can scale out cheaply — Ray for Python-native, Spark if already on Databricks/EMR, sharded single-node jobs for everything else.

2. **Will this dataset be re-scanned many times?** If yes, load it into a columnar database (DuckDB, ClickHouse, StarRocks) and query that. Most "repeated predicates on stable Parquet" workloads should not be repeated Parquet scans.

3. **What's the physical bytes touched after pruning and projection?** If under ~50 GB, a single fat node is ideal. If 50 GB to a few TB, sharded single-node jobs are usually best. Above that, aggregate storage bandwidth becomes the constraint regardless of engine.

4. **Is the storage path tuned?** If not, fix that before comparing engines. A badly tuned S3 client makes all engines look slow; a well-tuned one closes most of the gap to local NVMe.

5. **Is Parquet laid out well?** If not, fix that before anything else. No engine recovers from small files, tiny row groups, and missing statistics.

6. **Do you need stage-level fault tolerance on jobs longer than a few hours?** If yes, Spark's retry model earns its place. If no, simpler orchestration is better.

## 10. Summary

- **Logical size is the wrong variable.** `bytes_touched` is the right one.
- **I/O-bound by default.** Cheap predicates on columnar data are scan-limited, not compute-limited.
- **Spark's strengths (shuffle, skew, optimizer) don't apply to this workload.** For embarrassingly parallel predicates, it is usually over-engineered.
- **Sharded single-node jobs are the underrated default** for 100 GB – multi-TB at cheap predicates.
- **A columnar database beats repeated Parquet scans** for stable datasets queried often.
- **Storage tuning matters more than engine choice.** 30x spread on scan rate across storage options swamps engine differences.
- **Scan once, evaluate all predicates in one pass** is the single biggest structural optimization.
- **Caching sounds easy and isn't.** Before building a cache tier, try tuning the object-store client or loading into a database.
