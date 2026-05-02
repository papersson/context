# On Performance: A Progression from the Abstract Machine to Real Hardware

## The Lie Your Language Tells You

When you write

```c
double sum = 0;
for (size_t i = 0; i < n; i++) {
    sum += arr[i];
}
```

the language invites you to reason about a particular machine. It executes your instructions one at a time, in the order you wrote them, against a memory that is flat, infinite, and uniformly fast. Every read costs the same. Every write costs the same. Every `add` takes one step. The only way to make this loop faster is to touch fewer elements or do less work per element.

That machine does not exist. It has not existed for about thirty years. The distance between what your source code suggests and what the silicon actually does is where all of performance engineering lives.

Before tearing the abstraction apart it is worth naming it, because the pieces will fall one at a time. The abstract machine you were taught has four properties:

1. **Sequentiality** — one instruction happens, then the next.
2. **Uniform latency** — every memory location is equidistant from the CPU.
3. **Infinite memory** — no hierarchy, no paging, no faults.
4. **Zero-cost arithmetic** — an `add` is free compared to anything else.

Every one of these is false on modern hardware. Your compiler and your CPU go to elaborate lengths to preserve the appearance of the first while ruthlessly violating the other three. The rest of this essay is about what happens underneath, in the order that the lies unravel.

This is the meaning of the phrase *mechanical sympathy*. The code you write is not the code that runs. Between your source and the transistors there is a compiler, an ISA, an out-of-order pipeline, a cache hierarchy, a virtual memory system, and an operating system. Each of them is doing something — and if you do not know what they are doing, you will write code that fights them and wonder why it is slow.

## First Crack: The Cost of a Load

A modern x86 core at 3.5 GHz completes one clock cycle in roughly 0.29 nanoseconds. In a single cycle it can, under favorable conditions, retire four or more instructions, perform a fused multiply-add on eight double-precision floats simultaneously, and issue another pair of loads. Call an arithmetic op *cheap*: on the order of one cycle of latency, and fractional cycles of throughput when well pipelined.

A load from main memory — DRAM — takes roughly 80 to 100 nanoseconds on a modern server under good conditions. Call it 300 cycles.

Hold that ratio in your head. The gap between performing an arithmetic operation and retrieving a value from main memory is a factor of several hundred. If your algorithm performs one DRAM load for every few arithmetic ops, you do not have a compute-bound program. You have a memory-bound program pretending to be a compute-bound one. Most "CPU-bound" code in the wild — code that sits at 100% CPU in `top` and looks busy — is actually waiting on memory for most of those cycles. The CPU is running, but it is running past bubbles where an instruction is stalled waiting for a cache line to arrive.

This gap has a name: the *memory wall*. It did not happen overnight. In the early 1980s DRAM and CPU cycle times were comparable; a load was not qualitatively different from an add. Since then single-thread throughput has risen by roughly three orders of magnitude, while DRAM latency has improved by perhaps a factor of three. The ratio widened every year for thirty years. Every structural feature of the memory hierarchy — every cache level, every prefetcher, every reorder buffer — is hardware's response to this widening gap.

Hardware's first answer was to put a small, fast memory very close to the core.

## Caches: Locality Becomes the Target

A cache is a small, fast memory that holds recently touched data from a larger, slower memory. On a modern x86 server core the hierarchy looks roughly like this:

| Level | Size (per core unless noted)    | Latency (cycles) | Latency (ns) |
|-------|----------------------------------|------------------|--------------|
| L1 data | 32–48 KB                       | 4–5              | ~1–1.5       |
| L2    | 256 KB – 1 MB                    | 12–15            | ~3–5         |
| L3 (shared on socket) | 4–64 MB          | 30–50            | ~10–20       |
| DRAM (local socket) | tens of GB – TB    | 200–350          | ~60–100      |
| DRAM (remote socket, NUMA) | —          | 400–600          | ~150–250     |

The exact numbers vary by generation and manufacturer. The *orders of magnitude* and the *gaps between levels* are stable and are what matter. L1 to L3 is roughly 10x. L3 to DRAM is another 5–10x. Register to DRAM is roughly 1000x.

Caches move data in fixed-size blocks called *cache lines*. On every modern x86 processor this block is **64 bytes**. It is not a tunable, not a default, not a suggestion. It is the physical unit of movement between levels of the hierarchy. When you read a single byte that is not in L1, the hardware does not fetch a byte; it fetches the entire 64-byte line that contains it, aligned to a 64-byte boundary. If you read the next byte on that line a moment later, it is effectively free; it came along for the ride.

This is the mechanical origin of *spatial locality*: accesses near a recent access are cheap because the line is already resident. *Temporal locality* is the other half — reuse a recent value and it is still in cache.

You can see the line size for your machine directly:

```
$ getconf LEVEL1_DCACHE_LINESIZE
64
```

The implication for data layout is enormous. Consider two designs for a particle system.

Array of structs (AoS):

```c
struct Particle {
    float x, y, z;        //  0..11   position
    float vx, vy, vz;     // 12..23   velocity
    float mass;           // 24..27
    uint32_t flags;       // 28..31
    // other fields, pad to say 64 bytes
};
Particle particles[N];
```

Struct of arrays (SoA):

```c
struct Particles {
    float x[N],  y[N],  z[N];
    float vx[N], vy[N], vz[N];
    float mass[N];
    uint32_t flags[N];
};
```

In memory the AoS layout places all fields of particle 0 together, then all fields of particle 1, and so on:

```
[x0 y0 z0 vx0 vy0 vz0 m0 f0 ... pad][x1 y1 z1 vx1 vy1 vz1 m1 f1 ...][...]
|----------------- 64 bytes ---------|---------- 64 bytes ---------|
```

The SoA layout places all x's together, then all y's, and so on:

```
[x0 x1 x2 x3 ... x15][x16 ... x31][...]      // the x array
[y0 y1 y2 y3 ... y15][y16 ... y31][...]      // the y array
...
```

Now consider a hot loop that updates position from velocity:

```c
for (i = 0; i < N; i++) {
    p.x[i] += p.vx[i] * dt;
    p.y[i] += p.vy[i] * dt;
    p.z[i] += p.vz[i] * dt;
}
```

Under AoS, reading `x` for particle `i` pulls in the entire 64-byte struct — including mass, flags, and any other fields you do not touch in this loop. Your effective memory bandwidth is reduced by roughly the ratio of struct size to touched fields. With a 64-byte struct where you touch only 6 floats (24 bytes) per particle in this phase, you waste 40 out of every 64 bytes you move. Under SoA you have six tight streams where every byte pulled is a byte used, and the hardware prefetcher (described in a moment) sees three predictable stride-1 patterns.

I have watched that single change — identical algorithm, identical correctness, flipped layout — take a simulation from 1.2 seconds per frame to 180 milliseconds. No algorithm change. No parallelism. No SIMD (yet). Just respecting what cache lines are.

This is the point at which most developers first encounter *contextuality*. There is no universally fast layout. AoS is often better when you touch most fields of each object in a phase, because then the "came along for the ride" bytes are actually used and there is one fewer pointer to juggle. SoA is better when phases touch narrow subsets of fields across many objects. Hybrid layouts — AoSoA, where you store structs of short arrays — exist because reality rarely hands you a pure case. "Cache-friendly" does not mean any one layout; it means *the layout under which the bytes you pull are the bytes you use*.

## The Prefetcher, Strides, and Pointer Chasing

If every L1 miss cost you 100 ns of stall and you did them sequentially, even a well-laid-out stride-1 loop over a large array would drag. It does not, and the reason is *prefetching*.

The hardware prefetcher watches your access pattern. When it sees two or three accesses on a consistent stride — especially stride +1 cache line at a time — it starts issuing loads ahead of you, fetching lines before you demand them. If the prefetcher is doing its job, your latency-bound loads become throughput-bound. The CPU never actually stalls on them; by the time the instruction that uses each line executes, the line is already there.

Prefetchers are quite smart. Modern Intel and AMD cores prefetch on positive and negative strides, detect strides that skip, and can track multiple streams in parallel. They are not, however, magic. They are defeated by:

- **Pointer chasing.** A linked list has no stride the prefetcher can detect. Each `node = node->next` is a dependent load: the address of the next access is not known until the current load completes. This is the worst kind of memory access: a chain of serial full-latency misses.
- **Indirect accesses.** `arr[index[i]]` cannot be prefetched effectively because the address depends on data the prefetcher has not seen.
- **Random access** over a working set larger than L3. No predictable pattern, no prefetch.

Here is the mechanical reason pointer-chasing data structures are slow on modern hardware even when their asymptotic complexity is fine. A linked-list traversal of N elements might be `O(N)` in operation count and `O(N)` in DRAM loads, but those loads are *serial* at the full latency of whichever level they hit. A well-laid-out array traversal has the same count of operations, but the loads overlap — the CPU pipelines them and the prefetcher runs ahead. Same big-O, wildly different wall-clock.

This is also the first taste of a pattern you will see again: the thing that makes code fast is often not reducing work but arranging work so the hardware can overlap it. *Latency* is what you pay for a single operation when nothing else is happening. *Throughput* is what you pay for that operation when the machine is doing other useful things at the same time. Most optimization, past a certain point, is about converting latency into throughput.

You can see this in action. Run this against an array of a few million pointers, arranged two ways — once where each node points to its neighbor in memory, once where nodes are shuffled randomly across a huge arena:

```
$ perf stat -e cycles,instructions,L1-dcache-load-misses,LLC-load-misses ./walker
```

On the neighbor-ordered list you will see LLC-load-misses near zero and IPC (instructions per cycle) near the machine's best. On the shuffled list you will see LLC-load-misses grow to nearly one per node visited and IPC collapse — often below 0.1. Same code. Same data size. Same number of instructions retired. Two orders of magnitude in wall time, all of it spent waiting.

## When Caches Aren't Enough: Pipelines and ILP

Caches buy you bandwidth and amortize latency. They do not, on their own, do anything about serial dependencies or the fact that a load costs *something*, even when it hits. To push further the CPU must do useful work while waiting, and this is where out-of-order execution enters the story.

A modern x86 core does not execute one instruction at a time. It executes something much stranger. At any given moment, hundreds of instructions are *in flight* inside the core. They have been fetched, decoded, renamed to internal registers, and dispatched to a pool of reservation stations where they wait for their inputs. When an instruction's inputs are ready, it is issued to one of several execution ports, each of which handles certain classes of operations. Results are written back and eventually *retired* in program order — giving the illusion that execution was sequential, which is the contract the abstract machine still insists on.

The dimensions are worth internalizing for a modern big core:

- **Pipeline depth:** roughly 15–20 stages. An instruction takes that many cycles from fetch to retire under ideal conditions.
- **Reorder buffer (ROB):** roughly 300–600 in-flight instructions on recent big cores. This is the CPU's visible window into the future of your program.
- **Execution ports:** roughly 8–12, specialized. For example on recent Intel big cores: a few integer ALU ports, two or three load ports, one store-data port, two FMA ports (each capable of 512-bit wide operations), and so on.
- **Issue width:** 4–8 instructions per cycle sustained.

What this gives you is *instruction-level parallelism* (ILP): independent instructions execute simultaneously even though your code listed them sequentially. Dependent instructions serialize only as much as their dependencies require. The CPU is a small greedy scheduler trying to squeeze the most useful work out of every cycle.

But the scheduler needs independent work to schedule. Two forces conspire against it: control flow, which obscures what instructions are even coming next, and data dependencies, which force certain instructions to wait for others.

## Branches and the Predictor

Every `if`, every loop back-edge, every virtual call, every function return is a branch. Before the branch resolves, the CPU does not know which instructions to fetch next. It cannot afford to wait — a modern core fetches 16 or 32 bytes of instructions per cycle and dispatches several per cycle. If it stalled on every branch it would grind to a halt in loop-heavy code.

Instead, the CPU *predicts*. A branch predictor, which on modern cores is a multi-kilobyte data structure with history tables, loop detectors, and indirect-branch predictors, guesses which way the branch will go and begins speculatively fetching, decoding, and even executing down the predicted path. If the prediction is correct, the speculative work retires normally and the branch was effectively free. If it is wrong, all speculative work is discarded, the pipeline is flushed back to the branch, and the correct path begins — 15 to 20 cycles later.

That penalty is enormous. On a 3.5 GHz core a mispredict costs roughly 5 nanoseconds of wall time, during which the CPU did no useful work. A loop with a hard-to-predict branch inside can easily spend more than half its cycles in mispredict penalties.

The canonical demonstration: sum the elements of an integer array that are greater than some threshold.

```c
long sum = 0;
for (size_t i = 0; i < n; i++) {
    if (arr[i] > threshold) sum += arr[i];
}
```

Run this with random `arr`. Then sort `arr` first and run it again. On the same hardware, same data, the sorted version is two to three times faster. Nothing has changed except that in the sorted array, the branch is predictable: for a long stretch it goes the same way every time, and the predictor learns it instantly. In the unsorted array it is random coin flips, the predictor achieves its baseline 50%, and half of every iteration is a flush.

You can see this directly:

```
$ perf stat -e branches,branch-misses ./bench
# sorted:   branch-misses / branches < 1%
# unsorted: branch-misses / branches ~ 50%
```

Rule of thumb: a branch-miss rate above a few percent in a hot loop is an opportunity. The question is not "are my branches predictable?" — most are — but "which specific branches are unpredictable and what do they cost me?"

There are three common responses. The first is to make the branch predictable, usually by sorting or grouping inputs so similar cases run together. The second is to remove the branch entirely by turning it into a data operation: `sum += (arr[i] > threshold) ? arr[i] : 0` can, with any decent compiler, lower to a conditional move (`cmovg`) or masked add that has no control-flow change at all. The third is to turn the branch into a lookup — compute a mask, multiply, accumulate.

Here is what the branchful version looks like in x86-64 assembly:

```asm
    cmp     eax, threshold
    jle     .skip
    add     rsum, rax
.skip:
    ...
```

And the branchless version:

```asm
    cmp     eax, threshold
    cmovg   rtemp, rax          ; if greater, move rax into rtemp; else leave rtemp = 0
    add     rsum, rtemp
```

The branchless version always executes every instruction. It has no predictor, and therefore no mispredict. On data that is truly random it can be much faster. On data where the branch is highly predictable, it can be *slower* — because the predictor correctly skipped the work most of the time. Contextuality again: "branchless" is not unconditionally faster. It is faster when prediction fails.

One related pitfall: indirect branches — virtual calls, function pointers, interpreter dispatches — have their own predictor, which is less accurate and harder to train. A C++ virtual call in a tight loop where the dynamic type varies can mispredict frequently. This is part of what "devirtualization" buys you when the compiler or JIT can prove the target. It is also why switches over small enums often beat branches of virtual dispatches — the former have a jump table with better prediction structure, the latter have an indirect branch through a vtable slot.

## Data Dependencies and What They Serialize

Even with perfect branch prediction and perfect caches, a chain of dependent instructions cannot be executed in parallel. If `B` uses the result of `A`, `B` must wait for `A`. Out-of-order execution helps only when there is independent work to hide the latency.

The cleanest example is a reduction. Consider

```c
double s = 0;
for (i = 0; i < n; i++) s += a[i];
```

Each iteration's `s` depends on the previous iteration's `s`. A floating-point `add` has a latency of ~3–4 cycles on current hardware but a throughput of two per cycle. The dependency chain through `s` means the loop cannot run faster than one add per 3–4 cycles — even though the machine could, in principle, issue two adds per cycle. You are bottlenecked on *latency* of a serial chain, with the machine's throughput sitting idle.

The trick — which most compilers will not do by default for floating point, because it changes rounding — is to break the chain into independent accumulators:

```c
double s0=0, s1=0, s2=0, s3=0;
for (i = 0; i+3 < n; i += 4) {
    s0 += a[i];
    s1 += a[i+1];
    s2 += a[i+2];
    s3 += a[i+3];
}
double s = (s0 + s1) + (s2 + s3);
```

Now there are four independent chains. The machine can issue adds from all four streams in parallel, and throughput rises accordingly. This is the same loop doing the same work — but arranged so the out-of-order engine has something to do.

This is why `-ffast-math` (or `-fassociative-math`) often produces large speedups on numerical code: it gives the compiler permission to reassociate floating-point operations, and one of the first things it does is break reduction chains.

The general principle: the CPU will find parallelism if you leave it any. Code that looks simple and linear can be mechanically serial. Sometimes the shortest-looking source is the slowest because it hides dependencies that the compiler cannot safely break.

## SIMD: When One Instruction Isn't Enough

We have pushed the single-scalar CPU about as far as it will go. To get more arithmetic throughput, modern cores execute *vector* instructions: a single instruction operates on a short vector of values at once. On x86 this is SSE (128-bit, 4 floats or 2 doubles), AVX/AVX2 (256-bit, 8 floats), and AVX-512 (512-bit, 16 floats). On ARM this is NEON and SVE.

A well-vectorized loop over floats runs at 8 or 16 operations per cycle, per core. An unvectorized one runs at maybe 1 or 2. Vectorization is usually the largest single-core speedup available after you've fixed your data layout and memory access patterns, precisely because those fixes are what *enable* it.

Compilers will try to auto-vectorize. They often fail silently. The reasons are mechanical and worth knowing:

- **Pointer aliasing.** In C, two `float*` arguments might overlap. The compiler must assume they do unless told otherwise, which turns loads and stores into a data dependency across iterations. `restrict` (C) or `__restrict__` (C++) tells the compiler the pointers do not alias.
- **Loop-carried dependencies.** If iteration `i+1` depends on iteration `i`, no vector arithmetic is legal without reshaping. (Reductions are a subtype of this.)
- **Non-unit stride.** Gather/scatter exists in AVX-512 but is slow. Unit-stride access is dramatically better.
- **Alignment.** Unaligned SIMD loads used to be expensive; today they are cheap when the data happens to be aligned and mildly worse when it is not. What kills you is loads that straddle a cache line, especially a 4 KB page boundary, which causes a *split load* with severe penalty.
- **Control flow in the loop body.** If every iteration has an `if` that goes different ways, the compiler needs to mask out lanes. It can do this with predicated instructions but often gives up.
- **Function calls.** A call with side effects or unknown semantics will end vectorization at that point. Mark helpers `inline` or hoist them.

You can see whether a loop vectorized by asking the compiler to tell you:

```
gcc -O3 -fopt-info-vec-optimized -fopt-info-vec-missed
# or clang:
clang -O3 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize
```

Or by looking at the generated assembly and searching for wide registers:

```
objdump -d ./prog | grep -E 'ymm|zmm'     # AVX-256, AVX-512
```

Scalar code uses `xmm0..xmm15` in narrow ways or general-purpose registers. Vector code uses `ymm` (256-bit) or `zmm` (512-bit) and instructions like `vfmadd231ps`. If you expect vectorization and do not see those, something blocked the compiler and you have to find out what.

A concrete example. I once profiled a numerical kernel where a colleague had written a neat helper `float dot(const float* a, const float* b, int n)` called in a loop. The loop ran at maybe 1.5 GFLOPS on a machine capable of 200. The helper was not getting inlined because it was in another translation unit, the outer loop could not be vectorized because it called the un-inlined helper, and the inner loop inside the helper could not be vectorized because it did not know whether `a` and `b` aliased. Marking the function `static inline` in a header and adding `restrict` to the parameters took the same code from 1.5 to 80 GFLOPS. The algorithm did not change. The memory did not change. The only thing that changed was the compiler's information about what was safe.

This brings in the *cost of abstraction* thread. "Zero-cost abstraction" means the abstraction generates the same machine code as the hand-written version would have. That is true when the compiler can see enough to prove the abstraction away: when the function is inlined, when pointers do not alias, when the type is known at the call site, when the iteration is statically bounded. Outside those conditions, abstractions cost. A `std::vector` traversal is usually zero-cost. A `std::function` in a hot loop is not — it is an indirect call through a type-erased stub, with all the predictor woes of indirect branches. A polymorphic virtual call in a tight inner loop is not — it disables devirtualization-dependent optimizations across the call boundary.

## Virtual Memory's Quiet Tax: The TLB

So far the addresses in our code have been treated as if they went directly to DRAM. They do not. Every memory access issued by a user-space program is a *virtual* address that must be translated to a *physical* address by the memory management unit (MMU) before the access can complete. The translation lives in page tables, a per-process tree structure in memory; walking the tree to translate one address is itself several memory accesses.

If every load required a page-table walk, we would not have modern computing. The translations are cached in the *TLB* (Translation Lookaside Buffer), a small specialized cache right in the MMU. A TLB hit costs essentially nothing. A TLB miss triggers a hardware page walk — several dependent loads — and if any of those loads themselves miss cache, the cost compounds.

The scale of the TLB is what trips people up. A modern L1 dTLB has on the order of 64 entries. At the default page size of 4 KB, that covers 64 × 4 KB = 256 KB of address space. An L2 TLB might have 2048 entries, covering 8 MB. *Working sets larger than 8 MB of randomly accessed 4 KB pages will miss the TLB on nearly every access*, even if the data is resident in L3 or L2 cache. You get the cache latency *plus* the page-walk latency. Your cache-miss counters may look fine while the actual time is being spent in translation.

Symptoms of a TLB-bound workload:

```
$ perf stat -e dTLB-loads,dTLB-load-misses,dtlb_load_misses.walk_duration ./prog
```

If `dTLB-load-misses / dTLB-loads` is above a percent or two, or if `walk_duration` is a large fraction of cycles, you are paying the TLB tax.

The mitigation is *huge pages*. Instead of 4 KB pages, the kernel and CPU can use 2 MB or 1 GB pages. A TLB entry for a 2 MB page covers 512 times as much memory as a 4 KB entry. On Linux you get them via `madvise(..., MADV_HUGEPAGE)` or transparent huge pages (THP), or via explicit allocation from hugetlbfs. Applications with large working sets — databases, scientific simulations, JVMs with large heaps — routinely see 10–30% wall-time improvements from turning on huge pages, purely from fewer TLB misses and page walks. The wins are larger on workloads whose access patterns defeat the prefetcher because the TLB cost is exposed with nothing to hide it.

Huge pages are not free. They increase internal fragmentation, they can cause latency spikes when the kernel compacts memory to form them, and transparent huge pages in particular have a history of causing pathological stalls in latency-sensitive services (several large companies disable THP by default on their servers). This is another instance of the tension that runs through the whole discipline: *throughput vs latency vs determinism pull in different directions*. Huge pages buy throughput. They can cost you tail latency. Depending on whether you are running a batch analytics job or a low-latency trading system, either trade may be correct, and neither is universally right.

## Cores and Coherence

Everything up to now described a single core. Real machines have many. The hardware must maintain an illusion that matters for correctness: when multiple cores read and write the same memory, everyone agrees on what's there and in what order. This is *cache coherence*.

Private caches per core make this non-trivial. If core 0 has a modified copy of cache line X in its L1 and core 1 reads X, core 1 must somehow see the modified data. The protocol that maintains this is some variant of MESI (Modified, Exclusive, Shared, Invalid) or its extensions (MOESI, MESIF). The protocol is implemented in hardware and usually invisible. What is visible is its cost.

Every line in every private cache is in one of the states above. When core 1 reads a line that core 0 holds in Modified state, the protocol forces core 0 to write back (or forward) the data and downgrade its copy to Shared (or Invalid). When core 0 wants to *write* a line, it must first acquire it in Exclusive or Modified state, which means invalidating every other cache's copy. This back-and-forth is sometimes called *cache-line bouncing* or *ping-ponging*, and each ping-pong costs on the order of an L3 or cross-socket access — tens to hundreds of cycles, multiplied by however many cores are contending.

The implication for concurrent code is sharp. A shared counter incremented by every thread can trivially become the single slowest instruction in your program. Even with atomic increments — which are *correct* — the cache line holding the counter must move between cores on every operation. At that point you are not limited by CPU, not by algorithm, not even really by memory bandwidth. You are limited by the coherence traffic on the interconnect between cores.

This is the mechanical reason "why doesn't adding threads give me linear speedup on embarrassingly parallel work?" has so many answers. One of them — often the most surprising — is that the parallel work was not actually embarrassingly parallel. The threads touched shared state, even innocuously, and the coherence protocol turned throughput into a bottleneck you did not see in the code.

## False Sharing: Contention Without Sharing

The worst version of this is when the programmer is not even aware of sharing anything. Two variables, two threads, two cores, but the two variables happen to sit on the same 64-byte cache line. Every write by thread A invalidates the line in thread B's cache, forcing thread B to re-fetch it on its next read, which in turn invalidates thread A's copy on its next write. This is *false sharing*, and it is perhaps the most pernicious performance bug in multicore code, because the code looks correct, looks independent, passes every test, and is mysteriously slow.

The canonical example:

```c
struct counters {
    long a;   // updated by thread A
    long b;   // updated by thread B
};
```

`a` and `b` are 8 bytes each. They occupy the same 64-byte cache line. Each thread updates only its own counter, millions of times per second. Because every update pulls the whole line into Modified state on that core, the line ping-pongs between cores on every increment. A workload that you expected to see 2x speedup from two threads ends up 5–10x *slower* than the single-threaded version.

The fix is to pad the structure so each counter occupies its own cache line:

```c
struct counters {
    long a;
    char pad[56];    // fill out the rest of the 64-byte line
    long b;
};
```

Or more portably in C++:

```cpp
struct alignas(64) Counter {
    std::atomic<long> value;
};
Counter a, b;   // each on its own line
```

Detecting false sharing without being told about it is hard. CPU counters help. On Intel, `perf c2c` (cache-to-cache) is built for it:

```
$ perf c2c record ./prog
$ perf c2c report
```

It reports *HITMs* — loads that hit a modified line in another core's cache — by source line and by data address. A hot line with many HITMs from multiple CPUs is almost certainly false sharing (or true sharing that shouldn't be).

A production anecdote: a cache of small metrics objects, one per request handler, was giving a 12-core server almost no benefit over 6 cores. The metrics were thread-local in logic but allocated from a pool that placed adjacent objects on the same cache line. Each handler wrote its own metrics, but the writes invalidated neighbors' lines on other cores. Aligning allocations to 64 bytes — at a cost of roughly 40 bytes of padding per object, totally negligible — doubled throughput.

## NUMA: When Memory Has a Location

As core counts grew, putting them all on one bus to one memory controller stopped scaling. Modern multi-socket systems have multiple *NUMA nodes* — typically one per socket, sometimes more — each with its own memory controller and its own pool of DRAM. All cores can access all memory, but accessing memory attached to your local node is much faster than accessing memory on a remote node.

The numbers:

- Local DRAM latency: ~80–100 ns
- Remote DRAM latency (one hop over the interconnect): ~130–180 ns, often more under load
- Memory bandwidth: each NUMA node has its own channels; cross-socket bandwidth is more limited and shared

You can see your topology directly:

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-15 32-47
node 0 size: 128000 MB
node 1 cpus: 16-31 48-63
node 1 size: 128000 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

Those distances are relative (10 = local, 21 = one hop); the ratio roughly matches the latency ratio.

Linux's default memory allocation policy is *first-touch*: a page is placed on the NUMA node of the CPU that first writes to it (not the one that allocated it — the one that first touches it). This is a sensible default, but it means a common pattern — allocate a big buffer on the main thread, then hand chunks to worker threads for processing — can result in all the memory being on one node and all the work being on another. The workers pay the remote-access penalty for the duration of the job.

The fixes are all variations on "place data close to the CPU that will use it":

- `numactl --cpunodebind=0 --membind=0 ./prog` — pin this process to node 0's CPUs and force its allocations onto node 0.
- `mbind(2)`, `numa_alloc_onnode(3)` — programmatic placement.
- Parallelize *initialization* so each worker touches its own data first, letting first-touch do the right thing.
- Set `MADV_NUMA_BALANCING` or use `numad` to let the kernel move pages toward the cores that touch them.

You can check whether your job is suffering with `numastat -p <pid>`, which shows per-node memory use and remote-node accesses, and with hardware counters like `offcore_response` on Intel.

NUMA does not matter on single-socket machines. It matters a lot on dual-socket servers and matters more on four- and eight-socket ones. On many workloads it is the largest remaining performance gap after the single-core optimizations are done. A server with 64 cores can look like a server with 32 cores if half the cores are perpetually waiting on cross-socket traffic.

## Synchronization and Its Discontents

So far we have talked about coherence traffic caused by incidental sharing. What about *intentional* sharing — code that needs to coordinate?

At the hardware level, the atomic primitives — `lock xadd`, `lock cmpxchg`, `mfence`, and friends on x86 — are the building blocks. A `lock`-prefixed instruction acquires exclusive ownership of the relevant cache line, performs the read-modify-write atomically, and publishes the result. On uncontended data this costs perhaps 15–30 cycles — more than a plain store, but not catastrophic. On *contended* data, where many cores hammer the same line, costs explode nonlinearly, because every participating core must acquire and lose the line in turn.

Above atomics sit mutexes and above those lock-free data structures. The folklore is that lock-free is faster. The reality is: *it depends on contention*.

- An uncontended `pthread_mutex_lock` on Linux with the adaptive/futex-based implementation is roughly 15–25 ns of overhead per lock/unlock pair. That is comparable to a single atomic op.
- A contended mutex has two costs: the atomic op on the fast path, and — on the slow path — a syscall to wait in the kernel, which is ~1 μs minimum.
- A lock-free algorithm avoids the syscall but still pays for every atomic operation, and typically issues more of them than the locked version would. Under contention, a lock-free queue can spend all its time in CAS retry loops while a locking queue would have serialized and moved on.

The honest guidance is:

- Prefer a plain mutex first. Measure.
- If you are deep in a hot path and the mutex shows up in profiles as contended, look at the access pattern before you reach for lock-free. The first-order question is whether you can reduce contention (per-thread counters flushed periodically, sharding, batching).
- Lock-free is worth the complexity when you have quantitative evidence — not intuition — that blocking is the bottleneck, and when you know what you are trading. Lock-free code is harder to reason about, harder to debug, and harder for future readers.
- Reader-writer locks, RCU, seqlocks, and epoch-based reclamation exist for reasons. None are free. All are specific tools for specific contention patterns.

This is also where *throughput vs latency vs determinism* becomes sharpest. A mutex with 99th-percentile wait time of 10 ms under spikes is fine for a throughput-oriented batch job and disastrous for a latency-bound service. A lock-free queue with higher average cost but bounded worst-case can be the right answer for the latter — not because it is "faster" on average but because its tail is shorter.

## The OS Is There, and It Costs You

The CPU is not the only thing between your code and getting work done. The kernel is running too: handling interrupts, scheduling threads, managing memory, servicing syscalls. Most of the time it stays out of your way. Sometimes it does not.

A *syscall* — `read`, `write`, `recvmsg`, `futex`, `clock_gettime`, `getpid` — requires a user/kernel transition. On modern hardware with mitigations enabled, a no-op syscall costs roughly 100–500 nanoseconds. A syscall that does real work costs at least that much plus whatever the work takes. If your application makes a million small syscalls per second, you are spending a significant fraction of your cores on the kernel boundary itself.

A *context switch* — involuntary, when the scheduler preempts you; voluntary, when you block on I/O or a futex — costs on the order of 1–5 μs plus whatever indirect costs come from the caches and TLB of the new thread displacing the caches and TLB of the old one. That indirect cost can dominate. A tight-loop CPU-bound thread that gets context-switched out and back in may take milliseconds to recover its working-set warmth.

A *page fault* — your program accesses a virtual page not currently mapped to physical memory — comes in two flavors. *Minor faults* are handled by the kernel without I/O (e.g., zero-filling a page on first write, or remapping a copy-on-write page); they cost microseconds. *Major faults* require reading from disk (paging in from swap, or demand-loading an mmapped file); they cost milliseconds, which is an eternity. A latency-sensitive service that allows any of its hot memory to be paged out will have tail latencies measured in seconds the first time it goes quiet and then needs to respond quickly. Solutions include `mlock`/`mlockall` (pin pages in RAM), disabling swap on these hosts, and huge pages (fewer TLB entries to fault in).

The *allocator* is also in this category. A `malloc` of a small object is usually fast — a few hundred cycles — if the allocator has a suitable free chunk on a thread-local list. A `malloc` that needs to grab from a central pool involves a lock. A `malloc` that needs to grow the process's address space calls into the kernel (`brk` or `mmap`), which may trigger faults on subsequent writes, which cost microseconds each. Programs that allocate heavily in hot paths — particularly ones that were written in a language or style with pervasive small allocations — can spend a substantial fraction of their time in allocator internals and the associated kernel traffic. This is why high-performance C++ code often uses arenas, pools, and custom allocators, and why languages like Java and Go have spent a decade engineering allocators and collectors to mitigate the problem.

Which brings us to *garbage collection*, and a note about a class of problems that is invisible to CPU profilers. A garbage collector that pauses your application, even for a few milliseconds, is a latency event. Even concurrent collectors have write barriers and read barriers that cost, and they have phases — marking, compacting — that use memory bandwidth and evict your working set from cache. A flame graph might look clean; a latency histogram shows a bump at the pause time. The rule is: if you care about tail latency, you must measure tail latency directly, and you must attribute it to specific events. Average CPU profiles will not show you pauses of 2 ms that happen 1% of the time, and those pauses may be your dominant user-visible problem.

## Kernel Bypass

When syscalls, context switches, and kernel-side network or storage stacks themselves become the bottleneck — which happens in high-frequency networking, storage-intensive systems, and similar workloads — the answer is to get the kernel out of the fast path entirely.

- **DPDK** (Data Plane Development Kit): a user-space network stack that takes over the NIC, polls it directly from a pinned core, and moves packets into user memory with zero syscalls on the data path. Used by high-performance routers, load balancers, and trading systems. A DPDK pipeline can process tens of millions of packets per second per core, where the kernel stack would be at one or two million.
- **SPDK**: the same idea for NVMe storage.
- **io_uring** (Linux): not strictly kernel bypass, but a shared-memory ring interface between user space and kernel that lets you submit and complete large batches of I/O without a syscall per operation. For many workloads it closes most of the gap with full bypass while keeping kernel isolation.
- **XDP** (eXpress Data Path): run eBPF programs on packets right at the driver level, before the kernel networking stack sees them. Used for DDoS mitigation, load balancing, and fast-path filtering.

Kernel bypass is not a free lunch. You lose the kernel's isolation, scheduling, and abstractions. You generally burn a core at 100% for polling. You write or depend on a user-space stack that re-implements a significant chunk of what the kernel was doing for you, and you inherit the responsibility of making it correct. It is the right tool when you have measured that the kernel's per-operation overhead dominates and no amount of batching or coalescing will close the gap. It is the wrong tool when your actual problem is elsewhere — which it usually is.

## Measurement: Tails, Noise, Honesty

Everything above is useless if you cannot measure. Performance work is empirical; intuition about where time goes is wrong often enough that the only reliable practice is to measure first, form a hypothesis, verify the hypothesis with a targeted experiment, change one thing, and measure again. "It got faster on my laptop" is not evidence; it is an anecdote that might survive rigor or might not.

Several things go wrong with benchmarks that look fine on the surface.

*Averages hide tail behavior.* A service with an average latency of 2 ms and a 99.9th-percentile latency of 500 ms is not a 2 ms service; it is a 500 ms service with a lot of 2 ms noise. User-visible performance is usually governed by tails, not averages. Measure histograms. Report percentiles. Do not compare means when the distributions are long-tailed, which in practice they almost always are.

*Warmup and noise.* The first N iterations of any benchmark run at different speeds than the rest: caches are cold, the branch predictor is untrained, JITs have not compiled yet, frequency governors have not ramped up, THP has not yet been allocated. Throw them out. Then, run the benchmark enough times that the variance between runs is small compared to the differences you are trying to measure. On a multi-tenant machine (a cloud VM, a dev laptop running browsers), variance can be 20% or more; your "15% improvement" may be noise. Pin to cores (`taskset`), disable frequency scaling (`cpupower frequency-set -g performance`), disable turbo if you need determinism, and run on a quiet machine.

*Coordinated omission.* A subtler bug: when a benchmarking client sends one request, waits for the response, then sends the next, and a response is slow, the client has implicitly not sent the requests that would have been sent during the slow period. The slow period is under-sampled in the histogram. The true latency experienced by a real load is hidden. The fix is to drive load *open-loop*: issue requests on a schedule (e.g., at a target RPS) regardless of whether previous responses have come back, and measure latency from scheduled send time to received time. Tools that get this right (`wrk2`, proper `fortio` configs, custom harnesses) will show much longer tails than the naive closed-loop version.

*Microbenchmarks lie.* A microbenchmark runs a loop in a tight environment with hot caches and a warm predictor. The code it measures may run in production with cold caches, a polluted predictor, under memory pressure, with neighbors sharing the L3. Microbenchmark numbers are upper bounds on production performance, not estimates of it. Macrobenchmarks — running the actual system under actual load — are the only reliable guide to whether a change in the small makes a difference in the large. It is routine for a microbenchmark to show a 3x speedup that does not move the macrobenchmark at all, because the real bottleneck was somewhere else.

*Statistical significance.* When you compare two implementations, you are testing whether their means (or some percentile) differ more than variance alone would produce. Report confidence intervals or at least the range of run-to-run variation. A "10% speedup" that is within the run-to-run noise of either implementation is not a speedup.

## The Top-Down Method and the Toolchain

Given all this, how do you actually *find* where a program is spending time? Traditional sampling profilers — which interrupt the program periodically and record what function is executing — will tell you where the program counter sits. That is sometimes all you need. Often it is not, because the program counter only tells you *which instruction* was in flight, not *why it was slow*. An instruction sitting at the top of a profile for 40% of samples might be an arithmetic op that takes one cycle — but is waiting hundreds of cycles for the load that feeds it. The sample lands on the arithmetic op; the actual cost is the load.

This is why *hardware performance counters* matter. Every modern CPU exposes a set of events — retired instructions, executed cycles, L1 misses, L2 misses, L3 misses, branch mispredictions, TLB misses, stalls caused by frontend vs backend, and so on. A profile that combines PC sampling with counter sampling tells you not only where time was spent but *why* it was spent there.

On Linux, `perf` is the standard tool:

```
$ perf stat -e cycles,instructions,branches,branch-misses,\
L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses,\
dTLB-loads,dTLB-load-misses ./prog

 Performance counter stats for './prog':
   45,231,902,144   cycles
   18,432,112,901   instructions             # 0.41 insn per cycle
    3,912,401,221   branches
       42,103,991   branch-misses            # 1.08% of all branches
    7,203,445,901   L1-dcache-loads
    1,104,301,884   L1-dcache-load-misses    # 15.33% of L1 loads
      412,093,128   LLC-loads
      203,441,022   LLC-load-misses          # 49.36% of LLC loads
    7,203,445,901   dTLB-loads
       88,210,432   dTLB-load-misses         # 1.22% of dTLB loads
```

This output tells a specific story. IPC (instructions per cycle) of 0.41 on a machine that can sustain 3+ means the CPU is mostly stalled. LLC miss rate of 49% means half the loads that reach L3 go all the way to DRAM — this is a memory-bound workload. dTLB miss rate above 1% is high. Branch-miss rate of ~1% is fine. This program's problem is memory, and specifically capacity (it is missing out of L3, not just L1). You would look at working set sizes, data layout, and huge pages before looking at anything else.

A different shape:

```
  3.5 IPC, LLC miss rate 0.2%, branch-miss rate 12%, dTLB miss 0.05%
```

Completely different problem. The CPU is actually executing (3.5 IPC is very high). The memory system is fine. Branches are the issue — 12% of them mispredict. You go look for unpredictable branches.

And another:

```
  1.8 IPC, L1 miss rate 0.5%, LLC miss rate near zero, branch-miss 0.4%
```

Here the memory system and branches are fine. IPC is below peak. This is usually *front-end bound* — instruction fetch or decode can't keep up (often because the code is too large for the micro-op cache, or heavy indirect branches confuse the frontend) — or *backend bound on throughput* (some execution port is saturated, commonly the memory-load ports or the FP units, depending on the code). You need more information.

That more information comes from the *top-down microarchitectural analysis method*, originally developed at Intel. Rather than trying to reason about individual counters, top-down attributes every CPU cycle to one of four buckets:

- **Retiring** — a cycle that successfully produced useful work. More is better. Ideal code lives here.
- **Bad speculation** — a cycle spent executing instructions that were later squashed (branch mispredicts, machine clears). Usually indicates bad branches.
- **Front-end bound** — a cycle where the backend was ready but the frontend couldn't supply instructions. Instruction-cache issues, decoder throughput, micro-op cache misses, complex indirect branches.
- **Back-end bound** — the largest and most common case: the frontend had instructions ready, but the backend couldn't execute them. This splits further into *memory bound* (waiting on loads) and *core bound* (execution port saturation, long-latency divides, dependency chains).

`toplev.py` (from `pmu-tools`) or Intel VTune implement this automatically and hierarchically. You run it once and it tells you, at the top level, which of those four buckets dominates. Then you drill down into the one that dominates. "My code is 60% back-end-bound, of which 50% is memory-bound, of which the dominant category is L3 miss latency" is an actionable diagnosis. "My code is 35% retiring" is different and actionable in a different way.

The other tools worth knowing by name, with what each is for:

- **`perf record` / `perf report`** — sampling profiler. Build flame graphs with Brendan Gregg's `flamegraph.pl` or `perf script` pipelines.
- **`perf c2c`** — diagnose cache-to-cache traffic and false sharing.
- **VTune** (Intel) — the most feature-rich microarchitectural analyzer. Includes memory access analysis, threading analysis, and a polished top-down UI.
- **`likwid`** — a command-line suite for pinning, counter measurement, and predefined "performance groups" (MEM, FLOPS_DP, CACHE, etc.) that bundle related counters. `likwid-perfctr -g MEM ./prog` gives you a memory-bandwidth report immediately.
- **`numactl`, `numastat`** — observe and control NUMA placement.
- **`bpftrace`, `bcc`** — eBPF-based tracing. Useful when the problem is not in your code per se but in how your code interacts with the kernel: syscall latencies, scheduler behavior, I/O patterns. `bpftrace -e 'tracepoint:syscalls:sys_enter_read { @[pid] = count(); }'` and similar one-liners are extraordinarily useful for OS-level diagnosis.
- **`strace -c` or `perf trace`** — a quick summary of which syscalls a program is making and how much time each consumes. Often reveals "why is my program slow?" in ten seconds.
- **ftrace / `trace-cmd`** — kernel-level tracing for context switches, scheduler events, I/O, interrupts.

The boundary between "my program is slow" and "the system is slow" can only be crossed with kernel-aware tools. eBPF in particular has changed what is observable in production: you can attach probes to scheduling events, lock contention, page faults, and syscalls without modifying or restarting the process under observation. Problems that were invisible for decades — long-tail latencies caused by specific kernel behaviors — are now measurable.

## The Practitioner's Loop

If there is a single discipline that holds this together, it is this loop:

1. **Measure first.** Know what your program is doing before you change anything. Hardware counters, top-down, latency histograms, flame graphs. Form a concrete claim: "the program is back-end-bound on L3 misses caused by function X," not "I think function X is slow."
2. **Form a hypothesis.** Based on the measurement, a specific mechanism is responsible. "The hash-map probes are chasing pointers through an arena larger than L3."
3. **Predict the outcome of a change.** If your hypothesis is correct, a specific intervention will change a specific counter in a specific direction. "Switching to open addressing with a power-of-two size and linear probing should cut LLC misses by roughly 3x and raise IPC from 0.7 to ~2."
4. **Change one thing.** Not three. One.
5. **Measure again.** Compare against the prediction. If the counter moved as predicted and wall time followed, your model is confirmed. If the counter moved but wall time didn't, your model was incomplete — something else is now the bottleneck and you go back to step 1. If the counter didn't move, your intervention didn't work and your hypothesis may be wrong.
6. **Decide whether to keep the change.** Measured wins are kept; measured wash or loss is reverted, no matter how elegant the idea was.

This loop is slower than guessing. It is also the only reliable way to make real programs fast. The ratio of "plausible optimizations that don't help" to "plausible optimizations that help" is uncomfortably high, and intuition — even well-trained intuition — generates both kinds freely. The counters are the external check.

A few final threads worth tying off:

On **contextuality**: everything in this essay has conditions. SoA is better than AoS — except when it isn't. Lock-free is faster than mutex — except when it isn't. Huge pages improve performance — except when they spike latency. Vector code is faster than scalar code — except when the data doesn't fit the lanes, or when the conversion cost exceeds the savings, or when the compiler's scalar code was already getting SIMD via autovectorization. The only reliable answer to "is this fast?" is "measured, on this hardware, with this data, under this surrounding workload." Treat any unqualified performance claim — including claims in this essay — as a hypothesis to be checked, not a conclusion to be applied.

On **observability of the invisible**: cache misses, TLB misses, branch mispredictions, false sharing, and page faults do not appear in your source code, your logs, or your stack traces. They appear in counters, and only if you ask. The discipline of performance engineering is, to a large extent, the discipline of knowing *what to ask for* and *what normal looks like*. A branch-miss rate of 1% is fine; 15% is a bug. An LLC miss rate of 1% is fine; 40% means you have capacity problems. An IPC of 0.3 is a disaster on any code that should be retiring anything; an IPC of 3.5 is excellent. These reference points come from experience, and you build them by running the tools on code you understand and seeing what the numbers look like when things are going well, so you recognize them when things aren't.

On the **memory wall**: the single most common surprise for programmers moving from "algorithm and I/O" thinking to "microarchitecture" thinking is that most of their so-called CPU-bound code is actually memory-bound. If you remember one thing from this essay, let it be the ratio: an arithmetic op is one cycle; a DRAM load is three hundred. A program that looks busy at 100% CPU may, in reality, be spending two-thirds of its cycles stalled. The fix is rarely "do less arithmetic." The fix is usually "arrange the data so the hardware does not have to wait for it."

And on **mechanical sympathy**, finally: the goal is not to write assembly by hand. The goal is to hold in your head a sufficiently accurate model of what happens beneath your code that you can predict how a change will play out on the machine, and recognize, from a few counter values, what the machine is actually doing. The compiler, the ISA, the out-of-order core, the caches, the coherence protocol, the virtual memory system, the kernel — each is doing something specific at each moment. If you do not know what, you will fight them without realizing it. If you do, they will mostly work with you, and the remaining gap between your program and the machine's peak will be a gap you can measure, attribute, and close.

That is the work.
