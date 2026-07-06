# F. Computing Fundamentals -- Case Studies and Sources

Web-verified, real-world examples showing how the fundamentals in [f-computing-fundamentals.md](f-computing-fundamentals.md) actually shape production systems. Each subsection maps to the numbered section (SS) in that file. Sources were fetched and verified on 2026-07-06.

## Contents

1. [SS1 - CPU and Memory Hierarchy](#ss1-cpu-and-memory-hierarchy)
2. [SS5 - IO Models and Event Loops](#ss5-io-models-and-event-loops)
3. [SS7 - Disks and Filesystems](#ss7-disks-and-filesystems)
4. [SS9 - Serialization](#ss9-serialization)
5. [SS10 - Compression](#ss10-compression)
6. [SS11 - Hashing](#ss11-hashing)
7. [SS12 - Clocks](#ss12-clocks)
8. [All sources](#all-sources)

---

## SS1 - CPU and Memory Hierarchy

**Case study: Discord's Read States service (Go to Rust).** Discord's Read States service (tracks which channels/messages each user has read) ran on Go and hit a memory-management wall rather than a raw-throughput one: Go's garbage collector runs at least every two minutes, and each run had to scan Discord's entire in-memory LRU cache (tens of millions of entries) to determine what was still reachable, causing a recurring latency spike every ~2 minutes regardless of how little garbage was actually being generated. Discord rewrote the service in Rust, whose ownership model frees a cache entry's memory the instant it's evicted rather than deferring reclamation to a stop-the-world-style scan; the result was no periodic GC-driven spikes at all, and after further tuning the Rust version beat Go on latency, CPU, and memory simultaneously (p99 dropped from double-digit milliseconds to a small number of milliseconds). This is a direct, production illustration of why memory-layout and allocation strategy -- not just algorithmic complexity -- decide tail latency at scale.
- Discord Engineering, "Why Discord is switching from Go to Rust" -- https://discord.com/blog/why-discord-is-switching-from-go-to-rust (accessed 2026-07-06)

---

## SS5 - IO Models and Event Loops

**Case study: nginx's event-loop architecture solving C10K, and Cloudflare hitting its limits at internet scale.** nginx was built specifically to solve the C10K problem (Dan Kegel, 1999): instead of one OS thread per connection (which exhausts memory and context-switch budget in the thousands-of-connections range), nginx runs a small, fixed number of single-threaded worker processes, each driving a non-blocking event loop on top of `epoll` (Linux) that lets one worker efficiently watch and react to many thousands of live sockets. Cloudflare -- running nginx across its edge to serve over 10 million requests/second across 150+ data centers -- later found that a single blocking operation (synchronous disk I/O) inside that same event loop could stall an entire worker and dominate tail latency, with blocked event loops accounting for more than half of p99 time-to-first-byte in their measurements; they fixed it by moving blocking file operations off the event loop onto a thread pool, a direct, contemporary re-application of "don't block the event loop" at global scale.
- Cloudflare Blog, "How we scaled nginx and saved the world 54 years every day" -- https://blog.cloudflare.com/how-we-scaled-nginx-and-saved-the-world-54-years-every-day/ (accessed 2026-07-06)

---

## SS7 - Disks and Filesystems

**Case study: RocksDB (Meta/Facebook) forking LevelDB to fix write-amplification and SSD underutilization.** Facebook forked Google's LevelDB into RocksDB in 2012 because LevelDB's single-threaded compaction process caused severe write stalls and could not drive the I/O parallelism modern flash SSDs offered (contemporary flash could sustain over a million IOPS, but LevelDB's design left most of that capacity unused). RocksDB re-architected the LSM-tree engine with multi-threaded, tunable compaction so the storage engine could explicitly trade off read amplification, write amplification, and space amplification for a given workload -- the exact trade-off described conceptually in the fundamentals doc. RocksDB now underpins storage layers at Facebook/Meta (MyRocks, serving parts of the social graph), and is used in production at other web-scale companies including LinkedIn.
- Meta/Facebook RocksDB Engineering, "The History of RocksDB" -- http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html (accessed 2026-07-06)
- Facebook RocksDB Wiki, "RocksDB Overview" -- https://github.com/facebook/rocksdb/wiki/RocksDB-Overview (accessed 2026-07-06)

---

## SS9 - Serialization

**Case study 1: Amazon Dynamo -- consistent hashing as the partitioning scheme (foundational, e-commerce domain).** Amazon's Dynamo paper (the design underpinning DynamoDB) partitions data across storage nodes by hashing each key onto a fixed circular hash-value space ("the ring") and walking clockwise to the first node past that point; a node's arrival or departure then only reshuffles data for its immediate neighbors instead of the whole cluster. Naive consistent hashing produced uneven load because node positions were random and ignored hardware heterogeneity, so Dynamo assigns each physical node many "virtual nodes" (multiple ring positions) sized to that node's actual capacity, which is now the standard technique used by essentially every modern hash-partitioned data store. (This is technically the hashing section's territory, but Dynamo's design directly depends on serializing the key as an opaque byte array before hashing it -- the data-representation/serialization boundary the fundamentals doc calls out.)
- Amazon (DeCandia et al.), "Dynamo: Amazon's Highly Available Key-value Store" -- https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html (accessed 2026-07-06)

**Case study 2: Confluent/Kafka -- Avro plus Schema Registry to decouple producers and consumers.** In an event-streaming pipeline, producers and consumers are decoupled in time and space but still tightly coupled through the shape of the data they exchange -- a schema change on the producer side can silently break every consumer. Confluent's Schema Registry pairs with Avro specifically because Avro has well-defined, machine-checkable schema-evolution rules; the registry enforces a configurable compatibility mode (backward, forward, or full) before a new schema version is allowed, so producers and consumer teams can evolve their code independently instead of coordinating deploys. This is the production-grade version of the "schema evolution" rules described generically in the fundamentals doc, applied at company scale in fintech- and e-commerce-style event pipelines built on Kafka.
- Confluent, "Decoupling Systems with Apache Kafka, Schema Registry and Avro" -- https://www.confluent.io/blog/decoupling-systems-with-apache-kafka-schema-registry-and-avro/ (accessed 2026-07-06)
- Confluent Documentation, "Schema Evolution and Compatibility" -- https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html (accessed 2026-07-06)

---

## SS10 - Compression

**Case study: Meta/Facebook building Zstandard because "pick a point on the speed-vs-ratio line" wasn't good enough.** Facebook engineer Yann Collet created Zstandard (Zstd) because the incumbent standard, DEFLATE/gzip, hadn't meaningfully improved in roughly two decades, and every alternative (LZ4 for speed, xz for ratio) forced a hard choice between compression speed and compression ratio. Zstd's own published benchmarks (at the time of release) claimed roughly 3-5x faster compression than gzip at the same ratio, 10-15% smaller output than gzip at the same speed, and about 2x faster decompression regardless of ratio, plus 22 selectable compression levels (versus zlib's 9) so one algorithm can be dialed toward either extreme per use case. This is exactly the "tunable speed vs ratio" behavior the fundamentals doc describes, and it's why Zstd has since become a default codec inside Kafka, RocksDB, and the Linux kernel -- displacing gzip in many of those hot paths.
- Meta/Facebook Engineering, "Smaller and faster data compression with Zstandard" -- https://engineering.fb.com/2016/08/31/core-infra/smaller-and-faster-data-compression-with-zstandard/ (accessed 2026-07-06)

---

## SS11 - Hashing

**Case study 1: Amazon Dynamo -- consistent hashing with virtual nodes (see also SS9 above).** Dynamo hashes each key (an opaque byte array, via MD5 in the original design) onto a fixed circular hash space and locates the first node clockwise of that point; because a plain random placement of nodes on the ring produces uneven load, each physical node is given many "virtual node" positions sized to its actual capacity. This is the canonical, most-cited real-world production use of non-cryptographic hashing for sharding/partitioning described in the fundamentals doc's hashing decision table.
- Amazon (DeCandia et al.), "Dynamo: Amazon's Highly Available Key-value Store" -- https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html (accessed 2026-07-06)

**Case study 2: Apache Kafka -- CRC checksums on every record batch to catch accidental corruption.** Every Kafka record batch on the wire and on disk carries a CRC (CRC-32C/Castagnoli in the current message format, plain CRC32 in the pre-0.11 legacy format) covering the batch's bytes, computed by the producer and checked by brokers/consumers to detect corruption or truncation from a flaky disk or network link -- precisely the "detect accidental corruption, not deliberate tampering" use case the fundamentals doc assigns to CRC/checksums rather than cryptographic hashes.
- Apache Kafka Documentation, "Message Format" -- https://kafka.apache.org/34/implementation/message-format/ (accessed 2026-07-06)

---

## SS12 - Clocks

**Case study: Google Spanner's TrueTime -- turning clock uncertainty into an explicit interval instead of hiding it.** Rather than returning a single (unreliable) timestamp, Spanner's TrueTime API returns an explicit uncertainty interval `[earliest, latest]` guaranteed to bracket the true current time, backed by GPS and atomic clocks in each datacenter; Google reports this keeps 99th-percentile clock uncertainty under 1 millisecond. Spanner uses this directly for correctness: when committing a transaction it sets the commit timestamp to the interval's upper bound and then performs "commit wait" -- holding the commit until TrueTime's `earliest` has passed that timestamp -- which guarantees external consistency (global ordering of causally related transactions) purely from bounded clock uncertainty rather than from additional coordination rounds. This is the production-grade evolution of the "wall clocks can't safely order distributed events" problem the fundamentals doc previews, and it's the direct ancestor of Hybrid Logical Clocks used elsewhere in the industry.
- Google Cloud, "Strict Serializability and External Consistency in Spanner" -- https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner (accessed 2026-07-06)

---

## All sources

1. Discord Engineering, "Why Discord is switching from Go to Rust" -- https://discord.com/blog/why-discord-is-switching-from-go-to-rust
2. Cloudflare Blog, "How we scaled nginx and saved the world 54 years every day" -- https://blog.cloudflare.com/how-we-scaled-nginx-and-saved-the-world-54-years-every-day/
3. Meta/Facebook RocksDB Engineering, "The History of RocksDB" -- http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html
4. Facebook RocksDB Wiki, "RocksDB Overview" -- https://github.com/facebook/rocksdb/wiki/RocksDB-Overview
5. Amazon (DeCandia et al.), "Dynamo: Amazon's Highly Available Key-value Store" -- https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html
6. Confluent, "Decoupling Systems with Apache Kafka, Schema Registry and Avro" -- https://www.confluent.io/blog/decoupling-systems-with-apache-kafka-schema-registry-and-avro/
7. Confluent Documentation, "Schema Evolution and Compatibility" -- https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html
8. Meta/Facebook Engineering, "Smaller and faster data compression with Zstandard" -- https://engineering.fb.com/2016/08/31/core-infra/smaller-and-faster-data-compression-with-zstandard/
9. Apache Kafka Documentation, "Message Format" -- https://kafka.apache.org/34/implementation/message-format/
10. Google Cloud, "Strict Serializability and External Consistency in Spanner" -- https://cloud.google.com/blog/products/databases/strict-serializability-and-external-consistency-in-spanner

**Topics omitted (no distinct additional verified source beyond the above, or well-covered by adjacent sections):** SS2 Processes vs Threads, SS3 Concurrency/Context Switching, SS4 Locks/Atomicity, SS6 OS Scheduling/Virtual Memory, and SS8 Data Representation did not turn up a distinct, primary-source MAANGO/world-class engineering case study beyond what's already cited for the closely related sections above (e.g., Discord's Go-to-Rust case in SS1 also touches concurrency and memory management); rather than force a weak or tangential citation, these are left to the concept file's existing "where this resurfaces" pointers.
