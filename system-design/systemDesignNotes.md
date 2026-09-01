# System Design — Deep Dive Roadmap

We'll go from fundamentals → building blocks → distributed systems theory → scalability patterns → real-world case studies → interview problems.

---

## 1. System Design Fundamentals

**Definition:** system design is the process of defining the architecture, components, interfaces, and data flow of a software system to satisfy both its **functional requirements** (what it must do) and **non-functional requirements** (how well it must do it — scale, speed, reliability) — distinct from algorithm design, which optimizes a single computation rather than an entire system's structure.

**Functional vs non-functional requirements — Definition:** **functional** requirements describe specific features/behaviors (e.g. "users can post a tweet"); **non-functional** requirements describe quality attributes the system must exhibit while doing so (e.g. "the feed must load in under 200ms for 99% of requests, and remain available during a single data center outage") — system design interviews and real architecture work are dominated by the *non-functional* requirements, since they're what actually drive architectural tradeoffs.

**Scalability — Definition:** a system's ability to handle a growing amount of load (users, data, traffic) by adding resources, ideally with a roughly proportional (not degrading) relationship between resources added and capacity gained.

**Availability — Definition:** the proportion of time a system is operational and able to serve requests, typically expressed in "nines" (99.9% = ~8.7 hours of downtime/year, 99.99% = ~52 minutes/year) — achieved primarily through redundancy (no single point of failure).

**Reliability — Definition:** the probability a system performs its intended function correctly over a given period — related to but distinct from availability: a system can be "up" (available) while returning wrong or corrupted results (unreliable).

**Latency vs throughput — Definition:** **latency** is the time to complete a single operation (e.g. 50ms per request); **throughput** is the number of operations completed per unit time (e.g. 10,000 requests/second) — the two aren't simply inversely related: a system can increase throughput (via parallelism/batching) while individual-request latency stays flat or even increases slightly.

**Performance vs scalability — Definition:** **performance** is how fast a system responds under a *given* load; **scalability** is how well that performance holds up as load *increases* — a system can be fast at low load (good performance) but fall over entirely at 10x load (poor scalability), and vice versa.

**Back-of-the-envelope estimation — Definition:** quickly approximating a system's scale requirements (traffic, storage, bandwidth) using reasonable assumptions and simple arithmetic, to inform architecture decisions (e.g. "do we need sharding?") before any real implementation — a core system-design-interview skill.

```
Requests per second (RPS) ≈ Daily Active Users × avg requests/user/day ÷ 86,400
  e.g. 10M DAU × 20 requests/day ÷ 86,400 ≈ 2,300 RPS average
  (multiply by a peak factor, e.g. 3-5x, for peak RPS)

Storage estimate ≈ records/day × avg record size × retention period
  e.g. 5M posts/day × 1KB × 365 days ≈ 1.8 TB/year

Bandwidth estimate ≈ RPS × avg request/response size
```

**Vertical vs horizontal scaling — Definition:** **vertical scaling** ("scaling up") means adding more resources (CPU, RAM) to a single machine — simple, but bounded by the largest available machine, and creates a single point of failure. **Horizontal scaling** ("scaling out") means adding more machines and distributing load across them — effectively unbounded, but requires the application to be designed for distribution (statelessness, data partitioning) from the start.

---

## 2. Networking Fundamentals for System Design

**Client-server model — Definition:** the fundamental architecture where **clients** initiate requests and **servers** respond to them over a network — nearly every system covered in this roadmap is some elaboration of this basic model at scale.

**DNS resolution flow — Definition:** the process of translating a human-readable domain name into an IP address before a client can connect: browser cache → OS cache → recursive resolver (typically the ISP's or a public one like 8.8.8.8) → root nameserver → TLD nameserver (`.com`) → authoritative nameserver for the domain, which returns the actual IP.

**IP & TCP/UDP basics — Definition:** IP handles addressing/routing packets between machines; **TCP** is a connection-oriented protocol guaranteeing ordered, reliable delivery (via handshakes, acknowledgments, retransmission) — used when correctness matters more than raw speed (HTTP, database connections); **UDP** is connectionless and unreliable (no delivery guarantee, no ordering) but has lower overhead — used where speed/low-latency matters more than perfect delivery (video streaming, gaming, DNS queries).

**HTTP/HTTPS** — the application-layer protocol underlying most web/API traffic; HTTPS adds TLS encryption on top of HTTP.

**HTTP/1.1 vs HTTP/2 vs HTTP/3 — Definition:** **HTTP/1.1** opens a new TCP connection per parallel request (or reuses one connection sequentially, subject to head-of-line blocking); **HTTP/2** multiplexes many requests over a *single* TCP connection concurrently, plus header compression and server push; **HTTP/3** replaces TCP with **QUIC** (built on UDP), eliminating TCP-level head-of-line blocking and reducing connection-establishment latency — each generation primarily targets reducing the overhead/latency of establishing and multiplexing connections.

**REST — Definition:** an architectural style for APIs built around stateless requests to named **resources** using standard HTTP methods (see the Node.js notes' REST section for depth) — the dominant style for public/browser-facing APIs due to its simplicity and cacheability.

**gRPC — Definition:** a high-performance RPC (Remote Procedure Call) framework built on HTTP/2, using Protocol Buffers (a compact binary serialization format) instead of JSON — offers lower latency and smaller payloads than REST/JSON, at the cost of being less human-readable and less naturally browser-friendly — commonly used for internal service-to-service communication in a microservices architecture rather than public APIs.

**WebSockets (recap)** — a persistent, full-duplex connection allowing real-time bidirectional communication (see the Node.js/React notes).

**Long polling vs SSE vs WebSockets — Definition:** three approaches to server-to-client push, in increasing sophistication — **long polling** (client repeatedly sends a request that the server holds open until there's new data, then the client immediately re-requests) is the simplest and most compatible but least efficient; **SSE** (Server-Sent Events) provides genuine server push over a single long-lived HTTP connection, but only server→client; **WebSockets** provide true bidirectional push — the right choice depends on whether the client also needs to *send* frequent real-time messages (chat: WebSockets) or only *receive* them (live notifications: SSE is often sufficient and simpler).

---

## 3. Load Balancing

**Definition:** a load balancer distributes incoming traffic across multiple backend servers, improving availability (no single server is a single point of failure) and scalability (capacity grows by adding more backend servers behind it).

**Layer 4 vs Layer 7 load balancing — Definition:** an **L4** load balancer operates at the transport layer, routing based on IP/port information alone without inspecting the actual request content — faster, simpler, protocol-agnostic. An **L7** load balancer operates at the application layer, able to inspect HTTP headers/paths/cookies and route accordingly (e.g. `/api/*` to one service, `/images/*` to another) — more flexible, at some additional processing overhead. (Same distinction as AWS's NLB vs ALB, section 3 of the AWS notes.)

**Load balancing algorithms — Definition:**
- **Round robin** — requests distributed sequentially across servers in rotation — simple, assumes all servers/requests are roughly equal.
- **Least connections** — routes to whichever server currently has the fewest active connections — better than round robin when request processing time varies significantly.
- **IP hash** — routes based on a hash of the client's IP, so a given client consistently reaches the same server — a simple way to achieve session affinity without shared session storage.
- **Weighted variants** — assign different servers different weights (e.g. a more powerful server gets proportionally more traffic) on top of any of the above algorithms.

**Health checks** — see the AWS/Kubernetes notes; a load balancer periodically probes each backend and automatically stops routing to any that fail, preventing traffic from reaching a broken instance.

**Sticky sessions — Definition:** configuring the load balancer to route all requests from a given client to the *same* backend server for the duration of a session — needed when session state is stored in that server's local memory rather than a shared store; generally considered an anti-pattern at scale (it reintroduces server statefulness and complicates scaling/failover) in favor of externalizing session state (e.g. to Redis) so any server can handle any request.

**Global Server Load Balancing (GSLB) — Definition:** load balancing across *geographically distributed* data centers/regions (typically via DNS-based routing, like Route 53's latency/geolocation routing — see the AWS notes) — directs a user to the nearest/healthiest region, complementing (not replacing) the in-region load balancing described above.

---

## 4. Caching

**Definition:** caching stores a copy of frequently-accessed or expensive-to-compute data in a faster-access layer (memory, closer to the client) to avoid repeating that expensive work/fetch on every request.

**Cache-aside (lazy loading) — Definition:** the application checks the cache first; on a miss, it fetches from the source of truth (database), then writes the result into the cache for next time — the most common caching pattern (matches the Redis example already covered in the Node.js notes' section 11), since the cache only ever holds data that's actually been requested.

**Write-through — Definition:** every write goes to the cache *and* the underlying database synchronously, together, before the write is considered complete — keeps the cache always consistent with the database, at the cost of added write latency (every write pays the cache-write cost too).

**Write-back (write-behind) — Definition:** a write is applied to the cache immediately and acknowledged, with the actual database write happening asynchronously afterward — lowest write latency, but risks data loss if the cache fails before the deferred write completes.

**Write-around — Definition:** writes go directly to the database, bypassing the cache entirely; the cache is only populated on a subsequent read (cache-aside style) — avoids caching data that's written but never (or rarely) read again.

**Cache eviction policies — Definition:** rules for deciding what to remove when a (size-limited) cache is full — **LRU** (Least Recently Used — evict the item not accessed for the longest time, the most common default), **LFU** (Least Frequently Used — evict the item accessed the fewest times), **FIFO** (evict the oldest-inserted item regardless of access pattern).

**Cache invalidation strategies — Definition:** the mechanisms for removing/updating stale cached data once the underlying source changes — a TTL (time-to-live) expiration (simple, but tolerates some staleness up to the TTL window), explicit invalidation on write (immediately deletes/updates the cache entry when the source changes, more complex but more consistent), or a pub/sub-based invalidation broadcast across multiple cache nodes/instances. Famously one of the two hard problems in computer science ("cache invalidation and naming things").

**CDN caching (recap)** — caching static (or semi-static) content at edge locations close to users, covered in the AWS notes (CloudFront) and Node.js notes' API-caching context.

**Distributed caching at scale** — a single Redis/Memcached instance becomes a bottleneck/single point of failure past a certain scale; distributed caching shards cache data across multiple nodes (via consistent hashing, section 5) and/or replicates it for availability.

**Cache stampede / thundering herd — Definition:** occurs when a heavily-requested cache entry expires (or the cache restarts cold) and a large burst of concurrent requests all simultaneously miss the cache and hit the database at once, potentially overwhelming it — mitigated with techniques like request coalescing (only one request actually fetches from the DB, others wait for its result), staggered/jittered TTLs, or serving stale data briefly while a background refresh completes.

---

## 5. Databases at Scale

**SQL vs NoSQL tradeoffs (recap)** — see the SQL/MySQL and MongoDB notes for depth; at the system-design level, the key question is whether the data's relationships and consistency needs justify SQL's structure/joins/transactions, or whether NoSQL's flexible schema and horizontal-scale-friendly design better fits the access pattern.

**Replication (recap)** — see section 15 of the SQL notes / section 6 of the MongoDB notes; copies of data across multiple nodes for availability and read scaling.

**Sharding/partitioning strategies — Definition:**
- **Range-based sharding** — partitions data by a value range (e.g. user IDs 1–1M on shard A, 1M–2M on shard B) — simple, supports efficient range queries, but risks uneven load if data/traffic isn't evenly distributed across ranges (a "hot" range).
- **Hash-based sharding** — partitions by the hash of a key, distributing data more evenly regardless of key distribution — but makes range queries across shards expensive/impossible.
- **Directory-based sharding** — a separate lookup service maps each key to its shard explicitly, rather than a fixed formula — most flexible (can rebalance individual keys), but adds a lookup indirection and a new potential single point of failure/bottleneck.
- **Consistent hashing — Definition:** a hashing scheme that maps both data and shard/server nodes onto a conceptual ring (a fixed hash space); a key belongs to the first node found by walking clockwise from its hash position. Its key advantage: when a node is added or removed, only the keys in the immediately adjacent portion of the ring need to move — not a full remap of the entire dataset (which naive `hash(key) % num_servers` sharding would require on every scale-out event). Used by distributed caches (e.g. Memcached client-side sharding) and databases (Cassandra, DynamoDB internally).

**Database indexing recap** — see the SQL notes' indexing section; the same B-tree indexing principles apply, scaled to distributed databases with their own considerations for cross-shard index maintenance.

**Denormalization for scale** — see the SQL/MongoDB notes' denormalization sections; at large scale, avoiding expensive cross-shard joins (which sharding makes especially costly or impossible) often pushes designs toward deliberate denormalization.

**Polyglot persistence — Definition:** using different database technologies for different parts of the same system, each chosen for the specific access pattern it serves best (e.g. PostgreSQL for transactional order data, Elasticsearch for product search, Redis for session/cache data, S3 for uploaded images) — rather than forcing one database technology to serve every need.

---

## 6. Consistency, Availability & Distributed Systems Theory

**CAP theorem — Definition:** in the presence of a network **P**artition (nodes unable to communicate), a distributed system must choose between **C**onsistency (every read returns the most recent write) and **A**vailability (every request receives a non-error response) — it cannot guarantee both simultaneously during a partition. (When there's no partition, a well-designed system can and should provide both.)

```
     Consistency
        /  \
       /    \
  CP  /      \  AP
     /        \
Availability — Partition tolerance
```

**PACELC theorem — Definition:** an extension of CAP acknowledging that the tradeoff exists even *without* a partition: **if Partitioned**, choose **A**vailability or **C**onsistency (as in CAP); **Else** (normal operation), choose **L**atency or **C**onsistency — because achieving strong consistency during normal operation (e.g. synchronously replicating to all nodes before acknowledging a write) inherently costs latency compared to a weaker-consistency, lower-latency approach.

**Strong vs eventual consistency — Definition:** **strong consistency** guarantees that once a write completes, every subsequent read (from any node) sees it immediately; **eventual consistency** only guarantees that, given enough time with no new writes, all nodes will *eventually* converge to the same value — reads shortly after a write may return stale data. Choosing between them is a direct application of CAP/PACELC to a specific feature (e.g. a bank balance likely needs strong consistency; a social media "like" count can tolerate eventual consistency).

**Consistency models — Definition:** finer-grained points on the consistency spectrum between strong and eventual — **linearizability** (the strongest practical model: operations appear to take effect instantaneously at some point between their start and end, as if there were a single global order) and **causal consistency** (operations that are causally related — e.g. a reply to a comment — are seen by everyone in the same order, but unrelated operations may be seen in different orders by different observers).

**Consensus algorithms (conceptual) — Definition:** Paxos and Raft are algorithms that let a distributed set of nodes agree on a single value (e.g. "who is the current leader," or "what is the next entry in a replicated log") even when some nodes fail or messages are delayed/lost — the theoretical foundation underlying leader election in systems like etcd (Kubernetes' state store, see the Docker/K8s notes), MongoDB's replica set elections, and distributed databases requiring strong consistency guarantees. Raft is generally considered more understandable than Paxos while providing equivalent guarantees, which is why most modern systems (etcd, Consul) implement Raft specifically.

**Quorum reads/writes — Definition:** a technique for tuning the consistency/availability tradeoff explicitly — requiring a write to be acknowledged by **W** out of **N** replicas, and a read to consult **R** out of **N** replicas; if `W + R > N`, every read is guaranteed to overlap with the most recent write (strong consistency), while smaller `W`/`R` values favor lower latency/higher availability at the cost of potentially reading stale data.

**Split-brain — Definition:** a failure scenario where a network partition causes two (or more) subsets of a cluster to each independently believe they are the sole active leader/primary, potentially accepting conflicting writes on both sides — consensus algorithms (requiring a *majority* to elect a leader) are specifically designed to prevent this, since a true majority can only exist on one side of any partition.

**Vector clocks (brief) — Definition:** a mechanism for tracking causality between events across distributed nodes without relying on synchronized physical clocks — each node maintains a counter per node it knows about, incrementing its own and merging others' on message receipt, letting the system determine whether one event happened-before another or whether they were concurrent (and therefore potentially conflicting).

---

## 7. Messaging & Asynchronous Systems

**Synchronous vs asynchronous communication — Definition:** in **synchronous** communication, the caller blocks/waits for the callee to respond before proceeding (a typical REST/gRPC call); in **asynchronous** communication, the caller sends a message and continues immediately, with the result (if any) delivered later, decoupling the two parties' timing entirely.

**Message queues (recap)** — see the AWS notes' SQS section; durable, point-to-point delivery from producer(s) to consumer(s), with each message processed by exactly one consumer.

**Pub/sub systems (recap)** — see the AWS notes' SNS/EventBridge sections; one-to-many broadcast, where every subscriber receives its own copy of each published message.

**Event-driven architecture (recap)** — see the Node.js architecture notes; services communicate primarily by emitting/reacting to events rather than direct synchronous calls.

**Event sourcing — Definition:** instead of storing only an entity's *current* state, the system stores the full, immutable sequence of **events** that led to that state (e.g. `AccountCreated`, `FundsDeposited`, `FundsWithdrawn`), with current state derived by replaying those events — provides a complete audit trail and the ability to reconstruct state as of any point in time, at the cost of more complex querying (you generally need a separate read-optimized projection, see CQRS below) and eventual (not immediate) consistency of derived state.

**CQRS (Command Query Responsibility Segregation) — Definition:** separating the model used to **write** data (commands, which change state) from the model used to **read** data (queries) — the write side can be optimized for correctness/validation (often paired with event sourcing above), while one or more read-side models are optimized/denormalized specifically for the queries the application actually needs, kept in sync asynchronously from the write side's events.

**Delivery guarantees — Definition:**
- **At-most-once** — a message is delivered zero or one times; if delivery fails, it's simply lost — lowest overhead, acceptable only when occasional message loss is tolerable.
- **At-least-once** — a message is guaranteed to be delivered, but *may* be delivered more than once (e.g. if an acknowledgment is lost after the consumer processed it, the producer/broker retries) — the most common guarantee in practice (SQS, Kafka default), which pushes the burden of handling duplicates onto the consumer.
- **Exactly-once** — a message is guaranteed to be delivered and processed exactly one time — extremely difficult to guarantee end-to-end across a truly distributed system in the general case, and where offered (Kafka's exactly-once semantics, within specific constraints) usually comes with real cost/complexity/limitations.

**Idempotency — Definition:** a property of an operation such that performing it multiple times has the *same effect* as performing it once — the standard, practical answer to at-least-once delivery's duplicate-message problem: design consumers to be idempotent (e.g. via a unique message/idempotency ID checked against a "already processed" record) so a duplicate delivery is safely a no-op rather than double-charging a customer or double-decrementing inventory.

---

## 8. API Design

**REST API design principles (recap)** — see the Node.js notes' REST section; resource-oriented URLs, correct HTTP methods/status codes, statelessness.

**Pagination strategies — Definition:**
- **Offset-based** (`?page=3&limit=20`) — simple, allows jumping to an arbitrary page, but degrades in performance at high offsets and can show duplicate/skipped items if data changes between page requests.
- **Cursor-based** (`?after=<opaque_cursor>`) — uses a pointer to the last-seen item (often an encoded ID/timestamp) rather than a numeric offset — stays fast regardless of depth and stays stable under concurrent inserts/deletes, at the cost of not supporting "jump to page N" directly. (Same tradeoff already covered in the MongoDB/SQL notes.)

**Rate limiting algorithms — Definition:**
- **Token bucket** — a bucket holds tokens, refilled at a fixed rate up to a max capacity; each request consumes one token, and is rejected if the bucket is empty — allows short bursts up to the bucket's capacity while enforcing a long-term average rate.
- **Leaky bucket** — requests are queued and processed (or "leaked") at a strictly constant rate, regardless of how bursty the incoming traffic is — smooths out bursts entirely, unlike token bucket, at the cost of added latency for bursty traffic (requests wait in the queue).
- **Fixed window** — counts requests in a fixed time window (e.g. per calendar minute), resetting the counter at each window boundary — simple, but allows up to 2x the intended rate right at a window boundary (a burst at the end of one window plus a burst at the start of the next).
- **Sliding window** — counts requests within a continuously moving time window (either precisely, tracking every request's timestamp, or approximately, via a weighted combination of the current and previous fixed windows) — avoids the fixed-window boundary-burst problem at some added implementation complexity.

**API versioning (recap)** — see the AWS/Node.js notes; URL-path (`/v1/`), header, or query-parameter based versioning strategies for evolving an API without breaking existing clients.

**Idempotency keys — Definition:** a client-generated unique identifier attached to a mutating request (typically in a header, e.g. `Idempotency-Key`), which the server records and uses to detect and safely ignore a retried duplicate of the same logical request (e.g. a client retrying a payment request after a network timeout, without knowing whether the original actually succeeded) — a direct, API-layer application of the idempotency concept from section 7.

**Backpressure — Definition:** a mechanism by which a system experiencing more load than it can handle signals that pressure *back* to its callers/upstream (via slowing responses, rejecting new requests with an explicit error, or a queue filling up and refusing new items) rather than silently degrading or crashing — the same core concept as stream backpressure in the Node.js notes, applied at a whole-system level.

---

## 9. Microservices vs Monolith

**Monolithic architecture — Definition:** a single deployable application containing all of a system's functionality — simpler to develop, test, and deploy initially (one codebase, one deployment pipeline, straightforward local transactions), but as it grows, changes across teams/features become harder to isolate, and the entire application must scale (and be deployed) as one unit even if only one part is actually under heavy load.

**Microservices architecture — Definition:** the system is split into multiple independently deployable services, each typically owning its own data store and communicating over the network — enables independent scaling, deployment, and technology choice per service, and clearer team ownership boundaries, at the cost of significant added operational complexity: network calls replace in-process function calls (latency, partial failure), distributed transactions become hard (section below), and testing/debugging a request that spans many services is harder than debugging one process.

**Service boundaries (Domain-Driven Design, brief) — Definition:** DDD's concept of a **bounded context** — a specific, cohesive area of business functionality with its own internal model and language — is commonly used to decide where microservice boundaries should actually be drawn, favoring splits along real business-domain lines (orders, inventory, payments) rather than arbitrary technical splits, which tends to minimize the cross-service chatter/coupling that erodes microservices' benefits.

**Inter-service communication patterns** — synchronous (REST/gRPC, section 2) for request/response needs where the caller genuinely needs an immediate answer; asynchronous (message queues/events, section 7) for decoupled, eventually-consistent workflows — most real microservice systems use a mix of both, chosen per interaction.

**Service discovery — Definition:** the mechanism by which one service finds the current network location of another, given that service instances in a dynamically-scaled system are constantly being created/destroyed/rescheduled with changing IPs — implemented via a dedicated registry (Consul, etcd) or, in Kubernetes, transparently via the built-in Service/DNS abstraction (see the Docker/K8s notes' section 9).

**API Gateway pattern (recap)** — see the AWS notes; a single entry point handling cross-cutting concerns (auth, rate limiting, routing) for a system built from many backend services, so clients don't need to know about or directly call each individual microservice.

**Sidecar & service mesh pattern — Definition:** a **sidecar** is a helper container/process deployed alongside a main application container, handling cross-cutting infrastructure concerns (e.g. TLS, retries, metrics) transparently, without the application code needing to implement them itself; a **service mesh** (Istio, Linkerd) is the coordinated deployment of a sidecar proxy next to *every* service instance in a system, providing consistent traffic management, observability, and security across all inter-service communication without any application-code changes.

**Distributed transactions & the Saga pattern — Definition:** a traditional ACID transaction across multiple databases (owned by different microservices) isn't practically achievable the way a single-database transaction is; a **Saga** instead breaks a multi-step, cross-service business transaction into a sequence of local transactions, each with a corresponding **compensating transaction** that can undo its effect if a later step in the sequence fails — trading true atomicity for eventual consistency plus explicit rollback logic. (Orchestrated via a central coordinator, or choreographed via each service reacting to the previous step's event.)

**Circuit breaker pattern (recap)** — see the Node.js notes' section 10; stops calling a failing downstream service temporarily, letting it recover instead of piling up doomed requests — especially important in microservices, where one struggling service can otherwise cascade failure through every service that calls it.

---

## 10. Data Storage at Scale

**Object storage at scale (recap)** — see the AWS S3 notes; the standard choice for large binary blobs (images, videos, backups, logs) at any scale, rather than storing them directly in a relational/document database (see the MongoDB notes' production pitfalls).

**CDN (recap)** — see section 4 and the AWS notes; edge-caching static/semi-static content close to users.

**Blob storage vs database storage** — databases (relational or NoSQL) are optimized for structured, queryable data with relationships and transactional guarantees; object storage is optimized for storing and retrieving large, opaque files cheaply at massive scale — a well-designed system stores a *reference* (a URL/key) to a blob in the database, not the blob itself.

**Time-series databases — Definition:** databases (InfluxDB, TimescaleDB, Prometheus's own storage) purpose-built for data points indexed by time (metrics, sensor readings, logs) — optimized for high-throughput sequential writes and time-range queries/aggregations (e.g. "average CPU per minute over the last 24 hours"), typically with built-in downsampling/retention policies, outperforming a general-purpose database for this specific access pattern at scale.

**Search systems (inverted index) — Definition:** systems like Elasticsearch build a **inverted index** — a mapping from each word/term to the list of documents containing it (the inverse of a normal document→words mapping) — enabling fast full-text search across massive document collections, something a relational database's B-tree indexes are not designed for efficiently.

**Data warehouses vs OLTP databases — Definition:** an **OLTP** (Online Transaction Processing) database (standard PostgreSQL/MySQL/MongoDB usage) is optimized for many small, fast read/write transactions from an application; a **data warehouse** (Redshift, BigQuery, Snowflake) is optimized for large, complex analytical queries scanning huge volumes of historical data (aggregations across millions/billions of rows) — the two workloads have conflicting optimization needs, which is why production systems typically run OLTP for the live application and periodically **ETL** (Extract, Transform, Load) data into a separate warehouse for analytics, rather than running analytics queries against the live production database.

**Data lakes (brief) — Definition:** a centralized repository (typically built on object storage) holding raw data in its native format (structured, semi-structured, unstructured) at any scale, without requiring a schema to be defined upfront — schema is applied later, at read/query time ("schema-on-read"), unlike a data warehouse's "schema-on-write" — used as a flexible staging ground feeding downstream analytics/ML pipelines.

---

## 11. Scalability Patterns

**Horizontal scaling patterns (recap)** — see section 1; the general strategy of adding more machines rather than bigger ones.

**Stateless vs stateful services — Definition:** a **stateless** service holds no client-specific data between requests — any instance can handle any request, which is exactly what makes horizontal scaling and simple round-robin load balancing possible; a **stateful** service holds data specific to a client/session in its own memory/disk, requiring either sticky sessions (section 3, generally discouraged) or externalizing that state to a shared store (a database/Redis) so the service itself can remain effectively stateless.

**Database connection pooling at scale (recap)** — see the Node.js/SQL notes; sized carefully so that a horizontally-scaled fleet of application instances (each with its own pool) doesn't collectively exceed the database's connection limit.

**Read replicas & read/write splitting (recap)** — see the SQL/MongoDB/AWS notes; scaling read capacity independently of write capacity by routing reads to replicas.

**Queue-based load leveling — Definition:** placing a message queue between a producer of work and the (potentially slower or more variable-capacity) consumer that processes it, so bursts of incoming work are absorbed by the queue rather than overwhelming the downstream processor directly — the consumer processes work at a sustainable rate regardless of how spiky the arrival rate is.

**Bulkhead pattern — Definition:** partitioning a system's resources (thread pools, connection pools) into isolated groups per dependency/consumer, so that one failing or slow dependency exhausting its own allocated resources can't also starve resources needed by unrelated parts of the system — named after a ship's bulkheads, which contain flooding to one compartment rather than sinking the whole ship.

**Rate limiting & throttling (recap)** — see section 8; protecting a service from being overwhelmed by capping how much load any single client/caller can generate.

**Auto-scaling strategies (recap)** — see the AWS/Kubernetes notes (ASG, HPA); automatically adjusting capacity based on observed demand, reactively (metric-based) or proactively (scheduled, for known traffic patterns).

---

## 12. Reliability & Fault Tolerance

**Redundancy & replication (recap)** — see sections 5–6; running multiple copies of data/services so the failure of any single one doesn't take down the whole system.

**Failover strategies (recap)** — see the AWS/SQL/MongoDB notes; automatically (or manually) shifting traffic/responsibility to a healthy standby when a primary component fails.

**Graceful degradation — Definition:** designing a system to continue operating, with reduced functionality, when a non-critical dependency fails — e.g. an e-commerce site still lets users browse and purchase even if the "recommended products" service (a non-critical dependency) is down, simply omitting that section, rather than the whole page failing.

**Circuit breakers (recap)** — see sections 7/9; stop calling a failing dependency temporarily rather than repeatedly waiting on doomed calls.

**Retries with exponential backoff & jitter — Definition:** see the Node.js notes' retry pattern; **exponential backoff** increases the delay between retry attempts exponentially (500ms, 1s, 2s, 4s...) to avoid hammering a struggling service; **jitter** adds randomness to that delay so that many clients retrying after the same failure don't all retry in synchronized lockstep (which would itself cause a load spike) — the combination is the standard, robust retry strategy for distributed systems.

**Timeouts — Definition:** an explicit maximum time a caller will wait for a response before giving up and treating the call as failed — without timeouts, a slow/hung dependency can tie up the caller's resources (threads/connections) indefinitely, cascading the slowness upstream — every network call in a production system should have an explicit, deliberately-chosen timeout.

**Chaos engineering (brief) — Definition:** the practice of deliberately injecting failures (killing instances, adding network latency, simulating a region outage) into a system — often in production — to verify that the redundancy/failover mechanisms designed to handle those failures actually work as intended, rather than discovering they don't during a real, unplanned incident. Popularized by Netflix's "Chaos Monkey."

**Disaster recovery (RTO/RPO) — Definition:** **RTO** (Recovery Time Objective) is the maximum acceptable time to restore service after a disaster; **RPO** (Recovery Point Objective) is the maximum acceptable amount of data loss, measured as time (e.g. "at most 5 minutes of data may be lost") — these two numbers, set based on business requirements, directly determine which disaster-recovery strategy (see the AWS notes' multi-region DR strategies) is appropriate and justified for a given system.

---

## 13. Security in System Design

**Authentication vs authorization (recap)** — see the Node.js/AWS/SQL notes; who you are vs what you're allowed to do.

**API security (recap)** — rate limiting (section 8), WAF (see the AWS notes), input validation — the layered defenses protecting a public-facing API surface.

**Encryption in transit & at rest (recap)** — see the AWS/MongoDB/SQL notes; TLS for data moving across the network, storage-level encryption for data sitting on disk.

**Zero trust architecture (brief) — Definition:** a security model that assumes no implicit trust based on network location alone (unlike a traditional "trusted internal network behind a firewall" model) — every request, whether from inside or outside the network perimeter, must be authenticated and authorized explicitly, based on the principle "never trust, always verify" — increasingly the default assumption for cloud-native/microservices architectures where there's no longer a clean network perimeter to trust in the first place.

**DDoS mitigation (recap)** — see the AWS notes' Shield/WAF sections; absorbing or filtering out a flood of malicious traffic before it reaches the actual application, typically via a combination of scale (CDN edge capacity absorbing volume), rate limiting, and traffic-pattern-based filtering.

---

## 14. Observability at Scale

**The three pillars — Definition:** **metrics** (aggregated numerical measurements over time — request rate, error rate, latency percentiles), **logs** (discrete, timestamped event records, often per-request or per-error), and **traces** (the path of a single request as it flows across multiple services) — together giving complementary views: metrics answer "is something wrong and how bad," logs answer "what exactly happened," traces answer "where in a distributed call chain did it happen."

**Distributed tracing (recap)** — see the AWS/Kubernetes notes (X-Ray, OpenTelemetry); essential once metrics/logs alone can't localize a problem to a specific service in a multi-service request path.

**SLIs, SLOs, SLAs — Definition:**
- **SLI** (Service Level Indicator) — an actual measured metric (e.g. "99.95% of requests in the last 30 days completed in under 300ms").
- **SLO** (Service Level Objective) — an internal target for that indicator (e.g. "we aim for 99.9% of requests under 300ms") — the number engineering teams actually design and operate toward.
- **SLA** (Service Level Agreement) — an external, often contractual commitment to customers (e.g. "99.9% uptime, or you receive service credits") — typically set with more margin than the internal SLO, so normal SLO variance doesn't trigger SLA violations/penalties.

**Alerting strategy** — alerts should be tied to SLOs/user impact (error rate, latency breaching a threshold) rather than raw infrastructure metrics alone (e.g. "CPU is at 80%" is often not actionable on its own) — a common, hard-won lesson is to minimize alert *noise*, since engineers stop trusting (and responding to) alerting systems that cry wolf too often.

**Capacity monitoring** — tracking resource utilization trends over time (not just point-in-time) to anticipate when a system will need to scale, before it becomes an incident.

---

## 15. Real-Time & Streaming Systems

**Stream processing concepts — Definition:** processing a continuous, unbounded flow of data records as they arrive, rather than as a fixed, complete batch — enabling near-real-time analytics/reactions instead of waiting for a scheduled batch job to run over accumulated data.

**Kafka-style log-based messaging — Definition:** Apache Kafka models messaging as a durable, append-only, partitioned **log** — producers append messages to the end of a topic's log, and consumers independently track their own read position (offset) within it, able to re-read/replay from any earlier point — a fundamentally different model from a traditional queue (where a message is typically removed once consumed), enabling multiple independent consumer groups to each process the full stream at their own pace, and supporting replay for reprocessing/recovery.

**Real-time analytics pipelines** — a common architecture: events flow into Kafka (or a similar log) → a stream processor (Kafka Streams, Flink, Spark Streaming) computes rolling aggregations/transformations in near real time → results are written to a fast-serving store (a time-series DB, a cache, a dashboard-backing database) for low-latency querying.

**WebSocket-based real-time systems at scale — Definition:** since a WebSocket connection is stateful and persistent (tying a specific client to a specific server process), scaling a WebSocket-based system (chat, live notifications) requires a way for a message originating anywhere in the backend to reach the *specific* server process currently holding that user's connection — typically solved with a pub/sub layer (Redis pub/sub, or a dedicated message broker) that every server instance subscribes to, so any instance can publish a message and have it delivered to whichever instance actually holds the target connection.

**Presence systems (online/offline status) — Definition:** tracking and broadcasting whether users are currently online/active — typically implemented with a heartbeat (the client periodically pings, or the WebSocket connection's own liveness is used) plus a TTL-based record (e.g. a Redis key with a short expiration, refreshed on each heartbeat, treated as "offline" once it expires) — must account for the fact that a user can be connected to any one of many server instances, and that a connection can drop ungracefully (network loss) without a clean "going offline" signal ever being sent.

---

## 16. Designing Specific Systems (Case Studies)

Classic interview problems. Each is a full design exercise on its own — below is the core challenge and key components to reason about for each, applying the building blocks from sections 1–15.

**Design a URL shortener — Core challenge:** generating short, unique, non-guessable-enough codes for long URLs at high write throughput, and redirecting billions of reads with minimal latency. **Key components:** a key-generation strategy (base62 encoding of an auto-incrementing/distributed ID, or a hash with collision handling), a fast key-value store for the mapping (read-heavy — cache aggressively, section 4), a redirect service returning `301`/`302`, and analytics tracked asynchronously (not blocking the redirect path).

**Design a rate limiter — Core challenge:** enforcing a per-client request-rate limit consistently across many stateless, horizontally-scaled application servers. **Key components:** a shared, fast store (Redis) holding each client's counter/token-bucket state so all server instances see a consistent view; choice of algorithm (section 8); handling the store itself as a potential bottleneck/single point of failure at very high scale (sharding the limiter state by client key).

**Design a distributed cache — Core challenge:** storing key-value data in memory across many nodes, with fast lookups, even distribution, and graceful handling of node failure/addition. **Key components:** consistent hashing (section 5) for key-to-node mapping, replication for availability, an eviction policy (section 4) per node, and a client-side or proxy-based routing layer.

**Design a news feed / timeline system — Core challenge:** efficiently generating a personalized, roughly-chronological feed of content from many followed sources, for users who may follow a handful or millions of accounts. **Key components:** **fan-out-on-write** (precompute and push a new post into every follower's feed immediately — fast reads, expensive/impossible for accounts with huge follower counts) vs **fan-out-on-read** (assemble the feed at request time by querying all followed accounts' recent posts — cheap writes, more expensive reads) — most real systems use a **hybrid**: fan-out-on-write for typical users, fan-out-on-read for celebrity/high-follower accounts.

**Design a chat application (WhatsApp-style) — Core challenge:** real-time, reliable message delivery (including to offline recipients) at massive scale. **Key components:** WebSocket connections for online delivery (section 15's scaling approach), a message queue/store for offline recipients (delivered on reconnect), message ordering/delivery-status tracking (sent/delivered/read receipts), and end-to-end encryption considerations.

**Design a notification system — Core challenge:** delivering notifications across multiple channels (push, email, SMS, in-app) reliably and at scale, without duplicating or losing them. **Key components:** a queue per channel (decoupling notification *triggering* from actual *delivery*, section 7), per-user preference/rate-limiting logic (don't spam), retry with backoff for failed deliveries, and idempotency (section 7) to avoid duplicate sends on retry.

**Design a ride-sharing system (Uber-style) — Core challenge:** matching nearby riders and drivers in real time, and tracking live location at scale. **Key components:** a geospatial index (e.g. geohashing, or a quad-tree) for efficient "find nearby drivers" queries, a matching service, frequent low-latency location updates (often via a lightweight, high-throughput ingestion path — not the primary transactional database directly), and a real-time channel (WebSockets) for pushing live trip/location updates to both parties.

**Design a video streaming service (YouTube/Netflix-style) — Core challenge:** ingesting, storing, and efficiently delivering large video files to a global audience across varying device/network conditions. **Key components:** object storage (section 10) for raw/transcoded video files, a transcoding pipeline producing multiple resolutions/bitrates, a CDN (section 4/10) for delivery, and **adaptive bitrate streaming** (the client dynamically switches video quality based on measured network conditions, via a protocol like HLS/DASH that serves video as small, independently-fetchable chunks).

**Design a search autocomplete system — Core challenge:** returning ranked suggestions for a partial query string within tens of milliseconds, at high query volume. **Key components:** a **Trie** (prefix tree) or similar prefix-indexed structure for fast prefix matching, precomputed/cached top-N results per popular prefix (rather than computing rankings live on every keystroke), and periodic (not real-time) reindexing from query-frequency logs to keep suggestions current.

**Design a distributed file storage system (Google Drive/Dropbox-style) — Core challenge:** storing large files reliably and efficiently syncing changes across a user's multiple devices. **Key components:** chunking large files into smaller blocks (deduplicated across users/versions, and enabling resumable/parallel uploads), object storage for the actual chunk data, a metadata database tracking file/folder structure and chunk references, and a sync protocol that diffs local vs remote state to transfer only changed chunks.

**Design an e-commerce checkout & inventory system — Core challenge:** preventing overselling (two customers buying the last unit of stock) while keeping checkout fast under high concurrent load (e.g. a flash sale). **Key components:** atomic inventory decrement (a database transaction or `UPDATE ... WHERE stock > 0`, section 10/11 of the SQL notes on optimistic locking), a Saga (section 9) coordinating payment/inventory/shipping across services, and a queue to smooth a traffic spike (queue-based load leveling, section 11) rather than letting every request hit the database simultaneously.

**Design a web crawler — Core challenge:** discovering and fetching a huge number of web pages efficiently, politely (respecting `robots.txt`/rate limits per site), and without infinite loops or duplicate work. **Key components:** a URL frontier (a prioritized, deduplicated queue of URLs to crawl), a distributed fetcher pool, a dedup mechanism (a Bloom filter is a common space-efficient choice for "have we seen this URL/content before") and per-domain politeness/rate-limiting to avoid overwhelming any single site.

**Design a payment processing system — Core challenge:** guaranteeing correctness (never double-charging, never losing a payment) in a system that must talk to external, sometimes-unreliable payment providers. **Key components:** idempotency keys (section 8) on every payment request, a durable, auditable transaction log/state machine (often event-sourced, section 7) tracking each payment through its lifecycle (`initiated → authorized → captured → settled`), and a reconciliation process periodically comparing internal records against the payment provider's records to catch and resolve any discrepancies.

---

## 17. System Design Interview Approach

**Clarifying requirements — Definition:** the first step in any system design interview/exercise — explicitly asking about scope (which features are truly in scope), scale (users, requests/sec, data volume), and priorities (is this read-heavy or write-heavy; is strong consistency required) *before* proposing any architecture — a system designed against wrong or assumed requirements is wrong regardless of how well-executed it is.

**Estimating scale (recap)** — see section 1; translating clarified requirements into concrete numbers (RPS, storage, bandwidth) that then justify specific architectural choices (e.g. "at this write volume, a single database instance won't keep up, so we need X").

**High-level design first, then deep dive** — sketch the major components and how data flows between them at a high level before drilling into any single component's internals — establishes a shared, correct overall shape before investing detail in any one part, and makes it easy to redirect if the high-level approach itself needs to change.

**Identifying bottlenecks** — for each component in the high-level design, reason about what breaks first as scale increases (a single database? a synchronous call chain? a hot shard key?) — the core of what differentiates a system design conversation from just listing technologies.

**Trade-off discussions** — for every major decision (SQL vs NoSQL, sync vs async, strong vs eventual consistency, monolith vs microservices), explicitly articulate *why* — what's gained and what's given up — rather than presenting a single "correct" answer, since almost every real system-design decision is a genuine tradeoff with a context-dependent right answer.

**Communicating design decisions** — narrating the reasoning behind each choice as you go (not just the final diagram) is generally more valuable in an interview than arriving at a "perfect" architecture silently — it demonstrates the judgment process, which is what's actually being evaluated.

**Common mistakes to avoid** — jumping straight to a detailed/complex design before clarifying requirements or scale; over-engineering for a scale that was never asked for; ignoring failure modes/edge cases entirely; picking a specific technology by name without being able to explain what it actually provides and why it's needed here.

---

## 18. Production & Operational Concerns

**Deployment strategies at scale (recap)** — see the AWS/Kubernetes notes; blue/green, canary, and rolling deployments become essential (not optional) once a system has enough scale/criticality that any downtime or bad-deploy blast radius has real cost.

**Multi-region architecture (recap)** — see the AWS notes' DR strategies; running a system across multiple geographic regions for lower latency to distributed users and/or disaster recovery, at the cost of significantly more complex data replication/consistency reasoning (section 6) across regions.

**Cost vs performance tradeoffs** — nearly every scalability/reliability technique in this roadmap (more replicas, more caching, multi-region, stronger consistency) costs more to run — production system design continually weighs whether a given level of scale/reliability investment is actually justified by the system's real requirements and business value, rather than defaulting to the most robust possible design regardless of cost.

**Technical debt in system design — Definition:** deliberate or accumulated architectural shortcuts (a single database that should eventually be split, a synchronous call that should eventually be async) taken to ship faster now, with a known cost of future rework — a legitimate, often correct engineering tradeoff when consciously chosen and tracked, as opposed to accidental complexity from poor design.

**Evolving a system over time (scaling stages) — Definition:** most successful systems don't start with a fully distributed, multi-region, sharded architecture — they evolve through recognizable stages as load grows: single server → separate database server → add a cache → add a load balancer + multiple app servers → add read replicas → introduce a CDN → shard the database → split into services — recognizing which stage a system is actually at (and only solving the problems that stage genuinely has) is itself an important system-design skill, rather than always designing for hypothetical future scale from day one.
