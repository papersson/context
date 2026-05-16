# Context Library

Reusable context for LLM interactions, organized by topic.

Each file should be useful on its own. Prefer self-contained documents over documents that require following links. Some overlap is acceptable when it makes a document easier to use as a standalone prompt or reference.

## Structure

```text
ai/                    AI systems, agents, context engineering
  agent-instructions/  Drop-in directives for coding agents
  rl/                  Reinforcement learning for LLMs
analysis/              General-purpose audit and distillation prompts for any artifact
software/              Code quality, process, standards, systems
  architecture/        Codebase and system exploration mappers
  data/                Data-processing workload shapes and data-system concepts
  infrastructure/      Production infrastructure and deployment systems
  modeling/            Domain modeling, AND/OR notation, type design
  observability/       Monitoring, tracing, alerting, operational visibility
  performance/         Performance engineering, diagnostics, workload taxonomies
project/               Backlogs, prioritization
interviews/            Structured discovery conversations
writing/               Prose style, comprehension testing, flashcards
meetings/              Transcript cleaning and summarization
meta/                  Prompt-generation and research-spec tools
presentation/          Diagram and presentation-generation context
security/              Security review and audit context
```

## Usage

Everything here is context you feed to LLMs. Grab the smallest self-contained file that matches the task.

```bash
# Single file
cat software/de-slop.md | llm -s "$(cat -)" < diff.patch

# Combine context when the task genuinely spans topics
cat ai/context_engineering.md ai/hooks.md | llm -s "$(cat -)"

# Pipeline
cat transcript.json | llm -s "$(cat meetings/clean_teams_json_transcript.md)" | llm -s "$(cat meetings/summarize.md)"
```

## Principles

- **Organized by topic** -- find things by what you're working on, not by abstract type.
- **Self-contained** -- each file should work independently without requiring references to other files.
- **Practical** -- files should help with real LLM interactions, not just archive knowledge.
- **Quality over quantity** -- fewer well-crafted files beat many stubs.
- **Lightweight structure** -- add directories and indexes when retrieval pain demands them, not preemptively.
