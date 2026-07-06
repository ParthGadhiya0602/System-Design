# Concurrency vs Parallelism and Context Switching

*Juggling versus having many hands -- and the hidden tax the OS pays to fake the first.*

`⏱️ ~6 min · 3 of 12 · Computing Fundamentals`

> [!TIP] The gist
> **Concurrency** is *structure*: many tasks in progress over overlapping time, interleaved -- possible on a single core. **Parallelism** is *execution*: many tasks running at the literal same instant -- needs multiple cores. The OS creates the illusion of concurrency by rapidly switching between tasks (**context switching**), and each switch costs more than it looks -- mostly from cold caches, not the register save itself.

## Contents

- [Intuition](#intuition)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

A single chef cooking five dishes -- stirring one, then chopping for another while the first simmers -- is **concurrent**: five dishes are "in progress," but only one pair of hands is ever moving. Hire five chefs and all five dishes get worked on at the same instant -- that's **parallelism**.

Concurrency is about *managing* overlapping work. Parallelism is about *speeding it up* with more hands.

## How it works

**The distinction changes the problem you're solving.**

- Concurrency asks: how do I *correctly* manage interleaved tasks -- ordering, safe shared state, no one starving? This matters even on one core.
- Parallelism asks: how do I *speed up* work by splitting it into independent chunks? Adding cores only helps if the work actually splits -- the non-parallelizable fraction caps your speedup no matter how many cores you add.

A single-core event loop juggling 10,000 sockets is *highly concurrent, zero parallelism*. One CPU-bound loop spread across 8 cores is *parallel*. A busy web server is both.

---

**What a context switch is.** The OS suspends the running thread and resumes another, so many logical flows appear to progress "at once." It happens two ways:

- **Preemptive** -- a periodic timer interrupt yanks the CPU away, so no thread can hog a core forever.
- **Voluntary** -- a thread blocks on I/O or a lock, or yields.

Either way, the scheduler picks the next runnable thread and switches.

---

**What a switch actually costs.** Two parts:

1. **Direct cost** -- save the outgoing thread's registers and program counter, load the incoming one's. Genuinely cheap: ~1-2 microseconds.
2. **Indirect cost (the big one)** -- the resumed thread starts with **cold L1/L2 caches and a cold TLB**. It re-warms them by taking a burst of cache misses (~100 ns each) before it runs at full speed. Switching between threads of *different* processes is worse still, because the address space changes and address-translation caches may be flushed.

So the *effective* cost under real workloads is commonly several to tens of microseconds -- far more than the raw register save alone.

---

**Why this bites in practice.** Spawn far more runnable threads than you have cores -- say, one OS thread per request under heavy load -- and the scheduler spends a growing slice of wall-clock time *switching* instead of working. Throughput can actually **drop** as concurrency climbs past a point. That's the core justification for bounded worker pools and for event-driven designs that handle many tasks on few threads. (It's also a root cause of the C10K problem in [IO Models](05-io-models.md).)

## Trade-offs

| | Concurrency | Parallelism |
|---|---|---|
| Nature | Structural (tasks overlap in time) | Execution (tasks run at same instant) |
| Needs multiple cores? | No | Yes |
| Goal | Manage interleaving correctly | Speed up divisible work |
| Limit | Coordination complexity | Non-parallel fraction caps speedup |

More threads is not always more speed:
✅ Enough concurrency to keep cores busy while others wait on I/O
❌ Far more runnable threads than cores -- context-switch overhead eats throughput

## Remember

> [!IMPORTANT] Remember
> Concurrency is a way to *structure* a program; parallelism is a way to *run* it faster. And every context switch quietly pays a cache-and-TLB re-warming tax that dwarfs the visible register-save cost.

## Check yourself

1. A task runs on a single-core laptop. Can it be concurrent? Can it be parallel? Explain each.
2. You raise a server's thread pool from 50 to 5,000 threads on an 8-core box and throughput gets *worse*. What's the mechanism?

---

→ Next: [Locks and Atomicity](04-locks-and-atomicity.md)
↩ Comes back in: sizing thread pools and connection pools (L10, L3), and reasoning about parallelism across partitions in stream processing (L6).
