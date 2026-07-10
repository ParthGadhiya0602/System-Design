# Normalization Forms

_A step-by-step discipline for shaping relations so that every fact is stored exactly once — and a precise vocabulary (functional dependency, normal form) for proving when a schema has actually achieved that._

## Contents

- [The problem: what an unnormalized table actually costs you](#the-problem-what-an-unnormalized-table-actually-costs-you)
- [Functional dependencies: the theoretical backbone](#functional-dependencies-the-theoretical-backbone)
- [First normal form (1NF)](#first-normal-form-1nf)
- [Second normal form (2NF)](#second-normal-form-2nf)
- [Third normal form (3NF)](#third-normal-form-3nf)
- [Boyce-Codd normal form (BCNF)](#boyce-codd-normal-form-bcnf)
- [4NF and 5NF: a brief mention](#4nf-and-5nf-a-brief-mention)
- [Denormalization: the deliberate trade-off](#denormalization-the-deliberate-trade-off)
- [Trade-offs: normalized vs denormalized](#trade-offs-normalized-vs-denormalized)
- [How this connects](#how-this-connects)
- [Check yourself](#check-yourself)

## The problem: what an unnormalized table actually costs you

Consider a system like Instagram, where users publish content — posts, stories, reels — and imagine someone modeling it with one flat, "everything in one table" design, the kind a spreadsheet-first approach naturally produces. (This is an illustration of how such a domain *could* be shaped, not a claim about any real internal schema.)

```
Content_Unnormalized
content_id | content_type | username  | email          | bio        | caption      | hashtags                    | created_at
5001       | post         | ava.codes | ava@mail.com   | "iOS dev"  | "sunset run" | "#run, #sunset, #fitness"   | 2026-06-01
5002       | reel         | ava.codes | ava@mail.com   | "iOS dev"  | "coffee tour"| "#coffee, #travel"          | 2026-06-02
5003       | story        | ben.makes | ben@mail.com   | "3D artist"| "wip render" | "#blender, #3d"             | 2026-06-02
```

This single relation mixes facts about two different real-world things — a piece of content and the user who authored it — into one row, plus a multi-valued `hashtags` cell, and that mixing is what creates three classic **anomalies**:

- **Update anomaly** — Ava's `email` and `bio` are stored on every row she authors (5001, 5002). If she edits her bio, every row mentioning her account must be updated in lockstep; miss one and the table now silently disagrees with itself about her bio. The redundancy isn't just wasted space — it's a live correctness risk that grows with every post she makes.
- **Insertion anomaly** — you cannot record that "a brand-new account exists with this email and bio" until that account actually publishes a piece of content, because profile data only exists as a side-effect of a content row. A user who has signed up but posted nothing simply cannot be represented.
- **Deletion anomaly** — if content 5003 is the *only* row mentioning `ben.makes`, deleting that one story simultaneously (and unintentionally) erases Ben's entire profile — email, bio, the fact that the account exists at all — because the user fact was never separated from the content fact.

**Normalization is the systematic process of decomposing a relation like this into smaller relations, each describing exactly one kind of real-world fact, connected back together via the keys covered in the [relational model](01-relational-model.md#keys) — specifically so all three anomalies above become structurally impossible, not just something you promise to be careful about.** Each successive normal form (1NF, 2NF, 3NF, BCNF, ...) is a stronger, precisely defined rule that eliminates a specific category of redundancy, and a table can be tested and proven to satisfy — or violate — a given normal form, exactly the way it can be proven to satisfy a key constraint.

## Functional dependencies: the theoretical backbone

Every normal form above 1NF is defined in terms of one idea: the **functional dependency (FD)**.

**Definition:** for a relation `R`, attribute set `X` **functionally determines** attribute set `Y` (written `X → Y`) if, for any two tuples in `R`, whenever they agree on the values of `X`, they must also agree on the values of `Y`. Informally: knowing `X`'s value is enough to look up exactly one `Y` value — `X` acts like a lookup key for `Y`, even if `X` isn't the relation's overall primary key.

Some vocabulary that the normal-form definitions below lean on directly:

- **Determinant** — the left-hand side (`X`) of an FD; whatever functionally determines something else.
- **Trivial FD** — `X → Y` where `Y` is a subset of `X` (e.g. `{content_id, username} → username`); always true, never carries information, and normal-form definitions explicitly ignore these.
- **Full functional dependency** — `X → Y` holds, and removing *any* attribute from `X` breaks the dependency (no proper subset of `X` alone determines `Y`). This is the exact opposite of a **partial dependency**, below.
- **Partial dependency** — `X → Y` holds where `X` is a composite key, but some proper subset of `X` alone *already* determines `Y` — i.e., `Y` doesn't actually need the whole key, just part of it. This is precisely the redundancy 2NF eliminates.
- **Transitive dependency** — `X → Y` and `Y → Z` together imply `X → Z`, but the *real* determinant of `Z` is `Y`, not `X` directly — `Z` depends on `X` only "through" `Y`. This is precisely the redundancy 3NF eliminates.

In the unnormalized `Content` table above: `content_id → content_type, username, email, bio, caption, created_at` (content_id is the only candidate key, so it trivially determines everything about the content). But *within* that, smaller FDs also hold that have nothing to do with `content_id` at all: `username → email` (a username uniquely identifies one account's email) and `username → bio`. Once we add a tagged location later, `location_name → location_city` will join them. **These smaller, "hidden" FDs — dependencies that hold *inside* a row for reasons unrelated to the table's actual key — are exactly what the normal forms are designed to detect and separate out.** Normalization, formally, is the process of decomposing a relation so that every remaining non-trivial FD has a determinant that is a candidate key of its own relation — nothing depends on "part of the key" or "something other than the key."

## First normal form (1NF)

**Definition:** every attribute value must be **atomic** (indivisible) — no repeating groups, no multi-valued attributes, no nested relations inside a cell. This is precisely the assumption already built into the definition of a relation's [domain](01-relational-model.md#what-a-relation-actually-is) — 1NF is really just the formal statement "a relation, as Codd defined it, must already look like this."

**What it eliminates:** the inability to query, index, or constrain individual values when several are crammed into one cell, and the ambiguity of how many values a cell may hold.

**Violation example** — the `hashtags` column stuffs several tags into one string:

```
Content_violates_1NF
content_id | caption      | hashtags
5001       | "sunset run" | "#run, #sunset, #fitness"
```

This breaks immediately: you cannot `WHERE hashtag = '#fitness'` without parsing a string, cannot count posts per hashtag or build a "trending tags" query, cannot index individual tags, and cannot cleanly represent "zero hashtags" or "thirty hashtags" — the shape of the data isn't fixed by the schema at all, it's smuggled into a string.

**1NF fix** — pull the repeating group out into its own relation, one row per (content, hashtag) pairing:

```
Content (1NF)
content_id PK | content_type | username  | email        | bio        | caption      | created_at
5001          | post         | ava.codes | ava@mail.com | "iOS dev"  | "sunset run" | 2026-06-01
5002          | reel         | ava.codes | ava@mail.com | "iOS dev"  | "coffee tour"| 2026-06-02
5003          | story        | ben.makes | ben@mail.com | "3D artist"| "wip render" | 2026-06-02

Post_Hashtags (1NF)  -- composite key {content_id, hashtag}
content_id | hashtag
5001       | #run
5001       | #sunset
5001       | #fitness
5002       | #coffee
5002       | #travel
```

Every cell is now atomic, so this is a valid relation by Codd's definition, and hashtags are individually queryable and indexable — but the `Content` table is still riddled with the anomalies from the opening section (Ava's email and bio are still duplicated across every row she authors). 1NF is a **necessary floor, not a fix for redundancy** — the remaining normal forms do that work. Note that `Content`'s primary key is the *single* column `content_id`; that matters directly for the next section, because a partial dependency can only exist against a *composite* key.

## Second normal form (2NF)

**Definition:** the relation is in 1NF, **and** every non-key attribute is **fully** functionally dependent on the *entire* candidate key — no non-key attribute depends on only part of a composite key. (2NF is automatically satisfied whenever the primary key is a single column, since a partial dependency requires a *composite* key to be partial *on*.)

**What it eliminates:** the specific redundancy caused by a **partial dependency**.

Because `Content`'s key is the single column `content_id`, it cannot violate 2NF at all — there's no partial key to depend on part of. To see a genuine partial dependency we need a table with a real composite key. Consider the table that records who viewed each story:

```
Story_Views (violates 2NF)  -- candidate key {post_id, viewer_user_id}
post_id | viewer_user_id | viewer_username | viewed_at
5003    | 88             | om.travels      | 2026-06-02 09:15
5003    | 91             | lia.paints      | 2026-06-02 09:20
5010    | 88             | om.travels      | 2026-06-03 20:01
```

The candidate key is `{post_id, viewer_user_id}` — a single view is uniquely identified by *which story* and *which viewer*. Check each non-key attribute against that whole key:

- `viewed_at` depends on the *whole* key — the timestamp of *this viewer viewing this specific story* genuinely needs both parts. This one is fine and stays.
- `viewer_username` depends only on `viewer_user_id` (part of the key) — a **partial dependency**. `om.travels` is repeated on every single story that user 88 views. The three anomalies are all here in miniature: if user 88 changes their username, every one of their view rows must be updated together (**update**); a username can't be recorded until that user has viewed at least one story (**insertion**); and deleting user 88's last view row erases the fact that id 88 maps to `om.travels` at all (**deletion**).

**2NF fix** — split the partially-dependent attribute out into its own relation, keyed by the part it actually depends on:

```
Story_Views (2NF)                          Users (2NF)
post_id | viewer_user_id | viewed_at       user_id PK | username
5003    | 88             | 09:15           88         | om.travels
5003    | 91             | 09:20           91         | lia.paints
5010    | 88             | 20:01
```

Now `viewer_user_id` in `Story_Views` is a foreign key to `Users`, and each username lives exactly once regardless of how many stories that account viewed. Notice that `Users` describes the very same real-world thing as the `username`/`email`/`bio` fields still sitting inside `Content` — that's a hint the schema isn't finished, which 3NF makes precise.

## Third normal form (3NF)

**Definition:** the relation is in 2NF, **and** no non-key attribute is **transitively dependent** on the key — i.e., every non-key attribute must depend directly on the key, not on some *other* non-key attribute that itself depends on the key.

**What it eliminates:** the redundancy caused by a **transitive dependency** — a chain `key → A → B`, where `B` should really live in its own relation keyed by `A`.

**Violation, in the `Content` table:** suppose each post can be tagged with a location, so we add `location_name` and `location_city`:

```
Content (violates 3NF, location columns added)
content_id PK | content_type | username  | email        | ... | location_name    | location_city
5001          | post         | ava.codes | ava@mail.com | ... | Baker Beach      | San Francisco
5002          | reel         | ava.codes | ava@mail.com | ... | Blue Bottle Cafe | San Francisco
5003          | story        | ben.makes | ben@mail.com | ... | Baker Beach      | San Francisco
```

The key is `content_id`. But within it: `content_id → location_name` (each post is tagged at one place) **and** `location_name → location_city` (a given place sits in one city). So `content_id → location_city` only holds *transitively*, through `location_name` — the city doesn't actually depend on the post at all, it depends on the location. The symptom is the familiar redundancy pattern: `Baker Beach → San Francisco` is duplicated on every post tagged there, correcting the city means finding every such post (**update anomaly**), you can't record a location's city until some post tags it (**insertion anomaly**), and deleting the only post tagged at a location erases the location-to-city fact entirely (**deletion anomaly**).

**3NF fix** — split the transitively-dependent attribute out into its own relation, keyed by the thing it actually depends on:

```
Content (3NF)                                          Locations (3NF)
content_id PK | content_type | author_id FK | ... | location_name FK    location_name PK | location_city
5001          | post         | 12           | ... | Baker Beach         Baker Beach      | San Francisco
5002          | reel         | 12           | ... | Blue Bottle Cafe    Blue Bottle Cafe | San Francisco
5003          | story        | 27           | ... | Baker Beach
```

The `Content` table actually carried a *second* transitive dependency for exactly the same reason: `content_id → username → {email, bio}`. Those profile fields describe the user, not the piece of content, so they fold into the `Users` relation that already emerged in the 2NF step, and `Content` keeps just a foreign key to the author (`author_id`). Now every relation in the schema has exactly one theme (`Users` describes accounts, `Content` describes posts/stories/reels, `Locations` describes places, `Post_Hashtags`/`Story_Views` describe the many-to-many relationships between them), and every non-key attribute depends on nothing but "the whole key, and nothing but the key" — the classic informal summary of 3NF. All three original anomalies are now gone: a location's city is stored once and can be corrected in one place; a new account or a new location can be inserted before it appears on any content; and deleting a story no longer risks silently deleting a user's profile along with it.

## Boyce-Codd normal form (BCNF)

**Definition:** a stricter version of 3NF — for **every** non-trivial functional dependency `X → Y` in the relation, `X` must be a **superkey** of the whole relation. 3NF has a narrow exception BCNF closes: 3NF permits a determinant that is *not* a superkey, as long as everything it determines is itself part of some candidate key. BCNF allows no such exception — every determinant, full stop, must be a superkey.

**What it eliminates:** a rarer, subtler redundancy that only shows up with **overlapping composite candidate keys** — two or more candidate keys that share an attribute — where a 3NF-satisfying table can still redundantly store data. This case is uncommon enough in ordinary schema design that most practical designs stop at 3NF and never actually encounter it, but it's worth seeing the canonical shape once. (The content-domain example above never naturally produces a BCNF violation, so the classic abstract example below is the clearest way to see it.)

**Classic example:** a table tracking which instructor teaches which subject to which student, with the rule "each instructor teaches only one subject" (but a subject can be taught by multiple instructors, and a student can study a subject with more than one instructor over time):

```
Teaching
student  | subject | instructor
Priya    | Math    | Dr. Lee
Priya    | Physics | Dr. Rao
Omar     | Math    | Dr. Lee
Omar     | Math    | Dr. Singh
```

Candidate keys here are `{student, subject}` (a student studies a given subject with possibly-varying instructors over time — actually, given the FD below, this needs care) — more precisely, the two candidate keys are `{student, subject}` and `{student, instructor}`, because either pair, together with the business rule, determines the third attribute. The FD `instructor → subject` holds (each instructor teaches exactly one subject) — but `instructor` alone is **not** a superkey (multiple rows can share the same instructor, e.g. `Dr. Lee` appears twice, for two different students). This relation is already in 3NF (`subject` is part of a candidate key, so the 3NF exception clause forgives the dependency), yet it's still redundant: `Dr. Lee → Math` is stated twice, and if Dr. Lee is reassigned to teach Chemistry instead, both rows must be updated together or the table contradicts itself.

**BCNF fix** — split out the dependency whose determinant isn't a superkey:

```
Instructor_Subject (BCNF)          Teaching_Assignments (BCNF)
instructor PK | subject            student | instructor FK
Dr. Lee       | Math               Priya   | Dr. Lee
Dr. Rao       | Physics            Priya   | Dr. Rao
Dr. Singh     | Math               Omar    | Dr. Lee
                                    Omar    | Dr. Singh
```

Now `Dr. Lee teaches Math` is stated exactly once, no matter how many students study under him. **The trade-off BCNF makes explicit:** decomposing to BCNF can occasionally make it impossible to preserve every original functional dependency using only foreign-key constraints on the decomposed relations (a known limitation called "BCNF may not be dependency-preserving," `verify` a fully worked counterexample before teaching it in depth) — which is one practical reason many schema designers treat 3NF as "normalized enough" by default and only push to BCNF when they can concretely demonstrate this overlapping-key redundancy exists in their own data.

## 4NF and 5NF: a brief mention

Two further normal forms exist for even rarer cases, worth knowing by name but not worth deep study for most schema design work. **Fourth normal form (4NF)** eliminates redundancy caused by **multi-valued dependencies** — a table where two independent, unrelated multi-valued facts are forced together in one relation (e.g. a table listing every (employee, skill, language) combination as a plain Cartesian product of an employee's skills and languages, when skills and languages have nothing to do with each other) creates redundant rows for every skill/language pairing; the fix is splitting the two independent multi-valued facts into two separate relations. **Fifth normal form (5NF, also called project-join normal form)** goes one step further, eliminating redundancy from **join dependencies** that can't be reduced to a single multi-valued dependency, where a relation can only be losslessly reconstructed by joining three or more decomposed relations together (not just two). Both are genuinely rare in everyday OLTP schema design — most production schemas never need to reason about them explicitly — but they exist as the formal end of the same ladder: each normal form closes off one more precisely defined category of redundancy, all the way up to a relation that contains no redundancy expressible as a functional, multi-valued, or join dependency at all.

## Denormalization: the deliberate trade-off

Everything above pushes toward *more* decomposition, because each step provably removes some category of update/insert/delete anomaly. **Denormalization deliberately reverses part of that** — reintroducing controlled redundancy on purpose — and it is a real, common, professionally-accepted design choice, not a mistake. The key point is that the decision is made *per access pattern*: the same application will normalize some data hard and deliberately denormalize other data. Consider a system like WhatsApp, which mixes both.

**Where normalization stays the right answer — group-message status.** In a group chat, one message is delivered to many recipients, and each recipient reaches "delivered" and "read" at their own time. That is a genuine many-to-many status relationship, and the correct, normalized home for it is a junction table:

```
Message_Recipients  -- composite key {message_id, recipient_id}
message_id | recipient_id | delivered_at        | read_at
9001       | ava          | 2026-06-01 10:00:02 | 2026-06-01 10:03:11
9001       | ben          | 2026-06-01 10:00:04 | (null)
9001       | lia          | (null)              | (null)
```

Both `delivered_at` and `read_at` depend on the *whole* key `{message_id, recipient_id}` — this is already in 2NF/3NF and needs no further splitting. Trying to *avoid* this table — say, cramming "delivered to Ava+Ben, read by Ava" into a column on the single message row — would reintroduce a multi-valued attribute (a **1NF** violation) and an update anomaly, since every individual read-receipt tick would have to rewrite the one shared message row. Storing status once per `(message, recipient)` pair keeps each per-recipient fact in exactly one place; this is a small worked example of why a many-to-many status relationship earns its own table rather than being denormalized away.

**Where denormalization is the right answer — call records for analytics.** Metadata for a video/voice call — `call_id`, its participants, `started_at`, `ended_at`, and quality metrics such as average bitrate and packet-loss rate — is naturally logged as a single self-contained record (an event), rather than shredded across half a dozen fully normalized relations. The reason is a general truism about this shape of data: it is **write-once, read-many**. The record is written once when the call ends and thereafter only *read*, in bulk, by analytics and quality dashboards that scan huge numbers of calls. A strictly normalized design would force every such scan to re-join participant, device, and network tables; keeping the record wide and denormalized trades a little redundancy (the same participant may appear across many call rows) for scan-friendly reads. This directly foreshadows the OLTP-vs-OLAP topic below, where analytical stores lean deliberately wide/denormalized for exactly this reason.

With those two poles in mind, the general reasoning:

- **Why:** every join at query time costs something — the query planner must locate and combine matching rows from multiple relations, which usually means extra index lookups or a hash/merge join step, and that cost compounds as tables grow large and as more relations must be joined per query. If a specific read path is executed extremely often (e.g. "render a chat's message list with each message's per-recipient read state," or "assemble a user's feed"), storing a precomputed, redundant copy of that joined shape can turn several joins into a single-table read.
- **When it's the right call:** **read-heavy workloads** where the same joined shape is read far more often than the underlying facts change (the call-analytics dashboards above; a social feed; a leaderboard), and **at scale**, where the underlying tables have grown large enough, or are sharded/partitioned across multiple nodes, that a join would have to cross partition or network boundaries — at that point, the "just join it" answer that works fine on a single small database stops being cheap, and precomputing/duplicating data becomes the practical answer.
- **How it's usually done safely** rather than by hand-maintaining duplicate copies everywhere: **materialized views** (a query result physically stored and periodically or incrementally refreshed by the database itself), **read replicas with a denormalized reporting/analytics schema** separate from the normalized transactional schema (the natural home for those wide call records), or **application-managed denormalized caches/tables** kept in sync via triggers, background jobs, or change-data-capture pipelines (forward-ref, CDC, a later L2 topic) — each of these reintroduces redundancy in a *controlled*, single well-known place, rather than scattering copies across arbitrary application code paths the way the original unnormalized table did.
- **The anomaly risk doesn't vanish, it moves.** Denormalizing reintroduces exactly the same update-anomaly risk normalization eliminated — the redundant copy can now drift out of sync with the source of truth — the trade-off is a deliberate bet that (a) the read-performance win outweighs that risk for this specific access pattern, and (b) a controlled mechanism (the ones listed above) keeps the drift bounded and predictable rather than arbitrary.

## Trade-offs: normalized vs denormalized

| | Normalized (3NF/BCNF) | Denormalized |
|---|---|---|
| **Redundancy** | Minimal — each fact stored once | Deliberate duplication of some facts |
| **Update/insert/delete anomalies** | Structurally prevented | Reintroduced; must be managed (triggers, CDC, jobs, materialized views) |
| **Write cost** | Low — one row to change per fact | Higher — every denormalized copy must be updated, or accept staleness |
| **Read cost for a "real-world object"** | Higher — often needs joins across several relations | Lower — often a single-row/single-table lookup |
| **Storage** | Smaller (no duplicated data) | Larger (duplicated data) |
| **Best fit** | OLTP workloads with frequent, small writes to distinct entities; correctness-critical data (billing, inventory counts, ledgers, per-recipient message status) | Read-heavy workloads at scale; reporting/analytics; precomputed feeds/dashboards; write-once/read-many event logs; cases where a join would cross shard/network boundaries |
| **Typical mechanism** | Foreign keys + `JOIN` at query time | Materialized views, read replicas with a reporting schema, application-level denormalized tables kept in sync via CDC/triggers/background jobs |

## How this connects

- **Back to the relational model** — every definition here (relation, tuple, key, candidate key) is used exactly as [defined there](01-relational-model.md#what-a-relation-actually-is); functional dependency is the one genuinely new piece of vocabulary this topic adds on top.
- **Forward to indexing and joins** — normalization's cost is precisely "reconstructing one real-world object now needs a join across several relations"; the next L2 topics on indexing and query optimization are directly about making that join cost as low as possible, which is what makes normalization practical rather than merely theoretically clean.
- **Forward to OLTP vs OLAP** — OLTP schemas lean normalized (many small, frequent, correctness-critical writes to distinct entities — a piece of content, a user profile, a per-recipient message status — where anomalies would be actively dangerous); OLAP/analytical schemas lean deliberately denormalized (star/snowflake schemas, wide fact tables) because the workload is read-heavy, batch-updated, and optimized for scanning large amounts of data with minimal joins. The deliberately wide call-record logs from the denormalization section are a preview of exactly such an OLAP-style fact table, and a forthcoming L2 topic will make this contrast precise.
- **Forward to NoSQL/document modeling (L4)** — document databases (e.g. MongoDB) and wide-column stores routinely embed denormalized, nested copies of related data directly inside one document specifically to avoid the join altogether, because many NoSQL engines don't support efficient cross-partition joins at all — the modeling debate there ("embed vs reference") is, at its core, exactly this same normalized-vs-denormalized trade-off, just decided at schema-design time rather than left to be reversed later via a materialized view.
- **Forward to CDC and event sourcing (L2/L4)** — keeping a denormalized copy in sync with its normalized source of truth without manual, error-prone application code is precisely the problem change-data-capture pipelines are built to solve.

## Check yourself

- A single `Employees` table has columns `employee_id, department_id, department_name, department_budget`. Identify the functional dependency that causes a transitive-dependency problem, name which normal form is violated, and show the decomposition that fixes it.
- Explain, in terms of functional dependencies, exactly why a relation with a *single-column* primary key can never violate 2NF.
- Give one concrete reason a team might choose to denormalize a reporting table even though it reintroduces the exact update anomaly 3NF was designed to eliminate — and name one mechanism that keeps that reintroduced redundancy under control.
- In the BCNF `Teaching` example, explain why `{student, subject}` alone being a candidate key is not enough to make the table BCNF-compliant, and what additional FD causes the violation.
