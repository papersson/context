# Ankify

Convert a learning session into spaced repetition cards. The input is either a session transcript or a LIVE CONTEXT STATE from the internalize.md prompt.

## Core Principles

**Cards test gaps, not prior knowledge.** If the user already understood something before the session, don't card it. Card what was built during the session.

**Cards are atomic.** One idea per card. If an answer has two parts, make two cards.

**Cards are in the user's voice.** Mirror how they talked during the session. Don't formalize or academicize.

**Answers must be memorizable.** If you can't say the answer in one breath, it's too long. Tighten until you can.

**Multiple angles on hard concepts.** Important ideas get 2-3 cards approaching from different directions—not redundant, but reinforcing.

## The Fundamental Selection Question

Anki reviews are zero-sum. Every card competes for limited maintenance budget. Before creating any card, ask:

**If I forget this, will I reason incorrectly?**

A card is load-bearing if forgetting it breaks downstream reasoning. A card is isolated if you pass it in review but never retrieve it outside of Anki.

Signs a card is load-bearing:
- It's a prerequisite for understanding something else
- Forgetting it leads to wrong conclusions, not just missing facts
- It fires during actual thinking about real problems
- It's a reference frame that calibrates intuition (e.g., relative speeds of storage media)

Signs a card is isolated:
- It's a fact you could look up in 10 seconds
- You'd naturally re-encounter and re-learn it anyway
- It describes but doesn't diagnose or enable
- It's true but connects to nothing you reason about

## Card Priority Hierarchy

Cards are sorted into three tiers. When generating, output cards in this order.

### Tier 1: Model Inversions (Highest Priority)

Beliefs the user held confidently that turned out to be backwards—not magnitude errors or fuzzy areas, but inverted mental models.

Signs of a model inversion:
- User assumed X causes Y, but actually Y causes X
- User assumed A is faster/better than B, but B beats A
- User conflated two distinct concepts as one
- User had cause and effect reversed
- User's intuition pointed confidently in the wrong direction

Model inversions are dangerous because they feel like knowledge. The user doesn't know they're wrong—they think they understand. These must be carded.

### Tier 2: Load-Bearing Fundamentals

Facts or principles where forgetting leads to wrong conclusions. Not all facts—only ones that are prerequisites for correct reasoning.

Two sub-types:

**Reference frames**: Quantitative intuitions that calibrate architectural decisions. "HDD random access is ~10x slower than same-datacenter network" changes how you design systems. "PostgreSQL uses 8 KiB pages" doesn't.

**Prerequisite unlocks**: Understanding X requires Y. If Y decays, X becomes shaky. Card the dependency relationship, not just the fact.

### Tier 3: Breakthroughs (User's Synthesis)

Moments where the user synthesized a new way of seeing—not a fact they read, but a framing they built.

Signs of a breakthrough:
- User said "Oh, so it's just..." or "Wait, the whole point is..."
- The insight connects previously separate concepts
- It's in the user's words, not the source text's vocabulary
- Forgetting it means losing the synthesis, not just a fact

Breakthroughs are valuable because they can't be looked up. They're the user's own construction.

## Card Types by Function

Prefer cards that diagnose or enable over cards that merely describe.

| Type | Pattern | Value |
|------|---------|-------|
| **Describe** | "X is Y" | Lower—lookup table, easily re-learned |
| **Diagnose** | "When you see S, the cause is C" | Higher—pattern matcher for real problems |
| **Enable** | "To do X, you need Y first" | Higher—unlocks action and further learning |
| **Synthesize** | "X and Y are both instances of Z" | Highest—user's own framing |

## Card Selection Criteria

For each potential card, evaluate:

| Criterion | Question |
|-----------|----------|
| **Model inversion?** | Did this correct a confidently-held wrong belief? |
| **Load-bearing?** | If forgotten, will I reason incorrectly? |
| **Synthesis?** | Is this a framing I built, not a fact I read? |
| **Diagnose/Enable?** | Does this pattern-match problems or unlock action? |
| **Decay risk** | Will this fade without reinforcement? |
| **Retrievability** | Can it be stated in one breath? |

**Always card:** Model inversions, breakthroughs in user's own words.

**Card if:** Load-bearing + diagnoses or enables + likely to decay + can be stated concisely.

**Skip if:** Merely descriptive, easily looked up, would be re-learned naturally, isolated from reasoning chains.

## Process

1. **Extract candidates.** Scan for:
   - Model inversions (Tier 1)
   - Load-bearing fundamentals and reference frames (Tier 2)
   - Breakthrough moments in user's words (Tier 3)
   - Prerequisites that had to be built
   - Failed probes that were later passed

2. **Filter ruthlessly.** Apply the load-bearing test: "If forgotten, will I reason incorrectly?" Aim for 5-10 cards. Fewer is better.

3. **Classify tier.** Assign each card to Tier 1, 2, or 3.

4. **Draft in user's voice.** Use their phrasing. Keep fronts as questions. Keep backs tight.

5. **Check atomicity.** If a back has "and" or multiple sentences, split.

6. **Add angles.** For the 2-3 most important concepts, add a second card from a different direction.

7. **Verify retrievability.** Read each back aloud. One breath or less.

8. **Sort by tier.** Output Tier 1 first, then Tier 2, then Tier 3.

## Output Format

Output cards in Mochi markdown format, sorted by priority tier:
```
# Tier 1: Model Inversions

[Front]
---
[Back]
===
[Front]
---
[Back]

# Tier 2: Load-Bearing Fundamentals

[Front]
---
[Back]
===
[Front]
---
[Back]

# Tier 3: Breakthroughs

[Front]
---
[Back]
```

Use `---` to separate front from back within a card.
Use `===` to separate cards from each other.
Use `# Tier N: Label` headers to group by priority.

If a tier is empty, omit it.

## Anti-Patterns

**Don't card facts that are merely descriptive.** "X is Y" cards are low-value unless Y is load-bearing for reasoning.

**Don't card isolated facts.** If it doesn't connect to reasoning chains, it won't fire outside of Anki.

**Don't card easily looked-up facts.** Your time is valuable. Card what you need in your head, not what you can retrieve in 10 seconds.

**Don't card implementation details.** Unless the concrete grounding was the breakthrough itself.

**Don't card lists.** "Name the 4 types of X" is a bad card. Card each type separately only if load-bearing.

**Don't card the example.** Card the principle or the diagnostic pattern.

**Don't over-card.** 6 load-bearing cards beat 15 isolated ones. When in doubt, leave it out.

**Don't treat magnitude errors as model inversions.** Off by 10x is a memorization gap. "Faster thing is actually slower" is a model inversion. Know the difference.

## Example Cards by Tier

**Tier 1: Model Inversion**
```
Satellite internet has high latency. Is this because satellite signals travel slower than fiber?
---
No—opposite. Light in vacuum (satellite) is faster than light in glass (fiber). Latency is high because distance is enormous.
```

**Tier 2: Load-Bearing Fundamental**
```
When does columnar storage lose its advantage over row-oriented?
---
When you need all columns for specific rows. The reassembly overhead dominates.
```

**Tier 2: Reference Frame**
```
HDD random access vs same-datacenter network round-trip—which is faster?
---
Network is ~10x faster. This inverts the "network is slow" intuition and changes architectural decisions.
```

**Tier 2: Enable**
```
An abstraction won't stick despite re-reading. What's likely missing?
---
The concrete implementation. "Index" stayed abstract until understanding B-trees. Find the implementation.
```

**Tier 3: Breakthrough**
```
What unifies materialized views, caches, document collections, and search indexes?
---
Derived data and keeping it consistent. They're all precomputed answers that must be maintained.
```

## Final Check

Before outputting, verify:
- [ ] Model inversions are carded and in Tier 1
- [ ] Each card passes the load-bearing test
- [ ] Breakthroughs use user's own words
- [ ] Cards diagnose or enable, not just describe
- [ ] Each back is one breath or less
- [ ] No isolated facts that won't fire outside Anki
- [ ] Cards are sorted by tier
- [ ] Fewer than 10 cards total (ideally 5-7)
