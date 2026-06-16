# Garbage Collection Deep Dive

> **Phase 1 — Java Language Mastery → 1.11 JVM Deep Dive**
> Goal: Master how the JVM reclaims memory automatically — GC roots, reachability, the generational model, GC algorithms (Serial, Parallel, G1, ZGC, Shenandoah), and GC tuning.

---

## 0. The Big Picture

**Garbage Collection (GC)** is automatic memory management: the JVM **reclaims memory** occupied by objects that are no longer reachable, so you don't manually `free()` (recall Phase 0.1 — no manual delete in Java). The cost is occasional **pauses** and CPU overhead — understanding GC is key to backend **latency** and **throughput**.

```
Objects you stop using -> become unreachable -> GC reclaims their memory automatically
```

> GC is a top cause of **latency spikes** ("stop-the-world pauses") in production. Choosing and tuning the right collector is a senior-level skill (Phase 13).

---

## 1. GC Roots & Reachability

GC decides what's garbage via **reachability**: an object is **alive** if it's reachable from a **GC root**; otherwise it's garbage.

### 1.1 GC roots (the starting points)
| GC root | Examples |
|---------|----------|
| **Local variables / operand stacks** | Active method frames (per thread) |
| **Static fields** | Class-level references |
| **Active threads** | Running thread objects |
| **JNI references** | Native code references |

### 1.2 Reachability
```
GC roots -> objA -> objB -> objC      (all reachable -> ALIVE)
            objX -> objY              (no path from a root -> GARBAGE -> collected)
```
- The GC traces from roots, marking everything reachable. Unmarked objects are garbage.
- **Note:** circular references are NOT a problem for tracing GC — if a cycle isn't reachable from any root, the whole cycle is collected (unlike naive reference counting).

### 1.3 Reference types (awareness)
Java has four reference strengths affecting GC:
| Reference | GC behavior |
|-----------|-------------|
| **Strong** (normal) | Never collected while reachable |
| **Soft** | Collected only when memory is low (good for caches) |
| **Weak** | Collected at the next GC if only weakly reachable (`WeakHashMap` — Phase 1.6) |
| **Phantom** | For cleanup actions after collection |

---

## 2. The Mark-and-Sweep Algorithm (the basis)

The fundamental GC algorithm has two phases:
```
1. MARK:   trace from GC roots, mark all reachable objects
2. SWEEP:  reclaim memory of unmarked (unreachable) objects
(3. COMPACT: optionally, move survivors together to remove fragmentation)
```
- **Mark-Sweep:** simple but leaves **fragmentation** (free gaps — recall Phase 0.1 heap fragmentation).
- **Mark-Sweep-Compact:** also **compacts** survivors → no fragmentation, but moving objects costs time.
- **Copying:** copy live objects to a new region, discard the old wholesale (fast for areas with little surviving data — used for the young gen).

---

## 3. The Generational Hypothesis

> **Weak generational hypothesis: most objects die young.** A typical app creates many short-lived objects (loop temporaries, request-scoped data) and few long-lived ones.

The JVM exploits this by splitting the heap into generations (recall previous note) and collecting them differently:
```
HEAP
├── Young Generation (Eden + 2 Survivor spaces)  -> collected FREQUENTLY (cheap)
└── Old/Tenured Generation                        -> collected RARELY (expensive)
```

### 3.1 Object lifecycle through generations
```
1. new object -> Eden
2. Eden fills -> Minor GC: live objects copied to a Survivor space; dead ones discarded
3. survive several Minor GCs -> PROMOTED to Old generation
4. Old gen fills -> Major/Full GC (more expensive)
```
- Because most objects die young, **Minor GCs collect a lot of garbage cheaply** (the young gen is small and uses fast copying collection).

---

## 4. Minor GC vs Major GC vs Full GC

| GC type | Collects | Frequency | Cost |
|---------|----------|-----------|------|
| **Minor GC** | Young generation only | Frequent | Cheap, short pause |
| **Major GC** | Old generation | Less frequent | Expensive |
| **Full GC** | **Entire** heap (young + old) + sometimes metaspace | Rare | **Most expensive** (longest pause) |

> Frequent **Full GCs** are a red flag — usually a sign of memory pressure or a leak (later note). Minor GCs are normal and healthy.

---

## 5. Stop-The-World (STW) Pauses

> **Most GC work requires pausing all application threads** — a **stop-the-world (STW)** pause — so the GC can safely move/reclaim objects.

- During STW, your app **does nothing** → request latency spikes.
- Pause **length** depends on the collector and heap size: a Full GC on a large heap can pause for **seconds**.
- Modern collectors (G1, ZGC, Shenandoah) minimize STW by doing most work **concurrently** (alongside the app).
> Minimizing STW pauses is the central goal of modern GC and a key latency concern (Phase 13). A 2-second Full GC pause = 2 seconds of timeouts for every in-flight request.

---

## 6. GC Algorithms (Collectors)

HotSpot offers several collectors, each with different throughput/latency trade-offs. Choose based on your needs:

### 6.1 Serial GC
- **Single-threaded**, stops the world for all collection.
- Simple; low overhead; tiny heaps / single-core / small apps only.
- `-XX:+UseSerialGC`

### 6.2 Parallel GC (Throughput Collector)
- **Multi-threaded** young + old collection; still STW, but parallel → fast.
- Optimizes **throughput** (total work done) at the cost of pause length.
- Good for **batch jobs** where pauses are acceptable but throughput matters.
- `-XX:+UseParallelGC` (was the default before Java 9)

### 6.3 G1 GC (Garbage-First) — the default since Java 9
- Divides the heap into many equal-size **regions** (not contiguous generations).
- Collects regions with the **most garbage first** ("garbage-first") → predictable, bounded pauses.
- Does much work **concurrently**; aims for a **pause-time target** (`-XX:MaxGCPauseMillis`).
- **Mixed collections** clean young + some old regions together.
- **The general-purpose default** — good balance of throughput and latency for most server apps.
- `-XX:+UseG1GC`

### 6.4 ZGC (low-latency)
- **Sub-millisecond** pauses, even on **huge heaps** (terabytes).
- Most work is **concurrent**; uses **colored pointers** & load barriers to relocate objects without long STW.
- For **latency-sensitive** apps (trading, real-time) and very large heaps.
- `-XX:+UseZGC`

### 6.5 Shenandoah (low-latency)
- Also **concurrent compaction** → very low, heap-size-independent pauses (similar goals to ZGC).
- `-XX:+UseShenandoahGC`

### 6.6 Comparison
| Collector | Optimizes | Pauses | Best for |
|-----------|-----------|--------|----------|
| **Serial** | Simplicity | STW | Tiny apps / single core |
| **Parallel** | **Throughput** | Longer STW | Batch / throughput-bound |
| **G1** | Balance | Bounded, predictable | **General server default** |
| **ZGC** | **Latency** | Sub-millisecond | Low-latency / huge heaps |
| **Shenandoah** | **Latency** | Very low | Low-latency |
> **Default rule:** use **G1** unless you have a specific need. For strict low latency or very large heaps, consider **ZGC**. For pure throughput batch jobs, **Parallel**.

---

## 7. GC Tuning

### 7.1 Heap sizing (the most impactful)
```
-Xms2g           # initial heap (set = -Xmx to avoid resizing in prod)
-Xmx2g           # max heap
```
> Set `-Xms` = `-Xmx` in production to avoid heap-resizing overhead and surprises. ⚠️ **Never let the JVM swap** — if the heap exceeds physical RAM and pages to disk, GC (which scans the heap) triggers catastrophic page faults (recall Phase 0.1 virtual memory). Size the heap to fit comfortably in RAM.

### 7.2 In containers (critical — Phase 10)
```
-XX:MaxRAMPercentage=75.0    # use 75% of the CONTAINER's memory limit (not the host's)
```
> Modern JVMs are container-aware (respect cgroup limits), but set `MaxRAMPercentage` to leave headroom for off-heap (metaspace, thread stacks, native) — otherwise the **OS OOM-killer** kills your container.

### 7.3 GC logging & analysis
```
-Xlog:gc*                          # GC logging (Java 9+; was -XX:+PrintGCDetails)
-Xlog:gc*:file=gc.log:time,uptime  # to a file with timestamps
```
Analyze logs (GCeasy, GCViewer) for: pause times, frequency, allocation rate, and whether the old gen keeps growing (leak symptom).

### 7.4 Other useful flags
```
-XX:MaxGCPauseMillis=200           # G1 pause-time goal
-XX:+HeapDumpOnOutOfMemoryError    # dump heap on OOM (for analysis — next note)
-XX:+UseStringDeduplication        # G1: collapse duplicate String contents (Phase 1.4)
```

### 7.5 Common GC problems & symptoms
| Symptom | Likely cause |
|---------|--------------|
| Frequent **Full GCs** | Memory pressure / leak / under-sized heap |
| Long STW pauses | Large heap + wrong collector → try G1/ZGC |
| `OutOfMemoryError: GC overhead limit exceeded` | GC running constantly, reclaiming almost nothing → leak/too-small heap |
| Old gen keeps growing | **Memory leak** (lingering references — next note) |
| High allocation rate | Too many short-lived objects → reduce allocation / tune young gen |

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Thinking GC eliminates all memory issues | Leaks (lingering references) still cause OOM (next note) |
| Calling `System.gc()` | Just a hint; usually harmful — don't rely on it |
| Circular references "leak" | They don't — tracing GC handles cycles |
| Letting the JVM heap swap to disk | Catastrophic GC pauses — size to fit RAM |
| Ignoring container memory limits | Set `-XX:MaxRAMPercentage`; avoid OS OOM-kill |
| Using the wrong collector for the workload | G1 default; ZGC for low latency; Parallel for throughput |
| Not enabling GC logging in prod | Always log GC for diagnosis |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Latency (Phase 13):** STW pauses cause request latency spikes — collector choice (G1/ZGC) and tuning directly affect p99 latency.
- **Containers (Phase 10):** heap sizing vs container limits (`MaxRAMPercentage`) prevents OOM-kills; swap avoidance is essential.
- **Memory leaks (next note, Phase 13):** old-gen growth + frequent Full GCs signal a leak → heap dump analysis.
- **Throughput vs latency:** batch jobs (Parallel) vs APIs (G1/ZGC) — match the collector to the workload.
- **GraalVM native image (Phase 16.5):** different memory model (no JIT warm-up), still has GC.
- **Immutability & low allocation** (Phase 1.3) reduce GC pressure.

---

## 10. Quick Self-Check Questions

1. How does GC decide what to collect (roots & reachability)?
2. Why aren't circular references a problem for tracing GC?
3. What are the mark, sweep, and compact phases?
4. State the generational hypothesis and how the heap exploits it.
5. What's the difference between Minor, Major, and Full GC?
6. What is a stop-the-world pause, and why does it matter for latency?
7. Compare Serial, Parallel, G1, and ZGC — when use each?
8. What's the default collector since Java 9?
9. How do you size the heap safely, especially in containers? Why avoid swapping?
10. Name three symptoms of a GC/memory problem.

---

## 11. Key Terms Glossary

- **Garbage collection (GC):** automatic reclamation of unreachable objects.
- **GC root:** a starting reference for reachability (stack locals, statics, threads).
- **Reachability:** whether an object is reachable from a root (alive vs garbage).
- **Mark-sweep(-compact) / copying:** core GC algorithms.
- **Generational hypothesis:** most objects die young.
- **Young/Old gen, Eden, Survivor, promotion:** generational structure & flow.
- **Minor / Major / Full GC:** young / old / entire-heap collection.
- **Stop-the-world (STW):** pausing app threads during GC.
- **Serial / Parallel / G1 / ZGC / Shenandoah:** GC algorithms.
- **`-Xms` / `-Xmx`:** initial / max heap size.
- **`MaxRAMPercentage`:** container-aware heap sizing.
- **`MaxGCPauseMillis`:** G1 pause-time target.
- **Soft/weak/phantom references:** GC-sensitive reference strengths.

---

*Previous topic: **Runtime Data Areas & Execution Engine**.*
*Next topic: **JVM Options & Troubleshooting Tools**.*
