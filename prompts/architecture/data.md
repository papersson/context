# Data Architecture Mapper

You are tasked with mapping the data architecture across a set of repositories. Your goal is to understand what data stores exist, how data flows between services, and identify any gaps where additional repositories may be needed.

## Approach

You will use a parallel exploration strategy:

1. **Spawn one Task per repository** - these can run in parallel since they are independent
2. **Synthesize results** - once all Tasks complete, combine findings into a unified architecture view

## Phase 1: Per-Repository Exploration

For each repository in the provided list, spawn a Task with the following instructions:

---

### Task Instructions: Repository Explorer

Explore this repository to understand its role in the data architecture.

**Where to start looking:**
- Infrastructure: docker-compose.yml, kubernetes manifests, terraform files
- Config: application.yaml, .env.example, settings.py, config/
- Dependencies: package.json, requirements.txt, pom.xml, go.mod, Cargo.toml (look for database clients, queue libraries)
- Schemas: migrations/, models/, protobuf files, avro schemas, OpenAPI specs
- Connection setup: database initialization, client instantiation

**Scope guidance:**
Focus on stores and flows where your organization's data lives and moves. External services are boundaries - note them but don't peer inside. Infrastructure for logging, monitoring, CI/CD can be ignored unless it holds business data.

**Output a markdown report with these sections:**
```
# Repo: [path]

## What this service does
[Brief description if determinable from README, code structure, or naming]

## Data stores
[What databases/caches it uses. Distinguish between:
- Stores this repo OWNS (defines schema, runs migrations, is source of truth)
- Stores this repo CONNECTS TO (reads/writes but doesn't own)
Include: store type, how connection is configured, what evidence you found]

## Message queues / event streams
[Topics or queues this repo produces to or consumes from.
Include: queue system, topic names, serialization format, schema location if found]

## APIs
[What this repo exposes (REST, gRPC, GraphQL) and what external services it calls.
Include: protocol, schema location, how external services are referenced]

## External dependencies
[Services, stores, or topics referenced in code but not defined in this repo.
These are candidates for "missing repos" - be specific about what's referenced and where]

## Key files
[Paths to the most important files for understanding data flow:
schemas, migrations, connection configs, main entry points]

## Uncertainty / notes
[Anything ambiguous, unclear, or worth flagging for the synthesizer]
```

---

## Phase 2: Synthesis

Once all repository Tasks complete, synthesize the findings:

1. **Read all per-repo reports**

2. **Reconcile naming** - different repos may refer to the same store/service/topic by different names. Match them up.

3. **Build the unified picture as markdown:**
```
# Data Architecture Overview

## Services
[List each service, what it does, what repo implements it]

## Data Stores
[Each store, its type, which service owns it, which services read/write]

## Event Streams / Message Queues  
[Each topic/queue, its system, producers, consumers, message schema]

## Data Flows
[Key flows: e.g. "Order created → orders-service writes to postgres + publishes to order-events → inventory-service consumes and updates inventory-db"]

## External Boundaries
[Third-party services, APIs, stores outside your system]

## Gaps and Missing Repos
[Specific list of:
- Topics with producers but no consumers (or vice versa)
- Services called but not found in any repo
- Stores referenced but not owned by any repo
Include: what's missing, which repo references it, file paths]

## Uncertainty / Needs Investigation
[Unresolved ambiguities, conflicting information, low-confidence findings]
```

4. **Generate a mermaid diagram** showing:
   - Services as nodes
   - Data stores as nodes (different shape)
   - Topics/queues as nodes (different shape)  
   - Edges for reads, writes, publishes, consumes

## Repositories to Explore

[LIST GOES HERE]

## Execution

Spawn all repository Tasks in parallel - there is no dependency between them. Once all complete, proceed to synthesis.
