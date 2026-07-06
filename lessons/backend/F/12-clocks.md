# Clocks

*There are two kinds of time in a computer, using the wrong one is a bug, and no two machines ever quite agree on either.*

`⏱️ ~7 min · 12 of 12 · Computing Fundamentals`

## Contents

- [The gist](#the-gist)
- [Intuition](#intuition)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

> [!TIP] The gist
> A computer has two clocks: the **wall clock** (calendar time, but it can jump backward and forward) and the **monotonic clock** (only ever moves forward, for measuring durations). Use the right one or you get subtle bugs. And across machines, clocks never perfectly agree (skew), so you *cannot* trust wall-clock timestamps to order events on different machines.

## Intuition

Think of two clocks on your wall. One is a **calendar clock** synced to the official time signal — occasionally it gets nudged forward or back to stay correct. The other is a **stopwatch** — once started, it only counts up, never resets or jumps. To ask "what date is it?" you read the calendar clock. To ask "how long did that take?" you read the stopwatch. Mix them up and you might measure a task as taking *negative* time.

## How it works

**Wall clock — calendar time that can jump.**
Represents real-world time (e.g. seconds since the 1970 Unix epoch). It answers "what time is it now" and is right for timestamps humans interpret. But it **can jump** both ways: NTP corrections, manual changes, leap seconds. So it is *not safe* for measuring elapsed time — if it's adjusted backward mid-measurement, `end - start` can go negative.

<br>

**Monotonic clock — only ever forward.**
Guaranteed never to jump or be adjusted, exposed specifically for measuring durations (`CLOCK_MONOTONIC`, `System.nanoTime()`). Rule of thumb: **monotonic** for timeouts, latency, rate limiting, any elapsed interval; **wall clock** (stored as UTC / ISO-8601 / Unix timestamp) for "when did this happen."

<br>

**NTP, skew, and drift.**
**NTP** syncs a machine's wall clock against reference sources, organized in **strata** (stratum 0 = atomic clock/GPS, stratum 1 directly attached, and so on). Typical internet accuracy is a few to tens of milliseconds. Every machine's clock is driven by an imperfect quartz crystal that **drifts** (tens of ppm — can accumulate tens of ms per day uncorrected), and **skew** is the difference between two machines' clocks at any instant. Skew grows between syncs and shrinks (rarely to zero) right after one.

<br>

**Why you can't order distributed events by wall clock.**
Machine B's clock might read an *earlier* time than machine A's even though B's event genuinely happened *later*:

```
A's event (true time T)         B's event (true time T+5ms)
        |                                |
A clock: 10:00:00.000            B clock: 09:59:59.998  (B runs 7ms behind)
                                          ^ looks earlier, but happened later -> WRONG ordering
```

This motivates **logical clocks** (Lamport timestamps, capturing "happened-before" with counters), **vector clocks** (causal relationships across nodes), and hybrids like HLC — all developed later.

<br>

**TrueTime's answer: expose the uncertainty instead of hiding it.**
Rather than a single (unreliable) timestamp, return an interval `[earliest, latest]` guaranteed to bracket the true time, then *wait out* the uncertainty before committing:

```
now() returns an interval, not a point:

  [earliest -------- true now -------- latest]
  |<------ uncertainty (kept < ~1 ms) ------>|

commit at 'latest', then WAIT until 'earliest' passes it
-> any later transaction is guaranteed to see a strictly greater timestamp
```

## In the real world

**Google Spanner — TrueTime.** Instead of a single unreliable timestamp, Spanner's TrueTime API returns an explicit uncertainty interval `[earliest, latest]` guaranteed to contain the true current time, backed by GPS and atomic clocks in every datacenter — Google reports 99th-percentile clock uncertainty under 1 ms. On commit, Spanner stamps a transaction at the interval's upper bound and then performs **commit wait** — holding until TrueTime's `earliest` has passed that stamp — which guarantees global ordering of causally related transactions purely from bounded clock uncertainty, no extra coordination rounds. It's the production evolution of the "wall clocks can't order distributed events" problem, and the ancestor of Hybrid Logical Clocks used elsewhere.

See [f-computing-fundamentals-cases-and-sources.md](../../../research/backend/F/f-computing-fundamentals-cases-and-sources.md#ss12-clocks).

## Trade-offs

| | Wall clock | Monotonic clock |
|---|---|---|
| Answers "what time is it?" | ✅ yes (calendar/UTC) | ❌ meaningless absolute value |
| Measure elapsed duration | ❌ can jump, even go negative | ✅ only ever forward |
| Comparable across machines | ❌ skew makes it unsafe | ❌ not comparable at all |
| Adjusted by NTP / manual / leap second | yes | no |

✅ Store event times as wall-clock UTC. ✅ Time operations with the monotonic clock. ❌ Never order events on different machines by their wall-clock timestamps.

> [!IMPORTANT] Remember
> Two clocks, two jobs: wall clock for *when*, monotonic clock for *how long*. And across machines, clock skew means physical timestamps can't be trusted to say which event came first.

## Check yourself

1. A timeout is implemented with the wall clock. What happens the moment an NTP correction pushes the clock backward mid-wait — and which clock should it have used?
2. TrueTime returns an interval instead of a single timestamp, then does "commit wait." How does waiting out the uncertainty guarantee correct global ordering without extra network coordination?

---

→ Next: [L0. System-Design Foundations](../../L0/)
↩ Comes back in: L5 (logical/vector clocks and causal ordering), L4/L5 (Hybrid Logical Clocks and TrueTime for globally-ordered transactions), L7 (timeouts in retries and circuit breakers, always on a monotonic clock)
