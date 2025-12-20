# Concepts

Universal reference knowledge that applies anywhere.

## Purpose

Concepts contain stable, canonical knowledge that is broadly applicable across organizations and contexts. This is reference material that helps LLMs understand domains, methods, or technical topics that may not be well-represented in their training data.

## When to Use Concepts

**Use Concepts for:**
- Universal knowledge that applies at any company
- Stable reference material (rarely changes)
- Domain concepts, technical methods, industry standards
- Knowledge to fill gaps in LLM training

## Decision Rule

**Ask: "Would this knowledge be useful at another company?"**

Examples:
- "How contract classification generally works in legal tech" → concepts/domains/legal/
- "Context engineering principles for LLMs" → concepts/ai/
- "Software architecture patterns" → concepts/software/

## Current Categories

### [ai/](ai/)
AI, ML, and LLM concepts.
- `context_engineering_draft.md` - Context engineering principles and patterns

### [domains/](domains/)
Domain-specific universal knowledge.
- Cross-industry domain concepts
- Examples: legal tech, healthcare operations, logistics

### [software/](software/)
Software engineering concepts.
- Patterns, architectures, methodologies
- Examples: design patterns, testing strategies, architecture styles

## When to Add Concepts

Concepts should be **rarely added**. Before adding:

1. **Check if it's truly universal**: Would this apply elsewhere?
2. **Check if LLMs already know it**: Is this canonical knowledge likely in training data?
3. **Check if it's actually needed**: Are you solving a real problem?

Most domain knowledge is already in LLM training. Add concepts only when:
- The topic is specialized or emerging
- You need specific framing or methodology
- LLMs consistently lack this knowledge
- It will be referenced frequently

## Adding New Concepts

1. Verify it's universal, not organization-specific
2. Verify it fills a gap in LLM knowledge
3. Create file in appropriate category: `concepts/category/name.md`
4. Focus on stable, reference-quality content
5. Include sources or citations where relevant
6. Update this README

## Using Concepts

### As LLM Context
Add concepts to conversations when you need to establish shared understanding of a domain or method:

```bash
# Example: Add context engineering knowledge to a conversation
cat concepts/ai/context_engineering_draft.md
```

### For Onboarding
Use concepts to educate LLMs (or people) on domains before diving into specific details.

## Related Documentation

- [Instructions](../instructions/) - Multi-turn behavioral context
- [Prompts](../prompts/) - Single-purpose transformations
