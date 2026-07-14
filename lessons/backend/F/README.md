# F. Computing Fundamentals

_The bedrock. What a single machine actually does - so every distributed concept later rests on real mechanics, not magic._

Each lesson is a short, self-contained read (~5-8 min). Work through them in order, or jump to what you need.

| #   | Lesson                                                               | In one line                                                      |
| --- | -------------------------------------------------------------------- | ---------------------------------------------------------------- |
| 01  | [CPU & Memory Hierarchy](01-cpu-memory-hierarchy.md)                 | Why "fast" code stalls, and how caches hide the RAM gap.         |
| 02  | [Processes vs Threads](02-processes-vs-threads.md)                   | What's shared, what's private, and what each costs.              |
| 03  | [Concurrency vs Parallelism](03-concurrency-vs-parallelism.md)       | Juggling vs doing-at-once, and the cost of switching.            |
| 04  | [Locks & Atomicity](04-locks-and-atomicity.md)                       | Mutexes, semaphores, races, deadlock - sharing state safely.     |
| 05  | [I/O Models](05-io-models.md)                                        | Blocking vs event loops, and how one thread serves thousands.    |
| 06  | [OS Scheduling & Virtual Memory](06-os-scheduling-virtual-memory.md) | How every process gets its own private memory, for free.         |
| 07  | [Disks & Filesystems](07-disks-and-filesystems.md)                   | HDD vs SSD vs NVMe, and why sequential beats random.             |
| 08  | [Data Representation](08-data-representation.md)                     | Bits to text: encoding, Unicode/UTF-8, endianness.               |
| 09  | [Serialization](09-serialization.md)                                 | Turning objects into bytes: JSON vs Protobuf/Avro.               |
| 10  | [Compression](10-compression.md)                                     | Trading CPU for size: gzip, Snappy, LZ4, Zstd.                   |
| 11  | [Hashing](11-hashing.md)                                             | Fixed-size fingerprints, collisions, and where each hash fits.   |
| 12  | [Clocks](12-clocks.md)                                               | Wall vs monotonic time, skew, and why distributed order is hard. |

**Deeper reference:** [concepts](../../../research/backend/F/f-computing-fundamentals.md) · [case studies & sources](../../../research/backend/F/f-computing-fundamentals-cases-and-sources.md)

**Next level →** L0. System-Design Foundations
