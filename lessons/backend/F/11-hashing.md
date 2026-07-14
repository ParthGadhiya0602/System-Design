# Hashing

_One deterministic function that turns any input into a fixed-size fingerprint - and the whole trick is knowing which kind to use._

`⏱️ ~7 min · 11 of 12 · Computing Fundamentals`

## Contents

- [The gist](#the-gist)
- [Intuition](#intuition)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

> [!TIP] The gist
> A hash function maps any input to a fixed-size output, deterministically. Collisions are mathematically unavoidable - what matters is picking the _right kind_: fast non-cryptographic hashes for hash tables and sharding, CRC checksums for catching accidental corruption, cryptographic hashes for tamper-detection and content identity, and slow key-derivation functions (never a plain hash) for passwords.

## Intuition

A hash is like a **fingerprint**: any person (input), no matter their size, maps to a fixed-length print. The same person always gives the same print (deterministic), and two different people should give wildly different prints even if they look almost identical (the **avalanche effect** - flip one input bit, the output changes drastically). But because there are more people than possible fingerprints, two people _will_ eventually share one - a **collision**.

## How it works

**Collisions are guaranteed - resistance is the real property.**
Inputs are effectively infinite; outputs are fixed-size. By the pigeonhole principle, collisions _must_ exist. The meaningful question isn't "are there collisions" but "how hard is it to _find_ one on purpose" - **collision resistance**.

<br>

**Three families, three jobs.**

- **Non-cryptographic** (MurmurHash, xxHash, FNV, CityHash) - built purely for speed and even distribution, no defense against an adversary. Used for hash tables, sharding/partitioning keys, deduplication, load-balancer routing, Bloom filters. "Spread values evenly, fast."
- **Cryptographic** (SHA-256, SHA-3) - resist an adversary via pre-image, second-pre-image, and collision resistance. Used for content-addressed storage (Git names objects by their content hash), digital signatures, and HMAC. Note: **MD5 and SHA-1 are broken** - don't use them for security.
- **Checksums / CRC** (CRC32, CRC-32C) - detect _accidental_ corruption (bit flips from a flaky disk or network), not deliberate tampering. Fast, often hardware-accelerated; found in Ethernet frames, ZIP entries, gzip streams.

<br>

**Consistent hashing - hashing applied to distribute data.**
Map both keys _and_ nodes onto a circular hash space (a "ring"); a key belongs to the first node clockwise of it. Add or remove a node and only its immediate neighbor's keys move - not the whole cluster.

```
        node A
          o
   key3 . | . key1
        . | .
node D o--+--o node B   -> key1 walks clockwise -> lands on node B
        . | .
   key2 . | .
          o
        node C
```

To even out load (random node placement is lumpy), each physical node is given many **virtual nodes** - multiple ring positions sized to its capacity.

## In the real world

**Amazon Dynamo - consistent hashing with virtual nodes.** Dynamo (the design behind DynamoDB) hashes each key (an opaque byte array) onto a circular space and finds the first node clockwise. Because random node placement produced uneven load, each physical node gets many virtual-node positions sized to its actual capacity - now the standard technique in essentially every hash-partitioned data store.

**Apache Kafka - CRC on every record batch.** Every Kafka record batch carries a CRC (CRC-32C in the current format) over its bytes, computed by the producer and checked by brokers and consumers to catch corruption or truncation from a flaky disk or network - the textbook "accidental corruption, not tampering" job for a checksum rather than a cryptographic hash.

See [f-computing-fundamentals-cases-and-sources.md](../../../research/backend/F/f-computing-fundamentals-cases-and-sources.md#ss11-hashing).

## Trade-offs

Pick by _need_, not by habit:

| Need                                             | Use                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------- |
| Hash table / general in-memory lookup            | Fast non-cryptographic hash                                         |
| Sharding / partitioning, consistent hashing      | Non-cryptographic hash                                              |
| Detect accidental corruption (network/disk)      | CRC / checksum                                                      |
| Detect deliberate tampering, verify authenticity | Cryptographic hash / HMAC                                           |
| Content-addressed storage / dedup by identity    | Cryptographic hash                                                  |
| Password storage                                 | ❌ **not** a plain hash → slow, salted KDF (bcrypt, scrypt, Argon2) |

> [!IMPORTANT] Remember
> Same function, six different jobs - using the wrong family (a fast hash for passwords, a CRC against an attacker) is a classic, dangerous mistake. Match the hash to the threat model.

## Check yourself

1. Why is a fast cryptographic hash like SHA-256 still the _wrong_ tool for storing passwords, and what should you use instead?
2. In consistent hashing, why does adding one node only move a small fraction of keys - and what do virtual nodes fix?

---

→ Next: [Clocks](12-clocks.md)
↩ Comes back in: L4 (consistent hashing for partitioning and rebalancing), L9 (password hashing and HMAC in depth), L12 (Bloom filters, HyperLogLog and other probabilistic structures), L2 (hash indexes alongside B-trees)
