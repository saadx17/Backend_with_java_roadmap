CPU architecture is the fundamental design that determines how a processor receives instructions, processes data, and interacts with other hardware. It acts as the "brain" of your computer, translating software commands into the physical operations that power your device.

Inside every CPU, several specialized parts work together in a continuous **fetch-decode-execute** cycle:
- Arithmetic Logic Unit (ALU)
- Control Unit (CU) 
- Registers
- Cache
- Clock

#### WHY Does a Java Backend Engineer Need to Know This?
Before anything, the reason:
- **JVM performance**: the JVM is a program running on the CPU. Understanding the CPU explains WHY the JVM makes the design choices it does.

- **Cache misses**: the single biggest performance killer in real applications. Bad data structures cause cache misses. You need to know why `ArrayList` is faster than `LinkedList` for iteration even though both are O(n).

- **Thread synchronization cost**: why `synchronized` and `volatile` are expensive (cache coherence protocols).

- **JIT compiler optimizations**: loop unrolling, inlining, branch prediction hints, the JIT does these because of CPU architecture.

- **False sharing**: a concurrency bug where two threads slow each other down by writing to different variables that happen to be on the same cache line.

- **Why integer operations are faster than floating point**: different ALU units.

- **Clock cycles**: why some operations cost more than others (why a database disk read is catastrophically slower than a memory read).


# CPU
CPU stands for ==**Central Processing Unit**==. It is the primary electronic circuitry that acts as the "brain" of a computer. It fetches, decodes, and executes program instructions, handling everything from basic calculations to running your operating system and apps.

Its job, in the simplest terms:
```
1. READ an instruction from memory
2. DECODE what the instruction means
3. EXECUTE the instruction
4. STORE the result
5. Repeat, billions of times per second
```

Everything else, caches, registers, pipelines, branch predictors exists to make this loop **faster.**
Here is the overall architecture we will build up piece by piece:
```
┌─────────────────────────────────────────────────────────────┐
│                         CPU CHIP                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    CPU CORE                         │   │
│   │                                                     │   │
│   │   ┌──────────────┐    ┌──────────────────────────┐  │   │
│   │   │   REGISTERS  │    │          ALU             │  │   │
│   │   │  (fastest    │◄──►│  (does the actual math   │  │   │
│   │   │   storage)   │    │   and logic operations)  │  │   │
│   │   └──────────────┘    └──────────────────────────┘  │   │
│   │           │                        │                │   │
│   │   ┌───────▼────────────────────────▼──────────────┐ │   │
│   │   │           CONTROL UNIT                        │ │   │
│   │   │   (fetches, decodes, orchestrates everything) │ │   │
│   │   └───────────────────────┬───────────────────────┘ │   │
│   │                           │                         │   │
│   │   ┌───────────────────────▼───────────────────────┐ │   │
│   │   │              L1 CACHE (~32 KB)                │ │   │
│   │   │         (fastest memory, on-core)             │ │   │
│   │   └───────────────────────┬───────────────────────┘ │   │
│   │                           │                         │   │
│   │   ┌───────────────────────▼───────────────────────┐ │   │
│   │   │              L2 CACHE (~256 KB)               │ │   │
│   │   │         (fast memory, on-core)                │ │   │
│   │   └───────────────────────┬───────────────────────┘ │   │
│   └───────────────────────────┼─────────────────────────┘   │
│                               │                             │
│   ┌───────────────────────────▼───────────────────────────┐ │
│   │                L3 CACHE (~8-32 MB)                    │ │
│   │           (shared across all cores)                   │ │
│   └───────────────────────────┬───────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────┘
                                │
              ┌─────────────────▼──────────────────┐
              │              RAM                   │
              │    (8-64 GB typical, slow)         │
              └─────────────────┬──────────────────┘
                                │
              ┌─────────────────▼──────────────────┐
              │         STORAGE (SSD/HDD)          │
              │    (256 GB - 2 TB, very slow)      │
              └────────────────────────────────────┘
```

## PART 1: Registers
Registers are **storage locations built directly into the CPU core itself.**
They are:
- **The fastest storage that exists** (sub-nanosecond access, essentially 0 cycles)
- **Extremely small** (a modern CPU has maybe 16-32 general-purpose registers)
- **Each register holds one word** (64 bits on a 64-bit CPU)

Think of registers as the CPU's **working hands.** Before the CPU can do ANY math or logic on data, that data must first be loaded into a register.
```
You cannot do: RAM[address1] + RAM[address2]
You must do:
  LOAD RAM[address1] → Register1
  LOAD RAM[address2] → Register2
  ADD  Register1, Register2 → Register3
  STORE Register3 → RAM[address3]
```

### Types of Registers
CPU registers are ==tiny, ultra-fast memory storage locations inside the processor==. They hold data, memory addresses, and instructions currently being processed by the CPU to ensure operations happen as quickly as possible.

#### 1. General-Purpose Registers (GPRs)
Used to hold data being worked on. In x86-64 (the architecture of most servers/laptops):
```
64-bit name | 32-bit | 16-bit | 8-bit | Purpose (by convention)
──────────────────────────────────────────────────────────────────
RAX         | EAX    | AX     | AL    | Accumulator, return values
RBX         | EBX    | BX     | BL    | Base register
RCX         | ECX    | CX     | CL    | Counter (loops)
RDX         | EDX    | DX     | DL    | Data register
RSI         | ESI    | SI     | SIL   | Source index
RDI         | EDI    | DI     | DIL   | Destination index
RSP         | ESP    | SP     | SPL   | Stack pointer ← critical
RBP         | EBP    | BP     | BPL   | Base pointer (stack frame)
R8 - R15    |        |        |       | Additional registers
```
The naming is historical, `EAX` is the lower 32 bits of `RAX`.  
The JVM uses these registers internally when executing your Java [[Bytecode & JIT#What is ==Bytecode==?|bytecodes]].

#### 2. Program Counter (PC) / Instruction Pointer (RIP)

```
PC = address of the NEXT instruction to execute

After each instruction, PC automatically increments to the next instruction.
Jump/branch instructions modify PC to a different address.
When you call a method, PC jumps to that method's address.
When the method returns, PC goes back to where it was.
```
In the JVM, there is a **per-thread PC register**; this is why each thread can be at a different point of execution simultaneously.

#### 3. Stack Pointer (RSP)
```
Points to the top of the current stack frame.
Every method call:
  - Pushes a new stack frame (local variables, return address)
  - Decrements RSP
Every method return:
  - Pops the stack frame
  - Increments RSP
  - Restores PC to return address

Stack grows DOWNWARD in memory (high address to low address)

High addresses
│  main() frame        │ ← RSP when main starts
│  method1() frame     │ ← RSP after calling method1
│  method2() frame     │ ← RSP after calling method2
│  ...                 │
Low addresses
```
This is why you get `StackOverflowError` from deep recursion — you run out of stack space because RSP keeps decrementing.

#### 4. Flags Register (RFLAGS)
```
A special register where each bit is a flag:

ZF (Zero Flag)     → set to 1 if last result was zero
SF (Sign Flag)     → set to 1 if last result was negative
CF (Carry Flag)    → set to 1 if last operation had a carry out
OF (Overflow Flag) → set to 1 if last operation overflowed
PF (Parity Flag)   → set based on parity of result
```

These flags are how **comparison and branching** work:
```
if (a > b) {
    // do something
}
```

Compiles to roughly:
```
CMP  RAX, RBX      ; subtract RBX from RAX, don't store result, just set flags
JLE  else_branch   ; Jump if Less or Equal (check SF and ZF flags)
; ... "do something" code here
else_branch:
; ... else code here
```

#### 5. Floating Point Registers (XMM/YMM/ZMM)
Separate registers for floating point and SIMD operations:
```
XMM0 - XMM15   → 128-bit registers (used for float, double, and SSE)
YMM0 - YMM15   → 256-bit registers (AVX instructions)
ZMM0 - ZMM31   → 512-bit registers (AVX-512)

SIMD = Single Instruction, Multiple Data
Example: Add 4 floats simultaneously in one instruction
```
This is why `float` and `double` operations use different hardware than `int` and `long`.

### How Java Uses Registers
**Java does not use physical CPU registers at the language level because it is designed around a stack-based architecture.** Instead of forcing developers or the Java compiler to manage fixed hardware registers, Java targets an abstract computing machine called the **Java Virtual Machine** ([[JDK, JRE & JVM#What is JVM?|JVM]]).
```java title:syntax.java
public int add(int a, int b) {
    return a + b;
}
```

The JVM [[Bytecode & JIT#What is ==JIT==?|JIT]] compiler compiles this to something like:
```
; a is in EDI (first argument)
; b is in ESI (second argument)
ADD  EDI, ESI      ; EDI = EDI + ESI  (one instruction!)
MOV  EAX, EDI      ; return value goes in EAX
RET                ; return
```
Your 3-line Java method becomes 3 assembly instructions. The JIT is that good.

### Register Limits and Java Performance
Because there are **only ~16 general-purpose registers**, and your Java method might have 30 local variables, the JIT compiler must decide which variables get to live in registers and which get "spilled" to memory (the stack).
```java title:void_locals.java
// This method has too many local variables
// Some will be spilled to the stack — slower
public void manyLocals() {
    int a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p, q, r;
    // ... JIT cannot keep all in registers simultaneously
}
```
This is one reason **methods should be small**, not just for readability, but for JIT register allocation efficiency.

## PART 2: The ALU
An **ALU (Arithmetic Logic Unit)** is the fundamental building block of a CPU’s processing core. Often referred to as the "mathematical brain" of the computer, it executes all mathematical calculations and logical decisions required by the system's software.

Every mathematical and logical operation your Java code does eventually becomes an ALU operation.
```
          ┌─────────────────────────────┐
Input A ─►│                             │
          │           ALU               ├──► Result
Input B ─►│                             │
          │                             ├─► Flags (ZF, CF, OF, SF)
Oprtion ─►│ (what operation to do)      │
          └─────────────────────────────┘
```

### What Operations the ALU Performs?
It processes all data and instructions by executing four main categories of operations:
1. Arithmetic Operations
2. Logical Operations
3. Shift Operations
4. Comparison Operations

#### Arithmetic Operations
```
ADD   a + b          → integer addition
SUB   a - b          → integer subtraction
MUL   a × b          → integer multiplication
DIV   a ÷ b          → integer division (quotient)
MOD   a mod b        → remainder
INC   a + 1          → increment (optimized single operand)
DEC   a - 1          → decrement
NEG   -a             → negate (two's complement)
```

#### Logical Operations
```
AND   a & b          → bitwise AND
OR    a | b          → bitwise OR
XOR   a ^ b          → bitwise XOR
NOT   ~a             → bitwise NOT
```

#### Shift Operations
```
SHL   a << n         → shift left (multiply by 2ⁿ)
SHR   a >> n         → arithmetic shift right (divide by 2ⁿ, signed)
USHR  a >>> n        → logical shift right (unsigned, fills with 0)
```

#### Comparison Operations
```
CMP   a - b          → subtract but only set flags, don't store result
TEST  a & b          → AND but only set flags, don't store result
```

### Modern CPUs Have MULTIPLE ALUs
A modern CPU core doesn't have just one ALU. It has several specialized execution units:
```
┌────────────────────────────────────────────────────────┐
│                     CPU CORE                           │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐   │
│  │  Integer    │  │  Integer    │  │    Branch     │   │
│  │    ALU      │  │    ALU      │  │     Unit      │   │
│  │  (add, sub) │  │  (mul, div) │  │  (jumps, if)  │   │
│  └─────────────┘  └─────────────┘  └───────────────┘   │
│                                                        │
│  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │   Float/SIMD    │  │    Load/Store Unit          │  │
│  │      ALU        │  │  (memory read/write)        │  │
│  │ (float, double) │  │                             │  │
│  └─────────────────┘  └─────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```
This means the CPU can potentially execute **multiple operations at the same time** (called **superscalar execution**).

### Operation Latency (How Many Cycles Each Takes)
Not all ALU operations are equal in speed:
```
Operation          | Latency (clock cycles)
───────────────────┼────────────────────────
Integer ADD        | 1 cycle
Integer SUB        | 1 cycle
Integer MUL        | 3 cycles
Integer DIV        | 20-90 cycles (very slow!)
Shift (<<, >>, >>>)| 1 cycle
Bitwise AND, OR,XOR| 1 cycle
Float ADD          | 3-5 cycles
Float MUL          | 3-5 cycles
Float DIV          | 10-15 cycles
```

**This is why:**
```java title:latency.java
// Multiplying by a power of 2 — use shifts (1 cycle) not multiplication (3 cycles)
int result = x * 8;     // 3 cycles for MUL
int result = x << 3;    // 1 cycle for SHL  ← JIT does this automatically

// Division is SLOW
int result = x / 7;     // ~40 cycles ← JIT may replace with multiply+shift trick
```
The JIT compiler knows all these latencies and **automatically rewrites your code** to use faster operations when it can.

### The FPU (Floating-Point Unit)
Floating-point operations (on `float` and `double`) go to a **separate unit** from integer operations.
```java title:FPU.java
int a = 5 + 3;        // Goes to integer ALU — 1 cycle
double b = 5.0 + 3.0; // Goes to FPU — 3-5 cycles
```
This is why you sometimes see advice to use integer arithmetic in hot paths when possible.

## PART 3: Clock Cycles
Every CPU has a **crystal oscillator** that produces a regular electrical pulse — like a metronome.
```
Clock signal:
    ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
    │ │ │ │ │ │ │ │ │ │ │ │ │ │
────┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └────

Each ┌─┐ is one clock cycle
```
The CPU does one "tick" of work per clock cycle (in the simplest model).

### Clock Frequency (GHz)
```
1 Hz    = 1 cycle per second
1 kHz   = 1,000 cycles per second
1 MHz   = 1,000,000 cycles per second
1 GHz   = 1,000,000,000 cycles per second (1 billion)
3.5 GHz = 3,500,000,000 cycles per second

Modern CPUs: 3-5 GHz
```

### How Long Is One Clock Cycle?
```
At 3 GHz:

time per cycle = 1 / 3,000,000,000 = 0.000000000333 seconds
               = 0.333 nanoseconds
               ≈ 1/3 of a nanosecond

Light travels about 10 cm (4 inches) in 1/3 nanosecond.
Electricity in a wire travels slightly slower.

This means the CPU cycle time is physically limited
by the SIZE of the chip — signals can't travel farther
than ~10cm in one cycle, which is why CPUs are tiny.
```

### The Latency Table Every Developer Should Know
This is one of the most important things to internalize:
```
Operation                        | Latency        | Relative
─────────────────────────────────┼────────────────┼──────────────
CPU Register operation           | 0.3 ns         | 1x
L1 Cache hit                     | 1 ns           | 3x
L2 Cache hit                     | 3-4 ns         | 10x
L3 Cache hit                     | 10-40 ns       | 30-100x
RAM access                       | 60-100 ns      | 200x
SSD random read                  | 100,000 ns     | 300,000x
HDD random read (spinning disk)  | 10,000,000 ns  | 30,000,000x
Network round trip (same DC)     | 500,000 ns     | 1,500,000x
Network round trip (cross-world) | 150,000,000 ns | 450,000,000x
```

Let's make this concrete with a human-scale analogy:
```
If 1 CPU cycle = 1 second, then:

L1 Cache hit        = 3 seconds
L2 Cache hit        = 10 seconds
L3 Cache hit        = 30-100 seconds (1-2 minutes)
RAM access          = 6 minutes
SSD read            = 2-6 days
HDD read            = 1-12 months
Network (same DC)   = 5 days
Network (worldwide) = 5 years
```
**This is the most important performance intuition you can have.**

When your Java code accesses an object in RAM that's not in cache, you're waiting 60-100 ns. During that time, the CPU could have done **200+ arithmetic operations.**

### Cycles Per Instruction (CPI) and IPC
Modern CPUs don't just do one thing per cycle. They do **many things per cycle.**
```
CPI (Cycles Per Instruction): how many cycles per instruction (lower is better)
IPC (Instructions Per Cycle): how many instructions per cycle (higher is better)

A simple CPU:        1 IPC (1 instruction per cycle)
Modern Intel/AMD:    4-6 IPC (4-6 instructions per cycle!)

This is achieved through:
- Superscalar execution (multiple ALUs)
- Out-of-order execution (CPU reorders instructions for efficiency)
- Pipelining (overlap instruction stages)
- Branch prediction (speculatively execute before knowing branch result)
```

### CPU Pipelining
The CPU doesn't process one instruction completely, then start the next.  
It **overlaps** stages of multiple instructions:
```
Instruction stages:
1. IF = Instruction Fetch (read instruction from cache/memory)
2. ID = Instruction Decode (figure out what it means)
3. EX = Execute (ALU does the operation)
4. MEM = Memory access (read/write if needed)
5. WB = Write Back (store result to register)

Without pipelining (sequential):
Instruction 1: IF → ID → EX → MEM → WB
Instruction 2:                           IF → ID → EX → MEM → WB
Instruction 3:                                                IF → ID → EX → MEM → WB

With pipelining (overlapped):
Cycle:          1    2    3    4    5    6    7    8    9
Instruction 1:  IF   ID   EX   MEM  WB
Instruction 2:       IF   ID   EX   MEM  WB
Instruction 3:            IF   ID   EX   MEM  WB
Instruction 4:                 IF   ID   EX   MEM  WB
Instruction 5:                      IF   ID   EX   MEM  WB

5 instructions complete in 9 cycles instead of 25 cycles!
```

### Pipeline Hazards (When Pipelining Breaks Down)
##### Data Hazard
Instruction needs result of previous instruction that isn't done yet.
```
ADD R1, R2, R3    ; R1 = R2 + R3  (takes 3 cycles)
SUB R4, R1, R5    ; R4 = R1 - R5  ← needs R1, but it's not ready yet!
```
CPU must **stall** (insert bubbles/NOPs) or use **forwarding** (bypass WB stage).

##### Control Hazard
Branch instruction, CPU doesn't know which instruction comes next.
```
CMP R1, R2
JNE somewhere   ; do we fetch instructions after JNE or from 'somewhere'?
                ; we don't know until we execute CMP!
```
This leads to **branch prediction**, the CPU guesses which way the branch goes and speculatively executes. If wrong: **pipeline flush** (discard all speculative work, big performance cost).

### Branch Prediction and Java
```java title:branch_prediction.java
// PREDICTABLE loop — branch predictor gets it right every time
// CPU predicts "loop continues" and is right 999,999 times out of 1,000,000
for (int i = 0; i < 1_000_000; i++) {
    sum += array[i];
}

// UNPREDICTABLE branch — random pattern, predictor fails ~50% of time
// Each misprediction: ~15-20 cycle penalty (pipeline flush)
for (int i = 0; i < 1_000_000; i++) {
    if (random.nextBoolean()) {  // truly random — unpredictable!
        sum += array[i];
    }
}
```
**Real benchmark:** sorting an array before processing it can be **6x faster** than unsorted, purely due to branch prediction. [The famous Stack Overflow answer](https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-processing-an-unsorted-array) demonstrates this.

## PART 4: CPU Cache
A CPU cache is a small, ultra-fast hardware memory built directly into the processor. Its purpose is to store copies of frequently accessed data and instructions so the processor doesn't have to repeatedly wait on slower system RAM.

### Why Cache Exists
The fundamental problem: CPU speed doubled roughly every 2 years (Moore's Law),RAM speed improved much more slowly.

**Speed gap between CPU and RAM:**
1980: CPU and RAM similar speed
1990: CPU ~10x faster than RAM
2000: CPU ~100x faster than RAM  
2024: CPU ~200-500x faster than RAM

**Without cache, for every memory access:**
CPU would fetch data and then SIT IDLE for 200+ cycles waiting for RAM.
That's like asking your employee to do a task, 
then staring at a wall for 6 months waiting for the answer.

**Solution:** Add small, fast memory physically close to the CPU: the cache.

### Cache Hierarchy
Modern CPUs have multiple levels of cache:
```
┌──────────────────────────────────────────────────────────────┐
│   Speed: ◄──────────────────────── Fastest to Slowest ──────►│
│   Size:  ◄──────────────────────── Smallest to Largest ─────►│
│                                                              │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐           │
│  │   L1   │   │   L2   │   │   L3   │   │  RAM   │           │
│  │~32 KB  │   │~256 KB │   │~8-32 MB│   │ 8-64 GB│           │
│  │ 1-4 ns │   │ 3-10 ns│   │10-40 ns│   │60-100ns│           │
│  └────────┘   └────────┘   └────────┘   └────────┘           │
│  Per core     Per core    Shared all      External           │
│                            cores          chip               │
└──────────────────────────────────────────────────────────────┘
```

#### L1 Cache
Fastest, smallest, one per core.

- Usually split into L1i (instructions) and L1d (data)
- ~32 KB each
- 4-cycle access latency (essentially free compared to RAM)
- Holds the data the CPU is working on RIGHT NOW

#### L2 Cache
Medium, one per core.

- ~256 KB - 1 MB
- ~10-cycle access
- Holds data that was recently in L1 but got evicted

#### L3 Cache
Largest, shared across all cores.

- ~8-32 MB (varies by CPU)
- ~30-40 cycle access
- Shared between all cores on the chip (important for multi-threading!)
- Holds data that might be needed by any core

### Cache Lines: The Unit of Transfer
The cache doesn't store individual bytes. It stores **cache lines.**
```
Cache line = 64 bytes (on virtually all modern x86 CPUs)

When you access 1 byte of data:
  → The CPU loads ALL 64 bytes surrounding that address into cache
  → Because if you need byte at address X, you probably need X+1, X+2, etc. soon
  → This is called "spatial locality"

Memory:
  ┌────────────────────────────────────────────────────────────────┐
│ 0  1  2  3  4  5  6  7  8  9  10 11 ... 62 63 | 64 65 ... 127  │
└────────────────────────────────────────────────────────────────┘
  ←──────────── cache line 0 (64 bytes) ─────────────►
                                      ←─ cache line 1 ─►

If you access byte 0:
  → Entire cache line 0 (bytes 0-63) loaded into L1 cache
  → Bytes 1-63 are now "free" to access (already in cache!)
```

### Cache Hit vs Cache Miss
```
Cache HIT:  CPU needs data → data IS in cache → ~1-4 cycles ✓
Cache MISS: CPU needs data → data NOT in cache → must go to next level

L1 miss → check L2    (10 cycles)
L2 miss → check L3    (30-40 cycles)
L3 miss → go to RAM   (60-100 cycles) ← very painful
```

**Hit rate matters enormously:**
```
Scenario: 1,000,000 memory accesses

99% hit rate:
  990,000 hits   × 4 cycles   = 3,960,000 cycles
    10,000 misses × 100 cycles =   1,000,000 cycles
  Total: ~5,000,000 cycles

90% hit rate:
  900,000 hits   × 4 cycles   = 3,600,000 cycles
  100,000 misses × 100 cycles = 10,000,000 cycles
  Total: ~13,600,000 cycles  ← 2.7x SLOWER!
```

### Spatial Locality vs Temporal Locality
The cache works because programs exhibit two types of **locality:**

**Temporal Locality:** If you accessed data recently, you'll probably access it again soon.
```java title:temp_locality.java
// x is accessed repeatedly → stays in L1 cache
for (int i = 0; i < 1_000_000; i++) {
    sum += x;   // x is in register/L1, no RAM access after first
}
```

**Spatial Locality:** If you accessed address X, you'll probably access addresses near X soon.
```java title:spatial_locality.java
int[] array = new int[1_000_000];
// Array is contiguous in memory → cache lines load 16 ints at a time
// (16 ints × 4 bytes = 64 bytes = 1 cache line)
for (int i = 0; i < array.length; i++) {
    sum += array[i];   // Every 16th iteration causes a cache miss
    // The other 15 accesses are already in cache → very fast
}
```

### Cache-Friendly vs Cache-Unfriendly Code
#### Cache-Friendly: ArrayList (array-backed)
```java title:arraylist.java
List<Integer> arrayList = new ArrayList<>();
// Memory layout: [obj1][obj2][obj3][obj4][obj5][obj6][obj7][obj8]...
// All contiguous — one cache line holds multiple elements
// Iterating: extremely cache-friendly
for (Integer i : arrayList) { ... }  // FAST
```

#### Cache-Unfriendly: LinkedList (pointer-chasing)
```java title:linkedlist.java
List<Integer> linkedList = new LinkedList<>();
// Memory layout: [obj1→][   ][obj2→][   ][obj3→]...
//                      ↗         ↗         ↗
// Each node has a pointer to the next node
// Nodes are scattered RANDOMLY across memory
// Each "next" pointer dereference is likely a cache miss
for (Integer i : linkedList) { ... }  // SLOW (pointer chasing)
```
Even though both are O(n), ArrayList iteration can be **5-10x faster** due to cache behavior.

#### 2D Array Traversal: Row-Major vs Column-Major
```java title:2D_array.java
int[][] matrix = new int[1000][1000];

// Java stores 2D arrays in ROW-MAJOR order:
// [row0col0][row0col1][row0col2]...[row0col999][row1col0][row1col1]...
// Each row is contiguous in memory

// CACHE-FRIENDLY (row-major traversal):
for (int row = 0; row < 1000; row++) {
    for (int col = 0; col < 1000; col++) {
        sum += matrix[row][col];  // Sequential memory access ✓
    }
}

// CACHE-UNFRIENDLY (column-major traversal):
for (int col = 0; col < 1000; col++) {
    for (int row = 0; row < 1000; row++) {
        sum += matrix[row][col];  // Jumps 1000 ints forward each time ✗
        // Each access is in a DIFFERENT cache line
        // Every access is likely a cache miss
    }
}
// The cache-friendly version can be 10-50x faster on large matrices!
```

### False Sharing (A Critical Concurrency Bug)
This is one of the most subtle and damaging performance bugs in multithreaded Java code.
```
Two threads, two variables:
  Thread 1 writes to variable A
  Thread 2 writes to variable B

You'd think they don't interfere. But watch:

Memory layout:
  [A (8 bytes)][B (8 bytes)][...other data...]
  ←───────── 64-byte cache line ─────────→

A and B are on the SAME cache line!

What happens:
  Thread 1 (Core 1) writes to A
    → Cache line (containing A AND B) is marked "dirty" on Core 1
    → Core 2's copy of this cache line is now INVALID
    → Core 2 must reload the entire cache line from L3/RAM
  Thread 2 (Core 2) writes to B
    → Cache line is marked dirty on Core 2
    → Core 1's copy is now invalid
    → Core 1 must reload the entire cache line
  → They keep invalidating each other's cache lines!
  → This is FALSE SHARING — they're not sharing data, 
     but sharing a CACHE LINE
```

**Real false sharing example in Java:**
```java title:bad_counter.java
// FALSE SHARING - terrible performance with multiple threads
class BadCounter {
    long counter1 = 0;   // These two longs (16 bytes) 
    long counter2 = 0;   // fit in ONE 64-byte cache line
}

// Thread 1 increments counter1, Thread 2 increments counter2
// They constantly invalidate each other's cache → very slow
```

**Fixing false sharing with padding:**
```java title:good_counter.java
// PADDED - good performance
class GoodCounter {
    long counter1 = 0;
    // Pad to fill the rest of the cache line (64 bytes)
    // counter1 takes 8 bytes, so add 56 more bytes of padding:
    long p1, p2, p3, p4, p5, p6, p7;   // 7 × 8 = 56 bytes padding
    long counter2 = 0;                   // Now in a DIFFERENT cache line
    long p8, p9, p10, p11, p12, p13, p14; // More padding
}
```

Java 8+ solution using `@Contended` annotation:
```java title:better_counter.java
import sun.misc.Contended;

class BetterCounter {
    @Contended
    long counter1 = 0;  // JVM automatically adds padding around this
    
    @Contended
    long counter2 = 0;  // In its own cache line
}
// Must run with: -XX:-RestrictContended JVM flag
```

`LongAdder` in Java's standard library uses this technique internally — it's faster than `AtomicLong` under high contention because it uses multiple padded cells.

### Cache Eviction Policy
When the cache is full and needs to bring in new data, something must be removed.
Most CPUs use **LRU (Least Recently Used)** or a variant:
```
Cache has 4 slots:
State: [A][B][C][D]

Access E (cache miss):
  → Evict A (least recently used)
State: [B][C][D][E]

Access B (cache hit):
  → B is now most recently used
State: [C][D][E][B]

Access F (cache miss):
  → Evict C (least recently used)
State: [D][E][B][F]
```

This is exactly why Java's `LinkedHashMap(capacity, loadFactor, true)` with access-order mode **mirrors LRU cache behavior**, it was designed to match how CPU caches work.
```
// LRU Cache using LinkedHashMap — mirroring CPU cache eviction policy
int cacheSize = 100;
Map<Key, Value> lruCache = new LinkedHashMap<>(cacheSize, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<Key, Value> eldest) {
        return size() > cacheSize;
    }
};
```
### Multi-Core Cache Coherence (MESI Protocol)
When multiple CPU cores each have their own L1/L2 cache, **the same data could exist in multiple caches simultaneously.** If one core modifies it, the other cores' copies are stale.

The hardware solves this with the **MESI protocol:**
```
Each cache line can be in one of four states:

M = Modified   → This core has modified the data, no other cache has it
                  (must write back to RAM eventually)
E = Exclusive  → Only this cache has the data, it matches RAM
                  (can be used without notifying others)
S = Shared     → Multiple caches have this data, all match RAM
                  (read-only in this state)
I = Invalid    → This cache line's data is stale, don't use it
```

**State transitions:**
```
Initial: Core 1 and Core 2 both read variable X
  Core 1 cache line X: [Shared]
  Core 2 cache line X: [Shared]

Core 1 writes to X:
  Core 1: [Modified]  → sends "invalidate" signal to other cores
  Core 2: [Invalid]   → Core 2 is notified, marks its copy invalid

Core 2 reads X:
  Core 2 has [Invalid] → cache miss → must fetch from Core 1's [Modified] copy
  Core 1: [Shared]    → writes back to L3/RAM
  Core 2: [Shared]    → fetches from L3/RAM
```

**This is WHY `volatile` and `synchronized` have performance costs:**
```
volatile int x = 0;

// Thread 1:
x = 42;  // Must propagate to ALL other cores' caches immediately
         // → Triggers MESI protocol → cache line invalidation
         // → Other cores must reload cache line → expensive!

// Thread 2:
int val = x;  // Must see the latest value
              // → Might be a cache miss if Thread 1 recently wrote
```

Without `volatile`, Thread 2 might read a stale value from its own cache (L1 hit: fast, but wrong!). With `volatile`, the JMM (Java Memory Model) guarantees visibility but it forces cache coherence operations which are slow.

## PART 5: Putting It All Together
Let's trace exactly what happens when the JVM executes this:
```
public int sum(int[] array) {
    int total = 0;
    for (int i = 0; i < array.length; i++) {
        total += array[i];
    }
    return total;
}
```

**Step 1: JIT Compilation (first few calls)**
```
JVM interprets bytecode → slow
After ~10,000 calls, JIT compiler kicks in
JIT compiles bytecode → native machine code (x86-64 instructions)
JIT stores compiled code in code cache (also uses L1i cache)
```

**Step 2: Method Call**
```
PC (Instruction Pointer) → set to address of compiled sum() method
Stack frame pushed: [return address][total=0][i=0][array reference]
RSP (stack pointer) decremented
```

**Step 3: Loop — Registers**
```
Registers allocated:
  RAX ← array reference (pointer to array object in heap)
  RCX ← array.length    (loaded from array object header)
  RDX ← i (loop counter)
  RSI ← total (accumulator)
```

**Step 4: Memory Access — Cache**
```
First iteration (i=0):
  CPU fetches array[0] from heap memory
  array is NOT in cache → L1 miss → L2 miss → L3 miss → RAM access
  RAM sends 64-byte cache line: array[0] through array[15] (16 ints)
  Cache line stored in L1 data cache

Iterations 1-15:
  array[1] through array[15] → ALL in L1 cache → 1-4 cycle access
  Extremely fast — no memory fetches needed

Iteration 16:
  array[16] → new cache line needed → another RAM access
  Cache line: array[16] through array[31]

Pattern: 1 cache miss per 16 iterations (6.25% miss rate)
This is WHY sequential array iteration is so fast
```

**Step 5: ALU Operations**
```
Each iteration:
  LOAD: array[i] from register/cache → 1-4 cycles
  ADD:  total += array[i]            → 1 cycle
  INC:  i++                          → 1 cycle
  CMP:  i < array.length             → 1 cycle (sets flags)
  JL:   jump if less                 → 0-1 cycles (predicted correctly)

Total per iteration (cache hit): ~4-6 cycles
Total per iteration (cache miss): ~100+ cycles
```

**Step 6: JIT Optimizations**
The JIT actually transforms this loop further:
```
Loop unrolling (JIT may do 4 iterations per loop body):
  total += array[i];
  total += array[i+1];
  total += array[i+2];
  total += array[i+3];
  i += 4;
  // Reduces loop overhead (CMP, JL) by 4x

SIMD vectorization:
  With AVX2, JIT can add 8 integers simultaneously in one instruction
  256-bit YMM register holds 8 × 32-bit integers
  YMM0 = [a[0], a[1], a[2], a[3], a[4], a[5], a[6], a[7]]
  One VPADDD instruction adds 8 integers at once
  → Throughput: 8 array elements per cycle instead of 1
```
This is why **modern Java is extremely fast for numeric processing** — the JIT generates SIMD code that rivals hand-written C.

## PART 6: Multi-Core Architecture
Modern CPUs have multiple cores:
```
┌──────────────────────────────────────────────────────────────┐
│                      CPU PACKAGE (Die)                       │
│                                                              │
│  ┌────────────────────┐  ┌────────────────────┐              │
│  │      CORE 0        │  │      CORE 1        │              │
│  │  ┌──────┐ ┌─────┐  │  │  ┌──────┐ ┌─────┐  │              │
│  │  │ Regs │ │ ALU │  │  │  │ Regs │ │ ALU │  │              │
│  │  └──────┘ └─────┘  │  │  └──────┘ └─────┘  │              │
│  │  L1 Cache (32 KB)  │  │  L1 Cache (32 KB)  │              │
│  │  L2 Cache (256 KB) │  │  L2 Cache (256 KB) │              │
│  └────────────────────┘  └────────────────────┘              │
│                                                              │
│  ┌────────────────────┐  ┌────────────────────┐              │
│  │      CORE 2        │  │      CORE 3        │              │
│  │  ┌──────┐ ┌─────┐  │  │  ┌──────┐ ┌─────┐  │              │
│  │  │ Regs │ │ ALU │  │  │  │ Regs │ │ ALU │  │              │
│  │  └──────┘ └─────┘  │  │  └──────┘ └─────┘  │              │
│  │  L1 Cache (32 KB)  │  │  L1 Cache (32 KB)  │              │
│  │  L2 Cache (256 KB) │  │  L2 Cache (256 KB) │              │
│  └────────────────────┘  └────────────────────┘              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              L3 Cache (shared, 16-32 MB)             │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────┐  ┌──────────────────────────────┐   │
│  │   Memory Controller │  │      PCIe Controller         │   │
│  └─────────────────────┘  └──────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
         │                              │
    ┌────▼───┐                    ┌─────▼────┐
    │  RAM   │                    │ SSD/GPU  │
    └────────┘                    └──────────┘
```

### Hyper-Threading (SMT — Simultaneous Multi-Threading)
Each physical core often appears as **2 logical cores** to the OS:
```
Physical Core:
  Has ONE set of ALUs and cache
  Has TWO sets of registers and instruction pointers

When Thread A stalls (e.g., cache miss, waiting for memory):
  Physical core switches to Thread B's registers
  Thread B runs on the same ALU while Thread A waits
  → Better utilization of the ALU (it would be idle otherwise)

Physical cores: 4
Logical cores: 8 (with HyperThreading)

In Java: Runtime.getRuntime().availableProcessors() returns logical cores
```

### Why This Matters for Java Thread Pools
```
// Common thread pool sizing advice:
// I/O-bound tasks (database calls, HTTP requests):
int threads = Runtime.getRuntime().availableProcessors() * 2;
// Because threads will be blocked on I/O, not burning CPU

// CPU-bound tasks (complex computation, crypto):
int threads = Runtime.getRuntime().availableProcessors();
// Because more threads than cores = context switching overhead

// Actually for I/O: Virtual Threads (Java 21) are better
// They don't block real threads when waiting for I/O
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

---

## PART 7: Latency in a Spring Boot Context
Let's make this concrete for a backend engineer:
```
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

What happens at the CPU level:
```
1. HTTP request arrives (network interrupt → OS → JVM)

2. Thread wakes up (context switch: ~1-10 microseconds)
   → Thread's registers loaded from memory
   → L1/L2 cache may be cold (another thread was using this core)

3. JVM dispatches to Spring's DispatcherServlet
   → Compiled JIT code in code cache (L1i hit → fast)
   → Object references (HTTP context, request) → heap accesses

4. Spring resolves the controller method
   → ConcurrentHashMap lookup (Spring's handler map)
   → Likely L3 hit (warm, accessed frequently)

5. JPA/Hibernate executes findById(id)
   → Check Hibernate first-level cache (HashMap in memory)
   → If hit: return from RAM/L3 → ~40 ns
   → If miss: go to database

6. Database query (if cache miss):
   → JDBC: send bytes over network → milliseconds (500,000+ ns)
   → While waiting: this thread is BLOCKED
   → OS takes CPU away → another thread gets to run
   → When DB responds: OS wakes this thread

7. Object deserialization (ResultSet → User object)
   → Memory allocation in Eden space (heap)
   → Fast: TLAB allocation is just a pointer increment + check

8. Spring serializes User → JSON (Jackson ObjectMapper)
   → CPU-intensive: string building, field reflection
   → Gets JIT-compiled after enough calls

9. Response written to socket buffer → sent

Total time: 2-50 milliseconds (dominated by database/network)
```
The CPU itself completes each individual Java operation in nanoseconds. Your "slow" API is slow because of **network and I/O** — not the CPU.

## Quick Reference
```
REGISTERS
─────────────────────────────────────────────────
Fastest storage (sub-nanosecond)
~16 general-purpose registers (64-bit each)
All computation happens on register values
JVM uses them for local variables, method args, return values

ALU
─────────────────────────────────────────────────
Does actual arithmetic and logic
Multiple ALUs per core (integer, float, branch)
ADD/SUB/AND/OR/XOR: 1 cycle
MUL: 3 cycles
DIV: 20-90 cycles (avoid in hot paths!)
Modern CPUs: 4-6 instructions per cycle (superscalar)

CLOCK CYCLES
─────────────────────────────────────────────────
3-5 GHz = 3-5 billion cycles/second
1 cycle = ~0.3 nanoseconds at 3 GHz
Pipelining overlaps multiple instruction stages
Branch misprediction penalty: 15-20 cycles

CACHE HIERARCHY
─────────────────────────────────────────────────
L1: ~32 KB, 1-4 cycles,  per-core
L2: ~256 KB, 10 cycles,  per-core
L3: ~16 MB,  30-40 cycles, shared
RAM: 8-64 GB, 60-100 cycles, external
SSD: 100,000 cycles
Cache line: 64 bytes (unit of transfer)

KEY PRINCIPLES
─────────────────────────────────────────────────
Spatial locality → sequential access = fast
Temporal locality → reused data = fast
Cache misses → the #1 performance killer
False sharing → concurrency bug from shared cache lines
Branch misprediction → 15-20 cycle penalty
volatile/synchronized → force cache coherence → expensive
ArrayList > LinkedList for iteration (cache lines)
Row-major array traversal → cache friendly
```