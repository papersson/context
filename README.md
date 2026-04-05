# Context Library

Reusable context for LLM interactions, organized by topic.

## Structure

```
ai/                 AI systems, agents, context engineering
  rl/               Reinforcement learning for LLMs
software/           Code quality, process, standards
  architecture/     Codebase exploration mappers
  modeling/         Domain modeling (AND/OR notation, types)
project/            Backlogs, prioritization
interviews/         Structured discovery conversations
writing/            Prose style, comprehension testing, flashcards
meetings/           Transcript cleaning and summarization
```

`generate_deep_research_spec.md` lives at root -- it's a domain-agnostic meta-tool.

## Usage

Everything here is context you feed to LLMs. Grab what's relevant to what you're working on.

```bash
# Single file
cat software/de-slop.md | llm -s "$(cat -)" < diff.patch

# Combine context
cat ai/context_engineering.md ai/hooks.md | llm -s "$(cat -)"

# Pipeline
cat transcript.json | llm -s "$(cat meetings/clean_teams_json_transcript.md)" | llm -s "$(cat meetings/summarize.md)"
```

## Principles

- **Organized by topic** -- find things by what you're working on, not by abstract type
- **Self-contained** -- each file works independently
- **Quality over quantity** -- fewer well-crafted files beat many stubs
