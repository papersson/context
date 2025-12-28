# Testing, Types, and Distributed Systems: A Primer

---

## Part 1: The Core Trade-off

Testing and type systems serve the same goal: proving your code is correct. They differ in *how* they prove it.

**Types are proofs.** If it compiles, the property holds for all inputs, forever.

**Tests are samples.** They check specific cases—even property-based tests only check thousands of cases, not all possible cases.

**Safety vs Liveness.** Most testable properties are *safety* properties: "nothing bad ever happens." You can point to a specific moment when a safety property is violated. Liveness properties ("something good eventually happens") are harder to test—you can run for a while and check that progress occurred, but you can't prove "eventually" with finite test runs.

The art is knowing which properties to prove with types, which to sample with tests, and which require specialized techniques like simulation testing.

---

## Part 2: What Types Can Prove

| Type System Feature | What It Proves | What You Stop Testing |
|---------------------|----------------|----------------------|
| Static types | "This is an integer, not a string" | Type confusion |
| Refined/branded types | "This integer is positive" | Input validation |
| Sum types + exhaustiveness | "All cases are handled" | Missing case branches |
| Effect tracking | "All errors are handled" | Forgotten error paths |
| Linear/affine types | "Resource used exactly once" | Use-after-close, double-free, forgotten cleanup |
| Dependent types | "Array index in bounds, dimensions match" | Bounds checks, structural mismatches |

**Practical availability:**

- **TypeScript + Effect, Scala + ZIO:** Refined types, sum types, effect tracking
- **Rust:** Affine types (ownership), refined types via newtypes
- **Idris, Agda, Lean:** Dependent types (not mainstream)

### Parse, Don't Validate

Validation happens once, at the system boundary. After that, types carry the proof.

```typescript
// At the boundary — untrusted input enters
function withdrawEndpoint(request: Request): Response {
  const rawAmount: unknown = request.body.amount
  
  // Parse into branded type — validation happens HERE
  const amountResult = PositiveInt.parse(rawAmount)
  if (amountResult._tag === "Left") {
    return { status: 400, body: "Amount must be positive" }
  }
  
  // From here on, type guarantees validity
  return withdrawService.withdraw(userId, amountResult.right)
}

// Deep in domain — no defensive checks needed
function withdraw(amount: PositiveInt): Effect<...> {
  // Type guarantees amount is positive
}
```

---

## Part 3: What Tests Are For

Even with a strong type system, you still need tests for:

| Category | Why Types Can't Help | Testing Approach |
|----------|---------------------|------------------|
| Emergent invariants | "Total money conserved" spans multiple values over time | Property-based testing |
| Round-trip correctness | Types don't verify your parser matches your serializer | Property-based testing |
| Integration boundaries | Types don't know if your SQL query returns what you expect | Integration tests |
| Parsers you wrote | Types apply *after* parsing succeeds, not during | Fuzz testing |
| Ordering-dependent correctness | Types don't understand time or interleavings | DST or fault injection |

### Properties Depend on System Guarantees

This is subtle but critical: **the properties you can test depend on the guarantees your system claims to provide.**

Example: You have a distributed counter. Is this a valid property?

```
increment(counter)
value = read(counter)
assert value reflects the increment
```

**It depends on your consistency model.**

- Under strong consistency (linearizability): Yes, this is valid. If it fails, you have a bug.
- Under eventual consistency: No, this is *not* a valid property. A read might hit a replica that hasn't seen the increment yet. Testing this would produce false failures.

Before writing property tests, ask: "What does my system actually guarantee?" Your tests must align with those guarantees—no stronger, no weaker.

---

## Part 4: Property Identification

The hardest part of property-based testing is knowing what properties to test. Here are practical heuristics.

### Property Heuristics

| Heuristic | Pattern | Example |
|-----------|---------|---------|
| **Round-trip** | `decode(encode(x)) == x` | Serialization, compression, encryption |
| **Idempotence** | `f(f(x)) == f(x)` | Retry-safe operations, cache invalidation |
| **Conservation** | `sum(before) == sum(after)` | Money transfer, inventory movement |
| **Monotonicity** | `f(x) >= x` (or always <=) | Timestamps, sequence numbers, version vectors |
| **Commutativity** | `f(g(x)) == g(f(x))` | Operations that shouldn't depend on order |
| **Bounds** | `min <= f(x) <= max` | Percentages, probabilities, normalized values |
| **Negation of past bugs** | "The bug from last month never happens" | Regression properties |

### Input for Property Discovery

When identifying properties (whether manually or with LLM assistance), you need:

1. **Consistency guarantees / system model:** What does your system claim to provide? Strong consistency? Eventual? Causal?
2. **Past failure scenarios:** What bugs have you had? These are candidates for "negation" properties.
3. **API contract / specification:** What invariants does your interface promise to callers?

Without the consistency model, you'll write properties that are either too strong (false failures) or too weak (miss real bugs).

---

## Part 5: Testing Strategies

### Unit Tests

Always. Test your business logic functions directly. Don't test through HTTP handlers—test the functions handlers call.

### Property-Based Testing

Define invariants, generate random inputs, verify invariants hold across thousands of cases.

```typescript
fc.assert(
  fc.property(transferArbitrary, (transfer) => {
    const totalBefore = accounts.map(a => a.balance).sum()
    execute(transfer)
    const totalAfter = accounts.map(a => a.balance).sum()
    return totalBefore === totalAfter  // Conservation
  })
)
```

**Use when:**
- You have invariants that span operations (conservation, ordering)
- Round-trip serialization
- State machine transitions
- Any "for all X, Y should hold" reasoning

**Skip when:**
- Invariants are single-value constraints (use refined types instead)
- Just CRUD with database constraints enforcing invariants

**Note:** Property tests are mostly *safety* properties—things that must always hold. If you're trying to test "eventually X happens," that's liveness, and property tests aren't the right tool.

### Round-Trip Testing

Deserves special mention. Serialization breaks constantly in subtle ways:

- Floating point precision loss
- Date/timezone handling
- `undefined` vs `null` vs missing key
- BigInt precision loss in JSON
- Unicode normalization
- Enum serialization format changes

```typescript
test("User survives round-trip", () => {
  fc.assert(
    fc.property(userArbitrary, (original) => {
      const serialized = serialize(original)
      const parsed = parse(serialized)
      expect(parsed).toEqual(original)
    })
  )
})
```

This is probably the single highest-value property-based test for any system that serializes data.

### Fuzz Testing

Throw garbage at your code, see if it crashes.

**Use when:**
- You parse files (CSV, images, PDFs, XML)
- You wrote a custom query language or DSL
- You accept webhooks from external services
- You have any C/C++/Rust unsafe code
- You handle user-uploaded content

**Skip when:**
- Your framework handles parsing (JSON via FastAPI/Express)
- You're just passing validated input to an ORM

---

## Part 6: The Ordering Question

Most developers don't need DST or fault injection. The question is:

**Does your correctness depend on ordering, and who handles it?**

```
ORDERING DELEGATED                              ORDERING HANDLED BY YOU
│                                                                      │
▼                                                                      ▼

All writes in         Saga across            Custom event         Consensus
single DB             services               sourcing             protocol
transaction

Database              You handle             You handle           You handle
handles               partial failures       event ordering       everything
interleavings         and compensation       and replay

Regular tests         Property tests +       DST probably         DST definitely
are fine              stress tests           needed               needed
```

### Why Delegation Works

When you wrap operations in a database transaction, the database handles interleavings. Two concurrent transactions might execute in any order, but the database guarantees each sees a consistent view. **There's no interleaving bug your code can have** because your code doesn't control the interleaving.

When you implement your own ordering logic—optimistic concurrency, saga orchestration, event sourcing—you're now responsible for correctness under all possible interleavings. That's when you need specialized testing.

### Examples Where You Handle Ordering

- Optimistic concurrency control you implemented
- Read-modify-write without a transaction
- Read from cache, write to DB (cache invalidation races)
- Saga orchestration across services
- State machine spanning multiple requests
- Event sourcing with custom replay logic
- Any multi-step operation without a transaction wrapping it

### The Best Strategy

Push ordering problems into infrastructure that already solved them:

- Use database transactions instead of application-level locking
- Use transactional outbox instead of dual writes
- Use existing saga frameworks instead of rolling your own
- Use a stream processor (Kafka, Flink) instead of custom event ordering

Then you consume their guarantees instead of implementing your own. This isn't cheating—it's engineering.

---

## Part 7: When Ordering Is Your Problem

If you can't delegate ordering to infrastructure, you need specialized testing. Three approaches, in order of increasing investment.

### Fault Injection

**What it is:** Inject faults into real or production-like systems and observe behavior.

**Mechanism:** Kill processes, partition networks, corrupt disks, pause VMs—then check if invariants still hold.

**Tools:** Jepsen, Chaos Monkey, Gremlin, LitmusChaos

**When to use:**
- Existing system you can't redesign for testability
- Third-party dependencies you can't simulate (Postgres, Redis, S3)
- Limited bandwidth for DST investment
- Bugs likely come from environmental factors (hardware, OS, network)

**Limitation:** Failures may not be reproducible. You know *something* broke, but the exact interleaving that caused it may not be capturable. Debugging is harder.

**Example workflow:**
```bash
# Jepsen-style test
1. Start 5-node cluster
2. Run workload (concurrent writes)
3. Inject partition between nodes 1-2 and 3-4-5
4. Continue workload
5. Heal partition
6. Check: did we lose any acknowledged writes?
```

### Deterministic Simulation Testing (DST)

**What it is:** Control ALL nondeterminism so failures are perfectly reproducible.

**Core mechanism:** Replace nondeterministic operations with simulated versions controlled by a seeded random number generator.

```
Same seed → Same execution → Reproducible failure
```

If a test fails with seed 12345, you can re-run with seed 12345 and get the exact same failure. This makes debugging tractable.

**Nondeterminism sources to control:**

| Source | Visibility | Examples |
|--------|------------|----------|
| Network delays/ordering | Visible (grep imports) | HTTP calls, RPC, message queues |
| Database queries | Visible | SQL, Redis commands |
| Thread scheduling | Hidden | Lock acquisition order, async task ordering |
| Clock/time readings | Hidden | TTLs, retry backoff, token expiration, rate limits, "is this stale?" |
| Random number generation | Visible | Use seeded PRNG |

Time and concurrency are the hard ones. They hide in:
- Cache TTLs
- Retry backoff logic
- Token expiration checks
- Rate limiting windows
- Scheduled jobs
- "Is this record stale?" conditionals

**How DST controls these:**

Network:
```python
# Real code
response = http.post(server, request)

# DST version — same interface, simulated underneath
response = simulated_network.post(server, request)
# Simulator decides: when does this arrive? In what order?
# Decisions driven by seeded RNG — deterministic
```

Thread scheduling:
```python
# Simulator controls which "thread" advances
while pending_tasks:
    task = simulator.pick_next_task(seed)  # Deterministic choice
    task.run_until_yield_point()
# Can explore "Thread 1 first" vs "Thread 2 first" — same seed = same choice
```

**Three implementation strategies:**

| Strategy | How It Works | Trade-off |
|----------|--------------|-----------|
| Application-level | Design system with DST-aware abstractions from start (FoundationDB's Flow) | Most upfront work, most control, fastest tests |
| Runtime-level | Swap in deterministic libraries/runtimes (Rust's madsim, patched Go runtime) | Medium effort, requires compatible ecosystem |
| Machine-level | Custom hypervisor intercepts all nondeterminism (Antithesis) | Zero app changes, requires external tooling |

**Core trade-off:** Upfront design investment vs ease of retrofitting to existing code.

**When to use:**
- Greenfield system where you control the design
- Bugs are from logic/interleavings, not environmental factors
- Reproducibility is worth the investment
- You're building infrastructure others will depend on

### Model Checking

**What it is:** Verify your algorithm design *before* writing code.

**Mechanism:** Write a formal specification (TLA+, Alloy, P), model checker exhaustively explores all possible states and interleavings.

**What it catches that DST cannot:**
- Bugs in your *specification*—logic errors in what you intended to build
- Extremely rare interleavings that random exploration won't hit
- Abstract algorithm flaws independent of implementation

**When to use:**
- Designing a new distributed protocol
- Implementing consensus (Raft, Paxos, etc.)
- Building infrastructure where correctness is critical
- Before investing in implementation

**Limitation:** Tests your spec, not your code. Spec and implementation can drift apart. The text notes: "model checkers don't run your actual code... this risks that your specification and your implementation go out of sync."

**Complementary workflow for greenfield systems:**
```
1. Model check the algorithm (TLA+)
   → Catches design flaws before you write code

2. Implement the code

3. DST the implementation
   → Catches implementation bugs the spec didn't anticipate
   → Verifies implementation matches spec
```

### Decision Tree

```
Your correctness depends on ordering you handle yourself
│
├─► Is this a greenfield system?
│   │
│   ├─► Yes, designing new protocol/algorithm
│   │   └─► Model check design (TLA+) → Implement → DST implementation
│   │
│   └─► Yes, but simpler (just need ordering control)
│       └─► Can you design for testability?
│           ├─► Yes → DST (application or runtime level)
│           └─► No → Fault injection, accept lower reproducibility
│
├─► Is this an existing system?
│   │
│   ├─► Limited bandwidth?
│   │   └─► Fault injection (Jepsen) — pragmatic first step
│   │
│   └─► Have resources to invest?
│       └─► Runtime-level DST (madsim) or machine-level (Antithesis)
│
└─► Are bugs environmental (hardware, OS, network quirks)?
    └─► Fault injection — DST simulates software, not hardware
```

---

## Part 8: Decision Framework

```
Starting a new service
│
├─► Model domain with refined types
│   (Replaces input validation tests)
│
├─► Use sum types for enumerations
│   (Replaces exhaustiveness tests)
│
├─► Track effects in types
│   (Replaces error handling tests)
│
├─► Write unit tests for business logic
│   (Always)
│
├─► Property-based tests for:
│   ├─► Round-trip serialization (highest value)
│   ├─► Conservation laws
│   ├─► Idempotence (if you have retry logic)
│   └─► State machine transitions
│
├─► Fuzz tests if you parse untrusted input
│   (File uploads, custom formats, webhooks)
│
├─► Does correctness depend on ordering?
│   │
│   ├─► Can you delegate to infrastructure?
│   │   └─► Yes → Do that. Regular tests are fine.
│   │
│   └─► No, you handle ordering yourself
│       │
│       ├─► Greenfield + critical system?
│       │   └─► Model check → DST
│       │
│       ├─► Greenfield + moderate investment?
│       │   └─► DST (pick strategy based on ecosystem)
│       │
│       ├─► Existing system + limited bandwidth?
│       │   └─► Fault injection (Jepsen)
│       │
│       └─► Existing system + budget for tooling?
│           └─► Machine-level DST (Antithesis)
```

---

## Part 9: Summary

| Correctness Property | Prove With Types | Sample With Tests |
|---------------------|------------------|-------------------|
| "This value is positive" | Refined types | — |
| "All cases handled" | Sum types | — |
| "All errors handled" | Effect types | — |
| "Resource closed exactly once" | Linear types | — |
| "Serialization round-trips" | — | Property tests |
| "Total conserved across operations" | — | Property tests |
| "Operation is idempotent" | — | Property tests |
| "Parser doesn't crash on garbage" | — | Fuzz tests |
| "Correct under all orderings" | — | DST or fault injection |
| "Algorithm design is sound" | — | Model checking |

---

## Quick Reference: When To Use What

| Situation | Approach |
|-----------|----------|
| Input validation | Refined types + parse don't validate |
| Business logic correctness | Unit tests |
| Serialization | Property-based round-trip tests |
| Invariants spanning operations | Property-based tests |
| Parsing untrusted input | Fuzz tests |
| Ordering handled by database | Regular tests (delegation works) |
| Ordering handled by you (greenfield) | Model check → DST |
| Ordering handled by you (existing) | Fault injection first |
| Third-party dependencies | Fault injection (can't simulate) |
| Need perfect reproducibility | DST |
| Environmental bugs (hardware, OS) | Fault injection |

---

## Key Insights

1. **Types prove, tests sample.** Use the strongest tool that fits.

2. **Parse, don't validate.** Validation at boundary, types carry proof inward.

3. **Properties depend on guarantees.** Your testable properties are constrained by what your system claims to provide. Align tests with consistency model.

4. **Delegate ordering when possible.** Push ordering problems into infrastructure. Consume guarantees instead of implementing them.

5. **DST's power is reproducibility.** Same seed → same execution → same failure. This is what makes distributed bugs debuggable.

6. **Fault injection is the pragmatic path.** Existing system + limited bandwidth + third-party deps → fault injection first.

7. **Time and concurrency are where bugs hide.** Network calls are greppable. Time dependencies (TTLs, backoffs, expiration) hide everywhere.

8. **Model check before you implement.** For new protocols, verify the design before investing in code.
