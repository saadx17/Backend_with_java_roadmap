# Reactive Programming (Project Reactor & Spring WebFlux)

> **Phase 16 — Advanced Topics → 16.1 Reactive Programming**
> Goal: Understand reactive programming — Project Reactor (`Mono`/`Flux`, operators, backpressure, schedulers, error handling, hot vs cold), Spring WebFlux, R2DBC, and reactive security.

---

## 0. The Big Picture

Traditional Spring MVC is **thread-per-request, blocking**: each request holds a thread, which sits idle while waiting on I/O (DB, remote calls). Under high concurrency that's wasteful — threads are expensive (Phase 1.10). **Reactive programming** is an **asynchronous, non-blocking, event-driven** style where a **few threads** handle **many** concurrent requests by never blocking — they react to data as it arrives, with **backpressure** to avoid overwhelming consumers.

```
   BLOCKING (MVC):  1 request = 1 thread, BLOCKED while waiting on DB → 200 threads = 200 reqs
   REACTIVE (WebFlux): few event-loop threads, NEVER block → thousands of concurrent reqs on a few threads
```

> Builds on concurrency (Phase 1.10), functional programming (1.8 — lambdas/streams), and messaging back-pressure (11.1 §7). Powers **Spring Cloud Gateway** (12.4) and **WebClient** (5.9/12.2). ⚠️ It's powerful but **complex** — and **Java 21 virtual threads** (1.10/13.1) now offer much of the scalability benefit with **simpler blocking code**, so reactive is no longer the only answer.

---

## 1. Why Reactive? (the problem it solves)

| | Blocking (MVC) | Reactive (WebFlux) |
|--|----------------|--------------------|
| Model | Thread-per-request | Event loop, non-blocking |
| Threads under load | Many (one per in-flight request) | Few (handle thousands of requests) |
| Best for | CPU-bound, simple, moderate load | **High-concurrency I/O-bound**, streaming |
| Resource use | Higher (thread stacks, context switches) | Lower per request |
| Complexity | ⭐ Simple, easy to debug | High (operators, async stack traces) |
| Backpressure | None (can overwhelm) | ✅ Built-in (Reactive Streams) |
> ⭐ Reactive shines for **I/O-bound, high-concurrency, streaming** workloads (gateways, aggregators, many slow downstreams). ⚠️ For typical CRUD apps it adds complexity for little gain — and **virtual threads (Java 21)** give blocking code reactive-like scalability without the paradigm shift (1.10/13.1). Choose reactive deliberately.

---

## 2. Reactive Streams & Project Reactor

**Reactive Streams** is the spec (`Publisher`/`Subscriber`/`Subscription`/`Processor`) defining async streams **with backpressure** — now part of the JDK as `java.util.concurrent.Flow`. **Project Reactor** is Spring's implementation, providing two core publisher types:
| Type | Emits | Analogy |
|------|-------|---------|
| **`Mono<T>`** | **0 or 1** element | An async `Optional`/single value (e.g., `findById`) |
| **`Flux<T>`** | **0..N** elements (a stream) | An async `Stream`/`List` |
```java
Mono<User> user = userRepository.findById(id);            // 0..1 (R2DBC — §6)
Flux<Order> orders = orderRepository.findByUserId(id);    // 0..N stream

// Nothing runs until SUBSCRIBED — publishers are lazy blueprints
user.subscribe(u -> System.out.println(u));               // subscription triggers execution
```
> ⭐ **Reactive publishers are lazy** — declaring a `Mono`/`Flux` builds a pipeline blueprint; **nothing executes until something subscribes** (in WebFlux, the framework subscribes for you). This is the #1 beginner gotcha: "my code didn't run" = "you forgot to subscribe / return it."

---

## 3. Operators ⭐

You build pipelines by chaining **operators** (like Streams — Phase 1.8, but async). Each returns a new publisher.
```java
Flux.fromIterable(orderIds)                       // source
    .flatMap(id -> orderService.findById(id))      // async map (concurrent), flattens Mono<Order>
    .filter(order -> order.getTotal() > 100)        // keep some
    .map(OrderMapper::toDto)                         // transform (sync) → DTO (7.3)
    .take(10)                                        // limit
    .collectList();                                  // Flux<OrderDto> → Mono<List<OrderDto>>
```
| Operator group | Examples |
|----------------|----------|
| **Transform** | `map` (sync 1:1), `flatMap` (async, returns a publisher), `cast` |
| **Filter** | `filter`, `take`, `skip`, `distinct` |
| **Combine** | `zip` (combine in parallel), `merge`, `concat`, `flatMap` |
| **Reduce** | `reduce`, `collectList`, `count` |
| **Peek/side-effect** | `doOnNext`, `doOnError`, `doOnComplete`, `log` |
| **Error** | `onErrorReturn`, `onErrorResume`, `retry` (§5) |
| **Time** | `delayElements`, `timeout`, `interval` |
> ⭐ **`map` vs `flatMap`** is the key distinction: `map` transforms a value synchronously (1:1); **`flatMap` is for async calls** that themselves return a `Mono`/`Flux` (it subscribes and flattens). Calling a reactive repo/WebClient inside a pipeline → `flatMap`. ⚠️ **Never call `.block()`** inside a reactive pipeline (defeats the purpose, can deadlock the event loop).

---

## 4. Backpressure & Schedulers

### 4.1 Backpressure
The subscriber signals **how much** it can handle (`request(n)`), so a fast producer can't overwhelm a slow consumer (recall 11.1 §7). Strategies when overwhelmed: **buffer, drop, latest, error**.
```java
flux.onBackpressureBuffer(1000)   // buffer up to 1000, then error  (or onBackpressureDrop/Latest)
```
> ⭐ **Backpressure is reactive's superpower** — built-in flow control end-to-end (unlike plain blocking code). It's the same concept as messaging back-pressure (11.1) and rate limiting (7.1), at the stream level.

### 4.2 Schedulers (threading)
Reactor is concurrency-agnostic; **Schedulers** control which threads work runs on (`publishOn`/`subscribeOn`).
| Scheduler | For |
|-----------|-----|
| `Schedulers.parallel()` | CPU-bound work (fixed pool) |
| `Schedulers.boundedElastic()` | ⭐ **Blocking** I/O (wrap unavoidable blocking calls here so the event loop stays free) |
| `Schedulers.single()` / `immediate()` | Single thread / current |
```java
Mono.fromCallable(() -> legacyBlockingDao.load(id))   // a blocking call...
    .subscribeOn(Schedulers.boundedElastic());         // ...isolated off the event-loop threads
```
> ⚠️ ⭐ **Never block the event-loop threads.** If you must call blocking code (a legacy JDBC DAO, a blocking lib), isolate it on `boundedElastic`. One blocked event-loop thread can stall many requests. (BlockHound can detect accidental blocking in tests.)

---

## 5. Error Handling & Hot vs Cold

### 5.1 Error handling
Errors are **terminal signals** in a reactive stream (they end it) — handle them with operators, not try/catch:
```java
orderService.findById(id)
    .onErrorResume(NotFoundException.class, e -> Mono.empty())   // fallback to empty
    .onErrorReturn(fallbackOrder)                                // or a default
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200)))         // retry transient (12.3)
    .timeout(Duration.ofSeconds(2));                             // deadline
```
| Operator | Does |
|----------|------|
| `onErrorReturn` | Emit a fallback value |
| `onErrorResume` | Switch to an alternative publisher |
| `onErrorMap` | Translate the exception |
| `retry` / `retryWhen` | Re-subscribe on error (idempotent only — 12.3) |
| `doOnError` | Side-effect (log — 9.1) without handling |
> ⚠️ A `try/catch` won't catch errors in an async pipeline — they travel as the **error signal**. ⚠️ **MDC/trace context** doesn't follow threads automatically in reactive (recall 9.1 §3) — use Reactor's `Context` / Micrometer context propagation for `traceId` (12.7).

### 5.2 Hot vs Cold publishers
| | **Cold** ⭐ | **Hot** |
|--|-----------|---------|
| Behavior | Each subscriber gets the **full sequence from the start** (re-executes) | Emits regardless of subscribers; late subscribers **miss** earlier items |
| Example | `Flux.range`, a DB query (re-runs per subscribe) | A live event stream, `Sinks`, shared broadcast |
| Analogy | A movie on demand (starts at 0 for everyone) | A live TV broadcast (you see it from when you tune in) |
> ⭐ Most reactive sources (DB queries, HTTP) are **cold** — re-subscribing re-executes. **Hot** publishers (live streams, `Sinks.many()`) broadcast to current subscribers — used for SSE/WebSocket fan-out (16.3) and shared streams. Use `share()`/`cache()` to turn cold into hot.

---

## 6. Spring WebFlux & R2DBC

### 6.1 WebFlux (reactive web)
The reactive alternative to Spring MVC — same annotations or a functional router, returning `Mono`/`Flux`. Runs on **Netty** (event loop) by default.
```java
@RestController
class OrderController {
    @GetMapping("/orders/{id}")
    Mono<OrderDto> get(@PathVariable Long id) {        // return a publisher; WebFlux subscribes
        return orderService.findById(id).map(mapper::toDto);
    }
    @GetMapping(value="/orders/stream", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<OrderEvent> stream() { return orderService.streamEvents(); }   // SSE streaming (16.3)
}
```
> ⚠️ ⭐ **Reactive is end-to-end or not at all** — a reactive controller calling a **blocking** JDBC repo gains nothing (you'd block the event loop). To be truly reactive, the **whole stack** must be non-blocking: WebFlux + WebClient (5.9) + **R2DBC** (§6.2) + reactive Redis, etc.

### 6.2 R2DBC (reactive database access)
JDBC is **blocking** by design. **R2DBC** (Reactive Relational Database Connectivity) is the non-blocking SQL driver spec; Spring Data R2DBC gives reactive repositories.
```java
interface OrderRepository extends ReactiveCrudRepository<Order, Long> {
    Flux<Order> findByUserId(Long userId);   // returns a Flux, non-blocking
}
```
> ⚠️ R2DBC is **not JPA** — no lazy loading, no entity graph, simpler mapping (more like Spring Data JDBC). You trade JPA's conveniences (Phase 5.4) for non-blocking I/O. Many teams keep blocking JPA + virtual threads instead.

### 6.3 Reactive security
**Spring Security** has a reactive variant (`ServerHttpSecurity`, `ReactiveUserDetailsService`, reactive method security) for WebFlux — same concepts as Phase 5.6/15.2 but with reactive types and a non-blocking filter chain.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Forgetting to subscribe/return the publisher | Return it (WebFlux subscribes) — it's lazy (§2) |
| Calling `.block()` in a pipeline | Never; compose with operators (§3) |
| Blocking the event-loop thread | Isolate blocking on `boundedElastic` (§4.2) |
| `map` where `flatMap` is needed (async) | `flatMap` for calls returning publishers (§3) |
| try/catch for async errors | Error operators (`onErrorResume`...) (§5.1) |
| Reactive controller + blocking JPA | Go reactive end-to-end (R2DBC) (§6) |
| Losing traceId/MDC in reactive | Reactor `Context` / context propagation (§5.1, 12.7) |
| Using reactive for simple CRUD | Consider MVC + virtual threads (§1) |
| Expecting hot/cold to behave the same | Know re-execution vs broadcast (§5.2) |
| Hard-to-debug async stack traces | `checkpoint()`, `log()`, Reactor debug agent (§3) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Builds on **concurrency** (1.10 — and **virtual threads** as the simpler alternative), **functional** (1.8 — operators), **back-pressure** (11.1).
- **WebClient** (5.9/12.2) and **Spring Cloud Gateway** (12.4) are reactive.
- **R2DBC** vs **JPA** (5.4); **reactive Redis** (4.8); **reactive security** (5.6/15.2).
- **SSE/WebSocket** streaming (16.3) uses Flux/hot publishers.
- **Observability** context propagation (12.7); **performance** for I/O-bound concurrency (13.1).
- ⚠️ Weigh against **virtual threads** (13.1) — often the pragmatic choice now.
- Optional/advanced for **Project 7** (high-concurrency services).

---

## 9. Quick Self-Check Questions

1. What problem does reactive programming solve vs thread-per-request blocking?
2. When is reactive worth it, and how do virtual threads change the calculus?
3. What are `Mono` and `Flux`, and why are publishers lazy?
4. `map` vs `flatMap` — when use each? Why never `.block()` in a pipeline?
5. What is backpressure, and what strategies handle overflow?
6. What do schedulers do, and why isolate blocking calls on `boundedElastic`?
7. How is error handling done reactively (not try/catch)?
8. Cold vs hot publishers — difference and examples?
9. Why must reactive be end-to-end, and what is R2DBC (vs JPA)?
10. What happens to MDC/traceId in reactive, and how do you fix it?

---

## 10. Key Terms Glossary

- **Reactive programming:** async, non-blocking, event-driven, backpressured.
- **Reactive Streams / `Flow`:** the spec (Publisher/Subscriber/Subscription).
- **Project Reactor / `Mono` / `Flux`:** Spring's impl / 0..1 / 0..N publishers.
- **Operator (map/flatMap/filter/zip...):** composable stream transformation.
- **Lazy / subscribe:** nothing runs until subscribed.
- **Backpressure (buffer/drop/latest/error):** consumer-controlled flow control.
- **Scheduler (parallel/boundedElastic):** thread control; isolate blocking.
- **Error signal / onErrorResume / retryWhen:** terminal error / handling operators.
- **Hot vs cold publisher:** broadcast vs per-subscriber re-execution.
- **WebFlux / Netty:** reactive web framework / event-loop server.
- **R2DBC:** reactive (non-blocking) relational DB access.
- **Virtual threads:** Java 21 alternative for cheap blocking concurrency.

---

*This is the note for **Section 16.1 — Reactive Programming**.*
*Previous section in roadmap: **15.4 Dependency Security**.*
*Next section in roadmap: **16.2 GraphQL**.*
