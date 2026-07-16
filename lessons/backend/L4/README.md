# L4. NoSQL and Data at Scale

*Model, replicate, and partition data across many machines for large workloads.*

**Just getting started** -- 4 of 15 topics written. Work through them in order; each is a short, self-contained read (~5-8 min).

| # | Lesson | In one line | Status |
|---|--------|-------------|--------|
| 01 | [NoSQL families (KV, document, wide-column, graph, time-series, search, NewSQL, vector)](01-nosql-families.md) | Eight data models built for eight different access patterns, and the one question that decides which fits. | ✅ |
| 02 | [Replication (leader-follower, multi-leader, leaderless)](02-replication.md) | Keeping copies of the same data on multiple machines in sync — one writer, many writers, or no fixed writer at all. | ✅ |
| 03 | [Partitioning and sharding](03-partitioning-and-sharding.md) | Splitting data across many machines so no single node has to hold or serve all of it. | ✅ |
| 04 | [Rebalancing and hotspots](04-rebalancing-and-hotspots.md) | What happens when partitions grow unevenly, and how a cluster redistributes data without falling over. | ✅ |
| 05 | Consistent hashing (virtual nodes) | A hashing scheme that lets nodes join and leave a cluster while moving only a small slice of the keyspace. | ⚪ |
| 06 | Data modeling and denormalization | Designing the schema around the query instead of the query around the schema — NoSQL's core discipline. | ⚪ |
| 07 | Quorums (R + W > N) | Tuning how many replicas must agree on a read or write to trade consistency for availability. | ⚪ |
| 08 | Change data capture (CDC) + outbox pattern | Streaming a database's changes to everything else that needs to know about them, reliably. | ⚪ |
| 09 | Event sourcing | Storing every change as an immutable event instead of overwriting state, so the log itself is the source of truth. | ⚪ |
| 10 | CQRS | Splitting the read model from the write model so each can be optimized for what it actually does. | ⚪ |
| 11 | Vector databases / ANN search (HNSW) *(emerging)* | Finding the nearest neighbors of a high-dimensional embedding fast, without scanning everything. | ⚪ |
| 12 | Real-time OLAP (Pinot, Druid, ClickHouse) *(emerging)* | Running analytical queries over fresh, still-arriving data instead of yesterday's batch. | ⚪ |
| 13 | HTAP (hybrid transactional/analytical) *(emerging)* | One database trying to serve both transactions and analytics without an ETL pipeline in between. | ⚪ |
| 14 | Database branching / serverless DBs (Neon, PlanetScale) *(emerging)* | Databases that scale to zero and let you branch a copy the way you'd branch a git repo. | ⚪ |
| 15 | Data contracts (schema-registry-enforced) *(emerging)* | Making a data producer's schema a versioned, enforced contract instead of an informal agreement. | ⚪ |

**Deeper reference:** [research/backend/L4](../../../research/backend/L4/02-replication.md)
