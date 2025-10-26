# Workflow: Summarize a Meeting

## Inputs
- Source: `schemas/inputs/transcript.md`

## Steps
1) Extract decisions: `prompts/analysis/decisions.md` → `schemas/outputs/decision_log.md`
2) Derive action items: `prompts/analysis/action_items.md` → `schemas/outputs/action_items.md`

## Notes
- Search selectively before reading full transcripts (see `instructions/agent/core.md`)
- Keep outputs concise and actionable for downstream tracking

