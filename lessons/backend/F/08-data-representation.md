# Data Representation

*Everything a computer stores is just bytes — the meaning is a shared agreement about how to read them.*

`⏱️ ~6 min · 8 of 12 · Computing Fundamentals`

## Contents

- [The gist](#the-gist)
- [Intuition](#intuition)
- [How it works](#how-it-works)
- [Trade-offs](#trade-offs)
- [Remember](#remember)
- [Check yourself](#check-yourself)

> [!TIP] The gist
> A computer only stores bits (0/1), grouped into bytes (8 bits). Hex is a human shorthand for binary. Text becomes bytes through an *encoding* — UTF-8 won because it's ASCII-compatible and covers all of Unicode. And multi-byte numbers have a byte *order* (endianness) that both sides must agree on, or binary data silently corrupts.

## Intuition

A byte is like a **word written in an unknown alphabet**. The same 8 bits `01000001` could be the number 65, the letter "A", or one slice of a color code. Nothing about the bits themselves tells you which — you need a *convention* agreed in advance. Data representation is the study of those conventions.

## How it works

**Bits, bytes, and two's complement.**
A **bit** is 0 or 1. A **byte** is 8 bits → 256 values (0-255 unsigned, or -128 to 127 signed). Signed numbers use **two's complement**, a clever scheme where ordinary binary addition still works correctly across positive and negative values.

<br>

**Binary and hex.**
Hardware runs on binary (base 2). Humans prefer **hexadecimal** (base 16) because each hex digit maps to exactly 4 bits — so one byte is always exactly 2 hex digits (`0xFF` = `11111111` = 255). That clean mapping is why memory addresses, hashes, and color codes are shown in hex.

<br>

**Character encoding: ASCII → Unicode → UTF-8.**

- **ASCII** — a 7-bit code (128 values) for English letters, digits, punctuation. Fine for 1960s America, useless for the rest of the world's scripts and emoji.
- **Unicode** — a universal *numbering* of every character, called a **code point** (`U+0041` = "A", `U+1F600` = 😀). ~150,000 assigned and growing. But Unicode only assigns numbers; it doesn't say how to turn them into bytes.
- **UTF-8** — the dominant *encoding* of those code points into 1-4 bytes. Code points 0-127 encode as a single byte identical to ASCII, so all valid ASCII is automatically valid UTF-8. Compact for English, full Unicode coverage, drop-in ASCII compatibility — that trio is why it took over the web. (UTF-16, 2 or 4 bytes, survives inside JavaScript strings, Java, and Windows APIs for historical reasons.)

<br>

**Endianness — byte order for multi-byte values.**
When a 4-byte integer is stored or sent, its bytes need *some* order:

- **Big-endian** — most-significant byte first (the way humans write numbers, left to right). This is the historical **"network byte order."**
- **Little-endian** — least-significant byte first. Used internally by x86 and most ARM.

Get it wrong and binary data corrupts *silently* — reading a big-endian-written integer on a little-endian machine without conversion gives a garbage number, not a crash. Text formats like JSON dodge this entirely: they write numbers as ASCII digit characters, not raw multi-byte binary.

## Trade-offs

**Why a full-stack engineer should care:**

- ✅ Decode a byte stream with the *right* encoding → correct text.
- ❌ Wrong assumed encoding → "mojibake" (garbled characters).
- ✅ Binary formats that define byte order explicitly → portable across machines.
- ❌ Hand-parsing binary without respecting endianness → silent corruption.

Text formats (JSON) are self-describing and endianness-immune; binary formats are compact but demand an explicit, agreed byte layout.

> [!IMPORTANT] Remember
> Bytes carry no inherent meaning — text needs an agreed *encoding* and multi-byte numbers need an agreed *byte order*. Every "garbled text" or "wrong number over the wire" bug traces back to two sides disagreeing on the convention.

## Check yourself

1. Why is any valid ASCII file automatically a valid UTF-8 file, but not necessarily a valid UTF-16 file?
2. A service writes a 4-byte integer to a file on a big-endian machine; another reads it on a little-endian laptop with no conversion. What happens — a crash, or something worse?

---

→ Next: [Serialization](09-serialization.md)
↩ Comes back in: directly underpins Serialization (byte layouts) and Hashing (functions operate over raw bytes) in this same level
