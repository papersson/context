---
owner: Patrik Persson
last_updated: 2026-04-24
type: meta-prompt
usage: "Generate a deep, pedagogical topic-exploration prompt from a topic + current knowledge"
---

# Meta-Prompt: Generate Pedagogical Exploration Prompt

## Purpose
This prompt is a **prompt factory**. Given a technical topic and a brief description of what you already know, it emits a single self-contained prompt that guides a tutor-style LLM through the topic from first principles — building each step as a consequence of the previous step's unsolved problems.

It solves the problem that generic "explain X" prompts produce shapeless overviews: no progression, no tradeoffs, no grounding at the systems level.

## When to Use
1. **Filling a foundational gap.** You use something daily (HTTP, TLS, Kubernetes) but don't understand the layers underneath.
2. **Learning a new field.** You want the *history of problems* that shaped the current tools, not a tour of the tools.

## The Prompt Snippet
*Copy and paste the block below into your current LLM session, filling in `[TOPIC]` and the "What I already know" line:*

---

I need you to create a prompt that will guide a deep, pedagogical exploration of a technical topic. The prompt should follow this structure:

**Topic:** [TOPIC]

**What I already know:** [BRIEF DESCRIPTION OF EXISTING KNOWLEDGE, OR "starting from scratch"]

Generate a prompt that:

1. Starts from the simplest possible case / first principles
2. Builds a progression where each step emerges naturally from the problems of the previous step — don't predetermine the steps, let problems drive the narrative
3. At each step requires: what problem emerged, what the solution actually is at the systems/mechanical level, and what new problems it introduced
4. Identifies 5-8 threads (recurring concerns that weave through the progression) specific to this topic — these should be introduced with "weave in these threads wherever they naturally arise" rather than as separate sections
5. Identifies 6-10 questions specific to this topic that a thoughtful learner would ask — these should be introduced with "address these questions as they naturally arise in the progression" rather than as a FAQ
6. Specifies a target audience: "an experienced developer who understands [RELEVANT BASELINE KNOWLEDGE] but hasn't [SPECIFIC GAP]"
7. Demands concreteness: real examples, real formats, real commands, real config — not abstract descriptions. Tools and standards should be explained before being named.
8. Explicitly prohibits predetermining the progression: "don't predetermine what the inflection points are — let them emerge from the problems at each stage"

The tone should be direct and precise. No marketing language, no filler. The prompt should read like a brief from someone who knows exactly what kind of explanation they want but not the content itself.

Output only the generated prompt, nothing else.

---

## Example Invocations

```
Topic: Networking (from electrical signals to HTTP)
What I already know: I understand programming and can use HTTP APIs,
but I don't understand what happens below the application layer
```

```
Topic: Cryptography
What I already know: I use TLS and JWTs but don't understand the
primitives underneath — what signing, encryption, and hashing
actually do at the math level
```

## Iteration
Treat the meta-prompt as living. If the generated prompts are too long, too short, or missing a quality you care about, refine the snippet.
