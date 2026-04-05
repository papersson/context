# Observability Architecture Mapper

You are tasked with mapping the observability architecture across a set of repositories. Your goal is to understand how the system is monitored - logging, metrics, tracing, alerting, and dashboards.

## Approach

You will use a parallel exploration strategy:

1. **Spawn one Task per repository** - these can run in parallel since they are independent
2. **Synthesize results** - once all Tasks complete, combine findings into a unified architecture view

## Phase 1: Per-Repository Exploration

For each repository in the provided list, spawn a Task with the following instructions:

---

### Task Instructions: Repository Explorer

Explore this repository to understand how it's observed and monitored.

**Where to start looking:**
- Logging: logger initialization, logging config, log format definitions
- Metrics: prometheus client, statsd, micrometer, metrics endpoints
- Tracing: opentelemetry setup, jaeger/zipkin clients, trace context propagation
- Health: /health, /ready, /live endpoints, health check definitions
- Alerting: prometheus rules, alert definitions, datadog monitors, pagerduty config
- Dashboards: grafana JSON, dashboard definitions
- Runbooks: docs/runbooks/, oncall documentation
- Infrastructure: k8s probes, service mesh observability config

**Scope guidance:**
Focus on how this service is observed, not how it behaves. What signals does it emit? Where do they go? How would someone debug this service in production?

**Output a markdown report with these sections:**
```
# Repo: [path]

## Logging
[How this service logs:
- Logging library/framework
- Log format (JSON structured, plaintext, etc.)
- Log destination (stdout, file, aggregator)
- Log levels used
- Notable logging patterns (request IDs, correlation IDs)]

## Metrics
[How this service exposes metrics:
- Metrics library (prometheus client, statsd, micrometer, etc.)
- Metrics endpoint (/metrics, /prometheus, etc.)
- Collection method (pull vs push)
- Metrics destination (prometheus, datadog, cloudwatch, etc.)
- Key custom metrics defined
- Standard metrics (RED: rate/errors/duration, USE, etc.)]

## Tracing
[How this service participates in distributed tracing:
- Tracing library (opentelemetry, jaeger client, zipkin, datadog APM)
- Trace context propagation (headers used)
- Auto-instrumentation vs manual spans
- Trace destination (jaeger, zipkin, datadog, etc.)
- Sampling configuration if visible]

## Health and readiness
[How this service reports health:
- Health check endpoints
- What's checked (dependencies, resources)
- K8s liveness/readiness probes
- Load balancer health checks]

## Alerting
[Alert definitions in this repo:
- Alert rules (prometheus alerting rules, datadog monitors)
- Alert destinations (pagerduty, opsgenie, slack)
- Severity levels
- SLOs/SLIs defined]

## Dashboards
[Dashboard definitions in this repo:
- Grafana dashboards
- Datadog dashboards
- Other visualization definitions
- What's visualized]

## Runbooks and documentation
[Operational documentation:
- Runbooks location
- Incident response docs
- On-call procedures
- Known issues / troubleshooting guides]

## Observability gaps
[What's missing or unclear:
- No structured logging?
- No tracing?
- Missing health checks?
- No alerts defined?]

## Key files
[Most important files for understanding observability:
- Logging config
- Metrics setup
- Tracing initialization
- Alert definitions
- Dashboard JSON]

## Uncertainty / notes
[Anything unclear, ambiguous, or worth flagging]
```

---

## Phase 2: Synthesis

Once all repository Tasks complete, synthesize the findings:

1. **Read all per-repo reports**

2. **Identify the observability stack** - what tools/platforms are used across the system

3. **Assess coverage** - which services have good observability, which have gaps

4. **Build the unified picture as markdown:**
```
# Observability Architecture Overview

## Observability Stack
[What observability infrastructure exists:
- Log aggregation (ELK, Splunk, Datadog Logs, CloudWatch)
- Metrics platform (Prometheus, Datadog, CloudWatch Metrics)
- Tracing backend (Jaeger, Zipkin, Datadog APM, Tempo)
- Alerting system (Alertmanager, PagerDuty, Opsgenie)
- Dashboarding (Grafana, Datadog Dashboards)]

## Logging Patterns
[How logging works across services:
- Common logging libraries
- Log format standards (or lack thereof)
- Log aggregation flow
- Correlation ID propagation
- Inconsistencies]

## Metrics Patterns
[How metrics work across services:
- Standard metrics (RED, USE, etc.)
- Custom metrics patterns
- Collection infrastructure
- Naming conventions
- Inconsistencies]

## Tracing Patterns
[How distributed tracing works:
- Instrumentation approach
- Trace propagation headers
- Coverage (which services participate)
- Sampling strategies
- Gaps in trace continuity]

## Alerting Coverage
[What alerting exists:
- Services with alerts defined
- Alert severity patterns
- On-call routing
- SLO/SLI definitions
- Services without alerting]

## Dashboards Inventory
[What dashboards exist:
- Service-level dashboards
- System-wide dashboards
- Dashboard ownership
- Dashboard gaps]

## Operational Readiness
[How ready is each service for production incidents:
- Runbook coverage
- Health check completeness
- Debug tooling
- Incident response docs]

## Observability Maturity by Service
[Rate each service:
| Service | Logging | Metrics | Tracing | Alerts | Runbooks | Overall |
| ------- | ------- | ------- | ------- | ------ | -------- | ------- |
| ...     | ✓/△/✗  | ✓/△/✗   | ✓/△/✗   | ✓/△/✗  | ✓/△/✗    | High/Med/Low |]

## Standards vs Reality
[Where do services follow common patterns vs deviate:
- Standard approaches
- Notable deviations
- Legacy patterns]

## Gaps and Recommendations
[Critical observability gaps:
- Services with no tracing
- Services with no alerting
- Missing runbooks
- Inconsistent logging making correlation hard
- Blind spots in the system]

## Uncertainty / Needs Investigation
[Unresolved ambiguities, low-confidence findings]
```

5. **Generate a mermaid diagram** showing:
   - Services as nodes
   - Observability backends as nodes
   - Edges showing what signals flow where (logs, metrics, traces)
   - Color/style indicating maturity level

## Repositories to Explore

[LIST GOES HERE]

## Execution

Spawn all repository Tasks in parallel - there is no dependency between them. Once all complete, proceed to synthesis.
