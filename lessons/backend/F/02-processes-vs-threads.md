# Processes vs Threads

*Two ways to run more than one thing at once -- one isolated and safe, one cheap and shared.*

`⏱️ ~6 min · 2 of 12 · Computing Fundamentals`

> [!TIP] The gist
> A **process** is a running program with its own private memory and OS resources -- isolated, so one crashing can't corrupt another. A **thread** is a unit of execution *inside* a process; threads in the same process share memory, so they talk instantly and cheaply -- but an unguarded shared write can corrupt everything. Processes trade speed for safety; threads trade safety for speed.

## Contents

- [Intuition](#intuition)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Think of an office building.

- A **process** is a company with its own locked office: its own files, its own phones, its own front door. Another company down the hall can't rummage through its cabinets.
- A **thread** is an employee inside that one company. All employees share the same office, the same filing cabinets, the same whiteboard. They coordinate for free -- but if one scribbles over a shared document while another is reading it, chaos.

## How it works

**What each one owns.** The dividing line is the *address space*.

| Shared across threads in a process | Private to each thread |
|---|---|
| Heap memory / global variables | Stack (locals, call frames) |
| Open file descriptors / sockets | CPU registers, program counter |
| Loaded code (`.text`) | Thread-local storage (opt-in) |
| Memory mappings, signal handlers | Signal mask (POSIX) |

Every thread needs its own stack, registers, and program counter because each is scheduled independently and must track "where am I in the code" and "what are my locals."

---

**Why threads are both useful and dangerous.** Threads share the heap, so any thread can read or write the same object directly -- no copying, no messaging. That is *exactly* why they're fast and *exactly* why they're risky: unsynchronized shared writes are race conditions (covered soon in [Locks and Atomicity](04-locks-and-atomicity.md)).

---

**Cost of creating each.** Rough, order-of-magnitude:

- **New process** (`fork`): the OS sets up a fresh address space and duplicates page tables. Copy-on-write avoids physically copying pages until written, but the bookkeeping still costs ~hundreds of microseconds to low milliseconds -- more if an `exec` follows to load a new program.
- **New thread**: just a new stack and register set inside the *same* address space -- ~tens of microseconds, roughly 10-100x cheaper.

Switching *between* threads of one process is also cheaper than switching between processes, because the memory mappings don't change.

---

**Isolation is a deliberate trade.** The OS enforces separate address spaces via virtual memory, so a crash or corruption in one process can't touch another. That's why browsers put each tab in its own process, and why a buggy worker can be killed and restarted without disturbing its siblings. Threads give that up: one thread corrupting shared heap state can take down the whole process.

---

**Two consequences worth knowing.**

- **IPC (inter-process communication):** since processes don't share memory, cooperating ones need explicit channels -- pipes, Unix sockets, TCP, shared-memory segments, message queues. Data usually gets copied across the boundary (and often serialized), so it's strictly more expensive than a thread calling a shared function.
- **Green / lightweight threads:** modern runtimes multiplex many logical threads onto a few OS threads -- goroutines (Go), virtual threads (Java 21+), Erlang processes. An OS thread stack often reserves ~1-8 MB; green threads can start at a few KB and grow, giving near-"spawn one per task" convenience cheaply.

## Trade-offs

**Processes**
✅ Strong isolation -- one crash can't corrupt another
✅ Natural fault containment
❌ Expensive to create; communication needs IPC and copying

**Threads**
✅ Cheap to create; instant shared-state access
✅ Low switching cost within a process
❌ No isolation -- one bad write can crash them all
❌ Shared state must be synchronized by hand

## Remember

> [!IMPORTANT] Remember
> Same program, different bargain: processes buy safety with isolation and pay in communication cost; threads buy speed with shared memory and pay in the constant risk of races.

## Check yourself

1. A web browser puts each tab in its own process rather than its own thread. What specific benefit does that buy, and what does it cost?
2. Why does creating a thread cost roughly 10-100x less than creating a process?

---

→ Next: [Concurrency vs Parallelism](03-concurrency-vs-parallelism.md)
↩ Comes back in: worker-pool sizing for requests and background jobs (L10), process-level isolation as bulkheads (L7 reliability), and the "shared nothing, communicate via messages" philosophy behind service messaging (L6).
