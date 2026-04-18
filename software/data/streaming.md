# Streaming Data, From Kafka to DRAM

The word *streaming* gets used at several different levels of the stack, and it means something subtly different at each one. This document walks through those meanings in order — from distributed systems down to the memory hierarchy — and then pulls out the unifying idea that makes them all feel like the same word.

---

## 1. How data moves, abstractly

Before talking about streaming specifically, it helps to name the axes along which any data movement varies. Most confusion about streaming comes from conflating these.

**Boundedness.** A dataset is either finite (a CSV, last quarter's transactions, a Parquet table) or unbounded (clickstreams, sensor readings, database changes). This is a property of the data itself, not of how you process it.

**Latency.** The gap between when something happens and when a consumer sees it. This runs on a continuum: nightly batch (hours), micro-batch (seconds to minutes), continuous streaming (milliseconds), hard real-time (sub-millisecond with guarantees).

**Control flow direction.** Pull (request/response, SQL, HTTP GET) means the consumer dictates when data moves. Push (pub/sub, webhooks, SSE, WebSockets) means the producer does. This determines who can be overwhelmed — in pull, the producer waits; in push, the consumer can drown.

**Coupling and retention.** Point-to-point queues (one consumer takes each message), broadcast topics (many consumers each see every message), and shared durable logs (Kafka: partitioned, retained, replayable) are three very different shapes. The log is special because it unifies history and present — new consumers can rewind to replay events, and the log itself becomes a system of record rather than a transport.

**State on the consumer.** Stateless processing treats each record independently. Stateful processing maintains aggregates, joins, or windows — which means memory, checkpoints, and a recovery protocol.

**Delivery semantics.** At-most-once, at-least-once, exactly-once; plus ordering guarantees — none, per-key, or total.

Streaming isn't a single point in this space. It's a family of choices that tend to cluster, and the word gets applied to at least three distinct phenomena.

---

## 2. Streaming as unboundedness (the distributed-systems sense)

The unifying commitment of this sense is: *treat data as unbounded, and make the computation a long-lived thing that data flows through, rather than a short-lived thing that flows over data.*

In batch, the query is a function `f(finite_dataset) -> result`. You can wait until you have the whole input. You can sort, shuffle, re-scan. Correctness is well-defined: given this input, the output is X.

In streaming, the query is `f(input_so_far) -> output_so_far`, evaluated forever. You never have the whole input. "Sort" and "group by" stop being straightforwardly implementable because there's always more data coming. Correctness becomes trickier: what does "count per user per day" mean when events arrive late, or out of order, or when "today" hasn't ended?

This forces streaming systems to confront things batch can ignore:

**Event time vs processing time.** An event happened at T1; you see it at T2; T2 − T1 is arbitrary. Every serious streaming system must pick which notion of time the query operates in, and if it's event time, how to decide when results are "final."

**Windows and watermarks.** Since you can't wait for end-of-input, you impose artificial boundaries — tumbling, sliding, session. Watermarks encode the system's best guess that "no events older than X will still arrive," which is the only way to eventually emit a result without waiting forever.

**Backpressure.** In a batch job, the scheduler allocates resources. In streaming, producers and consumers run concurrently at different rates, and you need an explicit protocol — Reactive Streams credits, Kafka consumer lag with pause/resume, Flink's credit-based flow control, ZStream's pull-based semantics — for the slow side to tell the fast side to wait.

**The stream–table duality.** Every stream is a changelog: fold it with a reducer and you get a table. Every table is a snapshot of some stream at a point in time. Kafka Streams, Flink SQL, Materialize, and Delta Live Tables all lean hard on this: a SQL `GROUP BY` over a stream produces a continuously maintained table, and that table can be republished as a stream of updates.

The deep shift is that *batch is a special case of streaming, not the other way around*. A finite file is just a stream that ended. Exactly-once batch is easy because you can retry the whole job; exactly-once streaming requires transactional offsets, idempotent sinks, and careful state snapshots, because the job never ends and you can't just "run it again."

Once you see data as an unbounded log of immutable events, a lot of architecture clicks into place. CDC is streaming the database's changelog. Event sourcing is choosing the log as your system of record. Lambda vs Kappa is really an argument about whether to run batch alongside the stream or derive everything from the stream alone. Even request/response becomes a degenerate case — a stream of length one with a terminal marker.

---

## 3. Streaming as incremental consumption (the 5 TB case)

This is a lower-level, purely mechanical idea that gets called by the same name: you pull data through memory in chunks rather than materializing the whole input at once. It's about memory discipline, not about the shape of the data.

A 5 TB Parquet dataset on S3 is finite and bounded — you know exactly how many bytes there are, it has a definite end, and "the input" is a closed set. But you can't load it into a 64 GB machine, so you consume it incrementally.

**What "streaming from S3" actually means mechanically.** You issue ranged GET requests (or read objects one at a time from a listing) rather than downloading everything upfront. Bytes arrive as a TCP stream, get parsed into records (Parquet row groups, CSV lines, JSON objects), and those records flow through your processing pipeline. At any moment, only a bounded working set is in memory: some network buffers, a decoder's state, the current batch being processed, maybe some aggregation state. Processed records become garbage and get collected. The key property is that peak memory is *independent of total input size* — it depends on your chunk size and pipeline depth, not on the 5 TB.

This is what Spark does when reading a large Parquet dataset: partitions are read by executors in chunks, transformations are pipelined where possible, shuffles spill to disk when state exceeds memory. It's what DuckDB does with vectorized execution over Parquet on S3. It's what a hand-written ZIO Stream or fs2 pipeline does.

**Where this sense diverges from the unbounded one.** For a 5 TB bounded input, most of the hard problems of unbounded streaming don't apply. You don't need watermarks or windowing — the dataset has a definite end, so `GROUP BY user_id` is just a normal aggregation that emits when the input is exhausted. You can shuffle and sort globally if you're willing to spill to disk. Exactly-once is trivially achievable by making the job idempotent and retrying on failure. You can parallelize by partitioning the input. Progress is measurable — you know what fraction of the 5 TB you've consumed.

So the honest label for this case is **batch processing with streaming-style I/O**. The computational model is still `f(finite_input) -> result`. The execution strategy pulls data incrementally rather than materializing it.

**Why the terminology is muddled, and why it mostly doesn't matter.** The same tools serve both cases. Spark Structured Streaming and Spark batch share the same DataFrame API. Flink handles bounded and unbounded inputs with the same DataStream API and explicitly markets this as a feature. ZIO Streams and fs2 are happy to process a finite file or an infinite Kafka topic with the same abstractions, because at the level of "pull elements through a pipeline with backpressure," the two cases are identical.

The useful distinction to carry in your head:

- *"We stream data from S3"* → almost always incremental consumption of a bounded dataset. Engineering concern: memory and throughput.
- *"We have a streaming pipeline"* → usually a long-lived job processing an unbounded source. Engineering concerns: latency, late data, state management.

---

## 4. Streaming in the memory hierarchy (the hardware sense)

The word appears yet again at the hardware level, and here it means something more specific: *exploiting sequential access in a medium where sequential is much faster than random*. Going through the hierarchy from top to bottom:

**Registers.** No. Registers are a tiny fixed set of named slots (sixteen general-purpose 64-bit registers on x86-64, plus SIMD/vector registers) that the CPU reads and writes in a single cycle. There's no notion of pulling a sequence through them — they're the endpoint where computation happens. Data streams *into* registers from the cache hierarchy, not through them.

**L1 cache.** Not really. L1 is so close to the core (~4–5 cycle latency, ~32 KB per core) that the mental model is random access, not streaming. You don't "stream from L1" — you just access it.

**L2 cache.** Same story, with one wrinkle: hardware prefetchers start to matter here. If you're iterating over a large array, the prefetcher detects the stride and pulls cache lines from L3 or memory into L2 ahead of your access. That prefetching behavior is sometimes described as a "stream" of cache lines, but L2 is a staging area, not a source.

**DRAM.** Yes — this is where "streaming" becomes a genuinely important hardware term. Modern DRAM is fundamentally built around burst transfers: when you request a cache line (64 bytes on x86), the memory controller issues a burst that reads 8 consecutive 64-bit words in rapid succession. Sequential access is often an order of magnitude faster than random access because it amortizes row-activation cost, keeps the memory controller's prefetchers happy, and exploits open DRAM rows. The STREAM benchmark (Copy, Scale, Add, Triad) is the standard measure of sustainable sequential memory bandwidth, and it's called STREAM precisely because it captures this "pull a long contiguous sequence through the CPU" pattern. x86 even has dedicated *non-temporal* or "streaming" store instructions (`MOVNTPS`, `MOVNTDQ`) that bypass the cache when writing a large array you won't read back soon — the word is literally in the instruction mnemonics.

**SSD.** Yes, and for analogous reasons. NVMe SSDs have vastly higher sequential throughput (several GB/s on consumer drives, 7+ GB/s on PCIe 4.0/5.0) than random-access throughput, especially at small block sizes. The drive's controller, the OS page cache, and the filesystem all optimize for sequential reads: readahead kicks in, the flash translation layer can pipeline requests, and you avoid the per-IO overhead that dominates small random reads.

**Others.** Network (NIC DMA rings, kernel-bypass frameworks like DPDK, RDMA) is fundamentally stream-shaped — packets arrive as a continuous sequence and the whole stack is built around throughput under backpressure. GPU memory (HBM) has an even more extreme sequential-vs-random gap than CPU DRAM, which is why GPU kernels are obsessed with "coalesced" memory access. And network-attached storage and object stores sit at yet another level of the same pattern.

**The underlying principle.** "Streaming" becomes a meaningful word exactly when a medium has a significant gap between its *sequential throughput* and its *random-access latency*. DRAM, SSDs, networks, and GPU memory all have this property dramatically. Caches and registers don't, because their whole job is to make random access fast.

Each level of the memory hierarchy has a latency, a bandwidth, and a *sequentiality premium* — how much faster it is when accessed in order. The premium grows as you go down:

- Registers / L1: no premium.
- L2 / L3: small premium, mostly mediated by prefetchers.
- DRAM: large premium (≈10×).
- SSD: huge premium (≈100× for small random IOs vs large sequential).
- Network storage, tape: almost purely streaming media — random access is prohibitive.

---

## 5. The unifying idea

The three senses of streaming aren't unrelated — they're the same idea expressed at different scales.

**Hardware streaming:** exploit order at the memory/storage/network level, because sequential is dramatically faster than random.

**Incremental-consumption streaming:** pull a bounded-but-large dataset through memory in chunks, so peak memory is independent of total size.

**Unbounded streaming:** treat data as a never-ending log, and make the computation a long-lived pipeline that runs forever.

They stack. When you stream a 5 TB Parquet file from S3, you're stacking sequentialities: S3 returns bytes in order, the NIC receives them in order, they land in a DRAM buffer via DMA in order, your decoder walks that buffer sequentially, the CPU prefetcher sees the stride and keeps L2 warm, and the SIMD lanes consume contiguous elements from L1. Every level of the stack is doing the thing it's fastest at. Random access at any level would wreck the throughput of the whole pipeline.

This is why *cache-friendly*, *sequential access*, and *streaming* end up being near-synonyms in performance discussions. They all name the same underlying fact: modern hardware is dramatically faster at consuming data in order than out of order, and the gap only widens as you move further from the CPU. A well-designed data processing system is essentially one long stream-shaped path from cold storage to the register file, with as few random-access detours as possible.

And at the top of that stack sits the distributed-systems sense of streaming: an unbounded log of immutable events, processed by a long-lived pipeline under backpressure. The hardware exploits order in nanoseconds. Kafka exploits order in milliseconds. They're the same idea, recursively applied — *keep the pipeline full, exploit sequentiality, avoid materializing more than you have to.*

Whether you're looking at DRAM bursts, a ZIO stream over a Parquet file, or a Flink job consuming a Kafka topic, the shape of the computation is the same: data flows, the computation stays put, and order is everything.
