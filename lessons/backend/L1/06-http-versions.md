# HTTP/1.1, HTTP/2, HTTP/3 (QUIC)

*Same request, same response, same `GET /users/42` -- three times over. What changes across HTTP versions is never the words, only how fast and how cleanly the bytes that carry them get moved. And every version exists to kill the last one's bottleneck.*

`⏱️ ~8 min · 6 of 17 · Networking`

> [!TIP] The gist
> All three HTTP versions speak the **same language** (methods, status codes, headers) -- they just ride progressively better transports. **HTTP/1.1** carries one request at a time per connection, so browsers open ~6 connections to fake parallelism. **HTTP/2** multiplexes many streams over **one TCP connection** -- but because they share one TCP byte-stream, a single lost packet stalls *all* of them (the TCP head-of-line blocking you met in topic 4, resurfacing one layer up). **HTTP/3** runs over **QUIC-over-UDP** with independent per-stream ordering, so a lost packet stalls only its own stream -- plus a 1-RTT handshake and connection migration. The throughline is head-of-line blocking: H2 moved it from the app layer to the TCP layer; H3 finally eliminated it.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Picture a supermarket checkout.

**HTTP/1.1** is **one shopper per lane**. A lane can only ring up one cart at a time, so to serve more people faster you open ~6 lanes side by side. It works, but every lane needs its own cashier, its own setup -- and you're still capped at 6.

**HTTP/2** is **one wide conveyor belt** carrying everyone's items interleaved -- shopper A's milk, shopper C's bread, shopper B's eggs, all mixed together and sorted at the end. Far more efficient, one cashier. But the belt is a single track: if one item jams it, **the whole belt stops** -- everyone waits, even shoppers whose items were already past the jam.

**HTTP/3** gives each shopper their **own independent belt**. One jam stops only that shopper's lane; everybody else keeps moving. Same "one checkout, everyone interleaved" efficiency -- without the shared jam.

That jam is **head-of-line blocking**, and following it across the three versions *is* this whole topic.

## The concept

**Definition.** **HTTP (HyperText Transfer Protocol)** is a **stateless, application-layer (L7)** request/response protocol. A client sends a **request** (a method like `GET`/`POST`, a resource path, headers, an optional body); a server sends back a **response** (a status code like `200`/`404`, headers, an optional body). *Stateless* means the protocol itself remembers nothing between requests -- any "session" is bolted on top via cookies or tokens.

**The one idea that unlocks everything:** HTTP/1.1, HTTP/2, and HTTP/3 are the **same semantics over different transports**. `GET /users/42` means exactly the same thing in all three. What differs is only *how the bytes move*:

- **HTTP/1.1** and **HTTP/2** run on **TCP** (see [04-tcp.md](04-tcp.md)) -- inheriting all of TCP's guarantees *and* all of its costs.
- **HTTP/3** runs on **QUIC**, which itself runs on **UDP** (see [05-udp.md](05-udp.md)). This one swap is the spine of the whole story.

**Key terms:**
- **Multiplexing** -- carrying many independent request/response exchanges over a single connection at once.
- **Stream** -- one such logical request/response exchange, identified by an ID, interleaved with others on the wire.
- **Head-of-line (HOL) blocking** -- one stuck item at the front of a queue blocks everything behind it, even work that's already finished.
- **QUIC** -- a full transport protocol (reliability, per-stream ordering, congestion control, encryption) built in *user space* on top of UDP.

**What it is NOT:**
- **HTTP version ≠ API style.** REST, GraphQL, gRPC are conventions for what the *bodies and semantics* mean -- they sit *on top* of HTTP and are largely independent of the version. (gRPC does require HTTP/2, but that's gRPC's choice, not a general rule.)
- **HTTP/3 is NOT "HTTP over raw UDP."** It's HTTP over **QUIC** over UDP. UDP alone gives you none of QUIC's reliability, ordering, or encryption -- QUIC is the substantial layer that adds all of it back.

## How it works

Each version was designed to fix exactly one bottleneck in the version before it. Follow the fix.

### 1. HTTP/1.1 -- one request at a time (the concurrency wall)

**The problem:** a persistent HTTP/1.1 connection stays open (good -- no re-handshake per resource), but it can only usefully carry **one in-flight request at a time**. HTTP/1.1 *specified* **pipelining** (fire `/a`, `/b`, `/c` back-to-back without waiting), but responses had to come back in **strict request order** -- there was no way to tag which response answered which request. So if `/a` hit a slow query, `/b` and `/c` sat finished but stuck behind it. That's **application-layer head-of-line blocking**, and combined with buggy proxy support, browsers never enabled pipelining.

**The workaround:** open **~6 parallel TCP connections per origin** for crude parallelism -- 6 handshakes, 6 TLS setups, 6 congestion ramps. And to squeeze past that ceiling, front-end engineers piled on hacks: **domain sharding** (fake extra subdomains to unlock more connections), **spriting** (many images glued into one), **concatenation** (bundling files), **inlining** (base64 assets in the HTML). Every one exists to work around "one request at a time per connection."

### 2. HTTP/2 -- multiplexed streams (fixes app-layer HOL, inherits TCP's)

**The fix:** HTTP/2 (2015, born from Google's SPDY) makes the connection carry many requests at once, at the protocol level:

- **Binary framing** -- messages split into small binary **frames** tagged with a stream ID, so frames from *different* requests can be **interleaved** on the wire.
- **Streams + multiplexing** -- one TCP connection carries many concurrent **streams**. Fire `/a`, `/b`, `/c` together; the server sends back whichever is ready first, in any order. This kills the app-layer HOL blocking that doomed pipelining -- and makes 6 connections, sharding, spriting all unnecessary.
- **HPACK header compression** -- headers repeat constantly (same `Cookie`, `User-Agent`...). HPACK keeps a shared table on both ends so a repeated header is sent as a tiny index instead of full text.

```
HTTP/1.1  (6 connections, one request in flight per connection)
Conn1: --[req A]----[resp A]----[req D]----[resp D]--
Conn2: --[req B]-------[resp B]----[req E]-----------
Conn3: --[req C]--[resp C]---------------------------
       (each connection: strictly one request at a time)

HTTP/2  (1 connection, streams multiplexed, frames interleaved)
Conn1: --[A:HEADERS][B:HEADERS][C:HEADERS][A:DATA][C:DATA][B:DATA]--
       (streams A, B, C all in flight on ONE TCP connection;
        server sends whichever stream's data is ready first)
```

**The catch it can't fix:** all those streams share **one TCP connection**, and TCP guarantees strict in-order byte delivery. So if **one TCP segment is lost**, the OS withholds *every byte that arrived after it* -- for **all** streams -- until that gap is retransmitted (recall [TCP HOL blocking, topic 4](04-tcp.md#head-of-line-blocking)). Stream A's and C's data may have arrived perfectly, but the kernel won't hand *any* of it to the browser until stream B's lost packet is refilled. HTTP/2 didn't remove HOL blocking -- it **moved it down from the app layer to the TCP layer**, and arguably made it worse (more requests now ride the one connection that can stall).

### 3. HTTP/3 + QUIC -- independent streams (eliminates HOL blocking)

**The fix:** HTTP/3 (RFC 9114, 2022) keeps HTTP/2's model -- one connection, many streams, header compression -- but replaces **TCP with QUIC** (over UDP). QUIC implements reliability and ordering itself, in user space, and crucially **per stream**:

- **Loss stalls only its own stream.** A lost packet carrying stream B's data? QUIC retransmits just that piece; streams A and C are delivered immediately. There's no shared, strictly-ordered byte-stream to get stuck -- each stream is independently ordered. HOL blocking from packet loss is *gone*.
- **TLS 1.3 built in, encryption mandatory.** QUIC integrates TLS 1.3 directly into its handshake -- no separate "TLS on top" step, and most of the packet (not just the payload) is encrypted.
- **Faster handshake.** Fresh TCP+TLS needs ~2 RTTs (TCP handshake, *then* TLS). QUIC merges transport + crypto into **1 RTT**, and offers **0-RTT resumption** (send the first request immediately on a repeat visit -- with a replay-attack caveat, so it's limited to idempotent requests).
- **Connection migration.** TCP identifies a connection by its 4-tuple (IPs + ports) -- change your IP (Wi-Fi to cellular) and the connection breaks. QUIC uses an explicit **connection ID** independent of IP, so it survives the network switch without reconnecting.
- **QPACK** -- HPACK adapted so header compression stays correct even with QUIC's independent, out-of-order streams.

### The comparison table (the centerpiece)

| | **HTTP/1.1** | **HTTP/2** | **HTTP/3** |
|---|---|---|---|
| **Transport** | TCP | TCP | QUIC (over UDP) |
| **Multiplexing** | None (~6 connections/origin as workaround) | Many streams, 1 TCP connection | Many independent streams, 1 QUIC connection |
| **HOL blocking** | Yes -- **app layer** (pipelining needs strict order) | Fixed at app layer; **reappears at TCP layer** (one lost segment stalls all streams) | **Eliminated** -- loss stalls only its own stream |
| **Header compression** | None (plain repeated text) | HPACK | QPACK |
| **Encryption** | Optional (TLS layered on separately) | Effectively mandatory in browsers | **Mandatory**, TLS 1.3 built into QUIC |
| **Handshake (fresh conn)** | ~1 RTT TCP + ~1-2 RTT TLS | Same (TCP + TLS) | **~1 RTT** combined; **0-RTT** on resumption |
| **Connection migration** | No (tied to 4-tuple) | No (same TCP limit) | **Yes** (via connection ID) |

## In the real world

- **Google -- invented QUIC and measured the payoff.** Google deployed QUIC across Chrome and YouTube years before IETF standardization. Its SIGCOMM 2017 paper reports that the two fixes taught above delivered real, measured gains: the **1-RTT/0-RTT handshake** cut Google Search latency by **3.6-8%**, and **per-stream loss isolation** (no cross-stream HOL blocking) reduced **YouTube video rebuffering by 15-18%**. Not theoretical -- that's the "remove the shared ordered byte-stream" fix paying off at internet scale.

- **Cloudflare -- ships H3 at the edge and tracks adoption.** Cloudflare enabled HTTP/3 across its network and reports its share of browser-served traffic grew from **23% (May 2022) to 28% (May 2023)**, with mobile Safari's HTTP/3 share rising from under 3% to ~18% -- concentrated in exactly the latency-sensitive, mobile-heavy interactive traffic H3 helps most. It also open-sourced its QUIC/HTTP-3 implementation (`quiche`), citing QUIC's userspace nature as what lets it ship protocol improvements without waiting on OS/kernel changes.

- **Fastly -- edge H3 for the same two wins.** Fastly added HTTP/3 and QUIC across its edge in 2020, citing minimized head-of-line blocking, reduced handshake latency, and faster video start-up -- and notes that because QUIC lives in userspace, it integrates with existing tooling and iterates faster than a kernel-level TCP change ever could.

Full sourcing (Google SIGCOMM 2017, Cloudflare adoption posts, Fastly, RFCs 9114/9000/9204): [research/backend/L1/06-http-versions.md](../../../research/backend/L1/06-http-versions.md#real-world-and-sources).

## Trade-offs

| Version | ✅ Use it when | ❌ Watch out for |
|---|---|---|
| **HTTP/1.1** | Simple, low-traffic, internal service-to-service calls; trivial to debug (plain text); multiplexing gains don't matter | No real multiplexing; header repetition; app-layer HOL blocking under pipelining |
| **HTTP/2** | The practical default for public HTTPS today; multiplexing kills the old front-end hacks; gRPC is built on it | Still bottlenecked by **TCP-level HOL blocking** under any packet loss; harder to debug than plain text |
| **HTTP/3** | Latency-sensitive, high-traffic, or mobile-heavy consumer traffic; lossy networks; Wi-Fi to cellular handoff | UDP can be **blocked/throttled** by some firewalls, so servers must offer **H2 fallback** (advertised via `Alt-Svc`); higher CPU cost than kernel-tuned TCP; newer tooling |

**In short:** H3 is deployed *alongside* H2, not as a full TCP replacement -- because UDP's middlebox support is uneven, `Alt-Svc` lets a client fall back to H2 when QUIC is unreachable.

## Remember

> [!IMPORTANT] Remember
> Each HTTP version killed the previous one's bottleneck, and the **throughline is head-of-line blocking**. HTTP/1.1 had it at the **app layer** (pipelining needed strict order). HTTP/2 fixed *that* with multiplexed streams -- but by piling them on one TCP connection, it **pushed HOL blocking down to the TCP layer** (one lost segment stalls every stream). HTTP/3 finally **eliminated** it by moving transport onto **QUIC-over-UDP** with independent per-stream ordering. Same HTTP semantics throughout -- only the transport got smarter.

## Check yourself

1. HTTP/2 multiplexes all requests over **one** TCP connection. So why does a single lost packet still stall *every* stream -- and what specifically does HTTP/3 change so that loss stalls only one stream?
2. Why is HTTP/3 built on **UDP** rather than TCP, given that QUIC has to rebuild reliability and ordering that TCP already provides for free?
3. A colleague says "HTTP/2 solved head-of-line blocking." What's the precise flaw in that claim?

---

→ Next: [HTTPS / TLS Handshake](07-https-tls.md) (how two strangers agree on a shared secret and encrypt everything after)
↩ Comes back in: TLS 1.3, REST vs gRPC (gRPC-on-H2), load balancers, CDN
