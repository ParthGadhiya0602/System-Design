# Little's Law

A single, assumption-light equation that ties together how many requests are in flight, how fast they arrive, and how long each one takes.

## Contents
- [Overview and statement](#overview-and-statement)
- [Intuition for why it is true](#intuition-for-why-it-is-true)
- [The three forms and rearrangements](#the-three-forms-and-rearrangements)
- [Worked examples](#worked-examples)
- [Practical uses in system design](#practical-uses-in-system-design)
- [Caveats](#caveats)
- [How this connects onward](#how-this-connects-onward)

## Overview and statement

Little's Law is one of the most useful results you will ever apply in system
design, and it is short enough to memorize:

```
L = lambda * W
```

Three quantities, each an average measured over a long window:

- **L** = the average number of items *in the system* at any moment. In a web
  service this is the number of requests being worked on right now, also called
  **concurrency** or the number of **in-flight** requests. "In the system" means
  anything that has arrived but not yet left: waiting in a queue plus actively
  being processed.
- **lambda** (the Greek letter, pronounced "lambda") = the average **arrival
  rate**, i.e. how many items enter the system per unit time. In steady state
  this equals the **throughput** (departure rate), because what comes in must go
  out. Measured in items per second (for example, requests per second, req/s).
- **W** = the average **time in system** for one item, from the moment it
  arrives to the moment it leaves. For a request this is its end-to-end
  **latency** or response time (queue wait + service time). Measured in seconds.

"The system" can be whatever box you draw around it: a single server, a thread
pool, a database connection pool, a message queue, or an entire distributed
service. Little's Law holds for each, as long as you are consistent about what
is inside the box.

What makes the law remarkable is how few assumptions it needs. It does **not**
care about the distribution of arrivals (bursty or smooth), the distribution of
service times (fast or slow, uniform or wildly variable), the scheduling order
(FIFO, LIFO, priority), or the number of servers inside the box. The only real
requirement is that the system is **stable** over the window you measure: it is
not growing without bound, so on average things leave as fast as they arrive.
Given that, `L = lambda * W` is exact, not an approximation.

Because it relates three things you constantly care about (how much is in
flight, how fast work arrives, how long work takes), it lets you solve for any
one of them when you know the other two.

## Intuition for why it is true

You do not need heavy math to believe it. Here is a steady-state argument.

Imagine a system that has been running long enough to settle down, so the
average number inside is not trending up or down. That means, over a long
window, the number of items that arrived is essentially equal to the number
that departed: **in = out**. This is what "stable" means.

Now think about how the "stuff inside" accumulates. On average, items enter at
rate `lambda`. Each item that enters stays for an average time `W` before it
leaves. So at any snapshot, the items currently inside are exactly those that
arrived within roughly the last `W` seconds and have not yet left. How many
items arrive in a window of length `W`? About `lambda * W` of them. Therefore
the average number inside is `lambda * W`. That is the law.

A concrete picture: suppose a coffee shop serves people arriving at 1 per
minute (`lambda = 1/min`), and each person spends 5 minutes inside from door to
exit (`W = 5 min`). At any given moment, how many people are inside? The people
present are those who arrived in the last 5 minutes and have not yet left, which
is `1/min * 5 min = 5 people`. So `L = 5`. Double the arrival rate or double how
long each person stays, and the crowd inside doubles. That direct
proportionality is the heart of the law.

A more formal way to see it: plot cumulative arrivals and cumulative departures
over time. The vertical gap between the two curves at any instant is the number
in the system; the horizontal gap is the time each item spends inside. The area
between the curves can be measured either as "average count times elapsed time"
or as "number of items times average time each spent." Setting those two views
of the same area equal and dividing by elapsed time yields `L = lambda * W`.

## The three forms and rearrangements

Because it is one equation with three variables, algebra gives you three
questions it can answer. Know all two-known-solve-for-the-third:

```
L = lambda * W      given rate and latency, find concurrency (how many in flight)
W = L / lambda      given concurrency and rate, find average latency
lambda = L / W      given concurrency and latency, find sustainable throughput
```

- **`L = lambda * W`** answers *"How many requests will be in flight?"* Use it
  to size pools, buffers, and memory: if you know your traffic (`lambda`) and
  your response time (`W`), this tells you how many concurrent slots you must
  provide.
- **`W = L / lambda`** answers *"What latency results from a given concurrency
  cap and load?"* If your system can only hold `L` items at once (a hard pool
  limit) and work arrives at `lambda`, the average time each item spends inside
  is `L / lambda`. This is how a capped pool turns into queueing delay.
- **`lambda = L / W`** answers *"What is the maximum throughput I can sustain?"*
  If a resource can hold at most `L` in-flight items and each takes `W` to
  process, the ceiling on throughput is `L / W`. Push arrivals past that and the
  system stops being stable: the queue grows without bound.

That last form is especially powerful: it says throughput is bounded by
concurrency divided by latency. If you cannot add concurrency, the only way to
raise the throughput ceiling is to reduce latency.

## Worked examples

Keep units consistent throughout: seconds and per-second, or milliseconds and
per-millisecond, never mixed.

### (a) Concurrency: size a thread or connection pool

A service receives `lambda = 2,000` requests per second. Each request takes, on
average, `W = 0.05 s` (50 ms) to complete end to end.

```
L = lambda * W
L = 2,000 req/s * 0.05 s
L = 100 requests in flight on average
```

So on average about **100 requests are being handled simultaneously**. If each
in-flight request needs its own worker thread (a blocking, thread-per-request
model), you need a thread pool of at least ~100 to keep up. If each also holds a
database connection for its full duration, you need a connection pool sized in
the same ballpark. Provision below `L` and requests will queue or be rejected;
provision at `L` you are at the edge, so add headroom for variance and bursts
(the average hides peaks).

### (b) Capacity: how many servers

Suppose one server can safely hold `L = 100` in-flight requests (limited by its
thread pool, memory, or connection count). You need to support `500` concurrent
in-flight requests across the fleet.

```
servers = required concurrency / per-server concurrency
servers = 500 / 100
servers = ~5 servers
```

You need about **5 servers** just to hold the concurrency, before adding
redundancy for failures and headroom for spikes. Notice this reasoning is pure
Little's Law applied twice: you first convert traffic and latency into a
required `L`, then divide by what one box can hold.

### (c) Invert: find latency or a throughput ceiling given a cap

**Find latency from a concurrency cap.** Your connection pool is hard-capped at
`L = 50`. Traffic is `lambda = 1,000` req/s. What average time will each request
spend in the system (including waiting for a free connection)?

```
W = L / lambda
W = 50 / 1,000 req/s
W = 0.05 s = 50 ms
```

If the actual processing work were faster than 50 ms, the difference shows up as
**queue wait** for a connection. The cap forces the average time to 50 ms; you
cannot go faster on average while holding 50 slots at 1,000 req/s.

**Find the throughput ceiling.** A downstream dependency lets you hold at most
`L = 200` concurrent calls, and each call takes `W = 0.1 s` (100 ms).

```
lambda_max = L / W
lambda_max = 200 / 0.1 s
lambda_max = 2,000 req/s
```

You can push at most **~2,000 req/s** through that dependency. Beyond that,
either latency `W` must drop or the concurrency limit `L` must rise; otherwise
requests pile up and the system becomes unstable.

## Practical uses in system design

Little's Law shows up constantly once you start looking for it:

- **Sizing thread and connection pools.** Compute `L = lambda * W` for expected
  peak traffic, then set pool sizes at or above it with headroom. This prevents
  both under-provisioning (queueing, timeouts) and gross over-provisioning
  (wasted memory, context-switching overhead).
- **Sizing queues and buffers.** The average number of messages waiting is
  `lambda * W_queue`, where `W_queue` is average wait time. This tells you how
  much buffer memory to reserve and how deep a queue can get before it signals
  trouble.
- **Estimating server counts.** Convert traffic plus latency into required
  concurrency, divide by per-server capacity, add redundancy. A clean
  back-of-envelope for capacity.
- **Reasoning about memory and queue pressure.** If in-flight count `L` is too
  high (too many objects live in memory, queues too deep), the law says you have
  exactly two levers: **reduce `lambda`** (shed or throttle load) or **reduce
  `W`** (make each request faster). There is no third option. This is a sharp,
  first-principles insight: you cannot wish away concurrency without touching
  arrival rate or latency.
- **Relating throughput and latency.** The previous topic separated throughput
  (`lambda`) from latency (`W`). Little's Law is the bridge between them: for a
  fixed concurrency `L`, throughput and latency trade off directly
  (`lambda = L / W`). If latency doubles and you cannot add concurrency,
  sustainable throughput halves. This is why latency spikes so often cascade
  into throughput collapse: slow responses hold slots longer, in-flight count
  rises, the pool fills, and new arrivals get rejected.

## Caveats

The law is exact, but only for what it actually claims. Common misreadings:

- **It is about long-run averages in a stable system.** All three quantities are
  averages over a measurement window where arrivals roughly equal departures. If
  the system is growing (arrivals persistently exceed departures) it is not
  stable and `L` is not settling to a value; the law describes the steady state,
  not an overloaded one.
- **It says nothing about the tail.** `W` is the *average* time in system.
  Little's Law will not tell you your p99 latency or worst-case wait. Two systems
  with identical `L`, `lambda`, and `W` can have very different tail behavior.
  Size pools with the average, but protect against the tail separately (timeouts,
  headroom, backpressure).
- **It does not describe transients or bursts.** During a sudden spike, `L` can
  shoot far above `lambda * W` momentarily before the system settles (or before
  it sheds load). The law is the equilibrium, not the instantaneous trajectory.
  Always provision headroom above the average to absorb bursts.
- **Units must be consistent.** `lambda` in req/s times `W` in seconds gives a
  dimensionless count `L`. Mixing req/s with `W` in milliseconds gives an answer
  off by 1000. Always reconcile units first.
- **Define the box carefully.** `L`, `lambda`, and `W` must all refer to the same
  system boundary. If `W` is end-to-end latency, `L` is total in-flight
  including queue wait; if `W` is only service time, `L` excludes those waiting.
  Be explicit about where the box starts and ends.

## How this connects onward

Little's Law is a foundation you will lean on repeatedly:

- **Capacity planning and back-of-envelope estimation** use it to turn traffic
  and latency targets into pool sizes, server counts, and memory budgets.
- **Queueing theory and backpressure** build on it: it is the entry point to
  understanding how utilization drives wait time, why queues explode near full
  utilization, and why systems shed load rather than let `L` grow unbounded.
- **Thread pools, connection pools, and bounded concurrency** are sized and
  reasoned about with `L = lambda * W` at their core.
- **Autoscaling** effectively adjusts capacity so that per-node `L` stays within
  safe limits as `lambda` changes; the law is the math behind the scaling
  signal.

Carry away the one-line version: **concurrency equals arrival rate times time in
system.** Whenever a system feels overloaded, ask which of the three the numbers
are telling you to change.
