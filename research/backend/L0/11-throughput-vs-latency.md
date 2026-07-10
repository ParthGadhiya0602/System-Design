# Throughput vs Latency

Two different axes of "fast" that beginners routinely conflate: how long one request takes, versus how many requests a system finishes per second.

## Contents

- [Overview and definitions](#overview-and-definitions)
- [Why they are not the same, and can trade off](#why-they-are-not-the-same-and-can-trade-off)
- [How they relate](#how-they-relate)
- [The bottleneck view](#the-bottleneck-view)
- [Which to optimize when](#which-to-optimize-when)
- [A short worked illustration](#a-short-worked-illustration)
- [How this connects onward](#how-this-connects-onward)

## Overview and definitions

Two words get thrown around as if they mean the same thing. They do not.

- **Latency** is the time to complete _one_ request, measured start to finish. It answers **how long**. Units are time: milliseconds (ms), microseconds (us), seconds. Example: "this API responds in 40 ms."
- **Throughput** is the number of requests (or bytes, or jobs) _completed per unit of time_. It answers **how many**. Units are a count over time: requests per second (req/s or QPS, queries per second), messages per second, MB/s. Example: "this service handles 12,000 req/s."

The key idea: **latency is measured per operation; throughput is measured per unit time.** They are different axes. A system has both at once, and improving one does not automatically improve the other. You can be fast per request and still process few requests per second, or process a huge number per second while each individual request is slow.

A common beginner mistake is to say "the system is fast" and mean only one of these. Always ask: fast at _what_? A quick reply for a single user (latency), or a lot of work done overall (throughput)? Serious system design states a target for each separately, for example "p99 latency under 100 ms at 5,000 req/s."

(Note: latency is rarely a single number because different requests take different times. It is usually described by percentiles such as p50, p95, p99 - covered in the tail-latency topic. Here we treat "latency" as the time of a representative request.)

## Why they are not the same, and can trade off

Because they measure different things, the same system can score high on one and low on the other. Four combinations exist:

- **Low latency, low throughput** - a single fast worker. Each request finishes quickly, but only one is handled at a time, so the total rate is small.
- **High throughput, high latency** - big batches. You wait to collect 10,000 items, then process them all in one efficient pass. Rate per second is huge, but any individual item waited a long time before its result came back.
- **Low latency, high throughput** - the goal for many systems, and usually the hardest and most expensive to reach.
- **High latency, low throughput** - a badly designed or overloaded system: slow _and_ not much gets done.

**Highway analogy.** Picture a highway between two cities.

- **Latency** = the time it takes _one car_ to drive from start to finish. This depends on distance and speed limit.
- **Throughput** = the number of cars that pass a point _per hour_. This depends on how many lanes there are.

Now the crucial insight: **adding a lane raises throughput but does not make any single car arrive sooner.** A car still travels the same distance at the same speed limit - its latency is unchanged - but more cars flow per hour because they travel side by side. Conversely, raising the speed limit lowers each car's travel time (latency) without necessarily changing how many lanes carry traffic. Widening (more lanes) and speeding up (higher limit) are independent levers.

They can actively _trade off_. Batching (waiting to fill a bus) raises throughput - one trip carries 50 people instead of 1 - but each passenger waits for the bus to fill and arrives later. That is a latency cost paid to buy throughput.

## How they relate

Several standard techniques raise throughput. Some cost latency; one mostly does not.

- **Batching** - group many small operations into one larger operation to amortize fixed per-operation cost (a network round trip, a disk seek, a transaction commit). Throughput rises sharply. But early items in the batch wait for the batch to fill or for a timer to fire, so _tail_ latency for those items rises. Trade-off: bigger batches -> more throughput, more waiting.
- **Pipelining** - split work into stages and let stage 2 start on request A while stage 1 works on request B, like an assembly line. Throughput rises because stages run in parallel across requests. A single request's end-to-end latency is not reduced (it still passes through every stage), and can even grow slightly from hand-off overhead - but the system finishes far more requests per second.
- **Parallelism / concurrency** - run many workers at once. This is the cleanest lever: it multiplies throughput while each request's latency stays roughly the same, because requests are handled side by side rather than made individually faster. This is the highway "add a lane" move.

The relationship between how many things are in flight, how fast each finishes, and the overall rate is captured by **Little's Law**: for a stable system, `concurrency ~= throughput x latency` (the average number of requests in flight equals the arrival/completion rate times the average time each spends in the system). One line here as a preview; it is developed in depth in the next topic.

## The bottleneck view

A request usually passes through several stages: parse -> authenticate -> query the database -> serialize the response. Think of these as pipes of different widths connected in series.

**Throughput of the whole system is capped by the slowest stage - the bottleneck.** If parsing can do 50,000/s, auth 40,000/s, but the database can only do 2,000/s, the system does ~2,000/s. The fast stages simply idle, waiting on the slow one. Water flows only as fast as the narrowest pipe allows.

The practical consequence: **to raise throughput, widen the bottleneck, not the fast stages.** Optimizing parsing from 50,000/s to 80,000/s changes the system total by zero, because the database still caps it at 2,000/s. You must find the slowest stage and give it more capacity - more database workers, partitioning/sharding the data across machines, adding a cache in front, or replacing the slow component.

Two more rules follow:

- After you widen one bottleneck, the bottleneck _moves_ to whatever is now slowest. Capacity work is iterative: fix the slowest, re-measure, repeat.
- Widening the bottleneck raises throughput but does not necessarily lower latency. A single request still traverses every stage at its own speed. (In fact, if the bottleneck was overloaded and queueing, widening it can _lower_ latency by draining the queue - but that is removing wait time, not making the work itself faster.)

## Which to optimize when

The requirement decides. State a target for whichever matters, and often for both.

- **Optimize latency** for user-facing, interactive paths where a human waits on the result: loading a page, a search box, a checkout, a chat message, an autocomplete. Here the metric is a **tail percentile** (p99, p99.9), not the average, because a small fraction of slow requests is what users actually notice and complain about. A fast average with a slow tail still feels broken.
- **Optimize throughput** for bulk, background, and pipeline work where no single human is blocked on any one item: analytics jobs, log/event ingestion, batch ETL (extract-transform-load), video encoding, nightly report generation, training-data preparation. Here you care about total work per hour and cost per unit of work; adding a few seconds of latency to any single item is fine if the aggregate rate is high.
- **Optimize both, with an explicit target for each,** for most real production systems - for example, "serve p99 < 120 ms while sustaining 8,000 req/s." Stating them separately forces you to size for peak load (throughput) _and_ keep the per-request experience good (latency), which are different design problems.

The condition to flip your choice: if a human is waiting on the specific result, lead with latency; if the work is aggregate and deferrable, lead with throughput.

## A short worked illustration

Take one worker that spends 10 ms of processing on each request and handles requests one at a time (no overlap).

- Latency per request = **10 ms**.
- In one second (1,000 ms) it completes `1000 / 10 = 100` requests.
- Throughput = **~100 req/s**.

Now run **10 identical workers in parallel**, each still taking 10 ms per request, requests spread evenly across them:

- Latency per request = **still ~10 ms** - each worker does the same work at the same speed; nothing made a single request faster.
- Combined completions per second = `10 workers x 100 req/s each = ~1,000 req/s`.
- Throughput = **~1,000 req/s** - a 10x gain purely from running side by side.

The lesson in numbers: **parallelism multiplied throughput by 10 while leaving latency unchanged.** To _lower latency_ you would need a different move entirely - do less work per request, use a faster algorithm, cache the result, or use faster hardware - none of which is what adding workers did.

Cross-check with Little's Law: to be "in flight" on 10 requests at once (concurrency = 10) with each taking 10 ms (latency = 0.010 s), the sustainable rate is `concurrency / latency = 10 / 0.010 = 1,000 req/s`. The arithmetic agrees.

## How this connects onward

- **Little's Law** (next topic) makes the concurrency / throughput / latency relationship precise and shows how much parallelism you need to hit a target rate.
- **Capacity planning** (later levels) uses throughput targets plus per-request cost to decide how many machines, partitions, and replicas to provision, and to size for peak vs average load.
- **Queueing and backpressure** (later levels) explain what happens when arrival rate exceeds throughput: queues grow, latency spikes, and the system must shed or slow incoming load rather than fall over.
- **Batching** reappears as a deliberate throughput-for-latency trade in databases, message systems, and I/O.
- **Load balancing** (later levels) is how requests get spread across parallel workers so the added throughput is actually realized instead of piling onto one hot node.
