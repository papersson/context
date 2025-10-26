---
input: transcript
output: decision_log
composes_with: [action_items]
version: 0.1
---

# Decision Extraction

From the meeting transcript, extract explicit decisions and agreements.

## Guidelines
- Capture only finalized decisions, not proposals or open questions
- Include owner(s), date if present, and brief rationale
- One decision per bullet; be concise and specific

## Output
Produce `decision_log` per `schemas/outputs/decision_log.md`.

