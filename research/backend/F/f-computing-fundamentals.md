# F. Computing Fundamentals

The mechanics a single machine actually performs -- CPU, memory, processes, I/O, storage, bytes, and time -- that every distributed system later is built on top of.

## Contents

1. [CPU and Memory Hierarchy](#cpu-and-memory-hierarchy)
2. [Processes vs Threads](#processes-vs-threads)
3. [Concurrency vs Parallelism and Context Switching](#concurrency-vs-parallelism-and-context-switching)
4. [Locks, Mutexes, Semaphores, Race Conditions, Deadlock, and Atomicity](#locks-mutexes-semaphores-race-conditions-deadlock-and-atomicity)
5. [IO Models: Blocking, Non-Blocking, Async, and Event Loops](#io-models-blocking-non-blocking-async-and-event-loops)
6. [OS Scheduling and Virtual Memory](#os-scheduling-and-virtual-memory)
7. [Disks and Filesystems](#disks-and-filesystems)
8. [Data Representation](#data-representation)
9. [Serialization](#serialization)
10. [Compression](#compression)
11. [Hashing](#hashing)
12. [Clocks](#clocks)

---

## CPU and Memory Hierarchy

**What it is.** A CPU cannot touch data directly from RAM fast enough to keep up with modern clock speeds, so hardware inserts a hierarchy of progressively larger, slower, cheaper storage between the CPU core and main memory: **registers -> L1 cache -> L2 cache -> L3 cache -> RAM -> disk/SSD**. Each level is roughly an order of magnitude bigger and slower than the one above it.

**Why it exists.** A CPU core executes billions of instructions per second, but DRAM (main memory) access takes ~100ns -- long enough for a modern CPU to execute several hundred instructions while it waits. Without caching, the CPU would spend nearly all its time stalled waiting on memory (the "memory wall"). Small, fast SRAM caches placed physically close to the core exploit the fact that real programs re-use a small "working set" of data and instructions repeatedly.

**How it works internally.**
- **Registers**: a handful of storage slots inside the CPU itself (tens of them), accessed in ~1 clock cycle. Holds the operands of the instruction currently executing.
- **L1 cache**: split into instruction (L1i) and data (L1d) caches, typically ~32-64 KB each, private per core, ~1ns / ~4 cycles latency.
- **L2 cache**: larger (~256 KB-1 MB), still usually private per core, ~3-10ns.
- **L3 cache** (last-level cache, LLC): shared across all cores on the chip, several MB to tens of MB, ~10-20ns.
- **RAM (DRAM)**: gigabytes, ~100ns latency -- roughly ~100x slower than L1.
- **SSD / disk**: covered in depth later in this doc; orders of magnitude slower again.

Approximate ("Latency Numbers Every Programmer Should Know" style, order-of-magnitude, mark all as `~`):

| Operation | Approx latency |
|---|---|
| L1 cache reference | ~1 ns |
| L2 cache reference | ~4-7 ns |
| L3 cache reference | ~10-20 ns |
| Main memory (RAM) reference | ~100 ns |
| Read 4 KB randomly from SSD | ~100-150 us |
| Read 1 MB sequentially from RAM | ~a few us |
| Read 1 MB sequentially from SSD | ~1 ms |
| Disk (HDD) seek | ~5-10 ms |
| Round trip within same datacenter | ~0.5 ms |

The exact numbers shift with hardware generation; what should stick is the *shape*: each hop down the hierarchy costs roughly 5-100x more than the one above.

**Cache lines.** Caches don't move single bytes; they move fixed-size chunks called **cache lines** (commonly **64 bytes** on x86/ARM). Reading one byte pulls in the surrounding 64 bytes. This is why:
- **Spatial locality** (accessing memory addresses near ones you just accessed) is fast -- the data is already in the cache line you just pulled in.
- **Temporal locality** (re-accessing the *same* address soon) is fast -- it's still resident in cache, not yet evicted.

**Why cache-friendly code matters.** Iterating a contiguous array is fast because each cache line load serves ~16 consecutive `int`s (64 bytes / 4 bytes). Chasing pointers through a linked list scatters data across memory -- each node access likely misses cache and pays the ~100ns RAM penalty. Row-major vs column-major iteration of a 2D array can be a 10x+ difference purely from cache-line utilization. On multi-core chips, **false sharing** is the inverse problem: two unrelated variables written by different threads but sitting on the same 64-byte cache line force the cache-coherence protocol (e.g., MESI) to ping-pong that line between cores' caches on every write, silently serializing what looked like independent work.

**Trade-offs.** More cache = fewer memory stalls but caches cost die area and power, and are shared resources that must be kept coherent across cores (adding protocol overhead). Software can't control the cache directly, but layout (structs of arrays vs arrays of structs, padding to avoid false sharing) has an outsized effect on real throughput -- often more than algorithmic big-O for small/medium data sizes.

**Where this resurfaces.** B-tree node sizes are chosen to match disk/OS page sizes and minimize cache misses (L2 storage engines); columnar formats and vectorized query engines exploit spatial locality heavily (L13); understanding this hierarchy is the foundation for reasoning about why "everything is a cache" from CPU to CDN.

---

## Processes vs Threads

**What a process is.** A **process** is an instance of a running program: it owns its own **virtual address space** (its own private view of memory), its own set of OS-managed resources (open file descriptors, sockets, signal handlers), and metadata the OS uses to schedule and account for it (a process ID, memory maps, etc.).

**What a thread is.** A **thread** is a unit of execution *within* a process. Multiple threads in the same process share that process's address space and most of its resources, but each thread has its own **stack**, its own **registers**, and its own **program counter** (instruction pointer) -- because each thread is independently being scheduled and needs to track "where am I in the code" and "what are my local variables" separately.

**What's shared vs not:**

| Shared across threads in a process | Private per thread |
|---|---|
| Heap memory / global variables | Stack (local variables, call frames) |
| Open file descriptors / sockets | CPU registers, program counter |
| Loaded code (.text segment) | Thread-local storage (explicit opt-in) |
| Memory mappings, signal handlers | Signal mask (per-thread on POSIX) |

A process's threads can all read/write the same heap object directly with no IPC needed -- which is exactly why threads are useful (cheap, fast shared-state concurrency) and exactly why they're dangerous (unsynchronized shared writes = race conditions, see the locks section below).

**Cost of each (approximate).** Creating a new process (`fork` on POSIX systems) requires the OS to set up a new virtual address space and duplicate the parent's page tables (modern OSes use **copy-on-write** so pages aren't physically copied until written to, but the page-table bookkeeping itself still costs something) -- typically ~ hundreds of microseconds to low milliseconds, further multiplied if followed by `exec` to load a new program image. Creating a new thread within an existing process only needs a new stack and register set within the *same* address space -- typically ~tens of microseconds, roughly 10-100x cheaper than a process. Switching *between* threads of the same process is also cheaper than switching between processes (see next section) because the virtual-to-physical memory mappings don't need to change.

**Why processes are isolated at all.** Isolation is a deliberate trade-off: a crash or memory corruption in one process cannot directly corrupt another process's memory, because the OS enforces separate address spaces via virtual memory (see the OS scheduling/virtual-memory section). This is why a browser puts each tab in its own process (one tab crashing doesn't take down the browser) and why a buggy worker process can be killed and restarted without touching its siblings. Threads sacrifice that isolation for speed and easy data sharing -- one thread corrupting shared heap state (e.g., via a data race or buffer overflow) can crash or corrupt the entire process, taking every other thread down with it.

**Inter-process communication (IPC).** Because processes don't share memory by default, cooperating processes need explicit channels: pipes, Unix domain sockets, TCP sockets, shared-memory segments, or message queues. This is strictly more expensive than a thread just calling a shared function, because data usually must be copied across the address-space boundary (and often serialized -- see the Serialization section).

**Lightweight/"green" threads.** Many modern runtimes multiplex many logical threads onto a small pool of OS threads: goroutines (Go), virtual threads (Java 21+), Erlang processes. These get much of the cheapness of "just spawn one per task" while the runtime scheduler (not the OS) handles the actual multiplexing, often at a fraction of the memory/switching cost of OS threads (an OS thread stack commonly reserves ~1-8 MB of address space per thread by default; green threads can start at a few KB and grow).

**Where this resurfaces.** Worker-pool sizing for request handling and background jobs (L10 service design) is directly a threads/process trade-off; process-level isolation as a fault-containment strategy reappears in L7 reliability (bulkheads); the "shared nothing, communicate via message passing" philosophy of processes is the same philosophy behind service-to-service messaging in L6.

---

## Concurrency vs Parallelism and Context Switching

**Concurrency** is a *structural* property: a system is concurrent if it is composed of multiple tasks that can be **in progress at overlapping times**, making progress by being interleaved -- it does not require more than one CPU core. **Parallelism** is an *execution* property: work is parallel if multiple tasks are literally **executing at the same physical instant**, which requires multiple cores (or machines). A single-core machine running an event loop juggling 10,000 open sockets is highly concurrent but has zero parallelism. A single CPU-bound loop split across 8 cores is parallel. A web server handling many requests across many cores is both.

**Why the distinction matters.** It changes what kind of problem you're solving: concurrency is about correctly *managing* interleaved, possibly overlapping tasks (ordering, shared-state safety, avoiding starvation) -- relevant even with one core. Parallelism is about *speeding up* work that can be split into independent chunks; adding cores only helps if the work is actually parallelizable (see Amdahl's Law-style reasoning: the non-parallelizable fraction caps your speedup regardless of core count).

**Context switching.** A **context switch** is the OS suspending the currently running thread/process and resuming another, so that (from the wall clock's perspective) many logical flows appear to progress "at once" even on limited cores. The OS timer interrupts the CPU periodically (a **preemptive** switch) or a thread voluntarily yields/blocks on I/O or a lock (a **voluntary** switch); either way, the OS's scheduler picks the next runnable thread and switches to it.

**What a context switch actually costs.** There are two components:
1. **Direct cost**: saving the outgoing thread's registers and program counter, loading the incoming thread's -- this is genuinely cheap, on the order of ~1-2 microseconds.
2. **Indirect cost (the bigger one)**: the new thread starts with **cold L1/L2 caches and a cold TLB** (translation lookaside buffer, see next section) relative to what it was using before it was last switched out. It has to re-warm these by taking a burst of cache misses (~100ns each) and possibly TLB misses, before it runs at full speed again. Switching between threads of *different processes* is more expensive still because the virtual address space itself changes, which on many architectures requires flushing address-translation caches that would otherwise be reused.

Because of this indirect cost, the *effective* cost of a context switch under real workloads is commonly estimated at several microseconds to tens of microseconds -- much larger than the raw register-save/restore time alone.

**Why this matters practically.** If you spawn far more runnable threads than you have CPU cores (e.g., one OS thread per incoming request under high concurrency), the scheduler spends an increasing fraction of wall-clock time context-switching between them rather than doing useful work -- throughput can actually *drop* as concurrency increases past a certain point (this is one root cause behind the C10K problem discussed in the IO section). This is the core justification for bounded worker pools, and for event-driven/async architectures that handle many logical tasks on few OS threads.

**Where this resurfaces.** Sizing a thread pool or connection pool (L10, L3) is fundamentally a concurrency-vs-context-switch-overhead trade-off; the same interleaving-vs-simultaneity distinction reappears conceptually in stream processing (L6) when reasoning about parallelism across partitions.

---

## Locks, Mutexes, Semaphores, Race Conditions, Deadlock, and Atomicity

**Race condition.** A **race condition** occurs when the correctness of a program depends on the relative timing of events that aren't otherwise constrained -- typically, two or more threads reading and writing shared state without coordination. The canonical example: `counter++` looks like one operation but is actually three machine steps (read `counter` into a register, add 1, write it back). If two threads interleave those three steps, one increment can be silently lost. The segment of code that touches the shared state unsafely is called the **critical section**.

**Mutex (mutual exclusion lock).** A binary lock: at most one thread may hold it at a time. A thread calls `lock()` before entering a critical section and `unlock()` after leaving it; if another thread already holds it, the caller blocks (is parked by the OS, not spinning) until it's released. Convention (and often OS enforcement) is that **only the thread that acquired the mutex may release it** -- it models strict ownership of a critical section.

**Semaphore.** A **semaphore** is a counter-based synchronization primitive that allows up to **N** concurrent holders rather than just one (a semaphore with N=1 is called a "binary semaphore" and looks similar to a mutex, but critically, unlike a mutex, a semaphore has no ownership requirement -- any thread can `signal`/`release` it, not just the one that `wait`ed on it). Semaphores are the natural tool for bounding concurrent access to a *pool* of identical resources: e.g., capping the number of concurrent outbound DB connections to 20 by initializing a semaphore with count 20 -- each caller acquires a permit before using a connection and releases it after, and the 21st caller blocks until a permit frees up.

**Spinlock vs blocking lock.** A **spinlock** busy-waits in a tight loop re-checking the lock instead of yielding the CPU; it wastes CPU cycles while waiting but avoids the cost of a context switch, so it's a good trade only when the critical section is expected to be extremely short (a handful of instructions) and you're on a multi-core system where the lock holder is likely running concurrently on another core. A regular (blocking) lock parks the waiting thread so the OS can run something else, which is the right choice for anything longer -- otherwise you burn CPU for nothing.

**Deadlock.** A **deadlock** is a state where two or more threads are each waiting on a resource the other holds, so none can ever proceed. The classic example: thread A holds lock 1 and wants lock 2; thread B holds lock 2 and wants lock 1. Deadlock requires all four **Coffman conditions** simultaneously: (1) **mutual exclusion** (resources can't be shared), (2) **hold and wait** (a thread holds one resource while waiting for another), (3) **no preemption** (a resource can't be forcibly taken away), (4) **circular wait** (a cycle of threads each waiting on the next). Breaking any one of the four prevents deadlock in practice -- the most common technique is enforcing a **global lock ordering** (always acquire locks in the same, agreed order across the whole codebase, e.g., always lock the lower account ID before the higher one in a funds-transfer routine), which eliminates circular wait by construction. Other techniques: lock timeouts / `tryLock` with backoff (break "no preemption" by giving up and retrying), or reducing the need for multiple simultaneous locks in the first place. Related but distinct failure modes: **livelock** (threads actively change state in response to each other but never make real progress -- e.g., both politely stepping aside repeatedly) and **starvation** (a thread is perpetually denied a resource it needs, e.g., by a scheduler or lock policy that always favors other threads).

**Atomicity.** An operation is **atomic** if it appears to the rest of the system as indivisible -- it either has not started or has fully completed, with no observable intermediate state. Modern CPUs provide hardware-level atomic instructions such as **compare-and-swap (CAS)** (atomically: "if the current value equals X, set it to Y, and tell me whether it succeeded") and fetch-and-add. These are the building blocks both for implementing higher-level locks and for **lock-free programming**.

**Lock-free basics.** A lock-free data structure achieves thread-safety using only atomic instructions (typically CAS loops: read the current value, compute the new value, CAS it in, and retry the whole thing if another thread beat you to it) instead of blocking locks. The payoff is avoiding lock contention overhead and a class of problems locks introduce (deadlock, priority inversion, and a slow/paused thread blocking everyone else waiting on its lock). The cost is that lock-free algorithms are substantially harder to design and reason about correctly, are subject to subtle bugs like the **ABA problem** (a value changes from A to B and back to A between your read and your CAS, so the CAS wrongly appears to have detected "no change"), and under high contention can burn CPU retrying rather than actually making forward progress much faster than a well-designed lock would. In practice, lock-free structures are used selectively in hot, highly-contended, short-critical-section paths (e.g., counters, queues in high-throughput messaging systems), not as a blanket replacement for locks.

**Where this resurfaces.** Database row/table locking and MVCC (L2) are this same set of concepts applied to concurrent transactions instead of threads; distributed locks and fencing tokens (L5) are the network-scale analog of a mutex, needed precisely because a plain mutex only coordinates threads on one machine; idempotency and exactly-once semantics (L5, L6) exist because you often can't get true atomicity across a network call the way a CAS instruction gives it to you on one CPU.

---

## IO Models: Blocking, Non-Blocking, Async, and Event Loops

**Blocking I/O.** The calling thread issues a system call (e.g., `read()` on a socket) and is **suspended** by the OS until the operation completes -- if no data is available yet, the thread simply doesn't run again until it is. Simple to program (code reads top-to-bottom) but ties up one whole thread per in-flight operation.

**Non-blocking I/O.** The socket/file descriptor is put into non-blocking mode; a `read()` call returns immediately, either with data or with an error meaning "nothing available right now, try again later" (`EWOULDBLOCK`/`EAGAIN`). This avoids tying up a thread, but naively looping and re-calling `read()` (**busy-polling**) wastes CPU.

**Multiplexing: select / poll.** `select()` and `poll()` let a single thread ask the OS "tell me which of these N file descriptors are ready to read/write" and then block efficiently until at least one is. The problem: both require the caller to pass (and the kernel to re-scan) the *entire* set of watched descriptors on every call -- `select()` is also capped at a small fixed number of descriptors (historically 1024, `FD_SETSIZE`) and both are effectively **O(n)** per call as the number of connections grows, which becomes the bottleneck at high connection counts.

**epoll (and kqueue/IOCP).** `epoll` (Linux; `kqueue` on BSD/macOS, IOCP on Windows conceptually similar) fixes this by letting the kernel maintain a persistent **interest list** of watched descriptors and hand back only the small **subset that's actually ready** each time you ask -- an **O(1)**-ish, event-driven design instead of re-scanning everything. This is the mechanism that makes it practical for one thread to efficiently watch tens or hundreds of thousands of sockets at once, and it's the kernel primitive underneath essentially every high-performance event loop (nginx, Redis, Node.js's libuv, Envoy).

**The event loop pattern.** A single thread repeatedly: (1) ask the OS (via epoll/kqueue) which sockets are ready, (2) for each ready socket, run its associated callback/continuation, (3) repeat. This works extremely well for **I/O-bound** workloads (lots of waiting on network/disk, little CPU work per request) because one thread can juggle huge numbers of concurrent connections without the memory and context-switch overhead of a thread per connection. It works poorly if a callback does **CPU-bound** or blocking work, because that stalls the *entire* loop -- every other connection waits -- which is why event-loop-based systems push CPU-heavy work to a separate worker/thread pool.

**True async I/O.** Readiness-based APIs (epoll/kqueue) tell you *when you may perform the I/O without blocking*, but you still issue the actual read/write call yourself. True asynchronous I/O (e.g., Linux `io_uring`, Windows IOCP) goes further: you submit the I/O operation itself to the kernel and are notified only on **completion**, letting the kernel do more work with fewer round-trips into the application -- generally the more scalable design for very high-throughput I/O, and an active area of evolution beyond the older epoll model.

**The C10K problem.** In the late 1990s, handling 10,000 simultaneous client connections on one server was a well-known wall for typical thread-per-connection designs: at a rough ~1-8 MB stack reservation per OS thread, 10,000 threads alone could demand multiple gigabytes of address space, plus the OS scheduler was doing an O(n) `select()`-style scan and heavy context switching between all of them, starving actual request-processing throughput. Event-loop architectures built on epoll/kqueue (popularized by servers like nginx) directly solved this by decoupling "number of concurrent connections" from "number of OS threads," and the same principle scales the discussion further today toward "C10M" (millions of concurrent connections per machine) using techniques like kernel-bypass networking.

**Trade-offs, summarized.** Thread-per-connection: simple mental model (write straight-line blocking code), but memory and context-switch costs scale badly with connection count. Event loop / async: scales connection count far better on a single core, but requires non-blocking-safe code throughout (a single accidental blocking call stalls everyone) and can be harder to reason about (callback chains, or `async`/`await` sugar over the same underlying model).

**Where this resurfaces.** Reverse proxies, load balancers, and API gateways (L1) are almost universally epoll/event-loop-based for exactly this reason; backpressure and worker-pool design for request handling (L10, L6) build directly on "don't block the event loop"; this is also the mechanical reason WebSocket/SSE servers (L13) can hold huge numbers of idle-but-open connections cheaply.

---

## OS Scheduling and Virtual Memory

**OS scheduling.** The **scheduler** is the part of the OS kernel that decides which runnable thread/process gets the CPU next and for how long. Modern general-purpose OSes use **preemptive multitasking**: the scheduler can interrupt a running thread (via a periodic timer interrupt) even if it hasn't voluntarily yielded, ensuring no single thread can monopolize a core forever. Linux's default scheduler (CFS, the Completely Fair Scheduler, historically; newer kernels use EEVDF) approximates giving every runnable thread an equal share of CPU time over time by tracking each thread's accumulated "virtual runtime" and always picking whichever runnable thread has received the least CPU time so far, with priority/niceness values weighting that share. Typical scheduling time slices are on the order of a few to tens of milliseconds, not fixed but dynamically adjusted based on load.

**Virtual memory -- what and why.** **Virtual memory** gives every process the illusion of its own private, contiguous address space (e.g., 0 to some very large number), completely independent of where its data physically lives in RAM. This exists for three main reasons: (1) **isolation** -- one process cannot accidentally or maliciously read/write another's memory, since it can't even address it; (2) **simplicity** -- a program can be written/compiled as if it owns all of memory starting at address 0, without needing to negotiate physical layout with every other running program; (3) it enables **overcommit and swapping** -- the sum of all processes' virtual address spaces can exceed physical RAM, with the OS transparently moving less-used data to disk.

**Paging.** Virtual memory is implemented by dividing the address space into fixed-size **pages** (commonly **4 KB**; many systems also support **huge pages**, e.g., 2 MB or 1 GB, to reduce translation overhead for large workloads like databases) and physical RAM into equal-size **frames**. A per-process **page table** maps virtual pages to physical frames. Because a full page table for a large address space would itself be huge, real systems use **multi-level (hierarchical) page tables**, which only allocate table entries for the regions of the address space actually in use.

**Page faults.** A **page fault** happens when a program accesses a virtual page that has no current valid mapping to a physical frame. A **minor fault** is cheap -- the page exists in memory already (e.g., shared with another mapping, or was reclaimed but not yet reused) and the OS just fixes up the mapping. A **major fault** is expensive -- the OS must actually fetch the page's contents from disk/swap, costing something in the disk-latency ballpark (potentially milliseconds), which is why excessive major faults devastate performance.

**TLB (Translation Lookaside Buffer).** Walking the (multi-level) page table on every single memory access to translate virtual to physical addresses would be far too slow, so the CPU keeps a small, very fast on-chip cache of recent virtual-to-physical translations called the **TLB**. A TLB hit resolves an address in about as much time as an L1/L2 cache access; a **TLB miss** requires walking the page table structures in memory (multiple extra memory accesses) before the translation is known -- meaningfully slower, which is one reason huge pages help large-memory applications (a huge page covers far more address space per single TLB entry, so fewer TLB misses for a given working set).

**Swapping and thrashing.** When physical RAM is full, the OS can evict less-recently-used pages out to a disk **swap area** to free frames for active data, then page them back in on demand. This works fine occasionally, but if a process's active **working set** (the set of pages it's genuinely using) exceeds available RAM, the system can enter **thrashing**: it spends most of its time paging data in and out rather than doing useful work, because by the time it pages something back in, something else it needs has already been evicted. This is one reason an out-of-memory (**OOM**) killer exists on Linux -- rather than let the whole system thrash indefinitely, it sacrifices a process to restore headroom.

**Why this matters practically.** It explains why a process's reported memory footprint distinguishes **virtual size** (address space reserved, can be huge and mostly unused) from **resident set size / RSS** (physical RAM actually backing it); why databases and caches tune their memory carefully to avoid swap (swapping a database's working set is catastrophic for latency); and why huge pages are a common tuning knob for large in-memory workloads.

**Where this resurfaces.** A database's **buffer pool** manages hot pages in memory vs cold pages on disk using essentially the same eviction logic as OS virtual memory (L2 storage engines); capacity planning and "don't let the box swap" are recurring operational concerns in L7 reliability.

---

## Disks and Filesystems

**HDD (hard disk drive).** A mechanical device: data lives on spinning magnetic platters, read by a physical head that must **seek** (move the arm to the right track) and then wait for the **rotational latency** (wait for the right spot on the platter to rotate under the head). A typical seek costs ~5-10 ms and rotational latency at 7200 RPM adds a few more milliseconds, so a single random I/O on an HDD costs roughly **~10 ms** -- while sequential I/O (reading contiguous data, no repeated seeking) can sustain roughly ~100-200 MB/s. This asymmetry (random costs orders of magnitude more than sequential) is the single most important fact about HDDs.

**SSD (solid-state drive).** No moving parts -- data is stored in NAND flash cells. Random reads are dramatically faster than HDD (no mechanical seek), commonly in the tens-to-low-hundreds of microseconds. However, flash has its own quirk: it's organized into **pages** (commonly 4-16 KB, the unit of read/write) grouped into larger **blocks** (hundreds of KB to a few MB, the unit of erase), and flash can only be *written* to a page that has first been fully **erased** -- you cannot simply overwrite in place the way you can on disk or RAM.

**Write amplification.** Because pages can't be overwritten without an erase, and erase only happens at the whole-block granularity, an SSD controller handling a small logical write often has to: find/allocate a free page elsewhere (never overwrite in place), and periodically run **garbage collection** that copies any still-live pages out of a mostly-stale block before erasing and reclaiming it. The result is that the actual number of bytes physically written to the flash media can exceed the number of bytes the application logically asked to write -- this ratio is **write amplification**. High write amplification both slows effective write throughput and shortens the flash's usable lifetime (flash cells wear out after a bounded number of erase cycles). SSD firmware mitigates this with **over-provisioning** (reserving spare flash capacity to make garbage collection cheaper), **wear leveling** (spreading writes evenly across cells so none wears out early), and the **TRIM** command (the OS tells the SSD which logical blocks are no longer in use, so the controller doesn't waste effort preserving stale data during garbage collection).

**NVMe.** NVMe is not a storage medium -- it's a **host controller interface/protocol** for talking to flash storage, designed to replace the older SATA/AHCI interface (which was designed decades earlier around HDD assumptions: a single command queue, deep but not built for massive parallelism). NVMe connects storage directly over PCIe lanes and supports many (thousands of) parallel command queues, letting an SSD's inherent internal parallelism actually be exploited. The practical effect: NVMe SSDs commonly deliver on the order of hundreds of thousands to over a million IOPS (I/O operations per second), versus roughly ~100,000 IOPS ceilings typical of SATA SSDs and only ~100-200 IOPS for a mechanical HDD -- figures approximate and highly dependent on the specific device, but the *relative* gap (HDD << SATA SSD < NVMe SSD, often by multiple orders of magnitude) is the important takeaway.

**Sequential vs random I/O, restated.** HDDs are catastrophically slower at random I/O than sequential (often 100x+) because of physical seek/rotation costs; SSDs narrow that gap enormously for *reads* (no mechanical penalty), but random *writes* still tend to be less efficient than sequential writes on SSDs because of the erase-before-write/garbage-collection dynamics above -- sequential writes let the controller fill and eventually erase whole blocks cleanly, minimizing amplification.

**Filesystems.** A **filesystem** (ext4, XFS, APFS, NTFS, etc.) is the abstraction layered on top of a raw block device that organizes bytes into files and directories, tracks free space, and stores metadata (permissions, timestamps, and a mapping from a file to the disk blocks/pages holding its data -- an **inode** on Unix-like systems). Applications almost never talk to raw disk blocks directly; they go through the filesystem, which itself has to manage the same sequential-vs-random and write-amplification trade-offs internally (e.g., journaling filesystems write metadata changes to an append-only journal for crash-consistency, an application of the same "sequential writes are cheap and safe" idea used elsewhere).

**Why this matters for everything built on top.** Storage engines are designed *around* these physical realities: a **write-ahead log (WAL)** is append-only (sequential writes) specifically because sequential writes are cheap and safe on both HDD and SSD; an **LSM-tree** (log-structured merge-tree) converts random writes into sequential writes to an in-memory structure plus periodic sequential-write "compaction" of sorted files on disk, explicitly trading some read amplification and background CPU/I/O for much better write throughput than an update-in-place structure would get on spinning disks (and still favorably on SSDs, given the amplification concerns above).

**Where this resurfaces.** WAL, B-trees vs LSM-trees, and storage-engine internals generally (L2) are a direct continuation of this section; wide-column stores like Cassandra/HBase built on LSM-trees (L4) inherit these exact trade-offs.

---

## Data Representation

**Bits and bytes.** A **bit** is the smallest unit of information, 0 or 1. A **byte** is 8 bits, capable of representing 256 distinct values (0-255 unsigned, or -128 to 127 signed using **two's complement**, the standard scheme where negative numbers are represented so that ordinary binary addition still works correctly across positive and negative values).

**Binary and hexadecimal.** Binary (base 2) is what hardware natively operates on; hexadecimal (base 16, digits 0-9 and A-F) is used by humans as a compact, easy-to-convert stand-in for binary because each hex digit corresponds to exactly 4 bits -- so one byte is always exactly 2 hex digits (e.g., `0xFF` = `11111111` = 255). This is why memory addresses, hashes, and color codes are conventionally shown in hex.

**Character encoding.** A **character encoding** maps human-readable text to bytes. **ASCII** is a 7-bit encoding (128 possible values) covering unaccented English letters, digits, and basic punctuation -- sufficient for 1960s American computing but not for the rest of the world's scripts, emoji, or even accented Latin letters. **Unicode** solves this by defining a single universal mapping from "a character/symbol" to a numeric **code point** (e.g., `U+0041` for "A", `U+1F600` for the grinning-face emoji) covering essentially every script in current and historical use, plus symbols and emoji -- currently on the order of ~150,000 assigned code points (`~`, grows with each Unicode release). Unicode itself is just the *numbering scheme*; it doesn't say how those numbers are turned into bytes -- that's the job of an encoding like UTF-8.

**UTF-8.** A **variable-length** encoding of Unicode code points into 1 to 4 bytes: code points 0-127 (the original ASCII range) encode as a single byte identical to ASCII, so any valid ASCII text is automatically valid UTF-8 (full backward compatibility); less common characters cost more bytes. This combination -- compact for English/Latin text, full Unicode coverage, and drop-in ASCII compatibility -- is why UTF-8 became the dominant encoding on the web and in most modern systems and file formats. **UTF-16** (2 or 4 bytes per code point) is used internally by some runtimes and platforms (JavaScript string internals, Java, Windows APIs) largely for historical reasons predating UTF-8's dominance.

**Endianness.** When a multi-byte value (e.g., a 4-byte integer) is stored in memory or sent over a network, its bytes have to go in *some* order. **Big-endian** stores the most-significant byte first (the natural way a human writes a number, left to right); **little-endian** stores the least-significant byte first. x86 and most ARM configurations use little-endian internally; network protocols historically standardized on big-endian, commonly called **"network byte order"**, precisely so that machines with different native endianness could exchange binary data unambiguously by agreeing on one wire format regardless of local CPU convention. Getting this wrong silently corrupts binary data (a classic bug: reading a 4-byte integer written by a big-endian process on a little-endian machine without conversion yields a garbage value, not a crash) -- text formats like JSON have no such issue since they represent numbers as sequences of ASCII/UTF-8 digit characters, not raw multi-byte binary values.

**Why full-stack engineers should care.** Text-encoding mismatches cause "mojibake" (garbled text) when the byte stream is decoded with the wrong assumed encoding; binary serialization formats and file formats (images, audio, network packets) must define their byte order explicitly and unambiguously, and any code that manually parses binary data (rather than going through a library) needs to respect it.

**Where this resurfaces.** Directly underpins the next two sections: serialization formats define exact byte layouts (including endianness, for binary ones), and hash functions/checksums operate over raw bytes regardless of what they logically represent.

---

## Serialization

**What it is.** **Serialization** is converting an in-memory data structure (an object, a struct, a record) into a linear sequence of bytes suitable for storage or transmission; **deserialization** is the reverse. Every network call between services, every write to a file or a message queue, and every cache entry involves serialization somewhere.

**Text-based formats.** **JSON** (JavaScript Object Notation) represents data as nested objects/arrays of strings, numbers, booleans, and null, using human-readable text; it's ubiquitous in REST APIs because it's easy to read, debug, and produce/consume from virtually any language without a code generation step -- but it's self-describing at the cost of repetition (every record repeats its field *names*, not just values) and generally larger and slower to parse than binary alternatives for the same logical data. **XML** is an older, similarly text-based and self-describing format, more verbose still (explicit open/close tags), with schema definition via XSD/DTD; it remains common in enterprise integration and legacy SOAP-based APIs.

**Binary, schema-based formats.** **Protobuf** (Protocol Buffers, from Google), **Avro** (from the Hadoop ecosystem), and **Thrift** (originally Facebook, now Apache) are all compact binary serialization formats that require a predefined **schema** describing the fields and their types, and generate typed code in your target language from that schema.
- **Protobuf**: fields are declared in a `.proto` file with explicit numeric **field tags** (e.g., `string name = 1;`); the wire format encodes each field as a compact tag+type+value (using variable-length integers, "varints," for small numbers so they take fewer bytes) rather than repeating field names -- much smaller than the equivalent JSON for structured, numeric-heavy data. **Schema evolution** rules follow directly from this design: you can freely add new fields (old readers simply ignore unknown tags; missing fields on the reader side get default values), but you must never reuse or repurpose an existing field number, and removed fields' numbers should be explicitly reserved so they're never accidentally reassigned with different meaning later.
- **Avro**: the schema (itself expressed in JSON) travels with the data or, more commonly in streaming pipelines, is looked up from a shared **schema registry** rather than embedded per-message. Critically, Avro's encoded bytes don't contain field tags/names at all -- decoding requires knowing the exact writer's schema (or both the writer's and a compatible reader's schema, which Avro's design explicitly supports resolving), which makes it a natural fit for systems like Kafka where producer and consumer schemas may evolve independently over time, coordinated through a central registry.
- **Thrift**: conceptually similar to Protobuf (schema-defined, binary, compact, tagged fields), but historically bundled together with its own cross-language RPC framework, the same way gRPC bundles Protobuf with an HTTP/2-based RPC layer.

**Schema evolution, generally.** The core concern is compatibility across time: **backward compatibility** means new code can still read data written by old code (schema); **forward compatibility** means old code can still read data written by new code. The safe pattern across all these binary formats is the same: identify fields by a stable tag/number (not by their position or name alone), only ever *add* optional fields with sensible defaults, and never change the meaning of an existing tag.

**Text vs binary, the trade-off.** JSON/XML win on human-readability, debuggability (you can `curl` an endpoint and read the response), and zero-tooling interoperability (every language ships a JSON parser). Binary schema-based formats win on payload size (structured, numeric-heavy payloads can commonly be several times smaller in Protobuf than the equivalent JSON, `~`, highly dependent on the actual data shape -- string-heavy payloads shrink less), (de)serialization speed (no text parsing, no repeated key-name overhead), and compile-time type safety via generated code. The practical rule of thumb: public-facing, loosely-coupled, human-debugged APIs lean JSON/REST; high-throughput, tightly-coupled internal service-to-service calls and large data pipelines lean binary/schema'd.

**Where this resurfaces.** Kafka pipelines built on Avro plus a schema registry are a direct application of this section (L4, L6); the REST/JSON vs gRPC/Protobuf decision is this same trade-off restated at the API-design level (L10); Avro and Protobuf-derived columnar formats (Parquet) underpin data-lake storage (L13).

---

## Compression

**What it is.** **Compression** reduces the number of bytes needed to represent data by exploiting redundancy in it. This section covers **lossless** compression (the original data is reconstructed exactly) rather than lossy compression (images/video/audio accepting approximation), which is out of scope for this foundations level.

**How it works, conceptually.** Real-world data is rarely random -- it has statistical structure: repeated substrings, uneven symbol frequencies, predictable patterns. Two classical techniques exploit this: **dictionary coding** (the LZ family, e.g., LZ77) scans for previously-seen repeated sequences and replaces later occurrences with a short back-reference ("go back N bytes, copy M bytes") instead of repeating the literal bytes; **entropy coding** (Huffman coding, or more modern arithmetic/range coding) assigns shorter bit-codes to more frequent symbols and longer codes to rarer ones, so the average bits-per-symbol drops below what a fixed-width encoding would need. Most practical general-purpose compressors combine both ideas.

**gzip.** Implements the **DEFLATE** algorithm: LZ77-style dictionary matching followed by Huffman coding of the result. Good, well-understood general-purpose compression ratio at moderate speed; extremely widely supported (e.g., HTTP's `Content-Encoding: gzip`).

**Snappy.** Designed at Google explicitly prioritizing **speed over ratio** -- it deliberately compresses less than gzip in exchange for being much faster to compress and decompress, aimed at systems where compression sits directly on a hot read/write path (e.g., inside BigTable, Cassandra, historically Hadoop) and burning CPU on heavier compression would hurt overall throughput more than the extra bytes saved would help.

**LZ4.** Pushes the speed end even further than Snappy -- extremely fast decompression (commonly quoted in the multi-GB/s range) at a still-lower compression ratio; a common choice where CPU is the tightest resource, such as compressing message batches in real-time pipelines or Kafka topics where producers/consumers are latency-sensitive.

**Zstd (Zstandard).** A newer algorithm (from Facebook/Meta) that broke the old "pick a point on the speed-vs-ratio line and live with it" framing: at comparable speed to LZ4 it can match or beat gzip's compression ratio, and it exposes **tunable compression levels** (roughly 1 = fastest/lowest ratio, up to the high teens/twenties = slowest/highest ratio) so the same algorithm can be dialed toward either end of the trade-off per use case. This flexibility is a major reason Zstd has been adopted as a modern default across many systems (Kafka, the Linux kernel, RocksDB, and others support it directly), often displacing gzip.

**The central trade-off: speed vs ratio.** Fast-but-lower-ratio (LZ4, Snappy) suits hot paths and real-time systems where CPU time is scarce and data is compressed/decompressed constantly and repeatedly. Slow-but-higher-ratio (gzip at high settings, Zstd at max level, or LZMA/xz for archival) suits data compressed once and read rarely, such as cold storage or backups, where minimizing storage footprint matters more than compression CPU cost. There is no universally "best" choice -- it's always a deliberate trade-off matched to the access pattern.

**Where used.** HTTP response compression (network transfer size, L1); Kafka producer-side batch compression (network and disk footprint, L6); columnar data-lake formats like Parquet applying per-column compression (L13); database page/block compression (L2); log shipping and aggregation pipelines (L8).

**Where this resurfaces.** Choosing a Kafka compression codec is a direct, concrete instance of this trade-off (L6); storage-cost optimization for data lakes and warehouses revisits it at a larger scale (L13, L14).

---

## Hashing

**Hash function, defined.** A **hash function** is a deterministic function that maps an input of arbitrary size to an output of fixed size (the **hash** or **digest**). A good general-purpose hash function distributes outputs close to uniformly across its output range, and exhibits the **avalanche effect**: a tiny change to the input (flipping one bit) should produce a drastically different, unpredictable-looking output.

**Collisions.** A **collision** is two distinct inputs producing the same hash output. Because the space of possible inputs is (for any realistic hash) far larger than the fixed-size space of possible outputs, collisions are mathematically **guaranteed to exist** (pigeonhole principle) -- the practically meaningful property is not "no collisions" but **collision resistance**: how computationally hard it is to *find* one on purpose.

**Non-cryptographic hash functions.** Optimized purely for speed and good statistical distribution, with no attempt at resisting a deliberate adversary trying to engineer collisions. Examples: MurmurHash, xxHash, FNV, CityHash. Used for: in-memory hash tables/hash maps, hash-based data partitioning and sharding keys (e.g., consistent hashing), deduplication, load-balancer request routing, and probabilistic structures like Bloom filters -- anywhere you need "spread values around evenly, fast" and there's no adversary trying to break it.

**Cryptographic hash functions.** Designed to satisfy stronger, security-relevant guarantees against an adversary: (1) **pre-image resistance** -- given a hash output, it should be computationally infeasible to find *any* input that produces it; (2) **second pre-image resistance** -- given one input, infeasible to find a *different* input with the same hash; (3) **collision resistance** -- infeasible to find *any* two inputs with the same hash, even without a target. Modern examples: **SHA-256**, **SHA-3**; **MD5** and **SHA-1** are now considered cryptographically broken (practical collision attacks have been demonstrated against both) and should not be relied on for security purposes, though they may still linger in legacy non-security checksumming contexts. Used for: content-addressed storage (e.g., Git identifies every object by the SHA hash of its contents), digital signatures, and as a building block for **HMAC** (a keyed construction combining a cryptographic hash with a secret key to verify both integrity and authenticity of a message -- detailed in security, L9). Note that plain fast cryptographic hashing is the *wrong* tool for password storage -- passwords need slow, purpose-built key-derivation functions (bcrypt, scrypt, Argon2) precisely so that brute-forcing many guesses is expensive; this is covered in depth in L9.

**Checksums and CRC.** A **checksum** is a small value computed from a larger block of data specifically to detect *accidental* corruption (bit flips from a noisy network link, a flaky disk, etc.) -- it is not designed to resist a deliberate adversary and offers no cryptographic security guarantee. **CRC (Cyclic Redundancy Check)**, especially **CRC32**, is the most common example: fast to compute (frequently hardware-accelerated in modern CPUs and network interface cards), excellent at catching the kinds of random corruption that actually occur in practice, and used pervasively -- Ethernet frame trailers, ZIP file entries, gzip streams, many filesystems.

**Where each is used -- a quick decision guide:**

| Need | Use |
|---|---|
| Hash table / general-purpose in-memory lookup | Fast non-cryptographic hash |
| Sharding / partitioning key, consistent hashing | Non-cryptographic hash |
| Detect accidental data corruption (network/disk) | CRC / checksum |
| Detect deliberate tampering, verify authenticity | Cryptographic hash / HMAC |
| Content-addressed storage / dedup by content identity | Cryptographic hash |
| Password storage | Slow, salted key-derivation function (not a fast hash of any kind) |

**Where this resurfaces.** Consistent hashing for partitioning and rebalancing (L4) is built directly on non-cryptographic hash functions; password hashing and HMAC are developed in depth in L9 security; Bloom filters, HyperLogLog, and other probabilistic structures (L12) are hash functions applied at scale; hash indexes are one of the core index types alongside B-trees in L2.

---

## Clocks

**Wall clock (time-of-day clock).** Represents real-world calendar time -- e.g., seconds since the Unix epoch (1970-01-01 UTC). It is meant to answer "what time is it right now" and is used for timestamps that a human or another system needs to interpret as an actual date/time. Critically, a wall clock **can jump**, both forward and backward: it gets periodically corrected by NTP synchronization (see below), can be adjusted manually, and (rarely) must account for leap seconds. Because of this, a wall clock is **not safe** for measuring how long an operation took -- if the clock is adjusted backward mid-measurement, a naive `end_time - start_time` can even come out negative.

**Monotonic clock.** A clock guaranteed to only ever move **forward**, never adjusted by NTP corrections or manual changes, exposed by the OS specifically for measuring elapsed durations (e.g., `CLOCK_MONOTONIC` on Linux, `System.nanoTime()` in Java, the monotonic reading behind Go's `time.Now()`). The rule of thumb: use a **monotonic clock** for timeouts, latency measurement, rate limiting, and anything computing an elapsed interval; use the **wall clock** (ideally stored/transmitted as UTC in a standard format like ISO-8601 or a Unix timestamp) for recording "when did this event happen" from a human/calendar perspective.

**NTP (Network Time Protocol).** The protocol machines use to synchronize their wall clock against reference time sources, organized into a hierarchy of **strata**: stratum 0 is a reference clock itself (atomic clock, GPS receiver), stratum 1 servers are directly connected to stratum-0 sources, stratum 2 servers sync from stratum-1 servers, and so on. Over typical internet paths, NTP-synchronized clocks are commonly accurate to within roughly a few to tens of milliseconds (`~`, varies a lot with network path quality and how recently sync occurred); on a well-controlled local network it can be considerably tighter. For applications needing much tighter bounds, **PTP** (Precision Time Protocol) can achieve sub-microsecond synchronization on a LAN, at the cost of specialized hardware/network support -- used in latency-sensitive domains like finance and telecom, not general-purpose systems.

**Clock skew and drift.** **Clock skew** is the difference between two machines' clocks at a given instant. It arises because every machine's local clock is ultimately driven by an imperfect quartz crystal oscillator that **drifts** at some small rate (commonly on the order of tens of parts-per-million, `~`, which without correction can accumulate to tens of milliseconds of drift *per day*), and because NTP synchronization itself is not instantaneous or perfectly precise -- skew grows between sync intervals and shrinks (but rarely reaches exactly zero) right after a sync.

**Why this matters for distributed systems (previewed here, developed in depth later).** You cannot safely use wall-clock timestamps stamped by *different* machines to determine which of two events truly happened first: clock skew means machine B's clock might read an earlier time than machine A's even though B's event genuinely occurred after A's. Relying on physical clocks for ordering can silently produce wrong answers. This is precisely the motivation for **logical clocks** (Lamport timestamps, which capture a "happened-before" partial order using simple counters instead of physical time) and **vector clocks** (which capture causal relationships across multiple nodes), and for more advanced hybrid approaches like **Hybrid Logical Clocks (HLC)** (which combine a physical-time component with a logical counter) and Google Spanner's **TrueTime** (which exposes not a single timestamp but an explicit *uncertainty interval*, and waits out that uncertainty to guarantee global ordering) -- all covered in depth in L5 distributed systems theory, and revisited specifically as "Hybrid Logical Clocks vs TrueTime" in L4.

**Practical, single-machine relevance too.** Even without any distributed-systems angle, mixing up monotonic and wall clocks is a common real bug source: a timeout or benchmark implemented with the wall clock can misbehave the moment an NTP correction, a leap-second adjustment, or a manual clock change happens mid-measurement on the very same machine.

**Where this resurfaces.** Logical/vector clocks and causal ordering (L5); Hybrid Logical Clocks and TrueTime for globally-ordered transactions (L4, L5); timeout implementation as part of retries/circuit breakers (L7) always assumes a monotonic clock underneath.
