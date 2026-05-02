# Computational Work Taxonomy for Performance Engineering

Performance engineering does not have one clean taxonomy of computational work. That is not a failure of terminology; it reflects the problem. Performance depends on the interaction between the workload, algorithm, implementation, runtime, hardware, data layout, operating environment, and objective being optimized.

The same program can be compute-bound on one machine, memory-bound on another, network-bound at scale, and tail-latency-bound in production. A useful taxonomy is therefore not a single tree. It is a set of complementary lenses.

The working question is:

> What kind of work is this, what shape does it impose on the machine or system, and what limit is actually binding?

Use the lenses below together.

---

## 1. Workload contract

The workload contract describes what the system must deliver. It is the first classification because it determines what “performance” means.

Important axes:

- **Latency-sensitive vs throughput-oriented.** A trading path, checkout request, or autocomplete endpoint cares about response time. A batch ETL job, video transcode, or offline validation job cares mostly about total throughput and cost.
- **Interactive vs batch.** Interactive systems expose humans to latency and variance. Batch systems can often trade latency for throughput, utilization, and cost efficiency.
- **Deadline-based vs best-effort.** Real-time does not mean “fast”; it means deadline-constrained. A system is real-time if it predictably meets deadlines.
- **Open-loop vs closed-loop load.** Open-loop traffic arrives independently of system response. Closed-loop clients wait for a response before issuing more work. Closed-loop tests often hide tail latency.
- **Steady vs bursty.** A steady workload can be provisioned near average demand. A bursty workload needs buffering, autoscaling headroom, admission control, or graceful degradation.
- **Homogeneous vs heterogeneous requests.** Uniform requests are easier to batch and balance. Mixed request sizes produce queueing effects and tail amplification.
- **Single-tenant vs multi-tenant.** Multi-tenancy introduces noisy neighbors, fairness concerns, isolation overhead, and tail risk.
- **Read-heavy vs write-heavy.** Read-heavy systems often benefit from caching and replication. Write-heavy systems hit coordination, durability, and compaction limits.
- **Hot-keyed vs uniform.** A uniform distribution spreads load. Hot keys, hot shards, and hot partitions collapse apparent parallelism.
- **Stateful vs stateless.** Stateless workers can be replicated freely. Stateful workers bring placement, recovery, consistency, and data-locality concerns.

Common practitioner terms at this level:

- online
- offline
- batch
- streaming
- request/response
- background job
- hot path
- cold path
- latency-critical
- throughput-critical
- real-time
- best-effort
- bursty
- skewed
- multi-tenant

This lens prevents optimizing the wrong thing. A 2x throughput improvement that worsens p99 latency may be a regression for an interactive service and a win for a batch job.

---

## 2. Algorithmic or operator motif

The algorithmic motif describes the logical structure of the computation. It asks what kind of operation the program is performing before considering any particular machine.

Common motifs:

- **Dense linear algebra.** Matrix multiply, convolution, tensor operations. Usually regular, compute-heavy, SIMD/GPU-friendly, and well described by arithmetic intensity.
- **Sparse linear algebra.** Sparse matrix-vector multiply, graph-like numerical methods. Often memory-latency-bound or bandwidth-bound because structure is irregular.
- **Stencil or grid computation.** Each cell updates from neighboring cells. Local communication, blocking, halo exchange, and cache reuse dominate.
- **Spectral methods and FFTs.** Structured communication and data reordering. Often sensitive to memory movement and all-to-all phases.
- **N-body and particle methods.** Pairwise or hierarchical interactions. Can range from compute-heavy to memory-irregular depending on approximation.
- **Monte Carlo.** Often embarrassingly parallel with independent samples, but random-number generation and aggregation can matter.
- **Map.** Apply a function independently to each element. Usually easy to parallelize unless input/output movement dominates.
- **Reduce.** Combine many values into one or a few values. Associativity, tree reduction, contention, and numerical stability matter.
- **Scan / prefix sum.** Parallelizable but dependency-shaped; requires structured algorithms rather than naive independent work.
- **Sort.** Comparison, partitioning, memory movement, branch predictability, and communication dominate.
- **Join.** Database-style matching across relations. Hashing, sorting, indexing, skew, and repartitioning dominate.
- **Filter / projection / aggregation.** Columnar analytics core. Often scan-shaped and memory-bandwidth-sensitive.
- **Graph traversal.** BFS, PageRank, shortest paths, connected components. Usually irregular, pointer-heavy, load-imbalanced, and hard on caches.
- **Dynamic programming.** Dependency graph matters. May be wavefront-parallel, memory-intensive, or dominated by table layout.
- **Search / backtracking / branch-and-bound.** Irregular control flow, pruning effectiveness, and load imbalance dominate.
- **Compression / encoding / parsing.** Often branch-heavy, table-driven, SIMD-amenable in optimized implementations, and sensitive to input distribution.
- **Cryptography and hashing.** Usually compute-heavy with specialized instructions available on many CPUs.
- **Machine-learning tensor work.** Dense tensor kernels, attention, embeddings, sampling, all-reduce, and memory footprint can each dominate depending on phase.

This lens is useful for pedagogy, algorithm selection, and recognizing known implementation strategies. It is less useful by itself for diagnosis. Saying “graph workload” or “machine-learning workload” is too broad to predict performance without the other lenses.

---

## 3. Dependency and parallel structure

This lens classifies how pieces of work depend on each other.

Important categories:

- **Serial.** Each step depends on the previous one. Parallel hardware cannot help unless the algorithm changes.
- **Embarrassingly parallel.** Tasks are independent. Scaling is limited by input/output, scheduling overhead, shared resources, and stragglers rather than algorithmic dependencies.
- **Data parallel.** Same operation over many data items. Often maps well to SIMD, GPUs, vectorized engines, and distributed partitioning.
- **Task parallel.** Different tasks execute concurrently. Scheduling, load balance, and dependency management matter.
- **Pipeline parallel.** Work flows through stages. Throughput is limited by the slowest stage; latency is the sum of stage latencies plus queueing.
- **Divide and conquer.** Work splits recursively and combines results. Critical path, granularity, and merge cost matter.
- **Barrier-synchronized.** Workers periodically wait for each other. The slowest worker controls progress.
- **Reduction-heavy.** Parallel work converges into shared aggregation. Tree structure, associativity, and contention matter.
- **Shuffle-heavy.** Data must be repartitioned across workers. Network, serialization, skew, and spill behavior often dominate.
- **Coordination-heavy.** Progress requires agreement: locks, transactions, consensus, quorum writes, leases, global metadata, or serializable ordering.
- **Straggler-sensitive.** Completion time is determined by the slowest shard, request, worker, or dependency.

Useful concepts:

- **Work.** Total amount of computation.
- **Span or critical path.** Longest dependency chain.
- **Available parallelism.** Work divided by span.
- **Strong scaling.** Fixed problem size, more resources.
- **Weak scaling.** Problem size grows with resources.
- **Amdahl limit.** Serial fraction caps strong scaling.
- **Gustafson effect.** Larger problems can expose more parallel work.

This lens is useful for choosing between single-node optimization, multithreading, GPU execution, distributed execution, and algorithm redesign.

---

## 4. Data movement and locality

Modern performance is often dominated by moving data rather than computing on it. This lens asks where bytes are, where they need to go, and whether the access pattern cooperates with the hierarchy.

### Working-set level

Classify the working set by where it fits:

- registers
- L1 cache
- L2 cache
- last-level cache
- DRAM
- local SSD
- network storage or object storage
- remote memory or another service
- distributed across many machines

Crossing each boundary changes the performance model. A hash table that fits in L3 is a different workload from the same hash table in DRAM. A scan from local NVMe is different from a scan over object storage even when the logical query is identical.

### Access pattern

Common access patterns:

- **Sequential streaming.** Best case for prefetchers, disks, object stores, and vectorized execution.
- **Strided.** Good if the stride is small and predictable; bad if it wastes cache lines or defeats TLB reach.
- **Random.** Usually latency-bound unless enough independent requests overlap.
- **Pointer chasing.** Worst case for memory latency because each next address depends on the previous load.
- **Gather/scatter.** Common in sparse, graph, and vectorized code. Often limited by memory-level parallelism and cache behavior.
- **Blocked/tiled.** Structured to reuse data while it is still cache-resident.
- **Coalesced GPU access.** Neighboring GPU lanes access neighboring addresses. Crucial for GPU throughput.
- **Uncoalesced GPU access.** Lanes access scattered addresses, wasting memory transactions.

### Locality

Useful locality vocabulary:

- **Temporal locality.** Reuse the same data soon.
- **Spatial locality.** Use nearby data on the same cache line or page.
- **Cache-line utilization.** Fraction of fetched bytes that are actually used.
- **Reuse distance.** How much other data is touched before a value is reused.
- **Prefetchability.** Whether future addresses can be predicted early enough.
- **TLB reach.** Amount of memory covered by address-translation cache entries.
- **NUMA locality.** Whether memory is local to the socket doing the work.

### Latency hiding and pipeline depth

Some workloads are slow not because the lower tier lacks bandwidth, but because each individual access has high latency. Object storage, remote services, SSDs, DRAM, and even GPU memory all have versions of this problem. The standard solution is to keep enough independent work in flight that completions arrive continuously.

The useful rule is Little's Law:

```text
request_throughput = concurrency / latency
byte_throughput    = concurrency × bytes_per_request / latency
```

So the required pipeline depth is:

```text
concurrency = target_byte_throughput × latency / bytes_per_request
```

Examples:

- S3 range reads hide 50 ms request latency by keeping many GETs in flight.
- CPUs hide cache-miss latency with out-of-order execution and memory-level parallelism.
- GPUs hide memory latency with many resident warps.
- Databases hide page-read latency with prefetching.
- TCP fills high-latency links with a large enough window.

This only works when future work is predictable or independent. Sequential scans, chunked files, batched lookups, and independent shards can be prefetched. True pointer chasing, random point lookups, and coordination-dependent RPC chains cannot be hidden as easily because the next address or request is not known early enough.

The engineering pattern is a producer-consumer pipeline with bounded queues and backpressure. The producer fills buffers from the slow tier; the consumer runs the kernel; the queue must be deep enough to absorb variance but bounded by bytes so it cannot exhaust memory.

### Communication pattern

For parallel and distributed work:

- no communication
- nearest-neighbor exchange
- broadcast
- fanout/fanin
- gather
- scatter
- reduce
- all-reduce
- all-to-all
- shuffle
- quorum
- consensus
- gossip
- pub/sub
- request/response chain

This lens is often more predictive than the algorithm name. “LLM inference” can mean compute-heavy prefill, memory-bandwidth-bound decoding, KV-cache pressure, batching tradeoffs, or distributed tensor communication. “Analytics query” can mean metadata-only lookup, scan/filter, hash join, sort, repartition, or UDF execution.

---

## 5. Resource and bottleneck diagnosis

This lens classifies what is actually limiting observed performance. It should be based on measurement, not intuition.

Common bottleneck classes:

### Compute throughput bound

The machine is doing useful arithmetic or logical work near its execution capacity.

Typical fixes:

- reduce work
- choose a better algorithm
- vectorize
- use specialized instructions
- parallelize if memory and coordination allow
- offload to GPU or accelerator if the workload fits

### Memory bandwidth bound

The program streams bytes as fast as memory can supply them.

Typical fixes:

- reduce bytes moved
- use narrower types
- improve compression
- improve cache reuse
- change layout from array-of-structs to struct-of-arrays when scanning narrow fields
- stop adding threads after bandwidth saturates

### Memory latency bound

The CPU waits on individual loads that cannot be overlapped.

Typical fixes:

- improve locality
- replace pointer-heavy structures with arrays
- batch independent lookups
- increase memory-level parallelism
- prefetch only when addresses are predictable early
- use huge pages when TLB misses dominate

### Cache-capacity or TLB bound

The working set does not fit in relevant caches or translation caches.

Typical fixes:

- block/tile the computation
- shrink data structures
- improve layout
- use arenas
- use huge pages carefully
- partition working sets per thread or per socket

### Branch or speculation bound

The processor repeatedly guesses control flow wrong.

Typical fixes:

- sort or group input so branches become predictable
- specialize hot paths
- use branchless code when both sides are cheap
- reduce indirect calls
- use profile-guided optimization

### Frontend bound

The CPU cannot fetch, decode, or deliver instructions fast enough.

Typical fixes:

- reduce hot code size
- avoid excessive inlining
- reduce virtual dispatch in hot loops
- improve code layout
- specialize without exploding instruction cache footprint

### Core execution bound

Execution ports, dependency chains, divides, transcendentals, or other long-latency operations dominate.

Typical fixes:

- break dependency chains
- use multiple accumulators
- replace expensive operations
- approximate where allowed
- expose more instruction-level parallelism

### Synchronization bound

Threads spend time waiting on locks, atomics, barriers, or coherence.

Typical fixes:

- remove shared state
- shard state
- use per-thread data with periodic reduction
- shorten critical sections
- reduce barrier frequency
- avoid false sharing

### Kernel or syscall bound

Time is spent crossing into the kernel or being scheduled by it.

Typical fixes:

- batch syscalls
- use larger reads and writes
- use vectored I/O
- reduce context switches
- use asynchronous I/O
- use kernel bypass only at extreme rates

### Storage bound

Local disk, network storage, or object storage limits progress.

Typical fixes:

- reduce bytes read or written
- improve layout
- coalesce small I/O
- increase request concurrency to the latency × throughput requirement
- use bounded prefetch queues to overlap I/O and compute
- cache or pre-stage data
- choose storage with the right latency/bandwidth/IOPS profile

### Network bound

Network bandwidth, latency, packet rate, or remote service time dominates.

Typical fixes:

- reduce payload size
- batch requests
- avoid chatty protocols
- colocate dependent services
- use compression carefully
- remove unnecessary fanout
- cache remote data

### Coordination bound

The system is limited by agreement, ordering, transactions, global metadata, or consistency requirements.

Typical fixes:

- weaken consistency if the product allows it
- partition coordination domains
- make operations idempotent
- use commutative updates
- avoid global locks or global counters
- redesign around ownership or single-writer paths

### Tail-latency bound

The average is acceptable, but high percentiles violate the objective.

Typical causes:

- garbage collection
- page faults
- cold caches
- lock convoys
- queueing
- retries
- stragglers
- noisy neighbors
- remote dependency outliers
- autoscaling cold starts

Typical fixes:

- measure latency distributions, not averages
- remove pause sources
- hedge carefully
- cap queue lengths
- use admission control
- isolate tenants
- reduce fanout
- warm caches or pre-touch memory

This lens is the most directly actionable for diagnosis. The output should be a sentence like: “This is a scan-shaped, data-parallel workload that is currently memory-bandwidth-bound on decoded column buffers,” not “this is CPU-bound.”

---

## 6. Domain-specific vocabularies

Different fields developed different taxonomies because they optimize different systems.

### HPC vocabulary

Common terms:

- dense linear algebra
- sparse linear algebra
- stencil
- structured grid
- unstructured grid
- spectral method
- N-body
- Monte Carlo
- strong scaling
- weak scaling
- halo exchange
- all-reduce
- communication-avoiding algorithm
- arithmetic intensity
- roofline model

Best for:

- teaching computational motifs
- hardware architecture discussions
- numerical computing
- accelerator selection
- reasoning about floating-point throughput and memory bandwidth

Limitations:

- weaker for production services, databases, tail latency, and coordination-heavy systems

### Database vocabulary

Common terms:

- OLTP
- OLAP
- HTAP
- point lookup
- range scan
- full scan
- index seek
- filter
- projection
- aggregation
- hash join
- merge join
- nested-loop join
- sort
- repartition
- exchange
- materialization
- compaction
- log flush
- transaction contention

Best for:

- query tuning
- storage layout
- indexing strategy
- transactional vs analytical system design
- operator-level performance diagnosis

Limitations:

- OLTP/OLAP is too coarse for many modern mixed workloads
- operator-level vocabulary is better than broad labels

### Distributed-systems vocabulary

Common terms:

- stateless
- stateful
- partitioned
- replicated
- sharded
- leader-based
- quorum-based
- consensus-based
- eventually consistent
- linearizable
- serializable
- fanout
- backpressure
- retry storm
- head-of-line blocking
- straggler
- hot shard
- shuffle
- coordination-free

Best for:

- service architecture
- data placement
- consistency design
- scale-out behavior
- failure and recovery reasoning

Limitations:

- often ignores low-level CPU and memory behavior
- “distributed” can hide the fact that the bottleneck is a single lock, database row, queue, or metadata service

### SRE and observability vocabulary

Common terms:

- rate
- errors
- duration
- utilization
- saturation
- latency
- traffic
- errors
- saturation
- SLI
- SLO
- error budget
- p50
- p95
- p99
- p999
- burn rate

Best for:

- production diagnosis
- incident response
- alert design
- service-level performance
- capacity planning

Limitations:

- describes symptoms and objectives more than computational structure

### ML systems vocabulary

Common terms:

- training
- inference
- prefill
- decode
- batch size
- activation memory
- optimizer state
- embedding lookup
- attention
- KV cache
- data parallelism
- tensor parallelism
- pipeline parallelism
- model parallelism
- expert parallelism
- all-reduce
- parameter server

Best for:

- GPU/accelerator utilization
- model serving
- distributed training
- memory-footprint reasoning
- batching tradeoffs

Limitations:

- labels like “AI workload” or “LLM inference” are nearly useless without phase and bottleneck classification

---

## 7. Imprecise and misleading terms

Many common terms are useful shorthand but bad diagnoses.

### CPU-bound

Often means only “not obviously blocked on disk or network.” It can hide:

- useful arithmetic throughput
- memory stalls
- branch misprediction
- frontend starvation
- lock spinning
- garbage collection
- interpreter overhead
- allocator overhead
- kernel time

Better: say which CPU-side resource or pipeline stage is limiting.

### Compute-bound

Sometimes means arithmetic-bound, sometimes means “cores are busy.” These are different. A core can be busy retiring little useful work while stalled on memory or speculation.

Better: distinguish retiring useful work from stalled cycles.

### Memory-bound

Can mean:

- DRAM bandwidth-bound
- DRAM latency-bound
- cache-capacity-bound
- cache-conflict-bound
- TLB-bound
- NUMA-bound
- allocator-bound
- GC pressure
- paging or swapping

Better: specify the memory hierarchy level and whether the issue is bandwidth, latency, capacity, translation, or locality.

### I/O-bound

Can mean:

- disk bandwidth
- random IOPS
- fsync latency
- object-store request overhead
- network bandwidth
- network round-trip latency
- packet processing
- remote service latency
- serialization

Better: name the I/O device, access pattern, and limiting metric.

### Embarrassingly parallel

Means tasks do not need to communicate logically. It does not mean the workload scales perfectly. It may still be limited by:

- input reads
- output writes
- shared storage bandwidth
- scheduling overhead
- startup cost
- hot partitions
- stragglers
- final aggregation
- metadata services
- license or cost limits

Better: call it dependency-free, then separately analyze shared resources.

### Scalable

Meaningless unless you specify what scales:

- throughput
- latency under fixed load
- latency under growing load
- p99 latency
- data size
- tenant count
- geographic footprint
- engineering team size
- cost efficiency

Better: state the scaling dimension and target.

### Real-time

Does not mean fast. It means deadline-constrained. A system with a 50 ms average and unbounded p999 is not real-time if it misses deadlines. A slower system can be real-time if it is predictable.

### Streaming

Can mean:

- unbounded event processing
- incremental processing of bounded data
- sequential memory/storage access
- low-latency push delivery
- media streaming

Better: specify boundedness, latency objective, state model, and access pattern.

### Batch

Can mean offline, bounded, scheduled, high-latency-tolerant, or simply non-interactive. Microbatch systems blur the line.

Better: specify whether the data is bounded, when results are needed, and whether processing is periodic or continuous.

### OLTP and OLAP

Still useful, but too coarse. Modern systems mix transactions, analytics, search, vector retrieval, streaming ingestion, and ML inference.

Better: describe operation mix and dominant operators.

### Workload

Overloaded. It may refer to:

- arrival process
- operation mix
- data distribution
- input size
- concurrency level
- benchmark
- production trace
- tenant mix
- time-of-day pattern
- computational kernel

Better: explicitly define which meaning is intended.

---

## 8. What each taxonomy is good for

No classification is universally best.

| Purpose | Most useful lens |
|---|---|
| Teaching broad computational patterns | Algorithmic motifs |
| Choosing algorithms | Complexity, dependency structure, data movement |
| Choosing hardware | Arithmetic intensity, memory behavior, communication pattern |
| Low-level optimization | Resource and bottleneck diagnosis |
| Parallelization | Dependency structure and communication pattern |
| Database tuning | Operator-level vocabulary and storage layout |
| Distributed-system design | State, coordination, topology, consistency |
| Production diagnosis | Workload contract, queueing, saturation, tail latency |
| Capacity planning | Arrival process, service time, utilization, saturation |
| Cost optimization | Bottleneck class, scaling dimension, resource efficiency |
| Pedagogy | Motifs, skeletons, and concrete examples |

---

## 9. Recommended working taxonomy

A performance engineer should internalize five complementary classifications.

### 1. Workload contract

Ask:

- Is this latency-sensitive, throughput-oriented, deadline-based, or batch?
- Which percentile matters?
- Is load open-loop or closed-loop?
- Is it steady, bursty, or adversarial?
- What is the operation mix?
- What are the data sizes and distributions?
- Is there skew, hot-keying, or multi-tenancy?

### 2. Algorithmic or operator motif

Ask:

- Is this a scan, join, sort, reduce, graph traversal, stencil, dense tensor kernel, sparse lookup, parser, search, or something else?
- What known implementations exist for this motif?
- Is the logical structure regular or irregular?
- Is the work uniform or input-dependent?

### 3. Dependency and communication structure

Ask:

- What is independent?
- What is serial?
- Where are the barriers?
- Where are the reductions?
- Is there a shuffle?
- Is there coordination?
- What is the critical path?
- Where can stragglers enter?

### 4. Data movement and locality

Ask:

- What bytes move?
- Where is the working set?
- Is access sequential, strided, random, or pointer-chasing?
- Is reuse high or low?
- Are we bandwidth-bound or latency-bound?
- Is communication local, remote, fanout, all-to-all, or quorum-based?
- Are we moving bytes that could be avoided?

### 5. Observed bottleneck

Ask:

- What resource is saturated?
- Where are cycles going?
- Where are queues forming?
- What does the latency distribution show?
- What changes when load increases?
- What improves when more CPU, memory bandwidth, network bandwidth, storage throughput, or parallelism is added?

The most useful final classification combines all five:

> This is a latency-sensitive, fanout-heavy request path whose dominant operation is remote reads; it is dependency-light locally but tail-latency-bound by downstream p99s and retry amplification.

or:

> This is a batch, scan-shaped, data-parallel Parquet workload with cheap predicates; it is dependency-free but currently object-storage-throughput-bound and sensitive to row-group layout, projection, and request concurrency.

or:

> This is an irregular graph traversal with high pointer chasing and poor locality; it has abundant theoretical parallelism but is memory-latency-bound and load-imbalanced in practice.

---

## 10. Bottom line

The field is messy because “computational work” is not one thing. Algorithm names, workload names, bottleneck names, and system-design names describe different layers. Confusion happens when one layer is used as if it explained all the others.

Use a faceted classification:

1. What does the workload need to deliver?
2. What algorithmic or operator motif is it?
3. What dependencies and communication does it impose?
4. What data moves, and with what locality?
5. What resource, queue, or coordination point is actually limiting observed performance?

A good performance classification is not a label. It is a compact causal model of why the work has the performance it has, and what kind of intervention is likely to help.
