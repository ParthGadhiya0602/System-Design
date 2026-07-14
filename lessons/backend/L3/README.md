# L3. Caching and Data Access

*Speeding up reads and softening the load on your database — the right cache in the right place, keeping it in sync, and surviving the failure modes caching creates.*

**In progress** -- 2 of 8 topics written. Work through them in order; each is a short, self-contained read (~5-8 min).

| # | Lesson | In one line | Status |
|---|--------|-------------|--------|
| 01 | [Caching layers and strategies](01-caching-layers-strategies.md) | Where a cache can live (client, CDN, app, distributed, DB) and who keeps it in sync -- cache-aside, read-through, write-through, write-back. | ✅ |
| 02 | [Eviction policies (LRU, LFU, TTL)](02-eviction-policies.md) | When the cache is full, what gets thrown out first -- recency, frequency, or an explicit deadline, and why TTL composes with the other two instead of replacing them. | ✅ |
| 03 | Redis vs Memcached | Two popular distributed caches, one much simpler than the other -- picking between raw speed and richer data structures. | ⚪ |
| 04 | Cache stampede / dogpile / thundering herd | What happens when a hot key expires and a thousand requests all race to rebuild it at once. | ⚪ |
| 05 | Cache coherence and invalidation | Keeping a cached copy honest as the source of truth changes underneath it. | ⚪ |
| 06 | Negative caching | Caching the *absence* of data, so repeated "not found" lookups stop hammering the backing store. | ⚪ |
| 07 | CDN caching | Pushing content to the edge, near the user, so most requests never reach your origin at all. | ⚪ |
| 08 | Object / blob storage | Where large, immutable files live -- and why it's not the same problem as caching rows of a table. | ⚪ |

**Deeper reference:** [research/backend/L3](../../../research/backend/L3/01-caching-layers-strategies.md)
