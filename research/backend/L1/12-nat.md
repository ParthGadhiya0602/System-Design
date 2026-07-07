# NAT: Sharing One Public IP Among Many Private Hosts

_Your laptop's address is `192.168.1.10` and it will never appear anywhere on the public internet — every packet it sends gets a new source address stamped on it at the border of your network, and something has to remember exactly how to undo that on the way back._

`⏱️ ~7 min · 12 of 17 · L1 Networking`

## Contents

- [What NAT is and why it exists](#what-nat-is-and-why-it-exists)
- [The core mechanism: the NAT translation table](#the-core-mechanism-the-nat-translation-table)
- [Types of NAT](#types-of-nat)
- [Worked example: two devices, one public IP](#worked-example-two-devices-one-public-ip)
- [NAT traversal: the hard part for peer-to-peer](#nat-traversal-the-hard-part-for-peer-to-peer)
- [NAT vs a proxy](#nat-vs-a-proxy)
- [How NAT connects to the rest of the stack](#how-nat-connects-to-the-rest-of-the-stack)
- [Trade-offs and common confusions](#trade-offs-and-common-confusions)
- [Check yourself](#check-yourself)
- [Real-world and sources](#real-world-and-sources)

## What NAT is and why it exists

**NAT (Network Address Translation)** is a technique, typically performed by a router sitting at the boundary between a private network and the public internet, that **rewrites the IP addresses (and usually the port numbers) in packet headers** as packets cross that boundary. RFC 2663 defines the general terminology; RFC 3022 defines "traditional NAT" — the outbound-initiated, address-and-port-rewriting form almost every home and office router runs.

**Why it exists — IPv4 exhaustion.** [02-ip-addressing-and-subnets.md](02-ip-addressing-and-subnets.md) covered the problem this solves directly: IPv4 has only ~4.3 billion addresses, nowhere near enough for every device on earth to hold a unique public one, and RFC 1918 carved out private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) that any organization can reuse internally without coordinating with anyone else — precisely because those addresses are never supposed to be routable on the public internet. NAT is the mechanism that makes those private addresses *usable* for reaching the internet at all: it lets an entire home, office, or data center full of privately-addressed hosts share a small number of public IP addresses (often just one) by rewriting each outbound packet's source address to a public one at the border, and rewriting the matching inbound reply back to the correct private address on the way in.

**The private/public boundary.** A NAT device (a home router, a corporate firewall, a cloud NAT gateway) sits exactly on the line between two worlds: on the "inside," hosts use private, non-globally-routable addresses; on the "outside," every packet that leaves must carry an address the rest of the internet can actually route back to. NAT is the translator standing at that seam.

**"Translation," precisely defined.** Translation here means rewriting fields in the IP header (the source or destination address) and, in the common case, the transport header (the source or destination port) — done transparently, in-flight, without either endpoint's application needing to know it happened. The two real endpoints of the underlying TCP or UDP connection are still, conceptually, the original client and the original server — NAT does not terminate the connection or open a new one on the client's behalf (a distinction that matters a lot when contrasting NAT with a proxy, below).

**A secondary, incidental effect: crude perimeter behavior.** Because NAT only creates a translation table entry when an *internal* host initiates outbound traffic, a packet arriving from the internet with no matching entry has nowhere to go and is simply dropped. This *looks* like a firewall blocking unsolicited inbound connections, and in practice it does block a lot of unwanted inbound traffic by default — but it is a side effect of how the translation table is built, not a deliberate security policy, a distinction covered further below.

## The core mechanism: the NAT translation table

The heart of NAT is a table that maps an internal (private IP, private port) pair to an external (public IP, public port) pair, kept in sync in both directions.

**Outbound, step by step:**

1. Internal host `192.168.1.10` opens a TCP connection from local port `54321` to a web server at `93.184.216.34:443`. The packet's source is `192.168.1.10:54321`.
2. The packet reaches the NAT device (the router). The router rewrites the **source** address and port to its own public IP and a port it chooses from its own pool, e.g. `203.0.113.5:40000`.
3. The router records this mapping in its translation table: `192.168.1.10:54321 <-> 203.0.113.5:40000` (for this destination, `93.184.216.34:443`).
4. The rewritten packet, now genuinely addressed with a routable public source, continues onward across the internet.

**Return traffic, step by step:**

1. The server replies to `203.0.113.5:40000` (the only address it ever saw).
2. The NAT device receives the reply, looks up `203.0.113.5:40000` in its translation table, finds the matching entry, and rewrites the **destination** back to `192.168.1.10:54321`.
3. The packet is delivered internally to the correct host and port, exactly as if nothing happened from the application's point of view.

```
OUTBOUND
  Internal host                NAT device                  Internet
  192.168.1.10:54321  ----->  [rewrite src]  ----->  dst 93.184.216.34:443
                                src: 203.0.113.5:40000

  Translation table entry created:
  192.168.1.10:54321  <---->  203.0.113.5:40000

RETURN
  Internet                     NAT device                  Internal host
  93.184.216.34:443  ----->  [lookup + rewrite dst]  ----->  192.168.1.10:54321
                              dst: 192.168.1.10:54321
```

**Why the port matters so much here.** This is the direct payoff of everything learned about ports and the connection 4-tuple in [04-tcp.md](04-tcp.md#where-tcp-sits-and-the-4-tuple) and [05-udp.md](05-udp.md): a single public IP address has 65,536 possible ports, and the NAT device uses the **port** as the disambiguator that lets many different internal (IP, port) combinations share one public IP. Without ports to multiplex on, one public IP could support translation for only one internal host at a time — the port is what makes many-to-one sharing possible at all.

## Types of NAT

- **Static NAT** — a fixed, permanent one-to-one mapping between one private IP and one public IP (e.g. `192.168.1.10` always maps to `203.0.113.10`). Every private address that needs one gets its own dedicated public address; no address conservation happens, since it's still one public IP per host — it's used when a specific internal host (e.g. an internal server) needs a stable, predictable public address for inbound access, not to solve exhaustion.
- **Dynamic NAT** — a pool of public IP addresses is available, and each internal host is assigned one from the pool on demand (first-come, first-served) rather than a permanent, hard-coded mapping. Still one internal host per public IP at any given moment, so it only helps if the number of *simultaneously active* internal hosts is smaller than the pool of public IPs — it doesn't solve exhaustion at internet scale either, just spreads a smaller pool across a larger, less-simultaneously-active population.
- **PAT / NAPT (Port Address Translation, a.k.a. NAT overload)** — the ubiquitous case, and what "NAT" means in ordinary usage almost every time someone says it: **many internal hosts share ONE public IP**, disambiguated purely by port, exactly as shown in the mechanism section and worked example. This is what every home router does by default, and it's the form that actually solves the "not enough public IPv4 addresses for every device" problem, since dozens or hundreds of internal devices can share a single public IP simultaneously, each occupying a different port range.
- **CGNAT (Carrier-Grade NAT)** — ISPs applying PAT one level higher: instead of (or in addition to) a home router NAT-ing a household's devices behind one public IP, the ISP itself NATs *many customers'* home routers behind a smaller pool of public IPs it controls (RFC 6888 documents common CGNAT deployment behaviors, `verify` exact scope). This squeezes more use out of a shrinking supply of public IPv4 addresses, at the cost of stacking two layers of translation and making some of NAT's downsides (inbound connectivity, per-flow state limits, traceability of individual customers) worse, since an ISP-level NAT table has to track far more simultaneous flows than a single household's router ever would.

## Worked example: two devices, one public IP

A home has one router with public IP `203.0.113.5` and two devices behind it, both browsing the web at the same time:

- Device A: `192.168.1.10`, browser opens a connection from local port `54321` to `93.184.216.34:443`.
- Device B: `192.168.1.11`, browser opens a connection from local port `54321` (yes, the same local port number is perfectly fine — it's a *different* private IP) to a different server, `172.217.14.206:443`.

The router's translation table, after both connections are established:

| Private IP:Port | Public IP:Port | Remote destination |
|---|---|---|
| `192.168.1.10:54321` | `203.0.113.5:40000` | `93.184.216.34:443` |
| `192.168.1.11:54321` | `203.0.113.5:40001` | `172.217.14.206:443` |

Both devices appear to the outside world as coming from the exact same public IP, `203.0.113.5` — but each connection gets a distinct public **port** (`40000` vs `40001`), even though the internal port numbers happened to collide. When a reply arrives at `203.0.113.5:40000`, the router's table lookup unambiguously routes it back to `192.168.1.10:54321`; a reply at `203.0.113.5:40001` goes to `192.168.1.11:54321`. If both internal devices had *also* both happened to connect to the exact same remote server and port, the port-based demultiplexing is still what keeps them separate — the NAT device simply would not reuse `40000` for a second mapping while the first is still active, choosing a fresh public port (or, in some NAT implementations, reusing the same public port only if the full 5-tuple including destination is unique per RFC 3022's endpoint-independent vs endpoint-dependent mapping distinction, `verify` exact behavior varies by NAT implementation/RFC 4787 "Requirements for NAT Behavior"). This is exactly why a home network with a dozen devices can all browse the internet "simultaneously" through one public IPv4 address without ever colliding.

## NAT traversal: the hard part for peer-to-peer

**The problem.** NAT's translation table is only populated by *outbound* traffic. A packet arriving unsolicited from the internet, with no prior outbound packet from that internal host to that remote address, has no table entry to match, and the NAT device has no idea which internal host (if any) it's meant for — so it's dropped. This is fine for ordinary client-to-server traffic (the client always initiates), but it directly breaks the case of two peers, **each behind their own NAT**, trying to connect *directly* to each other for something like a video call or a file transfer: neither one has a public IP that the other can simply dial, and neither can "receive" an initiating packet the other side sends, because neither NAT has a table entry yet for the other peer's address.

**STUN (Session Traversal Utilities for NAT, RFC 8489).** A lightweight protocol where a host sends a request to a public STUN server, and the STUN server's reply simply tells the host what public (IP, port) mapping its own NAT assigned to that request — i.e., "this is what you look like from the outside." Both peers do this, exchange that discovered public mapping via some separate signaling channel (e.g. a server both peers already trust), and then each peer can attempt to send packets directly to the other's discovered public address.

**Hole punching.** The trick that makes STUN-discovered addresses actually usable: if both peers send an outbound packet toward each other's discovered public address at roughly the same time, each side's own NAT creates a translation table entry for that destination (because it looks, to each NAT, like ordinary outbound traffic the local host initiated) — and once both table entries exist, packets from the other peer arriving at that address now match an entry and get let through. It works for a large fraction of real-world NAT configurations, but not all (some NAT behaviors, particularly "symmetric" NAT, assign a different public port per destination, which defeats simple hole punching, `verify` exact NAT-type terminology/success rates before citing specifics).

**TURN (Traversal Using Relays around NAT, RFC 8656).** The fallback when direct hole punching fails: both peers instead send their traffic to a public relay server, which simply forwards it between them. This always works (it's just ordinary client-to-server traffic in both directions, nothing NAT-unfriendly about it), at the cost of extra latency (an added hop) and bandwidth cost borne by whoever runs the relay, since every byte of the "peer-to-peer" call now actually flows through a third-party server.

**ICE (Interactive Connectivity Establishment, RFC 8445).** The overall framework that ties this together: a host gathers a list of *candidate* addresses it might be reachable at (its local private address, its STUN-discovered public address, and a TURN relay address as a last resort), exchanges the candidate list with the other peer via signaling, and both sides systematically try each combination of candidates until they find one that actually works — preferring a direct path (cheapest, lowest latency) and falling back to a TURN relay only if nothing direct succeeds.

This entire chain — STUN, hole punching, TURN, ICE — exists specifically because of the NAT traversal problem described here, and it's exactly the machinery **WebRTC** (forward-ref, a later topic) uses under the hood to let two browsers establish a real-time peer-to-peer audio/video/data connection despite both very likely sitting behind NAT.

## NAT vs a proxy

[11-forward-and-reverse-proxies.md](11-forward-and-reverse-proxies.md) covered a proxy as an intermediary that **terminates** one connection and opens a fresh one on the other side, able to read and modify the actual content of a request. NAT is a fundamentally different kind of intermediary, and the contrast is worth being precise about:

| | NAT | Proxy |
|---|---|---|
| **Layer** | L3/L4 — rewrites IP addresses and transport ports | L4 (forward/reverse TCP proxy) or L7 (HTTP-aware proxy) |
| **Connection** | The transport connection is still, conceptually, end-to-end between the real client and real server — NAT only rewrites headers in flight | The proxy terminates the client's connection and opens an entirely separate connection to the server; there are genuinely two connections |
| **Visibility into payload** | None — NAT only touches address/port fields, never looks at or modifies the payload (with rare legacy exceptions like FTP-aware NAT helpers, noted below) | Full — a proxy, especially an L7 one, can read, log, cache, or rewrite the actual request/response content |
| **Transparency to endpoints** | Fully transparent — neither client nor server application code needs any awareness that NAT exists | Can be transparent or explicit — the client is often explicitly configured to use a forward proxy; a reverse proxy is typically transparent to the client but the origin server may see the proxy as "the client" |
| **Typical direction of use** | Applied to outbound traffic from a private network sharing a public IP (or DNAT for inbound, see below) | Forward proxies front outbound client traffic; reverse proxies front inbound traffic to a set of backend servers |
| **Primary purpose** | Address conservation (share one public IP among many hosts) | Access control, caching, load distribution, content inspection/modification, TLS termination |

The one-line version: **NAT rewrites where a packet says it's from/to; a proxy actually stands in the middle of the conversation as a full participant.**

## How NAT connects to the rest of the stack

- **Breaks the internet's original end-to-end principle.** The internet's classic design assumption was that any host could address any other host directly by IP. NAT breaks this by design — a private host has no address the outside world can dial into unsolicited, which is exactly the traversal problem above, and is a large part of why the internet today looks less like "everyone can reach everyone" than its original architecture intended.
- **Complicates protocols that embed IP addresses in their payload.** Some older protocols (classic **FTP** in active mode, **SIP** for VoIP signaling) put an IP address and port *inside* the application-layer payload itself, expecting the receiver to connect back to that address. Plain NAT, which only rewrites the IP/transport headers and never touches payload bytes, breaks this — the embedded address is still the private one, useless to the outside world. Real-world NAT devices historically shipped protocol-aware "ALG" (Application Layer Gateway) helpers that specifically parse and rewrite FTP/SIP payloads too, patching around this limitation, `verify` current prevalence of ALGs given IPv6/modern protocol adoption.
- **A reason IPv6 aims to reduce reliance on NAT.** [02-ip-addressing-and-subnets.md](02-ip-addressing-and-subnets.md) noted IPv6's address space (2^128) is vast enough that every device could plausibly get its own globally unique, routable address — removing the *scarcity* reason NAT exists in IPv4. This doesn't eliminate the incidental perimeter-like behavior some networks still want, but it removes the forced, mandatory translation IPv4 exhaustion imposes.
- **Why data-center servers typically get routable addresses, sitting behind load balancers, rather than relying on NAT.** A server that needs to accept inbound connections from arbitrary internet clients needs a stable, addressable presence — architectures typically solve this with public/routable addresses on load balancers (forward-ref, load balancers) fronting a private backend network, rather than trying to make inbound NAT traversal work at data-center scale.
- **Port forwarding — the manual escape hatch for inbound.** A NAT device can be explicitly configured to always forward inbound traffic on a specific public port to a specific internal (IP, port), pre-creating exactly the table entry that would otherwise only exist after outbound traffic. This is how a home user might expose a personal server behind a home router, or how self-hosted services accept inbound connections despite sitting behind PAT.
- **DNAT (Destination NAT) — the server-side mirror image, and its link to load balancers (forward-ref).** Everything above described NAT rewriting the *source* address of outbound traffic (SNAT). The reverse — rewriting the *destination* address of *inbound* traffic — is called DNAT, and it's conceptually exactly what a load balancer does when it accepts a client connection addressed to its own public/virtual IP and forwards it to a chosen backend server's private address: the load balancer is performing destination-side address translation, just dressed up with health checks and a balancing algorithm on top.
- **NAT is not a firewall — a common and important misconception (ties to L9 security).** NAT drops unsolicited inbound packets only because they have no matching table entry, a side effect of how outbound-initiated translation works, not because NAT evaluates any access-control policy. A real firewall makes deliberate allow/deny decisions based on rules; NAT makes no decision at all — it simply has nowhere to route a packet it never built a mapping for. A misconfigured static NAT mapping, a port forward, or a device that itself initiates outbound traffic can all still expose an internal host despite NAT being present, which is exactly why relying on NAT *as* a security boundary is a mistake worth flagging explicitly.

## Trade-offs and common confusions

**Benefits:**
- Conserves scarce public IPv4 address space by letting many private hosts share one (or a few) public IP(s) — PAT is the workhorse of the entire consumer and small-business internet.
- Provides incidental obscurity: internal network topology and addresses are never directly visible to the outside world, and unsolicited inbound connections are dropped by default.

**Costs:**
- Breaks true end-to-end connectivity — an internal host cannot be dialed into directly without extra machinery (port forwarding, STUN/TURN/ICE for P2P).
- Complicates peer-to-peer and any protocol that embeds addresses in its payload, requiring traversal techniques or protocol-aware ALGs.
- Adds real, finite state: the translation table has a maximum size and each entry consumes memory and needs an idle timeout to be reclaimed — a NAT device under heavy simultaneous-connection load can genuinely run out of table capacity or public ports (especially acute for CGNAT, tracking far more flows than a home router ever would).
- **Is not a security firewall**, despite widely being treated as one in casual conversation — it blocks unsolicited inbound traffic as an incidental consequence of table-based lookup, not as an enforced access-control policy, and should never be relied upon as a substitute for one.

**Clarifying the overlapping terms:**

| Term | What it actually does |
|---|---|
| **NAT** | Rewrites L3/L4 addresses in flight; connection stays conceptually end-to-end |
| **Proxy** | Terminates and re-originates the connection; can inspect/modify payload |
| **Firewall** | Makes deliberate allow/deny decisions on traffic based on rules; does not necessarily rewrite anything |

- "NAT" without qualification almost always means **PAT/NAPT** in practice — static and dynamic NAT (one-to-one and pool-based) exist and have specific uses, but the many-to-one, port-disambiguated form is what runs in the overwhelming majority of real deployments.
- NAT is fundamentally a **stopgap for IPv4 exhaustion**, not a permanent architectural ideal — IPv6's address abundance is designed specifically to make mandatory NAT unnecessary, even though some organizations may still choose address-hiding techniques for other reasons.

> [!IMPORTANT]
> NAT rewrites the (IP, port) in a packet's headers at the private/public boundary and remembers the mapping in a translation table so return traffic can be routed back — it is not a firewall (dropping unsolicited inbound is a side effect of table lookup, not a policy decision), and it is not a proxy (the underlying connection stays end-to-end; NAT never terminates or reads payload). PAT/NAPT, sharing one public IP across many hosts via port-based disambiguation, is what "NAT" means in almost every real deployment, and it exists because IPv4 doesn't have enough addresses to go around — a scarcity problem IPv6 is designed to make obsolete.

## Check yourself

- A colleague says "we don't need a firewall, our router already does NAT." What's wrong with that reasoning, and what specifically would still get through despite NAT being in place?
- Two devices behind the same home router both open connections to the internet using the same local port number. Explain, using the translation table concept, exactly why this doesn't cause a collision.
- Why can't two peers, each behind their own NAT, simply send a UDP packet directly to each other's public IP and have it work the first time? Walk through what STUN, hole punching, and (if needed) TURN each contribute to solving this.
- What's the precise difference between what NAT does to a connection and what a reverse proxy does to a connection, in terms of how many actual transport connections exist?

## Real-world and sources

**Home and office internet connectivity as the universal, everyday deployment of PAT.** Virtually every home router and most small-office gateways run PAT by default, sharing one ISP-assigned public IPv4 address across every device on the local network — the mechanism described throughout this document is running, right now, on essentially every residential internet connection in the world using IPv4.

**CGNAT as ISPs' extension of the same idea one layer up**, used by many residential and mobile ISPs to stretch a limited pool of public IPv4 addresses across a larger customer base than one-address-per-customer would allow, at the cost of some customers being unable to easily accept unsolicited inbound connections at all without additional configuration (`verify` specific ISP deployment details before citing any particular provider).

**WebRTC as the canonical consumer of STUN/TURN/ICE**, used by browser-based video calling and real-time data-channel applications specifically to establish direct peer-to-peer connections between two users each very likely sitting behind NAT (forward-ref, a later WebRTC topic covers this in full).

### Sources / further reading

- RFC 2663, "IP Network Address Translator (NAT) Terminology and Considerations" (defines the general NAT vocabulary used throughout this document)
- RFC 3022, "Traditional IP Network Address Translator (Traditional NAT)" (defines basic NAT and NAPT/PAT behavior)
- RFC 4787, "Network Address Translation (NAT) Behavioral Requirements for Unicast UDP" (defines endpoint-independent vs endpoint-dependent mapping/filtering behavior referenced in the worked example, `verify` exact terminology when citing precisely)
- RFC 6888, "Common Requirements for Carrier-Grade NATs (CGNs)" (defines expected CGNAT behavior)
- RFC 8445, "Interactive Connectivity Establishment (ICE)" (the NAT-traversal candidate-negotiation framework)
- RFC 8489, "Session Traversal Utilities for NAT (STUN)" (public-mapping discovery)
- RFC 8656, "Traversal Using Relays around NAT (TURN)" (relay-based fallback)
- W. Richard Stevens, "TCP/IP Illustrated, Volume 1" (general reference covering NAT mechanics alongside core IP/TCP/UDP behavior)
