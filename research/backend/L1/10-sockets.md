# Sockets: The Programming Handle Under Everything Else

_Every TCP handshake, every WebSocket upgrade, every TLS-wrapped HTTPS request you have studied so far is, at the code level, just a program calling a handful of decades-old operating-system functions on an object called a socket - this is where the abstraction stack finally touches the ground._

## Contents

- [What a socket is](#what-a-socket-is)
- [Socket identity and the 4-tuple, revisited](#socket-identity-and-the-4-tuple-revisited)
- [Listening sockets vs connected sockets](#listening-sockets-vs-connected-sockets)
- [The Berkeley socket API: server lifecycle](#the-berkeley-socket-api-server-lifecycle)
- [The Berkeley socket API: client lifecycle](#the-berkeley-socket-api-client-lifecycle)
- [Sequence diagram: the full syscall dance](#sequence-diagram-the-full-syscall-dance)
- [TCP sockets vs UDP sockets](#tcp-sockets-vs-udp-sockets)
- [Blocking vs non-blocking IO](#blocking-vs-non-blocking-io)
- [IO multiplexing: select, poll, epoll, kqueue](#io-multiplexing-select-poll-epoll-kqueue)
- [Unix domain sockets](#unix-domain-sockets)
- [Key OS-level details](#key-os-level-details)
- [Worked example: accepting one client, fd by fd](#worked-example-accepting-one-client-fd-by-fd)
- [Trade-offs and common confusions](#trade-offs-and-common-confusions)
- [Connects to](#connects-to)
- [Check yourself](#check-yourself)
- [Real-world and sources](#real-world-and-sources)

## What a socket is

A **socket** is the operating system's abstraction for a network communication endpoint - the handle a program holds so it can send and receive bytes to/from a remote peer using ordinary read/write-style operations, without the program having to touch IP headers, TCP segments, or a network interface driver directly. Everything you have learned so far - the TCP three-way handshake ([04-tcp.md](04-tcp.md)), UDP datagrams ([05-udp.md](05-udp.md)), HTTP requests ([06-http-versions.md](06-http-versions.md)), TLS records ([07-https-tls.md](07-https-tls.md)), WebSocket frames ([08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md)) - is, underneath, bytes moving through a socket.

**File descriptor, defined:** in Unix and Unix-like systems (Linux, macOS, BSD), the governing philosophy is "everything is a file" - a single small integer, the **file descriptor (fd)**, is the handle a process uses to refer to any open resource the kernel manages on its behalf: an open disk file, a pipe, a terminal, or a socket. When a program creates a socket, the kernel returns a file descriptor (e.g. `3`, `7`) that the program then uses for every subsequent operation on that socket - `read()`/`write()` or the socket-specific `send()`/`recv()` all take that same integer. This is not a cosmetic detail: because a socket **is** a file descriptor, it can be handed to any OS facility that already knows how to work with file descriptors in general - most importantly the multiplexing calls (`select`/`poll`/`epoll`/`kqueue`) covered below, which watch a set of file descriptors for readiness regardless of whether each one is a socket, a pipe, or a file. This is precisely what makes sockets composable with the rest of the OS's IO machinery instead of needing their own bespoke waiting mechanism.

**A socket is not the same thing as a connection, and not the same thing as a port** - a distinction worth being precise about before going further:

- A **port** is just a 16-bit number identifying an address on a host - it has no state, no buffers, no lifecycle of its own. It is a coordinate, not an object.
- A **socket** is a live, in-kernel **object** the OS creates and tracks: it has an address family (IPv4/IPv6/Unix), a type (stream/datagram), buffers, and (once bound/connected) an association with local and possibly remote addresses and ports. It is the object your program manipulates via a file descriptor.
- A **connection** is not a thing either side "has" in isolation - it is the **pairing of two sockets** (one on each host) that agree, via TCP's handshake and shared sequence-number state, to exchange an ordered byte stream. This maps directly onto the 4-tuple you already know: `(protocol, source IP, source port, destination IP, destination port)`.

## Socket identity and the 4-tuple, revisited

In [04-tcp.md](04-tcp.md#where-tcp-sits-and-the-4-tuple) you learned that a TCP connection is uniquely identified on a host by its 4-tuple: `(source IP, source port, destination IP, destination port)` (protocol is implicitly TCP). A **connected TCP socket is the local kernel object that embodies exactly one such 4-tuple's local half.** This is precisely why one server listening on a single port (say `:443`) can serve thousands of simultaneous clients from that one port number: every client connects with a different source IP and/or source port, so every resulting connected socket on the server has a distinct 4-tuple even though the destination IP:port (the server's own address) is identical every time. The port number alone was never the limiting resource - the full 4-tuple is, and it has vastly more room (theoretically up to ~65,535 distinct client-side ports per client IP, and effectively unlimited distinct client IPs).

## Listening sockets vs connected sockets

This distinction is the single most important thing to internalize before the syscall lifecycle, because it explains how a server handles many clients through one address:

- A **listening socket** is a socket in a special passive mode: it is not connected to any particular peer, has no 4-tuple in the "connected" sense (no fixed remote IP/port), and does exactly one job - accept incoming connection attempts and hand each one off as a brand-new socket. There is exactly one listening socket per address:port a server binds to.
- A **connected socket** is created *from* a listening socket, one per successfully established client connection, and it - and only it - has a full 4-tuple and can actually send/receive application data to/from that specific client.

The server never reads or writes application bytes on the listening socket itself; the listening socket's sole purpose is to keep manufacturing new connected sockets as clients connect.

## The Berkeley socket API: server lifecycle

The **Berkeley sockets API** (also called BSD sockets, introduced in 4.2BSD Unix in the early 1980s) is the standard set of syscalls virtually every OS and language runtime exposes for network programming, either directly (C, Go's low-level `net` package, Python's `socket` module) or wrapped in higher-level abstractions (Node.js's `net`/`http` modules, Java's `Socket`/`ServerSocket`) that still bottom out in these exact same calls. A server walks through five syscalls in order:

1. **`socket()`** - ask the kernel to create a new socket, specifying the address family (`AF_INET` for IPv4, `AF_INET6` for IPv6, `AF_UNIX` for local) and type (`SOCK_STREAM` for TCP, `SOCK_DGRAM` for UDP). Returns a file descriptor for a socket that exists but is not yet bound to any address, not yet listening, not yet connected - just an allocated kernel object.
2. **`bind()`** - claim a specific local IP address and port for this socket (e.g. `0.0.0.0:443` to listen on all interfaces, port 443). This is the step that fails with "address already in use" if another process already bound that exact IP:port (unless `SO_REUSEADDR` is set - relevant to the TIME_WAIT discussion below).
3. **`listen()`** - flip the socket into passive/listening mode and tell the kernel how large a **backlog** (a queue of pending/completed connection attempts) to maintain. This is the socket-level home of the SYN backlog concept from [04-tcp.md](04-tcp.md#syn-flooding-and-the-half-open-attack) - the kernel maintains this queue entirely in kernel space, independent of whether the application has called `accept()` yet.
4. **`accept()`** - block (in the simplest, synchronous model) until a completed connection sits in the backlog queue, then **dequeue it and return a brand-new, fully connected socket** (a new file descriptor, distinct from the listening socket's fd) representing that one client. The listening socket itself is untouched and immediately available to accept the next connection - this is the mechanism, concretely, by which one listening socket produces arbitrarily many connected sockets.
5. **`recv()`/`send()`** (or plain `read()`/`write()`, since it's a file descriptor) on the *connected* socket returned by `accept()` - this is where actual application bytes (an HTTP request, a WebSocket frame) flow.
6. **`close()`** - release the connected socket's file descriptor, which for TCP triggers the connection-termination sequence (FIN exchange, covered in [04-tcp.md](04-tcp.md#connection-termination)) and eventually frees the kernel's per-connection state.

## The Berkeley socket API: client lifecycle

A client's path is shorter, because it never listens or accepts - it only ever has one socket per connection it initiates:

1. **`socket()`** - same call, creates an unbound, unconnected socket.
2. **`connect()`** - given a destination IP:port, this is the call that actually **triggers the TCP three-way handshake** (SYN, SYN-ACK, ACK - see [04-tcp.md](04-tcp.md#the-three-way-handshake)): the kernel picks an ephemeral local port automatically (unless the program explicitly binds one first), sends the SYN, and `connect()` does not return successfully until the handshake completes (or times out/is refused). Once it returns, the socket is now a fully connected socket with a complete 4-tuple, ready for data.
3. **`send()`/`recv()`** - exchange application bytes.
4. **`close()`** - tear the connection down.

## Sequence diagram: the full syscall dance

```
      SERVER (fd table)                              CLIENT
      ------------------                              ------
      socket()  -> fd 3
      bind(fd 3, :443)
      listen(fd 3, backlog=128)
      [fd 3 is now a LISTENING socket]

                                                       socket() -> fd 3
                                                       connect(fd 3, server:443)
                        <---------- SYN ------------------
      [kernel places half-open entry
       in SYN queue for fd 3]
                        ----------- SYN-ACK -------------->
                        <---------- ACK -------------------
      [kernel moves entry from SYN queue
       to the completed "accept queue" for fd 3]
                                                       connect() RETURNS
                                                       (3-way handshake now done)

      accept(fd 3) -> fd 7
      [fd 7 is a NEW CONNECTED socket,
       fd 3 remains listening, untouched]

      recv(fd 7)   <----------- request bytes ---------  send(fd 3, request)
      send(fd 7, response) ------ response bytes ------>  recv(fd 3)

      close(fd 7)             <-- FIN/ACK teardown -->    close(fd 3)
```

Note precisely where the handshake happens: entirely **inside `connect()`** on the client side, and **inside the kernel, before `accept()` is even called**, on the server side - `accept()` does not perform the handshake, it only dequeues a connection whose handshake the kernel already finished. This is why a very slow or blocked application (one that isn't calling `accept()` often enough) can still have its backlog queue fill up with fully-handshaked, waiting connections - the handshake and the application's readiness to consume the connection are decoupled by that queue.

## TCP sockets vs UDP sockets

A **TCP socket** is created with `SOCK_STREAM` and follows the full lifecycle above: connection-oriented, requires `listen()`/`accept()` on the server and `connect()` on the client, delivers an ordered byte stream (see [04-tcp.md](04-tcp.md)). A **UDP socket** is created with `SOCK_DGRAM` and skips most of that ceremony entirely, matching UDP's connectionless nature from [05-udp.md](05-udp.md):

- No `listen()`, no `accept()` - there is no concept of a pending connection to accept, because there is no connection.
- The socket calls are `sendto()`/`recvfrom()` instead of `send()`/`recv()`, because every datagram carries (or must be given) its own destination address - there's no fixed peer implied by the socket the way a connected TCP socket implies one.
- `connect()` **can** still be called on a UDP socket, but it performs **no handshake whatsoever** - it purely records a fixed peer address locally in the kernel, as a convenience so the program can then use plain `send()`/`recv()` without specifying the address on every call, and so the kernel filters out datagrams from any other source. This is a pure userspace-ergonomics/filtering feature, not a network operation - a useful, easily-missed nuance (already flagged in [05-udp.md](05-udp.md#connectionless-unreliable-unordered-message-oriented-precisely)).
- One UDP socket can, without ever calling `connect()`, exchange datagrams with an unlimited number of distinct peers using `sendto()`/`recvfrom()` on each call - there's no per-peer connected-socket fan-out the way TCP's `accept()` produces one new socket per client.

## Blocking vs non-blocking IO

By default, socket operations are **blocking**: calling `recv()` on a socket with no data yet available **parks the calling thread** - the OS suspends it and does not schedule it to run again until data arrives (or an error/timeout occurs). This is simple to reason about (the code reads top-to-bottom exactly in the order things happen) but has a severe scaling consequence directly tied to the C10K problem you covered in [08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md#the-c10k-and-c10m-problem):

**Why thread-per-connection does not scale to millions:** the naive design - spawn one OS thread per connected socket, let each thread block on `recv()` for that client - works fine for hundreds or low thousands of connections, but breaks down well before a million for two compounding reasons: (1) **memory** - each OS thread carries a stack (commonly a default of hundreds of KB to a few MB, `verify` exact default varies by OS/runtime) plus kernel-side scheduling structures, so 100,000 threads alone can consume gigabytes before a single byte of actual connection data is stored; (2) **context-switch cost** - the OS scheduler has to time-slice across all these threads, and the CPU-cache-flushing, register-saving overhead of switching between thousands of runnable-or-blocked threads grows with thread count, eating CPU that should be doing application work. Long-lived connections (WebSockets, SSE, keep-alive HTTP) make this acute specifically because the threads sit blocked for a long time relative to the tiny amount of actual data exchanged.

**The fix - non-blocking sockets:** a socket can be put into non-blocking mode, where `recv()`/`send()` return **immediately** with a special "would block" error instead of parking the thread if no data is ready / no buffer space is available. This on its own doesn't solve anything by itself - naively looping and re-calling `recv()` (a busy-wait/spin loop) would burn 100% CPU checking sockets that have nothing to offer. What non-blocking mode actually enables is the next piece: an **event loop**.

## IO multiplexing: select, poll, epoll, kqueue

An **event loop** is a single thread that asks the kernel, in one call, "which of these thousands of sockets actually has something ready right now?" and only then does work on the ones that do - never blocking on any individual socket, never spinning uselessly on ones with nothing ready. This is precisely the mechanism `nginx`, `Redis`, `Node.js` (via `libuv`), and `Netty`/`Netty`-based JVM servers use to serve tens of thousands to millions of concurrent connections from a small, fixed number of OS threads (canonical, textbook designs - no additional citation needed beyond naming them). The multiplexing call itself has evolved through three generations, each fixing the previous one's core scaling flaw:

- **`select()`** - the original Berkeley call: give the kernel a bitmask of file descriptors to watch (readable/writable/error), it blocks until at least one is ready, and returns. Two structural limits: it typically caps the number of file descriptors it can watch (historically 1024, via `FD_SETSIZE`), and - the real scaling killer - **every call requires the kernel to scan every watched descriptor, O(n)**, and the *userspace program* also has to re-scan the returned bitmask to figure out which ones fired. At a few thousand sockets this scanning cost, repeated on every single event-loop iteration, dominates.
- **`poll()`** - removes the fixed fd-count limit (you pass a dynamically sized array instead of a fixed bitmask) but keeps the same fundamental flaw: it is still an **O(n) scan** of the whole watched set on every call, in both the kernel and userspace.
- **`epoll()` (Linux)** - fixes the scanning problem structurally: instead of re-describing your entire set of watched sockets on every call, you register interest **once** per socket (`epoll_ctl`), and the kernel maintains that interest list persistently, updating a small ready-list internally (typically via callbacks triggered when a socket's state changes) as events happen. Then `epoll_wait()` only needs to hand back the sockets that are actually ready - **proportional to the number of ready events, not the number of watched sockets** (commonly summarized as roughly O(1) per-event rather than O(n) per-call, `verify` precise complexity characterization varies by kernel implementation detail). This is the difference that lets a single event-loop thread watch hundreds of thousands of mostly-idle long-lived connections (exactly the WebSocket/SSE C10K scenario) and only pay cost proportional to the handful that actually have data, not the whole population.
- **`kqueue()` (BSD/macOS)** - the BSD-family equivalent of `epoll`, same core idea (persistent, kernel-tracked interest registration plus an efficient readiness-retrieval call), same scaling properties, different API surface. `IOCP` on Windows fills the analogous role with a somewhat different (completion-based rather than readiness-based) model - `verify` if citing IOCP specifics.

**Why this is exactly the C10K/C10M answer:** the C10K problem (Dan Kegel) was fundamentally the observation that thread-per-connection and `select()`-based multiplexing both fail to scale to ten thousand-plus concurrent connections for the reasons above; `epoll`/`kqueue` plus an event-loop architecture is the concrete, standard solution the industry converged on, and it is precisely what sits underneath the "how does one process hold a million WebSocket connections" answer you were pointed toward in the previous topic.

## Unix domain sockets

A **Unix domain socket** (address family `AF_UNIX`) uses the **exact same socket API** - `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`/`recv()` - but instead of an IP address and port, it's addressed by a **path on the local filesystem** (e.g. `/var/run/myapp.sock`), and communication never leaves the host: there's no IP stack, no TCP/IP headers, no checksumming, no loopback network interface traversal at all - the kernel just copies bytes directly between the two processes' buffers. This makes Unix domain sockets meaningfully faster and lower-overhead than even `localhost` TCP (`127.0.0.1`) for same-host inter-process communication, because they skip the entire network-stack processing that even a loopback TCP connection still formally goes through.

This is the standard mechanism for **sidecar** and same-host service-to-service communication - a common pattern in service mesh architectures (forward-ref) where an application container talks to a local proxy (e.g. an Envoy sidecar) over a Unix domain socket rather than a network port, and it's also how many reverse proxies talk to local application processes (e.g. nginx to a local application server) when both run on the same host. The trade-off is the obvious one: Unix domain sockets only work for same-host communication - the moment two processes are on different machines, you're back to `AF_INET`/`AF_INET6` sockets over the real network stack.

## Key OS-level details

A handful of kernel-level facts about sockets directly explain production behavior you've already touched on in earlier topics:

- **The accept queue / SYN backlog** - the `backlog` argument to `listen()` sizes the queue of connections the kernel will hold, ready for `accept()` to consume. This queue is exactly what a **SYN flood** attack (see [04-tcp.md](04-tcp.md#syn-flooding-and-the-half-open-attack)) targets - by flooding half-open (SYN-received, never ACK'd) entries, an attacker can exhaust this queue's capacity so legitimate completed connections have nowhere to sit and get dropped.
- **Socket send/receive buffers** - every connected socket has kernel-managed send and receive buffers (`SO_SNDBUF`/`SO_RCVBUF`); TCP's advertised **receive window** in [04-tcp.md](04-tcp.md#flow-control-protecting-the-receiver) is directly derived from how much free space is left in the receiving socket's buffer - flow control is not an abstract protocol concept floating independently, it is the kernel reporting real buffer occupancy on the actual socket object.
- **TIME_WAIT accumulation** - the side that initiates connection close (calls `close()` first) is the side whose socket lingers in the `TIME_WAIT` state for a period (commonly ~60 seconds `verify` exact default varies by OS) after teardown, as covered in [04-tcp.md](04-tcp.md#connection-termination). A busy server that closes many short-lived connections rapidly can accumulate large numbers of `TIME_WAIT` sockets, each still consuming a small amount of kernel memory and (depending on OS defaults) occasionally constraining new connections on the same 4-tuple until the state expires - a well-known production tuning concern (`SO_REUSEADDR`, adjusting the OS's `TIME_WAIT` handling, or architecting to have the *client* close first) rather than a bug.
- **Ephemeral port exhaustion (client-side)** - when a client calls `connect()` without an explicit local port, the OS auto-assigns one from a limited **ephemeral port range** (commonly a few tens of thousands of ports, e.g. `32768-60999` on many Linux defaults, `verify` exact range varies by OS/config). A single client machine making very high volumes of outbound connections to the *same* destination IP:port (a common pattern for a service calling one downstream heavily) can exhaust this range - each distinct 4-tuple needs its own free ephemeral port against that same destination, and once they're all in use (including many stuck in `TIME_WAIT`), new outbound connections fail. This is a genuine, frequently-hit real-world scaling gotcha for high-throughput proxies/load balancers, and is one of the concrete reasons connection pooling and connection reuse (keep-alive, forward-ref L2/L3) matter operationally, not just for latency.
- **File-descriptor limits (`ulimit`)** - since every socket is a file descriptor, and an OS process (and the OS overall) has a configurable maximum number of open file descriptors at once (`ulimit -n` on Unix, plus a system-wide ceiling), this is a hard ceiling on how many simultaneous connections a single process can hold - regardless of how efficient your event loop is, if the fd limit is 1,024 you cannot exceed roughly that many concurrently open sockets. Raising this limit (both per-process and system-wide) is a standard, necessary step in tuning any server meant to hold tens of thousands or more concurrent connections.

## Worked example: accepting one client, fd by fd

A web server process is already running with a listening socket bound to `0.0.0.0:80`, sitting at file descriptor `3`, backlog set to 128. Trace exactly what happens when one browser connects:

1. **Before the client connects:** the process's open file descriptors include `0` (stdin), `1` (stdout), `2` (stderr), and `3` (the listening socket). Fd 3 has no remote peer - it is purely in listening mode.
2. **The client's OS calls `connect()`**, sending a SYN. The server's kernel - not the application - receives it, and (independent of whether the application has called `accept()` yet) manages the handshake: SYN received, SYN-ACK sent, final ACK received. The kernel now places a **fully-established connection** into fd 3's accept queue. The application code has done nothing yet; this all happened inside the kernel.
3. **The application (already blocked in, or now calling) `accept(3)`** - the kernel dequeues that completed connection from fd 3's queue and creates a brand-new socket object for it, assigning it the next available file descriptor - say **fd 7**. `accept()` returns `7` to the application. Fd 3 is unchanged, still listening, immediately able to receive the next client's handshake and queue it too.
4. **The server calls `recv(7, buffer, ...)`** and reads the raw bytes of the client's HTTP request (`GET /index.html HTTP/1.1\r\nHost: example.com\r\n...`) directly off fd 7 - this socket, and only this socket, is associated with this one client's 4-tuple.
5. **The server calls `send(7, response_bytes, ...)`**, writing the HTTP response (`HTTP/1.1 200 OK\r\n...`) back out on the same fd 7 - the kernel hands these bytes to TCP, which segments, sequences, and eventually transmits them to exactly this client, using the state associated with fd 7's socket object.
6. **The server calls `close(7)`** (assuming non-keep-alive, or after the keep-alive timeout) - fd 7 is released, the TCP FIN/ACK teardown sequence runs for that one connection, and fd 7 becomes available for reuse by some future, unrelated socket. Fd 3, the listening socket, was never touched throughout any of this and is simultaneously handling other clients' handshakes in its own accept queue the entire time.

The concrete takeaway: **fd 3 (listening) never carries a single byte of HTTP traffic; fd 7 (connected) carries all of it for exactly one client** - and if a hundred more clients connect concurrently, the server ends up with a hundred more connected sockets (fd 8, 9, 10, ...), all fanned out from that same one listening socket on port 80.

## Trade-offs and common confusions

| Confusion | The precise distinction |
| --- | --- |
| "Socket" vs "port" | A port is a number (a coordinate); a socket is a stateful kernel object (an endpoint) that may be bound to a port. |
| "Socket" vs "connection" | A socket is one side's local object; a connection is the pairing of two sockets (client's and server's) identified together by the 4-tuple. |
| Listening socket vs connected socket | A listening socket accepts new connections and never carries data; a connected socket carries data for exactly one peer and never accepts anything. |
| Blocking IO vs event-driven IO | Blocking is simpler to write (linear code, one thread per connection) but does not scale past low thousands of connections due to memory + context-switch cost; event-driven (`epoll`/`kqueue` + non-blocking sockets) scales to hundreds of thousands-plus per process but requires callback/async-style code structure. |
| Thread-per-connection vs event loop | Thread-per-connection trades simplicity for a hard ceiling around thread memory/scheduling overhead; an event loop trades some code complexity for near-flat memory cost per idle connection - which is why long-lived-connection-heavy workloads (WebSockets, SSE) almost universally use the event-loop model. |
| `select`/`poll` vs `epoll`/`kqueue` | Both answer "which sockets are ready?" but `select`/`poll` re-scan the entire watched set every call (O(n)), while `epoll`/`kqueue` maintain persistent kernel-side registration and return only the ready subset - this difference is *the* reason C10K-scale servers exist. |

✅ Sockets let TCP/UDP/HTTP/TLS/WebSockets all be implemented as ordinary read/write on a file descriptor - no protocol needs its own bespoke IO mechanism.
✅ One listening socket, many connected sockets - a single port serves unlimited clients because the 4-tuple, not the port, is the unique key.
❌ A blocking, thread-per-connection design is easy to write but hits a hard wall well before a million concurrent connections.
❌ `select()`'s O(n) scan makes it a poor fit past roughly low thousands of watched sockets - it is a legacy API kept mainly for portability/simplicity in small-scale tools.

> [!IMPORTANT]
> A socket is the OS's file-descriptor-based handle for a network endpoint - the primitive underneath every protocol you have studied in this level. The single most important structural fact is that a **listening socket and a connected socket are different objects**: `accept()` never modifies the listening socket, it only manufactures a new connected socket per client, which is exactly how one server port serves unbounded numbers of simultaneously connected clients (each identified by its own unique 4-tuple). And the single most important scaling fact is that blocking, one-thread-per-socket designs cannot reach C10K/C10M scale - `epoll`/`kqueue`-driven event loops on non-blocking sockets can, because they let one thread cheaply ask "which of my thousands of sockets is actually ready?" instead of blocking on each one or scanning all of them every time.

## Connects to

- **Back to [04-tcp.md](04-tcp.md)** - the 4-tuple that identifies a TCP connection is exactly what makes a connected socket unique; `connect()`'s handshake and `close()`'s FIN teardown are the same state machine covered there, now seen from the syscall side.
- **Back to [05-udp.md](05-udp.md)** - `SOCK_DGRAM` sockets and `sendto()`/`recvfrom()` map directly onto UDP's connectionless, no-handshake model; `connect()` on a UDP socket is a local convenience only, never a network operation.
- **Back to [08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md)** - a WebSocket, after the HTTP upgrade handshake, *is* a long-lived connected TCP socket held open indefinitely; the C10K/C10M problem raised there is answered concretely here by `epoll`/`kqueue`-based event loops.
- **Back to [07-https-tls.md](07-https-tls.md)** - TLS is a layer that wraps a socket's byte stream (encrypting/decrypting what would otherwise be plaintext `send()`/`recv()` calls) - it does not replace the underlying socket, it sits on top of it.
- **Forward to load balancers** - an L4/L7 load balancer is, at its core, a program juggling enormous numbers of client-facing and backend-facing sockets simultaneously, using exactly this multiplexing machinery, then proxying bytes between the two connected sockets for each flow.
- **Forward to connection pooling (L2/L3)** - a "connection pool" is literally a cache of already-connected sockets kept open and reused across requests, specifically to avoid paying a fresh `connect()` handshake (and, for HTTPS, a fresh TLS handshake) per request.
- **Forward to service mesh / API gateways** - sidecar proxies commonly communicate with the local application over a Unix domain socket rather than a network socket, for the overhead reasons described above.
- **Forward to NAT** - a NAT device's translation table is, conceptually, tracking the same 4-tuple-level identity that a socket embodies locally, just from the perspective of a middlebox rather than an endpoint.

## Check yourself

- Explain precisely why a server can serve 50,000 simultaneous clients on port 443 using only one `bind()`/`listen()` call - what makes each client's connection distinct at the OS level?
- Where, exactly, does the TCP three-way handshake happen relative to the client's `connect()` call and the server's `accept()` call? Is `accept()` responsible for performing any part of the handshake?
- Why does thread-per-connection break down before reaching C10K/C10M scale, and what two costs specifically compound as thread count grows?
- What is the core difference between `select()`/`poll()` and `epoll()`/`kqueue()` that lets the latter pair scale to hundreds of thousands of watched sockets?
- A UDP socket calls `connect()`. What actually happens on the network as a result, and how does that differ from what happens when a TCP socket calls `connect()`?
- Why might a Unix domain socket be preferred over a `localhost` TCP socket for communication between an application and its local sidecar proxy?

## Real-world and sources

**nginx, Redis, and Node.js as canonical event-loop-on-epoll/kqueue designs.** All three are widely documented, textbook examples of a small, fixed number of threads (often one primary event-loop thread per worker) using non-blocking sockets plus `epoll` (Linux) or `kqueue` (BSD/macOS) to serve tens of thousands to hundreds of thousands of concurrent connections per process - the direct, standard-practice answer to the C10K problem raised in the previous topic. Netty (the JVM networking framework underlying many high-throughput Java services) follows the same event-loop-over-`epoll` model. These are stable architectural facts about well-known open-source systems, not claims requiring a web citation.

### Sources / further reading

- W. Richard Stevens, "UNIX Network Programming, Volume 1: The Sockets Networking API" - the canonical, exhaustive reference for the Berkeley sockets API covered in this topic.
- The original Berkeley Software Distribution (4.2BSD) sockets API, and its ongoing standardization as the POSIX sockets interface (IEEE Std 1003.1) - the stable, decades-old foundation this topic describes.
- `man 2 socket`, `man 2 bind`, `man 2 listen`, `man 2 accept`, `man 2 connect` (POSIX/Linux manual pages) - precise syscall semantics.
- `man 2 epoll_create` / `man 7 epoll` (Linux), `man 2 kqueue` (BSD/macOS) - the readiness-notification APIs discussed under IO multiplexing.
- Dan Kegel, "The C10K Problem" - the classic essay cataloguing why thread-per-connection and `select()`-based designs fail to scale, and surveying the event-driven alternatives that followed (referenced already in [08-websockets-sse-long-polling.md](08-websockets-sse-long-polling.md)).
