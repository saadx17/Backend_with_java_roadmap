# Load Testing

> **Phase 13 — Performance & Optimization → 13.2 Load Testing**
> Goal: Validate performance under load — tools (JMeter, Gatling, k6), test types (load/stress/spike/soak), key metrics (throughput, latency percentiles p50/p90/p95/p99, error rate), and micro-benchmarking with JMH.

---

## 0. The Big Picture

You can't claim a system is "fast" or "scalable" without **proving it under realistic load**. **Load testing** generates synthetic traffic to measure how your system behaves as concurrency rises — finding the breaking point, the latency curve, and bottlenecks (which you then profile — 13.1).

```
   Load generator (JMeter/Gatling/k6) ──N concurrent users──► Your system ──► measure:
        throughput (req/s), latency percentiles (p95/p99), error rate, resource saturation (9.2)
   Increase load → watch where latency/errors spike → that's your capacity limit
```

> Pairs with application performance (13.1 — load tests reveal bottlenecks to profile) and scalability (13.3 — prove horizontal scaling works). Uses observability metrics (9.2 — measure during the test), runs in CI/CD (10.3 — performance regression gates), and validates SLOs.

---

## 1. Why & When to Load Test

| Goal | Question answered |
|------|-------------------|
| **Capacity planning** | How many users/req-per-sec can we handle? |
| **Find bottlenecks** | Where does it break first (DB? pool? CPU?)? |
| **Validate SLOs** | Does p95 latency stay < target at expected load? |
| **Pre-release safety** | Will the new version regress performance? (CI gate — §6) |
| **Validate scaling** | Does adding instances actually increase throughput? (13.3) |
> ⭐ Test against a **production-like environment** with **realistic data volumes** and **realistic traffic patterns** (think time, mix of endpoints). ⚠️ Testing on a laptop against an empty DB tells you nothing — small data hides missing indexes (Phase 4.4), and a single machine can't simulate real concurrency.

---

## 2. Test Types ⭐

| Type | What | Reveals |
|------|------|---------|
| **Load test** | Expected/normal load over time | Does it meet SLOs at target traffic? |
| **Stress test** | Push **beyond** capacity until it breaks | The breaking point & failure mode (graceful? cascade?) |
| **Spike test** | Sudden sharp traffic surge | Recovery from bursts (autoscaling — 10.2/13.3, rate limiting — 7.1) |
| **Soak / endurance test** ⭐ | Sustained load for **hours/days** | **Memory leaks** (Phase 1.11), resource exhaustion, slow degradation |
| **Scalability test** | Increase load + instances together | Linear scaling? (13.3) |
| **Volume test** | Large data volumes | DB performance at scale (Phase 4.4) |
```
   Load  → "normal day"           Stress → "find the wall"
   Spike → "flash sale surge"     Soak   → "leak hunting over 24h"
```
> ⭐ **Soak tests catch what short tests miss**: a tiny memory leak or a connection that's never returned to the pool (13.1 §3) only shows up over hours. **Stress tests** verify you fail *gracefully* (circuit breakers/back-pressure — 12.3/11.1), not catastrophically.

---

## 3. Metrics That Matter ⭐

| Metric | Meaning |
|--------|---------|
| **Throughput** | Requests/transactions per second (RPS/TPS) |
| **Latency percentiles** ⭐ | p50/p90/p95/p99/p99.9 response times |
| **Error rate** | % of failed requests (5xx, timeouts) |
| **Concurrency** | Simultaneous users/connections |
| **Saturation** | CPU/memory/pool/DB usage during the test (9.2) |

### 3.1 Why percentiles, not averages ⭐
```
   Averages LIE. Example latencies (ms): 10,10,10,10,10,10,10,10,10,2000
   average ≈ 209ms  ← hides reality
   p50 = 10ms   p90 = 10ms   p99 = 2000ms  ← the truth: 1% of users wait 2s!
```
- **p95** = 95% of requests are faster than this (5% are slower). **p99** = the slow tail.
- ⭐ **Tail latency matters**: at scale, a request often fans out to many services (12.2), so a p99 in one service hits *many* user requests. SLOs are defined on **percentiles** (e.g., "p95 < 200ms"), never averages.
> ⭐ **Always report percentiles** (p50/p90/p95/p99). The average is meaningless for latency — it hides the tail that real users feel. This is why Micrometer Timers track percentiles (9.2 §1.2) and `@Timed(percentiles=...)`.

---

## 4. Tools: JMeter, Gatling, k6 ⭐

| Tool | Style | Note |
|------|-------|------|
| **Apache JMeter** | GUI + XML test plans (Java) | Mature, feature-rich, GUI-driven; can be resource-heavy per thread; widely used |
| **Gatling** ⭐ | Code (Scala/Java DSL) | High-performance (async/non-blocking → many virtual users per machine), great HTML reports, code = version-controlled |
| **k6** ⭐ | Code (JavaScript), Go-powered | Developer-friendly, scriptable, CI-friendly, cloud option; modern favorite |

```javascript
// k6 example — define load profile + thresholds (SLO as code)
import http from 'k6/http';
import { check } from 'k6';
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // ramp to 100 virtual users
    { duration: '5m', target: 100 },   // stay (load test)
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: { http_req_duration: ['p(95)<200'] },  // ⭐ FAIL if p95 ≥ 200ms (SLO gate)
};
export default function () {
  const res = http.get('https://api.example.com/orders');
  check(res, { 'status 200': (r) => r.status === 200 });
}
```
```scala
// Gatling — scenario DSL (excerpt)
setUp(scenario("orders").exec(http("get").get("/orders"))
   .injectOpen(rampUsers(100).during(120))).protocols(httpProtocol)
```
> ⭐ **Code-based tools (Gatling, k6)** are preferred for modern workflows — tests are version-controlled, reviewed (Phase 0.4), and run in CI (§6) with **thresholds as pass/fail gates**. JMeter remains common, especially with GUI-built plans. Generate load from a **separate machine** (a saturated load generator gives false numbers).

---

## 5. JMH (Java Microbenchmark Harness) ⭐

Load testing measures the **system**; **JMH** measures **a piece of Java code** (a method/algorithm) accurately. Naive microbenchmarks (`System.nanoTime()` in a loop) are **wrong** because of JIT warmup, dead-code elimination, and optimizations (Phase 1.11).
```java
@Benchmark
public int sumLoop() {                         // JMH handles warmup, iterations, isolation
    int s = 0; for (int i = 0; i < 1000; i++) s += i; return s;   // return → avoid dead-code elimination
}
// @Warmup, @Measurement, @Fork, @BenchmarkMode control the run; Blackhole consumes results
```
| JMH handles | Why it matters |
|-------------|----------------|
| **Warmup iterations** | Let the JIT compile/optimize before measuring (Phase 1.11) |
| **Dead-code elimination** | `Blackhole`/returns so the JVM doesn't optimize your code away |
| **Forking** | Separate JVMs to avoid profile pollution |
| **Statistics** | Reports with error margins |
> ⭐ Use **JMH** to compare two algorithm/implementation choices (e.g., `StringBuilder` vs `String +`, two collection choices — Phase 1.6/2.2). ⚠️ **Don't hand-roll microbenchmarks** — JIT warmup/optimizations make naive timings meaningless. JMH is for *code-level* questions; load testing is for *system-level* ones.

---

## 6. Load Testing in CI/CD & Methodology

```
1. Define SLOs (p95 < 200ms, error rate < 0.1% at 100 RPS)
2. Build realistic scenarios (endpoint mix, think time, data volume)
3. Run from a dedicated load generator vs a prod-like env
4. Measure percentiles + saturation (9.2); profile bottlenecks (13.1)
5. Set thresholds as PASS/FAIL gates in CI (10.3) → catch regressions
6. Fix → re-run → repeat
```
> ⭐ Wire a **smoke-level performance test with thresholds** into CI/CD (10.3) so a PR that regresses p95 **fails the build** (like the SonarQube gate — 10.3). Full stress/soak tests run on a schedule against staging.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Reporting averages | Report percentiles p95/p99 (§3.1) |
| Testing on a laptop / empty DB | Prod-like env + realistic data volume (§1) |
| Saturated load generator | Generate load from a separate, capable machine (§4) |
| No warmup → cold JVM skews results | Warm up before measuring (JIT — §5, Phase 1.11) |
| Hand-rolled microbenchmarks | Use JMH (§5) |
| Only load tests, never soak | Soak to catch leaks/exhaustion (§2) |
| Unrealistic traffic (no think time/mix) | Model real user behavior (§1) |
| No performance gate in CI | Add threshold-based gates (§6, 10.3) |
| Not measuring saturation | Watch CPU/pool/DB during tests (§3, 9.2) |
| Stress test without checking failure mode | Verify graceful degradation (§2, 12.3) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Reveals bottlenecks to **profile & fix** (13.1 — DB/N+1/pool/GC).
- Measures via **observability metrics** (9.2 — percentiles, saturation); SLOs on percentiles.
- Validates **scalability** (13.3 — does adding instances raise throughput?) and **resilience** (12.3 — stress/spike behavior).
- **CI/CD** (10.3) runs threshold-gated perf tests; runs against **K8s** staging (10.2).
- **JMH** answers code-level questions (collections — 1.6/2.2; strings — 1.4; JIT — 1.11).
- Validates **rate limiting/autoscaling** (7.1/10.2) under spikes.
- Used in **Projects 5/7** to prove performance targets.

---

## 9. Quick Self-Check Questions

1. Why load test, and what does testing on a laptop/empty DB miss?
2. Differentiate load, stress, spike, soak, and volume tests — what does each reveal?
3. Why do soak tests catch problems short tests don't?
4. Why are averages misleading for latency, and what do p95/p99 mean?
5. Why does tail latency matter more in microservices?
6. Compare JMeter, Gatling, and k6 — why prefer code-based tools?
7. What is a threshold, and how does it become a CI gate?
8. What is JMH for, and why can't you hand-roll microbenchmarks reliably?
9. Why generate load from a separate machine?
10. Describe a load-testing methodology tied to SLOs.

---

## 10. Key Terms Glossary

- **Load testing:** measuring system behavior under synthetic traffic.
- **Load / stress / spike / soak / volume test:** normal / beyond-capacity / surge / sustained / large-data.
- **Throughput (RPS/TPS):** requests per second.
- **Latency percentile (p50/p90/p95/p99):** response-time distribution points.
- **Tail latency:** the slow high-percentile requests.
- **Error rate / saturation / concurrency:** failures / resource usage / simultaneous load.
- **JMeter / Gatling / k6:** load-testing tools.
- **Virtual users / ramp / stages:** simulated concurrency over time.
- **Threshold / SLO:** pass-fail gate / service-level objective.
- **JMH / warmup / Blackhole / dead-code elimination:** Java microbenchmark harness & its concerns.

---

*This is the note for **Section 13.2 — Load Testing**.*
*Previous section in roadmap: **13.1 Application Performance**.*
*Next section in roadmap: **13.3 Scalability**.*
