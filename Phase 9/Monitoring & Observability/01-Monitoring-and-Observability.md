# Monitoring & Observability (Micrometer, Prometheus, Grafana, Tracing, Health)

> **Phase 9 — Logging & Monitoring → 9.2 Monitoring & Observability**
> Goal: Understand observability's three pillars and master metrics with **Micrometer** (Counter/Gauge/Timer/DistributionSummary, `@Timed`), **Prometheus** (pull model, PromQL, Actuator endpoint), **Grafana** (dashboards & alerting), **distributed tracing** (trace/span, Micrometer Tracing, Zipkin/Jaeger/OpenTelemetry, context propagation), and **health checks** (liveness/readiness).

---

## 0. The Big Picture

**Monitoring** = watching known signals (is the system up? how fast?). **Observability** = being able to ask *new* questions about your system's behavior from its outputs — without shipping new code. The **three pillars** of observability:

```
   LOGS (9.1)              METRICS (this note)          TRACES (this note)
   "what happened          "how is the system           "what path did THIS
    to this request?"       doing overall?"              request take across services?"
   text/JSON events        numbers over time            spans linked by a trace id
        │                        │                            │
        └─────────────► correlated by traceId (MDC — 9.1) ◄──┘
```

> Builds on logging (9.1 — the third pillar + MDC `traceId` glue), Spring Boot Actuator (Phase 5.2), and HTTP (Phase 0.2). It's the foundation for **observability in microservices** (Phase 12.7), **APM** (9.3), **performance work** (Phase 13), and **Kubernetes** health probes (Phase 10.2).

| Pillar | Question it answers | Tool here |
|--------|--------------------|-----------|
| **Logs** | What happened (discrete events)? | SLF4J/Logback (9.1) → ELK/Loki |
| **Metrics** | How much / how fast (aggregates over time)? | Micrometer → Prometheus → Grafana |
| **Traces** | Where did the time go across services? | Micrometer Tracing → Zipkin/Jaeger/OTel |

---

## 1. Metrics with Micrometer

**Micrometer** is "**SLF4J for metrics**" — a vendor-neutral **facade** (Phase 14.1 — like SLF4J in 9.1) over many monitoring backends (Prometheus, Datadog, New Relic, CloudWatch...). Spring Boot Actuator auto-configures it and instruments tons of things out of the box.

```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId></dependency>
<dependency><groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId></dependency>   <!-- chosen backend -->
```

### 1.1 The four core meter types ⭐
| Meter | Represents | Can go down? | Example |
|-------|-----------|:------------:|---------|
| **Counter** | A monotonically **increasing** count | ❌ no | orders placed, errors, requests |
| **Gauge** | A value that **goes up and down** *now* | ✅ yes | queue size, active connections, cache entries |
| **Timer** | **Duration** + count of events (latency) | — | request latency, method execution time |
| **DistributionSummary** | Distribution of **non-time** values | — | response payload size, batch sizes |

```java
@Service
@RequiredArgsConstructor
class OrderService {
    private final MeterRegistry registry;       // injected by Spring (Phase 5.1)

    void placeOrder(Order o) {
        // COUNTER — count events (with tags/labels for dimensions)
        registry.counter("orders.placed", "status", o.getStatus().name()).increment();

        // TIMER — measure how long something takes
        Timer.Sample sample = Timer.start(registry);
        process(o);
        sample.stop(registry.timer("orders.processing.time"));

        // DISTRIBUTION SUMMARY — distribution of a non-time value
        registry.summary("orders.item.count").record(o.getItems().size());
    }

    // GAUGE — track a live value (registry holds a weak ref to the source)
    @PostConstruct void initGauges() {
        registry.gauge("orders.queue.size", queue, Queue::size);
    }
}
```

### 1.2 `@Timed` annotation (declarative timing)
```java
@Timed(value = "orders.place", percentiles = {0.5, 0.95, 0.99})   // p50/p95/p99 (Phase 13.2)
public Order placeOrder(CreateOrderRequest req) { ... }
```
> ⭐ **Tags/labels** (`"status", "PAID"`) turn one metric into many **dimensions** you can slice in queries (§3 PromQL) — e.g., error rate *per endpoint*. ⚠️ **Beware high-cardinality tags** (userId, request id, full URL with ids) — each unique combination is a new time series → can blow up memory/storage. Keep tags low-cardinality (status, method, outcome).

### 1.3 What Spring auto-instruments
HTTP request latency (`http.server.requests` — with tags `uri`, `method`, `status`, `outcome`), JVM (heap/GC/threads — Phase 1.11), HikariCP connection pool (Phase 13.1), Tomcat, cache hit/miss (Phase 5.7), Kafka/RabbitMQ (Phase 11), Logback event counts. Huge value for free.

---

## 2. Prometheus (Metrics Storage & the Pull Model)

**Prometheus** is a time-series database + monitoring system that **scrapes (pulls)** metrics from your app's HTTP endpoint on an interval, stores them, and lets you query with **PromQL**.

```
   ┌──────────────┐   scrape every 15s     ┌────────────┐   query (PromQL)   ┌─────────┐
   │ Your app     │ ◄───── GET ──────────  │ Prometheus │ ◄───────────────── │ Grafana │
   │ /actuator/   │   /actuator/prometheus │  (TSDB)    │                    │  (§4)   │
   │  prometheus  │ ─────── metrics ─────► │            │ ──► Alertmanager ──► alerts  │
   └──────────────┘                        └────────────┘                    └─────────┘
```

- ⭐ **Pull model:** Prometheus calls *you* (scrapes your endpoint), rather than you pushing metrics out. This makes service discovery, health, and "is the target up?" natural. (Short-lived/batch jobs that can't be scraped use a **Pushgateway**.)
- Expose the endpoint via Actuator (Phase 5.2):
```yaml
management:
  endpoints.web.exposure.include: health,info,prometheus,metrics
  endpoint.health.probes.enabled: true     # liveness/readiness groups (§6)
  metrics.tags.application: order-service   # common tag on all metrics
```
Now `GET /actuator/prometheus` returns the Prometheus text format:
```
# HELP orders_placed_total ...
orders_placed_total{status="PAID",application="order-service"} 1834
http_server_requests_seconds_count{uri="/api/v1/orders",method="GET",status="200"} 5021
```

### 2.1 PromQL (querying)
```promql
rate(http_server_requests_seconds_count[5m])                        # requests/sec over 5m
sum by (uri) (rate(http_server_requests_seconds_count{status=~"5.."}[5m]))  # 5xx rate per endpoint
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))     # p95 latency
sum(jvm_memory_used_bytes{area="heap"}) / sum(jvm_memory_max_bytes{area="heap"})  # heap %
```
> ⭐ `rate()` over a counter gives a **per-second rate**; `histogram_quantile()` over a Timer's buckets gives **percentiles** (p95/p99 — Phase 13.2). PromQL's label filters (`{status=~"5.."}`) and `sum by (...)` aggregation are why low-cardinality tags (§1.2) matter.

---

## 3. The Four Golden Signals (what to actually watch)

Google SRE's **Four Golden Signals** — the highest-value things to monitor for any request-serving service:
| Signal | Meaning | Example metric |
|--------|---------|----------------|
| **Latency** | How long requests take (split success vs error!) | `http_server_requests` p50/p95/p99 |
| **Traffic** | Demand on the system | requests/sec |
| **Errors** | Rate of failed requests | 5xx rate, exception counters |
| **Saturation** | How "full" the system is | CPU, heap, thread pool, DB pool usage |
> ⭐ Also useful: **RED** (Rate, Errors, Duration — for services) and **USE** (Utilization, Saturation, Errors — for resources). Start dashboards/alerts from these, not from "every metric we can collect."

---

## 4. Grafana (Dashboards & Alerting)

**Grafana** is the visualization/alerting layer — it queries Prometheus (and Loki for logs, Tempo/Jaeger for traces) and renders **dashboards**, plus evaluates **alert rules**.

| Grafana piece | Does |
|---------------|------|
| **Data source** | Connects to Prometheus / Loki / Tempo / etc. |
| **Dashboard / panel** | Charts built from PromQL queries (latency, error rate, golden signals §3) |
| **Variables** | Template a dashboard by service/instance/endpoint |
| **Alert rule** | Fire when a query crosses a threshold (e.g., 5xx rate > 1% for 5m) |
| **Contact points / notifications** | Route alerts to Slack, PagerDuty, email |

```
Alert example: "High 5xx error rate"
  expr:  sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
       / sum(rate(http_server_requests_seconds_count[5m])) > 0.01
  for:   5m         # must hold 5 minutes (avoid flapping)
  → notify #oncall (PagerDuty)
```
> ⭐ Good alerts are **symptom-based** (user-facing: high latency/error rate) and **actionable** (a human can do something). ⚠️ Avoid alert fatigue: alert on the golden signals (§3), use `for:` durations to debounce, and tie severity to user impact. (Alertmanager can also do dedup/routing/silencing.)

---

## 5. Distributed Tracing

In a microservices request that hops across services (Phase 12), logs/metrics per service aren't enough — you need to see the **whole request path** and *where the time went*. **Tracing** does that.

| Term | Meaning |
|------|---------|
| **Trace** | The end-to-end journey of one request (a tree of spans), identified by a **traceId** |
| **Span** | One unit of work (an HTTP call, a DB query, a method) with start/end + metadata |
| **Parent/child spans** | Nesting that shows causality & timing |
| **Context propagation** | Passing trace/span ids across service boundaries (HTTP headers, Kafka headers) |

```
trace a1b2c3
└─ span: GET /api/orders            [gateway]      120ms
   └─ span: order-service.place     [order-svc]    100ms
      ├─ span: SELECT orders        [postgres]      15ms
      └─ span: POST /payments       [payment-svc]   70ms
         └─ span: charge card       [stripe]        60ms   ◄── here's the bottleneck
```

- **Micrometer Tracing** (successor to Spring Cloud Sleuth) auto-creates spans for HTTP/DB/messaging and **propagates context**, putting `traceId`/`spanId` into the **MDC** (9.1 §3) so your logs join the trace.
- **Context propagation standard: W3C Trace Context** (the `traceparent` HTTP header) — how the traceId travels between services (Phase 12.7).
- **Backends/exporters:** **Zipkin**, **Jaeger** (collect & visualize traces). **OpenTelemetry (OTel)** is the vendor-neutral standard (APIs + collector) for emitting traces/metrics/logs — increasingly the default (9.3/12.7).

```xml
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-zipkin</artifactId></dependency>
```
```yaml
management.tracing.sampling.probability: 0.1   # sample 10% of requests (cost vs visibility)
```
> ⭐ **Sampling**: tracing every request is expensive at scale, so you sample a fraction (e.g., 10%). The **traceId in logs** (9.1) is the cheap, always-on correlation; full spans are sampled. The trio — log line → trace → metric, all sharing `traceId` — is the heart of observability (deep-dived for microservices in Phase 12.7).

---

## 6. Health Checks (Liveness & Readiness)

Health checks let orchestrators (Kubernetes — Phase 10.2) and load balancers know if your app is OK. Spring Boot Actuator exposes them.

```
GET /actuator/health            -> {"status":"UP"}          (aggregate)
GET /actuator/health/liveness   -> liveness probe
GET /actuator/health/readiness  -> readiness probe
```

| Probe | Question | If it fails (K8s) |
|-------|----------|-------------------|
| **Liveness** | Is the app **alive** (not deadlocked/broken)? | **Restart** the pod |
| **Readiness** | Is the app **ready to serve traffic** (deps up, warmed)? | **Stop sending traffic** (remove from LB) — don't restart |
| **Startup** (K8s) | Has a slow-starting app finished booting? | Hold off liveness until started |

- ⚠️ **Liveness ≠ readiness.** A common mistake is making liveness depend on a database — if the DB blips, K8s **restarts** all your pods (making it worse). DB/dependency state belongs in **readiness** (stop traffic, don't kill the app).
- **Health indicators**: Actuator auto-checks DB, disk, Redis, Kafka, etc. (`db`, `diskSpace`, `redis`...). Write custom ones via `HealthIndicator`:
```java
@Component
class PaymentGatewayHealth implements HealthIndicator {
    public Health health() {
        return pingGateway() ? Health.up().build()
                             : Health.down().withDetail("gateway","unreachable").build();
    }
}
```
```yaml
management.endpoint.health.show-details: when-authorized   # ⚠️ don't leak internals publicly (Phase 15)
```
> ⭐ Spring Boot maps `health/liveness` and `health/readiness` directly to **Kubernetes probes** (Phase 10.2). Readiness also flips to "OUT_OF_SERVICE" during graceful shutdown so traffic drains before the pod stops.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| High-cardinality metric tags (userId, ids) | Keep tags low-cardinality (§1.2) |
| Counting log lines to get rates | Use a Counter/Timer metric (§1, 9.1) |
| Exposing all Actuator endpoints publicly | Expose only needed; secure them (§2, Phase 15) |
| Liveness depending on the DB | Put deps in **readiness**, not liveness (§6) |
| Alerting on causes, not symptoms | Alert on golden signals; user-facing (§3/§4) |
| No `for:` on alerts → flapping | Debounce with a duration (§4) |
| Tracing 100% of requests at scale | Sample (e.g., 10%) (§5) |
| Metrics without correlation to logs/traces | Share `traceId` across all three (§0/§5) |
| Using a Gauge for something monotonic | Counter for ever-increasing values (§1.1) |
| Forgetting percentiles (only averages) | Track p95/p99 (§1.2, Phase 13.2) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Actuator** (Phase 5.2) exposes health, metrics, and the Prometheus endpoint.
- **Micrometer** auto-instruments HTTP, JVM (Phase 1.11), HikariCP (Phase 13.1), cache (Phase 5.7), messaging (Phase 11).
- **MDC `traceId`** (9.1) is the bridge: logs ↔ traces.
- **Health probes** drive **Kubernetes** liveness/readiness (Phase 10.2) and graceful shutdown.
- **Distributed tracing** is essential in **microservices** (Phase 12.7 — OTel, W3C Trace Context, Zipkin/Jaeger).
- **Performance** (Phase 13) uses these metrics/percentiles to find bottlenecks & load-test (13.2).
- **APM** (9.3) is the commercial, all-in-one layer over these pillars.
- **Security** (Phase 15) — lock down Actuator; don't leak health details.
- Used in **Projects 5, 6, 7** (observability stacks).

---

## 9. Quick Self-Check Questions

1. What are the three pillars of observability, and what question does each answer?
2. What is Micrometer, and what are the four core meter types (with an example of each)?
3. Why are metric tags powerful but dangerous (cardinality)?
4. Explain Prometheus's pull model and what `/actuator/prometheus` exposes.
5. Write PromQL for: request rate, 5xx rate per endpoint, and p95 latency.
6. What are the Four Golden Signals?
7. What does Grafana do, and what makes a *good* alert?
8. Define trace, span, and context propagation. What is W3C Trace Context?
9. What's the relationship between Micrometer Tracing, OpenTelemetry, and Zipkin/Jaeger? Why sample?
10. Liveness vs readiness — what does each do in Kubernetes, and why must liveness NOT depend on the DB?

---

## 10. Key Terms Glossary

- **Observability:** ability to ask new questions of a system from its outputs (logs+metrics+traces).
- **Micrometer:** vendor-neutral metrics facade for the JVM.
- **Counter / Gauge / Timer / DistributionSummary:** the four meter types.
- **Tag/label / cardinality:** metric dimensions / number of unique series.
- **`@Timed`:** declarative method timing.
- **Prometheus / pull model / Pushgateway:** TSDB that scrapes targets / push for batch jobs.
- **PromQL / `rate()` / `histogram_quantile()`:** query language / rate / percentiles.
- **Four Golden Signals (RED/USE):** latency, traffic, errors, saturation.
- **Grafana / alert rule / contact point:** dashboards & alerting.
- **Trace / span / traceId / context propagation:** request journey / unit of work / id / passing ids across services.
- **Micrometer Tracing / OpenTelemetry / Zipkin / Jaeger:** tracing facade / standard / backends.
- **W3C Trace Context (`traceparent`):** standard header for propagation.
- **Sampling:** recording a fraction of traces.
- **Liveness / readiness / startup probe:** restart-if-broken / traffic-gating / slow-start health checks.
- **HealthIndicator:** custom Actuator health check.

---

*This is the note for **Section 9.2 — Monitoring & Observability**.*
*Previous section in roadmap: **9.1 Logging**.*
*Next section in roadmap: **9.3 APM (Application Performance Monitoring)**.*
