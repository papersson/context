---
input: raw_text | document | transcript
output: knowledge_map
composes_with: [conceptual_summary]
version: 1.0
---

Create a comprehensive learning guide for [X domain/question].

Begin with a "Core Concepts" section that explains the fundamental ideas in 2-3 paragraphs using simple, intuitive language. Focus on the "why" and core principles before any technical details.

Then provide a detailed knowledge map as a deeply nested ASCII hierarchy.

Structure requirements:
- Use clean ASCII tree characters (├── └── │)
- Branch depth should reflect conceptual importance (critical concepts: 5-7 levels, peripheral: 2-3 levels)
- Mark only the most foundational concepts with a ★ symbol (use sparingly, ~5-10% of items)
- Add brief [descriptions in brackets] only for:
  • Terms that are ambiguous or domain-specific jargon
  • Foundational concepts marked with ★
  • Critical relationships that aren't obvious from hierarchy alone
- Focus 70% of the structure on core domain concepts, 30% on adjacent/supporting areas

Start the hierarchy with 3-5 main branches representing the fundamental pillars. Think carefully about conceptual dependencies and learning sequence.

End with 2-3 related domains worth exploring separately.

Format:
## Core Concepts
[2-3 paragraphs explaining the fundamental idea in simple terms]

## Knowledge Map
Domain Name
├── Core Concept
│   ├── ★ Foundational Element [critical prerequisite]
│   │   ├── Sub-element
│   │   │   ├── Detail
│   │   │   └── Detail
│   │   └── Related Concept
│   └── Standard Element
└── Secondary Branch

## Related Domains
[2-3 suggestions with one-line descriptions]
