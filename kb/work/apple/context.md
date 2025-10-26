# Apple Context Pack (Paste into LLM)

## Role & Scope (one-liner)
I’m a consultant embedded with Apple via AFRY, focused on backend/infra + performance.

## Current Goals (edit per week)
- [ ] Onboard to repo(s) X
- [ ] Deliver feature Y (acceptance criteria below)
- [ ] Reduce latency in endpoint Z from A ms → B ms

## Definition of Done (DoD)
- Tests updated/added and passing
- Performance budget respected (state target)
- No secrets in code; configs via env/secret store
- PR includes rationale, tradeoffs, and rollback plan

## Tech/Env (adjust as you learn)
- Language: Scala (JVM), build via sbt
- Observability: logs, metrics, traces (name tools)
- Storage/queues: (redact names if needed)
- CI: (name), Branch: trunk-based or GitFlow (pick one)

## Operating Constraints
- Confidentiality/IP: No client secrets, source, or proprietary payloads in prompts.
- Allowed LLM content: generic algorithms, scaffolding, refactoring plans, performance strategies, unit test scaffolds, diffs without sensitive data.
- Disallowed: credentials, private URLs, internal token names, customer data, unique identifiers.
- When unsure: return a redacted placeholder and request a safe interface/fixture.

## Review Checklist (copy into PR)
- [ ] Problem & constraints stated
- [ ] Minimal viable change; diff is focused
- [ ] Tests + benchmarks present
- [ ] Failure modes / rollback noted
- [ ] Docs/changelog updated

## Acceptance Criteria Template
Feature: <name>
- Behavior: <what/when>
- Inputs/Outputs: <io>
- Perf: p95 <X ms> @ <N rps>, <M> CPU %
- Telemetry: <metrics/logs/traces to emit>
- Out of scope: <list>

## Links (keep generic; paste real ones when prompting privately)
- Repo(s): <redacted>
- Runbook(s): <redacted>
- Dashboards: <redacted>
