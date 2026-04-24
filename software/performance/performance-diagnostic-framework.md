# Performance Diagnostic Framework

## The Core Principle

Performance is the gap between the machine your source code imagines and the machine that actually runs it. The imagined machine executes one instruction at a time against flat, uniform-latency memory. The real machine pipelines, speculates, reorders, caches, coheres, translates addresses, traps into the kernel, and occasionally pauses the world for garbage collection. Every performance problem lives somewhere in the distance between those two.

You cannot close a gap you have not measured. Intuition about where time goes is wrong often enough that measurement is not a phase — it is the substrate.

The sequence matters:

1. **Characterize the workload** — measure what the machine is physically doing
2. **Identify the dominant bottleneck** — find which of a small set of physical limits is binding
3. **Derive the intervention** — let the bottleneck dictate what changes
4. **Choose the technique** — pick specific tools that target the bottleneck

Engineers routinely invert this. They start with a technique ("let's vectorize this", "let's go lock-free", "let's add more threads") or with folklore ("SoA is faster", "mutexes are slow", "branches are expensive") and work backward to justify the choice. This produces code that is clever where it does not matter and naive where it does. A well-vectorized loop that the memory system cannot feed is not faster. A lock-free queue that spends its life in CAS retries under contention is not faster. The framework below makes the correct ordering concrete.

---

## Step 1: Workload Characterization

Before any optimization discussion, answer these questions with numbers. Rough numbers are fine. The goal is to see the physical shape of what the program is doing before deciding anything. Use `perf stat`, `perf record`, `toplev.py`, `likwid-perfctr`, VTune, or whatever your platform exposes. Do not skip this step because you "already know where the problem is" — you are wrong roughly half the time, and the other half you are missing the second bottleneck that will bite you after you fix the first.

### 1.1 Work Profile — what instructions actually run?

| Question | Why it matters |
|----------|---------------|
| Which functions dominate CPU time? | A flame graph tells you where to look, but PC samples point at where instructions retire, not necessarily where time is spent |
| What is the IPC (instructions per cycle)? | Low IPC on modern wide cores means the CPU is stalled, not working |
| Is time in user code, runtime, or kernel? | Profiles that miss kernel time or GC pauses explain a different system than the one that is slow |
| How many instructions per unit of user work? | The most effective optimization is often doing less; a better algorithm changes the count by orders of magnitude |

**Calibration for IPC on a modern big core (issue width 4–8):**

| IPC | Interpretation |
|-----|---------------|
| < 0.5 | Severely stalled — almost certainly memory- or synchronization-bound |
| 0.5 – 1.5 | Typical for mixed workloads; stalls somewhere |
| 1.5 – 3.0 | Good — the core is mostly working |
| 3.0+ | Near peak — further wins require algorithmic or SIMD changes |

### 1.2 Memory Profile — what bytes move?

| Question | Why it matters |
|----------|---------------|
| What is the working set size? Does it fit in L1, L2, L3, or only DRAM? | Each tier is 3–10× slower than the previous; "fits in cache" is the single largest factor in most memory-bound code |
| What is the dominant access pattern — sequential, strided, random, pointer-chasing? | The hardware prefetcher helps strided; it cannot help pointer-chasing or random access |
| What fraction of each cache line fetched is actually used? | 8 bytes used out of a 64-byte line is 8× wasted bandwidth |
| What is the DRAM bandwidth consumed vs machine peak (STREAM)? | If you are near STREAM, more threads or wider SIMD will not help — you are bandwidth-bound |
| What are the miss rates at L1, L2, L3, and TLB? | These identify which tier is the bottleneck |

**Calibration:**

| Signal | Benign | Suspicious | Bad |
|--------|--------|------------|-----|
| L1 miss rate | < 5% | 5–15% | > 15% |
| LLC miss rate (of LLC loads) | < 10% | 10–30% | > 30% |
| dTLB miss rate | < 0.1% | 0.1–1% | > 1% |
| Cache-line utilization | > 50% | 25–50% | < 25% |
| Bandwidth vs STREAM | < 50% | 50–80% | > 80% (saturated) |

### 1.3 Control Flow Profile — what is unpredictable?

| Question | Why it matters |
|----------|---------------|
| What is the branch miss rate in the hot loop? | A 15–20 cycle mispredict penalty on a hot branch dominates quickly |
| How many indirect branches fire per unit of work? (virtual calls, function pointers, interpreter dispatch) | Indirect branch prediction is weaker than direct; indirect-heavy code front-end-bounds easily |
| Are branches data-dependent on incoming values? | Random data defeats the predictor; sorted or clustered data trains it |
| Is the hot loop small enough to fit in the µop cache? | Code size above µop cache (~1.5–4 KB depending on CPU) forces re-decoding every iteration |

**Calibration:** in a hot loop, branch miss rates above 2–3% are worth investigating; above 10% usually means the branch is effectively unpredictable and either the data needs reordering or the branch needs to become a data operation.

### 1.4 Concurrency Profile — what is shared or serialized?

| Question | Why it matters |
|----------|---------------|
| How does wall time scale with thread count? | A sub-linear curve reveals a shared bottleneck; the shape says which |
| What is the lock wait time (contention) as a fraction of total time? | Lock contention above a few percent in a hot path is the bottleneck |
| Are there cache lines with high cache-to-cache transfer traffic (HITMs)? | True sharing or false sharing; both serialize through coherence |
| What fraction of DRAM accesses are remote (cross-socket) on NUMA systems? | Remote accesses are 1.5–3× slower than local; high remote ratio means placement is wrong |
| How often are threads migrated between cores? | Migrations destroy cache warmth; high migration count defeats per-thread locality |

**Calibration on the parallel scaling curve:**

| Shape | Interpretation |
|-------|---------------|
| Near-linear to many cores | Work is genuinely independent and the memory system is not saturated |
| Linear then plateau | Hit a shared resource — usually DRAM bandwidth or coherence |
| Linear then regression | Added contention (lock, false sharing, or cross-socket traffic) |
| Barely moves | Almost entirely serial; look for a single lock or shared counter |

### 1.5 System Boundary Profile — what crosses into the kernel?

| Question | Why it matters |
|----------|---------------|
| How many syscalls per unit of user work? | Each syscall is hundreds of nanoseconds minimum; a million small syscalls per second is significant CPU |
| What is the rate of context switches, and are they voluntary or involuntary? | Voluntary = blocking on I/O or locks; involuntary = preemption. Both cost cache warmth |
| Minor vs major page faults? | Minor faults are microseconds (zero-fill, COW); major faults are milliseconds (disk I/O). Either can dominate tail latency |
| For managed runtimes: GC pause frequency and duration? | A 5 ms pause at 1% frequency is invisible in averages and dominant in the tail |
| Is I/O batched, or one-byte-at-a-time? | `read()` per byte pays the syscall cost per byte |

### 1.6 Latency Distribution — what happens in the tail?

| Question | Why it matters |
|----------|---------------|
| What are p50, p99, p99.9, and max? | User-visible performance is governed by tails; averages hide the system you actually have |
| What is the source of each tail bump? | GC, page fault, JIT compilation, kernel scheduling, lock spike, TLB shootdown — each has a fingerprint |
| Is the benchmark open-loop or closed-loop? | Closed-loop benchmarks hide tails via coordinated omission; the measured system is not the one under real load |
| Over what time window is the tail observed? | A 5-minute run may not see a GC that runs every 10 minutes |

---

## Step 2: Identify the Dominant Bottleneck

After characterization, most of the numbers will be unremarkable. A loop that retires at 2.5 IPC, hits L1 99% of the time, branches predict at 99.5%, and uses a single core has no dominant bottleneck worth optimizing — unless you need it faster than that, in which case the bottleneck is algorithmic or SIMD.

The dominant bottleneck is what is **consuming most of the cycles that are not doing useful work**. Top-down microarchitectural analysis attributes every cycle to one of four buckets, and this is almost always the first cut.

### The four top-down buckets

| Bucket | Meaning | Dominant if |
|--------|---------|-------------|
| **Retiring** | Cycle produced useful work | You want more throughput than the algorithm can provide at peak — look at work reduction, SIMD, parallelism |
| **Bad speculation** | Cycle executed instructions that were squashed | Branch mispredicts, machine clears — look at branch patterns and indirect calls |
| **Front-end bound** | Back-end ready, front-end could not supply µops | Instruction-cache misses, decoder throughput, µop cache misses, indirect branches |
| **Back-end bound** | Front-end ready, back-end could not execute | The big one: splits into *memory bound* (waiting on loads/stores) and *core bound* (port saturation, long-latency ops, dependency chains) |

### Beyond top-down: orthogonal bottlenecks

Top-down is a single-core view. A few bottlenecks do not fit cleanly into its buckets, and top-down will report them as back-end-bound while hiding the real cause:

| Bottleneck | Signal |
|------------|--------|
| **Coherence / false sharing** | Scaling plateau or regression with thread count; high HITM rate on specific lines (`perf c2c`) |
| **Lock contention** | High `futex` time, threads spending time off-CPU waiting |
| **NUMA** | High remote-DRAM access rate; latency spikes correlated with remote socket |
| **Syscalls / kernel** | `perf stat` shows high kernel time; `strace -c` shows per-syscall time dominating |
| **Page faults** | `minor-faults` or `major-faults` counter high; tail latency correlated with memory pressure |
| **GC / runtime pauses** | Tail latency bumps at regular intervals; runtime's own log confirms |
| **Thermal / frequency throttling** | Average frequency drops below nominal under sustained load |

### Thresholds: when does a bottleneck become dominant?

| Signal | Ignore | Investigate | Dominant |
|--------|--------|-------------|----------|
| Fraction of top-down bucket | < 20% | 20–40% | > 40% |
| Branch miss rate in hot loop | < 1% | 1–5% | > 5% |
| LLC miss rate | < 10% | 10–30% | > 30% |
| dTLB miss rate | < 0.1% | 0.1–1% | > 1% |
| Bandwidth vs STREAM | < 50% | 50–80% | > 80% |
| Lock wait / total time | < 1% | 1–10% | > 10% |
| Remote DRAM fraction (NUMA) | < 10% | 10–30% | > 30% |
| Syscall time / total | < 5% | 5–20% | > 20% |
| GC pause / wall time | < 1% | 1–5% | > 5% |

Most real workloads have one dominant bottleneck and one or two secondary ones that will emerge after the first is fixed. If you find yourself listing five dominant bottlenecks, you are either misattributing (top-down says "back-end bound" but the real cause is coherence traffic) or you have not measured carefully enough.

### Worked diagnostic sketches

**Sketch: JSON log parsing at 40 MB/s/core on a 3.5 GHz machine.**

Measured: IPC 1.1, LLC miss rate 2%, dTLB miss rate 0.05%, branch miss rate 12%, back-end-bound 35%, bad-speculation 28%. Single-threaded.

Diagnosis: branches dominate (28% bad speculation, 12% miss rate). JSON parsing is full of data-dependent branches on the next character. The secondary bottleneck is back-end — likely core-bound on the branchy control flow. Memory is not the problem. This is a control-flow problem. Interventions: SIMD-based parsing (simdjson-style), branchless state machines, or vectorized character classification.

**Sketch: Matrix-transpose kernel, 4K × 4K doubles, 200 ms.**

Measured: IPC 0.3, LLC miss rate 45%, dTLB miss rate 3%, branch miss rate 0.1%, back-end-memory-bound 70%.

Diagnosis: memory-latency-bound, almost entirely. The 32 KB stride on the column access defeats every level of cache and blows the TLB. Interventions: block the transpose (tiling), use huge pages, consider non-temporal stores if the destination will not be read soon.

**Sketch: Web service, p50 5 ms, p99 180 ms.**

Measured: average CPU modest, average IPC fine. But the p99 is not explained by any CPU metric. `perf sched` shows occasional off-CPU stretches of 100–150 ms; GC log shows young-gen pauses at ~120 ms every ~30 seconds.

Diagnosis: the tail is GC, not code. No amount of hot-path optimization fixes this. Interventions: tune the collector, reduce allocation rate, move allocation-heavy paths off the request thread, or change to a low-pause collector.

---

## Step 3: Derive the Intervention

Each bottleneck type has a matching family of interventions. The intervention does not have to be clever — it has to match the binding constraint.

### 3.1 Retiring-dominant (compute-bound on useful work)

You are already doing useful work every cycle. To go faster you must do less work or wider work.

| Intervention | When it applies |
|-------------|-----------------|
| **Better algorithm** | Always the first question. O(n²) at n=10⁶ is a different problem than O(n log n); no micro-optimization recovers the gap |
| **SIMD / vectorization** | Inner loop is uniform arithmetic over contiguous data, no loop-carried dependencies, no function calls |
| **Parallelism** | The work decomposes, and the memory system is not already saturated |
| **Strength reduction** | Replacing expensive ops (divide, modulo, transcendentals) with cheaper equivalents |
| **Precomputation / memoization** | Results are reused across calls |

### 3.2 Bad-speculation-dominant (branch misprediction)

The pipeline is constantly flushing speculative work.

| Intervention | When it applies |
|-------------|-----------------|
| **Sort or group input** | The branch becomes predictable because like cases cluster. Often the highest-leverage single change |
| **Branchless code (`cmov`, masks)** | Both branches of the condition are cheap to execute unconditionally, and the branch was genuinely unpredictable |
| **Profile-guided optimization (PGO)** | The compiler needs to know the common path to lay out code correctly |
| **Reduce indirect dispatch** | Devirtualize, inline, replace virtual calls with switches over small enums, hoist dispatch out of inner loops |
| **Specialized versions** | Split one generic loop into two specialized loops where the branch is hoisted out |

Do not reach for branchless code when the branch was already well-predicted. A correctly predicted branch that skips expensive work is often the fastest option; making it branchless executes the expensive work every time.

### 3.3 Front-end-bound

The back-end can execute but the front-end cannot supply µops fast enough.

| Intervention | When it applies |
|-------------|-----------------|
| **Shrink the hot code footprint** | Excessive inlining, large switch statements, or unrolled loops overflowing the µop cache |
| **Reduce indirect calls in the hot path** | Virtual dispatch and function pointers feed the weaker indirect predictor |
| **Selective inlining** | Inline small leaf functions; stop inlining at the point where code size starts to hurt |
| **Huge pages for code** | On very large binaries, iTLB misses can front-end-bound the program |

### 3.4 Back-end memory-bandwidth bound

You are moving bytes as fast as the memory system allows.

| Intervention | When it applies |
|-------------|-----------------|
| **Reduce bytes moved** | SoA layout when scanning narrow fields; quantization to smaller types (float32 → int8); compression |
| **Increase reuse** | Tile/block the computation so each cache line is touched many times before eviction |
| **Streaming stores (non-temporal writes)** | Writing data that will not be re-read; bypasses cache, frees bandwidth |
| **Fewer threads, not more** | Above the bandwidth knee, additional threads add contention without throughput |

### 3.5 Back-end memory-latency bound

Bandwidth is fine; the CPU is waiting on the round-trip for individual loads.

| Intervention | When it applies |
|-------------|-----------------|
| **Improve locality** | Move related data together; reorder traversal; replace pointer chains with arrays |
| **Increase memory-level parallelism** | Batch multiple independent chains of dependent loads so the CPU can overlap them |
| **Software prefetch** | Last resort; effective only when prefetch distance can be computed and the pattern truly defeats the hardware prefetcher |
| **Huge pages** | Cuts TLB misses when working set is large and random |
| **Change data structure** | Linked list → array; chained hash table → open-addressed; pointer tree → implicit heap |

### 3.6 Back-end core-bound

An execution port is saturated or a dependency chain serializes.

| Intervention | When it applies |
|-------------|-----------------|
| **Break dependency chains** | Multiple accumulators in reductions; independent iteration streams |
| **Rebalance port pressure** | Replace two shifts with a shift and an add; the scheduler has freedom when ports differ |
| **Reduce long-latency ops** | Divides, square roots, FP transcendentals are multi-dozen-cycle; replace with approximations if semantics allow |

### 3.7 Coherence / false sharing

Threads are correct but the cache line holding their "independent" data is ping-ponging.

| Intervention | When it applies |
|-------------|-----------------|
| **Pad to cache-line boundaries** | `alignas(64)` on per-thread structures |
| **Per-thread data with periodic reduction** | Replace shared counter with per-thread counter summed occasionally |
| **Sharding** | Partition state so each partition is touched by one thread |

### 3.8 Lock contention

Threads are blocking on each other.

| Intervention | When it applies |
|-------------|-----------------|
| **Reduce critical section** | Move work out of the lock; the lock protects the update, not the computation |
| **Finer-grained locks** | Per-bucket, per-key, per-partition instead of global |
| **Lock-free data structures** | When blocking is unacceptable *and* measurement proves the lock is the bottleneck *and* contention is moderate |
| **Eliminate sharing** | Per-thread accumulators, single-writer ownership, message passing |

Reach for lock-free last. Most "lock contention" problems are really "too much shared state" problems and disappear when you reduce sharing.

### 3.9 NUMA

The data is on the wrong socket for the thread that needs it.

| Intervention | When it applies |
|-------------|-----------------|
| **Pin threads and memory** | `numactl --cpunodebind=N --membind=N` for per-socket workers |
| **Parallel first-touch** | Each worker initializes its own region, so first-touch places pages locally |
| **Interleave** | `numactl --interleave=all` for workloads whose access pattern is unpredictable across sockets |
| **Shard by NUMA node** | Treat each socket as an independent instance, like separate machines |

### 3.10 Kernel / syscall-bound

Time is in the kernel, not in your code.

| Intervention | When it applies |
|-------------|-----------------|
| **Batch I/O** | `writev`, `sendmmsg`, larger `read`/`write` sizes; one syscall for many operations |
| **io_uring** | Submit and complete I/O in bulk through a shared ring; typically closes most of the gap with kernel bypass |
| **Kernel bypass (DPDK, SPDK)** | Only when syscall overhead truly dominates and you can dedicate cores to polling |
| **Reduce context switches** | Pin threads; avoid blocking primitives in hot paths; use lock-free producer-consumer where appropriate |

### 3.11 Tail-latency-bound

The average is fine; the tail is the problem.

| Intervention | When it applies |
|-------------|-----------------|
| **Identify the pause source** | GC, page fault, JIT, scheduler, lock spike — each has a specific fix |
| **Reduce allocation rate** | Fewer collections; often the single most effective change for managed-runtime tails |
| **Pin memory (`mlock`)** | Eliminates major faults for latency-critical regions |
| **Pretouch / warm up** | First access to a page costs; touch everything before timing starts |
| **Low-pause collectors** | ZGC, Shenandoah (Java); Go's concurrent collector; carefully tuned G1 |
| **Move heavy work off the request path** | Pre-compute, precompile, pre-allocate; the request thread does only latency-critical work |
| **Hedged requests** | Send a second request after a timeout; return the first response. Trades throughput for tail |

---

## Step 4: Technique Choices

The technique is the last decision, not the first. By this point, the bottleneck has been identified and the intervention family is fixed. Now pick specific tools.

### Selection criteria, in order

1. **Is it a change in data or in code?** Changes in data layout and access pattern are usually higher-leverage and lower-risk than changes in instructions. Explore these first.
2. **Is there a simpler intervention that would also work?** A 10% win from huge pages with no code change beats a 15% win from hand-written SIMD. Operational cost matters.
3. **Does the technique compose with the rest of the system?** SIMD inside a virtual call gains you nothing because the compiler cannot see through the dispatch. Lock-free inside a single-writer path is pointless.
4. **Can you measure whether it worked?** The counter you expect to move must actually move. If it does not, your model was wrong and you learn something. If it does, and wall time does not follow, there is a secondary bottleneck.

### Common mappings (not prescriptions)

| Intervention | Typical technique | When to deviate |
|--------------|-------------------|-----------------|
| Reduce bytes moved | SoA layout, narrower types, column pruning | AoS when every field is touched in the phase |
| Improve locality | Tiling, space-filling curves, arena allocation | Tiling's overhead exceeds its benefit below certain sizes |
| SIMD | Compiler auto-vectorization (with `restrict`, inline, `-O3 -march=native`) | Intrinsics or hand-written SIMD when the compiler cannot see enough; ISPC or similar DSLs for portable SIMD |
| Branchless | Ternary that lowers to `cmov`; compiler mask operations | Keep the branch when well-predicted and skips expensive work |
| Reduce sharing | Per-thread state, thread-local allocators, sharded maps | Pay the coordination cost only where threads must agree |
| Lock-free | `std::atomic` with acquire/release on a proven bottleneck | A well-padded mutex under low contention is simpler and often equivalent |
| Huge pages | Transparent Huge Pages or explicit `hugetlbfs` | Disable THP on latency-sensitive services that see compaction spikes |
| Kernel bypass | io_uring first; DPDK/SPDK only at extreme rates | Batching inside standard syscalls often closes most of the gap |
| GC tuning | Adjust heap size and region size before changing collectors | Change collectors when pause-time requirements cannot be met by tuning |
| Observability | `perf stat`, `perf record`, `perf c2c`, `bpftrace` on Linux; VTune on Intel; Instruments on macOS | Use the platform's native profiler before hunting for exotic tools |

---

## Techniques as Emergent Outcomes

Optimization techniques are not a menu. They are outcomes that fall out of specific bottleneck combinations. Recognizing when each is forced — and when it is over-engineering — is most of the skill.

### SoA (Struct of Arrays) layout

**Emerges when:** phases scan narrow subsets of fields across many objects, and cache-line utilization under AoS would be low. The AoS layout wastes bandwidth proportional to unused fields.

**What it looks like:** Each field is its own contiguous array. Hot loops become simple stride-1 streams that the hardware prefetcher trivially handles.

**Real example:** Particle simulations, ECS (entity-component systems) in games, columnar analytics engines (Parquet, DuckDB, ClickHouse) — all of which scan one or two columns at a time over millions of rows.

**Over-engineering signal:** Converting to SoA when each phase touches most fields of each object anyway. The bytes came along for the ride and were going to be used; SoA buys nothing and loses the one-pointer convenience of AoS.

### Branchless code

**Emerges when:** a hot branch is genuinely unpredictable (miss rate 20%+) and both sides of the branch are cheap to execute unconditionally.

**What it looks like:** A `cmov`, a mask-and-add, a table lookup, or a predicated SIMD lane. Control flow becomes data flow.

**Real example:** Sorting network primitives, bit-parallel string matching, vectorized filters where a predicate selects which lanes contribute.

**Over-engineering signal:** Making every branch branchless. A predictable branch that skips expensive work is often the fastest option; the predictor already made the branch free and now you are executing the expensive side every iteration.

### SIMD / vectorization

**Emerges when:** the workload is retiring-bound or core-bound on arithmetic, the inner loop is uniform, and memory can keep the vector units fed.

**What it looks like:** Contiguous stride-1 access, no function calls in the loop, no aliasing (thanks to `restrict`), reductions with multiple accumulators, predicated lanes instead of branches.

**Real example:** Numerical kernels (BLAS, FFT), image and video codecs, JSON parsing (simdjson), hashing and compression (CRC, zstd).

**Over-engineering signal:** SIMD on a memory-bandwidth-bound loop. If the loop is already saturating DRAM bandwidth, wider arithmetic buys nothing — the data is not arriving any faster. Vectorize after fixing layout, not before.

### Lock-free data structures

**Emerges when:** blocking is unacceptable (latency-sensitive or progress-required), measurement shows the lock is the bottleneck, and contention is moderate enough that CAS retries do not dominate.

**What it looks like:** Atomic operations with acquire/release ordering, epoch-based or hazard-pointer reclamation, carefully paired memory fences. Correctness is hard; performance under high contention is not guaranteed to beat a mutex.

**Real example:** Per-CPU run queues in modern schedulers, lock-free queues in trading systems, hazard pointers in DBMS buffer pools, RCU in the Linux kernel.

**Over-engineering signal:** Reaching for lock-free because "mutexes are slow." Uncontended mutexes are ~20 ns. The usual fix for contended mutexes is less sharing, not more sophisticated synchronization.

### Huge pages

**Emerges when:** the working set is large (> 10s of MB) and randomly accessed, making the 4 KB TLB the binding constraint.

**What it looks like:** A TLB entry covers 2 MB or 1 GB instead of 4 KB. Random access over a multi-GB heap becomes tractable. Translation cost nearly disappears.

**Real example:** Databases with multi-GB buffer pools (Postgres, MySQL on large hosts), JVMs with large heaps, scientific simulations with gigabyte working sets, in-memory caches (Redis, memcached) on large instances.

**Over-engineering signal:** Enabling transparent huge pages on a latency-sensitive service without understanding THP compaction. THP can cause seconds-long stalls as the kernel defragments to form a 2 MB page. Many latency-sensitive shops disable THP and use explicit `hugetlbfs` only.

### Kernel bypass (DPDK, SPDK, XDP)

**Emerges when:** syscall and kernel-stack overhead demonstrably dominate, often at 10M+ packets/sec or 1M+ IOPS per core.

**What it looks like:** User-space drivers, polled I/O, huge pages for DMA buffers, dedicated cores spinning at 100%, zero syscalls on the data path.

**Real example:** High-frequency trading, software load balancers (Katran at Meta), packet-processing appliances, NVMe-heavy storage systems.

**Over-engineering signal:** DPDK for a web service doing 10K requests/sec. The kernel network stack is not your bottleneck at that rate; your application logic is. You are paying the cost of bypass (dedicated cores, complexity, lost isolation) for no benefit.

### Sharding and per-thread data

**Emerges when:** a shared data structure is the bottleneck on a multi-core system, but the work itself is independent per partition.

**What it looks like:** Each thread owns a slice of state. Aggregation is periodic, not per-op. No coherence traffic on the hot path.

**Real example:** ScyllaDB's shard-per-core architecture, Redis Cluster's hash slots, per-CPU counters in the Linux kernel, sharded map implementations.

**Over-engineering signal:** Sharding a workload that does not actually have a shared-state bottleneck. Sharding adds rebalancing and cross-shard query cost; if a single-threaded or lock-free version would have worked, you paid for complexity you did not need.

### Software prefetch

**Emerges when:** the access pattern is data-dependent (hash probes, tree walks, index lookups) but the *next* address is computable several iterations in advance.

**What it looks like:** `__builtin_prefetch(next)` issued at a carefully tuned distance ahead of use. The pipeline overlaps the load with unrelated work.

**Real example:** B+-tree traversals in databases, hash-join probes, graph algorithms with a known frontier.

**Over-engineering signal:** Sprinkling prefetches into a sequential loop. The hardware prefetcher already handles this and extra prefetches add pollution. If the pattern is stride-1, the hardware is smarter than your hints.

### NUMA pinning

**Emerges when:** measurement shows substantial cross-socket DRAM access on a multi-socket machine, and latency or bandwidth is the bottleneck.

**What it looks like:** `numactl --cpunodebind=N --membind=N`, parallel first-touch initialization, per-socket instances.

**Real example:** Databases on multi-socket servers (Postgres, Oracle), JVM services sharded per-socket, scientific codes with domain decomposition.

**Over-engineering signal:** NUMA pinning on a single-socket machine. There is no NUMA. Pinning there just costs you load balancing.

---

## The Over-Optimization Checklist

Before reaching for a complex intervention, run this checklist. Each "yes" is a warning that you may be adding complexity for no measured gain.

| Question | If yes... |
|----------|-----------|
| Have you measured this with hardware counters, not just wall time? | Wall time tells you something is slow; counters tell you why. Guess-based optimization is gambling. |
| Is the code you are about to optimize actually hot? | Flame graphs or `perf record` should show it above some threshold. A 20% speedup of a function that runs 0.5% of the time is 0.1% overall. |
| Is the improvement you expect larger than run-to-run variance? | If variance is ±10% and you expect a 5% win, you cannot tell whether it worked. |
| Is the benchmark representative of production? | Cold caches, real data distributions, realistic contention, production traffic shape. Microbenchmarks are upper bounds on production wins. |
| Is the benchmark open-loop? | Closed-loop benchmarks hide tails. If you care about p99, you need open-loop load generation (`wrk2`, not `ab`). |
| Have you checked what the compiler already does with `-O3 -march=native`? | Auto-vectorization, devirtualization, inlining, PGO often capture the "obvious" optimizations without code changes. |
| Does the "optimization" foreclose a simpler one? | Hand-written SIMD makes future algorithm changes expensive. Lock-free code is hard to modify. Weigh the long-term cost. |
| Is the dominant bottleneck still what you thought? | After one fix, the bottleneck shifts. Re-measure before the next change. |
| Is the code already fast enough? | The correct amount of optimization is what the workload requires, not the maximum the hardware allows. |

Every "probably not" is an invitation to stop. Optimization is not a virtue; it is a cost paid in complexity, maintenance, and foreclosed flexibility. The fastest code you can ship is the code that is fast enough with the fewest tricks.

---

## Putting It Together: A Worked Diagnostic

**Problem: An in-memory analytics service answers aggregation queries over a columnar dataset. Users report the p99 latency is 4× the median. The team wants to know whether this is fixable and, if so, what to change.**

### Step 1: Characterize

The team runs the workload under `perf stat --topdown` and a flame graph over a representative query mix.

- **Work profile**: Hot functions are in the aggregation kernel (predicate evaluation and sum), IPC averages 1.6 — reasonable. No significant time in the allocator or the serializer.
- **Memory profile**: Working set is ~20 GB, well beyond L3. Scan-heavy access patterns on columnar data are stride-1. LLC miss rate 18%, dTLB miss rate 2%, bandwidth at 72% of STREAM during scans.
- **Control flow**: Branch miss rate 0.4% — predictable.
- **Concurrency**: 16 threads, scaling linear to 8 and sublinear to 16. Some coherence traffic on the query coordinator's shared state.
- **System boundary**: Low syscall rate. Minor faults spike correlating with memory growth phases.
- **Latency distribution**: p50 30 ms, p99 120 ms. The tail is not random — bumps correspond to queries that scan newly-loaded data regions.

### Step 2: Identify the dominant bottleneck

Top-down shows 55% back-end memory-bound, of which most is DRAM latency (not bandwidth — bandwidth at 72% is substantial but not saturated). The dTLB miss rate at 2% is high enough to matter. The sublinear scaling above 8 threads plus the bandwidth number suggests the *second* bottleneck is bandwidth saturation, which will emerge after fixing the first.

The tail-latency story is separate: bumps correlate with new memory regions, suggesting TLB misses and first-touch minor faults on pages that have not been exercised.

**Primary bottleneck: memory latency, specifically TLB pressure on random access patterns in the aggregation kernel.**

**Secondary: bandwidth saturation will emerge at higher thread counts.**

**Tail driver: minor faults and cold TLB on recently loaded pages.**

### Step 3: Derive the intervention

- For memory latency and TLB: enable huge pages for the columnar data arena. A 2 MB TLB entry covers 512× the address space of a 4 KB entry, which should substantially reduce dTLB miss rate on large scans.
- For cold-page tails: pre-touch newly loaded regions before they enter the query-serving set. A background warmer thread that reads each new page once eliminates the first-query minor-fault spike.
- For the forthcoming bandwidth bottleneck: narrow types where possible (the current code uses int64 aggregates for counts that fit in int32), and verify that the columnar format actually stores only the columns being read per query. These are changes the team would make *after* confirming the TLB fix.

Not proposed: hand-written SIMD (compiler is already vectorizing the inner loops), lock-free coordinator state (coherence traffic is minor and not the bottleneck), NUMA pinning (single-socket machine), kernel bypass (I/O is not the issue).

### Step 4: Technique

- **Huge pages**: Allocate the columnar arena with `MAP_HUGETLB` against an explicit `hugetlbfs` mount, rather than relying on transparent huge pages. Explicit hugetlbfs avoids THP compaction pauses that would show up in the tail — exactly the metric the team cares about.
- **Pre-touch**: A dedicated warmer thread reads one byte per 2 MB page after load. Low CPU cost, eliminates the first-touch fault on the query thread.
- **Type narrowing**: Audit the aggregation kernel for unnecessary int64 promotion. Mechanical, low-risk.

### Predicted outcome

- dTLB miss rate drops from 2% toward 0.1%.
- LLC miss rate roughly unchanged (data still does not fit in LLC; we are addressing translation, not capacity).
- Back-end memory-bound fraction drops from 55% to ~35% as TLB-walk cycles are eliminated.
- p99 drops substantially because the cold-page fault tail is gone; p50 improves modestly.

### Step 5: Measure again

Run the same counter set on the changed system. If dTLB misses dropped, back-end memory-bound fraction dropped, and wall time followed, the model was correct. If dTLB misses dropped but wall time did not, there is a secondary bottleneck that has now become binding — most likely bandwidth, which the team already knew was next. Apply the type-narrowing change and re-measure.

If dTLB misses did *not* drop, the huge pages are not being used (common: the allocator silently falls back to 4 KB pages). Verify with `/proc/meminfo` and `pmap -XX <pid>`.

---

## Summary

The framework in three sentences:

**Measure the workload's physical behavior before changing anything.** Most performance problems have one dominant bottleneck — compute, memory bandwidth, memory latency, branches, synchronization, kernel boundary, or tail pauses — and the intervention is dictated by which one. An optimization you cannot trace to a measured bottleneck, and cannot verify with a counter that moves in the predicted direction, is an optimization you do not know worked.
