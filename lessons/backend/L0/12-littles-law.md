# Little's Law

*One tiny equation ties together how much is in flight, how fast work arrives, and how long each piece takes -- and it holds for almost any system.*

`⏱️ ~5 min · 12 of 13 · System-Design Foundations`

> [!TIP] The gist
> **`L = lambda * W`**: the number of requests **in flight** (`L`) equals the **arrival rate** (`lambda`, throughput) times the **time each spends in the system** (`W`, latency). It is *distribution-free* -- it does not care about bursts, service-time spread, or scheduling order, only that the system is stable. That makes it the go-to tool for sizing thread pools, connection pools, buffers, and servers straight from throughput and latency.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture a busy coffee shop.

People walk in at some steady rate -- say **1 per minute**. Each person spends **5 minutes** inside, from door to exit (order, wait, grab drink, leave).

At any random moment, how many people are inside? The ones present are exactly those who arrived in the last 5 minutes and have not left yet -- about `1/min * 5 min = 5 people`.

That is the whole law. The crowd inside equals **how fast people arrive** times **how long each one stays**. Double the arrival rate *or* double how long each person lingers, and the crowd doubles. Nothing about the coffee shop's layout, the order people are served, or whether they clump at the door changes that relationship.

Swap "people" for "requests," "how long they stay" for "latency," and "crowd inside" for "concurrency," and you have Little's Law for a server.

## The concept

Little's Law states, for any stable system:

```
L = lambda * W
```

Three quantities, each an **average** over a measurement window:

- **`L`** -- the average number of items **in the system** at once: everything that has arrived but not yet left (waiting *plus* being processed). For a service this is **concurrency**, the number of **in-flight** requests. Dimensionless (a count).
- **`lambda`** -- the average **arrival rate**, items entering per unit time. In steady state this equals **throughput** (what comes in must go out). Units: items per second (for example, req/s).
- **`W`** -- the average **time in system** for one item, arrival to departure. For a request this is end-to-end **latency** (queue wait + service time). Units: seconds.

**What makes it remarkable:** it needs almost no assumptions. It does *not* care whether arrivals are bursty or smooth, whether service times are uniform or wildly variable, the scheduling order (FIFO, priority, LIFO), or how many servers sit inside the box. The one requirement is that the system is **stable** over the window -- not growing without bound, so on average things leave as fast as they arrive. Given that, the equation is **exact**, not an approximation.

**"The system" is whatever box you draw:** a single server, a thread pool, a connection pool, a queue, or a whole service -- as long as `L`, `lambda`, and `W` all describe the *same* box.

Because it is one equation with three variables, algebra gives you **three questions** it can answer:

```
L = lambda * W      rate + latency   -> concurrency (how many in flight)
W = L / lambda      concurrency + rate -> average latency
lambda = L / W      concurrency + latency -> sustainable throughput
```

That last form is sharp: **throughput is capped by concurrency divided by latency.** If you cannot add concurrency, the only way to raise the throughput ceiling is to cut latency.

## How it works

Think of it as arrivals flowing through a box that always holds about `L` items:

```
  arrivals            [ system holding L ]            departures
  lambda/s   ------>   in flight = lambda * W   ------>   lambda/s
```

The single rule for the rest of this section: **keep units consistent** -- seconds with per-second, or ms with per-ms, never mixed.

### (a) Concurrency: size a thread or connection pool

A service receives `lambda = 2,000` req/s. Each request takes `W = 0.05 s` (50 ms) end to end.

```
L = lambda * W
L = 2,000 req/s * 0.05 s
L = 100 requests in flight (on average)
```

So about **100 requests are being handled simultaneously**. In a thread-per-request model you need a pool of **~100 threads** to keep up; if each request also holds a DB connection for its full duration, size the connection pool in the same ballpark. Provision below `L` and requests queue or get rejected -- so add headroom for variance and bursts (the average hides peaks).

### (b) Capacity: how many servers

One server safely holds `L = 100` in-flight requests (its thread/memory/connection limit). You need **500** concurrent in-flight requests across the fleet.

```
servers = required concurrency / per-server concurrency
servers = 500 / 100
servers = ~5 servers
```

You need about **5 servers** just to hold the concurrency -- before adding redundancy for failures and headroom for spikes. This is Little's Law applied twice: turn traffic + latency into required `L`, then divide by what one box holds.

### (c) Invert: find the throughput ceiling

A downstream dependency lets you hold at most `L = 200` concurrent calls, and each call takes `W = 0.1 s` (100 ms). What is the most you can push through?

```
lambda_max = L / W
lambda_max = 200 / 0.1 s
lambda_max = 2,000 req/s
```

At most **~2,000 req/s** fits through that dependency. Beyond it, either `W` must drop or the concurrency limit `L` must rise -- otherwise requests pile up and the system stops being stable.

### The key insight

If in-flight count `L` is too high (memory bloated, queues too deep), the law says you have **exactly two levers**: reduce `lambda` (shed or throttle load) or reduce `W` (make each request faster). There is no third option. This is why latency spikes cascade into throughput collapse: slow responses hold slots longer, `L` climbs, the pool fills, and new arrivals get rejected.

## Trade-offs

Little's Law is exact -- but only for what it actually claims:

| Caveat | What it means |
|---|---|
| **Long-run averages, stable system** | All three are averages over a window where arrivals roughly equal departures. It describes steady state, not an overloaded, still-growing system. |
| **Says nothing about the tail** | `W` is the *average* time. It will not give you p99 or worst-case wait; protect the tail separately (timeouts, backpressure). |
| **Not transients or bursts** | During a spike, `L` can momentarily shoot far above `lambda * W`. Provision headroom above the average to absorb bursts. |
| **Units must be consistent** | req/s times seconds gives a clean count; mixing req/s with ms is off by 1000. Reconcile units first. |
| **Define the box** | `L`, `lambda`, `W` must all refer to the same boundary. End-to-end `W` pairs with total in-flight `L` (including queue wait); service-time `W` pairs with just the ones being processed. |

## Remember

> [!IMPORTANT] Remember
> **`L = lambda * W`** -- concurrency equals arrival rate times time in system. From just **throughput** and **latency** you can size concurrency, pools, buffers, and server counts. When a system feels overloaded, the law tells you there are only two levers on in-flight count: cut `lambda` or cut `W`.

## Check yourself

1. A service handles `lambda = 4,000` req/s with average latency `W = 25 ms`. How many requests are in flight on average, and roughly how big should a thread-per-request pool be?
2. Your connection pool is hard-capped at `L = 50` and traffic is `lambda = 1,000` req/s. What average time will each request spend in the system, and where does that time go if the actual processing is faster than that?

---

→ Next: [Universal Scalability Law](13-universal-scalability-law.md)
↩ Comes back in: capacity planning, queueing/backpressure, thread pools, autoscaling
