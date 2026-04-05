# Agent Tool Interfaces: A First Principles Analysis

## 1. The Problem

LLMs have exactly one capability: given text, produce text.

They cannot directly:
- Read from anywhere (files, APIs, databases)
- Write to anywhere (files, APIs, databases)
- Execute code
- Make network requests
- Maintain state between calls

**The boundary is the context window.** During inference, the LLM can only see what's in the context. Everything else is inaccessible unless something:
- Puts it into context (for reading)
- Interprets LLM output as instructions (for writing/acting)

### The External World

"External world" = anything not in the current context window.

| Layer | Examples |
|-------|----------|
| Local filesystem | Files, directories |
| Local processes | Shell commands, scripts, running programs |
| Local services | Databases, localhost APIs |
| Remote services | SaaS APIs, third-party APIs, the internet |
| Physical devices | Printers, IoT, hardware |

To access any of these, a **tool layer** is required. The tool layer bridges LLM reasoning and external effects:
- **Reading**: Tool fetches external state → injects into context
- **Writing/Acting**: Tool interprets LLM output → executes against external state

---

## 2. First Principles

### Bedrock Constraints

| ID | Type | Principle |
|----|------|-----------|
| C0 | Constraint | LLMs cannot act on the external world directly. A tool layer is required to bridge reasoning and external effects. |
| C1a | Constraint | **Pre-training** encodes deep knowledge of stable, widely-represented patterns (Unix, CLIs, common protocols). This is the LLM's strong foundation—50 years of representation. |
| C1b | Constraint | **Post-training** can adapt LLMs to newer patterns (MCPs, Skills, specific tools). This is narrower and controlled by the trainer. |
| C2 | Constraint | Context windows are finite. Every token has opportunity cost. |

### Parked (Needs Refinement)

| ID | Type | Principle |
|----|------|-----------|
| C3 | [Parked] | LLMs cannot self-verify correctness. Feels true but weight unclear relative to C0-C2. May be derivative rather than fundamental. |

### Axioms (Strong Preferences, Not Yet Grounded)

These emerged during analysis but don't derive cleanly from C0-C2:

- **Prefer rigor/testability** — Structured, testable interfaces over convenient but opaque ones
- **Transparency** — Ability to see and verify what the agent does
- **Maintainability** — Minimize moving parts, avoid unnecessary abstractions
- **Dual-use** — Tools that work for both humans and agents

---

## 3. The Solution Space

Given C0 (LLMs need a tool layer), there are multiple architectures for that layer:

| Approach | What It Is | Discovery | Execution |
|----------|------------|-----------|-----------|
| **Raw Bash** | LLM writes shell commands directly | LLM's training knowledge | Shell executes |
| **MCPs** | Protocol-based tool servers (JSON-RPC) | Tool descriptions in system prompt | MCP server executes |
| **Skills** | Markdown instructions + bundled scripts | YAML metadata → instructions → files | Bash executes scripts |
| **CLIs** | Standalone command-line programs | `--help` (on-demand) | Shell executes CLI |

All solve C0. They differ in how well they address C1a, C1b, and C2.

### Shared Structure

Every tool interface must handle:

1. **Discovery** — How does the LLM learn what capabilities exist?
2. **Invocation** — How does the LLM express intent to use a capability?
3. **Execution** — How does the action actually happen?

---

## 4. Comparative Analysis

| Dimension | CLIs | Skills | MCPs |
|-----------|------|--------|------|
| Leverages pre-training (C1a) | **Strong** | Weak | Weak |
| Benefits from post-training (C1b) | Unlikely | Yes | Yes |
| Context efficiency (C2) | Good (`--help`) | Good (progressive) | Poor (upfront descriptions) |
| Rigor/testability | **Strong** | Weak | Medium |
| Human usability | **Native** | Agent-only | Agent-only |
| Barrier to create | Medium-high | **Low** | Medium |
| Ecosystem/reuse | Weak (no marketplace) | Growing | **Strong** |
| Maintenance burden | You maintain | You maintain | **Others maintain** |
| Infrastructure needs | None (shell) | None (filesystem) | Server process |
| Works on hosted infra | No | Yes | Yes |

---

## 5. The Case for Each Approach

### When CLIs Win

| Argument | Derives From |
|----------|--------------|
| Leverages pre-training knowledge—LLMs know CLI patterns deeply | C1a |
| On-demand discovery via `--help`—don't pay context cost until needed | C2 |
| Progressive disclosure: help → subcommand help → execute | C2 |
| Rigorous and testable—standard testing frameworks, typed args, CI/CD | Axiom: rigor |
| Human-debuggable—you can run exactly what the agent ran | Axiom: transparency |
| No new abstractions—uses 50 years of Unix conventions | C1a + Axiom: maintainability |
| Works outside LLM environment—humans and scripts use it directly | Axiom: dual-use |
| Structured interface—argument parsing, validation, subcommands | Better than Skills |
| Composable via Unix pipes | Existing pattern |

**Choose CLIs when:**
- You're building for a local agent you control
- You want rigor, testability, human-debuggability
- You're wrapping internal/custom services anyway
- You don't trust abstractions you don't control
- You value pre-training leverage (C1a) over post-training bets (C1b)

### When Skills Win

| Argument | Reasoning |
|----------|-----------|
| Progressive loading is built-in | Metadata → instructions → resources, designed for C2 |
| Thinking-level guidance, not just syntax | "When to use," "how to think about this domain" |
| Low barrier to create | Markdown + bash, no build step |
| Anthropic is training on Skills (C1b) | Models will improve at Skills over time |
| Bundled resources | Schemas, templates, examples in one package |
| No server infrastructure | Filesystem-based (unlike MCPs) |
| Purpose-built abstraction | Native to agent use case |

**Choose Skills when:**
- You need to ship fast, iterate quickly
- The capability is more about "guidance" than "execution"
- You're betting on Anthropic's post-training investment
- Non-technical contributors need to author capabilities

### When MCPs Win

| Argument | Reasoning |
|----------|-----------|
| Protocol standardization | Any client can talk to any server |
| Ecosystem/marketplace | Hundreds of pre-built integrations |
| Maintained by others | Slack MCP maintained by community, not you |
| Anthropic training on MCPs (C1b) | Models improving at MCP patterns |
| Multi-tool composition | One client, many servers |
| Solves N×M problem | N agents × M services → N+M integrations |
| Server can be remote | Works for team/org setups |
| Works on hosted infrastructure | When you can't run arbitrary CLIs |

**Choose MCPs when:**
- The integration already exists (don't rebuild Slack/GitHub)
- You're building for hosted/shared infrastructure
- You want ecosystem benefits and external maintenance
- You're solving the N×M problem at org scale

---

## 6. Agent-Focused CLIs: Deep Dive

### What They Are

Distinct from general-purpose CLIs (git, curl, gh):
- **General-purpose CLIs** exist independent of agents; agents just use them well
- **Agent-focused CLIs** are built primarily for a specific agent to consume

Agent-focused CLIs trade some "dual-use" benefit for agent-optimized design, while retaining CLI rigor.

### Architecture: Three Tiers

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT CLI ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIER 1: External Known CLIs (leverages C1a)                    │
│  └─ gh, slack-cli, kubectl, etc.                                │
│     LLM knows these from pre-training                           │
│                                                                 │
│  TIER 2: Internal Service CLI (requires instruction)            │
│  └─ your-company-cli                                            │
│     Wraps internal APIs LLM has never seen                      │
│     Agent uses --help to learn                                  │
│                                                                 │
│  TIER 3: Workflow CLIs (optional)                               │
│  └─ Codified multi-step operations                              │
│     Tradeoff: determinism vs flexibility                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Tier 3 Decision Framework

| Workflow Type | Recommendation |
|---------------|----------------|
| Standardized, repeatable, high-stakes | Codify as CLI (e.g., `deploy-to-prod`) |
| Variable, exploratory, low-stakes | Let LLM orchestrate |
| Parameterized common path | CLI with flags for variations |

### Designing for Agent Experience (AX)

CLIs can be optimized for agent consumption:

```bash
mycli --help          # Usage/syntax (standard)
mycli --guide         # When to use, domain considerations, preferred approaches
mycli --examples      # Common workflows with context
mycli --json          # Structured output for parsing
mycli --dry-run       # Preview without executing
```

### Pros and Cons

**Pros:**

| Pro | Derives From |
|-----|--------------|
| Leverages pre-training (CLI patterns) | C1a |
| On-demand discovery (`--help`) | C2 |
| Rigorous, testable, structured | Axiom: rigor |
| Human-debuggable | Axiom: transparency |
| No new abstractions | C1a + maintainability |
| Works outside LLM environment | Axiom: dual-use |
| Composable via pipes | Unix convention |

**Cons:**

| Con | Implication |
|-----|-------------|
| Must instruct agent about custom CLIs | Costs some C2 |
| Must instruct "use `--help` first" | System prompt overhead |
| More work to build than Skills | Real engineering required |
| Potential duplication across CLIs | Multiple CLIs may wrap same APIs |
| No agent-focused marketplace | Ecosystem gap (solvable) |
| Coupling to specific agent | Less reusable than general-purpose CLIs |
| No post-training optimization (C1b) | Anthropic trains on MCPs/Skills, not your CLI |

---

## 7. Derivation Chain

How decisions trace back to principles:

| Decision | Derives From |
|----------|--------------|
| Use external known CLIs (gh, etc.) directly | C1a — LLM already knows them |
| Build internal CLI for unknown APIs (not MCP/Skill) | C1a (CLI pattern known) + C2 (`--help` cheaper than front-loading) |
| Express capabilities as CLIs rather than Skills | Axiom: rigor/testability (doesn't derive from C0-C2) |
| Codify high-stakes workflows as CLI commands | C2 (one call < many) + Axiom: determinism |
| Let LLM orchestrate variable workflows | C1a — can improvise with known patterns |

---

## 8. Open Questions

### Axioms Not Fully Grounded

These preferences don't derive cleanly from C0-C2:
- Rigor/testability preference
- Transparency preference
- Maintainability / "no unnecessary abstractions"
- Dual-use (humans + agents)

Are these independent values, or do they trace to something deeper?

### Ecosystem Gaps

CLI approach lacks:
- Agent-focused marketplace/registry
- Standard for "agent-friendly CLI conventions"
- Curation layer on existing package managers

These are solvable but currently unsolved.

### Hybrid Approaches

Not fully explored:
- Use MCPs for commodity integrations (Slack, GitHub) you don't want to maintain
- Use CLIs for internal/custom services where you want control
- When does hybrid make sense vs. picking one approach?

---

## 9. Summary

**The fundamental problem:** LLMs cannot access the external world (filesystem, processes, services, APIs, devices) without a tool layer.

**Three architectures for the tool layer:** MCPs (protocol-based), Skills (markdown + scripts), CLIs (standalone programs).

**The case for agent-focused CLIs:**
- Leverage deep pre-training on CLI patterns (C1a)
- Efficient context use via on-demand `--help` (C2)
- Rigor, testability, human-debuggability
- No new abstractions to learn or maintain
- Trade-off: more work to build, no ecosystem, no post-training optimization

**The case against (or for alternatives):**
- MCPs: ecosystem, external maintenance, N×M problem
- Skills: low barrier, thinking-level guidance, post-training investment

**Architecture for agent-focused CLIs:**
- Tier 1: External known CLIs (leverage C1a)
- Tier 2: Internal service CLI (wrap unknown APIs)
- Tier 3: Optional workflow CLIs (determinism vs flexibility tradeoff)
