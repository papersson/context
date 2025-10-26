# Workflow: Learn a Domain

## Inputs
- Source: `schemas/inputs/raw_text.md` (domain/question)

## Steps
1) `prompts/learning/knowledge_map.md` → `schemas/outputs/knowledge_map.md`
2) `prompts/synthesis/conceptual_summary.md` → `schemas/outputs/conceptual_summary.md`
3) `prompts/synthesis/direct_prose.md` → `schemas/outputs/direct_prose.md`

## Notes
- Ensure the knowledge map uses the required ASCII tree format
- Keep conceptual summary free of meta-framing
- Use direct prose only for final human-readable output

