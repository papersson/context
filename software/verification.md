# Software verification: context primer

As of 2026-11-03. Paste this at the top of a chat about software verification. It tells you, the model, what I think, which words I use, what I've run, and how I want answers. Section 11, the doing ladder, goes stale first. Claims carry tags: `[src: …]` points to a file in my project folder (key in section 12), `[my view]` is my position, and `[beyond]` is general knowledge that hasn't been checked against a source here.

## 1. How to answer me

- **Name the kind of verification you mean**: example test, property test, stateful property test, fuzzing, mutation testing, deterministic simulation (DST), model checking, types, or proof. "Test it" hides the choice that matters.
- **Give a concrete property or a small example before any category.** I learn from what can be run, and taxonomy is what I'm trying to get away from.
- **Ground the answer in this doc and say when you go beyond it.** I want to know which claims I can push back on.
- **End with one thing I can run today**, sized to an evening or less. Push me toward doing, not reading.
- **Assume a human wrote the code** unless the question is about agents.
- **Don't force links between topics that stand on their own.** For example, preview environments and DST are not two halves of one idea.
- **Prefer my running example (section 5)** when an example is needed. Invent a new one only when it doesn't fit.

## 2. Where I stand

I'm a software engineer into FP, type-driven design and reliability, and I build daily with AI coding agents. What I've run myself:

- Assertions and example tests, including `python -O` silently stripping an assertion `[src: ep1-runs]`.
- Hypothesis properties, including watching a failing case shrink from `{10: 5}, {10: 5}` to `{0: 1}, {0: 1}` `[src: ep2-runs]`.
- A Rust lecture on a framebuffer: exhaustive small-scope tests, metamorphic relations, proptest and cargo-mutants `[src: grid]`.
- A prompt-optimization loop scored by mutation testing `[src: mutation]`.
- A merge sort whose laws I wrote and whose proofs an AI wrote, checked by Bend `[src: bend]`.
- The same agent run with and without a tool to run the tests `[src: agent-claims]`.

What I haven't done yet: written TLA+, run TLC, written a Lean or Dafny proof, or called Z3 directly. My `/coding` workflow has one agent write a failing test, hashes it at red, and a different agent implements against the frozen test `[src: coding-workflow]`.

## 3. Words I use

- **A check = driver + oracle + environment.** The driver produces inputs and event orders. The oracle decides right from wrong. The environment is the world the code runs in. A check is only as strong as its weakest part. `[my view]` The three terms are standard, but the triple as a framing is mine. Its closest relatives in the literature are Ammann and Offutt's RIPR model and Barr et al.'s survey of the oracle problem `[beyond]`.
- **Law, invariant, spec.** A law is a property of a pure function (`decode(encode(x)) == x`). An invariant holds in every reachable state (`sold <= supplied`). A spec is the set of claims, written down.
- **Oracle-free (metamorphic) relations.** "Twice equals once." "Left part, then right part, equals the whole." They're weaker than an oracle, since a function that does nothing passes both, but they can't share the oracle's mistake `[src: grid]`.
- **Environment: fidelity vs control.** Fidelity is how closely it behaves like production. Control comes in two kinds:
  - *construction*: what goes into the environment (hermetic builds, previews);
  - *execution*: the clock, interleavings and faults, which only simulation controls.
  
  Use "test environment" for the place itself, not "test harness" or "testbed" `[src: env-terms]`.
- **Run each check in the smallest environment that contains the bug's cause** (code, dependencies, data shape, deployed shape, real traffic, event order) `[my view]`.
- **Fails open.** A checker that reports green when it checked nothing `[src: mutation]`.

## 4. Which check for which claim

Start from the claim, not the tool. Each row gives the cheapest check that can settle the claim and what that check can't see. If a claim is about the order of events, no check that runs one thing at a time can see it.

| Claim | Cheapest check that settles it | Concrete property | What it can't see |
|---|---|---|---|
| An order is never both paid and refunded | Types (a sum type) | `Paid \| Refunded` instead of two booleans | Transitions over time |
| A payment timeout doesn't mark the order paid | Example test on the error path | timeout → `PaymentPending`, not `Paid` | Inputs nobody wrote down |
| Merging carts loses nothing | Property test | `merged[p] == account[p] + guest[p]` for all carts `[src: ep2-runs]` | Two merges in flight at once |
| A saved cart loads back | Property test (round-trip) | `load(save(c)) == c`. It found JSON turning key `0` into `"0"` `[src: ep2-runs]` | The real database's data shape |
| The request parser never crashes | Coverage-guided fuzzing | no crash on any input. The toy parser crashed at input 10,235, where blind search never did `[src: dst]` | Wrong output that's still valid |
| My tests would notice a bug | Mutation testing | every mutant is killed. A merge using `max` instead of sum passed every test except "nothing is lost" `[src: ep2-runs]` | Bugs that no mutant resembles |
| No two buyers get the last item | Model checking the design, DST on the code, a history check on the real DB | `sold <= supplied` under every interleaving `[src: harness]` | A stateful property test that runs steps one at a time: 201 sequences and 9,718 steps passed `[src: dst]` |
| Sort is correct for every list | Proof | `is_sorted(sort xs)` and `count(sort xs, x) == count(xs, x)` `[src: bend]` | Performance, and the unproven parallel variant |
| Addition mod 2²⁵⁶ is the addition I meant | Symbolic model checking / SMT (Apalache with Z3) | the identity law, counterexample `x = 2^256` | Hypothesis missed it in 100,000 examples `[src: konnov]` |
| The deployed task has its payment key | Deployed shape (preview) | the task definition has `PAYMENT_KEY` `[src: harness]` | Real traffic and data volume |
| The agent's "fixed" is true | Run the test yourself, outside the agent's say-so | the test passes after the change `[src: agent-claims]` | Whatever the test doesn't assert |

## 5. The running example: the tea-shop checkout

A checkout handler reads the stock. If the stock is at least one, it writes back stock minus one and records the order. Two requests arrive a few milliseconds apart: both read 1, both write 0, and both record an order. This is a **lost update**. "Our instances are stateless" doesn't help, because the instances share a database whose default isolation lets a separate read and write interleave `[src: dst]`.

The invariant is written once, by a human: **`sold <= supplied`**.

The snippet below runs. It is also a tiny exhaustive model checker: it tries every schedule of four steps between two requests.

```python
from itertools import product

def checkout(db):                  # one request; each yield lets the other run
    stock = db["stock"]            # read
    yield
    if stock >= 1:
        db["stock"] = stock - 1    # write
        yield
        db["orders"] += 1

def checkout_fixed(db):            # check and write as one step,
    if db["stock"] >= 1:           # like UPDATE ... SET stock = stock - 1
        db["stock"] -= 1           #      WHERE stock > 0
        yield
        db["orders"] += 1

def run(handler, schedule):
    db = {"stock": 1, "orders": 0}            # supplied = 1
    reqs = {"A": handler(db), "B": handler(db)}
    for who in schedule:                      # the schedule picks who steps next
        if who in reqs:
            try: next(reqs[who])
            except StopIteration: del reqs[who]
    for r in reqs.values():                   # let leftovers finish
        for _ in r: pass
    return db

for handler in (checkout, checkout_fixed):
    bad = [s for s in map("".join, product("AB", repeat=4))
           if run(handler, s)["orders"] > 1]  # invariant: sold <= supplied
    print(handler.__name__, f"{len(bad)} of 16 schedules oversell", bad[:3])
```

Output (Python 3, 2026-11-03):

```
checkout 8 of 16 schedules oversell ['ABAA', 'ABAB', 'ABBA']
checkout_fixed 0 of 16 schedules oversell []
```

How each kind of check sees this one bug:

1. **Example test**, one request: stock goes from 1 to 0, and the test passes. One request at a time never collides. *(ran)* `[src: dst]`
2. **Stateful property test** (a Hypothesis state machine with `checkout` and `restock`): 201 sequences and 9,718 steps, all passing. The steps run one after another, so the framework searches which requests happen, not when they overlap. *(ran)* `[src: dst]`
3. **DST**: one seed chooses every delay and interleaving. Seed 8 is the first to oversell. 2,543 of 10,000 seeds fail, the search takes 0.26 s, and seed 62 replays identically, timestamps included. After the fix (one conditional `UPDATE`), 0 of 10,000 seeds fail. *(ran, in the DST film)* `[src: dst]`
4. **Model checking, TLA+ with TLC**: a PlusCal model of two checkouts should return the shortest bad trace (both read 1, both write 0) and then pass every state once the decrement is atomic. *(planned: ladder rung 4)* `[src: harness]`
5. **History check against real Postgres**: run concurrent clients at the default isolation level, record every request and response, and check `sold <= supplied`. Then try the conditional `UPDATE` and `SERIALIZABLE`. This is the only level that tests the database's real semantics instead of a model of them. *(planned: rung 6)* `[src: harness]`

When the DST film adds a network and crashes, it also finds a second bug. The payment succeeds, the reply is slow, and the service times out and restocks, so the customer is charged with no order. "A timeout doesn't mean the call failed. It means you don't know." `[src: dst]`

## 6. Designing so it can be checked

Each rule comes with its reason. Most of a harness's strength comes from design decisions, not from the tools `[src: harness]`.

- **Make illegal states unrepresentable**, because a type turns "tested on some inputs" into "true for all inputs" `[src: minsky]`.
- **Pure core, thin shell** (`decide(state, command) -> events`), because pure code is where property tests, model checking and proofs are cheap.
- **Own the seams: time, randomness, network, disk and threads**, because a test can't control what the code reads directly. Retrofitting the seams is expensive `[src: greef]`.
- **Write the state machine down**, because ordering bugs live in implicit machines (a status column plus scattered `if`s). In the checkout, the bug is the gap between "read stock" and "write stock".
- **Assert the negative space**, because an assertion is an oracle that runs in every environment, simulation and production included. TigerBeetle has more than 3,000 `[src: greef]`.
- **Make operations idempotent and data versioned**, because that makes "applied twice equals applied once" and "old reader, new writer" into testable laws.
- **"This can't be tested" usually means "not yet" or "not easily"**, because testability is an architecture choice `[src: hashimoto]`.

## 7. Who writes what when agents write the code

**Position** `[my view]`: the human owns the oracle. That means the laws, the invariants, the theorem statements, and the decision about which claims exist at all. The agent may write the implementation, the proofs, the generators and the glue. Acceptance must be mechanical, by a checker the agent can't edit. If the laws are wrong or incomplete, the gate passes and the program is still wrong, so the risk left over sits entirely in the spec.

Evidence:

- **bend-sort.** I wrote 43 lines of laws. The AI wrote a 54-line sort and 411 lines of proof, including about 20 helper lemmas I never asked for. `bend PROOF.bend` is the gate, so I never read the proofs. Leaving out the permutation law would have been the classic spec bug, because returning the empty list is sorted `[src: bend]`.
- **"Done" is only as good as the agent's context.** I gave the same model, task and loop twice. With a test tool, it said "Fixed" and was right. Without one, it said "Fixed" and the test still failed. The split held in 3 of 3 runs each `[src: agent-claims]`.
- **The evil genie.** Told to make the tests pass, the agent "eventually deletes the tests, or makes them pass in some trivial way". The more powerful and unyielding the validator, the harder the loop pushes toward gaming it (Wilson) `[src: wilson]`. My `/coding` workflow answers this by hashing the test at red and failing the step if the hash changes or the diff hardcodes outputs `[src: coding-workflow]`.
- **Agents use techniques badly.** Told "use QuickCheck" or "use Lean", agents mostly wrote their usual tests in the new framework, with weak generators and irrelevant properties. No technique instruction clearly beat no instruction `[src: luu]`.
- **Proofs at scale.** Claude proved 239 OSS-Fuzz entry points (123 libraries) free of crashes for inputs under 64 MB, in Rocq, with no human review of the proofs. The trusted base is the RISC-V and Linux models, the theorem statements and the checker. Attempting the proofs surfaced hundreds of bugs `[src: anthropic-fv]`.
- **The checker is the hard part.** In my prompt-optimization loop, the checker failed open twice. First, the generated tests redefined the functions under test, so every mutant survived. Second, the harness defined the tests but never called them, so everything "passed". Only the half of the check that needed tests to fail exposed either one `[src: mutation]`.

## 8. When does green mean it works?

- **It must be deterministic or it doesn't count.** Every random choice comes from a printed seed, so every red result reproduces. A flaky test gets fixed or deleted, never retried `[src: harness]`.
- **Measure the checks.** Coverage says what ran. A mutation score says how much the tests constrain the code `[src: harness]`.
- **Assume a checker fails open until you've seen it go red.** A run against correct code proves nothing on its own `[src: mutation]`.
- **Test the error handlers.** In Yuan et al.'s study of five distributed systems, 92% of catastrophic failures came from mishandling non-fatal errors that the software explicitly signalled. In 58%, simple testing would have caught it: 35% were trivial (an empty handler, aborting on an overly general exception, a TODO), and 23% would have shown up under statement coverage of the handler `[src: yuan]`.
- **Test no more and no less than the system promises.** A read-your-write test is wrong for an eventually consistent store `[src: correctness-notes]`.
- **Every escaped bug improves the harness.** Name the smallest environment that contains its cause. Then ask what was missing: the driver, the oracle or the environment `[src: harness]`.

## 9. What no check gives

- **Only claims someone wrote down get checked.** A property nobody thought of is checked by nothing.
- **No general decision procedure exists.** Real tools restrict the language (types, proofs), sample (tests), or answer "not sure" `[src: halting]`.
- **Production is the only full-fidelity environment.** A harness makes what's left small and observable, not zero `[src: harness]`.
- **Assurance is a budget, not a level.** It's set by what a bug costs. A model of a payment protocol usually pays for itself. A proof of a pricing function rarely does `[src: harness]`.

## 10. Where my sources disagree

These are live questions. Argue them with me rather than settling them for me.

- **Example tests vs search.** Minsky defends "the unreasonable effectiveness of example-based testing" when the type system is expressive. Wilson runs some systems on simulation alone, yet concedes "there is a place where you should just write the stupid unit test" `[src: wilson]`. Where is the line?
- **Do formal methods help agents yet?** Luu found that naming a formal method didn't help, because agents used it badly `[src: luu]`. Anthropic and Theorem report autonomous proofs at scale `[src: anthropic-fv]`. Is that a different task (a fixed, mechanical spec) or a real conflict?
- **What's the best teacher for writing properties?** Konnov says PBT is random and so an unreliable teacher, and that model checking teaches property writing better `[src: konnov]`. That bears directly on my ladder.
- **Simple testing vs heavy machinery.** Yuan says simple tests of error handlers would prevent most catastrophic failures `[src: yuan]`. The DST camp says the dangerous bugs live in interleavings `[src: wilson, greef]`. They may be counting different failures, and the DST film's timeout bug is both.
- **What a proof costs.** For seL4, Minsky says about 25 person-years `[src: minsky]`. The Anthropic article says 2 to build and 20 to verify `[src: anthropic-fv]`. Check the seL4 papers before quoting either figure.

## 11. My doing ladder

1. ✓ Assertions and example tests (episode 1)
2. ✓ Properties and shrinking (episode 2, grid rules)
3. ✓ Mutation testing as a checker (prompt-optimization loop, cargo-mutants)
4. → **Next: a PlusCal model of the double-sale checkout, about 30 lines, checked with TLC.** Read the counterexample trace, then make the decrement atomic and watch every state pass. One evening.
5. A stateful Hypothesis test of the cart or a bank account, about 40 lines.
6. The lost update against real Postgres in a container, with a history check (default isolation, then the conditional `UPDATE`, then `SERIALIZABLE`).
7. Mutation score of the checkout's tests, with and without the stateful test.
8. A first Lean 4 proof of a small law, for example that cart merge is commutative.
9. A solver finds the `add256` identity bug (Z3 directly, or Dafny).

Open question: build rungs 4 to 7 as one "verification ladder" repo, or keep them as separate experiments?

## 12. Sources and tag key

All paths are relative to my Software Verification project folder, not this repository. The talks and papers are the primary sources. `syntheses/` and `context/` overlap and are my digests.

| Tag | File |
|---|---|
| ep1-runs, ep2-runs | verification-series/verification-1/code/RUNS.md, verification-2/code/RUNS.md |
| dst | syntheses/dst-the-last-item-sold-twice.md |
| harness | "A harness for knowing a change works", claude.ai/artifact/FvVxzsM18UfF44VbFyLULd |
| env-terms | briefs/term-for-test-environment-answer.md (vocabulary only; I rejected its "opposite corners" framing) |
| grid | syntheses/grid-rules-property-tests.md |
| mutation | syntheses/mutation-score-as-checker.md, notes/property-test-generator/ |
| bend | syntheses/bend-sort-human-laws-ai-proofs.md |
| agent-claims | syntheses/agent-claims-tests-pass.md |
| coding-workflow | syntheses/coding-workflow-verifier-first.md |
| halting | syntheses/halting-problem.md |
| correctness-notes | notes/ensuring-software-correctness.md |
| wilson, greef, hashimoto, minsky | sources/talks/*.md (excerpts in context/) |
| yuan | sources/papers/osdi14-paper-yuan.pdf (Findings 10 to 12) |
| luu, konnov, anthropic-fv | sources/articles/*.md (summaries, not full text) |
