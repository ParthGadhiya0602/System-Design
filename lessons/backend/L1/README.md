# L1. Networking

*How bytes actually travel. Every system you'll design talks over a network -- this level is the ground truth beneath requests, connections, proxies, and CDNs. Start with the layer map; everything else hangs on it.*

**In progress** -- topics 1-15 of 17 are written. Work through them in order; each is a short, self-contained read (~5-8 min).

| # | Lesson | In one line | Status |
|---|--------|-------------|--------|
| 01 | [OSI and TCP/IP Models](01-osi-and-tcp-ip-models.md) | The layer map every later topic hangs on -- and where each protocol lives. | ✅ |
| 02 | [IP Addressing and Subnets](02-ip-addressing-and-subnets.md) | How hosts are named and grouped, and how a router decides "local or route it." | ✅ |
| 03 | [DNS Deep](03-dns-deep.md) | Turning names into IPs -- the hierarchy, caching, and record types behind every lookup. | ✅ |
| 04 | [TCP](04-tcp.md) | Reliable, ordered byte streams: the handshake, flow control, and congestion control. | ✅ |
| 05 | [UDP](05-udp.md) | Fire-and-forget datagrams -- when losing "reliable" buys you speed. | ✅ |
| 06 | [HTTP/1.1, HTTP/2, HTTP/3 (QUIC)](06-http-versions.md) | The web's application protocol and how each version killed the last one's bottleneck. | ✅ |
| 07 | [HTTPS / TLS Handshake](07-https-tls.md) | How two strangers agree on a shared secret and encrypt everything after. | ✅ |
| 08 | [WebSockets, SSE, Long-Polling](08-websockets-sse-long-polling.md) | Three ways to push server data to a client in near-real-time. | ✅ |
| 09 | [REST vs gRPC vs GraphQL](09-rest-grpc-graphql.md) | Three API styles and the trade-offs that decide between them. | ✅ |
| 10 | [Sockets](10-sockets.md) | The programming interface where the network stack meets your code. | ✅ |
| 11 | [Forward and Reverse Proxies](11-forward-and-reverse-proxies.md) | Two intermediaries, opposite directions -- who they front for and why. | ✅ |
| 12 | [NAT](12-nat.md) | How many private hosts share one public IP -- and what it breaks. | ✅ |
| 13 | [Load Balancers](13-load-balancers.md) | Spreading traffic across servers, and the L4-vs-L7 choice that shapes everything. | ✅ |
| 14 | [API Gateway](14-api-gateway.md) | The single front door for many services: auth, routing, rate limiting -- done once. | ✅ |
| 15 | [CDN Internals](15-cdn-internals.md) | Serving content from the edge -- caching, invalidation, and how a POP decides. | ✅ |
| 16 | Anycast / BGP Basics | How one IP lives in many places and how the internet routes between networks. | ⚪ |
| 17 | WebRTC | Peer-to-peer media and data straight between browsers. | ⚪ |

**Deeper reference:** [research/backend/L1](../../../research/backend/L1/01-osi-and-tcp-ip-models.md)

**Next level →** L2. Databases
