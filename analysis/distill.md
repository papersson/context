# Distill

Audit a first-iteration artifact: extract the compressed model it is now evidence for, separate accidental from essential complexity, and specify a cleaner rebuild from that model. Works on any artifact — code, design, schema, doc, system.

---

You are auditing a FIRST ITERATION of an artifact. A first iteration is
necessarily uncompressed: it was built before its own shape was understood,
so it carries incidental complexity that exists only because the builder
didn't yet know what mattered. Your job is to extract the compressed model
the artifact is now evidence for, and specify a cleaner rebuild from it.

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

Produce:

1. ACCIDENTAL COMPLEXITY TO DISCARD — parts that exist only as scaffolding,
   build-order artifacts, defensive cruft, or premature structure. For each:
   why it is incidental, not essential.

2. ESSENTIAL CORE TO PRESERVE LOSSLESSLY — the generative behavior the rebuild
   must reproduce exactly. Be specific; compression must not touch this.

3. THE COMPRESSED MODEL — the vocabulary, abstractions, and structure in which
   the artifact becomes short. State the names/concepts FIRST, then show how
   the rebuild falls out of them. The model is the deliverable; the rebuild is
   a consequence of it.

4. GENERALIZATION CHECK — name 2–4 plausible "neighbors": the same thing under
   one changed assumption. Show the model makes each cheap. If the model only
   reproduces the exact artifact and no neighbors, say so explicitly — that is
   transcription, not understanding, and the model is not yet right.

5. STOP POINT — identify the irreducible core that must NOT be compressed
   further, and why further abstraction there would add complexity, not remove it.

6. SECOND-SYSTEM WARNINGS — flag any place your proposed elegance is itself new
   incidental complexity (generality nobody asked for, abstraction serving the
   model rather than the PURPOSE). Be adversarial toward your own cleverness.

Do not rewrite the full artifact unless asked. Deliver the model and the audit;
the rebuild is mechanical once the model is right.
