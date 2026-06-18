# Resilience4j (Fault Tolerance)

> **Phase 12 — Microservices → 12.3 Resilience4j**
> Goal: Master fault-tolerance patterns with Resilience4j — Circuit Breaker, Retry, Rate Limiter, Bulkhead, Timeout (TimeLimiter) — and their Spring Boot integration.

---

## 0. The Big Picture

In a distributed system, **failure is normal** (12.1/12.2): a downstream service is slow, down, or overloaded. Without protection, one slow dependency exhausts your threads and **cascades** into a system-wide outage. **Resilience4j** is a lightweight fault-tolerance library (the modern successor to Netflix Hystrix) providing composable patterns to **fail fast, recover, and isolate**.

```
   Order Service ──calls──► Payment Service (slow/down)
   Without resilience: every request waits → threads pile up → Order Service ALSO dies → CASCADE 💥
   With resilience:    circuit breaker trips → fail fast / fallback → Order stays alive ✅
```

> Directly protects the inter-service calls from 12.2 (WebClient/Feign). Builds on timeouts/idempotency (Phase 7.1), connects to the API Gateway (12.4 — rate limiting), observability (12.7 — emits metrics), and Kubernetes (10.2 — autoscaling complements it). Resilience4j is also a **design pattern** (Circuit Breaker — Phase 14.1).

---

## 1. The Five Core Patterns

| Pattern | Protects against | One-liner |
|---------|------------------|-----------|
| **Circuit Breaker** | Cascading failure from a failing dependency | Stop calling a broken service; fail fast |
| **Retry** | Transient/intermittent failures | Try again (with backoff) before giving up |
| **Rate Limiter** | Overload / quota | Cap the rate of calls |
| **Bulkhead** | Resource exhaustion | Isolate concurrent calls so one dependency can't sink everything |
| **TimeLimiter (Timeout)** | Hanging calls | Give up after a deadline |

> ⭐ These **compose**: e.g., `TimeLimiter` → `CircuitBreaker` → `Retry` → `Bulkhead` around one call. Order matters (§7). Each is independently configurable and emits metrics (12.7).

---

## 2. Circuit Breaker ⭐

Like an electrical breaker: after too many failures, it **"opens"** and **fails fast** (no more calls to the broken dependency) for a cool-down, then **tests** recovery.
```
        failures exceed threshold
   CLOSED ───────────────────────► OPEN ──(after wait duration)──► HALF-OPEN
   (calls flow)                     (fail fast,                    (allow a few test calls)
        ▲                            return fallback)                    │
        └──────── test calls succeed ───────────────────────────────────┘
                  test calls fail → back to OPEN
```
| State | Behavior |
|-------|----------|
| **CLOSED** | Normal — calls pass; failures counted |
| **OPEN** | Trips after failure-rate threshold → **fail fast** (call fallback, don't hit downstream) |
| **HALF-OPEN** | After a wait, allow limited test calls; success → CLOSED, failure → OPEN |
```java
@CircuitBreaker(name = "payment", fallbackMethod = "chargeFallback")
public PaymentResult charge(ChargeRequest req) {
    return paymentClient.charge(req);          // (WebClient/Feign — 12.2)
}
PaymentResult chargeFallback(ChargeRequest req, Throwable t) {
    return PaymentResult.pending();            // graceful degradation
}
```
```yaml
resilience4j.circuitbreaker.instances.payment:
  sliding-window-size: 20
  failure-rate-threshold: 50          # open if ≥50% of last 20 calls fail
  wait-duration-in-open-state: 10s    # cool-down before half-open
  permitted-number-of-calls-in-half-open-state: 3
  slow-call-duration-threshold: 2s    # count slow calls as failures
```
> ⭐ The breaker gives **fail-fast + graceful degradation** (the **fallback**): instead of hanging, return a cached value, a default, or a "try later" — keeping *your* service alive. This is the single most important resilience pattern for sync calls.

---

## 3. Retry

Transient failures (a brief network blip, a momentary 503) often succeed on a second try. **Retry** re-attempts a failed call a limited number of times, ideally with **exponential backoff + jitter**.
```yaml
resilience4j.retry.instances.payment:
  max-attempts: 3
  wait-duration: 500ms
  enable-exponential-backoff: true
  exponential-backoff-multiplier: 2     # 500ms, 1s, 2s...
  retry-exceptions: [java.io.IOException]
```
```java
@Retry(name = "payment", fallbackMethod = "chargeFallback")
public PaymentResult charge(ChargeRequest req) { ... }
```
> ⚠️ **Only retry idempotent operations** (recall idempotency — Phase 7.1; GET/PUT safe, POST needs an idempotency key)! Retrying a non-idempotent `POST /payments` could **double-charge**. ⚠️ Add **jitter** to avoid a **retry storm** (all clients retrying in lockstep, hammering a recovering service). Don't retry on 4xx (client errors won't fix themselves).

---

## 4. Rate Limiter

Caps how many calls are permitted per time window — protecting *you* from overwhelming a downstream, or enforcing a quota. (Distinct from API-gateway rate limiting for inbound traffic — 12.4 / Phase 7.1 §5; this is for outbound/internal protection.)
```yaml
resilience4j.ratelimiter.instances.payment:
  limit-for-period: 100        # 100 calls...
  limit-refresh-period: 1s     # ...per second
  timeout-duration: 0          # fail fast if no permit
```
```java
@RateLimiter(name = "payment")
public PaymentResult charge(...) { ... }
```

---

## 5. Bulkhead

Named after a ship's watertight compartments: **isolate** concurrent calls to one dependency so a flood to it can't consume *all* your threads and sink the whole service.
```
   Without bulkhead: 200 slow calls to Payment use ALL threads → Inventory calls starve too 💥
   With bulkhead:    Payment limited to 25 concurrent → Inventory unaffected ✅ (isolation)
```
| Bulkhead type | Mechanism |
|---------------|-----------|
| **SemaphoreBulkhead** | Limits concurrent calls via a semaphore (no extra threads) |
| **ThreadPoolBulkhead** | Dedicated thread pool per dependency (isolation + offload) |
```yaml
resilience4j.bulkhead.instances.payment:
  max-concurrent-calls: 25
```
> ⭐ Bulkheads provide **failure isolation** between dependencies — a core microservices principle (12.1). One sick dependency degrades only its own slice, not the entire service.

---

## 6. TimeLimiter (Timeout)

Caps how long a call may take; on expiry it's cancelled and treated as a failure (feeding the circuit breaker). Essential — a call with no timeout can hang forever.
```yaml
resilience4j.timelimiter.instances.payment:
  timeout-duration: 2s
  cancel-running-future: true
```
```java
@TimeLimiter(name = "payment")          // works with CompletableFuture / reactive types
public CompletableFuture<PaymentResult> charge(...) { ... }
```
> ⚠️ **Always set timeouts on remote calls** (also at the client level — WebClient `.timeout()`, 12.2). A missing timeout is the most common cause of cascading hangs. The TimeLimiter + CircuitBreaker combo turns a hang into a fast failure + open circuit.

---

## 7. Combining Patterns & Spring Boot Integration

### 7.1 Setup
```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId></dependency>
<dependency><groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId></dependency>
```
- Annotations (`@CircuitBreaker`, `@Retry`, etc.) are AOP-based (recall Spring AOP — Phase 5.1) — they wrap your method.
- Resilience4j **auto-publishes Micrometer metrics** (circuit state, call counts, latency) → Prometheus/Grafana (9.2/12.7) and exposes Actuator health/events.

### 7.2 Order of decorators ⭐
When stacked, the typical (recommended) execution order, outer→inner:
```
   Retry( CircuitBreaker( RateLimiter( TimeLimiter( Bulkhead( call ) ) ) ) )
```
- **Retry outermost** → it re-runs the whole protected call.
- **CircuitBreaker** inside Retry → a tripped breaker fails fast and the retry sees it.
- (Resilience4j applies a default aspect order; you can configure it.)
> ⭐ Combine deliberately: `TimeLimiter` (don't hang) → `CircuitBreaker` (stop calling broken deps) → `Retry` (handle transient blips) → `Bulkhead` (isolate) — each with a **fallback** for graceful degradation. ⚠️ Be careful that Retry × CircuitBreaker don't amplify load on a struggling service.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| No timeouts → hanging calls cascade | TimeLimiter + client timeout (§6, 12.2) |
| Retrying non-idempotent operations | Only retry idempotent calls / use idempotency keys (§3, Phase 7.1) |
| Retry without backoff/jitter → retry storm | Exponential backoff + jitter (§3) |
| Retrying on 4xx | Retry only transient (5xx/IO) errors (§3) |
| No fallback → user sees raw failure | Provide graceful fallbacks (§2) |
| One dependency exhausts all threads | Bulkhead isolation (§5) |
| Breaker thresholds too sensitive/lax | Tune window/threshold/wait (§2) |
| Retry stacked over an open breaker amplifying load | Mind decorator order/interaction (§7) |
| Ignoring resilience metrics | Watch circuit state in Grafana (§7, 12.7) |
| Resilience instead of fixing root cause | Resilience buys time; also fix & autoscale (10.2/13) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- Wraps the **remote calls** from 12.2 (WebClient/OpenFeign — Feign integrates Resilience4j directly).
- **AOP-based** (Phase 5.1); auto **Micrometer metrics** → Prometheus/Grafana + alerts (9.2/12.7).
- **Retry + idempotency** (Phase 7.1) — never double-execute; pairs with messaging at-least-once (Phase 11).
- **Circuit Breaker** is also a GoF-adjacent **design pattern** (Phase 14.1) and an **architecture** concern (14.4).
- **Rate limiting** also at the **gateway** (12.4) for inbound; complements **autoscaling** (HPA — 10.2) & scalability (Phase 13.3).
- **Security** — fail-closed where appropriate (Phase 15).
- Essential in **Project 7 (Microservices E-Commerce)**.

---

## 10. Quick Self-Check Questions

1. Why is fault tolerance essential in microservices, and what is a cascading failure?
2. Name the five Resilience4j patterns and what each protects against.
3. Explain the circuit breaker states (CLOSED/OPEN/HALF-OPEN) and what a fallback does.
4. When is it safe to retry, and why are backoff + jitter important?
5. What's the difference between the rate limiter here and gateway rate limiting (12.4)?
6. What problem does a bulkhead solve, and what are the two types?
7. Why must remote calls always have a TimeLimiter/timeout?
8. What's a sensible order when combining the patterns, and what's the risk of Retry × CircuitBreaker?
9. How does Resilience4j integrate with Spring (AOP, metrics)?
10. Why is resilience a complement to, not a replacement for, fixing root causes and autoscaling?

---

## 11. Key Terms Glossary

- **Resilience4j:** lightweight fault-tolerance library (Hystrix successor).
- **Cascading failure:** one failing dependency taking down callers.
- **Circuit Breaker (CLOSED/OPEN/HALF-OPEN):** trips to fail fast on a broken dependency.
- **Fallback / graceful degradation:** alternative response when a call fails.
- **Retry / backoff / jitter:** re-attempt / increasing delay / randomization.
- **Retry storm:** synchronized retries overwhelming a recovering service.
- **Rate Limiter:** cap calls per window.
- **Bulkhead (semaphore/thread-pool):** isolate concurrency per dependency.
- **TimeLimiter / timeout:** deadline for a call.
- **Sliding window / failure-rate threshold:** circuit breaker tuning.
- **Decorator order:** how stacked resilience patterns nest.

---

*This is the note for **Section 12.3 — Resilience4j**.*
*Previous section in roadmap: **12.2 Inter-Service Communication**.*
*Next section in roadmap: **12.4 API Gateway**.*
