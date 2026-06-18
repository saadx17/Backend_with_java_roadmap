# API Gateway (Spring Cloud Gateway)

> **Phase 12 — Microservices → 12.4 API Gateway**
> Goal: Understand the API Gateway pattern and Spring Cloud Gateway — routes, predicates, filters, rate limiting with Redis, and centralized JWT validation.

---

## 0. The Big Picture

With many microservices (12.1), you don't want clients calling each service directly (they'd need to know every address, handle auth N times, deal with CORS, etc.). An **API Gateway** is a **single entry point** that sits in front of all services and handles **cross-cutting concerns** — routing, authentication, rate limiting, CORS, logging — so the services don't have to.

```
                       ┌──────────────────── API GATEWAY ────────────────────┐
   Clients ──────────► │ auth (JWT) · rate limit · routing · CORS · logging   │
   (web/mobile)        └───┬──────────────┬───────────────┬──────────────────┘
                           ▼              ▼               ▼
                     Order Service   Payment Service  Inventory Service  (12.1/12.2)
```

> The gateway is the edge of your microservices system. It builds on REST (Phase 7), security/JWT (Phase 5.6/15.2), rate limiting (Phase 7.1 §5), resilience (12.3), and service discovery (12.2). On Kubernetes it complements **Ingress** (10.2). It's the front door used in the **Strangler Fig** migration (12.1 §6).

---

## 1. Why an API Gateway? (responsibilities)

| Concern | Without a gateway | With a gateway |
|---------|-------------------|----------------|
| Routing | Clients know every service URL | One URL; gateway routes by path |
| Auth | Each service validates tokens | ⭐ Centralized JWT validation (§5) |
| Rate limiting | Per-service, inconsistent | Centralized (§4) |
| CORS | Configured everywhere | One place (Phase 5.3) |
| TLS termination | Per service | At the edge |
| Cross-cutting (logging/tracing) | Duplicated | Centralized (12.7) |
| Aggregation / BFF | Client does it | Gateway can (12.2 §6) |
> ⭐ The gateway centralizes **edge cross-cutting concerns**, keeping services focused on business logic (single responsibility — 12.1). ⚠️ Don't put **business logic** in the gateway (it becomes a bottleneck/coupling point) — keep it thin. Beware it being a single point of failure → run multiple replicas (10.2).

---

## 2. Spring Cloud Gateway

The Spring ecosystem's gateway — **reactive** (built on Spring WebFlux / Project Reactor — Phase 16.1), non-blocking, high-throughput. Its model: **Route = Predicate(s) + Filter(s) + a target URI**.
```
   A request comes in →
     match a ROUTE (via PREDICATES) → apply FILTERS (modify req/resp) → forward to target URI
```
| Building block | Meaning |
|----------------|---------|
| **Route** | A rule: if predicates match, apply filters and forward to a URI |
| **Predicate** | A condition to match a request (path, method, header, host...) |
| **Filter** | Modifies the request/response (add headers, rate limit, auth, rewrite path...) |

---

## 3. Routes & Predicates

```yaml
spring.cloud.gateway.routes:
  - id: order-service
    uri: lb://order-service          # "lb://" = load-balanced via service discovery (12.2)
    predicates:
      - Path=/api/orders/**          # match these paths
      - Method=GET,POST
    filters:
      - StripPrefix=1                # drop /api before forwarding
      - AddRequestHeader=X-Gateway, true

  - id: payment-service
    uri: lb://payment-service
    predicates:
      - Path=/api/payments/**
```
| Common predicate | Matches on |
|------------------|-----------|
| `Path=/api/orders/**` | URL path |
| `Method=GET,POST` | HTTP method |
| `Header=X-Tenant, .*` | A request header |
| `Host=*.example.com` | Host |
| `Query=version, 1` | Query param |
| `After/Before/Between` | Time windows |
> ⭐ `uri: lb://order-service` uses **client-side load balancing** over **service discovery** (12.2 — Eureka or K8s). The gateway resolves the logical name to healthy instances. Routes can also be defined in Java (`RouteLocator` bean).

---

## 4. Filters (incl. Rate Limiting with Redis)

Filters transform requests/responses or enforce policies. Built-in ones + custom.
| Built-in filter | Does |
|-----------------|------|
| `StripPrefix` / `RewritePath` | Adjust the path before forwarding |
| `AddRequestHeader` / `AddResponseHeader` | Inject headers |
| `RequestRateLimiter` | ⭐ Rate limit (with Redis) |
| `CircuitBreaker` | Wrap routes with Resilience4j (12.3) |
| `Retry` | Retry failed downstream calls (idempotent only — 12.3) |
| `SetStatus` / `RedirectTo` | Response control |

### 4.1 Distributed rate limiting with Redis ⭐
A single rate limit across **all** gateway replicas needs **shared state** → Redis (Phase 4.8). Spring Cloud Gateway's `RequestRateLimiter` uses a Redis-backed **token-bucket**.
```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 100    # tokens/sec
      redis-rate-limiter.burstCapacity: 200    # bucket size (burst)
      key-resolver: "#{@userKeyResolver}"      # rate-limit per user/API key/IP
```
```java
@Bean KeyResolver userKeyResolver() {           // what to rate-limit by
    return exchange -> Mono.just(exchange.getRequest()
        .getHeaders().getFirst("X-User-Id"));   // per-user limit
}
```
On exceed → **429 Too Many Requests** + `RateLimit-*` / `Retry-After` headers (recall Phase 7.1 §5).
> ⭐ **Redis-backed** limiting is correct across replicas (an in-memory counter per instance would let `N × limit` through). This is the inbound counterpart to Resilience4j's outbound rate limiter (12.3) and the rate-limiting design from Phase 7.1.

---

## 5. Centralized JWT Validation ⭐

The gateway authenticates **once** at the edge: validate the JWT (signature, expiry, issuer — Phase 5.6/15.2), reject if invalid (401), and forward identity (claims) to downstream services (which can then trust it, optionally re-verify, and do authorization).
```yaml
# Gateway as an OAuth2 Resource Server (Spring Security — Phase 5.6)
spring.security.oauth2.resourceserver.jwt.issuer-uri: https://auth.example.com
```
```
   Client ──Authorization: Bearer <JWT>──► GATEWAY
        validate signature/expiry/issuer (JWKS)
        ├─ invalid → 401 (don't forward)
        └─ valid   → forward + propagate claims (e.g., X-User-Id / pass token)  ──► services
```
| Approach | Note |
|----------|------|
| **Validate at gateway, trust downstream** | Simplest; services live on a trusted network (zero-trust caveat below) |
| **Validate at gateway AND each service** | Defense in depth (zero trust) — services re-verify the JWT |
| **Token relay** | Forward the original token so services do their own authZ |
> ⭐ Centralizing **authentication** at the gateway avoids duplicating it everywhere; services focus on **authorization** (RBAC/ABAC — 15.2). ⚠️ In a **zero-trust** model, don't blindly trust internal traffic — have services validate too. The gateway should **never** be the *only* line of security.

---

## 6. Gateway vs Kubernetes Ingress

| | **API Gateway** (Spring Cloud Gateway) | **K8s Ingress** (10.2) |
|--|----------------------------------------|------------------------|
| Layer | Application (rich logic) | Cluster edge (HTTP routing/TLS) |
| Features | Auth, rate limit, filters, aggregation, circuit breaking | Path/host routing, TLS termination |
| Where | A service you write/run | An Ingress controller (nginx/Traefik) |
> ⭐ They can coexist: **Ingress** handles cluster-edge routing/TLS; the **API Gateway** (behind it) handles app-level concerns (auth/rate limit/aggregation). Or the gateway is exposed directly via a LoadBalancer Service. Modern alternative: the **Kubernetes Gateway API** and **service meshes** (Istio/Linkerd) push some of this to infrastructure.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Business logic in the gateway | Keep it thin (edge concerns only) (§1) |
| In-memory rate limiting across replicas | Redis-backed shared limit (§4.1) |
| Gateway as the only security layer | Defense in depth; services validate too (§5) |
| Single gateway instance (SPOF) | Run multiple replicas (10.2) |
| Blocking calls in a reactive gateway | Keep it non-blocking (WebFlux — 16.1) |
| Hardcoding service URLs | `lb://` + discovery (§3, 12.2) |
| Duplicating auth in every service | Centralize authN at gateway (§5) |
| No circuit breaker/timeouts on routes | Add Resilience4j filters (§4, 12.3) |
| Confusing gateway with Ingress | They operate at different layers (§6) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Front door for the services from **12.1/12.2**; uses **service discovery** (`lb://` — 12.2) and **Resilience4j** filters (12.3).
- **Reactive** (Spring WebFlux / Reactor — Phase 16.1).
- **JWT/OAuth2** validation = Spring Security (Phase 5.6/15.2); **rate limiting** = Phase 7.1 §5 + **Redis** (Phase 4.8).
- **CORS** centralized (Phase 5.3); **tracing/logging** at the edge (12.7/9.1).
- Complements **K8s Ingress** (10.2); used in **Strangler Fig** migration (12.1 §6).
- **Observability** (12.7) — gateway is the first span in a trace.
- Core of **Project 7 (Microservices E-Commerce)**.

---

## 9. Quick Self-Check Questions

1. What problems does an API Gateway solve, and what should you NOT put in it?
2. What is the Route = Predicate + Filter + URI model in Spring Cloud Gateway?
3. Give examples of predicates and what they match.
4. What do `lb://` URIs do?
5. How do filters work, and name several built-in ones.
6. Why does distributed rate limiting need Redis, and what happens on exceed (status/headers)?
7. How does centralized JWT validation work, and what gets forwarded downstream?
8. Why is the gateway not sufficient as the only security layer (zero trust)?
9. Gateway vs Kubernetes Ingress — how do they differ and coexist?
10. Why must the gateway be highly available, and how (replicas)?

---

## 10. Key Terms Glossary

- **API Gateway:** single entry point handling edge cross-cutting concerns.
- **Spring Cloud Gateway:** reactive (WebFlux) gateway.
- **Route / predicate / filter:** rule / match condition / request-response transform.
- **`lb://`:** load-balanced URI via service discovery.
- **RequestRateLimiter / token bucket / Redis:** distributed rate limiting.
- **KeyResolver:** what to rate-limit by (user/IP/API key).
- **Centralized JWT validation / token relay:** authenticate once at the edge / forward token.
- **Zero trust:** don't implicitly trust internal traffic.
- **BFF (Backend-for-Frontend):** edge aggregation per client type.
- **Ingress / Gateway API / service mesh:** K8s edge routing / standard / infra-level traffic mgmt.

---

*This is the note for **Section 12.4 — API Gateway**.*
*Previous section in roadmap: **12.3 Resilience4j**.*
*Next section in roadmap: **12.5 Configuration Management**.*
