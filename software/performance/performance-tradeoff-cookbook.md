# Performance Tradeoff Cookbook

Tunable parameters are rarely pure improvements. They usually move pressure from one resource, percentile, failure mode, or workload shape to another. A larger value may improve throughput while worsening latency. A smaller value may reduce memory while increasing CPU. A safer setting may cost money. A faster setting may reduce debuggability or recovery margin.

Use this cookbook when changing system parameters. The point is not to memorize the “right” values. The point is to recognize the tradeoff each knob encodes.

A good tuning note should say:

```text
We changed X from A to B because workload shape W is dominated by bottleneck Y.
We expect metric M to improve and metric N to get worse or consume more headroom.
We will keep the change only if production-like measurement confirms that tradeoff.
```

---

## Core principles

### 1. A knob usually favors a workload shape

Most settings encode a preference:

- small batches favor latency;
- large batches favor throughput;
- small blocks favor random access;
- large blocks favor streaming;
- more concurrency favors latency hiding;
- less concurrency favors memory, fairness, and stability.

Do not ask “is bigger better?” Ask “bigger is better for which workload?”

### 2. Tuning moves the bottleneck

If you increase I/O concurrency, the bottleneck may move from request latency to network bandwidth, then to decompression CPU, then to memory bandwidth. That is success, not failure. Re-measure after each major change.

### 3. Defaults are policies, not truths

Defaults usually optimize for safety, broad compatibility, and moderate resource use. They are often wrong for extreme workloads: high-throughput scans, low-latency services, huge working sets, or GPU pipelines.

### 4. Bound everything that can grow

Queues, buffers, retries, caches, connection pools, thread pools, logs, and metrics cardinality all need bounds. Unbounded systems fail by turning a transient mismatch into memory exhaustion, retry storms, or cost explosions.

### 5. Tune against the objective, not the average

For services, p99 may matter more than mean latency. For batch, total cost or wall-clock may matter more than p99. For real-time systems, deadline misses matter more than throughput.

### 6. Prefer removing work over making unnecessary work cheaper

The largest performance wins usually come from the highest layer that can avoid work entirely. An application change might remove a database query, skip serialization, batch remote calls, avoid an allocation-heavy path, or use a better algorithm. A storage-device tuning may make I/O faster, but if the application still performs unnecessary queries, most of the cost has already been accepted higher in the stack.

Tune as high as possible; observe as low as necessary.

---

## Where to tune vs where to observe

Performance tuning is usually most effective closest to where the work is created. For application-driven workloads, that often means the application itself: change the algorithm, reduce queries, avoid requests, batch operations, or stop moving data that does not need to move.

That does not mean observation should only happen at the application layer. The best evidence often comes from lower layers: CPU profiles, system-call counts, filesystem statistics, block-device latency, network retransmits, database execution plans, lock waits, and cache-miss counters. A slow query may be caused by application behavior, but understood through database and operating-system measurements.

The rule:

```text
Tune where work can be eliminated or reshaped.
Observe wherever the cost becomes visible.
```

| Layer | What tuning can eliminate or reduce | Typical win shape | Common tradeoff |
|---|---|---|---|
| **Product / workload** | Features, requests, freshness requirements, consistency requirements, data retained | Can be enormous because the work disappears | Requires product or business decision |
| **Application** | Algorithms, database queries, RPCs, serialization, allocations, duplicate work, request batching | Often the largest engineering win | Requires code change and correctness testing |
| **Query / database** | Scans, joins, locks, indexes, materialization, transaction scope | Large when data access is the bottleneck | Storage, write amplification, freshness, migration cost |
| **Runtime** | GC pressure, allocation overhead, thread scheduling, async behavior, JIT behavior | Medium to large when runtime overhead is visible | Memory footprint, complexity, pause behavior |
| **System-call / I/O API** | Syscall count, synchronous waits, copying, small reads/writes | Medium when boundary crossing or blocking dominates | More complex flow control and error handling |
| **Filesystem** | Record size, journaling behavior, cache behavior, readahead, metadata overhead | Often workload-specific; large for I/O-heavy systems | May improve scans while hurting random I/O, or vice versa |
| **Storage device** | Queue depth, RAID layout, disk type, device cache, IOPS/bandwidth limits | Often percentage wins unless storage is the dominant bottleneck | Cost, durability semantics, operational complexity |
| **Network** | Buffer sizes, batching, compression, connection reuse, routing, placement | Large for chatty or bandwidth-heavy systems | Memory, CPU, locality, fairness, cross-zone cost |
| **Kernel / hardware** | Scheduler behavior, NUMA placement, huge pages, interrupts, CPU frequency, kernel bypass | Important near hardware limits | Portability, complexity, isolation, debuggability |

This table is not an argument against low-level tuning. Low-level tuning is essential when the measured bottleneck is low-level. The warning is about leverage: if a higher layer can avoid work, that usually beats optimizing the lower layer that executes it.

Examples:

- Removing an unnecessary query can beat tuning the database buffer cache.
- Returning fewer columns can beat increasing network buffers.
- Avoiding JSON serialization can beat optimizing syscalls.
- Changing a full scan into an indexed lookup can beat upgrading disks.
- Reducing fanout can beat tuning downstream connection pools.
- Doing one pass over data can beat caching repeated passes.

Use lower-layer tuning when one of these is true:

- the higher layer is already doing the necessary minimum work;
- the workload contract prevents changing the higher-level behavior;
- the measured bottleneck is genuinely in the lower layer;
- the lower-layer change is simpler, safer, or more reversible than an application change;
- the same lower-layer improvement benefits many workloads at once.

Avoid lower-layer tuning when it only makes waste faster. If the application reads 100 columns to use 3, the first fix is projection discipline, not a faster disk. If a service makes 20 serial RPCs it does not need, the first fix is request shape, not TCP buffers. If a job scans the same data 50 times, the first fix is scan sharing, not more workers.

---

## Glossary and terminology

Use these terms consistently when discussing tradeoffs.

### Measurement terms

| Term | Meaning |
|---|---|
| **Latency** | Time for one unit of work to complete. For a request, this is usually time from send to response. For storage, it may be time from issuing an I/O to first byte or completion. |
| **Throughput** | Work completed per unit time: requests/sec, rows/sec, bytes/sec, tokens/sec, jobs/hour. |
| **Bandwidth** | Data throughput, usually bytes/sec or bits/sec. Network bandwidth and memory bandwidth are both throughput measures. |
| **IOPS** | I/O operations per second. Important for small random I/O where operation count matters more than bytes/sec. |
| **Utilization** | Fraction of a resource's capacity currently in use. 90% CPU utilization means the CPUs are busy 90% of the time. |
| **Saturation** | A resource has more work queued than it can immediately serve. Saturation is stronger than utilization: a disk at 90% utilization may be fine; a disk with persistent queueing is saturated. |
| **Headroom** | Spare capacity before saturation. Low headroom often means better cost efficiency but worse burst tolerance and tail latency. |
| **p50 / p95 / p99 / p999** | Percentiles of latency or duration. p99 means 99% of observations are at or below that value and 1% are worse. |
| **Tail latency** | High-percentile latency. In user-facing systems, p99 can matter more than the average. |
| **Jitter** | Variation in latency over time. Low jitter means more predictable latency. |
| **Wall-clock time** | Real elapsed time from start to finish, as opposed to CPU time summed across threads. |
| **CPU time** | Time spent executing on CPU cores. Ten threads running for one second each consume roughly ten CPU-seconds. |

### Queueing and concurrency terms

| Term | Meaning |
|---|---|
| **Concurrency** | Number of units of work active at the same time. This can mean requests in flight, threads running, tasks scheduled, or I/O operations outstanding. |
| **Parallelism** | Work actually executing simultaneously on multiple cores, devices, or machines. Concurrency can exist without parallelism. |
| **In-flight work** | Work that has been issued but has not completed. In-flight requests are consuming capacity somewhere even if the caller is waiting asynchronously. |
| **Queue depth** | Number of items waiting or allowed to wait in a queue. Deeper queues can hide latency but increase memory and waiting time. |
| **Backpressure** | A mechanism that slows producers when consumers or downstream systems cannot keep up. Bounded queues are a common backpressure mechanism. |
| **Admission control** | Rejecting or delaying new work before it enters the system to prevent overload collapse. |
| **Head-of-line blocking** | A slow item blocks later items behind it, even if those later items could otherwise complete quickly. |
| **Little's Law** | Relationship between concurrency, throughput, and latency: `concurrency = throughput × latency`. For byte streams: `concurrency = target_byte_throughput × latency / bytes_per_request`. |
| **Open-loop load** | Requests arrive on an external schedule, independent of response time. This reveals overload and tail latency more honestly. |
| **Closed-loop load** | A client waits for a response before sending the next request. This can hide overload because slow responses reduce offered load. |

### Data movement terms

| Term | Meaning |
|---|---|
| **Batch** | A group of logical work items processed together to amortize overhead. Larger batches usually improve throughput and hurt latency. |
| **Chunk / block / record / page** | A physical unit of data movement or storage. Naming varies by layer: filesystems have blocks, databases have pages, columnar formats have row groups/pages, object stores often use chunks or ranges. |
| **Working set** | The data actively touched over a time window. Performance changes sharply when the working set no longer fits in cache, memory, local disk, or another tier. |
| **Cache hit** | Requested data was found in a faster tier. |
| **Cache miss** | Requested data was not found in the faster tier and must be fetched from a slower tier. |
| **Hit rate** | Fraction of accesses served from cache. A high hit rate matters only if misses are expensive enough. |
| **Eviction** | Removing data from cache to make room for other data. Bad eviction policy can make a cache harmful. |
| **TTL** | Time to live: how long cached data is considered valid. Longer TTLs reduce load but increase staleness. |
| **Freshness** | How up-to-date a value is relative to the source of truth. |
| **Staleness** | How old a value may be while still being served. This is a correctness/product requirement, not just a performance detail. |
| **Prefetch** | Fetching data before it is demanded. Works when future accesses are predictable. |
| **Readahead** | Storage-level prefetch for sequential reads. Helps scans; wastes work on random access. |
| **Read amplification** | Reading more physical data than the logical query needs. Example: reading a 4 MB block to use 4 KB. |
| **Write amplification** | Writing more physical data than the logical update implies. Common with indexes, replication, compaction, and copy-on-write formats. |
| **Metadata overhead** | Cost of listing, planning, opening, stat-ing, scheduling, or tracking many small objects/files/partitions rather than processing payload bytes. |
| **Locality** | How close needed data is in time, address space, topology, or storage layout. Better locality usually means fewer expensive misses or remote accesses. |

### Storage and durability terms

| Term | Meaning |
|---|---|
| **Sequential I/O** | Reading or writing adjacent data in order. Usually maximizes bandwidth. |
| **Random I/O** | Reading or writing scattered locations. Usually limited by latency or IOPS. |
| **fsync** | Operation that forces buffered writes to durable storage. Improves durability but can dominate write latency. |
| **WAL** | Write-ahead log: durable log written before applying changes to main data structures. Used for crash recovery. |
| **Checkpoint** | A durable point-in-time state that reduces how much log or work must be replayed after failure. |
| **Compaction** | Rewriting data into a more efficient layout, often to remove obsolete versions and reduce read amplification. Common in LSM stores and columnar lakes. |
| **Replication factor** | Number of copies of data. Higher replication improves availability and read locality but increases write and storage cost. |
| **Quorum** | Minimum number of replicas that must acknowledge an operation. Larger quorums improve consistency/durability properties but raise latency and reduce availability during failures. |
| **RPO** | Recovery point objective: how much data loss is acceptable after a failure. |
| **RTO** | Recovery time objective: how long recovery is allowed to take. |

### Network and distributed-system terms

| Term | Meaning |
|---|---|
| **RTT** | Round-trip time between two endpoints. Determines the latency floor for request/response protocols. |
| **Bandwidth-delay product** | Amount of data needed in flight to fill a link: `bandwidth × RTT`. TCP windows and request concurrency must be large enough to fill high-latency links. |
| **Fanout** | One request creates many downstream requests. Fanout can reduce median latency through parallelism while worsening tail latency because the slowest dependency dominates. |
| **Retry amplification** | Retries increase total load, often exactly when the system is already unhealthy. |
| **Retry budget** | A cap on retry volume so retries cannot overwhelm the system. |
| **Hedged request** | A duplicate request sent after a delay; the first successful response wins. Can reduce tail latency at the cost of extra load. |
| **Circuit breaker** | Mechanism that stops calling a failing dependency for a period, usually serving errors or fallbacks instead. |
| **Rate limit** | Cap on accepted request rate. Protects systems from overload but can reject legitimate work. |
| **Hot shard / hot partition / hot key** | A partition receives disproportionate traffic, limiting scale despite high aggregate capacity elsewhere. |
| **Cross-zone / cross-region traffic** | Network traffic crossing availability-zone or region boundaries. Often slower and more expensive than local traffic. |

### Runtime and hardware terms

| Term | Meaning |
|---|---|
| **Context switch** | CPU switches from one thread/process to another. Too many context switches waste CPU and harm cache locality. |
| **Syscall** | User code enters the kernel. Necessary for I/O and OS services, but high syscall rates can dominate small operations. |
| **GC pause** | Garbage collector pauses application work. Even short pauses can dominate p99 latency. |
| **Allocation rate** | How quickly a program allocates memory. High allocation rates create allocator and GC pressure. |
| **RSS** | Resident set size: physical memory currently held by a process. |
| **NUMA** | Non-uniform memory access. On multi-socket machines, memory is faster from the local socket than from a remote socket. |
| **TLB** | Translation lookaside buffer: cache for virtual-to-physical address translations. TLB misses can dominate large random working sets. |
| **Huge pages** | Larger memory pages, often 2 MB or 1 GB instead of 4 KB. They increase TLB reach but can complicate memory management and tail latency. |
| **Polling** | Repeatedly checking for work. Low latency under load but burns CPU when idle. |
| **Interrupts** | Device or kernel notifies CPU when work arrives. Efficient when idle, but can add latency and overhead at very high rates. |
| **Kernel bypass** | Avoiding normal kernel networking/storage paths for extreme throughput or latency. Gains performance at the cost of complexity and lost OS services. |
| **SIMD** | Single instruction, multiple data. One CPU instruction operates on multiple values. Helps regular compute-heavy loops, not memory-bound loops. |

### Observability terms

| Term | Meaning |
|---|---|
| **Metric cardinality** | Number of distinct time series created by label combinations. High cardinality improves slicing but can overload metrics systems. |
| **Sampling** | Keeping only some events/traces/logs. Reduces cost but can hide rare failures. |
| **Histogram bucket** | A boundary used to count observations into ranges. More buckets improve percentile accuracy but increase metric volume. |
| **Retention** | How long telemetry or data is kept. Longer retention helps investigations and trend analysis but costs storage. |
| **SLO** | Service-level objective: target reliability or performance level users can expect. |
| **Error budget** | Amount of allowed unreliability under an SLO. It converts reliability into an explicit tradeoff against release speed or risk. |

---

## Tier 1: Universal knobs

These appear in almost every system.

| Knob | Smaller / lower favors | Larger / higher favors | Watch for | Measure |
|---|---|---|---|---|
| **Batch size** | Lower latency, faster first result, lower memory | Higher throughput, better amortization of fixed overhead | Head-of-line blocking, bursty memory, worse p99 | Throughput, p50/p99 latency, memory per worker |
| **Chunk / block / record size** | Random I/O, selective reads, cache efficiency | Sequential scans, compression, fewer metadata operations | Wasted reads for point lookups; too many requests if too small | Bytes read per useful byte, IOPS, scan rate, cache hit rate |
| **Buffer size** | Lower memory per connection/task, more scalable fanout | Higher per-flow throughput, fewer stalls | Per-connection memory explosion, longer queueing | Memory per connection, throughput, retransmits/stalls |
| **Queue depth** | Lower memory, lower waiting time, less tail amplification | Latency hiding, smoother producer/consumer mismatch | OOM, stale queued work, hidden overload | Queue length, age of oldest item, drops, p99 latency |
| **In-flight requests** | Lower memory, less downstream pressure | Higher bandwidth, latency hiding | Thundering herd, retries, downstream saturation | Request rate, concurrency, error rate, saturation, p99 |
| **Worker / thread count** | Less contention, less memory, less scheduling overhead | More parallelism, better CPU or I/O utilization | Lock contention, cache thrash, context switches, memory bandwidth saturation | CPU utilization, run queue, context switches, throughput scaling curve |
| **Connection pool size** | Protects databases and downstreams, lower memory | More concurrent work, better utilization | Overwhelming downstream, idle connection overhead | Pool wait time, active connections, downstream CPU/locks/p99 |
| **Cache size** | Lower memory/cost, less eviction impact on other work | Higher hit rate, lower backend load | Stale data, memory pressure, poor eviction behavior | Hit rate, miss penalty, memory use, eviction rate |
| **Cache TTL** | Freshness, correctness, faster invalidation | Fewer backend calls, lower latency | Stale reads, synchronized expiry storms | Staleness, backend QPS, hit rate, error rate after deploys |
| **Compression level** | Lower CPU, lower latency | Fewer bytes, lower network/storage cost | CPU saturation, decompression amplification | CPU time, compressed size, wall time, network bytes |
| **Timeout duration** | Faster failure, fewer held resources | Fewer false timeouts, better success under transient slowness | Premature failure or resource pileup | Timeout rate, request duration distribution, retry rate |
| **Retry count / retry budget** | Less load amplification, faster surfacing of faults | More resilience to transient failures | Retry storms, duplicate work, tail amplification | Attempts per request, downstream QPS, success after retry, p99 |
| **Prefetch distance** | Less wasted work and cache pollution | Better latency hiding | Fetching unused data, memory blowup | Stall time, cache hit rate, queue fill level, wasted fetched bytes |
| **Admission limit** | Stable latency, protected dependencies | More accepted work, higher peak throughput | User-visible rejection vs overload collapse | Rejection rate, queue length, p99, downstream saturation |

---

## Tier 2: Storage and data-system knobs

These matter when the workload moves large volumes of data, serves mixed read/write traffic, or maintains durable state.

| Knob | Smaller / lower favors | Larger / higher favors | Watch for | Measure |
|---|---|---|---|---|
| **File or object size** | Fine-grained skipping, faster rewrites, easier parallelism | Streaming throughput, fewer requests, less metadata | Small-file metadata tax; large-file rewrite cost | Files per partition, request count, scan throughput, planning time |
| **File count / partition count** | Lower metadata overhead, simpler planning | More parallelism, selective reads, easier incremental writes | Too many tiny files or too little parallelism | Scheduler time, request rate, bytes skipped, task skew |
| **Filesystem block / record size** | Random I/O, cache efficiency for small reads | Sequential throughput, backup throughput | Wasted cache and read amplification | IOPS, read amplification, cache hit rate, sequential bandwidth |
| **Database page size** | Point lookups, less wasted cache per row | Range scans, fewer page reads, better sequential access | Poor fit for mixed workloads | Buffer-pool hit rate, rows per page, read amplification |
| **Parquet row-group size** | Predicate skipping, lower memory per group | Compression, scan throughput, fewer metadata operations | Poor skipping if too large; request overhead if too small | Row groups skipped, bytes read, scan throughput, memory per task |
| **Column chunk / page size** | Fine-grained filtering and decoding | Compression and sequential decode throughput | Metadata overhead or wasted reads | Page skips, decode CPU, bytes touched |
| **Index count** | Faster writes, less storage, less maintenance | Faster reads, more access paths | Write amplification, planner complexity, stale unused indexes | Read latency, write latency, index size, index usage |
| **Index granularity** | Smaller index, lower write cost | More precise pruning and lookup | False positives vs index bloat | Rows scanned after index, index memory, update cost |
| **Materialized views** | Lower storage, simpler freshness | Fast reads, predictable query latency | Staleness, rebuild cost, invalidation complexity | Query latency, refresh lag, storage overhead, rebuild duration |
| **Replication factor** | Lower write cost, lower storage | Availability, read locality, failover safety | Write amplification, consistency lag | Write latency, replica lag, read locality, failover time |
| **Consistency level / quorum size** | Lower latency, higher availability during failures | Stronger read/write guarantees | Stale reads or reduced availability | Read/write latency, stale-read rate, failed quorum rate |
| **WAL/fsync frequency** | Higher throughput, lower write latency | Stronger durability, smaller data-loss window | Lost acknowledged writes after crash | fsync latency, commits/sec, recovery point objective |
| **Checkpoint interval** | Faster recovery, bounded replay | Higher steady-state throughput | Long recovery or checkpoint interference | Recovery time, checkpoint duration, write stalls |
| **LSM compaction aggressiveness** | Higher write throughput now | Better reads, lower read amplification, space cleanup | Background I/O storms, write stalls | Read amplification, write amplification, space amplification |
| **Compaction target file size** | Faster compactions, finer skipping | Fewer files, better scan throughput | Small-file buildup or expensive rewrites | File count, compaction backlog, query planning time |
| **Local NVMe cache size** | Lower cost, simpler statelessness | Faster repeated reads, less remote I/O | Cache invalidation, warmup time, ephemeral loss | Hit rate, warmup time, remote bytes avoided |
| **Object-store request concurrency** | Lower memory, lower request cost, fewer bursts | Higher throughput, hides request latency | Client throttling, prefix hot spots, memory pressure | In-flight GETs, bytes/sec, p99 GET latency, error/retry rate |
| **Object-store chunk size** | More parallelism, better retry granularity | Better per-request efficiency, fewer requests | Too many tiny range requests or poor skip granularity | Request count, bytes/request, retry cost, throughput |

---

## Tier 3: Networking and distributed-system knobs

These matter when work crosses process, host, zone, region, or service boundaries.

| Knob | Smaller / lower favors | Larger / higher favors | Watch for | Measure |
|---|---|---|---|---|
| **Socket send/receive buffer** | Lower memory per connection | Higher throughput on high-latency links | Per-connection memory blowup | Throughput, retransmits, memory per connection |
| **TCP / protocol window** | Lower memory, faster feedback | Fills long-fat pipes | Bufferbloat, unfairness | Bandwidth-delay product utilization, RTT, drops |
| **Request batch size** | Lower latency, finer failure isolation | Higher throughput, fewer network round trips | Head-of-line blocking, retrying large batches | RPC rate, bytes/RPC, p99, partial failure rate |
| **RPC fanout width** | Lower downstream pressure, lower tail amplification | Lower wall-clock for parallel remote work | p99 grows with dependency count | Fanout count, slowest dependency latency, error rate |
| **Hedging delay** | Lower extra load | Better p99 when stragglers dominate | Duplicate work, overload during incidents | Hedge rate, winner distribution, downstream QPS, p99 |
| **Load-balancing policy** | Simplicity and even spread | Locality, cache warmth, specialized routing | Hot spots, unfairness, stale endpoint state | Per-backend load, cache hit rate, queue time |
| **Shard count** | Less metadata, fewer cross-shard operations | More parallelism, smaller per-shard working sets | Rebalancing cost, tiny shards, cross-shard queries | Per-shard load, hot-shard ratio, rebalance time |
| **Partition key granularity** | Simpler queries, fewer partitions | Better distribution, selective reads | Hot partitions or too many partitions | Skew, partition count, bytes skipped, metadata time |
| **Replica placement spread** | Lower network cost, better locality | Failure isolation, availability | Correlated failure vs cross-zone latency/cost | Cross-zone bytes, failover safety, read latency |
| **Rate limit** | Backend protection, stable latency | More user-visible capacity | Rejections vs overload collapse | Reject rate, p99, backend saturation, error rate |
| **Circuit-breaker threshold** | Faster isolation of failing dependency | Fewer false opens | Premature degradation or cascading failure | Open rate, dependency errors, fallback success, recovery time |
| **Keepalive interval** | Lower background traffic | Faster dead-peer detection | Connection churn or slow failure detection | Dead connection duration, keepalive traffic, reconnects |
| **Serialization format** | Human readability, compatibility | Compactness, speed, schema control | Debuggability vs CPU/bytes | Encode/decode CPU, payload size, schema error rate |
| **Compression on network payloads** | Lower CPU, lower latency for small payloads | Lower bandwidth and egress cost | CPU saturation, added latency on small messages | Payload bytes, CPU per request, p99, egress cost |
| **Cross-region replication lag target** | Lower write cost, simpler operation | Better recovery point, fresher remote reads | Write amplification, cross-region cost | Lag, cross-region bytes, failover data loss |

---

## Tier 4: Runtime, OS, and hardware knobs

These matter when CPU, memory hierarchy, scheduler behavior, runtime pauses, or kernel boundaries dominate.

| Knob | Smaller / lower favors | Larger / higher favors | Watch for | Measure |
|---|---|---|---|---|
| **GC heap size** | Lower memory footprint, sometimes shorter pauses | Fewer collections, higher throughput | Long pauses, memory pressure, container OOM | Allocation rate, GC time %, pause distribution, RSS |
| **GC pause target** | Lower tail latency | Higher throughput and less collector overhead | CPU overhead, reduced allocation throughput | p99/p999, GC CPU, allocation stalls |
| **Allocator arena count** | Lower memory fragmentation | Less allocator lock contention | RSS growth, fragmentation | Allocation latency, lock contention, RSS, fragmentation |
| **Thread stack size** | More threads in same memory | Safer deep call stacks | Stack overflow or wasted memory | Thread count, memory per thread, stack faults |
| **Spin before sleep** | Lower CPU waste | Lower wakeup latency under short waits | Burning cores, power, noisy-neighbor effects | CPU utilization while idle, wakeup latency, lock wait |
| **CPU affinity / pinning** | Scheduler flexibility, easier load balance | Cache locality, lower jitter, NUMA control | Imbalanced cores, operational complexity | Migrations, cache misses, run queue per core, p99 |
| **NUMA interleaving** | Simpler placement, balanced bandwidth | Locality when access is unpredictable | Remote access for localizable workloads | Remote DRAM %, bandwidth per socket, latency |
| **NUMA binding** | Scheduler flexibility | Local memory latency, deterministic placement | Imbalance, bad placement after reschedule | NUMA misses, per-socket CPU/memory, p99 |
| **Huge pages** | Memory flexibility, less fragmentation risk | Fewer TLB misses, better large-working-set throughput | Compaction stalls, internal fragmentation | dTLB misses, page-fault latency, RSS, p99 |
| **Page cache pressure / dirty ratio** | Lower writeback stalls, safer memory headroom | Higher write buffering throughput | Sudden flush storms, memory pressure | Dirty bytes, writeback time, stalls, major faults |
| **Readahead size** | Random access, less cache pollution | Sequential scan throughput | Wasted reads on random workloads | Readahead hits, wasted bytes, scan bandwidth |
| **Polling vs interrupts** | Lower idle CPU, power efficiency | Lower latency, high packet or I/O rate | Dedicated spinning cores, wasted CPU | Interrupt rate, packet latency, CPU idle %, drops |
| **Kernel bypass** | OS integration, safety, simpler ops | Extreme packet/storage throughput and latency | Lost tooling/isolation, dedicated cores | Packets/sec, syscall time, CPU/core, tail latency |
| **Async I/O depth** | Lower memory and simpler flow control | Higher device/network utilization | Harder backpressure, bursty completions | I/O queue depth, device utilization, completion latency |
| **SIMD width / vectorization** | Portability, simpler code, less downclock risk | Higher arithmetic throughput | Memory-bound loops do not improve; code complexity | IPC, vector utilization, bandwidth, frequency |
| **JIT warmup threshold** | Faster startup, less profiling overhead | Better optimized hot code | Cold-start latency, deoptimized paths | Warmup time, steady-state throughput, compile time |

---

## Tier 5: Operations and observability knobs

These matter in production systems where reliability, debuggability, and cost are part of performance.

| Knob | Smaller / lower favors | Larger / higher favors | Watch for | Measure |
|---|---|---|---|---|
| **Autoscaling target utilization** | Headroom, lower latency, resilience | Lower cost, higher utilization | Paying for idle or scaling too late | CPU/memory utilization, p99, cost, scale events |
| **Autoscaling cooldown** | Faster adaptation | Stability, less oscillation | Thrashing or slow response | Scale frequency, pending work, p99 during bursts |
| **Minimum replica count** | Lower baseline cost | Warm capacity, failure tolerance | Cold starts vs idle spend | Cold-start latency, cost, failover behavior |
| **Deployment batch size** | Smaller blast radius, easier rollback | Faster rollout, less operational overhead | Slow deploys or large incidents | Rollout duration, error rate during deploy, rollback time |
| **Health-check interval** | Lower overhead | Faster failure detection | False positives, unnecessary restarts | Detection time, check load, restart rate |
| **Health-check timeout** | Faster removal of bad instances | Fewer false removals during transient slowness | Flapping or slow failure isolation | Failed checks, false positives, user errors |
| **Log verbosity** | Lower cost, less noise, less I/O | Debuggability, incident forensics | Missing evidence or log storms | Log volume, ingestion cost, useful events per incident |
| **Metric cardinality** | TSDB stability and cost | Debug precision, per-tenant visibility | Cardinality explosions or blind aggregation | Series count, query latency, storage cost |
| **Trace sampling rate** | Lower cost and overhead | Better forensics and rare-event capture | Missing critical traces or too much cost | Sampled traces/sec, storage, incident usefulness |
| **Tail-based sampling buffer** | Lower memory and latency | Better decision quality for trace retention | Dropping traces before knowing they matter | Buffer occupancy, late decision drops, retained error traces |
| **Histogram bucket count** | Lower storage/cardinality | More accurate percentile/SLO analysis | Coarse p99 or metric bloat | Bucket series count, quantile error, query cost |
| **Retention period** | Lower storage cost | Longer trend analysis and incident history | Losing needed history or keeping expensive noise | Storage cost, query use by age, compliance needs |
| **Alert threshold sensitivity** | Fewer pages, less fatigue | Faster detection | Missed incidents or noisy alerts | Page count, false positives, time to detect |
| **SLO window length** | Faster feedback | Stability, less noise | Flappy alerts or slow burn detection | Burn rate, alert duration, incident correlation |

---

## Tier 6: ML, GPU, and data-loader knobs

These matter when accelerators, model quality, and input pipelines interact.

| Knob | Smaller / lower favors | Larger / higher favors | Watch for | Measure |
|---|---|---|---|---|
| **Training batch size** | Generalization, lower memory, more frequent updates | GPU utilization, throughput, stable kernels | Quality regression, optimizer changes, OOM | Tokens/sec or samples/sec, loss curve, memory, utilization |
| **Inference batch size** | Lower latency, simpler scheduling | Higher throughput, lower cost per request | p99 latency, head-of-line blocking | p50/p99, throughput, GPU utilization, queue age |
| **Microbatch size** | Pipeline responsiveness, lower activation memory | Better kernel efficiency | Pipeline bubbles, overhead | Bubble %, step time, memory, utilization |
| **Gradient accumulation steps** | More frequent optimizer updates | Larger effective batch without more memory | Longer feedback loop, stale gradients | Step time, convergence, memory, samples/sec |
| **Sequence / context length** | Lower memory, faster inference/training | Better quality/capability for long context | Quadratic attention cost, KV-cache growth | Latency by length, memory/token, quality metrics |
| **Precision** | Numerical stability, simpler debugging | Higher throughput, lower memory | Accuracy loss, overflow/underflow | Throughput, memory, validation quality, numerical errors |
| **Activation checkpointing** | Faster compute, simpler execution | Lower memory, larger models/batches | Extra recompute CPU/GPU time | Memory saved, step time, utilization |
| **Data-loader worker count** | Lower CPU and memory overhead | Better accelerator feeding | CPU contention, duplicated memory, startup overhead | GPU idle %, loader queue, CPU utilization, RSS |
| **Data-loader prefetch batches** | Lower memory | Fewer accelerator stalls | Memory blowup, stale shuffled data | Queue fill, GPU idle %, memory, batch age |
| **Shuffle buffer size** | Lower memory, faster startup | Better randomness/statistical quality | Poor randomness or memory pressure | Sample distribution, training quality, memory |
| **Checkpoint frequency** | Higher training throughput | Less lost work after failure | Expensive pauses or large recovery loss | Checkpoint time, lost work on failure, storage bytes |
| **Distributed all-reduce bucket size** | Earlier overlap with backprop, lower latency per bucket | Better bandwidth efficiency | Poor overlap or delayed gradient sync | Communication overlap %, step time, network utilization |
| **Tensor parallel degree** | Less communication, simpler runtime | Fits larger models, uses more GPUs | All-reduce/all-gather overhead | Step latency, communication time, memory per GPU |
| **Pipeline parallel depth** | Lower scheduling complexity | Fits larger models, uses more GPUs | Pipeline bubbles, complex balancing | Bubble %, stage imbalance, step latency |
| **KV-cache size / retention** | Lower memory, more concurrent sessions | Longer context reuse, faster decoding | Memory fragmentation, eviction quality | Tokens/sec, memory/session, eviction rate, p99 |

---

## Common tradeoff patterns

### Latency vs throughput

Examples:

- batch size
- microbatch interval
- request batching
- compression level
- queue depth
- inference batch size

The common failure is optimizing throughput with large batches and then discovering that p99 latency is unacceptable.

### Memory vs speed

Examples:

- cache size
- prefetch buffers
- queue depth
- connection pools
- data-loader workers
- huge pages

The common failure is hiding latency by storing more in memory until the system becomes fragile under bursts.

### Locality vs fairness

Examples:

- CPU pinning
- NUMA binding
- sticky load balancing
- shard affinity
- cache-aware routing

The common failure is improving one worker's locality while creating hot spots or starving others.

### Freshness vs load reduction

Examples:

- cache TTL
- materialized views
- async replication
- batch ETL interval
- metric scrape interval

The common failure is treating stale data as a performance detail when it is actually a product or correctness question.

### Durability vs write throughput

Examples:

- fsync frequency
- replication factor
- quorum size
- checkpoint interval
- commit batching

The common failure is improving benchmark throughput by silently increasing the amount of data that can be lost or replayed after a crash.

### Tail latency vs resource efficiency

Examples:

- autoscaling target utilization
- admission limits
- GC pause targets
- hedged requests
- overprovisioning
- worker isolation

The common failure is running everything near saturation and then being surprised that p99 explodes.

### Simplicity vs peak performance

Examples:

- kernel bypass
- lock-free algorithms
- manual SIMD
- custom allocators
- explicit NUMA placement
- hand-rolled caching layers

The common failure is paying permanent complexity for a bottleneck that was never measured or no longer matters.

---

## How to use this cookbook during a change

Before changing a knob, write down:

1. **Workload shape.** Is it latency-sensitive, throughput-oriented, batch, interactive, random-access, streaming, read-heavy, write-heavy, or coordination-heavy?
2. **Current bottleneck.** What measurement says this knob is relevant?
3. **Expected win.** Which metric should improve?
4. **Expected cost.** Which metric might worsen?
5. **Safety bound.** What prevents runaway memory, retries, cost, or queueing?
6. **Rollback condition.** What observation means the tuning was wrong?

Example:

```text
Change: Increase S3 range-read concurrency from 16 to 64.
Workload: Batch scan, sequential range reads, CPU has idle time waiting on input.
Expected win: Higher read throughput and lower GPU idle time.
Expected cost: More memory in in-flight buffers, higher request rate, possible throttling.
Safety bound: Queue capped by bytes, retry budget capped, per-worker memory alert.
Keep if: GPU idle time falls and S3 errors/retries do not rise materially.
Rollback if: RSS or retry rate grows, or p99 GET latency worsens enough to offset throughput.
```

---

## Short checklist

When tuning, ask:

- Does this favor latency or throughput?
- Does it favor random access or streaming?
- Does it consume more memory to save time?
- Does it consume more CPU to save bytes?
- Does it reduce load by accepting staleness?
- Does it increase success rate by risking load amplification?
- Does it improve average performance while hurting p99?
- Does it improve one tenant, shard, or connection while hurting fairness?
- Does it improve steady state while making recovery worse?
- Is the setting bounded, observable, and reversible?

If you cannot name both sides of the tradeoff, you probably do not understand the knob yet.
