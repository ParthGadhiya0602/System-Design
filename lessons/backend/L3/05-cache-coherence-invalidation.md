# Cache Coherence and Invalidation

*A price changes in the database at 3:00:00.000pm. Every cache that had the old price cached doesn't know that happened — until something tells it, or its clock runs out.*

`⏱️ ~8 min · 5 of 8 · L3`

> [!TIP] The gist
> A cache is a **copy**, not the truth — so the instant the database changes, every cached copy is potentially wrong until something corrects it. **Cache coherence** is how close "corrected" stays to "instantly," and **invalidation** is the mechanism that does the correcting: delete the stale entry (safer) or update it in place (riskier under concurrent writers). Even the safe choice — delete-on-write — has a classic race where a slow concurrent reader can repopulate the *just-deleted* stale value right back into the cache. Nothing here reaches strong consistency for free; every mechanism only narrows the window.

## Intuition

You write a new phone number on a sticky note and put it on the fridge (the database). Five people in the house each already copied the old number into their own phone's contacts (the caches). Updating the sticky note doesn't update anyone's phone — someone has to *tell* each person, or they'll keep dialing the old number until they happen to check the fridge again.

That's the whole problem: **updating the source of truth and correcting every copy are two separate actions**, and nothing forces them to happen together.

## The concept

**Cache coherence** is the property that every copy of a piece of data — the database's copy, and every cache's copy — reflects the same value, or converges to the same value within a bounded, known amount of time. **Invalidation** is the mechanism that closes the gap: telling a cache "the thing you're holding is no longer current," either by deleting it or by pushing the new value in.

The gap has a name and precise edges — the **staleness window**:

- **Starts** the instant the database write commits.
- **Ends** the instant every cached copy has been corrected (deleted/updated) or has naturally expired via TTL.
- **Width** depends entirely on the mechanism — from single-digit milliseconds (a synchronous delete on the write path) up to the *full TTL* (if nothing actively invalidates at all).

Worth separating this cleanly from two lookalikes:

- It is **not** the ordinary cache-aside miss from [topic 01](01-caching-layers-strategies.md) — that's an *absent* value being fetched once, expected behavior. Coherence is about a value that's **present and wrong**, being served as if it were correct.
- It is **not** a [cache stampede](04-cache-stampede.md) — a stampede is a *load* problem (a value vanishes, N requests race to refill it). Invalidation is a *correctness* problem (a value is still sitting there, still being served, and it disagrees with the database). Reaching for stampede fixes like locking or coalescing does nothing about a *hit* that returns the wrong answer.

The reason this doesn't have a free, total fix: **the database write and the cache correction are two separate operations against two separate systems**, with no single transaction spanning both. Everything below is a strategy for closing that gap as tightly as the data's tolerance for staleness demands — never a way to make it literally zero without real cost.

## How it works

**1. Delete-on-write beats update-on-write**

Once you decide to actively invalidate on write (rather than just wait out the TTL), there are exactly two things to do to the stale entry:

- **Update it** — push the new value straight into the cache. Sounds strictly better (skip the future miss!) — but under **concurrent writers** to the same key, two updates can land **out of order** relative to their database writes. The cache can end up holding the *older* write's value even though the database correctly holds the newer one, and nothing looks wrong — the cache was "just updated," it just happened to be updated with stale data.
- **Delete it** — remove the entry and let the next reader repopulate it from the database via the normal cache-aside path. This costs one extra read, but whichever write actually committed *last* in the database is what the next read will see, because the read always goes back to the source of truth rather than trusting a value some earlier concurrent write pushed in.

This is why **cache-aside + delete-on-write** is the standard production default, and why Meta's original Memcache engineering guidance explicitly recommends deleting over setting.

**2. But delete-on-write still races — the dual-write problem**

Deleting correctly, at the correct time, doesn't mean a stale value can't reappear right after. Walk through two concurrent requests: **A** is writing a price update ($50 → $60); **B** is an ordinary read of the same product, landing at almost the same moment.

```mermaid
sequenceDiagram
    participant A as Request A (writer)
    participant B as Request B (reader)
    participant Cache
    participant DB

    Note over Cache: product:8214 not cached (recent TTL lapse)
    B->>Cache: GET product:8214
    Cache-->>B: miss
    B->>DB: SELECT price WHERE id=8214
    DB-->>B: $50 (old price - A hasn't committed yet)
    A->>DB: UPDATE price=$60 WHERE id=8214
    DB-->>A: commit OK
    A->>Cache: DEL product:8214
    Cache-->>A: deleted (correctly empty)
    Note over B: B's read already returned $50;<br/>B now writes that stale value back in
    B->>Cache: SET product:8214, $50
    Cache-->>B: ack
    Note over Cache: product:8214 cached at $50 again -<br/>STALE, with no further invalidation coming
```

Every step individually succeeded — A's write and delete both worked, B's read and populate both worked. The bug lives purely in the **interleaving**: B's stale read landed *before* A's write, but B's cache-populate landed *after* A's delete. The database now correctly holds $60 while the cache holds $50, freshly written, with a fresh TTL — and nothing is coming to fix it except that TTL eventually lapsing. **Whichever operation touches the cache last wins, and "last" doesn't mean "most recently correct."**

**3. Narrowing the gap — no single fix, layer several**

- **A short TTL as the ultimate backstop.** Since the race above leaves a stale value with nothing else pending to correct it, TTL is the only thing that unconditionally bounds how long the damage survives — which is why it's kept even in systems doing active invalidation for the common case.
- **Write-through, structurally.** No independent "read-then-populate" step exists for a concurrent reader to race against, since every write and cache update happens synchronously through the same path. Strongest fix, at write-through's usual cost (every write pays for the synchronous DB write too).
- **Delete twice.** Issue the normal delete right after the write, then a *second* delete for the same key a few hundred milliseconds later, specifically to clean up whatever a racing reader might have repopulated in between. Cheap, heuristic, genuinely effective because the race window is narrow.
- **Drive invalidation off the database's own commit log, not application code.** The race exists because the delete is a separate, application-issued step that can be slow, crash, or race with a reader. Change data capture (CDC) — reading the database's write/replication log directly and firing invalidation from *that* — removes application code from the path entirely: one authoritative signal, in commit order, per write. The **outbox pattern** gets a similar guarantee without reading the engine's internal log. Full mechanics of both live in L4.

**Multi-node wrinkle.** None of the above automatically reaches every copy if "the cache" isn't singular — a sharded cluster's replicas, 50 app instances each keeping their own in-process (L1) copy, or a CDN's edge nodes are all independent places the same stale value can persist. The common broadcast fix is **pub/sub**: the writer publishes "key X changed" and every subscriber deletes its own local copy on receipt — but this is fire-and-forget (a disconnected or restarting subscriber simply never gets the message), so it's always paired with a TTL backstop, never relied on alone.

## In the real world

**Meta — hunting the exact race above, at planet scale.** Meta's engineering post "Cache made consistent" describes hardening TAO's (its distributed graph cache over MySQL) consistency guarantee from **six nines to ten nines** of cache writes agreeing with the database within a 5-minute window, and names the canonical bug it chases down as "a cache fill landing after a write but before its invalidation message" — the identical interleaving as the dual-write race above, just at Meta's fan-out. To catch it, Meta built **Polaris**, a service that continuously samples cache replicas for consistency violations, resolving one production bug within about 30 minutes of it surfacing. Separately, Meta's older Memcache paper describes **mcsqueal**, a daemon that reads each database's **commit log directly** and drives invalidation from that — the CDC approach above, chosen specifically because deriving invalidations from the commit log guarantees they can never outrace the data change itself to a replica region. ([Meta Engineering](https://engineering.fb.com/2022/06/08/core-infra/cache-made-consistent/); [NSDI 2013 paper](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf))

**Netflix — EVCache deletes across regions, and accepts the resulting staleness on purpose.** Netflix's cross-region cache replication sends only a `DELETE` for an invalidated key, never the new value — "we don't send the new data, we just send a DELETE" — propagated via Kafka rather than a synchronous cross-region call. They name the trade-off explicitly: it's fine "if Ireland and Virginia occasionally have slightly different recommendations for you," with measured cross-region propagation around 400ms at p99 — a deliberate, per-data-type staleness budget rather than a universal guarantee. ([Netflix Tech Blog](https://netflixtechblog.com/caching-for-a-global-netflix-7bcc457012f1))

## Trade-offs

| Mechanism | Staleness window (worst case) | Fixes the dual-write race? | Best for |
| --- | --- | --- | --- |
| **TTL / passive expiry only** | Full TTL duration | No — doesn't address it, just bounds it | Data tolerant of minutes-scale staleness |
| **Update-on-write** | Milliseconds, absent ordering bugs | No — adds its own out-of-order bug | Rarely recommended alone under concurrent writers |
| **Delete-on-write** | Milliseconds, absent the race | No — the race happens even with correct deletes | The safe default active-invalidation choice |
| **Write-through** | None — never diverges | Yes — no independent read/write path to race | Correctness-critical data (balances, inventory) |
| **CDC / outbox-driven invalidation** | Sub-second, driven by log lag | Yes — one ordered, authoritative signal per commit | High-value data where residual race risk is unacceptable |
| **Pub/sub broadcast (multi-node)** | Milliseconds when delivered; unbounded if lost | No — orthogonal, addresses fan-out not the race | Multi-node/L1 fleets, always paired with TTL |

> [!IMPORTANT] Remember
> Whichever operation touches the cache **last** wins — and "last" has no relationship to "most recently correct." Delete-on-write is safer than update-on-write, but even a correct, on-time delete can be silently undone by a slower concurrent reader's stale repopulation. TTL is what eventually cleans that up when nothing else does — which is why every mechanism here keeps TTL as a backstop, never drops it.

## Check yourself

- A cache entry is corrected via `DEL` immediately after its database write commits, yet a reader still observes a stale value shortly afterward, with no further invalidation ever firing. Walk through the interleaving that produces this — why doesn't calling the delete "correct" and "on time" prevent it?
- A distributed cache cluster has three nodes holding a hot key, plus 20 app instances each keeping their own in-process copy. A write updates the row and issues one `DEL` against one of the three cluster nodes. How many stale copies remain, and what would it actually take to correct all of them?

→ Next: Negative caching
↩ comes back in: L4 (CDC and the outbox pattern give this topic's race its structurally strongest fix), L5 (consistency models generalize the "eventual, not strong" ceiling named here to every replicated system, not just caches)
