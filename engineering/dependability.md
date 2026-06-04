# Dependability

> A working taxonomy for reliability and resilience engineering. The umbrella that the worked instances (see [beam.md](./beam.md)) hang under.

## Contents

- [The root concept](#the-root-concept)
- [Attributes](#attributes)
- [Threats](#threats)
- [Means](#means)
- [The MTTF and MTTR lever](#the-mttf-and-mttr-lever)
- [Resilience](#resilience)
- [Security](#security)
- [Cross-cutting patterns](#cross-cutting-patterns)
- [Enforcement layers](#enforcement-layers)
- [Measures](#measures)
- [Worked instances](#worked-instances)
- [Glossary](#glossary)
- [Sources](#sources)

## The root concept

Dependability is the ability to deliver service that can justifiably be trusted. The same idea has a second, operationally sharper phrasing: the ability to avoid service failures that are more frequent or more severe than is acceptable. The two definitions describe one property from two directions. The first is about the warrant for trust: not merely that the system works, but that you have grounds to expect it to keep working. The second is about the failure budget: every real system fails eventually, so dependability is the discipline of keeping failures rare enough and mild enough that the people relying on the service can tolerate them.

Both phrasings come from Avizienis, Laprie, Randell, and Landwehr, whose 2004 taxonomy this document follows throughout. That taxonomy is the spine of everything below.

### Why the word exists

"Dependability" is a deliberately neutral umbrella. It was chosen above more familiar words, and the choice is the whole point. The hazard it avoids is what you might call the everyday-synonym trap: in ordinary speech, "reliable," "available," "safe," and "dependable" all mean roughly "good, doesn't break." Engineering needs them to mean different, sometimes opposed, things. If the umbrella term were "reliability," the field would be naming the whole by one of its parts, and the part it picked would quietly bias every conversation toward continuity of service at the expense of the other goals.

That bias matters because the goals genuinely conflict. [Reliability](#attributes) measures continuity of correct service over an interval (how long the system runs without a single break). [Availability](#attributes) measures readiness for correct service at any given moment (what fraction of the time it is up). These are not the same property, and a system can be excellent at one while terrible at the other. A process that crashes and restarts in fifty milliseconds every hour has dreadful reliability (its mean time to failure is one hour) and outstanding availability (it is down for fifty milliseconds out of every hour, roughly 99.9986% uptime). The BEAM's "let it crash" philosophy makes exactly this trade on purpose: it sacrifices reliability to maximize availability, on the bet that a fast clean restart from a known-good state beats a long uninterrupted run that might be quietly accumulating corruption. See [the MTTF and MTTR lever](#the-mttf-and-mttr-lever) for the arithmetic.

You cannot even state that trade-off without a word that sits above both reliability and availability and treats them as siblings rather than synonyms. That word is dependability. It exists so that "we are deliberately lowering reliability to raise availability" is a coherent engineering sentence instead of a contradiction in terms.

### The taxonomy at a glance

The taxonomy has three branches. Read them as three different questions you can ask about a system:

- **Attributes** are the goals: the dimensions along which you decide whether the service is dependable enough. They answer *what do we want?*
- **Threats** are the impairments: the things that undermine the attributes, arranged as a causal chain. They answer *what goes wrong, and in what order?*
- **Means** are the methods: the techniques you apply to attain the attributes in the face of the threats. They answer *what do we do about it?*

```
dependability
│
├── ATTRIBUTES  ── the goals (what dependable means)
│     ├── Availability       readiness for correct service        (% uptime)
│     ├── Reliability        continuity of correct service        (MTTF)
│     ├── Safety             absence of catastrophic consequences
│     ├── Integrity          absence of improper state alteration
│     └── Maintainability    ability to be repaired / modified    (MTTR)
│           (+ Confidentiality  ──►  joins these to form SECURITY)
│
├── THREATS  ── the impairments (the causal chain)
│     fault ──activation──► error ──propagation──► failure
│     (cause)              (bad state)            (visible deviation)
│     │
│     └── recursion: a component's FAILURE is a FAULT
│                    for the system that contains it
│
└── MEANS  ── the methods (how to attain the attributes)
      ├── Fault Prevention    stop faults being introduced
      ├── Fault Tolerance     deliver correct service despite faults
      ├── Fault Removal       reduce number / severity of faults
      └── Fault Forecasting   estimate present / future fault behavior
```

The three branches are not independent lists; they interlock, and the rest of the document is mostly about how. The means act on the threats to protect the attributes. Each means targets a specific link in the threat chain: fault prevention and fault removal work on the *fault* end (keep faults out, or find and excise the ones that got in), while fault tolerance works downstream, after a fault has already activated into an [error](#threats), to stop that error from propagating into a visible [failure](#threats). Fault forecasting stands slightly apart: it measures rather than acts, telling you how much of the other three you still need.

The recursion line in the Threats branch is worth pausing on, because it is what lets the taxonomy describe systems of any scale with one vocabulary. A failure is a deviation visible at a system's boundary. But one system's boundary is another system's interior: when a disk fails, that failure is, from the database's point of view, a *fault* it must now tolerate. The same three branches apply at every level, and the art of building dependable systems is largely about choosing where to draw the boundaries so that failures stay contained inside a region small enough to absorb them. That region has a name, the [fault-containment region](#threats), and it is defined where the threat chain is.

The branches map onto the sections that follow. [Attributes](#attributes) defines the five goals and the reliability-versus-availability distinction in full. [Threats](#threats) defines fault, error, and failure, the fault classes, and the containment region. [Means](#means) defines the four methods. From there the document specializes: [the MTTF and MTTR lever](#the-mttf-and-mttr-lever) turns the availability goal into arithmetic and shows the two cultures that pull on it, [Resilience](#resilience) extends the whole frame to cover the faults nobody anticipated, and [Cross-cutting patterns](#cross-cutting-patterns) develops the recurring structural ideas (state isolation versus fault containment, the three cures for shared mutable state, the operating-system lineage) that the worked instances such as [beam.md](./beam.md) draw on.

## Attributes

The attributes are the goals of dependability: the dimensions along which you specify what "trustworthy service" means for a given system, and against which you measure whether you got it. The taxonomy names five, plus confidentiality, which joins them to form security. No system maximizes all of them at once; they trade against each other, and a large part of dependability engineering is choosing which to favor and being able to state the choice precisely. The vocabulary here exists so that a trade like "we will sacrifice reliability to buy availability" is a sentence you can actually say.

Each attribute below comes with its standard measure and a concrete example. The measures recur throughout the document; their formal definitions are owned by [Measures](#measures), and the availability formula and the lever it sits on are owned by [The MTTF and MTTR lever](#the-mttf-and-mttr-lever). This section uses the measures by name and defers the arithmetic to those sections.

### Availability

Availability is readiness for correct service: the probability that the system is delivering correct service at an arbitrary instant you sample it. It is an instantaneous, point-in-time property. It does not care whether the system has been up continuously or has flickered up and down a thousand times, only whether it happens to be up right now and what fraction of sampled instants find it up.

The measure is the fraction of time the system is available, usually quoted as a percentage and colloquially as "nines": 99.9% ("three nines") is about 8.8 hours of downtime per year, 99.999% ("five nines") about 5.3 minutes per year. Availability aggregates two underlying quantities, how rarely the system breaks (MTTF) and how fast it recovers (MTTR); the relationship is owned by [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

Concrete example: a stateless web request handler behind a load balancer. If a worker crashes and is replaced in 50 ms, the load balancer routes around the gap and a client sampling the service almost never finds it down. The service is highly available even though individual workers die routinely.

### Reliability

Reliability is continuity of correct service over an interval: the probability that the system delivers correct service throughout a whole window of time `[0, t]` without a single failure in that window. It is an interval property, not a point property. The distinction from availability is exact and load-bearing: availability asks "are you up now?", reliability asks "have you stayed up the entire time?". A system can score well on one and badly on the other.

The measure is MTTF (mean time to failure): the expected length of correct operation before the next failure. Higher MTTF means longer unbroken runs. For systems that must complete an uninterruptable mission, the relevant figure is the reliability function `R(t)`, the probability of surviving to time `t`; both are owned by [Measures](#measures).

Concrete example: flight-control software during a single flight. What matters is not the percentage of years it is up but the probability of zero failures across the next several hours of continuous operation. A restart, however fast, is a failed mission. Reliability, not availability, is the attribute being specified.

### Safety

Safety is the absence of catastrophic consequences for the users and the environment. It is not about correctness in general; it singles out a subset of failures, the ones with catastrophic cost, and demands their absence. A system can fail frequently and still be safe, provided none of its failures are catastrophic, and a system can be unsafe while rarely failing if its rare failures are catastrophic.

The measure is the rate or probability of catastrophic failures specifically, often expressed as a target such as "no more than 10⁻⁹ catastrophic failures per hour of operation" for life-critical systems. Safety analysis partitions the failure space into catastrophic and benign and bounds only the catastrophic part.

Concrete example: a railway signaling system whose design ensures that any detected fault drives every signal to red. Trains stop, which is a service failure (no trains move), but no collision occurs. The system is unavailable yet safe. This is the canonical fail-safe posture, and it shows directly why safety and the other attributes are distinct goals rather than facets of one.

### Integrity

Integrity is the absence of improper state alterations: the system's data and internal state are changed only through legitimate, intended operations, never corrupted by faulty or unauthorized ones. "Improper" covers both accidental corruption (a bit flip, a wild write, a partial update left half-applied by a crash) and malicious tampering, which is why integrity belongs both to dependability and, paired with access control, to security.

The measure is less standardized than the timing attributes; it is typically expressed as the probability of undetected corruption, or operationally as the strength of the detection and rejection machinery (checksums, write-ahead logging, assertions that halt on a violated invariant, end-to-end verification). The engineering target is usually not "corruption never happens" but "corruption is never accepted silently."

Concrete example: a storage engine that checksums every block and refuses to serve a block whose checksum fails. Corruption on disk does not become corruption returned to the caller. The shared design rule across the dependability cultures in this document — "crash rather than continue on corrupt state," better a dead node than a corrupted one running — is an integrity-first stance: a halt is treated as preferable to propagating a bad state, trading availability for integrity at the moment of detection.

### Maintainability

Maintainability is the ability of the system to undergo repair and modification: how readily a failed system can be restored to correct service, and how readily it can be changed to fix faults or meet new requirements. It is the attribute that governs recovery speed, and through recovery speed it feeds directly into availability.

The measure is MTTR (mean time to repair): the expected time from failure to restored correct service. Lower MTTR is better. MTTR spans whatever the recovery path actually requires: detecting the failure, locating the cause, applying the repair, and bringing the service back. The recovery granularity matters as much as the raw number; restarting one [BEAM process](#glossary) in microseconds and rescheduling a whole Kubernetes pod in tens of seconds are both MTTR reductions, separated by orders of magnitude and by the size of what must be rebuilt.

Concrete example: "let it crash" supervision in the BEAM. When a worker hits an error, it is killed and a supervisor restarts it from a known-good initial state in well under a millisecond. The system does not attempt in-place diagnosis or repair of the corrupted process; it discards and replaces. That is maintainability engineered for minimal MTTR, and it is the recovery half of the availability story explored in [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

### Confidentiality and Security

Confidentiality is the absence of unauthorized disclosure of information: data is readable only by parties entitled to read it. It is the one attribute in this group that is purely a security concern rather than a general dependability concern, since accidental disclosure to no one in particular is rarely the worry; the threat is an adversary.

Security is not a sixth peer attribute but a composite: the conjunction of confidentiality, integrity, and availability, the three viewed specifically under the assumption of a malicious actor rather than random faults. Integrity and availability thus appear in both groupings, read once against accidental faults (dependability) and once against deliberate attack (security). The full treatment, including the threat-model difference that makes security its own discipline, is owned by [Security](#security).

### Reliability is not availability

The two timing attributes are the pair most often conflated, and keeping them separate is what lets you describe the BEAM's central trade.

Consider a process that crashes once an hour and is restarted by its supervisor in 50 ms.

- Its **reliability is terrible.** MTTF is one hour; the probability of running a multi-hour interval without a failure is low. If the workload were a single uninterruptable mission lasting hours, this system would almost certainly fail it.
- Its **availability is excellent.** It is down for 50 ms out of every ~3,600,050 ms, roughly 99.9986% available — comfortably past four nines. A client sampling it at a random instant essentially always finds it up.

Same system, same crashes, opposite verdicts on the two attributes. The resolution is that they measure different things: reliability integrates over an interval and a single failure anywhere in the interval breaks it; availability samples a point and only asks whether that point is up.

This is exactly the trade "let it crash" makes on purpose. The BEAM does not try to keep a process running through an error (that would be a reliability play). It lets the process die fast and rebuilds it fast, minimizing MTTR and maximizing availability while deliberately accepting a low MTTF. The strategy raises availability *by* lowering reliability — it converts long uncertain runs into short certain ones plus fast recovery. You cannot even state this trade without two separate words for the two attributes, which is one reason the umbrella term [dependability](#the-root-concept) was chosen over "reliability": "reliability" names only one attribute and cannot describe a system that sacrifices that attribute to improve the whole.

### Attributes trade against each other

The five attributes are not independently maximizable; pushing one often costs another, and a dependability specification is largely a statement of which to favor.

- **Safe but unavailable.** The fail-safe brake above maximizes safety by failing closed. Every detected fault stops the trains, so availability drops with each fault; the design buys absence-of-catastrophe at the cost of readiness-for-service. The opposite posture, fail-operational, keeps running through faults to preserve availability and must then work much harder to stay safe.
- **Available but unreliable.** The crash-restart process above is the inverse: it spends reliability to buy availability.
- **Integrity over availability.** The "halt on detected corruption" rule spends availability (the node goes down) to protect integrity (no bad state escapes). A system that instead limped along on suspect state would be more available and less trustworthy.
- **Maintainability shaping availability.** Because availability aggregates MTTF and MTTR, investing in fast recovery (maintainability) is one of the two ways to raise availability, the other being to break less often. The two routes are different engineering cultures and are the subject of [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

The practical consequence: "make it dependable" is underspecified until you say which attributes, to what level, against which threats. The rest of this document is largely about the [means](#means) for hitting a chosen attribute profile and the [threats](#threats) that profile must withstand.

## Threats

A *threat*, in dependability terminology, is anything that can impair the delivery of correct service. The taxonomy organizes threats into a causal chain of three concepts: **fault**, **error**, and **failure**. These are not synonyms for "bug" used loosely. They name three distinct positions in a cause-and-effect sequence, and the discipline of keeping them separate is what lets you reason about where to intervene.

### The fault -> error -> failure chain

The chain runs:

```
FAULT  --activation-->  ERROR  --propagation-->  FAILURE
(cause)                 (wrong state)            (visible deviation)
```

**Fault.** A fault is the adjudged or hypothesized *cause* of an error. It is the underlying defect: a coding mistake, a malformed input, a cosmic-ray bit flip, a loose connector, a wrong configuration value. A fault can sit in the system indefinitely without doing anything. A division that mishandles a zero denominator is a fault from the moment it is written, but it does nothing until a zero arrives. A fault in this quiescent state is **dormant** (also called *latent*, though that word is more commonly attached to errors; see below). It becomes relevant only when something exercises it.

**Activation** is the event that turns a dormant fault into an error. It is the moment the faulty code path runs, the bad input reaches the parser, the marginal hardware crosses its threshold. The same fault may be activated by some inputs and never by others, which is why faults can lurk through years of testing and surface only under a rare production workload. The activation pattern depends on the fault and the conditions that trigger it jointly, so a fault that is statistically almost never activated is, in practice, almost as good as absent until conditions change.

**Error.** An error is the part of the system state that is incorrect: a wrong value in a register, a corrupted record, a violated invariant, a data structure that no longer satisfies its own consistency rules. The error is the fault *manifested in state*. It is internal and, by itself, still invisible from outside the system. A bank balance that has been computed wrong is an error the instant it is stored, whether or not anyone has read it yet.

**Propagation** is the mechanism by which an error spreads. A wrong value gets read and used in another computation, producing more wrong values; a corrupted structure is passed to another module that trusts it; a bad message is sent to a peer. Propagation is what turns a small, contained error into a cascade, and it is precisely what containment mechanisms (below) exist to stop. Propagation within a component moves an error toward that component's interface; propagation *across* the interface is what produces a failure.

**Failure.** A failure (more fully, a *service failure*) is the event at which delivered service deviates from correct service as seen at the system's external interface. The error has propagated all the way to the boundary and become observable: a wrong answer returned to a caller, a request that hangs, a crash, a transaction that commits the wrong amount. Correct service means service that implements the system function; a failure is a transition from correct to incorrect service, and *service restoration* is the reverse transition.

The three terms are positional, not intrinsic. The same defect is a fault when you consider its cause, an error when you consider its effect on state, and a failure when you consider its effect at the interface. Asking "is this a fault or a failure?" about a thing in isolation is a category error: the answer depends on which boundary you are looking across.

### The recursion: one component's failure is the enclosing system's fault

The boundary-relative nature of these terms produces the single most useful structural property in the taxonomy. **A component's failure is a fault for the system that contains it.**

When a subsystem fails at its interface, that failure becomes an *external fault* presented to whatever consumes the subsystem. A disk that returns corrupt sectors has failed as a disk; to the filesystem above it, that corrupt return is an input fault to be tolerated or to be activated into a filesystem-level error. A microservice that returns a 500 has failed as a service; to its caller, that 500 is a fault the caller must handle. The chain therefore nests: fault -> error -> failure at level *n* hands a fresh fault to level *n+1*, which may absorb it (no error, no failure) or activate it into its own error and eventually its own failure.

This recursion is the formal reason fault tolerance is even possible. If a component's failure were simply the whole system's failure, there would be nothing to do. Because the failure of a part is only a *fault* to the whole, the whole gets a chance to detect it and recover before it propagates to the system interface. A BEAM supervisor restarting a crashed worker is exactly this: the worker process *failed*, that failure is a fault the supervisor handles, and the system as a whole never fails. The same structure explains why fault-tolerance techniques compose: each layer's recovery converts a lower-level failure back into a non-event for the layer above.

### Fault classes

Faults are classified along several independent axes. A given real fault occupies one position on each axis simultaneously; the axes are orthogonal, not mutually exclusive.

**Phase of creation: development vs. operational.** *Development faults* are introduced during design, coding, and construction: a logic bug, a wrong algorithm, a build misconfiguration, a flaw baked into hardware at fabrication. They are present before the system is put into service. *Operational faults* arise during use: a transient voltage spike, a worn-out component, an operator typo, a malformed packet from a peer. The distinction matters because the *means* that address them differ. Development faults are the target of fault removal (verification, testing) and fault prevention; operational faults are largely the province of fault tolerance, since you cannot test away a power glitch that has not happened yet.

**System boundary: internal vs. external.** An *internal fault* originates within the system boundary (a bug in your own code, a failing component you own). An *external fault* originates outside it and crosses the interface inward: bad input from a client, a failed dependency, environmental interference, a malicious request. The recursion above generates external faults wholesale: every dependency's failure is an external fault to you.

**Dimension: hardware vs. software.** *Hardware faults* affect physical components: aging, wear, electromigration, radiation-induced bit flips, manufacturing defects. *Software faults* are defects in code or data, always development faults in origin even when they activate only under operational conditions. The practical asymmetry is that hardware faults are dominated by *permanent* and *random* physical processes, while software faults are deterministic with respect to state: the same code with the same inputs and the same state reproduces the same fault every time, which is what makes software faults testable in principle even when they are elusive in practice.

**Persistence: transient, intermittent, permanent.** A *permanent* fault, once present, stays until removed: a logic bug, a burned-out chip. A *transient* fault appears once, triggered by a temporary condition (a single cosmic-ray strike, a one-off race window), and is gone before you look for it; the underlying hardware or code is fine afterward. An *intermittent* fault recurs irregularly without an obvious permanent cause, often a marginal hardware component or a timing-sensitive race that activates only under specific, hard-to-reproduce conditions. Intermittent faults are the expensive ones: they survive long enough to cause repeated failures but vanish whenever you try to isolate them.

#### Bohrbug vs. Heisenbug

A specialization of the persistence axis for software faults, formalized and popularized by Jim Gray in *Why Do Computers Stop and What Can Be Done About It?* (1985). Gray's report coined "Bohrbug" and framed the Bohrbug/Heisenbug hypothesis; the term "Heisenbug" itself predates the report.

- A **Bohrbug** is a solid, deterministic software fault, named after the Bohr atom model: it activates reproducibly under a well-defined set of conditions. Give it the same input and state and it fails every time. Bohrbugs are unpleasant but tractable: because they reproduce, testing and debugging can corner them, and fault removal eventually eliminates them.
- A **Heisenbug** is a software fault whose activation depends on subtle, fragile conditions: timing, memory layout, scheduling order, uninitialized values that happen to be benign under a debugger. Named for the observer effect, it changes or disappears when you try to observe it, because attaching a debugger, adding logging, or recompiling perturbs exactly the timing or layout that triggered it.

The distinction has a sharp operational consequence Gray drew out: **Heisenbugs are why "just restart it" works.** A Bohrbug will fail again immediately on the same input after a restart, so restarting buys nothing. A Heisenbug depends on a transient confluence of conditions unlikely to recur on the next attempt, so a fresh process with a clean state often gets past the point that just failed. This is the empirical foundation under "let it crash" (see the BEAM worked instance, [./beam.md](./beam.md)): in mature, well-tested production software the residual faults that survive into operation are disproportionately Heisenbugs, and a cheap fast restart converts most of them into non-events. The strategy works *because* of the fault distribution it assumes, and it does nothing for a Bohrbug except restart-loop.

### Errors: latent vs. detected

An error has a second state distinction that mirrors the dormant/active one for faults. An error is **latent** (also *undetected*) as long as it is present in the state but no detection mechanism has flagged it. It is **detected** once a checking mechanism, an assertion, a checksum, a type check, a consistency audit, an exception, recognizes that the state is wrong. A latent error is the dangerous one: it can propagate freely, corrupting more state and getting closer to the interface, for as long as it stays unnoticed. The entire purpose of error detection (the first half of fault tolerance, the other being recovery) is to shorten the interval between activation and detection so that recovery can act before propagation reaches the boundary. An undetected error that silently produces a wrong external result is the worst case: a failure with no warning, sometimes called a *silent* or *Byzantine* failure depending on context.

The design stance "crash rather than continue on corrupt state," shared by both the BEAM and TigerBeetle cultures, is precisely a decision about latent errors: convert every detectable error into an immediate, loud, contained failure (a crash) rather than risk it staying latent and propagating. A dense assertion is a detection mechanism that refuses to let an error remain latent.

### Failures: domain, severity, and consistency

Not all failures are equivalent, and the taxonomy classifies them so that you can specify which kinds a system must avoid and which it may tolerate.

**Failure domain.** The *domain* describes *what* deviates:

- **Content (value) failures**: the delivered value is wrong. The function returns, the request completes, but the answer is incorrect.
- **Timing failures**: the value may be correct but it arrives at the wrong time, too late (a missed deadline, a hung request) or, in real-time systems, too early. A failure that is both wrong in value and wrong in timing collapses to the worst case: a *halt* failure, where service simply stops (the system delivers nothing), is the limiting timing failure.

**Consistency across users.** When multiple consumers observe a service, a failure is **consistent** if every consumer sees the same (wrong) result, and **inconsistent** (*Byzantine*) if different consumers see different, mutually contradictory results. Inconsistent failures are the hardest to tolerate because no single consumer can detect the disagreement alone; they are the failure class that consensus protocols and the state-machine-replication approach (Schneider 1990) are built to survive.

**Severity.** Failures are graded by consequence, from *minor* (cost comparable to the benefit of correct service) up to *catastrophic* (cost incommensurably higher, e.g. loss of life or irrecoverable data destruction). The severity grading is what connects the threats branch to the *safety* attribute: a system is safe when its failures stay below the catastrophic threshold, regardless of how often the merely-minor ones occur. The grading also drives where you spend: you tolerate cheap frequent failures and prevent rare expensive ones.

### Fault-containment regions

A **fault-containment region** (equivalently, an **error-confinement region**) is a boundary within the system across which an error is *guaranteed* not to propagate. It is the structural unit of fault tolerance: it draws a line and asserts that whatever goes wrong on the inside cannot corrupt the state on the outside.

The two names emphasize the two readings of the same boundary. As a *fault-containment* region, it bounds the consequences of a fault: the fault may activate and produce errors inside, but those errors are confined. As an *error-confinement* region, it names the propagation guarantee directly: errors stay in. The guarantee is only as strong as the mechanism that enforces it, and the strength of that mechanism is exactly what distinguishes the substrates this corpus compares. A BEAM process is a fault-containment region enforced by the language runtime (no expressible shared mutable reference can carry an error across the boundary). An OS process is one enforced by hardware (the MMU refuses the cross-boundary memory access). A Java thread is *not* a reliable fault-containment region, because the shared heap gives an error a path across the boundary that nothing prevents. The containment is real only when something, hardware, runtime, type system, makes propagation across the boundary impossible rather than merely discouraged.

How big a region you draw determines how much one fault can take down, the **blast radius**, and the discipline of building systems out of well-chosen containment regions is the **bulkhead** pattern, named for the watertight compartments that keep a breach in one section of a ship from flooding the rest. These metaphors and the full account of how state isolation mechanically *produces* fault containment belong to the cross-cutting treatment; see [Cross-cutting patterns](#cross-cutting-patterns). For the purposes of the threats branch, the load-bearing point is narrower: a fault-containment region is the place where the recursion above is made to pay off, the boundary at which a contained failure becomes a mere fault for the enclosing system to handle instead of a failure of the whole.

## Means

The *means* are the methods by which dependability is attained. The taxonomy names four, and they form two complementary pairs. Fault **prevention** and fault **tolerance** provide the ability to deliver service that can be trusted: one keeps faults out, the other works through the faults that get in anyway. Fault **removal** and fault **forecasting** provide the confidence that the trust is justified: one reduces faults that are present, the other estimates how many remain and what they will do. Real systems apply all four; the interesting question is the *balance*, which is the subject of [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

The four are defined here and exemplified with the worked instances this corpus returns to throughout: the BEAM, TigerBeetle/TigerStyle, and state-machine replication.

### Fault prevention

Fault prevention stops faults from being introduced into the system in the first place. It operates before and during construction, at development time, and its instruments are quality-control disciplines applied to the process of building the thing: structured design, code review, type systems, coding standards, and the deliberate exclusion of constructs known to spawn faults.

The mechanism is constraint. Every prevention technique narrows the space of programs you are allowed to write so that large classes of fault become unexpressible. A strong static type system rejects programs that confuse an integer for a pointer or call a method that does not exist; the corresponding faults cannot reach a running system because the program containing them does not compile. Dijkstra's argument in [*Go To Statement Considered Harmful*](#sources) is a prevention argument: remove the construct whose undisciplined use makes control flow impossible to reason about, and the faults that ride on that confusion go with it. The same logic drives MISRA C, the [Power of Ten](#sources) rules for safety-critical code, and the prohibition of dynamic allocation after startup in avionics software.

The **TigerStyle** discipline behind TigerBeetle is a concentrated example of fault prevention as a culture. Its rules are constraints chosen to make whole fault classes impossible rather than merely unlikely:

- **Bound everything.** Every loop has a fixed upper limit, every queue a fixed capacity, every buffer a compile-time size. An unbounded loop is a potential hang and an unbounded buffer a potential overflow; bounding both at the source removes the fault class instead of detecting its symptoms later.
- **Static allocation.** All memory is allocated at startup and never afterward. There is no allocator on the hot path to fail, fragment, or block, and out-of-memory becomes a startup condition rather than a runtime surprise.
- **Assertions as executable invariants.** Dense assertions state what must be true at each point, so a violated invariant halts execution at the fault rather than letting a corrupted [error](#threats) propagate into a [failure](#threats).
- **Explicit, named limits.** Magic numbers become named constants carrying the constraint that produced them, which keeps the bound auditable.

Prevention has a ceiling. It addresses faults whose shape you can anticipate and constrain against. It does nothing for the fault you did not foresee, the hardware that flips a bit, or the operating condition no one specified. That residue is the domain of the next means.

### Fault tolerance

Fault tolerance delivers correct service despite faults that are present and active. It assumes prevention is incomplete (it always is) and arranges for the system to keep working, or to recover quickly, when a fault activates into an [error](#threats). Tolerance operates at runtime, on the live system, after the fault has already done something.

Every fault-tolerance scheme decomposes into two phases.

1. **Error detection** identifies that the system state is wrong. Detection mechanisms include assertions, type and range checks, watchdog timers, heartbeats, checksums and error-correcting codes, acceptance tests on a result, and the comparison of redundant computations. A fault that is never detected is tolerated by accident at best; detection is the precondition for everything that follows.
2. **Recovery** transforms the erroneous state back into a state from which correct service can resume. Recovery itself splits into *error handling*, which eliminates the error from the state, and *fault handling*, which prevents the same fault from activating again (for example by reconfiguring around a failed component).

#### Redundancy: the raw material of tolerance

Tolerance requires redundancy, because to deliver correct service through a fault you need something extra to fall back on or to check against. Redundancy comes in three kinds, distinguished by what resource is duplicated.

| Kind | What is replicated | Cost paid | Typical use |
|---|---|---|---|
| **Spatial** (space) | Hardware or computation, run in parallel | Extra hardware, extra power, copies of state | Triple modular redundancy; replicated servers; RAID mirroring |
| **Temporal** (time) | The same computation, repeated | Extra latency | Retrying a transient failure; re-executing on a single node to catch a transient fault |
| **Informational** (information) | Extra bits derived from the data | Storage and bandwidth for the redundant bits | Checksums, ECC memory, erasure coding, parity |

The choice among them follows the fault you are tolerating. Information redundancy catches and corrects bit-level corruption cheaply but assumes the computation itself is sound. Time redundancy handles *transient* faults (a glitch that will not recur on retry) but is useless against a deterministic *Bohrbug* that will fail identically every time. Space redundancy handles permanent faults and, when the replicas are independent, the transient and even some design faults the others do not share; it is the most powerful and the most expensive.

#### Recovery strategies

Once an error is detected, three strategies move the system back to correct service, distinguished by their direction in time relative to the error.

- **Rollback (backward recovery).** Return to a state saved before the error: a checkpoint, a snapshot, an uncommitted transaction's pre-image. The system retries from there. Rollback is general and requires no understanding of what the error was, but it discards work and needs saved state to roll back to. Database transaction abort is the canonical case.
- **Roll-forward (forward recovery).** Construct a new correct state ahead of the error without returning to a prior one, typically by reconstructing the missing information. Erasure-coded storage rebuilding a lost block from parity is forward recovery: it never had the block to roll back to, it computes a correct replacement.
- **Compensation.** Carry enough redundancy in the live state that the error can be *masked* in place, with no rollback and no reconstruction step. Triple modular redundancy voting out a faulty replica's output is compensation: the correct answer is produced continuously, and the error never becomes a visible failure. For long-running transactions that cannot hold locks to completion, *compensating transactions* (a semantic undo, like issuing a refund for a charge that should not have happened) are the application-level form.

#### The BEAM: tolerance through isolation plus supervision

The BEAM is fault tolerance built on [state isolation](#cross-cutting-patterns) rather than on detection of specific errors. Each [BEAM process](#glossary) has its own private heap and shares no mutable memory with any other, so an error inside one process cannot corrupt another's state. When a process hits an error it crashes; a *supervisor* process detects the death and restarts the failed worker from a known-good initial state. Detection is generic (the process died), recovery is rollback to the supervisor's initial child specification, and the [fault-containment region](#threats) is the single process: a few hundred bytes, recovered in microseconds, with a [blast radius](#glossary) of exactly one execution unit.

What makes this a recovery *guarantee* rather than merely a recovery *mechanism* is the isolation underneath it. Because nothing was shared, a crash cannot have left damage outside the crashed process, so restarting from a clean state is sufficient by construction. The full ladder (process, preemption, isolation, copying message-passing, serialized state access, let-it-crash) and the rung-by-rung contrast with Akka are developed in [beam.md](./beam.md). The point for the taxonomy: "let it crash" is a tolerance strategy that trades [reliability](#attributes) for [availability](#attributes), and it works only because isolation makes the rollback sound.

#### State-machine replication: tolerance through agreement

The strongest spatial-redundancy scheme for software is *state-machine replication* (SMR). Model the service as a deterministic state machine: given the same sequence of input commands, it produces the same sequence of states and outputs. Run several replicas of that machine, feed every replica the identical command sequence, and each independently computes the same result. If one replica fails, the others have the answer. Schneider's [tutorial](#sources) is the canonical statement: the entire problem reduces to making every non-faulty replica agree on the same total order of commands.

That agreement is *consensus*, and it is the hard core of the technique. A consensus protocol (Viewstamped Replication, Paxos, Raft) lets a set of replicas agree on each command despite some of them crashing, despite messages being lost, delayed, or reordered, and despite the network partitioning. Oki and Liskov's [Viewstamped Replication](#sources) was an early protocol that integrated replication with the consensus needed to drive it. SMR tolerates the failure of a minority of replicas (typically up to *f* failures with 2*f*+1 replicas for crash faults) while presenting a single, consistent service to clients.

SMR and the BEAM are complementary, not competing. The BEAM contains and recovers from faults *within* a node at microsecond granularity; SMR keeps the *service* available when an entire node is lost, at the cost of running and coordinating replicas. TigerBeetle uses exactly this division: a deterministic, single-threaded core for high-integrity local execution, replicated by consensus for availability across nodes. Its preferred phrasing, "better a dead node than a corrupted one running," is an SMR design stance: a replica that detects an internal error should crash and let consensus cover its absence rather than continue on suspect state.

### Fault removal

Fault removal reduces the number and severity of faults that are present in the system. Where prevention keeps faults from entering, removal goes looking for the ones that got in and takes them out. It runs in two settings: *during development* (verify, diagnose, correct) and *during use* (corrective and preventive maintenance on the deployed system).

Development-time removal follows a verify-diagnose-correct cycle. *Verification* checks whether the system satisfies its properties; when a check fails, *diagnosis* locates the fault responsible, and *correction* removes it. Verification techniques span a spectrum of rigor and cost.

- **Testing.** Exercise the system on selected inputs and check the outputs. Testing is the workhorse of removal: cheap, scalable, and able to find faults in real implementations rather than models. Its limit is coverage. Testing shows the presence of faults on the inputs you tried, never their absence on the inputs you did not. The leverage is in input selection. Yuan et al., [*Simple Testing Can Prevent Most Critical Failures*](#sources), found that a large fraction of catastrophic distributed-system failures were reachable by tests that simply exercised error-handling code paths at all, which most test suites neglect.
- **Static analysis.** Examine the program without running it: linters, type checkers (here type systems serve removal, flagging faults in existing code, as well as prevention), data-flow analysis, and model checkers that explore a system's reachable states for invariant violations. Holzmann's SPIN and the [Power of Ten](#sources) rules are built around making code amenable to this kind of mechanical checking.
- **Formal methods.** Prove, against a mathematical model of the system, that a property holds for *all* inputs and executions. This is the only verification that yields absence rather than mere presence of faults, and it is correspondingly expensive, bounded by the gap between the verified model and the deployed code.

#### Deterministic simulation testing

A technique that has become central to high-integrity systems and deserves naming on its own: **deterministic simulation testing** (DST). The whole system is built to run inside a simulated world whose every source of nondeterminism (clock, scheduling, network message order and delay, disk and node faults) is supplied by a controllable, seeded pseudo-random generator. The simulation injects faults aggressively (drop messages, partition the network, crash and restart replicas, corrupt disk sectors) and checks that the system's invariants survive.

DST's defining property is *reproducibility*. Because the entire run is a deterministic function of a seed, any failure the simulator finds can be replayed exactly from that seed, every time, which collapses the usual nightmare of debugging a once-seen concurrency or distributed-systems Heisenbug into a deterministic, steppable trace. Running the simulation at accelerated time and across enormous numbers of seeds compresses years of rare fault combinations into hours of wall-clock testing. TigerBeetle is designed around DST end to end; the deterministic, statically allocated, single-threaded core described under prevention exists in part to make this exhaustive simulation feasible. DST also depends on the dense assertions from TigerStyle: an assertion is the oracle that tells the simulator an invariant has been violated, so prevention discipline and removal technique reinforce each other.

#### Maintenance

Fault removal during use is maintenance. *Corrective* maintenance removes faults that have produced reported errors; *preventive* maintenance removes faults uncovered (for instance by analysis or by removal performed on a sibling system) before they activate in this one. A delivered patch is corrective removal; the rollout of that patch ahead of any local incident is preventive removal.

### Fault forecasting

Fault forecasting estimates the present number, the future incidence, and the likely consequences of faults. The other three means change the system's fault behavior; forecasting *measures and predicts* it, and so it is the means that produces the evidence behind any claim of dependability. It evaluates dependability in two ways.

- **Qualitative evaluation** identifies and ranks the failure modes, or the combinations of events, that would lead to system failure. Failure Mode and Effects Analysis (FMEA) and fault-tree analysis are the standard instruments: they enumerate how the system can fail and trace each failure back to its contributing faults.
- **Quantitative evaluation** assigns probabilities or rates to those failures: estimating [MTTF, MTTR](#measures), and the resulting [availability and reliability](#attributes). Reliability modeling builds these estimates from component failure rates and the system's redundancy structure (reliability block diagrams, Markov models, stochastic Petri nets), and field data refines them over time.

Modeling predicts from assumed failure rates. To get *empirical* numbers, and to test whether the model's assumptions hold under real activation, forecasting reaches for fault injection.

- **Fault injection** deliberately introduces faults (corrupted inputs, killed processes, induced bit-flips, delayed or dropped messages) and observes how the system responds. It serves forecasting by measuring detection coverage and recovery behavior under controlled, deliberately triggered faults, and it doubles as a removal technique when the injected fault exposes a real defect in the tolerance machinery. The DST faults described above are fault injection inside a deterministic simulator; classical fault injection runs against the real system.
- **Chaos engineering** is fault injection moved into production, at system scale. Pioneered as Netflix's Chaos Monkey and described by Basiri et al. in [*Chaos Engineering*](#sources), it injects realistic faults (terminating instances, degrading the network, exhausting resources) into the live system to discover, empirically, whether the system's actual dependability matches what was designed and modeled. Its premise is that the only trustworthy forecast of how a complex production system behaves under fault is an observation of that system under fault. This connects to [resilience](#resilience): chaos engineering probes for the gap between work-as-imagined and work-as-done, surfacing failure modes no FMEA enumerated.

The four means interlock. Prevention and removal reduce the faults that tolerance must handle at runtime; tolerance covers the residue that prevention and removal inevitably miss; forecasting measures what that residue is and whether the tolerance machinery actually copes with it, feeding the result back into the next round of prevention and removal. How a given system *weights* these four, in particular whether it invests primarily in keeping faults out (prevention) or in recovering fast when they activate (tolerance), is the central design choice examined in [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

## The MTTF and MTTR lever

This is where [the attributes](#attributes) (what you want) meet [the means](#means) (how you get it). Every preceding section names a goal or names a method. This one is the gear that connects them, and it does so through a single approximate identity that every availability decision can be read against.

### The availability formula

A repairable system spends its life alternating between two states: working and being repaired. Two metrics summarize that cycle.

- **MTTF** (mean time to failure): the average length of a working interval. How rarely it breaks. (Defined fully in [Measures](#measures).)
- **MTTR** (mean time to repair): the average length of a repair interval. How fast it recovers.
- **MTBF** (mean time between failures): the average length of one full cycle for a repairable system, `MTBF = MTTF + MTTR`. For systems where repair time is negligible against uptime, MTBF and MTTF are used almost interchangeably, which is a common source of confusion; keep them distinct when MTTR is large.

Steady-state availability is the fraction of time the system is in the working state:

```
A ≈ MTTF / (MTTF + MTTR)
```

The approximation sign matters. The identity holds exactly only under stationary assumptions (failures and repairs drawn from fixed distributions, no wear-out trend, no correlated failures). Real systems violate these, so treat `A` as a planning estimate, not a guarantee. It is still the right back-of-envelope tool, because it makes the central fact obvious: **availability depends only on the *ratio* of uptime to downtime, not on either quantity alone.** A system with MTTF of one hour and MTTR of 50 milliseconds has the same availability as one with MTTF of one year and MTTR of about 7.3 minutes. They break wildly differently. They are up the same fraction of the time.

That decoupling is the whole reason [dependability](#the-root-concept) needs Availability and Reliability as *separate* attributes rather than one. Reliability is MTTF alone (continuity of correct service over an interval). Availability is the ratio. A process that crashes and restarts in 50 ms every hour has terrible reliability (MTTF of one hour) and excellent availability:

```
A ≈ 3600 / (3600 + 0.05) ≈ 0.99998611   ≈ 99.9986%
```

You cannot even *state* that trade ("I will accept frequent failures in exchange for fast recovery") without an umbrella term that holds both attributes at once. This is the concrete payoff of choosing "dependability" over "reliability" as the root: the BEAM's "let it crash" is, in these terms, a deliberate sacrifice of Reliability to maximize Availability, and the formula is what lets you say so precisely.

### The nines

Availability targets are conventionally quoted as "nines": the number of leading 9s in the percentage. Each additional nine cuts allowed downtime by a factor of ten.

| Availability | "Nines" | Downtime / year | Downtime / day |
|---|---:|---|---|
| 90% | one nine | 36.5 days | 2.4 hours |
| 99% | two nines | 3.65 days | 14.4 minutes |
| 99.9% | three nines | 8.77 hours | 1.44 minutes |
| 99.99% | four nines | 52.6 minutes | 8.64 seconds |
| 99.999% | five nines | 5.26 minutes | 864 ms |
| 99.9999% | six nines | 31.5 seconds | 86.4 ms |

Two things to read off the table. First, the downtime budget is brutal at the top: five nines leaves about five minutes per year for *all* causes combined (crashes, deploys, config errors, dependency outages, the repair itself). Second, because `A` is a ratio, you can hit a given row from very different places. Five nines is reachable by an MTTF of years with an MTTR of minutes (a rarely-failing system repaired by hand), or by an MTTF of minutes with an MTTR of milliseconds (a frequently-failing system repaired automatically in-process). Same row, opposite engineering cultures. That is the lever.

### The two levers

The formula has two inputs you can push on. Each corresponds to a different one of [the means](#means), and each names a real engineering culture.

**Raise MTTF — break less often.** This is [fault prevention](#means) (and fault removal): keep faults out of the system, and verify away the ones that slip in, so working intervals get longer. The exemplar is the TigerStyle culture behind TigerBeetle: bound every loop and every queue, allocate all memory statically at startup, assert invariants densely (two assertions per function as a floor), and subject the whole system to deterministic simulation testing that replays injected faults against a single-threaded deterministic core. The bet is that a fault prevented never has to be recovered from. The cost is paid up front, in developer effort and in design constraints, and it buys a long MTTF. (See the prevention-leaning culture in [Cross-cutting patterns](#cross-cutting-patterns) under the three cures.)

**Lower MTTR — recover faster.** This is [fault tolerance](#means): assume faults will occur at runtime, contain them, and restore correct service quickly. The exemplar is the BEAM. Its recovery is fast on three axes that compound:

- *Latency:* recovery is a process restart, microseconds, not a reboot or a redeploy.
- *Locus:* recovery happens *in the runtime*, under a supervisor that is already resident, with no external orchestrator in the loop.
- *Blast radius:* the unit of failure and restart is one [BEAM process](#glossary) with its own private heap, so the [fault-containment region](#threats) is a single execution unit. Recovering it disturbs nothing else.

The bet is the inverse of TigerStyle's: rather than push MTTF toward infinity, push MTTR toward zero, and accept a short MTTF as the price. A handler that crashes every hour but restarts in 50 ms still clears five nines, as the worked example above shows. The cost is the machinery that makes restart *safe* to do reflexively (state isolation, so a crash cannot have corrupted a neighbor) which lives in the runtime, not the application code. The thesis that state isolation is what upgrades restart from a mechanism into a guarantee is developed in [beam.md](./beam.md) and in [Cross-cutting patterns](#cross-cutting-patterns).

These are not exclusive. A real system pushes both inputs. But they trade off in *attention and architecture*, and most systems and cultures lean primarily on one. The prevention lean and the tolerance lean are the two cultures of dependability mapped in detail under [Cross-cutting patterns](#cross-cutting-patterns); both, notably, share the same priority order (Safety > Performance > Developer-experience) and the same final rule (crash rather than continue on corrupt state). They disagree on which means buys the most availability per unit of effort.

### Kubernetes: the same lever, coarse

Kubernetes lowers MTTR, the same lever the BEAM pulls, which makes the comparison precise rather than rhetorical. A liveness probe detects a failed container, the controller schedules a replacement, and desired state is restored without human intervention. That is fault tolerance via automated recovery, identical in *kind* to OTP supervision.

It differs in *granularity* on exactly the three axes listed above:

| | BEAM | Kubernetes |
|---|---|---|
| Recovery latency | microseconds | seconds to minutes (probe interval + reschedule + image pull + startup) |
| Locus of recovery | in-runtime, resident supervisor | external control loop, out of process |
| Blast radius (containment region) | one process, private heap | one pod (often a whole service instance and its in-memory state) |

The lever is the same; the resolution is three to six orders of magnitude coarser. A BEAM process restart loses the state of one in-flight request. A pod restart loses every connection and every piece of in-memory state that pod was holding, and takes seconds to minutes to come back, during which the formula is counting that downtime against your nines. Neither is "better" in the abstract: the BEAM's fine grain costs you a language runtime that forbids shared mutable references, while Kubernetes' coarse grain works for *any* containerized process regardless of language or internal design. They sit at different points on one axis (recovery granularity), pulling the one lever (MTTR), under the one formula. That is the value of stating the lever explicitly: once you see Kubernetes and OTP supervision as the same move at different granularities, "which recovery mechanism" becomes a question about acceptable MTTR and acceptable blast radius, answerable against the downtime budget in the nines table rather than by preference.

## Resilience

Classic fault tolerance (see [Means](#means)) is built against a list. You enumerate the faults you expect (a disk fails, a node partitions, a message is lost), and you engineer detection and recovery for each. The faults outside the list are, by construction, unhandled. Resilience is the concept that addresses what happens when reality serves a fault you never wrote down.

### Laprie's extension

Jean-Claude Laprie, a co-author of the dependability taxonomy this document is built on, proposed resilience in 2008 as a deliberate extension of that taxonomy rather than a competing idea. His one-line definition: resilience is **the persistence of dependability when facing changes**.

The operative word is *changes*. The original taxonomy treats a system as a fixed object with a fixed fault model: given these components and these anticipated faults, deliver trustworthy service. But real systems change while running. Load shifts by orders of magnitude, dependencies get upgraded under you, the deployment topology mutates, usage patterns drift away from anything the designers tested, new failure modes appear that no one enumerated. Laprie's question is whether dependability survives those changes, not just whether it holds under the conditions assumed at design time. Dependability is a property at an instant; resilience is dependability that holds across change over time.

### Resilience engineering

The term arrives in computing with baggage from an older field. **Resilience engineering** comes out of safety science (Erik Hollnagel, David Woods, Nancy Leveson), where it was developed to explain why complex, high-consequence systems (aviation, healthcare, power grids, chemical plants) succeed and fail. Its central claims transfer cleanly to software.

The first claim is that safety is not a stockpile of correct decisions but an **adaptive capacity**: the ability of a system, including its human operators, to adjust how it functions when conditions exceed what was designed for. A system can satisfy every specified safety requirement and still be brittle, because brittleness shows up precisely at the boundary where the specification runs out. Resilience is performance *near and past that boundary*.

The second claim is the gap between **work-as-imagined** and **work-as-done**. Work-as-imagined is the system as its designers and procedures describe it: the architecture diagram, the runbook, the documented control flow. Work-as-done is what operators actually do to keep it running: the manual restarts, the traffic they shed during incidents, the undocumented order in which services must boot, the workarounds that have quietly become load-bearing. The two always diverge, and the divergence is not sloppiness to be eliminated. It is where the system absorbs the surprises the design did not anticipate. Richard Cook's *How Complex Systems Fail* (1998) makes the same point from the failure side: complex systems run in a degraded mode as a matter of course, and the operators' ongoing adaptations are what keep the latent faults from lining up into a failure.

The unifying theme is **coping with surprise**. Anticipated faults are a design problem you can close out before shipping. Surprise (the fault not on the list, the combination no one modeled, the environment that moved) is a property of operation, and the only defense is a system, technical and human, that can adapt in the moment.

### Resilience versus classic fault tolerance

The distinction reduces to a single axis: **anticipated versus unanticipated faults**.

| | Classic fault tolerance | Resilience |
|---|---|---|
| Fault model | Enumerated in advance | Open; includes the unforeseen |
| Engineered against | A known list of faults | Change and surprise |
| Mechanism | Specific detection + recovery per fault | General adaptive capacity |
| Defends the system at | Conditions assumed at design time | The boundary where assumptions run out |
| Owner | Designers | Designers *and* operators |

The two are layers, not rivals. Fault tolerance handles the faults you can name. Resilience is what remains when the named faults are exhausted and something unnamed arrives. A dependable system needs both: the list buys you the common cases cheaply, and adaptive capacity catches what the list missed.

### Let-it-crash is resilience-flavored

The BEAM's "let it crash" strategy sits unusually well on this axis, and the reason is worth stating precisely. A conventional recovery routine is anticipatory: it handles a *specific* error because a programmer foresaw that error, wrote a handler, and named the recovery. The faults it covers are exactly the faults someone enumerated.

Let-it-crash inverts the relationship. A BEAM process does not catch the fault, classify it, and recover in place. It crashes on *any* unexpected condition, and a supervisor restarts it from a known-good initial state (the full mechanism, the ladder, and the [containment guarantee](#cross-cutting-patterns) that makes restart trustworthy live in [beam.md](./beam.md)). The recovery path does not depend on knowing what went wrong. It is the *same* path for a fault the author anticipated and a fault no one ever imagined: terminate the contaminated unit, discard its state, restart clean.

That is the resilience property. The recovery mechanism is decoupled from the fault model, so it covers faults that were never enumerated. The system copes with a surprise it was not specifically engineered against, which is exactly Laprie's persistence-of-dependability-under-change applied at the granularity of a single process. It does not make let-it-crash a complete resilience strategy: it handles transient and state-corruption faults superbly and does nothing for a deterministic bug that recurs on every restart (a Bohrbug, in Gray's terms; see [Threats](#threats)), which will simply crash the restarted process again. The point is narrower and real: a recovery mechanism that does not need to name the fault is, to that extent, resilient rather than merely fault-tolerant.

## Security

Dependability and security are sibling disciplines built on a shared core. Security is the conjunction of three [attributes](#attributes): confidentiality (absence of unauthorized disclosure of information), integrity (absence of improper state alteration), and availability (readiness for correct service). Two of these, integrity and availability, are already dependability attributes; confidentiality is the addition that distinguishes security from dependability. The same taxonomy of [threats](#threats) and [means](#means) applies to both, which is why Avizienis et al. treat them under one framework rather than two.

Engineers usually meet this overlap as the **CIA triad** (confidentiality, integrity, availability), the standard mnemonic in security work. It maps cleanly onto the dependability attributes:

```
Security attribute   Dependability relation
------------------   ----------------------------------------
Confidentiality      security-only (the distinguishing attribute)
Integrity            shared with dependability
Availability         shared with dependability
```

A denial-of-service attack is an availability problem. Tampering with a record is an integrity problem. Both are failures in the dependability sense; the only thing that marks them as security concerns is that a human adversary caused them on purpose.

### Attacks as malicious faults

The connection runs deeper than a shared attribute list. An attack is a [fault](#threats), specifically a deliberately induced one. The causal chain is identical: an attacker introduces a fault (a malformed packet, an injected SQL fragment, a forged token), the fault activates into an error (a corrupted parser state, a privilege the attacker should not hold), and the error propagates to a failure (data exfiltrated, a service knocked offline). What changes is the fault's origin. Ordinary fault classes are accidental: a programmer's mistake, a cosmic-ray bit flip, a bad input that nobody anticipated. A malicious fault is chosen by an intelligent agent who is searching for it.

That difference in origin has one practical consequence worth stating plainly. Accidental faults follow the statistics of nature: independent, roughly random, describable by a failure-rate distribution you can measure and feed to [fault forecasting](#means). Malicious faults follow the strategy of an adversary: correlated, adaptive, and aimed precisely at whatever assumption your design leans on hardest. A fault model that assumes independence (the kind that justifies redundancy as a defense) does not transfer to the security setting, because an attacker who beats one replica beats its identical siblings the same way. Fault tolerance against accidental faults and fault tolerance against malicious faults are continuous as concepts and different as engineering problems.

### Containment under an adversary

Read adversarially, the [means](#means) keep their structure. [Fault prevention](#means) becomes secure development: input validation, memory-safe languages, avoiding the constructs that let a fault be introduced in the first place. [Fault removal](#means) becomes penetration testing and security auditing, the same verification activity pointed at faults an attacker would exploit. [Fault tolerance](#means) becomes the runtime containment of attacks that succeed anyway.

This last one is where the dependability vocabulary pays off most directly, because the security mechanisms an engineer already knows are [fault-containment regions](#threats) under another name. Sandboxing confines untrusted code to a region it cannot escape, so that a compromise of the code is an error that cannot propagate to the host: the sandbox boundary is an error-confinement boundary, and the [blast radius](#cross-cutting-patterns) of a sandbox escape is exactly what the sandbox was sized to limit. Privilege separation splits a program into components running at different authority levels (the classic example is OpenSSH, which separates the network-facing parser from the privileged authentication logic), so that a fault in the exposed component cannot reach the privileged one. That is the [bulkhead](#cross-cutting-patterns) pattern applied to trust rather than to crashes. The principle of least privilege is the same idea stated as policy: give each component the minimum authority it needs, which minimizes the blast radius of its compromise.

The [two isolations](#cross-cutting-patterns) carry over as well. State isolation between security domains (separate address spaces, separate containers, separate machines) mechanically guarantees that an attacker who corrupts one domain has not thereby corrupted another, for the same reason a crashed BEAM process cannot have damaged its neighbor: if you share nothing, a compromise cannot reach across a boundary that does not exist. The enforcement layer is the lever, exactly as it is for accidental faults. Hardware enforces process isolation through the MMU, the runtime enforces sandbox boundaries, the type system forbids certain classes of memory-corruption fault before the program runs. An attacker's job is to find the layer your design trusted and the boundary it did not actually enforce.

Security is its own discipline with its own large literature, and this section does not attempt to cover it. The point worth carrying forward is narrow: the dependability taxonomy is the right frame for the part of security that overlaps it. Treat an attack as a fault whose distribution is chosen by an adversary, and the rest of the vocabulary (containment regions, blast radius, enforcement layers, the prevention-tolerance-removal split) transfers without modification.

## Cross-cutting patterns

Some techniques recur across every branch of the taxonomy. They show up under different names in operating systems, language runtimes, distributed databases, and container schedulers, and recognizing the shared shape is what lets you transfer an insight from one to another. This section gives the full treatment of the patterns the rest of the document references by name: how state isolation produces fault containment, the three ways to defuse shared mutable state, the control/data plane split, the redundancy taxonomy, and the vocabulary of failure semantics.

### State isolation produces fault containment

The single most reused idea in dependable system design is this: if two parts of a system share no mutable memory, a failure in one cannot corrupt the state of the other. The property is so useful that systems keep rediscovering it and re-implementing it at whatever enforcement layer is available to them.

To state it precisely, two distinct properties are usually bundled under the overloaded word "isolation," and the argument falls apart if you do not separate them.

**State isolation** (also called *shared-nothing*) is a property of *memory*: two [execution units](#glossary) share no mutable memory. Each can read and write only its own state. Any communication between them happens by an explicit mechanism (a copy, a message, a syscall) rather than by both touching the same address.

**Fault containment** is a property of *failure*: a [failure](#threats) in one unit cannot damage another unit. This is the [fault-containment region](#threats) realized in practice, the boundary an [error](#threats) is guaranteed not to cross.

The link between them is the load-bearing claim:

> State isolation *mechanically guarantees* fault containment. If a unit shares no mutable memory with anyone, then at the instant it fails, it cannot have left anyone else's state corrupted, because it had no way to write to anyone else's state in the first place.

This is what upgrades fault containment from a hope into a guarantee. A system can *try* to contain faults by discipline (everyone agrees not to reach into each other's data), but discipline is an [error](#threats) source like any other code. State isolation removes the possibility structurally: there is no expressible operation that writes across the boundary, so there is nothing to forget and nothing to get wrong. The recovery action that depends on this (restart the failed unit, see [Means](#means)) is only as trustworthy as the guarantee that the failed unit did not already poison the data the restart will rely on.

#### The OS-monitor lineage

The operating system discovered this pattern first, and it discovered it the hard way, one failure mode at a time. The history is worth tracing because each fix corresponds exactly to a mechanism that later systems reinvent.

A 1950s machine that ran one program at a time needed no isolation. The program *was* the system; if it corrupted memory or looped forever, it only hurt itself, and an operator power-cycled the machine. Isolation has no meaning when there is only one thing to isolate.

Batch processing changed that. To avoid an operator manually loading each job, machines gained a small resident program, the **monitor** (the ancestor of the kernel), that sat in memory and loaded jobs one after another. The monitor had to coexist with arbitrary, untrusted job code, and that coexistence created three new failure modes. Each one forced a hardware mechanism, and each of those mechanisms is one rung of the modern isolation story.

| Failure mode the monitor introduced | Hardware mechanism that fixed it | What it is, in taxonomy terms |
|---|---|---|
| A job overwrites the monitor's memory (or another job's) | Memory protection: base/limit registers, later the MMU and per-process address spaces | Hardware-enforced **state isolation** |
| A job issues raw I/O and corrupts a device or another job's data | Dual-mode operation: privileged (supervisor) vs. user mode; I/O only via syscalls | A controlled door through the isolation boundary |
| A job loops forever and never returns control | The timer interrupt: hardware forcibly suspends the running job | Hardware-enforced **preemption** |

Read the right column top to bottom and the monitor becomes a recognizable shape. The kernel is a [control plane](#cross-cutting-patterns) (a supervisor in the [Means](#means) sense): it decides what runs and reacts when something goes wrong, while the jobs are the data plane that does the actual work. The address space is a private heap that one job cannot reach out of. The timer interrupt is preemption that no user code can defeat. The kernel does not trust job code to behave; it arranges that misbehavior is *contained* by hardware regardless of intent.

#### The same pattern at four enforcement layers

Once you see the shape, it appears everywhere, distinguished mainly by *what granularity* it operates at and *which layer* enforces it.

| System | Unit of isolation | Cost per unit | Enforced by |
|---|---|---|---|
| OS process | Address space | Kilobytes to megabytes, milliseconds to create | Hardware (MMU) |
| BEAM process | Private heap | Hundreds of bytes, microseconds to create | Language runtime (no expressible cross-process mutable reference) |
| Container | Namespaced view of kernel resources | Megabytes, hundreds of milliseconds | Kernel (namespaces + cgroups) |
| Rust ownership | A value with a single mutable owner | Zero runtime cost (compile-time only) | Type system |

The [BEAM](./beam.md)'s contribution is not the idea. The OS had memory protection and preemptive scheduling decades earlier. The contribution is *granularity and cost*. The OS gives you isolation at the price of an address space, expensive enough that you ration processes and reach for threads (which share memory and give up the isolation) when you need many concurrent units. The BEAM pushes the same containment property down to a unit costing hundreds of bytes and microseconds, cheap enough that you can afford one per request, per connection, per session. Isolation stops being a scarce resource you budget and becomes the *default unit of program structure*. The enforcement layer also moves: the OS uses the MMU (hardware), while the BEAM uses the language (there is no way to even write down a reference into another process's heap). Different layer, same guarantee.

This is also why "porting the actor API is not the same as adopting the BEAM's properties." The properties live in the enforcement layer (the runtime), not in the shape of the code. An actor library on a shared-heap runtime gives you the *interface* of isolation without the *enforcement*, so the containment guarantee degrades to containment-by-discipline. [beam.md](./beam.md) works this out rung by rung against Akka.

### Three cures for shared mutable state

The disease under all of this is **shared mutable state under concurrency**: when more than one execution unit can write the same memory without coordination, you get [race conditions](#glossary), and the result depends on timing in ways that are hard to test and harder to reproduce. The classic remedy, a lock, has a structural weakness: a lock protects *code* (the critical section), and the association between the lock and the data it guards lives only in the programmer's head. You can forget to take it, take the wrong one, or take them in an order that deadlocks.

There are exactly three ways to defeat the disease, distinguished by *when* and *at what cost* they act. They map onto the same enforcement-layer axis as isolation: runtime, design-time, compile-time.

**Cure 1: Isolate.** Give each unit its own private heap and copy data between them ([the BEAM](./beam.md)). There is no shared mutable state because there is no sharing.
- *Cost:* copy bandwidth, garbage collection pressure, memory for many heaps.
- *Buys:* high concurrency density (millions of cheap units) and fine-grained fault containment (one unit's blast radius is one unit).

**Cure 2: Serialize.** Run the work on a single execution unit, so there is no second unit to race against (the TigerBeetle core, whose [state machine](#glossary) executes single-threaded). With one writer, no concurrent access exists to coordinate.
- *Cost:* you cannot use other cores for that work, and the failure domain is coarse (the whole single-threaded unit fails together).
- *Buys:* determinism (the same inputs always produce the same execution, which makes the system exhaustively testable by simulation), and branchless, cache-friendly throughput because there are no locks or memory fences on the hot path. High availability comes not from concurrency but from [replication](#cross-cutting-patterns): run several identical replicas and keep them in step with [consensus](#glossary).

**Cure 3: Prevent at compile time.** Let the type system forbid shared mutable aliasing before the program runs (Rust's ownership and borrowing: either many readers *or* one writer, never both, checked statically).
- *Cost:* developer effort, the program must be expressed in a form the checker can prove safe.
- *Buys:* zero runtime cost, and safe sharing wherever the compiler can prove no data race exists, so you are not forced to copy or serialize when you do not need to.

The three cures are the same taxonomy as the enforcement layers: cure 1 acts in the **runtime**, cure 2 is a **design-time** decision about structure, cure 3 acts in the **type system / compile time**. None is strictly best; each trades a different resource (memory and copy cost, core utilization, developer effort) to buy back safety.

One connection worth making explicit: serialization (cure 2) is the same idea as a [BEAM](./beam.md) process handling its mailbox one message at a time, applied to the *whole data plane* instead of to one small unit. A process accesses its own state serially because the only door in is the mailbox, which is drained one message at a time. That serial access is equivalent to a critical section under a fair mutex, except the lock is attached to the *state* (the process owns its heap) rather than to the *code*, so you cannot forget to take it. TigerBeetle's core does this without paying for the isolation-and-copy of cure 1, because there is no second concurrent unit to isolate it from: with a single data-plane execution unit, serial access is automatic.

### Control plane and data plane

A recurring structural split: separate the part of the system that *decides what to do* from the part that *does it*.

The **control plane** decides. It is typically O(1) per decision, runs rarely relative to the work, and is allowed to be branchy and stateful: it inspects situations, makes policy choices, and reacts to events. A scheduler, a supervisor, a Kubernetes control loop, an OS kernel handling a fault: all control plane.

The **data plane** does. It is O(N) in the work, runs constantly, and should be tight and predictable: ideally branchless, allocation-free on the hot path, doing the same thing to every item. A network card forwarding packets, a worker process serving requests, TigerBeetle's state machine applying transactions: all data plane.

The split matters for dependability because the two planes have different failure profiles and want different treatment. The data plane is where throughput lives, so it must stay fast and simple; you make it dependable by keeping it small enough to verify and by *containing* its failures rather than letting it handle them inline. The control plane is where recovery logic lives, so it is allowed to be complex, because it runs rarely and its job is precisely to react to data-plane failures. A [supervisor](#means) is a control plane whose data plane is the worker it manages: the worker does the work and, on failure, dies; the supervisor decides what to do about the death (restart, escalate, give up). This is the same division as the OS kernel (control) over user jobs (data) from the monitor lineage above. Putting recovery logic in the data plane would slow the hot path and entangle "do the work" with "decide what to do when the work fails," which are different concerns with different correctness requirements.

### Redundancy

[Fault tolerance](#means) requires redundancy: to keep delivering correct service through a fault, something has to be available to take over or to cross-check. Redundancy is spent along three axes, and they can be combined.

**Space redundancy** replicates *hardware or units*: multiple copies of a component, so the loss of one leaves others. RAID mirrors, dual power supplies, replicated database nodes, multiple BEAM processes ready to be restarted into.

**Time redundancy** replicates *the computation*: do the same work again (retry a failed request, recompute and compare). It costs latency rather than hardware, and it only helps against *transient* faults, the ones that do not recur on a repeat. A retry against a deterministic bug just reproduces the bug.

**Information redundancy** replicates *bits*: extra data that lets you detect or correct corruption without a second copy of everything. Parity, checksums, ECC memory, error-correcting codes. It catches and sometimes repairs corruption in storage and transmission.

Cutting across these axes is a distinction in *what kind of fault* the redundancy defends against, which determines whether the copies should be identical.

**Replication** runs *identical* copies of the same implementation. This defends against *independent* faults: hardware failure, a crashed node, a lost packet, the kinds of fault that hit one copy and not the others by chance. The [state machine replication](#glossary) approach (Schneider 1990) is the canonical form: deterministic replicas fed the same inputs in the same order via [consensus](#glossary) stay in identical states, so any survivor can answer. This is how a single-threaded system like TigerBeetle gets high availability: it does not make one replica concurrent, it runs several and keeps them in step. Replication does *not* defend against a bug in the implementation, because identical copies share identical bugs and all fail the same way on the same input.

**N-version programming** runs *different, independently written* implementations of the same specification and votes on their outputs. This targets the fault replication cannot touch: a *design or software fault*, a deterministic [Bohrbug](#glossary) that every identical replica would hit together. The hope is that independent teams make independent mistakes. In practice the defense is weaker than it looks, because independent developers tend to make *correlated* errors on the hard parts of the spec, and the cost (building and maintaining N implementations) is high. N-version programming sees limited use outside high-assurance domains like avionics.

The choice between them follows the fault class. For crashes and transient hardware faults, replicate. For systematic software faults, identical replication is useless and you need either genuine diversity (N-version) or, more commonly, [fault removal](#means) before deployment and the bet that survives a crash through [recovery](#means) rather than through voting.

### Failure semantics

How a component fails is as important to the enclosing system as whether it fails, because the enclosing system's recovery logic is designed against an *assumed* failure mode. Get the assumption wrong and the recovery is built on sand. These terms name the contract a component offers about *how* it will fail.

**Fail-stop.** On failure, the component halts completely and *visibly*: it stops producing output, and other components can detect that it has stopped (Schneider's idealization). This is the friendliest failure mode to design around, because "it is either working correctly or visibly gone" is a clean two-state contract. Most recovery schemes, including supervision and many consensus protocols, assume fail-stop.

**Fail-fast.** On detecting an internal [error](#threats), the component stops *immediately* rather than continuing in a questionable state. The emphasis is on *speed of stopping*: detect the broken invariant and halt before the error can propagate into output or persist into shared state. This is the "crash rather than continue on corrupt state" principle ("better a dead node than a corrupted one running") that both the [BEAM](./beam.md)'s let-it-crash and TigerBeetle's dense assertions embody. Fail-fast is how a component makes itself *approximately* fail-stop: it converts an internal error into a clean halt that the outside can treat as fail-stop.

**Fail-silent.** A failed component produces *no output at all* (as opposed to wrong output). It is silent rather than lying. A fail-silent component looks, from outside, like a fail-stop one, the failure manifests as absence, which downstream timeouts can detect.

**Fail-safe.** On failure, the component moves to a state that is *safe* in the real-world sense, even if it is not operational. A railway signal that defaults to red on power loss, a milling machine that retracts the cutter, an authorization service that *denies* on failure. This trades [availability](#attributes) for [safety](#attributes): the safe state is often the non-serving state. Fail-safe only makes sense where there is a meaningful safe fallback; for a pure information service there may be no "safe" state distinct from "down."

**Byzantine.** The component fails *arbitrarily*: it may produce wrong output, inconsistent output to different observers, or actively misleading output, whether from a subtle bug, memory corruption, or malice. This is the hardest failure mode to tolerate because the component violates every clean contract above. It does not stop, it does not stay silent, and you cannot trust what it says. Tolerating Byzantine faults requires far more redundancy (classically, more than two-thirds of replicas must be correct) and cryptographic verification, which is why most systems explicitly *assume away* Byzantine failure and design only for fail-stop. The assumption is a design decision with teeth: a system built for fail-stop behaves unpredictably when a component instead fails Byzantine.

These modes form a rough lattice from easiest to hardest to tolerate: fail-stop and fail-silent (cleanly absent) are easiest, fail-safe adds a real-world safety requirement, and Byzantine (arbitrary, possibly adversarial) is hardest. Engineering effort largely goes into pushing components *up* this lattice, making a component that could fail arbitrarily instead fail fast and clean, so the rest of the system can assume the easy case.

**Graceful degradation** is the system-level counterpart. Rather than the whole system failing when a component fails, it sheds or reduces function and keeps delivering its most important service. A streaming service that drops to lower resolution under load, a site that serves stale cached data when its database is unreachable, a request path that disables a recommendation widget but still completes the checkout. Graceful degradation depends on the patterns above: you need [fault containment](#cross-cutting-patterns) so the failed component does not take its neighbors with it, and you need failure semantics clean enough (fail-stop or fail-fast) that the surrounding system can *detect* the failure and route around it. It is the difference between a [failure](#threats) (the whole service deviates from correct behavior) and a degraded but still-acceptable service that, by the [dependability](#the-root-concept) definition, has not failed at all.

## Enforcement layers

A dependability property is only as strong as the layer that enforces it. The same property — say, [state isolation](#cross-cutting-patterns) — can be a hardware fact, a runtime invariant, a compile-time proof, or a written-down convention nobody checks. These are not equivalent. The enforcement layer determines three things: the **granularity** at which the property holds (one byte, one process, one pod, one datacenter), the **cost** of obtaining it, and, most importantly, the **strength of the guarantee** (mechanically impossible to violate, versus checked at runtime, versus "we agreed not to").

This section reads the dependability stack bottom-up, from the silicon to the org chart. It is the structural complement to the [historical recurrence in Cross-cutting patterns](#cross-cutting-patterns): there the argument is that the *same containment idea* reappears at each layer; here the argument is that *where you place the enforcement* sets the price and the strength.

### The principle

Two rules govern the whole stack.

**Lower layers give stronger guarantees at coarser granularity and higher cost-to-change.** The MMU cannot be talked out of a page fault. But it operates on pages and address spaces, it costs a hardware context switch to cross, and you cannot reshape it from application code. Hardware enforcement is absolute and rigid.

**Higher layers give cheaper, finer, more flexible enforcement at weaker guarantees.** A code-review checklist that says "always copy this struct before sharing it" costs nothing to write and can express any rule you like, but it is enforced by human attention and fails silently the first time someone forgets. Organizational enforcement is flexible and porous.

The engineering question is never "which layer is best" but "what is the weakest layer I can enforce this property at and still get a guarantee I can stake the system on?" Push enforcement *down* when the property is load-bearing and an adversary (a bug, a hostile input, a tired engineer) will eventually test it. Keep enforcement *up* when the property is cheap to honor and the cost of rigidity outweighs the cost of an occasional violation.

### The layer map

Each row names a layer, what it can isolate or protect, the mechanism that enforces it, and the granularity/cost/guarantee trade. The rows are ordered from the strongest, coarsest, most rigid (hardware) to the most flexible, finest in intent, weakest in guarantee (organizational).

| Layer | Isolates / protects | Enforcement mechanism | Granularity | Cost | Guarantee strength |
|---|---|---|---|---|---|
| **Hardware** | Memory between address spaces; privileged operations; runaway compute | MMU / base-limit registers (memory protection); dual-mode user/supervisor bit; timer interrupt (preemption); ECC, lockstep cores, watchdogs | Page / address space / core | Silicon area; TLB misses; context-switch cost | **Absolute.** Cannot be violated from software at all. |
| **OS kernel** | Processes from each other; the kernel from user code | Per-process [virtual address spaces](#cross-cutting-patterns) built on the MMU; syscall boundary; the scheduler's [preemption unit](#cross-cutting-patterns) (the OS thread, suspended on timer interrupt) | OS process / thread | ~µs–ms context switch; syscall crossing; kernel memory per process | **Hardware-backed.** A user process cannot reach another's memory or monopolize a core. |
| **Language runtime** | Execution units inside one OS process from each other; the runtime's own invariants | Managed heaps + GC; no expressible cross-unit mutable reference; runtime-level [preemption](#cross-cutting-patterns). The BEAM: private per-process heaps + [reduction counting](#cross-cutting-patterns), so [state isolation](#cross-cutting-patterns) holds with no MMU and no OS process per actor. | [Execution unit](#cross-cutting-patterns) (BEAM process, goroutine, fiber) | Copy-on-send bandwidth; GC; runtime complexity | **Runtime-enforced.** As strong as the runtime is sound — strong against application bugs, not against a runtime/NIF escape. |
| **Type system** | Aliasing and mutation patterns; protocol/state-machine misuse | Static analysis before any code runs. Rust: ownership + borrow checker forbids shared mutable aliasing at compile time; typestate encodes legal call sequences. | Reference / value / API call site | Developer effort; expressiveness lost to what the checker can prove | **Compile-time proof.** Violations are not runtime errors; they are non-programs. Zero runtime cost; only as good as the proof's assumptions (`unsafe`, FFI). |
| **Container / VM** | Filesystem, network, PID, and resource namespaces between workloads on one host | OS namespaces + cgroups (containers, leaning back on the kernel/MMU); a hypervisor with a second-level page table (VMs); seccomp/capabilities narrowing the syscall surface | Container / VM / cgroup | Image/boot overhead; VM memory duplication; syscall-filter cost | **Kernel- or hypervisor-backed.** VMs near-hardware-strong; containers share one kernel, so a kernel-level escape crosses the boundary. |
| **Distributed / orchestration** | Whole nodes; the system from a node's total failure | Replication and consensus (state-machine replication, viewstamped replication); health-checking and reschedule (Kubernetes restarting a pod); quorums tolerating f failures | Node / pod / replica | Network round-trips; replica hardware (N×); coordination latency | **Statistical / quorum-based.** Tolerates a *bounded number* of independent failures; correlated failure defeats it. The crash is assumed, not prevented. |
| **Organizational / process** | The system from human error and from faults nobody anticipated | Code review, tests in CI, runbooks, on-call, change management, deterministic-simulation-testing culture ([TigerStyle](#the-mttf-and-mttr-lever)), blameless postmortems | Commit / deploy / decision | Human time; process drag; depends on discipline | **Conventional.** No mechanism enforces it; it holds exactly as far as people follow it. The layer that catches the *unanticipated* — see [Resilience](#resilience). |

### Reading the map: same property, different layers

The map's value is comparative. Pick one property and watch where it can live.

**State isolation** (execution units share no mutable memory). The OS gives it at process granularity, MMU-enforced, at the cost of a heavyweight process and a context switch to communicate. The [BEAM](./beam.md) gives the *same property* at process granularity costing hundreds of bytes, enforced by the runtime instead of the MMU — cheap enough to be the default unit of program structure. Rust gives it at *reference* granularity, enforced at compile time, with no runtime cost at all but bought with developer effort and lost expressiveness. These are the [three cures for shared mutable state](#cross-cutting-patterns) — isolate, prevent-at-compile-time — read off as a single property migrating down and up the enforcement stack. The fourth option, [serialize](#cross-cutting-patterns) (TigerBeetle's single-threaded core), sidesteps the property entirely: with one execution unit there is no second unit to isolate from.

**Preemption** (no unit can monopolize a core). Hardware provides the timer interrupt; the OS builds preemptive thread scheduling on it; the BEAM rebuilds the *same guarantee* one layer up with [reduction counting](#cross-cutting-patterns), at [execution-unit](#cross-cutting-patterns) granularity, because OS preemption operates on threads and the BEAM has millions of processes per thread. Akka, by contrast, cannot provide actor-level preemption at all: it has no enforcement layer below the OS thread to put it at, so a long message handler starves its dispatcher. The property is absent not by oversight but because the layer that could enforce it is not available to a JVM library.

This is the general lesson the [Akka counter-example](./beam.md) teaches in one line: **a property is only present if some layer actually enforces it.** Copying an actor API (organizational/library layer) does not summon the runtime-layer enforcement that gives the API its meaning. "[Porting the API is not the same as adopting the properties](./beam.md)" is exactly the statement that the API layer sits *above* the layer where the guarantee is manufactured.

### Failure containment crosses layers too

[Fault-containment regions](#threats) nest along this same stack, and a [failure](#threats) at one layer is a fault for the layer above (the recursion in [Threats](#threats)). A Rust borrow error is contained at compile time and never becomes a runtime fault at all. A BEAM process crash is contained by the runtime to one private heap; supervision restarts it. A container OOM-kill is contained by cgroups to one workload; the orchestrator reschedules it. A node loss is contained by consensus to one replica; the quorum carries on. A property the lower layer failed to contain bubbles up until some layer's [fault-tolerance](#means) mechanism catches it — or, if none does, becomes a system [failure](#threats) and the [organizational layer](#resilience) (on-call, postmortem) becomes the containment region of last resort.

The [blast radius](#cross-cutting-patterns) of a fault is therefore set by the *lowest layer that successfully contains it*, and the [MTTR](#the-mttf-and-mttr-lever) of recovery is set by *that layer's recovery latency*: microseconds for a BEAM supervisor restart, seconds-to-minutes for a Kubernetes reschedule, minutes-to-hours for a human paged out of bed. Choosing an enforcement layer is choosing a point on both axes at once: finer containment and faster recovery as you push down, at the price of the rigidity and cost the lower layers demand.

## Measures

Dependability is a quality you eventually have to argue about with numbers. This section collects the quantitative apparatus in one place: the time-based metrics, the reliability function that ties them together, the availability table everyone cites, and the operational layer (SLO, SLA, error budget) that turns these figures into commitments and decisions. The availability formula itself, `A ≈ MTTF / (MTTF + MTTR)`, and the prevention-versus-tolerance reading of it belong to [The MTTF and MTTR lever](#the-mttf-and-mttr-lever); this section defines the inputs to that formula and the surrounding measures.

### Time-based metrics: MTTF, MTTR, MTBF

Three mean-time figures describe a component's behavior over its operating life. They are expectations (statistical means) over many observed lifetimes or over a population of identical components, not guarantees about any single run.

**MTTF (mean time to failure).** The expected length of a single interval of correct service: how long the system runs, on average, before it fails. This is the metric of [reliability](#attributes) (continuity of correct service). MTTF is defined for components that are not repaired during the interval of interest, or equivalently for one uptime interval at a time. A disk rated for an MTTF of 1.2 million hours is not promising any individual disk will last 137 years; it means that across a large population, the failures average out to one per 1.2 million device-hours of operation.

**MTTR (mean time to repair, sometimes "to recovery").** The expected time from a failure until correct service is restored. This is the metric of [maintainability](#attributes). MTTR bundles everything on the recovery path: detecting the failure, diagnosing it, performing the repair or restart, and validating that service is back. Where you draw the boundary matters and should be stated. A bare process restart and a full incident response with human paging produce very different MTTR figures for the same crash.

**MTBF (mean time between failures).** For a repairable system that cycles through fail-repair-run-fail, the expected time from the start of one failure to the start of the next:

```
MTBF = MTTF + MTTR
```

MTBF measures the full cycle; MTTF measures only the up portion of it. The distinction is usually negligible when MTTR ≪ MTTF (a system that runs for months and recovers in seconds), and the two terms are then used loosely. It stops being negligible exactly in the regime the BEAM optimizes for, where a component may fail often (short MTTF) but recover almost instantly (tiny MTTR). There the up-fraction MTTF/MTBF is what governs availability, and conflating MTTF with MTBF hides the entire trade. See the [reliability-versus-availability example](#attributes) and [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

### Failure rate and the reliability function

**Failure rate λ (lambda).** The rate at which a component fails, in failures per unit time. The general form is the *hazard rate* h(t): the instantaneous probability of failure at time t given survival up to t. Under the common simplifying assumption that this rate is constant over the operating life, h(t) collapses to a single number λ, and λ is then the reciprocal of MTTF:

```
λ = 1 / MTTF          (constant-hazard case)
```

A component with MTTF = 10,000 hours has λ = 10⁻⁴ failures per hour. Failure rates add: two independent components in series, either of whose failure fails the whole, give a system rate λ_sys = λ₁ + λ₂, so the combined MTTF is smaller than either part's. This is why adding components in a series dependency chain (more services any one of which can take you down) lowers reliability even when each part is individually good.

**The reliability function R(t).** R(t) is the probability that the component is still delivering correct service at time t, having started correct at t = 0. Under constant hazard λ it is the exponential:

```
R(t) = e^(−λt)
```

This is the single most cited reliability formula, and its shape is worth internalizing. At t = MTTF (that is, t = 1/λ), R = e⁻¹ ≈ 0.368: a component is more likely than not to have already failed by the time it reaches its mean lifetime. The mean sits well into the tail because the distribution is right-skewed; most failures happen before the mean, a few survivors drag it upward. Reliability targets are therefore quoted at a mission time t much shorter than MTTF. To achieve R(t) = 0.99 over a mission of length t, you need MTTF ≈ t / 0.01 = 100t. Wanting 99% survival over a one-year mission demands an MTTF on the order of a century.

**What the constant-hazard assumption buys and costs.** Setting h(t) = λ (constant) is what makes the math clean: it gives the memoryless exponential, where a unit that has run for a year is exactly as likely to fail in the next hour as a brand-new one. The assumption is reasonable during the flat middle of a component's life, the bottom of the classic *bathtub curve* (high early-life "infant mortality" from latent defects, a long flat useful-life region, then rising "wear-out" at the end). It is wrong at both ends. Mechanical parts wear out, so their hazard climbs with age; freshly deployed software has a burn-in period where latent [faults](#threats) get activated and removed, so its hazard falls. Software in steady operation, dominated by transient Heisenbugs (Gray's term for timing- and state-dependent bugs that vanish under observation, as opposed to deterministic Bohrbugs) rather than wear, often does approximate constant hazard reasonably well, which is part of why the exponential model is the default for systems work. When you see R(t) = e^(−λt), read it as "constant-hazard approximation," and remember the approximation breaks near deployment and near end-of-life.

### The availability "nines" table

Availability A is the fraction of time the system is ready for correct service, a number in [0, 1] usually quoted as a percentage and named by its count of leading nines. The formula relating A to MTTF and MTTR is owned by [The MTTF and MTTR lever](#the-mttf-and-mttr-lever); here is what the resulting numbers mean operationally. The useful way to feel an availability figure is as a downtime budget: `(1 − A)` multiplied by the length of the period.

```
Availability   "nines"   Downtime / year     Downtime / 30-day month   Downtime / day
99%            two        3.65 days           7.2 hours                 14.4 min
99.9%          three      8.77 hours          43.2 min                  1.44 min
99.95%         —          4.38 hours          21.6 min                  43.2 s
99.99%         four       52.6 min            4.32 min                  8.64 s
99.999%        five       5.26 min            25.9 s                    864 ms
99.9999%       six        31.6 s              2.59 s                    86.4 ms
```

Two things in this table do real work. First, each additional nine costs an order of magnitude: the engineering effort and redundancy to go from three nines to four is not 33% more than to reach three, it is a different regime. Second, the per-period columns expose what a target actually permits. "Five nines" sounds absolute, but it still allows five minutes of downtime a year, and one bad ten-minute incident blows a five-nines annual budget by itself. This is the table that makes the [MTTF/MTTR lever](#the-mttf-and-mttr-lever) concrete: the reliability-versus-availability example (a process that crashes every hour but recovers in 50 ms) lands at A ≈ 1 − (0.05 s / 3600 s) ≈ 99.9986%, comfortably into four nines, despite a one-hour MTTF that would be judged catastrophic if you only looked at reliability.

A caution the nines hide: availability as defined is time-averaged and says nothing about the distribution of outages. The same 99.99% can mean one 52-minute outage a year or 4,380 ten-second blips, and these are very different products for a user. Availability also depends on what you count as "down" and over what window, which is exactly the ambiguity the operational layer below exists to pin down.

### The operational layer: SLI, SLO, SLA, error budget

The metrics above describe a system's intrinsic behavior. The operational layer turns them into measured commitments. The vocabulary comes from Google's Site Reliability Engineering practice and is now industry-standard.

**SLI (service level indicator).** A quantitative measurement of some aspect of service quality, defined precisely enough to compute from telemetry. Examples: the fraction of HTTP requests returning non-5xx within 300 ms over a rolling window; the fraction of minutes in which the service answered health checks. An SLI is the operational, measurable stand-in for an attribute (availability, latency, [integrity](#attributes)). Getting the SLI definition right (which events count, what window, measured from where) is most of the work, because everything below inherits its precision or its sloppiness.

**SLO (service level objective).** A target value or range for an SLI over a stated period: "99.95% of requests succeed within 300 ms, measured monthly." The SLO is the internal goal the team commits to and engineers against. It is chosen, not discovered: a deliberate decision about how reliable the service needs to be, set with reference to what users actually require rather than to the highest number achievable. Picking an SLO higher than users need is a real cost, because every additional nine spends engineering effort that could have gone elsewhere (see the order-of-magnitude jumps in the nines table).

**SLA (service level agreement).** A contract with users or customers that includes consequences (refunds, credits, penalties) for missing a stated reliability level. The SLA is the externally promised, legally or commercially binding figure. The standard discipline is to set the SLO *stricter* than the SLA, so the team trips its internal alarm and reacts before the contractual line is crossed. An SLA promising 99.9% is typically backed by an internal SLO of 99.95% or better. Not every service has an SLA, but most production services should have SLOs.

**Error budget.** The complement of the SLO, `1 − SLO`, read as a quantity of permitted failure rather than a target of success. An SLO of 99.9% over a 30-day month grants an error budget of 0.1%, which the nines table converts to 43.2 minutes of allowed downtime (or, for a request-success SLI, 0.1% of the month's requests). The budget reframes reliability as a spendable resource. As long as the budget has room, the system is meeting its objective by definition, and failures within budget are acceptable, including failures spent deliberately: deploys, configuration changes, and chaos experiments all draw it down. When the budget is exhausted, the standard policy is to halt feature risk and redirect effort to reliability until the budget refills.

The error budget is the mechanism that converts the whole dependability vocabulary into day-to-day decisions, and it makes the central trade of this corpus operational. A "let it crash" architecture ([fault tolerance](#means) with tiny MTTR) spends almost nothing from the budget per crash, which is what lets a low-reliability, high-availability design pass an availability SLO that a low-MTTR design could not. A prevention-first culture ([fault prevention and removal](#means), the TigerStyle posture) instead aims to never draw the budget down in the first place. Same budget, two strategies for staying inside it, which is the [prevention-versus-tolerance map](#the-mttf-and-mttr-lever) seen from the operational side.

## Worked instances

This section places concrete systems within the taxonomy. Each is a pointer, not a treatment: which [attributes](#attributes) it optimizes, which [means](#means) it relies on, and where the deeper material lives. The first is treated in full in its own companion document.

### The BEAM / Erlang-OTP

The Erlang virtual machine (BEAM) and its OTP libraries are the corpus's primary fault-tolerance exemplar; see [beam.md](./beam.md) for the full telling. Taxonomically it bets on fault [tolerance](#means): it assumes faults will occur at runtime and structures the system to contain and recover from them. The mechanism is [state isolation](#cross-cutting-patterns) (shared-nothing [BEAM processes](#glossary) with private heaps) upgrading supervision from a restart mechanism into a restart guarantee, so a crash in one process is a [fault-containment region](#threats) of a few hundred bytes. The attribute it maximizes is [availability](#attributes), not reliability: "let it crash" deliberately trades a short MTTF (processes die often) for fast recovery and high uptime, the classic low-[MTTR](#the-mttf-and-mttr-lever) play. Recovery is in-runtime and microsecond-scale, with a one-process [blast radius](#threats).

### TigerStyle / TigerBeetle

TigerBeetle is a financial accounting database; TigerStyle is the engineering culture behind it. It sits at the opposite pole from the BEAM on the same [MTTF/MTTR lever](#the-mttf-and-mttr-lever): it raises MTTF rather than lowering MTTR. The primary means are fault [prevention](#means) and fault [removal](#means): static memory allocation, bounded loops, dense assertions, and deterministic simulation testing that replays the system under injected faults to surface bugs before they ship. Its core is a single-threaded [execution unit](#glossary), the SERIALIZE cure from [Cross-cutting patterns](#cross-cutting-patterns): with no second unit to race against, it gets determinism and branchless throughput without paying for isolation. The attributes are [integrity](#attributes) and [reliability](#attributes), with the operating rule "better a dead node than a corrupted one running." High [availability](#attributes) comes not from the single node but from running it as a replicated state machine, with consensus across replicas covering the loss of any one. The verifiability of the single deterministic replica is what makes that crash-the-whole-thing strategy safe.

### Tandem NonStop (process pairs)

Tandem's NonStop systems (1970s onward, the lineage Jim Gray wrote up in *Why Do Computers Stop*) targeted [availability](#attributes) for transaction processing through hardware and software redundancy. The signature technique is the **process pair**: a primary process runs the work while a backup process on a separate processor receives periodic checkpoints of its state. When the primary's processor fails, the backup takes over from the last checkpoint. This is fault [tolerance](#means) by spatial redundancy, with the processor boundary serving as the [fault-containment region](#threats). NonStop is also where Jim Gray formalized and popularized the durable distinction between *Bohrbugs* (deterministic, reproducible faults that a retry will hit again) and *Heisenbugs* (timing- and state-dependent faults that often vanish on retry); his report coined "Bohrbug" and framed the Bohrbug/Heisenbug hypothesis, though the term "Heisenbug" itself predates it. Process-pair failover works precisely because most production faults in a tested system are Heisenbugs, so restarting the backup from a clean checkpoint usually succeeds where re-running the same code would not.

### Kubernetes

Kubernetes is a container orchestrator and belongs to the fault [tolerance](#means) branch, operating the same low-[MTTR](#the-mttf-and-mttr-lever) lever as the BEAM but at a much coarser granularity. Its control loops continuously compare desired state against observed state and reconcile the difference: a crashed container is restarted, a dead node's pods are rescheduled onto survivors, a failing pod is removed from service routing. The [execution unit](#glossary) and [fault-containment region](#threats) is the pod (an OS-process group inside a container), the [blast radius](#threats) is a whole pod, and recovery latency is seconds to minutes rather than microseconds. The attribute optimized is [availability](#attributes). The contrast with the BEAM is one of scale on a shared axis: both lower MTTR by detecting failure and recreating the failed unit, but Kubernetes does it across machines at pod granularity where the BEAM does it inside one VM at process granularity.

### The telephone switch (Ericsson AXD301)

The AXD301 ATM switch is the canonical Erlang production system and the empirical anchor for the BEAM's claims. It was built to telecom carrier-grade requirements: the often-cited target is "five nines" or better availability (see [Measures](#measures) for what a [nine](#measures) buys you), meaning a few minutes of downtime per year, including time spent on software upgrades. It reportedly reached availability figures around nine nines in field measurement. Taxonomically it is the BEAM thesis at production scale: fault [tolerance](#means) through [state isolation](#cross-cutting-patterns) and OTP supervision, optimizing [availability](#attributes) by minimizing MTTR, plus hot code loading so that maintenance and upgrade do not count as downtime, which ties the [maintainability](#attributes) attribute directly into the availability number. It is the worked instance that turned "let it crash" from an idea into a deployed result, and it is the system Armstrong's thesis was written to explain.

## Glossary

Terms are defined tersely here; the section that owns each concept gives the full treatment. Follow the cross-reference where one is shown.

- **Availability** — readiness for correct service; the fraction of time the system is up, A ≈ MTTF/(MTTF+MTTR). See [Attributes](#attributes) and [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).
- **BEAM process** — a lightweight execution unit inside the Erlang VM (not an OS process); a few hundred bytes, millions can coexist, each with its own private heap.
- **Blast radius** — how much of the system a single fault can take down; small when fault-containment regions are fine-grained.
- **Bohrbug / Heisenbug** — a Bohrbug reproduces deterministically (a solid bug you can chase down); a Heisenbug depends on timing/ordering/state and vanishes under observation. The distinction was formalized and popularized by Gray (1985), whose report coined "Bohrbug" and framed the Bohrbug/Heisenbug hypothesis (the term "Heisenbug" itself predates the report). Let-it-crash exploits the fact that a restart often clears a Heisenbug's triggering state.
- **Bulkhead** — the watertight-compartment metaphor for a containment boundary: a partition that stops a local failure from flooding the whole system.
- **Control plane / data plane** — the control plane decides what to do (O(1), branchy, e.g. a supervisor); the data plane does it (O(N), tight, e.g. the worker). See [Cross-cutting patterns](#cross-cutting-patterns).
- **Dependability** — the ability to deliver service that can justifiably be trusted; equivalently, the ability to avoid failures more frequent or severe than acceptable. The umbrella term. See [The root concept](#the-root-concept).
- **Error** — a fault manifested in system state: a wrong value or broken invariant, not yet externally visible. The middle link of fault → error → failure. See [Threats](#threats).
- **Execution unit** — an independently schedulable sequence of instructions; the genus of which thread, BEAM process, actor, fiber, and goroutine are species.
- **Failure** — the externally visible deviation of delivered service from correct service; the activation of an error across the system boundary. A component's failure is a fault for the enclosing system. See [Threats](#threats).
- **Fail-stop / fail-fast / fail-safe / fail-silent** — failure semantics a component is designed to exhibit. *Fail-stop*: on failure the component halts and others can detect that it halted (the assumption state-machine replication relies on). *Fail-fast*: detect a bad state and stop immediately rather than continue corrupt ("better a dead node than a corrupted one running"). *Fail-safe*: on failure move to a state with no catastrophic consequences (the brake engages). *Fail-silent*: a failed component emits nothing rather than wrong output (no Byzantine/arbitrary outputs). See [Cross-cutting patterns](#cross-cutting-patterns).
- **Fault** — the cause of an error: a bug, bad input, or hardware glitch; may lie dormant until activated. The first link of fault → error → failure. See [Threats](#threats).
- **Fault-containment region** — (a.k.a. error-confinement region) a boundary within which an error is guaranteed not to propagate; a BEAM process's private heap is one. See [Threats](#threats).
- **Fault prevention / tolerance / removal / forecasting** — the four means. *Prevention*: stop faults being introduced. *Tolerance*: deliver correct service despite faults, via error detection plus recovery and redundancy. *Removal*: reduce the number/severity of faults via verification and testing. *Forecasting*: estimate present and future fault behavior. See [Means](#means).
- **Integrity** — absence of improper system-state alteration. One of the five attributes; joins Confidentiality and the others to form Security. See [Attributes](#attributes).
- **Maintainability** — ability of a system to be repaired or modified; measured by MTTR. See [Attributes](#attributes).
- **MTTF / MTTR / MTBF** — mean time to failure (how rarely it breaks), mean time to repair (how fast it recovers), and mean time between failures = MTTF + MTTR for repairable systems. See [Measures](#measures) and [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).
- **Preemption unit** — the granularity at which a scheduler can forcibly suspend a running execution unit: the OS thread for an OS (timer interrupt), the process for the BEAM (reduction exhaustion).
- **Reduction counting** — the BEAM's preemption mechanism: each process gets a budget of reductions (roughly, function calls) and is forcibly suspended when it spends them, yielding per-process fairness no user code can defeat.
- **Reliability** — continuity of correct service over an interval; measured by MTTF, expressed as R(t). Distinct from availability: a process that crashes and restarts in 50 ms every hour has poor reliability but excellent availability. See [Attributes](#attributes).
- **Resilience** — the persistence of dependability when facing changes (Laprie, 2008); in resilience engineering, the adaptive capacity to cope with unanticipated faults and surprise. See [Resilience](#resilience).
- **Safety** — absence of catastrophic consequences for users and the environment. One of the five attributes. See [Attributes](#attributes).
- **Shared-nothing / state isolation** — a memory property: execution units share no mutable memory. State isolation mechanically guarantees fault containment — if you share nothing, a crash cannot have corrupted anyone else's state. See [Cross-cutting patterns](#cross-cutting-patterns).
- **State-machine replication** — running a deterministic state machine on multiple replicas fed the same input sequence (via consensus) so they stay identical; tolerates a replica's fail-stop failure by serving from the survivors (Schneider, 1990). TigerBeetle achieves availability this way around a single-threaded core.
- **Supervision** — the OTP/BEAM pattern in which a supervisor process (a control plane) monitors worker processes and restarts them on failure per a declared strategy. The restart is a *guarantee* rather than a mere *mechanism* only because state isolation ensures the crash corrupted nothing outside the dead process.

## Sources

The seed reading list, organized by the branch of the taxonomy each entry primarily serves. Several works span branches; each is filed where it does the most work and cross-referenced where relevant. This section is itself the reference list — entries below carry their own citations.

Three **start here** entries are marked ▶. Read those first: they establish the vocabulary the rest of the corpus reuses.

### Taxonomy root

The conceptual spine. These define [dependability](#the-root-concept), the [attributes / threats / means](#the-root-concept) split, and the extension to resilience.

▶ **Avizienis, A., Laprie, J.-C., Randell, B., Landwehr, C.** "Basic Concepts and Taxonomy of Dependable and Secure Computing." *IEEE Transactions on Dependable and Secure Computing*, 1(1):11–33, 2004.
The canonical source for everything in [The root concept](#the-root-concept), [Attributes](#attributes), [Threats](#threats), and [Means](#means). The fault/error/failure causal chain, the fault-containment region, the five attributes, and the four means all trace to this paper. If you read one item, read this.

**Laprie, J.-C.** "From Dependability to Resilience." *38th IEEE/IFIP International Conference on Dependable Systems and Networks (DSN)*, fast abstract, 2008.
Extends the 2004 taxonomy with resilience defined as "the persistence of dependability when facing changes." Owned by [Resilience](#resilience); listed here too because it is a direct continuation of the root taxonomy by the same author.

### Threats and failure studies

Empirical and conceptual work on what [faults, errors, and failures](#threats) actually look like in deployed systems — the fault classes, their activation, and how failures cascade.

**Gray, J.** "Why Do Computers Stop and What Can Be Done About It?" Tandem Technical Report 85.7, 1985.
Formalized and popularized the Bohrbug / Heisenbug distinction: a Bohrbug is deterministic and reproducible (and so removable by testing); a Heisenbug is timing- or state-dependent and often vanishes under observation (and so is better handled by tolerance than removal). Gray's report coined "Bohrbug" and framed the Bohrbug/Heisenbug hypothesis; the term "Heisenbug" itself predates the report. This split is why the same system needs both [fault removal](#means) and [fault tolerance](#means). Also the early articulation of process-pairs and fail-fast — crash on detected corruption rather than continue — which both worked instances in this corpus inherit.

**Yuan, D., et al.** "Simple Testing Can Prevent Most Critical Failures: An Analysis of Production Failures in Distributed Data-Intensive Systems." *11th USENIX Symposium on Operating Systems Design and Implementation (OSDI)*, 2014.
A failure-propagation study: most catastrophic failures arise from a few lines of error-handling code that were never exercised. Belongs to threats because it characterizes how errors propagate to failures; doubles as evidence for [fault removal](#means), since the same study shows lightweight testing would have caught them.

**Cook, R.** "How Complex Systems Fail." 1998.
Eighteen short propositions on failure in complex sociotechnical systems: failures are normally the result of multiple latent faults combining, and the system runs in a degraded mode most of the time. A bridge to the resilience literature below.

### Fault prevention

Disciplines and constraints that stop faults being introduced in the first place — the [prevention](#means) means, and the culture that bets on it (TigerStyle/TigerBeetle).

**Holzmann, G.** "The Power of Ten: Rules for Developing Safety-Critical Code." *IEEE Computer*, 39(6):95–97, 2006.
Ten coding rules (bounded loops, no recursion, no dynamic allocation after init, dense assertions, restricted scope) that make code statically analyzable. The direct ancestor of TigerStyle. A prevention-first discipline: shrink the space in which faults can exist.

**Dijkstra, E. W.** "Go To Statement Considered Harmful." *Communications of the ACM*, 11(3):147–148, 1968.
The origin of structured control flow as a prevention discipline — restrict the language so whole classes of fault become unexpressible. The same move recurs in this corpus at the [type-system enforcement layer](#enforcement-layers): Rust forbidding shared mutable aliasing is "go-to considered harmful" applied to memory.

### Fault tolerance

Delivering correct service despite faults, via error detection plus recovery and redundancy — the [tolerance](#means) means. This is the branch under which both [worked instances](#worked-instances) sit.

▶ **Armstrong, J.** "Making Reliable Distributed Systems in the Presence of Software Errors." PhD thesis, KTH Royal Institute of Technology, Stockholm, 2003.
The thesis behind Erlang/OTP: state isolation, let-it-crash, and supervision. The direct primary source for [beam.md](./beam.md). Read it for the argument that recovery is more tractable than removal for software faults you cannot enumerate in advance.

**Schneider, F. B.** "Implementing Fault-Tolerant Services Using the State Machine Approach: A Tutorial." *ACM Computing Surveys*, 22(4):299–319, 1990.
The foundational treatment of replicated state machines: make a service fault-tolerant by running deterministic replicas that process the same input sequence in the same order. The theory behind TigerBeetle's consensus-replicated, deterministic single-threaded core — redundancy across replicas rather than isolation within a node (the SERIALIZE cure in [Cross-cutting patterns](#cross-cutting-patterns)).

**Oki, B., Liskov, B.** "Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems." *7th ACM Symposium on Principles of Distributed Computing (PODC)*, 1988.
The consensus protocol (predating and closely related to Paxos) that TigerBeetle implements for high availability. Pairs with Schneider: Schneider gives the state-machine model, VR gives a concrete agreement protocol to keep the replicas in step across view changes.

**Nygard, M.** *Release It! Design and Deploy Production-Ready Software*, 2nd ed. Pragmatic Bookshelf, 2018.
The practitioner's catalogue of stability patterns: circuit breakers, bulkheads, timeouts, back-pressure. Where the academic sources give the theory of [fault-containment regions](#threats), this gives the operational patterns that implement them in service architectures.

### Fault removal

Reducing the number and severity of faults already present, via verification and testing — the [removal](#means) means.

**Yuan, D., et al.** (2014). See [Threats and failure studies](#threats-and-failure-studies) above. Its prescriptive half is a fault-removal argument: targeted testing of error paths removes most critical faults cheaply.

Deterministic simulation testing — the technique TigerBeetle uses to remove faults by replaying entire workloads (including injected faults) against a deterministic build — does not have a single canonical paper in this list; it is treated in [Cross-cutting patterns](#cross-cutting-patterns) and the TigerBeetle [worked instance](#worked-instances). Schneider (above) supplies the determinism requirement that makes such replay sound.

### Fault forecasting

Estimating present and future fault behavior — the [forecasting](#means) means, and the operational practice that turns estimates into targets.

**Beyer, B., Jones, C., Petoff, J., Murphy, N. R. (eds.).** *Site Reliability Engineering: How Google Runs Production Systems*. O'Reilly, 2016.
SLOs, error budgets, and the operational machinery for measuring and forecasting reliability at scale. Connects forecasting to the [measures](#measures) (nines, SLOs) and to the [MTTF/MTTR lever](#the-mttf-and-mttr-lever): error budgets are the management interface to that trade.

**Basiri, A., et al.** "Chaos Engineering." *IEEE Software*, 33(3):35–41, 2016.
Forecasting by experiment: inject faults into production to discover how the system behaves before a real fault forces the question. Empirical fault forecasting; also probes the unanticipated faults that [Resilience](#resilience) is concerned with.

### Resilience and safety

The extension of dependability to changing and surprising conditions, drawn from safety science — owned by [Resilience](#resilience).

**Hollnagel, E., Woods, D. D., Leveson, N. (eds.).** *Resilience Engineering: Concepts and Precepts*. Ashgate, 2006.
The founding collection of resilience engineering: adaptive capacity, coping with surprise, and the gap between work-as-imagined and work-as-done. The frame for why let-it-crash recovers from faults that were never enumerated.

**Leveson, N.** *Engineering a Safer World: Systems Thinking Applied to Safety*. MIT Press, 2011.
Treats [safety](#attributes) as a system control property rather than a component-reliability property (the STAMP model). The argument that safety and reliability are distinct attributes — a system can be reliable and unsafe, or safe and unreliable — which the taxonomy encodes by listing them separately.

### Measures and operations

Quantifying dependability and operating against the numbers — feeds [Measures](#measures) and [The MTTF and MTTR lever](#the-mttf-and-mttr-lever).

**Beyer, B., et al. (eds.).** *Site Reliability Engineering* (2016). See [Fault forecasting](#fault-forecasting). Filed primarily there; its definitions of availability, SLOs, and error budgets are the operational source for [Measures](#measures).

**Nygard, M.** *Release It!* (2018). See [Fault tolerance](#fault-tolerance). Its stability-pattern catalogue is also operational guidance for keeping MTTR low.

### Reading order

For a first pass through the corpus:

1. **Avizienis et al. (2004)** ▶ — the taxonomy and vocabulary everything else reuses.
2. **Armstrong (2003)** ▶ — the fault-tolerance worked instance, and the cleanest argument for recovery over removal.
3. **Gray (1985)** ▶ — Bohrbug/Heisenbug and fail-fast, the bridge between removal and tolerance.

From there, follow the branch you care about: Schneider + Oki/Liskov for the consensus/replication path (TigerBeetle), Holzmann + Dijkstra for the prevention path, Hollnagel et al. + Leveson for resilience and safety, Beyer et al. for operations and measurement.