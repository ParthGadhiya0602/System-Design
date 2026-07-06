# OS Scheduling and Virtual Memory

*How the OS shares one CPU fairly and gives every program its own private illusion of memory.*

`⏱️ ~7 min · 6 of 12 · Computing Fundamentals`

> [!TIP] The gist
> The **scheduler** decides which thread runs next and for how long, and can *preempt* a running thread so none hogs a core. **Virtual memory** gives every process its own private address space, mapped to physical RAM in fixed **pages** via a **page table** -- delivering isolation, simplicity, and the ability to overcommit RAM by swapping to disk. The **TLB** caches translations to keep it fast; letting a working set exceed RAM causes **thrashing**.

## Contents

- [Intuition](#intuition)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

## Intuition

Two illusions the OS maintains for you:

- **Scheduling** is a teacher calling on students. Everyone gets turns; no one monopolizes; the teacher can cut someone off mid-sentence (preemption) to keep it fair.
- **Virtual memory** is a hotel where every guest is told they have room "number 1." Each guest addresses their own room 1, and the front desk secretly maps each private "room 1" to a different real room. No guest can even *name* another's room -- perfect isolation, and the hotel can pretend to have more rooms than it physically owns.

## How it works

**The scheduler.** The kernel component that picks which runnable thread gets the CPU next. Modern OSes use **preemptive multitasking**: a periodic timer interrupt lets the scheduler interrupt a running thread even if it never yields, so nothing monopolizes a core. Linux's scheduler (historically CFS, the Completely Fair Scheduler; newer kernels use EEVDF) approximates equal CPU share by tracking each thread's accumulated "virtual runtime" and always running whoever has had the least so far, weighted by priority/niceness. Time slices run a few to tens of milliseconds, adjusted by load.

---

**Virtual memory -- what and why.** Every process gets its own private, contiguous-looking address space (0 to some huge number), independent of where data physically sits in RAM. Three reasons it exists:

1. **Isolation** -- a process can't read or write another's memory; it can't even *address* it.
2. **Simplicity** -- a program is compiled as if it owns all memory from address 0, no negotiation with other programs.
3. **Overcommit and swapping** -- total virtual space can exceed physical RAM, with the OS moving cold data to disk transparently.

---

**Paging.** The address space is divided into fixed-size **pages** (commonly **4 KB**; **huge pages** of 2 MB or 1 GB reduce translation overhead for big workloads), and RAM into equal-size **frames**. A per-process **page table** maps virtual pages to physical frames. Since a full flat table would be enormous, real systems use **multi-level page tables** that only allocate entries for regions actually in use.

---

**Page faults.** A **page fault** fires when a program accesses a virtual page with no valid mapping.

- **Minor fault** -- cheap; the page is already in memory (shared, or reclaimed-but-not-reused) and the OS just fixes the mapping.
- **Major fault** -- expensive; the OS must fetch the page from disk/swap, costing disk-latency (potentially milliseconds). Excessive major faults wreck performance.

---

**TLB -- making translation fast.** Walking a multi-level page table on *every* memory access would be far too slow, so the CPU keeps a small on-chip cache of recent virtual-to-physical translations: the **Translation Lookaside Buffer**. A TLB hit resolves an address about as fast as an L1/L2 access; a **TLB miss** forces a page-table walk (extra memory accesses). Huge pages help large-memory apps because one huge-page entry covers far more address space, so fewer misses.

---

**Swapping and thrashing.** When RAM is full, the OS evicts least-recently-used pages to a disk **swap area**, paging them back on demand. Fine occasionally -- but if a process's active **working set** (the pages it genuinely needs) exceeds RAM, the system enters **thrashing**: it spends most of its time paging in and out instead of working, because whatever it pages back in evicted something else it needs. That's why Linux has an **OOM killer** -- rather than thrash forever, it sacrifices a process to restore headroom.

---

**Why this shows up in practice.** It explains the two memory numbers you see for a process: **virtual size** (address space reserved, often huge and mostly unused) vs **resident set size / RSS** (physical RAM actually backing it). It's also why databases and caches tune memory carefully to *avoid swap* -- swapping a database's working set is catastrophic for latency -- and why huge pages are a common knob for large in-memory workloads.

## Trade-offs

| Mechanism | Buys you | Costs you |
|---|---|---|
| Preemptive scheduling | Fairness; no monopolizing | Context-switch overhead |
| Virtual memory / paging | Isolation, simplicity, overcommit | Translation overhead, page-fault latency |
| TLB | Fast address translation | Misses trigger page-table walks |
| Swapping | Survive memory pressure | Thrashing if working set > RAM |

✅ Keep a database/cache working set within RAM; consider huge pages for large in-memory data
❌ Let a latency-sensitive service swap; ignore the virtual-vs-RSS distinction when sizing boxes

## Remember

> [!IMPORTANT] Remember
> Virtual memory buys isolation and the illusion of endless RAM -- but the illusion holds only while your working set fits in physical memory. Cross that line and you thrash.

## Check yourself

1. A process reports 20 GB virtual size but only 2 GB RSS on a 4 GB machine, and runs fine. How is that possible?
2. A database's latency collapses under load and the box is swapping heavily. In virtual-memory terms, what has happened, and why is swapping so damaging here specifically?

---

→ Next: [Disks and Filesystems](07-disks-and-filesystems.md)
↩ Comes back in: a database's buffer pool managing hot vs cold pages with the same eviction logic (L2 storage engines), and "don't let the box swap" capacity planning (L7 reliability).
