# Interview Prompts

This directory contains system prompts for extracting information and clarifying intent.

These prompts act as active interviewers rather than passive question lists. They identify vague language ("fast," "easy," "robust"), force trade-off decisions, and maintain a running log of what has been decided.

## Prompt Selector

| Goal | File |
| :--- | :--- |
| **Clarify thoughts** regarding a fuzzy idea, decision, or architecture. Focuses on trade-offs and "anti-goals." | `self-clarification.md` |
| **Extract knowledge** from an expert. Maps mental models, ontology, and hidden heuristics. | `knowledge-extraction.md` |
| **Map a workflow.** Documents the actual process (exceptions, workarounds, "shadow IT") rather than the official policy. | `process-clarification.md` |
| **Define requirements.** Separates problems from solutions. Focuses on quantified success criteria and constraints. | `requirements-gathering.md` |

## The Live Context Artifact

Each prompt appends a **Live Context** block to the bottom of every response. This tracks decisions, definitions, and open conflicts in real-time.

Do not ignore this block. It serves as the save state for the conversation and the final specification when the interview concludes.

## Usage Modes

The prompts adjust their behavior based on the input provided.

**Interviewer Mode (Standard)**
Start talking or ask the model to help clarify a topic. It will interview you, using disambiguation loops and mirror checks to drill into your thinking.
*   *Use for: Brainstorming, rubber-ducking, architecture decisions.*

**Copilot Mode (Analyst)**
Paste notes, a brain dump, or a meeting transcript. The model will analyze the input for gaps and contradictions, update the Live Context, and suggest the next questions to ask a stakeholder. It will not ask you questions directly.
*   *Use for: Processing meeting notes or coaching during a live call.*

## Reducing Cognitive Load

These interviews require deep thinking. Reduce logistical friction to make them easier to use.

**Use Voice Input**
These prompts work well with voice dictation. Speaking allows for rambling, which often contains the tacit knowledge the prompts are designed to extract. The model filters the noise.

**Use Text Expanders**
Map these prompts to system snippets (e.g., Raycast, Espanso) for quick access.
*   `;int-self` → Pastes `self-clarification.md`
*   `;int-reqs` → Pastes `requirements-gathering.md`

**Inject Context**
The model starts with no context. Paste a primer alongside the prompt to skip basic questions.
> "Context: Senior Engineer. Stack: Python/React. Domain: FinTech. Risk Tolerance: Low."

**The "Synthesize" Command**
End the session at any time by typing "Synthesize." The model will stop questioning and output the final Live Context state and a summary of open items.
