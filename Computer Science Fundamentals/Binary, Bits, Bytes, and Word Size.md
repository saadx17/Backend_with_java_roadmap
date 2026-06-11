## PART 0: Basics
Binary, bits, bytes, and word size are the fundamental building blocks of digital computing. They dictate how data is represented, stored, and processed inside a device.
#### WHY Does This Matter for a Java Backend Engineering?

- Java's [[Variables & Data Types#What are Primitive Types?|primitive data]] types (`int`, `long`, `byte`, `short`) are defined by their **bit size**.
- Integer overflow bugs come from not understanding **binary limits**.
- Understanding **memory** (heap, stack) requires knowing how data is physically stored.
- Performance optimization requires knowing how CPUs read data (word size, alignment)
- [[Operators#`Bitwise` Operators|Bitwise Operators]] in Java (`&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`) make zero sense without this.
- Networking (IP addresses, subnet masks, ports) is all binary.

#### Summery
```java title:bits_bytes.java
BITS AND BYTES
─────────────────────────────────────────────
1 bit      = 0 or 1
4 bits     = 1 nibble = 1 hex digit
8 bits     = 1 byte
```

```java title:power_of_2.java
POWERS OF 2 (memorize these)
─────────────────────────────────────────────
2⁰  = 1
2¹  = 2
2²  = 4
2³  = 8
2⁴  = 16
2⁵  = 32
2⁶  = 64
2⁷  = 128
2⁸  = 256
2¹⁰ = 1,024         (≈ 1 thousand)
2¹⁶ = 65,536
2²⁰ = 1,048,576      (≈ 1 million)
2³² = 4,294,967,296  (≈ 4 billion)
2⁶⁴ = 18.4 quintillion
```

```java title:java_types.java
JAVA TYPES
─────────────────────────────────────────────
boolean → 1 bit (stored as byte)
byte    → 8 bits  (-128 to 127)
char    → 16 bits (0 to 65,535, unsigned)
short   → 16 bits (-32,768 to 32,767)
int     → 32 bits (-2.1B to 2.1B)
float   → 32 bits (IEEE 754)
long    → 64 bits (-9.2Q to 9.2Q)
double  → 64 bits (IEEE 754)
```

```java title:word_size.java
WORD SIZE (modern 64-bit system)
─────────────────────────────────────────────
Word = 8 bytes = 64 bits
Max RAM addressable = 2⁶⁴ bytes (theoretical)
Object reference = 8 bytes (4 with CompressedOops)

HEX PREFIX:  0xFF
BIN PREFIX:  0b11001100
LONG SUFFIX: 100L
FLOAT SUFFIX: 3.14f
```

## PART 1: What Is a Bit?
A **bit** (short for "binary digit") is the most fundamental and smallest unit of data in computing and digital communications. A computer is, at its core, a machine made of billions of tiny **switches**.
Each switch has **two states:**
```
OFF = 0
ON  = 1
```
That's it. That single switch is called a **BIT**.

**Bit = Binary Digit**
The word **binary** means "two states." Unlike our normal number system (decimal) which has 10 digits (0-9), binary only has **2 digits:** `0` and `1`.

#### Why Only Two States?
Because electronics is **reliable with two states:**
- There IS voltage → 1
- There is NO voltage → 0

You can't reliably distinguish between 10 different voltage levels at billions of operations per second. But you CAN reliably distinguish between "voltage present" and "no voltage."
This is why all computers, at their lowest level, speak binary.

#### How Bits Work?
At the physical level, bits are stored in hardware using different binary systems: 

- **Memory:** Stored as electrical voltages. A higher voltage represents a 1, and a lower voltage represents a 0.
- **Hard Drives:** Stored as magnetic fields. Different polarities represent 1 and 0.
- **CDs/DVDs:** Stored as tiny physical "pits" (which represent a 1) and flat "lands" (which represent a 0).

## PART 2: What Is a Byte?
A **byte** is the fundamental unit of digital information in computing and telecommunications.

A single bit (0 or 1) can't represent much. So we **group bits together.** The standard group size is **8 bits = 1 BYTE**.

```java title:byte.java
1 BYTE = 8 BITS

Example byte:  1 0 1 1 0 1 1 0
               ^             ^
            bit 7           bit 0
           (leftmost)      (rightmost)
           Most Significant  Least Significant
           Bit (MSB)         Bit (LSB)
```

#### Why Bytes Matter?
Because a single bit can only represent two values (0 or 1), computer scientists group bits together into bytes to create larger, usable units of data.
- **Text representation:** One byte is traditionally used to represent a single text character, such as a letter, number, or punctuation mark.
- **Combinations:** By arranging 8 bits in different orders, a single byte can represent exactly 256 different values (2⁸).

#### Why 8 Bits?
Historically, early computers experimented with 6-bit, 7-bit groups. 8 bits became the standard because:

- It's a **power of 2** (2³)
- 8 bits can represent **256 different values**, enough for a character set (letters, digits, symbols)
- It divides cleanly into larger power-of-2 sizes (16, 32, 64)
- The `ASCII` character encoding needed 7 bits, so 8 bits (1 byte) gave a clean container with room to spare.

## PART 3: How Binary Numbers Work?
The **binary number system** (base-2) is the language used by all modern computers and digital devices. Unlike the decimal system (base-10) which uses ten digits (0–9), binary only uses two digits: **0 and 1**. In digital hardware, these digits represent physical states:
- **0 means "off"** (no electricity)
- **1 means "on"** (electricity flowing)

#### Normal Number System (Decimal)
We already know decimal (base 10). Let's make the structure **explicit** so we can mirror it in binary.

The number **4 7 3** in decimal means:
```
4   7   3
|   |   |
|   |   └── 3 × 10⁰ = 3 × 1     =    3
|   └──────  7 × 10¹ = 7 × 10    =   70
└──────────  4 × 10² = 4 × 100   =  400
                                  ------
                                    473
```
Each position is a **power of 10**. We move left → power increases.

#### Binary Number System (Base 2)
Same idea, but each position is a **power of 2** instead of 10.
```java title:power-of-2.java
Powers of 2:
2⁰ = 1
2¹ = 2
2² = 4
2³ = 8
2⁴ = 16
2⁵ = 32
2⁶ = 64
2⁷ = 128
```

Let's read the binary number **1 0 1 1 0 1 1 0**
```
Posit:7    6    5    4    3    2    1    0
Bit:  1    0    1    1    0    1    1    0
      |    |    |    |    |    |    |    |
      |    |    |    |    |    |    |    └─ 0 × 2⁰ = 0 × 1   =   0
      |    |    |    |    |    |    └─────  1 × 2¹ = 1 × 2   =   2
      |    |    |    |    |    └─────────   1 × 2² = 1 × 4   =   4
      |    |    |    |    └─────────────    0 × 2³ = 0 × 8   =   0
      |    |    |    └─────────────────     1 × 2⁴ = 1 × 16  =  16
      |    |    └─────────────────────      1 × 2⁵ = 1 × 32  =  32
      |    └─────────────────────────       0 × 2⁶ = 0 × 64  =   0
      └─────────────────────────────        1 × 2⁷ = 1 × 128 = 128
                                            -----                                                               182
```
So binary `10110110` = decimal `182`

## PART 4: How Many Values Can N Bits Represent?
With **n bits**, you can represent **2ⁿ different values** (from 0 to 2ⁿ - 1).
```java title:n_bits.java
1 bit  → 2¹ =          2 values  (0 or 1)
2 bits → 2² =          4 values  (0, 1, 2, 3)
3 bits → 2³ =          8 values  (0 to 7)
4 bits → 2⁴ =         16 values  (0 to 15)
8 bits → 2⁸ =        256 values  (0 to 255)    ← 1 byte
16 bits→ 2¹⁶=      65,536 values  (0 to 65,535)
32 bits→ 2³²= 4,294,967,296 values (0 to ~4.29 billion)
64 bits→ 2⁶⁴= 18,446,744,073,709,551,616 values (enormous)
```

#### Why This Matters in Java
```java title:why_matters.java
byte  →  8 bits  → 2⁸  = 256 values  → range: -128 to 127
short → 16 bits  → 2¹⁶ = 65,536      → range: -32,768 to 32,767
int   → 32 bits  → 2³² = ~4.29B      → range: -2,147,483,648 to 2,147,483,647
long  → 64 bits  → 2⁶⁴ = enormous    → range: -9.2 quintillion to 9.2 quintillion
```
The ranges are split into **negative and positive**, that's because of **signed representation**.

## PART 5: Converting Between Decimal and Binary
#### Decimal → Binary (Division Method)
Convert **182** to binary:
```java title:182.java
182 ÷ 2 = 91  remainder 0   ← LSB (Least Significant Bit, rightmost)
 91 ÷ 2 = 45  remainder 1
 45 ÷ 2 = 22  remainder 1
 22 ÷ 2 = 11  remainder 0
 11 ÷ 2 =  5  remainder 1
  5 ÷ 2 =  2  remainder 1
  2 ÷ 2 =  1  remainder 0
  1 ÷ 2 =  0  remainder 1   ← MSB (Most Significant Bit, leftmost)

Read remainders bottom to top: 1 0 1 1 0 1 1 0
```
Result: **182 = 10110110 in binary** ✓ (matches what we calculated earlier)

#### Binary → Decimal (Sum of Powers Method)
We already did this above. For quick mental math, remember powers of 2:
```
128  64  32  16  8  4  2  1
```

Example: `00001101`
```
0×128 + 0×64 + 0×32 + 0×16 + 1×8 + 1×4 + 0×2 + 1×1
= 0 + 0 + 0 + 0 + 8 + 4 + 0 + 1
= 13
```

## PART 6: Hexadecimal (The Shorthand for Binary)
Binary is verbose. Writing `10110110` for 182 is tedious.
**Hexadecimal (hex, base 16)** is the shorthand programmers use because **every 4 bits maps to exactly 1 hex digit.**
Hex uses digits: **0-9 and A-F**

```java title:hexa.java
Decimal | Binary | Hex
--------|--------|----
   0    |  0000  |  0
   1    |  0001  |  1
   2    |  0010  |  2
   3    |  0011  |  3
   4    |  0100  |  4
   5    |  0101  |  5
   6    |  0110  |  6
   7    |  0111  |  7
   8    |  1000  |  8
   9    |  1001  |  9
  10    |  1010  |  A
  11    |  1011  |  B
  12    |  1100  |  C
  13    |  1101  |  D
  14    |  1110  |  E
  15    |  1111  |  F
```

#### Converting Binary to Hex
Split binary into groups of 4 from the right, convert each group:

```java title:con_bina-hex.java
Binary: 1011 0110
           B    6

Hex: B6
```
So `10110110` binary = `B6` hex = `182` decimal.

In code, hex is written with `0x` prefix:
```java title:0x.java
int value = 0xB6;  // = 182 in decimal
System.out.println(value); // prints 182
```

#### Why Hex is Used in Programming
It allows developers to compress long binary strings into shorter, easy-to-read, and error-resistant codes.
```java title:hex.java
// Memory addresses
0x00007FFE4A3B1C20   // actual memory address you'd see in a debugger

// Colors (RGB)
0xFF5733             // FF=red(255), 57=green(87), 33=blue(51)

// Bitmasks
int mask = 0xFF;     // = 11111111 in binary — isolates the last 8 bits

// UUID
"550e8400-e29b-41d4-a716-446655440000"  // hex digits

// Error codes
0x80000000           // common error flag pattern
```

## PART 7: Signed vs Unsigned Numbers

#### The Problem: Representing Negative Numbers
So far we've only talked about **positive** numbers (0 and up). But computers need to represent negative numbers too.
**The solution Java uses:** Two's Complement.

#### Two's Complement (How Java Stores Negative Numbers)
For a **signed** number, the **leftmost bit (MSB)** is the **sign bit:**
```java title:bool.java
0 → positive
1 → negative
```

For an 8-bit signed byte:
```java title:8bit.java
0 1 1 1 1 1 1 1  = +127   (largest positive)
0 0 0 0 0 0 0 1  = +1
0 0 0 0 0 0 0 0  =  0
1 1 1 1 1 1 1 1  = -1
1 0 0 0 0 0 0 1  = -127
1 0 0 0 0 0 0 0  = -128   (largest negative, no positive equivalent)
```

#### How to Calculate Two's Complement
To find the binary representation of a negative number **-n**:
- **Step 1:** Write the binary of the positive version `n`  
- **Step 2:** Flip all bits (0→1, 1→0) — this is called "one's complement"  
- **Step 3:** Add 1

Example: What is **-6** in 8-bit binary?
```java title:exp.java
Step 1: +6 in binary     →  0 0 0 0 0 1 1 0
Step 2: Flip all bits    →  1 1 1 1 1 0 0 1
Step 3: Add 1            →  1 1 1 1 1 0 1 0
                                            ^
Result: -6 = 11111010
```

**Verify:** Does `00000110 + 11111010 = 0`?
```java title:verify.java
  0 0 0 0 0 1 1 0   (+6)
+ 1 1 1 1 1 0 1 0   (-6)
─────────────────
1 0 0 0 0 0 0 0 0   (the carry out of bit 7 is discarded, leaving 00000000 = 0) ✓
```

#### Why Two's Complement? (Not the Only Option)
Computers could use other methods (sign-magnitude, one's complement), but two's complement wins because:

- **Only one representation of zero** (no +0 and -0 confusion)
- **Addition and subtraction use the same circuit** (no special negative number hardware needed)
- **All major languages including Java use it**

### Java Integer Overflow
Now you understand why this happens:
```java title:integer.java
int max = Integer.MAX_VALUE;  // 2,147,483,647 = 01111111 11111111 11111111 11111111
System.out.println(max);      // 2147483647

int overflow = max + 1;       // 10000000 00000000 00000000 00000000
System.out.println(overflow); // -2147483648  ← wraps around to most negative!
```

The addition flipped the sign bit from 0 to 1 → suddenly it's negative. This is **integer overflow**, a real bug in production systems.

Real-world example: The [Ariane 5 rocket explosion](https://en.wikipedia.org/wiki/Ariane_flight_V88) (1996) was caused by an integer overflow. $500 million rocket destroyed.

## PART 8: Word Size
A **word** is the **natural unit of data** that a CPU processes in one operation.
The **word size** is determined by the CPU architecture:
```java title:word_size.java
8-bit  CPU  → word = 1 byte  (very old, Arduino-class microcontrollers)
16-bit CPU  → word = 2 bytes (old, 286/386 era)
32-bit CPU  → word = 4 bytes (Pentium era, still common in embedded)
64-bit CPU  → word = 8 bytes (all modern laptops, servers, phones)
```

Your computer right now is almost certainly **64-bit**. Every server you deploy to in production is **64-bit**.

#### What "64-bit" Actually Means
When we say a CPU is 64-bit, it means:

**1. Registers are 64 bits wide**
```java title:64bits.java
CPU Register (like a variable inside the CPU itself):

64-bit register can hold:
[63][62][61]...[2][1][0]   ← 64 bit positions
 ↑                   ↑
 MSB                LSB

Maximum value: 2⁶⁴ - 1 = 18,446,744,073,709,551,615
```

**2. Memory addresses are up to 64 bits**
```java title:memory-adress.java
32-bit address space: 2³² bytes = 4 GB maximum RAM
64-bit address space: 2⁶⁴ bytes = 16 exabytes (theoretical)
                                  (actual CPUs use 48 bits = 256 TB)

This is why 32-bit systems couldn't use more than 4GB of RAM!
```

**3. The CPU can process 64 bits in one instruction**
```java title:cpu_process.java
One CPU operation on 64-bit machine:

Can add two 64-bit numbers in one clock cycle
Can read/write 8 bytes (64 bits) from memory in one operation
```

#### How Word Size Affects Java
```java title:word_size
// On a 64-bit JVM:
int  i = 42;    // 32 bits, but stored in a 64-bit register (wastes 32 bits)
long l = 42L;   // 64 bits, fits perfectly in a 64-bit register

// Object references (pointers) on 64-bit JVM:
// Normal: 8 bytes (64 bits) per reference
// With CompressedOops (default): 4 bytes (32 bits) per reference
//   → The JVM uses a trick to compress heap pointers when heap < 32GB
//   → This saves significant memory
//   → -XX:+UseCompressedOops (enabled by default)
```

#### Memory Alignment and Word Boundaries
CPUs read memory most efficiently when data is aligned to word boundaries.
```java title:memory_word.java
Word size = 8 bytes (64-bit system)
Memory addresses: 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10...

ALIGNED (efficient):
Address 0: [byte0][byte1][byte2][byte3][byte4][byte5][byte6][byte7]
           ←────────────── 8 bytes = 1 word ──────────────────────→
           CPU reads this in ONE memory operation ✓

UNALIGNED (inefficient):
A long starting at address 3:
Addr 3: [byte0][byte1][byte2][byte3][byte4][byte5][byte6][byte7]
        ─────────────────────────────────────────────────────
CPU must read TWO words and stitch them together = SLOWER ✗
```

**Java (JVM) handles alignment automatically.** Object fields are padded and reordered so they align to word boundaries. You don't manually manage this, but it affects memory usage.

## PART 9: Common Byte Sizes (The Data Size Stack)
The data size stack organizes digital information measurements hierarchically, beginning from the fundamental binary unit up to massive, enterprise-scale storage capacities.

```java title:data_size.java
1 bit       → 0 or 1
4 bits      → 1 nibble (half a byte, 1 hex digit)
8 bits      → 1 byte
              → can store: 1 ASCII character, a small number (0-255)

1,024 bytes        = 1 Kilobyte  (KB)  = 2¹⁰ bytes
1,048,576 bytes    = 1 Megabyte  (MB)  = 2²⁰ bytes
1,073,741,824 bytes= 1 Gigabyte  (GB)  = 2³⁰ bytes
                   = 1 Terabyte  (TB)  = 2⁴⁰ bytes
                   = 1 Petabyte  (PB)  = 2⁵⁰ bytes
                   = 1 Exabyte   (EB)  = 2⁶⁰ bytes
```

**Note:** Hard drive manufacturers use **decimal** (1 KB = 1000 bytes), operating systems use **binary** (1 KB = 1024 bytes). This is why a "1 TB" hard drive shows as ~931 GB in Windows/Mac.

### Real-World Reference Points
```java title:ref.java
1 byte       → 1 character ('A', '5', '@')
100 bytes    → a tweet (with room to spare)
1 KB         → a short text email
100 KB       → a small image
1 MB         → a book (~1 million characters), a medium-quality photo
10 MB        → a high-quality photo
1 GB         → ~1,000 books, a movie at medium quality
16 GB        → typical laptop RAM today
1 TB         → typical hard drive today
```

## PART 10: Java Primitive Types Through the Lens of Bits
Now everything clicks together:
```java title:primitive_types.java
// INTEGERS (whole numbers — stored as Two's Complement)
byte    // 8 bits  → -128                     to  127
short   // 16 bits → -32,768                  to  32,767
int     // 32 bits → -2,147,483,648           to  2,147,483,647
long    // 64 bits → -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807

// FLOATING POINT (IEEE 754 standard)
float   // 32 bits → ~7 decimal digits of precision
double  // 64 bits → ~15-16 decimal digits of precision

// OTHER
char    // 16 bits → 0 to 65,535 (UNSIGNED, stores Unicode code point)
boolean // 1 bit logically, but JVM uses 1 byte (8 bits) in practice
        // JVM spec doesn't define the exact size
```