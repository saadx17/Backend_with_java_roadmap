**Virtual memory** is an abstraction: each process sees its own large, private, contiguous **virtual address space**, while the OS + hardware **MMU** transparently map those virtual addresses to scattered **physical RAM** (and sometimes disk). **Paging** is the mechanism that makes this work by splitting memory into fixed-size chunks called **pages**.

> Programs use **virtual addresses**. Hardware uses **physical addresses**. The **MMU** translates between them on every memory access.

## 1. Why Virtual Memory Exists
Without virtual memory, programs would use physical RAM addresses directly, which causes serious problems:

| Problem (without VM) | How virtual memory solves it |
|----------------------|------------------------------|
| Programs could read/corrupt each other's memory | **Isolation** — each process has its own address space |
| Total program size limited by physical RAM | **More memory than RAM** via swapping to disk |
| Memory fragmentation makes large contiguous blocks impossible | Virtual space looks **contiguous** even if physical is scattered |
| Programs must be loaded at fixed addresses | Programs run at **any** physical location, transparently |
| No protection between code/data regions | **Per-page permissions** (read/write/execute) |

This is the foundation of the **process isolation** discussed in the previous note.

## 2. Virtual vs Physical Address Space

```
   PROCESS A (virtual)          PROCESS B (virtual)
   +----------------+           +----------------+
   |  0x0000...     |           |  0x0000...     |
   |  Stack/Heap... |           |  Stack/Heap... |
   |  0xFFFF...     |           |  0xFFFF...     |
   +-------+--------+           +-------+--------+
           |   mapped by MMU            |
           v        (page tables)       v
   +-----------------------------------------------+
   |              PHYSICAL RAM (shared)            |
   |  scattered frames belonging to A and B        |
   +-----------------------------------------------+
                  + DISK (swap) for overflow
```

- Both processes can use the **same virtual addresses** (e.g., `0x400000`) but they map to **different physical RAM** — hence isolation.
- The virtual address space can even be **larger than physical RAM** (the extra lives on disk until needed).
- 
## 3. Paging - The Core Mechanism
To make mapping manageable, memory is divided into **fixed-size blocks**:

- **Page** = a fixed-size block of *virtual* memory (typically **4 KB**).
- **Frame (page frame)** = a same-size block of *physical* RAM.

The OS maps each virtual **page** to a physical **frame**. Because they're the same size, any page can go in any free frame, eliminating external fragmentation.
```
Virtual pages           Physical frames
+--------+              +--------+
| Page 0 |---------+    | Frame0 | <- Page 3
+--------+         |    +--------+
| Page 1 |------+  +--->| Frame1 | <- Page 0
+--------+      |       +--------+
| Page 2 |--+   +------>| Frame2 | <- Page 1
+--------+  |           +--------+
| Page 3 |--|---------->| Frame3 | (free)
+--------+  +---------->| Frame4 | <- Page 2
                        +--------+
(pages need NOT be contiguous in physical RAM)
```

## 4. Address Translation: The Page Table
The mapping from virtual pages → physical frames is stored in a **page table** (one per process, maintained by the OS).

A virtual address is split into two parts:
```
   Virtual Address (e.g. 32-bit, 4KB pages)
  +------------------------+--------------+
  |   Page Number (high)   | Offset (low) |
  +------------------------+--------------+
            |                     |
            v                     |
     Page Table lookup            |
            |                     |
            v                     v
  +------------------------+--------------+
  |   Frame Number         | Offset       |  = Physical Address
  +------------------------+--------------+
```

- **Page number** → index into the page table → gives the **frame number**.
- **Offset** → position within the page → copied directly (page and frame are same size).

### Multi-level page tables
A flat page table for a 64-bit space would be enormous. Real systems use **multi-level (hierarchical) page tables** (e.g., x86-64 uses 4 levels), so only the parts of the table that are actually used consume memory.

## 5. The MMU (Memory Management Unit)
The **MMU** is a **hardware** component (part of the CPU) that performs virtual→physical translation automatically on every memory access, using the page tables the OS set up.

```
CPU issues virtual address -> MMU translates -> accesses physical RAM
                               (uses page table + TLB)
```

The OS *manages* page tables; the MMU *uses* them in hardware. This division is why translation is fast enough to happen billions of times per second.

## 6. The TLB (Translation Lookaside Buffer)
Looking up the page table in RAM on *every* access would be slow (and multi-level tables mean multiple lookups). So the MMU caches recent translations in the **TLB**- a small, very fast cache of virtual-page → physical-frame mappings.

```
CPU -> MMU checks TLB
        |
        +-- TLB HIT  -> translation instantly available (fast path)
        |
        +-- TLB MISS -> walk the page table in RAM, then cache result in TLB
```

- **TLB hit:** near-instant translation.
- **TLB miss:** slower page-table walk, then the result is cached.
- **TLB flush:** on a **process context switch**, translations for the old process are invalid → the TLB is flushed/tagged. This is a major reason process context switches are more expensive than thread switches (recall previous note).

## 7. Page Faults
A **page fault** is a CPU exception raised when a process accesses a virtual page that is **not currently in physical RAM**.

```
Access virtual page
       |
   In RAM? --yes--> proceed normally
       |
       no
       v
   PAGE FAULT (trap to OS kernel)
       |
       +-- valid page on disk? -> load it into a free frame, update page table, resume
       |
       +-- truly invalid address? -> SEGMENTATION FAULT (kill process)
```

Types of page faults:
- **Minor (soft) fault:** page is in RAM but not yet mapped in this process's table (e.g., shared page) — cheap to fix.
- **Major (hard) fault:** page must be read from **disk** (swap or the executable file) — expensive (disk I/O).
- **Invalid fault:** illegal access → **segfault** / crash.

This is the same mechanism behind **demand paging** during program loading (previous note).

## 8. Demand Paging & Lazy Allocation
The OS is **lazy**: it doesn't load or back memory with RAM until it's actually used.

- **Demand paging:** pages are brought into RAM only when first accessed (via page faults), not all at once. → faster startup, less RAM used.
- **Lazy allocation:** when you "allocate" memory (e.g., a big array), the OS may just reserve virtual addresses; physical frames are assigned only when you **touch** each page.
- **Copy-on-write (COW):** after `fork()`, parent and child *share* the same physical frames marked read-only. Only when one **writes** does the OS copy that page. Makes process creation cheap.

## 9. Swapping / Paging to Disk
When physical RAM runs low, the OS moves less-used pages from RAM to a reserved area on **disk** (the **swap space** / page file) to free frames for active pages.

```
RAM full -> OS picks a "victim" page (e.g., least recently used)
         -> writes it to swap on disk
         -> frees that frame for the new page
         -> later access to the swapped page -> major page fault -> load back
```

- Lets total memory usage exceed physical RAM.
- **Thrashing:** if the working set is too large, the system spends most of its time swapping pages in/out instead of doing work → severe slowdown. A warning sign for under-provisioned servers.

### Page replacement algorithms (which page to evict)
| Algorithm | Idea |
|-----------|------|
| **FIFO** | Evict the oldest-loaded page (simple, not great) |
| **LRU** | Evict the least recently used page (good, common) |
| **Clock / Second-chance** | Approximation of LRU, cheaper to implement |
| **Optimal** | Evict the page used furthest in the future (theoretical benchmark) |

## 10. Memory Protection & Sharing
Each page-table entry carries **permission bits**, enforced by hardware:

- **Read / Write / Execute** flags → e.g., code pages are read+execute (not writable), data pages are read+write (not executable — defends against code injection: "W^X").
- **User / Kernel** flag → user code can't touch kernel pages.
- Violations trigger a fault → **segfault**.

**Sharing:** the same physical frame can be mapped into multiple processes' page tables, used for **shared libraries** (one copy of libc in RAM for all processes) and **shared memory IPC**.

## 11. How This Connects to Java / the JVM
Virtual memory underpins everything the JVM does:

- The JVM process gets a **virtual address space**; the **heap** (`-Xms`/`-Xmx`) is carved from it — these are virtual addresses backed lazily by frames.
- `-Xms` (initial) vs `-Xmx` (max): reserving a large `-Xmx` reserves virtual space, but physical RAM is only consumed as the heap actually fills (lazy allocation).
- **GC and page locality:** garbage collectors that compact memory improve **page/cache locality**, reducing TLB and cache misses → real performance impact.
- **Swapping is the enemy of the JVM:** if the heap gets paged to disk, GC (which scans large regions) triggers massive page faults → catastrophic pauses. Production rule: **don't let the JVM swap**; size the heap to fit in RAM.
- **Containers (Phase 10):** memory limits (cgroups) interact with virtual memory; that's why you set `-XX:MaxRAMPercentage` so the JVM respects the container's RAM, avoiding OOM-kills.
- **Huge pages:** some high-performance setups use large (e.g., 2 MB) pages to reduce TLB misses for big heaps.

## 12. Why This Matters for a Backend (Java) Engineer

- Explains **process isolation** (why one service can't corrupt another's memory).
- Explains why **reserving** a big heap doesn't immediately consume RAM (lazy allocation).
- Explains **GC pause spikes** when memory is swapped — a classic production incident.
- Informs **server/container sizing**: keep working set in RAM, avoid thrashing.
- Connects to the earlier **context-switch cost** (TLB flush on process switch).
- Grounds your understanding of `OutOfMemoryError` vs OS-level **OOM-killer** (Linux killing the process when physical RAM + swap are exhausted).

## 13. Quick Self-Check Questions

1. What problem does virtual memory solve that direct physical addressing cannot?
2. What is the difference between a **page** and a **frame**?
3. How is a virtual address split, and how does the page table translate it?
4. What is the **MMU**, and what is the **TLB** — how do they relate?
5. Why is the TLB flushed on a process context switch?
6. Distinguish a **minor** fault, a **major** fault, and a **segfault**.
7. What is **demand paging** and **copy-on-write**?
8. What is **thrashing**, and why is swapping disastrous for a JVM heap?

## 14. Key Terms

- **Virtual memory:** abstraction giving each process its own large private address space.
- **Virtual address:** address used by programs (translated before hitting RAM).
- **Physical address:** actual location in RAM.
- **Page:** fixed-size block of virtual memory (~4 KB).
- **Frame:** fixed-size block of physical RAM (same size as a page).
- **Page table:** per-process map of pages → frames.
- **MMU:** hardware unit that translates virtual → physical addresses.
- **TLB:** cache of recent address translations.
- **Page fault:** exception when accessing a page not in RAM.
- **Minor/major fault:** in-RAM remap vs disk load.
- **Segmentation fault:** illegal memory access → crash.
- **Demand paging:** load pages only when accessed.
- **Lazy allocation:** back virtual memory with RAM only when touched.
- **Copy-on-write (COW):** share pages until one side writes.
- **Swap / page file:** disk area used to extend memory.
- **Thrashing:** excessive swapping that cripples performance.
- **Page replacement (FIFO/LRU/Clock):** policy for choosing pages to evict.
- **W^X / permission bits:** per-page read/write/execute protection.