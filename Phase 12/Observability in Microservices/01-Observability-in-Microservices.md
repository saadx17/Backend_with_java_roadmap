# Observability in Microservices

> **Phase 12 — Microservices → 12.7 Observability in Microservices**
> Goal: Apply observability at scale — centralized log aggregation (ELK / Loki), distributed tracing across services (OpenTelemetry, Zipkin/Jaeger, W3C Trace Context), and metrics aggregation.

---

## 0. The Big Picture

In a monolith, one log file and one process tell the whole story. In microservices, a single user request **fans across many services, containers, and hosts** (12.1/12.2) — so you must **aggregate** logs, **stitch** traces across service boundaries, and **roll up** metrics centrally. This is the three pillars (9.1/9.2) applied to a distributed system, glued together by a shared **`traceId`**.

```
   User request ──► Gateway ──► Order ──► Payment ──► Inventory
                     │           │          │            │
   Each emits LOGS, METRICS, SPANS — all tagged with the SAME traceId
                     ▼
   ┌──────────── Central observability ────────────┐
   │ Logs → ELK/Loki   Traces → Jaeger/Tempo        │
   │ Metrics → Prometheus → Grafana (one pane)       │
   └─────────────────────────────────────────────────┘
```

> Directly extends **9.1 (logging/MDC)** and **9.2 (metrics/tracing/health)** to many services. Relies on the `traceId` in MDC (9.1), Micrometer/Prometheus (9.2), runs on Kubernetes (10.2), and traces the calls (12.2) and sagas (12.6). Often paired with **APM** (9.3).

---

## 1. Centralized Log Aggregation

Each service logs to **stdout** as **structured JSON** (9.1 §5) in its container (10.1); a shipper collects and centralizes logs so you can search across all services in one place.
| Stack | Components |
|-------|-----------|
| **ELK / Elastic** | **E**lasticsearch (store/search — Phase 16.4) + **L**ogstash/Beats (ship/parse) + **K**ibana (UI) |
| **EFK** | Elasticsearch + **Fluentd/Fluent Bit** + Kibana (common on K8s) |
| **Grafana Loki** ⭐ | Lightweight, label-based log store; queried in Grafana alongside metrics/traces (cheaper than ELK) |
```
   service stdout (JSON, 9.1) ──► Fluent Bit / Promtail (shipper) ──► Elasticsearch / Loki ──► Kibana / Grafana
```
- ⭐ Logs **must carry `traceId`, `service`, `level`** (9.1 §5) so you can: filter by service, and **jump from a log line to its full trace** (§2).
- ⚠️ Scrub PII/secrets before logs leave the service (9.1 §6 / Phase 15.3).
> ⭐ **Loki + Grafana** is the modern lightweight choice (logs, metrics, traces in one Grafana UI). **ELK** is more powerful/heavier. Either way: **centralize**, and **correlate by traceId**.

---

## 2. Distributed Tracing (across services) ⭐

The defining microservices observability capability: follow **one request** as it hops across services, seeing where time went and where it failed. (Concepts — **9.2 §5**.)
```
trace a1b2c3 (one request)
└─ span: GET /orders        [gateway]      130ms
   └─ span: order.place     [order-svc]    115ms
      ├─ span: SELECT        [postgres]      12ms
      └─ span: POST /pay     [payment-svc]   80ms   ◄── propagated via W3C traceparent header
         └─ span: charge     [stripe]        70ms
```
| Element | Note |
|---------|------|
| **Trace / span / traceId** | Whole request / one unit of work / shared id (9.2) |
| **Context propagation** ⭐ | The traceId must travel **across** services — over HTTP headers, Kafka/RabbitMQ headers (Phase 11) |
| **W3C Trace Context (`traceparent`)** | The standard header format for propagation (vendor-neutral) |
| **Micrometer Tracing** | Spring's tracing facade; auto-instruments HTTP/DB/messaging; puts traceId/spanId in **MDC** (9.1) |
| **OpenTelemetry (OTel)** ⭐ | The vendor-neutral standard (API + agent + Collector) to emit/route traces (9.3) |
| **Backends** | **Zipkin**, **Jaeger**, **Grafana Tempo** collect & visualize |
```yaml
management.tracing.sampling.probability: 0.1   # sample 10% (cost vs visibility — 9.2)
```
> ⭐ **Propagation is the crux**: without passing `traceparent` across every hop (HTTP and async — Phase 11), the trace breaks. Spring auto-propagates over HTTP; for **async messaging** ensure trace context rides in message headers. Standardize on **OpenTelemetry + W3C Trace Context** for portability (9.3). **Sample** to control cost; keep `traceId` in every log (§1) for always-on correlation.

### 2.1 The correlation trifecta
```
   Grafana/Kibana: see a 500 error LOG → click traceId → open the TRACE (which service/span failed)
                   → check that service's METRICS (saturation/latency) → root cause
```
> ⭐ Logs ↔ traces ↔ metrics, all joined by `traceId`/labels, is what makes a distributed system debuggable. This is the whole point of observability.

---

## 3. Metrics Aggregation

Each service exposes metrics (Micrometer → `/actuator/prometheus` — 9.2); **Prometheus scrapes all instances** (auto-discovered on K8s), and **Grafana** aggregates into per-service and system-wide dashboards.
| Concern | Approach |
|---------|----------|
| Discovery | Prometheus auto-discovers Pods/Services (K8s service discovery — 10.2) |
| Per-service dashboards | Golden signals per service (9.2 §3: latency/traffic/errors/saturation) |
| System-wide rollups | Aggregate across services (`sum by (service)`) |
| Long-term/HA storage | **Thanos** / **Mimir** / **Cortex** (scale Prometheus) |
| Alerting | Alertmanager / Grafana alerts on golden signals (9.2 §4) |
| Cardinality | ⚠️ Keep tags low-cardinality across many services (9.2 §1.2) |
> ⭐ Add a **`service`/`application` tag** to every metric (9.2) so you can slice by service. For many clusters/long retention, federate Prometheus with **Thanos/Mimir**. Build dashboards from the **golden signals** per service, not from "every metric."

---

## 4. Health & Readiness (recap in context)

Every service exposes liveness/readiness (9.2 §6) → Kubernetes probes (10.2). At the fleet level, readiness gates traffic during rolling deploys (10.2 §9) so observability also confirms a deploy is healthy (smoke tests — 10.3) before/after rollout.

---

## 5. Putting It Together (the stack)

```
   Services (Spring Boot, Micrometer Tracing + structured logs)   [each: 9.1/9.2]
        │ logs → Fluent Bit/Promtail     │ metrics → scraped       │ traces → OTel
        ▼                                 ▼                          ▼
   Loki / Elasticsearch            Prometheus (+Thanos)        Jaeger / Tempo / Zipkin
        └──────────────── Grafana / Kibana (correlate by traceId) ──────────────┘
                                   + APM (Datadog/New Relic — 9.3) optionally on top
```
> ⭐ Modern default: **OpenTelemetry** for instrumentation (portable — 9.3) + **Grafana stack** (Loki/Mimir/Tempo) or **ELK + Jaeger**, or a commercial **APM** (9.3). The key principles are **centralize, correlate by traceId, sample traces, low-cardinality metrics, scrub PII**.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Per-service logs only (no aggregation) | Centralize (ELK/Loki) (§1) |
| Logs without traceId/service | Add them (correlation) (§1, 9.1) |
| Broken traces (context not propagated) | Propagate W3C `traceparent` over HTTP **and** messaging (§2) |
| Tracing 100% of requests at scale | Sample (§2, 9.2) |
| High-cardinality metric tags across services | Keep low-cardinality (§3, 9.2) |
| Vendor-locked instrumentation | Standardize on OpenTelemetry (§2/§5, 9.3) |
| PII/secrets in logs/traces | Scrub (§1, Phase 15.3) |
| Dashboards of "everything" | Golden signals per service (§3, 9.2) |
| No correlation between pillars | Join logs/traces/metrics by traceId (§2.1) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Scales **9.1 (logging/MDC)** + **9.2 (metrics/tracing/health)** + **9.3 (APM)** to many services.
- **Micrometer Tracing / OpenTelemetry** instrument Spring; `traceId` in **MDC** (9.1) joins logs↔traces.
- **Propagation** across **HTTP (12.2)** and **messaging (Phase 11)**; traces **sagas** (12.6).
- **Prometheus/Grafana** aggregate; auto-discovery via **K8s** (10.2); **CI/CD** smoke tests (10.3).
- **Security** — scrub PII/secrets (Phase 15.3).
- The operational backbone of **Project 7 (Microservices E-Commerce)** & **Project 6 (Pipeline)**.

---

## 8. Quick Self-Check Questions

1. Why is observability harder in microservices than in a monolith?
2. How do logs get centralized, and what fields must they carry for correlation?
3. Compare ELK and Loki.
4. What is context propagation, and why does it make or break distributed tracing?
5. What is W3C Trace Context, and why standardize on OpenTelemetry?
6. How do you propagate trace context across async messaging?
7. Why sample traces, and why keep traceId in every log anyway?
8. How are metrics aggregated across services, and what scales Prometheus?
9. Describe the correlation trifecta (logs↔traces↔metrics) for root-cause analysis.
10. What are the key principles of microservices observability?

---

## 9. Key Terms Glossary

- **Log aggregation (ELK/EFK/Loki):** centralizing logs for cross-service search.
- **Fluent Bit / Fluentd / Promtail / Logstash:** log shippers.
- **Distributed tracing / trace / span / traceId:** following a request across services.
- **Context propagation / W3C Trace Context (`traceparent`):** passing trace ids across hops.
- **Micrometer Tracing / OpenTelemetry / Collector:** instrumentation facade / standard / router.
- **Zipkin / Jaeger / Tempo:** trace backends.
- **Sampling:** recording a fraction of traces.
- **Prometheus / Thanos / Mimir / Cortex:** metrics scrape / scalable long-term storage.
- **Golden signals:** latency, traffic, errors, saturation (per service).
- **Correlation trifecta:** logs↔traces↔metrics joined by traceId.

---

*This is the note for **Section 12.7 — Observability in Microservices**.*
*Previous section in roadmap: **12.6 Distributed Patterns**.*
*This completes **Phase 12 — Microservices**.*
*Next: **Phase 13 — Performance & Optimization**.*
