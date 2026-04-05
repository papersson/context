# Codebase Architecture Mapper

You are tasked with mapping the codebase architecture across a set of repositories. Your goal is to understand how code is organized - the relationships between repos, shared code, ownership patterns, and structural conventions.

## Approach

You will use a parallel exploration strategy:

1. **Spawn one Task per repository** - these can run in parallel since they are independent
2. **Synthesize results** - once all Tasks complete, combine findings into a unified architecture view

## Phase 1: Per-Repository Exploration

For each repository in the provided list, spawn a Task with the following instructions:

---

### Task Instructions: Repository Explorer

Explore this repository to understand its structure and relationships to other code.

**Where to start looking:**
- Root directory structure (get the lay of the land)
- README (often explains organization)
- Package manifests: package.json, requirements.txt, go.mod, Cargo.toml, pom.xml, BUILD files
- CODEOWNERS, OWNERS, MAINTAINERS files
- Documentation: docs/, doc/, wiki/
- Module structure: src/, lib/, pkg/, internal/, packages/

**Scope guidance:**
Focus on code organization, not runtime behavior. Understand what code lives here, how it's structured, who owns it, and how it relates to other repos.

**Output a markdown report with these sections:**
```
# Repo: [path]

## Repo type
[What kind of repo is this?
- Monorepo (multiple projects/services)
- Single service
- Shared library
- Infrastructure config
- Documentation
- Tooling/scripts
- Monorepo component (lives within larger monorepo)]

## Languages and frameworks
[Primary technologies:
- Language(s)
- Major frameworks
- Runtime/platform]

## Internal structure
[How is code organized within this repo?
- Top-level directory layout
- Major modules/packages
- Separation patterns (by feature, by layer, etc.)
- Test organization]

## Internal dependencies
[References to other internal repos or packages:
- Shared libraries used (with repo/package names)
- Internal tools or utilities
- Generated code from other repos
- How dependencies are referenced (git submodule, package registry, vendored)]

## External dependencies
[Key external dependencies:
- Major frameworks/libraries (not exhaustive, just notable ones)
- Unusual or notable dependencies]

## Ownership
[Who owns this code?
- CODEOWNERS rules
- Team references in docs
- Maintainers listed
- Contact information]

## Code conventions
[Notable patterns and conventions:
- Code style / linting
- Commit conventions
- PR/review requirements
- Documentation standards]

## Monorepo structure (if applicable)
[If this is a monorepo:
- What projects/packages live here
- How they relate to each other
- Shared code patterns
- Build coordination]

## Key files
[Most important files for understanding this codebase:
- Main README
- Architecture docs
- CODEOWNERS
- Root build config]

## Uncertainty / notes
[Anything unclear, ambiguous, or worth flagging]
```

---

## Phase 2: Synthesis

Once all repository Tasks complete, synthesize the findings:

1. **Read all per-repo reports**

2. **Classify repos** - group by type (service, library, infra, etc.)

3. **Map relationships** - internal dependencies between repos

4. **Build the unified picture as markdown:**
```
# Codebase Architecture Overview

## Repository Inventory
[Categorized list of all repos:

### Services
[Service repos]

### Libraries
[Shared library repos]

### Infrastructure
[Infra config repos]

### Tools
[Tooling/scripts repos]

### Other
[Anything else]]

## Technology Landscape
[What technologies are used:
- Languages (and their prevalence)
- Major frameworks
- Common patterns across repos]

## Internal Dependency Graph
[How repos depend on each other:
- Shared libraries and their consumers
- Common utilities
- Generated code dependencies
- Circular dependencies if any]

## Code Sharing Patterns
[How code is shared across repos:
- Internal package registries
- Git submodules
- Vendoring
- Monorepo shared packages
- Copy-paste (antipattern but worth noting)]

## Ownership Map
[Who owns what:
- Teams and their repos
- Shared ownership areas
- Orphaned/unclear ownership]

## Conventions and Standards
[Common patterns:
- Standardized repo structure
- Shared linting/formatting
- Common documentation patterns
- Consistent vs inconsistent areas]

## Monorepo Analysis (if applicable)
[For monorepos:
- What lives together and why
- Internal package relationships
- Build coordination patterns]

## Structural Issues
[Problems identified:
- Orphaned repos
- Unclear ownership
- Inconsistent structure
- Tangled dependencies
- Duplicate code across repos]

## Uncertainty / Needs Investigation
[Unresolved ambiguities, low-confidence findings]
```

5. **Generate a mermaid diagram** showing:
   - Repos as nodes (shaped/colored by type)
   - Dependency edges between repos
   - Ownership groupings
   - Clusters for monorepo components

## Repositories to Explore

[LIST GOES HERE]

## Execution

Spawn all repository Tasks in parallel - there is no dependency between them. Once all complete, proceed to synthesis.
