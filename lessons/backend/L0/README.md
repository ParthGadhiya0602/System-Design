# L0. System-Design Foundations

*The frame for the whole track. Before any component, learn how to think: requirements drive the design, numbers size it, and every choice names its trade-off.*

Each lesson is a short, self-contained read (~5-8 min). Work through them in order, or jump to what you need.

| # | Lesson | In one line |
|---|--------|-------------|
| 01 | [What System Design Is](01-what-system-design-is.md) | The design mindset: derive from requirements, name every trade-off. |
| 02 | [Functional vs Non-Functional Requirements](02-functional-vs-non-functional-requirements.md) | What the system does vs how well -- and why numbers turn a diagram into engineering. |
| 03 | [Client-Server Model](03-client-server-model.md) | The request/response contract that underpins nearly every system. |
| 04 | [Request Lifecycle End-to-End](04-request-lifecycle-end-to-end.md) | Follow one request from browser to database and back. |
| 05 | [Back-of-Envelope Estimation](05-back-of-envelope-estimation.md) | Turning "it should be fast" into QPS, storage, and bandwidth math. |
| 06 | [Latency Numbers Every Engineer Should Know](06-latency-numbers.md) | The order-of-magnitude table that grounds every design in reality. |
| 07 | [Availability, Reliability, Scalability, Maintainability](07-availability-reliability-scalability-maintainability.md) | The four "-ilities" that define non-functional goals. |
| 08 | [SLA, SLO, SLI](08-sla-slo-sli.md) | Promises, targets, and the measurements behind them. |
| 09 | Vertical vs Horizontal Scaling | Bigger machine vs more machines -- and where each hits a wall. |
| 10 | Percentiles and Tail Latency | Why the average lies and p99 is what users feel. |
| 11 | Throughput vs Latency | Work-per-second vs time-per-request, and how they trade off. |
| 12 | Little's Law | The tiny formula linking concurrency, throughput, and latency. |
| 13 | Universal Scalability Law | Why adding machines eventually stops helping -- and can hurt. |

**Deeper reference:** [concepts](../../../research/backend/L0/01-what-system-design-is.md)

**Next level →** L1. Networking
