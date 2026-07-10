# Request Lifecycle End-to-End

Follow one request from a user's click to the pixels on screen, naming every hop it passes through and what each hop costs, can break, and can add to scale.

## Contents

- [Overview and the mental model](#overview-and-the-mental-model)
- [Hop 0: the client and browser](#hop-0-the-client-and-browser)
- [Hop 1: DNS resolution](#hop-1-dns-resolution)
- [Hop 2: connection setup (TCP and TLS)](#hop-2-connection-setup-tcp-and-tls)
- [Hop 3: CDN and the edge](#hop-3-cdn-and-the-edge)
- [Hop 4: the load balancer](#hop-4-the-load-balancer)
- [Hop 5: the application server](#hop-5-the-application-server)
- [Hop 6: cache lookup (the read path)](#hop-6-cache-lookup-the-read-path)
- [Hop 7: the database (on a cache miss)](#hop-7-the-database-on-a-cache-miss)
- [Hop 8: async offload](#hop-8-async-offload)
- [Hop 9: the response travels back](#hop-9-the-response-travels-back)
- [How each hop connects onward](#how-each-hop-connects-onward)
- [Check yourself](#check-yourself)

## Overview and the mental model

Trace a single, concrete request: a user opens `https://app.example.com/feed` in a browser. What looks instant to them is a chain of a dozen or so **hops** - discrete stops where some component receives the request, does a little work, and passes it along. This topic walks that chain end to end. Deep protocol mechanics (how DNS packets are formed, how TCP guarantees delivery, how a load balancer hashes) belong to later levels; here we build the _map_ so you can reason about where a system spends its time, where it fails, and where it grows.

Three questions to ask at **every** hop - this is the whole mental model:

1. **Latency added.** Every hop costs time: some to do work, and often a **round trip** - one message out to a component and one back - which is bounded by the speed of light and the physical distance. A round trip within one data center is ~0.5 ms; across a continent ~40 ms; across the planet ~150 ms. Hops that require a round trip to a far-away machine dominate; hops served from local memory are nearly free.
2. **Failure point.** Every hop is something that can be slow, return an error, or be completely down. A request's reliability is the product of every hop working. More hops means more ways to fail, which is why later topics obsess over retries, timeouts, and fallbacks _at each hop_.
3. **Scaling knob.** Every hop is a place you can insert or duplicate a component to handle more load: add a cache, add servers behind a balancer, push content to the edge. Knowing the hops tells you where the levers are.

The single most useful rule of thumb: **end-to-end latency at p99 is roughly the sum of the slow hops on the critical path.** (p99, the "99th percentile," is the latency below which 99% of requests finish; it is what your slowest 1% of users actually feel, and it is usually far worse than the average.) The **critical path** is the sequence of hops that must complete _before the response can be sent_ - anything done in parallel or offloaded (see Hop 8) does not count. So to make a system faster you find the slowest hop on that path and attack it; shaving a fast hop changes nothing. Keep this additive picture in mind as we walk the chain.

## Hop 0: the client and browser

Before any network traffic, the request begins inside the client. The browser **parses the URL** `https://app.example.com/feed` into its parts: the scheme (`https`, meaning use TLS), the host (`app.example.com`, a name that must be turned into an IP address), and the path (`/feed`, what to ask for). Then, critically, it checks its **local caches** before touching the network:

- The **HTTP cache** stores responses the server previously said were reusable (via caching headers). If a fresh copy of `/feed` or its assets is already cached locally, the browser serves it with **zero** network cost - ~0 ms.
- The **DNS cache** stores recent name-to-IP answers so the browser can skip Hop 1 entirely.
- Connection reuse: a still-open connection to the host lets it skip Hop 2.

- **Latency:** cache hits are effectively free (~0 ms); a miss means the full chain below.
- **Failure:** stale cached data can show the user something out of date; a corrupted local cache can wedge the app.
- **Scaling knob:** correct caching headers push work off your servers entirely. The guiding principle of the whole chain: **the cheapest request is the one you never make.** Every hop you can legitimately skip with a cache is latency, load, and failure surface removed for free.

## Hop 1: DNS resolution

The browser has a _name_ (`app.example.com`) but the network routes by _number_ - an **IP address** like `93.184.216.34`. **DNS** (the Domain Name System) is the distributed lookup that translates one to the other. The client asks a **resolver** (a DNS server, usually run by the ISP or a public provider); the resolver either has the answer cached or walks the DNS hierarchy to find the servers authoritative for `example.com` and returns the IP.

Two ideas matter at composition level:

- **Caching and TTL.** Every DNS answer carries a **TTL** (time-to-live) - how long it may be cached before it must be looked up again. High TTLs make lookups nearly free (served from cache) but mean changes (like moving to a new IP) take that long to propagate. This is a direct availability-vs-agility trade.
- **GeoDNS steering.** DNS can return _different_ IPs to different users based on their location, handing each user the address of the nearest data center or CDN edge. This is the first place a global system steers traffic geographically.

- **Latency:** a cached answer is ~0 ms; an uncached full resolution can cost ~20-120 ms of round trips - a surprisingly heavy first-request tax.
- **Failure:** if DNS is misconfigured or down, the site is unreachable _even though every server is healthy_ - the client never learns where to go. DNS outages are among the most total.
- **Scaling knob:** GeoDNS and multiple resolvers spread and localize lookups; the deep mechanics are the Networking level.

## Hop 2: connection setup (TCP and TLS)

Now the client has an IP and must open a **connection** before sending the request. For `https` this is two handshakes stacked:

- The **TCP handshake** establishes a reliable byte pipe to the server. It costs one round trip (the classic "SYN, SYN-ACK, ACK" exchange) before any data can flow.
- The **TLS handshake** then negotiates encryption keys so the connection is private and the server's identity is verified. It historically costs one to two more round trips.

So a brand-new secure connection can cost **2-3 round trips before the request is even sent** - ~80-150 ms across a continent, and this is pure overhead, not work. This is exactly why **connection reuse** matters so much:

- **Keep-alive** holds a connection open after a response so the _next_ request skips both handshakes.
- Modern protocols multiplex many requests over one connection and cut handshake round trips further.

- **Latency:** first connection is expensive (round trips); reused connections are near-free for setup.
- **Failure:** handshakes fail on expired certificates, protocol mismatches, or packet loss - and they fail _before_ your application logic ever runs.
- **Scaling knob:** connection pooling and keep-alive amortize setup cost across many requests. Terminating TLS at the edge or load balancer (below) moves this cost off your app servers. Deep protocol detail is the Networking level.

## Hop 3: CDN and the edge

A **CDN** (Content Delivery Network) is a fleet of servers placed in many locations worldwide, close to users. The request (often steered here by GeoDNS in Hop 1) first lands on the nearest **edge** server. The CDN's job is to serve **static or cacheable content** - images, scripts, stylesheets, fonts, and increasingly cacheable API responses and whole pages - from that nearby edge instead of your distant origin.

The defining event is **hit vs miss**:

- **Cache hit:** the edge already has the content and returns it immediately. The request never travels to your origin. Latency is just the short hop to the nearby edge - ~5-30 ms.
- **Cache miss:** the edge does not have it (or it is expired), so the edge forwards the request to your **origin** (your load balancer and servers), caches the answer for next time, and returns it. This request pays the full long-distance trip plus the edge overhead.

- **Latency:** hits are fast and local; misses add the origin round trip on top.
- **Failure:** a CDN outage can block traffic to an otherwise healthy origin; a bad cache configuration can serve stale or wrong content globally.
- **Scaling knob:** the CDN absorbs the vast majority of read traffic for static content, shielding your origin from load - often the single biggest offload in the whole chain. Deep CDN behavior is the Networking and Caching levels.

## Hop 4: the load balancer

For anything the CDN cannot serve, the request reaches your **origin**, and its public front door is the **load balancer** (LB). This is the single logical address the client (or CDN) talks to; behind it sit many application servers the client never sees. The LB's core job is to **spread incoming requests across the healthy servers**, and to stop sending traffic to any server that fails its health check.

One line on the two flavors, deferred in depth to the Load Balancing level:

- An **L4 (transport-layer) load balancer** routes by IP and port without looking at the request contents - fast and simple.
- An **L7 (application-layer) load balancer** reads the request (path, headers) and can route by content, e.g. send `/feed` to one pool and `/search` to another.

The LB is also the natural place for **TLS termination**: it completes the TLS handshake from Hop 2 so the app servers behind it can handle plain, already-decrypted traffic and not each carry that cost.

- **Latency:** a well-run LB adds little (~1-5 ms) but is one more round trip in the path.
- **Failure:** the LB is a potential single point of failure - if it is down, everything behind it is unreachable, so in practice it is itself made redundant.
- **Scaling knob:** the LB is _the_ mechanism that turns many interchangeable stateless servers into one scalable service; adding a server is just registering it with the LB.

## Hop 5: the application server

The LB hands the request to one **application server** - the process that runs your actual business logic. It parses the request, checks who is asking (authentication) and whether they may do it (authorization), validates input, and orchestrates the work needed to build the `/feed` response: reading data, applying rules, assembling a result.

The property that makes this hop scale is **statelessness**: the server keeps no per-user memory between requests, so _any_ server in the pool can handle _any_ request. Session data lives in a shared store or in a token the client presents. Because servers are interchangeable, the LB can freely spread load and you can add or remove servers at will. (This is the causal chain from the client-server topic: stateless servers behind a logical address are what the load balancer exploits.)

- **Latency:** the CPU work itself is often small (~1-10 ms), but this server usually _waits_ on the hops below it (cache, database, other services) - its total time is dominated by what it calls, not what it computes.
- **Failure:** bugs, crashes, memory exhaustion, or slow downstream calls piling up all surface here; one overloaded server is why health checks and timeouts exist.
- **Scaling knob:** because it is stateless, you scale it **horizontally** - add more identical instances behind the LB. This is the workhorse scaling move of the whole system.

## Hop 6: cache lookup (the read path)

To build `/feed` the app server needs data. Before hitting the database, it checks a **cache** - a fast, usually in-memory store holding recently or frequently used data. This is the **read path**, and its defining event, again, is hit vs miss:

- **Cache hit:** the data is present and returned in ~0.5-2 ms from memory. The database is never touched.
- **Cache miss:** the data is absent, so the server falls through to the database (Hop 7), then typically writes the result into the cache so the _next_ request hits.

Because memory is orders of magnitude faster than disk-backed database queries, and because most read-heavy systems ask for the same popular data over and over, the cache absorbs the bulk of reads:

- **Latency:** hits are near-free; the value is skipping the slower database hop.
- **Failure:** a stale cache can serve outdated data (the consistency trade); a cache outage can cause a **thundering herd** where every request suddenly falls through to the database at once and overwhelms it.
- **Scaling knob:** this is the **number-one lever for read-heavy systems.** Raising the cache hit rate directly cuts database load and tail latency. Cache strategies, eviction, and invalidation are the Caching level.

## Hop 7: the database (on a cache miss)

On a miss, the request reaches the **database** - the durable **source of truth**, the authoritative, persisted copy of the data that survives restarts and crashes. This is where correctness ultimately lives, and it is typically the slowest, most contended hop, because it must read from (or write to) durable storage and enforce consistency.

Two composition-level ideas:

- **Index vs scan.** An **index** is a precomputed lookup structure that lets the database jump straight to the rows it needs - turning a query from a **full scan** (reading every row, cost grows with table size) into a fast direct lookup (~1-10 ms). A missing index is one of the most common reasons a query, and thus the whole request, is slow. Index internals are the Databases level.
- **Read replicas and sharding (one line each, deep later).** A **read replica** is a copy of the database that serves reads, spreading read load across machines. **Sharding** splits the data across multiple databases by key so no single machine holds it all. Both are how the database hop scales past one box; the mechanics are the Databases and Scaling levels.

- **Latency:** an indexed query ~1-10 ms; an unindexed scan or a contended write can be far worse and is a frequent tail-latency culprit.
- **Failure:** the database is the hardest hop to make redundant without trade-offs (it holds state), so its availability and consistency choices shape the whole system.
- **Scaling knob:** replicas for reads, sharding for size and writes - but each adds consistency complexity, unlike the stateless hops above.

## Hop 8: async offload

Not everything the request touches must finish _before the user gets a response_. Consider opening `/feed`: recording analytics, sending a notification, updating a recommendation model, or resizing an uploaded image can all happen _after_ the response is sent. The pattern is **async offload**: the app server drops a message onto a **queue** (a durable buffer of pending work) and returns immediately; separate **worker** processes pull from the queue and do the slow work later.

This directly serves the mental model: **work moved off the critical path stops counting toward the user's latency.** The response goes back fast; the heavy lifting happens in the background.

- **Latency:** removes slow work from the request's critical path entirely - the user waits only for the queue write (~a few ms), not the work itself.
- **Failure:** the queue decouples producer from consumer, so a slow or down worker no longer breaks the user's request - work simply waits and is retried. This also **smooths load spikes**: a burst of work queues up instead of overwhelming everything at once.
- **Scaling knob:** add more workers to drain the queue faster; scale producers and consumers independently. Queues, workers, and delivery guarantees are the Messaging/Async level.

## Hop 9: the response travels back

The response now retraces the path outward. The app server hands its result to the load balancer, which returns it toward the caller; if the request came via the CDN, the edge may cache the response for future users before passing it on; the bytes travel back over the (still-open, ideally reused) connection; and the browser receives it, renders `/feed`, and may store it in its HTTP cache so the _next_ view of this page skips much of the chain entirely.

- **Latency:** the return trip carries the same physical round-trip cost as the outbound path; large responses also cost transfer time, so response _size_ matters, not just hop count.
- **Failure:** a response can be lost in transit (triggering a client retry) or arrive but fail to render.
- **Scaling knob:** compressing responses and caching them at the edge and in the browser shrink both this hop and all future requests - closing the loop back to Hop 0's principle, _the cheapest request is the one you never make._

## How each hop connects onward

This topic is the map; each hop is a whole world explored later. As a full-stack engineer moving into system design, you will revisit every stop below in depth:

- **DNS, TCP, TLS, and connection handling** -> the Networking level.
- **Load balancers (L4 vs L7), health checks, and traffic routing** -> the Load Balancing level.
- **CDNs, cache strategies, invalidation, and hit rates** -> the Caching level.
- **Stateless services, horizontal scaling, and elasticity** -> the Scaling level.
- **Databases, indexing, replication, and sharding** -> the Databases level.
- **Queues, workers, and delivery guarantees** -> the Messaging/Async level.
- **Availability, redundancy, timeouts, and failover across all hops** -> the Reliability level.

The value of holding the whole chain in your head first: when a system is slow, you can locate the slow hop; when it fails, you know which components sit in the blast radius; and when it must grow, you know which knob to turn. Every later topic is a deep dive into one hop of this exact request.

## Check yourself

- Recite the mental-model question you should ask at every hop. Why does end-to-end p99 track the sum of the _slow_ hops on the critical path, and why does offloaded work not count?
- A brand-new secure connection can cost 2-3 round trips before any request data is sent. Which handshakes cause this, and what technique removes the cost for subsequent requests?
- Explain "hit vs miss" for both the CDN (Hop 3) and the cache (Hop 6). In each case, what does a miss cost that a hit avoids?
- Why is the read cache called the number-one lever for read-heavy systems? What failure mode appears if the cache suddenly goes down?
- Give two pieces of work involved in loading `/feed` that could be moved to an async queue, and explain how that improves the user's latency without dropping the work.
- The site is unreachable but every application server is healthy and the database is fine. Name two earlier hops that could independently cause this, and why.
