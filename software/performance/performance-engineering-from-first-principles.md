# Performance Engineering from First Principles

Begin with the lie.

Imagine a machine that executes one instruction at a time against flat, infinite, uniform-latency memory:

```c
while (running) {
    instr = memory[pc];
    pc += sizeof(instr);
    execute(instr, registers, memory);
}
```

On that machine, performance is mostly algorithm choice, less work, and less I/O. That model is useful for correctness. It is wrong for modern hardware.

The code you write is not what runs. Between source and transistors there is a compiler or JIT, an ISA, decode into micro-ops, out-of-order execution, speculation, caches, TLBs, coherence, the OS, interrupts, and physical topology. Performance engineering is learning where the real machine differs from the imagined one.

## Pipelines: one instruction is not one action

An instruction requires fetch, decode, register reads, address generation, execution, memory access, writeback, and retirement. If the CPU waited for all of that before starting the next instruction, clocks would be slow. So CPUs pipeline work.

A modern x86 core may have a pipeline in the mid-teens to low twenties of stages, retire several micro-ops per cycle, and keep hundreds of micro-ops in flight. Integer add is around 1 cycle. L1 load latency is around 4-5 cycles. A branch mispredict costs roughly 15-20 cycles. At 3 GHz, 1 cycle is about 0.33 ns.

The new goal is not merely fewer instructions. It is keeping useful work flowing through the machine.

## Out-of-order execution: latency hiding needs independence

A sequence like this is a dependency chain:

```asm
add     rax, rbx
imul    rax, rcx
add     rdx, rax
```

The multiply waits for the add. The final add waits for the multiply. Pipelines help less when every instruction depends on the previous one.

Modern CPUs rename registers, schedule ready micro-ops, execute out of order, and retire in order. This hides latency when independent work exists. Pointer chasing defeats it:

```c
for (Node *p = head; p; p = p->next)
    sum += p->value;
```

The next address is unknown until the current load completes. A main-memory miss is about 70-120 ns, or hundreds of cycles. Traversing a random linked list can be limited to about one node per DRAM round trip. It is not compute-bound; it is latency-bound.

## Branch prediction: the CPU guesses the future

A pipeline must fetch future instructions before branches are resolved. So it predicts. Correct predictions are nearly free. Wrong predictions flush speculative work.

```c
if (a[i] < threshold)
    count++;
```

With sorted data, the branch changes direction once and is predictable. With random 50/50 data, it is close to unpredictable. Same algorithm, same number of elements, different hardware behavior.

Measure it:

```bash
perf stat -e cycles,instructions,branches,branch-misses ./program
```

Bad hot-loop output looks like:

```text
1,000,000,000 branches
  486,000,000 branch-misses   # 48.6%
```

A few percent branch misses in a hot loop may matter. Near 50% usually means disaster. Branchless code using `cmov` or masks can help when both sides are cheap and the branch is unpredictable. A predictable branch that skips expensive work is often better kept.

## Front end: the CPU must be fed

The front end fetches instruction bytes, predicts branches, decodes x86 instructions into micro-ops, and feeds the out-of-order back end. It can bottleneck because of large hot code, excessive inlining, instruction-cache misses, indirect calls, virtual dispatch, interpreters, and instruction-TLB misses.

A virtual call may compile to:

```asm
mov     rdi, [rbx+rax*8]   ; object pointer
mov     rcx, [rdi]         ; vtable
call    [rcx+16]           ; indirect call
```

Costs: pointer loads, indirect branch prediction, no inlining, no vectorization across the call, poor locality if objects are scattered.

If the compiler devirtualizes and inlines, it may become straight-line code. Abstractions are zero-cost only when the optimizer erases them. Inlining exposes optimization but increases code size; too much can cause instruction-cache pressure.

Use top-down analysis:

```bash
perf stat --topdown ./workload
```

Example:

```text
retiring          18%
bad speculation    5%
frontend bound    53%
backend bound     24%
```

That says the back end often waits for useful micro-ops.

## Caches: memory is not uniform

Typical rough latencies:

```text
L1 data cache       ~1-2 ns, 4-5 cycles
L2 cache            ~3-5 ns, 10-15 cycles
L3 / LLC            ~12-20 ns, 35-60 cycles
DRAM                ~70-120 ns, 200-350 cycles
remote NUMA memory  worse, often 130-250+ ns
```

The memory wall is the fact that compute improved faster than memory latency. Many apparently compute-bound programs are memory-bound in disguise.

The fundamental data-transfer unit is usually a 64-byte cache line on x86. Loading one 4-byte int that misses fetches the whole 64-byte line. Cache-friendly code uses most bytes fetched and reuses them before eviction.

## Data layout: AoS versus SoA

Array of structs:

```c
struct Particle {
    float x, y, z;
    float vx, vy, vz;
    uint32_t id, flags;
}; // 32 bytes
Particle p[N];
```

If a loop only sums `x`, a 64-byte cache line contains two particles:

```text
[p0.x p0.y p0.z p0.vx p0.vy p0.vz p0.id p0.flags]
[p1.x p1.y p1.z p1.vx p1.vy p1.vz p1.id p1.flags]
```

Only 8 of 64 bytes are useful.

Struct of arrays:

```c
struct Particles {
    float *x, *y, *z;
    float *vx, *vy, *vz;
    uint32_t *id, *flags;
};
```

The `x` array cache line is:

```text
[x0 x1 x2 x3 x4 x5 x6 x7 x8 x9 x10 x11 x12 x13 x14 x15]
```

All 64 bytes are useful. Same algorithm, up to 8x less memory traffic for that loop, often 3x-8x faster in practice. But SoA is not universally better; if every operation uses all fields of one object, AoS may be better.

## Access pattern: row-major versus column-major

C arrays are row-major:

```c
double a[4096][4096];
```

Good:

```c
for (int i = 0; i < 4096; i++)
    for (int j = 0; j < 4096; j++)
        sum += a[i][j];
```

Bad:

```c
for (int j = 0; j < 4096; j++)
    for (int i = 0; i < 4096; i++)
        sum += a[i][j];
```

The good inner loop has 8-byte stride. The bad one has 32768-byte stride. Same arithmetic; the bad version may be 10x slower due to cache and TLB misses.

Measure:

```bash
perf stat -e cycles,instructions,LLC-loads,LLC-load-misses,dTLB-loads,dTLB-load-misses ./matrix
```

Bad signs: low IPC, high `LLC-load-misses`, high `dTLB-load-misses`.

## Prefetch and pointer chasing

Hardware prefetchers detect simple streams like `a[i], a[i+1], ...`. They struggle with data-dependent addresses such as linked lists and hash-table probes. Software prefetch can help but is fragile:

```c
__builtin_prefetch(&a[i + 64]);
```

Too late: useless. Too early: evicted. Too much: pollution. Often the better fix is changing the data structure: linked list to array, pointer tree to B-tree-like layout, chained hash table to flat/open-addressed table.

## TLBs and pages

Virtual addresses must be translated to physical addresses. The TLB caches translations. With 4 KiB pages, random access over a large heap can miss the TLB constantly. A TLB miss may trigger a hardware page-table walk requiring multiple memory accesses.

Measure:

```bash
perf stat -e dTLB-loads,dTLB-load-misses,iTLB-load-misses,page-faults,minor-faults,major-faults ./workload
```

Huge pages increase TLB reach: 512 entries cover 2 MiB with 4 KiB pages but 1 GiB with 2 MiB pages. They can help, but may increase memory waste, complicate NUMA placement, and create latency spikes when transparent huge pages split or collapse.

## Page faults, allocators, and GC

`malloc` may reserve virtual memory without backing it physically. First touch of each page may cause a minor fault: allocate page, zero it, update page tables. This can cost microseconds and hurt tail latency.

Allocators add locks, atomics, metadata, fragmentation, TLB pressure, and remote NUMA allocation. Managed runtimes add GC pauses and object-graph locality problems. CPU profiles often miss time spent blocked, faulting, or stopped by GC.

Observe page faults:

```bash
perf stat -e page-faults,minor-faults,major-faults ./server
sudo bpftrace -e 'tracepoint:exceptions:page_fault_user { @[comm] = count(); }'
```

Use runtime GC logs for Java, Go, .NET, etc.

## SIMD and vectorization

Scalar loop:

```c
for (int i = 0; i < n; i++)
    y[i] += a * x[i];
```

Scalar assembly handles one float. AVX2 can handle eight floats:

```asm
vbroadcastss ymm2, xmm0
.L:
vmovups      ymm1, [rdi+rax]
vfmadd213ps  ymm1, ymm2, [rsi+rax]
vmovups      [rsi+rax], ymm1
add          rax, 32
cmp          rax, rdx
jne          .L
```

Vectorization fails when the compiler cannot prove safety or profitability: aliasing, loop-carried dependencies, calls, branches, non-contiguous access, unknown trip counts, FP semantics, or exceptions.

Ask the compiler:

```bash
clang -O3 -march=native -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -Rpass-analysis=loop-vectorize file.c

gcc -O3 -march=native -fopt-info-vec-optimized -fopt-info-vec-missed file.c
```

Inspect assembly:

```bash
objdump -drwC -Mintel ./program | less
perf record -g ./program
perf annotate
```

SIMD helps only if arithmetic is limiting. If memory bandwidth is limiting, wider arithmetic may do little.

## Threads: parallelism exposes shared bottlenecks

Embarrassingly parallel work still shares memory bandwidth, LLC, TLBs, memory controllers, allocators, scheduler, power budget, and NUMA links. A streaming scan might scale like:

```text
1 thread   18 GB/s
2          35 GB/s
4          68 GB/s
8         105 GB/s
16        112 GB/s
32        110 GB/s
```

No linear speedup because memory bandwidth saturated.

Measure bandwidth:

```bash
likwid-perfctr -C 0-15 -g MEM ./scan
```

If you are near STREAM bandwidth, reduce bytes transferred; do not add threads.

Also inspect scheduler effects:

```bash
perf stat -e task-clock,context-switches,cpu-migrations,page-faults ./workload
```

High migrations destroy cache warmth. Pinning with `taskset` or affinity can help determinism but may hurt load balancing.

## False sharing: sharing cache lines, not variables

Cache coherence works on 64-byte cache lines. If different threads update different variables on the same line, the line ping-pongs between cores.

Bad:

```c++
std::atomic<uint64_t> counters[8];
```

Likely one line:

```text
[c0 c1 c2 c3 c4 c5 c6 c7]
```

Each thread owns a different counter, but every increment invalidates the same line on other cores.

Fix:

```c++
struct alignas(64) Counter {
    std::atomic<uint64_t> value;
    char pad[64 - sizeof(std::atomic<uint64_t>)];
};
Counter counters[8];
```

Find it:

```bash
perf c2c record -g -- ./program
perf c2c report --stdio
```

Look for high HITM on a shared cache line.

## Locks and lock-free code

An uncontended mutex is often just a user-space atomic and is cheap. A contended mutex may spin, enter the kernel via futex, sleep, wake, and suffer scheduler delay.

```bash
strace -c -e futex ./program
perf lock record ./program
perf lock report
```

A plain mutex is fine when contention is low and blocking is acceptable. Lock-free is worth considering when blocking is unacceptable and measurement proves the lock is the bottleneck. Lock-free code can be slower under contention: `lock cmpxchg` causes cache-line ownership traffic and failed CAS retries.

Often the best optimization is not lock-free sharing but less sharing: sharding, per-thread data, batching, single-writer ownership, and periodic reduction.

## NUMA

NUMA exists because one memory system does not scale forever. Each socket has local memory; remote socket memory is accessed over an interconnect and is slower.

Inspect:

```bash
numactl --hardware
numastat -p <pid>
```

Run pinned:

```bash
numactl --cpunodebind=0 --membind=0 ./server
numactl --interleave=all ./batch_job
```

Linux often uses first-touch allocation. If one thread initializes a large array, all pages may land on its NUMA node. Later threads on another socket pay remote latency. Parallel first-touch fixes this when paired with thread pinning.

## Kernel boundary

Syscalls and context switches are not free. A syscall may cost hundreds of ns if hot and nonblocking; blocking costs much more. Tiny reads/writes pay kernel overhead per byte. Batch I/O.

Count syscalls:

```bash
strace -c ./server
perf stat -e syscalls:sys_enter_read,syscalls:sys_enter_write ./program
```

Use eBPF for lower-overhead observability:

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_sendto { @[comm] = count(); }'
sudo runqlat-bpfcc
sudo offcputime-bpfcc -p <pid> 10
```

Kernel bypass avoids the normal kernel data path. DPDK maps NIC rings into user space, uses huge pages, polls instead of interrupts, and processes packets in batches. It is useful when kernel overhead dominates, such as very high packet rates. Costs: dedicated spinning cores, complexity, weaker isolation, and harder operations.

## Diagnosing bound type

Use counters and top-down analysis:

```bash
perf stat -d ./program
perf stat --topdown ./program
perf record -g -- ./program
perf report
perf annotate
```

Interpretation:

- High retiring: optimize instruction count, SIMD, algorithms.
- High bad speculation: branches, indirect calls, exceptions; use branch layout, PGO, branchless code where appropriate.
- High frontend bound: instruction cache/uop delivery/code size/indirect dispatch.
- High backend bound with high bandwidth: memory-bandwidth bound; reduce bytes.
- High backend bound with low bandwidth and many LLC misses: memory-latency bound; improve locality, increase independent work, change data structures.
- High dTLB misses/page faults: reduce random heap access, use arenas or huge pages carefully, pre-touch.
- High context switches/syscalls/off-CPU time: batch, reduce blocking, inspect locks and scheduler.

## Measurement discipline

Benchmarks lie easily. Compile optimized code. Use representative data. Warm up JITs and caches. Avoid dead-code elimination. Control CPU frequency and background noise when needed. Run multiple trials. Inspect distributions, not averages. p99 and p99.9 matter for services.

If you request too many counters, `perf` may multiplex and estimate. Use smaller counter groups for precision.

## The practical model

The useful model is:

```text
generated machine code
running through speculative out-of-order pipelines
fed by instruction and data caches
limited by branch prediction, memory latency/bandwidth, TLBs,
cache coherence, OS scheduling, syscalls, and hardware topology
```

The questions are:

```text
What work is actually being done?
What bytes move?
Where do pipeline slots go?
What is shared?
What is serialized?
What is unpredictable?
What is remote?
What happens in the tail?
```

There is no universally fast code. Branchless code can win on random data and lose on predictable data. SoA can win for scanning fields and lose for object-centric operations. Huge pages can reduce TLB misses and hurt NUMA or memory waste. Lock-free can beat a mutex or collapse under contention. Inlining can remove abstraction overhead or create instruction-cache pressure. More threads can improve throughput or saturate memory bandwidth and worsen latency.

Performance engineering is not a bag of tricks. It is the loop:

```text
measure
form a hardware-level hypothesis
change code or layout
measure again
keep the change only if the workload and counters agree
```

If the explanation is only “it got faster on my laptop,” you do not yet know what happened.
