# System Design Learning Roadmap

The master learning path for this repo. It sequences everything a full-stack engineer needs to go from **true beginner to expert**. It is read at the start of most sessions, so it is kept dense and scannable.

> **Status: FINAL — locked v1 (2026-07-06).** The structure (levels, topics, sequencing) is frozen. Track progress by updating each topic's `Status`; structural changes are versioned updates, not ad-hoc edits.

## Mission: learning-first

The goal is **deep mastery of every concept** - roughly 90% of the effort goes into truly understanding how each thing works, why it exists, what it trades off, and how it composes with everything else. Real-world examples (how MAANGO companies actually apply it) and interview weight are **secondary signals**: they tell us where to ground the learning and how to prioritize when time is short, but they are not the point. A topic is understood when you can derive it, not just recall it.

Every topic aims to be learned first-principles-first, then reinforced with a concrete real example, then eventually applied in a practice attempt.

## Tracks

- **Backend / Distributed-Systems track (F, L0-L15)** - the core spine. Start here.
- **Frontend System-Design track (FE-F, FE-L0 to FE-L8)** - a parallel, adjacent track. Can be interleaved once backend L0-L1 are done.
- **Side tracks (optional, not sequenced):** DSA (coding round) and IT-trends R&D (web research).

## Status legend

| Icon | Status      | Meaning                                           |
| ---- | ----------- | ------------------------------------------------- |
| ⚪   | todo        | No material generated yet.                        |
| 🔄   | in-progress | Material currently being generated.               |
| ✅   | done        | Material generated (concept + examples + lesson). |

Order: ⚪ -&gt; 🔄 -&gt; ✅. Status tracks whether the **material exists**, not whether it was studied/practiced (those are async and not tracked here).

## Weight legend

Weight is a **priority signal, not the focus** - everything gets mastered, but when time is tight, do heavier weights first.

| Weight | Meaning                                                             |
| ------ | ------------------------------------------------------------------- |
| 🟥     | Core - foundational, appears everywhere, master deeply.             |
| 🟧     | Important - common and high-value, expected at senior level.        |
| 🟨     | Good-to-know - rounds out expertise; often emerging or specialized. |

Items marked **(emerging)** are newer or meaningfully evolved since the classic syllabus was written - learn them, but know they are the modern frontier rather than settled canon.

## Recommended starting point

Start at **F. Computing Fundamentals** (backend). It requires no prerequisites and underpins everything above it. Then move to **L0 Foundations**, then L1+ in order.

## What's next (current pointer)

> **F. Computing Fundamentals is** `done` (concept + verified examples + lesson). Next in order:
>
> 1. **L0. System-Design Foundations** - the vocabulary and estimation skills every design uses.
> 2. **L1. Networking** - how traffic reaches and moves through a system.
> 3. Then proceed down the backend track in order (L2+); interleave the frontend track once backend L0-L1 material exists.

---

# Backend / Distributed-Systems Track

## F. Computing Fundamentals

**Goal:** Understand what a single machine actually does so every distributed concept later rests on real mechanics, not magic. · **Prereqs:** none. **Material:** Lessons (12 bite-sized) · Concepts · Case studies & sources

| Topic                                                               | Status | Weight |
| ------------------------------------------------------------------- | ------ | ------ |
| CPU and memory hierarchy (registers, caches, cache lines, locality) | ✅     | 🟥     |
| Processes vs threads                                                | ✅     | 🟥     |
| Concurrency vs parallelism; context switching                       | ✅     | 🟥     |
| Locks, mutexes, semaphores; race conditions; deadlock; atomicity    | ✅     | 🟥     |
| I/O models (blocking, non-blocking, async, epoll, event loops)      | ✅     | 🟥     |
| OS scheduling and virtual memory (paging, TLB)                      | ✅     | 🟧     |
| Disks (HDD/SSD/NVMe) and filesystems                                | ✅     | 🟧     |
| Data representation (binary/hex, ASCII/Unicode/UTF-8, endianness)   | ✅     | 🟧     |
| Serialization (JSON, XML, Protobuf, Avro, Thrift)                   | ✅     | 🟥     |
| Compression (gzip, Snappy, LZ4, Zstd)                               | ✅     | 🟧     |
| Hashing (crypto vs non-crypto, collisions, checksums/CRC)           | ✅     | 🟥     |
| Clocks (monotonic vs wall clock, NTP)                               | ✅     | 🟧     |

## L0. System-Design Foundations

**Goal:** Understand what system design is and the vocabulary/estimation skills every design uses. · **Prereqs:** F. **Material:** Lessons

| Topic                                                                            | Status | Weight |
| -------------------------------------------------------------------------------- | ------ | ------ |
| What system design is; the design mindset                                        | ✅     | 🟥     |
| Functional vs non-functional requirements                                        | ✅     | 🟥     |
| Client-server model                                                              | ✅     | 🟥     |
| Request lifecycle end-to-end (browser-&gt;DNS-&gt;LB-&gt;server-&gt;DB-&gt;back) | ✅     | 🟥     |
| Back-of-envelope estimation (QPS / storage / bandwidth)                          | ✅     | 🟥     |
| Latency numbers every engineer should know                                       | ✅     | 🟥     |
| Availability, reliability, scalability, maintainability                          | ⚪     | 🟥     |
| SLA / SLO / SLI                                                                  | ⚪     | 🟧     |
| Vertical vs horizontal scaling                                                   | ⚪     | 🟥     |
| Percentiles and tail latency                                                     | ⚪     | 🟥     |
| Throughput vs latency                                                            | ⚪     | 🟥     |
| Little's Law                                                                     | ⚪     | 🟧     |
| Universal Scalability Law                                                        | ⚪     | 🟨     |

## L1. Networking

**Goal:** Know how traffic reaches, enters, and moves through a system, and the protocols/edge components involved. · **Prereqs:** L0.

| Topic                                             | Status | Weight |
| ------------------------------------------------- | ------ | ------ |
| OSI and TCP/IP models                             | ⚪     | 🟧     |
| IP addressing and subnets                         | ⚪     | 🟧     |
| DNS deep (resolution, records, GeoDNS, caching)   | ⚪     | 🟥     |
| TCP (handshake, flow control, congestion control) | ⚪     | 🟥     |
| UDP                                               | ⚪     | 🟧     |
| HTTP/1.1, HTTP/2, HTTP/3 (QUIC)                   | ⚪     | 🟥     |
| HTTPS / TLS handshake                             | ⚪     | 🟥     |
| WebSockets / SSE / long-polling                   | ⚪     | 🟥     |
| REST vs gRPC vs GraphQL                           | ⚪     | 🟥     |
| Sockets                                           | ⚪     | 🟧     |
| Forward and reverse proxies                       | ⚪     | 🟥     |
| NAT                                               | ⚪     | 🟨     |
| Load balancers (L4/L7, algorithms, health checks) | ⚪     | 🟥     |
| API gateway                                       | ⚪     | 🟧     |
| CDN internals                                     | ⚪     | 🟥     |
| Anycast / BGP basics                              | ⚪     | 🟨     |
| WebRTC                                            | ⚪     | 🟨     |

## L2. Storage and Relational Databases

**Goal:** Master the relational model and how a single database engine actually stores, indexes, and transacts data. · **Prereqs:** L0.

| Topic                                                        | Status | Weight |
| ------------------------------------------------------------ | ------ | ------ |
| Relational model                                             | ⚪     | 🟥     |
| Normalization forms                                          | ⚪     | 🟧     |
| SQL depth (joins, aggregation, subqueries, window functions) | ⚪     | 🟥     |
| ACID                                                         | ⚪     | 🟥     |
| Transactions and isolation levels                            | ⚪     | 🟥     |
| MVCC                                                         | ⚪     | 🟧     |
| Locking (row/table, optimistic/pessimistic)                  | ⚪     | 🟧     |
| Indexing (B-tree, hash, LSM-tree)                            | ⚪     | 🟥     |
| Write-ahead log (WAL)                                        | ⚪     | 🟧     |
| Storage engines                                              | ⚪     | 🟧     |
| Query planning and optimization                              | ⚪     | 🟧     |
| Connection pooling                                           | ⚪     | 🟧     |
| OLTP vs OLAP                                                 | ⚪     | 🟥     |

## L3. Caching and Data Access

**Goal:** Speed up reads and offload databases with the right caching layer, strategy, and failure handling. · **Prereqs:** L2.

| Topic                                                                                | Status | Weight |
| ------------------------------------------------------------------------------------ | ------ | ------ |
| Caching layers and strategies (read-through, write-through, write-back, cache-aside) | ⚪     | 🟥     |
| Eviction policies (LRU, LFU, TTL)                                                    | ⚪     | 🟥     |
| Redis vs Memcached                                                                   | ⚪     | 🟥     |
| Cache stampede / dogpile / thundering herd                                           | ⚪     | 🟧     |
| Cache coherence and invalidation                                                     | ⚪     | 🟧     |
| Negative caching                                                                     | ⚪     | 🟨     |
| CDN caching                                                                          | ⚪     | 🟧     |
| Object / blob storage                                                                | ⚪     | 🟥     |

## L4. NoSQL and Data at Scale

**Goal:** Model, replicate, and partition data across many machines for large workloads. · **Prereqs:** L2, L3.

| Topic                                                                                  | Status | Weight |
| -------------------------------------------------------------------------------------- | ------ | ------ |
| NoSQL families (KV, document, wide-column, graph, time-series, search, NewSQL, vector) | ⚪     | 🟥     |
| Replication (leader-follower, multi-leader, leaderless)                                | ⚪     | 🟥     |
| Partitioning and sharding                                                              | ⚪     | 🟥     |
| Rebalancing and hotspots                                                               | ⚪     | 🟧     |
| Consistent hashing (virtual nodes)                                                     | ⚪     | 🟥     |
| Data modeling and denormalization                                                      | ⚪     | 🟥     |
| Quorums (R + W &gt; N)                                                                 | ⚪     | 🟧     |
| Change data capture (CDC) + outbox pattern                                             | ⚪     | 🟧     |
| Event sourcing                                                                         | ⚪     | 🟧     |
| CQRS                                                                                   | ⚪     | 🟧     |
| Vector databases / ANN search (HNSW) (emerging)                                        | ⚪     | 🟨     |
| Real-time OLAP (Pinot, Druid, ClickHouse) (emerging)                                   | ⚪     | 🟨     |
| HTAP (hybrid transactional/analytical) (emerging)                                      | ⚪     | 🟨     |
| Database branching / serverless DBs (Neon, PlanetScale) (emerging)                     | ⚪     | 🟨     |
| Data contracts (schema-registry-enforced) (emerging)                                   | ⚪     | 🟨     |
| Hybrid Logical Clocks vs TrueTime                                                      | ⚪     | 🟨     |

## L5. Distributed Systems Theory

**Goal:** Reason rigorously about consistency, consensus, time, and coordination under partial failure. · **Prereqs:** L4.

| Topic                                                               | Status | Weight |
| ------------------------------------------------------------------- | ------ | ------ |
| CAP and PACELC                                                      | ⚪     | 🟥     |
| Consistency models (strong, eventual, causal, read-your-writes)     | ⚪     | 🟥     |
| Linearizability vs serializability                                  | ⚪     | 🟧     |
| Consensus (Paxos, Raft, ZAB)                                        | ⚪     | 🟥     |
| Leader election                                                     | ⚪     | 🟧     |
| Quorums                                                             | ⚪     | 🟧     |
| Logical and vector clocks                                           | ⚪     | 🟧     |
| Hybrid logical clocks                                               | ⚪     | 🟨     |
| Gossip protocol                                                     | ⚪     | 🟧     |
| Failure detectors (phi accrual)                                     | ⚪     | 🟨     |
| Merkle trees / anti-entropy / read-repair / hinted handoff          | ⚪     | 🟧     |
| Distributed locking and fencing tokens                              | ⚪     | 🟧     |
| Idempotency                                                         | ⚪     | 🟥     |
| Delivery semantics (at-most/at-least/exactly-once)                  | ⚪     | 🟥     |
| 2PC / 3PC and saga                                                  | ⚪     | 🟧     |
| FLP impossibility                                                   | ⚪     | 🟨     |
| Byzantine fault tolerance                                           | ⚪     | 🟨     |
| CRDTs                                                               | ⚪     | 🟧     |
| Chain replication                                                   | ⚪     | 🟨     |
| Split-brain                                                         | ⚪     | 🟧     |
| Durable execution / workflow engines (Temporal, Cadence) (emerging) | ⚪     | 🟨     |
| Deterministic simulation testing (emerging)                         | ⚪     | 🟨     |

## L6. Messaging and Streaming

**Goal:** Move data asynchronously between services and process unbounded streams correctly. · **Prereqs:** L4.

| Topic                                                       | Status | Weight |
| ----------------------------------------------------------- | ------ | ------ |
| Queues vs logs                                              | ⚪     | 🟥     |
| Pub/sub                                                     | ⚪     | 🟥     |
| Kafka internals (partitions, consumer groups, offsets, ISR) | ⚪     | 🟥     |
| RabbitMQ / SQS                                              | ⚪     | 🟧     |
| Ordering guarantees                                         | ⚪     | 🟧     |
| Exactly-once processing                                     | ⚪     | 🟧     |
| Dead-letter queues                                          | ⚪     | 🟧     |
| Backpressure                                                | ⚪     | 🟧     |
| Stream processing (Flink, Kafka Streams, Spark)             | ⚪     | 🟧     |
| Event-time vs processing-time                               | ⚪     | 🟧     |
| Windows and watermarks                                      | ⚪     | 🟨     |
| Lambda vs Kappa architecture                                | ⚪     | 🟨     |

## L7. Reliability and Resilience (SRE)

**Goal:** Keep systems fast, resilient, and available under failure and load. · **Prereqs:** L3.

| Topic                                                          | Status | Weight |
| -------------------------------------------------------------- | ------ | ------ |
| Rate limiting (token/leaky bucket, sliding window)             | ⚪     | 🟥     |
| Circuit breakers                                               | ⚪     | 🟥     |
| Bulkheads                                                      | ⚪     | 🟧     |
| Retries with backoff + jitter                                  | ⚪     | 🟥     |
| Hedged requests (with caveats)                                 | ⚪     | 🟨     |
| Retry storms                                                   | ⚪     | 🟧     |
| Load shedding                                                  | ⚪     | 🟧     |
| Admission control                                              | ⚪     | 🟧     |
| Backpressure                                                   | ⚪     | 🟧     |
| Redundancy and failover                                        | ⚪     | 🟥     |
| Multi-region and disaster recovery                             | ⚪     | 🟧     |
| Graceful degradation                                           | ⚪     | 🟧     |
| Health checks                                                  | ⚪     | 🟧     |
| Capacity planning                                              | ⚪     | 🟧     |
| Autoscaling                                                    | ⚪     | 🟧     |
| Deploys (blue-green, canary, rolling, shadow)                  | ⚪     | 🟥     |
| Progressive delivery + feature flags                           | ⚪     | 🟧     |
| Chaos engineering                                              | ⚪     | 🟧     |
| Incident response / postmortems / on-call                      | ⚪     | 🟧     |
| Error-budget math                                              | ⚪     | 🟧     |
| Platform engineering / internal developer platforms (emerging) | ⚪     | 🟨     |
| FinOps embedded in provisioning (emerging)                     | ⚪     | 🟨     |

## L8. Observability

**Goal:** See inside a running system - measure, trace, and alert on what matters without drowning in cost. · **Prereqs:** L7.

| Topic                                              | Status | Weight |
| -------------------------------------------------- | ------ | ------ |
| Metrics (types, cardinality)                       | ⚪     | 🟥     |
| High-cardinality metrics management                | ⚪     | 🟧     |
| Structured logging and aggregation                 | ⚪     | 🟥     |
| Distributed tracing (OpenTelemetry)                | ⚪     | 🟥     |
| The three pillars (metrics, logs, traces)          | ⚪     | 🟥     |
| Continuous profiling (the "4th pillar") (emerging) | ⚪     | 🟨     |
| RED and USE methods                                | ⚪     | 🟧     |
| Alerting                                           | ⚪     | 🟧     |
| Dashboards                                         | ⚪     | 🟨     |
| SLIs                                               | ⚪     | 🟧     |

## L9. Security

**Goal:** Secure identity, data, and the perimeter; handle abuse, privacy, and compliance. · **Prereqs:** L1.

| Topic                                      | Status | Weight |
| ------------------------------------------ | ------ | ------ |
| Authentication vs authorization            | ⚪     | 🟥     |
| OAuth2 / OIDC                              | ⚪     | 🟥     |
| JWT vs sessions                            | ⚪     | 🟥     |
| Cookies                                    | ⚪     | 🟧     |
| Password hashing (bcrypt, argon2, salting) | ⚪     | 🟧     |
| HMAC                                       | ⚪     | 🟧     |
| TLS / mTLS                                 | ⚪     | 🟥     |
| Encryption at rest and in transit          | ⚪     | 🟥     |
| Key management and secrets                 | ⚪     | 🟧     |
| RBAC / ABAC                                | ⚪     | 🟧     |
| Zero trust                                 | ⚪     | 🟧     |
| CORS / CSP                                 | ⚪     | 🟧     |
| DDoS mitigation / WAF                      | ⚪     | 🟧     |
| API abuse prevention                       | ⚪     | 🟧     |
| Threat modeling                            | ⚪     | 🟧     |
| Multi-tenancy                              | ⚪     | 🟧     |
| Privacy and compliance (GDPR, PCI, HIPAA)  | ⚪     | 🟧     |
| Passkeys / FIDO2 (emerging)                | ⚪     | 🟨     |
| SBOM / supply-chain security (emerging)    | ⚪     | 🟨     |

## L10. API and Service Design

**Goal:** Design clean, evolvable, robust interfaces between services and clients. · **Prereqs:** L1.

| Topic                                 | Status | Weight |
| ------------------------------------- | ------ | ------ |
| REST design and maturity (Richardson) | ⚪     | 🟥     |
| Pagination                            | ⚪     | 🟧     |
| Versioning                            | ⚪     | 🟧     |
| Idempotency keys                      | ⚪     | 🟥     |
| gRPC                                  | ⚪     | 🟧     |
| GraphQL                               | ⚪     | 🟧     |
| Webhooks                              | ⚪     | 🟧     |
| Async / long-running APIs             | ⚪     | 🟧     |
| API gateways                          | ⚪     | 🟧     |
| Backends-for-frontends (BFF)          | ⚪     | 🟧     |
| Contract testing                      | ⚪     | 🟨     |

## L11. Architecture Patterns

**Goal:** Choose and justify system-level structure and data-movement patterns. · **Prereqs:** L2, L5.

| Topic                                                                 | Status | Weight |
| --------------------------------------------------------------------- | ------ | ------ |
| Monolith vs microservices vs modular monolith                         | ⚪     | 🟥     |
| Service discovery                                                     | ⚪     | 🟧     |
| Service mesh (Envoy, Istio)                                           | ⚪     | 🟧     |
| Ambient / sidecar-less mesh (emerging)                                | ⚪     | 🟨     |
| Event-driven architecture                                             | ⚪     | 🟥     |
| Outbox pattern                                                        | ⚪     | 🟧     |
| Serverless / FaaS                                                     | ⚪     | 🟧     |
| Hexagonal / clean architecture                                        | ⚪     | 🟧     |
| Strangler-fig pattern                                                 | ⚪     | 🟨     |
| SOA                                                                   | ⚪     | 🟨     |
| Cell-based architecture (emerging)                                    | ⚪     | 🟨     |
| Data pipelines (ETL / ELT / zero-ETL)                                 | ⚪     | 🟧     |
| Data lakes / warehouses / lakehouse (Iceberg, Delta, Hudi) (emerging) | ⚪     | 🟨     |

## L12. Scalability and Performance Patterns

**Goal:** Apply the concrete techniques that make systems scale reads, writes, and geography. · **Prereqs:** L4, L7.

| Topic                                                                  | Status | Weight |
| ---------------------------------------------------------------------- | ------ | ------ |
| Fan-out on write vs fan-out on read                                    | ⚪     | 🟥     |
| Read/write splitting                                                   | ⚪     | 🟧     |
| Geo-distribution / geo-partitioning                                    | ⚪     | 🟧     |
| Sharding patterns                                                      | ⚪     | 🟥     |
| Hot-key mitigation                                                     | ⚪     | 🟧     |
| Batching                                                               | ⚪     | 🟧     |
| Connection pooling                                                     | ⚪     | 🟧     |
| Probabilistic structures (Bloom filter, HyperLogLog, Count-Min Sketch) | ⚪     | 🟧     |
| Quotas                                                                 | ⚪     | 🟨     |

## L13. Specialized Systems

**Goal:** Design domain-specific subsystems: real-time, geospatial, search, ML, and AI-serving. · **Prereqs:** L4, L6.

| Topic                                                                        | Status | Weight |
| ---------------------------------------------------------------------------- | ------ | ------ |
| Real-time delivery at scale (WebSockets/SSE)                                 | ⚪     | 🟥     |
| Push notifications                                                           | ⚪     | 🟧     |
| Geospatial (geohash, quadtree, S2)                                           | ⚪     | 🟧     |
| Search and typeahead (inverted index)                                        | ⚪     | 🟥     |
| Recommendation systems                                                       | ⚪     | 🟧     |
| ML system design (feature stores, model serving, inference)                  | ⚪     | 🟧     |
| Big data (MapReduce, Hadoop, Spark)                                          | ⚪     | 🟧     |
| Real-time analytics / OLAP                                                   | ⚪     | 🟧     |
| LLM inference serving (vLLM, PagedAttention) (emerging)                      | ⚪     | 🟨     |
| RAG architecture (routing, reranking, eval-as-CI) (emerging)                 | ⚪     | 🟨     |
| Model routing (emerging)                                                     | ⚪     | 🟨     |
| GPU scheduling / prefill-decode disaggregation (emerging)                    | ⚪     | 🟨     |
| Fintech: idempotency + double-entry ledgers                                  | ⚪     | 🟧     |
| Live-streaming latency-tiered protocols (WebRTC, LL-HLS, RTMP; MOQ emerging) | ⚪     | 🟨     |

## L14. Cloud and Infrastructure

**Goal:** Package, deploy, network, and operate systems on modern cloud and edge platforms. · **Prereqs:** L7.

| Topic                              | Status | Weight |
| ---------------------------------- | ------ | ------ |
| Containers (Docker)                | ⚪     | 🟥     |
| Orchestration (Kubernetes deep)    | ⚪     | 🟥     |
| Infrastructure as code (Terraform) | ⚪     | 🟧     |
| CI/CD                              | ⚪     | 🟥     |
| GitOps                             | ⚪     | 🟧     |
| Cloud primitives                   | ⚪     | 🟧     |
| Cloud networking                   | ⚪     | 🟧     |
| Cost optimization                  | ⚪     | 🟧     |
| Multi-region active-active         | ⚪     | 🟧     |
| Edge computing                     | ⚪     | 🟧     |
| WASM at the edge (emerging)        | ⚪     | 🟨     |
| eBPF (emerging)                    | ⚪     | 🟨     |

## L15. Applied Design Practice

**Goal:** Apply everything under interview conditions; each attempt goes in `practice/` and is graded. This level carries the **highest interview weight**. · **Prereqs:** L0-L5 minimum, plus the levels each problem exercises.

**Craft (learn before / alongside the problems):**

| Topic                           | Status | Weight |
| ------------------------------- | ------ | ------ |
| The design framework            | ⚪     | 🟥     |
| Driving ambiguity and scoping   | ⚪     | 🟥     |
| Trade-off articulation          | ⚪     | 🟥     |
| Whiteboarding and communication | ⚪     | 🟥     |

**Classic problems:**

| Problem                     | Status | Weight |
| --------------------------- | ------ | ------ |
| URL shortener               | ⚪     | 🟥     |
| Pastebin                    | ⚪     | 🟧     |
| Rate limiter                | ⚪     | 🟥     |
| Twitter / newsfeed          | ⚪     | 🟥     |
| Instagram                   | ⚪     | 🟧     |
| WhatsApp chat               | ⚪     | 🟥     |
| YouTube / Netflix streaming | ⚪     | 🟥     |
| Uber location               | ⚪     | 🟧     |
| Google Drive / Dropbox sync | ⚪     | 🟧     |
| Web crawler                 | ⚪     | 🟧     |
| Typeahead / autocomplete    | ⚪     | 🟥     |
| Notification system         | ⚪     | 🟧     |
| Ticketmaster                | ⚪     | 🟧     |
| Payments / e-commerce       | ⚪     | 🟥     |
| Distributed cache           | ⚪     | 🟧     |
| Job scheduler               | ⚪     | 🟧     |
| Log / metrics analytics     | ⚪     | 🟧     |
| Ad-click aggregator         | ⚪     | 🟧     |
| Leaderboard                 | ⚪     | 🟨     |
| Design Kafka                | ⚪     | 🟧     |
| Design DynamoDB             | ⚪     | 🟧     |

---

# Frontend System-Design Track

Parallel/adjacent track. Can start once backend **L0-L1** material exists (needs request lifecycle + HTTP/CDN context).

## FE-F. Frontend Fundamentals

**Goal:** Understand what the browser and JS runtime actually do so every FE design choice rests on real mechanics, not magic. · **Prereqs:** backend L0-L1.

| Topic                                                                                                 | Status | Weight |
| ----------------------------------------------------------------------------------------------------- | ------ | ------ |
| JS engine internals (V8, JIT, hidden classes)                                                         | ⚪     | 🟥     |
| Event loop; microtasks vs macrotasks; call stack                                                      | ⚪     | 🟥     |
| Rendering pipeline (parse -&gt; DOM/CSSOM -&gt; render tree -&gt; layout -&gt; paint -&gt; composite) | ⚪     | 🟥     |
| GPU compositing and layers                                                                            | ⚪     | 🟧     |
| Reflow and repaint                                                                                    | ⚪     | 🟥     |
| Browser memory and garbage collection                                                                 | ⚪     | 🟧     |
| Module systems (ESM vs CommonJS; dual-package hazard)                                                 | ⚪     | 🟧     |
| How CSS works (cascade, specificity, box model)                                                       | ⚪     | 🟧     |
| Browser storage (cookies, localStorage, sessionStorage, IndexedDB, Cache API)                         | ⚪     | 🟥     |
| Browser networking (fetch, client-side HTTP caching, preload scanner)                                 | ⚪     | 🟧     |

## FE-L0. Foundations

**Goal:** Establish the FE system-design vocabulary and a repeatable framework for designing any UI. · **Prereqs:** FE-F.

| Topic                                                                                                                                                         | Status | Weight |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------ |
| FE system-design framework (requirements -&gt; entities -&gt; API contract -&gt; component tree -&gt; data flow -&gt; performance -&gt; a11y/i18n -&gt; wrap) | ⚪     | 🟥     |
| Component architecture basics                                                                                                                                 | ⚪     | 🟥     |
| Separation of concerns                                                                                                                                        | ⚪     | 🟧     |
| Critical rendering path recap                                                                                                                                 | ⚪     | 🟥     |

## FE-L1. Rendering and Delivery

**Goal:** Choose a rendering strategy and ship assets efficiently across the network. · **Prereqs:** FE-L0, backend L1.

| Topic                                                                        | Status | Weight |
| ---------------------------------------------------------------------------- | ------ | ------ |
| CSR / SSR / SSG / ISR / streaming SSR                                        | ⚪     | 🟥     |
| React Server Components                                                      | ⚪     | 🟧     |
| Suspense and concurrent rendering                                            | ⚪     | 🟧     |
| Islands architecture / partial hydration                                     | ⚪     | 🟧     |
| Resumability (Qwik) (emerging)                                               | ⚪     | 🟨     |
| Partial prerendering (emerging)                                              | ⚪     | 🟨     |
| View Transitions API (emerging)                                              | ⚪     | 🟨     |
| Edge rendering (emerging)                                                    | ⚪     | 🟨     |
| Server-driven UI (SDUI) - backend ships the component tree/layout (emerging) | ⚪     | 🟧     |
| Isomorphic / universal rendering                                             | ⚪     | 🟧     |
| Custom renderers / reconcilers (constrained-device UIs, e.g. TV)             | ⚪     | 🟨     |
| Hydration and hydration cost                                                 | ⚪     | 🟥     |
| Bundling / tree-shaking / code-splitting                                     | ⚪     | 🟥     |
| Asset caching and CDN                                                        | ⚪     | 🟧     |
| Image and font optimization                                                  | ⚪     | 🟧     |
| Critical CSS                                                                 | ⚪     | 🟧     |

## FE-L2. Data and State

**Goal:** Manage client/server data, reactivity, caching, and large lists correctly. · **Prereqs:** FE-L1.

| Topic                                                                                 | Status | Weight |
| ------------------------------------------------------------------------------------- | ------ | ------ |
| Client state vs server state                                                          | ⚪     | 🟥     |
| Server-cache (TanStack Query / SWR)                                                   | ⚪     | 🟥     |
| Client stores (Zustand, Jotai, Redux; Zustand-over-Redux trend)                       | ⚪     | 🟧     |
| Signals / fine-grained reactivity (Solid, Svelte 5 runes, Angular signals) (emerging) | ⚪     | 🟨     |
| React Compiler auto-memoization (emerging)                                            | ⚪     | 🟨     |
| State machines (XState)                                                               | ⚪     | 🟨     |
| URL state                                                                             | ⚪     | 🟧     |
| Form state                                                                            | ⚪     | 🟧     |
| Normalization                                                                         | ⚪     | 🟧     |
| Pagination and infinite scroll                                                        | ⚪     | 🟥     |
| List virtualization                                                                   | ⚪     | 🟧     |
| Optimistic updates                                                                    | ⚪     | 🟧     |
| Prefetching                                                                           | ⚪     | 🟧     |
| Client data fetching (GraphQL vs REST vs tRPC (emerging), Server Actions (emerging))  | ⚪     | 🟧     |
| BFF pattern                                                                           | ⚪     | 🟧     |
| GraphQL Federation / one-graph aggregation for the client                             | ⚪     | 🟧     |
| SDUI data contracts (sections/screens/actions schema)                                 | ⚪     | 🟨     |

## FE-L3. Performance

**Goal:** Measure and hit performance targets that map to real user experience and revenue. · **Prereqs:** FE-L1.

| Topic                                                       | Status | Weight |
| ----------------------------------------------------------- | ------ | ------ |
| Core Web Vitals (LCP / INP / CLS; INP replaced FID in 2024) | ⚪     | 🟥     |
| Performance budgets                                         | ⚪     | 🟧     |
| Resource hints (preload / prefetch / preconnect)            | ⚪     | 🟧     |
| Code-splitting and lazy loading                             | ⚪     | 🟥     |
| Critical CSS                                                | ⚪     | 🟧     |
| Compression (Brotli)                                        | ⚪     | 🟧     |
| Long tasks and scheduler.yield (emerging)                   | ⚪     | 🟨     |
| Hydration cost                                              | ⚪     | 🟧     |
| Debounce and throttle                                       | ⚪     | 🟧     |
| Memoization                                                 | ⚪     | 🟧     |
| Web workers                                                 | ⚪     | 🟧     |
| Bundle analysis                                             | ⚪     | 🟧     |

## FE-L4. Real-Time, Offline and Collaboration

**Goal:** Build live, offline-capable, and collaborative UIs with correct conflict handling. · **Prereqs:** FE-L2, backend L13.

| Topic                                                             | Status | Weight |
| ----------------------------------------------------------------- | ------ | ------ |
| WebSockets / SSE / polling (client-side)                          | ⚪     | 🟥     |
| WebRTC                                                            | ⚪     | 🟧     |
| WebTransport (emerging)                                           | ⚪     | 🟨     |
| CRDT vs OT (Yjs and custom)                                       | ⚪     | 🟧     |
| Presence and conflict resolution                                  | ⚪     | 🟧     |
| PWA and service workers                                           | ⚪     | 🟧     |
| Offline-first / local-first (emerging)                            | ⚪     | 🟨     |
| IndexedDB and Cache API                                           | ⚪     | 🟧     |
| Streaming AI/LLM UI responses (SSE, partial rendering) (emerging) | ⚪     | 🟨     |

## FE-L5. Tooling, Build and Testing

**Goal:** Build, split, and verify frontends at team and org scale with modern toolchains. · **Prereqs:** FE-L1.

| Topic                                                                    | Status | Weight |
| ------------------------------------------------------------------------ | ------ | ------ |
| Bundlers (Vite/Rolldown, esbuild, Rspack (emerging), Turbopack, webpack) | ⚪     | 🟥     |
| Monorepos                                                                | ⚪     | 🟧     |
| Micro-frontends: what and why (independent deploys, team/org autonomy)   | ⚪     | 🟧     |
| MFE composition: build-time integration (published packages)             | ⚪     | 🟨     |
| MFE composition: server-side / edge composition                          | ⚪     | 🟨     |
| MFE composition: run-time (iframes, JS entry, Web Components)            | ⚪     | 🟨     |
| Module Federation (webpack / Rspack)                                     | ⚪     | 🟧     |
| MFE cross-app routing, communication and shared state                    | ⚪     | 🟧     |
| MFE style isolation and a shared design system                           | ⚪     | 🟧     |
| MFE shared dependency / version management (duplication cost)            | ⚪     | 🟧     |
| Micro-frontend trade-offs and when NOT to use them                       | ⚪     | 🟥     |
| Progressive delivery for frontend bundles (canary + rollback)            | ⚪     | 🟨     |
| Testing (unit and integration)                                           | ⚪     | 🟥     |
| E2E testing (Playwright)                                                 | ⚪     | 🟧     |
| Component testing                                                        | ⚪     | 🟧     |
| Visual regression testing                                                | ⚪     | 🟨     |

## FE-L6. Accessibility, i18n and Design Systems

**Goal:** Ship accessible, international UIs with a reusable, well-designed component system. · **Prereqs:** FE-L0.

| Topic                                                                                   | Status | Weight |
| --------------------------------------------------------------------------------------- | ------ | ------ |
| Accessibility (WCAG 2.2, ARIA, focus management, screen readers, keyboard navigation)   | ⚪     | 🟥     |
| Internationalization (i18n) and localization                                            | ⚪     | 🟧     |
| Design systems (design tokens, theming, headless/unstyled components e.g. Radix/shadcn) | ⚪     | 🟧     |
| Component API design                                                                    | ⚪     | 🟧     |
| Web Components (niche, not a React replacement)                                         | ⚪     | 🟨     |

## FE-L7. Frontend Observability and Security

**Goal:** See inside the running client and defend it against abuse, leaks, and supply-chain attacks. · **Prereqs:** FE-L3.

| Topic                                       | Status | Weight |
| ------------------------------------------- | ------ | ------ |
| RUM and Core Web Vitals field data          | ⚪     | 🟧     |
| Error tracking (Sentry)                     | ⚪     | 🟧     |
| Session replay (with privacy scrubbing)     | ⚪     | 🟨     |
| Cross-stack error-to-trace correlation      | ⚪     | 🟨     |
| Client security (XSS, CSRF, CSP, SRI, CORS) | ⚪     | 🟥     |
| npm supply-chain security (emerging risk)   | ⚪     | 🟨     |
| iframe sandboxing                           | ⚪     | 🟨     |

## FE-L8. Applied Problems

**Goal:** Apply the FE track under interview conditions; attempts go in `practice/` and are graded. This level carries the **highest interview weight**. · **Prereqs:** FE-L0 + the FE levels each problem exercises.

| Problem                                       | Status | Weight |
| --------------------------------------------- | ------ | ------ |
| Typeahead widget                              | ⚪     | 🟥     |
| Infinite feed                                 | ⚪     | 🟥     |
| Carousel                                      | ⚪     | 🟧     |
| Google Docs collaborative editor              | ⚪     | 🟧     |
| Chat UI                                       | ⚪     | 🟥     |
| Video player (with LL-HLS for low latency)    | ⚪     | 🟧     |
| E-commerce PDP (performance as revenue lever) | ⚪     | 🟥     |
| Design system                                 | ⚪     | 🟧     |
| Live dashboard                                | ⚪     | 🟧     |
| Spreadsheet                                   | ⚪     | 🟨     |
| Collaborative whiteboard                      | ⚪     | 🟨     |

---

# Side Tracks (optional, not sequenced here)

| Track              | Notes                                                               |
| ------------------ | ------------------------------------------------------------------- |
| DSA (coding round) | Separate coding-interview prep. Run in parallel on its own cadence. |
| IT-trends R&D      | Web research on emerging tech; feeds `research/` opportunistically. |
