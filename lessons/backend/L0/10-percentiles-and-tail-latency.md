# Percentiles and Tail Latency

*"Our average response time is 100 ms" tells you almost nothing -- the slow requests hiding behind that average are the ones your users remember.*

`⏱️ ~6 min · 10 of 13 · System-Design Foundations`

> [!TIP] The gist
> A single latency number is a lie. Latency is a **distribution** -- a spread of one value per request -- and it's **right-skewed**: most requests are fast, a few are very slow. The **mean** gets dragged up by those slow ones and describes no real request, so you measure **percentiles** instead: p50 (the typical request), p99 and p99.9 (the slow **tail**). The tail is what users actually feel, and **fan-out amplifies it** -- when one request waits on many backends, a rare 1% slow event becomes the common case. Target the tail, not the average.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture a coffee shop that serves 100 customers one morning. Ninety-nine are in and out in 90 seconds. But one order gets lost, and that customer waits 10 minutes, fuming.

Write it up as an **average** and the shop looks great: `(99 x 1.5 min + 1 x 10 min) / 100` is about 1.6 minutes a customer. The number is comforting and completely misses the point -- there is a furious person in the corner, and *they* are the one who leaves the one-star review.

Nobody experiences the average. Each customer experiences their own wait. The whole art of measuring latency is refusing to let that one 10-minute wait get smeared into invisibility.

## The concept

**Latency** is the time a system takes to do work; in practice we measure **response time** -- the total time a client waits from sending a request to getting the full response, including network, queueing, and retries, not just the computing.

The key move: latency is not one value, it's a **distribution**, and real ones are **right-skewed**. There's a floor (a request can't take less than the real work) but no ceiling (it can get stuck for a long time), so a long "tail" of slow requests stretches to the right.

To summarize a distribution honestly, use **percentiles**. A percentile answers: *what value is this fraction of requests faster than?* Sort every response time fastest to slowest and read off a position:

- **p50** (the **median**): half of requests are faster, half slower -- the **typical** request.
- **p90 / p95**: 90% / 95% are faster.
- **p99**: 99% are faster; the slowest 1% are slower.
- **p99.9** ("three nines"): 99.9% are faster; the slowest 1 in 1000 are slower.

So **"p99 = 200 ms"** reads: 99 of every 100 requests finish under 200 ms, and 1 takes longer. (It says nothing about *how much* longer -- for that you look further out, at p99.9 or the max.)

The high percentiles -- p99, p99.9, and beyond -- are the **tail**. p50 is what a typical user sees; the tail is what your *unluckiest* users see, or what anyone sees on an unlucky request.

**What it is not:** the mean is not a percentile, and the tail is not "an error" -- those slow requests usually succeed, they're just slow. A wide gap between p50 and p99 signals a heavy tail; p50 close to p99 means a tight, predictable distribution.

## How it works

### Worked example: the mean lies, the median tells the truth

Ten requests complete with these response times (ms):

```
20, 22, 25, 25, 28, 30, 32, 35, 40, 2000
```

- **Mean** = (20 + 22 + 25 + 25 + 28 + 30 + 32 + 35 + 40 + 2000) / 10 = 2257 / 10 = **~226 ms**
- **Median (p50)** = average of the 5th and 6th sorted values = (28 + 30) / 2 = **29 ms**

The mean (~226 ms) is roughly **8x** the median -- and it describes *no actual request*: nine were under 40 ms, one was 2000 ms, and nothing landed anywhere near 226 ms. The mean sits in an empty gap between the fast cluster and the lone straggler.

The median (29 ms) honestly names the typical request, and the tail immediately exposes the 2000 ms outlier that the mean smears away. Report the median for "typical" and the tail for "worst realistic case" -- almost never decide on the mean alone.

### Fan-out: why the tail becomes the common case

At small scale a 1% slow rate sounds negligible. At scale it becomes the *majority* experience, because of **fan-out**.

**Fan-out** means one user-facing request internally triggers many backend calls -- to shards, services, cache nodes, replicas -- and can't return until all of them respond. The overall response time is then governed by the **slowest** of those calls, not the average one.

Suppose each backend call has a p99 of 200 ms -- a ~99% chance of being fast, a ~1% chance of being slow. If a request fans out to N backends and waits for all:

```
P(all N fast)          = 0.99^N
P(at least one slow)   = 1 - 0.99^N
```

- **N = 1:**   1 - 0.99^1   = **~1%**  chance the request hits a slow call
- **N = 10:**  1 - 0.99^10  = **~10%** chance
- **N = 100:** 1 - 0.99^100 = **~63%** chance

Read that last line slowly. Touch 100 backends each with a "great" p99, and **~63% of user requests** hit at least one backend in its slow tail. The backend's rare 1% event has become the user's **median**. That's **tail amplification**: independent tails compound as fan-out grows.

(The math assumes independent calls each with exactly a 1% slow rate; real correlations shift the exact figure. The direction and magnitude are the point.)

The design consequence is decisive: **service-level targets should be tail percentiles (p99, p99.9), not the mean or median.** Optimizing the mean can leave the tail untouched while more and more users land in it.

### What causes the tail (and why you can't average it)

The tail isn't slow happy-path code -- it's occasional, transient conditions that stall a normally-fast request:

- **GC pauses** -- a managed runtime freezes in-flight requests to reclaim memory.
- **Queueing** -- a request waits in line when the server is busy; delay spikes as utilization nears capacity.
- **Cold cache** -- a miss takes the slow path to the backing store (a bimodal fast/slow split).
- **Contention / locks** -- threads stall on a shared lock, row, or resource.
- **Retries** -- a retried call pays for the first attempt *plus* the second, landing far out in the tail.

And a measurement trap: **you cannot average percentiles.** Server A's p99 = 100 ms and server B's p99 = 300 ms does *not* make the fleet p99 = 200 ms -- percentiles are order statistics, not additive. Instead, record each server's distribution as a **histogram**, **merge the histograms** (sum the bucket counts), then compute the percentile from the combined data. And **measure at the client** where possible -- a server reporting "handled in 5 ms" hides the queueing, network, and retry time the user actually waited through.

## Trade-offs

| Choice | Why teams do it | The cost |
|---|---|---|
| Target **p99 / p99.9** instead of the mean | Matches what users and fan-out actually expose; catches the slow requests that drive complaints | Needs many samples (thousands for p99.9) and histogram-based aggregation, not simple averages |
| Chase **ever-higher percentiles** (p99.99, max) | Fewer users ever hit a bad request; critical for high-fan-out systems | Sharply diminishing returns -- squeezing the far tail means hedged requests, over-provisioning, and rising cost/complexity for a shrinking slice of traffic |
| Optimize the **mean** | Easy to compute and report | Can leave the tail untouched or worse while users increasingly land in it |

## Remember

> [!IMPORTANT] Remember
> Optimize the **tail, not the average**. The mean hides your slowest requests; percentiles expose them. At scale, fan-out turns a rare per-backend 1% slow event into the *common* user experience -- the backend's p99 becomes the user's median. Measure p50 for "typical" and p99/p99.9 for "what my unlucky users feel," aggregate from merged histograms (never average percentiles), and measure at the client.

## Check yourself

1. A service reports mean = 50 ms and p99 = 1200 ms. What does the gap tell you about the shape of the distribution, and which number belongs in a service-level goal?
2. A request fans out to 50 backends, each with p99 = 100 ms, and waits for all of them. Roughly what fraction of user requests will see at least one call exceed 100 ms? (Use `1 - 0.99^N`.)

---

→ Next: [Throughput vs Latency](11-throughput-vs-latency.md)
↩ Comes back in: SLOs, load balancing, caching, reliability, observability
