# APM — Application Performance Monitoring

> **Phase 9 — Logging & Monitoring → 9.3 APM (Application Performance Monitoring)**
> Goal: Understand commercial/all-in-one **APM** platforms — what they add over DIY observability, the **Java agent** auto-instrumentation model, the major players (**New Relic, Datadog, Elastic APM**), and **error tracking with Sentry**.

---

## 0. The Big Picture

In 9.1 (logs) and 9.2 (metrics + traces) you assembled observability from open-source parts (Logback, Micrometer, Prometheus, Grafana, Zipkin/OTel). **APM** platforms package all three pillars — **plus deep code-level performance insight** — into one commercial product, usually installed with **near-zero code changes** via a **Java agent**. The trade-off: convenience & depth vs cost & vendor lock-in.

```
   DIY observability (9.1/9.2)            APM platform (this note)
   ──────────────────────────            ────────────────────────
   Logback + Loki   (logs)        │
   Micrometer + Prometheus (metrics)  ──►  ONE agent → ONE vendor UI:
   Micrometer Tracing + Jaeger (traces)    logs + metrics + traces + code profiling
   Grafana (dashboards)           │        + auto dashboards + alerting + error tracking
   you wire it all together       │        mostly auto-instrumented, hosted/SaaS
```

> Builds directly on 9.1 (logs) and 9.2 (metrics, traces, the three pillars). APM is what many companies use instead of, or alongside, the OSS stack. Connects to performance work (Phase 13), microservices observability (Phase 12.7), and the JVM agent mechanism (Phase 1.14 — instrumentation/bytecode).

---

## 1. What APM Adds Over DIY

| Capability | DIY (9.1/9.2) | APM platform |
|------------|---------------|--------------|
| Logs + metrics + traces | Wire up separately | Unified in one UI, auto-correlated |
| **Code-level profiling** | Async Profiler/JFR manually (Phase 13.1) | Continuous, automatic (slow methods, N+1 queries) |
| **Auto-instrumentation** | Add Micrometer/OTel deps | Java agent, often no code changes |
| Service/dependency maps | Build manually | Auto-generated topology |
| Database/external-call insight | Manual spans | Automatic SQL & HTTP span breakdown |
| Setup effort | High | Low (drop in an agent) |
| Cost | Infra/ops time | $$$ subscription (often per-host/per-GB) |
| Vendor lock-in | Low (OSS) | Higher |

> ⭐ APM's killer feature is **automatic code-level visibility**: it shows *which method/SQL query* in *which trace* is slow, often pinpointing N+1 queries (Phase 5.4), slow downstream calls, and GC pauses (Phase 1.11) — with little manual instrumentation. The cost is money and lock-in; many teams blend OSS + OTel to stay portable (§4).

---

## 2. The Java Agent Model (Auto-Instrumentation)

APMs (and OpenTelemetry) typically attach a **Java agent** — a `.jar` loaded via the `-javaagent` JVM flag — that **rewrites bytecode at class-load time** (using the `java.lang.instrument` API — recall instrumentation/bytecode, Phase 1.14) to insert timing/tracing hooks into frameworks (Spring MVC, JDBC, HTTP clients, Kafka...).

```bash
# No code changes — just a JVM flag at startup:
java -javaagent:/opt/newrelic/newrelic.jar -jar app.jar
java -javaagent:/opt/datadog/dd-java-agent.jar -Ddd.service=order-service -jar app.jar
java -javaagent:/opt/opentelemetry-javaagent.jar -jar app.jar     # OTel agent (vendor-neutral)
```

| Aspect | Note |
|--------|------|
| **How** | `-javaagent` → instruments bytecode as classes load (Phase 1.14) |
| **What it captures** | Web requests, DB queries, external HTTP, cache, messaging — automatically |
| **Code changes** | Usually **none** (agent) — optionally add a custom API for business spans |
| **Config** | System properties / env vars (service name, environment, sample rate) |
| **In containers** | Bake the agent into the image / set `JAVA_TOOL_OPTIONS` (Phase 10.1) |
| ⚠️ Overhead | Small but non-zero CPU/latency cost; tune sampling (9.2 §5) |

> ⭐ The agent model is why APM is "drop-in": no rewriting code. The same mechanism powers the **OpenTelemetry Java agent**, which can export to *any* OTel-compatible backend (vendor-neutral — avoids lock-in). ⚠️ Don't stack multiple competing agents (conflicts/overhead).

---

## 3. The Major APM Players

| Platform | Notes |
|----------|-------|
| **New Relic** | Mature full-stack APM; `newrelic.jar` agent; APM + infra + logs + browser/RUM; NRQL query language; usage-based pricing |
| **Datadog** | Very popular all-in-one (APM + infra + logs + metrics + RUM + security); `dd-java-agent`; strong dashboards/integrations; per-host + ingestion pricing |
| **Elastic APM** | Part of the **Elastic Stack** (Elasticsearch/Kibana — Phase 16.4); APM agent or OTel; self-hostable (cost control) or Elastic Cloud; integrates logs/metrics in one stack |
| (others) | Dynatrace, AppDynamics, Instana, Honeycomb, Grafana Cloud (Tempo/Mimir/Loki) |

| If you want... | Lean toward |
|----------------|-------------|
| Best-in-class all-in-one SaaS | Datadog / New Relic |
| Already on the Elastic/ELK stack | Elastic APM |
| Self-host / cost control / OSS-leaning | Elastic APM (self-managed) or Grafana stack + OTel |
| Avoid vendor lock-in | **OpenTelemetry** instrumentation → swap backends (§2/§4) |

> ⭐ Whatever you pick, prefer instrumenting with **OpenTelemetry** where possible so the *data* is portable even if you change the *backend* later. All major APMs accept OTel data now.

---

## 4. OpenTelemetry & APM (the portability strategy)

**OpenTelemetry (OTel)** is the CNCF vendor-neutral standard for generating and exporting **traces, metrics, and logs** (recall 9.2 §5). It decouples *instrumentation* from *backend*:
```
   App + OTel agent/SDK ──► OTel Collector ──► export to: New Relic | Datadog | Elastic | Jaeger | Grafana
        (instrument once)        (route/process)              (swap backends freely)
```
- Instrument once with OTel; point the **Collector** at whichever APM/backend you want.
- Spring Boot's **Micrometer Tracing** can bridge to OTel (9.2 §5).
- ⭐ This is the modern recommendation: **OTel for portability**, APM vendor for the polished UI/analytics — without locking your telemetry to one vendor's proprietary agent.

---

## 5. Error Tracking with Sentry

APM focuses on *performance*; **error tracking** (Sentry being the best known) focuses on **exceptions** — capturing, **de-duplicating/grouping**, and alerting on errors with rich context for debugging.

| Sentry feature | What it gives you |
|----------------|-------------------|
| **Exception capture** | Full stack trace + local variables + request context |
| **Grouping/fingerprinting** | Aggregates the *same* error into one issue (not 10,000 duplicate emails) |
| **Release tracking** | Which deploy introduced the error (ties to CI/CD — Phase 10.3) |
| **Context & breadcrumbs** | User, request, tags, the events leading up to the error |
| **Alerting** | Notify on new/regressed/spiking errors (Slack, PagerDuty) |
| **Source integration** | Suspect commit / assignee |

```java
// Spring Boot integration (sentry-spring-boot-starter) — auto-captures unhandled exceptions
// application.yml:
sentry:
  dsn: https://<key>@o123.ingest.sentry.io/456
  environment: production
  traces-sample-rate: 0.1        // Sentry also does lightweight performance/tracing
```
> ⭐ **Logs (9.1) record errors; Sentry makes them *actionable*** — grouped, deduped, with the stack trace + variables + which release + how many users affected. ⚠️ As with logs, **scrub PII/secrets** before sending to Sentry (Phase 15.3) — Sentry has data-scrubbing settings; configure them. Sentry complements (doesn't replace) APM and metrics.

---

## 6. APM vs the OSS Stack — Choosing

| Factor | OSS (Prometheus/Grafana/Loki/Jaeger — 9.1/9.2) | Commercial APM |
|--------|-----------------------------------------------|----------------|
| Cost | Infra + engineering time | Subscription ($$$, scales with hosts/data) |
| Setup/maintenance | You operate it | Managed/SaaS |
| Code-level depth | Manual (profilers — Phase 13.1) | Automatic, deep |
| Lock-in | Low | Higher (mitigate with OTel) |
| Best for | Cost-sensitive, OSS-savvy teams, full control | Teams wanting fast, deep, unified insight |

> ⭐ Common real-world pattern: **OTel instrumentation** + a **managed backend** (APM SaaS *or* Grafana Cloud) + **Sentry** for errors. Start simple (Actuator + Micrometer + logs — 9.1/9.2), add APM/Sentry as scale and on-call pain grow. (This is an **awareness-level** topic — know the landscape and trade-offs; you rarely build an APM, you adopt one.)

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Stacking multiple APM agents | One agent; avoid conflicts/overhead (§2) |
| Proprietary-only instrumentation → lock-in | Prefer OpenTelemetry (§4) |
| Sending PII/secrets to APM/Sentry | Scrub/redact (§5, Phase 15.3) |
| Treating Sentry as a metrics tool | Sentry = error tracking; use Micrometer for metrics (9.2) |
| Logging-as-error-alerting | Use Sentry/alerting for actionable, grouped errors (§5) |
| Ignoring agent overhead/sampling | Tune sampling (9.2 §5); measure cost |
| APM without correlating to logs/traces | Share traceId across pillars (§0, 9.2) |
| Assuming APM replaces good design | APM finds problems; you still fix N+1s, tuning (Phase 13) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Builds on 9.1 (logs) + 9.2 (metrics/traces)** — APM unifies all three.
- **Java agent** uses JVM instrumentation (Phase 1.14) and ships in your **container image** (Phase 10.1) / set via `JAVA_TOOL_OPTIONS`.
- **OpenTelemetry** bridges from **Micrometer Tracing** (9.2) and is the portability layer (also Phase 12.7).
- **Sentry** ties errors to **releases** from **CI/CD** (Phase 10.3).
- **Performance tuning** (Phase 13.1) acts on what APM reveals (slow methods, N+1 — Phase 5.4, GC — Phase 1.11).
- **Microservices observability** (Phase 12.7) often standardizes on OTel + an APM/aggregator.
- **Security** (Phase 15.3) — scrub PII/secrets from telemetry.
- Relevant to **Projects 5–7** when adding production-grade monitoring.

---

## 9. Quick Self-Check Questions

1. What does an APM platform add over a DIY Prometheus/Grafana/Jaeger stack?
2. How does the Java agent model work, and why does it need no code changes?
3. Which JVM flag attaches an agent, and what underlying mechanism does it use?
4. Name three major APM platforms and a distinguishing trait of each.
5. What is OpenTelemetry's role, and how does it mitigate vendor lock-in?
6. What problem does Sentry solve that plain logging doesn't (grouping/context)?
7. What must you scrub before sending data to APM/Sentry, and why?
8. When would you choose the OSS stack vs a commercial APM?
9. Why shouldn't you run multiple APM agents at once?
10. Where does APM fit relative to actually fixing performance (Phase 13)?

---

## 10. Key Terms Glossary

- **APM (Application Performance Monitoring):** all-in-one platform for perf + the three pillars + code insight.
- **Java agent (`-javaagent`):** jar that instruments bytecode at class load (no code changes).
- **Auto-instrumentation:** automatic capture of web/DB/HTTP/messaging spans.
- **New Relic / Datadog / Elastic APM:** major commercial APM platforms.
- **OpenTelemetry (OTel) / Collector:** vendor-neutral telemetry standard / routing component.
- **Sentry:** error-tracking platform (capture, group/dedupe, release tracking).
- **Fingerprinting/grouping:** aggregating identical errors into one issue.
- **Release tracking:** mapping errors to the deploy that introduced them.
- **Service/dependency map:** auto-generated topology of services & calls.
- **Vendor lock-in:** dependence on one vendor's proprietary format (mitigated by OTel).
- **Data scrubbing:** removing PII/secrets from telemetry before export.

---

*This is the note for **Section 9.3 — APM (Application Performance Monitoring)**.*
*Previous section in roadmap: **9.2 Monitoring & Observability**.*
*This completes **Phase 9 — Logging & Monitoring**.*
*Next: **Phase 10 — Containerization & Deployment**.*
