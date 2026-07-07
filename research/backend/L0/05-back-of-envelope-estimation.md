# Back-of-Envelope Estimation

Quick order-of-magnitude math that turns vague requirements ("build a Twitter") into concrete numbers (QPS, GB, Gbps) so your architecture choices are justified rather than guessed.

## Contents
- [1. What it is and why it matters](#1-what-it-is-and-why-it-matters)
- [2. The method: users to QPS to storage to bandwidth](#2-the-method-users-to-qps-to-storage-to-bandwidth)
- [3. Cheatsheets: numbers to memorize](#3-cheatsheets-numbers-to-memorize)
- [4. Fully worked example: a Twitter-like service at 100M DAU](#4-fully-worked-example-a-twitter-like-service-at-100m-dau)
- [5. How this connects onward](#5-how-this-connects-onward)

## 1. What it is and why it matters

**Back-of-envelope estimation** is doing rough arithmetic - the kind that fits on the back of an envelope - to size a system before you design it. You take a few known or assumed inputs (how many users, how often they act, how big each item is) and multiply your way to the quantities that actually drive architecture: **QPS** (queries per second - how many requests hit the system each second), storage in bytes, and bandwidth in bits or bytes per second.

The point is to answer questions like: Do I need **one server or a thousand**? Will my data fit on a single disk, or is it **gigabytes, terabytes, or petabytes**? Can one database absorb the write load, or must I split it across machines?

**Precision is explicitly NOT the goal.** Whether the real answer is 8,000 QPS or 11,000 QPS does not change your design - both say "one beefy server or a small cluster." But the difference between 10,000 QPS and 10,000,000 QPS is the difference between a single node and a globally sharded fleet. Estimation exists to land you in the right **order of magnitude** (the nearest power of ten), because that is the granularity at which architecture decisions flip.

Two habits make estimation trustworthy:

- **Round aggressively.** Use 100,000 seconds per day instead of 86,400. Use 1,000 instead of 1,024 for a kilobyte. Round user counts and payload sizes to one significant figure. Errors from rounding are tiny compared to the uncertainty in your assumptions, and clean numbers keep the mental math fast and error-free.
- **State every assumption out loud.** "Assume 100M daily users, each posts twice a day, each post is ~1 KB." If an assumption is wrong, a reviewer can correct that one number and rerun the multiplication. Hidden assumptions are the only way estimation actually misleads you.

A good estimate is a *chain of clearly labeled assumptions and multiplications ending in a number you can defend*. The number itself matters less than the fact that your architecture is now grounded in one.

## 2. The method: users to QPS to storage to bandwidth

Almost every estimate flows through the same four stages. Each stage feeds the next.

### (a) Users to actions

Start with the population and how busy it is.

- **DAU** (daily active users) - the count of distinct users who use the product on a given day. Often given, or derived from total registered users (a common rule of thumb: DAU is some fraction of monthly active users; assume and state it).
- **Actions per user per day** - how many times an average user performs the operation you care about (posts, reads a feed, uploads a photo). Estimate separately per action type, because reads and writes behave very differently.

```
daily_actions = DAU x actions_per_user_per_day
```

### (b) Actions to QPS

Convert a per-day count into a per-second rate.

```
average_QPS = daily_actions / 86,400   (~ daily_actions / 100,000)
```

Real traffic is not flat across the day - it clusters around peak hours (evenings, lunchtime, time zones overlapping). So you size for the peak, not the average:

```
peak_QPS ~= 2 to 3 x average_QPS
```

(Use the higher multiplier for spiky consumer apps, lower for steady background workloads. State which you chose.)

Then **split reads from writes**, because they hit different parts of the system (writes go to primary databases; reads can be served from caches and replicas). Most consumer systems are read-heavy - a common assumption is something like 100:1 or 1000:1 reads to writes; pick a ratio, state it, and compute read QPS and write QPS separately.

### (c) QPS to storage

Storage is about *what accumulates over time*, so it depends on retention, not on the per-second rate.

```
raw_daily_bytes = bytes_per_item x items_created_per_day
storage = raw_daily_bytes x retention_days
```

Then apply the multipliers that real systems impose:

- **Replication** - data is copied to N machines for durability and availability (typically x3). Multiply.
- **Indexes and overhead** - secondary indexes, metadata, and internal structures add roughly tens of percent to well over 100% on top of raw data. Add a fudge factor and say so.
- **Growth** - project forward. If the item rate grows, storage over `Y` years is the sum of each year's accumulation, not just year-one times Y.

### (d) QPS to bandwidth

Bandwidth is a *rate* - it comes from QPS multiplied by payload size, not from stored volume.

```
bandwidth = QPS x bytes_per_payload
```

Track two directions separately; they are usually very asymmetric:

- **Ingress** - data flowing *into* the system (uploads, writes). Write QPS x request payload.
- **Egress** - data flowing *out* to users (feed reads, downloads, media). Read QPS x response payload. For media-heavy systems egress dwarfs ingress by orders of magnitude, which is exactly what forces a CDN.

Convert to bits when comparing against network links (they are rated in bits per second): multiply bytes by 8.

## 3. Cheatsheets: numbers to memorize

All figures below are approximate (`~`) and meant for mental math, not benchmarks.

### Time

- 1 day ~= 86,400 s ~= 10^5 s (round up; makes division trivial)
- 1 hour = 3,600 s
- **1 million actions/day ~= 12 QPS** (since 10^6 / 10^5 = 10; use ~12 to be a little safe). Handy inverse: **1 QPS ~= 100,000 actions/day**.
- 1 billion actions/day ~= 12,000 QPS

### Powers of two and data sizes

Data sizes step by 2^10 ~= 1,000 (a "thousand-ish"):

- 1 KB (kilobyte) ~= 10^3 bytes
- 1 MB (megabyte) ~= 10^6 bytes
- 1 GB (gigabyte) ~= 10^9 bytes
- 1 TB (terabyte) ~= 10^12 bytes
- 1 PB (petabyte) ~= 10^15 bytes

Rule of thumb: every 1,000x in bytes moves you one row down (KB to MB to GB to TB to PB).

### Typical item sizes (rough)

- Short text post / tweet ~ 1 KB (text plus metadata)
- One database row (few columns) ~ 100 bytes to 1 KB
- Compressed thumbnail ~ 10 KB
- Web-sized photo ~ 100 KB to 1 MB
- 1 minute of streaming video (SD/HD) ~ 5 to 50 MB
- JSON API response (typical) ~ 1 to 10 KB

### Machine capacity anchors (order of magnitude)

- Application server (stateless, JSON): ~ 1,000 to 10,000 QPS per node
- SQL database node writes: ~ hundreds to a few thousand writes/s before you must scale out (reads scale further via replicas and cache)
- In-memory cache (Redis-like), single node: ~ 100,000+ simple ops/s
- Message queue broker: ~ tens of thousands to 100,000s of messages/s per node
- Disk (SSD) sequential throughput: ~ hundreds of MB/s; single spinning disk ~ 100 MB/s
- **Network: 1 Gbps ~= 125 MB/s** (divide bits by 8); 10 Gbps ~= 1.25 GB/s

Use these as *ceilings*: divide your required QPS or bandwidth by the per-node anchor to get a rough machine count. If the count is ~1, you are single-node; if it is thousands, you are into serious distributed territory.

## 4. Fully worked example: a Twitter-like service at 100M DAU

Design a microblogging service: users post short messages and read a home feed. Let us size it end to end.

### Assumptions (stated up front)

- DAU ~= 100M (10^8)
- Each user **posts** ~ 2 times/day (write action)
- Each user **reads** ~ 100 feed views/day (read action)
- A post is ~ 1 KB (text plus metadata)
- A feed view returns ~ 20 posts ~= 20 KB response
- Peak = 3x average
- Retention: store all posts, projected over 5 years
- Replication factor: 3; add ~100% overhead for indexes and metadata

### Write QPS

```
daily_writes = 100M users x 2 posts = 200M writes/day
average_write_QPS = 200M / 10^5 s ~= 2,000 QPS
peak_write_QPS ~= 3 x 2,000 = ~6,000 QPS
```

Interpretation: ~6,000 writes/s at peak. That is above a single SQL node's comfortable write ceiling - it points toward sharded or write-optimized storage, but it is not extreme.

### Read QPS

```
daily_reads = 100M users x 100 feed views = 10,000M = 10^10 reads/day
average_read_QPS = 10^10 / 10^5 s ~= 100,000 QPS
peak_read_QPS ~= 3 x 100,000 = ~300,000 QPS
```

**Read:write ratio ~= 100,000 : 2,000 = ~50:1** - strongly read-heavy. This is the single most important number in the whole estimate.

### Storage over time

```
raw_bytes_per_day = 200M posts x 1 KB = 200M KB = ~200 GB/day
raw_per_year = 200 GB x 365 ~= ~73 TB/year   (~70 TB rounded)
raw_over_5_years ~= 5 x 70 TB = ~350 TB
```

Apply the multipliers:

```
x3 replication:            350 TB x 3 = ~1,050 TB ~= ~1 PB
+100% index/metadata:      ~1 PB x 2  = ~2 PB total
```

Interpretation: **~2 PB of managed storage** over 5 years. Far beyond one machine's disk - this is inherently a sharded, multi-node storage problem. (Media like images/video would add orders of magnitude more; here we sized text only.)

### Egress bandwidth

Reads dominate outbound traffic:

```
egress = peak_read_QPS x feed_response_size
       = 300,000 QPS x 20 KB
       = 6,000,000 KB/s = ~6 GB/s
convert to bits: ~6 GB/s x 8 = ~48 Gbps
```

Interpretation: **~48 Gbps of peak egress**. A single 10 Gbps link cannot carry it; you need many machines behind a load balancer and, since much of the payload is repeated popular content, a CDN and caching layer to keep it off the origin.

Ingress by contrast:

```
ingress = peak_write_QPS x post_size = 6,000 x 1 KB = ~6 MB/s (~48 Mbps)
```

Ingress is ~1,000x smaller than egress - textbook asymmetry.

### Architectural conclusions the numbers force

- **~50:1 read-heavy** -> put a **cache** in front of reads and serve from **read replicas**; the primary should mostly handle the ~6,000 write QPS.
- **~2 PB and growing** -> **shard** the data store across many nodes; no single database instance is viable. Growth projection means capacity planning must be continuous, not one-time.
- **~48 Gbps egress, much of it repeated content** -> front the system with a **CDN** and edge caches so origin servers and databases are shielded from read fan-out.
- **~6,000 peak write QPS** -> above a lone SQL node's comfort zone; use partitioned writes and consider fan-out-on-write vs fan-out-on-read for feed delivery (a downstream design question the numbers now frame).

Notice every conclusion traces to a specific number. That is the entire value of the exercise.

## 5. How this connects onward

Back-of-envelope estimation is the bridge between requirements and architecture. It turns the **non-functional requirements** (NFRs) - the "how much / how fast / how big" constraints - into figures you carry through the rest of a design:

- The read:write ratio and QPS decide **caching and replication** strategy (later caching and data-scaling levels).
- Storage volume and growth decide **partitioning and sharding** (data storage and scalability levels).
- Bandwidth and egress asymmetry decide **load balancing and content delivery / CDN** (networking and delivery levels).
- Peak QPS versus single-node ceilings decides how many machines and thus the **distributed-systems** concerns (consistency, coordination, failure) you must confront (distributed theory and reliability levels).
- Every **capacity plan, cost estimate, and scaling milestone** downstream starts from these numbers and refines them.

In the end-to-end design framework, estimation is the step that comes right after clarifying requirements and right before you sketch the high-level design - it is what makes the rest of the design defensible with math instead of intuition.
