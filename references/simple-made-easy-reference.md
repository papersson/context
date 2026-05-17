# Making Simple Easy — A Reference

A self-contained distillation of the principles. You don't need the original talk to use this.

---

## 1. The core distinction

| | **Simple** | **Easy** |
|---|---|---|
| Root meaning | *one fold/braid* — one role, task, or concept | *to lie near* — nearby, familiar, within reach |
| Opposite | Complex (*braided together*) | Hard (*tortuous*) |
| Nature | **Objective** — you can inspect for entanglement | **Relative** — always "easy for whom?" |
| About | The thing itself | Your relationship to the thing |

**Easy ≠ simple.** Most arguments about technology collapse because people say "simple" when they mean "easy," and "easy" when they mean "familiar to me." Keep these separate or you can never have an objective discussion about the qualities that actually matter.

Two more words worth precision:

- **Complect** *(verb)*: to braid, interleave, or entwine. This is where complexity comes from. Treat it as a bad word — don't do it.
- **Compose** *(verb)*: to place together. Composing *simple* components is how robust software gets built.

---

## 2. The three faces of "easy" (and the one we ignore)

Something is easy when it is near along one of three axes:

1. **At hand** — installed, in the toolkit. (Easy to fix: install it, get it approved.)
2. **Familiar** — near to what you already know. (Easy to fix: learn it.)
3. **Within mental capability** — near to what you can hold in your head.

We are infatuated with #1 and #2 and avoid talking about #3 (ego). But #3 is the one that doesn't yield to effort: **we can learn more, but we can't get much smarter.** Everyone juggles roughly the same small number of balls. The only lever for #3 is to *make the thing simpler* so it comes within reach — you cannot move your brain closer to the complexity.

Employers also over-weight #1 and #2: familiar + at-hand code makes programmers replaceable. Neither has anything to do with whether the code can be understood, relied on, or changed.

---

## 3. Judge the artifact, not the construct

You program with **constructs** (languages, libraries). Users run **artifacts** (the long-lived running system). All the properties that matter — correctness, reliability, debuggability, changeability — are properties of the artifact.

- "I only had to type 16 characters" is a fact about the construct. Irrelevant.
- "Every time we ask it the same question we get a different answer" is a fact about the artifact. Decisive.

**Rule:** evaluate every construct by the complexity of the artifact it *yields*, never by the convenience or familiarity of typing it in.

Complexity you didn't need but your tool introduced is **incidental complexity** — Latin, roughly, for "your fault." You chose the loom; the knotted output is on you.

---

## 4. Why complexity is the binding constraint

- Understanding is hard-capped: you can only hold a few things in mind at once.
- Entanglement is combinatorial: every intertwining forces you to drag a second thing into mind to reason about the first.
- You cannot change what you cannot reason about. Tests and types do not grant this — every field bug *passed the type checker and all the tests*. They are guard rails (safety nets), not steering. They are secondary because they don't touch entanglement.
- Reasoning here means **informal reasoning** ("it logically can't be in this part, so look there first"), not formal proof.
- Ignoring complexity feels fast early and gets slower forever: sprints increasingly redo prior work. Simplicity costs a little ramp-up and then compounds.

---

## 5. Substitution table — prefer the right column

Each "complex" item is complex because it *complects* (braids) the listed concerns. Reach for the simpler tool by default.

| Complex (what it complects) | Simpler alternative |
|---|---|
| **State / objects** — value, identity, time (inextricably) | Values; persistent/immutable collections |
| **Methods** — function + state (+ namespace) | Functions; real namespaces |
| **Vars / mutable variables** — value + time | Managed refs (Clojure/Haskell-style: time abstraction **+** value extraction) |
| **Inheritance** — types together | Polymorphism à la carte |
| **Switch / pattern matching** — many who/what pairs, closed, in one place | Polymorphism à la carte (open, ideally runtime-open) |
| **Syntax** — meaning + order | Data |
| **Imperative loops** — what + how | Set functions |
| **Fold** — operation + order (subtle) | Set functions |
| **Actors** — what + who | Queues |
| **ORM** — logic + representation + ... | Declarative data manipulation (SQL / Datalog / LINQ) |
| **Conditionals scattered as policy** — rules + program structure | Rule systems (e.g. Prolog-style) |
| **Inconsistency / eventual consistency** | Transactions + values |

A note on managed references: they don't make state *simple*, but unlike bare `var`s they (a) warn you state is present and (b) let you **extract a value** out of the time-varying thing. That extraction is your path back to simplicity — without it, a passed reference poisons everything downstream.

---

## 6. Three traps that look like simplicity but aren't

1. **Cardinality ≠ simplicity.** Simple does not mean "only one." More small, untangled things beats a few knotted ones. Simplifying often *increases* the part count.
2. **Code organization ≠ simplicity.** Modules, layers, and separate classes are *enabled by* simplicity, not a source of it. Complex things partitioned and stratified give zero benefit and hide the entanglement ("this module assumes that one never returns 17").
3. **Familiarity ≠ simplicity.** "I already know it" and "it's unentangled" are unrelated. The most dangerous constructs are often the most familiar and the easiest to use.

---

## 7. Designing your own simple constructs

**Abstract** = *draw away from the physical/representational*. It does **not** mean "hide stuff."

Decompose any design along the questions — keeping each strand from touching the others:

- **What** — the operations. Express as **small** sets of function specifications (interfaces / protocols / type classes). Specifications only, defined purely in terms of values and other abstractions. Keep them tiny; giant interfaces resist being broken up. Never let *what* leak *how* — not bluntly (a concrete class where an interface belongs) and not subtly (semantics that dictate the algorithm, e.g. fold's ordering). Cleanly separated, *how* becomes someone else's problem ("database, you figure out how").
- **How** — the implementations. Wire them in via polymorphism constructs, not switches/matches. Each implementation an island. The more declarative, the freer the implementer.
- **Who** — the entities/data and subcomponents. Inject subcomponents as arguments; don't hardwire them. Expect *more*, smaller subcomponents than feels natural. Watch for hidden cross-dependencies between them.
- **When / where** — guard fiercely. Direct A→B calls complect *where* (A must know where B is) and *when* (whenever A runs). Put a **queue** between them.
- **Why** — policy and rules. Don't strew conditionals through the code; gather them into a declarative/rules system that a non-programmer could read. (English-string test DSLs are not this — write code that does the work and can be read.)

**Information is already simple.** The only thing you can do to it is ruin it. Objects exist to wrap I/O devices, not data. Represent data *as data* — maps, sets, sequences. Don't mint a class for every new fact; that ties logic to representation and kills generic, write-once data manipulation.

---

## 8. Complexity that is genuinely not your fault

**Environmental / inherent complexity:** programs contend for memory, CPU, etc. with neighbors and with themselves. *Inherent* ≈ "not your fault." It's an implementation reality, not the customer's problem, and you can't excuse a bad result with "GC problems."

Hard open problem: resource policies **don't compose** ("size the thread pool to the core count" stops working the moment two parts both do it). Splitting these decisions per-component makes things *more* complex, because they need a single, better-informed point of decision. Be aware of this; don't pretend local choices are simplifying it.

---

## 9. Working checklist

**Before choosing a tool**
- [ ] What artifact does this yield — not, how nice is it to type?
- [ ] Default to the right column in §5; justify any move off it.
- [ ] Separate "is it simple?" from "is it familiar/at-hand?" Solve the latter two yourself (editor, tutorials, approval).

**While designing**
- [ ] Run who / what / when / where / why; keep the strands apart.
- [ ] Interfaces small and specification-only; many small subcomponents, injected.
- [ ] Queues between stages; no direct A→B time/place coupling.
- [ ] Policy gathered declaratively, not scattered.
- [ ] Data stays data.

**As a habit**
- [ ] Run entanglement radar: ignore names/semicolons, look for connections between things that could be independent.
- [ ] When state is unavoidable, box it behind a true functional interface (same input → same output) and provide value extraction.
- [ ] Spend the up-front simplicity ramp; expect more, straighter pieces rather than fewer knotted ones.

---

## 10. The one-line summary

Simplicity is a *choice*, made under constant vigilance, by judging artifacts not constructs, refusing to complect, and composing small unentangled pieces — because the only complexity you can afford is the inherent kind.
