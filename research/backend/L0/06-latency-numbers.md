# Latency Numbers Every Engineer Should Know

A short table of order-of-magnitude timings that lets you turn "N disk reads" or "M network round trips" into a latency budget you can sanity-check before you write a line of code.

## Contents

- [Why These Numbers Matter](#why-these-numbers-matter)
- [Units and Terms First](#units-and-terms-first)
- [The Canonical Latency Table](#the-canonical-latency-table)
- [Derived Intuitions to Keep](#derived-intuitions-to-keep)
- [How to Use It in Estimation](#how-to-use-it-in-estimation)
- [How This Connects Onward](#how-this-connects-onward)
- [Check Yourself](#check-yourself)

## Why These Numbers Matter

When you design a system you make a claim like "this endpoint returns in under
200 ms at p99." (**p99** = the 99th-percentile latency: 99 out of 100 requests
are at least this fast; it is the number users actually feel on a bad day, not
the average.) That claim is only believable if you can add up where the time
goes.

Every operation your code triggers has a rough, physically-grounded cost:
reading from memory, reading from a disk, or sending a message to another
machine. If you know those costs to within an order of magnitude, you can take a
proposed design -- say "we do 20 random disk reads and 3 network round trips per
request" -- and turn it into a **latency budget**: a running sum of expected
time you compare against your target. If the sum blows the target, the design is
wrong on paper and you fix it before building it.

The single lesson underneath the whole table is this ordering:

```
memory  <<  SSD  <<  network (same building)  <<  disk seek  <<  cross-continent
```

Each `<<` means "much slower than" -- and end to end this ladder spans roughly
**8 orders of magnitude** (a factor of ~100,000,000). A main-memory read is
~100 nanoseconds; a round trip across the planet is ~100-300 milliseconds. That
is the difference between one second and about 3 years scaled up. Almost every
system-design decision -- caching, indexing, replication, putting servers near
users -- is an attempt to move work *down* this ladder toward the fast end.

You are not meant to memorize the digits. **Memorize the ratios.** The exact
nanosecond counts drift every year with hardware; the *relationships between
tiers* (memory vs. SSD vs. disk vs. network) have been stable for a long time
and are what you reason with.

## Units and Terms First

Time units, each 1000x the last:

- **ns** = nanosecond = one-billionth of a second (10^-9 s).
- **us** = microsecond = one-millionth of a second (10^-6 s) = 1000 ns.
- **ms** = millisecond = one-thousandth of a second (10^-3 s) = 1000 us.

Other terms used below:

- **Cache (CPU)** = tiny, very fast memory on the processor itself. **L1** is
  smallest and fastest, **L2** larger and slightly slower. These hold data the
  CPU is actively using.
- **Main memory / RAM** = the gigabytes of working memory in the machine;
  fast, but off-chip, so much slower than cache.
- **SSD (solid-state drive)** = flash-based persistent storage; no moving parts.
- **HDD (hard disk drive)** = spinning-platter persistent storage; a physical
  head must move to the right track before reading -- that move is the **disk
  seek**.
- **Round trip (RTT, round-trip time)** = time for a message to travel to
  another machine and for the reply to come back.
- **Sequential vs. random access** = reading bytes that sit next to each other
  (sequential) versus jumping to scattered locations (random). Sequential is
  dramatically faster on every storage tier.
- **Branch mispredict** = the CPU guesses which way an `if` will go to keep
  busy; when it guesses wrong it must throw away work and restart -- a small but
  measurable penalty.
- **Mutex lock/unlock** = the cost of acquiring and releasing a lock that
  protects shared data between threads (uncontended case).

## The Canonical Latency Table

Values in the spirit of the widely-circulated Jeff Dean / Peter Norvig list.
Every figure is **order-of-magnitude and hardware/year-dependent** -- treat the
`~` as "roughly, on today's commodity hardware." Ranges reflect real variation
across devices and workloads.

| Operation                                  | Approx. time  | In nanoseconds (~) |
| ------------------------------------------ | ------------- | ------------------ |
| L1 cache reference                         | ~0.5 ns       | ~0.5               |
| Branch mispredict                          | ~5 ns         | ~5                 |
| L2 cache reference                         | ~7 ns         | ~7                 |
| Mutex lock/unlock (uncontended)            | ~25 ns        | ~25                |
| Main memory (RAM) reference                | ~100 ns       | ~100               |
| Compress 1 KB                              | ~1-3 us       | ~1,000-3,000       |
| Read 1 MB sequentially from memory         | ~3-10 us      | ~3,000-10,000      |
| Send 1 KB over a 1 Gbps network            | ~10 us        | ~10,000            |
| SSD random read                            | ~16-150 us    | ~16,000-150,000    |
| Read 1 MB sequentially from SSD            | ~0.05-1 ms    | ~50,000-1,000,000  |
| Round trip within the same datacenter      | ~0.5 ms       | ~500,000           |
| Disk seek (HDD, move the head)             | ~5-10 ms      | ~5,000,000-10,000,000   |
| Read 1 MB sequentially from HDD            | ~5-20 ms      | ~5,000,000-20,000,000   |
| Round trip cross-continent                 | ~100-150 ms   | ~100,000,000-150,000,000 |
| Round trip around the world                | ~200-300 ms   | ~200,000,000-300,000,000 |

Read the far-right column top to bottom and watch the zeros pile up: that visual
growth is the "8 orders of magnitude" made concrete.

## Derived Intuitions to Keep

These are the takeaways worth burning into memory. Round the table to three
anchor points and everything else hangs off them:

- **RAM ~100 ns. SSD random read ~100 us. HDD seek ~10 ms.**
  Each step is roughly **~1000x** slower than the last.
  - SSD random read is **~1000x slower than RAM** (100 us vs. 100 ns).
  - HDD seek is **~100x slower than an SSD random read** (10 ms vs. 100 us),
    and **~100,000x slower than RAM**.
  - This gap is *the reason caches exist*. Keeping hot data in RAM instead of
    reaching for disk is not a small optimization -- it is a 1000x-to-100,000x
    difference. It is also why the industry moved from HDD to SSD for
    latency-sensitive data.

- **A same-datacenter round trip (~0.5 ms) is ~5000x a RAM read (~100 ns).**
  Talking to another machine in the same building is cheap by human standards
  but enormous by CPU standards. This is why chatty designs that make many
  small network calls per request ("N+1" calls, per-item lookups) fall over:
  each hop costs thousands of RAM reads. Batch requests; fetch many things in
  one round trip.

- **Cross-continent (~100-150 ms) is a hard physics floor.**
  Light in fiber travels at roughly two-thirds of the speed of light in a
  vacuum. The distance across a continent, there and back, simply takes on the
  order of ~100 ms no matter how much money you spend. You cannot optimize this
  away in code. The only fix is to **not send the request that far**: move the
  data or the computation closer to the user (edge caches / CDN, multi-region
  deployments, read replicas near users). Around-the-world round trips
  (~200-300 ms) make this even starker.

- **Sequential beats random on every tier.**
  Reading 1 MB in one contiguous sweep is far cheaper than 1 MB scattered as
  many random reads, because you pay the "find it" cost (seek on HDD, lookup
  overhead on SSD) only once. Data layouts that keep related data adjacent
  (append-only logs, sorted files, column stores) exploit this directly.

- **Compression can be a net win when it saves a slower hop.**
  Compressing 1 KB costs ~1-3 us of CPU. If that shrinks the bytes you must
  push over a network round trip (~0.5 ms same-DC, ~100 ms cross-continent) or
  read off disk, the CPU cost is trivially repaid. Rule of thumb: **spend the
  fast resource (CPU/RAM) to avoid the slow one (network/disk)** -- as long as
  the compression actually reduces the amount of the slow work.

## How to Use It in Estimation

The method has three steps:

1. **Count the operations per request** -- how many RAM reads, SSD/disk reads,
   and network round trips does one request cause? Focus on the *slow* ones
   (disk and network); RAM/cache time usually rounds to zero next to them.
2. **Multiply each count by its cost** from the table.
3. **Sum, then compare to the target** (e.g., p99 under 200 ms). If you are over
   budget, the design must change: cache to remove disk hits, batch to remove
   round trips, denormalize to remove lookups, or move closer to remove
   distance.

Worked mini-example. A profile page needs 1 user record plus 20 of the user's
posts, and the target is **p99 under 100 ms**.

- Naive design: 1 lookup for the user + 1 lookup per post = **21 random reads**,
  each a same-datacenter round trip to the database at ~0.5 ms.
  - `21 x ~0.5 ms = ~10.5 ms` of round-trip time.
  - Comfortably under 100 ms -- *if* those reads hit RAM/SSD-backed storage.

- Now suppose those 21 reads instead each cause an **HDD seek** (~10 ms),
  because the data is on spinning disk and not cached:
  - `21 x ~10 ms = ~210 ms`. **Over budget** -- the design fails on paper.
  - Fixes ranked by impact: put the hot data in RAM/SSD (removes the ~10 ms
    seek, back to the ~10 ms total above); **batch** the 20 post reads into a
    single query (21 round trips -> 2), cutting round-trip count by ~10x.

- And if the database sits on another continent, add ~100-150 ms *per round
  trip* on top of everything. Twenty-one cross-continent trips would be seconds
  -- which is why you keep the datastore in the same region as the service and
  push a **read replica** or **cache** near the user.

Notice you reached a correct verdict -- and identified the right fix -- with
arithmetic only, before building anything. That is the entire point of the
table.

## How This Connects Onward

These numbers are the quantitative backbone for most later topics:

- **Caching** exists because RAM is ~1000x faster than SSD and ~100,000x faster
  than an HDD seek; the whole discipline is deciding what to keep in the fast
  tier.
- **Storage engines and indexing** are built to turn expensive random reads and
  seeks into cheap sequential ones, and to minimize disk touches per query.
- **Content delivery networks (CDNs) and edge caching** attack the
  cross-continent physics floor by serving data from a location near the user.
- **Multi-region architecture and replication** likewise fight distance and
  place data close to where it is read.
- **Back-of-the-envelope estimation** in later design practice reuses exactly
  the count-multiply-sum method shown here.

You will meet each of these in later levels; when you do, come back to this
ladder -- almost every technique is a move toward its fast end.

## Check Yourself

1. Put these in order fastest to slowest and give the rough ratio between
   neighbors: RAM read, HDD seek, SSD random read, same-DC round trip.
2. Why can no amount of code optimization get a single cross-continent round
   trip below ~100 ms? What is the only real fix?
3. A request makes 50 random reads. Estimate total time if each hits RAM; if
   each hits an SSD; if each causes an HDD seek. Which of these fits a 100 ms
   p99 budget?
4. When is compressing data before sending it a net win, and when is it wasted
   work?
5. You are told to memorize this table for an exam. What should you actually
   commit to memory, and why not the exact digits?
