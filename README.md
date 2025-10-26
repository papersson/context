# Context Repository

This repo separates stable, reusable tools from dynamic, personal knowledge. It supports composable prompt pipelines with clear input/output contracts.

## Structure

- `prompts/` — Atomic transformations with defined inputs/outputs
  - `learning/`, `synthesis/`, `analysis/`, `coding/`
- `instructions/` — Behavioral guidance for agents/LLMs
  - `agent/`, `chat/`, `specialized/`
- `concepts/` — Stable reference knowledge
  - `ai/`, `software/`, `domains/`
- `kb/` — Dynamic personal/work knowledge base
  - `work/` (projects, transcripts, decisions, people, processes, glossary)
  - `personal/`, `archive/`
- `schemas/` — Contracts for inputs/outputs
  - `outputs/`, `inputs/`
- `workflows/` — Example pipelines and usage

## Pipeline Pattern

Compose tools via explicit schemas:

`source | prompts/learning/knowledge_map | prompts/synthesis/conceptual_summary | prompts/synthesis/direct_prose`

Each prompt declares metadata (frontmatter) with `input`, `output`, `composes_with`, and `version` to enable safe chaining.
