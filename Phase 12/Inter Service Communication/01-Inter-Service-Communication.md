# Inter-Service Communication

> **Phase 12 — Microservices → 12.2 Inter-Service Communication**
> Goal: Understand how microservices talk to each other — synchronous REST, gRPC (protobuf, streaming), Spring HTTP clients (WebClient, OpenFeign), asynchronous messaging brokers, and service discovery (Eureka, Consul, Kubernetes DNS).

---

## 0. The Big Picture

Once you have multiple services (12.1), they must communicate over the **network** — which is unreliable, slow(er), and partial-failure-prone (the "fallacies of distributed computing"). You choose between **synchronous** (request/response: REST, gRPC) and **asynchronous** (events/messages: Kafka, RabbitMQ — Phase 11), and you need **service discovery** to find each other.

```
   SYNC (request/response)                ASYNC (events — Phase 11)
   Order ──REST/gRPC──► Payment           Order ──event──► [ broker ] ──► Payment
        ◄── response ──                    (fire & forget, decoupled — 12.1/11.4)
   tight-ish coupling, immediate           loose coupling, eventual consistency
```

> Builds on REST (Phase 7), messaging (Phase 11), and microservices fundamentals (12.1). Communication failures are why we need **resilience** (12.3); discovery often comes from **Kubernetes** (10.2). Choosing sync vs async is a recurring design decision (recall 11.1).

---

## 1. Synchronous vs Asynchronous (the first decision)

| | **Synchronous** (REST/gRPC) | **Asynchronous** (messaging) |
|--|------------------------------|------------------------------|
| Pattern | Request/response | Events/commands via broker |
| Coupling | Temporal (both must be up) | Decoupled (broker buffers) |
| Consistency | Immediate | Eventual (Phase 11.4) |
| Failure impact | Cascades (need resilience — 12.3) | Isolated (broker absorbs) |
| Use for | Queries needing an answer now | Notifications, workflows, fan-out |
> ⭐ **Prefer async/events for cross-service workflows** (decoupling, resilience — 12.1/11.4), and use **sync** for queries that need an immediate response. ⚠️ Long **synchronous call chains** (A→B→C→D) are fragile (cumulative latency + cascading failure) — the **distributed monolith** smell (12.1 §7). Break them with async or aggregation.

---

## 2. REST (Synchronous, JSON over HTTP)

The default sync mechanism — the same REST you designed in Phase 7, now between services.
- ✅ Universal, simple, human-readable, tooling everywhere (OpenAPI — 7.2).
- ⚠️ Text JSON is verbose; HTTP/1.1 overhead; no built-in streaming.
- Often **internal** APIs are still REST; the **API Gateway** (12.4) fronts external REST.
> Use REST for straightforward request/response between services and for public APIs. For high-throughput, low-latency internal calls, consider gRPC (§3).

---

## 3. gRPC (High-Performance RPC)

**gRPC** is a high-performance RPC framework: you define services & messages in a **`.proto`** file (Protocol Buffers), and it generates strongly-typed client & server code. Runs over **HTTP/2** with **binary** Protobuf serialization.
```proto
// order.proto — the contract (language-agnostic)
syntax = "proto3";
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);                 // unary
  rpc StreamOrders (StreamRequest) returns (stream Order);        // server streaming
}
message GetOrderRequest { int64 id = 1; }
message Order { int64 id = 1; string status = 2; double total = 3; }
```
| Feature | Note |
|---------|------|
| **Protobuf** | Compact **binary** serialization (smaller/faster than JSON); schema-defined (recall Avro — 11.2 §9) |
| **HTTP/2** | Multiplexing, header compression, low latency |
| **Code generation** | Typed stubs in many languages (contract-first) |
| **Streaming** | Unary, **server-streaming**, **client-streaming**, **bidirectional** |
| **Strong contracts** | `.proto` is the source of truth; backward-compatible field evolution (numbered fields) |

| | REST/JSON | gRPC/Protobuf |
|--|-----------|---------------|
| Payload | Text (JSON) | Binary (compact) |
| Transport | HTTP/1.1 (usually) | HTTP/2 |
| Streaming | Limited (SSE/WebSocket — 16.3) | ✅ First-class |
| Browser-native | ✅ Yes | ❌ Needs grpc-web/proxy |
| Human-readable | ✅ | ❌ |
| Best for | Public APIs, simplicity | **Internal high-perf service-to-service**, streaming, polyglot |
> ⭐ Use **gRPC for internal, performance-critical, high-volume service-to-service** calls and streaming; use **REST for public/external** APIs and simplicity. Spring Boot supports gRPC via the `grpc-spring-boot-starter` / Spring gRPC.

---

## 4. Spring HTTP Clients: WebClient & OpenFeign

For calling other services' REST APIs from Spring:

### 4.1 WebClient (reactive, modern) ⭐
The modern, non-blocking client (Phase 5.9; reactive — Phase 16.1). Works for both reactive and (via `.block()`) blocking code; **`RestTemplate` is in maintenance** — prefer WebClient (or the new `RestClient` for synchronous).
```java
@Service @RequiredArgsConstructor
class PaymentClient {
    private final WebClient webClient;   // configured with base URL/timeouts
    Mono<PaymentResult> charge(ChargeRequest req) {
        return webClient.post().uri("/payments")
            .bodyValue(req)
            .retrieve()
            .bodyToMono(PaymentResult.class)
            .timeout(Duration.ofSeconds(2));     // timeout (12.3)
    }
}
```

### 4.2 OpenFeign (declarative)
**Spring Cloud OpenFeign** — define an interface; Feign generates the HTTP client. Clean, integrates with service discovery (§5) & Resilience4j (12.3).
```java
@FeignClient(name = "payment-service")     // resolved via discovery/load balancing (§5)
interface PaymentClient {
    @PostMapping("/payments")
    PaymentResult charge(@RequestBody ChargeRequest req);
}
// inject and call like a normal bean — Feign does the HTTP under the hood
```
| Client | Style | Note |
|--------|-------|------|
| **WebClient** | Reactive/fluent | ⭐ Default for new code; non-blocking; timeouts/retries |
| **RestClient** | Synchronous fluent (Boot 3.2+) | Modern blocking alternative |
| **OpenFeign** | Declarative interface | Great with discovery + Resilience4j; very readable |
| **RestTemplate** | Synchronous template | Legacy/maintenance — avoid in new code |
> ⭐ Always set **timeouts** and wrap calls with **circuit breakers/retries** (Resilience4j — 12.3) — a slow downstream must not hang your service.

---

## 5. Service Discovery

Service instances are dynamic (scale up/down, restart, change IPs — recall K8s Pods are ephemeral, 10.2). **Service discovery** lets a caller find healthy instances by **logical name** instead of hardcoded IPs, and **load-balance** across them.

| Approach | How |
|----------|-----|
| **Client-side discovery** | Client queries a **registry**, picks an instance, calls it (e.g., Eureka + Spring Cloud LoadBalancer) |
| **Server-side discovery** | A load balancer/router resolves the name (e.g., K8s Service §5.3) |

### 5.1 Eureka (Netflix / Spring Cloud)
A **service registry**: instances **register** themselves and send heartbeats; clients **look up** by name. Client-side load balancing via Spring Cloud LoadBalancer.
```
   Order Service ──register──► [ Eureka Registry ] ◄──register── Payment Service (3 instances)
   Order looks up "payment-service" → gets instance list → load-balances the call
```

### 5.2 Consul
HashiCorp **Consul** — service discovery + health checking + **KV config store** (overlaps with config — 12.5) + multi-datacenter. More than just a registry.

### 5.3 Kubernetes DNS ⭐ (the cloud-native default)
On Kubernetes (10.2), a **Service** *is* discovery: a stable DNS name (`payment-service`) load-balances across the Pods automatically — **no Eureka needed**.
```
   http://payment-service           → resolves via K8s DNS → load-balanced across Pods (10.2)
   http://payment-service.ns.svc.cluster.local
```
> ⭐ **If you run on Kubernetes, you usually don't need Eureka/Consul** — K8s Services + DNS handle discovery and load balancing natively (kube-proxy — 10.2). Eureka/Consul shine outside K8s or for richer features. Don't add a registry you don't need.

---

## 6. API Composition / Aggregation

A client (or the **gateway** — 12.4) sometimes needs data from several services. Options:
- **API Composition / aggregator**: a service/gateway calls several services and merges results (sync; watch latency — do calls in parallel).
- **CQRS read model** (Phase 11.4): maintain a denormalized view updated by events → one fast read, no fan-out at query time. ⭐ Preferred for high-read scenarios.
> ⚠️ Chatty aggregation = many sync calls = latency + failure surface. Prefer event-built read models (CQRS) or **BFF (Backend-for-Frontend)** patterns at the edge.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Long synchronous call chains | Async/events or aggregation (§1, Phase 11) |
| No timeouts on remote calls | Always set timeouts + circuit breakers (§4, 12.3) |
| Hardcoded service IPs/URLs | Service discovery / K8s DNS (§5) |
| `RestTemplate` in new code | WebClient / RestClient (§4) |
| JSON everywhere even for hot internal paths | Consider gRPC for internal high-perf (§3) |
| Adding Eureka on Kubernetes needlessly | Use K8s Services/DNS (§5.3) |
| Chatty API composition (serial calls) | Parallelize or use CQRS read models (§6) |
| Sync coupling = distributed monolith | Async + correct boundaries (§1, 12.1) |
| Breaking the wire contract | Version REST (7.1) / numbered proto fields / schema registry (11.2) |
| No tracing across calls | Propagate trace context (12.7) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **REST** = Phase 7 between services; **OpenAPI** (7.2) documents internal contracts; **DTOs** (7.3) are the wire models (also an ACL — 12.1 §3).
- **WebClient/OpenFeign** (Phase 5.9) for calls; **reactive** (Phase 16.1) underlies WebClient.
- **Async** alternative = Kafka/RabbitMQ (Phase 11) → event-driven (11.4).
- **Resilience** (12.3) wraps every remote call (timeouts, circuit breaker, retry).
- **Service discovery** ↔ **Kubernetes** Services/DNS (10.2); **API Gateway** (12.4) fronts and routes.
- **Observability** (12.7) — distributed tracing follows calls; **CQRS** read models (11.4) reduce fan-out.
- Central to **Project 7 (Microservices E-Commerce)**.

---

## 9. Quick Self-Check Questions

1. Sync vs async inter-service communication — trade-offs and when to use each?
2. Why are long synchronous call chains dangerous?
3. What is gRPC, and how do Protobuf + HTTP/2 + streaming compare to REST/JSON?
4. When choose gRPC vs REST?
5. Compare WebClient, RestClient, OpenFeign, and RestTemplate — which for new code?
6. Why must remote calls always have timeouts + circuit breakers?
7. What is service discovery, and client-side vs server-side?
8. How do Eureka, Consul, and Kubernetes DNS provide discovery — and when do you NOT need a registry?
9. What is API composition, and why might a CQRS read model be better?
10. How do you keep wire contracts from breaking across services?

---

## 10. Key Terms Glossary

- **Synchronous / asynchronous communication:** request-response vs event/message.
- **gRPC / Protobuf / HTTP/2:** RPC framework / binary schema format / multiplexed transport.
- **Streaming (unary/server/client/bidi):** gRPC call shapes.
- **WebClient / RestClient / OpenFeign / RestTemplate:** Spring HTTP clients.
- **Service discovery (client-side/server-side):** finding service instances by name.
- **Service registry (Eureka / Consul):** where instances register & are looked up.
- **Kubernetes Service / DNS:** native cloud discovery + load balancing.
- **Load balancing:** distributing calls across instances.
- **API composition / aggregator / BFF:** combining data from multiple services.
- **CQRS read model:** denormalized event-built view to avoid query-time fan-out.

---

*This is the note for **Section 12.2 — Inter-Service Communication**.*
*Previous section in roadmap: **12.1 Microservices Fundamentals**.*
*Next section in roadmap: **12.3 Resilience4j**.*
