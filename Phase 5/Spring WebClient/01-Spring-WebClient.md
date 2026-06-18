# Spring WebClient (Reactive HTTP Client)

> **Phase 5 — Spring Framework & Spring Boot → 5.9 Spring WebClient**
> Goal: Master making outbound HTTP calls from Spring — WebClient configuration, the HTTP methods, error handling, timeouts, retry, connection pooling — and how it compares to RestTemplate (and RestClient).

---

## 0. The Big Picture

Backends often need to **call other services** over HTTP — third-party APIs, microservices (Phase 12), payment gateways. **WebClient** is Spring's modern HTTP client for making these **outbound** calls. (Recall HTTP, Phase 0.2 — now your app is the *client*.)

```
Your Spring app  --HTTP request-->  another API/service  --response-->  your app
                 (made via WebClient)
```

> Where Spring MVC (Phase 5.3) handles *incoming* requests, WebClient makes *outgoing* ones. Essential for microservices (Phase 12) and integrating external APIs (payment, email, etc. — Projects 5, 7).

---

## 1. WebClient vs RestTemplate vs RestClient

Spring has had several HTTP clients; know the landscape:
| Client | Style | Status |
|--------|-------|--------|
| **`RestTemplate`** | Blocking, synchronous | **Legacy** (maintenance mode — not removed, but no new features) |
| **`WebClient`** | Reactive, non-blocking (also usable blocking) | **Recommended** — the future |
| **`RestClient`** (Spring 6.1+) | Blocking, synchronous, fluent | Modern blocking option (WebClient's fluent API, sync) |
> ⚠️ **`RestTemplate` is legacy** (in maintenance mode) — Spring recommends **`WebClient`** going forward. WebClient is **reactive/non-blocking** (built on Project Reactor — Phase 16.1) but can be used in a blocking style too. If you want a *simple blocking* client without reactive types, the newer **`RestClient`** (Spring 6.1+) offers WebClient's fluent API synchronously. (The roadmap names WebClient as "the future.")

---

## 2. Creating a WebClient

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")          // base URL for all calls
            .defaultHeader(HttpHeaders.CONTENT_TYPE, "application/json")
            .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer ...")  // recall auth headers, Phase 0.2
            .build();
    }
}
```
> Configure a `WebClient` as a **bean** (Phase 5.1) and inject it where needed. Set common config (base URL, default headers, timeouts §6) once. Build via `WebClient.builder()`.

---

## 3. Making Requests (GET, POST, PUT, DELETE)

WebClient uses a **fluent API** chaining the method, URI, body, and response handling (recall HTTP methods, Phase 0.2):
```java
// GET -> deserialize JSON to an object:
User user = webClient.get()
    .uri("/users/{id}", id)                              // path variable
    .retrieve()                                          // execute & get the response
    .bodyToMono(User.class)                              // body -> Mono<User> (reactive — Phase 16.1)
    .block();                                            // block to get the value (sync usage)

// GET a list:
List<User> users = webClient.get().uri("/users")
    .retrieve()
    .bodyToFlux(User.class)                              // Flux = 0..N elements (Phase 16.1)
    .collectList().block();

// POST with a body:
User created = webClient.post()
    .uri("/users")
    .bodyValue(new CreateUserRequest("Alice", "alice@x.com"))   // serialize to JSON
    .retrieve()
    .bodyToMono(User.class)
    .block();

// PUT / DELETE:
webClient.put().uri("/users/{id}", id).bodyValue(update).retrieve().bodyToMono(User.class).block();
webClient.delete().uri("/users/{id}", id).retrieve().toBodilessEntity().block();
```
| Step | Does |
|------|------|
| `.get()`/`.post()`/etc. | The HTTP method (Phase 0.2) |
| `.uri(...)` | The path (+ path variables/query params) |
| `.bodyValue(obj)` | Set the request body (serialized to JSON via Jackson) |
| `.retrieve()` | Execute the request |
| `.bodyToMono(T)` / `.bodyToFlux(T)` | Deserialize the body to a `Mono` (0/1) / `Flux` (0..N) |
| `.block()` | Block to get the result synchronously |
> ⚠️ **`Mono<T>` (0 or 1 element) and `Flux<T>` (0..N)** are reactive types from **Project Reactor** (Phase 16.1). Calling **`.block()`** turns the async result into a synchronous value — fine in a blocking (MVC) app, but in a **fully reactive** app (WebFlux — Phase 16.1) you should **never block** (it defeats the purpose). Recall `Optional`/`CompletableFuture` (Phase 1.8/1.10) — `Mono` is conceptually similar (an async, possibly-empty value).

---

## 4. Error Handling

By default, `.retrieve()` throws a `WebClientResponseException` for 4xx/5xx responses (recall status codes, Phase 0.2). Handle errors explicitly:
```java
User user = webClient.get().uri("/users/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError,          // handle 4xx (Phase 0.2)
        resp -> Mono.error(new UserNotFoundException(id)))
    .onStatus(HttpStatusCode::is5xxServerError,          // handle 5xx
        resp -> Mono.error(new ServiceUnavailableException()))
    .bodyToMono(User.class)
    .onErrorResume(WebClientResponseException.NotFound.class,
        ex -> Mono.empty())                              // fall back on 404 (Phase 16.1 operators)
    .block();
```
> Map HTTP error statuses to your own exceptions (then handled by `@RestControllerAdvice` — Phase 5.3). `onErrorResume`/`onErrorReturn` (reactive operators — Phase 16.1) provide fallbacks. ⚠️ **Calling other services means new failure modes** (timeouts, 5xx, network errors — recall Phase 0.2) — always handle them (this is what resilience patterns address — Phase 12.3).

---

## 5. Timeouts (Essential!)

> ⚠️ **Always set timeouts on outbound calls.** Without a timeout, a slow/hung downstream service blocks your thread indefinitely → cascading failures (recall blocking I/O, Phase 0.1; thread exhaustion, Phase 1.10).
```java
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .responseTimeout(Duration.ofSeconds(5))                       // overall response timeout
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);          // connection timeout
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
// Or per-request:
.retrieve().bodyToMono(User.class).timeout(Duration.ofSeconds(5)).block();
```
> A hung downstream call without a timeout can exhaust your thread pool and bring down your *whole* service (a cascading failure). Timeouts are the **first line of resilience** (Phase 12.3 — `TimeLimiter`).

---

## 6. Retry

For transient failures (a brief network blip, a 503), **retry** with backoff:
```java
.retrieve()
.bodyToMono(User.class)
.retryWhen(Retry.backoff(3, Duration.ofMillis(500))    // up to 3 retries, exponential backoff
    .filter(ex -> ex instanceof WebClientResponseException.ServiceUnavailable))  // only retry transient errors
.block();
```
> ⚠️ Retry **only idempotent operations** (GET/PUT/DELETE — recall idempotency, Phase 0.2) — retrying a POST could create duplicates! Use **exponential backoff with jitter** to avoid hammering a struggling service (recall Phase 12.3). Don't retry on 4xx (client errors won't succeed on retry). Retry is a core resilience pattern (Phase 12.3 — Resilience4j).

---

## 7. Connection Pooling

WebClient (via Reactor Netty) **pools connections** by default (recall connection pooling, Phase 4.7/4.6 — reuse expensive connections rather than opening one per call):
```java
ConnectionProvider provider = ConnectionProvider.builder("custom")
    .maxConnections(50)                          // max pooled connections
    .pendingAcquireTimeout(Duration.ofSeconds(5))
    .build();
HttpClient httpClient = HttpClient.create(provider);
```
> Connection pooling avoids the TCP/TLS handshake cost (Phase 0.2) on every call — critical for performance when calling a service frequently. Tune `maxConnections` like a thread/DB pool (Phase 1.10/4.7).

---

## 8. Putting It Together (a service client)
```java
@Service
public class PaymentClient {
    private final WebClient webClient;
    public PaymentClient(WebClient webClient) { this.webClient = webClient; }   // injected (Phase 5.1)

    public PaymentResult charge(ChargeRequest request) {
        return webClient.post()
            .uri("/charges")
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatusCode::isError,
                resp -> resp.bodyToMono(String.class).map(PaymentException::new))
            .bodyToMono(PaymentResult.class)
            .timeout(Duration.ofSeconds(5))                    // timeout (§5)
            .retryWhen(Retry.backoff(2, Duration.ofMillis(300)))  // retry (§6)
            .block();
    }
}
```

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `RestTemplate` for new code | Use `WebClient` (or `RestClient` for simple blocking) |
| No timeout on outbound calls | Always set timeouts (cascading failure risk) |
| Retrying non-idempotent POSTs | Only retry idempotent operations (Phase 0.2) |
| Retry without backoff | Use exponential backoff + jitter |
| Not handling 4xx/5xx | Map errors to exceptions (`onStatus`) |
| `.block()` in a reactive (WebFlux) app | Never block in fully reactive code (Phase 16.1) |
| New WebClient per request | Reuse a configured bean (connection pooling) |
| Ignoring downstream failures | Add resilience (timeout/retry/circuit breaker — Phase 12.3) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Microservice communication** (Phase 12.2) — WebClient (or OpenFeign) for service-to-service REST calls.
- **External API integration** — payment (Stripe — Projects 5, 7), email, third-party data.
- **Resilience patterns** (Phase 12.3): timeouts/retry here are the basics; circuit breakers, bulkheads, rate limiters (Resilience4j) build on them.
- **Reactive types** (`Mono`/`Flux`) connect to **Spring WebFlux / Project Reactor** (Phase 16.1).
- **Idempotency** (Phase 0.2/7) determines what's safe to retry.
- **Connection pooling** parallels DB pooling (Phase 4.7) — reuse expensive connections.
- **Distributed tracing** (Phase 12.7) propagates trace context across WebClient calls.

---

## 11. Quick Self-Check Questions

1. What is WebClient for, and how does it relate to Spring MVC?
2. How does WebClient compare to RestTemplate and RestClient?
3. How do you make a GET and a POST request with WebClient?
4. What are `Mono` and `Flux`, and what does `.block()` do (and when must you NOT use it)?
5. How do you handle 4xx/5xx errors?
6. Why are timeouts essential on outbound calls?
7. When is it safe to retry, and how should you retry?
8. Why does connection pooling matter for an HTTP client?

---

## 12. Key Terms Glossary

- **WebClient:** Spring's modern (reactive) HTTP client for outbound calls.
- **RestTemplate / RestClient:** legacy blocking / modern fluent blocking clients.
- **`retrieve()` / `bodyToMono` / `bodyToFlux`:** execute / deserialize to 0-1 / 0-N.
- **`Mono<T>` / `Flux<T>`:** reactive single / multi-element types (Project Reactor — Phase 16.1).
- **`.block()`:** convert a reactive result to synchronous (avoid in reactive apps).
- **`onStatus` / `onErrorResume`:** error handling for HTTP status / exceptions.
- **Timeout:** max wait for a response (resilience essential).
- **Retry / backoff:** re-attempt transient failures with increasing delay.
- **Connection pooling:** reuse HTTP connections (performance).
- **Idempotency:** safe-to-retry property (GET/PUT/DELETE).

---

*This is the note for **Section 5.9 — Spring WebClient**.*
*This completes **Section 5.x core Spring** — Phase 5 sections 5.1–5.9 are done.*
*Next: **Project 4/5 (REST API & E-Commerce Backend)**, then **Phase 6 — Testing**.*
