# L2. Storage and Relational Databases

*The relational model and how a single database engine actually stores, indexes, and transacts data -- the ground truth beneath every caching layer, replication scheme, and NoSQL trade-off that comes later.*

**In progress** -- 12 of 13 topics written so far. Work through them in order; each is a short, self-contained read (~5-8 min).

| # | Lesson | In one line | Status |
|---|--------|-------------|--------|
| 01 | [Relational model](01-relational-model.md) | Relations, tuples, keys, algebra, and why separating "what" from "how" is what made SQL win. | ✅ |
| 02 | [Normalization forms](02-normalization-forms.md) | Eliminating redundancy and update anomalies, one normal form at a time. | ✅ |
| 03 | [SQL depth (joins, aggregation, subqueries, window functions)](03-sql-depth.md) | Beyond the basics -- combining rows, collapsing them, nesting queries, and per-row aggregates without collapsing anything. | ✅ |
| 04 | [ACID](04-acid.md) | The four guarantees a transaction makes -- and what breaks without each one. | ✅ |
| 05 | [Transactions and isolation levels](05-transactions-isolation-levels.md) | The menu of how strictly "concurrent transactions look like they ran one at a time" is actually enforced -- and the named bugs each weaker setting lets through. | ✅ |
| 06 | [MVCC](06-mvcc.md) | How readers and writers avoid blocking each other, by keeping multiple versions of the same row instead of overwriting it. | ✅ |
| 07 | [Locking (row/table, optimistic/pessimistic)](07-locking.md) | The other way to control concurrent access -- and when it beats MVCC. | ✅ |
| 08 | [Indexing (B-tree, hash, LSM-tree)](08-indexing.md) | The structures that turn a full table scan into a handful of lookups. | ✅ |
| 09 | [Write-ahead log (WAL)](09-write-ahead-log.md) | How a database survives a crash without losing a committed write. | ✅ |
| 10 | [Storage engines](10-storage-engines.md) | In-place B-tree pages vs. append-and-compact LSM-trees -- the physical write path hiding behind every SQL query. | ✅ |
| 11 | [Query planning and optimization](11-query-planning-optimization.md) | How declarative SQL becomes an actual, efficient execution plan -- and why the same query can flip between an index scan and a full scan overnight. | ✅ |
| 12 | [Connection pooling](12-connection-pooling.md) | Why opening a fresh connection per request doesn't scale, and what fixes it -- from client-side pools to PgBouncer to serverless. | ✅ |
| 13 | OLTP vs OLAP | Two very different workloads, two very different ways of laying out data. | ⚪ |

**Deeper reference:** [research/backend/L2](../../../research/backend/L2/01-relational-model.md)

**Next level →** L3. Caching and Data Access
