# GraalVM & Native Image

> **Phase 16 — Advanced Topics → 16.5 GraalVM / Native Image**
> Goal: Understand GraalVM native images — ahead-of-time (AOT) compilation, the benefits (fast startup, low memory) and trade-offs (build time, reflection limits), how Spring Boot supports native, and when it's worth it.

---

## 0. The Big Picture

Normally Java runs on the **JVM**: bytecode is interpreted then **JIT-compiled** at runtime (Phase 1.11) — fast peak performance, but **slow startup** and high memory (the JVM warms up). **GraalVM Native Image** does **ahead-of-time (AOT)** compilation: it compiles your app + needed JDK + dependencies into a **single standalone native executable** ahead of time — **starting in milliseconds** with a **much smaller memory footprint**.

```
   JVM:    bytecode → interpret → JIT compile (warmup) → fast peak. Startup ~seconds, RAM high.
   Native: AOT compile everything → a single OS binary. Startup ~10-50ms, RAM low, no warmup.
```

> Builds on the JVM/JIT (Phase 1.11). Hugely relevant to **containers/serverless** (Phase 10.1 — tiny fast images), **Kubernetes scaling** (10.2 — fast scale-up), and **microservices** (Phase 12). Spring Boot 3+ has first-class native support.

---

## 1. JIT (JVM) vs AOT (Native) ⭐

| | **JVM (JIT)** | **Native Image (AOT)** |
|--|---------------|------------------------|
| Compilation | At runtime (warms up) | Ahead of time (build) |
| **Startup** | Seconds | ⭐ **Milliseconds** (10–100×) |
| **Memory** | Higher (JVM overhead) | ⭐ Much lower |
| **Peak throughput** | ⭐ Often **higher** (JIT optimizes hot paths with runtime profile) | Sometimes slightly lower (no runtime profiling*) |
| Build time | Fast | ⚠️ **Slow** (minutes) |
| Image size | JRE + jar | Small standalone binary |
| Reflection/dynamic | ✅ Full | ⚠️ Needs configuration (§3) |
> ⭐ **Native trades peak throughput & build speed for instant startup and low memory.** ⚠️ For a **long-running, throughput-heavy** service, the JVM's JIT may actually be *faster at peak* (it optimizes using live profiling). Native wins where **startup time and memory** dominate. (*Profile-Guided Optimization can recover some peak performance.)

---

## 2. When Native Image Is Worth It ⭐

| Great for | Less ideal for |
|-----------|----------------|
| **Serverless / FaaS** (AWS Lambda — 16.7) — cold start kills you | Long-running, CPU-bound services where peak throughput matters |
| **Fast autoscaling** (10.2 HPA) — new instances ready instantly | Apps relying heavily on reflection/dynamic class loading |
| **High-density / low-memory** deployments (cost) | Teams unwilling to absorb slow builds / native debugging |
| Short-lived CLI tools / jobs (10.2 Jobs) | Rapid dev iteration (build time hurts) |
| Tiny container images (10.1) | Heavy use of agents/JMX/some profilers (9.3) |
> ⭐ **The killer use case is serverless & rapid scaling**: a function that must respond instantly from cold, or a service that scales from 2→50 pods in seconds (10.2). The fast startup + low RAM also cuts cloud cost at high density. ⚠️ If your app runs 24/7 under steady high load, the JVM is often the better, simpler choice.

---

## 3. The Closed-World Assumption & Reflection ⚠️

Native Image uses **static analysis at build time** to determine all reachable code (the **closed-world assumption**) — it must know *everything* the app will do up front. This breaks Java's **dynamic** features unless configured:
| Dynamic feature (Phase 1.14) | Native issue |
|------------------------------|--------------|
| **Reflection** | Must be declared (build can't see runtime-only reflective calls) |
| **Dynamic proxies** (Spring AOP — 5.1/14.1) | Must be registered |
| **Resources / classpath scanning** | Must be configured |
| **Serialization / JNI** | Must be declared |
```
   Solution: reachability metadata / config files (reflect-config.json, etc.)
   Spring Boot's AOT engine generates most of this for you (§4)
```
> ⚠️ ⭐ The **closed-world assumption** is the core constraint: anything determined only at runtime (reflection — Phase 1.14) must be **declared at build time** via metadata. Historically this was painful; today **frameworks generate it** (§4) and libraries ship metadata, but **unsupported libraries can still break** in native mode — test thoroughly.

---

## 4. Spring Boot Native Support

Spring Boot 3+ (built on **Spring AOT** and the **GraalVM native build tools**) makes native a first-class option:
```bash
# Maven — produce a native executable (needs a GraalVM JDK)
mvn -Pnative native:compile
# Or build a native container image (no local GraalVM needed) via buildpacks:
mvn -Pnative spring-boot:build-image
```
| Spring native piece | Does |
|---------------------|------|
| **Spring AOT processing** | Analyzes the app at build time → generates reachability metadata + optimized config |
| **`@RegisterReflectionForBinding` / hints** | Declare reflection your code needs |
| **Buildpacks / native build tools** | Produce the native binary / container image |
| **Testing in native** | Run tests in native mode to catch issues early |
> ⭐ Spring's **AOT engine** does the heavy lifting — generating most reachability metadata automatically so your Spring app "just works" natively in common cases. You add **hints** only for custom reflection. ⚠️ Still **test in native** (some libraries/features differ); slow builds belong in CI, not the inner dev loop (use the JVM for daily dev, native for the release artifact).

---

## 5. Trade-offs Summary & Alternatives

| Benefit | Cost |
|---------|------|
| Instant startup (ms) | Slow build (minutes) |
| Low memory footprint | Reflection/dynamic config needed (§3) |
| Small self-contained binary | Lower peak throughput (sometimes) |
| Great for serverless/scaling | Harder debugging/profiling; library compatibility risk |
> ⭐ **Alternatives that reduce startup/memory without going native:** **CDS/AppCDS** (class-data sharing), **Project Leyden** (emerging AOT for the JVM), and **Project CRaC** (Coordinated Restore at Checkpoint — snapshot a warmed JVM and restore in ms). And **virtual threads** (Phase 1.10/16.1) address concurrency, not startup. Choose native deliberately; it's not a default. This is an **awareness-level** topic — know *what* it is and *when* to reach for it.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Assuming native is always faster | JVM JIT often wins at *peak* throughput (§1) |
| Native for a steady long-running service | JVM is usually simpler/faster there (§2) |
| Unconfigured reflection breaks at runtime | Provide hints/metadata; Spring AOT helps (§3/§4) |
| Building native in the dev inner loop | Dev on JVM; build native in CI (§4) |
| Skipping native tests | Test in native mode — libraries differ (§4) |
| Expecting all libraries to work | Check native compatibility/metadata (§3) |
| Ignoring slow build time | Plan CI for it (§5) |
| Forgetting alternatives (CRaC/AppCDS/Leyden) | Consider them if peak matters (§5) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Contrasts the **JVM/JIT** (Phase 1.11); limited by **reflection** (Phase 1.14).
- Big win for **containers** (smaller/faster images — 10.1), **Kubernetes** fast scaling (10.2), **serverless** (16.7 — Lambda cold starts).
- **Spring Boot 3 AOT** generates native config; **buildpacks** build images (10.1).
- Alternatives: **virtual threads** (1.10/16.1 — concurrency), CRaC/AppCDS/Leyden (startup).
- **Observability/APM agents** (9.3) may need native-aware versions.
- Optional/advanced; could optimize a service in **Project 6/7**.

---

## 8. Quick Self-Check Questions

1. How does Native Image (AOT) differ from the JVM (JIT) in compilation, startup, and memory?
2. Why might the JVM have higher *peak* throughput than native?
3. What are the ideal use cases for native images, and where is the JVM better?
4. What is the closed-world assumption, and why does it conflict with reflection?
5. How do you handle reflection/dynamic features in native?
6. How does Spring Boot's AOT support make native viable?
7. Why develop on the JVM but build native in CI?
8. What are the main trade-offs of going native?
9. What alternatives reduce startup/memory without native (CRaC/AppCDS/Leyden)?
10. Why is native especially valuable for serverless?

---

## 9. Key Terms Glossary

- **GraalVM:** high-performance JDK with a native-image AOT compiler.
- **Native Image:** standalone AOT-compiled executable (no JVM at runtime).
- **AOT vs JIT:** ahead-of-time compilation vs runtime just-in-time (Phase 1.11).
- **Closed-world assumption:** all reachable code known at build time.
- **Reachability metadata / hints:** declarations for reflection/proxies/resources.
- **Spring AOT:** build-time processing that generates native config.
- **Buildpacks / native build tools:** produce native binaries/images.
- **Cold start:** startup latency (critical for serverless).
- **CRaC / AppCDS / Project Leyden:** JVM startup-optimization alternatives.

---

*This is the note for **Section 16.5 — GraalVM / Native Image**.*
*Previous section in roadmap: **16.4 Elasticsearch**.*
*Next section in roadmap: **16.6 Advanced Database**.*
