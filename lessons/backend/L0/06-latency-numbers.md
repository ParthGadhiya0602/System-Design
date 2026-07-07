# Latency Numbers Every Engineer Should Know

*If a memory read took one second, a round trip across the planet would take about three years -- same machine, same request, wildly different worlds.*

`⏱️ ~6 min · 6 of 13 · System-Design Foundations`

> [!TIP] The gist
> A handful of order-of-magnitude timings let you turn "N disk reads + M network round trips" into a **latency budget** you can check on paper before writing code. Don't memorize the digits -- they drift with hardware every year. Memorize the **ratios** and the ladder: `memory << SSD << network (same building) << disk seek << cross-continent`. End to end that ladder spans ~8 orders of magnitude (a factor of ~100,000,000). Almost every design move -- caching, indexing, CDNs, multi-region -- is an attempt to slide work *down* toward the fast end.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

The tiers of a computer feel like they run at similar speeds. They do not -- they run in completely different universes, and our intuition has no feel for the gap.

So scale it up to human time. Pretend a single **L1 cache read** (~0.5 ns) takes **1 second**. On that scale:

- Reading from **RAM** (~100 ns) -- about **3-4 minutes**.
- An **SSD** random read (~100 us) -- about **2.5 days**.
- An **HDD seek** (~10 ms) -- about **8 months**.
- A **cross-continent** round trip (~100-150 ms) -- roughly **3 to 10 years**.

Same request, same code -- but one branch of it finishes before you blink and another takes years. That is the ~8-orders-of-magnitude gap you are budgeting against.

## The concept

**Definition.** A **latency budget** is a running sum of the expected time a request will take, built by counting the slow operations it triggers (disk reads, network round trips) and multiplying each by its rough cost. You compare that sum against a target -- say **p99 under 200 ms** -- to decide, on paper, whether a design can possibly meet its goal.

> **p99** = the 99th-percentile latency: 99 of 100 requests are at least this fast. It is what users feel on a bad day, not the average.

**Why ratios, not digits.** The exact nanosecond counts change every year with hardware. The *relationships between tiers* -- memory vs SSD vs disk vs network -- have been stable for decades and are what you actually reason with. (This table was popularized by Jeff Dean and Peter Norvig.)

**The ladder, slowest step to slowest:**

```
memory  <<  SSD  <<  network (same building)  <<  disk seek  <<  cross-continent
```

Each `<<` means "much slower than." The key terms:

- **Cache / RAM** -- fast working memory; RAM is off-chip so much slower than the CPU's on-chip L1/L2 caches.
- **SSD** (solid-state drive) -- flash storage, no moving parts.
- **HDD** (hard disk drive) -- spinning platters; a physical head must **seek** to the right track first.
- **Round trip (RTT)** -- time for a message to reach another machine and the reply to return.
- **Sequential vs random** -- reading adjacent bytes vs jumping to scattered ones; sequential is far cheaper on every tier.

**What it is / isn't.** A latency budget is a *sanity check made of labeled arithmetic*, not a benchmark and not a guarantee. Its whole worth is catching a design that fails on paper -- for a tenth of the cost of catching it after you build it.

## How it works

Read the far-right column top to bottom and watch the zeros pile up -- that growth *is* the 8 orders of magnitude made concrete. Every figure is order-of-magnitude and hardware-dependent; treat `~` as "roughly, on today's commodity hardware."

| Operation | Approx. time | In nanoseconds (~) |
|---|---|---|
| L1 cache reference | ~0.5 ns | ~0.5 |
| Main memory (RAM) reference | ~100 ns | ~100 |
| SSD random read | ~100 us | ~100,000 |
| Round trip, same datacenter | ~0.5 ms | ~500,000 |
| Disk seek (HDD, move the head) | ~10 ms | ~10,000,000 |
| Round trip, cross-continent | ~100-150 ms | ~100,000,000-150,000,000 |

**The intuitions that hang off it:**

- **Each big step is ~1000x the last.** SSD random read is **~1000x slower than RAM**; HDD seek is **~100x slower than SSD** and **~100,000x slower than RAM**. *This is why caches exist* -- keeping hot data in RAM instead of on disk is not a tweak, it is a 1000x-to-100,000x win.
- **A same-DC round trip (~0.5 ms) is ~5000x a RAM read.** Cheap by human standards, enormous by CPU standards -- which is why "chatty" designs that make many tiny network calls per request fall over. **Batch** and **parallelize** the hops.
- **Cross-continent (~100-150 ms) is a physics floor.** Light in fiber is ~2/3 the speed of light; the distance simply takes ~100 ms there and back, no matter how much you spend. You cannot code around it -- you can only **not send the request that far** (move data/compute closer).
- **Sequential >> random on every tier**, because you pay the "find it" cost once instead of per item.

### Worked mini-example: a profile page

A profile page needs 1 user record + 20 of the user's posts. Target: **p99 under 100 ms.**

**Naive design** -- 1 lookup for the user + 1 per post = **21 random reads**, each a same-DC round trip to the database at ~0.5 ms:

```
21 x ~0.5 ms  =  ~10.5 ms      -> comfortably under 100 ms  (if reads hit RAM/SSD-backed storage)
```

**Same design, but the data is on spinning disk and uncached** -- each read now costs an HDD seek (~10 ms):

```
21 x ~10 ms   =  ~210 ms       -> OVER budget. Fails on paper.
```

Fixes, ranked by impact:
1. **Put hot data in RAM/SSD** -- removes the ~10 ms seek, back to ~10 ms total.
2. **Batch** the 20 post reads into one query -- 21 round trips become 2, cutting round-trip count ~10x.

**And if the database sits on another continent**, add ~100-150 ms *per round trip*: 21 cross-continent trips would be *seconds*. That is why you keep the datastore in the same region and push a **read replica** or **cache** near the user.

You reached the right verdict -- and the right fix -- with arithmetic alone, before building anything. That is the entire point of the table.

## Trade-offs

Practical takeaways, all of them a move down the ladder:

- **Cache to skip slow hops.** Serving from RAM avoids the SSD (~1000x), disk (~100,000x), or a network trip -- the biggest lever you have.
- **Batch and parallelize round trips.** Many small calls per request is the classic killer; one bigger call, or concurrent calls, collapses the cost.
- **Move compute/data closer for cross-region.** You can't beat the speed of light, so shorten the distance: edge caches / CDNs, multi-region deployments, replicas near users.
- **Spend the fast resource to save the slow one.** CPU/RAM (cheap) to avoid network/disk (expensive) -- e.g. compress a payload to shrink a slow hop, *as long as* it actually reduces the slow work.

## Remember

> [!IMPORTANT] Remember
> Memorize the **ratios, not the digits**: `memory << SSD << network << disk seek << cross-continent`, each big step ~1000x the last, ~8 orders of magnitude end to end. Count the slow operations a request triggers, multiply by their rough cost, sum, and compare to your p99 target -- and if you're over budget, slide the work down the ladder (cache, batch, move closer).

## Check yourself

1. Put these in order fastest to slowest and give the rough ratio between neighbors: RAM read, HDD seek, SSD random read, same-DC round trip.
2. A request makes 50 random reads. Estimate the total time if each hits RAM, if each hits an SSD, and if each causes an HDD seek. Which of these fits a 100 ms p99 budget -- and what is the *only* real fix if the database is on another continent?

---

→ Next: [Availability, reliability, scalability, maintainability](07-availability-reliability-scalability-maintainability.md)
↩ Comes back in: caching, storage/indexing, CDN, multi-region
