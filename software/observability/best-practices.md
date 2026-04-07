# Observability Best Practices

A practitioner's guide to instrumenting, monitoring, and debugging production systems. Each practice states what to do, why, and what tradeoff it makes. Companion to [progression.md](progression.md), which covers the history and conceptual foundations.

---

## 1. Instrumentation

### Start with auto-instrumentation, layer manual instrumentation for business context

Auto-instrumentation (via OpenTelemetry agents or SDK hooks) gives you broad coverage of HTTP calls, database queries, and messaging operations with zero code changes. Manual instrumentation then fills the gaps that matter: business-critical paths like payment processing, checkout flows, or domain logic invisible to generic instrumentation.

A span saying `POST /api/v1/orders 500` is useful. A span that also carries `order.total_amount=459.99`, `customer.tier=enterprise`, and `payment.provider=stripe` is actionable.

**Common mistake:** Instrumenting everything manually from day one. Teams burn weeks adding spans to every function, creating noise that drowns signal. The opposite mistake: relying solely on auto-instrumentation and never adding business context, leaving you blind to domain-specific failures.

**Tradeoff:** Auto-instrumentation is low-effort but noisy — it captures every HTTP call including health checks and internal service mesh traffic. Manual instrumentation is precise but requires maintenance as code evolves.

### Follow OTel semantic conventions rigorously

OpenTelemetry semantic conventions define standard attribute names across signal types. Span naming rules are specific per protocol:

| Protocol | Span name format | Example |
|----------|-----------------|---------|
| HTTP server | `{method} {route_template}` | `GET /users/:id` |
| HTTP client | `{method}` | `GET` |
| RPC | `{package}.{service}/{method}` | `grpc.health.v1.Health/Check` |
| Database | `{operation} {collection}` | `SELECT users` |
| Messaging | `{destination} {operation}` | `orders publish` |

Never use the raw URI path as a span name. `GET /users/a8f2e-4b1a` creates unbounded cardinality. Use the route template: `GET /users/:id`.

**Common mistake:** Inventing your own attribute names (`latency_ms`, `statusCode`, `dbQuery`) instead of using conventions. This fragments your ability to correlate signals across services and makes vendor-provided dashboards useless.

### Propagate context explicitly across async boundaries

OpenTelemetry stores the current span context in thread-local storage (or language equivalent). When execution moves to a different thread, goroutine, async callback, or message queue consumer, that storage is empty. Context does not follow automatically.

| Boundary | Problem | Solution |
|----------|---------|----------|
| Java thread pools | `ExecutorService` submissions lose context | `Context.taskWrapping(rawExecutor)` |
| Python `run_in_executor()` | Thread pool breaks `contextvars` chain | Capture context, `context.attach(ctx)` in worker |
| Node.js `setTimeout` | AsyncLocalStorage may not be initialized | `context.bind(activeContext, callback)` |
| Go goroutines | `context.Context` must be passed explicitly | Always pass `ctx` as first parameter |
| Message queues | Producer and consumer are different processes | Inject context into message headers on produce, extract on consume |
| Connection pools | Cached context references go stale | Never store context on long-lived objects |
| Timer/cron callbacks | No parent request context exists | Create a new root span; don't link to a non-existent parent |

**Common mistake:** Assuming that because "it works in development" (single-threaded, synchronous), context propagation is fine. It breaks silently — you get disconnected trace fragments instead of end-to-end traces, and nobody notices until an incident requires cross-service debugging.

### Use baggage deliberately and defensively

OTel Baggage propagates key-value pairs across service boundaries via HTTP headers. Useful for tenant IDs, feature flags, and debug flags.

**Security concern:** Baggage is sent in plain HTTP headers, visible to any network observer or downstream service — including third-party APIs. Never put credentials, PII, or sensitive business data in baggage. Validate incoming baggage with allowlists. Strip baggage before calling external services.

**Tradeoff:** Baggage adds header size to every cross-service call. Keep values small (IDs, not objects). Copy important baggage values to span attributes explicitly if you want them searchable.

---

## 2. Structured Logging

### Require a human-readable message on every event, then add attributes

The format you store logs in and the format you read them in are separate concerns. Structure and readability aren't in tension if you emit both. A log record should have:

- A **message** (the human-readable summary you scan in a list view)
- **Structured attributes** (the machine-queryable fields you filter and aggregate on)

This is how OpenTelemetry's log data model works: a `Body` (message) plus `Attributes` (structured fields). It's also how Sentry's error events work — a human-readable exception message plus queryable, groupable structured context.

Most implementations that feel bad to use adopted structured logging by cargo cult: JSON everything, no message conventions, no schema discipline. You end up with data that's simultaneously unreadable to humans (it's JSON) and unqueryable by machines (the fields are inconsistent). The fix isn't to abandon structure — it's to do both layers well.

**Common mistake:** Dumping JSON blobs without a uniform schema or readable message field. A wall of `{"timestamp":"2024-...` in a log viewer is a tooling failure and a discipline failure, not evidence that structured logging is wrong.

### Consistency is structure — make it explicit

"You don't need structure, you need consistency" is a common argument. But consistency *is* structure — it's just implicit structure maintained by developer convention rather than explicit structure enforced by a schema.

The question is where you pay the cost of imposing that structure:

| Approach | Cost paid by | Failure mode |
|----------|-------------|--------------|
| **Write-time structure** (structured logging) | Developer, once, at log emission | Inconsistent field names across services |
| **Read-time extraction** (Splunk-style) | Pipeline maintainer + system at ingest | Parsing config breaks when log format changes |
| **Query-time search** (full-text grep) | Everyone, every query, forever | Ambiguous results, can't aggregate |

Write-time structure is the most expensive to adopt and the cheapest to operate. Tools that make unstructured logs queryable (Splunk's field extraction, ML-based log clustering) are compensating for the absence of structure, not evidence that structure is unnecessary.

### Emit canonical log lines

Pioneered by Stripe, a canonical log line is a single, information-dense log entry emitted at the end of each request that consolidates all key data for that request:

```
canonical-log-line
  http_method=POST http_path=/v1/charges http_status=200
  duration=0.034 request_id=req_abc123
  user_id=usr_xyz customer_tier=enterprise
  database_queries=12 db_duration=0.021
  cache_hit=false rate_remaining=98
```

During an incident, this lets you query directly: "which requests are failing, for which customers, and why?" without stitching together dozens of fragmented log entries per request.

**Tradeoff:** Canonical log lines require middleware cooperation to accumulate fields throughout request processing. They're emitted last — if the process crashes mid-request, you get nothing. Pair with a request-start log for crash detection. The canonical line doesn't replace detailed debug logging — it sits above it as the authoritative record.

### Include these fields in every structured log line

```json
{
  "timestamp": "2025-11-15T14:23:01.456Z",
  "level": "error",
  "service": "payment-service",
  "version": "2.3.1",
  "environment": "production",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "message": "Payment processing failed",
  "error.type": "PaymentDeclinedException",
  "error.message": "Card declined: insufficient funds"
}
```

**Mandatory:** timestamp (ISO 8601, UTC), level, service name, trace_id, span_id, message. **Strongly recommended:** service version, environment, request_id.

The `trace_id` and `span_id` fields are what enable clicking from a trace span directly to its corresponding log entries, and vice versa. Without them, correlation is manual timestamp-matching.

**Common mistake:** Using local timestamps instead of UTC. Using inconsistent field names across services (`requestId` vs `request_id` vs `reqId`). Including stack traces as unstructured strings inside the message field instead of as structured `error.*` fields.

### Use log levels with discipline

| Level | When to use | Production default | Alert? |
|-------|------------|-------------------|--------|
| **ERROR** | Operation failed, cannot complete. Something is broken. | Always on | Page or ticket |
| **WARN** | Unexpected but recoverable. Degraded condition. | Always on | Slack/ticket |
| **INFO** | Significant business events. Canonical log lines. State transitions. | On (carefully curated) | No |
| **DEBUG** | Diagnostic detail for troubleshooting. Internal state. | Off in production | No |

Every INFO log should be something an operator would want to see during normal operations. If nobody ever reads it, it shouldn't be INFO. Heuristic: if your service handles 1000 req/s, can an operator meaningfully scan the INFO output? If not, you're logging too much at INFO.

**Common mistake:** Using ERROR for client errors. A 404 is not an error in your service — it's the correct response to a bad client request. Reserve ERROR for things where *your service* failed to do its job.

### Handle PII at the pipeline level

PII in logs is a GDPR/CCPA liability. If PII reaches the database, it's already a compliance problem even if never displayed. Use OpenTelemetry Collector processors or log pipeline tools (Fluentd, Vector) to redact before data reaches storage.

Strategies: mask (show partial: `****1234`), redact (replace: `[REDACTED]`), hash (deterministic pseudonym: `sha256(email)` for correlation without exposure).

**Tradeoff:** Pipeline-level redaction can miss novel PII patterns. Application-level sanitization is more precise but harder to enforce. The mature approach is defense in depth: application-level best practices plus pipeline-level safety net.

---

## 3. Metrics

### RED for services, USE for infrastructure

**RED Method** (Tom Wilkie) — measures user-facing service health:
- **Rate**: Requests per second
- **Errors**: Failed requests per second (or error ratio)
- **Duration**: Latency distribution (histogram, not averages)

**USE Method** (Brendan Gregg) — measures infrastructure resource health:
- **Utilization**: Average time resource was busy (percentage)
- **Saturation**: Degree to which work is queued (queue length)
- **Errors**: Error event count

**Alert on RED (symptoms). Investigate with USE (causes).** If your API is slow (RED tells you), USE tells you whether it's because the database server's CPU is saturated, the disk is full, or the network is dropping packets.

**Common mistake:** Alerting on USE metrics (`CPU > 80%`) instead of RED metrics (`p99 > 500ms`). High CPU is a cause, not a symptom. Users don't care if CPU is high; they care if the site is slow. A CPU alert at 3 AM with no user impact is noise.

**Tradeoff:** USE requires knowing your resource topology. In serverless/managed environments, many USE metrics are unavailable. RED works regardless of infrastructure model.

### Guard cardinality as a first-class concern

Every unique combination of label values creates a separate time series. A metric with labels `{method, endpoint, status_code, customer_id}` where `customer_id` has 100,000 values creates an explosion that will crash Prometheus or blow your vendor bill.

**Rules:**
- Never use unbounded values as label values (user IDs, email addresses, request IDs, full URLs)
- Replace specific paths with route templates: `/users/a8f2e` becomes `/users/:id`
- If `sum()` over all values of a label isn't meaningful, it probably shouldn't be a label
- Set `sample_limit` in scrape configs as a safety net
- Monitor cardinality: query `count({__name__=~".+"}) by (__name__)` to find runaway metrics

**Common mistake:** Adding a `version` label with a git SHA that changes every deploy. Or a `pod_name` label in Kubernetes where pod names are ephemeral. These create phantom time series that accumulate until retention expires.

**Tradeoff:** Low cardinality means less granularity. You can't break down metrics by individual customer without a customer label. The answer: use high-cardinality data in traces and logs (where it's searched, not indexed as time series), and keep metrics low-cardinality.

### Follow Prometheus naming conventions

```
# Counters (always _total suffix)
http_requests_total{method="GET", status="200"}

# Histograms (auto-generates _bucket, _sum, _count)
http_request_duration_seconds_bucket{le="0.1"}

# Gauges (no suffix convention)
node_memory_available_bytes
process_open_fds
```

**Base units always:** seconds (not milliseconds), bytes (not megabytes), ratios 0-1 (not percentages 0-100). This prevents unit confusion across teams and enables consistent PromQL.

**Never put label names in metric names:** `http_requests_by_method_total` is wrong. Use `http_requests_total{method="GET"}`.

### Design histogram buckets around your SLOs

Default Prometheus buckets (`.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10`) are rarely appropriate. Design buckets around your actual latency targets.

If your SLO is p99 < 300ms, concentrate buckets around the SLO boundary:

```
buckets: [0.01, 0.025, 0.05, 0.1, 0.15, 0.2, 0.25, 0.3, 0.5, 1.0, 2.5, 5.0]
```

Rules of thumb:
- Exponential spacing for distributions spanning multiple orders of magnitude
- The largest bucket should cover 2-3x your maximum expected latency
- For high-cardinality histograms (per-endpoint), fewer buckets (8-10)
- For low-cardinality histograms (service-level), more buckets (15-20)
- Use Prometheus native histograms (v2.40+) when available — they use exponential bucketing automatically

**When to use summaries instead of histograms:** Only when you need accurate quantiles from a single instance and cannot aggregate across instances. In practice, this is rare. Histograms can be aggregated across instances via `histogram_quantile()`; pre-computed summary quantiles cannot be meaningfully aggregated.

### Track totals and failures, not successes and failures separately

Use `api_requests_total` + `api_failures_total` instead of `api_successes_total` + `api_failures_total`.

Error rate: `rate(api_failures_total[5m]) / rate(api_requests_total[5m])`.

The alternative requires `rate(failures) / (rate(successes) + rate(failures))` — more complex, and if either counter resets independently, the calculation breaks.

---

## 4. Distributed Tracing

### Trace service boundaries and meaningful operations, not every function call

**Trace:**
- Inbound/outbound HTTP/gRPC requests
- Database queries
- Message queue publish/consume
- Cache operations
- Business-critical code paths (checkout, payment, authentication)

**Don't trace:**
- Every function call within a service (span explosion)
- Pure computation (CPU-bound sorting, serialization)
- Health check endpoints
- Internal helper methods

**Tradeoff:** Under-instrumenting means gaps where you can't see what happened. Rule of thumb: add a span when execution crosses a meaningful boundary (network, process, significant business logic) or when you need to measure duration of a specific operation.

### Add attributes to existing spans before creating child spans

When you want to add business context, first consider enriching the current span:

```python
# Prefer: adding attributes to the current span
span = trace.get_current_span()
span.set_attribute("order.total", 459.99)
span.set_attribute("customer.tier", "enterprise")

# Avoid: creating a trivial span just to carry attributes
with tracer.start_as_current_span("set-order-context") as span:
    span.set_attribute("order.total", 459.99)
    # no actual work happens in this span
```

New spans should represent actual units of work with meaningful duration, not attribute carriers.

### Use tail-based sampling in the OTel Collector

Head-based sampling (decide at trace start) is simple but blind to outcomes. Tail-based sampling (decide after trace completes) lets you keep interesting traces and drop routine ones:

```yaml
processors:
  tail_sampling:
    decision_wait: 30s
    num_traces: 100000
    policies:
      # Always keep errors
      - name: errors-always
        type: status_code
        status_code:
          status_codes: [ERROR]
      # Always keep slow traces
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 2000
      # Sample normal traffic at 10%
      - name: baseline-sample
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

**Critical:** Specific policies (errors, latency) must come before the probabilistic catch-all — evaluation is first-match-wins.

**Critical:** Generate span-based metrics *before* tail sampling, so your metrics reflect 100% of traffic even though you only store 10% of traces.

**Common mistake:** Setting `decision_wait` too low (in-flight traces get sampled incomplete) or `num_traces` too low (traces dropped before sampling decision).

**Tradeoff:** Tail sampling requires a centralized collector that sees all spans for a trace, adds memory overhead (buffering spans during `decision_wait`), and delays trace availability. For most organizations, keeping 100% of interesting traces while discarding 90% of routine traffic is worth it.

### Correlate across all three signals

- **Trace-to-log**: Include `trace_id` and `span_id` in every log line. Your platform links from a trace span to corresponding log entries.
- **Trace-to-metric**: Use exemplars to attach trace IDs to metric data points:
  ```
  http_request_duration_seconds_bucket{le="0.5"} 4892 # {trace_id="abc123"} 0.48
  ```
  This lets you click from a latency spike on a dashboard to the specific trace that caused it.
- **Log-to-trace**: Given a `trace_id` from a log line, jump to the full distributed trace view.

This three-way correlation is the difference between "something is broken" and "here is the exact request that broke, across all services it touched."

---

## 5. Alerting and SLOs

### Define SLIs on user-facing symptoms, not internal metrics

**Good SLIs:**
- Availability: proportion of successful requests (`status < 500 / total`)
- Latency: proportion of requests faster than threshold (`p99 < 300ms`)
- Correctness: proportion of requests returning correct results
- Freshness: proportion of data updated within threshold

**Not SLIs:** CPU utilization, memory usage, queue depth, thread count. These are useful for debugging but don't tell you whether users are happy.

### Use multi-window, multi-burn-rate alerting

From the Google SRE Workbook — the gold standard for SLO-based alerting. Each alert requires both a long window (detect sustained issues) and a short window (confirm the issue is active now). The short window should be 1/12 the long window.

For a 99.9% SLO:

| Severity | Long window | Short window | Burn rate | Budget consumed | Action |
|----------|------------|--------------|-----------|----------------|--------|
| Page (urgent) | 1 hour | 5 minutes | 14.4x | 2% | Wake someone up |
| Page (moderate) | 6 hours | 30 minutes | 6x | 5% | Interrupt current work |
| Ticket | 3 days | 6 hours | 1x | 10% | Fix this week |

```promql
# Page-level alert: fires when BOTH conditions are true
(
  job:slo_errors_per_request:ratio_rate1h > (14.4 * 0.001)
  and
  job:slo_errors_per_request:ratio_rate5m > (14.4 * 0.001)
)
```

**Why this matters:** Simple error-rate alerting fires on transient spikes. Single-window alerting keeps firing long after resolution. Multi-window multi-burn-rate balances precision, recall, detection time, and reset time.

**Tradeoff:** More configuration complexity and more recording rules to maintain. Low-traffic services may need different approaches because burn rates become noisy with small request counts.

### Formalize error budget policies

An error budget policy formalizes what happens when the budget is exhausted:

- **Budget exhausted:** Halt all non-critical changes and releases until the service is back within SLO
- **Single incident > 20% of quarterly budget:** Mandatory postmortem with action items
- **Pattern threshold > 20% in same outage class:** Must address in the following quarter's planning
- **Disagreements:** Escalate to VP Engineering

The error budget policy is what makes SLOs operational. Without it, the SLO is a number on a dashboard that nobody acts on. With it, teams have permission — and obligation — to prioritize reliability when the data demands it.

### Every alert links to a runbook

If you can't write a runbook for an alert, that alert shouldn't exist. Each alert should state:

1. The symptom and business impact
2. A link to the relevant dashboard
3. A link to the runbook with diagnostic and remediation steps
4. Who owns the service

The average on-call engineer receives ~50 alerts/week, but only 2-5% require human intervention. The rest are noise that erodes trust in the alerting system.

---

## 6. Dashboards

### Maintain a three-tier hierarchy

**Tier 1 — Fleet Overview.** One dashboard showing the health of all services. RED metrics for each service (request rate, error rate, p50/p99 latency). One row per service, ordered by data flow. This is the "is anything on fire?" dashboard.

**Tier 2 — Service Detail.** One dashboard per service showing deeper RED metrics broken down by endpoint, plus USE metrics for the service's infrastructure dependencies. This is the "what part of this service is broken?" dashboard.

**Tier 3 — Debug/Investigation.** Ad-hoc dashboards for specific subsystems, database performance, cache hit rates, queue depths. These support active investigation and may be temporary.

During an incident, you start at Tier 1 (which service?), drill to Tier 2 (which endpoint?), then to Tier 3 (what infrastructure component?). Without this hierarchy, you're clicking through 50 dashboards hoping to find the right one.

### Fight dashboard sprawl actively

Every dashboard costs attention. Practices:

- Maintain a curated list of approved dashboards per service
- Review quarterly; archive dashboards nobody viewed in the last 30 days
- Use template variables instead of duplicating dashboards per environment/region
- Use dashboard-as-code (Jsonnet, Grafonnet, Terraform) to prevent drift
- Kill one-off incident dashboards after the postmortem

---

## 7. Cost Management

### Understand where cost concentrates

Over 50% of observability spend typically goes to logs alone. The cost multiplier effect: organizations pay to store the same telemetry data multiple times across metrics, logs, and traces — expenses can grow 3-10x faster than traffic.

### Apply a four-layer reduction model

1. **Data creation.** Don't generate data you'll never use. Remove health-check and readiness-probe logging. Set production log level to INFO minimum. Curate what gets instrumented.

2. **Data movement.** Pre-ingestion filtering and aggregation at the edge. Use OTel Collector processors to drop, filter, and aggregate before data reaches your backend. Route low-value data to cheap storage directly.

3. **Data storage.** Tiered retention:
   - Hot (7-14 days): full-resolution data in your primary observability platform
   - Warm (30-90 days): aggregated data or downsampled metrics
   - Cold (archive): S3/GCS for long-term compliance retention at minimal cost

   This alone can save 30-50%.

4. **Data usage.** Recording rules for frequently-used aggregations. Don't run expensive full-scan queries in dashboards that refresh every 30 seconds.

### Sample aggressively, but never blindly

- **Traces:** Tail-sample at 10% baseline, 100% errors and slow requests. Can reduce trace storage 80-90%.
- **Logs:** ERROR 100%, WARN 100%, INFO 10% (hash-based for session consistency), DEBUG 0% in production.
- **Metrics:** Adjust scrape intervals by criticality. Critical services: 15s. Less critical: 60s (75% cost reduction).

**Always generate metrics from 100% of data *before* sampling raw events.** Your aggregate numbers should reflect all traffic; your detailed investigation data can be sampled.

### Consider the wide-event alternative

The Honeycomb argument: instead of paying for separate metrics, logs, and traces systems, emit one wide structured event per request containing all relevant context. Derive metrics, trace views, and log-style queries from this single data source.

**Cost implication:** Store data once instead of three times. One pipeline, one query language, one retention policy.

**Tradeoff:** Requires a backend that supports high-cardinality, columnar, schema-on-read queries. Traditional tools (Prometheus + ELK + Jaeger) don't support this model. You're either buying a platform built for this or building something on ClickHouse/Druid.

---

## 8. Organizational Practices

### Run observability as a platform team with self-service guardrails

**Centralize:** Tooling selection, pipeline management, naming standards, cost governance. The platform team provides golden paths — starter templates for common service types, standard OTel Collector configs, curated dashboard templates.

**Delegate:** Dashboard creation, alert configuration, SLO definition, custom instrumentation within standards. Teams own their own observability within the guardrails.

**Common mistake:** Either fully centralized (bottleneck: every team waits for the platform team) or fully decentralized (chaos: every team uses different naming, different tools, inconsistent quality).

### Include observability gaps in every post-incident review

Every postmortem should answer:

- Did we detect the incident from alerts, or did a customer report it?
- Could we see the incident across all affected services in a single trace?
- Did our dashboards show the right signals, or did we need ad-hoc queries?
- Were the logs we needed available, or had they been sampled/rotated away?
- What would have reduced time-to-detection or time-to-resolution?

Then track action items to close the gaps.

### Treat instrumentation as code review criteria

Instrumentation should be reviewed in PRs alongside business logic:

- Does the new endpoint have appropriate spans and attributes?
- Are new error paths logged at the right level with context?
- Do new metrics follow naming conventions and avoid cardinality risks?
- Is PII handled appropriately?

### Observability readiness as a launch gate

Minimum bar before a service goes to production:

- Auto-instrumentation enabled
- Canonical log lines emitting with mandatory fields
- SLO defined with error budget policy
- Service appears on the fleet overview dashboard
- At least one SLO-based alert configured with a runbook

---

## 9. Anti-Patterns

### Logging everything, querying nothing

Teams enable DEBUG in production "just in case," generating terabytes/day. Over 90% is never queried. The cost is enormous and finding useful data requires wading through noise. **Fix:** Curate what you log. Use canonical log lines for the per-request record. Reserve DEBUG for targeted, temporary investigation.

### Alerting on causes, not symptoms

`CPU > 80%` fires at 3 AM. The engineer investigates, finds no user impact, goes back to sleep. Repeat three times a week until everyone ignores CPU alerts — and the one time high CPU actually causes latency, nobody notices. **Fix:** Alert on SLO violations (symptoms). Use USE metrics for investigation after the symptom alert fires.

### Ignoring cardinality until it explodes

A developer adds `user_email` as a Prometheus label. Works in staging with 100 users. In production with 2 million users, Prometheus OOMs and monitoring goes dark during an incident. **Fix:** Code review for metrics. Set `sample_limit` in scrape configs. Monitor `prometheus_tsdb_head_series`.

### Three pillars as three silos

Metrics in Prometheus, logs in Elasticsearch, traces in Jaeger. Three tools, three query languages, zero correlation. An engineer sees a latency spike, can't jump to the corresponding trace, can't find relevant logs without manually copying timestamps. **Fix:** Ensure trace-to-log and trace-to-metric correlation via shared `trace_id`. Use exemplars. Choose tooling that supports cross-signal navigation.

### Dashboard-driven development

Building 30 dashboards before launch, covering every conceivable metric. After a month, 25 are never viewed. The 5 that matter are lost in the sprawl. **Fix:** Launch with one RED dashboard. Add panels when actual incidents reveal gaps. Archive dashboards not viewed in 30 days.

### Not testing observability in staging

Traces that work in development break in production because staging doesn't exercise the same async paths or message queues. Alerts that work in staging fire constantly in production because traffic patterns differ. **Fix:** Staging should mirror production's OTel Collector pipeline, sampling config, and alert rules. Run synthetic traffic that exercises production-like paths.

---

## Sources

- Google SRE Workbook — [Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/), [Error Budget Policy](https://sre.google/workbook/error-budget-policy/)
- OpenTelemetry — [Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/), [Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/), [Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- Stripe — [Canonical Log Lines](https://stripe.com/blog/canonical-log-lines)
- Brendan Gregg — [The USE Method](https://www.brendangregg.com/usemethod.html)
- Charity Majors — [Observability 2.0](https://charity.wtf/tag/observability-2-0/)
- Cindy Sridharan — [Distributed Systems Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/) (O'Reilly)
- Prometheus — [Naming](https://prometheus.io/docs/practices/naming/), [Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
