# TigerStyle

A distillation of TigerBeetle's engineering doctrine, with examples in TypeScript, C, and Rust.

> Source: [`TIGER_STYLE.md`](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md). Lineage: NASA's [Power of Ten — Rules for Developing Safety-Critical Code](https://spinroot.com/gerard/pdf/P10.pdf) by Gerard Holzmann.

---

## What TigerStyle is

TigerStyle is the coding methodology of [TigerBeetle](https://github.com/tigerbeetle/tigerbeetle), the financial-transactions database. It descends from NASA's Power of Ten rules (originally for JPL Mars missions) and extends them for distributed systems.

This document is a language-agnostic distillation. Examples are in TypeScript by default, with C or Rust used where a rule is genuinely about systems-level concerns (memory layout, pointer aliasing, allocation lifetimes). The rules transfer; the syntax does not.

---

## The priority order

Every rule downstream of this ordering:

1. **Safety**
2. **Performance**
3. **Developer experience**

Where mainstream advice ("DRY", "small abstractions", "ergonomic APIs") conflicts with safety, safety wins. The TigerBeetle authors are explicit about this.

> "Style is necessary only where understanding is missing." — *Let Over Lambda*

Style is design, not aesthetics. It is compressed design experience. The rules below exist because someone, somewhere, lost data or shipped a bug when the rule was missing.

---

## Safety

### Simple control flow. No recursion. Bound every loop.

Recursion makes the maximum stack depth invisible at the call site. Use iteration with an explicit upper bound. If a loop genuinely cannot terminate (event loop, server accept loop), assert that.

```typescript
// Forbidden: recursion. Stack depth is not visible at the call site.
function walk(node: Node | null): void {
  if (node) walk(node.next);
}

// Iterative + bounded. The bound is the documentation.
function walk(head: Node | null, NODE_COUNT_MAX: number): void {
  let node = head;
  let steps = 0;
  while (node !== null) {
    if (steps >= NODE_COUNT_MAX) throw new Error("walk: bound exceeded");
    node = node.next;
    steps++;
  }
}
```

For event loops, where non-termination is intentional, assert it:

```typescript
while (true) {
  const event = await queue.next();
  process(event);
}
// The "no return" is intentional and should be reflected in the type:
// function eventLoop(): never { ... }
```

**Why:** the bound is visible in code and asserted at runtime. Stack overflows and runaway loops become impossible by construction.

### Put a limit on everything.

Every queue, every loop, every concurrency level, every cache, every retry budget, every connection pool. If everything has a limit, backpressure emerges automatically — there is nothing that can grow without bound, so nothing to "handle backpressure for".

```typescript
// All limits live together, named consistently, sorted by descending significance.
export const limits = {
  replicas_max: 6,
  clients_max: 1024,
  pipeline_prepare_queue_max: 16,
  request_size_max_bytes: 1024 * 1024,
  request_timeout_ms_max: 30_000,
  retries_max: 5,
} as const;
```

**Why:** unbounded growth is the cause, not the symptom. Capping every dimension at design time forces you to reason about saturation up front.

### Static allocation after init.

Compute worst-case memory at startup, allocate once, never allocate again. In languages without that discipline, simulate it: pre-allocate buffers, pools, and slabs at boot, and forbid runtime allocation on hot paths.

```c
// C: every dimension bounded at compile time, no malloc on hot paths.
#define REPLICAS_MAX 6
#define CLIENTS_MAX  1024
#define PIPELINE_MAX 16

typedef struct {
    Replica replicas[REPLICAS_MAX];
    Client  clients[CLIENTS_MAX];
    Prepare pipeline[PIPELINE_MAX];
    /* every dimension explicit, every byte accounted for at link time */
} Cluster;

static Cluster cluster;  /* one allocation, in .bss */
```

In Rust, the equivalent is allocating bounded `Vec`s once at startup and treating them as fixed-size arrays:

```rust
struct Pool<T, const N: usize> {
    slots: [Option<T>; N],
    free: u32,
}
// Construct once at boot. After that, the pool's capacity never changes.
```

In TypeScript, the discipline is weaker but still useful: pre-create object pools for hot paths instead of `new`-ing in the inner loop.

**Why:** allocation latency is unbounded (page faults, GC, fragmentation). Static allocation makes worst-case latency a property of the design, not the runtime. As a side effect, every queue has a known size, which means backpressure is automatic.

### Assertion density ≥ 2 per function. Split compound assertions.

Assertions detect programmer errors. Unlike *operating* errors, programmer errors mean the code itself is wrong — there is no recovery. The only correct response is to crash, because continuing on corrupt assumptions is worse than stopping. Assertions also serve as load-bearing documentation of invariants.

```typescript
function applyTransfer(
  debit: Account,
  credit: Account,
  amount: bigint,
): void {
  // Pre-conditions — split, never compound.
  assert(amount > 0n);
  assert(amount <= MAX_TRANSFER_AMOUNT);
  assert(debit.id !== credit.id);
  assert(debit.ledger === credit.ledger);

  const debit_after = debit.balance - amount;
  const credit_after = credit.balance + amount;

  // Post-conditions on derived values.
  assert(debit_after >= 0n);
  assert(credit_after >= debit.balance + 1n);

  debit.balance = debit_after;
  credit.balance = credit_after;
}
```

Never write `assert(a && b && c)`. When it fails, the line number tells you which clause broke.

**Why:** assertions catch the bug at the place where the assumption was made, not three frames up where it manifests as garbage output. Two-per-function is a forcing function: if you can't write two, you probably don't understand the function's contract yet.

### Pair assertions across code paths.

For every property you want to enforce, find at least two places to assert it. Validate data right before writing it to disk *and* immediately after reading it back. Validate at the boundary *and* before use.

```typescript
// Path 1: about to persist.
function writeBlock(block: Block, storage: Storage): void {
  assert(block.checksum === hash(block.data));
  assert(block.size === block.data.length);
  storage.write(block);
}

// Path 2: just retrieved.
function readBlock(slot: number, storage: Storage): Block {
  const block = storage.read(slot);
  assert(block.checksum === hash(block.data));   // catches bit rot
  assert(block.size === block.data.length);      // catches truncation
  return block;
}
```

**Why:** redundant checks at independent code paths catch failures the original side missed (corruption, truncation, mis-routed reads). One side fails, the other catches.

### Assert positive *and* negative space.

Bugs hide at the boundary between "valid" and "invalid". Assert what you expect *and* what must not be true.

```typescript
function commitPrepare(prepare: Prepare, state: State): void {
  // Positive space — what we expect.
  assert(prepare.op > state.committed_op);
  assert(prepare.checksum === hash(prepare.body));

  // Negative space — what must NEVER be true.
  assert(prepare.op !== 0);                        // 0 is reserved
  assert(prepare.checksum !== 0n);                 // catches zero-init bugs
  assert(prepare.client_id !== state.replica_id);  // clients ≠ replicas

  state.committed_op = prepare.op;
}
```

**Why:** zero-initialized memory, swapped arguments, and uninitialized struct fields all produce values that pass naive positive checks. Asserting "this can't be the sentinel value" catches them.

### 70-line function limit.

Hard cap. One screen. There are many ways to split a wall of code into 70-line chunks; only a few feel right. Use those.

```typescript
// Bad: 200-line function with seven nested switch statements.

// Good: parent dispatches, leaves do the work.
function handleMessage(replica: Replica, msg: Message): void {
  switch (msg.kind) {
    case "prepare":     onPrepare(replica, msg);    return;
    case "prepare_ok":  onPrepareOk(replica, msg);  return;
    case "commit":      onCommit(replica, msg);     return;
    case "ping":        onPing(replica, msg);       return;
  }
  msg.kind satisfies never;
}
```

**Why:** scrolling to follow a function destroys your working memory. The constraint forces you to find the natural seams in the logic.

### Push ifs up, push fors down.

Centralize control flow in parent functions. Move tight loops over data into leaf functions that have no branches at all.

```typescript
// Bad: branching inside the hot loop.
function processEvents(events: Event[]): void {
  for (const e of events) {
    if (e.kind === "transfer") applyTransfer(e);
    else if (e.kind === "account") createAccount(e);
    else throw new Error(`unknown: ${e.kind}`);
  }
}

// Good: branch ONCE outside, loop branchlessly inside.
function processEvents(events: Event[]): void {
  if (events.length === 0) return;
  switch (events[0].kind) {
    case "transfer": applyTransfers(events as Transfer[]); return;
    case "account":  createAccounts(events as Account[]); return;
  }
}

function applyTransfers(transfers: Transfer[]): void {
  for (const t of transfers) {
    // No outer-kind switch in here. Pure data plumbing.
    apply(t);
  }
}
```

**Why:** CPUs love predictable inner loops. A mispredicted branch per element, multiplied across millions of elements, is real cost. The branch belongs once, in the parent.

### Explicit-sized types.

Use `i32`, `u64`, `i16` — never platform-dependent types like `usize` or `int`. A binary serialized on x86_64 must read identically on aarch64.

```rust
// Bad: layout depends on the platform.
struct Header {
    op: usize,
    flags: usize,
}

// Good: identical on every platform.
#[repr(C)]
struct Header {
    op: u64,
    flags: u32,
    _padding: u32,
}
```

In TypeScript, the analogous discipline is using `bigint` for 64-bit IDs (instead of `number`, which loses precision above 2^53), and being explicit about byte widths in serialization code.

**Why:** silent platform-dependent layout = corruption when nodes of different architectures share data.

### Handle every error.

> "92% of catastrophic distributed-system failures are the result of incorrect handling of non-fatal errors explicitly signaled in software."
> — Yuan et al., *Simple Testing Can Prevent Most Critical Failures*, OSDI '14

Use a type system that forces you to acknowledge errors. Catch nothing silently.

```typescript
// Bad: swallows errors.
async function fetchAccount(id: bigint): Promise<Account | null> {
  try {
    return await db.fetch(id);
  } catch {
    return null;
  }
}

// Good: errors are part of the return type. The compiler forces handling.
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

async function fetchAccount(id: bigint): Promise<Result<Account, FetchError>> {
  // ...
}

// Caller MUST destructure the result.
const result = await fetchAccount(id);
if (!result.ok) {
  // Handle FetchError explicitly. No silent null.
  return;
}
use(result.value);
```

In Rust, this is `Result<T, E>` with `#[must_use]`. In Zig, it's the `error{...}!T` union. The point is: errors are values, and the compiler refuses to let you ignore them.

**Why:** silent failures are the root cause of most production catastrophes in distributed systems. Make the type system enforce that every error is dealt with.

### No library defaults.

Pass options explicitly at every call site. Defaults change silently when the library updates.

```typescript
// Bad: hidden contract with the library. Default cache policy could change.
const result = await fetch(url);

// Good: every option spelled out. Future-proof against default changes.
const result = await fetch(url, {
  method: "GET",
  cache: "no-store",
  redirect: "manual",
  credentials: "omit",
  signal: abortController.signal,
});
```

**Why:** defaults are invisible at the call site. When the library bumps a major version and changes a default, every implicit call silently changes behavior. Explicit options pin the contract.

---

## Performance

### Design-time, not profile-time.

> "The best time to solve performance, to get the huge 1000× wins, is in the design phase, which is precisely when we can't measure or profile."

Performance is a design property. Sketch the resource budget first, code second. Encode the design's invariants as compile-time or boot-time assertions.

```typescript
// Constants and their relationships are checked at startup, not hoped for.
const PIPELINE_MAX = 16;
const CHECKPOINT_INTERVAL = 1024;
const WAL_SLOTS = 2048;

// Boot-time assertion: the WAL must be large enough to hold all in-flight
// prepares plus a checkpoint margin. If this fails, we never start.
function checkInvariants(): void {
  assert(WAL_SLOTS >= CHECKPOINT_INTERVAL + PIPELINE_MAX * 2);
  assert(WAL_SLOTS % CHECKPOINT_INTERVAL === 0);
}
```

**Why:** profiling can shave 20%. Design rewrites win 1000×. The expensive performance bugs are architectural; you cannot profile your way out of "I picked the wrong data structure."

### Back-of-envelope sketches across (network, disk, memory, CPU) × (bandwidth, latency).

Before writing code, write the budget. Optimize the slowest resource first, weighted by frequency of access.

```typescript
// In a comment block above the relevant module:
/*
 * Throughput budget for a single replica:
 *
 * Resource   Bandwidth        Latency
 * ────────── ──────────────── ──────────────
 * Network    1 GbE = 125 MB/s ~0.5 ms RTT
 * Disk       NVMe = 3 GB/s    ~50 µs read
 * Memory     50 GB/s          ~100 ns hit, ~100 ns miss
 * CPU        ~3 GHz, 1 IPC    ~0.3 ns/inst
 *
 * Target: 1M transfers/sec.
 * Per-transfer budget: 1 µs CPU, 0.1 µs network, 0 µs disk (batched).
 *
 * Implication: cannot afford a cache miss per transfer.
 *              Hot data must fit in L2.
 */
```

**Why:** writing the budget down before writing code is the cheapest way to discover that your design is two orders of magnitude off-target.

### Control plane vs data plane.

The control plane decides *what* to do (O(1), can afford to be branchy). The data plane *does it* (O(N), must be branchless). Lift control out of the inner loop.

```typescript
// Control plane: O(1), once per batch. Safety asserts everywhere.
function processBatch(batch: Batch): Result<void, BatchError> {
  assert(batch.events.length > 0);
  assert(batch.events.length <= BATCH_SIZE_MAX);

  switch (batch.kind) {
    case "transfers": return executeTransfers(batch.events as Transfer[]);
    case "accounts":  return createAccounts(batch.events as Account[]);
  }
}

// Data plane: O(N), millions of iterations. Branchless, tight, hot.
function executeTransfers(transfers: Transfer[]): Result<void, BatchError> {
  for (let i = 0; i < transfers.length; i++) {
    const t = transfers[i];
    // No safety branches. The control plane validated the batch shape.
    accounts[t.debit].balance  -= t.amount;
    accounts[t.credit].balance += t.amount;
  }
  return { ok: true, value: undefined };
}
```

**Why:** a single mispredicted branch per iteration, multiplied by N, is real cost. Lift it out and amortize.

### Batch everything.

Amortize the cost of every expensive operation across many small ones.

```typescript
// Bad: per-item disk write. 1ms fsync × 8000 items = 8s.
for (const transfer of transfers) {
  await db.append(transfer);
  await db.fsync();
}

// Good: one fsync amortized over the batch. 1ms / 8000 = 125ns/item.
await db.appendAll(transfers);
await db.fsync();
```

TigerBeetle batches at every level: ~8000 transfers per prepare, 32 prepares per memory flush, 1024 prepares per checkpoint. Each level amortizes the next level's overhead.

**Why:** fixed costs (network round-trips, fsyncs, page faults, syscalls) dominate small operations. Batching turns them into negligible per-item costs.

### CPU as sprinter — extract hot loops without `self`.

When a function holds a reference to a struct, the compiler often cannot prove its fields don't alias other state. Pulling hot loops out into free functions with primitive arguments lets the compiler keep values in registers.

```rust
// Bad: compiler must conservatively reload self.left and self.right
// every iteration in case some method call mutated them.
impl Compaction {
    fn merge(&mut self) {
        while self.left.has_next() && self.right.has_next() {
            // ...
        }
    }
}

// Good: primitive slice arguments, no aliasing risk, registers used freely.
fn merge_inner(left: &[Value], right: &[Value], out: &mut [Value]) -> usize {
    let mut i = 0;
    let mut j = 0;
    let mut k = 0;
    while i < left.len() && j < right.len() {
        if left[i].key <= right[j].key { out[k] = left[i]; i += 1; }
        else                            { out[k] = right[j]; j += 1; }
        k += 1;
    }
    k
}
```

In TypeScript, the same instinct applies: if the inner loop reads `this.foo.bar.baz`, copy it to a local once and loop over the local.

**Why:** aliasing is a register killer. Free functions with primitive args give the compiler the freedom to optimize.

---

## Developer experience: naming

Naming is the longest section of TigerStyle. Names are the API of the codebase to your future self.

### snake_case for everything.

Functions, variables, files. The underscore is the only "space" available; use it.

```typescript
// TigerStyle preference.
function read_sector(slot_index: number): Sector { /* ... */ }
const headers_per_sector = sector_size / header_size;
```

If your project uses `camelCase` (idiomatic in TS/JS), keep that — but apply every other naming rule below regardless of case style.

### Units and qualifiers last, sorted by descending significance.

```typescript
// Good — descending significance: subject → unit → bound.
const latency_ms_max = 100;
const latency_ms_min = 10;
const latency_ms_p99 = 80;
const replicas_max   = 6;
const replicas_min   = 3;

// Bad — adjective first.
const max_latency_ms = 100;
const min_latency_ms = 10;
```

**Why:** alphabetic sort groups related variables. `latency_ms_*` clusters together; `^latency_` greps everything. The naming order matches how you'd describe it in prose: "the maximum latency in milliseconds" → `latency_ms_max`.

### Infuse names with meaning.

A boring name conveys identity; a meaningful name conveys *contract*.

```typescript
// Boring: tells you what the parameter is.
function createCluster(allocator: Allocator): Cluster { /* ... */ }

// Meaningful: tells you the lifetime model.
function createCluster(gpa: Allocator): Cluster { /* ... */ }       // general-purpose; caller must free
function parseConfig(arena: Allocator): Config { /* ... */ }        // arena; bulk-freed
```

Even when the language doesn't have these allocator semantics natively, the principle transfers: pick names that encode the *role* of a value, not just its type.

```typescript
// Bad: type repeated in name.
function send(message: Message, connection: Connection): void;

// Good: name describes role.
function send(reply: Message, peer: Connection): void;
```

### Match character lengths for related variables.

Symmetrical names produce symmetrical code, which is easier to read.

```typescript
// Good: same length, columns line up.
const source_offset = 0;
const target_offset = 16;
copy(target.subarray(target_offset, target_offset + len),
     source.subarray(source_offset, source_offset + len));

// Bad: mismatched lengths break the visual symmetry.
const src_offset  = 0;
const dest_offset = 16;
```

**Why:** a swapped argument or transposed offset is much easier to spot when the surrounding code is visually symmetric.

### Helper prefix matches the calling function.

The call hierarchy should be visible from names alone, before opening any function bodies.

```typescript
function recover(replica: Replica, callback: (r: Replica) => void): void {
  recover_prepares(replica, () => recover_done(replica, callback));
}

function recover_prepares(replica: Replica, done: () => void): void {
  // ...
  recover_prepares_callback(replica, done);
}

function recover_prepares_callback(replica: Replica, done: () => void): void { /* ... */ }
function recover_done(replica: Replica, callback: (r: Replica) => void): void { /* ... */ }
```

**Why:** `read_sector` and `read_sector_callback` tell you the latter is part of the former's flow. You can navigate the codebase by reading file outlines.

### Nouns, not participles.

Pick names that survive being used in metrics, log lines, and section headers.

```typescript
// Bad: adjective/participle. What's `config.preparing_max`?
replica.preparing
config.preparing_max  // awkward

// Good: noun. Composes everywhere.
replica.pipeline
config.pipeline_max
metric: "replica.pipeline.depth"
```

**Why:** nouns compose. Participles need rephrasing every time they leave the code.

### Callbacks last in argument lists.

Argument order should mirror execution order: inputs first, continuation last.

```typescript
function recover(
  replica: Replica,
  storage: Storage,
  callback: (replica: Replica, error: Error | null) => void,
): void { /* ... */ }
```

**Why:** the callback runs last, so it reads last. You scan left-to-right and the code runs left-to-right.

### Order in files matters.

Top-down readability. `main` first. Inside types: fields, then nested types, then methods.

```typescript
// 1. Imports

// 2. Public types
export interface Replica { /* ... */ }

// 3. Constants
const PIPELINE_MAX = 16;

// 4. Public functions (main entry points first)
export function start(config: Config): Replica { /* ... */ }

// 5. Private helpers
function on_prepare(replica: Replica, msg: Prepare): void { /* ... */ }
```

```typescript
// Inside a class/struct: fields, then types, then methods.
class Tracer {
  // 1. Fields
  time: Time;
  process_id: ProcessId;

  // 2. Nested types (if any)
  static readonly SAMPLE_INTERVAL_MS = 100;

  // 3. Methods (constructor first)
  constructor(time: Time) { this.time = time; this.process_id = newId(); }
  trace(event: Event): void { /* ... */ }
}
```

When unsure, sort alphabetically — the descending-significance naming rule means alphabetic order tends to also be semantic order.

---

## Cache invalidation

Bugs in this category come from state diverging across copies, names, or scopes. The fix is to keep state in one place and reference it close to use.

### Don't alias.

Two names for the same value will eventually drift.

```typescript
// Bad: `op` is a snapshot of state.committed_op, but the code uses both.
const op = state.committed_op;
do_work(op);
// ... 30 lines later, after some intervening logic ...
log(`committed: ${state.committed_op}`);  // is `op` still equal to this?

// Good: one name.
do_work(state.committed_op);
log(`committed: ${state.committed_op}`);
```

**Why:** every alias is a chance for the program's understanding of "now" to diverge from reality.

### `*const T` for types > 16 bytes you don't intend to copy.

In C / C++ / Rust, large struct arguments by value cause silent stack copies. Pass `*const T` (or `&T` in Rust) to make non-mutation explicit.

```rust
// Bad: every call copies the entire 256-byte Header onto the stack.
fn validate(header: Header) -> bool { /* ... */ }

// Good: caller and callee share one view.
fn validate(header: &Header) -> bool { /* ... */ }
```

In TypeScript, all object types are reference-passed by default, so this rule is automatic. But in performance-sensitive code, watch for accidental spread (`{...obj}`) or `structuredClone` calls in hot paths.

### In-place initialization via out-pointers.

For complex types, construct them where they will live, rather than building on the stack and moving them.

```c
/* Out-pointer initialization. The struct lives where the caller put it.
 * Pointers into self.fields are stable from the first line. */
int journal_init(Journal *self, Storage *storage, uint8_t replica_id) {
    self->storage = storage;
    self->replica_id = replica_id;
    self->slot_count = WAL_SLOTS;
    /* self->slots is a fixed-size array embedded in self — stable address. */
    return 0;
}

/* Caller decides where it lives. */
static Journal journal;  /* in .bss, address fixed for program lifetime */
journal_init(&journal, &storage, replica_id);
```

In Rust, the equivalent is `MaybeUninit<T>` with in-place writes, used when fields hold self-pointers.

**Why:** if any field of a struct stores a pointer back to the struct itself (or a sibling field), constructing-then-moving invalidates the pointer. In-place init avoids that whole class of bug. The rule is *viral*: once one field needs in-place init, the container does too.

### Reduce return-type dimensionality.

Each step up the type lattice multiplies the number of cases the caller has to handle.

```
void   <  bool   <  number   <  number | undefined   <  Result<number, E>
```

```typescript
// Bad: caller has 3 paths to handle (success, missing, error).
async function getBalance(id: bigint): Promise<Result<bigint | null, DbError>> { /* ... */ }

// Good: split the cases. Existence is a separate question from retrieval.
async function accountExists(id: bigint): Promise<Result<boolean, DbError>> { /* ... */ }
async function getBalance(id: bigint): Promise<Result<bigint, DbError>> { /* ... */ }  // requires the account to exist
```

**Why:** dimensionality is viral. A `T | undefined` return forces every caller up the chain to handle `undefined`. Killing the optionality at the source kills it everywhere.

### POCPOU — place-of-check to place-of-use.

Compute and check variables close to where they are used. Distance between check and use is where bugs hide.

```typescript
// Bad: the slot is checked, then 40 lines pass, then it's used.
const slot = compute_slot(op);
assert(slot < SLOT_COUNT);
// ... 40 lines of unrelated work, possibly mutating SLOT_COUNT ...
self.headers[slot] = header;   // is this still safe?

// Good: check protects the exact next line.
const slot = compute_slot(op);
assert(slot < SLOT_COUNT);
self.headers[slot] = header;
```

**Why:** the human reader maintains "this was checked" in working memory only briefly. The compiler doesn't track it at all. Keeping check-and-use adjacent makes the protection self-evident and locally verifiable.

---

## Off-by-one

### Treat `index`, `count`, and `size` as conceptually distinct types.

They have different semantics: `index` is 0-based, `count` is 1-based, `size` is in unit-bytes. Casting between them requires deliberate work (`+1`, `* unit`).

```typescript
// Branded types make the conversion visible.
type Index = number & { __brand: "index" };
type Count = number & { __brand: "count" };
type Bytes = number & { __brand: "bytes" };

function index_to_count(i: Index): Count { return (i + 1) as Count; }
function count_to_bytes(n: Count, unit: Bytes): Bytes {
  return (n * unit) as Bytes;
}

const i: Index = 0 as Index;
const n: Count = 16 as Count;
const sz: Bytes = (16 * 8) as Bytes;

assert(i < n);                  // index < count ✓
assert(n <= 1024);              // count <= max_count ✓
```

**Why:** swapping `count` and `index` is the single most common off-by-one. Making them distinct types in the type system catches it at compile time.

### Show your intent in division.

When you divide, you have made a decision: round down, round up, or assert exactness. Encode that decision in the call.

```typescript
function div_exact(a: number, b: number): number {
  assert(a % b === 0, `div_exact: ${a} not divisible by ${b}`);
  return a / b;
}

function div_floor(a: number, b: number): number {
  return Math.floor(a / b);
}

function div_ceil(a: number, b: number): number {
  return Math.floor((a + b - 1) / b);
}

// At call sites, the choice is documentation:
const headers_per_sector = div_exact(SECTOR_SIZE, HEADER_SIZE);
const sector_count       = div_ceil(headers_size, SECTOR_SIZE);
const sector_index       = div_floor(slot_index, headers_per_sector);
```

**Why:** raw `/` hides whether rounding is intentional. Named functions force the author to think — and let the reader trust the intent.

---

## By the numbers

- **Indent: 4 spaces.** Visible at distance.
- **Line length: 100 columns, hard.** Two copies fit side-by-side on a 200-column screen. Never go beyond — no horizontal scrolling.
- **Always brace `if`, except single-line:** defends against `goto fail`-style bugs.

```typescript
// Single-line, allowed unbraced (if your language permits).
if (replica === primary) return;

// Multi-line, MUST brace.
if (replica === primary) {
  commit();
}
```

---

## Dependencies & tooling

### Zero dependencies (where you can).

> "Dependencies inevitably lead to supply chain attacks, safety and performance risk, and slow install times. For foundational infrastructure in particular, the cost of any dependency is further amplified throughout the rest of the stack."

The TigerBeetle posture is extreme — they vendor their compiler. Your project probably can't go that far, but the question is worth asking for every dependency: *what is the lifetime cost of this?*

When you do need utilities, prefer to write them yourself in a `stdx`-style local module rather than pulling a small npm package. Owning the surface kills version drift, supply chain risk, and license attrition.

```typescript
// stdx/bounded_array.ts — you own this. No left-pad incident possible.
export class BoundedArray<T> {
  private items: (T | undefined)[];
  private len = 0;
  constructor(public readonly capacity: number) {
    this.items = new Array(capacity);
  }
  push(item: T): void {
    assert(this.len < this.capacity);
    this.items[this.len++] = item;
  }
  // ...
}
```

### One toolbox.

Write your scripts in your project's primary language. `scripts/release.ts`, not `scripts/release.sh`.

```
scripts/
├── release.ts
├── ci.ts
├── changelog.ts
└── deploy.ts
```

**Why:** cross-platform, type-checked, no Bash quoting bugs, no "works on my machine" surprises. As the team grows, dimensionality reduction is force-multiplied.

---

## What's distinctive about TigerStyle

The rules are interesting in isolation, but the meta-shape is what makes the doctrine worth reading even if you'll never write the same kind of system.

1. **Every rule is motivated.** Not "DRY, period" but "aliases increase the probability state gets out of sync." Each rule comes with a *why*. They preach this ("Always say why") and live it.

2. **Style is design, not aesthetics.** *"Style is necessary only where understanding is missing."* It's compressed design experience, not decoration.

3. **Constraints over guidelines.** 70 lines, 100 columns, ≥2 assertions, no recursion. Hard numeric limits — easier to enforce, easier to argue with when wrong. A guideline can be ignored; a number cannot.

4. **It optimizes for the future reader under stress.** Tight scopes, paired checks, named units, explicit types. The implicit reader is a tired engineer at 3 AM, on call.

5. **It rejects clean-code orthodoxy.** No "single responsibility," no "tell don't ask." Functions are big where the problem is big. Abstractions are suspect ("never zero cost"). The 12,000-line `replica.zig` isn't a failure of the style — it *is* the style.

The unifying claim is that **safety, performance, and clarity are not in tension if you design for them up front**. The rules are the up-front design, made portable.

> "It's called TigerBeetle, not only because it's fast, but because it's small!"
