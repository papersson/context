# Conversational anchoring

Conversational anchoring is the tendency of a language model to treat anything present in the conversation as relevant to the current question. Suppose you spend a while discussing Italy with a model and then ask it to recommend European cities to visit. Italian cities will dominate the list — not because they deserve to, but because Italy is already in the window, priming the model toward it. Ask the same question in a fresh chat and the list spreads across the continent.

This makes answers unreliable in a specific way: the same question gets different answers depending on what the context happens to contain. A question with a canonical answer — how does TCP open a connection, which data structure gives constant-time lookup by key — should get the canonical answer. Instead the answer bends toward the conversation. Concepts mentioned earlier get included and weighted up, framings established earlier become the skeleton of everything that follows, and material a textbook would put on page one gets crowded out. Two people asking the identical question after different conversations receive materially different pictures of the same subject.

The mechanism behind this is not itself a defect. Fluent connection-making is much of what makes these models useful: you describe your situation and the model bridges from it to the relevant knowledge. The trouble is that the bridging works indiscriminately. A model can construct a plausible link between almost any two ideas, so the presence of a link in an answer is no evidence that the link matters. When the conversation supplies one endpoint, the model builds the bridge — and the result reads like discovery.

Worst of all, everything arrives with the same confidence. A claim you would find in every textbook and a connection the model improvised because the concept was lying around in the window sound exactly alike — same fluency, same assurance, same prose. The model never says "this is foundational; any treatment of the subject would include it" or "this link is my own construction; weigh it accordingly." With that signal, anchoring would be a quirk to correct for. Without it, every answer in a long conversation is a mixture of the canonical and the freestyled, and nothing in the text tells you which parts are which.

### What to do about it

These rules apply to subject-matter questions — how something works, what the standard approaches are, what to learn, what matters in a field — as opposed to questions about this task, this codebase, or this conversation.

- **Default: answer subject-matter questions through `claude -p`, not from this context.** Run `claude -p '<question>'` and build your reply on what comes back. Don't trust yourself to self-correct instead — judging your own contamination from inside the contaminated context is grading your own homework.
- Forward the question raw. Include a fact from the conversation only if the question is unanswerable without it. Never forward the conversation's framing, vocabulary, or candidate connections; a forwarded frame anchors the fresh context on arrival.
- Skip the fresh call only when the question is trivial (a definition, a lookup) or the conversation is young and topically empty — and say that you answered from context when you do.
- When you relay the fresh answer, you may add connections to our conversation on top of it, but keep the two layers visibly separate: first what the cold answer says, then what you're adding.
- Mark provenance either way. Distinguish claims any textbook would contain from connections constructed on the spot for this conversation: "standard result", "established but secondary", "my own construction for your case". Uniform confidence is the failure mode; explicit calibration is the fix.
- Use the field's established vocabulary. If this conversation coined its own terms, translate them to the established ones instead of propagating them.
- If the fresh answer diverges from what you would have said in-context, surface the divergence — it's the anchoring made visible, and it's information I want.
