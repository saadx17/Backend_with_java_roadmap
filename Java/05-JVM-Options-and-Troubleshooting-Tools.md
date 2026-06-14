# JVM Options & Troubleshooting Tools

> **Phase 1 — Java Language Mastery → 1.11 JVM Deep Dive**
> Goal: Master the essential JVM tuning options and the diagnostic tools (jps, jstack, jmap, jstat, jcmd, JFR, VisualVM, MAT) you'll use to troubleshoot production Java systems.

---

## 0. The Big Picture

When a production JVM misbehaves — high CPU, memory growth, slow responses, OOM — you need to **observe** it. The JDK ships with powerful command-line and GUI tools to inspect threads, memory, GC, and performance **live**. Knowing these is what makes you able to debug real incidents (Phase 13).

```
Problem -> pick the right tool -> capture data (thread dump / heap dump / JFR) -> analyze -> fix
```

> This builds on the data areas (heap, stacks, metaspace) and GC from the previous notes — the tools expose exactly those.

---

## 1. Essential JVM Options

JVM options come in three flavors:
- **Standard** (`-`): stable across JVMs (`-Xmx`, `-cp`).
- **`-X`**: non-standard but common (`-Xms`, `-Xss`).
- **`-XX`**: advanced/experimental, may change (`-XX:+UseG1GC`). `+`/`-` toggles boolean flags.

### 1.1 Memory sizing (recall the data-areas & GC notes)
```
-Xms2g                          # initial heap size
-Xmx2g                          # max heap size (set = -Xms in prod)
-Xss512k                        # thread stack size (affects recursion depth & thread count)
-XX:MetaspaceSize=128m          # initial metaspace
-XX:MaxMetaspaceSize=256m       # cap metaspace (prevent class-loader leak runaway)
-XX:MaxRAMPercentage=75.0       # container-aware heap sizing (use in Docker/K8s)
```

### 1.2 GC selection & tuning (recall the GC note)
```
-XX:+UseG1GC                    # G1 (default since Java 9)
-XX:+UseZGC                     # ZGC (low latency)
-XX:MaxGCPauseMillis=200        # G1 pause-time goal
-Xlog:gc*:file=gc.log:time      # GC logging (Java 9+)
-XX:+UseStringDeduplication     # G1: dedupe duplicate Strings (Phase 1.4)
```

### 1.3 Diagnostics on failure (always set these in prod!)
```
-XX:+HeapDumpOnOutOfMemoryError          # auto heap dump on OOM (crucial for post-mortem)
-XX:HeapDumpPath=/var/dumps/             # where to write it
-XX:+ExitOnOutOfMemoryError              # exit so the orchestrator restarts cleanly
-XX:+CrashOnOutOfMemoryError             # (alternative) crash with details
```
> ⚠️ **Always enable `-XX:+HeapDumpOnOutOfMemoryError` in production.** When an OOM strikes, you get a heap dump to analyze the leak (§5) — without it, you're blind. Pair with `-XX:+ExitOnOutOfMemoryError` so a crashed pod restarts (Phase 10) rather than limping on.

### 1.4 Other useful options
```
-Dproperty=value                # set a system property (e.g., -Dspring.profiles.active=prod)
-jar app.jar                    # run a JAR
-cp / -classpath                # classpath
-XX:+FlightRecorder             # enable JFR (usually on by default now)
```

---

## 2. Diagnostic Tools — Overview

The first step is always **find the PID** (recall Phase 0.3 `ps`/`jps`), then point a tool at it.

| Tool | Purpose | Type |
|------|---------|------|
| `jps` | List running JVMs and their PIDs | CLI |
| `jinfo` | View/modify JVM flags & system properties | CLI |
| `jstat` | GC & memory statistics over time | CLI |
| `jstack` | Thread dump (what every thread is doing) | CLI |
| `jmap` | Heap info / trigger a heap dump | CLI |
| `jcmd` | Swiss-army diagnostic (does most of the above) | CLI |
| `jconsole` / VisualVM | Live GUI monitoring | GUI |
| **JFR** + Mission Control | Low-overhead profiling/recording | Built-in |
| Async Profiler | CPU/alloc flame graphs | External |
| Eclipse MAT | Heap dump analysis (find leaks) | GUI |

---

## 3. Command-Line Tools (the daily kit)

### 3.1 jps — find JVMs
```bash
jps -l        # list PIDs + main class/JAR
# 12345 com.example.MyApp
# 12346 sun.tools.jps.Jps
```

### 3.2 jstack — thread dump (for hangs, deadlocks, high CPU)
Captures a snapshot of **every thread's stack** — the go-to for **deadlocks**, **hung** apps, and **high CPU**:
```bash
jstack 12345 > threads.txt      # dump all threads of PID 12345
```
What you find:
- **Deadlocks** (jstack explicitly reports "Found one Java-level deadlock").
- Threads stuck in `BLOCKED`/`WAITING` (recall Phase 1.10 states).
- Which method a runaway thread is executing (correlate with `top -H` to find the hot thread).
> Take **multiple** thread dumps a few seconds apart — threads stuck in the same place across dumps = the problem.

### 3.3 jmap — heap info & heap dumps (for memory issues)
```bash
jmap -heap 12345                          # heap configuration & usage summary
jmap -histo 12345 | head -20              # histogram: top classes by instance count/size
jmap -dump:live,format=b,file=heap.hprof 12345   # full heap dump (analyze in MAT)
```
> `jmap -histo` quickly shows **which classes dominate memory** — a fast first look at a leak. A full dump (`-dump`) is for deep analysis in **Eclipse MAT** (§5).

### 3.4 jstat — GC statistics over time (for GC behavior)
```bash
jstat -gc 12345 1000              # GC stats every 1000ms (1s)
jstat -gcutil 12345 1000          # GC utilization % (Eden, Survivor, Old, Metaspace, GC counts/times)
```
Watch for: Old gen filling and never shrinking (leak), frequent Full GCs, rising GC time.

### 3.5 jcmd — the modern all-in-one
`jcmd` consolidates most diagnostics:
```bash
jcmd 12345 help                          # list available commands
jcmd 12345 Thread.print                  # thread dump (like jstack)
jcmd 12345 GC.heap_info                  # heap info
jcmd 12345 GC.heap_dump /tmp/heap.hprof  # heap dump (like jmap)
jcmd 12345 VM.flags                      # all JVM flags in effect
jcmd 12345 VM.system_properties          # system properties
jcmd 12345 JFR.start duration=60s filename=rec.jfr   # start a Flight Recorder recording
```
> **`jcmd` is the recommended modern tool** — it replaces much of `jstack`/`jmap`/`jinfo` with one command and is the entry point to JFR.

### 3.6 jinfo
```bash
jinfo -flags 12345               # JVM flags
jinfo -sysprops 12345            # system properties
```

---

## 4. GUI & Profiling Tools

### 4.1 jconsole / VisualVM
- **jconsole:** simple live monitoring (heap, threads, classes, MBeans).
- **VisualVM:** richer — live charts, thread monitoring, basic CPU/memory **profiling**, heap dump viewing. Great for dev/staging.

### 4.2 Java Flight Recorder (JFR) + JDK Mission Control (JMC)
- **JFR:** a **built-in, very low-overhead** event recorder — captures allocations, GC, locks, method profiling, I/O, etc. Suitable for **production** (overhead ~1%).
- **JMC (Mission Control):** the GUI to **analyze** JFR recordings — flame graphs, hot methods, allocation sources, GC analysis.
```bash
jcmd 12345 JFR.start duration=120s filename=app.jfr settings=profile
# then open app.jfr in JDK Mission Control
```
> **JFR is the go-to production profiler** — low overhead, always available, deep insight. Learn it.

### 4.3 Async Profiler
A popular external **low-overhead sampling profiler** producing **flame graphs** for CPU and allocation hotspots — excellent for finding where time/memory goes (Phase 13).

### 4.4 Eclipse MAT (Memory Analyzer Tool)
The standard tool for **analyzing heap dumps** (`.hprof`):
- **Leak Suspects report:** automatically flags likely leaks.
- **Dominator tree:** shows which objects retain the most memory.
- **Path to GC roots:** for a suspected object, shows *what's keeping it alive* (the reference chain — recall reachability, GC note) → pinpoints the leak.
> When you have an OOM heap dump, **MAT's "path to GC roots"** tells you exactly which reference is preventing collection — the key to fixing leaks (next note).

---

## 5. The Troubleshooting Playbook (which tool for which problem)

| Symptom | Tools & approach |
|---------|------------------|
| **High CPU** | `top -H -p <pid>` (find hot thread) → `jstack`/JFR → see what it's running |
| **App hung / unresponsive** | `jstack` ×2 → look for deadlocks / blocked threads |
| **Deadlock** | `jstack` (reports it explicitly) |
| **Memory growing / OOM** | `jstat -gcutil` (old gen growth) → heap dump (`jcmd GC.heap_dump`) → **MAT** (path to GC roots) |
| **Metaspace OOM** | Check class count growth → class-loader leak (proxies/redeploys) |
| **Slow / latency spikes** | GC logs (`-Xlog:gc*`) → JFR/Async Profiler for hotspots |
| **General profiling** | **JFR** + Mission Control, or Async Profiler (flame graphs) |

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| No `-XX:+HeapDumpOnOutOfMemoryError` in prod | Always enable it (post-mortem analysis) |
| Taking one thread dump for a hang | Take several; compare |
| Heap dump on a huge heap blocks the app | It pauses the JVM; do it carefully (or use JFR for live data) |
| Heavy profilers in production | Use **JFR** (low overhead) |
| Reading a histogram and guessing | Use MAT's "path to GC roots" to confirm the leak |
| Ignoring GC logs | They reveal pause/leak patterns |
| `-Xmx` larger than container limit | OS OOM-kill — use `MaxRAMPercentage` |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Production incident response (Phase 13):** these tools *are* how you debug real CPU/memory/latency problems.
- **Spring Boot Actuator** (Phase 5.2, 9) exposes some of this (heap, threads, metrics) over HTTP — complementary to JVM tools.
- **Container tuning (Phase 10):** `-XX:MaxRAMPercentage`, heap dump paths to mounted volumes, JFR for profiling pods.
- **Memory-leak diagnosis (next note):** heap dumps + MAT find the lingering references.
- **APM tools** (Datadog, New Relic — Phase 9.3) build on JVM/JFR data for continuous monitoring.
- **Performance tuning (Phase 13):** JFR/Async Profiler flame graphs guide optimization.

---

## 8. Quick Self-Check Questions

1. What are the three categories of JVM options (`-`, `-X`, `-XX`)?
2. Which options set heap, stack, and metaspace sizes? Which for containers?
3. Why always enable `-XX:+HeapDumpOnOutOfMemoryError` in production?
4. How do you find a running JVM's PID?
5. Which tool for a deadlock/hang? For high CPU? For memory growth?
6. What does `jmap -histo` show vs a full heap dump?
7. What is `jcmd`, and why is it recommended?
8. What is JFR, and why is it suitable for production?
9. In Eclipse MAT, what feature pinpoints a leak's cause?

---

## 9. Key Terms Glossary

- **JVM options (`-`/`-X`/`-XX`):** standard / non-standard / advanced flags.
- **`-Xms`/`-Xmx`/`-Xss`:** heap initial/max, thread stack size.
- **`-XX:+HeapDumpOnOutOfMemoryError`:** auto heap dump on OOM.
- **`jps`:** list JVMs & PIDs.
- **`jstack`:** thread dump (deadlocks, hangs, hot threads).
- **`jmap`:** heap info / histogram / heap dump.
- **`jstat`:** GC & memory stats over time.
- **`jcmd`:** modern all-in-one diagnostic.
- **Thread dump / heap dump:** snapshot of threads / of heap memory.
- **JFR (Java Flight Recorder):** low-overhead event recorder.
- **JDK Mission Control (JMC):** GUI to analyze JFR recordings.
- **Async Profiler:** sampling profiler (flame graphs).
- **Eclipse MAT:** heap-dump analyzer (leak suspects, path to GC roots).

---

*Previous topic: **Garbage Collection Deep Dive**.*
*Next topic: **Common JVM Issues & Bytecode Basics**.*
