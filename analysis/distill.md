# Distill

Audit a first-iteration artifact: extract the compressed model it is now evidence for, separate accidental from essential complexity, and specify a cleaner rebuild from that model. Works on any artifact — code, design, schema, doc, system.

---

You are auditing a FIRST ITERATION of an artifact. A first iteration is
necessarily uncompressed: it was built before its own shape was understood,
so it carries incidental complexity that exists only because the builder
didn't yet know what mattered. Your job is to figure out, from scratch, what
this thing should have been — and the only way to do that without inheriting
the original's mistakes is to refuse to let its current shape anchor your
thinking.

I will give you:
- ARTIFACT: [code / design / schema / doc / system]
- PURPOSE: what it must do — its essential behavior, to be preserved exactly
- AUDIENCE: who must read and maintain the rebuild, and what concepts/
  vocabulary that audience already shares
- HARD CONSTRAINTS: things that are essential (not incidental) and must survive

OPERATING PRINCIPLE (do not violate): target the shortest description that is
decompressible by the AUDIENCE — not the shortest description. Maximal
compression destroys understanding (code golf, slick-but-opaque proofs). The
goal is minimal length RELATIVE TO the concepts the audience already holds.
Choosing the right shared vocabulary is the actual work; shrinking is not.

ANCHORING is the single biggest failure mode of this audit. The phases below
are ordered to prevent it. Do them in order. Do not produce the rebuild while
still looking at the original's structure.

PHASE 1 — ABSORB BEHAVIOR

Read the artifact, but only for what it DOES. Translate everything into
purpose-language: inputs, outputs, invariants, edge cases handled, failure
modes survived.

Forbidden in this phase: naming the artifact's functions, modules, files,
fields, classes, or sections. If you catch yourself writing "the FooManager
does X" — stop. Write "X happens" instead. You are extracting behavior, not
cataloguing structure.

Output: PURPOSE-as-observed. A behavioral spec dense enough that a stranger
could verify the rebuild matches it.

PHASE 2 — SET ASIDE

The artifact is now gone. Treat it as if you'd never seen it.

The only inputs to the next phase are: PURPOSE, AUDIENCE, HARD CONSTRAINTS,
and the behavioral spec from Phase 1. Do not consult the artifact again
until Phase 4. If you cannot recall a behavior in Phase 3 without re-reading
the artifact, that is a hole in Phase 1's output — fix it there, not by
peeking.

State explicitly: "Artifact set aside. Working from spec only."

PHASE 3 — RE-DERIVE

Design from scratch. This is the deliverable.

Name the concepts and vocabulary in which this problem becomes short for the
AUDIENCE. State the names FIRST. Then show how the behavioral spec falls out
of them — not by enumerating cases, but by composition.

If your rebuild looks structurally similar to the original, that is a signal
you are still anchored. Try again with different vocabulary. The right model
often makes the original look unrecognizable, not improved.

PHASE 4 — RECONCILE

Bring the artifact back, with the polarity REVERSED: the rebuild is the
baseline; the artifact is the audit target.

For each behavior in the original not in your rebuild, classify:
- ABSORB — essential behavior you missed. Fold it in and show how the
  vocabulary already covered it, or extend the vocabulary minimally.
- LEAVE — scaffolding, build-order artifact, defensive cruft, or premature
  structure. State why it is incidental.

Default is LEAVE. Presence in the original is not evidence of essentiality;
the artifact has to earn its way back in by passing the absorption test
against PURPOSE and HARD CONSTRAINTS.

PHASE 5 — ADVERSARIAL SELF-CHECK

Apply to the rebuild, not the original.

a) SECOND-SYSTEM WARNINGS. Flag any place your proposed elegance is itself
   new incidental complexity: generality nobody asked for, abstraction
   serving the model rather than the PURPOSE, vocabulary the AUDIENCE
   doesn't already hold. Be adversarial toward your own cleverness.

b) GENERALIZATION CHECK. Name 2–4 plausible "neighbors": the same problem
   under one changed assumption. Show the model makes each cheap. If the
   model only reproduces the behavioral spec and no neighbors, say so —
   that is transcription, not understanding, and the model is not yet right.

Do not rewrite the full artifact unless asked. Deliver the model and the
audit; the rebuild is mechanical once the model is right.
