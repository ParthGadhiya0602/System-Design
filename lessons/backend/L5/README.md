# L5. Distributed Systems Theory

*Reason rigorously about consistency, consensus, time, and coordination under partial failure.*

**In progress** -- 1 of 22 topics written. Work through them in order; each is a short, self-contained read (~5-8 min).

| # | Lesson | In one line | Status |
|---|--------|-------------|--------|
| 01 | [CAP and PACELC](01-cap-and-pacelc.md) | The unavoidable fork every replicated system hits — consistency or availability during a partition, latency or consistency the rest of the time. | ✅ |
| 02 | Consistency models (strong, eventual, causal, read-your-writes) | Explodes CAP's single "C or not-C" into the full hierarchy of guarantees a system can actually offer. | ⚪ |
| 03 | Linearizability vs serializability | Defines CAP's "C" with full rigor, and separates it cleanly from the transactional "serializability" it's often confused with. | ⚪ |
| 04 | Consensus (Paxos, Raft, ZAB) | How a group of unreliable nodes agrees on one value, even when some of them fail. | ⚪ |
| 05 | Leader election | Picking the one node that gets to coordinate, and detecting when it needs replacing. | ⚪ |
| 06 | Quorums | Revisited with the full consensus vocabulary — the majority arithmetic that decides which side of a partition gets to make progress. | ⚪ |
| 07 | Logical and vector clocks | Ordering events across machines without a shared clock, and detecting when two events are truly concurrent. | ⚪ |
| 08 | Hybrid logical clocks | Revisited — the practical, software-only middle ground between logical counters and wall-clock time. | ⚪ |
| 09 | Gossip protocol | Spreading state around a cluster without anyone needing to know the whole membership up front. | ⚪ |
| 10 | Failure detectors (phi accrual) | Deciding whether a silent node is dead or just slow — and how confident to be about it. | ⚪ |
| 11 | Merkle trees / anti-entropy / read-repair / hinted handoff | The machinery that finds and heals the drift AP systems deliberately allow. | ⚪ |
| 12 | Distributed locking and fencing tokens | Making sure only one node acts at a time, and catching the zombie that thinks it still holds the lock. | ⚪ |
| 13 | Idempotency | Making retries safe by guaranteeing an operation applied twice has the same effect as applied once. | ⚪ |
| 14 | Delivery semantics (at-most/at-least/exactly-once) | What a messaging system actually promises about whether your message arrives, and how many times. | ⚪ |
| 15 | 2PC / 3PC and saga | Coordinating a transaction across multiple services or databases, and the alternative that avoids blocking on it. | ⚪ |
| 16 | FLP impossibility | Why no algorithm can guarantee consensus in a fully asynchronous network — the theoretical floor everything above works around. | ⚪ |
| 17 | Byzantine fault tolerance | Reaching agreement even when some nodes actively lie, not just fail silently. | ⚪ |
| 18 | CRDTs | Data structures designed so concurrent, conflicting updates always merge automatically, with no coordination. | ⚪ |
| 19 | Chain replication | Arranging replicas in a line instead of a star, for strong consistency with a different latency/throughput profile. | ⚪ |
| 20 | Split-brain | What happens when a cluster's coordination breaks and two sides both think they're in charge. | ⚪ |
| 21 | Durable execution / workflow engines (Temporal, Cadence) *(emerging)* | Making long-running, multi-step business processes survive crashes and retries automatically. | ⚪ |
| 22 | Deterministic simulation testing *(emerging)* | Testing distributed systems by replaying the same simulated failures deterministically, instead of hoping to hit them in production. | ⚪ |

**Deeper reference:** [research/backend/L5](../../../research/backend/L5/01-cap-and-pacelc.md)

**Comes from:** [L4. NoSQL and Data at Scale](../L4/README.md) — replication, quorums, and clocks are the toolbox; L5 is the law those tools operate under.
