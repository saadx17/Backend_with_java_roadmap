# Instruction Cycle (Fetch, Decode, Execute)

> **Phase 0 — Computer Science Fundamentals → 0.1 How Computers Work**
> Goal: Understand the fundamental loop a CPU repeats billions of times per second to run *any* program — the heartbeat of computation.

---

## 0. The One-Sentence Summary

The **instruction cycle** (a.k.a. **fetch–decode–execute cycle** or **machine cycle**) is the continuous loop a CPU performs: **fetch** the next instruction from memory, **decode** what it means, **execute** it, then (often) **store** the result — and repeat, forever, until the program ends or the machine halts.

---

## 1. The Core Loop

```
        +-----------------------------------------+
        |                                         |
        v                                         |
   +---------+    +---------+    +---------+    +---------+
   | FETCH   | -> | DECODE  | -> | EXECUTE | -> | STORE   |
   +---------+    +---------+    +---------+    +---------+
   get next       figure out     perform the    write the
   instruction    what it means  operation      result back
        ^                                         |
        |                                         |
        +-----------------------------------------+
              (PC advances; loop repeats)
```

This loop runs **continuously** from the moment the computer boots until it shuts down. At 3 GHz, the CPU is churning through this cycle billions of times per second.

---

## 2. The Players (recap from CPU architecture note)

| Component | Role in the cycle |
|-----------|-------------------|
| **Program Counter (PC)** | Holds the **address of the next instruction** |
| **Instruction Register (IR)** | Holds the **instruction currently** being processed |
| **Control Unit (CU)** | Decodes the instruction and orchestrates the steps |
| **ALU** | Performs the actual computation in the execute step |
| **Registers** | Hold operands and results |
| **Memory (RAM) / Cache** | Where instructions and data are read from / written to |

---

## 3. Step 1 — FETCH

**Goal:** retrieve the next instruction from memory into the CPU.

```
1. The PC holds the address of the next instruction (e.g., 0x1000).
2. The CPU reads the instruction at that memory address
   (via cache if possible, else RAM).
3. The fetched instruction is placed into the Instruction Register (IR).
4. The PC is incremented to point to the FOLLOWING instruction.
```

> The PC increment is what makes a program run **sequentially** by default. **Jumps/branches** (loops, `if`, method calls) work by *overwriting* the PC with a different address instead of just incrementing it.

---

## 4. Step 2 — DECODE

**Goal:** figure out what the fetched instruction actually means.

```
1. The Control Unit reads the instruction in the IR.
2. It splits it into:
     - OPCODE   : what operation to perform (e.g., ADD, LOAD, JUMP)
     - OPERANDS : what to operate on (registers, memory addresses, constants)
3. The CU generates the control signals to route data and tell the ALU
   (or memory unit) what to do.
```

Example machine instruction (conceptual):
```
   ADD  R1, R2, R3       ; opcode=ADD, operands: R1 = R2 + R3
   |    |   |   |
 opcode dst src1 src2
```

---

## 5. Step 3 — EXECUTE

**Goal:** actually perform the operation.

Depending on the opcode, the execute step does different things:
| Instruction type | What execute does |
|------------------|-------------------|
| **Arithmetic/Logic** | ALU computes (e.g., add R2+R3), sets status flags |
| **Load** | Reads data from memory into a register |
| **Store** | Writes a register's value to memory |
| **Branch/Jump** | Updates the PC to a new address (changes control flow) |
| **Compare** | ALU subtracts and sets flags (no result stored) |

---

## 6. Step 4 — STORE / WRITE-BACK

**Goal:** save the result so it's available later.

- The result of the execute step is written back to a **register** or to **memory**.
- Status **flags** are updated (zero, carry, sign, overflow) for the next conditional branch.

> Some textbooks fold STORE into EXECUTE, giving the classic **three-stage** "fetch–decode–execute." Others list it separately (and add a **memory access** stage), giving a **five-stage** pipeline: *Fetch → Decode → Execute → Memory → Write-back*. Both describe the same idea.

---

## 7. Worked Micro-Example

Suppose we want to compute `c = a + b`. In machine terms (simplified):

```
Memory: a=5 at addr 100, b=3 at addr 104, c at addr 108

Instruction 1: LOAD R1, [100]   ; R1 = a (5)
   Fetch: read LOAD from PC; Decode: opcode=LOAD; Execute: read addr100;
   Store: R1 = 5

Instruction 2: LOAD R2, [104]   ; R2 = b (3)
   ... R2 = 3

Instruction 3: ADD  R3, R1, R2  ; R3 = R1 + R2
   Execute: ALU computes 5+3=8; Store: R3 = 8 (Zero flag = 0)

Instruction 4: STORE [108], R3  ; c = R3
   Execute/Memory: write 8 to addr 108
```

Four instructions, each going through the full cycle — that's how a single line of high-level code (`c = a + b;`) becomes hardware action.

---

## 8. Interrupts: Breaking the Cycle

The cycle isn't purely sequential — it can be interrupted. After (or during) an instruction, the CPU **checks for interrupts**:

```
... Execute -> Store -> CHECK for interrupt?
                          |
                  yes --> save current state (PC, registers)
                          jump to Interrupt Service Routine (ISR)
                          handle the event (e.g., key press, I/O done, timer)
                          restore state, resume the cycle
                          |
                  no  --> continue to next fetch
```

This is the hardware basis for:
- **Multitasking / context switching** (timer interrupt → scheduler).
- **I/O completion** (device interrupt → handle data; see Phase 0 I/O note).
- **Exceptions/traps** (e.g., divide-by-zero, page fault → Phase 0 virtual memory note).

---

## 9. Pipelining: Overlapping Cycles (performance)

A naive CPU finishes one instruction's whole cycle before starting the next — wasteful, since each stage uses different hardware. **Pipelining** overlaps stages like an assembly line:

```
Without pipelining (one at a time):
  I1: F D E W
  I2:         F D E W
  I3:                 F D E W

With pipelining (overlapped):
  I1: F D E W
  I2:   F D E W
  I3:     F D E W
  I4:       F D E W
```
Result: ideally **one instruction completes every cycle**, dramatically increasing throughput.

**Pipeline hazards** (why it's not perfect):
- **Data hazard:** an instruction needs a result not yet ready.
- **Control hazard:** a branch changes the PC, invalidating fetched instructions (**branch misprediction** → pipeline flush).
- **Structural hazard:** two instructions need the same hardware unit.

> This connects to **branch prediction** and **speculative execution** from the CPU note — they exist to keep the pipeline full despite branches.

---

## 10. How This Maps to Java / the JVM

- **Bytecode interpretation:** the JVM's interpreter runs its *own* fetch–decode–execute loop over **Java bytecode** (each opcode like `iadd`, `iload`, `invokevirtual` is fetched, decoded, executed) — a software echo of the hardware cycle.
- **JIT compilation:** hot bytecode is compiled to **native machine instructions** that run directly through the *hardware* instruction cycle — far faster than interpreting (Phase 1.11).
- **Branches & loops:** Java `if`/`for`/`while` compile to branch instructions; tight, predictable loops pipeline well (helps explain micro-benchmark behavior, Phase 13).
- **Interrupts → threads:** the timer-interrupt mechanism is what enables the OS to preempt Java threads (context switching note).

---

## 11. Why This Matters for a Backend (Java) Engineer

- Demystifies what "running code" actually *is* at the lowest level.
- Explains the JVM's **interpret-then-JIT** model and why startup is interpreted but steady-state is fast.
- Grounds intuition about **branch-heavy vs branch-light** code and why predictable code is faster.
- Connects interrupts to **concurrency, I/O, and exceptions** — recurring backend concerns.

---

## 12. Quick Self-Check Questions

1. What are the stages of the instruction cycle? What does each accomplish?
2. What does the Program Counter do, and how do branches change normal flow?
3. In the decode step, what are an opcode and operands?
4. Give an example of what the execute step does for an arithmetic vs a branch instruction.
5. How do interrupts alter the cycle, and name two things they enable.
6. What is pipelining, and what is a pipeline hazard?
7. How does the JVM's bytecode interpreter relate to the hardware instruction cycle?
8. Why does JIT-compiled code run faster than interpreted bytecode?

---

## 13. Key Terms Glossary

- **Instruction cycle / machine cycle:** the fetch–decode–execute(–store) loop.
- **Fetch:** retrieve the next instruction from memory into the IR.
- **Decode:** interpret the opcode and operands.
- **Execute:** perform the operation (ALU/memory/branch).
- **Store / write-back:** save the result to a register or memory.
- **Opcode:** the part of an instruction specifying the operation.
- **Operand:** the data/registers/addresses an instruction acts on.
- **Program Counter (PC):** address of the next instruction.
- **Instruction Register (IR):** holds the current instruction.
- **Interrupt:** signal that diverts the CPU to handle an event.
- **ISR (Interrupt Service Routine):** code that handles an interrupt.
- **Pipelining:** overlapping instruction stages for throughput.
- **Pipeline hazard:** data/control/structural conflict that stalls the pipeline.
- **Branch misprediction:** wrong branch guess → pipeline flush.
- **Bytecode:** JVM instructions interpreted/JIT-compiled by the JVM.

---

*Previous topic: **CPU architecture basics (ALU, registers, cache, clock cycles)**.*
*Next topic in roadmap: **RAM vs ROM vs Storage**.*
