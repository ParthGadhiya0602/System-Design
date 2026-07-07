# What System Design Is; The Design Mindset

System design is the discipline of deciding how software and infrastructure components fit together so a system meets its goals at a required scale, cost, and reliability. This first topic builds the mental model and the habits you will use for everything that follows.

## Contents
- [What System Design Actually Is](#what-system-design-actually-is)
- [The Three Altitudes: DSA vs LLD vs HLD](#the-three-altitudes-dsa-vs-lld-vs-hld)
- [Why It Is Open-Ended](#why-it-is-open-ended)
- [The Design Mindset](#the-design-mindset)
- [What Understanding Looks Like Here](#what-understanding-looks-like-here)
- [How This Connects to the Rest of the Curriculum](#how-this-connects-to-the-rest-of-the-curriculum)

## What System Design Actually Is

**System design** is the act of choosing and arranging the parts of a software system so that, working together, they satisfy a set of goals. The parts are things like clients, load balancers, application servers, databases, caches, message queues, and background workers. The goals are the requirements: what the system must do (features) and how well it must do it (scale, speed, reliability, cost).

Notice the emphasis on *fit together*. A single component in isolation is rarely interesting. What makes design hard and valuable is composition: the way a cache changes what the database has to do, the way a queue lets a slow task stop blocking a fast request, the way adding a second server forces you to answer "where does the user's session live now?" The whole is a different thing from the sum of its parts, and design is reasoning about that whole.

Three properties define nearly every design goal, and you will meet them constantly:

- **Scale** - how much work the system must handle: requests per second (a **QPS**, queries per second), number of users, amount of data stored, size of each item. A system for 10,000 users and one for 10 billion are not the same design with a bigger server; they are structurally different systems.
- **Cost** - money and effort: servers, storage, bandwidth, and the human time to build and operate the thing. Infinite resources would make design trivial; the constraint is what forces choices.
- **Reliability** - how often the system is correct and available when needed, and how gracefully it behaves when a part fails. At scale, hardware and networks fail constantly, so "what happens when this breaks?" is a design question, not an afterthought.

So a compact definition: **system design is deciding how components fit together to meet defined goals at a required scale, cost, and reliability.** Everything else in this track is either a component you can reach for, a technique for combining them, or a way of reasoning about the trade-offs between arrangements.

One more framing that helps: design is mostly about *managing constraints you cannot remove*. You cannot make the network instantaneous, storage infinite, or machines immortal. You arrange components to hide, tolerate, or work within those limits.

## The Three Altitudes: DSA vs LLD vs HLD

When engineers say "design," they can mean work at three different altitudes. Confusing them is a common early mistake, so pin down the distinctions.

- **DSA / coding altitude (inside one function or one process).** **DSA** (data structures and algorithms) is about correctness and efficiency of computation on one machine: choosing a hash map over a list, writing an O(n log n) sort, avoiding an accidental O(n^2). The unit of thought is code that runs in a single process. *One-line example: "Given a list of user IDs, deduplicate them in O(n) using a hash set."*

- **Low-Level Design (LLD) altitude (inside one service/codebase).** **LLD** is object- and module-level design: the classes, interfaces, and their relationships within a single application; how responsibilities are split; which design patterns apply; how the code stays extensible and testable. The unit of thought is a well-structured codebase, still typically one deployable service. *One-line example: "Design the classes for a parking-lot system: `ParkingLot`, `Spot`, `Vehicle`, `Ticket`, and how a `PricingStrategy` plugs in."*

- **High-Level Design (HLD) altitude (across many services and machines).** **HLD** is the arrangement of distributed components: which services exist, what data stores back them, where caches and queues sit, how requests flow, how the system is partitioned and replicated to hit its scale and reliability targets. The unit of thought is boxes-and-arrows across a network. *One-line example: "Design a URL shortener for 100M new links/day: an API tier behind a load balancer, a key-generation scheme, a partitioned key-value store, and a read cache."*

The three are nested, not rival. Good HLD still needs good LLD inside each service and good DSA inside each function. But they are answered with different tools and at different times.

**This track is mostly HLD.** We care about how components combine across a network at a given scale, and about the trade-offs between arrangements. LLD (classes and patterns) and DSA (algorithms) are covered elsewhere and appear here only when a specific choice inside a component matters to the whole (for example, which data structure a rate limiter keeps in memory).

## Why It Is Open-Ended

System design has no single correct answer, and internalizing this early prevents a lot of wasted effort looking for "the" solution.

The core reason is that **the requirements, not the feature list, determine the system.** Two products with an identical feature list are completely different systems if their scale, latency, or reliability targets differ. A photo-sharing app for 10,000 users can be one database and one server; the "same" app for 10 billion photos needs partitioned storage, a content-delivery layer, replicated metadata, and asynchronous processing pipelines. Same features, different system, because the numbers changed.

The deeper reason is that **every choice is a trade-off: you gain something and you pay something.** There is no free improvement. A few recurring examples of the shape of the trade:

- **Cache to make reads fast** - you pay with staleness (the cached copy can be out of date) and added complexity (invalidation).
- **Replicate data across machines for availability** - you pay with the cost of keeping copies consistent, and you may have to accept that a read can return slightly old data.
- **Split into many services for team scalability and independent deployment** - you pay with network calls, partial failures, and harder debugging compared to one process.
- **Add a queue to smooth out load spikes** - you pay with extra latency and eventual (not immediate) processing.

Because every axis trades against another, "best" is only meaningful relative to *which goals matter most for this system*. A design that is excellent for a system optimizing cost can be wrong for one optimizing latency. So the honest output of design is not a single answer but *a defensible choice given stated priorities* - plus an awareness of the conditions under which you would choose differently.

This is why an interviewer or a senior engineer keeps asking "why?" - they are not testing recall of an architecture; they are checking that you understand what you traded away and whether that trade fits the goals.

## The Design Mindset

The mindset is a small set of habits. Internalize these and most designs become a matter of applying them in order; skip them and even simple problems get muddy.

**(a) Clarify before you build.** The first move is never to draw boxes; it is to ask questions until the problem is pinned down. What exactly does this system do? Who uses it and how? What is in scope and what is explicitly out? Requirements are almost always underspecified on purpose, and building against your unstated assumptions is the most common way to design the wrong system. Say your assumptions out loud so they can be corrected cheaply.

**(b) Requirements drive the design; numbers size it.** Separate two kinds of requirements. **Functional requirements** are what the system does ("users can post a message, followers can read it"). **Non-functional requirements** are the qualities it must have ("reads return in under 200 ms," "99.9% available," "handle 50,000 reads/second," "store 5 years of data"). Functional requirements decide *which* components you need; non-functional requirements, expressed as concrete numbers, decide *how big and how many*. A design without numbers is just a diagram; the numbers are what turn it into engineering. "It should be fast" is not a requirement - "p99 latency under 200 ms at 50k QPS" is.

**(c) Everything is a trade-off - name it.** For every non-trivial choice, state it in this shape: *"I'll use X because [it serves the priority that matters here], but if Y mattered more I'd switch to Z."* For example: "I'll use a read-through cache because reads dominate and staleness of a few seconds is acceptable; but if the data had to be exactly current on every read, I'd drop the cache and read from the primary." Naming the trade does three things: it proves you understand the cost, it exposes the assumption that justifies the choice, and it leaves a clear switch-point for when conditions change. A choice presented without its cost is a red flag, not a decision.

**(d) Start simple, then scale to relieve a real, quantified bottleneck.** Begin with the simplest design that could possibly work - often a single server and a single database. Then evolve it *only* in response to a specific, numbered pressure. "The single database can't serve 50k reads/second, so I add a read replica / cache" is good reasoning. "I'll use a sharded, multi-region, event-sourced architecture" stated up front, before any number forces it, is **premature complexity** - you pay all its costs and may need none of its benefits. Let each added component earn its place by removing a bottleneck you can point to with a number.

**(e) Data first - access patterns constrain more than compute.** Decide what data you store, how it is shaped, and *how it will be read and written* before you obsess over servers. **Access patterns** - which queries are frequent, what is read-heavy vs write-heavy, what must be looked up by which key, what must be consistent - constrain the design far more than raw compute does. Stateless application servers are easy: you add more of them. State is the hard part, because data has to live somewhere, be kept correct, and be found quickly. Getting the data model and its access patterns right early avoids the most expensive rewrites later.

These habits compose into a repeatable order you will formalize later: clarify requirements, put numbers on them, sketch the API and data model, draw a simple high-level design, then deep-dive and scale where the numbers demand it, always naming the trade at each step.

## What Understanding Looks Like Here

The goal of this track is not to memorize a catalog of finished architectures and pattern-match a problem to the nearest one. Memorized designs fail the moment the requirements shift - and requirements always shift.

Real understanding is **being able to derive a design from principles.** Concretely, you understand a topic when you can:

- Explain *why* a component exists - the specific problem it solves - not just what it is.
- Predict what happens when you add or remove it, including the new problem it creates (the cost side of the trade).
- Re-derive the choice when a number changes: if reads jump 100x, or the consistency requirement tightens, or the budget is cut, you can reason forward to a different, still-defensible design.
- State the condition under which you would choose the alternative.

A useful self-test: if you can only recite "the standard design for X," you are pattern-matching. If you can rebuild that design from its requirements and explain what each piece buys and costs, you understand it. Everything in this track is written to push you toward the second state - which is also, incidentally, what senior engineering and strong interviews actually reward.

## How This Connects to the Rest of the Curriculum

This topic is the frame for the whole track; each later stage deepens one habit introduced here.

- **Requirements gathering** turns habit (a) and (b) into a method: how to elicit functional and non-functional requirements and lock down scope before designing.
- **Back-of-the-envelope estimation** turns habit (b) into skill: computing the QPS, storage, bandwidth, and memory numbers that size the system, so your design rests on arithmetic rather than adjectives.
- **The building blocks** - load balancers, caches, databases, message queues, and the rest - give you the vocabulary of components to compose, each studied for what it solves and what it costs. These are the layers explored across the networking, database, backend, and reliability levels that follow.
- **Distributed-systems theory** (consistency, availability, the trade-offs forced by the network) explains *why* the costs in habit (c) are unavoidable - it is the physics behind the trade-offs.
- **Scalability and performance patterns** turn habit (d) into technique: the specific moves - replication, partitioning, caching layers, asynchronous processing - for relieving a quantified bottleneck.
- **Applied design practice** puts it all together: taking a problem from requirements through estimation, data model, high-level design, deep-dive, and bottleneck analysis - the full framework, with the trade named at every step.

Read the rest of the track through the lens of this topic: every component you meet is a tool for meeting a requirement at a scale, and every time you reach for it, name what you gain and what you pay.

---

## Real-world & sources

**Amazon - requirements-first via the customer, not the build.** Amazon's "Working Backwards" process forces teams to start by defining the customer experience and iteratively work backwards from it, before any engineering resources are committed. The PR/FAQ document (a mock press release plus FAQ) makes teams answer "why will this be compelling enough for customers to buy?" and is explicitly used to decide which products *not* to build - institutionalizing requirements-first thinking at the mechanism level rather than leaving it to individual discipline.

**AWS - naming trade-offs and deciding by stated priorities.** The AWS Well-Architected Framework treats architecture as a continuous set of trade-offs across six pillars and states plainly that you make trade-offs between pillars based on business context, and those business decisions drive engineering priorities - e.g. trading reliability for lower cost in dev environments, or the reverse for mission-critical workloads (while noting security and operational excellence are generally not traded off). This makes "decide by stated priorities" an explicit, named exercise rather than an implicit gut call.

**Google - design docs that force explicit non-goals.** Google's internal design-doc culture uses a "Goals / Non-Goals" section where non-goals are things that *could* reasonably be in scope but are deliberately excluded (e.g. choosing not to pursue full ACID compliance for a given system). This makes scope-narrowing and trade-off decisions visible and reviewable *before* a design is built, not discovered mid-implementation.

**Stripe (fintech) - get the design right first; correctness over premature flexibility.** Stripe runs a lightweight written API design-review so that "even with our versioning system available, we do as much as we can to avoid using it by trying to get the design of our APIs right the first time" - favoring upfront rigor over reactive patching. Its idempotency-key design shows the data/correctness-first mindset in practice: because networks are unreliable, endpoints are built to be safely retryable so financial operations converge to a correct state under failure - trading implementation complexity for operational correctness rather than optimizing raw throughput first.

**Sources**
- [An insider look at Amazon's culture and processes](https://www.aboutamazon.com/news/workplace/an-insider-look-at-amazons-culture-and-processes) (aboutamazon.com, accessed 2026-07-06)
- [Definitions - AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/definitions.html) (docs.aws.amazon.com, accessed 2026-07-06)
- [Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/) (industrialempathy.com, accessed 2026-07-06)
- [APIs as infrastructure: future-proofing Stripe with versioning](https://stripe.com/blog/api-versioning) (stripe.com, accessed 2026-07-06)
- [Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) (stripe.com, accessed 2026-07-06)
