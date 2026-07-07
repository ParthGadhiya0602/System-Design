# SLA / SLO / SLI

Three nested terms that turn the fuzzy word "reliable" into something you can measure, target, and promise.

## Contents

- [Overview](#overview)
- [SLI (Service Level Indicator)](#sli-service-level-indicator)
- [SLO (Service Level Objective)](#slo-service-level-objective)
- [SLA (Service Level Agreement)](#sla-service-level-agreement)
- [Relationship and a Worked Example](#relationship-and-a-worked-example)
- [Error Budget](#error-budget)
- [How This Connects Onward](#how-this-connects-onward)
- [Check Yourself](#check-yourself)

## Overview

"The service should be reliable" is not an engineering statement - it cannot be measured, alerted on, or enforced. SLI, SLO, and SLA are three related terms that make reliability concrete, each one broader than the last:

- **SLI** - the narrowest: a single measured metric of how the service is actually behaving right now (a number).
- **SLO** - broader: an internal target for that metric that the team commits to hitting (a goal).
- **SLA** - broadest: an external, often contractual promise about that target, with consequences if it is missed (a promise with teeth).

Think of it as measurement -> goal -> promise. You cannot set a sensible goal without first deciding what to measure, and you cannot make a credible promise to a customer without first having a goal you consistently hit internally. Every later reliability decision - what to alert on, when to freeze deploys, how much redundancy to build - traces back to these three definitions, so getting them precise matters more than it first appears.

## SLI (Service Level Indicator)

**Definition.** An SLI is a quantitative metric that measures one aspect of the service's behavior over time, chosen specifically because it reflects what the USER experiences, not what is convenient for the server to report. It is a measurement, not a target - an SLI has no "good" or "bad" value by itself until an SLO is attached to it.

Good SLIs are almost always expressed as a ratio or percentile over a window, for example:

- **Availability / success rate** - fraction of requests that returned a valid (non-error) response: `good requests / total requests`.
- **Latency** - a percentile of response time, e.g. p99 latency, because averages hide the slow tail that a fraction of real users actually feel.
- **Combined correctness + speed** - the most user-centric SLIs combine both, e.g. "fraction of requests served in under 200 ms AND with a 2xx status code". A request that is fast but returns an error, or correct but too slow, does not count as "good".
- **Durability / freshness** - for storage or data-pipeline systems, e.g. fraction of writes not lost, or fraction of data that is no more than N minutes stale.

**Why "reflect the user's experience" matters.** A server-side metric like "CPU usage" or "average response time measured inside the process" can look fine while users are suffering (e.g. a slow database call that never reaches the CPU-bound code path, or a fast average hiding a painful p99). SLIs should be measured as close to the user as practical - ideally at the load balancer or client - so they capture what was actually delivered, not what the internals think they delivered.

## SLO (Service Level Objective)

**Definition.** An SLO is an internal target value (or range) for an SLI over a defined time window, that the owning team commits to meeting. It answers "how good does this SLI need to be for the service to be considered healthy?"

Example SLO: "99.9% of GET requests succeed with a 2xx status and complete in under 300 ms, measured over a rolling 30-day window."

Properties of a well-formed SLO:

- It names a specific SLI, a target percentage, and a time window (rolling 30 days is the most common choice because it smooths out weekday/weekend and monthly-billing-run effects).
- It is what the team **aims for and alerts on** - dashboards, paging alerts ("SLO burn rate too high"), and go/no-go decisions for risky changes are built around SLOs, not raw SLIs.
- It should be **stricter than the SLA** it backs (if there is one). The SLO is the internal bar; the SLA is the external promise. Keeping the SLO tighter builds in a safety margin so that normal operational noise does not cause an SLA breach - by the time you are at risk of missing the SLA, you have already had ample internal warning from SLO alerts.

Setting the SLO too loose (say, matching the SLA exactly) removes that warning margin: any bad week pushes you straight into breaching a customer-facing contract with no room to react. Setting it far too tight wastes engineering effort chasing reliability improvements no user can perceive and eats into velocity for no real benefit. The right SLO is set from user expectations and past performance, not from an arbitrary round number.

## SLA (Service Level Agreement)

**Definition.** An SLA is an external, often contractual promise made to customers (or between organizations) about a service level, paired with defined CONSEQUENCES if the promise is broken - service credits, refunds, penalty payments, or a right to terminate the contract. It is fundamentally a business and legal artifact, not just an engineering number, though it is built on top of engineering measurements (SLIs) and internal targets (SLOs).

Example SLA clause: "The service will be available 99.5% of the time per calendar month; if availability falls below this, the customer receives a service credit of 10% of that month's fees, scaling up to 30% credit below 99.0%."

Key characteristics:

- **Consequences attached.** This is what separates an SLA from an SLO - missing an SLO triggers an internal alert and a conversation; missing an SLA triggers a financial or contractual penalty.
- **Deliberately looser than the SLO.** Because breaching it costs real money or trust, an SLA is set with margin below what the team actually expects to deliver (see the worked example below), and it typically covers fewer, coarser-grained guarantees than the full set of internal SLOs (a customer contract might promise only overall availability, while internally the team tracks availability, three latency percentiles, and an error-rate SLO).
- **Negotiated, not purely engineered.** The number in an SLA is shaped by sales, legal, and competitive pressure as much as by what is technically achievable, which is exactly why it should never be treated as the team's actual operating target.

## Relationship and a Worked Example

Putting the three together as one sentence: **the SLI is what you measure, the SLO is the goal you set for that measurement, and the SLA is the promise you make to a customer about it - backed by penalties if you fail.**

Typical strictness ordering: **SLO is stricter than SLA.** The team aims higher internally than what it has promised externally, so that ordinary variance does not turn into a contractual breach.

Worked example - a cloud storage API:

| Term | Statement |
|------|-----------|
| SLI  | Fraction of `PUT`/`GET` requests that complete successfully (2xx) within 200 ms, measured at the load balancer. |
| SLO  | 99.95% of requests meet the SLI over a rolling 30-day window. (Internal target; drives paging alerts and deploy freezes.) |
| SLA  | 99.9% availability per calendar month, or the customer receives a 15% service credit; below 99.0%, a 30% credit. (External contract with sales/finance consequences.) |

Notice the SLA (99.9%) is deliberately looser than the SLO (99.95%). This is intentional, not sloppy: it gives the team roughly a 0.05-percentage-point cushion of normal operational variance to absorb before a bad month turns into a customer-facing, revenue-impacting breach. If the SLO and SLA were set to the same number, the team would be signing a contract with zero margin for error.

## Error Budget

**Definition.** The error budget is the allowed amount of "badness" implied by an SLO: `error budget = 1 - SLO`. It converts the abstract percentage into a concrete, spendable quantity of allowed failure over the SLO's time window.

Worked calculation: an SLO of 99.9% over a 30-day month allows an error budget of `1 - 0.999 = 0.001`, i.e. 0.1% of the window may fail:

```
30 days x 24 hours x 60 minutes = 43,200 minutes/month
0.1% of 43,200 minutes ~= 43.2 minutes/month
```

So a 99.9% SLO gives roughly **~43.8 minutes** of allowed failure per 30-day month (the exact figure depends on whether the SLI is availability, an error rate applied to request volume, or something else - the arithmetic above is the standard availability-style approximation).

**Why this reframes reliability as a resource.** Instead of treating every failure as purely bad and every nine of reliability as purely good, the error budget treats "unreliability" as a currency the team is allowed to spend:

- **Spending the budget on purpose.** Risky but valuable actions - a rapid feature rollout, a schema migration, an aggressive canary rollout, a new region launch - are allowed to consume error budget, because some risk is the acceptable cost of shipping and improving the system.
- **Freezing when it runs out.** If the budget for the current window is exhausted (too many incidents, too much latency degradation), the team stops taking on new risk - deploys slow down or freeze, and effort shifts to reliability work - until the budget resets with the rolling window or the next period begins.
- **A shared, objective decision rule.** This turns "should we ship this risky change now?" from a subjective argument into a data-driven check: is there budget left, yes or no. It also aligns incentives between engineers who want to ship features and those who want stability - both are working against the same number.

## How This Connects Onward

These three definitions are the vocabulary the rest of reliability engineering is built on. Choosing and measuring SLIs correctly depends on the observability practices covered later (metrics, dashboards, and alerting pipelines that can actually compute an SLI in real time). Deciding how strict an SLO can be, and how to protect it, connects forward into the reliability and resilience mechanisms - redundancy, failover, rate limiting, circuit breakers, retries, graceful degradation - that keep a system inside its target. Error-budget math also feeds directly into capacity planning and rollout strategy: how much headroom to provision, and how aggressively to roll out changes, is shaped by how much budget is currently available to spend.

## Check Yourself

1. A team reports "average response time is 80 ms" as its SLI. Why might this hide a real user-facing problem, and what would be a better-chosen SLI?
2. Why must an SLO be stricter than its corresponding SLA rather than equal to it?
3. What is the single feature that distinguishes an SLA from an SLO?
4. A service has a 99.95% SLO over a rolling 30-day month. Approximately how many minutes of allowed failure does that give it? (Show the arithmetic.)
5. Your team's error budget for the month is fully spent with two weeks still to go. What should happen next, and why is this a healthier response than ignoring the budget and continuing to ship as usual?
