All memory trades **speed** against **size** and **cost**. Faster memory is smaller and more expensive per byte.

![[Memory_Hierarchy.png]]

> Everything above the "persistence line" disappears when power is lost; everything below survives. This single fact drives the entire design of how programs load and save data.

## 1. Volatile vs Non-Volatile

| Term | Meaning | Examples |
|------|---------|----------|
| **Volatile** | Loses its contents when power is removed | RAM, CPU cache, registers |
| **Non-volatile** | Retains contents without power | ROM, SSD, HDD, flash, optical |

This is *the* defining distinction. It's why you must **save your work**, until saved, it only exists in volatile RAM.

> ⚠️ Note: Java's `volatile` keyword is a **completely different concept** (it's about thread visibility, Phase 1.10). Don't confuse hardware volatility with the Java keyword.

## 2. RAM (Random Access Memory)
The computer's **main working memory**, where the OS, running programs, and their active data reside while in use.

**RAM** = fast, **volatile** working memory where running programs and their data live *temporarily* (lost on power-off).

### 2.1 Key characteristics
- **Volatile**: contents lost on power-off.
- **Fast**: far faster than storage (nanoseconds vs microseconds/milliseconds).
- **Random access**: any byte can be accessed directly in roughly the same time (vs sequential tape).
- **Read AND write**: freely modifiable.
- **Limited capacity**: measured in GBs (e.g., 8–64 GB typical).

### 2.2 Why it exists
The CPU is too fast to work directly from slow storage. Programs are **loaded from storage into RAM** to run (recall the "How programs are loaded into memory" note), where the CPU (via cache) can access them quickly.

### 2.3 Types of RAM
| Type | Description |
|------|-------------|
| **DRAM** (Dynamic) | Main system RAM; cheap, dense, must be constantly *refreshed* |
| **SRAM** (Static) | Faster, no refresh needed, expensive; used for **CPU cache** |

### 2.4 What lives in RAM at runtime
This is exactly the **process memory layout** from earlier notes — code, globals, **heap**, and **stack** all live in RAM (mapped via virtual memory):
```
RAM holds (per process): Text/Code | Data | BSS | Heap | Stack
```

## 3. ROM (Read-Only Memory)
Small, **non-volatile** memory containing essential, rarely-changed instructions — most importantly the **firmware** that starts the computer.

**ROM** = small, **non-volatile** memory holding firmware needed to boot the machine (rarely changes).

### 3.1 Key characteristics
- **Non-volatile** — keeps its contents without power.
- Traditionally **read-only** (modern variants are writable but slowly/rarely).
- Small capacity.
- Contains **firmware** — low-level startup code.

### 3.2 What ROM does: booting
When you power on, the CPU has no program in RAM yet. It runs code from ROM:
```
Power on -> CPU runs firmware in ROM (BIOS/UEFI)
         -> firmware initializes hardware (POST)
         -> finds the bootloader on storage
         -> loads the OS from storage into RAM
         -> OS takes over
```
This is the **bootstrapping** process — ROM solves the chicken-and-egg problem of "you need a program to load programs."

### 3.3 ROM variants
| Type | Writable? |
|------|-----------|
| **ROM** | No (fixed at manufacture) |
| **PROM** | Once |
| **EPROM** | Erasable with UV light |
| **EEPROM / Flash** | Electrically rewritable (modern firmware uses this — enables BIOS/UEFI updates) |

## 4. Storage (Secondary Storage)
Large-capacity, **non-volatile** memory that **permanently** holds the OS, applications, and user files.

**Storage** = large, **non-volatile** memory (SSD/HDD) that **persists** files and programs *permanently* but is much slower than RAM.

### 4.1 Key characteristics
- **Non-volatile** - data persists across power cycles.
- **Large** - hundreds of GB to many TB.
- **Slower** than RAM (especially HDDs).
- **Cheaper** per byte than RAM.
- Where files, databases, and installed programs actually live.

### 4.2 Storage technologies
| Type | How it works | Speed | Notes |
|------|--------------|-------|-------|
| **HDD** (Hard Disk Drive) | Spinning magnetic platters + moving read/write head | Slow (mechanical seek ~10 ms) | Cheap, high capacity |
| **SSD** (Solid State Drive) | Flash memory, no moving parts | Fast (~tens of µs) | More expensive, durable, now standard |
| **NVMe SSD** | SSD over the PCIe bus | Very fast | Highest performance consumer storage |
| **Optical** (CD/DVD/Blu-ray) | Laser-read discs | Slow | Mostly legacy |
| **Tape** | Magnetic tape | Very slow, sequential | Archival backups |

> **HDD vs SSD** is a huge real-world performance factor: SSDs eliminate the mechanical seek time of HDDs, which is why databases and servers favor them.

## 5. How They Work Together
The full flow:
```
1. Power on:
      ROM firmware (BIOS/UEFI) runs -> initializes hardware

2. Boot:
      OS is loaded from STORAGE -> into RAM

3. Run a program:
      Program file copied from STORAGE -> into RAM (as a process)
      CPU executes it from RAM (via CACHE and REGISTERS)

4. Work with data:
      Active data lives in RAM (fast)
      To keep data permanently -> SAVE it back to STORAGE

5. Power off:
      RAM is wiped (volatile)
      STORAGE and ROM retain their contents (non-volatile)
```

This flow ties together every prior note: ROM boots → OS/program loaded from storage into RAM → CPU runs the instruction cycle on data in registers/cache → results saved back to storage.

## 6.RAM vs. ROM vs. Storage

| Property | **RAM** | **ROM** | **Storage (SSD/HDD)** |
|----------|---------|---------|------------------------|
| Volatility | Volatile | Non-volatile | Non-volatile |
| Speed | Fast | Moderate | Slow(er) |
| Capacity | GBs | KBs–MBs | Hundreds GB–TBs |
| Read/Write | Read & write | Mostly read | Read & write |
| Cost per byte | High | Low | Lowest |
| Purpose | Active working memory | Boot firmware | Permanent storage |
| Loses data on power off? | **Yes** | No | No |
| Example contents | Running app, heap, stack | BIOS/UEFI | Files, OS install, DB |

## 7. How This Maps to Java / Backend

- **JVM heap lives in RAM** (`-Xmx` caps it). When you create objects with `new`, they occupy RAM. Power off → gone.
- **Persistence requires storage:** that's exactly *why* databases, files, and caches-with-disk exist. A Java app's in-memory state (`HashMap`, list, etc.) vanishes on restart unless persisted to storage (DB/file) — a core reason **databases** exist (Phase 4).
- **RAM is the constraint:** `OutOfMemoryError` happens when the heap can't fit in available RAM. Server sizing = matching working set to RAM (ties to virtual memory / swapping note — swapping to storage is catastrophically slow).
- **SSD vs HDD** affects database and log performance dramatically.
- **Caching strategy** (Phase 5.7, 13): keep hot data in fast RAM (e.g., Redis, in-process cache) to avoid slow storage/DB reads — the entire memory hierarchy applied at the system level.

## 8. Why This Matters for a Backend (Java) Engineer

- Explains **volatile vs persistent** state — the foundation of why we need databases.
- Grounds **memory limits** and `OutOfMemoryError` / server sizing decisions.
- Explains why **swapping RAM to disk** wrecks performance (storage is orders of magnitude slower).
- Underpins **caching** as a core scalability technique (trade cheap-but-slow storage reads for fast RAM hits).
- Clarifies the (unrelated!) naming clash with Java's `volatile` keyword.

## 9. Quick Self-Check Questions

1. What does "volatile" mean for memory, and which memories are volatile?
2. Why must programs be loaded from storage into RAM to run?
3. What is the purpose of ROM, and what is bootstrapping?
4. Why is RAM faster but smaller than storage?
5. What's the difference between DRAM and SRAM (and where is each used)?
6. Why are SSDs faster than HDDs?
7. Walk through what happens to RAM, ROM, and storage when you power off.
8. How does the memory hierarchy explain why caching improves backend performance?

## 10. Key Terms

- **RAM:** Random Access Memory — fast, volatile, main working memory.
- **ROM:** Read-Only Memory — non-volatile firmware/boot memory.
- **Storage / secondary storage:** large non-volatile memory (SSD/HDD) for files.
- **Volatile:** loses contents without power.
- **Non-volatile:** retains contents without power.
- **DRAM / SRAM:** dynamic (main RAM) vs static (cache) RAM.
- **Firmware:** low-level software stored in ROM (e.g., BIOS/UEFI).
- **BIOS / UEFI:** firmware that boots the computer.
- **Bootstrapping / booting:** startup process loading the OS into RAM.
- **HDD:** mechanical magnetic-platter storage.
- **SSD / NVMe:** flash-based (and PCIe-based) storage; faster, no moving parts.
- **Memory hierarchy:** layered speed/size/cost tradeoff (registers→cache→RAM→storage).
- **Random access:** any location reachable directly in ~constant time.