# Context Repository

A curated knowledge base for Falck, optimized for LLM context retrieval and organizational knowledge management.

## Purpose

This repository provides high-quality, stable documentation that can be easily retrieved and used as context for LLM interactions (Claude, ChatGPT, etc.) or future AI agents. It follows the principle of **trust over volume** - better to have 100 pages you can trust than 10,000 where some are wrong.

## Structure

### `falck-knowledge-base/`
The central knowledge base for Falck-specific information.

- **`domains/`** - Business capabilities (contracts, hr, finance, operations)
  - Organized by what the organization does, not who does it
  - Each domain contains processes, system usage, and domain-specific glossaries
  - Stable across organizational changes

- **`systems/`** - Technical systems documentation
  - Comprehensive guides for tools and platforms (Ironclad, Workday, etc.)
  - What it is, how it works, how we use it, how to get access
  - Reusable context that can be referenced from multiple domains

- **`processes/`** - Cross-cutting workflows
  - Processes that span multiple domains or teams
  - Examples: vendor onboarding, compliance reporting

- **`people/`** - Organizational structure and contacts
  - Current org chart and team structures
  - Team → domain ownership mappings
  - Updated as organization changes

- **`glossary.md`** - Central terminology
  - Cross-cutting, established terms used across Falck
  - For ambiguous terms, points to domain-specific glossaries

### Other Directories

- **`instructions/`** - Multi-turn behavioral context
  - Shapes how agents/chatbots work across entire conversations
  - Applied once at conversation start (system prompt layer)
  - Defines "who you are" and "how you approach problems"
  - Examples: search methodology, agent personality, context engineering principles

- **`prompts/`** - Single-purpose transformations
  - Discrete input → output tasks
  - Often one-shot, sometimes chained in sequences
  - Defines "do this specific thing"
  - Examples: clean transcript, summarize meeting, change writing style

- **`concepts/`** - Universal reference knowledge
  - Knowledge that applies anywhere, not Falck-specific
  - Stable reference material (AI, software, domain concepts)
  - Rarely updated - canonical knowledge often in LLM training already

## How to Use This Repository

### For Manual LLM Context
1. Navigate to relevant domain or system documentation
2. Copy-paste markdown content into your LLM conversation
3. Reference multiple docs as needed for complete context

### For Building Chatbots/Agents
- Use documentation as system prompts or RAG source material
- Each document is comprehensive and context-efficient
- Frontmatter provides metadata (owner, last updated, relevance)

### For Knowledge Management
- Every document has an owner responsible for accuracy
- Documentation is the work, not something done after
- Start with high-value, frequently-referenced content
- Promote mature documentation from project repos to central KB

## Knowledge Base Principles

### 1. The Knowledge Base Is the Product
Treat it as a product with a roadmap, owner, and investment.

### 2. Trust Over Volume
Quality and freshness matter more than comprehensive coverage.

### 3. Every Page Has an Owner
Documents without owners become stale. Owner listed in frontmatter.

### 4. Make It Faster Than Asking
Searching must be faster than asking a colleague, or people will ask.

### 5. Documentation Is the Work
Documentation is part of completing the task, not an afterthought.

### 6. Never Migrate Garbage
Start clean with high-value pages. Don't bulk-migrate existing content.

## Document Standards

### Frontmatter
All knowledge base documents should include:
```yaml
---
owner: name (email)
owner_team: team name
last_updated: YYYY-MM-DD
status: draft | stable | deprecated
---
```

### Lifecycle
- **Project repos** contain raw notes, drafts, and public docs during active work
- **Central KB** receives promoted copies of mature, stable documentation
- Project repos remain source of truth; KB has published versions
- Documents updated in-place, versioned via git history

## Contributing

1. Identify frequently-referenced knowledge
2. Create or update documentation in appropriate location
3. Add required frontmatter
4. Ensure you or someone commits to being the owner
5. Keep it current - stale docs destroy trust

## Getting Started

Start by exploring:
- `domains/contracts/` - contract management processes
- `glossary.md` - key terminology
- `people/org-structure.md` - current organizational structure

---

**Repository Owner:** Patrik Persson (patrik.persson@falck.com)
**Last Updated:** 2025-11-05
