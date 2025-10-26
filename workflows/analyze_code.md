# Workflow: Analyze Code

## Inputs
- Source: codebase files (searched and selected)

## Steps
1) Search: Use `instructions/agent/core.md` search strategy to locate relevant files
2) Understand: Summarize core logic with `prompts/synthesis/conceptual_summary.md` → `schemas/outputs/conceptual_summary.md`
3) Propose: Draft changes (diffs) guided by the summary and project norms

## Notes
- Prefer structural search where available (e.g., `ast-grep`), else `rg`
- Propose diffs for focused changes rather than full rewrites

