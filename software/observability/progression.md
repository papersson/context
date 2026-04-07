# Observability in Software Systems: A History Built from Problems

## Stage 0: One Program, One Screen

You write a program. It does something wrong. You add:

```c
printf("got here, x=%d\n", x);
```

This is the ur-form of all observability: make the invisible visible. The program's internal state is opaque by default — you can only observe its external behavior (output, exit code). Print statements punch holes in that opacity.

This works because you have a single program, running on your machine, right now. You can read the output as it scrolls by. You can add a print, recompile, rerun. The feedback loop is tight.

**What fails:** You deploy the program to a server. Nobody is watching stdout. The process runs as a daemon. Output goes... where?

---

## Stage 1: Log Files

The first inflection point is trivially obvious: redirect output to a file.

```
./myserver >> /var/log/myserver.log 2>&1
```

Now you have persistence. When something goes wrong at 3 AM, you can read the log at 9 AM. The Unix tradition standardized this quickly — programs write to stderr for diagnostics, and the system provides a facility for collecting these messages.

That facility, on Unix, was **syslog** (RFC 3164, dating to the early 1980s). The idea: instead of each program managing its own log file, programs send messages to a system daemon (`syslogd`) over a local socket. The daemon handles writing to files, rotating them, and optionally forwarding them elsewhere. Messages get a severity level and a facility tag:

```
<34>Oct 11 22:14:15 myhost myserver[12345]: Connection accepted from 10.0.0.5
```

That `<34>` encodes facility=4 (auth) and severity=2 (critical) as `4*8 + 2`. It's a 1-byte priority field designed when bytes were expensive.

This gave you something new: a single place to look. Instead of tailing ten different files, you could check `/var/log/messages`. System administrators — the ancestors of today's SREs — could grep through these files when a machine misbehaved.

**The tradeoff syslog made:** Uniformity over richness. Every message is a flat string with a timestamp and severity. The content is whatever the programmer decided to print. There's no schema, no structure, no way to query "show me all messages where response_time > 500ms" without writing a regex.

**What fails:** You now have five servers. Something is wrong, but you don't know which server the user's request hit. You SSH into each machine, grep through logs, try to correlate timestamps. This doesn't scale.

---

## Stage 2: Centralized Log Collection

The problem is clear: logs are stranded on individual machines. The solution is equally clear: send them somewhere central.

Syslog actually anticipated this — it supports forwarding messages over UDP to a remote syslog server. But UDP is lossy, the protocol has no authentication, and you're now sending all your log data over the network. Early solutions were fragile.

The pattern that emerged: run an agent on each machine that tails log files and ships them to a central store. The store needs to handle ingestion at scale and support search. This is what systems like **Elasticsearch** (a full-text search engine built on Apache Lucene) became popular for. The typical stack — sometimes called "ELK" — was:

- **Logstash** or **Fluentd**: agents that collect, parse, and forward logs
- **Elasticsearch**: indexes and stores them
- **Kibana**: a web UI for searching and visualizing

Now an SRE during an incident can open a browser, type a query, and see logs from all machines in one view. A developer debugging their service can filter to just their service's logs. A security team investigating a breach can search for a specific IP across every service.

But centralization introduced new problems:

**Cost.** You're now storing every log line from every service, probably multiple copies (for redundancy in the search index). A medium-sized deployment might generate tens of gigabytes of logs per day. A large one, terabytes. Storage and indexing are not free.

**Noise.** When everything is in one place, finding the signal becomes harder, not easier. Grep worked on one machine because the haystack was small. Now you have a much bigger haystack, and the needles look the same as the hay — unstructured text strings.

**Parsing fragility.** Every service logs in a different format:

```
# Service A
2024-03-15 10:23:45 INFO  [req-abc123] User login successful for user_id=789

# Service B
[2024-03-15T10:23:45.123Z] INFO: auth - login ok uid:789 ip:10.0.0.5

# Service C
Mar 15 10:23:45 auth-svc: LOGIN user=789 result=ok duration=45ms
```

To make these searchable, you need to parse them. That means writing regex patterns or grok expressions for every format. When a developer changes their log format, the parser breaks silently. You get partially parsed data — or worse, misparsed data.

**What fails:** The fundamental problem is that logs are strings written by humans for humans. There is no contract between the producer and the consumer. This is where the story forks in two directions simultaneously.

---

## Stage 3a: Structured Logging

The parsing problem has an elegant solution: don't parse. Instead, emit logs that are already machine-readable.

Instead of:

```
2024-03-15 10:23:45 INFO User login successful for user_id=789 from 10.0.0.5 in 45ms
```

Emit:

```json
{"timestamp":"2024-03-15T10:23:45.123Z","level":"info","event":"user_login","user_id":789,"ip":"10.0.0.5","duration_ms":45,"service":"auth","host":"prod-auth-03"}
```

This is **structured logging**: each log entry is a set of typed key-value pairs rather than a formatted string. The code looks different too:

```python
# Unstructured
logger.info(f"User login successful for user_id={user_id} from {ip} in {duration}ms")

# Structured
logger.info("user_login", user_id=user_id, ip=ip, duration_ms=duration)
```

The logging library handles serialization. The transport layer doesn't need to parse. The storage layer can index individual fields. Now you can query: "show me all logins where `duration_ms > 500` and `service = auth`" without regex.

**The tradeoff structured logging makes:** Developer ergonomics for queryability. Structured logs are harder to read raw — `tail -f` on a stream of JSON is less pleasant than reading formatted text. You now need tooling to make them human-readable during development. You also need discipline: if a developer logs `user_id` as `uid` in one place and `userId` in another, you're back to parsing problems, just at a different layer. This is a **schema problem** — and there's no universal solution for it, though conventions like semantic conventions in OpenTelemetry try to standardize common field names.

But structured logging alone doesn't solve a deeper problem that was emerging simultaneously.

---

## Stage 3b: Metrics — A Different Data Model Entirely

While the logging world was struggling with parsing and cost, a parallel realization was forming: **most of the time, you don't need the individual events. You need aggregates.**

Consider this question: "Is my service healthy right now?" To answer it from logs, you'd have to:

1. Query all log entries from the last minute
2. Count the ones that indicate errors
3. Count the total
4. Compute the ratio
5. Compare it to some threshold

And you'd need to do this continuously — not just when someone asks, but every minute, so you can tell when something changes. Running that log query every minute across terabytes of data is expensive.

The insight: if what you actually want is "requests per second" and "error rate" and "p99 latency", compute those numbers as events happen and store just the numbers. This is **metrics** — numerical measurements collected over time.

At the systems level, a metric is:

```
http_requests_total{method="GET", path="/api/users", status="200"} 14532
http_requests_total{method="GET", path="/api/users", status="500"} 23
http_request_duration_seconds{method="GET", path="/api/users", quantile="0.99"} 0.45
```

Each metric has a name, a set of labels (key-value pairs that identify what's being measured), and a numeric value. There are a few fundamental types:

- **Counter**: a value that only goes up (total requests, total errors, total bytes sent). You derive rates from it: "requests per second" = change in counter / time.
- **Gauge**: a value that goes up and down (current memory usage, number of active connections, queue depth).
- **Histogram**: a distribution of values (request latencies). You pre-define buckets (e.g., <10ms, <50ms, <100ms, <500ms, <1s) and count how many observations fall into each. This lets you compute percentiles without storing every individual value.

The system that popularized this model was **Prometheus**, created at SoundCloud around 2012, inspired by Google's internal Borgmon. Its architecture was distinctive:

- Each service exposes an HTTP endpoint (`/metrics`) that returns its current metric values in a text format
- A central Prometheus server **pulls** (scrapes) these endpoints at a regular interval (typically every 15 seconds)
- The server stores the data in a custom **time-series database** (TSDB) optimized for this access pattern
- A query language (PromQL) lets you compute rates, aggregates, and joins over these time series

### Why time-series data is stored differently from log data

A time-series database exploits a very specific access pattern: you're writing a fixed set of series with new timestamps continuously, and you're reading contiguous time ranges of those series. This is radically different from full-text search over logs.

The key optimizations:
- **Columnar storage**: all timestamps for a series are stored together, all values together. This compresses extremely well because consecutive timestamps are nearly identical (each 15s apart) and values change slowly (CPU usage doesn't jump from 5% to 95% every sample). Delta-of-delta encoding and XOR-based float compression (from a Facebook paper on their Gorilla TSDB) can achieve 1-2 bytes per sample.
- **Append-only writes**: new data always goes to the end. No random updates. This enables very high write throughput.
- **Time-based partitioning**: data is organized into blocks by time range (e.g., 2-hour blocks). Querying "last hour" only touches one block. Old blocks can be compacted and dropped efficiently (retention management).

Compare this to Elasticsearch for logs: each log entry is a document with arbitrary fields, indexed for full-text search. The index overhead is massive — a common rule of thumb is that Elasticsearch uses 2-3x the raw data size in disk for indexing. A time-series database storing the same information as numerical aggregates might use 1/100th the storage.

**The tradeoff metrics make:** Detail for cost and speed. When you store "there were 23 errors in the last 15 seconds," you've lost which 23 requests failed, what the error messages were, which users were affected. Metrics tell you *that* something is wrong and roughly *where*. Logs tell you *what* happened specifically. You need both.

**Who consumes metrics vs. logs:**
- **Metrics** are primarily for operational visibility: dashboards that SREs watch during incidents, alerting systems that page someone at 3 AM, executive uptime reports (monthly error rate to SLA percentage), capacity planning (memory growth trend to when do we need to provision more).
- **Logs** are primarily for diagnosis: once you know *something* is wrong (from metrics or alerts), you look at logs to understand *what* specifically. Security teams also live in logs — audit trails, access records, forensic evidence.

This division isn't absolute — you can derive counts from logs and you can annotate metrics — but it explains why these became separate systems rather than one unified platform.

---

## Stage 4: Alerting — Closing the Loop

Dashboards are passive. Someone has to look at them. The natural next step: make the system tell you when something is wrong.

An alert rule is a metrics query plus a condition plus a duration:

```yaml
# Prometheus alerting rule
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 5% for 5 minutes on {{ $labels.service }}"
```

This says: if the 5xx error rate exceeds 5% for 5 consecutive minutes, fire an alert. A component like **Alertmanager** (Prometheus's companion) receives fired alerts and handles routing (who gets paged), grouping (don't send 50 separate pages for 50 instances), silencing (suppress known issues during maintenance), and deduplication.

This introduced a new problem: **alert fatigue.** If you set thresholds too low, you get paged for noise. Too high, you miss real incidents. And thresholds are crude — a 5% error rate might be catastrophic for a payments service and normal for a best-effort recommendations service.

### SLOs: A Better Model for Alerting

The deeper insight came from Google's SRE practice: instead of alerting on symptoms directly, define what "good enough" means for users and alert on deviations from that.

A **Service Level Indicator (SLI)** is a measurement of user-visible behavior: "the proportion of HTTP requests that complete in under 300ms with a non-error status." This is what users actually experience.

A **Service Level Objective (SLO)** is a target for that indicator over a time window: "99.9% of requests should be successful over a 30-day rolling window." This gives you 0.1% error budget — about 43 minutes of total downtime per month.

Now alerting becomes: "we're burning through our error budget faster than expected." If your budget is 0.1% over 30 days, and in the last hour you've consumed 5% of that budget, something is wrong even if the instantaneous error rate looks acceptable.

```
# SLO burn rate alert
# If we're consuming error budget 14x faster than sustainable, page immediately
# (this means we'd exhaust the monthly budget in ~2 days)
- alert: SLOBudgetBurnHigh
  expr: slo:error_budget_burn_rate:1h > 14
  for: 5m
```

**The relationship between alerting and SLOs:** Traditional alerting asks "is this metric outside a threshold?" SLO-based alerting asks "are users having a bad enough experience that we should interrupt an engineer?" The threshold isn't arbitrary — it's derived from a business decision about acceptable reliability. This connects engineering to business in a way raw metric thresholds never could.

This is also where **accountability** enters distinct from **operational visibility**. An SLO tracked over months produces an SLA compliance report. It answers the executive question "how reliable were we?" and the customer question "did you meet your contractual commitment?" The same data serves debugging (what's breaking?) and audit (did we deliver?).

---

## Stage 5: The Distributed Systems Problem

Everything above works for a world where one request hits one service on one machine. But by the mid-2010s, that world was disappearing. Microservices, SOA, whatever you want to call the architecture: a single user request might fan out across 5, 10, 50 services.

A user reports "the page is slow." You check your metrics: latency is elevated on the API gateway. You look at the gateway's logs: it's waiting on the user service. You check the user service: it's waiting on the database. But wait — the user service also calls the permissions service, which calls the policy service. The slow request might have taken a path that hits five services, and the bottleneck could be anywhere in that chain.

**You can't grep your way through a distributed system.** Even with centralized logging, correlating events across services requires some shared identifier. Some teams started doing this ad hoc — generating a "request ID" at the edge and passing it through as a header, then including it in every log line:

```
# API Gateway
{"request_id":"abc-123","msg":"incoming request","path":"/api/users/789"}

# User Service
{"request_id":"abc-123","msg":"fetching user","user_id":789}

# Database proxy
{"request_id":"abc-123","msg":"query executed","table":"users","duration_ms":234}
```

Now you can search all logs for `request_id=abc-123` and see the full journey. But this still has problems:

1. You see a flat list of events. You don't see the *structure* — which service called which, in what order, how long each step took.
2. You need to instrument every service to propagate the ID. One service that doesn't forward it breaks the chain.
3. You don't get timing information unless every service logs it consistently.

### Distributed Tracing

The solution was formalized in Google's **Dapper** paper (2010) and implemented in open-source systems starting with Zipkin (2012) and Jaeger (2017).

The core data model is a **trace** composed of **spans**. A trace represents a single end-to-end operation (one user request). Each span represents one unit of work within that trace:

```
Trace ID: abc-123-def-456

Span: API Gateway (120ms)
+-- Span: Auth middleware (5ms)
+-- Span: User Service call (95ms)
|   +-- Span: Cache lookup (2ms, cache miss)
|   +-- Span: Database query (80ms)
|   |   +-- Span: Connection pool wait (60ms)  <-- HERE'S YOUR PROBLEM
|   +-- Span: Serialize response (3ms)
+-- Span: Response serialization (2ms)
```

Each span has:
- A **trace ID** (shared by all spans in the trace)
- A **span ID** (unique to this span)
- A **parent span ID** (which span caused this one)
- A start time and duration
- A set of **attributes** (key-value pairs, like HTTP method, status code, database query)
- Optional **events** (timestamped annotations within the span, like "cache miss" or "retry attempt 2")
- A **status** (ok, error)

**How context actually propagates across services:**

When Service A makes an HTTP call to Service B, the tracing library injects the trace context into HTTP headers:

```
GET /api/users/789 HTTP/1.1
Host: user-service
traceparent: 00-abc123def456-span789-01
```

The `traceparent` header (defined in the W3C Trace Context standard) carries the trace ID, the parent span ID, and sampling flags. When Service B's tracing library receives this request, it extracts the context, creates a new span with the received trace ID and parent span ID, and continues the chain. The same mechanism works for gRPC (metadata headers), message queues (message attributes), and any other transport — you just need the instrumentation to inject/extract from the right place.

This is called **context propagation**, and it's the fundamental mechanism that makes distributed tracing work. Without it, you have isolated spans with no way to connect them into a tree.

**The tradeoff tracing makes: sampling.**

A trace for a single request might contain 50 spans. If you handle 10,000 requests per second, that's 500,000 spans per second. Storing all of them is expensive, and most of them are boring — successful requests that completed normally.

Most tracing systems sample: only keep a fraction of traces. Common strategies:

- **Head-based sampling**: decide at the edge whether to trace this request (e.g., trace 1% of requests). Simple, but you might miss the interesting ones.
- **Tail-based sampling**: collect all spans, but only persist traces that are "interesting" (errors, high latency, specific user). This requires buffering complete traces before deciding, which is architecturally more complex.
- **Adaptive sampling**: adjust the rate based on traffic volume — sample more during low traffic, less during peaks.

Sampling trades completeness for cost. A 1% sample rate means you'll see the pattern of a slow database query, but you might miss a specific rare error. This is acceptable because tracing answers "where in the call chain is time being spent?" not "what happened to this specific request?" (For that, you'd need logs.)

---

## Stage 6: Why Three Pillars? Convergence and Divergence

By the late 2010s, the industry had arrived at "three pillars of observability": logs, metrics, and traces. This framing was popularized by practitioners and vendors, and it's worth asking: is this a fundamental distinction or a historical accident?

**The argument for fundamental:** Each pillar captures a genuinely different type of information with different storage and query requirements:

|            | Logs                              | Metrics                                | Traces                                  |
|------------|-----------------------------------|----------------------------------------|-----------------------------------------|
| Data model | Arbitrary events with fields      | Numeric time series with labels        | Trees of timed spans                    |
| Cardinality| Unbounded (each event is unique)  | Bounded (fixed set of series)          | Moderate (one trace per request, sampled)|
| Storage    | Full-text/columnar index          | Time-series database                   | Span storage with graph queries         |
| Primary query | "Show me events matching X"    | "What's the rate/percentile of X over time?" | "Show me the call tree for request X" |
| Cost driver| Volume (bytes ingested)           | Cardinality (number of unique series)  | Volume x depth (spans per trace x traces stored) |

These are genuinely different data models. Trying to store metrics in a log index wastes enormous storage. Trying to store arbitrary log events in a time-series database doesn't work (the cardinality explodes). Trying to reconstruct a trace from logs requires correlating across services and inferring the tree structure.

**The argument for historical accident:** All three capture facts about what a system did. A span IS an event — a structured log entry with a parent pointer. A metric CAN be derived from a stream of events (count events per second = counter). The three-pillar separation exists partly because different teams built different tools at different times, and storage engines were optimized for different access patterns. If you had a single store that handled all patterns efficiently, the distinction might dissolve.

**The practical reality is somewhere in between.** The distinction is real enough that specialized storage still matters, but the boundaries are blurring. Systems like Grafana Loki store logs but support deriving metrics from log queries. Tracing systems embed log-like events within spans. Some newer databases (ClickHouse, Databricks) handle all three in a single columnar store with good-enough performance for each, at the cost of not being best-in-class for any one.

---

## Stage 7: OpenTelemetry — The Instrumentation Problem

By 2019, there was a different kind of problem. Not "how do we store this data" but "how do we generate it."

Every vendor had their own instrumentation library. If your Go service used Datadog, you'd import the Datadog tracing library. If you then wanted to switch to Jaeger, you'd rewrite your instrumentation code. Libraries (HTTP frameworks, database drivers) couldn't add built-in tracing because they'd have to pick a vendor.

Two prior efforts — **OpenTracing** (a vendor-neutral tracing API) and **OpenCensus** (Google's instrumentation library for traces and metrics) — merged in 2019 to form **OpenTelemetry** (OTel). It is now the second most active CNCF project after Kubernetes.

OpenTelemetry provides:

1. **A specification**: defines the data model for traces, metrics, and logs. What a span looks like, what attributes it carries, what metric types exist.

2. **APIs and SDKs** for every major language: you instrument your code once using the OTel API, and configure at deployment time where the data goes (Jaeger, Datadog, Honeycomb, whatever).

3. **Semantic conventions**: standardized attribute names. An HTTP server span should have `http.request.method`, `url.path`, `http.response.status_code`. A database span should have `db.system`, `db.statement`. This solves the "one team logs `user_id`, another logs `uid`" problem — at least for common operations.

4. **Auto-instrumentation**: libraries for common frameworks that add tracing and metrics without changing application code. For example, OTel's Java agent can instrument Spring Boot, gRPC, JDBC, etc., by bytecode manipulation at startup.

5. **The Collector**: a standalone process that receives telemetry data, processes it (batching, filtering, sampling, enriching), and exports it to one or more backends. This decouples your application from your observability vendor:

```
[Your Service] -> OTel SDK -> [OTel Collector] -> Jaeger (traces)
                                               -> Prometheus (metrics)
                                               -> Loki (logs)
```

The **OTLP protocol** (OpenTelemetry Protocol) is the wire format — a protobuf-based protocol for transmitting traces, metrics, and logs over gRPC or HTTP.

What OTel represents architecturally is the separation of **instrumentation** from **backend**. You instrument once, export anywhere. This is the same pattern as JDBC (write SQL once, swap databases) or POSIX (write code once, run on any Unix). Standardization at the interface layer enables competition and interoperability at the implementation layer.

**What problems OpenTelemetry introduced:**

- **Complexity**: the API surface is large. Configuring the SDK, collector pipelines, sampling strategies, and resource detection is non-trivial. A minimal setup requires understanding contexts, propagators, exporters, and processors.
- **Performance overhead**: auto-instrumentation adds latency (typically microseconds per span, but it adds up). The SDK itself consumes memory. The collector is another process to operate.
- **Incomplete convergence**: logs support in OTel was the last pillar added and is still maturing. Many organizations use OTel for traces and metrics but keep their existing logging pipeline (Fluentd to Elasticsearch).

---

## Stage 8: High-Cardinality, Wide Events, and the "Observability" Distinction

Around 2016-2018, Charity Majors and the team at Honeycomb started arguing that the industry's approach was fundamentally backwards. Their argument:

Traditional **monitoring** asks predefined questions: "Is the error rate above 5%? Is latency above 200ms? Is disk usage above 80%?" You decide in advance what matters, create dashboards and alerts for those things, and hope that when something breaks, it's one of the things you anticipated.

**Observability** — borrowed from control theory, where a system is "observable" if you can infer its internal state from its external outputs — is the ability to understand a system's internal state from its external outputs. Applied to software: can you ask *arbitrary, novel* questions about your system's behavior without having anticipated them? Can you debug a problem you've never seen before?

The argument is that metrics fail this test. Metrics are pre-aggregated: you decided at instrumentation time which dimensions to track (HTTP method, path, status code). If the problem is "requests from user 789 on iOS 17.2 in the EU region are slow because they hit a specific database shard," you can't answer that from metrics unless you had all of those as labels — and you can't have all possible combinations as labels because the **cardinality** (number of unique label combinations) would explode your time-series database.

The proposed solution: **wide structured events**. Instead of three separate systems, emit a single rich event per unit of work (per request, per job, per transaction) with every attribute you might care about:

```json
{
  "trace_id": "abc123",
  "service": "api-gateway",
  "duration_ms": 342,
  "http.method": "GET",
  "http.path": "/api/users/789",
  "http.status": 200,
  "user.id": 789,
  "user.plan": "enterprise",
  "user.region": "eu-west-1",
  "app.version": "2.3.1",
  "build.commit": "a1b2c3d",
  "db.shard": "shard-07",
  "db.duration_ms": 289,
  "cache.hit": false,
  "feature_flags.new_query_engine": true,
  "endpoint.handler": "getUserProfile",
  "k8s.pod": "api-gateway-7f8b9-xk2lm",
  "k8s.node": "node-pool-3-vm-12"
}
```

Store these events in a columnar store that can handle high-cardinality queries: "GROUP BY `db.shard` WHERE `duration_ms` > 500 AND `user.region` = 'eu-west-1'" — ad hoc, after the fact, without having predicted this combination matters.

**This is the conceptual distinction between monitoring and observability:** monitoring checks known conditions; observability lets you explore unknown unknowns. In practice, most organizations do both — metrics and alerts for known failure modes, high-cardinality event data for novel investigation.

**The tradeoff:** Wide events are expensive. Each event might have 50-200 fields. At high request rates, this is a lot of data. You're back to the cost/completeness tension — sampling helps, but then you might miss the rare event that matters. Columnar stores (like ClickHouse) have made this more tractable than it was a decade ago, but it's still fundamentally more expensive than pre-aggregated metrics.

---

## Stage 9: Profiling — The Fourth Pillar, or Something Else?

There's a category of questions that none of the above answers well: "Why is my service using 4 GB of memory? Which function is consuming 80% of CPU?"

**Continuous profiling** captures resource usage at the code level — CPU time per function, memory allocation per call site, lock contention per mutex. The dominant technique is **sampling profiling**: periodically (e.g., 100 times per second) interrupt the program and record the call stack. Aggregate these samples and you get a statistical picture of where time is spent:

```
main
+-- handleRequest        35%
    +-- parseJSON         12%
    +-- queryDatabase     18%
    |   +-- waitForConn    15%  <-- connection pool bottleneck
    +-- serializeResponse  5%
```

This is typically visualized as a **flame graph** (invented by Brendan Gregg at Netflix in 2011): a stacked bar chart where the x-axis is the proportion of samples and the y-axis is the call stack depth. You can immediately see which code paths dominate resource consumption.

Tools like **pprof** (Go's built-in profiler), **async-profiler** (JVM), **perf** (Linux kernel), and commercial products like Datadog Continuous Profiler and Pyroscope run in production, continuously sampling. The overhead is low — sampling at 100 Hz adds roughly 1-2% CPU overhead.

**Is it a fourth pillar?** It depends on your definition. Profiling answers "which code is expensive?" while the other three answer "what happened and when?" Profiling is more about *why* at the code level than *what* at the system level. OpenTelemetry has added profiling to its specification (as of 2024), treating it as a signal type alongside traces, metrics, and logs. Whether that makes it a "pillar" or a complementary tool is mostly a semantic debate — what matters is that it fills a real gap.

The gap: metrics tell you CPU is at 90%. Traces tell you requests are slow. Profiling tells you that `json.Unmarshal` is responsible and the fix is to switch to a streaming parser. Without profiling, you're staring at traces saying "the database call took 200ms" without knowing whether that's the query execution, the serialization, or the connection pool wait.

---

## Stage 10: The Current State and Open Problems

As of the mid-2020s, here's where the field stands:

**What's largely solved:**
- Instrumentation standardization (OpenTelemetry is the clear winner)
- Time-series storage and querying (Prometheus, Mimir, VictoriaMetrics, and cloud-native offerings are mature)
- Log aggregation at scale (even if expensive)
- Distributed tracing as a concept (widely adopted, well-understood)

**What's still hard:**

**Cost.** Observability infrastructure is often one of the largest line items after compute and storage. Organizations routinely spend millions annually on Datadog, Splunk, or equivalent. The cost scales with the data, and the data scales with the system complexity. Every improvement in observability generates more data, which costs more to store and query. This creates perverse incentives: teams reduce logging and sampling rates to save money, losing visibility precisely when they need it most.

**Correlation across signals.** You have traces in Jaeger, metrics in Prometheus, logs in Loki, profiles in Pyroscope. A single incident requires correlating across all four. "The p99 latency spiked" (metrics) then "this trace shows a slow database call" (traces) then "the database logged a deadlock retry" (logs) then "the ORM's connection pool code has a hot lock" (profiles). Grafana and other platforms are building unified views, but the cross-referencing is still manual. OTel's exemplars (linking a specific metric data point to a trace ID) help:

```
# A histogram bucket with an exemplar pointing to a specific trace
http_request_duration_seconds_bucket{le="0.5"} 4892 # {trace_id="abc123"} 0.48
```

This lets you click from a latency spike on a dashboard to the specific trace that caused it. But it's still early, and many organizations don't have it wired up.

**The "too much data, not enough insight" problem.** Modern observability systems are excellent at collecting data and mediocre at surfacing answers. During an incident, an SRE still has to manually form hypotheses, query dashboards, and correlate signals. The push toward LLM-assisted incident response is an attempt to automate this — "here are all the signals from the last 10 minutes, what changed?" — but it's early and unreliable.

**Multi-tenancy and privacy.** When observability data contains user IDs, request bodies, and database queries, it becomes a privacy concern. Logs might contain PII that shouldn't be stored (or should be stored encrypted). Traces crossing organizational boundaries (your service calls a vendor's API) raise questions about data ownership. This creates tension between completeness (more context = better debugging) and compliance (less data = less risk).

### Security and Compliance — The Other Consumers

Everything discussed above primarily serves operational debugging. But the same data serves other purposes:

**Security teams** (SIEM — Security Information and Event Management) want logs for threat detection: unusual login patterns, privilege escalation, data exfiltration. They need long retention (months to years) and different query patterns (behavioral anomalies across many dimensions over long time windows). This is why security logging often ends up in separate systems — the retention and query requirements diverge from operational debugging.

**Audit/compliance** requires proof that certain things happened (or didn't). Who accessed this database? When was this configuration changed? This demands **immutable, tamper-evident storage** — you can't allow operators to delete logs that might show their mistakes. This is a fundamentally different trust model from operational logging, where you actively want to prune and compress.

**Product/analytics teams** want to understand user behavior. How many users completed the checkout flow? Where did they drop off? This is technically the same as trace data (a user session is a trace through the product), but the analysis is different — funnels, cohorts, A/B test comparisons rather than latency distributions.

---

## Looking Back: The Shape of the Progression

The entire history follows a pattern:

1. You have no visibility -> add printf
2. Printf doesn't persist -> write to files
3. Files are on separate machines -> centralize them
4. Centralized text is unqueryable -> structure it
5. Individual events are too expensive to query in aggregate -> pre-aggregate into metrics
6. Static thresholds generate noise -> tie alerting to user experience (SLOs)
7. Single-service visibility breaks in distributed systems -> propagate context (tracing)
8. Predefined dimensions can't answer novel questions -> high-cardinality wide events
9. System-level signals can't explain code-level bottlenecks -> profiling
10. Separate systems are hard to correlate -> unify instrumentation (OpenTelemetry)

At each step, the solution to the previous problem created a new problem. More data solved the visibility problem and created the cost problem. Aggregation solved the cost problem and created the detail problem. Distribution solved the scale problem and created the correlation problem.

The field is still somewhere between steps 10 and 11, where step 11 is probably "automated reasoning over unified telemetry" — a system that doesn't just collect and display data but actively tells you what's wrong and why. Whether LLMs, causal inference, or something else gets us there remains an open question.

The deepest lesson from this history is that observability is not a tool or a stack — it's a property of a system. A system is observable to the degree that you can understand its internal state from its outputs. Every tool in this progression is an attempt to increase that degree, and every tradeoff is a choice about which dimensions of understanding to prioritize given finite resources.
