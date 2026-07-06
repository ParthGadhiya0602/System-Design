# Disks and Filesystems

*Where your bytes actually live when the power goes out, and why the machinery underneath decides how fast you can read and write them.*

`⏱️ ~7 min · 7 of 12 · Computing Fundamentals`

## Contents

- [The gist](#the-gist)
- [Intuition](#intuition)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

> [!TIP] The gist
> Persistent storage comes in two flavors: spinning HDDs (mechanical, cheap, slow at random access) and SSDs (flash, fast, but can't overwrite in place). The single most important fact is that **sequential I/O is far cheaper than random I/O** — and almost every storage engine on Earth is designed around that one truth.

## Intuition

Picture a **vinyl record player (HDD)**: to play a song, a physical arm has to swing to the right groove (a *seek*) and then wait for the disc to spin the song under the needle. Jumping between tracks all over the record is slow; playing straight through is smooth.

Now picture a **wall of numbered mailboxes (SSD)**: any box is instantly reachable, no arm to move. But there's a catch — you can't erase one letter in a box, you can only empty a whole **row** of boxes at once. That quirk shapes everything about how SSDs behave.

## How it works

**HDD — the mechanical one.**
Data sits on spinning magnetic platters. A read means the arm must *seek* to the right track (~5-10 ms) and wait for *rotational latency*. So a single random read costs roughly **~10 ms**, while reading contiguous data sequentially sustains ~100-200 MB/s. Random is orders of magnitude worse than sequential — remember this one asymmetry above all else.

<br>

**SSD — the flash one.**
No moving parts, so random reads drop to tens of microseconds. But flash is organized into **pages** (the read/write unit, ~4-16 KB) grouped into larger **blocks** (the erase unit). You can't overwrite a page in place — the block must be *erased* first.

<br>

**Write amplification.**
Because of erase-before-write, a small logical write forces the SSD to write to a fresh page elsewhere and later run **garbage collection** (copy live pages out of a stale block, then erase it). The upshot: the drive physically writes *more* bytes than you asked for. That ratio is **write amplification** — it slows writes and wears out flash cells (they die after a bounded number of erases). Firmware fights back with over-provisioning, wear leveling, and the **TRIM** command.

<br>

**NVMe** is not a medium — it's a *protocol* connecting flash over PCIe with thousands of parallel command queues, unlocking an SSD's internal parallelism. Rough IOPS ladder: HDD ~100-200 « SATA SSD ~100k < NVMe SSD hundreds of thousands to 1M+.

<br>

**Filesystems** (ext4, XFS, APFS, NTFS) sit on top of the raw block device: they organize bytes into files and directories, track free space, and store metadata (an **inode** on Unix maps a file to its blocks). Journaling filesystems even write metadata to an append-only journal first — the same "sequential writes are cheap and safe" idea applied for crash-consistency.

## In the real world

**RocksDB (Meta).** In 2012 Facebook forked Google's LevelDB into RocksDB because LevelDB's single-threaded compaction caused write stalls and left modern flash badly underused — contemporary SSDs could sustain over a million IOPS, but LevelDB couldn't drive that parallelism. RocksDB re-architected the LSM-tree engine with multi-threaded, tunable compaction so a team can deliberately balance read, write, and space amplification for their workload. It now underpins storage at Meta (MyRocks) and is used at other web-scale companies like LinkedIn.

See [f-computing-fundamentals-cases-and-sources.md](../../../research/backend/F/f-computing-fundamentals-cases-and-sources.md#ss7-disks-and-filesystems).

## Trade-offs

| | HDD | SSD (flash) |
|---|---|---|
| Random read | ❌ ~10 ms (seek + rotation) | ✅ tens of µs |
| Sequential throughput | ✅ ~100-200 MB/s | ✅✅ much higher |
| Overwrite in place | ✅ yes | ❌ no (erase whole block first) |
| Cost per GB | ✅ cheap | ❌ pricier |
| Wears out | rarely | ❌ bounded erase cycles |

The universal design move: **turn random writes into sequential ones**. A write-ahead log (WAL) is append-only; an LSM-tree buffers writes in memory and flushes them sequentially — trading some read cost for far better write throughput on both HDD and SSD.

> [!IMPORTANT] Remember
> Sequential I/O is cheap; random I/O is expensive. Every storage engine you'll ever meet is bending over backwards to make its writes look sequential.

## Check yourself

1. Why can't an SSD simply overwrite a page the way RAM can — and what background process does it run to reclaim space?
2. A logging service appends events one after another. Why is this workload naturally friendly to *both* HDDs and SSDs?

---

→ Next: [Data Representation](08-data-representation.md)
↩ Comes back in: L2 (WAL, B-trees vs LSM-trees, storage-engine internals), L4 (wide-column stores like Cassandra/HBase built on LSM-trees)
