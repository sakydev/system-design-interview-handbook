# Content Plan

The source of truth for what every page in the handbook covers. 27 concept pages, 8 case studies, 8 nav groups.

## Page conventions

Every page follows the pattern established by `design-process.html`:

- Hero with title, one-line description, and 4-6 pills naming the page's key terms.
- Shared sidebar nav (left), content sections (center), "On this page" TOC (right).
- Content is broken into numbered `<section>` blocks, each with an icon, title, and section tag.
- At least one ASCII diagram per page in a `.diagram` block. Diagrams show data flow or structure, not decoration.
- Callouts: `tip` (💡) for interview-specific advice, `warn` (⚠️) for common mistakes, plain (ℹ️) for context.
- Reference tables (`.ref-table`) for anything comparative: option A vs B, signal vs response.
- Every page ends with a Quick Reference section: one table that compresses the whole page into a cheat sheet.
- Every concept page includes at least one "in the interview" callout: when this topic comes up, what the interviewer is probing, what a strong answer sounds like.
- Cross-link related pages inline (e.g. Replication links to Consistency Models and Consensus).

Tone: direct, opinionated, no filler. Explain the tradeoff behind every mechanism. The reader is preparing for an interview, not writing a thesis; depth stops where interview usefulness stops.

---

## 1. Foundations

*How to think about a design before you draw a single box.*

### 1.1 Design Process (`design-process.html`) — DONE

The interview framework itself. Already written: 5 stages with time budget, clarifying requirements, identifying the bottleneck, sketching high-level design, naming tradeoffs, enumerating failure modes, common mistakes, stage-by-stage cheat sheet.

### 1.2 Estimation (`estimation.html`)

Back-of-envelope math. The goal: turn "1M daily users" into QPS, storage, and bandwidth numbers in under five minutes.

- **Numbers to memorize.** Powers of two, latency numbers (L1 cache through cross-region round trip), throughput of one server / one SSD / one DB node, seconds per day (~86,400, round to 100K).
- **The QPS derivation.** DAU → requests per user per day → average QPS → peak QPS (2-5x multiplier). Worked example end to end.
- **Storage estimation.** Object size × write rate × retention. Metadata vs payload. Replication multiplier (usually 3x). Worked example: tweets for a year.
- **Bandwidth and memory.** Ingress vs egress. Cache sizing via the 80/20 rule: 20% of daily data serves 80% of reads.
- **Rounding discipline.** Round aggressively, keep units visible, sanity-check the result against a known system. Precision theater wastes interview minutes.
- **Quick Reference:** one table of the memorized numbers, one table of the standard derivation formulas.

### 1.3 Consistency Models (`consistency-models.html`)

What "consistent" means and what each level costs.

- **CAP theorem.** What it says, what it doesn't. Partition tolerance isn't optional, so the real choice is C vs A during a partition. Common misreadings.
- **PACELC.** The extension that matters in practice: even without partitions, you trade latency vs consistency.
- **The consistency spectrum.** Strong / linearizable → sequential → causal → eventual. One concrete anomaly per level (e.g. read-your-writes violation: user posts a comment, refreshes, comment is gone).
- **Session guarantees.** Read-your-writes, monotonic reads, monotonic writes. Cheap to provide, often all a product needs.
- **ACID vs BASE.** What each letter means, where each model fits. ACID within one node is cheap; across nodes it's expensive (links to Distributed Transactions).
- **Choosing in an interview.** Map product requirements to a model: money needs strong, social feeds tolerate eventual, collaborative editing needs causal.
- **Quick Reference:** table of models × guarantee × example system × example anomaly avoided.

### 1.4 Tradeoffs (`tradeoffs.html`)

A reference page for the cost behind every decision. Short sections, heavily tabular.

- **Why tradeoffs are the grading rubric.** Interviewers grade the "I chose X, it costs Y, worth it because Z" sentence, not the choice itself.
- **The core axes.** Latency vs consistency. Throughput vs latency. Availability vs consistency. Cost vs durability. Simplicity vs flexibility. Build vs buy. One short section each with a concrete example.
- **Read vs write optimization.** Denormalization, materialized views, fan-out-on-write vs fan-out-on-read as the canonical example.
- **The complexity budget.** Every component added must be justified by a requirement. How to decide when the simple version fails.
- **Quick Reference:** the "you choose / you pay with" table from the Design Process page, expanded to ~15 rows.

---

## 2. Networking & Communication

*How data moves and how services find and talk to each other.*

### 2.1 Networking Basics (`networking-basics.html`)

What happens between the browser and your load balancer. Foundation for everything else.

- **Anatomy of a request.** DNS resolution → TCP handshake → TLS handshake → HTTP request → response. Where latency accrues at each step.
- **DNS.** Recursive vs authoritative resolution, TTLs, DNS-based load balancing and its staleness problem, GeoDNS for routing users to the nearest region.
- **TCP vs UDP.** Guarantees each gives, cost of those guarantees, when UDP wins (video, gaming, QUIC).
- **HTTP versions.** 1.1 (head-of-line blocking, keep-alive), HTTP/2 (multiplexing), HTTP/3/QUIC (UDP-based, faster handshakes). What each fixed.
- **TLS.** Handshake cost, session resumption, TLS termination at the load balancer and what that implies about internal traffic.
- **CDNs and edge delivery.** How a CDN works (edge PoPs, origin pull vs push), what belongs on a CDN, cache keys and invalidation, signed URLs for private content. This is CDN's home page; Caching cross-references it.
- **Quick Reference:** request-path latency table (DNS, TCP, TLS, TTFB), TCP vs UDP table, HTTP version comparison table.

### 2.2 APIs & Protocols (`apis.html`)

Designing the contract between clients and services.

- **REST.** Resources, verbs, status codes, statelessness. Why it's the default. Where it creaks (chatty clients, over/under-fetching).
- **gRPC.** Protobuf, HTTP/2 transport, streaming modes. When it wins: internal service-to-service, low latency, strict contracts. Browser limitations.
- **GraphQL.** One endpoint, client-shaped queries. The N+1 problem, caching difficulty, when it's worth it (many client types, aggregation layers).
- **Request-response vs event-driven.** Sync coupling vs async decoupling. When to pick each; links to Messaging.
- **API design details that come up.** Pagination (offset vs cursor, why cursor wins at scale), versioning strategies, idempotency keys for safe retries on mutations, rate limit headers.
- **Quick Reference:** REST vs gRPC vs GraphQL table (transport, contract, caching, browser support, best fit), pagination comparison table.

### 2.3 Real-Time Delivery (`realtime.html`)

Pushing data to clients instead of waiting for them to ask.

- **The options ladder.** Short polling → long polling → Server-Sent Events → WebSockets. How each works, one diagram covering all four.
- **Choosing between them.** Update frequency, directionality (server-push only vs bidirectional), proxy/firewall friendliness, client complexity. SSE is underused; WebSockets aren't always the answer.
- **Connection state is the hard part.** Persistent connections make servers stateful: sticky routing or a connection registry, heartbeats and dead-connection detection, reconnect with backoff and resume (last-event-id, message replay).
- **Scaling to a million connections.** Memory per connection, connection gateways as a dedicated tier, pub/sub backbone (e.g. Redis, Kafka) fanning out to gateway nodes, draining connections on deploy.
- **Delivery semantics over flaky links.** Client acks, server-side message buffers, ordering per channel. Links to Messaging and Chat System case study.
- **Quick Reference:** table of the four techniques × latency × direction × cost × best fit.

### 2.4 Messaging (`messaging.html`)

Decoupling producers from consumers.

- **Why async at all.** Absorbing spikes, decoupling failure domains, smoothing write load. The cost: eventual consistency, harder debugging (links to Tradeoffs).
- **Message queues.** Point-to-point, competing consumers, work distribution. SQS/RabbitMQ model. Visibility timeouts, dead-letter queues.
- **Pub/sub.** Fan-out to many subscribers, topics, ephemeral vs durable subscriptions.
- **Event streaming / log-based.** Kafka model: partitioned append-only log, consumer groups, offsets, retention, replay. Why partition count bounds parallelism. Ordering guaranteed per partition only.
- **Delivery guarantees.** At-most-once, at-least-once, exactly-once. Why exactly-once is really at-least-once plus idempotent consumers (links to Concurrency Control).
- **Back-pressure.** What happens when consumers fall behind: bounded queues, load shedding, producer throttling. Lag as the key metric.
- **Quick Reference:** queue vs pub/sub vs stream table (delivery, ordering, replay, consumer model, canonical tool, best fit).

### 2.5 Load Balancing & Discovery (`service-discovery.html`)

Getting requests to a healthy instance.

- **Load balancing algorithms.** Round robin, weighted, least connections, least response time, IP/consistent hash (session affinity, cache locality). When each matters.
- **L4 vs L7.** What each layer sees, what each can do (L7: routing by path/header, TLS termination, retries), cost difference.
- **Health checks.** Active vs passive, shallow vs deep, the danger of a deep health check taking out a whole fleet.
- **Service discovery.** Client-side vs server-side discovery, registries (Consul, etcd, DNS-based), how instances register and deregister, what happens when the registry is stale.
- **API gateway.** Edge concerns in one place: authn, rate limiting, routing, request shaping. Gateway vs load balancer vs reverse proxy, and why the terms blur.
- **Global load balancing.** GeoDNS and anycast, routing users to regions (links to Multi-Region).
- **Quick Reference:** algorithm table (how it works, best for), L4 vs L7 table.

---

## 3. Data & Storage

*Choosing and shaping where the data lives.*

### 3.1 SQL vs NoSQL (`sql-vs-nosql.html`)

The real tradeoffs, not the mythology.

- **What relational actually buys.** Schema as enforced contract, joins, transactions, mature query planners. Why "SQL doesn't scale" is mostly false below huge scale.
- **The NoSQL families.** Key-value (Redis, DynamoDB), document (MongoDB), wide-column (Cassandra), graph (Neptune, Neo4j — pointer to Specialized Stores). One paragraph each: data model, access pattern it optimizes, what it gives up.
- **The actual decision criteria.** Access patterns known up front vs ad hoc queries. Need for multi-row transactions. Scale of writes. Schema flexibility as benefit vs liability.
- **Modeling differences.** Normalize-then-join vs denormalize-and-duplicate. Query-first design in NoSQL: you model around access patterns, not entities.
- **The middle ground.** Postgres JSONB, NewSQL (Spanner, CockroachDB): distributed SQL exists and interviewers know it.
- **Quick Reference:** decision table (requirement → leans SQL / leans NoSQL / either), family comparison table.

### 3.2 Indexing & Storage Engines (`indexing.html`)

How databases find rows and why write-heavy stores are built differently.

- **B-tree indexes.** Structure, why lookups are O(log n), range scans, cost of each additional index on write throughput.
- **LSM trees.** Memtable → SSTables → compaction. Why writes are fast (sequential I/O), why reads pay (check multiple levels), how Bloom filters cut that read cost. This is why Cassandra/RocksDB/LevelDB exist.
- **B-tree vs LSM.** Read-optimized vs write-optimized, write amplification vs read amplification vs space amplification.
- **Index strategy.** Composite indexes and column order, covering indexes (index-only reads), partial indexes, when the query planner ignores your index (leading wildcards, functions on columns, low selectivity).
- **The write cost of indexes.** Every index is a second copy maintained on every write. Indexing everything is a write-heavy anti-pattern.
- **Quick Reference:** B-tree vs LSM table, index type table (type, what it speeds up, what it costs).

### 3.3 Sharding & Partitioning (`sharding-partitioning.html`)

Splitting data across nodes when one node isn't enough.

- **Vertical vs horizontal partitioning.** Splitting by column/table vs by row. Sharding = horizontal across machines.
- **Choosing a shard key.** The single most important decision: even distribution, aligned with the dominant query pattern. Cross-shard queries as the tax for choosing wrong.
- **Hash vs range sharding.** Hash: even spread, no range queries. Range: locality and range scans, hot spots on sequential keys (timestamps, auto-increment).
- **Consistent hashing.** The rehash-everything problem with naive modulo, the ring, virtual nodes for even spread, how it limits data movement to 1/n on membership change. Diagram required.
- **Hot spots.** Celebrity keys, key salting, splitting a hot shard, dedicated capacity for known-hot keys.
- **Rebalancing.** Fixed partition count vs dynamic splitting, moving shards without downtime.
- **Unique ID generation.** Why auto-increment breaks under sharding. UUIDs (v4 randomness kills B-tree locality, v7 fixes it), Snowflake IDs (timestamp + machine + sequence: sortable, decentralized), ticket servers. The URL Shortener case study exercises this.
- **Quick Reference:** hash vs range table, ID scheme comparison table (scheme, sortable?, coordination?, size, gotcha).

### 3.4 Replication (`replication.html`)

Copies of data for durability and read scale.

- **Why replicate.** Durability, read scaling, geographic locality, failover. Not the same problem as sharding; most real systems do both.
- **Leader-follower.** Write to leader, replicate to followers, read from anywhere. Sync vs async replication and the durability/latency trade. Replication lag and the anomalies it causes (links to Consistency Models session guarantees).
- **Failover.** Detecting leader death, promoting a follower, split brain, why async replication can lose acknowledged writes during failover.
- **Multi-leader.** Multi-region writes, write conflicts, conflict resolution (last-write-wins and its data loss, CRDTs in one paragraph).
- **Leaderless / quorum.** Dynamo-style: W + R > N, sloppy quorums, hinted handoff, read repair and anti-entropy.
- **Write-ahead logs.** The durability primitive underneath all of it: log first, apply second. Also the replication stream and the basis of CDC (links to Distributed Transactions).
- **Quick Reference:** leader-follower vs multi-leader vs leaderless table (write path, conflict risk, consistency, best fit), quorum cheat sheet (N/W/R combinations and what they give).

### 3.5 Specialized Stores (`specialized-stores.html`)

The right store for workloads that punish general-purpose databases.

- **Time series.** Write-heavy, append-only, time-bucketed queries. Compression via delta encoding, downsampling and retention tiers. InfluxDB, Timescale, Prometheus's model.
- **Columnar / OLAP.** Row vs column layout, why analytics scans want columns, compression benefits. OLTP vs OLAP as the real split; warehouses (BigQuery, Redshift, ClickHouse) vs the operational DB.
- **Graph.** When relationship traversal depth kills relational joins (friends-of-friends). Property graph model, adjacency locality. When a graph DB is overkill (most of the time).
- **Blob / object storage.** S3 model: flat namespace, HTTP access, 11-nines durability, no filesystem semantics. Storing metadata in a DB and payload in object storage as the standard pattern. Presigned URLs for direct upload/download so payload bypasses your servers. Multipart upload for large files.
- **Geospatial indexing.** Why B-trees fail on 2D nearest-neighbor. Geohash (prefix = proximity, cell-boundary problem), quadtrees (adaptive density), S2/H3 in one paragraph. Backing store options (Redis GEO, PostGIS). The Ride Sharing case study exercises this.
- **Quick Reference:** table of store type × workload shape × canonical tools × the tell in an interview question ("per-second metrics" → time series, "nearby" → geospatial).

### 3.6 Caching (`caching.html`)

The highest-leverage scaling tool and the bugs it invites.

- **Where caches live.** Browser → CDN (links to Networking Basics) → gateway/edge → application cache (Redis/Memcached) → DB internal caches. A read's journey through all layers, one diagram.
- **Cache-aside.** The default pattern: check cache, miss → read DB → populate. Who owns population, what happens on failure.
- **Write strategies.** Write-through (consistency, slower writes), write-behind (fast writes, loss risk on crash), write-around. When each fits.
- **Eviction.** LRU, LFU, TTL. Sizing the cache (links to Estimation's 80/20 rule).
- **Invalidation.** TTL-based staleness vs explicit invalidation on write vs event-driven invalidation via CDC. Why "there are only two hard things" is earned.
- **Failure modes.** Cache stampede / thundering herd (locking, request coalescing, probabilistic early refresh), hot keys (local caching, key replication), cache penetration on nonexistent keys (negative caching, Bloom filter), avalanche on cache-tier restart (jittered TTLs, warming).
- **Quick Reference:** write-strategy table, failure mode → mitigation table.

---

## 4. Data Processing

*Turning raw event streams into answers.*

### 4.1 Batch & Stream Processing (`batch-stream-processing.html`)

The page behind "design ad click aggregation", "design trending topics", "design a metrics pipeline".

- **The two paradigms.** Batch: bounded input, high throughput, high latency (hours). Stream: unbounded input, results in seconds. Most real systems need both.
- **Batch fundamentals.** MapReduce in one diagram (map → shuffle → reduce), why shuffle is the expensive part, modern engines (Spark) keeping the model. ETL vs ELT in one paragraph.
- **Stream fundamentals.** Events flowing through processing topologies (Flink, Kafka Streams model). Stateless vs stateful operators.
- **Windowing.** Tumbling, sliding, session windows. Event time vs processing time, out-of-order events, watermarks as the "we've probably seen everything before T" signal.
- **Exactly-once processing.** Checkpointing, replayable sources, idempotent or transactional sinks. What "exactly-once" claims actually mean (links to Messaging, Concurrency Control).
- **Lambda and kappa architectures.** Batch layer + speed layer vs stream-only with replay. Why kappa is winning.
- **Top-k and counting at scale.** The trending-topics shape: count-min sketch for frequencies, HyperLogLog for distinct counts, heavy hitters per window, merging partial results across partitions. Exact counting is unaffordable; say so and name the sketch.
- **Quick Reference:** batch vs stream table, window type table, question-shape → technique table ("trending" → count-min + windows, "unique visitors" → HyperLogLog, "daily report" → batch).

### 4.2 Search (`search.html`)

Full-text search and the typeahead problem.

- **Why not `LIKE '%term%'`.** Full scans, no relevance, no tokenization. The moment a product needs search, it needs an index built for it.
- **Inverted indexes.** Term → posting list of document IDs. How one is built: tokenize → normalize (lowercase, stemming, stop words) → index. Diagram of documents becoming an inverted index.
- **Query side.** AND/OR via posting-list intersection/union, phrase queries via positions.
- **Relevance.** TF-IDF intuition (rare terms matter more), BM25 as the default, ranking signals beyond text (recency, popularity, personalization) in one paragraph.
- **Search infrastructure.** Elasticsearch/Lucene model: documents, shards, near-real-time indexing (refresh interval = staleness window). Keeping the search index in sync with the source-of-truth DB via CDC or dual-write, and why dual-write drifts.
- **Typeahead / autocomplete.** A different problem: prefix matching under ~100ms. Tries with precomputed top-k per node, weighting by popularity, fuzzy matching in one paragraph. Precompute-heavy, read-cheap. Feeds the Typeahead case study.
- **Quick Reference:** DB query vs search engine table, search vs typeahead comparison (index structure, latency budget, update pattern).

---

## 5. Reliability

*Patterns for surviving failure instead of pretending it won't happen.*

### 5.1 Resilience Patterns (`resilience-patterns.html`)

Keeping one failure from becoming an outage.

- **Failure is the default.** At scale, something is always broken. Design goal: contain, degrade, recover.
- **Timeouts.** Every remote call needs one. Picking budgets, propagating deadlines through call chains, why a missing timeout turns slow into down.
- **Retries.** Only on transient + idempotent operations. Exponential backoff, jitter (why synchronized retries create waves), retry budgets, retry storms and amplification through layered retries.
- **Circuit breaker.** Closed → open → half-open state machine, diagram. Failing fast to protect both caller and struggling callee.
- **Bulkheads.** Isolating resource pools (connections, threads, instances) per dependency so one slow dependency can't drain everything.
- **Load shedding & graceful degradation.** Dropping low-priority work under stress, serving stale/partial results vs failing hard, feature flags as kill switches.
- **Cascading failure anatomy.** One worked chain: slow DB → thread pool exhaustion → healthy service marked unhealthy → traffic reshuffles → next node tips over. How each pattern above breaks the chain.
- **Quick Reference:** pattern table (pattern, protects against, key parameter, gotcha).

### 5.2 Distributed Transactions (`distributed-transactions.html`)

Keeping multiple systems in agreement without a shared transaction.

- **The problem.** One business action, multiple stores/services. Local ACID doesn't stretch across the network.
- **Two-phase commit.** Prepare + commit, coordinator diagram. Why it's rare in practice: blocking on coordinator failure, latency, participants holding locks.
- **Sagas.** Sequence of local transactions + compensating actions. Choreography vs orchestration, diagram of a payment saga with a compensation path. Designing compensations (semantic undo, not rollback).
- **The outbox pattern.** The dual-write problem (DB commit + event publish can't be atomic), writing events to an outbox table in the same local transaction, relaying to the broker. The default answer to "how do you reliably publish events".
- **Change data capture.** Tailing the WAL (Debezium model) to turn the database into an event source. Outbox vs raw CDC.
- **Idempotency as the glue.** Every step must tolerate replay (links to Concurrency Control).
- **Quick Reference:** 2PC vs saga vs outbox table (consistency, latency, failure behavior, complexity, use when).

### 5.3 Concurrency Control (`concurrency-control.html`)

Correctness when the same data is touched twice at once.

- **Where races come from.** Retries, concurrent users, at-least-once delivery, multi-node writes. Interview tell: "what if this request arrives twice?"
- **Idempotency.** Natural idempotency (sets, upserts) vs idempotency keys (client-generated key, server dedup store, TTL). Designing the dedup store.
- **Delivery semantics.** At-most-once / at-least-once / exactly-once recap from the consumer's seat: exactly-once effect = at-least-once delivery + idempotent processing.
- **Optimistic concurrency.** Version columns / compare-and-swap: read v, write if still v, retry on conflict. Wins when conflicts are rare.
- **Pessimistic locking.** Row locks, `SELECT FOR UPDATE`. Wins when conflicts are common (inventory, seat booking). Deadlock risk.
- **Distributed locks.** Lock service on Redis/ZooKeeper/etcd, lease TTLs, the expired-lease-but-still-running problem, fencing tokens as the fix. Why "just use a Redis lock" needs caveats.
- **Logical clocks.** Why wall clocks can't order events across machines (skew). Lamport clocks (ordering), vector clocks (detecting concurrency, Dynamo-style conflict detection). Intuition level, not proofs.
- **Quick Reference:** technique table (technique, use when, cost), idempotency-key flow diagram.

### 5.4 Consensus (`consensus.html`)

How machines agree on one value when any of them can fail.

- **Why consensus is needed.** Leader election, distributed locks, cluster membership, config. Anywhere "exactly one" must be true across machines.
- **Why it's hard.** Asynchronous networks: can't distinguish slow from dead. FLP in one sentence, quorums as the practical answer.
- **Quorums.** Majority overlap as the core trick: any two majorities share a node, so two leaders can't both win. Why clusters are 3/5/7 nodes.
- **Raft.** The one to know: terms, leader election (randomized timeouts), log replication, commit on majority ack. Two diagrams: election, log replication. What happens on leader crash, network partition, split vote.
- **Paxos.** One paragraph: earlier, equivalent power, notoriously hard; Raft was designed to be teachable. Name Multi-Paxos, move on.
- **Consensus in the wild.** ZooKeeper/etcd/Consul as consensus-as-a-service; DBs using Raft internally (CockroachDB, etcd, Kafka KRaft). The pattern: keep the consensus cluster small, put coordination there, keep data on the big fleet.
- **What consensus costs.** Round trips and quorum waits: low throughput. Never put high-volume data through it (links to Replication for data-plane alternatives).
- **Quick Reference:** "coordination need → tool" table, Raft state cheat sheet.

---

## 6. Architecture & Scale

*Growing a system without it falling over.*

### 6.1 Microservices (`microservices.html`)

When to split a system and when splitting is the wrong call.

- **Monolith first.** Why the default should be a well-structured monolith: one deploy, local calls, real transactions. Most systems die from complexity, not scale.
- **What microservices actually buy.** Independent deploys, independent scaling, team autonomy, fault isolation. Note that the driver is usually organizational (Conway's law), not technical.
- **What they cost.** Network calls replace function calls (latency, partial failure), distributed transactions (links to Sagas), operational surface (deploys, observability, versioned contracts), local dev pain.
- **Drawing boundaries.** Around business capabilities, not entities. Database-per-service and why shared databases recouple everything. Getting data across boundaries: APIs, events, read replicas of other services' data.
- **Decomposition path.** Strangler fig: route by route, feature by feature. Never big-bang rewrites.
- **In the interview.** Don't default to microservices. "I'd start with a modular monolith and split X out when Y" is a senior answer.
- **Quick Reference:** monolith vs microservices table, "signals you're ready to split" checklist.

### 6.2 Scaling Patterns (`scaling-patterns.html`)

The standard moves, in the order you reach for them.

- **Vertical vs horizontal.** Buy a bigger box vs add boxes. Vertical is underrated early; the ceiling and single-point-of-failure problem.
- **Statelessness.** The property that makes horizontal scaling trivial. Moving state out: sessions to Redis/JWT, files to object storage, local caches acknowledged as best-effort.
- **The scaling sequence.** A narrative: one server → separate DB → load balancer + N app servers → cache layer → read replicas → async work to queues → shard. Each step named with the bottleneck that forces it (links to Estimation, Sharding, Caching, Messaging).
- **Autoscaling.** Metric-driven scale-out, warm-up time, why scale-in is the dangerous direction, over-provisioning headroom.
- **Rate limiting & throttling.** Protecting the system from its clients. Token bucket, leaky bucket, fixed window, sliding window log/counter: how each works, burst behavior. Where to enforce (gateway). Feeds the Rate Limiter case study.
- **Quick Reference:** scaling sequence table (bottleneck → move), rate-limit algorithm comparison table.

### 6.3 Multi-Region (`multi-region.html`)

What breaks when you leave one data center.

- **Why go multi-region.** Latency to global users, disaster recovery, data residency law. Each implies a different design; pin down which one is driving.
- **Active-passive.** Standby region, failover promotion. RTO/RPO defined, replication lag as data-loss window, why failover that's never tested doesn't work.
- **Active-active.** Users served from nearest region. The write problem: single global write region vs region-local writes with conflict handling (links to Multi-Leader Replication) vs partitioning users by home region.
- **Data residency.** GDPR-style constraints: EU data stays in EU. Regional partitioning with global metadata, and what it does to "show me all users" queries.
- **Routing users.** GeoDNS, anycast, what happens to in-flight sessions when a region dies.
- **The cost.** Cross-region latency (~100ms+) makes synchronous cross-region calls poison. Design rule: regions are islands that sync asynchronously.
- **Quick Reference:** active-passive vs active-active table (RTO/RPO, cost, write handling, complexity), region checklist.

---

## 7. Operations

*Running the system in production.*

### 7.1 Metrics, Logs & Traces (`metrics-logs-traces.html`)

The three signals and what each is for.

- **Three signals, three questions.** Metrics: "is something wrong?" (cheap, aggregated, alertable). Logs: "what exactly happened?" (rich, expensive, searchable). Traces: "where in the chain?" (request-scoped, cross-service).
- **Metrics.** Counters, gauges, histograms. The four golden signals (latency, traffic, errors, saturation). Percentiles over averages: why p99 matters and average lies. Cardinality as the thing that kills metrics systems.
- **Logs.** Structured logging, centralized aggregation, sampling under volume, correlation IDs tying a request's logs together.
- **Distributed tracing.** Spans, trace context propagation across service and queue boundaries, OpenTelemetry as the standard, sampling strategies (head vs tail).
- **In the interview.** One sentence at wrap-up: "I'd instrument the golden signals per service, alert on the SLO, and trace across the queue boundary." Cheap points, rarely claimed.
- **Quick Reference:** signal comparison table (cost, cardinality, question answered, retention).

### 7.2 SLI, SLO & SLA (`slo-sla.html`)

Defining "working" with numbers.

- **The three terms.** SLI: the measurement (p99 latency, error rate). SLO: internal target (99.9% of requests < 300ms). SLA: external contract with penalties. SLA looser than SLO, always.
- **Choosing SLIs.** Measure what users experience (request success, latency at the edge), not what machines do (CPU). Few and meaningful over many and noisy.
- **The nines.** 99.9% vs 99.99% in minutes-per-month of allowed downtime; each nine multiplies cost. Composite availability: serial dependencies multiply failure (five 99.9% services in a row ≈ 99.5%).
- **Error budgets.** 100% minus SLO = budget to spend on deploys, experiments, risk. Budget exhausted → freeze features, fix reliability. Turns reliability arguments into arithmetic.
- **Alerting.** Symptoms not causes: alert on user-facing SLI breach, not CPU. Burn-rate alerts in one paragraph. Alert fatigue as a real outage cause.
- **Quick Reference:** SLI/SLO/SLA table, availability table (nines → downtime/month), error-budget flow.

### 7.3 Security Basics (`security-basics.html`)

Enough security to design responsibly. Interview depth, not a security course.

- **Authentication.** Sessions + cookies vs JWTs: where state lives, revocation (easy for sessions, hard for JWTs), token expiry + refresh tokens. OAuth2/OIDC in one paragraph: delegated authorization, "sign in with X".
- **Authorization.** AuthN vs AuthZ distinction. RBAC as the default model, where it's enforced (gateway coarse, service fine-grained).
- **Encryption.** In transit: TLS everywhere, including service-to-service (mTLS in one paragraph). At rest: disk/DB encryption, field-level for the truly sensitive. Hashing vs encryption; passwords get bcrypt/argon2, never reversible encryption.
- **Secrets management.** No secrets in code/env files, secret stores (Vault, cloud KMS), rotation.
- **The perimeter.** Rate limiting as abuse defense (links to Scaling Patterns), input validation at the edge, principle of least privilege between services, defense in depth in one paragraph.
- **Quick Reference:** sessions vs JWT table, checklist of security lines to say during a design ("TLS everywhere, hashed passwords, authZ at the gateway, secrets in a vault").

---

## 8. Case Studies

*Worked designs from requirements to tradeoffs. Rough difficulty order.*

Every case study follows the Design Process structure explicitly, so the framework gets rehearsed each time:

1. **Requirements** — functional + non-functional, with the clarifying questions a candidate should ask, and the answers assumed.
2. **Estimation** — QPS, storage, bandwidth, worked with visible arithmetic.
3. **High-level design** — ASCII diagram, one box per responsibility, narrated.
4. **Deep dives** — the 2-3 components interviewers actually probe for this question.
5. **Tradeoffs & failure modes** — what was chosen, what it costs, what breaks and what happens.
6. **Quick Reference** — the design on one card: key numbers, key decisions, the deep-dive answers compressed.

Each deep dive links back to the concept page that covers it in general form.

### 8.1 URL Shortener (`url-shortener.html`)

The classic warm-up. Read-heavy ratio (~100:1), tiny payloads.

- Requirements: shorten, redirect, custom aliases?, expiry?, click analytics as explicit scope cut.
- Estimation: writes/day → read QPS, storage for N years of mappings (small: this is the lesson — it fits in cache).
- Design: API → key generation → store → redirect path.
- Deep dives: **short-key generation** (hash-and-truncate collisions vs counter + base62 vs pre-generated key pool; links to Unique ID Generation), **redirect path latency** (cache-aside, 301 vs 302 and the caching/analytics tradeoff), **hot links** (cache handles it; links to Caching).
- Failure modes: cache down, duplicate submissions, key exhaustion math.

### 8.2 Rate Limiter (`rate-limiter.html`)

Infra-shaped: no product UI, all correctness and latency.

- Requirements: limit by what key (user/IP/API key), hard vs soft limits, response on breach (429 + headers), accuracy vs performance tolerance.
- Estimation: decisions per second at gateway scale, memory per tracked key × keys.
- Design: where it sits (gateway middleware), rules config, counter store.
- Deep dives: **algorithm choice** (token bucket vs sliding window counter, burst behavior; links to Scaling Patterns), **distributed counting** (local counters + sync vs centralized Redis: accuracy vs latency vs single point, race conditions and atomic ops/Lua), **failure behavior** (fail open vs fail closed, and why the answer depends on what's being protected).
- Failure modes: Redis down, clock skew across nodes, thundering herd at window reset.

### 8.3 Typeahead (`typeahead.html`)

The compact search-shaped one. Latency budget ~100ms, precompute-heavy.

- Requirements: suggestions after each keystroke, top-k by popularity, personalization?, freshness of trending queries (explicit scope decision).
- Estimation: keystrokes → QPS amplification (every keystroke is a query), index size for top queries.
- Design: client (debounce) → suggestion service → precomputed store; offline pipeline aggregating query logs.
- Deep dives: **the data structure** (trie with top-k per node vs precomputed prefix → suggestions table; links to Search), **the update pipeline** (batch aggregation of query logs, merging trending signals; links to Batch & Stream Processing), **serving at scale** (shard by prefix, hot prefixes, cache in front).
- Failure modes: stale suggestions, offensive-suggestion filtering, hot prefix shard.

### 8.4 Chat System (`chat-system.html`)

Real-time delivery, ordering, presence, fan-out.

- Requirements: 1:1 + group chat, delivery/read receipts, presence, offline delivery, message history. Group size cap as a key clarifying question.
- Estimation: DAU → messages/day → peak message QPS, connection count, storage/year.
- Design: chat gateways holding WebSockets → message service → store; pub/sub or routing tier between gateways.
- Deep dives: **connection layer** (WebSockets, gateway tier, which gateway has which user: registry vs pub/sub; links to Real-Time Delivery), **ordering & IDs** (per-conversation ordering, sequence numbers vs Snowflake timestamps; links to Sharding, Concurrency Control), **delivery guarantees** (acks, server buffer, sync-on-reconnect, idempotent dedup on client), **group fan-out** (per-recipient delivery, large groups → pull model), **presence** (heartbeats, last-seen, why presence is allowed to be approximate).
- Failure modes: gateway dies with 50K connections (reconnect storm, jitter), message store slow, user on two devices.

### 8.5 News Feed (`news-feed.html`)

The fan-out question. Read path vs write path tension.

- Requirements: follow graph, post, ranked feed on open, media via CDN (pointer to Video Streaming for payload side).
- Estimation: feed reads vs posts (read-heavy), fan-out write amplification math for average vs celebrity user — the numbers that justify the hybrid.
- Design: post service → fan-out workers → per-user feed cache; feed service assembling on read.
- Deep dives: **push vs pull vs hybrid** (fan-out-on-write for normal users, fan-out-on-read for celebrities, the threshold; links to Tradeoffs, Caching), **feed storage** (per-user sorted list in Redis, capped length, regeneration on miss), **ranking** (chronological vs scored; scoring pipeline as batch/stream hybrid; links to Batch & Stream Processing), **celebrity hot keys** (links to Sharding hot spots).
- Failure modes: fan-out queue backlog (feed staleness), cache eviction of a feed, unfollow racing fan-out.

### 8.6 Video Streaming (`video-streaming.html`)

The storage-heavy one. Large payloads change every number.

- Requirements: upload, transcode, watch (adaptive quality), thumbnails; likes/comments/recommendations as scope cuts.
- Estimation: uploads/day × size → ingest bandwidth and storage/year (the petabyte moment), egress via CDN as the dominant cost.
- Design: upload service → object storage → transcoding pipeline → CDN → player. Metadata DB separate from payload throughout.
- Deep dives: **upload path** (presigned URLs direct to object storage, multipart/resumable upload; links to Specialized Stores blob section), **transcoding pipeline** (async queue → workers, resolution ladder, HLS/DASH segmenting, chunk-level parallelism; links to Messaging), **delivery** (adaptive bitrate: player picks segment quality from manifest, CDN serves segments, origin shield; links to Networking Basics CDN section), **metadata vs payload split** as the load-bearing pattern.
- Failure modes: transcoding backlog on viral upload day, CDN miss storm on sudden-hot video, partial upload cleanup.

### 8.7 Ride Sharing (`ride-sharing.html`)

Geospatial + real-time matching under write-heavy location load.

- Requirements: rider requests, driver matching, live location tracking, trip lifecycle, surge/pricing as scope cut.
- Estimation: active drivers × location update frequency → write QPS (write-heavy: the lesson), match requests/sec.
- Design: location ingestion → geo index; matching service; trip service; real-time channels to both apps.
- Deep dives: **location ingestion** (high-frequency writes, in-memory geo store, batching, TTL on stale locations), **geo indexing** (geohash cells, neighbor-cell search for radius queries, cell size tradeoff; links to Specialized Stores geospatial), **matching** (candidate set → rank → offer with lock/lease on driver so two riders can't match one driver; links to Concurrency Control), **trip state machine** (states, who can transition, idempotent transitions).
- Failure modes: geo store down (degrade to larger cells / last-known), driver accepts as request times out, city-center hot cells.

### 8.8 Job Scheduler (`job-scheduler.html`)

The coordination-heavy one. Ties together queues, locks, idempotency, leader election.

- Requirements: one-off + recurring (cron) jobs, retries with backoff, at-least-once execution with exactly-once effect, priorities, misfire policy as a clarifying question.
- Estimation: jobs/day, peak fire-rate (top-of-minute spikes), payload storage.
- Design: job store → due-job poller/scheduler tier → execution queue → worker fleet → status tracking.
- Deep dives: **finding due jobs** (DB polling with `FOR UPDATE SKIP LOCKED` vs delay queues vs timing wheel; index on next-fire-time), **exactly-once effect** (at-least-once firing + idempotent execution, dedup keys, visibility timeout for dead workers re-queue; links to Concurrency Control, Messaging), **scheduler HA** (leader election for the polling tier, or partitioned pollers; links to Consensus), **recurring jobs** (computing next fire time, misfire policies: run-once-now vs skip vs catch-up).
- Failure modes: worker dies mid-job, clock skew between schedulers, thundering herd at minute boundaries, poison job retry loop.

---

## Build note

Before generating pages: extract the sidebar nav into a template stamped by a build script. 35 pages of hand-copied nav will drift. The script also updates the hero pill counts on `index.html` (8 topic groups, 27 concept pages, 8 case studies).
