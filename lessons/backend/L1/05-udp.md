# UDP: Fire-and-Forget Datagrams

*TCP spends a round trip and a pile of bookkeeping manufacturing guarantees before it sends your first byte. UDP just fires the datagram and gets out of the way -- every guarantee TCP hands you for free becomes something you either don't need or build yourself.*

`⏱️ ~7 min · 5 of 17 · Networking`

> [!TIP] The gist
> **UDP** is the other main L4 transport -- and it's TCP's opposite. It's **connectionless** (no handshake), **unreliable** (no ACKs, no retransmission -- lost is lost), **unordered**, and **message-oriented** (one `send()` = one `recv()`). It's basically **IP + ports + an optional checksum**, an 8-byte header vs TCP's 20+. You give up reliability, ordering, and congestion control; you gain **zero setup latency, no head-of-line blocking, tiny overhead, and multicast**. If you need *some* of TCP's guarantees, you build exactly that piece on top -- which is precisely what QUIC/HTTP-3 does.

## Contents

- [Intuition](#intuition)
- [The concept](#the-concept)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

If TCP is a **registered letter** -- you get delivery confirmation, the post office retries if it fails, and pages arrive in order -- then UDP is **shouting a message across a crowded room**.

You call it out and keep going. Maybe they heard you, maybe the noise swallowed a word, maybe two things you yelled land out of order. You don't wait for a "got it," and you don't repeat yourself unless *you* decide it mattered. It's fast, it's cheap, and nobody manages a "connection" -- but nothing is guaranteed.

That's the whole trade. TCP buys you certainty with time and bookkeeping. UDP hands you speed and control, and leaves certainty up to you.

## The concept

**Definition.** **UDP (User Datagram Protocol)** is a **transport-layer (L4)** protocol -- TCP's sibling -- that adds almost nothing to IP: just **ports** (so a packet reaches the right *process*, not just the right *host*) and an **optional checksum** (basic corruption detection). That's the entire feature list. Defined by RFC 768 in 1980, it's one of the shortest, oldest specs on the internet.

**The four words that are UDP's whole contract:**

- **Connectionless** -- no handshake, no shared session state, no connection lifecycle. Every datagram is independent. The OS keeps essentially no per-peer memory (unlike TCP's state block per connection).
- **Unreliable** -- **no guarantee** a datagram arrives. If a router drops it, the sender is never told. No ACK, no retransmission.
- **Unordered** -- datagrams sent 1, 2, 3 may arrive 2, 1, 3. UDP delivers them in *arrival* order and does nothing to resequence.
- **Message-oriented (datagram)** -- one `send()` produces exactly one self-contained **datagram**; one `recv()` returns exactly that payload, whole, never merged or split. This is the direct opposite of TCP's byte stream.

**Two terms, precisely:** *Best-effort* means the network tries to deliver but promises nothing -- not arrival, not order, not single delivery. *Datagram* means a single, self-contained unit that carries enough to be routed on its own, with no dependency on any datagram before or after -- the opposite of a "stream" where bytes only make sense as a continuous sequence.

**What UDP is NOT:**

- **NOT "broken TCP."** It's a deliberate minimal design so apps that don't need TCP's guarantees aren't forced to pay for them.
- **"Unreliable" does NOT mean "usually fails."** Most UDP datagrams, on most networks, arrive perfectly fine. "Unreliable" describes the absence of a *guarantee*, not a poor track record.
- **NOT insecure by nature.** Connectionless just means no session setup -- it's still a real L4 protocol with ports, and it can be secured (DTLS is literally "TLS for datagrams"). Reliability and security are unrelated axes.
- **NOT a byte stream.** Message boundaries are preserved -- that's a structural gift, not a limitation.

**Key terms:** datagram, best-effort, connectionless, checksum, message-oriented, head-of-line blocking (avoided), multicast.

## How it works

### 1. TCP vs UDP -- the centerpiece

Every property below is defined by direct contrast with [TCP](04-tcp.md). Read this table as "what TCP does, and what UDP deliberately declines to do":

| | TCP | UDP |
|---|---|---|
| **Connection setup** | 3-way handshake, 1 RTT before data | None -- send immediately, **0 RTT** |
| **Connection state** | Per-connection state block (kernel memory) | Essentially none -- fire to thousands of peers cheaply |
| **Reliability** | Guaranteed via ACK + retransmission | None -- loss is silent and permanent unless the app handles it |
| **Ordering** | Strict, in-order byte stream | None -- delivered in arrival order |
| **Message boundaries** | None (byte stream; app must frame) | **Preserved** -- 1 send = 1 receive |
| **Flow control** | Yes (`rwnd`) | None -- receiver drops what it can't buffer |
| **Congestion control** | Yes (`cwnd`, CUBIC/BBR) | None built in -- the app must add its own if sending at volume |
| **Head-of-line blocking** | Yes, structural -- one loss stalls all later data | **No** -- loss affects only that one datagram |
| **Header size** | 20+ bytes | **8 bytes, fixed** |
| **Multicast/broadcast** | Not supported (point-to-point only) | **Supported** |

Notice the pattern: **every field TCP carries that UDP lacks corresponds exactly to a guarantee UDP doesn't make.** No sequence numbers (nothing tracked in order), no ACK numbers (nothing acknowledged), no window (no flow control), no flags (no connection states to signal).

### 2. The header -- 8 lean bytes

UDP's header is fixed at **8 bytes: four 16-bit fields**, nothing else.

```
+----------------+----------------+
|  Source Port   | Destination Port|
+----------------+----------------+
|     Length     |    Checksum     |
+----------------+----------------+
|          Data (payload)         |
+----------------+----------------+
```

- **Source port** -- who sent it (so a reply can find its way back).
- **Destination port** -- which process/service receives it (e.g. `:53` for DNS). This port is the *only* thing UDP does that IP alone couldn't: IP gets a datagram to the right **host**, the port gets it to the right **process**.
- **Length** -- header + data, in bytes.
- **Checksum** -- detects accidental bit-corruption; it is *not* a security mechanism (anyone can compute a valid one). Optional over IPv4, mandatory over IPv6.

Contrast: TCP's *minimum* header is 20 bytes -- 2.5x heavier before any options. On a 50-byte DNS query, that difference is a meaningful fraction of the wire cost.

### 3. Why choose UDP -- the trade, with a worked scenario

You give up reliability, ordering, flow and congestion control. In exchange you gain **zero setup latency (no handshake RTT), no head-of-line blocking, minimal overhead, and multicast** -- plus full control to build only the reliability you actually need.

Here's why that trade wins for real-time media. A live voice call sends a small audio frame every 20ms. The call is 3 seconds in, and the frame due at `t=3.00s` is dropped by a congested router. The next frame (`t=3.02s`) arrives fine.

| | Under TCP | Under UDP |
|---|---|---|
| The dropped frame | TCP detects the gap and **withholds the already-arrived `t=3.02s` audio** until it retransmits the lost frame (head-of-line blocking) | The `t=3.00s` frame is simply gone; `t=3.02s` is delivered immediately, unaffected |
| Perceived effect | The call **stalls**, then dumps a burst of now-stale audio -- worse than a brief gap | A tiny glitch for one frame; the call keeps flowing in real time |
| Who decided | TCP's design -- ordering is non-negotiable | The app's choice -- retransmitting stale audio is useless, so it doesn't |

For live media, **late data is worthless data**, so TCP's insistence on delivering everything in order is actively harmful. UDP lets the app say "give me the *next* frame, now."

And when an app needs *some* of TCP's guarantees, it builds exactly that piece on top of UDP -- its own sequence numbers, selective ACKs, forward error correction, or congestion control (RFC 8085 recommends the last one for any high-volume UDP sender, so it doesn't destabilize shared links). This is the whole **QUIC** insight (next topic): rebuild reliability, per-stream ordering, and congestion control in user space over UDP -- and fix the parts TCP got wrong, like head-of-line blocking across multiplexed streams.

## In the real world

Settled protocol behavior -- no industry sweep needed. The canonical UDP users, and why each picks it:

- **DNS** (see [DNS Deep](03-dns-deep.md)) -- small, one-shot, latency-sensitive lookups where a TCP handshake's RTT would be pure waste; falls back to TCP only for oversized responses or zone transfers.
- **Live video, VoIP/RTP, WebRTC** -- real-time media where a dropped frame is tolerable but a stall is not; retransmitting stale audio/video is worse than a brief glitch.
- **Online gaming** -- frequent small state updates where the newest supersedes the stale; games often layer thin custom reliability only on the few messages that truly must not be lost.
- **QUIC / HTTP-3** -- deliberately builds its own reliability, per-stream ordering, and congestion control *on* UDP, specifically to escape TCP's structural head-of-line blocking. It's the clearest proof at internet scale that "UDP has no reliability" means "the OS won't give it to you for free," not "you can't have it."

Full sourcing (RFC 768, RFC 8085, RFC 9000/QUIC, *TCP/IP Illustrated Vol. 1*): [research/backend/L1/05-udp.md](../../../research/backend/L1/05-udp.md#real-world-and-sources).

## Trade-offs

**UDP wins ✅**

- **Zero setup latency** -- no handshake RTT; the datagram leaves the instant you call `send()`.
- **No head-of-line blocking** -- one loss affects only that datagram, never later ones.
- **Lean** -- 8-byte header, near-zero per-peer state.
- **Message boundaries preserved** -- no framing bugs.
- **Multicast/broadcast** -- one datagram to many receivers.

**UDP costs ❌**

- **No reliability** -- loss is silent and permanent unless you handle it.
- **No ordering, no dedup** -- the app must number and resequence if it cares.
- **No flow or congestion control** -- a badly-behaved UDP sender can flood a link (congestion collapse); the app must be well-behaved.

**When TCP wins:** anything needing complete, ordered delivery -- web pages, APIs, databases, file transfer. **When UDP wins:** minimum latency over perfect delivery (real-time media, gaming), tiny one-shot request/response (DNS), multicast, or when you want to build your *own* tailored reliability (QUIC).

## Remember

> [!IMPORTANT] Remember
> UDP hands you **raw best-effort datagrams**: IP plus ports plus an optional checksum, and nothing more. You trade *all* of TCP's guarantees -- handshake, reliability, ordering, flow and congestion control -- for **speed and control**. And if you need reliability, you don't switch back to TCP; you build **exactly the piece you need** on top. That's the QUIC insight in one sentence.

## Check yourself

1. Why does DNS default to UDP for ordinary queries instead of TCP -- and why does it fall back to TCP for very large responses?
2. Your app needs *some* ordering but can't tolerate a handshake's setup latency. How do you get both? (Hint: what does QUIC do?)
3. A colleague says "UDP is unreliable, so it's risky in production." Using the *formal* meaning of "unreliable," what's imprecise about that -- and what actually breaks a shared network if a UDP app misbehaves?

---

→ Next: [HTTP/1.1, HTTP/2, HTTP/3 (QUIC)](06-http.md) (the web's application protocol, and how each version killed the last one's bottleneck)
↩ Comes back in: HTTP/3 & QUIC, WebRTC, load balancers (L4), NAT
