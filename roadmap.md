# System Design Learning Roadmap

The master learning path for this repo. It sequences everything a full-stack engineer needs to go from **true beginner → expert** for MAANGO system-design interviews. It is read at the start of most sessions, so it is kept dense and scannable.

## Two parallel tracks

- **Backend / Distributed-Systems track (L0–L10)** - the core program. This is the main spine.
- **Frontend System-Design track (FE-L0–FE-L6)** - a parallel, adjacent track. Can be interleaved once backend L0–L1 are done.
- Side tracks (optional, not sequenced here): **DSA** (coding round, owned by `dsa-expert`) and **IT-trends R&D** (owned by `it-trends-researcher`).

## How status works

| Status        | Meaning                                                           |
| ------------- | ----------------------------------------------------------------- |
| `todo`        | Not started. No research/lesson yet.                              |
| `in-progress` | Research or lesson being written / actively studied.              |
| `learned`     | Lesson exists and learner has studied it.                         |
| `practiced`   | Applied in a `practice/` attempt and graded by `design-reviewer`. |

Order of maturity: `todo` → `in-progress` → `learned` → `practiced`. A topic is only truly "done" when `practiced`.

## Recommended starting point

Start at **L0 Foundations** (backend). It requires no prerequisites and every later topic depends on it. Do not touch FE track or any L1+ topic until L0 is `learned`.

## What's next (current pointer)

> **L0 research is written** (`research/l0-foundations.md`, all 9 topics). Learner is actively studying L0 Topic 1 (what system design is) live. **No `lessons/` files exist yet** - the gap is turning research into learner-facing lessons.
>
> Continue L0 in prerequisite order. Research already exists, so the immediate move is `tutor` writing lessons from `research/l0-foundations.md` (§2, §3, §4):
>
> 1. _Functional vs non-functional requirements_ (research §2) - owner `system-design-expert`; lesson by `tutor`.
> 2. _Client–server model_ (research §3) - owners `system-design-expert`, `network-expert`; lesson by `tutor`.
> 3. _Request lifecycle end-to-end_ (research §4) - owners `system-design-expert`, `network-expert`; lesson by `tutor`.
>
> After these, finish L0 (§5 estimation, §6 latency, §7 -ilities, §8 SLA/SLO/SLI, §9 scaling), then mark L0 `learned`. No practice problem yet - first attempt is **L10: URL shortener** after L0–L2.

---

# Backend / Distributed-Systems Track

## L0 - Foundations

**Goal:** Understand what system design is and the vocabulary/estimation skills every design uses. · **Prereqs:** none. · **Research:** `research/l0-foundations.md` (complete, all 9 topics).

| Topic                                                        | Status      | Owning expert(s)                         | Related practice problem |
| ------------------------------------------------------------ | ----------- | ---------------------------------------- | ------------------------ |
| What system design is; the design mindset                    | in-progress | system-design-expert                     | -                        |
| Functional vs non-functional requirements                    | todo        | system-design-expert                     | all L10                  |
| Client–server model                                          | todo        | system-design-expert, network-expert     | -                        |
| Request lifecycle end-to-end (browser→DNS→LB→server→DB→back) | todo        | system-design-expert, network-expert     | -                        |
| Back-of-envelope estimation (QPS / storage / bandwidth)      | todo        | system-design-expert                     | URL shortener, Twitter   |
| Latency numbers every engineer should know                   | todo        | system-design-expert                     | -                        |
| Availability, reliability, scalability, maintainability      | todo        | system-design-expert, reliability-expert | -                        |
| SLA / SLO / SLI                                              | todo        | reliability-expert                       | -                        |
| Vertical vs horizontal scaling                               | todo        | system-design-expert                     | -                        |

## L1 - Networking & Entry Points

**Goal:** Know how traffic reaches and enters a system, and the protocols/edge components involved. · **Prereqs:** L0.

| Topic                             | Status | Owning expert(s)                   | Related practice problem           |
| --------------------------------- | ------ | ---------------------------------- | ---------------------------------- |
| IP addressing & routing basics    | todo   | network-expert                     | -                                  |
| DNS (resolution, records, GeoDNS) | todo   | network-expert                     | -                                  |
| TCP vs UDP                        | todo   | network-expert                     | -                                  |
| HTTP/1.1, HTTP/2, HTTP/3 (QUIC)   | todo   | network-expert                     | -                                  |
| HTTPS / TLS handshake             | todo   | network-expert, security-expert    | -                                  |
| WebSockets / SSE / long-polling   | todo   | network-expert                     | WhatsApp chat, notification system |
| REST vs gRPC vs GraphQL           | todo   | network-expert, backend-expert     | -                                  |
| Load balancers (L4 vs L7)         | todo   | network-expert, reliability-expert | -                                  |
| Reverse proxy                     | todo   | network-expert                     | -                                  |
| API gateway                       | todo   | network-expert, backend-expert     | -                                  |
| CDN                               | todo   | network-expert, cloud-expert       | YouTube/Netflix, Google Drive      |

## L2 - Core Building Blocks

**Goal:** Master the reusable components (cache, queue, storage, DB basics) that appear in nearly every design. · **Prereqs:** L0, L1.

| Topic                                                               | Status | Owning expert(s)                   | Related practice problem                 |
| ------------------------------------------------------------------- | ------ | ---------------------------------- | ---------------------------------------- |
| Caching layers & strategies (read/write-through, write-back, aside) | todo   | backend-expert, database-expert    | typeahead, newsfeed                      |
| Cache eviction (LRU/LFU/TTL); Redis vs Memcached                    | todo   | backend-expert, database-expert    | distributed cache                        |
| Cache invalidation & thundering herd                                | todo   | backend-expert, reliability-expert | distributed cache                        |
| Message queues & pub/sub (Kafka / RabbitMQ / SQS)                   | todo   | backend-expert                     | notification system, ad-click aggregator |
| Async / background processing                                       | todo   | backend-expert                     | notification system, job scheduler       |
| Object / blob storage                                               | todo   | cloud-expert, backend-expert       | YouTube/Netflix, Google Drive            |
| Relational vs NoSQL                                                 | todo   | database-expert                    | -                                        |
| ACID vs BASE                                                        | todo   | database-expert                    | payments/e-commerce                      |
| Indexing & B-trees                                                  | todo   | database-expert                    | -                                        |
| Query optimization                                                  | todo   | database-expert                    | -                                        |

## L3 - Data at Scale

**Goal:** Scale data across many machines and model it for large workloads. · **Prereqs:** L2.

| Topic                                                                                | Status | Owning expert(s)                | Related practice problem           |
| ------------------------------------------------------------------------------------ | ------ | ------------------------------- | ---------------------------------- |
| Replication (leader-follower, multi-leader, leaderless)                              | todo   | database-expert                 | -                                  |
| Partitioning & sharding                                                              | todo   | database-expert                 | Twitter, Instagram                 |
| Consistent hashing                                                                   | todo   | database-expert, backend-expert | distributed cache, design DynamoDB |
| NoSQL families (KV / document / wide-column / graph / time-series / search / NewSQL) | todo   | database-expert                 | design DynamoDB                    |
| Data modeling & denormalization                                                      | todo   | database-expert                 | newsfeed, Instagram                |
| Change data capture (CDC)                                                            | todo   | database-expert, backend-expert | -                                  |
| Event sourcing                                                                       | todo   | backend-expert, database-expert | payments/e-commerce                |
| CQRS                                                                                 | todo   | backend-expert                  | payments/e-commerce                |

## L4 - Distributed Systems Theory

**Goal:** Reason rigorously about consistency, consensus, and coordination. · **Prereqs:** L3.

| Topic                                                           | Status | Owning expert(s)                         | Related practice problem          |
| --------------------------------------------------------------- | ------ | ---------------------------------------- | --------------------------------- |
| CAP & PACELC                                                    | todo   | system-design-expert, database-expert    | -                                 |
| Consistency models (strong, eventual, causal, read-your-writes) | todo   | database-expert, system-design-expert    | -                                 |
| Consensus (Paxos / Raft / ZAB)                                  | todo   | system-design-expert, database-expert    | design Kafka                      |
| Leader election                                                 | todo   | system-design-expert, reliability-expert | design Kafka                      |
| Quorums (R + W > N)                                             | todo   | database-expert                          | design DynamoDB                   |
| Logical clocks (Lamport, vector clocks)                         | todo   | system-design-expert                     | -                                 |
| Gossip protocol                                                 | todo   | system-design-expert, reliability-expert | design DynamoDB                   |
| Distributed locking                                             | todo   | backend-expert, reliability-expert       | Ticketmaster, job scheduler       |
| Idempotency                                                     | todo   | backend-expert                           | payments/e-commerce               |
| Delivery semantics (at-most / at-least / exactly-once)          | todo   | backend-expert                           | design Kafka, ad-click aggregator |
| 2PC & Saga pattern                                              | todo   | backend-expert, database-expert          | payments/e-commerce               |

## L5 - Reliability & Operations (SRE)

**Goal:** Keep systems fast, resilient, and observable under failure and load. · **Prereqs:** L2 (L4 helpful).

| Topic                                              | Status | Owning expert(s)                   | Related practice problem |
| -------------------------------------------------- | ------ | ---------------------------------- | ------------------------ |
| Rate limiting (token/leaky bucket, sliding window) | todo   | reliability-expert, backend-expert | rate limiter             |
| Circuit breakers                                   | todo   | reliability-expert                 | -                        |
| Bulkheads                                          | todo   | reliability-expert                 | -                        |
| Retries w/ backoff + jitter                        | todo   | reliability-expert                 | -                        |
| Load shedding                                      | todo   | reliability-expert                 | -                        |
| Backpressure                                       | todo   | reliability-expert, backend-expert | -                        |
| Redundancy & failover                              | todo   | reliability-expert, cloud-expert   | -                        |
| Multi-region & disaster recovery                   | todo   | cloud-expert, reliability-expert   | -                        |
| Observability (metrics/logs/traces, RED/USE)       | todo   | reliability-expert                 | log/metrics analytics    |
| Capacity planning                                  | todo   | reliability-expert, cloud-expert   | -                        |
| Autoscaling                                        | todo   | cloud-expert, reliability-expert   | -                        |
| Deploys (blue-green / canary / rolling)            | todo   | cloud-expert, reliability-expert   | -                        |
| Feature flags                                      | todo   | reliability-expert                 | -                        |
| Chaos engineering                                  | todo   | reliability-expert                 | -                        |

## L6 - Security & Cross-Cutting

**Goal:** Secure the system and handle multi-tenancy, privacy, and abuse. · **Prereqs:** L1 (L5 helpful).

| Topic                           | Status | Owning expert(s)                    | Related practice problem |
| ------------------------------- | ------ | ----------------------------------- | ------------------------ |
| Authentication vs authorization | todo   | security-expert                     | -                        |
| OAuth2 / OIDC                   | todo   | security-expert                     | -                        |
| JWT vs sessions                 | todo   | security-expert, backend-expert     | -                        |
| TLS in depth                    | todo   | security-expert, network-expert     | -                        |
| Encryption at rest / in transit | todo   | security-expert                     | payments/e-commerce      |
| Key management                  | todo   | security-expert, cloud-expert       | -                        |
| DDoS mitigation                 | todo   | security-expert, network-expert     | -                        |
| WAF                             | todo   | security-expert                     | -                        |
| API abuse prevention            | todo   | security-expert, reliability-expert | rate limiter             |
| Multi-tenancy                   | todo   | backend-expert, security-expert     | -                        |
| Privacy & compliance (GDPR/PCI) | todo   | security-expert                     | payments/e-commerce      |

## L7 - Architecture Patterns

**Goal:** Choose and justify system-level structure. · **Prereqs:** L2, L4.

| Topic                                         | Status | Owning expert(s)                     | Related practice problem |
| --------------------------------------------- | ------ | ------------------------------------ | ------------------------ |
| Monolith vs microservices vs modular monolith | todo   | system-design-expert, backend-expert | -                        |
| Service discovery                             | todo   | backend-expert, cloud-expert         | -                        |
| Service mesh                                  | todo   | cloud-expert, reliability-expert     | -                        |
| Event-driven architecture                     | todo   | backend-expert                       | notification system      |
| Serverless / FaaS                             | todo   | cloud-expert                         | -                        |
| Lambda vs Kappa architecture                  | todo   | backend-expert, database-expert      | ad-click aggregator      |
| Data pipelines / ETL                          | todo   | database-expert, backend-expert      | log/metrics analytics    |
| Data lakes & warehouses                       | todo   | database-expert, cloud-expert        | log/metrics analytics    |

## L8 - Real-Time & Specialized

**Goal:** Handle real-time delivery and domain-specific subsystems. · **Prereqs:** L3, L5.

| Topic                                             | Status | Owning expert(s)                | Related practice problem                   |
| ------------------------------------------------- | ------ | ------------------------------- | ------------------------------------------ |
| Real-time delivery (WebSockets / SSE at scale)    | todo   | backend-expert, network-expert  | WhatsApp chat                              |
| Push notifications                                | todo   | backend-expert                  | notification system                        |
| Geospatial (geohash / quadtree)                   | todo   | backend-expert, database-expert | Uber location                              |
| Stream processing (Flink / Kafka Streams / Spark) | todo   | backend-expert, database-expert | ad-click aggregator, log/metrics analytics |
| Big data (MapReduce / Hadoop)                     | todo   | database-expert, backend-expert | web crawler                                |
| Search & typeahead                                | todo   | database-expert, backend-expert | typeahead                                  |
| Recommendation systems                            | todo   | backend-expert, database-expert | newsfeed, YouTube/Netflix                  |
| ML system design basics                           | todo   | backend-expert                  | recommendation, ad-click aggregator        |

## L9 - Interview Craft & Senior Signal

**Goal:** Run a design interview end-to-end and show senior-level judgment. · **Prereqs:** L0–L4 minimum (more depth = better).

| Topic                                                                                        | Status | Owning expert(s)     | Related practice problem |
| -------------------------------------------------------------------------------------------- | ------ | -------------------- | ------------------------ |
| The design framework (requirements→estimation→API→data model→HLD→deep-dive→bottlenecks→wrap) | todo   | system-design-expert | all L10                  |
| Driving ambiguity & scoping                                                                  | todo   | system-design-expert | all L10                  |
| Trade-off articulation                                                                       | todo   | system-design-expert | all L10                  |
| Whiteboarding & communication                                                                | todo   | system-design-expert | all L10                  |
| Company flavors (MAANGO variations)                                                          | todo   | system-design-expert | all L10                  |

## L10 - Practice Problems

**Goal:** Apply everything under interview conditions; each attempt goes in `practice/` and is graded by `design-reviewer`. · **Prereqs:** framework (L9) + the levels each problem exercises (see notes).

| Problem                     | Status | Owning expert(s)                                       | Key levels exercised |
| --------------------------- | ------ | ------------------------------------------------------ | -------------------- |
| URL shortener               | todo   | system-design-expert                                   | L0–L2 (start here)   |
| Pastebin                    | todo   | system-design-expert                                   | L0–L2                |
| Rate limiter                | todo   | reliability-expert, system-design-expert               | L2, L5               |
| Twitter / newsfeed          | todo   | system-design-expert, database-expert                  | L2–L3, L8            |
| Instagram                   | todo   | system-design-expert, database-expert                  | L2–L3, L8            |
| WhatsApp chat               | todo   | system-design-expert, backend-expert                   | L1, L4, L8           |
| YouTube / Netflix streaming | todo   | system-design-expert, cloud-expert                     | L1–L2, L8            |
| Uber location               | todo   | system-design-expert, backend-expert                   | L3, L8               |
| Google Drive / Dropbox sync | todo   | system-design-expert, backend-expert                   | L2–L4                |
| Web crawler                 | todo   | system-design-expert, backend-expert                   | L2, L8               |
| Typeahead / autocomplete    | todo   | system-design-expert, database-expert                  | L2, L8               |
| Notification system         | todo   | system-design-expert, backend-expert                   | L2, L7–L8            |
| Ticketmaster                | todo   | system-design-expert, database-expert                  | L2, L4               |
| Payments / e-commerce       | todo   | system-design-expert, database-expert, security-expert | L2–L4, L6            |
| Distributed cache           | todo   | backend-expert, database-expert                        | L2–L3                |
| Job scheduler               | todo   | system-design-expert, backend-expert                   | L2, L4               |
| Log / metrics analytics     | todo   | reliability-expert, database-expert                    | L5, L7–L8            |
| Ad-click aggregator         | todo   | database-expert, backend-expert                        | L4, L7–L8            |
| Leaderboard                 | todo   | backend-expert, database-expert                        | L2–L3                |
| Design Kafka                | todo   | backend-expert, database-expert                        | L3–L4                |
| Design DynamoDB             | todo   | database-expert                                        | L3–L4                |

---

# Frontend System-Design Track

Parallel/adjacent track owned mainly by `frontend-expert`. Can start once backend **L0–L1** are `learned` (needs request lifecycle + HTTP/CDN context).

## FE-L0 - Foundations

**Goal:** Frontend design vocabulary and how the browser turns bytes into pixels. · **Prereqs:** backend L0.

| Topic                                       | Status | Owning expert(s)                      | Related practice problem     |
| ------------------------------------------- | ------ | ------------------------------------- | ---------------------------- |
| FE system-design framework                  | todo   | frontend-expert, system-design-expert | all FE-L6                    |
| Browser rendering / critical rendering path | todo   | frontend-expert                       | video player, live dashboard |

## FE-L1 - Rendering & Delivery

**Goal:** Choose a rendering strategy and ship assets efficiently. · **Prereqs:** FE-L0, backend L1 (CDN).

| Topic                                    | Status | Owning expert(s)                | Related practice problem |
| ---------------------------------------- | ------ | ------------------------------- | ------------------------ |
| CSR / SSR / SSG / ISR / streaming SSR    | todo   | frontend-expert                 | e-commerce PDP           |
| Hydration & islands                      | todo   | frontend-expert                 | e-commerce PDP           |
| Bundling / tree-shaking / code-splitting | todo   | frontend-expert                 | design system            |
| Asset caching & CDN                      | todo   | frontend-expert, network-expert | e-commerce PDP           |
| Image / font optimization                | todo   | frontend-expert                 | e-commerce PDP           |

## FE-L2 - Data & State

**Goal:** Manage client/server data, caching, and large lists. · **Prereqs:** FE-L1.

| Topic                                            | Status | Owning expert(s)                | Related practice problem   |
| ------------------------------------------------ | ------ | ------------------------------- | -------------------------- |
| Client cache vs server cache (React Query / SWR) | todo   | frontend-expert                 | infinite feed              |
| Normalization                                    | todo   | frontend-expert                 | chat UI                    |
| Pagination & infinite scroll                     | todo   | frontend-expert                 | infinite feed              |
| List virtualization                              | todo   | frontend-expert                 | infinite feed, spreadsheet |
| Optimistic updates                               | todo   | frontend-expert                 | chat UI                    |
| Prefetching                                      | todo   | frontend-expert                 | infinite feed              |
| GraphQL vs REST from the client                  | todo   | frontend-expert, network-expert | e-commerce PDP             |

## FE-L3 - Performance

**Goal:** Measure and hit performance targets. · **Prereqs:** FE-L1 (FE-L2 helpful).

| Topic                             | Status | Owning expert(s)                    | Related practice problem  |
| --------------------------------- | ------ | ----------------------------------- | ------------------------- |
| Core Web Vitals (LCP / INP / CLS) | todo   | frontend-expert                     | e-commerce PDP            |
| Performance budgets               | todo   | frontend-expert                     | design system             |
| Debounce / throttle               | todo   | frontend-expert                     | typeahead widget          |
| Memoization                       | todo   | frontend-expert                     | spreadsheet               |
| Web workers                       | todo   | frontend-expert                     | spreadsheet, video player |
| RUM & error tracking              | todo   | frontend-expert, reliability-expert | live dashboard            |

## FE-L4 - Real-Time & Offline

**Goal:** Build live, offline-capable, collaborative UIs. · **Prereqs:** FE-L2, backend L8 (real-time).

| Topic                                    | Status | Owning expert(s)                | Related practice problem |
| ---------------------------------------- | ------ | ------------------------------- | ------------------------ |
| WebSockets / SSE / polling (client-side) | todo   | frontend-expert, network-expert | chat UI, live dashboard  |
| PWA & service workers                    | todo   | frontend-expert                 | -                        |
| Offline-first                            | todo   | frontend-expert                 | -                        |
| IndexedDB                                | todo   | frontend-expert                 | Google Docs editor       |
| OT / CRDT basics                         | todo   | frontend-expert                 | Google Docs editor       |

## FE-L5 - Quality & Scale

**Goal:** Ship accessible, international, secure UIs at org scale. · **Prereqs:** FE-L1 (FE-L3 helpful).

| Topic                                     | Status | Owning expert(s)                 | Related practice problem |
| ----------------------------------------- | ------ | -------------------------------- | ------------------------ |
| Accessibility (WCAG)                      | todo   | frontend-expert                  | design system            |
| Internationalization (i18n)               | todo   | frontend-expert                  | design system            |
| Design systems                            | todo   | frontend-expert                  | design system            |
| Micro-frontends / module federation       | todo   | frontend-expert                  | -                        |
| Client security (XSS / CSRF / CSP / CORS) | todo   | frontend-expert, security-expert | e-commerce PDP           |

## FE-L6 - Problems

**Goal:** Apply the FE track under interview conditions; attempts go in `practice/`, graded by `design-reviewer`. · **Prereqs:** FE-L0 + relevant FE levels.

| Problem            | Status | Owning expert(s) | Key levels exercised  |
| ------------------ | ------ | ---------------- | --------------------- |
| Typeahead widget   | todo   | frontend-expert  | FE-L2–L3 (start here) |
| Infinite feed      | todo   | frontend-expert  | FE-L2–L3              |
| Carousel           | todo   | frontend-expert  | FE-L1, FE-L3          |
| Google Docs editor | todo   | frontend-expert  | FE-L4                 |
| Chat UI            | todo   | frontend-expert  | FE-L2, FE-L4          |
| Video player       | todo   | frontend-expert  | FE-L1, FE-L3          |
| E-commerce PDP     | todo   | frontend-expert  | FE-L1–L3, FE-L5       |
| Design system      | todo   | frontend-expert  | FE-L1, FE-L5          |
| Live dashboard     | todo   | frontend-expert  | FE-L3–L4              |
| Spreadsheet        | todo   | frontend-expert  | FE-L2–L3              |

---

# Side Tracks (optional, not sequenced here)

| Track              | Owner                | Notes                                                               |
| ------------------ | -------------------- | ------------------------------------------------------------------- |
| DSA (coding round) | dsa-expert           | Separate coding-interview prep. Run in parallel on its own cadence. |
| IT-trends R&D      | it-trends-researcher | Web research on emerging tech; feeds `research/` opportunistically. |
