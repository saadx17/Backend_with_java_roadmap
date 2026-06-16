# Common JVM Issues & Bytecode Basics

> **Phase 1 — Java Language Mastery → 1.11 JVM Deep Dive**
> Goal: Recognize and fix the common JVM problems (OutOfMemoryError variants, StackOverflowError, memory leaks, leaks via class loaders/threads) and read bytecode basics with `javap`.

---

## 0. The Big Picture

This note ties the JVM internals together into **practical diagnosis**: the errors you'll actually hit in production, what causes them, and how to fix them — plus a peek at **bytecode** (the language between `javac` and the JVM). It draws on every prior 1.11 note (data areas, GC, tools).

---

## 1. OutOfMemoryError (the OOM family)

`OutOfMemoryError` is an **`Error`** (not a recoverable exception — recall Phase 1.5) thrown when the JVM can't allocate memory. The **message** tells you *which* region failed — each has a different cause and fix:

### 1.1 `OutOfMemoryError: Java heap space`
- **Cause:** the **heap** is full — too many live objects, undersized heap, or a **memory leak** (§3).
- **Fix:** increase `-Xmx` (if genuinely needed), **or** find & fix the leak (heap dump + MAT — previous note). If it grows unboundedly over time → it's a leak, not a sizing issue.

### 1.2 `OutOfMemoryError: Metaspace`
- **Cause:** too many **classes** loaded (Metaspace full — recall it holds class metadata).
- **Common in:** dynamic proxy generation (CGLIB), repeated app **redeploys** without releasing class loaders (**class-loader leak** — §4), excessive code generation.
- **Fix:** raise `-XX:MaxMetaspaceSize`, but really → fix the class-loading leak.

### 1.3 `OutOfMemoryError: GC overhead limit exceeded`
- **Cause:** the GC is running almost constantly (>98% of time) but reclaiming almost nothing (<2%) — the heap is effectively full of live objects.
- **Meaning:** usually a **leak** or a too-small heap. The JVM gives up rather than thrash forever.

### 1.4 `OutOfMemoryError: unable to create new native thread`
- **Cause:** the OS won't create more threads (too many threads, or per-thread stack memory exhausted native memory).
- **Fix:** reduce thread count (use bounded pools — Phase 1.10!), lower `-Xss`, or raise OS limits (`ulimit`). **Virtual threads** (Phase 1.10) largely avoid this.

| OOM message | Region | Typical fix |
|-------------|--------|-------------|
| `Java heap space` | Heap | Fix leak / raise `-Xmx` |
| `Metaspace` | Metaspace | Fix class-loader leak / raise `MaxMetaspaceSize` |
| `GC overhead limit exceeded` | Heap (GC thrashing) | Fix leak / raise heap |
| `unable to create new native thread` | Native (threads) | Fewer threads / pools / virtual threads |

> ⚠️ **Don't "fix" an OOM by just raising `-Xmx`** — if it's a leak, you only delay the inevitable. Diagnose first (heap dump).

---

## 2. StackOverflowError

Thrown when a thread's **stack** is exhausted (recall the data-areas note, Phase 0.1 guard page):
- **Cause:** deep or **infinite recursion** (most common — recall Phase 1.2), or extremely deep call chains.
```java
int bad(int n) { return bad(n + 1); }   // no base case -> StackOverflowError
```
- **Fix:** add/verify a **base case**, convert recursion to **iteration** (Phase 1.2), or (rarely) increase `-Xss`.
> Read the stack trace: a `StackOverflowError` shows the **same frames repeating** — that repeating method is your runaway recursion.

---

## 3. Memory Leaks in Java (Yes, They Exist!)

> **GC doesn't prevent memory leaks.** A "leak" in Java = objects that are **no longer needed but still reachable** (from a GC root — recall GC note), so the GC **can't** collect them. They accumulate → eventually OOM.

### 3.1 Common causes
| Leak source | Why |
|-------------|-----|
| **Unbounded caches / collections** | A `static Map`/`List` that only grows, never evicts |
| **Static fields holding objects** | Statics live forever → anything they reference lives forever |
| **Listeners/callbacks not deregistered** | Registered objects stay reachable via the listener list |
| **`ThreadLocal` not removed in pools** | Pooled threads retain values (recall Phase 1.10!) |
| **Inner classes holding the outer instance** | Non-static inner class keeps the outer alive (recall Phase 1.3) |
| **Unclosed resources** | Connections/streams holding native memory (recall Phase 1.5/1.9) |

```java
// Classic leak: an ever-growing static cache with no eviction
static final Map<String, Object> cache = new HashMap<>();   // grows forever -> OOM
// Fix: bounded cache with eviction (LinkedHashMap LRU / Caffeine / Redis)
```

### 3.2 Diagnosing a leak
1. **Symptom:** old gen keeps growing across GCs; frequent Full GCs; eventually `OutOfMemoryError: Java heap space` (recall jstat/GC logs).
2. **Capture:** a **heap dump** (`-XX:+HeapDumpOnOutOfMemoryError`, or `jcmd GC.heap_dump`).
3. **Analyze in MAT:** "Leak Suspects" + **"path to GC roots"** shows *what reference is keeping the objects alive* → fix that reference.

### 3.3 Prevention
- Use **bounded** caches with eviction (Caffeine, Redis — Phase 5.7).
- **`remove()` ThreadLocals** in pooled threads (Phase 1.10).
- **Deregister** listeners; use **weak references** where appropriate (`WeakHashMap` — Phase 1.6).
- **Close resources** (try-with-resources — Phase 1.5/1.9).
- Prefer **`static`** nested classes when the outer instance isn't needed (Phase 1.3).

---

## 4. Thread & Class-Loading Leaks

### 4.1 Thread leaks
Threads that are created but **never terminated** (e.g., executors never shut down — recall Phase 1.10) accumulate, consuming memory and eventually causing `unable to create new native thread`.
- **Fix:** always `shutdown()` executors; use bounded pools; ensure threads exit.

### 4.2 Class-loader leaks (the Metaspace killer)
When an app server **redeploys** an app, the old class loader (and all its loaded classes) should be garbage-collected. But if **anything** still references the old loader (a lingering thread, a `ThreadLocal`, a static reference from a shared library), the **entire old class loader and all its classes stay in Metaspace** → repeated redeploys → `OutOfMemoryError: Metaspace`.
> This is the classic "redeploy a few times and the server dies" problem. The fix: eliminate references that pin the old class loader (often a `ThreadLocal` set by the app on a shared pool thread, or a JDBC driver registered in a shared registry).

---

## 5. Bytecode Basics

**Bytecode** (recall the overview note) is the instruction set `javac` produces and the JVM executes. Reading it occasionally helps you understand what your code *really* does (e.g., how `+` on Strings compiles, how lambdas work).

### 5.1 Viewing bytecode with `javap`
```bash
javac MyClass.java          # compile to MyClass.class
javap -c MyClass            # disassemble: show the bytecode
javap -p -c MyClass         # include private members
javap -v MyClass            # verbose: constant pool, flags, everything
```

### 5.2 Example
```java
public int add(int a, int b) { return a + b; }
```
disassembles (roughly) to:
```
public int add(int, int);
  Code:
     0: iload_1        // push local var 1 (a) onto operand stack
     1: iload_2        // push local var 2 (b)
     2: iadd           // pop two ints, add, push result
     3: ireturn        // return the int on top of the stack
```
> The JVM is a **stack machine**: most instructions push/pop the **operand stack** (recall the data-areas note). `iload`/`istore` move between locals and the stack; `iadd` operates on the stack top.

### 5.3 The `invoke` instructions (method calls — recall Phase 1.3 polymorphism)
Method calls compile to specific instructions — knowing them clarifies dispatch:
| Instruction | Used for |
|-------------|----------|
| `invokevirtual` | Normal (overridable) instance method calls → **dynamic dispatch** |
| `invokeinterface` | Calls through an interface reference |
| `invokestatic` | `static` method calls (no dispatch) |
| `invokespecial` | Constructors, `private` methods, `super.method()` (no dispatch) |
| `invokedynamic` | **Lambdas** and dynamic languages (resolved at runtime) |

> `invokedynamic` is *how lambdas work* (recall Phase 1.8 — no extra class file). `super.method()` uses `invokespecial` precisely to **bypass** dynamic dispatch (recall Phase 1.3). Seeing these in bytecode confirms the dispatch behavior.

### 5.4 When you'd actually read bytecode
- Understanding how a language feature compiles (String concat, lambdas, switch, autoboxing).
- Debugging tricky behavior or library internals.
- Bytecode-manipulation tools (ASM, ByteBuddy) used by frameworks (Spring AOP/proxies, mocking — Phase 1.14, 5, 6).
> You rarely read bytecode day-to-day, but knowing `javap` and the invoke instructions deepens your mental model and demystifies framework "magic."

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| "GC means no memory leaks" | Reachable-but-unneeded objects still leak |
| Fixing OOM by only raising `-Xmx` | Diagnose first; a leak will OOM again |
| Misreading which OOM you have | The message names the region — fix accordingly |
| Ignoring `unable to create new native thread` | Bound your thread pools / use virtual threads |
| ThreadLocal not removed (pool) | Always `remove()` — causes heap & class-loader leaks |
| Repeated redeploys killing Metaspace | Find what pins the old class loader |
| Treating `StackOverflowError` as a memory bug | It's recursion depth — fix the base case |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Production stability (Phase 13):** OOM/leak/thread diagnosis is core SRE/senior work — these are the most common JVM incidents.
- **Caches (Phase 5.7):** unbounded caches are the #1 heap-leak source → use Caffeine/Redis with eviction.
- **Executors/`@Async` (Phase 1.10, 5):** un-shut-down pools cause thread leaks; bounded pools are mandatory.
- **Redeploys / app servers:** class-loader leaks (often via `ThreadLocal`/drivers) cause Metaspace OOM.
- **Frameworks (Phase 1.14, 5, 6):** Spring AOP, proxies, and mocking use bytecode manipulation (ASM/ByteBuddy/CGLIB) — `invokedynamic`/proxies explained here.
- **Virtual threads (Phase 1.10)** mitigate native-thread exhaustion.

---

## 8. Quick Self-Check Questions

1. Name the OOM variants and the region/cause of each.
2. Why is raising `-Xmx` not a real fix for a leak?
3. What causes `StackOverflowError`, and how do you read its stack trace?
4. What is a memory leak in Java if GC exists? Name three common causes.
5. How do you diagnose a heap leak (tools & steps)?
6. What is a class-loader leak, and why does it kill Metaspace on redeploys?
7. How do you view bytecode, and what kind of machine is the JVM?
8. What do `invokevirtual`, `invokestatic`, `invokespecial`, and `invokedynamic` mean?

---

## 9. Key Terms Glossary

- **`OutOfMemoryError`:** unrecoverable error when memory can't be allocated (region-specific).
- **Heap space / Metaspace / GC overhead / native thread OOM:** the OOM variants.
- **`StackOverflowError`:** thread stack exhausted (usually recursion).
- **Memory leak:** unneeded-but-reachable objects accumulating.
- **Thread leak:** threads never terminated.
- **Class-loader leak:** an old loader (and its classes) pinned in Metaspace.
- **Heap dump / MAT / path to GC roots:** capture / analyze / find what retains objects.
- **Bytecode:** `.class` instructions executed by the JVM (stack machine).
- **`javap`:** disassembler to view bytecode.
- **Operand stack:** bytecode working space.
- **`invokevirtual`/`invokeinterface`/`invokestatic`/`invokespecial`/`invokedynamic`:** method-call instructions.
- **ASM / ByteBuddy / CGLIB:** bytecode-manipulation libraries (used by frameworks).

---

*Previous topic: **JVM Options & Troubleshooting Tools**.*
*This completes **Section 1.11 — JVM Deep Dive**.*
*Next section in roadmap: **1.12 Java Modules (JPMS)**.*
