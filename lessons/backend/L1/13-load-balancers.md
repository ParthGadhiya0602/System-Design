# Load Balancers (L4/L7, Algorithms, Health Checks)

*One stable address out front. Behind it, a pool of servers that grows, shrinks, and occasionally dies -- and clients never notice any of it.*

`⏱️ ~8 min · 13 of 17 · Networking`

> [!TIP] The gist
> A **load balancer (LB)** sits in front of a pool of backend servers and **spreads incoming requests across them** so no one server is swamped -- which is what makes **horizontal scaling** (topic 0) and **high availability** actually work. It's a **specialized reverse proxy** (topic 11) with two jobs: distribute load, and be a stable front (one **VIP**) for a changing set of backends. The big choice: **L4** routes by IP+port without reading content (fast, dumb, often DNAT from topic 12) vs **L7** terminates the connection and reads the HTTP request (path/host/cookie routing, TLS termination -- smart, more work). An **algorithm** (round robin, least connections, consistent hashing) picks *which* backend. And the piece that turns "distribution" into "availability" is **health checks** -- eject dead backends, re-add revived ones. Catch: the LB itself must not become the new single point of failure.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture a busy restaurant with a **maître d'** at the door and a floor of waiters.

Diners don't wander in and pick a waiter. The maître d' seats each arriving party, **spreading them across the floor** so no single waiter is buried while others stand idle. That's *distribution*.

Crucially, the maître d' only seats parties with waiters who are **actually on shift and not overwhelmed** -- if a waiter goes home sick mid-service, they simply stop getting new tables. That's a *health check*, and it's why the floor keeps running smoothly even when a waiter drops out.

And if one party needs the *same* waiter all night (they've started a tab, the waiter knows their order), the maître d' **remembers that pairing**. That's a *sticky session*.

A load balancer is that maître d': spread the load, route only to healthy servers, and -- when it must -- remember a client-to-server pairing.

## The concept

**Definition.** A **load balancer** is a component that sits in front of a group of backend servers and **distributes incoming requests across them**, so no single server is overwhelmed while others sit idle, and so the system keeps serving even when individual servers fail.

**Two jobs, always:**

1. **Spread the load** -- split traffic across a pool by some algorithm, so total capacity scales with the *number* of servers, not with whatever one machine can handle.
2. **Be a stable front for a changing pool** -- clients only ever know one address; servers can be added, upgraded, or crash entirely invisibly.

**Why it matters -- the two problems one server can't solve:**

- **Horizontal scaling** (back-ref L0). A bigger machine has a hard ceiling and a built-in single point of failure. Adding more machines has no ceiling -- but *only* if something out front can spread requests across the new ones. The LB is that something: it makes a whole pool look like one server from the outside.
- **High availability** -- routing around failure. If 1 of 10 backends crashes, an LB that's health-checking simply stops sending it traffic and serves from the other 9. "Just add servers" gives you fault tolerance *only* if something notices a server is down and stops sending it work.

**Key terms:**

- **Backend pool** (a.k.a. target group / upstream) -- the set of servers the LB distributes across. Membership changes constantly; clients never know.
- **Virtual IP (VIP)** -- the single stable address clients connect to. It belongs to *no* real backend -- the LB owns it, answers on it, then forwards to a chosen backend.

**A load balancer is a specialized reverse proxy** (back-ref [11-forward-and-reverse-proxies.md](11-forward-and-reverse-proxies.md)). A reverse proxy fronts a pool and can be a choke point for TLS termination, routing, caching. A load balancer is that same mechanism, specialized for *"distribute across a pool, by an algorithm, informed by health checks."* Every LB is a reverse proxy; not every reverse proxy is an LB.

**What it is NOT:** it is **not just "round robin."** Distribution alone is the easy part -- **health checks** are what make it *highly available*, not merely *balanced*. And the LB itself **must not be a single point of failure** (covered below), or it defeats its own purpose.

## How it works

### L4 vs L7 -- the centerpiece choice

This split comes straight from the OSI map (back-ref [01-osi-and-tcp-ip-models.md](01-osi-and-tcp-ip-models.md)): *how far up the stack does the LB look before deciding?*

- **L4 (transport) LB** decides using only IP + port + the TCP/UDP 4-tuple, and **never reads the payload.** Mechanically it usually just rewrites packet headers -- **DNAT** (back-ref [12-nat.md](12-nat.md)): swap the destination from the VIP to a chosen backend, swap replies back. It doesn't know or care whether the bytes are HTTP, gRPC, or a database protocol.
- **L7 (application) LB** **terminates** the connection (a real TCP/TLS endpoint, like the reverse proxy in topic 11) and **reads the actual HTTP request** -- method, path, headers, cookies -- before deciding. Because it understands the request, it can route on content and do **TLS termination** (back-ref [07-https-tls.md](07-https-tls.md)).

| | **L4 load balancer** | **L7 load balancer** |
|---|---|---|
| **What it sees** | IP, port, TCP/UDP 4-tuple | Full HTTP request: path, host, headers, cookies |
| **Routing decisions** | IP/port only -- can't tell requests apart within a connection | Content-aware: path, host, header, cookie, method |
| **Speed / overhead** | Very fast; often just a header rewrite, no full termination | Slower: terminate + TLS decrypt + parse per request |
| **TLS termination** | Typically no (passthrough) | Yes -- a primary job |
| **Content routing** | No | Yes (`/api` vs `/images` etc.) |
| **Typical use** | Raw TCP/UDP, extreme throughput, protocol-agnostic | HTTP(S) APIs and web traffic needing smart routing |

### Algorithms -- which backend gets this request?

Once the LB knows it's forwarding *somewhere*, the **algorithm** picks *which* backend:

- **Round robin** -- cycle in order: 1, 2, 3, 1, 2... Simple, stateless. **Weighted** round robin gives bigger servers a proportionally larger share.
- **Least connections** -- send to whichever backend has the *fewest active connections*. Needs bookkeeping, but adapts to real load when requests vary in cost.
- **IP hash** -- hash the client IP to a backend deterministically -> same client, same backend (a simple form of stickiness, no cookies).
- **Consistent hashing** -- a hashing scheme where adding/removing a backend remaps only a *small* fraction of keys (unlike naive `hash % N`, which reshuffles nearly everything when `N` changes). Great for stickiness and cache locality with a resizing pool. Full mechanics get a dedicated deep-dive at the partitioning level (forward-ref L4).

**Worked example -- round robin vs least connections diverge.** Three idle backends, five requests arrive in order, and request 2 is slow (still running when 3, 4, 5 arrive).

| Request | Round robin | Least connections (active conns) |
|---|---|---|
| 1 | A | A (all 0) |
| 2 (slow) | B | B (A:1, B:0, C:0) |
| 3 | C | C (A:1, B:1, C:0) |
| 4 | A | A (all tied at 1) |
| 5 | **B** -- but B is still stuck on request 2! | **B or C** (A:2, B:1, C:1) -- *never A*, and won't pile onto B |

Round robin blindly sends request 5 to the still-busy B just because "it's B's turn." Least connections sees B is loaded and routes elsewhere. The moment request costs are uneven -- the normal case -- the two algorithms genuinely make different choices.

### Health checks -- this is the HA

Distribution only helps if the backends are alive; an LB that keeps feeding a dead server makes things *worse*. **Health checks are what turn distribution into high availability.**

- **Active** -- the LB periodically probes each backend (e.g. `GET /health` or a TCP connect) on a fixed interval. Fail the **unhealthy threshold** (N consecutive fails) -> eject from the pool. Pass the **healthy threshold** later -> re-admit. Two thresholds (not one) exist to avoid *flapping* a backend in and out on a single transient blip.
- **Passive** -- observe the real traffic already flowing; if a backend starts timing out or erroring, mark it unhealthy -- no dedicated probe endpoint needed.

**Ejection and re-entry:** an unhealthy backend just stops being a candidate the algorithm can pick; the LB keeps checking it and re-admits it once healthy again -- so a crashed-and-restarted server rejoins automatically, no human involved.

```
                        VIP (one stable address)
   Client --request-->  [ Load Balancer ]
                          |    |    |   \___ active probe: GET /health
                (algorithm-selected, healthy only)
                          v    v    v
                    [Back A] [Back B] [Back C]
                    healthy  healthy  UNHEALTHY -> ejected
```

### Sticky sessions + keeping the LB itself alive

Some apps need every request from a client to hit the **same** backend:

- **In-memory session state** -- if a backend holds cart/login state only in its own memory, the user must keep landing there or their session vanishes.
- **WebSockets** (back-ref [08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md)) -- a single long-lived connection can't be re-balanced mid-stream, so it's pinned to one backend for its lifetime.

Stickiness is done via a **cookie (L7)**, **IP hash**, or **consistent hashing**. But **stickiness undermines the very thing an LB is for** -- once pinned, the LB can't freely rebalance a hot client, and if the pinned backend dies, its in-memory state is gone regardless of health checks. The better fix: make backends **stateless** and keep session state in an external shared store any backend can read (forward-ref caching / L4 databases), so stickiness becomes an optimization, not a requirement. WebSockets are the genuine exception.

**And the LB itself must be HA.** A single LB instance is a single point of failure -- if it dies, a perfectly healthy pool is unreachable. Fixes: **active-passive pairs with a floating VIP** (VRRP-style failover), **active-active** instances, **health-checked DNS**, or **anycast** (forward-ref [16-anycast-bgp.md](16-anycast-bgp.md)). Real systems layer it:

```
Client -> DNS/GSLB (nearest healthy region) -> L4 LB (fast, coarse) -> L7 LB (content-aware, TLS) -> backend pool
```

Each tier solves a different scale-vs-precision trade-off, pushing expensive per-request work down to only the tier that needs it.

## In the real world

- **AWS productizes the split.** Elastic Load Balancing ships two products that *are* this document's centerpiece: **NLB (Network Load Balancer)** operates at **L4** (routes on the IP/port tuple, doesn't inspect payload, built for extreme performance); **ALB (Application Load Balancer)** operates at **L7** (terminates HTTP(S), routes on URL path and host header, integrates with WAF -- which NLB structurally cannot, since it never parses the payload). The clearest real articulation of "L4 = fast and blind, L7 = slower and content-aware" -- as two separately purchasable products. ([AWS ELB features](https://aws.amazon.com/elasticloadbalancing/features/))
- **Google Maglev** is a software **L4** LB on commodity Linux (production since 2008, published NSDI 2016). Routers spread packets across Maglev machines via **ECMP**, and each machine uses **consistent hashing over the 5-tuple** to pick a backend -- so every machine independently converges on the *same* backend for a connection, with **no state sync** needed. L4 routing + consistent-hash stickiness, both live. ([Maglev paper](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/))
- **Cloudflare** runs a Maglev-derived scheduler behind **BGP anycast**: because every LB machine computes the identical consistent-hash outcome, it doesn't matter which one a router picks, and failing an LB (or a whole data center) out is just *withdrawing a BGP route* -- no DNS TTL wait. HA-of-the-LB-itself in production. ([Cloudflare blog](https://blog.cloudflare.com/high-availability-load-balancers-with-maglev/))
- **Stripe (fintech)** routes API traffic through **HAProxy**, with the backend pool kept current by HashiCorp Consul (config regenerated ~every 60s) while HAProxy independently runs **active health checks every 2 seconds** -- so a failed machine is ejected in *seconds*, not a minute. Health-checks-as-HA, in practice. ([Stripe blog](https://stripe.com/blog/service-discovery-at-stripe))

Full sourcing: [research/backend/L1/13-load-balancers.md](../../../research/backend/L1/13-load-balancers.md#real-world-and-sources).

## Trade-offs

| Axis | ✅ Benefit | ❌ Cost |
|---|---|---|
| **L4 load balancing** | Very fast, protocol-agnostic, connection-sticky by construction | No content routing, no TLS-aware decisions -- "dumb" by design |
| **L7 load balancing** | Content-aware routing, TLS termination, cookie stickiness | CPU cost of parse + decrypt per request; protocol-specific |
| **Round robin** | Simple, stateless | Ignores real backend load |
| **Least connections** | Adapts to uneven, real load | Must track live connection counts |
| **Sticky sessions** | Lets stateful/WebSocket backends work at all | Undermines even distribution; complicates failover -- prefer stateless + shared store |
| **DNS-based (GSLB)** | Cheap, works at global/geographic scale (back-ref [03-dns-deep.md](03-dns-deep.md)) | Reacts to failure only as fast as TTLs/caching allow -- minutes |
| **A real LB w/ health checks** | Fast, health-aware failure reaction (seconds) | Extra infra that itself must be made HA |

**LB vs reverse proxy vs API gateway, one line:** all three are the same underlying pattern (an intermediary fronting a pool). A **reverse proxy** is the general term; a **load balancer** is one specialized for algorithm-driven distribution + health checks; an **API gateway** is one specialized for API concerns (auth, transformation, per-client rate limiting) and *also* often balances load (forward-ref [14-api-gateway.md](14-api-gateway.md)).

## Remember

> [!IMPORTANT] Remember
> A load balancer spreads traffic across a **healthy backend pool** so you can **scale horizontally** and **survive failures** -- it's a reverse proxy specialized for that one job. **L4** = fast IP/port forwarding without reading content (often DNAT); **L7** = terminate the connection and route on the actual request (path/host/cookie) plus TLS termination. The **algorithm** picks *which* backend and genuinely changes outcomes on uneven traffic. And **health checks are what turn distribution into high availability** by ejecting dead backends and re-admitting revived ones. Last catch: the LB sits in every request's path, so it must itself be made HA (redundant pairs + floating VIP, or anycast) -- or it becomes the very single point of failure it was built to remove.

## Check yourself

1. Your app must route `/api/*` and `/images/*` to *different* backend pools. L4 or L7 -- and why can't the other one do it?
2. Two servers; one is bogged down by a slow request that's still running. The next request arrives -- does round robin or least connections handle it better, and what does each actually do?
3. You add a load balancer to a system to make it highly available. Why does this *not* automatically remove your single point of failure -- and name two ways to fix that.

---

→ Next: [API Gateway](14-api-gateway.md) (the single front door: auth, routing, rate limiting)
↩ Comes back in: API gateway, CDN, Anycast/BGP, rate limiting & circuit breakers (reliability), consistent hashing (partitioning), autoscaling (cloud)
