# Context Repository

A curated collection of prompts, instructions, and concepts for LLM interactions.

## Purpose

This repository provides reusable building blocks for working with LLMs (Claude, ChatGPT, etc.). It separates concerns into three categories based on how and when context is used.

## Structure

### `prompts/`
Single-purpose transformations for discrete input → output tasks.

- **What:** "Do this specific thing right now"
- **When:** One-shot tasks, workflow building blocks
- **Examples:** Clean a transcript, summarize a meeting, extract requirements

Categories: interviews, modeling, meetings, learning, synthesis, activity, meta

### `instructions/`
Multi-turn behavioral context that shapes how agents work across entire conversations.

- **What:** "Who you are and how you approach problems"
- **When:** Applied once at conversation start (system prompt layer)
- **Examples:** Search methodology, agent personality, context engineering principles

Categories: agent, chat, specialized

### `concepts/`
Universal reference knowledge that applies anywhere.

- **What:** Stable, canonical knowledge for domains or methods
- **When:** Fill gaps in LLM knowledge, establish shared understanding
- **Examples:** Context engineering principles, domain concepts, technical methods

Categories: ai, domains, software

## Quick Start

### Using Prompts
```bash
# One-shot transformation
cat transcript.json | llm -s "$(cat prompts/meetings/clean_teams_json_transcript.md)"

# Chain prompts
cat raw.json | llm -s "$(cat prompts/meetings/clean_teams_json_transcript.md)" > clean.txt
cat clean.txt | llm -s "$(cat prompts/meetings/summarize_transcript.md)" > summary.txt
```

### Using Instructions
```bash
# Set up agent behavior
cat instructions/agent/core.md > system_prompt.txt
```

### Using Concepts
```bash
# Add domain knowledge to conversation
cat concepts/ai/context_engineering_draft.md
```

## Design Principles

1. **Separation of concerns** - Prompts do tasks, instructions shape behavior, concepts provide knowledge
2. **Composability** - Mix and match pieces for different workflows
3. **Self-contained** - Each file works independently
4. **Quality over quantity** - Better to have fewer, well-crafted prompts than many mediocre ones

## See Also

Each directory has its own README with detailed usage:
- [prompts/README.md](prompts/README.md)
- [instructions/README.md](instructions/README.md)
- [concepts/README.md](concepts/README.md)
