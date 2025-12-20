# Service Architecture Mapper

You are tasked with mapping the service architecture across a set of repositories. Your goal is to understand what services exist, how they communicate with each other, and how they're deployed.

## Approach

You will use a parallel exploration strategy:

1. **Spawn one Task per repository** - these can run in parallel since they are independent
2. **Synthesize results** - once all Tasks complete, combine findings into a unified architecture view

## Phase 1: Per-Repository Exploration

For each repository in the provided list, spawn a Task with the following instructions:

---

### Task Instructions: Repository Explorer

Explore this repository to understand its role in the service architecture.

**First, determine what this repo is:**
- A deployable service?
- A shared library?
- Infrastructure configuration?
- A CLI tool?
- Something else?

Not everything is a service. Identify this early - it shapes what you look for.

**Where to start looking:**
- README (often describes what this is and how to run it)
- Dockerfile, docker-compose.yml (reveals how it runs)
- Kubernetes manifests: deployment.yaml, service.yaml, ingress.yaml
- Terraform, CloudFormation, Pulumi files
- API definitions: openapi.yaml, *.proto files, GraphQL schemas
- Entry points: main.go, index.ts, app.py, Application.java
- Config files: application.yaml, .env.example (reveals external dependencies)
- CI/CD config: .github/workflows, Jenkinsfile, .gitlab-ci.yml (reveals deploy targets)
- Client code: HTTP clients, gRPC stubs, SDK usage

**Scope guidance:**
Focus on service identity, communication patterns, and deployment shape. Note data stores lightly ("uses postgres", "publishes to kafka") but don't trace data flows deeply - that's a separate concern.

**Output a markdown report with these sections:**
```
# Repo: [path]

## What this is
[Service? Library? Infra config? CLI tool? Brief description of purpose]

## Service identity
[If this is a service:
- Service name (as referenced by others)
- What it does
- Main entry point
- Port(s) it listens on]

## APIs exposed
[What this service exposes to others:
- Protocol (HTTP/REST, gRPC, GraphQL, other)
- Schema location (openapi spec, proto files, etc.)
- Key endpoints or methods
- Public-facing vs internal-only]

## Services called
[Other services this repo communicates with:
- Service name (as referenced in code)
- Protocol
- How it's discovered (env var, hardcoded URL, service discovery)
- Sync (request/response) vs async (fire and forget)]

## Infrastructure
[How this service is deployed:
- Container? Serverless? Bare process?
- Orchestration (k8s, ECS, lambda, etc.)
- Scaling configuration if visible
- Ingress / load balancer / API gateway]

## Platform dependencies
[Cloud and platform services used:
- Cloud primitives (S3, SQS, etc.)
- Service mesh
- Secrets management
- Other infrastructure services]

## Data stores (light touch)
[Just list what data stores it connects to - don't trace flows:
- Databases
- Caches
- Message queues]

## External dependencies
[Services referenced but not found in this repo:
- Service names
- How they're referenced
- File paths where referenced]

## Key files
[Most important files for understanding this service:
- API definitions
- Deployment configs
- Main entry point
- Client configurations]

## Uncertainty / notes
[Anything unclear, ambiguous, or worth flagging]
```

---

## Phase 2: Synthesis

Once all repository Tasks complete, synthesize the findings:

1. **Read all per-repo reports**

2. **Categorize repos** - separate services from libraries, infra config, tools

3. **Reconcile naming** - different repos may refer to the same service by different names (hostname, env var name, internal name). Match them up.

4. **Build the unified picture as markdown:**
```
# Service Architecture Overview

## Service Inventory
[Table or list of all services:
- Service name
- Repo
- What it does
- Protocol(s) exposed]

## Libraries and Shared Code
[Non-service repos: shared libraries, utilities, SDKs]

## Communication Patterns
[How services talk to each other:
- Sync (HTTP, gRPC) vs async (queues, events)
- Service mesh or direct
- Internal vs external traffic]

## Service Dependencies
[For each service, what it depends on:
- Other internal services
- Platform services
- External services]

## Entry Points
[What's public-facing:
- API gateways
- Ingress points
- Public endpoints]

## Infrastructure Topology
[High-level deployment picture:
- What runs where
- Clusters, regions if visible
- Orchestration systems]

## Gaps and Missing Repos
[Specific list of:
- Services called but not found in any repo
- Unresolved service references
Include: what's missing, which repo references it, how it's referenced]

## Uncertainty / Needs Investigation
[Unresolved ambiguities, conflicting information, low-confidence findings]
```

5. **Generate a mermaid diagram** showing:
   - Services as nodes
   - Communication edges (labeled with protocol)
   - External services as distinct nodes
   - Entry points / ingress marked
   - Group by deployment unit if clear

## Repositories to Explore

[LIST GOES HERE]

## Execution

Spawn all repository Tasks in parallel - there is no dependency between them. Once all complete, proceed to synthesis.
