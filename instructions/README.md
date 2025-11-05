# Instructions

Multi-turn behavioral context for shaping how agents and chatbots work across entire conversations.

## Purpose

Instructions define **who the LLM is** and **how it approaches problems**. They are applied once at conversation start (system prompt layer) and persist throughout the session.

## Instructions vs. Prompts

**Use Instructions for:**
- Behavioral guidance that applies across an entire conversation
- "How to think" or "how to approach tasks"
- Agent personality and methodology
- System prompt content
- Applied once, influences all subsequent interactions

**Use Prompts for:**
- Single-purpose transformations (input → output)
- "Do this specific thing right now"
- One-shot tasks or workflow building blocks
- See [prompts/](../prompts/) instead

## Decision Rule

**Ask: "Is this persistent session behavior or a discrete task?"**

Examples:
- ✓ "When searching, use this methodology" → instructions/search/
- ✗ "Search for X and return results" → prompts/search/
- ✓ "You are a helpful agent who prioritizes clarity" → instructions/agent/
- ✗ "Rewrite this text for clarity" → prompts/writing-style/

## Current Categories

### [agent/](agent/)
Core agent behavior and personality.
- `core.md` - Fundamental agent behavioral principles

### [chat/](chat/)
Conversational interface behaviors.
- To be documented as patterns emerge

### [search/](search/)
Search methodology and approach.
- `search_knowledge_base.md` - How to search knowledge bases effectively

## Using Instructions

### For Agent Setup
Copy instruction files into your agent's system prompt at conversation start:

```bash
# Example: Setting up an agent with core behavior and search methodology
cat instructions/agent/core.md > system_prompt.txt
cat instructions/search/search_knowledge_base.md >> system_prompt.txt
```

### For Chatbots
Include instructions in the system message or configuration:

```python
system_message = open('instructions/agent/core.md').read()
```

### Combining Instructions
Instructions are composable. Combine multiple instruction files for specialized agents:
- Core agent behavior + search methodology
- Core behavior + domain-specific guidelines
- Chat behavior + interview technique

## Adding New Instructions

1. Verify it's behavioral, not task-specific
2. Ask: "Does this shape how the LLM works across multiple turns?"
3. Create file in appropriate category: `instructions/category/name.md`
4. Include clear examples of when to use it
5. Document in this README
6. Consider if it should be combined with other instructions

## Related Documentation

- [Prompts](../prompts/) - Single-purpose transformations
- [Concepts](../concepts/) - Universal reference knowledge
- [Falck Knowledge Base](../falck-knowledge-base/) - Falck-specific operational knowledge
