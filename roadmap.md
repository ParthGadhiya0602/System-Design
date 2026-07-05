# System Design Learning Roadmap

The master learning path for this repo. It sequences everything a full-stack engineer needs to go from **true beginner → expert** for MAANGO system-design interviews. It is read at the start of most sessions, so it is kept dense and scannable.

## Two parallel tracks

- **Backend / Distributed-Systems track (L0–L10)** - the core program. This is the main spine.
- **Frontend System-Design track (FE-L0–FE-L6)** - a parallel, adjacent track. Can be interleaved once backend L0–L1 are done.
- Side tracks (optional, not sequenced here): **DSA** (coding round) and **IT-trends R&D** (web research).

## How status works

| Status        | Meaning                                                           |
| ------------- | ----------------------------------------------------------------- |
| `todo`        | Not started. No research/lesson yet.                              |
| `in-progress` | Research or lesson being written / actively studied.              |
| `learned`     | Lesson exists and learner has studied it.                         |
| `practiced`   | Applied in a `practice/` attempt and graded.                      |

Order of maturity: `todo` → `in-progress` → `learned` → `practiced`. A topic is only truly "done" when `practiced`.

## Recommended starting point

Start at **L0 Foundations** (backend). It requires no prerequisites and every later topic depends on it. Do not touch FE track or any L1+ topic until L0 is `learned`.

## What's next (current pointer)

> **L0 research is written** (`research/l0-foundations.md`, all 9 topics). Learner is actively studying L0 Topic 1 (what system design is) live. **No `lessons/` files exist yet** - the gap is turning research into learner-facing lessons.
>
> Continue L0 in prerequisite order. Research already exists (`research/l0-foundations.md` §2, §3, §4), so the immediate move is to work through:
>
> 1. _Functional vs non-functional requirements_ (research §2).
> 2. _Client–server model_ (research §3).
> 3. _Request lifecycle end-to-end_ (research §4).
>
> After these, finish L0 (§5 estimation, §6 latency, §7 -ilities, §8 SLA/SLO/SLI, §9 scaling), then mark L0 `learned`. No practice problem yet - first attempt is **L10: URL shortener** after L0–L2.

---

# Backend / Distributed-Systems Track

## L0 - Foundations

**Goal:** Understand what system design is and the vocabulary/estimation skills every design uses. · **Prereqs:** none. · **Research:** `research/l0-foundations.md` (complete, all 9 topics).

| Topic                                                        | Status      | Related practice problem |
| ------------------------------------------------------------ | ----------- | ------------------------ |
| What system design is; the design mindset                    | in-progress | -                        |
| Functional vs non-functional requirements                    | todo        | all L10                  |
| Client–server model                                          | todo        | -                        |
| Request lifecycle end-to-end (browser→DNS→LB→server→DB→back) | todo        | -                        |
| Back-of-envelope estimation (QPS / storage / bandwidth)      | todo        | URL shortener, Twitter   |
| Latency numbers every engineer should know                   | todo        | -                        |
| Availability, reliability, scalability, maintainability      | todo        | -                        |
| SLA / SLO / SLI                                              | todo        | -                        |
| Vertical vs horizontal scaling                               | todo        | -                        |

## L1 - Networking & Entry Points

**Goal:** Know how traffic reaches and enters a system, and the protocols/edge components involved. · **Prereqs:** L0.

| Topic                             | Status | Related practice problem           |
| --------------------------------- | ------ | ---------------------------------- |
| IP addressing & routing basics    | todo   | -                                  |
| DNS (resolution, records, GeoDNS) | todo   | -                                  |
| TCP vs UDP                        | todo   | -                                  |
| HTTP/1.1, HTTP/2, HTTP/3 (QUIC)   | todo   | -                                  |
| HTTPS / TLS handshake             | todo   | -                                  |
| WebSockets / SSE / long-polling   | todo   | WhatsApp chat, notification system |
| REST vs gRPC vs GraphQL           | todo   | -                                  |
| Load balancers (L4 vs L7)         | todo   | -                                  |
| Reverse proxy                     | todo   | -                                  |
| API gateway                       | todo   | -                                  |
| CDN                               | todo   | YouTube/Netflix, Google Drive      |

## L2 - Core Building Blocks

**Goal:** Master the reusable components (cache, queue, storage, DB basics) that appear in nearly every design. · **Prereqs:** L0, L1.

| Topic                                                               | Status | Related practice problem                 |
| ------------------------------------------------------------------- | ------ | ---------------------------------------- |
| Caching layers & strategies (read/write-through, write-back, aside) | todo   | typeahead, newsfeed                      |
| Cache eviction (LRU/LFU/TTL); Redis vs Memcached                    | todo   | distributed cache                        |
| Cache invalidation & thundering herd                                | todo   | distributed cache                        |
| Message queues & pub/sub (Kafka / RabbitMQ / SQS)                   | todo   | notification system, ad-click aggregator |
| Async / background processing                                       | todo   | notification system, job scheduler       |
| Object / blob storage                                               | todo   | YouTube/Netflix, Google Drive            |
| Relational vs NoSQL                                                 | todo   | -                                        |
| ACID vs BASE                                                        | todo   | payments/e-commerce                      |
| Indexing & B-trees                                                  | todo   | -                                        |
| Query optimization                                                  | todo   | -                                        |

## L3 - Data at Scale

**Goal:** Scale data across many machines and model it for large workloads. · **Prereqs:** L2.

| Topic                                                                                | Status | Related practice problem           |
| ------------------------------------------------------------------------------------ | ------ | ---------------------------------- |
| Replication (leader-follower, multi-leader, leaderless)                              | todo   | -                                  |
| Partitioning & sharding                                                              | todo   | Twitter, Instagram                 |
| Consistent hashing                                                                   | todo   | distributed cache, design DynamoDB |
| NoSQL families (KV / document / wide-column / graph / time-series / search / NewSQL) | todo   | design DynamoDB                    |
| Data modeling & denormalization                                                      | todo   | newsfeed, Instagram                |
| Change data capture (CDC)                                                            | todo   | -                                  |
| Event sourcing                                                                       | todo   | payments/e-commerce                |
| CQRS                                                                                 | todo   | payments/e-commerce                |

## L4 - Distributed Systems Theory

**Goal:** Reason rigorously about consistency, consensus, and coordination. · **Prereqs:** L3.

| Topic                                                           | Status | Related practice problem          |
| --------------------------------------------------------------- | ------ | --------------------------------- |
| CAP & PACELC                                                    | todo   | -                                 |
| Consistency models (strong, eventual, causal, read-your-writes) | todo   | -                                 |
| Consensus (Paxos / Raft / ZAB)                                  | todo   | design Kafka                      |
| Leader election                                                 | todo   | design Kafka                      |
| Quorums (R + W > N)                                             | todo   | design DynamoDB                   |
| Logical clocks (Lamport, vector clocks)                         | todo   | -                                 |
| Gossip protocol                                                 | todo   | design DynamoDB                   |
| Distributed locking                                             | todo   | Ticketmaster, job scheduler       |
| Idempotency                                                     | todo   | payments/e-commerce               |
| Delivery semantics (at-most / at-least / exactly-once)          | todo   | design Kafka, ad-click aggregator |
| 2PC & Saga pattern                                              | todo   | payments/e-commerce               |

## L5 - Reliability & Operations (SRE)

**Goal:** Keep systems fast, resilient, and observable under failure and load. · **Prereqs:** L2 (L4 helpful).

| Topic                                              | Status | Related practice problem |
| -------------------------------------------------- | ------ | ------------------------ |
| Rate limiting (token/leaky bucket, sliding window) | todo   | rate limiter             |
| Circuit breakers                                   | todo   | -                        |
| Bulkheads                                          | todo   | -                        |
| Retries w/ backoff + jitter                        | todo   | -                        |
| Load shedding                                      | todo   | -                        |
| Backpressure                                       | todo   | -                        |
| Redundancy & failover                              | todo   | -                        |
| Multi-region & disaster recovery                   | todo   | -                        |
| Observability (metrics/logs/traces, RED/USE)       | todo   | log/metrics analytics    |
| Capacity planning                                  | todo   | -                        |
| Autoscaling                                        | todo   | -                        |
| Deploys (blue-green / canary / rolling)            | todo   | -                        |
| Feature flags                                      | todo   | -                        |
| Chaos engineering                                  | todo   | -                        |

## L6 - Security & Cross-Cutting

**Goal:** Secure the system and handle multi-tenancy, privacy, and abuse. · **Prereqs:** L1 (L5 helpful).

| Topic                           | Status | Related practice problem |
| ------------------------------- | ------ | ------------------------ |
| Authentication vs authorization | todo   | -                        |
| OAuth2 / OIDC                   | todo   | -                        |
| JWT vs sessions                 | todo   | -                        |
| TLS in depth                    | todo   | -                        |
| Encryption at rest / in transit | todo   | payments/e-commerce      |
| Key management                  | todo   | -                        |
| DDoS mitigation                 | todo   | -                        |
| WAF                             | todo   | -                        |
| API abuse prevention            | todo   | rate limiter             |
| Multi-tenancy                   | todo   | -                        |
| Privacy & compliance (GDPR/PCI) | todo   | payments/e-commerce      |

## L7 - Architecture Patterns

**Goal:** Choose and justify system-level structure. · **Prereqs:** L2, L4.

| Topic                                         | Status | Related practice problem |
| --------------------------------------------- | ------ | ------------------------ |
| Monolith vs microservices vs modular monolith | todo   | -                        |
| Service discovery                             | todo   | -                        |
| Service mesh                                  | todo   | -                        |
| Event-driven architecture                     | todo   | notification system      |
| Serverless / FaaS                             | todo   | -                        |
| Lambda vs Kappa architecture                  | todo   | ad-click aggregator      |
| Data pipelines / ETL                          | todo   | log/metrics analytics    |
| Data lakes & warehouses                       | todo   | log/metrics analytics    |

## L8 - Real-Time & Specialized

**Goal:** Handle real-time delivery and domain-specific subsystems. · **Prereqs:** L3, L5.

| Topic                                             | Status | Related practice problem                   |
| ------------------------------------------------- | ------ | ------------------------------------------ |
| Real-time delivery (WebSockets / SSE at scale)    | todo   | WhatsApp chat                              |
| Push notifications                                | todo   | notification system                        |
| Geospatial (geohash / quadtree)                   | todo   | Uber location                              |
| Stream processing (Flink / Kafka Streams / Spark) | todo   | ad-click aggregator, log/metrics analytics |
| Big data (MapReduce / Hadoop)                     | todo   | web crawler                                |
| Search & typeahead                                | todo   | typeahead                                  |
| Recommendation systems                            | todo   | newsfeed, YouTube/Netflix                  |
| ML system design basics                           | todo   | recommendation, ad-click aggregator        |

## L9 - Interview Craft & Senior Signal

**Goal:** Run a design interview end-to-end and show senior-level judgment. · **Prereqs:** L0–L4 minimum (more depth = better).

| Topic                                                                                        | Status | Related practice problem |
| -------------------------------------------------------------------------------------------- | ------ | ------------------------ |
| The design framework (requirements→estimation→API→data model→HLD→deep-dive→bottlenecks→wrap) | todo   | all L10                  |
| Driving ambiguity & scoping                                                                  | todo   | all L10                  |
| Trade-off articulation                                                                       | todo   | all L10                  |
| Whiteboarding & communication                                                                | todo   | all L10                  |
| Company flavors (MAANGO variations)                                                          | todo   | all L10                  |

## L10 - Practice Problems

**Goal:** Apply everything under interview conditions; each attempt goes in `practice/` and is graded. · **Prereqs:** framework (L9) + the levels each problem exercises (see notes).

| Problem                     | Status | Key levels exercised |
| --------------------------- | ------ | -------------------- |
| URL shortener               | todo   | L0–L2 (start here)   |
| Pastebin                    | todo   | L0–L2                |
| Rate limiter                | todo   | L2, L5               |
| Twitter / newsfeed          | todo   | L2–L3, L8            |
| Instagram                   | todo   | L2–L3, L8            |
| WhatsApp chat               | todo   | L1, L4, L8           |
| YouTube / Netflix streaming | todo   | L1–L2, L8            |
| Uber location               | todo   | L3, L8               |
| Google Drive / Dropbox sync | todo   | L2–L4                |
| Web crawler                 | todo   | L2, L8               |
| Typeahead / autocomplete    | todo   | L2, L8               |
| Notification system         | todo   | L2, L7–L8            |
| Ticketmaster                | todo   | L2, L4               |
| Payments / e-commerce       | todo   | L2–L4, L6            |
| Distributed cache           | todo   | L2–L3                |
| Job scheduler               | todo   | L2, L4               |
| Log / metrics analytics     | todo   | L5, L7–L8            |
| Ad-click aggregator         | todo   | L4, L7–L8            |
| Leaderboard                 | todo   | L2–L3                |
| Design Kafka                | todo   | L3–L4                |
| Design DynamoDB             | todo   | L3–L4                |

---

# Frontend System-Design Track

Parallel/adjacent track. Can start once backend **L0–L1** are `learned` (needs request lifecycle + HTTP/CDN context).

## FE-L0 - Foundations

**Goal:** Frontend design vocabulary and how the browser turns bytes into pixels. · **Prereqs:** backend L0.

| Topic                                       | Status | Related practice problem     |
| ------------------------------------------- | ------ | ---------------------------- |
| FE system-design framework                  | todo   | all FE-L6                    |
| Browser rendering / critical rendering path | todo   | video player, live dashboard |

## FE-L1 - Rendering & Delivery

**Goal:** Choose a rendering strategy and ship assets efficiently. · **Prereqs:** FE-L0, backend L1 (CDN).

| Topic                                    | Status | Related practice problem |
| ---------------------------------------- | ------ | ------------------------ |
| CSR / SSR / SSG / ISR / streaming SSR    | todo   | e-commerce PDP           |
| Hydration & islands                      | todo   | e-commerce PDP           |
| Bundling / tree-shaking / code-splitting | todo   | design system            |
| Asset caching & CDN                      | todo   | e-commerce PDP           |
| Image / font optimization                | todo   | e-commerce PDP           |

## FE-L2 - Data & State

**Goal:** Manage client/server data, caching, and large lists. · **Prereqs:** FE-L1.

| Topic                                            | Status | Related practice problem   |
| ------------------------------------------------ | ------ | -------------------------- |
| Client cache vs server cache (React Query / SWR) | todo   | infinite feed              |
| Normalization                                    | todo   | chat UI                    |
| Pagination & infinite scroll                     | todo   | infinite feed              |
| List virtualization                              | todo   | infinite feed, spreadsheet |
| Optimistic updates                               | todo   | chat UI                    |
| Prefetching                                      | todo   | infinite feed              |
| GraphQL vs REST from the client                  | todo   | e-commerce PDP             |

## FE-L3 - Performance

**Goal:** Measure and hit performance targets. · **Prereqs:** FE-L1 (FE-L2 helpful).

| Topic                             | Status | Related practice problem  |
| --------------------------------- | ------ | ------------------------- |
| Core Web Vitals (LCP / INP / CLS) | todo   | e-commerce PDP            |
| Performance budgets               | todo   | design system             |
| Debounce / throttle               | todo   | typeahead widget          |
| Memoization                       | todo   | spreadsheet               |
| Web workers                       | todo   | spreadsheet, video player |
| RUM & error tracking              | todo   | live dashboard            |

## FE-L4 - Real-Time & Offline

**Goal:** Build live, offline-capable, collaborative UIs. · **Prereqs:** FE-L2, backend L8 (real-time).

| Topic                                    | Status | Related practice problem |
| ---------------------------------------- | ------ | ------------------------ |
| WebSockets / SSE / polling (client-side) | todo   | chat UI, live dashboard  |
| PWA & service workers                    | todo   | -                        |
| Offline-first                            | todo   | -                        |
| IndexedDB                                | todo   | Google Docs editor       |
| OT / CRDT basics                         | todo   | Google Docs editor       |

## FE-L5 - Quality & Scale

**Goal:** Ship accessible, international, secure UIs at org scale. · **Prereqs:** FE-L1 (FE-L3 helpful).

| Topic                                     | Status | Related practice problem |
| ----------------------------------------- | ------ | ------------------------ |
| Accessibility (WCAG)                      | todo   | design system            |
| Internationalization (i18n)               | todo   | design system            |
| Design systems                            | todo   | design system            |
| Micro-frontends / module federation       | todo   | -                        |
| Client security (XSS / CSRF / CSP / CORS) | todo   | e-commerce PDP           |

## FE-L6 - Problems

**Goal:** Apply the FE track under interview conditions; attempts go in `practice/`, graded. · **Prereqs:** FE-L0 + relevant FE levels.

| Problem            | Status | Key levels exercised  |
| ------------------ | ------ | --------------------- |
| Typeahead widget   | todo   | FE-L2–L3 (start here) |
| Infinite feed      | todo   | FE-L2–L3              |
| Carousel           | todo   | FE-L1, FE-L3          |
| Google Docs editor | todo   | FE-L4                 |
| Chat UI            | todo   | FE-L2, FE-L4          |
| Video player       | todo   | FE-L1, FE-L3          |
| E-commerce PDP     | todo   | FE-L1–L3, FE-L5       |
| Design system      | todo   | FE-L1, FE-L5          |
| Live dashboard     | todo   | FE-L3–L4              |
| Spreadsheet        | todo   | FE-L2–L3              |

---

# Side Tracks (optional, not sequenced here)

| Track              | Notes                                                               |
| ------------------ | ------------------------------------------------------------------- |
| DSA (coding round) | Separate coding-interview prep. Run in parallel on its own cadence. |
| IT-trends R&D      | Web research on emerging tech; feeds `research/` opportunistically. |
