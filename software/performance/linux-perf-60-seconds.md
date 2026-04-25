# Linux 60-Second Performance Triage

You are doing a fast first-pass diagnostic on a Linux host with a suspected performance issue. Goal: in roughly sixty seconds of wall time, narrow the problem to a subsystem and recommend a deeper tool. You are *not* finding root cause — you are locating where root cause likely lives, so the next step is targeted instead of speculative.

The structure follows Brendan Gregg's 60-second checklist (originally a 2015 Netflix Tech Blog post, expanded in *Systems Performance*, 2nd ed.). Command interpretations below are written for agent use.

---

## Operating mode

You have shell access via the Bash tool on the affected host. Run the ten commands below **in order**. Each is fast (one to two seconds of sampling, ten seconds for the `1`-interval ones). The whole pass should complete inside a minute.

Rules of the road:

- **Read-only.** Do not run anything that changes system state. No restarts, no `kill`, no config writes, no package installs.
- **Tolerate missing tools.** `sysstat` (`mpstat`, `pidstat`, `iostat`, `sar`) may not be installed on minimal images. If a command fails, note it and continue. Do not attempt to install packages.
- **Note privilege limits.** `dmesg` may require root on hardened kernels. If denied, say so.
- **Sample once, not in a loop.** For commands invoked with `1` as an interval (per-second sampling), capture a single interval (or a brief 3–5 second window) and stop. The point is a snapshot, not monitoring.

If the user has pasted output instead of granting shell access, parse what is provided and explicitly list any commands from the checklist that are missing from the paste.

---

## The checklist

### 1. `uptime`

Three load averages: one, five, and fifteen minutes. The *trend* matters more than the absolute number.

- 1-min much higher than 15-min → load is rising; the incident is currently arriving
- 1-min much lower than 15-min → load is falling; incident may be resolving
- All three high and similar → sustained load
- High load with low CPU usage usually means processes blocked in uninterruptible sleep (D-state, typically I/O wait)

### 2. `dmesg -T | tail`

Recent kernel ring-buffer messages. Scan for:

- **OOM killer** invocations (`Out of memory: Killed process …`)
- **Disk errors** (`I/O error`, `ata`, `nvme`, `medium error`)
- **Filesystem errors** (`EXT4-fs error`, `XFS: …`)
- **Network/NIC events** (link up/down flaps, driver resets)
- **Hardware events** (MCE, thermal throttling, ECC)
- **TCP/socket pressure** (`TCP: out of memory`, `nf_conntrack: table full`)

Any of these immediately localizes the problem and short-circuits the rest of the checklist.

### 3. `vmstat -SM 1` (a few seconds)

System-wide snapshot in megabytes. Key columns:

- `r` (run queue): sustained value greater than CPU count → CPU saturation
- `b` (blocked on I/O): persistent non-zero → I/O wait
- `si` / `so` (swap in / out): non-zero with active page-in/out → memory pressure plus swap activity (bad)
- `us` / `sy` / `id` / `wa` / `st`: CPU time breakdown
  - high `wa` → I/O-bound
  - high `sy` → kernel-heavy workload (syscalls, locks, network stack)
  - high `st` → hypervisor stealing cycles (noisy neighbor on a VM)
- `free` / `buff` / `cache`: idle memory vs page cache (page cache is good, not waste)

### 4. `mpstat -P ALL 1` (a few seconds)

Per-CPU breakdown. Reveals patterns invisible to system-wide aggregates:

- One CPU at 100% while others idle → single-threaded bottleneck (or interrupt-pinned to one core)
- All CPUs evenly loaded → parallel work; if not scaling, suspect memory bandwidth or coherence
- High `%irq` / `%soft` concentrated on one CPU → interrupt steering issue (look at `/proc/interrupts`, `irqbalance`)
- Non-trivial `%steal` on any CPU (VM only) → hypervisor contention

### 5. `pidstat 1` (a few seconds)

Per-process CPU consumption, sampled each second. Better than a `top` snapshot for catching bursty processes. Identify:

- Unexpected processes burning CPU
- User vs system time per process — a userland app showing high `%system` often means syscall storms or kernel-side contention
- Kernel threads (`kworker`, `ksoftirqd`, `migration`) at the top → kernel-side work dominating

### 6. `iostat -sxz 1` (a few seconds)

Per-device disk statistics, with idle devices skipped. Critical fields:

- `r/s`, `w/s`: IOPS
- `rMB/s`, `wMB/s`: throughput
- `await` / `r_await` / `w_await`: average I/O completion time including queue. Compare to device class:
  - NVMe ~0.05–0.5 ms
  - SATA SSD ~0.2–1 ms
  - HDD ~5–20 ms
- `aqu-sz`: average queue depth. Sustained high depth with high `await` → device is overwhelmed
- `%util`: percent of time device had at least one outstanding request. Above ~80% suggests saturation on single-queue devices; less informative on multi-queue NVMe (which can have many requests in flight without being saturated)

### 7. `free -m`

Memory in megabytes. Three things to read:

- **`available`** (not `free`): kernel's estimate of memory reclaimable for new allocations without swapping. This is the number that actually matters.
- **`Swap used`**: any non-zero deserves a glance. Cross-reference with vmstat's `si`/`so` — used swap with no current swap I/O is mostly inert; used swap with active swap I/O is bad.
- **`buff/cache`**: file-system cache. Large values are normal and healthy.

### 8. `sar -n DEV 1` (a few seconds)

Per-interface network throughput. Look for:

- Saturation against link rate (1 GbE ≈ 125 MB/s, 10 GbE ≈ 1.25 GB/s, 25 GbE ≈ 3.1 GB/s)
- Packet rate vs throughput ratio: many small packets stress the CPU/kernel more than fewer large ones
- Errors / drops (`rxerr/s`, `txerr/s`, `rxdrop/s`, `txdrop/s`) — non-zero is suspicious

### 9. `sar -n TCP,ETCP 1` (a few seconds)

TCP-level statistics. Key signals:

- `active/s`, `passive/s`: outbound and inbound connection establishment rates — context for what the workload is doing
- `retrans/s`: retransmissions. Sustained non-zero indicates loss somewhere (network path, peer overload, or local TX/RX buffer pressure)
- `iseg/s`, `oseg/s`: total segments — needed to put `retrans/s` in proportion (retrans rate as a fraction of segments is more meaningful than the raw count)

### 10. `top` (one screen, then quit)

Final sanity check. Confirms the picture from the previous commands and surfaces anything missed: a runaway process, a kernel thread consuming CPU, unexpected memory growth, an unfamiliar process name.

---

## Output format

Produce a single triage report in this shape:

```
## Triage summary
<one sentence: subsystem implicated and severity, or "no anomaly detected">

## Findings
- CPU: <observation with the specific number that drove it, or "no anomaly">
- Memory: <...>
- Disk: <...>
- Network: <...>
- Kernel: <dmesg findings, or "no recent kernel events">

## Most likely bottleneck class
<one of: cpu-saturation, cpu-imbalance, memory-pressure, swap-thrashing,
disk-io-bound, network-saturation, kernel-error, hypervisor-contention,
no-anomaly-detected, mixed (specify)>

## Recommended next step
<a single concrete command or methodology, with one-sentence rationale.
Examples:
- "Run `perf stat --topdown -a sleep 10` to attribute the saturated CPU
  cycles to retiring / bad speculation / front-end / back-end."
- "Capture a 30s flame graph: `perf record -F 99 -ag -- sleep 30 &&
  perf script | flamegraph.pl > flame.svg` to localize the hot path."
- "Investigate the OOM event: `journalctl -k --since '1 hour ago' |
  grep -iE 'oom|kill'` to identify the killed process and its peak RSS."
- "Diagnose retrans source with `ss -tin` per-socket and `tcpdump` on
  the affected interface."
- "Apply the USE method per resource (man page: man usemethod or
  brendangregg.com/usemethod.html) — the fast-path metrics are clean,
  so the next step is structured per-resource interrogation.">
```

---

## Rules for the report

- **Cite the number.** For each anomaly, include the specific value observed and the threshold or normal range you compared against. "Run queue of 47 on a 16-CPU box" beats "high run queue."
- **No speculation past the data.** If `%steal` is high, say "hypervisor contention is the likely cause" — do not name the noisy neighbor.
- **A clean pass is a result.** "No obvious bottleneck in the fast-path metrics" is a valid conclusion. It points the user away from steady-state CPU/IO/memory and toward latency tails, application-level profiling, or transient events outside the sample window.
- **Note what this checklist misses.** It is biased toward steady-state, machine-wide issues. It is weak on:
  - Tail latency (a 5 ms p99 spike every 30 seconds will not show up here)
  - GC pauses in managed runtimes (use the runtime's own logs)
  - Lock contention in user code (need `perf lock` or `bpftrace`)
  - Cache / TLB / branch-prediction issues (need `perf stat` with counter events)
  - False sharing (need `perf c2c`)

  If the user's symptom description suggests one of these, say the 60-second pass may be insufficient and recommend the right deeper tool directly.

- **Hand off cleanly.** The recommended next step should be specific enough that the user can copy-paste it. Vague advice ("look into memory") is not a recommendation.
