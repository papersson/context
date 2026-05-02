# Hiding S3 Latency with Pipelined Prefetching

## The problem

You have terabytes of data sitting in S3 and a cluster of expensive compute — GPUs training a model, CPUs running a vectorized scan, simulation nodes chewing through a grid. The data won't fit in memory, so it has to stream from S3 as the computation runs. The naive way to do this — fetch a chunk, process it, fetch the next chunk, process it — wastes most of what you're paying for: the GPUs sit idle waiting on S3, the network card sits idle most of the time, and a job that should take an hour takes a day.

The goal of the pattern this article describes is to make the kernel — the inner computational loop, whatever it is — see a steady stream of in-memory data, running at full hardware speed, while the IO subsystem absorbs all of S3's latency invisibly in the background. Done right, S3 behaves like another throughput tier in the memory hierarchy: slower than local storage, but hidden behind enough parallelism that the kernel rarely waits for it. Done wrong, you're paying cluster prices to wait on the network.

## The two numbers that define S3

S3 has two performance characteristics that pull in opposite directions, and understanding the tension between them is the entire reason the pattern exists.

**Per-request latency is high and mostly fixed.** A single GET request often takes roughly 20 to 100 milliseconds from request issued to first byte received, with a longer tail. Connection reuse, same-region placement, and a tuned client can remove avoidable overhead, but they do not remove the basic round-trip and service-handling floor. No amount of cleverness makes one S3 GET behave like a local memory load.

**Aggregate bandwidth is very high if you issue enough independent requests.** S3 is a massively distributed system. Each prefix supports thousands of requests per second; spread requests across enough objects and prefixes and the service-side ceiling is often far above the client-side ceiling. On the client side, a properly configured EC2 instance can pull data from S3 at close to its network line rate — tens of gigabits per second on a mid-tier instance, hundreds on a large one. For most well-sharded scan workloads, the bottleneck is your network card or client, not S3 itself.

This is the key tension: individual requests are slow, but the *system* is fast if you ask many things at once. The whole pattern is a way to exploit the second property to hide the first.

## Why serial fetch-then-process is catastrophic

Concrete arithmetic. Suppose your kernel processes 64 MB chunks at, say, 5 GB/s — a reasonable rate for a GPU consuming a training batch. Each chunk takes ~13 ms of compute. Each S3 fetch takes ~50 ms of latency.

In a serial loop, every iteration is 50 ms of fetch followed by 13 ms of compute. Effective throughput is 64 MB / 63 ms ≈ 1 GB/s. The GPU runs at 20% utilization. Your network card, capable of 12 GB/s on a typical instance, runs at less than a tenth of its capacity. You're paying for a Ferrari and using it as a bicycle.

Worse, the two resources you're underusing — compute and network — are independent. The GPU sits idle *during* the fetch. The network sits idle *during* the compute. They're never both working at once. The kernel is starved, and the system is wasted.

Fixing this is the whole point.

## The core insight: turn latency into a throughput problem

You cannot reduce S3's per-request latency. That is a hard physical floor. But you can keep many requests in flight simultaneously. While request 1 is waiting for its first byte, you issue requests 2, 3, 4, ..., 50. By the time request 1 completes, request 2 is about to complete, then request 3, and so on. The stream of completions arrives at a rate set by your *concurrency*, not by individual request latency.

The arithmetic that makes this precise is **Little's Law**, the foundational equation of all queueing systems:

> **request throughput = concurrency / latency**
>
> **byte throughput = concurrency × bytes_per_request / latency**

Read it as: the rate at which work completes equals the number of items in flight divided by the time each item takes. Rearranged: to achieve a target byte throughput, you need concurrency proportional to target throughput times latency, divided by bytes per request.

Worked example. Latency is 50 ms per S3 GET. You want 10 GB/s of ingest at 10 MB per fetch — so 1000 fetches per second. Plugging in: concurrency = request_rate × latency = 1000 × 0.05 = **50 fetches in flight**, continuously. Equivalently: concurrency = 10 GB/s × 0.05 s / 10 MB = 50. If you want 20 GB/s with the same chunk size and latency, you need 100 in flight. The formula tells you exactly how deep the pipeline needs to be.

This is not a trick or a heuristic. It's the same arithmetic that governs why TCP needs a window size proportional to bandwidth times round-trip time, why GPUs need thousands of threads to hide DRAM latency, why CPUs use out-of-order execution to hide cache misses, and why every database engine prefetches. It is one principle showing up at every scale of the computing stack.

Once you've internalized Little's Law, latency hiding stops feeling like a trick and starts feeling like arithmetic. The question is never *whether* concurrency hides latency — it does, that's a theorem — but whether you have *enough* concurrency to hide it at your target throughput.

## The pipeline structure

The structural realization of Little's Law in code is a producer-consumer pipeline:

1. A pool of **IO workers** issues asynchronous range-GET requests against S3, many at a time. As bytes arrive, each chunk is decompressed and deserialized into an in-memory representation — a tile, a batch, a record group, whatever the unit is.

2. Completed chunks land in a **bounded queue** in RAM.

3. The **kernel** — the GPU training step, the vectorized scan, the simulation update — pulls chunks off the queue and processes them at full hardware speed.

4. As the kernel finishes a chunk, the slot it freed allows another fetch to proceed.

The two sides run independently. The IO side is bounded by S3's aggregate bandwidth and your network link. The compute side is bounded by your CPU/GPU's processing rate. Provided the IO side delivers chunks at least as fast as the kernel consumes them, the kernel runs at hardware speed and never sees S3 latency. The fetches still take 50 ms each — but the kernel doesn't care, because some other fetch is always finishing while it works.

The minimal version of this pattern, with exactly two buffers — one being processed, one being filled — is called **double buffering** and predates computing as we know it. It's how film projectors switched reels, how disk controllers handled platter reads, how graphics pipelines avoided tearing. Deeper pipelines with N buffers in flight are the natural generalization, and Little's Law tells you how to choose N.

## Backpressure and bounded queues

The queue between producer and consumer must be **bounded** — must have a fixed maximum size — and this matters more than it might first appear. Without bounding, you have a latent out-of-memory failure waiting for the right traffic pattern to trigger it.

The reason: producer and consumer run independently and at potentially very different rates. If the producer (IO) is faster than the consumer (kernel) — even temporarily — completed chunks pile up in memory waiting to be processed. With an unbounded queue, a fast producer feeding a slow consumer will eventually exhaust RAM and crash the process.

This isn't paranoia. S3 latencies have long tails, so completions are bursty. You might have 100 fetches outstanding all complete within a few hundred milliseconds of each other, and suddenly there's many gigabytes of data sitting in memory waiting to be consumed. If the kernel is slowed by anything — a longer-than-usual batch, a GC pause, contention with another process — the queue grows. With no upper bound, growth doesn't stop until the OOM killer arrives.

The fix is to bound the queue and have producers **block when it's full**. When the queue has room, IO workers enqueue completed chunks freely. When it's full, they wait until the consumer pulls something off, freeing a slot. This blocking propagates the consumer's rate back to the producer automatically — when the kernel slows, IO naturally slows to match — and is called **backpressure**. It's the foundational pattern for any sustained-throughput producer-consumer system.

In Python's asyncio it's `asyncio.Queue(maxsize=N)`. In Java, `BlockingQueue` with a capacity. In Go, a buffered channel `make(chan T, N)`. In Rust with Tokio, `tokio::sync::mpsc::channel(N)`. The pattern is universal because the underlying problem is universal.

Sizing involves a real tradeoff. **Too small** and you lose latency hiding — if the queue holds only one or two chunks, any hiccup in IO latency stalls the consumer because there's no buffer to absorb it. **Too large** and you waste memory and increase end-to-end latency, since chunks sit in the queue longer before being processed. The right size is roughly "enough to absorb expected variance in producer rate." A common heuristic: hold a few seconds of consumer throughput. If the kernel processes 1 GB/sec and you want to absorb up to two seconds of IO stalls, the queue should hold ~2 GB. Combined with Little's Law for pipeline depth, you now have concrete numbers for both concurrency and queue size.

A practical refinement: bound the queue by **bytes**, not just by chunk count, when chunk sizes vary. A queue capped at 16 chunks behaves very differently if chunks are sometimes 10 MB and sometimes 200 MB. Bounding by bytes keeps memory predictable regardless of chunk-size variance.

## Conditions for the pattern to work

Three things must be true for pipelined prefetching to actually hide S3 latency:

**Sufficient concurrency.** Little's Law gives you the floor — concurrency must be at least target byte throughput times latency, divided by bytes per request. Too few outstanding fetches and the kernel will stall waiting for data; you'll see it as drops in GPU utilization or idle CPU cores. The fix is more in-flight requests, more IO workers, deeper async machinery, or larger chunks when that is compatible with the format and memory budget.

**Sufficient aggregate bandwidth.** Your network link, your S3 client implementation, and the S3 service itself must collectively be able to deliver data at least as fast as the kernel consumes it. On AWS this is usually solvable by choosing an appropriate instance type and using a good client (AWS CRT, s5cmd, the Rust object_store crate, recent boto3 with TransferManager). But it must be *checked*, not assumed — naive clients can leave 80% of your NIC capacity on the table.

**Sufficient compute time per byte.** The kernel must spend enough time on each chunk that the IO subsystem can prepare the next one. This is **arithmetic intensity** in roofline terms — useful work performed per byte fetched. High intensity (a transformer layer, a complex query, a physics update) means the kernel is the bottleneck and pipelining works beautifully. Low intensity (summing bytes, simple filters) means you're fundamentally IO-bound, and no amount of pipelining gives you more than S3 can supply.

When all three hold, the kernel sees a steady stream of in-RAM data and runs at full speed. S3 effectively disappears. When any one fails, the pattern breaks down in a characteristic way — and recognizing which condition is failing tells you what to fix.

## Horizontal scaling and the NIC

The bandwidth ceiling for a single machine is its **NIC** — Network Interface Card, sometimes Network Interface Controller. This is the hardware that connects the machine to the network, and it has a fixed maximum throughput. On AWS, NIC bandwidth is tied to instance size: small instances get a few Gbps, mid-tier instances get 25–50 Gbps, the largest network-optimized and accelerated-computing instances get 100–400 Gbps, with the very newest GPU instances pushing higher. You don't physically swap NICs in the cloud; you pick an instance type and inherit its networking.

When the NIC saturates, you have two choices: vertical scaling (a larger instance with a fatter NIC) or horizontal scaling (more instances, each with its own NIC). For kernel-shaped workloads on S3, horizontal is the dominant choice, for several reinforcing reasons:

**S3 is itself horizontally scaled on the server side.** It's designed for many concurrent clients, with no central bottleneck that adding more clients runs into. Per-prefix request limits and per-connection bandwidth limits exist because each request hits a particular shard, but spreading across more shards and more clients gives essentially linear aggregate scaling.

**The compute side usually scales horizontally anyway.** Large training runs, big Spark jobs, distributed simulations already use tens to thousands of nodes for compute reasons — more GPUs, more cores, more RAM than fit in any single box. Each node has its own NIC, so aggregate ingest bandwidth scales linearly with cluster size essentially for free. A 1000-node cluster with 100 Gbps NICs has 100 Tbps of theoretical aggregate ingest capacity. At that scale, real limits become account quotas, request distribution, S3 prefix layout, cross-AZ or cross-region paths, retries, cost, and the ability of the compute side to consume what the storage side can deliver.

**Vertical scaling has hard ceilings and worsening price-per-unit.** The largest instances cap at hundreds of Gbps, and the price per Gbps gets worse as you climb. Two mid-tier instances usually give you more aggregate bandwidth at lower cost than one giant one.

**Horizontal architectures are more resilient.** A node failure degrades performance proportionally rather than killing the job entirely.

Vertical scaling earns its keep in narrower cases: single-node ML jobs that fit in one machine and don't want distributed-systems complexity, latency-sensitive workloads where coordination overhead would hurt, or jobs with awkward partitioning where sharding makes the algorithm worse. But for the typical TB-scale-data, kernel-shaped pattern, the answer is "scale out, not up."

The clean way to think about it: pick an instance size where compute and NIC are well-matched (so neither sits idle), then scale horizontally to whatever total throughput you need. AWS roughly designs instance types so this balance is automatic — bigger instances get proportionally bigger NICs — so the choice mostly comes down to operational considerations rather than performance ones.

## The local NVMe tier

S3 isn't a single tier; it's the slowest, largest layer in a hierarchy that includes local NVMe storage, RAM, and CPU caches. Using all of them well is often the difference between a pipeline that runs fast and one that's merely correct.

A quick clarification, since terminology is muddled. **SSD** (solid state drive) is the broad category — any flash-based storage. **NVMe** (Non-Volatile Memory Express) is a *protocol* for talking to flash storage over the PCIe bus, much faster than the older SATA protocol that was originally designed for spinning disks. **Local NVMe** means an NVMe drive physically attached to the machine, as opposed to network-attached storage (EBS on AWS, which goes over the network and adds latency even though it's also flash underneath). The performance gap is large: a single modern local NVMe drive does ~7 GB/s sequential reads at sub-100-µs latency; a small array of them does 30+ GB/s. Beefy AWS instances ship with serious local NVMe — a `p5.48xlarge` has eight 3.84 TB drives, ~30 TB total at 80+ GB/s aggregate.

Local NVMe is ephemeral on cloud instances — wiped when the instance stops — but for the staging pattern this doesn't matter, because the source of truth remains S3.

When does local NVMe help? Two cases stand out:

**Repeated passes over a working set.** If your job iterates over the same data multiple times — most ML training does, walking through the dataset for many epochs — staging once from S3 to local NVMe and reading from NVMe thereafter is dramatically faster. S3 might give you 10 GB/s; local NVMe gives you 30+. Over 100 epochs, that's 100x less S3 traffic and substantial wall-clock savings.

**Working sets that fit local but not RAM.** If your data is too big for RAM (hundreds of GB or more) but fits comfortably on local NVMe (multiple TB), NVMe acts as a fast intermediate tier. The kernel reads from RAM, RAM is fed from NVMe, NVMe is fed from S3 once at the start. Each tier hides the latency of the one below it via the same pipelined-prefetching pattern, just at different scales.

When does it not help? Single-pass jobs where each byte is read exactly once — there's nothing to cache, so paying to stage through NVMe just adds a step. The stream-from-S3-directly approach wins.

The general principle: every level of the memory hierarchy (CPU caches, RAM, local NVMe, network-attached storage, S3) is faster and smaller than the one below it. The same hierarchical-tiling, prefetching, and overlap principles apply at every level. You're not solving a fundamentally new problem; you're applying old machinery at a new scale.

## What enables it in practice

Several pieces of infrastructure have to cooperate for the pattern to work:

**Nonblocking network I/O and request concurrency** — usually an event-loop HTTP client, async/await, futures, callbacks, or a bounded worker pool. The important property is not a specific OS API; it is the ability to keep many HTTP requests in flight without dedicating one heavyweight thread to each request. Under the hood this is usually epoll, kqueue, IOCP, or a runtime-specific networking layer.

**Chunked, range-readable file formats** — Parquet, ORC, Zarr, WebDataset, MosaicML MDS, TFRecord. These let you fetch independent pieces in parallel rather than interpreting one compressed byte stream from the beginning. S3 supports arbitrary range GETs, but the file format must make those ranges meaningful: splittable compression, row groups, tiles, shards, indexes, or offsets. Columnar formats additionally let you skip irrelevant columns entirely, reducing bytes-fetched dramatically for selective queries.

**A tuned S3 client** that does range-GET splitting on large objects, manages connection pools, retries failed requests, and uses HTTP/2 multiplexing or many parallel HTTP/1.1 connections. Default SDK clients are often fine for correctness and modest throughput, but they are frequently under-concurrent for line-rate scans. Specialized clients like AWS CRT, s5cmd, or Rust's `object_store` crate often do meaningfully better, especially at high throughput.

**A bounded queue** between IO and compute, sized via the principles above — enough to absorb variance, capped to prevent OOM. In-process, in-memory, microsecond overhead per operation. *Not* Redis or Kafka — those solve a different problem (cross-process or cross-service messaging with durability) and would add network round-trips inside what should be the fastest part of the system.

**Optionally, a local cache tier** if working-set characteristics warrant it, with its own pipelined fill from S3.

The combination of these — async I/O, chunked formats, fast clients, bounded in-memory queues — is what most modern data-loading frameworks (PyTorch DataLoader, NVIDIA DALI, Ray Data, Spark's Parquet readers) implement under the hood. Understanding the underlying pattern lets you debug and tune them when defaults aren't enough.

## Failure modes

The pattern fails — or gets you wrong answers, or quietly underperforms — in several recognizable ways. Knowing the failure modes is most of what separates engineers who can make these systems fast from engineers who copy a config and hope.

**Random access patterns.** Pipelined prefetching works because future accesses are predictable — the kernel will want chunk N+1 after chunk N, so you can fetch ahead. If access is genuinely random (point lookups into a TB-scale dataset), there's nothing to prefetch, latency dominates each access, and no amount of pipelining helps. The fix is a different storage tier: a key-value store, an indexed database, or a caching layer in front of S3. Don't try to make S3 do random-access work.

**Working set thrashing.** If the working set is larger than aggregate cluster RAM+NVMe and gets touched repeatedly, every pass refetches from S3, and you've hit S3 bandwidth as a hard ceiling. The fix isn't more pipelining — it's algorithmic. Either restructure to do more work per byte fetched (raise arithmetic intensity, the same lever roofline analysis points at), restructure to need only a single pass, or restructure to operate on a smaller subset.

**Trivially fast kernels.** If the kernel does almost no work per byte — counting rows, summing a column, simple format conversion — you're IO-bound by definition, regardless of how good your pipeline is. Your throughput ceiling is S3 aggregate bandwidth, period. This might still be fast in absolute terms (tens of GB/s on a cluster), but the kernel's compute capacity is fundamentally wasted, and no amount of prefetching changes that. Recognize this regime; don't waste time optimizing the IO path further when the IO is already saturated.

**Variable chunk sizes.** A queue bounded in *count* may not be bounded in *bytes* if chunk sizes vary widely. A Parquet row group might unexpectedly be 10x normal size; an image dataset might have one outlier of unusual dimensions. Bound by bytes when this matters.

**Decompression amplification.** A 10 MB compressed chunk might be 100 MB after decompression. If you size buffers based on compressed size but hold decompressed data, you can be off by 10x on actual memory use. Either size based on decompressed size, or stream decompression so the full decompressed chunk never exists at once.

**Multiple pipelines in one process.** If a single Python process runs several data loaders (one per GPU, say), each with its own queue, the per-pipeline limits multiply. Aggregate process memory can be many times what a glance at one queue suggests. Track total, not per-pipeline.

**Framework defaults that don't fit your workload.** PyTorch DataLoader, Ray Data, Spark — all have configurable buffer sizes with defaults assuming modest chunks. If you're working with large images, long sequences, or custom decoders, the defaults can be far too generous (OOM) or far too stingy (stalls). Read the configuration knobs; don't assume defaults are right.

**Memory leaks in the IO path.** If chunks aren't released after the consumer finishes — a stray reference held in a logging callback, a circular reference in a GC'd language, a debug accumulator that nobody removed — memory grows even with bounded queues. Profilers and heap dumps catch this, but the failure mode is insidious because nothing is structurally wrong.

## The principle, in one paragraph

You cannot make a single S3 fetch fast — that latency is set by physics and you don't get to negotiate. But you can keep enough fetches in flight that *completions arrive continuously*, at a rate set by concurrency rather than by individual latency. Run your kernel against a bounded queue fed by these completions, and the kernel sees a steady stream of in-RAM data while the IO subsystem absorbs all the latency in the background. Little's Law tells you exactly how much concurrency you need; bounded queues with backpressure keep the system from blowing up; horizontal scaling lets you grow the pattern across a cluster; local NVMe acts as a fast intermediate tier when working sets warrant it. This is the same trick CPUs use to hide DRAM latency, GPUs use to hide memory latency with warps, TCP uses to fill long-fat pipes, and database engines use to prefetch pages — applied to whichever tier of the memory hierarchy is currently the slow one. The pattern is older than computing; only the slow tier keeps changing. Today it's S3.
