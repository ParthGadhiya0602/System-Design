# Availability, Reliability, Scalability, Maintainability

The four core non-functional "-ilities" that describe HOW WELL a system behaves (as opposed to WHAT it does) - defined precisely, because interviewers penalize using them interchangeably.

## Contents

- [Overview](#overview)
- [Availability](#availability)
- [Reliability](#reliability)
- [Scalability](#scalability)
- [Maintainability](#maintainability)
- [When Each Dominates](#when-each-dominates)
- [Check Yourself](#check-yourself)

## Overview

A functional requirement says what the system must do ("transfer money", "return search results"). A non-functional requirement (NFR) says how well it must do it. The four NFRs below are the ones every system-design discussion returns to. They sound similar in casual speech but mean distinct things, and conflating them leads to wrong design choices.

Crisp one-line definitions:

- **Availability** - the fraction of time the system is up and answering requests. It is about "is it responding at all?"
- **Reliability** - the probability the system performs CORRECTLY (does the right thing) over a period of time. It is about "is the answer right?"
- **Scalability** - the ability to handle growth (more load, data, users) by adding resources, ideally without a redesign. It is about "does it still work when 10x bigger?"
- **Maintainability** - how easily the system can be operated, understood, and changed over its lifetime. It is about "can humans keep improving it cheaply?"

Two common confusions to fix immediately:

1. **Availability vs reliability.** A system can be available but unreliable (it always responds, but sometimes with corrupt or stale data). It can be reliable but unavailable (when it does respond it is always correct, but it is down a lot). You want both; they are measured differently.
2. **Scalability vs performance.** Performance is how fast one request is served at a given load. Scalability is how performance holds up as load grows. A fast system that collapses at 2x traffic is performant but not scalable.

The rest of this note defines each term from first principles, then shows which one should drive the design depending on the prompt.

## Availability

**Definition.** Availability is the proportion of time a system is operational and able to serve correct responses to requests. It is usually expressed as a percentage over a window (a month or a year):

```
Availability = uptime / (uptime + downtime)
```

"Up" means reachable AND returning valid responses within acceptable latency - a server that accepts connections but returns HTTP 500 to everything is not available in any useful sense.

**The "nines".** Availability targets are quoted as "number of nines". The downtime figures below are standard and exact (computed from the percentage times the length of the period), so memorize the shape of the table.

| Availability | Name        | Downtime per year | Downtime per month | Downtime per day |
|--------------|-------------|-------------------|--------------------|------------------|
| 90%          | one nine    | 36.5 days         | 72 hours (3 days)  | 2.4 hours        |
| 99%          | two nines   | 3.65 days         | 7.2 hours          | 14.4 minutes     |
| 99.9%        | three nines | 8.76 hours        | 43.2 minutes       | 1.44 minutes     |
| 99.99%       | four nines  | 52.6 minutes      | 4.32 minutes       | 8.64 seconds     |
| 99.999%      | five nines  | 5.26 minutes      | 25.9 seconds       | 864 milliseconds |

Read the jump carefully: each extra nine cuts allowed downtime by 10x, and the cost of achieving it rises steeply. Three nines (about 43 minutes/month) is achievable with careful engineering; five nines (about 26 seconds/month) leaves no room for a human to even notice an incident, so it requires fully automated failover and heavy redundancy. Most consumer web services target three to four nines; only critical infrastructure justifies five.

**How availability is achieved.**

- **Redundancy** - run more than one instance of every component so the loss of one does not take the system down. This is the foundation of all the others.
- **Failover** - automatically shift traffic from a failed instance to a healthy one, ideally in seconds, without human action.
- **Health checks** - continuously probe instances so the system knows which are healthy and can route around the sick ones.
- **Eliminate single points of failure (SPOF)** - any component that has only one instance and whose failure kills the whole system. A lone database, a single load balancer, one network link: each is a SPOF. Redundancy exists to remove SPOFs.

**Availability multiplies across dependencies in series.** If a request must pass through several components in sequence and ALL of them must be up for the request to succeed, the overall availability is the product of the individual availabilities:

```
A_total = A_1 x A_2 x ... x A_n
```

Example: 5 independent components each at 99.9% (0.999) give:

```
0.999^5 ~= 0.995 = 99.5%
```

So five three-nines components in a chain yield only about two-and-a-half nines end to end - roughly 3.6 hours of downtime per month instead of 43 minutes. This is why deep call chains are dangerous and why redundancy is applied per component: putting two instances of a component in parallel raises that component's availability (a parallel pair each at 99% gives 1 - 0.01^2 = 99.99%), which then lifts the series product back up.

## Reliability

**Definition.** Reliability is the probability that a system performs its intended function CORRECTLY for a specified period under stated conditions. The key word is "correctly": a reliable system does the right thing, not merely responds. Availability asks "did you answer?"; reliability asks "was the answer right, and did the operation actually complete as intended?"

**Available but unreliable.** Concretely, a service can have excellent uptime yet be unreliable if it:

- returns stale or corrupted data,
- silently drops writes (returns success but loses the record),
- double-charges a payment on retry,
- returns a result computed from a partially failed backend.

In all these cases the system is "up" (availability looks good on a dashboard) but not doing the right thing (reliability is poor). This is why the two are measured and engineered separately.

**MTBF and MTTR.** Reliability engineering uses two failure-timing metrics:

- **MTBF (Mean Time Between Failures)** - average time the system runs correctly before a failure. Higher is better (failures are rarer).
- **MTTR (Mean Time To Repair/Recover)** - average time to detect and restore service after a failure. Lower is better (failures are shorter).

They connect to availability:

```
Availability ~= MTBF / (MTBF + MTTR)
```

The important insight: you can raise availability EITHER by making failures rarer (bigger MTBF) OR by making recovery faster (smaller MTTR). In practice lowering MTTR is often cheaper and more impactful than chasing a bigger MTBF - failures are inevitable, so systems that detect and heal in seconds (fast rollback, automated failover, good alerting) end up more available than systems that try to never fail. "Design for fast recovery, not just for no failures."

**Related concepts.**

- **Durability** - once data is acknowledged as written, it survives failures (crashes, power loss, disk death) and is not lost. Durability is reliability applied to stored data specifically.
- **Fault tolerance** - the system continues operating correctly (perhaps degraded) despite the failure of some components. Fault tolerance is a means to reliability and availability, usually via redundancy plus the ability to isolate and route around faults.

## Scalability

**Definition.** Scalability is the ability of a system to handle growth - more requests per second, more data, more concurrent users - by adding resources, ideally with cost growing linearly or sub-linearly and WITHOUT a rewrite. A useful working definition for interviews: "the system scales if it can meet 10x the load with roughly 10x the resources and no architectural rewrite."

**Linear vs sub-linear scaling.** Ideal (linear) scaling means doubling resources doubles capacity. Reality is often worse than linear because of overhead that grows with system size:

- **Coordination overhead** - nodes must talk to each other (consensus, cache invalidation, lock management); this chatter grows as nodes are added.
- **Contention** - many workers fighting over a shared resource (a lock, a hot row, a single queue) serialize and stop benefiting from more hardware.
- **Hot shards / skew** - if load is unevenly distributed, adding capacity to the cluster does not help the one overloaded shard that is the actual bottleneck.

When these dominate you get sub-linear scaling: 2x resources yield only, say, ~1.6x capacity, and eventually adding machines stops helping (or even hurts). Good scalable design keeps the fraction of work that must be coordinated or serialized as small as possible.

**Vertical vs horizontal (preview).** Two ways to add resources:

- **Vertical (scale up)** - a bigger machine (more CPU, RAM). Simple, but has a hard ceiling and a single-machine SPOF.
- **Horizontal (scale out)** - more machines working together. Effectively unbounded, but requires the work to be distributable and introduces coordination cost.

These are covered in depth in a later foundations topic; here it is enough to know horizontal scaling is what makes near-linear growth possible past the limit of one machine.

## Maintainability

**Definition.** Maintainability is how easily a system can be operated, understood, and changed throughout its life. It is not glamorous, but over a system's lifetime it is usually the DOMINANT cost - most engineering money is spent evolving and operating software after the first version ships, not writing that first version. A design that is fast and scalable but that no one can safely change is a liability.

A widely used breakdown (from Kleppmann) splits maintainability into three facets:

1. **Operability** - make it easy for operations teams to keep the system running smoothly: good visibility into runtime behavior, sensible defaults, easy deployment and rollback, self-healing where possible, and clear runbooks. "Can operators do their job without heroics?"
2. **Simplicity** - manage complexity so new engineers can understand the system. Remove accidental complexity (complexity that comes from a poor implementation, not from the problem itself) via good abstractions, clear boundaries, and consistent patterns. "Can a newcomer form a correct mental model quickly?"
3. **Evolvability** (also called extensibility or modifiability) - make it easy to change the system as requirements shift: loose coupling, good test coverage, and modular boundaries so a change in one place does not ripple everywhere. "Can we adapt to new requirements cheaply and safely?"

Maintainability is largely a property of how the system is structured and instrumented, not of raw performance, which is why it is easy to ignore early and expensive to retrofit.

## When Each Dominates

No design maximizes all four at once - they trade against each other (heavy redundancy for availability adds cost and complexity that can hurt maintainability; aggressive horizontal scaling adds coordination that can hurt reliability). The skill is identifying which NFR the prompt actually stresses and letting it drive the design.

- **Availability and reliability dominate** for systems where being down or being wrong causes direct harm or loss: payments and banking (a wrong or lost transaction is unacceptable), healthcare (patient safety), and core infrastructure (DNS, auth, networking) that everything else depends on. Here you invest in redundancy, failover, durability, correctness guarantees, and fast recovery even at high cost.
- **Scalability dominates** for viral or fast-growing consumer apps where usage can jump by orders of magnitude in weeks: social feeds, live-streaming, trending e-commerce launches. Here the priority is an architecture that scales out gracefully, because being unable to grow means losing the whole opportunity - and a brief inconsistency is often tolerable.
- **Maintainability dominates** for long-lived systems that many teams will change over years - and, crucially, EVERY successful system eventually becomes this. A system that ships fast but ossifies will lose to one that keeps evolving cheaply.

In an interview, listen for the signal in the prompt ("must never lose a payment" -> reliability/durability; "handle a celebrity's post going viral" -> scalability; "a platform many teams build on for years" -> maintainability) and state explicitly which NFR you are optimizing and what you are trading away.

These four NFRs connect forward to the rest of the curriculum: the formal contracts that quantify availability and reliability (SLA / SLO / SLI) come next; the mechanisms that deliver availability (redundancy, failover, load balancing, health checks) and reliability (durability, fault tolerance, fast recovery) recur throughout the reliability and network levels; scalability is realized through the scaling, caching, and sharding levels; and maintainability is served by the observability and architecture-pattern levels. Keep the definitions here precise and the later material slots in cleanly.

## Check Yourself

1. A service returns HTTP 200 for every request but 5% of responses contain stale data. Is its problem availability or reliability? Which metric would you look at?
2. Five microservices sit in a request chain, each at 99.95% availability. What is the end-to-end availability, and what is one way to raise it?
3. Your system has a large MTBF but a 4-hour MTTR. A competitor has a smaller MTBF but a 2-minute MTTR. Who likely has higher availability, and why does that argue for investing in recovery?
4. You double your servers and throughput rises only ~1.5x. Name two mechanisms that could explain the sub-linear scaling.
5. For a hospital medication-dosing system versus a photo-sharing app that just went viral, which NFR leads each design, and what would you trade away?
