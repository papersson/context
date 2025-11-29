# Ankify

Convert a learning session into spaced repetition cards. The input is either a session transcript or a LIVE CONTEXT STATE from the internalize.md prompt.

## Core Principles

**Cards test gaps, not prior knowledge.** If the user already understood something before the session, don't card it. Card what was built during the session.

**Cards are atomic.** One idea per card. If an answer has two parts, make two cards.

**Cards are in the user's voice.** Mirror how they talked during the session. Don't formalize or academicize.

**Answers must be memorizable.** If you can't say the answer in one breath, it's too long. Tighten until you can.

**Multiple angles on hard concepts.** Important ideas get 2-3 cards approaching from different directions—not redundant, but reinforcing.

## Card Priority: Model Inversions First

**Highest priority: cards that correct inverted mental models.** These are beliefs the user held confidently that turned out to be backwards or fundamentally wrong—not just magnitude errors or fuzzy areas.

Signs of a model inversion:
- User assumed X causes Y, but actually Y causes X
- User assumed A is faster/better than B, but B beats A
- User conflated two distinct concepts as one
- User had cause and effect reversed
- User's intuition pointed confidently in the wrong direction

Model inversions are dangerous because they feel like knowledge. The user doesn't know they're wrong—they think they understand. These must be carded.

**Magnitude errors are lower priority.** Being off by 10x on a reference number is a memorization gap, not a conceptual blind spot. Card these only if the magnitude itself is load-bearing for reasoning (e.g., knowing HDD is slower than network changes architectural decisions).

## Card Selection Criteria

For each potential card, evaluate:

| Criterion | Question |
|-----------|----------|
| **Model inversion?** | Did this correct a confidently-held wrong belief? |
| **Fundamentality** | Does this apply beyond this specific example? |
| **Was it a gap?** | Did the user struggle with this, or did it come easily? |
| **Generativity** | Does understanding this unlock reasoning about other things? |
| **Decay risk** | Will this fade without reinforcement? |
| **Retrievability** | Can it be stated in one breath? |

**Always card:** Model inversions.

**Card if:** Fundamental + was a gap + generative + likely to decay + can be stated concisely.

**Skip if:** User already knew it, or it's too specific to the example, or it's derivable from other cards.

## Card Types

**Concept cards:** What is X? / What does X mean?
- Use when a definition or mental model was built during the session.

**Reasoning cards:** Why X? / What causes X?
- Use when the user had to learn the *why*, not just the *what*.

**Application cards:** Given [situation], what would you do/expect?
- Use when transfer was tested and initially failed.

**Boundary cards:** When does X break? / What assumption does X require?
- Use when edge cases were explored.

**Reference frame cards:** What's the rough magnitude of X?
- Use when quantitative intuition was built.

**Inversion cards:** You assume X. When is this wrong?
- Use when a confident wrong belief was corrected. Frame the front as the trap—the intuition that leads you astray.

## Process

1. **Extract candidates.** Scan for:
   - **Model inversions first** (beliefs that were backwards, not just fuzzy)
   - Gaps that were resolved (from BLIND SPOTS SURFACED)
   - Breakthrough moments (from BREAKTHROUGH MOMENTS)
   - Prerequisites that had to be built (from DEPENDENCY GRAPH)
   - Failed probes that were later passed

2. **Filter.** Apply selection criteria. Aim for 5-10 cards per session. Fewer is better than more.

3. **Draft in user's voice.** Use their phrasing where possible. Keep fronts as questions. Keep backs tight.

4. **Check atomicity.** If a back has "and" or multiple sentences, split into multiple cards.

5. **Add angles.** For the 2-3 most important concepts, add a second card that approaches from a different direction (e.g., "Why X?" after "What is X?", or "When does X fail?" after "How does X work?").

6. **Verify retrievability.** Read each back aloud. If it takes more than one breath, tighten.

## Output Format

Output cards in Mochi markdown format:
```
[Front]
---
[Back]
===
[Front]
---
[Back]
```

Use `---` to separate front from back within a card.
Use `===` to separate cards from each other.

## Anti-Patterns

**Don't card facts that were never fuzzy.** The user's time is valuable. Card the gaps.

**Don't card implementation details.** Unless the concrete grounding was the breakthrough itself.

**Don't card lists.** "Name the 4 types of X" is a bad card. Card each type separately if they matter.

**Don't card the example.** Card the principle. "2M × 200 = 400M" is not a card. "How do you calculate total load from fan-out?" is.

**Don't over-card.** 6 good cards beat 15 mediocre ones. When in doubt, leave it out.

**Don't treat magnitude errors as model inversions.** Off by 10x is a memorization gap. "Faster thing is actually slower" is a model inversion. Know the difference.

## Example Transformation

**Session gap:** User didn't understand why writes can be async but reads cannot.

**Breakthrough moment:** "Oh—posting is save + deliver. User only waits for save."

**Bad card:**
```
What is the difference between sync and async operations in distributed systems?
---
Synchronous operations block the caller until completion. Asynchronous operations return immediately and complete in the background. In the context of social media timelines, posting can be async because the user only needs acknowledgment that the post was saved, while viewing must be sync because the user needs the actual data before they can proceed.
```

**Good card:**
```
Fan-out on write: why can posting involve background work but viewing cannot?
---
Posting = save + deliver to followers. User waits for save only.
Viewing = get the data. Nothing to defer.
```

**Example inversion card:**

**Model inversion:** User assumed satellite signals are slower than fiber because satellite is "worse technology."

**Bad card:**
```
How fast do satellite signals travel?
---
At the speed of light in vacuum, which is faster than light in fiber.
```

**Good card:**
```
Satellite internet has high latency. Is this because satellite signals travel slower than fiber?
---
No—opposite. Light in vacuum (satellite) is faster than light in glass (fiber). Satellite latency is high because distance is enormous.
```

## Final Check

Before outputting, verify:
- [ ] Model inversions are carded (highest priority)
- [ ] Each card tests one thing
- [ ] Each back is one breath or less
- [ ] Cards target gaps, not prior knowledge
- [ ] Important concepts have multiple angles
- [ ] No lists or enumerations
- [ ] Phrasing matches user's voice
