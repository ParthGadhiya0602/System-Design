# Availability, Reliability, Scalability, Maintainability

*A shop can be open all day and still hand you the wrong change -- "up" and "correct" are not the same promise, and neither is "handles the lunch rush."*

`⏱️ ~7 min · 7 of 13 · System-Design Foundations`

> [!TIP] The gist
> Four non-functional "-ilities" describe **how well** a system behaves, not what it does -- and they are constantly confused. **Availability** = is it responding at all? **Reliability** = is the response correct? **Scalability** = does it still work at 10x load? **Maintainability** = can humans keep changing it cheaply? Two confusions to kill on sight: available is not the same as reliable (a system can answer every request with stale data), and scalable is not the same as fast (a quick system that collapses at 2x traffic is performant, not scalable).

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture a corner shop, and the four "-ilities" become four different questions about it:

- **Availability** -- is the shop *open* when you show up? A shop closed two days a month has an availability problem.
- **Reliability** -- when it is open, does it give you the *right change* and the item you actually asked for? An always-open shop that keeps shortchanging you is available but unreliable.
- **Scalability** -- can it handle the *lunch rush* by adding a second till and a second clerk, instead of the line grinding to a halt?
- **Maintainability** -- can the owner *renovate the back room* or swap the fridge without shutting the whole place down for a month?

Same shop, four independent judgments. A system can ace one and fail another, which is exactly why we name them separately.

## The concept

**Definition -- the four NFRs.** A *functional* requirement says what the system must do ("transfer money"). A *non-functional* requirement (NFR) says how well. These four are the ones every design discussion returns to:

- **Availability** -- the fraction of time the system is up and answering requests. `Availability = uptime / (uptime + downtime)`. "Up" means reachable *and* returning valid responses in acceptable time -- a server that accepts connections but returns HTTP 500 to everything is not available in any useful sense.
- **Reliability** -- the probability the system performs its function *correctly* over a period. The key word is *correctly*: reliability asks "was the answer right and did the operation actually complete?", not just "did you answer?"
- **Scalability** -- the ability to handle growth (more load, data, users) by adding resources, ideally without a redesign. Working test: *meets 10x the load with roughly 10x the resources and no architectural rewrite.*
- **Maintainability** -- how easily the system can be operated, understood, and changed over its lifetime. Over a system's life this is usually the *dominant* cost.

**The two classic confusions:**

1. **Available vs reliable.** Available = it responds. Reliable = it responds *correctly*. A system can be available but unreliable (always answers, sometimes with corrupt or stale data) or reliable but unavailable (always correct when it answers, but down a lot). You want both; they are measured differently.
2. **Scalable vs fast.** *Performance* is how fast one request is served at a given load. *Scalability* is how performance holds up as load grows. Fast-but-fragile is performant, not scalable.

**Key terms to lock in:**

- **SPOF (single point of failure)** -- a component with only one instance whose failure kills the whole system (a lone database, one load balancer). Redundancy exists to remove SPOFs.
- **MTBF (mean time between failures)** -- average time running correctly before a failure. Higher is better.
- **MTTR (mean time to recover)** -- average time to detect and restore service after a failure. Lower is better.
- **Durability** -- once a write is acknowledged, it survives crashes and is not lost. Reliability applied to stored data.
- **Fault tolerance** -- the system keeps operating correctly (perhaps degraded) despite some components failing. A *means* to availability and reliability, usually via redundancy.

## How it works

### Availability: the "nines"

Availability targets are quoted as a "number of nines." Each extra nine cuts allowed downtime by **10x** -- and the cost of achieving it rises just as steeply. These figures are exact (percentage times the length of the period):

| Availability | Name        | Downtime per year | Downtime per month |
|--------------|-------------|-------------------|--------------------|
| 90%          | one nine    | 36.5 days         | 72 hours (3 days)  |
| 99%          | two nines   | 3.65 days         | 7.2 hours          |
| 99.9%        | three nines | 8.76 hours        | 43.2 minutes       |
| 99.99%       | four nines  | 52.6 minutes      | 4.32 minutes       |
| 99.999%      | five nines  | 5.26 minutes      | 25.9 seconds       |

Three nines (~43 min/month) is reachable with careful engineering. Five nines (~26 sec/month) leaves no time for a human to even *notice* an incident, so it demands fully automated failover and heavy redundancy. Most consumer web services target three to four nines; only critical infrastructure justifies five.

Availability is achieved with **redundancy** (more than one instance of every component), **failover** (auto-shift traffic to a healthy instance), and **health checks** (probe instances so traffic routes around sick ones) -- all in service of eliminating SPOFs.

### Worked example 1: availability multiplies in series

If a request must pass through several components in sequence and **all** must be up, the overall availability is the *product* of the individual ones:

```
A_total = A_1 x A_2 x ... x A_n
```

Five independent components, each at three nines (0.999):

```
0.999 x 0.999 x 0.999 x 0.999 x 0.999  =  0.999^5  ~=  0.995  =  99.5%
```

So five three-nines components in a chain yield only about **two-and-a-half nines** end to end -- roughly **3.6 hours** of downtime per month instead of 43 minutes. This is why deep call chains are dangerous, and why redundancy is applied *per component*: putting two instances of a component in parallel raises that component's availability, which lifts the whole product back up.

```
Two instances each at 99% in parallel:  1 - (0.01 x 0.01)  =  1 - 0.0001  =  99.99%
```

### Reliability: MTBF, MTTR, and fast recovery

A service can have excellent uptime yet be unreliable if it returns stale data, silently drops writes (returns success but loses the record), or double-charges a payment on retry. It looks "up" on a dashboard but is doing the wrong thing.

MTBF and MTTR connect straight to availability:

```
Availability  ~=  MTBF / (MTBF + MTTR)
```

### Worked example 2: recovery beats never-failing

Two systems, same window:

- **System A** -- fails rarely (MTBF ~1000 h) but each outage takes 4 h to fix (MTTR ~4 h): `1000 / (1000 + 4) ~= 99.6%`.
- **System B** -- fails more often (MTBF ~200 h) but recovers in 2 min (MTTR ~0.033 h): `200 / (200 + 0.033) ~= 99.98%`.

B fails 5x as often yet is far more available, because it *heals fast*. The lesson: you can raise availability by making failures rarer (bigger MTBF) **or** shorter (smaller MTTR) -- and lowering MTTR (fast rollback, automated failover, good alerting) is usually cheaper and more impactful. **Design for fast recovery, not just for no failures.**

### Scalability: linear vs sub-linear

Ideal *linear* scaling means doubling resources doubles capacity. Reality is often worse because overhead grows with size: **coordination** (nodes chattering for consensus or cache invalidation), **contention** (workers fighting over a lock or hot row), and **hot shards / skew** (adding capacity doesn't help the one overloaded shard). When these dominate you get sub-linear scaling -- 2x resources yield only ~1.6x capacity. Good scalable design keeps the coordinated/serialized fraction of work as small as possible. (Vertical vs horizontal scaling gets its own lesson next level down.)

### Maintainability: three facets

Most engineering money is spent *after* v1 ships. Maintainability (per Kleppmann) has three facets:

- **Operability** -- can operators keep it running without heroics? (visibility, easy deploy/rollback, runbooks)
- **Simplicity** -- can a newcomer form a correct mental model quickly? (remove accidental complexity via good abstractions)
- **Evolvability** -- can we adapt to new requirements cheaply and safely? (loose coupling, tests, modular boundaries)

## Trade-offs

No design maximizes all four -- heavy redundancy for availability adds complexity that can hurt maintainability; aggressive scale-out adds coordination that can hurt reliability. The skill is spotting which NFR the prompt stresses and letting it drive the design.

| Domain / signal | Dominant NFR | What you invest in | What you trade away |
|---|---|---|---|
| Payments, banking, healthcare, core infra (DNS/auth) | Reliability + availability | Durability, correctness guarantees, redundancy, fast recovery | Cost, some latency, simplicity |
| Viral / fast-growing consumer (feeds, live-streaming, hot launches) | Scalability | Scale-out architecture, elasticity | Strong consistency (brief staleness OK) |
| Long-lived platforms many teams build on for years | Maintainability | Clean boundaries, tests, observability | Some raw performance / short-term speed |

Every successful system eventually becomes the third row. In an interview, listen for the signal ("must never lose a payment" -> reliability/durability; "a celebrity post goes viral" -> scalability; "many teams build on it for years" -> maintainability), then state which NFR you optimize and what you trade.

## Remember

> [!IMPORTANT] Remember
> **Availability is not reliability** -- one asks "did you answer?", the other "was the answer right?" And **each extra nine is ~10x harder and costlier** than the last, while availability *multiplies down* across a chain of dependencies. So push failures to be short (low MTTR), add redundancy per component to kill SPOFs, and let the domain decide which "-ility" wins.

## Check yourself

1. A service returns HTTP 200 for every request, but 5% of responses contain stale data. Is its problem availability or reliability -- and which metric would you look at?
2. Five microservices sit in a request chain, each at 99.95% availability. Roughly what is the end-to-end availability, and name one way to raise it.

---

→ Next: [SLA / SLO / SLI](08-sla-slo-sli.md)
↩ Comes back in: reliability/SRE, redundancy & failover, sharding, observability
