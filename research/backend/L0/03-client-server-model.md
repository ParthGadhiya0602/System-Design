# Client-Server Model

The foundational shape of almost every networked system: one program asks for something, another program provides it, and they talk by passing messages over a network.

## Contents

- [What the client-server model is](#what-the-client-server-model-is)
- [The request-response interaction](#the-request-response-interaction)
- [Statelessness vs stateful servers](#statelessness-vs-stateful-servers)
- [Tiers](#tiers)
- [Why client-server over peer-to-peer](#why-client-server-over-peer-to-peer)
- [The key enabler for later](#the-key-enabler-for-later)
- [Check yourself](#check-yourself)

## What the client-server model is

The client-server model is a way of splitting a system into two cooperating roles that run as **separate processes** and communicate over a network by exchanging **request** and **response** messages.

Define the terms precisely, because the whole model rests on them:

- A **process** is a running program with its own memory. "Separate processes" means the client and server do not share memory directly; the only way they exchange information is by sending bytes to each other over a network (or, on one machine, over a local channel that behaves like one). This separation is the point: it lets the two halves run on different machines, be written in different languages, and be scaled independently.
- A **client** is the process that **initiates** the interaction. It decides _when_ to ask and _what_ to ask for. It generally holds no authoritative copy of the data it works with; it holds a view, a cache, or a draft, and it defers to the server for the truth. A web browser, a mobile app, a `curl` command, and one microservice calling another are all clients in the moment they send a request.
- A **server** is the process that **waits** for requests and provides a service in response. It does not initiate; it listens on a known address and answers whoever contacts it. Crucially, the server holds the **authoritative state** - the canonical version of the data and the rules for changing it. When two clients disagree about what a value is, the server's answer wins.

Two subtleties worth internalizing early:

1. **Client and server are roles, not machines.** The same physical machine, or even the same program, can be a server in one interaction and a client in another. A web application server is a _server_ to the browser but a _client_ to the database it queries. The label describes who is asking and who is answering _for a given request_, not a permanent identity.
2. **The server is a rendezvous point.** Because the server has a stable, known address and is always listening, clients that have never heard of each other can coordinate through it. Two phones exchange a message not by finding each other but by both talking to the same server. This indirection is quietly responsible for most of what makes centralized systems easy to reason about.

## The request-response interaction

The default interaction is **request-response**: the client sends one request, the server does some work, the server sends back exactly one response, and that unit of work is complete. Break down its defining properties:

- **Client-initiated.** Nothing happens until the client asks. The server cannot spontaneously send data to a client that has not requested it. This is a hard constraint of the basic model, and much of the awkwardness of "how does the server tell the client something changed?" comes from it.
- **Synchronous from the client's view.** After sending a request, the client typically waits for the matching response before it considers the operation done. ("Synchronous" here means the client's logical flow blocks on the answer, even if the underlying code uses async I/O so the thread is not literally frozen.)
- **One response per request.** Each request is paired with one response. Requests are usually independent units, which is what allows different requests to be routed to different servers (see statelessness below).

A request carries: what is being requested (an operation and a target, e.g. a URL and a method), who is asking (identity or a token), and any input data (a body or parameters). A response carries: an outcome status (success, client error, server error), and any returned data. Keeping this envelope explicit is why request-response systems are easy to log, cache, retry, and load-balance - every interaction is a self-describing message.

**Contrast: server push and streaming.** Plain request-response cannot handle "tell me the moment something new happens" - a chat message from someone else, a live score, a stock tick. Polling (the client re-asking on a timer) approximates it but wastes requests and adds latency. The real fixes invert or extend the flow:

- **Server push** lets the server send data to the client without a fresh request each time, over a connection the client opened once and kept open.
- **Streaming** sends many messages over that one long-lived connection instead of a single response.

Technologies such as WebSockets (bidirectional) and Server-Sent Events (server-to-client) implement these patterns and are covered in a later level. The point here: request-response is the baseline, and push/streaming are deliberate departures from it for cases where the client should not have to keep asking.

## Statelessness vs stateful servers

**State** is any information the server remembers _about a particular client_ across more than one request - who is logged in, what is in a shopping cart, how far through a multi-step flow the client is.

- A **stateful server** keeps that per-client information in its own local memory between requests. Request 2 relies on something request 1 left behind _on that specific server_. This is simple to code but has a hidden cost: that client is now tied to that one server, because only it holds the memory. If the server restarts, the state is gone; if you add more servers, they cannot help this client.
- A **stateless server** keeps **no per-client state locally between requests**. Every request arrives carrying everything needed to handle it. The server may do heavy work, but when the response is sent it forgets the client entirely. Two consecutive requests from the same client can be handled by two different server instances and neither notices.

Statelessness does not mean "no state anywhere" - it means the state does not live _inside the request-handling process_. It moves to one of two places:

1. **A shared store** both/all servers can read and write - a database, a cache, or a session store. The server becomes a stateless worker in front of shared, authoritative state.
2. **The token / request itself.** The client presents a signed token (or other self-contained credential) on every request that encodes who it is and what it is allowed to do. The server verifies it and trusts it without looking anything up locally.

Why this matters so much: because any stateless server can handle any request, you can put many identical servers behind one address and route each request to whichever is free. That is exactly what makes **horizontal scaling** (adding more servers) easy - there is nothing to migrate and no client is pinned to a box. Stateful servers fight this at every turn. The mechanics of scaling this way are a later topic; here, just anchor the causal chain: _statelessness enables interchangeable servers, which enables horizontal scale._

## Tiers

A **tier** is a separately deployable layer of the system with a distinct responsibility. Tiers are how the two-role client-server idea grows into real architectures. They are a _logical_ separation - a tier can run as one process or many - and each tier can be scaled and changed independently of the others.

- **2-tier (client to database).** The client talks more or less directly to a data store. The client (or a thin server) holds the application logic and issues data operations. Simple and low-latency, but business rules live near the edge and are hard to change centrally, and exposing the data store toward clients is a security and coupling problem. Fine for small internal tools; rarely the shape of a public system.

- **3-tier (presentation / application / data).** The standard, default layout:
  - **Presentation tier** - what the user interacts with (browser UI, mobile app, or a server that renders/serves the UI). It formats requests and displays responses; it holds no authoritative truth.
  - **Application tier** (a.k.a. logic / business tier) - the servers that hold the business rules, validate input, enforce permissions, and orchestrate work. This is where request-response is served and where statelessness usually lives.
  - **Data tier** - the databases, caches, and other stores that hold the authoritative state.

  The value is separation of concerns _and_ independent scaling: a read-heavy app can run many application servers against a smaller data tier; a UI change never touches the database. Each tier can also be secured at its boundary, so only the application tier ever touches the data tier.

- **n-tier / microservices.** The application tier is itself split into many services, each owning one capability (payments, search, notifications) and often its own data. "n-tier" is the general idea of more than three layers; "microservices" is a specific style where those layers are many small, independently deployed services that call one another - every such call is itself a client-server request-response. This buys independent deployment and scaling per capability, at the cost of more network hops, more failure modes, and harder consistency. It is the right choice when teams and load justify the overhead, not by default.

Default guidance: reach for **3-tier** first; move to microservices only when a specific pressure (team autonomy, divergent scaling needs, deploy independence) demands it.

## Why client-server over peer-to-peer

In **peer-to-peer (P2P)**, there is no dedicated server role: every node is both client and server, requesting from and providing to its peers, with no central authority holding the truth.

Client-server centralizes authority in the server, and that central point is what makes several hard problems easy:

- **Consistency.** One authoritative copy of the state means there is always a single answer to "what is the current value?" In P2P, the same data lives on many equal peers and you must reconcile disagreements - a genuinely hard distributed problem.
- **Security and access control.** A central server is one place to authenticate clients, enforce permissions, and validate every change. In P2P, trust and enforcement must be spread across nodes you may not control.
- **Updates and operability.** You upgrade logic, fix data, or change rules in one place and every client sees it. Rolling changes across a swarm of independent peers is far harder.
- **Simplicity.** Clients can be thin; the hard logic lives server-side where you control it.

The cost is real and honest: the server becomes a **focal point for scaling and availability**. All load converges on it, and if it is down the service is down. This is not a flaw to hide - it is precisely the problem the rest of this track spends its time solving (replication, load balancing, caching, redundancy, failover). The trade is deliberate: accept a central bottleneck because a bottleneck you control is easier than distributed disagreement you cannot.

**Where P2P still wins.** When the goal is moving large volumes of identical content or removing central control, P2P shines: bulk content distribution (many peers sharing pieces of the same large file, so total capacity grows with the number of participants), and decentralization by design (systems that intentionally have no single owner or single point to censor or shut down). Many real systems are hybrids - a central server for coordination and authority, P2P for the heavy bulk transfer.

## The key enabler for later

Here is the single idea from this topic that unlocks the rest of the track.

A client does not talk to a _machine_. It talks to a **logical address** - a name or address it was told to use. The client sends its request there and expects a response; it has no knowledge of, and no interest in, how many machines actually sit behind that address or which one answers.

That indirection is the crack through which all scaling enters. Because the client is loyal only to the address and (if the servers are stateless) does not care which instance replies, you can put a **load balancer** at that address and hide _many_ physical servers behind it. The load balancer receives each request and forwards it to one of the servers; the client sees one endpoint and one response, exactly as before. Add servers, remove servers, replace a crashed one - the client's view never changes.

So the pieces connect like this: the **client-server model** gives you a clean request-response contract; **statelessness** makes servers interchangeable; a **logical address** decouples the client from any specific server; and a **load balancer** exploits all of that to spread load across many machines. How addresses resolve to machines is the Networking level; how load balancers distribute and health-check traffic is the Load Balancing level; how statelessness turns into elastic capacity is the Scaling level. This topic is the contract those later mechanisms build on.

## Check yourself

- A single program acts as a server in one interaction and a client in another. Give an example and explain why the labels are per-request, not per-machine.
- Why can a stateless server be swapped mid-session without the client noticing, while a stateful one cannot? Where did the state go?
- Your feature needs the server to notify the client the instant data changes. Why does plain request-response fail, and what class of technology fixes it?
- Name the three tiers of a 3-tier architecture and one thing each is responsible for. Why can they scale independently?
- State one problem client-server makes easier than P2P and the one cost you accept in return.
- Explain, in one sentence each, how "the client talks to a logical address" and "the servers are stateless" together make a load balancer possible.
