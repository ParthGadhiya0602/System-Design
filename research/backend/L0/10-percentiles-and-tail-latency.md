# Percentiles and Tail Latency

Averages hide the slow requests that actually break user experience; to reason about performance you measure the distribution of latencies, and the high percentiles (the tail) are what your service-level goals should target.

## Contents
- [Overview](#overview)
- [Percentiles defined](#percentiles-defined)
- [Why the mean is misleading](#why-the-mean-is-misleading)
- [Why tail latency matters at scale](#why-tail-latency-matters-at-scale)
- [Causes of the tail and mitigations](#causes-of-the-tail-and-mitigations)
- [Measuring it right](#measuring-it-right)
- [How this connects onward](#how-this-connects-onward)

## Overview

**Latency** is the time a system takes to do work. In practice we usually measure **response time**: the total time a client waits from sending a request to receiving the full response. Response time includes the actual service (processing) time plus everything else the request experiences along the way: network transit, time spent waiting in queues, retries, and so on. People often say "latency" loosely to mean "response time"; the distinction matters because most of the pain a user feels lives in the *waiting*, not the *computing*.

The central lesson of this topic: **a single latency number is a lie**. If someone tells you "our average response time is 100 ms," you have learned almost nothing useful. Latency is not a single value, it is a *distribution* -- a spread of many different values, one per request. Two systems can have the same average while one feels fast and reliable and the other feels erratic and broken. To understand and control performance, you must look at the shape of that distribution, especially its slow end.

Latency distributions in real systems are almost never symmetric. They are **right-skewed**: most requests are fast and cluster near a low value, but a minority are much slower, forming a long "tail" stretching to the right. There is a hard floor (a request cannot take less than zero time, and usually not less than some minimum of real work) but no ceiling (a request can get stuck for a very long time). This asymmetry is exactly why the mean is a poor summary and why we reach for percentiles instead.

## Percentiles defined

A **percentile** answers the question: "what value is this fraction of my requests faster than?" To compute it, imagine sorting every request's response time from fastest to slowest, then picking the value at a given position.

- **p50** (the 50th percentile, also called the **median**): the value that half of requests are faster than and half are slower than. It represents the "typical" request.
- **p90**: 90% of requests are faster than this value; the slowest 10% are slower.
- **p95**: 95% are faster; slowest 5% are slower.
- **p99**: 99% are faster; slowest 1% are slower.
- **p99.9** (the "three nines" percentile): 99.9% are faster; the slowest 1 in 1000 are slower.

So a statement like **"p99 = 200 ms"** means: 99 out of every 100 requests complete in under 200 ms, and 1 out of 100 takes longer. It says nothing about *how much* longer that slow 1% takes -- for that you look further out (p99.9, p99.99, or the maximum).

The high percentiles -- p99, p99.9, and beyond -- are collectively called the **tail** (or **tail latency**). These are your slowest requests. A useful mental model: p50 tells you what a typical user sees, while the tail tells you what your *unluckiest* users see, or what any user sees on their unluckiest requests. The more percentiles you look at, the more of the distribution's shape you can see:

- p50 far below p99 means most requests are fast but a meaningful minority are slow -- a heavy tail.
- p50 close to p99 means the distribution is tight and predictable.

## Why the mean is misleading

The **mean** (arithmetic average) adds up all response times and divides by the count. Its fatal weakness for latency is that it is dominated by extreme values and cannot show you the shape of the distribution. A handful of very slow requests barely move the average, yet those are precisely the requests that ruin someone's experience.

Consider a small worked example. Suppose 10 requests complete with these response times (ms):

```
20, 22, 25, 25, 28, 30, 32, 35, 40, 2000
```

- **Mean** = (20 + 22 + 25 + 25 + 28 + 30 + 32 + 35 + 40 + 2000) / 10 = 2257 / 10 = **~226 ms**
- **Median (p50)** = average of the 5th and 6th sorted values = (28 + 30) / 2 = **29 ms**

The mean (~226 ms) is roughly 8x the median (29 ms), and worse, it describes *no actual request*: nothing took anywhere near 226 ms -- nine requests were under 40 ms and one took 2000 ms. The mean lands in an empty gap between the fast cluster and the slow outlier. The median (29 ms) honestly describes the typical request, and if we looked at the tail we would immediately see the 2000 ms straggler that the mean smears into invisibility.

Now flip the intuition around. Suppose that one 2000 ms request is a real user waiting two seconds for a page. The mean averages that pain away into a comfortable-looking 226 ms. But **users do not experience the average** -- each user experiences their own request. The person who waited 2000 ms had a terrible experience regardless of how fast everyone else's requests were. This is why you report and target the median for "typical" and the tail for "worst realistic case," and why you almost never make decisions on the mean alone.

A second, subtler problem: means cannot be compared safely across changing traffic mixes, and they hide bimodal behavior. If a service has a fast path (cache hit) and a slow path (cache miss), the distribution has two humps; the mean sits in the valley between them and misrepresents both. Percentiles expose that structure.

## Why tail latency matters at scale

At small scale a 1% slow rate sounds negligible. At scale it becomes the common case, because of **fan-out** and **tail amplification**.

**Fan-out** means a single user-facing request internally triggers many backend calls -- to shards, microservices, cache nodes, or replicas -- and cannot return until all of them (or a required subset) have responded. The overall response time is therefore governed by the **slowest** of those backend calls, not the average one. Waiting on the slowest of many parallel operations is where the tail dominates.

Here is the arithmetic idea. Suppose each backend call independently has a p99 of 200 ms -- meaning any single call has a ~1% chance of exceeding 200 ms, or a ~99% chance of being under it. If one user request fans out to N backends and must wait for all N, the probability that *every* call comes back fast is:

```
P(all N fast) = 0.99^N
```

so the probability that *at least one* is slow (exceeds 200 ms) is:

```
P(at least one slow) = 1 - 0.99^N
```

Plugging in values:

- N = 1:   1 - 0.99^1   = ~1%   chance the request hits a slow call
- N = 10:  1 - 0.99^10  = ~9.6% chance
- N = 100: 1 - 0.99^100 = ~63%  chance

Read that last line carefully. If a request touches 100 backends each with a "great" p99, then **~63% of user requests** will experience at least one backend in its slow tail. In other words, the backend's rare 1% event becomes the *majority* experience at the top level. The per-backend p99 has effectively become the user's median. This is tail amplification: independent tails compound as fan-out grows.

(The exact numbers assume the calls are independent and each has exactly a 1% slow rate; real systems have correlations that can make this better or worse. The point is the direction and magnitude, not the precise percentage.)

The consequence for design is decisive: **your real service-level targets should be tail percentiles (p99, p99.9), not the mean or even the median.** Optimizing the mean can leave the tail untouched -- or make it worse -- while your users increasingly land in it. When you set goals (covered under service-level objectives in a later level), you state them as "p99 under X ms," because that is what fan-out and repeated usage expose users to.

## Causes of the tail and mitigations

Tail latency usually is not caused by slow code on the happy path. It is caused by occasional, transient conditions that make a normally-fast request wait. Common causes:

- **Garbage collection (GC) pauses**: managed-runtime servers periodically pause to reclaim memory, freezing in-flight requests for milliseconds to seconds.
- **Queueing**: when a request arrives while the server or a thread pool is busy, it waits in line. Queue delay grows sharply as utilization approaches capacity (a queueing effect explored in later levels).
- **Cold caches**: a request that misses the cache must fetch from a slower backing store, taking a slow path while cache hits take the fast path -- producing a bimodal distribution.
- **Contention and locks**: threads waiting on a shared lock, database row, or other contended resource stall unpredictably.
- **Retries**: a failed or timed-out call that is retried pays the cost of the first attempt plus the second, landing far out in the tail.
- **Slow disks / I/O**: an occasional slow disk read, page fault, or storage hiccup delays otherwise-fast requests.
- **Noisy neighbors**: on shared hardware or virtualized/multi-tenant infrastructure, another workload's CPU, memory, or I/O burst steals resources and slows your requests.

High-level mitigations (each is detailed in later levels; named here so you know they exist):

- **Timeouts**: cap how long you will wait, converting an unbounded stall into a bounded, handleable failure.
- **Hedged requests**: after a short delay, send a duplicate request to another replica and take whichever returns first, so one slow node cannot dictate the response time.
- **Load shedding**: deliberately reject or degrade some requests when overloaded, protecting the latency of the rest instead of letting queues explode for everyone.
- **More even load balancing**: spread work so no single node runs hot and develops a long queue; uneven balancing manufactures tail latency.
- **Reducing fan-out or making it partial**: return once a "good enough" subset of backends respond, rather than waiting for the slowest.

## Measuring it right

Percentiles are easy to get wrong when aggregating across many servers or time windows.

- **You cannot average percentiles.** The average of two servers' p99 values is *not* the p99 of their combined traffic. Percentiles are order statistics, not additive quantities; averaging them produces a number with no real meaning. If server A has p99 = 100 ms and server B has p99 = 300 ms, the fleet's true p99 could be anywhere depending on each server's traffic volume and full distribution -- it is not 200 ms.
- **Aggregate from histograms, not from summary numbers.** The correct approach is to record each server's latency *distribution* as a histogram (counts of requests falling into latency buckets), then merge the histograms by summing the bucket counts, and only then compute percentiles from the merged distribution. Histograms merge correctly because you are recombining the underlying data, not pre-summarized statistics.
- **Measure at the client where possible.** A server that reports "I handled this in 5 ms" is telling the truth about its own processing but hiding queueing delay, network time, and retries that the client actually waited through. Measuring at (or close to) the client captures the *response time the user really experienced*, including the time a request sat in a queue before the server even started it -- a source of tail latency that server-side timing misses entirely.
- **Collect enough samples.** To speak about p99.9 meaningfully you need many thousands of requests; with only a few hundred samples the extreme percentiles are dominated by noise.

## How this connects onward

- **Service-level objectives (reliability level):** tail percentiles become the formal targets you promise and measure against ("p99 < 200 ms"), and the basis for error budgets.
- **Load balancing (networking level):** even distribution of requests prevents hot nodes with long queues, directly shaping the tail; hedging and retries interact with balancing choices.
- **Caching (later levels):** cache hit ratio splits the distribution into fast and slow paths; improving hit rate pulls the tail in, while cache misses and stampedes push it out.
- **Reliability engineering:** timeouts, hedged requests, and load shedding are the concrete tools for cutting the tail; they trade extra work or rejected requests for predictable latency.
- **Observability (later levels):** histogram-based metrics, distributed tracing, and per-request timing are how you actually *see* the distribution and locate which hop in a fan-out created the tail.
- **Estimation and capacity planning (this foundational level):** the same fan-out arithmetic that amplifies the tail also drives how you size systems, since queueing and utilization directly move the high percentiles.

### Check yourself
- A service reports mean = 50 ms and p99 = 1200 ms. What does this tell you about the shape of the distribution, and which number would you put in an SLO?
- A request fans out to 50 backends, each with p99 = 100 ms, and waits for all of them. Roughly what fraction of user requests will see at least one call exceed 100 ms? (Use 1 - 0.99^N.)
- Why is merging histograms the correct way to get a fleet-wide p99, but averaging each server's p99 is not?
- Give two reasons a server-side latency of 5 ms might correspond to a client-observed response time of 300 ms.
