---
input: decision_log
output: action_items
composes_with: []
version: 0.1
---

# Action Items Synthesis

Convert decisions into concrete action items.

## Guidelines
- Each item has: owner, task, due date (if given or infer reasonable), status `todo`
- Keep tasks atomic and verifiable
- Reference the originating decision ID

## Output
Produce `action_items` per `schemas/outputs/action_items.md`.

