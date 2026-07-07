# Vertical vs Horizontal Scaling

When one machine can no longer keep up with demand, you have exactly two directions to add capacity: make the machine bigger (scale up) or add more machines (scale out). This note builds both from first principles and shows when each wins.

## Contents

1. [Overview](#overview)
2. [Vertical scaling (scale up)](#vertical-scaling-scale-up)
3. [Horizontal scaling (scale out)](#horizontal-scaling-scale-out)
4. [Statelessness the enabler](#statelessness-the-enabler)
5. [Rule of thumb](#rule-of-thumb)
6. [How this connects onward](#how-this-connects-onward)

## Overview

Every server has finite resources: CPU cores, RAM, disk throughput, and network bandwidth. As traffic or data grows, the server eventually saturates one of these resources; requests queue, latency climbs, and errors appear. That point is the moment you need to "scale."

There are only two fundamental ways to add capacity:

- **Vertical scaling (scale up):** keep one machine, but make that machine more powerful (more CPU, more RAM, faster disk, a bigger instance type).
- **Horizontal scaling (scale out):** keep the machines roughly the same, but add more of them and spread the work across the fleet.

A useful mental picture: a single cashier at a shop. To handle more customers you can either train and equip that one cashier to work faster (vertical), or open more checkout lanes and route customers between them (horizontal). Each approach solves the same problem with very different consequences for cost, availability, and complexity. Real systems almost always use a blend, and knowing which lever to pull where is the core skill.

A term used throughout: a **single point of failure (SPOF)** is any component whose failure takes down the whole system because nothing else can do its job.

## Vertical scaling (scale up)

Vertical scaling means giving one machine more resources. In the cloud this is usually a one-line change to a bigger instance type (for example, doubling vCPUs and RAM); on-premises it means installing more RAM, faster or additional disks, or a CPU with more cores. The application still runs as a single process (or single node) on a single box.

**Why it is attractive:**

- **Simplest possible change.** You are not changing your architecture, only its size. Often there is *no code change at all* the same program just has more headroom.
- **No distributed-systems complexity.** With one node there is no load balancing, no data partitioning, no cross-machine coordination, and no network partitions to reason about.
- **Strong consistency is trivial.** All state lives in one place, so every read sees the latest write by construction. There is no replication lag or conflict resolution to worry about (these become central problems the moment you scale out state).
- **Simpler operations and debugging.** One set of logs, one process to profile, one machine to monitor.

**Why it eventually fails you:**

- **Hard ceiling.** There is a biggest machine money can buy. Once you are on it, you cannot scale up further, period.
- **Super-linear cost.** The largest instances cost disproportionately more per unit of CPU or RAM than mid-range ones. Doubling capacity can far more than double the price near the top of the range, so scaling up gets expensive well before you hit the ceiling.
- **Single point of failure.** One machine means one thing to fail. If it dies (hardware fault, kernel panic, bad deploy), the whole service is down until it recovers. Vertical scaling does nothing for availability.
- **Downtime to resize.** Resizing typically requires a reboot or an instance replacement, so scaling up usually means a maintenance window. (Some platforms and workloads allow live resizing, but the common case is a restart.)

**Use vertical scaling when:**

- You are early stage or at moderate scale, where simplicity is worth more than headroom.
- The workload is genuinely hard to distribute for example a single relational database primary that must serve strongly consistent writes. Scaling that node up is the standard first move *before* the far more invasive step of sharding.
- You want a quick capacity win to buy time while you design a horizontal architecture.

## Horizontal scaling (scale out)

Horizontal scaling means adding more machines (nodes) and putting a **load balancer** in front a component that distributes incoming requests across the fleet so no single node is overwhelmed. Each node runs a copy of the application; together they form a pool of interchangeable workers.

**Why it is powerful:**

- **Near-unbounded capacity.** Need more throughput? Add more nodes. There is no single-machine ceiling; you scale by count, not by the size of one box.
- **Fault tolerance.** If the fleet is designed well, any one node can die and the load balancer simply routes around it. No single node is a SPOF, so the system survives individual failures (this must be designed for it is not automatic).
- **More linear cost.** You grow with many commodity (ordinary, mass-produced) machines rather than a few exotic ones, so cost tends to track capacity more proportionally than the super-linear curve of scaling up.
- **Elasticity.** You can add nodes when traffic rises and remove them when it falls **autoscaling** so you pay for capacity closer to what you actually use.

**Why it is hard:**

- **Distributed-systems complexity.** You now must handle load balancing, **partitioning/sharding** (splitting data across machines), **replication** (keeping copies in sync), and the **consistency** questions that follow (which copy is authoritative, how stale can a read be).
- **Network partitions.** Machines communicate over a network that can drop, delay, or reorder messages. Nodes can disagree about the current state, forcing explicit trade-offs between consistency and availability.
- **Harder debugging and operations.** A single request may touch many nodes; failures are partial and intermittent. You need distributed tracing, aggregated logging, and fleet-wide monitoring to understand what happened.

**Use horizontal scaling when:**

- Scale is large or unpredictable and would blow past any single machine.
- **High availability (HA)** is a requirement you cannot tolerate one machine being a SPOF.
- You have a read-heavy web tier (many independent requests), which spreads across nodes naturally.
- You expect ~10x or more growth, where designing to scale out early is cheaper than a painful re-architecture later.

## Statelessness the enabler

Horizontal scaling works cleanly only when any node can handle any request. That property depends on **statelessness**.

A server is **stateless** when it keeps no client-specific data locally between requests. Everything it needs to serve a request either arrives in the request itself or is fetched from a shared backend. Because no node holds unique in-memory data, the load balancer can send a user's next request to a completely different node and the result is identical. Nodes become interchangeable and disposable exactly what lets you add, remove, or replace them freely.

To make an app tier stateless, you push each kind of local state out to a shared place:

- **Session and cache data** move to a shared in-memory store (for example, a Redis cache) or a database, so every node reads the same source.
- **Uploaded files and other large blobs** move to object storage instead of a node's local disk, so the file is reachable from any node.
- **User identity** travels inside the request as a signed token, so any node can verify who the caller is without looking up a local session.

The catch: the shared stores you pushed state into databases, caches, message brokers are themselves **stateful**, and scaling *them* out is the genuinely hard part of the problem. That is why the app tier is the "easy" tier to scale horizontally, while scaling state is deferred to later levels (sharding, replication, consensus).

**Anti-pattern sticky sessions.** A sticky session pins a given user to one specific node (usually so that node can keep the user's session in local memory). It looks convenient, but it reintroduces the problems statelessness solved: that node becomes a SPOF for those users, load spreads unevenly, and you cannot freely remove or replace nodes without dropping sessions. Prefer shared session state and keep nodes interchangeable.

## Rule of thumb

**Scale up first, scale out when forced.**

Scaling up is cheap in engineering effort and free of distributed complexity, so it is usually the right first move. Reach for horizontal scaling when one of three forces makes vertical scaling untenable:

- **Ceiling** you have run out of bigger machines.
- **Availability** you cannot accept a single machine as a SPOF.
- **Cost** the super-linear price of large machines makes many small ones cheaper for the same capacity.

In practice, mature systems do **both at once**, applied to different tiers:

- The **stateless app tier** is scaled *out* many interchangeable nodes behind a load balancer, autoscaled with traffic.
- The **database tier** is scaled *up* give the primary node a bigger box and it stays vertical as long as possible, because scaling state out is invasive. Only when the DB is forced past its ceiling (or needs HA) do you take on replication and sharding.

So "vertical vs horizontal" is rarely an either/or verdict for the whole system; it is a per-component decision driven by that component's ceiling, availability needs, and cost curve.

## How this connects onward

This topic is the hinge between running on one machine and running as a distributed system. The mechanics each raise here are studied in depth at later levels:

- **Load balancing** how requests are actually spread across a horizontal fleet, and how the balancer detects and routes around dead nodes.
- **Sharding / partitioning** splitting a dataset across many machines so no single node holds it all the primary way to scale stateful stores out.
- **Replication** keeping multiple copies of data for availability and read throughput, and the lag that copies introduce.
- **Consistency models** the rules for what a read is allowed to see once data lives on more than one node the trade-offs that appear the moment you scale state out.
- **Autoscaling** adding and removing nodes automatically in response to load, which turns horizontal capacity into an elastic, cost-tracking resource.

Master this distinction and its enabler, statelessness, and every one of those later topics reads as a detailed answer to a question you already understand: how do we add capacity without a single machine becoming the limit.
