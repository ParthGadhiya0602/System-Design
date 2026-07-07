# API Gateway: The Single Front Door for API Traffic

_A reverse proxy that stopped being generic: instead of "route and maybe cache," its whole job is everything an API client needs at the edge, so every backend service doesn't have to reimplement it._

`⏱️ ~9 min · 14 of 17 · L1 Networking`

## Contents

- [What an API gateway is and why it exists](#what-an-api-gateway-is-and-why-it-exists)
- [The functions an API gateway centralizes](#the-functions-an-api-gateway-centralizes)
- [How it works internally: the request pipeline](#how-it-works-internally-the-request-pipeline)
- [API gateway vs reverse proxy vs load balancer](#api-gateway-vs-reverse-proxy-vs-load-balancer)
- [Worked example: one request through the gateway](#worked-example-one-request-through-the-gateway)
- [BFF: one gateway for everyone, or one per client type](#bff-one-gateway-for-everyone-or-one-per-client-type)
- [North-south vs east-west traffic](#north-south-vs-east-west-traffic)
- [Trade-offs and risks](#trade-offs-and-risks)
- [How this connects to the rest of the stack](#how-this-connects-to-the-rest-of-the-stack)
- [Trade-offs and common confusions](#trade-offs-and-common-confusions)
- [Check yourself](#check-yourself)
- [Real-world and sources](#real-world-and-sources)

## What an API gateway is and why it exists

An **API gateway** is a single entry point that sits in front of a set of backend services and handles the **cross-cutting concerns** of API traffic — the things every API needs (auth, rate limiting, routing, logging) but that have nothing to do with any one service's actual business logic — so that individual services don't each have to reimplement them.

**Key vocabulary:**

- **Cross-cutting concern** — a piece of functionality that applies uniformly across many different services rather than belonging to any single one of them. Authentication, rate limiting, TLS, and logging are the canonical examples: every service needs them, none of them is what makes the "orders service" different from the "payments service." Left unmanaged, a cross-cutting concern gets copy-pasted (and inevitably re-implemented slightly differently, and eventually inconsistently) into every service that needs it.
- **Single entry point** — one well-known address that all external API clients talk to, regardless of how many backend services actually exist behind it or how those services are organized.

**The core motivation.** In a monolith, there's one process, so "who checks the API key" and "who logs the request" are just... code inside that one process. In a **microservices** architecture (forward-ref L11), there might be dozens or hundreds of independently deployed services. Without a gateway, an external client would need to know the network address of every individual service it wants to call, and every one of those services would need to independently implement authentication, rate limiting, TLS termination, and request logging — the same logic, duplicated N times, with N chances to get it wrong or drift out of sync. An API gateway centralizes all of that in one place: clients only ever talk to the gateway, and the gateway is the only thing backend services need to trust.

**Framing: an API gateway is a specialized reverse proxy.** [11-forward-and-reverse-proxies.md](11-forward-and-reverse-proxies.md) already introduced this exact idea as a forward reference: a reverse proxy fronts a group of servers and can act as a single choke point for cross-cutting concerns. [13-load-balancers.md](13-load-balancers.md) specialized that same underlying mechanism for *distributing load across an identical pool*. An API gateway specializes it again, in a different direction: instead of "spread traffic across copies of the same service," its job is "centralize everything an API client needs, and route to *different* services based on what's being asked for." Every API gateway is (mechanically) a reverse proxy; not every reverse proxy is an API gateway.

## The functions an API gateway centralizes

Each of these is a full topic elsewhere in this curriculum; here they're introduced only far enough to see *why* the gateway is the natural place to put them.

- **Request routing** — deciding which backend *service* a request goes to, based on path (`/orders/*` -> orders service), host, or API version (`/v1/*` vs `/v2/*`). This looks like L7 load balancing (topic 13) but at a different granularity: an L7 LB routes among *identical replicas of one service*; a gateway routes among *different services* entirely, then (often) hands off to a load balancer or does its own load balancing across that service's replicas once it knows which service to call.
- **Authentication and authorization** — validating an API key, JWT, or OAuth token at the edge, before a request ever reaches a backend, so backend services can simply trust that anything reaching them has already been authenticated (forward-ref L9 security for the actual token mechanics — signature verification, OAuth flows, token introspection).
- **Rate limiting, throttling, and quotas** — capping how many requests a given client (identified by API key, token, or IP) can make per second/minute/day, and rejecting the excess (typically `429 Too Many Requests`) before it ever burdens a backend (forward-ref L7/L8 for the actual algorithms — token bucket, sliding window, etc.).
- **TLS termination** — the gateway holds the certificate and completes the TLS handshake with the client (back-ref [07-https-tls.md](07-https-tls.md)), exactly as an L7 load balancer or reverse proxy does; traffic onward to backends may be re-encrypted or plain HTTP depending on network trust boundaries.
- **Request/response transformation** — rewriting headers, translating between protocols (e.g. exposing a REST facade over an internal gRPC service, back-ref [09-rest-grpc-graphql.md](09-rest-grpc-graphql.md)), and handling API versioning so old and new client versions can coexist against evolving backends.
- **Response aggregation / composition** — fanning a single incoming request out to multiple backend services and combining their results into one response, so the client makes one call instead of several. This is the same motivation behind the BFF pattern and GraphQL's single-schema aggregation from topic 9 — a gateway doing this is essentially hand-rolling a lightweight version of that idea at the routing layer.
- **Caching** — storing and serving repeat responses (e.g. for cacheable `GET` endpoints) directly at the gateway, so identical requests don't hit a backend at all (forward-ref CDN/caching, a later topic in this level and beyond).
- **Observability** — logging every request, emitting metrics (request rate, latency, error rate per route), and injecting/propagating a trace/request ID so a single request can be followed across every service it touches, even though the client never sees that internal fan-out (forward-ref L8 reliability/observability).
- **Load balancing** — once the gateway knows *which* service a request is for, it typically also balances across that service's healthy replicas (back-ref [13-load-balancers.md](13-load-balancers.md)) — the gateway either does this itself or delegates to a dedicated LB tier behind it.
- **Circuit breaking, retries, and timeouts** — protecting the whole system from a slow or failing backend by cutting it off temporarily, retrying transient failures with backoff, and bounding how long the gateway will wait on any one backend call (forward-ref L7 reliability, where the full mechanics live).
- **API management / developer portal** — issuing and managing API keys, defining usage plans/tiers, publishing documentation, and giving API consumers (often external, third-party developers) a self-service way to get access — the "Apigee-style" layer on top of the pure traffic-handling functions above.

## How it works internally: the request pipeline

Mechanically, a request passing through a gateway moves through a **pipeline of stages** (often literally implemented as middleware/plugins that run in a defined order), where each stage can inspect, modify, or reject the request before it reaches the next:

```
                         ┌─────────────── API Gateway ───────────────┐
                         │                                            │
Clients                  │  TLS        Auth       Rate      Routing   │       Backend microservices
(mobile, web,   ───────► │  terminate  (JWT/key)  limit     (path/    │ ────► ┌───────────┐
 partner API)            │                        check     version)  │       │ Orders     │
                         │                                            │  ┌──► │ service    │
                         │  Transform   Aggregate/   Observability    │  │    └───────────┘
                         │  (rewrite,   compose      (log, metrics,   │──┤    ┌───────────┐
                         │   version)   (fan-out)    trace ID)        │  ├──► │ Users      │
                         │                                            │  │    │ service    │
                         └────────────────────────────────────────────┘  │    └───────────┘
                                                                          │    ┌───────────┐
                                                                          └──► │ Shipping   │
                                                                               │ service    │
                                                                               └───────────┘

Contrast — the alternative, without a gateway:

Clients ──────────────────────────────────────────────────────────►  Orders service   (own auth, own rate limit, own TLS...)
        ──────────────────────────────────────────────────────────►  Users service    (own auth, own rate limit, own TLS...)
        ──────────────────────────────────────────────────────────►  Shipping service (own auth, own rate limit, own TLS...)

  clients must know every service's address, and every service duplicates the same cross-cutting logic
```

A request fails fast at whichever stage rejects it — an invalid token never reaches the routing stage, an over-quota client never reaches a backend at all — which is exactly the point: the more of these failures the gateway can catch before a backend does any work, the more backend capacity is protected for genuinely valid traffic.

## API gateway vs reverse proxy vs load balancer

This is the key distinction to nail, since topic 13 explicitly flagged that these terms overlap in practice. All three are, mechanically, **the same underlying pattern** — an L7-capable intermediary that terminates client connections and fronts a set of backends. What differs is **purpose and altitude**:

- A **reverse proxy** is the generic term: any intermediary that fronts one or more servers, on the servers' behalf, for *any* reason (relaying, shielding origin IPs, caching, TLS termination).
- A **load balancer** is a reverse proxy specialized for **distributing load across a pool of *identical* backend replicas**, using an algorithm and health checks (topic 13). Its defining question is "which of these interchangeable copies should handle this request?"
- An **API gateway** is a reverse proxy specialized for **API cross-cutting concerns and routing to *different* services**, plus client-facing API management. Its defining question is "what does this client need (auth, rate limiting, the right service, the right shape of response), and which of my several different backend services actually answers this request?"

They are routinely **layered**, not mutually exclusive: a common real deployment puts a load balancer in front of a fleet of gateway instances (so the gateway tier itself is horizontally scaled and highly available), and the gateway in turn performs its own load balancing across each backend service's replicas once it has decided which service to call. Many concrete tools (Envoy, Kong, nginx with the right plugins) can be configured to *be* all three at once, which is exactly why the terms blur in casual conversation — the distinction lives in configuration and intent, not in some fundamentally different piece of hardware.

### Required comparison

| | Reverse proxy | Load balancer | API gateway |
|---|---|---|---|
| **Core question answered** | "Relay/shield/cache on behalf of the server(s)?" | "Which identical replica handles this?" | "What does this API client need, and which *service* handles this?" |
| **Backend pool** | One or more servers, often just one | A pool of interchangeable replicas of the *same* service | Many *different* services, each possibly with its own replica pool |
| **Primary concerns** | Generic relaying, TLS termination, caching, hiding origin | Distribution algorithm, health checks, stickiness | Auth, rate limiting, routing by API contract, transformation, aggregation, API management |
| **Client-facing identity** | Usually invisible/transparent to the client | Usually invisible/transparent to the client | Explicitly the client's contract — versioned, documented, key-managed |
| **Relationship to the others** | The general category both others belong to | A specialization of reverse proxy | A specialization of reverse proxy; often sits with, or performs, load balancing itself |

## Worked example: one request through the gateway

A mobile client calls `GET /api/orders/123`, with header `Authorization: Bearer <JWT>` and `X-Api-Key: <key>`.

1. **TLS terminate.** The gateway completes the TLS handshake as the client's endpoint (back-ref topic 7) and decrypts the request. The backend services never see the raw TLS handshake at all.
2. **Validate the JWT.** The gateway checks the token's signature and expiry (forward-ref L9 for exactly how) *before* doing anything else. An invalid or expired token gets a `401` immediately — no backend is ever touched.
3. **Check the rate limit for this API key.** The gateway looks up this client's request count against its configured quota (forward-ref L7/L8 for the algorithm). If the client is over quota, it returns `429` here — again, no backend involved.
4. **Route to the orders service.** The gateway parses the path (`/api/orders/*`), matches it to the orders service, and (using its own load-balancing logic, or a dedicated LB behind it) picks a healthy orders-service replica to call.
5. **Optionally aggregate.** If this endpoint is designed to return a fuller picture, the gateway can *also* call the users service (for the customer's name) and the shipping service (for tracking status) in parallel, then combine all three results into one JSON response — the client only ever made one call.
6. **Add a trace ID.** The gateway generates (or propagates, if the client sent one) a request/trace ID and attaches it to every downstream call, so all three service calls triggered by this one request can be correlated later in logs/traces (forward-ref L8).
7. **Return the combined response** to the client, with the gateway itself, not any individual backend, having done TLS, auth, rate limiting, routing, aggregation, and tracing.

**What the backends therefore didn't have to do:** the orders, users, and shipping services never validated a JWT, never checked a rate limit, never terminated TLS, and never generated a trace ID — they simply received a plain, already-authenticated internal request and did their one job (look up an order, look up a user, look up a shipment). That's the entire value proposition of the gateway in one trace.

## BFF: one gateway for everyone, or one per client type

A **Backend for Frontend (BFF)**, briefly introduced in [09-rest-grpc-graphql.md](09-rest-grpc-graphql.md), is an edge layer tailored to a *specific* client type — mobile, web, or a third-party partner API — rather than one generic gateway serving every client identically.

- **One gateway for all clients** — simpler to operate (one thing to deploy, monitor, and secure), but every client gets the same shape of response, even though a mobile app might want a lean, aggregated payload (to save battery/bandwidth) while a web dashboard wants a richer, more granular one.
- **A BFF per client type** — a mobile BFF, a web BFF, and a partner-API BFF each sit behind (or as part of) the gateway tier, each shaping/aggregating responses specifically for that client's needs, while still relying on the shared gateway (or shared underlying services) for the common cross-cutting concerns like auth and rate limiting.

The two are not mutually exclusive: it's common to have one API gateway handling the truly universal cross-cutting concerns (TLS, auth, rate limiting) with per-client BFF layers behind it doing the client-specific shaping and aggregation — a layered specialization of the same idea, much like the LB-in-front-of-gateway pattern above. Deep BFF implementation patterns on the client side belong to the frontend track (forward-ref).

## North-south vs east-west traffic

- **North-south traffic** — traffic entering or leaving the system, between external clients and the services behind the edge. This is the API gateway's job: it is, by definition, a north-south component.
- **East-west traffic** — traffic between services *inside* the system, calling each other to fulfil a request (e.g. the orders service calling the inventory service internally). Managing *that* traffic's cross-cutting concerns (internal auth, retries, circuit breaking, encryption between services) is the job of a **service mesh** (forward-ref L11), a conceptually related but structurally different piece of infrastructure — a service mesh's sidecars sit next to *every* service instance, not just at one edge.

Keeping this distinction straight matters because it's a very common confusion: an API gateway does not, by itself, solve service-to-service traffic management inside the system — that's a different problem, addressed by a different piece of infrastructure, covered fully later.

## Trade-offs and risks

- **Simplification, at the cost of centralization risk.** The gateway is what makes the microservices world manageable from the outside — but everything now depends on it. If it goes down, *every* API client is affected, regardless of how healthy the backend services behind it are. This is the exact same single-point-of-failure problem load balancers face (back-ref topic 13's HA section), and the exact same fixes apply: redundant gateway instances behind a load balancer or VIP, health checks, and horizontal scaling of the gateway tier itself.
- **An extra network hop, and extra CPU per request.** Every request now pays the cost of TLS termination, token validation, and routing logic at the gateway *in addition to* whatever the backend itself does — a small but real latency tax (commonly single-digit to low tens of milliseconds depending on what the gateway does, `verify` for any specific implementation) on every single API call, and a throughput ceiling the gateway tier itself must be sized to handle.
- **The temptation to overload it with business logic.** A gateway should stay thin: routing, policy, and cross-cutting concerns — not domain logic like "calculate this order's discount" or "decide which service owns this data." Once business rules creep into gateway configuration, the gateway becomes a hidden, hard-to-test, hard-to-deploy-independently piece of every service's actual behavior — effectively turning a set of independently deployable microservices back into a **distributed monolith**, where nothing can actually be changed or deployed independently because everything funnels through one shared, business-logic-laden chokepoint. Keeping the gateway's job strictly to routing/policy is what preserves the independence microservices are supposed to provide in the first place.
- **Single gateway vs BFFs vs micro-gateways.** A single shared gateway is simplest to run but can become a bottleneck for both traffic *and* team coordination (every team's routing/policy changes go through one shared config). Per-client BFFs (above) address the client-shaping problem. **Micro-gateways** — a lighter-weight gateway instance per service or per team, rather than one giant shared gateway — trade a little duplicated infrastructure for team-level independence, at the cost of losing some of the "one place to see everything" simplicity of a single gateway.
- **Managed vs self-hosted vs build-your-own.** A fully managed service (e.g. AWS API Gateway) removes essentially all operational burden but ties routing/policy configuration to that vendor's model and pricing. A self-hosted, open-source gateway (e.g. Kong, or a gateway built on Envoy) gives full control and portability at the cost of running and scaling it yourself. Building a bespoke in-house gateway (the historical example being Netflix's Zuul) gives maximum flexibility for a company's specific scale/routing needs, at the highest engineering and maintenance cost — most organizations today reach for an existing managed or open-source gateway rather than building one from scratch, `verify` current industry default.

## How this connects to the rest of the stack

- **Back to [11-forward-and-reverse-proxies.md](11-forward-and-reverse-proxies.md)** — an API gateway is, mechanically, a reverse proxy specialized for API cross-cutting concerns; connection termination and header handling from that topic apply directly.
- **Back to [13-load-balancers.md](13-load-balancers.md)** — a gateway performs L7-style routing and often load balancing across each service's replicas once it has decided which service to call; the two are frequently layered (LB in front of a gateway tier, or gateway doing its own LB behind).
- **Back to [07-https-tls.md](07-https-tls.md)** — TLS termination at the gateway is the exact same mechanism already covered.
- **Back to [09-rest-grpc-graphql.md](09-rest-grpc-graphql.md)** — response aggregation and the BFF pattern connect directly to GraphQL's single-schema aggregation and the diverse-client motivation covered there; a gateway can implement a lightweight version of the same idea even without GraphQL.
- **Forward to authentication/authorization internals (L9 security)** — this topic covers *that* the gateway validates tokens at the edge; the actual OAuth/OIDC/JWT mechanics live there.
- **Forward to rate limiting algorithms and circuit breaking (L7/L8 reliability)** — this topic covers *that* the gateway enforces quotas and can circuit-break a failing backend; the algorithms and failure-handling patterns are developed fully there.
- **Forward to observability (L8)** — request logging, metrics, and distributed tracing, briefly introduced here, get their full treatment there.
- **Forward to service mesh (L11)** — the east-west counterpart to the gateway's north-south job; a structurally different piece of infrastructure for service-to-service traffic.
- **Forward to microservices architecture and composition (L11)** — the gateway is a core piece of infrastructure that makes a microservices architecture usable from the outside.
- **Forward to CDNs (a later topic in this level)** — a CDN can sit in front of the gateway entirely, caching and serving some responses at the network edge before a request ever reaches the gateway at all.

## Trade-offs and common confusions

**Gateway vs load balancer vs reverse proxy, restated in one line each.** Reverse proxy = generic intermediary; load balancer = reverse proxy specialized for distributing load across *identical* replicas; API gateway = reverse proxy specialized for API cross-cutting concerns and routing across *different* services. All three are commonly layered together, and many real tools implement all three at once.

**North-south vs east-west.** The gateway manages traffic entering/leaving the system (north-south); it does not, by itself, manage service-to-service traffic inside the system (east-west) — that's the service mesh's job.

**Keep the gateway thin.** Routing and policy belong in the gateway; business/domain logic does not. The moment business rules live in gateway configuration, independently deployable microservices risk collapsing back into a distributed monolith.

**Single gateway vs BFFs vs micro-gateways.** A single shared gateway is simplest; per-client BFFs solve response-shaping for diverse clients; micro-gateways trade some duplication for team-level independence — these are complementary layering choices, not mutually exclusive alternatives.

**Managed vs self-hosted vs build-your-own.** A managed service removes operational burden but couples you to a vendor; an open-source self-hosted gateway gives control at the cost of running it; building your own gives maximum fit at the highest cost — most teams today default to managed or open-source rather than building from scratch.

**The gateway as SPOF/bottleneck.** Exactly like a load balancer, an API gateway sits directly in every request's path and must itself be made highly available (redundant instances, health checks, horizontal scaling) or it becomes the very single point of failure the microservices architecture behind it was trying to avoid.

| | Benefit | Cost |
|---|---|---|
| Centralizing cross-cutting concerns at the gateway | Backends stay simple, consistent auth/rate-limiting/logging everywhere | Gateway becomes a critical dependency every request relies on |
| Response aggregation at the gateway | Client makes one call instead of several | Extra fan-out latency, gateway does more per-request work |
| Single shared gateway | Simple to operate and observe | Can bottleneck traffic and team coordination |
| Per-client BFFs / micro-gateways | Tailored responses, team-level independence | More infrastructure, more to keep consistent |
| Managed gateway (e.g. AWS API Gateway) | Zero operational burden | Vendor coupling, less low-level control |
| Self-hosted gateway (e.g. Kong, Envoy-based) | Full control, portable | You run, scale, and patch it yourself |

> [!IMPORTANT]
> An API gateway is a reverse proxy specialized for API cross-cutting concerns — auth, rate limiting, TLS termination, routing to different services, transformation, aggregation, and observability — centralizing them so backend services don't each reimplement them. It differs from a load balancer (which distributes load across *identical* replicas) and from a generic reverse proxy (the umbrella category) by *purpose*: its defining question is what a client needs and which different service answers a request, not which interchangeable copy should. It handles north-south traffic (client-to-system); a service mesh handles east-west traffic (service-to-service) separately. Keep it thin — routing and policy, never business logic — and make it highly available, because it sits in the path of every single request the system serves.

## Check yourself

- A team has a single service and puts a reverse proxy in front of it purely to terminate TLS and hide the origin IP. Is that an API gateway? Why or why not?
- Walk through the mobile `GET /api/orders/123` example: at which pipeline stage does an invalid JWT get rejected, and why does that ordering matter for protecting backend capacity?
- Why is "the gateway calculates the order's final discount" a warning sign, even though the gateway is technically capable of running that logic?
- A company serves both a mobile app and a rich internal admin dashboard from the same underlying services. What's the trade-off between giving them one shared gateway response shape vs a BFF each?
- Why doesn't an API gateway, by itself, solve the problem of one internal microservice needing to call another securely and resiliently? What piece of infrastructure does?

## Real-world and sources

**Netflix Zuul — the canonical build-your-own edge gateway.** Zuul is Netflix's own edge service, the "front door" for all requests from every device (TV, mobile, browser) into the Netflix backend; Netflix's own announcement post states it handles over 50,000 requests per second at peak, across more than 1,000 device types. Mechanically it is exactly the "pipeline of stages" described above, implemented as **filters** with four types: `PRE` filters (authenticate, choose the origin server, log debug info) run before routing; `ROUTING` filters build and send the actual origin request; `POST` filters (add response headers, gather metrics, stream the response back) run after; and `ERROR` filters handle failures at any stage. Filters are dynamically compiled and hot-loaded at runtime (originally in Groovy) without restarting the server, which is what let Netflix reconfigure routing, canary a new region, or shed load without a deploy. This ties directly to two concepts above: it is the historical **build-your-own** point of the managed/self-hosted/build-your-own trade-off, and its filter chain is a concrete, shipped instance of the request pipeline (TLS/auth/routing/observability as ordered stages) this file describes abstractly. *(Netflix Technology Blog, "Announcing Zuul: Edge Service in the Cloud"; Netflix/zuul GitHub wiki — accessed 2026-07-07.)*

**AWS API Gateway — the managed-service articulation of auth + throttling + stages.** AWS's official docs describe throttling implemented with the same **token bucket algorithm** taught generically in this curriculum's rate-limiting forward-ref: a token is consumed per request, tokens refill at a steady-state rate, and a burst limit caps the bucket size; exceeding it returns `429 Too Many Requests` — the exact status code this file's worked example uses. Throttling can be layered at four levels (AWS account-wide per-Region limits, per-account limits, per-API/per-stage limits, and per-client limits tied to API keys via a "usage plan"), and AWS explicitly recommends **not** using API keys alone for auth/authorization — instead pairing the gateway with IAM roles, a Lambda authorizer, or a Cognito user pool. This is a direct, concrete instance of two things this file teaches abstractly: "stages" as a first-class gateway concept (dev/staging/prod each get independent throttle settings), and the managed end of the managed-vs-self-hosted-vs-build-your-own spectrum, where auth and rate limiting are configured, not coded. *(AWS documentation, "Throttle requests to your REST APIs for better throughput in API Gateway," docs.aws.amazon.com/apigateway — accessed 2026-07-07.)*

**Stripe — a fintech gateway-adjacent rate-limiting architecture.** Stripe's own engineering blog describes a layered defense at the edge of their API, also built on a **token bucket** (a centralized bucket per API key/user; a request takes a token, tokens drip back in over time, an empty bucket means the request is rejected) — the same algorithm as AWS API Gateway's, independently arrived at, which is a useful signal that token bucket is close to an industry default for this exact cross-cutting concern. Stripe layers four limiters rather than one: a per-second request-rate limiter (their most frequently triggered, rejecting "millions of requests" in a given month, per Stripe's own figures), a concurrent-in-flight-requests limiter (roughly 12,000 rejections/month), a fleet-level load shedder that reserves a percentage of infrastructure for critical operations, and a worker-utilization load shedder as a last-resort circuit breaker (only ~100 rejections/month, reserved for genuine incidents). This maps directly onto this file's "circuit breaking, retries, and timeouts" bullet and shows a real system layering *multiple* cross-cutting defenses at the edge rather than a single flat rate limit, exactly the kind of nuance the abstract "rate limiting" bullet above gestures at but defers to L7/L8. *(Stripe engineering blog, "Scaling your API with rate limiters," stripe.com/blog/rate-limiters — accessed 2026-07-07.)*

### Sources / further reading

- Netflix Technology Blog — ["Announcing Zuul: Edge Service in the Cloud"](https://netflixtechblog.com/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee) (accessed 2026-07-07)
- [Netflix/zuul GitHub wiki](https://github.com/Netflix/zuul/wiki) — filter types and architecture reference (accessed 2026-07-07)
- AWS documentation — ["Throttle requests to your REST APIs for better throughput in API Gateway"](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) (accessed 2026-07-07)
- Stripe engineering blog — ["Scaling your API with rate limiters"](https://stripe.com/blog/rate-limiters) (accessed 2026-07-07)
