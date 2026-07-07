# Functional vs Non-Functional Requirements

Before you draw a single box or pick a single database, you write down what the system must do and how well it must do it. That list is the requirements, and splitting it into two kinds is the first real move in any design. This topic teaches that split, why it matters more than any technology choice, and how to extract it from a vague prompt.

## Contents
- [What a Requirement Is, and the Two Kinds](#what-a-requirement-is-and-the-two-kinds)
- [Functional Requirements: What the System Does](#functional-requirements-what-the-system-does)
- [Non-Functional Requirements: How Well the System Does It](#non-functional-requirements-how-well-the-system-does-it)
- [Why FRs Converge but NFRs Diverge](#why-frs-converge-but-nfrs-diverge)
- [How to Elicit Requirements](#how-to-elicit-requirements)
- [How This Connects Onward](#how-this-connects-onward)

## What a Requirement Is, and the Two Kinds

A **requirement** is a statement of something the system must be true of, agreed before you build it. It is the contract the design has to satisfy. Everything downstream - which components you pick, how you arrange them, what you trade away - exists only to meet requirements. Design without requirements is not design; it is decoration. You cannot call an arrangement "good" or "bad" except relative to what it was supposed to achieve.

Requirements come in two fundamentally different kinds, and telling them apart is the skill this topic builds:

- **Functional requirements (FRs)** describe *what the system does* - the features, behaviors, and capabilities a user can observe. "A user can post a message." "A user can search for a product by name." They are about *function*: verbs the system performs.
- **Non-functional requirements (NFRs)** describe *how well the system does it* - the quality attributes and constraints under which those functions must run. "A search returns in under 200 milliseconds." "The system stays up 99.99% of the time." "No posted message is ever lost." They are about *qualities and limits*, not features.

A quick test to classify any statement: if it describes a capability a user would *ask for by name* ("let me upload a photo"), it is functional. If it describes a *property of how that capability behaves* - speed, uptime, safety, cost, correctness under failure - it is non-functional. "Upload a photo" is an FR; "upload finishes in under 2 seconds for a 10 MB file, and the photo is never silently corrupted" is two NFRs attached to that FR.

**Why separate them at all, and why first?** Three reasons.

1. **They are gathered differently.** FRs come from product and users ("what should it do?"). NFRs come from scale, money, and risk ("how much, how fast, how safe, how reliable?"). Mixing them hides the NFRs, which are the ones most often left unstated and most likely to sink a design.
2. **They drive different parts of the design.** FRs shape the *interface* - the API and the data model, the surface the user touches. NFRs shape the *architecture* - the number of servers, the choice of database, whether you add a cache or a queue, how you replicate and partition. You need both, but they act on different layers.
3. **NFRs decide almost everything expensive.** As the next sections show, two systems with an identical feature list can be structurally unrelated because their NFRs differ. If you skip the NFRs, you have not scoped the problem; you have only named it.

So the very first move in a design is: enumerate the functional requirements to know *what* you are building, then pin down the non-functional requirements to know *what kind of thing* it has to be. Only then do you start choosing components.

## Functional Requirements: What the System Does

A **functional requirement** is a specific behavior the system must exhibit - a thing a user or another system can do with it, or that it does on their behalf. Good FRs read as short, testable statements, usually with an actor and a verb: "A user can create an account." "A user can follow another user." "The system sends a notification when a followed user posts." Each one is a capability you could demonstrate and check off.

FRs are what most people think of first because they are the visible product. But their real job in design is that **they map directly onto two artifacts you will build next: the API surface and the data model.**

- **FRs map to the API surface.** Each functional requirement implies one or more operations the system must expose. "A user can post a message" becomes an endpoint like `createMessage(userId, text) -> messageId`. "A user can read a conversation" becomes `getMessages(conversationId, pagination) -> [messages]`. Turning FRs into a list of operations - their inputs, outputs, and who can call them - is how a feature list becomes a concrete interface. If an FR does not produce at least one operation, either it is not really functional or you have missed something.

- **FRs map to the data model.** Each functional requirement implies things the system must *remember* and relationships between them. "A user can follow another user" tells you there is a `User` entity and a `follows` relationship (many-to-many) between users. "A user can post a message in a conversation" tells you there are `Message` and `Conversation` entities and that a message belongs to a conversation and an author. The nouns in your FRs become entities; the verbs connecting them become relationships and operations. This is why writing FRs carefully pays off twice - it hands you a first draft of both the API and the schema.

**Picking a core subset to design deeply.** Real systems have dozens of features, and you cannot - and should not - design all of them to the same depth. The discipline is to **choose a small core subset of FRs that exercises the interesting parts of the system, design those deeply, and explicitly defer the rest.** For a chat system, the core is usually "send a message" and "receive/read messages in a conversation"; features like message editing, reactions, read receipts, and profile pictures are named and then set aside. You choose the core by asking which features (a) are central to the product's value and (b) stress the hard parts of the system - the high-volume paths, the ones with tight latency or consistency needs. Deferring is not skipping: you *state* the deferred items so it is clear you scoped deliberately, not from oversight. Designing a narrow slice well beats sketching everything shallowly, because depth is where the real trade-offs live.

## Non-Functional Requirements: How Well the System Does It

A **non-functional requirement** is a quality attribute or constraint the system must satisfy while performing its functions - a statement about *how well*, not *what*. Where FRs are a list of features, NFRs are a set of dials, and the setting of each dial changes the machine you have to build. The good ones are quantified: not "it should be fast" but "95% of reads complete in under 100 ms." Vague NFRs are nearly useless because you cannot design toward or test against them.

Here are the key non-functional requirements you will meet, each with a one-line definition. Learn these names - they are the vocabulary of the rest of the track.

- **Scalability** - the ability to handle growth (more users, requests, or data) by adding resources, ideally without redesigning. The question it answers: "what happens when load goes up 10x or 100x?"
- **Availability** - the fraction of time the system is up and able to serve requests, usually stated in "nines" (99.9% is roughly 8.8 hours of downtime a year; 99.99% is about 52 minutes). The question: "how much downtime is acceptable?"
- **Latency / performance** - how long a single operation takes, measured end to end. Latency is per-request response time; performance is the broader notion of being fast enough. Because a single average hides pain, latency is reported as **percentiles**: **p50** (the median - half of requests are faster) and **p99** (the 99th percentile - only 1 in 100 requests is slower). A great p50 with a terrible p99 means most users are happy but a meaningful, often vocal, minority suffers - so tail latency (p99, p999) is frequently the real target.
- **Throughput** - the amount of work per unit time the system can sustain: requests per second, messages per second, bytes per second. Latency is about *one* request; throughput is about *how many* at once. They interact - pushing throughput near the limit usually inflates latency.
- **Consistency** - whether all readers see the same, most recent data. **Strong consistency** means a read always reflects the latest write; **eventual consistency** means readers may briefly see stale data but all copies converge over time. This single dial reshapes the whole storage design.
- **Durability** - the guarantee that once data is accepted (committed), it is not lost, even through crashes, power loss, or disk failure. Often expressed like availability, in nines of data retention. "Never lose a committed order" is a durability requirement.
- **Reliability / fault tolerance** - the ability to keep working correctly when parts fail, and to fail gracefully when they must. Reliability is the broad property "it behaves correctly over time"; fault tolerance is the specific ability to survive component failures (a dead server, a lost network link) without taking the whole system down.
- **Security / privacy** - protecting data and operations from unauthorized access, tampering, or disclosure (security), and honoring rules about who may see or use personal data and for what (privacy). Includes authentication, authorization, encryption, and data-handling constraints.
- **Cost** - the money and effort to build and operate the system: compute, storage, bandwidth, and human time. Cost is the constraint that makes every other dial a trade-off instead of a free choice; with infinite budget most NFRs are trivial.
- **Maintainability** - how easily the system can be understood, changed, extended, and operated by the team over time. Covers code and architecture clarity, modularity, and how safely you can deploy changes.
- **Observability** - the degree to which you can understand what the system is doing from the outside, via logs, metrics, and traces. You cannot operate or debug what you cannot see; observability is what lets you know an NFR is being met or violated.

The crucial point: **these NFRs, far more than the FRs, decide the architecture.** The feature "send a message" is the same whether you serve a hundred users or a billion. But the answers to "how fast, how many per second, how consistent, how durable, how available, at what cost" are what force you toward a single database or a partitioned cluster, toward a synchronous write path or a queue, toward one region or many. When you find yourself reaching for a cache, a replica, a shard, or a message queue, it is virtually always an NFR - not an FR - driving the decision.

## Why FRs Converge but NFRs Diverge

Here is the observation that makes this topic matter: **for a given product, different engineers will write nearly the same functional requirements, but their non-functional requirements - and therefore their architectures - can be wildly different.** FRs converge; NFRs diverge; the design follows the NFRs.

The reason FRs converge is that the product idea largely fixes them. Ask ten engineers to list the features of a chat app and you will get almost the same list every time: send a message, receive messages, see conversation history, show who is online. The features are the shared understanding of "what a chat app is."

The reason NFRs diverge is that they encode *scale, quality, and stakes*, which the product idea alone does not fix - they come from the specific context (how many users, how much money, how bad is failure). And because NFRs drive the architecture, divergent NFRs produce divergent systems from the same feature list.

Make it concrete with two chat systems that share **the identical FR** "a user sends a message and other participants see it":

- **Best-effort chat** (say, an ephemeral in-game lobby chat). NFRs: messages should *usually* arrive within a second; occasional loss or reordering is acceptable; no long-term storage needed; strong consistency not required; moderate scale. This design can be simple - a single service, messages pushed over open connections, little or no persistence, no complex ordering or delivery guarantees. Cheap and easy because the NFRs are lenient.

- **Real-time, ordered, never-lose-a-message chat** (say, a chat that doubles as a system of record). NFRs on the *same feature*: every accepted message is durably stored and never lost (high durability); messages are delivered in the exact order sent within a conversation (ordering guarantee); delivered to every participant even if they were offline (reliable delivery with per-user state); p99 delivery under, say, 200 ms; available 99.99% of the time across regions. Now the design must add durable, ordered storage per conversation, an acknowledgement and retry protocol, per-recipient delivery tracking, replication for availability, and sequencing to preserve order. It is a structurally different, far larger system - built from the *same one-line feature*.

Same FR, radically different builds, entirely because the NFRs differ. This is why, in any design exercise, nailing the NFRs is where the real thinking is - and why "what are the non-functional requirements?" is the question that separates a scoped design from a hand-wave. The features tell you what to build; the NFRs tell you *what it has to be*, and that is the design.

## How to Elicit Requirements

Requirements rarely arrive complete. A prompt like "design a chat app" or a product brief gives you the rough FRs and almost none of the NFRs. **Eliciting requirements** is the skill of asking the right targeted questions to turn a vague prompt into a concrete, numeric spec - and, where you cannot get an answer, stating an explicit assumption and moving on. Do not stall waiting for perfect information; a stated assumption is a first-class part of a design.

Ask about these areas, roughly in this order. Each question exists to pin down an NFR or to size one later.

- **Users and scale.** How many total users? How many active at once (concurrently)? Expected growth? This sizes everything downstream. Convert a fuzzy "a lot of users" into a number: "assume 100 million registered users, 10 million daily active."
- **Read vs write ratio.** For each core operation, how many reads per write? Most systems are read-heavy (e.g. 100 reads per write), and that ratio decides whether you optimize the read path with caches and replicas or the write path with partitioning. "Assume a 100:1 read:write ratio" is a design-shaping assumption.
- **Data size, shape, and retention.** How big is each item (a message is bytes; a video is megabytes)? What is the structure (relational, document, blob, time-series)? How long must it be kept - forever, 30 days, until the user deletes it? Size times rate gives storage growth; shape steers the storage engine; retention decides deletion and archival.
- **Latency and consistency targets.** How fast must each core operation feel, stated as a percentile ("p99 read under 100 ms")? Does the user need to see their own and others' latest writes immediately (strong consistency), or is a brief delay acceptable (eventual)? These two together often decide the hardest parts of the storage and caching design.
- **Availability target.** How much downtime is tolerable, in nines? Is this a system where an outage is an annoyance, or one where it is a lost sale, a safety issue, or a legal problem? The target ("99.99%") sets how much redundancy and failover you must design.
- **Special constraints.** Regulatory or privacy rules (where data may live, who may see it), security needs (encryption, auth), geographic distribution (one region or global), cost ceilings, and any hard integration or platform limits. These are NFRs that can dominate the design when present.

**Turning vague answers into concrete NFRs.** The move is always the same: replace an adjective with a number and a unit. "It should be fast" becomes "p99 latency under 200 ms for reads." "It should handle a lot of traffic" becomes "sustain 50,000 requests per second at peak." "We can't lose data" becomes "durability of committed writes; zero acceptable loss; availability 99.99%." A requirement you cannot measure is one you cannot design toward or verify, so quantify relentlessly - and when the person cannot give you a number, you supply a reasonable one and label it an assumption: "Assuming 10M daily active users each sending 20 messages a day..." Stating the assumption makes your design auditable: anyone can challenge the number, and the design updates accordingly. This is exactly how a senior engineer works - not by knowing the numbers, but by making the numbers explicit and reasoning from them.

## How This Connects Onward

Requirements are the root of the whole design process; nearly every later topic is downstream of what you settle here.

- **Estimation sizes the NFRs.** The next step after gathering requirements is back-of-the-envelope estimation: turning the numbers you elicited (users, request rates, item sizes) into concrete quantities - queries per second, storage per year, bandwidth, memory for a cache. Estimation is where the NFRs stop being targets and become dimensioned figures that pick components. Vague requirements cannot be estimated; that is why quantifying them here is not optional.
- **The design framework runs on requirements.** The standard flow - requirements, then estimation, then API design, then data model, then high-level design, then deep dives and bottleneck analysis - starts with this topic precisely because everything after it consumes its output. The API and data model come straight from the FRs; the high-level architecture comes straight from the NFRs.
- **The "-ilities" are the NFRs, studied in depth.** The quality attributes named here - scalability, availability, reliability, consistency, durability, and the rest - each get their own deep treatment in later levels. What you learned here is the map; those levels are the territory. Scalability patterns, reliability and fault-tolerance engineering, consistency and distributed-systems theory, security, and cost and operations all trace back to an NFR you first named at requirements time.
- **SLA, SLO, and SLI formalize the NFRs.** Later you will meet Service Level Objectives (SLO, an internal target like "p99 under 100 ms"), Service Level Indicators (SLI, the measured value), and Service Level Agreements (SLA, the external promise with consequences). These are the operational, contractual form of the non-functional requirements you draft here - the same quality attributes, now measured, promised, and enforced.
- **Every component choice traces back to a requirement.** When you later choose a load balancer, a cache, a particular database, a queue, a replication strategy, or a partitioning scheme, the honest justification is always "because requirement X demanded it." A design decision that cannot be traced to a requirement is either unjustified or is meeting a requirement you failed to write down. Getting the requirements right, and quantified, is therefore the highest-leverage thing you do in any design.
