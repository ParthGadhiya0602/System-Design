# Throughput vs Latency

*"Is it fast?" is two questions wearing one coat -- how long one request takes, and how many requests finish per second.*

`⏱️ ~5 min · 11 of 13 · System-Design Foundations`

> [!TIP] The gist
> **Latency** is the time for *one* request (measured in ms). **Throughput** is *how many* requests finish per second (measured in req/s). They are **separate axes**: a system has both at once, and improving one does not automatically improve the other. Adding parallel workers multiplies throughput while leaving each request's latency unchanged -- more lanes, not a higher speed limit. They can even trade off: batching buys throughput by making individual items wait.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture a highway between two cities.

**Latency** is the time it takes *one car* to drive from start to finish. That depends on the distance and the speed limit.

**Throughput** is the number of cars that pass a point *per hour*. That depends on how many lanes there are.

Now the crucial insight: **adding a lane raises throughput but does not make any single car arrive sooner.** A car still covers the same distance at the same speed limit -- its latency is unchanged -- but more cars flow per hour because they travel side by side. Raising the *speed limit* is the other lever: it lowers each car's travel time without adding lanes.

Widening (more lanes) and speeding up (higher limit) are independent moves. Confusing them is the classic beginner mistake: "the system is fast" -- fast at *what*? A quick reply for one user, or a lot of work done overall?

## The concept

**Latency** is the time to complete *one* request, start to finish. It answers **how long**. Units are time: milliseconds (ms), microseconds (us), seconds. Example: "this API responds in 40 ms." (Latency is really a distribution described by percentiles like p99 -- see the previous topic. Here we treat it as the time of a representative request.)

**Throughput** is the number of requests -- or bytes, or jobs -- completed *per unit of time*. It answers **how many**. Units are a count over time: requests per second (req/s, or QPS), messages per second, MB/s. Example: "this service handles 12,000 req/s."

The key distinction: **latency is measured per operation; throughput is measured per unit time.** Because they measure different things, the same system can score high on one and low on the other. You can be fast per request yet handle few per second (one quick worker), or handle a huge number per second while each item is slow (big batches).

Serious design states a target for **each** separately, for example "p99 under 100 ms at 5,000 req/s." A few terms carry the rest of the lesson:

- **Bottleneck** -- the slowest stage in a pipeline; it caps the whole system's throughput.
- **Batching** -- grouping many operations into one to raise throughput (often at a latency cost).
- **Concurrency** -- how many requests are in flight at once; the lever that multiplies throughput.

## How it works

### They move independently

Several standard techniques raise throughput. Some cost latency; one mostly does not.

- **Parallelism / concurrency** -- run many workers at once. The cleanest lever: it multiplies throughput while each request's latency stays roughly the same, because requests are handled side by side rather than made individually faster. This is the "add a lane" move.
- **Batching** -- group many small operations into one larger operation to amortize a fixed per-operation cost (a network round trip, a disk seek, a commit). Throughput rises sharply, but early items wait for the batch to fill, so their latency rises. A latency cost paid to buy throughput.
- **Pipelining** -- split work into stages so stage 2 handles request A while stage 1 handles request B, like an assembly line. Throughput rises; a single request's end-to-end latency is not reduced (it still passes through every stage).

### The bottleneck caps throughput

A request passes through stages in series -- think of pipes of different widths connected end to end. **Throughput of the whole system is capped by the slowest stage, the bottleneck.**

```
 request --> [ parse ] --> [ auth ] --> [  database  ] --> response
             50,000/s      40,000/s        2,000/s
                                        ^^^^^^^^^^^^
                                        bottleneck  -->  system does ~2,000/s
```

The fast stages simply idle, waiting on the slow one -- water flows only as fast as the narrowest pipe allows. So **to raise throughput, widen the bottleneck, not the fast stages.** Speeding parse from 50,000/s to 80,000/s changes the total by *zero*; the database still caps it at 2,000/s. Give the slow stage more capacity (more workers, sharding, a cache), then re-measure -- the bottleneck simply *moves* to whatever is now slowest.

### Worked example: 10x throughput, same latency

One worker spends **10 ms** of processing per request, one at a time:

- Latency per request = **10 ms**.
- In one second (1,000 ms) it completes `1000 / 10 = 100` requests.
- Throughput = **~100 req/s**.

Now run **10 identical workers in parallel**, each still 10 ms per request:

- Latency per request = **still ~10 ms** -- nothing made a single request faster.
- Combined = `10 workers x 100 req/s = ~1,000 req/s`.
- Throughput = **~1,000 req/s** -- a 10x gain purely from running side by side.

Parallelism multiplied throughput by 10 and left latency untouched. To *lower* latency you would need a different move entirely: do less work per request, a faster algorithm, a cache, or faster hardware.

*Little's Law preview (next topic):* for a stable system, `concurrency ~= throughput x latency`. Cross-check: 10 in flight at 10 ms each = `10 / 0.010 = 1,000 req/s`. The arithmetic agrees.

## Trade-offs

| Lead with... | When | Metric to target |
|---|---|---|
| **Latency** | User-facing, interactive paths where a human waits: page load, search box, checkout, chat, autocomplete | tail percentile (p99, p99.9) |
| **Throughput** | Bulk, background, pipeline work with no human blocked per item: analytics, log/event ingestion, batch ETL, video encoding | total work/hour, cost per unit |
| **Both, explicit target each** | Most real production systems | e.g. "p99 < 120 ms at 8,000 req/s" |

The flip condition: if a human is waiting on the specific result, lead with latency; if the work is aggregate and deferrable, lead with throughput.

## Remember

> [!IMPORTANT] Remember
> Latency (time per request) and throughput (requests per second) are **separate axes**. More parallelism buys **throughput, not lower per-request latency** -- adding lanes moves more cars per hour but does not make any one car arrive sooner. Raise throughput by widening the **bottleneck**, the slowest stage; lower latency with an entirely different set of moves.

## Check yourself

1. A single worker takes 20 ms per request. What is its throughput, and what happens to *latency* and *throughput* if you run 5 such workers in parallel?
2. A pipeline runs parse (30,000/s) -> validate (10,000/s) -> write (4,000/s). What is the system's throughput, and which stage should you optimize to raise it?

---

→ Next: [Little's Law](12-littles-law.md)
↩ Comes back in: capacity planning, queueing/backpressure, batching, load balancing
