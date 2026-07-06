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

| Status        | Meaning                                          |
| ------------- | ------------------------------------------------ |
| `todo`        | No material generated yet.                       |
| `in-progress` | Material currently being generated.              |
| `done`        | Material generated (concept + examples + lesson).|

Order: `todo` -> `in-progress` -> `done`. Status tracks whether the **material exists**, not whether it was studied/practiced (those are async and not tracked here).

## Weight legend

Weight is a **priority signal, not the focus** - everything gets mastered, but when time is tight, do heavier weights first.

| Weight | Meaning                                                              |
| ------ | ------------------------------------------------------------------- |
| 🟥      | Core - foundational, appears everywhere, master deeply.             |
| 🟧      | Important - common and high-value, expected at senior level.        |
| 🟨      | Good-to-know - rounds out expertise; often emerging or specialized. |

Items marked **(emerging)** are newer or meaningfully evolved since the classic syllabus was written - learn them, but know they are the modern frontier rather than settled canon.

## Recommended starting point

Start at **F. Computing Fundamentals** (backend). It requires no prerequisites and underpins everything above it. Then move to **L0 Foundations**, then L1+ in order.

## What's next (current pointer)

> **F. Computing Fundamentals is `done`** (concept + verified examples + lesson). Next in order:
>
> 1. **L0. System-Design Foundations** - the vocabulary and estimation skills every design uses.
> 2. **L1. Networking** - how traffic reaches and moves through a system.
> 3. Then proceed down the backend track in order (L2+); interleave the frontend track once backend L0-L1 material exists.

---

# Backend / Distributed-Systems Track

## F. Computing Fundamentals

**Goal:** Understand what a single machine actually does so every distributed concept later rests on real mechanics, not magic. · **Prereqs:** none.
**Material:** [Lessons (12 bite-sized)](lessons/backend/F/README.md) · [Concepts](research/backend/F/f-computing-fundamentals.md) · [Case studies & sources](research/backend/F/f-computing-fundamentals-cases-and-sources.md)

| Topic                                                                     | Status | Weight |
| ------------------------------------------------------------------------- | ------ | ------ |
| CPU and memory hierarchy (registers, caches, cache lines, locality)       | done   | 🟥      |
| Processes vs threads                                                       | done   | 🟥      |
| Concurrency vs parallelism; context switching                             | done   | 🟥      |
| Locks, mutexes, semaphores; race conditions; deadlock; atomicity          | done   | 🟥      |
| I/O models (blocking, non-blocking, async, epoll, event loops)            | done   | 🟥      |
| OS scheduling and virtual memory (paging, TLB)                            | done   | 🟧      |
| Disks (HDD/SSD/NVMe) and filesystems                                       | done   | 🟧      |
| Data representation (binary/hex, ASCII/Unicode/UTF-8, endianness)         | done   | 🟧      |
| Serialization (JSON, XML, Protobuf, Avro, Thrift)                         | done   | 🟥      |
| Compression (gzip, Snappy, LZ4, Zstd)                                     | done   | 🟧      |
| Hashing (crypto vs non-crypto, collisions, checksums/CRC)                 | done   | 🟥      |
| Clocks (monotonic vs wall clock, NTP)                                     | done   | 🟧      |

## L0. System-Design Foundations

**Goal:** Understand what system design is and the vocabulary/estimation skills every design uses. · **Prereqs:** F.

| Topic                                                        | Status | Weight |
| ------------------------------------------------------------ | ------ | ------ |
| What system design is; the design mindset                    | todo   | 🟥      |
| Functional vs non-functional requirements                    | todo        | 🟥      |
| Client-server model                                          | todo        | 🟥      |
| Request lifecycle end-to-end (browser->DNS->LB->server->DB->back) | todo   | 🟥      |
| Back-of-envelope estimation (QPS / storage / bandwidth)      | todo        | 🟥      |
| Latency numbers every engineer should know                   | todo        | 🟥      |
| Availability, reliability, scalability, maintainability      | todo        | 🟥      |
| SLA / SLO / SLI                                              | todo        | 🟧      |
| Vertical vs horizontal scaling                               | todo        | 🟥      |
| Percentiles and tail latency                                 | todo        | 🟥      |
| Throughput vs latency                                        | todo        | 🟥      |
| Little's Law                                                 | todo        | 🟧      |
| Universal Scalability Law                                    | todo        | 🟨      |

## L1. Networking

**Goal:** Know how traffic reaches, enters, and moves through a system, and the protocols/edge components involved. · **Prereqs:** L0.

| Topic                                        | Status | Weight |
| -------------------------------------------- | ------ | ------ |
| OSI and TCP/IP models                        | todo   | 🟧      |
| IP addressing and subnets                    | todo   | 🟧      |
| DNS deep (resolution, records, GeoDNS, caching) | todo | 🟥      |
| TCP (handshake, flow control, congestion control) | todo | 🟥    |
| UDP                                          | todo   | 🟧      |
| HTTP/1.1, HTTP/2, HTTP/3 (QUIC)              | todo   | 🟥      |
| HTTPS / TLS handshake                        | todo   | 🟥      |
| WebSockets / SSE / long-polling              | todo   | 🟥      |
| REST vs gRPC vs GraphQL                      | todo   | 🟥      |
| Sockets                                      | todo   | 🟧      |
| Forward and reverse proxies                  | todo   | 🟥      |
| NAT                                          | todo   | 🟨      |
| Load balancers (L4/L7, algorithms, health checks) | todo | 🟥    |
| API gateway                                  | todo   | 🟧      |
| CDN internals                                | todo   | 🟥      |
| Anycast / BGP basics                         | todo   | 🟨      |
| WebRTC                                       | todo   | 🟨      |

## L2. Storage and Relational Databases

**Goal:** Master the relational model and how a single database engine actually stores, indexes, and transacts data. · **Prereqs:** L0.

| Topic                                         | Status | Weight |
| --------------------------------------------- | ------ | ------ |
| Relational model                              | todo   | 🟥      |
| Normalization forms                           | todo   | 🟧      |
| SQL depth (joins, aggregation, subqueries, window functions) | todo | 🟥 |
| ACID                                          | todo   | 🟥      |
| Transactions and isolation levels             | todo   | 🟥      |
| MVCC                                          | todo   | 🟧      |
| Locking (row/table, optimistic/pessimistic)   | todo   | 🟧      |
| Indexing (B-tree, hash, LSM-tree)             | todo   | 🟥      |
| Write-ahead log (WAL)                         | todo   | 🟧      |
| Storage engines                               | todo   | 🟧      |
| Query planning and optimization               | todo   | 🟧      |
| Connection pooling                            | todo   | 🟧      |
| OLTP vs OLAP                                   | todo   | 🟥      |

## L3. Caching and Data Access

**Goal:** Speed up reads and offload databases with the right caching layer, strategy, and failure handling. · **Prereqs:** L2.

| Topic                                                              | Status | Weight |
| ----------------------------------------------------------------- | ------ | ------ |
| Caching layers and strategies (read-through, write-through, write-back, cache-aside) | todo | 🟥 |
| Eviction policies (LRU, LFU, TTL)                                 | todo   | 🟥      |
| Redis vs Memcached                                                | todo   | 🟥      |
| Cache stampede / dogpile / thundering herd                        | todo   | 🟧      |
| Cache coherence and invalidation                                 | todo   | 🟧      |
| Negative caching                                                  | todo   | 🟨      |
| CDN caching                                                       | todo   | 🟧      |
| Object / blob storage                                             | todo   | 🟥      |

## L4. NoSQL and Data at Scale

**Goal:** Model, replicate, and partition data across many machines for large workloads. · **Prereqs:** L2, L3.

| Topic                                                                             | Status | Weight |
| --------------------------------------------------------------------------------- | ------ | ------ |
| NoSQL families (KV, document, wide-column, graph, time-series, search, NewSQL, vector) | todo | 🟥 |
| Replication (leader-follower, multi-leader, leaderless)                            | todo   | 🟥      |
| Partitioning and sharding                                                          | todo   | 🟥      |
| Rebalancing and hotspots                                                           | todo   | 🟧      |
| Consistent hashing (virtual nodes)                                                 | todo   | 🟥      |
| Data modeling and denormalization                                                 | todo   | 🟥      |
| Quorums (R + W > N)                                                                | todo   | 🟧      |
| Change data capture (CDC) + outbox pattern                                         | todo   | 🟧      |
| Event sourcing                                                                     | todo   | 🟧      |
| CQRS                                                                               | todo   | 🟧      |
| Vector databases / ANN search (HNSW) (emerging)                                    | todo   | 🟨      |
| Real-time OLAP (Pinot, Druid, ClickHouse) (emerging)                              | todo   | 🟨      |
| HTAP (hybrid transactional/analytical) (emerging)                                | todo   | 🟨      |
| Database branching / serverless DBs (Neon, PlanetScale) (emerging)                | todo   | 🟨      |
| Data contracts (schema-registry-enforced) (emerging)                              | todo   | 🟨      |
| Hybrid Logical Clocks vs TrueTime                                                 | todo   | 🟨      |

## L5. Distributed Systems Theory

**Goal:** Reason rigorously about consistency, consensus, time, and coordination under partial failure. · **Prereqs:** L4.

| Topic                                                             | Status | Weight |
| ---------------------------------------------------------------- | ------ | ------ |
| CAP and PACELC                                                   | todo   | 🟥      |
| Consistency models (strong, eventual, causal, read-your-writes) | todo   | 🟥      |
| Linearizability vs serializability                              | todo   | 🟧      |
| Consensus (Paxos, Raft, ZAB)                                     | todo   | 🟥      |
| Leader election                                                 | todo   | 🟧      |
| Quorums                                                          | todo   | 🟧      |
| Logical and vector clocks                                       | todo   | 🟧      |
| Hybrid logical clocks                                           | todo   | 🟨      |
| Gossip protocol                                                 | todo   | 🟧      |
| Failure detectors (phi accrual)                                 | todo   | 🟨      |
| Merkle trees / anti-entropy / read-repair / hinted handoff      | todo   | 🟧      |
| Distributed locking and fencing tokens                          | todo   | 🟧      |
| Idempotency                                                     | todo   | 🟥      |
| Delivery semantics (at-most/at-least/exactly-once)              | todo   | 🟥      |
| 2PC / 3PC and saga                                              | todo   | 🟧      |
| FLP impossibility                                               | todo   | 🟨      |
| Byzantine fault tolerance                                       | todo   | 🟨      |
| CRDTs                                                           | todo   | 🟧      |
| Chain replication                                              | todo   | 🟨      |
| Split-brain                                                     | todo   | 🟧      |
| Durable execution / workflow engines (Temporal, Cadence) (emerging) | todo | 🟨   |
| Deterministic simulation testing (emerging)                     | todo   | 🟨      |

## L6. Messaging and Streaming

**Goal:** Move data asynchronously between services and process unbounded streams correctly. · **Prereqs:** L4.

| Topic                                                        | Status | Weight |
| ------------------------------------------------------------ | ------ | ------ |
| Queues vs logs                                               | todo   | 🟥      |
| Pub/sub                                                      | todo   | 🟥      |
| Kafka internals (partitions, consumer groups, offsets, ISR) | todo   | 🟥      |
| RabbitMQ / SQS                                               | todo   | 🟧      |
| Ordering guarantees                                          | todo   | 🟧      |
| Exactly-once processing                                      | todo   | 🟧      |
| Dead-letter queues                                           | todo   | 🟧      |
| Backpressure                                                 | todo   | 🟧      |
| Stream processing (Flink, Kafka Streams, Spark)             | todo   | 🟧      |
| Event-time vs processing-time                               | todo   | 🟧      |
| Windows and watermarks                                       | todo   | 🟨      |
| Lambda vs Kappa architecture                                 | todo   | 🟨      |

## L7. Reliability and Resilience (SRE)

**Goal:** Keep systems fast, resilient, and available under failure and load. · **Prereqs:** L3.

| Topic                                              | Status | Weight |
| -------------------------------------------------- | ------ | ------ |
| Rate limiting (token/leaky bucket, sliding window) | todo   | 🟥      |
| Circuit breakers                                   | todo   | 🟥      |
| Bulkheads                                          | todo   | 🟧      |
| Retries with backoff + jitter                      | todo   | 🟥      |
| Hedged requests (with caveats)                     | todo   | 🟨      |
| Retry storms                                       | todo   | 🟧      |
| Load shedding                                      | todo   | 🟧      |
| Admission control                                  | todo   | 🟧      |
| Backpressure                                       | todo   | 🟧      |
| Redundancy and failover                            | todo   | 🟥      |
| Multi-region and disaster recovery                 | todo   | 🟧      |
| Graceful degradation                               | todo   | 🟧      |
| Health checks                                      | todo   | 🟧      |
| Capacity planning                                  | todo   | 🟧      |
| Autoscaling                                        | todo   | 🟧      |
| Deploys (blue-green, canary, rolling, shadow)      | todo   | 🟥      |
| Progressive delivery + feature flags               | todo   | 🟧      |
| Chaos engineering                                  | todo   | 🟧      |
| Incident response / postmortems / on-call         | todo   | 🟧      |
| Error-budget math                                  | todo   | 🟧      |
| Platform engineering / internal developer platforms (emerging) | todo | 🟨 |
| FinOps embedded in provisioning (emerging)         | todo   | 🟨      |

## L8. Observability

**Goal:** See inside a running system - measure, trace, and alert on what matters without drowning in cost. · **Prereqs:** L7.

| Topic                                                     | Status | Weight |
| --------------------------------------------------------- | ------ | ------ |
| Metrics (types, cardinality)                              | todo   | 🟥      |
| High-cardinality metrics management                       | todo   | 🟧      |
| Structured logging and aggregation                        | todo   | 🟥      |
| Distributed tracing (OpenTelemetry)                       | todo   | 🟥      |
| The three pillars (metrics, logs, traces)                 | todo   | 🟥      |
| Continuous profiling (the "4th pillar") (emerging)        | todo   | 🟨      |
| RED and USE methods                                       | todo   | 🟧      |
| Alerting                                                  | todo   | 🟧      |
| Dashboards                                                | todo   | 🟨      |
| SLIs                                                      | todo   | 🟧      |

## L9. Security

**Goal:** Secure identity, data, and the perimeter; handle abuse, privacy, and compliance. · **Prereqs:** L1.

| Topic                                            | Status | Weight |
| ------------------------------------------------ | ------ | ------ |
| Authentication vs authorization                  | todo   | 🟥      |
| OAuth2 / OIDC                                     | todo   | 🟥      |
| JWT vs sessions                                  | todo   | 🟥      |
| Cookies                                          | todo   | 🟧      |
| Password hashing (bcrypt, argon2, salting)      | todo   | 🟧      |
| HMAC                                             | todo   | 🟧      |
| TLS / mTLS                                        | todo   | 🟥      |
| Encryption at rest and in transit                | todo   | 🟥      |
| Key management and secrets                        | todo   | 🟧      |
| RBAC / ABAC                                       | todo   | 🟧      |
| Zero trust                                        | todo   | 🟧      |
| CORS / CSP                                        | todo   | 🟧      |
| DDoS mitigation / WAF                             | todo   | 🟧      |
| API abuse prevention                              | todo   | 🟧      |
| Threat modeling                                  | todo   | 🟧      |
| Multi-tenancy                                    | todo   | 🟧      |
| Privacy and compliance (GDPR, PCI, HIPAA)        | todo   | 🟧      |
| Passkeys / FIDO2 (emerging)                       | todo   | 🟨      |
| SBOM / supply-chain security (emerging)          | todo   | 🟨      |

## L10. API and Service Design

**Goal:** Design clean, evolvable, robust interfaces between services and clients. · **Prereqs:** L1.

| Topic                                | Status | Weight |
| ------------------------------------ | ------ | ------ |
| REST design and maturity (Richardson) | todo  | 🟥      |
| Pagination                           | todo   | 🟧      |
| Versioning                           | todo   | 🟧      |
| Idempotency keys                     | todo   | 🟥      |
| gRPC                                 | todo   | 🟧      |
| GraphQL                              | todo   | 🟧      |
| Webhooks                             | todo   | 🟧      |
| Async / long-running APIs            | todo   | 🟧      |
| API gateways                         | todo   | 🟧      |
| Backends-for-frontends (BFF)         | todo   | 🟧      |
| Contract testing                     | todo   | 🟨      |

## L11. Architecture Patterns

**Goal:** Choose and justify system-level structure and data-movement patterns. · **Prereqs:** L2, L5.

| Topic                                                    | Status | Weight |
| -------------------------------------------------------- | ------ | ------ |
| Monolith vs microservices vs modular monolith           | todo   | 🟥      |
| Service discovery                                        | todo   | 🟧      |
| Service mesh (Envoy, Istio)                              | todo   | 🟧      |
| Ambient / sidecar-less mesh (emerging)                  | todo   | 🟨      |
| Event-driven architecture                                | todo   | 🟥      |
| Outbox pattern                                           | todo   | 🟧      |
| Serverless / FaaS                                        | todo   | 🟧      |
| Hexagonal / clean architecture                           | todo   | 🟧      |
| Strangler-fig pattern                                    | todo   | 🟨      |
| SOA                                                      | todo   | 🟨      |
| Cell-based architecture (emerging)                       | todo   | 🟨      |
| Data pipelines (ETL / ELT / zero-ETL)                    | todo   | 🟧      |
| Data lakes / warehouses / lakehouse (Iceberg, Delta, Hudi) (emerging) | todo | 🟨 |

## L12. Scalability and Performance Patterns

**Goal:** Apply the concrete techniques that make systems scale reads, writes, and geography. · **Prereqs:** L4, L7.

| Topic                                                              | Status | Weight |
| ----------------------------------------------------------------- | ------ | ------ |
| Fan-out on write vs fan-out on read                               | todo   | 🟥      |
| Read/write splitting                                              | todo   | 🟧      |
| Geo-distribution / geo-partitioning                              | todo   | 🟧      |
| Sharding patterns                                                 | todo   | 🟥      |
| Hot-key mitigation                                               | todo   | 🟧      |
| Batching                                                         | todo   | 🟧      |
| Connection pooling                                               | todo   | 🟧      |
| Probabilistic structures (Bloom filter, HyperLogLog, Count-Min Sketch) | todo | 🟧 |
| Quotas                                                           | todo   | 🟨      |

## L13. Specialized Systems

**Goal:** Design domain-specific subsystems: real-time, geospatial, search, ML, and AI-serving. · **Prereqs:** L4, L6.

| Topic                                                        | Status | Weight |
| ------------------------------------------------------------ | ------ | ------ |
| Real-time delivery at scale (WebSockets/SSE)                 | todo   | 🟥      |
| Push notifications                                           | todo   | 🟧      |
| Geospatial (geohash, quadtree, S2)                          | todo   | 🟧      |
| Search and typeahead (inverted index)                       | todo   | 🟥      |
| Recommendation systems                                      | todo   | 🟧      |
| ML system design (feature stores, model serving, inference) | todo   | 🟧      |
| Big data (MapReduce, Hadoop, Spark)                         | todo   | 🟧      |
| Real-time analytics / OLAP                                   | todo   | 🟧      |
| LLM inference serving (vLLM, PagedAttention) (emerging)     | todo   | 🟨      |
| RAG architecture (routing, reranking, eval-as-CI) (emerging) | todo  | 🟨      |
| Model routing (emerging)                                     | todo   | 🟨      |
| GPU scheduling / prefill-decode disaggregation (emerging)   | todo   | 🟨      |
| Fintech: idempotency + double-entry ledgers                 | todo   | 🟧      |
| Live-streaming latency-tiered protocols (WebRTC, LL-HLS, RTMP; MOQ emerging) | todo | 🟨 |

## L14. Cloud and Infrastructure

**Goal:** Package, deploy, network, and operate systems on modern cloud and edge platforms. · **Prereqs:** L7.

| Topic                                    | Status | Weight |
| ---------------------------------------- | ------ | ------ |
| Containers (Docker)                      | todo   | 🟥      |
| Orchestration (Kubernetes deep)          | todo   | 🟥      |
| Infrastructure as code (Terraform)       | todo   | 🟧      |
| CI/CD                                    | todo   | 🟥      |
| GitOps                                   | todo   | 🟧      |
| Cloud primitives                         | todo   | 🟧      |
| Cloud networking                         | todo   | 🟧      |
| Cost optimization                        | todo   | 🟧      |
| Multi-region active-active               | todo   | 🟧      |
| Edge computing                           | todo   | 🟧      |
| WASM at the edge (emerging)              | todo   | 🟨      |
| eBPF (emerging)                          | todo   | 🟨      |

## L15. Applied Design Practice

**Goal:** Apply everything under interview conditions; each attempt goes in `practice/` and is graded. This level carries the **highest interview weight**. · **Prereqs:** L0-L5 minimum, plus the levels each problem exercises.

**Craft (learn before / alongside the problems):**

| Topic                                | Status | Weight |
| ------------------------------------ | ------ | ------ |
| The design framework                 | todo   | 🟥      |
| Driving ambiguity and scoping        | todo   | 🟥      |
| Trade-off articulation               | todo   | 🟥      |
| Whiteboarding and communication      | todo   | 🟥      |

**Classic problems:**

| Problem                     | Status | Weight |
| --------------------------- | ------ | ------ |
| URL shortener               | todo   | 🟥      |
| Pastebin                    | todo   | 🟧      |
| Rate limiter                | todo   | 🟥      |
| Twitter / newsfeed          | todo   | 🟥      |
| Instagram                   | todo   | 🟧      |
| WhatsApp chat               | todo   | 🟥      |
| YouTube / Netflix streaming | todo   | 🟥      |
| Uber location               | todo   | 🟧      |
| Google Drive / Dropbox sync | todo   | 🟧      |
| Web crawler                 | todo   | 🟧      |
| Typeahead / autocomplete    | todo   | 🟥      |
| Notification system         | todo   | 🟧      |
| Ticketmaster                | todo   | 🟧      |
| Payments / e-commerce       | todo   | 🟥      |
| Distributed cache           | todo   | 🟧      |
| Job scheduler               | todo   | 🟧      |
| Log / metrics analytics     | todo   | 🟧      |
| Ad-click aggregator         | todo   | 🟧      |
| Leaderboard                 | todo   | 🟨      |
| Design Kafka                | todo   | 🟧      |
| Design DynamoDB             | todo   | 🟧      |

---

# Frontend System-Design Track

Parallel/adjacent track. Can start once backend **L0-L1** material exists (needs request lifecycle + HTTP/CDN context).

## FE-F. Frontend Fundamentals

**Goal:** Understand what the browser and JS runtime actually do so every FE design choice rests on real mechanics, not magic. · **Prereqs:** backend L0-L1.

| Topic                                                                   | Status | Weight |
| ----------------------------------------------------------------------- | ------ | ------ |
| JS engine internals (V8, JIT, hidden classes)                           | todo   | 🟥      |
| Event loop; microtasks vs macrotasks; call stack                        | todo   | 🟥      |
| Rendering pipeline (parse -> DOM/CSSOM -> render tree -> layout -> paint -> composite) | todo | 🟥 |
| GPU compositing and layers                                              | todo   | 🟧      |
| Reflow and repaint                                                      | todo   | 🟥      |
| Browser memory and garbage collection                                  | todo   | 🟧      |
| Module systems (ESM vs CommonJS; dual-package hazard)                   | todo   | 🟧      |
| How CSS works (cascade, specificity, box model)                        | todo   | 🟧      |
| Browser storage (cookies, localStorage, sessionStorage, IndexedDB, Cache API) | todo | 🟥 |
| Browser networking (fetch, client-side HTTP caching, preload scanner)  | todo   | 🟧      |

## FE-L0. Foundations

**Goal:** Establish the FE system-design vocabulary and a repeatable framework for designing any UI. · **Prereqs:** FE-F.

| Topic                                                                                       | Status | Weight |
| ------------------------------------------------------------------------------------------- | ------ | ------ |
| FE system-design framework (requirements -> entities -> API contract -> component tree -> data flow -> performance -> a11y/i18n -> wrap) | todo | 🟥 |
| Component architecture basics                                                               | todo   | 🟥      |
| Separation of concerns                                                                      | todo   | 🟧      |
| Critical rendering path recap                                                               | todo   | 🟥      |

## FE-L1. Rendering and Delivery

**Goal:** Choose a rendering strategy and ship assets efficiently across the network. · **Prereqs:** FE-L0, backend L1.

| Topic                                       | Status | Weight |
| ------------------------------------------- | ------ | ------ |
| CSR / SSR / SSG / ISR / streaming SSR       | todo   | 🟥      |
| React Server Components                     | todo   | 🟧      |
| Suspense and concurrent rendering           | todo   | 🟧      |
| Islands architecture / partial hydration    | todo   | 🟧      |
| Resumability (Qwik) (emerging)              | todo   | 🟨      |
| Partial prerendering (emerging)             | todo   | 🟨      |
| View Transitions API (emerging)             | todo   | 🟨      |
| Edge rendering (emerging)                    | todo   | 🟨      |
| Server-driven UI (SDUI) - backend ships the component tree/layout (emerging) | todo | 🟧 |
| Isomorphic / universal rendering            | todo   | 🟧      |
| Custom renderers / reconcilers (constrained-device UIs, e.g. TV) | todo | 🟨 |
| Hydration and hydration cost                | todo   | 🟥      |
| Bundling / tree-shaking / code-splitting    | todo   | 🟥      |
| Asset caching and CDN                       | todo   | 🟧      |
| Image and font optimization                 | todo   | 🟧      |
| Critical CSS                                | todo   | 🟧      |

## FE-L2. Data and State

**Goal:** Manage client/server data, reactivity, caching, and large lists correctly. · **Prereqs:** FE-L1.

| Topic                                                                            | Status | Weight |
| -------------------------------------------------------------------------------- | ------ | ------ |
| Client state vs server state                                                     | todo   | 🟥      |
| Server-cache (TanStack Query / SWR)                                              | todo   | 🟥      |
| Client stores (Zustand, Jotai, Redux; Zustand-over-Redux trend)                  | todo   | 🟧      |
| Signals / fine-grained reactivity (Solid, Svelte 5 runes, Angular signals) (emerging) | todo | 🟨 |
| React Compiler auto-memoization (emerging)                                       | todo   | 🟨      |
| State machines (XState)                                                          | todo   | 🟨      |
| URL state                                                                        | todo   | 🟧      |
| Form state                                                                       | todo   | 🟧      |
| Normalization                                                                    | todo   | 🟧      |
| Pagination and infinite scroll                                                   | todo   | 🟥      |
| List virtualization                                                              | todo   | 🟧      |
| Optimistic updates                                                               | todo   | 🟧      |
| Prefetching                                                                      | todo   | 🟧      |
| Client data fetching (GraphQL vs REST vs tRPC (emerging), Server Actions (emerging)) | todo | 🟧 |
| BFF pattern                                                                       | todo   | 🟧      |
| GraphQL Federation / one-graph aggregation for the client                        | todo   | 🟧      |
| SDUI data contracts (sections/screens/actions schema)                            | todo   | 🟨      |

## FE-L3. Performance

**Goal:** Measure and hit performance targets that map to real user experience and revenue. · **Prereqs:** FE-L1.

| Topic                                                    | Status | Weight |
| -------------------------------------------------------- | ------ | ------ |
| Core Web Vitals (LCP / INP / CLS; INP replaced FID in 2024) | todo | 🟥      |
| Performance budgets                                      | todo   | 🟧      |
| Resource hints (preload / prefetch / preconnect)         | todo   | 🟧      |
| Code-splitting and lazy loading                          | todo   | 🟥      |
| Critical CSS                                             | todo   | 🟧      |
| Compression (Brotli)                                     | todo   | 🟧      |
| Long tasks and scheduler.yield (emerging)                | todo   | 🟨      |
| Hydration cost                                           | todo   | 🟧      |
| Debounce and throttle                                    | todo   | 🟧      |
| Memoization                                             | todo   | 🟧      |
| Web workers                                             | todo   | 🟧      |
| Bundle analysis                                          | todo   | 🟧      |

## FE-L4. Real-Time, Offline and Collaboration

**Goal:** Build live, offline-capable, and collaborative UIs with correct conflict handling. · **Prereqs:** FE-L2, backend L13.

| Topic                                                        | Status | Weight |
| ------------------------------------------------------------ | ------ | ------ |
| WebSockets / SSE / polling (client-side)                     | todo   | 🟥      |
| WebRTC                                                       | todo   | 🟧      |
| WebTransport (emerging)                                      | todo   | 🟨      |
| CRDT vs OT (Yjs and custom)                                  | todo   | 🟧      |
| Presence and conflict resolution                             | todo   | 🟧      |
| PWA and service workers                                      | todo   | 🟧      |
| Offline-first / local-first (emerging)                       | todo   | 🟨      |
| IndexedDB and Cache API                                      | todo   | 🟧      |
| Streaming AI/LLM UI responses (SSE, partial rendering) (emerging) | todo | 🟨   |

## FE-L5. Tooling, Build and Testing

**Goal:** Build, split, and verify frontends at team and org scale with modern toolchains. · **Prereqs:** FE-L1.

| Topic                                                          | Status | Weight |
| -------------------------------------------------------------- | ------ | ------ |
| Bundlers (Vite/Rolldown, esbuild, Rspack (emerging), Turbopack, webpack) | todo | 🟥 |
| Monorepos                                                      | todo   | 🟧      |
| Micro-frontends: what and why (independent deploys, team/org autonomy) | todo | 🟧 |
| MFE composition: build-time integration (published packages)   | todo   | 🟨      |
| MFE composition: server-side / edge composition                | todo   | 🟨      |
| MFE composition: run-time (iframes, JS entry, Web Components)   | todo   | 🟨      |
| Module Federation (webpack / Rspack)                           | todo   | 🟧      |
| MFE cross-app routing, communication and shared state          | todo   | 🟧      |
| MFE style isolation and a shared design system                 | todo   | 🟧      |
| MFE shared dependency / version management (duplication cost)   | todo   | 🟧      |
| Micro-frontend trade-offs and when NOT to use them             | todo   | 🟥      |
| Progressive delivery for frontend bundles (canary + rollback)  | todo   | 🟨      |
| Testing (unit and integration)                                 | todo   | 🟥      |
| E2E testing (Playwright)                                        | todo   | 🟧      |
| Component testing                                              | todo   | 🟧      |
| Visual regression testing                                      | todo   | 🟨      |

## FE-L6. Accessibility, i18n and Design Systems

**Goal:** Ship accessible, international UIs with a reusable, well-designed component system. · **Prereqs:** FE-L0.

| Topic                                                                              | Status | Weight |
| ---------------------------------------------------------------------------------- | ------ | ------ |
| Accessibility (WCAG 2.2, ARIA, focus management, screen readers, keyboard navigation) | todo | 🟥 |
| Internationalization (i18n) and localization                                       | todo   | 🟧      |
| Design systems (design tokens, theming, headless/unstyled components e.g. Radix/shadcn) | todo | 🟧 |
| Component API design                                                               | todo   | 🟧      |
| Web Components (niche, not a React replacement)                                     | todo   | 🟨      |

## FE-L7. Frontend Observability and Security

**Goal:** See inside the running client and defend it against abuse, leaks, and supply-chain attacks. · **Prereqs:** FE-L3.

| Topic                                            | Status | Weight |
| ------------------------------------------------ | ------ | ------ |
| RUM and Core Web Vitals field data               | todo   | 🟧      |
| Error tracking (Sentry)                          | todo   | 🟧      |
| Session replay (with privacy scrubbing)          | todo   | 🟨      |
| Cross-stack error-to-trace correlation           | todo   | 🟨      |
| Client security (XSS, CSRF, CSP, SRI, CORS)      | todo   | 🟥      |
| npm supply-chain security (emerging risk)        | todo   | 🟨      |
| iframe sandboxing                                | todo   | 🟨      |

## FE-L8. Applied Problems

**Goal:** Apply the FE track under interview conditions; attempts go in `practice/` and are graded. This level carries the **highest interview weight**. · **Prereqs:** FE-L0 + the FE levels each problem exercises.

| Problem                                        | Status | Weight |
| ---------------------------------------------- | ------ | ------ |
| Typeahead widget                               | todo   | 🟥      |
| Infinite feed                                  | todo   | 🟥      |
| Carousel                                       | todo   | 🟧      |
| Google Docs collaborative editor               | todo   | 🟧      |
| Chat UI                                        | todo   | 🟥      |
| Video player (with LL-HLS for low latency)     | todo   | 🟧      |
| E-commerce PDP (performance as revenue lever)  | todo   | 🟥      |
| Design system                                  | todo   | 🟧      |
| Live dashboard                                 | todo   | 🟧      |
| Spreadsheet                                    | todo   | 🟨      |
| Collaborative whiteboard                       | todo   | 🟨      |

---

# Side Tracks (optional, not sequenced here)

| Track              | Notes                                                               |
| ------------------ | ------------------------------------------------------------------- |
| DSA (coding round) | Separate coding-interview prep. Run in parallel on its own cadence. |
| IT-trends R&D      | Web research on emerging tech; feeds `research/` opportunistically. |
