# Architecture Mappers

Multi-repo exploration prompts that extract high-signal architectural overviews.

## Purpose

These prompts scan multiple repositories in parallel to build unified architectural views. Each prompt applies a specific lens — data, services, build, code, observability — to understand one dimension of a system.

**Use case:** You have 10-50 repos and need to understand how they fit together without reading everything manually.

## How It Works

All prompts follow the same pattern:

```
Input: List of repository paths
     ↓
Phase 1: Parallel exploration (one Task per repo)
     ↓
     Per-repo markdown reports
     ↓
Phase 2: Synthesis
     ↓
Output: Unified overview + Mermaid diagram
```

**Phase 1** spawns independent Tasks that explore each repo looking for specific signals (config files, schemas, dependencies, etc.). Each Task produces a structured markdown report.

**Phase 2** reads all reports, reconciles naming differences, and synthesizes a unified picture with a visual diagram.

## Available Maps

| Prompt | Lens | Key Questions Answered |
|--------|------|------------------------|
| [data.md](data.md) | Data | What stores exist? Who owns them? How does data flow? |
| [service.md](service.md) | Services | What services exist? How do they communicate? How are they deployed? |
| [build.md](build.md) | Build/Deploy | How is code built? What CI/CD exists? Where do artifacts go? |
| [code.md](code.md) | Codebase | How is code organized? Who owns what? What are the dependencies? |
| [observability.md](observability.md) | Observability | How is the system monitored? Logging, metrics, tracing, alerting? |

## Usage

1. Copy the prompt into a conversation with an agent that can spawn Tasks
2. Replace `[LIST GOES HERE]` with your repository paths:
   ```
   ## Repositories to Explore

   - /path/to/repo-a
   - /path/to/repo-b
   - /path/to/repo-c
   ```
3. Run — the agent will explore in parallel and synthesize

**Example:**
```bash
# Prepare prompt with repo list
cat prompts/architecture/service.md | sed 's|\[LIST GOES HERE\]|/repos/api\n/repos/worker\n/repos/gateway|' > prompt.txt

# Run with Claude Code or similar
claude -s "$(cat prompt.txt)"
```

## Combining Maps

Run multiple mappers on the same repos to build a complete picture:

1. **service.md** — What exists and how it's deployed
2. **data.md** — How data moves through those services
3. **build.md** — How code becomes running software
4. **observability.md** — How you'd debug it in production

Each map is independent — run them in parallel or sequence based on your needs.

## Adding New Maps

To create a new architectural lens, follow this template:

```markdown
# [X] Architecture Mapper

You are tasked with mapping the [X] architecture across a set of repositories...

## Approach
[Same parallel exploration pattern]

## Phase 1: Per-Repository Exploration

### Task Instructions: Repository Explorer

**Where to start looking:**
[Files and patterns specific to this lens]

**Scope guidance:**
[What to focus on, what to ignore]

**Output a markdown report with these sections:**
[Sections specific to this lens]

## Phase 2: Synthesis

[Reconciliation steps]
[Unified output template]
[Mermaid diagram spec]

## Repositories to Explore

[LIST GOES HERE]

## Execution

Spawn all repository Tasks in parallel...
```

Key principles:
- Each lens has specific "where to look" guidance
- Per-repo output is structured and consistent
- Synthesis reconciles naming and builds unified view
- Mermaid diagram visualizes the architecture
- Parallel execution for speed

## Limitations

These prompts work best when architectural information is in code:
- Config files, manifests, schemas
- Dependency declarations
- Infrastructure-as-code

They may miss:
- Tribal knowledge not in repos
- External services without code references
- Runtime behavior that isn't declared

For security architecture specifically, sensitive details are often not in repos — expect gaps that need human input.
