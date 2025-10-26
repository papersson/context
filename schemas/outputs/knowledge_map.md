# Output Schema: knowledge_map

Defines the structure produced by `prompts/learning/knowledge_map.md`.

## Sections
- Core Concepts
- Knowledge Map (ASCII tree)
- Related Domains

## Core Concepts
- 2–3 paragraphs, simple language, explains the “why” before details
- No meta commentary about sources

## Knowledge Map
- Title line with domain name
- ASCII tree using `├──`, `└──`, `│`
- Depth reflects importance: critical 5–7 levels, peripheral 2–3
- Mark truly foundational nodes with `★` (about 5–10% of items)
- Optional bracketed descriptions only for:
  - Ambiguous terms or domain-specific jargon
  - Foundational nodes marked with `★`
  - Non-obvious critical relationships

Example:

```
Domain Name
├── Core Concept
│   ├── ★ Foundational Element [critical prerequisite]
│   │   ├── Sub-element
│   │   │   ├── Detail
│   │   │   └── Detail
│   │   └── Related Concept
│   └── Standard Element
└── Secondary Branch
```

## Related Domains
- 2–3 domains worth exploring separately, each with a one-line description

