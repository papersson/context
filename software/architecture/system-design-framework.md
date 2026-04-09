# System Design Decision Framework

## The Core Principle

Architecture is the shape that constraints force onto a system. Without constraints, every system is a single process reading and writing a single database. Complexity is justified only when a specific, measured property of the workload demands it.

The sequence matters:

1. **Characterize the workload** — measure the physical reality
2. **Identify dominant constraints** — find the two or three things that actually force decisions
3. **Derive the architecture** — let constraints dictate structure
4. **Choose technology** — pick tools that serve the architecture

Engineers routinely invert this. They start with technology ("we should use Kafka") or patterns ("let's do event sourcing") and work backward to justify the choice. This produces systems that are complex in the wrong places and simple in the wrong places. The framework below makes the correct ordering concrete and repeatable.

---

## Step 1: Workload Characterization

Before any design discussion, answer these questions with numbers. Estimates are fine. Orders of magnitude are sufficient. The goal is to see the physical shape of the system before deciding anything.

### 1.1 Data Producers

**Who or what generates data, and at what rate?**

| Question | Why it matters |
|----------|---------------|
| How many distinct sources produce data? | 1 source vs. 10,000 sources changes everything about ingestion |
| What is the peak write rate? (events/sec, rows/sec, MB/sec) | Determines whether a single writer suffices or you need partitioning |
| Is production bursty or steady? | Bursty traffic needs buffering; steady traffic can be processed inline |
| What is the payload size per write? | 200 bytes vs. 50 MB per event drives different storage and transport choices |
| Is data produced in order, or can events arrive out of order? | Out-of-order arrival forces explicit ordering or tolerance of disorder |

**Real examples to calibrate your intuition:**

- **Stripe's payment processing**: ~thousands of payment events per second during peak, each event ~1-5 KB. Bursty (Black Friday), ordered per-merchant but not globally. This is a medium-write, high-consistency workload.
- **Datadog's metric ingestion**: billions of data points per minute from millions of agents, each point ~100 bytes. Steady with diurnal patterns. This is an extreme-write, low-consistency-per-point workload.
- **A company's internal expense reporting tool**: maybe 50 submissions per day, each ~10 KB with an attached receipt image. Completely steady. This is a trivial-write workload where write scaling is irrelevant.

### 1.2 Data Consumers

**Who or what reads data, and what do they need?**

| Question | Why it matters |
|----------|---------------|
| How many distinct consumers exist? | 1 consumer vs. 1,000 consumers changes delivery and caching strategy |
| What read patterns dominate? Point lookups, range scans, aggregations, full scans? | Determines index strategy, storage format, and whether precomputation helps |
| What is the peak read rate? | Read-heavy systems need caching or read replicas; write-heavy systems don't |
| Do consumers need the same view of data or different views? | Multiple views push toward materialized views, CQRS, or read-optimized stores |
| Is read latency critical (user-facing) or tolerant (background job)? | User-facing reads under 200ms constraint shape everything downstream |

**Real examples:**

- **Twitter's home timeline (pre-X)**: Hundreds of millions of users, each requesting a personalized feed (fan-out-on-read or fan-out-on-write). Read-dominant, latency-critical, every user sees a different view. This forced precomputation of timelines.
- **Spotify's Wrapped**: Reads happen once a year per user. The entire read workload is a single annual burst of aggregate queries over a year of listening history. Precompute everything into static artifacts, serve them from a CDN.
- **An internal admin dashboard for a 50-person company**: 3 people use it, each loads it maybe 10 times a day. Read latency of 2 seconds is fine. There is no read scaling problem. A single SQL query is the architecture.

### 1.3 Data Shape and Lifecycle

| Question | Why it matters |
|----------|---------------|
| What is the total dataset size? Will it fit in memory? On a single disk? | In-memory vs. disk-based vs. distributed storage is a foundational fork |
| What is the growth rate? | A system that grows 1 GB/year and one that grows 1 TB/day have nothing in common |
| Is data mutable or append-only? | Mutability requires coordination; append-only enables replication and replay |
| What is the retention policy? | Infinite retention vs. 30-day TTL changes storage cost and compaction strategy |
| Are there relationships between data items? How complex? | Deep relational graphs push toward relational databases; isolated documents push toward document stores |
| What is the schema stability? | Stable schemas enable columnar formats and strong typing; evolving schemas need flexible formats |

**Real examples:**

- **GitHub's repository storage**: Append-only (git objects are immutable), relationships are DAGs (commits, trees, blobs), total size is petabytes, growth is steady. The immutability of git objects means replication is simple — you never update, only add.
- **Figma's document model**: Highly mutable (users edit in real time), complex internal structure (a tree of design nodes with properties), single documents are typically under 100 MB, but they change hundreds of times per second during active editing. Mutability + real-time + collaborative = CRDT territory.
- **A financial ledger**: Append-only by regulation, simple record shape, relationships are minimal (each entry references an account), grows steadily, retained forever. The append-only + forever-retention combination makes event sourcing natural — not because it's trendy, but because the domain literally is a sequence of immutable events.

### 1.4 Freshness and Consistency

| Question | Why it matters |
|----------|---------------|
| How stale can a consumer's view be? Milliseconds? Seconds? Minutes? Hours? | This single number eliminates or requires entire categories of architecture |
| Do all consumers need the same staleness guarantee? | If some need real-time and some tolerate hours, you may need two read paths |
| What happens if a consumer sees stale data? Annoyance? Financial loss? Safety hazard? | The consequence of staleness determines how much you invest in freshness |
| Do two concurrent readers need to see the same data? | If yes, you need linearizable reads. If no, eventual consistency is fine |
| Can a write be lost? What is the cost of losing one? | Determines durability guarantees: fsync policy, replication factor, ack semantics |

**The freshness spectrum, with real examples:**

| Freshness | Example | Architectural implication |
|-----------|---------|--------------------------|
| < 10ms | Stock trading (limit order book at Jane Street) | Data must live in-process memory. No network hops in the read path. No GC pauses. |
| 10-100ms | Multiplayer game state (Rocket League server) | In-memory with network replication. UDP over TCP. Interpolation on the client. |
| 100ms - 1s | Collaborative editing (Google Docs) | Operational transforms or CRDTs over WebSockets. Server can mediate. |
| 1-10s | Social media feed (Instagram) | Cache invalidation is viable. Push on write, poll on read, or SSE/WebSocket for active users. |
| 10s - 5min | Monitoring dashboard (Grafana) | Periodic polling. Pre-aggregated rollups. No push infrastructure needed. |
| 5min - 1hr | Business analytics (Looker) | Materialized views refreshed on schedule. Query caching. |
| Hours - Days | Data warehouse (weekly business review) | Batch ETL. Full rebuilds of aggregates overnight. Simplest possible pipeline. |

### 1.5 Failure Tolerance

| Question | Why it matters |
|----------|---------------|
| What happens if the system is down for 1 minute? 1 hour? 1 day? | Availability requirements determine redundancy investment |
| Is partial degradation acceptable? (e.g., reads work but writes don't) | Graceful degradation avoids over-provisioning for rare failures |
| Can work be retried? Is it idempotent? | Retryable + idempotent workloads can use simpler, less reliable infrastructure |
| What is the blast radius of a single component failure? | Determines whether you need isolation boundaries (separate processes, separate clusters) |

---

## Step 2: Identify Dominant Constraints

After characterization, most of the numbers will be unremarkable. A system with 100 requests per second, 10 GB of data, 5-second freshness tolerance, and single-digit consumers has no dominant constraints. The architecture is "a web server and a database." Do not make it more complicated.

The dominant constraints are the properties that **rule out the simple default**. Usually there are one to three. Rarely more.

### How to find them

For each characterization dimension, ask: **does this value force me away from a single-process, single-database system?**

| Property | Threshold where it starts to matter | What it forces |
|----------|-------------------------------------|----------------|
| Write rate | > 10,000/sec sustained | Write partitioning, async writes, or append-only stores |
| Read rate | > 50,000/sec sustained | Read replicas, caching layers, or precomputation |
| Total data size | > what fits on one machine (~1-10 TB) | Distributed storage, sharding |
| Freshness requirement | < 1 second | Push-based delivery, in-memory state, WebSockets or SSE |
| Consistency requirement | Must be linearizable | Consensus protocols, single-leader writes, reduced availability |
| Availability requirement | > 99.99% | Multi-region, redundant everything, no single points of failure |
| Consumer diversity | > 3 fundamentally different read patterns | Multiple read-optimized stores or materialized views |
| Payload size | > 1 MB per event | Separate data and metadata paths, streaming transfers, object storage |
| Burst ratio (peak/avg) | > 100x | Buffering layer (queue) between producers and processors |
| Regulatory / compliance | PII, financial, healthcare data | Encryption at rest, audit logs, geographic constraints, retention policies |

Most systems have zero or one dominant constraint. A few have two. If you find yourself listing five dominant constraints, you are either building something genuinely complex (a global-scale real-time financial system) or you are over-specifying.

### Worked examples

**Example: Slack**

Characterization reveals:
- Write rate: moderate (humans type slowly; even millions of users produce maybe low thousands of messages/sec)
- Read rate: moderate per-channel but high in aggregate across millions of concurrent connections
- Freshness: < 1 second (users expect messages to appear instantly)
- Consistency: per-channel ordering matters; cross-channel ordering does not
- Availability: high (people rely on it for work)
- Data shape: append-only messages, mutable channel metadata

Dominant constraints: **sub-second delivery to many concurrent consumers** and **per-channel ordering**. Not write throughput. Not data volume. The architecture is shaped by the push delivery requirement (WebSocket connections, pub/sub per channel) and per-channel ordering (partition by channel). Everything else can be simple.

**Example: Stripe**

Characterization reveals:
- Write rate: moderate (thousands/sec, not millions)
- Freshness: seconds are fine for dashboards; milliseconds matter for webhook delivery
- Consistency: **critical** — a payment must either fully succeed or fully fail; double-charging is catastrophic
- Durability: **critical** — losing a payment record is a regulatory and financial disaster
- Availability: very high — merchants lose money if Stripe is down

Dominant constraints: **consistency (exactly-once payment processing)** and **durability (no data loss)**. Not scale. Stripe's architecture is shaped by the need for idempotent writes, distributed transactions across payment processors, and strong audit logging. The throughput could be handled by a single Postgres instance; the consistency and durability requirements are what force the complexity.

**Example: Internal HR dashboard at a 200-person company**

Characterization reveals:
- Write rate: a few writes per day (someone updates a record)
- Read rate: a few reads per hour
- Freshness: minutes to hours are fine
- Data size: a few MB
- Consumers: 2-3 HR staff

Dominant constraints: **none**. There is no property of this workload that forces any architectural decision. The correct architecture is a web framework with an ORM and a single relational database. If someone proposes microservices, a message queue, or a caching layer for this system, they are adding risk for zero benefit.

---

## Step 3: Derive the Architecture

Architecture is the set of decisions about state, computation, communication, and failure handling. Each decision should trace to a dominant constraint. If you can't explain which constraint forces a decision, the default applies: **pick the simplest option**.

### 3.1 State Management

**Where does data live?**

Start with the simplest option that satisfies the constraints and stop there.

| Constraint forcing the decision | State management response |
|---------------------------------|---------------------------|
| No dominant constraints | Single relational database. One instance. No replicas. |
| Read rate exceeds single-node capacity | Add read replicas. Keep a single writer. |
| Write rate exceeds single-node capacity | Partition (shard) writes by a natural key. Each partition is its own independent database. |
| Data exceeds single-machine storage | Distributed storage (sharded database, object store, or HDFS-class system). |
| Sub-second freshness + many consumers | In-memory cache or in-memory data grid in front of persistent storage. |
| Multiple fundamentally different read patterns | Materialized views or separate read-optimized stores (CQRS). The write store is normalized; the read stores are denormalized for their specific access pattern. |
| Data is append-only + must be auditable | Event log as the primary store. Derived views are projections of the log. This is event sourcing, but only when the domain naturally produces immutable events — not as a default choice. |
| Sub-millisecond access required | Data lives in the application process memory. No database in the hot path. |

**When to shard, and by what key:**

Sharding is the decision with the highest operational cost. Avoid it unless write throughput or data size forces it. When you must shard:

- Shard by the key that co-locates data that is accessed together. Discord shards by guild (server) because nearly all queries are scoped to a single guild. Slack shards by workspace for the same reason.
- If no natural co-location key exists, shard by a hash of the primary key for even distribution. You give up range queries.
- Never shard by a key that creates hot partitions. Sharding by date means all writes hit today's partition. Sharding by user ID when one user has 10,000x the traffic of others concentrates load.

**Mutability model:**

| Pattern | When it fits | Real example |
|---------|-------------|--------------|
| Mutable records in a relational database | Default for most applications | Any CRUD app. Rails + Postgres. |
| Append-only log + derived views | Domain events are the source of truth; multiple consumers derive different views | Financial ledgers, Kafka-based event systems at LinkedIn |
| Immutable snapshots + diffs | Data changes often but you need point-in-time views | Git (snapshots of tree state), Datomic (immutable database of facts) |
| CRDTs or operational transforms | Multiple writers, no central coordinator, conflicts must merge automatically | Figma (multiplayer design), Google Docs (collaborative text editing) |

### 3.2 Computation Model

**How does work get done?**

| Constraint | Computation model |
|------------|-------------------|
| No dominant constraints | Synchronous request-response. A web server handles a request, queries the database, returns a response. This is the correct default. |
| High latency tolerance + large data volume | Batch processing. Read everything, compute, write results. MapReduce, Spark jobs, nightly ETL. Simplest model for non-interactive workloads. |
| Continuous input + bounded latency | Stream processing. Events flow through a DAG of operators. Each operator transforms or aggregates. Flink, Kafka Streams. |
| Independent items + CPU-bound processing | Data parallelism. Partition the work, process partitions independently, merge results. Embarrassingly parallel. |
| Complex state per entity + concurrent access | Actor model. Each entity is an actor with its own state and mailbox. Akka, Erlang/OTP, Orleans (virtual actors). |
| Multi-step workflow with failure handling | Orchestrated pipeline with checkpointing. Temporal, Airflow, Step Functions. Each step is retryable. The orchestrator tracks progress. |
| Mixed interactive + background work | Request-response for the interactive path. Queue + workers for background processing. This is the most common real-world hybrid. |

**The computation model choice is often obvious once you know the freshness and throughput constraints:**

- If a user is waiting for the result: request-response.
- If the result is needed within seconds but no one is blocking on it: async worker consuming from a queue.
- If the result is needed within minutes to hours: batch job.
- If data arrives continuously and results must be continuously updated: stream processing.

**Real example — Uber's trip pricing:**

The freshness constraint is tight (price must appear in < 1 second when user requests a ride), the computation involves real-time data (surge pricing based on current demand), and the request rate is high. This points to request-response for the user-facing path with precomputed pricing models updated by a streaming pipeline that processes demand signals. Two computation models, each traced to a specific constraint: the user-facing latency requirement drives synchronous pricing, and the real-time demand signal drives the streaming model update.

### 3.3 Communication Patterns

**How do components talk to each other?**

| Pattern | When to use | When not to use |
|---------|-------------|-----------------|
| Synchronous request-response (HTTP, gRPC) | Caller needs the result before continuing. Default for service-to-service calls. | Callee is slow or unreliable. You'll block the caller and cascade failures. |
| Async message queue (RabbitMQ, SQS) | Producer and consumer operate at different speeds. Work can be retried. Result is not needed immediately. | You need a response back to the caller. Request-reply over queues is an anti-pattern; it adds latency and complexity vs. a direct call. |
| Log-based messaging (Kafka, Kinesis) | Multiple consumers need the same events. Events must be replayed. Ordering within a partition matters. High throughput. | Single consumer, no replay need. A simple queue is operationally cheaper. Kafka is not a job queue. |
| Pub/sub (Redis pub/sub, NATS) | Real-time fan-out to many consumers. Fire-and-forget. No durability needed. | Messages can't be lost. Pub/sub has no persistence; if a subscriber is down, it misses messages. |
| WebSocket / SSE | Server must push data to clients with low latency. Long-lived connections. | Request-response suffices. Don't hold open a WebSocket to poll every 30 seconds; a periodic HTTP request is simpler. |
| Shared database | Two components need the same data. Both are in the same trust boundary. The schema is the contract. | Components are owned by different teams. The schema becomes a coupling point that's hard to evolve. |

**A decision tree for communication:**

1. Does the caller need a response before continuing? → **Synchronous call.**
2. Can the work happen later? → **Queue.**
3. Do multiple consumers need the same events? → **Log-based messaging.**
4. Must the server push to the client unprompted? → **WebSocket / SSE.**
5. None of the above? → **Synchronous call.** It's the default.

### 3.4 Failure Handling

Failure handling is not a separate architectural decision — it falls out of the choices above. Each state management and communication choice implies a failure model.

| Choice you made | Failure mode to handle | Standard approach |
|-----------------|------------------------|-------------------|
| Single database | Database goes down, everything is unavailable | Automated failover to a standby. Acceptable for most workloads. |
| Read replicas | Replica lag causes stale reads | Read-your-writes consistency: after a write, route that user's reads to the primary for a few seconds. |
| Sharded database | One shard goes down, partial outage | Shard-level redundancy. The blast radius is 1/N of users. Accept this or add replicas per shard. |
| Synchronous service calls | Downstream service is slow or down, caller blocks | Circuit breakers (stop calling after N failures), timeouts (always set them), retries with backoff (only if idempotent). |
| Message queue | Consumer crashes mid-processing | At-least-once delivery + idempotent processing. The consumer must handle re-delivery without side effects. |
| Streaming pipeline | Processing node crashes, loses in-flight state | Checkpointing. Periodically snapshot processing state. On crash, replay from last checkpoint. Flink does this automatically. |

**The idempotency principle:** If any component can fail and retry, every operation in the retry path must be idempotent. This is not optional. Design idempotency keys into APIs from day one. Stripe assigns an idempotency key to every payment API call; retrying the same key returns the original result without re-executing the payment.

---

## Step 4: Technology Choices

Technology is the last decision, not the first. By this point, the architecture has been determined by the constraints. Technology selection is now a matter of picking tools that fit the architecture, not tools that impose one.

### Selection criteria, in order of importance

1. **Operational maturity**: Can your team run this in production? A technology you understand and can debug at 3 AM beats a theoretically superior one you've never operated. PostgreSQL with read replicas will outperform a cutting-edge distributed database you don't understand — not in benchmarks, but in production reliability.

2. **Fit to the derived architecture**: Does it support the computation model, state management, and communication patterns you've chosen? Don't pick a tool and then contort the architecture to fit it.

3. **Ecosystem and longevity**: Is it actively maintained? Are there escape hatches? Can you migrate away if needed? This matters more than features.

4. **Performance at your scale**: Note — *at your scale*. A tool that handles 1 million events/sec is not better than one that handles 10,000/sec if you need 500/sec. Over-provisioning capability is not free; it comes with operational complexity.

### Common mappings (not prescriptions)

| Architectural need | Typical tools | When to deviate |
|--------------------|---------------|-----------------|
| Single relational database | PostgreSQL | Almost never. Postgres covers an extraordinary range of workloads. |
| Document store for flexible schema | MongoDB, DynamoDB | Postgres JSONB columns often suffice; use a document store when you truly need document-level atomicity at scale |
| In-memory cache | Redis, Memcached | If your read latency target is met by the database, you don't need a cache |
| Message queue | SQS, RabbitMQ | SQS if you're on AWS and want zero ops; RabbitMQ if you need routing features |
| Event log / streaming | Kafka, Kinesis, Redpanda | Only if you need durable, replayable, multi-consumer event streams. If you have one consumer and no replay need, use SQS. |
| Stream processing | Flink, Kafka Streams | Only when continuous computation over unbounded data is a real requirement |
| Batch processing | Spark, dbt, plain SQL | dbt on top of your warehouse if the transformations are SQL-expressible; Spark only when data volume forces it |
| Real-time client push | WebSockets via your framework, or a managed service | Don't build your own pub/sub for browser push when services like Ably or Pusher exist, unless scale demands it |
| Search | Elasticsearch, OpenSearch, Postgres full-text | Postgres full-text search is good enough for most applications. Elasticsearch when you need faceting, fuzzy matching, or relevance tuning at scale |
| Object / blob storage | S3, GCS, MinIO | This is commodity infrastructure. Pick your cloud provider's option. |

---

## Patterns as Emergent Outcomes

Architectural patterns are not a menu. They are outcomes that fall out of specific constraint combinations. Here is how to recognize when each pattern is forced by the constraints — and when it's over-engineering.

### Stateless Request-Response

**Emerges when:** No dominant constraints. Moderate traffic. Single-digit second freshness is fine. Standard CRUD operations.

**What it looks like:** Web framework + relational database. Horizontal scaling by adding stateless application servers behind a load balancer.

**Real example:** Basecamp runs a large portion of its product on a Rails monolith with a few Postgres and MySQL databases. The workload does not demand more.

**Over-engineering signal:** Adding a caching layer, message queue, or microservice decomposition to a stateless CRUD app that handles < 1,000 requests/sec.

### CQRS (Command Query Responsibility Segregation)

**Emerges when:** Read and write patterns are fundamentally different. Consumers need data in shapes that are expensive to derive from the write-optimized schema at read time.

**What it looks like:** One model for writes (normalized, optimized for consistency), separate model(s) for reads (denormalized, optimized for query patterns). Write-side publishes events; read-side subscribes and updates materialized views.

**Real example:** A marketplace where sellers manage inventory (write side: normalized product catalog) and buyers search and filter products (read side: denormalized search index with pre-aggregated facets). Etsy's search infrastructure works like this — the product catalog is authoritative, but search is served from a separately built index.

**Over-engineering signal:** Using CQRS when read and write models are nearly identical. If your read queries can efficiently hit the same schema your writes use, the read/write split adds operational cost (keeping views in sync) with no benefit.

### Event Sourcing

**Emerges when:** The history of changes is as important as the current state. The domain is naturally a sequence of events. Audit trails are a regulatory requirement. You need to replay events to build new views retroactively.

**What it looks like:** The append-only event log is the source of truth. Current state is derived by replaying events or maintained as a projection that's updated on each new event.

**Real example:** Accounting and financial systems. A bank account's state is the result of applying every transaction in order. The event log (the ledger) is the system of record, not the current balance. This isn't a pattern choice — it's how the domain works.

**Over-engineering signal:** Event sourcing for a user profile service. If you just need the current state and have no regulatory or business need for the full history, event sourcing adds schema evolution complexity, snapshot management, and eventual consistency overhead for no gain. A mutable row in a database is the right model.

### Actor Model

**Emerges when:** The system manages many independent entities that each have mutable state and receive concurrent messages. The entities have non-trivial lifecycles.

**What it looks like:** Each entity is an actor with a mailbox. Messages are processed sequentially per actor, eliminating concurrency bugs within an entity. Actors communicate asynchronously.

**Real example:** Online games (each game session is an actor), IoT device management (each device is an actor), telephony (each call is an actor — Erlang was built for this). Orleans at Microsoft uses virtual actors for Halo's game services: each game, player, and match is a virtual actor that's activated on demand and deactivated when idle.

**Over-engineering signal:** Using actors for stateless request processing. If your "actors" don't have meaningful state between messages, they're just function calls with extra indirection.

### Data Parallelism / MapReduce

**Emerges when:** The computation is embarrassingly parallel — each item can be processed independently. The data volume is large. Latency tolerance is high (minutes to hours).

**What it looks like:** Partition the data, process each partition independently, combine results. Classic MapReduce, Spark jobs, or even `xargs -P` on a single machine.

**Real example:** Building a search index over billions of web pages (Google's original MapReduce use case). Each page is processed independently (map), then results are combined into an inverted index (reduce).

**Over-engineering signal:** Using Spark to process 100 MB of data. Your laptop can do this in seconds with a Python script.

### Pipeline Architecture

**Emerges when:** Data must flow through multiple transformation stages. Each stage has different scaling needs. Stages should be independently deployable and testable.

**What it looks like:** Directed acyclic graph of processing stages connected by queues or streams. Each stage reads from an input, transforms, writes to an output.

**Real example:** Datadog's metric processing pipeline: agents send metrics → ingestion tier normalizes and routes → aggregation tier computes rollups → storage tier writes to time-series database → query tier serves dashboards. Each stage scales independently because the throughput bottleneck shifts between them.

**Over-engineering signal:** Building a pipeline for a single transformation that runs once a day. A cron job that runs a script is the correct architecture.

### Pub/Sub and Event-Driven Architecture

**Emerges when:** Multiple independent consumers need to react to the same events. Producers should not know about consumers. New consumers may be added without modifying producers.

**What it looks like:** Events are published to topics. Consumers subscribe to topics they care about. The event bus decouples producers from consumers.

**Real example:** When a user signs up for GitHub, multiple systems react: a welcome email is sent, a usage analytics event is recorded, a free-tier resource quota is provisioned, the onboarding flow is triggered. The signup service publishes a single `user.created` event; each downstream system subscribes independently.

**Over-engineering signal:** Pub/sub between two services where one always calls the other. A direct function call or HTTP request is simpler. Pub/sub's value is decoupling *many* consumers; for one consumer, it adds indirection without benefit.

---

## The Over-Engineering Checklist

Before finalizing an architecture, run this checklist. Each "yes" is a warning that you may be adding complexity without a forcing constraint.

| Question | If yes... |
|----------|-----------|
| Could a single Postgres instance handle this workload for the next 2 years? | You probably don't need a distributed database, sharding, or NoSQL. |
| Is the total dataset under 100 GB? | You probably don't need distributed storage. |
| Is the peak traffic under 1,000 requests/sec? | You probably don't need a caching layer, CDN (for API responses), or horizontal write scaling. |
| Is the freshness requirement > 5 seconds? | You probably don't need WebSockets, streaming, or push infrastructure. |
| Is there only one consumer of each data set? | You probably don't need CQRS, event sourcing, or a message bus. |
| Is the team fewer than 10 engineers? | You probably don't need microservices. A monolith with clear module boundaries is cheaper to develop, deploy, test, and debug. |
| Can the entire system be down for 5 minutes without material business impact? | You probably don't need multi-region, active-active, or sophisticated failover. |
| Is the data model simple and stable? | You probably don't need a schema registry, event versioning, or a flexible document store. |

Every "probably don't need" is a decision **not** to add a component. Each component you don't add is a component you don't have to operate, monitor, debug at 3 AM, or explain to the next engineer.

---

## Putting It Together: A Worked Example

**Problem: Design a system for a mid-size e-commerce company's order processing.**

### Step 1: Characterize

- **Producers**: Customers placing orders, ~50 orders/minute at peak (Black Friday). ~1 KB per order payload.
- **Consumers**: Warehouse fulfillment system (needs orders to pick and pack), finance system (needs to record revenue), customer-facing order status page, analytics dashboard.
- **Data shape**: Orders are append-mostly (created, then status changes). Immutable line items. ~500K orders/year. Total dataset ~5 GB after a few years.
- **Freshness**: Customer status page should update within 30 seconds. Warehouse can tolerate 1-2 minutes. Finance can tolerate hours. Analytics can tolerate a day.
- **Consistency**: An order must not be lost. Double-processing an order (shipping twice) would be costly.
- **Availability**: An outage during checkout means lost revenue. An outage in the dashboard is annoying but not critical.

### Step 2: Identify dominant constraints

- Write rate: ~1/sec at peak. **Not a constraint.**
- Read rate: Trivial. **Not a constraint.**
- Data size: 5 GB. **Not a constraint.**
- Freshness: 30 seconds for order status. **Not a constraint** (simple polling or a 30-second cache TTL works).
- Consistency: Orders must not be lost or double-processed. **This is a dominant constraint.** It forces at-least-once delivery with idempotent processing.
- Consumer diversity: 4 consumers with different latency needs. **This is a mild constraint.** It suggests a pattern where order events are published and consumed independently, but the low volume means this could also be done with direct database queries.

Dominant constraints: **durability/consistency of order processing** and **mild consumer diversity**.

### Step 3: Derive architecture

- **State**: Single PostgreSQL database. Orders table, line items table, order status history table. 5 GB fits comfortably on one machine for the next decade.
- **Computation**: Synchronous request-response for order placement (customer is waiting). Async workers for downstream processing (warehouse notification, finance recording).
- **Communication**: Order placement writes to the database and publishes an event to a simple queue (SQS or a Postgres-backed queue like `LISTEN/NOTIFY` with a polling fallback). Warehouse system, finance system, and analytics each consume from the queue. The order status page queries the database directly.
- **Failure handling**: Order placement is wrapped in a database transaction. The event publication uses the transactional outbox pattern (write the event to an outbox table in the same transaction as the order; a separate process reads the outbox and publishes to the queue). This ensures an order is never created without its event being published. Each consumer is idempotent using the order ID as the idempotency key.

### Step 4: Technology

- **Database**: PostgreSQL.
- **Queue**: Amazon SQS (if on AWS) or a Postgres-based queue (if staying simple). Not Kafka — there's no need for replay, log compaction, or partitioned ordering at 1 event/sec.
- **Application**: A single web application in whatever framework the team knows. Not microservices — there's no scaling or organizational reason to split this up.
- **Monitoring**: Standard application metrics. Alert on queue depth (are consumers falling behind?) and on failed order writes.

Total component count: 1 application server, 1 database, 1 queue, a few consumer workers. This handles the full workload. Adding anything else (Redis, Elasticsearch, a separate analytics database, a service mesh) would require a specific justification that traces back to a constraint — and none does.

---

## Summary

The framework in three sentences:

**Measure the workload before designing anything.** Most systems have one or two properties that force real decisions; everything else should be as simple as possible. An architecture you can't trace back to a measured constraint is an architecture you don't need.
