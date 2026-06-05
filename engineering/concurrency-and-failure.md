# Concurrency, state, and failure

Most concurrency bugs and most recovery problems share a root: mutable state that is shared when it should be isolated, or located where a failure can reach it. Tiers 1 and 2 are about taming that state. Tier 3 is about the design calls that decide how much of it you create in the first place.

The rules are language-agnostic. The examples reach across Rust, Python, Go, Node, the JVM, and the Erlang BEAM, because the same shapes recur on every runtime. What changes is whether the runtime enforces the safe thing for you or leaves it to your discipline.

Three tiers, roughly by altitude. Tier 1 is the daily code: the state and concurrency decisions you make inside a single service. Tier 2 is systems: what happens when something fails and how you recover. Tier 3 is design judgment: the calls that shape everything downstream. A five-question pocket version at the end covers the recurring checks, not all ten rules.

## Tier 1: daily code (state and concurrency)

### 1. Shared mutable state is the problem, and there are three cures. Name which one you're using.

There are exactly three ways to make concurrent access to mutable state safe:

- **Isolate**: don't share. Each execution unit gets its own copy and they communicate by passing messages rather than touching common memory. A BEAM process or a `multiprocessing` worker is this cure.
- **Serialize**: allow sharing, but force one-at-a-time access. Locks, queues, and single-owner designs are this cure. A `Mutex`, a Python `Lock`. (An actor's mailbox does both: it serializes access to the actor's own state on top of isolating it from other actors.)
- **Prevent**: make the unsafe access impossible to express, so it can't compile. Rust's borrow checker is this cure: aliasing and mutation can't coexist, and the program won't build if they do.

Watch for deadlock the moment you serialize with more than one lock: threads that take locks in different orders can each hold what the other needs and wait forever. The fix is a single global lock order, fewer locks, or switching to the isolate cure so there's nothing to lock.

Apply it by saying out loud which cure you're using when you see concurrent access. In Rust the borrow checker is prevention, and reaching for `Arc<Mutex<T>>` is you consciously opting back into serialization. In Python a `multiprocessing` worker is isolation, a `Lock` is serialization. If the honest answer is "none of the three," you have a race you haven't hit yet.

### 2. Learn to see the read-modify-write across a yield point. It's the universal race shape.

One pattern underlies almost every race: read a value, then write a value, with a gap in between where something else can run. The gap is the danger. It can be an `await`, a thread switch, a lock release, a network call, or just the space between checking a condition and acting on it.

This bites even where you think you're safe. Single-threaded JavaScript still yields at every `await`, and Python releases the GIL between bytecodes (on a timer, every few milliseconds by default), so another thread can run mid-sequence. "Single-threaded" and "has a GIL" do not mean "atomic."

The shape shows up as check-then-create (`if not exists: create`), increment (`balance = balance + x`), read-then-update, and cache-fill. The moment a gap opens in the middle, ask "what if someone else runs here?" Then close it with a transaction, an atomic compare-and-swap, or routing the whole thing through one owner. Or consciously accept the race.

The gap is also where a task gets cancelled or a timeout fires. A read-modify-write that can be abandoned halfway needs either atomicity or guaranteed cleanup on the cancel path, or it leaves partial state and leaked resources behind. The classic bug: a timeout fires between the read and the write, the caller gives up, but the write still lands and the cleanup never runs. Ask of any abandonable sequence the same thing rule 5 asks of a crash: is it safe to stop here?

### 3. Know whether your scheduler is cooperative or preemptive, and what can starve it.

Node's event loop, Python's asyncio, and a UI main thread schedule cooperatively on a single thread: a task runs until it voluntarily yields, and one CPU-bound or blocking call that never yields freezes everything else on that thread. Thread-pool executors (Scala `Future`s, an Akka dispatcher) are preemptive underneath, so the kernel can interrupt a hot task, but they still starve when CPU-bound or blocking tasks occupy every thread in the pool. Same family of failure, different mechanism: instant freeze versus gradual pool exhaustion.

"Event loop blocked," "dispatcher starved," "UI thread janky," and "Future pool exhausted" are that one phenomenon under different names. The fix is always the same family: get the heavy or blocking work off the shared executor, onto a worker thread, a process pool, a dedicated dispatcher, or `run_in_executor`. And measure it: scheduler lag is a metric worth putting on a dashboard.

## Tier 2: systems (failure and recovery)

### 4. Design the failure model alongside the happy path. Treat blast radius as first-class.

For every component, ask: when this fails, what else goes down with it? The answer is its blast radius: the region a single fault can take down. You want it small and known before the failure happens, not discovered during one.

"Let it crash" is not "be careless." It's "stop writing defensive code for bugs you can't foresee; instead contain the unit so a failure can only kill it, then restart clean." The industry names for that containment: bulkheads and circuit breakers, timeouts, supervised restarts, process and container isolation, idempotency with retry. A crash should be a contained, recoverable event, not a catastrophe, in any language.

The restart is only safe if the crash couldn't have poisoned anything outside the unit. That's what rule 5 is about: containment buys you nothing if the corrupted state was shared, so blast radius and durable-record-outside are the same argument seen from two ends.

### 5. Recovery needs a durable record that lives outside the thing that fails.

Containing the blast radius (rule 4) only helps if the state you need after the crash wasn't inside the blast. So: if the only copy of your state is inside the process that crashes, the crash takes the state down with it. Recovery depends on putting the durable record somewhere the failure can't reach: a different process, a different machine, or durable storage that outlives the process (disk, an external store). This is cure 1 (isolate) applied to failure: the record lives where the crash can't corrupt it, so recovery has something uncorrupted to rebuild from.

This is the database as source of truth, separate from your stateless service. It's the write-ahead log, event sourcing, and crash-only software, where the only way to stop is to crash and the only way to start is to recover, so the recovery path is exercised constantly instead of rotting until the day you need it.

Retry is the other half. A crash between doing the work and recording it guarantees replays, so retry is safe only if doing the work twice equals doing it once. Make the write replay-safe (a dedup key, an upsert, a conditional update) rather than reaching for exactly-once, which distributed systems don't actually give you. Retry without this turns one operation into two: a double charge, a double send.

The design question to ask of any service: can I `kill -9` this and restart it and still be correct? If the answer is no, you have hidden state that needs externalizing. A service that passes this test recovers by being killed and restarted, which is why a fast restart (low mean-time-to-recovery) can keep a system available even when its processes crash often: each crash is brief and clean instead of a long outage.

## Tier 3: design judgment (the calls that matter most)

These five are the decisions you make once, early, that determine how much shared state and how many failure modes Tiers 1 and 2 ever have to deal with. They're upstream of the daily rules.

### 6. The cost of a primitive shapes your architecture more than you'd expect. Know your unit costs.

Cheapness is a license; expense forces sharing; sharing breeds the bugs from Tier 1. Connection pools, thread pools, and worker pools all exist for one reason: some unit was too expensive to have one per request, so you multiplex, and every pool reintroduces contention and shared state. Concretely: a database connection is expensive, so you pool it; the pool is shared mutable state, so two requests can race on a checked-out connection or its transaction state, which is exactly the read-modify-write of rule 2.

Before choosing a decomposition, know what your unit costs. A thread is on the order of a megabyte (reserved stack); a goroutine or virtual thread is kilobytes; a database connection is precious; an object is free. Pick a unit cheap enough to map one per domain-thing, or recognize that you're multiplexing and budget for the sharing costs. Go and Java's Project Loom (virtual threads) exist precisely to make the unit cheap enough to stop pooling.

### 7. Find where your invariants live, and align your boundaries with them.

Where you draw a boundary (call it the "cut") decides which invariants are cheap and which need coordination. An invariant inside one thing is cheap to keep. An invariant spanning two things forces coordination back in. Per-entity atomicity is not cross-entity atomicity, and the gap between them is where the hard bugs live.

This is the deep reason behind DDD aggregate boundaries, transaction scope, microservice boundaries, and sharding keys. The rule: don't split apart things that must stay mutually consistent, and don't bundle things that share no invariant. The most painful bugs and the most painful migrations are invariants that straddle a boundary someone drew wrong. When someone proposes splitting a service, ask what invariant now spans the split, and how you'll keep it consistent across the gap. A transaction? A saga? Redrawing the line?

### 8. Make the safe thing the default, not a discipline to remember.

A discipline you must remember everywhere will be forgotten somewhere. The runtimes that are good at concurrency are good because they make the safe thing the default: the BEAM gives you no way to share mutable state, so you can't forget to isolate. Off such a runtime, every safe behavior (yielding, locking, isolating, sending only immutable data) becomes a standing obligation you carry by hand. This is why prevention (cure 3) beats serialization-by-discipline: a guarantee in the substrate can't be forgotten, a convention can.

Your job as an API or system designer is to move that obligation into the substrate. Make illegal states unrepresentable with types. Make the correct path the path of least resistance. Push the discipline into a wrapper that owns the lock, a constructor that can't produce a bad value, a linter that fails the build. Rust's whole value proposition is this principle applied to memory. The question to ask of any guarantee is which layer actually enforces it: hardware, the language runtime, the type system, or nothing but convention. Only the first three can't be forgotten.

### 9. Prefer correctness that's structural over correctness that's probabilistic.

Rule 8 was about who has to remember the safe thing. This is about what counts as evidence that it's safe. A lost update that passes the naive test and fails under load is the trap. Timing-dependent correctness is a Heisenbug: green in CI, red in production, and brutally hard to reproduce at your desk.

When you can, make a property true by construction (a type, a single owner, a transaction, a total function) rather than betting that "the bad window is small." "It works on my machine" and "it passed the test" are not evidence of a structural guarantee; they're evidence that the bad window didn't open during the test. Determinism is a feature, partly because it's the only kind of correctness you can actually test for.

### 10. Porting the API is not adopting the properties. Verify the guarantee lives in the substrate.

Akka has the actor syntax and the supervision trees, but not the BEAM's isolation and preemption guarantees, because those live in the runtime, not the API. Copying the interface copies the mechanism and leaves the guarantee behind. Concretely: hand a mutable object to another Akka actor in a local message and it's reachable from both, so each actor's one-at-a-time mailbox no longer protects it. The serialization was real; the isolation it assumed was only a convention.

The pattern generalizes to every "we adopted X so we get Y" claim. Be skeptical of cargo-culted patterns: "we use microservices / CQRS / actors / event sourcing, so we get scalability / consistency / resilience." Ask where the guarantee is actually enforced. The same-named tool on a different substrate (a different runtime, consistency model, or failure detector) often carries quietly different guarantees.

## The pocket version

Five questions, applied everywhere:

1. Where is the shared mutable state, and which of the three cures am I using? (isolate / serialize / prevent)
2. Is there a read-modify-write across a yield point here, or heavy work starving a shared executor?
3. When this fails, what's the blast radius, and can I restart it to a known-good state?
4. Which of my invariants spans a boundary, and what keeps it consistent across the gap?
5. Is the safe thing the default, or a discipline someone has to remember?
