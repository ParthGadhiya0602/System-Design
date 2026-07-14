# Serialization

_Turning a living in-memory object into a flat stream of bytes - so it can survive a trip across the network or a night on disk._

`⏱️ ~7 min · 9 of 12 · Computing Fundamentals`

## Contents

- [The gist](#the-gist)
- [Intuition](#intuition)
- [How it works](#how-it-works)
- [In the real world](#in-the-real-world)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

> [!TIP] The gist
> Serialization flattens an object into bytes for storage or transmission; deserialization rebuilds it. Text formats (JSON, XML) are human-readable but bulky; binary schema-based formats (Protobuf, Avro, Thrift) are compact and fast but need a predefined schema. The hard part is not the encoding - it's **schema evolution**: changing the shape of your data without breaking readers written yesterday.

## Intuition

An in-memory object is like a piece of **flat-pack furniture assembled in your living room** - full of pointers and nested references. You can't mail it as-is. Serialization is disassembling it back into a flat, boxed kit; deserialization is following the instructions to rebuild it at the other end. The "schema" is the assembly manual both factories agree on.

## How it works

**Text formats - self-describing, human-friendly.**
**JSON** nests objects and arrays of strings, numbers, booleans, null - readable, debuggable, parseable in any language with zero code generation. The cost: every record repeats its field _names_, so it's larger and slower to parse than binary. **XML** is an older, even more verbose text format (open/close tags), still common in legacy SOAP and enterprise integration.

<br>

**Binary schema-based formats - compact, fast, typed.**
**Protobuf** (Google), **Avro** (Hadoop), and **Thrift** (Facebook → Apache) all require a predefined **schema** and generate typed code from it.

- **Protobuf** - fields get numeric **tags** in a `.proto` file (`string name = 1;`). The wire format stores tag + type + value using varints, never field names - much smaller than JSON for structured, numeric data.
- **Avro** - the schema (itself JSON) travels with the data or, more often, lives in a shared **schema registry**. Avro's bytes contain _no_ field tags at all, so decoding requires knowing the writer's schema - a natural fit for Kafka, where producers and consumers evolve independently.
- **Thrift** - like Protobuf, but historically bundled with its own RPC framework (as gRPC bundles Protobuf over HTTP/2).

<br>

Here's the shape of the same record two ways:

```mermaid
flowchart LR
    A["User { id: 7, name: 'Ada' }"] --> B["JSON (text)<br/>{\"id\":7,\"name\":\"Ada\"}<br/>repeats field names<br/>~26 bytes"]
    A --> C["Protobuf (binary)<br/>tag+varint + tag+len+'Ada'<br/>no field names<br/>~8 bytes"]
```

<br>

**Schema evolution - the real challenge.**
Data written today gets read for years, by code at different versions:

- **Backward compatible** - new code reads old data.
- **Forward compatible** - old code reads new data.

The safe pattern is identical across all binary formats: identify fields by a stable **tag/number** (never position or name alone), only ever _add_ optional fields with sensible defaults, and never change the meaning of an existing tag. Removed field numbers should be reserved so they're never reused.

## In the real world

**Confluent / Kafka - Avro plus a Schema Registry.** In an event-streaming pipeline, producers and consumers are decoupled in time but still coupled through the _shape_ of the data - one careless schema change can silently break every consumer. Confluent's Schema Registry pairs with Avro precisely because Avro has machine-checkable evolution rules: the registry enforces a compatibility mode (backward, forward, or full) _before_ allowing a new schema version, so producer and consumer teams evolve independently instead of coordinating deploys. This is the production-grade version of the evolution rules above, running in fintech- and e-commerce-scale Kafka pipelines.

See [f-computing-fundamentals-cases-and-sources.md](../../../research/backend/F/f-computing-fundamentals-cases-and-sources.md#ss9-serialization).

## Trade-offs

|                             | Text (JSON / XML)              | Binary + schema (Protobuf / Avro) |
| --------------------------- | ------------------------------ | --------------------------------- |
| Human-readable / debuggable | ✅ `curl` and read it          | ❌ needs tooling                  |
| Zero-tooling interop        | ✅ every language has a parser | ❌ needs schema + codegen         |
| Payload size                | ❌ larger (repeats names)      | ✅ often several times smaller    |
| (De)serialization speed     | ❌ slower text parsing         | ✅ faster                         |
| Compile-time type safety    | ❌                             | ✅ generated typed code           |

Rule of thumb: **public, loosely-coupled, human-debugged APIs → JSON/REST; high-throughput internal service-to-service calls and big data pipelines → binary/schema'd.**

> [!IMPORTANT] Remember
> Choosing a format is easy; keeping it readable across years of independent deploys is the real work. Add fields, never repurpose a tag.

## Check yourself

1. Avro's encoded bytes contain no field names or tags. What must a consumer therefore have in order to decode a message - and why does that make a schema registry so useful?
2. You add a new optional field to a Protobuf message. Why can old code, unaware of that field, still safely read the new data?

---

→ Next: [Compression](10-compression.md)
↩ Comes back in: L4/L6 (Kafka + Avro + schema registry pipelines), L10 (REST/JSON vs gRPC/Protobuf at the API layer), L13 (Parquet columnar formats for data lakes)
