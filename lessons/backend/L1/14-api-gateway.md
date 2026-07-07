# API Gateway (The Single Front Door for Your Services)

*One address for every client. Behind it, dozens of services -- and the same plumbing (auth, rate limits, TLS) done once at the door instead of copy-pasted into every one of them.*

`⏱️ ~7 min · 14 of 17 · Networking`

> [!TIP] The gist
> An **API gateway** is a **single entry point** in front of many backend services that handles the **cross-cutting concerns** of API traffic -- auth, rate limiting, routing, TLS termination, transformation, aggregation, observability -- so each service doesn't reimplement them. It's a **specialized reverse proxy** (topic 11), the next specialization after the load balancer (topic 13): an LB spreads load across *identical* replicas; a gateway routes to *different* services and centralizes API plumbing. Keep it **thin** (routing and policy, never business logic) or it becomes a distributed-monolith chokepoint, and make it **highly available** -- it's the entry point for everything.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture an **airport terminal**.

Every traveler is heading somewhere different -- gate 12, gate 34, a connecting flight. But nobody does security, passport control, and boarding-pass checks *at the gate*. Imagine if they did: every single gate would need its own scanner, its own ID desk, its own security staff -- the same checks, duplicated dozens of times, each slightly inconsistent.

Instead there's **one checkpoint** at the entrance. Everyone passes through it *once* -- security, ID, ticket -- and only then gets routed to the right gate. The checkpoint centralizes the shared work so the gates stay simple: a gate just boards a plane.

An API gateway is that checkpoint. Clients pass through **one front door** that does auth, rate limiting, and TLS *once*, then routes each request to the right backend service -- so the services stay simple.

## The concept

**Definition.** An **API gateway** is a single entry point that sits in front of a set of backend services and handles the **cross-cutting concerns** of API traffic -- the things every API needs (auth, rate limiting, routing, TLS, logging) but that have nothing to do with any one service's business logic -- so individual services don't each reimplement them.

**Two key terms:**

- **Cross-cutting concern** -- functionality that applies uniformly across *many* services rather than belonging to any one. Auth, rate limiting, TLS, and logging are the classics: every service needs them, none of them is what makes the "orders service" different from the "payments service." Left unmanaged, a cross-cutting concern gets copy-pasted into every service -- and drifts inconsistent over time.
- **Single entry point** -- one well-known address all external clients talk to, regardless of how many services actually exist behind it or how they're organized.

**Why it exists.** In a **monolith**, "who checks the API key" is just code inside one process. In a **microservices** architecture (forward-ref L11) there may be dozens or hundreds of independently deployed services. Without a gateway, every client would need to know each service's address, and every service would independently implement auth, rate limiting, TLS, and logging -- the same logic duplicated N times, with N chances to get it wrong. The gateway centralizes all of it: clients only talk to the gateway, and the gateway is the only thing backends need to trust.

**It's a specialized reverse proxy.** Topic 11 introduced the idea: a reverse proxy fronts a pool and can be a choke point for cross-cutting concerns. Topic 13 specialized it one way -- *distribute load across an identical pool*. An API gateway specializes it a different way -- *centralize everything an API client needs, and route to different services*. Every gateway is (mechanically) a reverse proxy; not every reverse proxy is a gateway.

**What it is NOT.** A gateway must stay **thin** -- routing, policy, and cross-cutting concerns only. It is **not** the place for business logic ("calculate this order's discount," "decide which service owns this data"). The moment domain rules live in gateway config, your independently deployable services collapse back into a **distributed monolith** -- everything funnels through one shared, business-logic-laden chokepoint that nothing can be changed around independently.

## How it works

### The concerns it centralizes

Each is a full topic elsewhere; here's just enough to see *why the gateway is the natural home* for it:

- **Routing** -- pick the right *service* by path (`/orders/*`), host, or version (`/v1/*` vs `/v2/*`). Like L7 LB routing (topic 13) but across *different services*, not identical replicas.
- **Auth / token validation** -- validate an API key, JWT, or OAuth token *at the edge*, so backends can trust anything reaching them (token mechanics -> L9 security).
- **Rate limiting / quotas** -- cap requests per client and reject the excess (`429 Too Many Requests`) before a backend is burdened (algorithms -> L7/L8).
- **TLS termination** -- the gateway holds the cert and completes the handshake (back-ref [07-https-tls.md](07-https-tls.md)), exactly like an L7 LB.
- **Transformation / protocol translation** -- rewrite headers, expose a REST facade over internal gRPC (back-ref [09-rest-grpc-graphql.md](09-rest-grpc-graphql.md)), handle versioning.
- **Response aggregation** -- fan one request out to several services, combine into one response, so the client makes one call.
- **Caching** -- serve repeat `GET` responses at the edge so they never hit a backend (forward-ref CDN).
- **Observability** -- log every request, emit per-route metrics, inject a **trace ID** so one request can be followed across every service it touches (forward-ref L8).
- **Load balancing / circuit breaking** -- once it knows the service, it balances across that service's healthy replicas (back-ref topic 13) and cuts off slow/failing backends (forward-ref L7/L8).

Mechanically, a request moves through a **pipeline of stages** (middleware/plugins in a defined order), each able to inspect, modify, or **reject** before the next:

```
                       ┌───────────── API Gateway ─────────────┐
Clients                │  TLS      Auth      Rate     Routing   │      Backend services
(mobile, web,  ──────► │  term.    (JWT/key) limit    (path/    │ ───► ┌──────────┐
 partner API)          │                     check    version)  │  ┌─► │ Orders   │
                       │                                        │  │   └──────────┘
                       │  Transform  Aggregate  Observability   │──┤   ┌──────────┐
                       │  (rewrite)  (fan-out)  (trace ID)      │  ├─► │ Users    │
                       └────────────────────────────────────────┘  │   └──────────┘
                                                                    │   ┌──────────┐
                                                                    └─► │ Shipping │
                                                                        └──────────┘

Without a gateway -- the alternative:

Clients ──────────────►  Orders service    (own auth, own rate limit, own TLS...)
        ──────────────►  Users service     (own auth, own rate limit, own TLS...)
        ──────────────►  Shipping service  (own auth, own rate limit, own TLS...)
   clients must know every address; every service duplicates the same plumbing
```

A request **fails fast** at whichever stage rejects it -- an invalid token never reaches routing; an over-quota client never reaches a backend. The more failures the gateway catches early, the more backend capacity is protected for genuine traffic.

### Worked example -- one request through the gateway

A mobile client calls `GET /api/orders/123` with `Authorization: Bearer <JWT>` and `X-Api-Key: <key>`:

1. **TLS terminate.** The gateway is the client's TLS endpoint (back-ref topic 7) and decrypts the request. Backends never see the handshake.
2. **Validate the JWT.** Signature + expiry checked *first*. Invalid/expired -> `401` immediately, **no backend touched**.
3. **Check the rate limit** for this API key against its quota. Over quota -> `429` here, **no backend involved**.
4. **Route to the orders service.** Parse `/api/orders/*`, match the orders service, pick a healthy replica (own LB logic, or a dedicated LB behind).
5. **Optionally aggregate.** Also call users (customer name) and shipping (tracking) in parallel, combine into one JSON -- the client made **one** call.
6. **Add a trace ID.** Generate or propagate one, attach it to every downstream call so all three can be correlated in logs (forward-ref L8).
7. **Return** the combined response -- with the gateway, not any backend, having done TLS, auth, rate limiting, routing, aggregation, and tracing.

**What the backends therefore didn't do:** orders, users, and shipping never validated a JWT, checked a rate limit, terminated TLS, or made a trace ID -- they got a plain, already-authenticated internal request and did their one job. That's the entire value proposition in one trace.

### Gateway vs reverse proxy vs load balancer

Topic 13 flagged that these terms overlap. All three are the **same underlying pattern** -- an L7-capable intermediary that terminates client connections and fronts backends. What differs is **purpose**:

| | Reverse proxy | Load balancer | API gateway |
|---|---|---|---|
| **Core question** | "Relay / shield / cache for the server(s)?" | "Which *identical* replica handles this?" | "What does this API client need, and which *different service* answers?" |
| **Backend pool** | One or more servers | Interchangeable replicas of the *same* service | Many *different* services, each with its own pool |
| **Primary concern** | Generic relaying, TLS, caching, hiding origin | Distribution algorithm, health checks, stickiness | Auth, rate limiting, routing by API contract, transformation, aggregation |
| **Client-facing?** | Usually transparent | Usually transparent | Explicitly the client's contract -- versioned, documented, key-managed |
| **Relationship** | The general category | A specialization of reverse proxy | A specialization of reverse proxy; often *also* load-balances |

They're routinely **layered**, not exclusive: a common deployment puts an LB in front of a fleet of gateway instances (so the gateway tier is itself HA), and the gateway load-balances across each service's replicas once it picks a service. Tools like Envoy, Kong, and nginx can be configured to *be* all three -- which is exactly why the terms blur; the distinction lives in intent, not hardware.

**One more axis:** a gateway handles **north-south** traffic (external clients <-> your system). Traffic *between* internal services (**east-west**) is a **service mesh's** job (forward-ref L11) -- a structurally different piece whose sidecars sit next to *every* service, not at one edge.

## In the real world

- **Netflix Zuul -- build-your-own edge gateway.** Zuul is Netflix's "front door" for all requests from every device into the backend -- over **50,000 requests/second** at peak across 1,000+ device types (Netflix's own figures). It's the pipeline-of-stages made concrete: composable **filters** in four types -- `PRE` (auth, choose origin, log), `ROUTING` (send the origin request), `POST` (add headers, metrics, stream response), and `ERROR` -- dynamically hot-loaded at runtime so routing can change without a deploy. The canonical *build-your-own* point of the trade-off spectrum. ([Netflix tech blog](https://netflixtechblog.com/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee))
- **AWS API Gateway -- the managed option.** Throttling uses the **token bucket** algorithm (a token per request, steady refill, burst cap), returning `429` when exceeded, layered across four levels (account, per-account, per-API/stage, per-client via usage plans + API keys). AWS explicitly says *don't* use API keys alone for auth -- pair the gateway with IAM, a Lambda authorizer, or Cognito. Auth + rate limiting *configured, not coded*. ([AWS docs](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html))
- **Stripe (fintech) -- rate limiting at the API edge.** Stripe layers **four** limiters (not one) on a **token bucket**: a per-second request-rate limiter (their most-triggered, rejecting *millions* of requests a month), a concurrent-in-flight limiter, a fleet-level load shedder reserving capacity for critical operations, and a worker-utilization circuit breaker of last resort. Independently arriving at token bucket -- like AWS -- is a strong signal it's the industry default for this concern. ([Stripe blog](https://stripe.com/blog/rate-limiters))

Full sourcing: [research/backend/L1/14-api-gateway.md](../../../research/backend/L1/14-api-gateway.md#real-world-and-sources).

## Trade-offs

| Axis | ✅ Benefit | ❌ Cost |
|---|---|---|
| **Centralizing cross-cutting concerns** | Backends stay simple; consistent auth/rate-limit/logging everywhere | Every request now depends on the gateway |
| **Response aggregation** | Client makes one call instead of several | Extra fan-out latency; gateway does more per request |
| **The extra hop** | One place to enforce policy and observe traffic | Small latency tax (TLS + token + routing) on *every* call |
| **Single shared gateway** | Simplest to operate and observe | SPOF/bottleneck -- must be HA (back-ref topic 13) |
| **Overloading with logic** | -- | Business rules in the gateway = distributed monolith |

**Which gateway?** *Managed* (AWS API Gateway) removes ops burden but couples you to a vendor; *self-hosted* (Kong, Envoy-based) gives control at the cost of running it; *build-your-own* (Zuul) gives max fit at the highest cost -- most teams today reach for managed or open-source. **Single gateway vs per-client BFFs** (back-ref [09-rest-grpc-graphql.md](09-rest-grpc-graphql.md)): one shared gateway is simplest, but a mobile BFF and a web BFF can each shape responses for their client while sharing the common cross-cutting concerns.

## Remember

> [!IMPORTANT] Remember
> An API gateway is the **single front door** that does the API plumbing -- auth, rate limiting, routing, TLS, transformation, aggregation, observability -- **once**, so backends stay simple. It's a **reverse proxy specialized for API cross-cutting concerns**: unlike a load balancer (which distributes across *identical* replicas), its job is routing to *different* services and managing the client-facing API. It handles **north-south** traffic (client <-> system); a service mesh handles east-west. Keep it **thin** (no business logic) or it becomes a distributed-monolith chokepoint, and make it **highly available** -- it sits in the path of every single request.

## Check yourself

1. Why put auth and rate limiting in the gateway instead of implementing them in each microservice? What breaks if you do it per-service?
2. Both a load balancer and an API gateway sit in front of your backends. What's the difference in *purpose* -- what question does each answer?
3. Why is "the gateway calculates the order's final discount" a warning sign, even though the gateway is technically capable of running that logic?

---

→ Next: [CDN Internals](15-cdn-internals.md) (serving content from the edge: caching, invalidation, POP selection)
↩ Comes back in: CDN, service mesh (east-west), auth (L9), rate limiting & circuit breakers (reliability), microservices (architecture)
