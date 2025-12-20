# Build & Deploy Architecture Mapper

You are tasked with mapping the build and deployment architecture across a set of repositories. Your goal is to understand how code becomes running software - the build systems, CI/CD pipelines, artifacts produced, and deployment mechanisms.

## Approach

You will use a parallel exploration strategy:

1. **Spawn one Task per repository** - these can run in parallel since they are independent
2. **Synthesize results** - once all Tasks complete, combine findings into a unified architecture view

## Phase 1: Per-Repository Exploration

For each repository in the provided list, spawn a Task with the following instructions:

---

### Task Instructions: Repository Explorer

Explore this repository to understand how it's built and deployed.

**Where to start looking:**
- CI configuration: .github/workflows/, Jenkinsfile, .gitlab-ci.yml, .circleci/, azure-pipelines.yml
- Build files: Makefile, BUILD, BUILD.bazel, build.gradle, pom.xml, package.json (scripts section), Cargo.toml, go.mod
- Container: Dockerfile, docker-compose.yml, .dockerignore
- Deployment: k8s manifests, helm charts, terraform, kustomize overlays, deploy scripts
- README (often has build/deploy instructions)
- Scripts directory: scripts/, tools/, bin/

**Scope guidance:**
Focus on the pipeline from source code to running artifact. Note what's built, how it's built, where artifacts go, and how deployment happens. Don't trace runtime behavior - that's service architecture.

**Output a markdown report with these sections:**
```
# Repo: [path]

## What gets built
[What artifact(s) does this repo produce?
- Docker image(s)
- Binary/executable
- Library/package
- Static assets
- Nothing (infra-only repo)]

## Build system
[How is it built?
- Build tool (make, bazel, gradle, npm, cargo, etc.)
- Key build commands
- Build configuration location
- Build dependencies (tools required)]

## CI/CD pipeline
[How is build/deploy automated?
- CI system (GitHub Actions, Jenkins, GitLab CI, etc.)
- Pipeline stages (lint, test, build, deploy)
- Triggers (push, PR, tag, manual)
- Branch strategy if visible]

## Artifacts and registries
[Where do built artifacts go?
- Container registry
- Artifact store (Artifactory, Nexus, S3)
- Package registry (npm, PyPI, Maven Central, internal)
- Artifact naming/tagging convention]

## Deployment mechanism
[How does code get deployed?
- Deployment tool (kubectl, helm, argocd, terraform, custom scripts)
- Environments (dev, staging, prod)
- Deployment triggers (automatic, manual, gitops)
- Rollback mechanism if visible]

## Environment configuration
[How are environment-specific settings handled?
- Config management approach
- Secrets management (vault, sealed secrets, env vars)
- Environment overlays (kustomize, helm values)
- Feature flags if present]

## Dependencies on other builds
[Does this build depend on artifacts from other repos?
- Base images
- Shared libraries
- Generated code
- Other artifacts]

## Key files
[Most important files for understanding build/deploy:
- CI config
- Main build file
- Dockerfile
- Deploy config]

## Uncertainty / notes
[Anything unclear, ambiguous, or worth flagging]
```

---

## Phase 2: Synthesis

Once all repository Tasks complete, synthesize the findings:

1. **Read all per-repo reports**

2. **Identify patterns** - common build tools, shared CI patterns, standard environments

3. **Map dependencies** - which builds depend on artifacts from other builds

4. **Build the unified picture as markdown:**
```
# Build & Deploy Architecture Overview

## Build Systems
[What build tools are used across repos:
- Primary build systems
- Standardized vs varied
- Common patterns]

## CI/CD Systems
[What CI/CD infrastructure exists:
- CI platforms in use
- Common pipeline patterns
- Shared CI components (reusable workflows, shared libraries)]

## Artifact Flow
[How artifacts move through the system:
- Registries and stores
- Naming conventions
- Retention policies if visible]

## Environments
[What environments exist:
- Environment names and purposes
- Promotion flow (dev → staging → prod)
- Environment-specific configuration patterns]

## Deployment Patterns
[How deployment happens:
- Deployment tools
- GitOps vs push-based
- Manual vs automatic
- Rollback approaches]

## Build Dependency Graph
[Which repos depend on artifacts from other repos:
- Base image dependencies
- Library dependencies
- Generated code dependencies]

## Common Patterns
[Standardized approaches across repos:
- Shared CI templates
- Common build configurations
- Standard deployment patterns]

## Inconsistencies
[Where repos deviate from common patterns:
- Different build tools
- Non-standard pipelines
- Unique deployment mechanisms]

## Gaps and Issues
[Problems identified:
- Missing CI/CD
- Broken or outdated pipelines
- Unclear deployment paths
- Missing artifact storage]

## Uncertainty / Needs Investigation
[Unresolved ambiguities, low-confidence findings]
```

5. **Generate a mermaid diagram** showing:
   - Repos as source nodes
   - Build stages
   - Artifact registries
   - Environments as deployment targets
   - Dependencies between builds

## Repositories to Explore

[LIST GOES HERE]

## Execution

Spawn all repository Tasks in parallel - there is no dependency between them. Once all complete, proceed to synthesis.
