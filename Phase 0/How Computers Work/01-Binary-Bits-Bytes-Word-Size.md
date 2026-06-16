# Binary, Bits, Bytes, Word Size

> **Phase 0 — Computer Science Fundamentals → 0.1 How Computers Work**
> Goal: Understand how computers represent *everything* using only two states, and the fundamental units (bit, byte, word) that all higher concepts build on.

---

## 0. The One-Sentence Summary

Computers store and process all information as **bits** (0 or 1). Bits group into **bytes** (8 bits), bytes group into **words** (the natural processing unit of a CPU, e.g., 64 bits). Everything — numbers, text, images, code — is ultimately just patterns of bits.

---

## 1. Why Binary? (Base-2)

Computers are electronic. The simplest, most reliable way to store/transmit information electronically is with **two distinct states**:

| State | Physical meaning |
|-------|------------------|
| `0` | Low voltage / off / no charge |
| `1` | High voltage / on / charge present |

Two states are easy to tell apart reliably even with electrical noise → robust and cheap. This is why computers use **binary (base-2)** rather than decimal (base-10).

> A transistor — the basic building block of a CPU — is essentially a tiny switch that is either ON (1) or OFF (0). Modern chips have **billions** of them.

---

## 2. The Bit

- **Bit** = **Bi**nary dig**it** — the smallest unit of information.
- Holds exactly one value: **0** or **1**.
- *n* bits can represent **2ⁿ** different combinations.

| Bits | Combinations (2ⁿ) | Example range (unsigned) |
|------|-------------------|--------------------------|
| 1 | 2 | 0–1 |
| 2 | 4 | 0–3 |
| 4 | 16 | 0–15 |
| 8 | 256 | 0–255 |
| 16 | 65,536 | 0–65,535 |
| 32 | ~4.29 billion | 0 – 4,294,967,295 |
| 64 | ~1.8 × 10¹⁹ | very large |

---

## 3. The Byte

- **Byte** = **8 bits**, grouped together.
- The standard **unit of addressable memory** — memory is measured and addressed in bytes, not bits.
- One byte = 2⁸ = **256** possible values (0–255 unsigned).

```
A single byte:
  bit position:  7  6  5  4  3  2  1  0
  value:         1  0  1  1  0  1  0  0   = ?

  128 + 0 + 32 + 16 + 0 + 4 + 0 + 0 = 180
```

### 3.1 Nibble
- **Nibble** = 4 bits = half a byte. Maps neatly to one **hexadecimal** digit (0–F).

### 3.2 Larger units (two conventions — important!)
| Unit | Decimal (SI, base-10) | Binary (IEC, base-2) |
|------|----------------------|----------------------|
| Kilobyte | 1 KB = 1,000 bytes | 1 KiB = 1,024 bytes |
| Megabyte | 1 MB = 1,000,000 | 1 MiB = 1,048,576 |
| Gigabyte | 1 GB = 10⁹ | 1 GiB = 2³⁰ |
| Terabyte | 1 TB = 10¹² | 1 TiB = 2⁴⁰ |

> Disk manufacturers use **decimal** (1 TB = 10¹² bytes); operating systems often use **binary** (1 TiB = 2⁴⁰). That's why a "1 TB" drive shows up as ~931 GB in your OS.

---

## 4. Number Systems You Must Know

| System | Base | Digits | Prefix | Use |
|--------|------|--------|--------|-----|
| Binary | 2 | 0–1 | `0b` | How hardware works |
| Octal | 8 | 0–7 | `0` | Unix file permissions (`chmod 755`) |
| Decimal | 10 | 0–9 | none | Human everyday |
| Hexadecimal | 16 | 0–9, A–F | `0x` | Compact binary (memory addresses, colors, bytes) |

### 4.1 Why hexadecimal is everywhere
**One hex digit = exactly 4 bits (one nibble).** So **two hex digits = one byte**. This makes hex a compact, human-friendly way to read binary:

```
Binary:  1011 0100
Hex:        B    4     ->  0xB4
Decimal:          180
```

You'll see hex constantly in: memory addresses (`0x7FFE...`), color codes (`#FF5733`), byte dumps, and Java literals (`int x = 0xFF;`).

### 4.2 Binary ↔ Decimal conversion
**Binary → Decimal:** multiply each bit by its positional power of 2 and sum.
```
1011 = 1×2³ + 0×2² + 1×2¹ + 1×2⁰ = 8 + 0 + 2 + 1 = 11
```
**Decimal → Binary:** repeatedly divide by 2, read remainders bottom-up.
```
13 / 2 = 6 r1
 6 / 2 = 3 r0
 3 / 2 = 1 r1
 1 / 2 = 0 r1     ->  read up:  1101  = 13
```

---

## 5. Representing Different Data

Everything is bits — *interpretation* gives meaning:

| Data type | How it's encoded in bits |
|-----------|--------------------------|
| **Unsigned integer** | Direct binary value |
| **Signed integer** | **Two's complement** (see §6) |
| **Floating point** | IEEE 754 (sign + exponent + mantissa) |
| **Text / characters** | Encodings: ASCII (1 byte), UTF-8 (1–4 bytes), UTF-16 |
| **Images** | Pixels → RGB byte values |
| **Instructions** | Opcodes → bit patterns the CPU decodes |

> The **same bits** mean different things depending on how they're interpreted. `01000001` is the number 65, the character 'A' (ASCII), or part of a machine instruction — context decides.

---

## 6. Signed Numbers: Two's Complement (essential)

How do you store negative numbers with only 0s and 1s? Modern computers use **two's complement**:
- The **leftmost bit is the sign bit** (0 = positive, 1 = negative).
- To negate a number: **invert all bits, then add 1.**

```
+5 in 8 bits:  0000 0101
invert:        1111 1010
add 1:         1111 1011   = -5
```

An *n*-bit two's-complement integer ranges from **−2ⁿ⁻¹ to 2ⁿ⁻¹ − 1**.

> This is exactly why Java's `int` (32-bit) ranges from −2,147,483,648 to +2,147,483,647, and why `Integer.MAX_VALUE + 1` **overflows** to `Integer.MIN_VALUE` (the bits wrap around).

---

## 7. Word Size

- A **word** is the **natural unit of data a CPU processes in one operation** — typically the size of its registers and the width of its data bus.
- Common word sizes: **32-bit** and **64-bit** (older systems: 8/16-bit).

### 7.1 What word size determines
| Affects | 32-bit | 64-bit |
|---------|--------|--------|
| Register width | 32 bits | 64 bits |
| Max directly addressable memory | 2³² = **4 GB** | 2⁶⁴ = **~16 exabytes** (practically huge) |
| Integer processed per operation | up to 32 bits | up to 64 bits |

> The biggest practical reason desktops moved to 64-bit: a 32-bit system **cannot address more than 4 GB of RAM** (2³² bytes). 64-bit removes that ceiling.

### 7.2 Word size ≠ byte
A byte is always 8 bits. A **word** depends on the CPU architecture. People also use **half-word**, **double-word**, etc., relative to the architecture's word.

---

## 8. Endianness (awareness)

When a multi-byte value is stored in memory, which byte comes first?
- **Big-endian:** most significant byte first (network byte order).
- **Little-endian:** least significant byte first (x86, most PCs).

```
The 4-byte value 0x12345678 stored at address A:
  Big-endian:    A:12  A+1:34  A+2:56  A+3:78
  Little-endian: A:78  A+1:56  A+2:34  A+3:12
```
> Matters when reading binary files or network protocols. The JVM abstracts this away for normal code, but it surfaces in `ByteBuffer` and serialization.

---

## 9. How This Maps to Java

These fundamentals directly define Java's primitive types (Phase 1.2):

| Java type | Size (bits) | Notes |
|-----------|-------------|-------|
| `byte` | 8 | Two's complement, −128 to 127 |
| `short` | 16 | −32,768 to 32,767 |
| `char` | 16 | **Unsigned**, a UTF-16 code unit (0–65,535) |
| `int` | 32 | −2³¹ to 2³¹−1 |
| `long` | 64 | −2⁶³ to 2⁶³−1 |
| `float` | 32 | IEEE 754 single precision |
| `double` | 64 | IEEE 754 double precision |
| `boolean` | (1 logically) | JVM-dependent storage |

- Java's **bitwise operators** (`&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`) operate directly on these bit patterns.
- The unsigned right shift `>>>` exists *because* of two's complement / the sign bit.
- Java's sizes are **fixed across all platforms** (unlike C), which is part of "write once, run anywhere."

---

## 10. Why This Matters for a Backend (Java) Engineer

- Explains **integer overflow** bugs (e.g., counters, IDs hitting `int` limits → use `long`).
- Explains **character encoding** issues (UTF-8 vs UTF-16) — a constant source of real bugs in APIs and databases.
- **Hex** literacy is needed for reading memory addresses, hashes, UUIDs, color codes, and byte streams.
- **KB vs KiB** confusion affects capacity planning and metrics.
- Bit manipulation underlies flags, permissions, bitmasks, Bloom filters, and performance tricks.
- Word size (32 vs 64-bit JVM) affects pointer size and memory footprint (compressed oops in the JVM).

---

## 11. Quick Self-Check Questions

1. Why do computers use binary instead of decimal?
2. How many values can *n* bits represent? How many in a byte?
3. Why is hexadecimal convenient for representing bytes?
4. Convert `1101` to decimal and `13` to binary.
5. How does two's complement represent −5 in 8 bits?
6. Why can a 32-bit system only address 4 GB of RAM?
7. What's the difference between a byte and a word?
8. What is endianness, and when does it matter?

---

## 12. Key Terms Glossary

- **Bit:** smallest unit of data; 0 or 1.
- **Byte:** 8 bits; basic unit of addressable memory.
- **Nibble:** 4 bits; one hex digit.
- **Binary (base-2):** number system computers use natively.
- **Hexadecimal (base-16):** compact representation; 1 digit = 4 bits.
- **Two's complement:** standard scheme for signed integers.
- **Overflow:** result exceeds the range a type can hold; wraps around.
- **Word:** the CPU's natural processing unit (register/bus width).
- **Word size:** number of bits in a word (e.g., 64-bit).
- **Endianness:** byte ordering of multi-byte values (big/little).
- **KB vs KiB:** decimal (1000) vs binary (1024) size units.
- **Encoding:** scheme mapping characters to byte patterns (ASCII, UTF-8).

---

*This is the first note of **Section 0.1 — How Computers Work**.*
*Next topic in roadmap: **CPU architecture basics (ALU, registers, cache, clock cycles)**.*
