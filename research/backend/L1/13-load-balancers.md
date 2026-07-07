# Load Balancers: L4/L7, Algorithms, and Health Checks

_A reverse proxy with one very specific job: never let a single backend take the full weight of the world, and never let clients notice when a backend quietly dies._

`⏱️ ~9 min · 13 of 17 · L1 Networking`

## Contents

- [What a load balancer is and why it exists](#what-a-load-balancer-is-and-why-it-exists)
- [L4 vs L7 load balancing](#l4-vs-l7-load-balancing)
- [Load-balancing algorithms](#load-balancing-algorithms)
- [Health checks: how a load balancer delivers availability](#health-checks-how-a-load-balancer-delivers-availability)
- [Session persistence / sticky sessions](#session-persistence-sticky-sessions)
- [High availability of the load balancer itself](#high-availability-of-the-load-balancer-itself)
- [Where load balancers sit: types and tiers](#where-load-balancers-sit-types-and-tiers)
- [How load balancers connect to the rest of the stack](#how-load-balancers-connect-to-the-rest-of-the-stack)
- [Trade-offs and common confusions](#trade-offs-and-common-confusions)
- [Check yourself](#check-yourself)
- [Real-world and sources](#real-world-and-sources)

## What a load balancer is and why it exists

A **load balancer** is a component that sits in front of a group of backend servers and **distributes incoming requests across them**, so that no single server is overwhelmed while others sit idle, and so that the overall system keeps working even when individual servers fail.

Two big jobs, always:

1. **Spread the load** — split incoming traffic across a pool of servers according to some algorithm, so aggregate capacity scales with the number of servers rather than being capped by whatever one machine can handle.
2. **Provide a stable front for a changing set of backends** — clients (and DNS) only ever need to know about one stable address; servers can be added, removed, upgraded, or crash, entirely invisibly to the client.

**Key vocabulary:**

- **Backend pool / target group / upstream pool** — the set of servers the load balancer distributes traffic across. Membership changes over time (scale-out adds servers, failures remove them) without clients ever needing to know.
- **Virtual IP (VIP)** — the single, stable IP address (or hostname) that clients actually connect to. It is not the address of any one real backend; the load balancer owns it and is the thing that actually answers on it, then forwards traffic onward to whichever real backend it selects.

**Why this matters — the two problems it solves that a single server cannot:**

- **Horizontal scaling** (back-ref L0 vertical vs horizontal scaling). Vertical scaling — buying a bigger machine — has a hard ceiling and a single point of failure baked in. Horizontal scaling — adding more, typically smaller, machines — has no such ceiling, but it only works at all if something in front of the pool can actually spread requests across the new machines. The load balancer is that "something": it is the piece of infrastructure that *makes* horizontal scaling usable from the outside as if it were still one server.
- **High availability** — routing around failure. If one backend in a pool of ten crashes, a load balancer that is health-checking the pool simply stops sending it traffic and keeps serving from the other nine. Without a load balancer, "just add more servers" doesn't automatically give you fault tolerance — something still has to *notice* a server is down and *stop* sending it requests, which is exactly the health-check machinery covered later in this document.

**Framing: a load balancer is a specialized reverse proxy.** [11-forward-and-reverse-proxies.md](11-forward-and-reverse-proxies.md) already established that a reverse proxy fronts a group of servers, terminates the client's connection, and can act as a single choke point for cross-cutting concerns — TLS termination, routing, caching, and load distribution among them. A load balancer is that same mechanism, specialized specifically for the "distribute traffic across a pool, using an algorithm, informed by health checks" job. Every load balancer is a reverse proxy; not every reverse proxy is a load balancer (a reverse proxy fronting exactly one backend, doing only TLS termination, is not balancing anything).

```
                     Virtual IP (VIP)
Client ----request---->  [ Load Balancer ]
                             |   |   |
                     (health-checked, algorithm-selected)
                             v   v   v
                        [Backend 1] [Backend 2] [Backend 3]
                         (healthy)   (healthy)   (UNHEALTHY - ejected)
```

## L4 vs L7 load balancing

This is the centerpiece distinction of the whole topic, and it comes directly from [01-osi-and-tcp-ip-models.md](01-osi-and-tcp-ip-models.md), where the L4-vs-L7 line was first introduced as "how far up the stack does this device look before making a decision."

### L4 (transport-layer) load balancing

An **L4 load balancer** makes its routing decision using only transport-layer (and below) information — source/destination IP and port, the TCP/UDP connection 4-tuple ([04-tcp.md](04-tcp.md#where-tcp-sits-and-the-4-tuple)) — and **never reads the application payload at all.** It does not know or care whether the bytes flowing through it are HTTP, gRPC, a database protocol, or anything else.

Mechanically, an L4 load balancer typically works by rewriting packet headers, not by terminating and re-originating a full application-level connection:

- **DNAT-based forwarding** (back-ref [12-nat.md](12-nat.md#dnat-destination-nat-the-server-side-mirror-image-and-its-link-to-load-balancers-forward-ref)) — the LB rewrites the destination IP/port of inbound packets from its own VIP to a chosen backend's real address, and rewrites the reverse traffic's source back to the VIP on the way out, exactly mirroring the DNAT mechanism already covered.
- **Direct Server Return (DSR)** — a variant where the LB rewrites and forwards only the inbound request packets to the backend, but the backend's response traffic goes **directly** back to the client, bypassing the LB entirely on the return path (the backend is configured to recognize and answer as the VIP). This trades a bit of setup complexity for removing the LB from the (often much larger) response-traffic path entirely — very effective when responses are large relative to requests, e.g. serving video or bulk downloads, `verify` exact configuration variance across implementations.

**Consequences of only seeing L4 data:**

- **Very fast.** No parsing of application content, often no full connection termination — sometimes packets are just forwarded with a header rewrite, which is dramatically cheaper per packet than terminating a TCP connection and reading an HTTP request.
- **Protocol-agnostic.** Works for any TCP or UDP traffic — HTTP, gRPC, raw database connections, custom binary protocols — since it never needs to understand what's inside.
- **Connection-oriented / "sticky by construction."** Once a TCP connection is established and mapped to a backend, all packets belonging to that same connection (same 4-tuple) go to the same backend for the connection's lifetime — there's no mechanism to reroute mid-connection, because the LB isn't inspecting anything that would let it make a different decision per request.
- **Cannot make content-based decisions.** It cannot route `/api/*` differently from `/images/*`, cannot read a cookie, cannot look at a Host header — all of that lives inside the payload it never opens.

Canonical example: **AWS NLB (Network Load Balancer)** operates at L4; **LVS (Linux Virtual Server)** is a long-standing open-source L4 load balancer built on Linux kernel-level packet forwarding, `verify` current implementation details.

### L7 (application-layer) load balancing

An **L7 load balancer** **terminates** the client's connection (exactly like the reverse proxy mechanism in topic 11 — it is a genuine TCP/TLS endpoint, not a pass-through) and **reads the actual application-layer request** before deciding what to do with it: the HTTP method, URL path, headers, cookies, and (if it chooses to buffer it) the body.

Because it actually understands the request, an L7 load balancer can do everything an L4 one cannot:

- **Content-based routing** — path-based (`/api/*` to one pool, `/static/*` to another), host-based (different hostnames to different backend pools from the same LB), header- or cookie-based (route by an API version header, an A/B test cookie, a feature flag).
- **TLS termination** (back-ref [07-https-tls.md](07-https-tls.md)) — the LB holds the certificate, completes the handshake with the client, and can then inspect the now-decrypted request to make its routing decision, optionally re-encrypting to the backend or speaking plain HTTP internally.
- **Request rewriting** — modifying headers, injecting `X-Forwarded-For` (back-ref topic 11), rewriting paths, compressing responses.
- **Cookie-based session stickiness** — reading or injecting a cookie that pins a client to a specific backend across multiple requests (developed fully below).

The cost: more work per request (parsing the request, often decrypting TLS), and being protocol-specific (an L7 HTTP load balancer only understands HTTP; it can't usefully "load balance" an arbitrary raw TCP protocol it has no parser for).

Canonical examples: **AWS ALB (Application Load Balancer)**, **nginx**, **HAProxy**, **Envoy** all operate (or can be configured to operate) at L7.

### Required comparison

| | L4 load balancer | L7 load balancer |
|---|---|---|
| **What it sees** | IP addresses, ports, TCP/UDP 4-tuple | Full HTTP request: method, path, headers, cookies, (optionally) body |
| **Routing decisions** | Based on IP/port only; cannot differentiate requests within one connection | Based on path, host, header, cookie, method — content-aware |
| **Speed / overhead** | Very fast, low CPU per packet, often no full connection termination | Slower per request: connection termination, TLS decrypt, request parsing |
| **TLS termination** | Typically no (passthrough is common) — `verify` some managed L4 LBs support TLS termination for specific protocols | Yes — this is one of its primary jobs |
| **Content-based routing** | No | Yes |
| **Stickiness** | Connection-level, by construction (4-tuple pins a connection to one backend) | Request-level control (can pin via cookie across many separate connections/requests) |
| **Typical use** | Raw TCP/UDP services, extreme throughput needs, protocol-agnostic traffic, DB connections | HTTP(S) APIs and web traffic needing smart routing |
| **Canonical examples** | AWS NLB, LVS | AWS ALB, nginx, HAProxy, Envoy |

## Load-balancing algorithms

Once a load balancer has decided *that* it's forwarding a request somewhere, it still has to decide **which backend, out of the pool**. That decision is the algorithm.

- **Round robin** — cycle through the backend pool in a fixed order, one request (or connection) at a time: backend 1, then 2, then 3, then back to 1. Simple, requires no state about backend load, and works well when every backend has identical capacity and requests are roughly uniform in cost.
  - **Weighted round robin** — the same cycling, but each backend gets a weight, and receives a proportional share of requests (e.g. a backend with weight 2 gets twice as many requests as one with weight 1) — used for **heterogeneous** server pools, e.g. some backends are on bigger instances than others.
- **Least connections** — send the next request to whichever backend currently has the **fewest active connections**. This requires the LB to track live connection counts per backend, but adapts to real, current load rather than blindly cycling — much better when requests vary widely in how long they take or how expensive they are.
  - **Weighted least connections** — combines both ideas: divide the "fewest active connections" count by a capacity weight, so a bigger backend can carry proportionally more concurrent connections before being deprioritized.
- **Least response time / least load** — extends least-connections with an additional signal: pick the backend with the best combination of low active-connection count *and* fast recent response times (some implementations also fold in CPU/memory metrics reported by the backend). More adaptive still, at the cost of the LB needing to continuously measure and maintain response-time statistics per backend.
- **IP hash / hash-based** — compute a hash of the client's IP (or another key, like the 4-tuple) and use it to deterministically pick a backend: the same client always lands on the same backend as long as the pool doesn't change. This is a simple form of **stickiness** (see below) achieved purely through the algorithm, with no cookies or extra state required.
- **Consistent hashing** — a hashing scheme designed so that when a backend is added or removed, only a small, predictable fraction of the existing key-to-backend mappings change (rather than nearly all of them, which is what happens with naive `hash(key) % N` the moment `N` changes). Used here for **stickiness with minimal disruption** and for **cache locality** (e.g. routing all requests for a given cache key to the same backend so its local cache stays warm) when the backend pool is expected to resize. This is introduced here at the level of "why a load balancer would want it"; the full mechanics (hash rings, virtual nodes, rebalancing math) get a dedicated deep treatment in the databases/partitioning level, since the same technique is central to sharding data across nodes.

### Worked example: round robin vs least connections making different choices

Three backends, all currently idle. Five requests arrive in sequence, but request 2 happens to be an expensive, long-running one that takes far longer to finish than the others.

**Round robin**, oblivious to actual load, simply cycles:

| Request | Backend chosen (round robin) |
|---|---|
| 1 | Backend A |
| 2 (slow, still running) | Backend B |
| 3 | Backend C |
| 4 | Backend A |
| 5 | Backend B — sent here even though B is still busy with request 2! |

By request 5, round robin blindly sends more work to Backend B, which is still tied up with the slow request 2, simply because it's "B's turn" in the cycle — it has no idea B is overloaded.

**Least connections**, by contrast, tracks that Backend B still has an active connection from request 2 and route request 5 elsewhere:

| Request | Active connections before routing | Backend chosen (least connections) |
|---|---|---|
| 1 | A:0, B:0, C:0 | Backend A (tie, picks A) |
| 2 (slow, still running) | A:1, B:0, C:0 | Backend B |
| 3 | A:1, B:1, C:0 | Backend C |
| 4 | A:1, B:1, C:1 | tie — picks A (implementation-defined tie-break) |
| 5 | A:2, B:1, C:1 | Backend B or C (both tied at 1) — **never A**, and correctly avoids piling onto B beyond its fair share |

This is exactly why algorithm choice matters in practice: round robin is cheap and stateless but blind to real load; least connections costs a small amount of bookkeeping but adapts to reality, and the two algorithms genuinely produce different outcomes for the identical request stream the moment request costs are uneven — which is the normal case in most real systems.

## Health checks: how a load balancer delivers availability

Distributing load only helps if every backend in the pool is actually alive; a load balancer that keeps sending traffic to a dead server is actively making things worse. **Health checks are the mechanism that makes a load balancer deliver high availability, not just distribution** — this is the direct payoff of the "route around failure" job described at the top of this document.

- **Active health checks** — the load balancer itself periodically sends a probe request to each backend (commonly an HTTP `GET /health` or a simple TCP connect) on a fixed **interval** (e.g. every 5-10 seconds), independent of real client traffic. If a backend fails to respond, or responds with an unhealthy status, for a configured number of consecutive checks (the **unhealthy threshold**), the LB marks it unhealthy and stops routing new traffic to it. It's typically re-added only after a number of consecutive **successful** checks (the **healthy threshold**) once it starts responding again — this two-threshold design (rather than acting on a single failed or single successful check) exists specifically to avoid flapping a backend in and out of the pool on one transient blip.
- **Passive health checks** — instead of (or in addition to) separate probes, the LB observes the real traffic it's already sending: if requests to a given backend start timing out or returning errors at an elevated rate, it marks that backend unhealthy based on that observed failure pattern, without needing a dedicated health-check endpoint at all.
- **Ejection and re-entry** — an unhealthy backend is pulled out of the pool (it simply stops being a candidate the algorithm can select) but is not necessarily removed permanently; the LB keeps checking it (actively, passively, or both) in the background, and re-admits it to the pool once it passes the healthy threshold again — this is what lets a backend that crashed, restarted, and came back up automatically rejoin without any manual intervention.

Tuning intervals and thresholds is itself a trade-off: checking too infrequently means a dead backend keeps receiving (and failing) real requests for longer before being ejected; checking too aggressively adds load and can eject a backend on a brief, harmless blip. The deeper reliability math around failure detection, retries, and cascading failure belongs to the reliability level (forward-ref L7/L8) — this topic's job is just to establish *that* health checking is the mechanism, and the active/passive distinction.

## Session persistence / sticky sessions

Some applications need every request from a given client to land on the **same** backend, not just any healthy one:

- **In-memory session state** — if a backend keeps a user's session data (cart contents, login state) only in its own process memory rather than in a shared external store, the user must keep hitting that same backend or their session appears to vanish.
- **Stateful connections, especially WebSockets** (back-ref [08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md)) — a WebSocket connection is a single, long-lived, stateful connection; there is no meaningful way to "load balance" a single already-established connection across multiple backends mid-stream, so the entire connection must be pinned to one backend from the moment it's established.

**Methods:**

- **Cookie-based stickiness (L7)** — the load balancer sets a cookie (either its own, or observes an application-set one) on the first response identifying which backend served the client; on subsequent requests, it reads that cookie and routes back to the same backend, as long as it's still healthy.
- **IP hash (L4 or L7)** — deterministically maps a client's IP to a backend via a hash function, achieving stickiness without any cookie at all — the trade-off is that clients behind the same NAT'd IP (back-ref [12-nat.md](12-nat.md)) all collapse onto the same backend, and a client whose IP changes (e.g. switching networks) loses stickiness.
- **Consistent hashing** — as introduced above, hashing a client or session key onto a hash ring so the same key maps to the same backend, with minimal remapping if the pool changes size — a smoother version of IP hash that also handles a resizing pool gracefully.

**The trade-off — stickiness undermines the very thing load balancing is for.** Once a client is pinned, the load balancer can no longer freely redistribute their traffic to whichever backend is currently least loaded; a "sticky" backend that happens to attract many long, expensive sessions becomes a hot spot the LB cannot correct for without breaking those clients' sessions. Stickiness also complicates failover: if the pinned backend dies, the client's in-memory state genuinely is gone, no matter how good the health check is. The better long-term fix, in most architectures, is to make backends **stateless** — hold no session state locally at all, and instead keep it in an external, shared store (a cache or database, forward-ref caching/L3 and L4 databases) that *any* backend can read — so that any healthy backend can serve any request and stickiness becomes an optimization rather than a requirement. WebSockets are a genuine exception to "just go stateless," since the connection itself is inherently pinned to one server for its duration regardless of what state is stored where.

## High availability of the load balancer itself

Everything above assumes the load balancer is always there to do its job — but a single load balancer instance is itself a single point of failure: if it goes down, it doesn't matter how healthy the backend pool is, because nothing is left routing traffic to it at all. "Who balances the balancer?" is a real, standard question in system design.

**Common solutions:**

- **Active-passive LB pairs with a floating/virtual IP** — two (or more) load balancer instances are deployed; one is actively serving traffic bound to the VIP, the other stands by. If the active instance's health checks fail (monitored by the passive instance or an external watchdog), the VIP is reassigned ("failed over") to the passive instance, which takes over serving traffic — typically achieved with a protocol like VRRP (Virtual Router Redundancy Protocol, `verify` exact protocol in a given deployment) that lets a floating IP move between physical/virtual hosts.
- **Active-active LB pairs** — multiple LB instances all actively serve traffic simultaneously (rather than one standing idle), typically fronted by DNS round-robin or anycast routing clients to whichever instance is closest/healthy, so capacity and failover are combined rather than paying for an idle standby.
- **Health-checked DNS** — DNS records for the LB tier itself can be health-checked (back-ref [03-dns-deep.md](03-dns-deep.md)), so that if an entire LB instance (or an entire region's LB tier) becomes unreachable, DNS stops handing out its address to new clients — though this inherits DNS's caching-latency limitation already covered in that topic: a client (or resolver) holding a cached, now-stale record keeps trying a dead address until its TTL expires.
- **Anycast** — the same IP address is announced from multiple physical locations, and network routing (BGP) naturally sends each client to the topologically nearest one; if a location fails, routing reconverges to the next-nearest instance without the client needing to do anything or wait out a DNS TTL at all (forward-ref, anycast/BGP get a full treatment in a later topic).

**The layered reality — DNS LB -> L4 LB -> L7 LB -> servers.** Real, large-scale systems rarely rely on one single load-balancing layer; balancing typically happens in tiers, each handling a different scope of the problem:

```
Client
  |
  v
DNS-based / GSLB layer   (global: routes to the nearest healthy region/data center)
  |
  v
L4 load balancer          (regional: extremely fast, coarse distribution across many
  |                         L7 LB instances or backend pools, absorbs huge raw throughput)
  v
L7 load balancer          (local: content-aware routing - path, host, cookie -
  |                         TLS termination, fine-grained algorithm + health checks)
  v
Backend server pool
```

Each tier exists because it's solving a different scale/precision trade-off: DNS/GSLB makes coarse, slow-to-change, geography-scale decisions cheaply; L4 handles enormous raw connection volume fast and cheaply; L7 makes fine-grained, content-aware decisions at the cost of doing real per-request work — pushing that expensive work down to only the tier that actually needs it.

## Where load balancers sit: types and tiers

- **Hardware load balancers** — dedicated physical appliances (historically from vendors like F5, Citrix `verify` current market), built to push very high throughput with specialized silicon; capital-expensive and less flexible to reconfigure, but proven and often used where extreme, predictable throughput matters most.
- **Software load balancers** — ordinary software running on commodity servers or VMs: **nginx**, **HAProxy**, **Envoy** are the canonical examples. Cheaper, far more flexible (reconfigure and redeploy via code/config), and now the dominant approach in most modern cloud-native architectures.
- **Cloud-managed load balancers** — a cloud provider operates the load balancing tier as a managed service, so the user never runs or patches the LB software itself. **AWS NLB** (L4) and **AWS ALB** (L7) are canonical examples of this split implemented as distinct managed products; **Google's Maglev** is a published, influential design for a software network load balancer built to run on commodity hardware at very large scale (`verify` specifics before citing beyond "Google published a paper describing it"). Deeper cloud-specific configuration and autoscaling integration belongs to the cloud level (forward-ref L14).
- **Global Server Load Balancing (GSLB)** — load balancing decisions made at the DNS/geography layer, deciding which **region or data center** a client should be routed to at all, before any local L4/L7 balancing happens within that region. This is typically implemented via DNS (back-ref [03-dns-deep.md](03-dns-deep.md)'s GeoDNS) or anycast, and is a *global* decision, distinct from *local/regional* balancing across a pool of servers within one data center.
- **Client-side load balancing** — rather than a dedicated LB device/service in the traffic path at all, the client (or, more commonly, a sidecar proxy sitting alongside it) maintains its own view of the healthy backend pool and picks a target itself before sending a request. This is how many gRPC clients and service-mesh sidecars operate for internal service-to-service traffic, removing an extra network hop through a centralized LB for calls that stay inside a trusted internal network (forward-ref, service mesh at L11).

## How load balancers connect to the rest of the stack

- **Back to [11-forward-and-reverse-proxies.md](11-forward-and-reverse-proxies.md)** — a load balancer is a reverse proxy specialized for distributing traffic across a backend pool with an algorithm and health checks; everything about connection termination, `X-Forwarded-For`, and TLS termination-vs-passthrough from that topic applies directly to an L7 load balancer.
- **Back to [01-osi-and-tcp-ip-models.md](01-osi-and-tcp-ip-models.md)** — the L4-vs-L7 distinction, first introduced abstractly there, is exactly what separates NLB-style from ALB-style load balancing in practice.
- **Back to [12-nat.md](12-nat.md)** — L4 load balancing is, mechanically, DNAT (rewriting the destination address of inbound traffic to a chosen backend) with a balancing algorithm and health checks layered on top.
- **Back to [07-https-tls.md](07-https-tls.md)** — an L7 load balancer terminating TLS is doing exactly the TLS-termination mechanics already covered, just specifically at the load-balancing tier.
- **Back to [08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md)** — the need for connection-level or session-level stickiness is driven directly by WebSockets' single, long-lived, stateful connection.
- **Back to [03-dns-deep.md](03-dns-deep.md)** — round-robin DNS was already covered there as the cheapest, crudest form of load distribution, and its caching/TTL limitation is exactly why real load balancers (which react to health instantly, not on a TTL) exist as a complement to, not a replacement for, DNS-based GSLB.
- **Forward to consistent hashing's full depth (databases/partitioning level)** — this topic introduces consistent hashing only far enough to explain stickiness and cache locality; the hash-ring mechanics and rebalancing math get their full treatment there, applied to sharding data rather than routing requests.
- **Forward to rate limiting and circuit breakers (reliability level)** — health checks are this topic's mechanism for availability; the deeper failure-handling patterns (retries, backoff, circuit breakers, rate limiting per client) that often sit at the same architectural layer as a load balancer belong there.
- **Forward to autoscaling (cloud level)** — a load balancer's backend pool changing size dynamically in response to load is the load-balancing half of a story whose other half (deciding *when* to add/remove backend instances) belongs to autoscaling.
- **Forward to service mesh and sidecars (L11)** — client-side load balancing, briefly introduced above, is developed fully there.
- **Forward to CDNs (a later topic in this level)** — a CDN is, among other things, a geographically distributed edge tier that itself performs a form of load distribution (routing a client to the nearest edge node), conceptually related to GSLB.
- **Forward to API gateways (a later topic)** — an API gateway is a reverse proxy specialized for API-specific concerns (auth, transformation, per-client rate limiting) and often sits logically alongside or behind a load balancer; the two are related but distinct specializations of the same underlying reverse-proxy mechanism.

## Trade-offs and common confusions

**L4 vs L7, restated as one line each:** L4 is fast, protocol-agnostic, and dumb (connection-sticky by construction, no content awareness); L7 is smart and content-aware (path/host/cookie routing, TLS termination) at the cost of per-request overhead and being protocol-specific.

**The load balancer as a potential single point of failure.** Ironically, the very component introduced to provide high availability for the backend pool can itself become the system's weakest link if it isn't made highly available in its own right — hence active-passive/active-active LB pairs, floating VIPs, health-checked DNS, and anycast, all covered above.

**Sticky sessions vs stateless scaling.** Stickiness solves an immediate problem (stateful backends, WebSockets) but works against even load distribution and complicates failover; the generally preferred long-term architecture is stateless backends plus an external, shared session store, with stickiness reserved for cases (like WebSockets) where it's structurally unavoidable.

**Round robin vs least connections.** Round robin is stateless and cheap but ignores actual backend load; least connections (and its weighted/least-response-time variants) costs a small amount of bookkeeping but adapts to real, uneven request costs — the worked example above shows the two algorithms genuinely diverge on an identical request stream.

**DNS-based load balancing vs a real load balancer.** Round-robin DNS (back-ref topic 3) is cheap and can operate at global/geographic scale, but reacts to a failed backend only as fast as DNS caching/TTLs allow — potentially minutes. A dedicated load balancer with active health checks reacts in the interval between checks (seconds), because it is a live, in-path component making a fresh decision on every request rather than a record cached ahead of time by resolvers it doesn't control.

**Hardware vs software vs cloud-managed.** Hardware appliances offer proven, very high raw throughput at high capital cost and low flexibility; software LBs (nginx, HAProxy, Envoy) are flexible, cheap, and dominant in modern architectures; cloud-managed LBs (NLB, ALB) trade some control for zero operational burden.

**"Load balancer" vs "reverse proxy" vs "API gateway" — resolving the overlap.** All three are, mechanically, the same underlying pattern (an intermediary fronting a backend pool). A **reverse proxy** is the general term for that pattern. A **load balancer** is a reverse proxy specialized specifically for distributing traffic using an algorithm and health checks. An **API gateway** is a reverse proxy specialized for API-specific cross-cutting concerns (authentication, request/response transformation, per-client rate limiting, protocol translation) sitting at the edge of a set of API services — it very often performs load balancing too, but that's one of several jobs it does, not its defining one (forward-ref, API gateways get their own topic).

| | Benefit | Cost |
|---|---|---|
| L4 load balancing | Very fast, protocol-agnostic | No content-based routing, no TLS-aware decisions |
| L7 load balancing | Content-aware routing, TLS termination, cookie stickiness | CPU cost of parsing/decrypting, protocol-specific |
| Round robin | Simple, no state needed | Ignores real backend load |
| Least connections | Adapts to real, uneven load | Requires tracking live connection counts |
| Sticky sessions | Lets stateful/WebSocket backends work at all | Undermines even distribution; complicates failover |
| DNS-based (GSLB) balancing | Cheap, works at global scale | Caching/TTL delay reacting to failures |
| A dedicated LB with health checks | Fast, health-aware reaction to failure | Extra infrastructure that itself needs to be made HA |

> [!IMPORTANT]
> A load balancer is a reverse proxy specialized to spread traffic across a backend pool and keep that pool healthy. L4 balancing routes by IP/port alone, without reading application content, favoring speed and protocol-agnosticism over intelligence; L7 balancing terminates the connection and reads the actual request, trading overhead for content-aware routing, TLS termination, and cookie-based stickiness. The algorithm (round robin, least connections, hash-based, consistent hashing) decides *which* backend gets a given request, and different algorithms genuinely make different choices on the same traffic; health checks (active and passive) are what actually deliver high availability, by ejecting and re-admitting backends based on whether they're really alive. And because the load balancer sits directly in every request's path, it must itself be made highly available — usually via redundant LB instances behind a floating VIP, health-checked DNS, or anycast — or it simply becomes the new single point of failure it was built to eliminate.

## Check yourself

- A team puts an L4 load balancer in front of their HTTP API and asks it to route `/api/v1/*` to one service and `/api/v2/*` to another. Why can't an L4 load balancer do this, and what would they need instead?
- Walk through why round robin and least connections can make different routing decisions for the fifth request in an identical stream, if the second request happens to be unusually slow.
- A backend fails a single health check due to a one-second network blip, then immediately passes the next ten checks. Why do most load balancers use a consecutive-failure threshold rather than ejecting a backend after just one failed check?
- A junior engineer proposes solving session stickiness by simply always sending every client to the same backend via IP hash, forever. What real distribution and failover problems does this create, and what's the generally preferred alternative architecture?
- Why is a single load balancer instance, despite being the thing that provides HA for the backend pool, itself a single point of failure — and name two distinct techniques used to fix that.

## Real-world and sources

**Google Maglev — the canonical software L4 load balancer, consistent hashing + ECMP.** Maglev is Google's software network load balancer, running on commodity Linux servers, serving Google's production traffic since 2008 and published at USENIX NSDI 2016. Routers spread packets across a pool of Maglev machines using **ECMP (Equal-Cost Multi-Path)** routing; each Maglev machine then applies **consistent hashing over the packet's 5-tuple** (protocol, source/dest IP, source/dest port) to pick a backend, so that all Maglev machines independently converge on the same backend for a given connection without needing to synchronize state — and a single Maglev machine can saturate a 10 Gbps link. This is a direct, real implementation of two concepts in this document at once: L4 routing decided purely from IP/port/protocol data, and consistent hashing used for connection stickiness that survives pool resizing. Maglev's design (ECMP + consistent-hash software LB, announced via anycast) is also the basis Cloudflare cites for its own edge load balancer.

**AWS NLB vs ALB — the official, productized L4/L7 split.** AWS Elastic Load Balancing ships two separate managed products that map directly onto this document's centerpiece distinction: **Network Load Balancer (NLB)** operates at L4, routing on the TCP/UDP connection's IP/port tuple without inspecting payload, supports static/Elastic IPs and long-lived TCP connections, and is recommended by AWS for "extreme performance" and protocol-agnostic traffic; **Application Load Balancer (ALB)** operates at L7, terminating HTTP(S) and making routing decisions on URL path, host header, and other request content, and integrates with AWS WAF for request-level inspection — something NLB structurally cannot do because it never decrypts or parses the payload. Both support cross-zone load balancing and configurable active health checks (TCP/HTTP/HTTPS). This is the clearest real-world articulation available of "L4 is fast and blind, L7 is slower and content-aware" as two genuinely different, separately purchasable products rather than a theoretical distinction. (Verified against AWS's ELB feature-comparison page, accessed 2026-07-07.)

**Cloudflare — anycast + a Maglev-derived scheduler as HA-of-the-LB-itself.** Cloudflare's edge load-balancing layer is built on the same Maglev consistent-hashing scheduler described in Google's paper: routers use ECMP (hashing strictly on the packet's 5-tuple) to pick which of many load balancer machines handles an inbound packet, and because every LB machine computes the identical consistent-hash outcome for a given connection, *it doesn't matter which one the router picks* — they all route it to the same backend, so no state needs to be synchronized between load balancers. Service IPs are announced from multiple data centers via **BGP anycast**, so failing a load balancer, or an entire data center, out of rotation is a matter of withdrawing a BGP route rather than waiting on a DNS TTL. This is a real production instance of two patterns in this document's "HA of the load balancer itself" section: anycast-based failover, and a stateless design that sidesteps the active/passive-pair problem entirely by making any instance interchangeable for a given connection.

**Stripe — HAProxy as the production L7/L4-capable software load balancer, tuned health-check intervals.** Stripe routes its API traffic through **HAProxy**, with the backend pool kept current by HashiCorp Consul: Consul Template regenerates HAProxy's config (via DNS-based service entries) roughly every 60 seconds, and HAProxy independently runs **active health checks every 2 seconds** against backends, so a failed machine is detected and ejected from rotation in seconds rather than waiting a full minute for the next config regeneration. HAProxy's graceful-restart feature lets in-flight connections finish before a config reload replaces the running process. This is a concrete, fintech-domain instance of this document's active-health-check mechanism (fixed probe interval, fast ejection) and of software load balancers (HAProxy, alongside nginx/Envoy) being the dominant real-world choice over hardware appliances. (Verified against Stripe's engineering blog, accessed 2026-07-07.)

### Sources / further reading

- Eisenbud et al., ["Maglev: A Fast and Reliable Software Network Load Balancer"](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/) — USENIX NSDI 2016, Google Research (accessed 2026-07-07).
- AWS, ["Elastic Load Balancing features" (NLB vs ALB comparison)](https://aws.amazon.com/elasticloadbalancing/features/) — official product docs (accessed 2026-07-07).
- Cloudflare Engineering Blog, ["High-availability load balancers with Maglev"](https://blog.cloudflare.com/high-availability-load-balancers-with-maglev/) (accessed 2026-07-07).
- Stripe Engineering Blog, ["Service discovery at Stripe"](https://stripe.com/blog/service-discovery-at-stripe) (accessed 2026-07-07).
