# Forward and Reverse Proxies

*You've mastered the direct client-to-server connection -- sockets, TCP, TLS. Now meet the deliberate third party you slide in between: the proxy. Everything interesting about it reduces to one question -- which side does it front?*

`⏱️ ~7 min · 11 of 17 · Networking`

> [!TIP] The gist
> A **proxy** is an intermediary that relays requests and responses on behalf of one side. It's not a passive wire -- it **terminates one connection and opens a second**, so it can read, cache, modify, route, authenticate, or reject traffic. The whole distinction is *which side it fronts*: a **forward proxy** fronts the **client** (hides the client -- the destination sees the proxy's IP), a **reverse proxy** fronts the **servers** (hides the backends -- the client only ever talks to the proxy). The reverse proxy is the big one: it's the single choke point at the edge where you centralize TLS, load balancing, caching, routing, auth, and rate limiting so backends stay simple. That's exactly why **load balancers, API gateways, and CDNs are all specialized reverse proxies.**

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Two everyday people capture the whole topic.

A **forward proxy is a personal assistant** who makes calls *on your behalf*. You tell them who to call; they dial out; the person on the other end sees the assistant's number and name, not yours. The assistant **fronts you and hides you** from the outside world. *You* hired them, *you* configured them -- they serve your interests (your privacy, your rules).

A **reverse proxy is a company receptionist** who takes *every* incoming call and routes it to the right employee. Callers dial one published number and never reach an employee directly -- they don't even know which person (or how many) sit behind the desk. The receptionist **fronts the employees and hides them**. The *company* set this up -- it serves the employees' interests (their focus, their security).

Same job -- an intermediary relaying the conversation -- but the assistant hides the *caller* and the receptionist hides the *callees*. That single flip is the entire difference between forward and reverse.

## The concept

**Definition.** A **proxy** (same root as a "proxy vote" -- acting *on behalf of* someone) is software or an appliance that sits **between a client and a server** and relays the conversation, so both sides talk to the proxy instead of to each other directly.

**It is not a passive wire.** This is the non-negotiable mechanical fact: a proxy is a genuine endpoint of **two separate connections**. It **terminates** the first TCP connection (completes the handshake, becomes a real endpoint with its own socket and TCP state) and **originates a second, entirely separate** connection to the other side. Nothing runs "through" it the way a signal runs through a cable -- it copies (or transforms) data from one connection to the other at the application level. That's precisely what lets it read a full HTTP request, decide what to do, and only then choose what to send onward: cache it, log it, modify it, block it, or route it elsewhere.

**The one distinction that matters -- which side it fronts:**

- A **forward proxy** fronts the **client**. It's configured *by* the client side and **hides the client** from the server. Serves the client's interests.
- A **reverse proxy** fronts the **server(s)**. It's set up *by* the server operator and **hides the backends** from the client. Serves the servers' interests -- even though it's the client that talks to it.

**What it is NOT.** Not a router (a router forwards IP packets at L3 by destination address without terminating any connection). Not a fixed technology -- "proxy" is a *role*; the same software (nginx, HAProxy, Envoy) can play either role, or both at once. And forward ≠ reverse -- mixing these up is the #1 confusion in the whole topic.

**Key terms:** intermediary, terminate/re-originate, forward proxy, reverse proxy, backend/upstream pool, TLS termination, L4 vs L7, `X-Forwarded-For`.

## How it works

### Forward proxy -- fronting the client

Sits in front of a group of **clients**, representing them to the internet. The client is configured (explicitly, or transparently intercepted on its network) to send outbound traffic through the proxy. From the destination's view, the request came from the **proxy's IP** -- it never sees the real client.

```
[Client A] \
[Client B] --->  Forward Proxy  --->  Internet  --->  [Destination Server]
[Client C] /       (client's IP hidden; server sees the proxy)
```

The clients know about it; the destination usually doesn't. Typical uses:

- **Corporate egress control / content filtering** -- route all employee traffic through one choke point to enforce policy, block sites, and log access.
- **Caching for a group of clients** -- many clients behind one proxy share cached copies of popular resources.
- **Anonymity / privacy** -- the destination only ever sees the proxy's IP.
- **Geo-bypass** -- routing through a proxy in another region makes the request appear to originate there (the mechanism behind a VPN).

### Reverse proxy -- fronting the servers

Sits in front of a **backend pool**, representing them to clients. The client sends its request to what it believes is "the server" -- no client-side config is required or even possible -- and the reverse proxy decides which backend handles it, then relays the response. The backends often aren't even reachable directly from the internet; the proxy is the only public door.

```
[Client X] \                        /---> [Backend 1]
[Client Y] --->  Reverse Proxy  ---+----> [Backend 2]
[Client Z] /   (backends' IPs      \---> [Backend 3]
                hidden; client sees only the proxy)
```

**The centralization win:** because *every* request funnels through this one point, it's where you put every cross-cutting concern -- so each backend can stay simple and just do business logic:

- **TLS termination** -- the proxy holds the cert, terminates HTTPS, speaks plain HTTP onward. No per-backend certificate management.
- **Load balancing** -- spread requests across instances with health checks.
- **Caching** -- answer repeat requests without touching a backend.
- **Routing** -- send `/api/*`, `/static/*`, `/checkout/*` to different backends from one hostname.
- **Auth + rate limiting** -- validate a token / throttle a client *once*, before any backend sees the request.
- **Security shielding** -- hide backend topology entirely; absorb and filter malicious traffic.

**The side-by-side rule -- ask "which side is hidden, and which side configured anything?"**

| | Forward proxy | Reverse proxy |
|---|---|---|
| **Sits in front of** | The client(s) | The server(s) |
| **Who configures it** | The client (or its network admin) | The server operator |
| **Whose IP is hidden** | The client's (server sees the proxy) | The backends' (client sees the proxy) |
| **Who it serves** | The client | The servers |
| **Does the client know it's there?** | Yes -- configured, or transparently intercepted | Usually no -- believes it talks to "the server" |
| **Does the server know it's there?** | Usually no -- sees only the proxy as "the client" | Yes -- backends are deliberately deployed behind it |
| **Typical uses** | Filtering, egress policy, anonymity, geo-bypass | TLS termination, load balancing, caching, routing, shielding |
| **Canonical software** | Squid, corporate web proxies, VPN clients | nginx, HAProxy, Envoy, Cloudflare edge |

Same mechanism both times -- terminate one connection, originate another. Only **which side it fronts** flips.

### Under the hood

**Two connections, always.** The proxy terminates the client-facing connection and re-originates a separate backend-facing one. The real client's IP lived only in that first, now-closed connection -- so the backend natively sees only the *proxy's* IP as the source.

**L4 vs L7** (back to [01-osi-and-tcp-ip-models.md](01-osi-and-tcp-ip-models.md)):

- An **L4 proxy** works at TCP/UDP level -- it sees IPs, ports, raw bytes, and can even do **TCP passthrough** (relay encrypted bytes without decrypting). Fast and protocol-agnostic, but blind to HTTP: it can't read a path, a Host header, or a cookie.
- An **L7 proxy** parses the HTTP request -- so it can route by path, rewrite headers, terminate TLS, and cache by semantics. Full visibility and control, at the cost of CPU (parsing, often decrypting) and being tied to the specific protocol.

**Restoring the client's identity -- `X-Forwarded-For`.** Because the backend only sees the proxy's IP, L7 reverse proxies add headers to the forwarded request: **`X-Forwarded-For`** (the original client IP) and **`X-Forwarded-Proto`** (`http` vs `https`, since the backend may now get plain HTTP even though the client used HTTPS). Without them, a backend logs *every* request as coming from one IP -- breaking per-client rate limiting, geolocation, and audit logs. The catch: `X-Forwarded-For` is just an ordinary header, so a client talking to a misconfigured proxy can **spoof** it. A correct proxy overwrites it, and backends must only trust values from a known proxy hop.

**One request, traced** (`GET https://api.example.com/orders/42`, three backends behind nginx):

1. **DNS** resolves `api.example.com` to the *proxy's* public IP -- backends may have no public IP at all.
2. **Connection 1 (client ↔ proxy):** TCP + TLS handshake; the proxy presents the cert and **decrypts** the request.
3. **L7 decision:** reads path `/orders/42`, matches a routing rule, picks a healthy backend (say Backend 2) from the pool.
4. **Rewrite:** adds `X-Forwarded-For: <client IP>` and `X-Forwarded-Proto: https`.
5. **Connection 2 (proxy ↔ Backend 2):** a *separate* TCP connection (often pooled from a prior request) carries the rewritten request.
6. **Backend 2 responds** -- it sees the proxy's internal IP as the source, with the real client only in the header.
7. **The proxy relays (and maybe caches)** the response back over Connection 1, re-encrypting it -- as if the proxy itself had answered.

## In the real world

**Cloudflare -- reverse proxy as security shield at internet scale.** Cloudflare describes itself in exactly these terms: "a reverse proxy is a network of servers that sits in front of web servers and either forwards requests to those web servers, or handles requests on behalf of the web servers." It **terminates TLS at the edge** ("decrypt all incoming requests and encrypt all outgoing responses, freeing up valuable resources on the origin server") and **hides origin IPs** ("a web site or service never needs to reveal the IP address of their origin servers") -- the same "backend topology is invisible to the client" property, deployed in front of millions of unrelated origins, with anycast absorbing DDoS traffic before it reaches them.

**nginx -- the canonical L7 reverse proxy that a load balancer specializes.** nginx's own docs frame reverse-proxy uses as "distribute the load among several servers, seamlessly show content from different websites, or pass requests for processing to application servers over protocols other than HTTP" -- i.e. **load balancing, aggregation, and protocol translation** as core use cases, not add-ons. That directly confirms: a load balancer *is*, mechanically, a reverse proxy specialized for one of its jobs.

**Envoy (built at Lyft) -- the reverse-proxy pattern on every service hop.** Lyft built Envoy as an **edge reverse proxy** for inbound traffic, then -- because the same terminate-one/originate-another mechanism works just as well between internal services -- redeployed it as a **sidecar proxy** next to every service instance, forming a mesh across "over a hundred services" transiting millions of requests per second. The reverse-proxy choke point, pushed down to every service-to-service call, not just the public edge. Envoy is now the de facto service-mesh data plane.

Full sourcing (Cloudflare, nginx, Lyft/Envoy docs): [research/backend/L1/11-forward-and-reverse-proxies.md](../../../research/backend/L1/11-forward-and-reverse-proxies.md#real-world-and-sources).

## Trade-offs

| Axis | ✅ Benefit | ❌ Cost |
|---|---|---|
| **Centralizing TLS / caching / auth / routing** (reverse proxy) | Backends stay simple; one place to patch, secure, scale; shields backend topology | Adds a hop (small latency); becomes a bottleneck / SPOF if not run redundantly |
| **L7 (HTTP-aware) proxying** | Full routing, rewriting, caching, auth control | CPU cost of parsing + often decrypting; protocol-specific |
| **L4 (passthrough) proxying** | Fast, protocol-agnostic, preserves end-to-end encryption | No visibility into or control over application content |
| **`X-Forwarded-For`** | Backends regain per-client logging / rate-limiting / geo | Spoofable if the proxy doesn't overwrite it and backends blindly trust it |

**The SPOF is mitigated the usual way** -- run the proxy *tier* itself redundantly (multiple instances behind DNS round-robin or anycast). The centralization is worth the hop.

**Forward vs reverse, restated crisply:** if the *client* set it up and the *server* only sees the proxy, it's **forward**. If the *server operator* set it up and the *client* only sees the proxy, it's **reverse**.

**Overlapping terms in one line:** **NAT** rewrites addresses in-place at L3/L4 *without* terminating a connection (same packets, new address fields) -- a different mechanism for a similar "hide the internal address" outcome. A **load balancer** is a reverse proxy specialized to distribute load. A **gateway** is a loose term for an entry/exit point -- an *API* gateway happens to be a reverse proxy.

## Remember

> [!IMPORTANT] Remember
> Same mechanism -- an intermediary that **terminates one connection and originates a second** -- but a **forward proxy fronts and hides the CLIENT** while a **reverse proxy fronts and hides the SERVERS**. The reverse proxy is the **choke point** where you centralize TLS, caching, auth, routing, and rate limiting so backends stay simple -- which is exactly why **load balancers, API gateways, and CDNs are all specialized reverse proxies.** What differs between them is the *policy* applied once you're sitting in that position, not the proxy mechanism.

## Check yourself

1. Your company wants to **filter and monitor all employee web traffic** through one point. Forward or reverse proxy -- and whose IP does the destination website see?
2. Why does putting a **reverse proxy in front of your backends** let the backends stay simple? Name two cross-cutting concerns it absorbs.
3. A backend **logs every request as coming from one single IP**. What's happening, in terms of TCP connections -- and which header fixes it (plus the gotcha with trusting that header)?

---

→ Next: [NAT](12-nat.md) (sharing one public IP across many private hosts -- addresses rewritten in place, no connection termination)
↩ Comes back in: load balancers, API gateway, CDN (all specialized reverse proxies), service mesh (sidecar proxies), security (WAF + the `X-Forwarded-For` trust problem)
