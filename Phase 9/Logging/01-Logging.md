# Logging (SLF4J, Logback, MDC & Structured Logging)

> **Phase 9 — Logging & Monitoring → 9.1 Logging**
> Goal: Master application logging — log **levels** and when to use each, the **SLF4J** facade with parameterized messages, **MDC** for request context, **Logback** configuration (`logback-spring.xml`, appenders, rolling policies, filters), **structured JSON** logging, and best practices for *what NOT to log*.

---

## 0. The Big Picture

Logging is how a running backend **tells you what it's doing**. Unlike a debugger (Phase 1.1 — only works locally), logs are your eyes in production: tracing a request, diagnosing an error at 3 a.m., proving what happened. The goal is logs that are **leveled** (filter by severity), **structured** (machine-parseable), **contextual** (which request/user/trace?), and **safe** (no secrets/PII).

```
   Your code                Facade            Implementation        Destination
   ─────────                ──────            ──────────────        ───────────
   log.info("...")  ──►   SLF4J API   ──►   Logback (default)  ──►  Console / File / JSON
                          (interface)        (the engine)            ──► aggregator (ELK/Loki — 9.2/12.7)
```

> Logging is the **first pillar of observability** (logs + metrics + traces — 9.2). It connects to **MDC ↔ distributed tracing** (9.2/12.7), gets shipped to **ELK/Loki** for searching (12.7), and is a **security** concern (don't log secrets — Phase 15.3). In Spring Boot, Logback + SLF4J come preconfigured via `spring-boot-starter` (Phase 5.2).

---

## 1. Log Levels

Levels rank severity so you can **filter**: set the threshold to `INFO` in prod and `DEBUG`/`TRACE` only when investigating.

| Level | Meaning | Use for | Prod default? |
|-------|---------|---------|:-------------:|
| **TRACE** | Finest detail | Step-by-step flow, loop internals | ❌ off |
| **DEBUG** | Diagnostic detail | Variable values, branch decisions (dev) | ❌ off |
| **INFO** | Normal significant events | Startup, request handled, order placed | ✅ on |
| **WARN** | Something odd but recovered | Retry succeeded, deprecated call, fallback used | ✅ on |
| **ERROR** | A failure needing attention | Unhandled exception, failed transaction | ✅ on |
| (FATAL) | (Logback maps to ERROR) | App-down conditions | — |

```
TRACE < DEBUG < INFO < WARN < ERROR     (left = noisier/lower, right = severe/higher)
Threshold = INFO  →  logs INFO, WARN, ERROR ; suppresses DEBUG, TRACE
```
> ⭐ **Pick the right level.** ⚠️ Logging everything at `INFO`/`ERROR` creates noise (alert fatigue) and cost; under-logging blinds you. `WARN` = recovered/expected-but-notable; `ERROR` = something broke and a human may need to act. Don't use `ERROR` for handled validation failures (that's normal traffic → `INFO`/`DEBUG`).

---

## 2. SLF4J — the Logging Facade

**SLF4J** (Simple Logging Facade for Java) is an **abstraction/interface**; you code against it, and a real engine (Logback, Log4j2) provides the implementation at runtime. This is the **bridge/facade pattern** (Phase 14.1) — swap the backend without changing code.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    // (Lombok shortcut: @Slf4j generates this `log` field — Phase 5)

    public void placeOrder(Order order) {
        log.info("Placing order id={} total={}", order.getId(), order.getTotal());  // parameterized
        try {
            // ...
        } catch (PaymentException e) {
            log.error("Payment failed for order id={}", order.getId(), e);  // pass exception LAST
        }
    }
}
```

### 2.1 Parameterized logging ⭐ (always do this)
```java
log.debug("User {} bought {} items for {}", userId, count, total);   // ✅ placeholders
log.debug("User " + userId + " bought " + count + " items");          // ❌ string concatenation
```
| Why parameterized (`{}`) wins | |
|--------------------------------|--|
| **Performance** | The string is only built **if** the level is enabled — no wasted concatenation when DEBUG is off (avoids the `isDebugEnabled()` guard) |
| **Safety** | No accidental `null + ""` issues |
| **Readability** | Message template stays clean; great for structured logging (§5) |
| **Exceptions** | Pass the `Throwable` as the **last** arg (no `{}` for it) → full stack trace logged |

> ⚠️ The classic anti-pattern: `log.debug("x=" + expensiveToString())` builds the string **even when DEBUG is disabled**. With `{}` placeholders, SLF4J skips the work entirely when the level is off. This matters in hot paths (Phase 13.1).

---

## 3. MDC — Mapped Diagnostic Context

**MDC** (Mapped Diagnostic Context) is a per-thread key-value map that **automatically attaches context** (request id, user id, trace id) to **every** log line in that thread — so you don't have to pass it into every `log.info(...)`.

```java
import org.slf4j.MDC;

// e.g., in a servlet Filter / interceptor (Phase 5.3), at the start of each request:
MDC.put("traceId", traceId);          // or UUID.randomUUID().toString()
MDC.put("userId", String.valueOf(currentUserId));
try {
    chain.doFilter(req, res);         // every log inside this request now carries traceId & userId
} finally {
    MDC.clear();                       // ⚠️ ALWAYS clear — threads are reused from a pool!
}
```
Then reference MDC keys in the log pattern (§4): `%X{traceId}`.
```
2026-06-16 09:30:01 INFO  [traceId=a1b2c3 userId=42] OrderService - Placing order id=99
2026-06-16 09:30:01 INFO  [traceId=a1b2c3 userId=42] PaymentService - Charged 59.90
                              ▲ same traceId stitches the whole request together
```

| MDC point | Note |
|-----------|------|
| Per-thread storage | Context is bound to the current thread (a `ThreadLocal`) |
| ⚠️ Always `MDC.clear()` (finally) | Pooled threads reuse → stale context leaks into the next request |
| ⚠️ Async/reactive caveat | Work on another thread (Phase 5.3 `@Async`, WebFlux 16.1) loses MDC — propagate it manually or use context-propagation libs |
| Filled automatically by tracing | Micrometer Tracing / Sleuth put `traceId`/`spanId` in MDC (9.2/12.7) |

> ⭐ **MDC is the glue between logging and distributed tracing.** A `traceId` in every log line lets you grep all logs for one request across services (12.7). Spring Boot's tracing auto-populates `traceId`/`spanId` into the MDC and the default log pattern.

---

## 4. Logback Configuration

Logback is Spring Boot's **default** logging engine. Configure it with **`logback-spring.xml`** in `src/main/resources` (the `-spring` variant lets you use Spring features like `<springProfile>` — Phase 5.1).

```xml
<configuration>

  <!-- 1) Console appender (human-readable, for local/dev & containers — Phase 10) -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level [%X{traceId:-}] %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <!-- 2) Rolling file appender (size + time based rotation) -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>logs/app-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>  <!-- daily + gz -->
      <maxFileSize>100MB</maxFileSize>      <!-- roll at 100MB -->
      <maxHistory>30</maxHistory>           <!-- keep 30 days -->
      <totalSizeCap>3GB</totalSizeCap>      <!-- cap total disk usage -->
    </rollingPolicy>
    <encoder><pattern>%d %-5level [%X{traceId:-}] %logger - %msg%n</pattern></encoder>
  </appender>

  <!-- 3) Level filter example: only WARN+ to a separate appender -->
  <!-- <filter class="ch.qos.logback.classic.filter.ThresholdFilter"><level>WARN</level></filter> -->

  <!-- Per-package levels -->
  <logger name="com.example" level="DEBUG"/>
  <logger name="org.hibernate.SQL" level="DEBUG"/>   <!-- see generated SQL (Phase 5.4) -->

  <!-- Profile-specific config (Phase 5.1) -->
  <springProfile name="prod">
    <root level="INFO"><appender-ref ref="FILE"/></root>   <!-- prod: JSON/file, no console spam -->
  </springProfile>
  <springProfile name="dev">
    <root level="DEBUG"><appender-ref ref="CONSOLE"/></root>
  </springProfile>

</configuration>
```

| Concept | Meaning |
|---------|---------|
| **Appender** | *Where* logs go — `ConsoleAppender`, `RollingFileAppender`, async, custom |
| **Encoder / pattern** | *How* a line is formatted (`%d` date, `%level`, `%logger`, `%msg`, `%X{key}` MDC, `%n` newline) |
| **Rolling policy** | When/how to rotate files (size, time, both) + retention (`maxHistory`, `totalSizeCap`) |
| **Filter** | Include/exclude by level/condition (e.g., `ThresholdFilter`, `LevelFilter`) |
| **Logger (named)** | Per-package level override |
| **Root logger** | Default level + appenders for everything |
| **`<springProfile>`** | Profile-conditional config (needs `logback-spring.xml`, not `logback.xml`) |

> ⭐ Spring Boot also exposes simple props for the common cases (no XML needed): `logging.level.com.example=DEBUG`, `logging.file.name=app.log`, `logging.pattern.console=...` in `application.yml` (Phase 5.2). Use `logback-spring.xml` when you need appenders/rolling/JSON/profiles.

### 4.1 Async appender (performance)
Wrap an appender in `AsyncAppender` so logging I/O happens off the request thread (a bounded queue) — reduces latency impact in hot paths (Phase 13.1). ⚠️ It can **drop** logs when the queue is full (tune `queueSize`/`discardingThreshold`).

---

## 5. Structured (JSON) Logging

In production, logs are shipped to an aggregator (**ELK/Elasticsearch**, **Loki/Grafana**, Datadog — 9.2/9.3/12.7) and **searched**. Plain text is hard to query; **structured JSON** logs are key-value documents you can filter/aggregate (`level:ERROR AND userId:42`).

```jsonc
// one log line = one JSON object
{
  "@timestamp": "2026-06-16T09:30:01.123Z",
  "level": "ERROR",
  "logger": "com.example.OrderService",
  "message": "Payment failed for order id=99",
  "traceId": "a1b2c3",          // from MDC (§3) → correlate with traces (9.2)
  "userId": "42",
  "service": "order-service",   // which microservice (Phase 12)
  "stack_trace": "..."
}
```
How to produce it:
- Add **`logstash-logback-encoder`** and use `LogstashEncoder`/`net.logstash.logback.encoder` as the appender encoder, **or**
- Spring Boot 3.4+ has **built-in structured logging**: `logging.structured.format.console=ecs` (Elastic Common Schema) / `logstash` / `gelf`.

```yaml
# Spring Boot 3.4+ — zero-dependency JSON logs
logging:
  structured:
    format:
      console: ecs        # or logstash / gelf
```

| Text logs | Structured JSON logs |
|-----------|----------------------|
| Human-friendly, easy local | Machine-queryable, aggregation-friendly |
| Hard to filter/aggregate at scale | Filter by any field (level, traceId, userId) |
| Good for dev console | ⭐ Standard for prod + ELK/Loki (12.7) |

> ⭐ Best of both: **text/pretty in dev** (console), **JSON in prod** (via `<springProfile>` — §4). Always include `traceId`/`service` fields so logs join with traces (9.2) and you can search across microservices (12.7).

---

## 6. Best Practices — and What NOT to Log ⚠️

### 6.1 What NOT to log (security — Phase 15.3) ☠️
| Never log | Why |
|-----------|-----|
| **Passwords, secrets, API keys, tokens (JWT)** | Logs are stored/shipped widely → credential leak (Phase 5.6/15) |
| **PII** (full card numbers, SSNs, full emails, addresses) | Privacy/GDPR violation (Phase 15.3) — **mask** (`****1234`) |
| **Full request/response bodies with sensitive fields** | Same as above; redact |
| **Session ids / cookies** | Session hijacking risk |
> ☠️ **Logging a password or token is a security incident.** Mask/redact at the source. Assume logs are readable by ops, in backups, and in the aggregator. (GDPR/PII deep-dive in Phase 15.3.)

### 6.2 General best practices
| Do | Why |
|----|-----|
| Use the **right level** (§1) | Filterable, low noise |
| **Parameterized** messages (§2.1) | Performance + clean |
| Include **context via MDC** (traceId, userId) (§3) | Correlate across lines/services |
| Log **actionable** info on ERROR (what failed + ids + the exception) | Faster diagnosis |
| **Structured JSON** in prod (§5) | Queryable at scale |
| Log at **boundaries** (incoming request, outbound call, errors) | High signal |
| Don't **double-log** (log-and-rethrow then log again) | Noise/duplication |
| Don't log inside tight loops at INFO | Volume/cost |
| Don't use logging for control flow or as a metric | Use metrics (9.2) for counts/rates |
| Set up **log rotation/retention** (§4) | Disk safety, cost |

> ⭐ **Logs answer "what happened to *this* request?"; metrics (9.2) answer "how is the system doing *overall*?"** Don't try to compute rates by counting log lines — emit a metric (a `Counter`/`Timer` — 9.2) instead.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| String concatenation in log calls | Use `{}` parameterized messages (§2.1) |
| Logging secrets/PII/tokens | Never; mask/redact (§6.1, Phase 15.3) |
| Forgetting `MDC.clear()` | Always clear in `finally` (pooled threads) (§3) |
| Using `ERROR` for handled/validation cases | Reserve ERROR for real failures (§1) |
| Expensive `toString()` even when level off | Placeholders skip the work (§2.1) |
| Plain text logs in prod + ELK | Structured JSON (§5) |
| Exception passed via `{}` (no stack trace) | Pass `Throwable` as the **last** arg (§2) |
| No rolling/retention → disk fills | Configure rolling policy + caps (§4) |
| `logback.xml` but expecting `<springProfile>` | Use `logback-spring.xml` (§4) |
| Counting log lines to get metrics | Emit metrics instead (§6.2, 9.2) |
| MDC lost on async/reactive threads | Propagate context (§3, Phase 16.1) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Spring Boot** ships SLF4J + Logback via `spring-boot-starter` (Phase 5.2); `application.yml` props or `logback-spring.xml` configure it.
- **`<springProfile>`** ties logging to **profiles** (Phase 5.1) — pretty in dev, JSON in prod.
- **Filters/interceptors** (Phase 5.3) are where you set MDC per request.
- **MDC `traceId`/`spanId`** is auto-populated by **Micrometer Tracing** and is the bridge to **distributed tracing** (9.2) and **log aggregation** (ELK/Loki — 12.7).
- **Metrics (9.2)** complement logs; **APM (9.3)** consumes both.
- **Security (Phase 15.3)** — never log secrets/PII; masking & GDPR.
- **Performance (Phase 13.1)** — async appenders, avoid hot-path logging cost.
- **Containers (Phase 10)** — log to **stdout** (the 12-factor way) so the platform collects it.
- Used in every project (4–7) for diagnosability.

---

## 9. Quick Self-Check Questions

1. Name the log levels in order; when do you use WARN vs ERROR?
2. What is SLF4J, and why is it a *facade*? What's the default engine in Spring Boot?
3. Why are parameterized (`{}`) messages better than concatenation — performance and safety?
4. How do you log an exception so the stack trace appears?
5. What is MDC, what problem does it solve, and why must you always `MDC.clear()`?
6. What are appenders, encoders/patterns, rolling policies, and filters in Logback?
7. Why use `logback-spring.xml` over `logback.xml`?
8. Why is structured JSON logging preferred in production, and how do you enable it (Boot 3.4+)?
9. List five things you must never log, and what to do instead.
10. What's the division of labor between logs and metrics?

---

## 10. Key Terms Glossary

- **SLF4J:** logging facade/interface you code against.
- **Logback / Log4j2:** logging engine implementations (Logback = Boot default).
- **Log level:** TRACE < DEBUG < INFO < WARN < ERROR (severity threshold).
- **Parameterized logging:** `{}` placeholders; message built only if level enabled.
- **MDC:** per-thread context map → `%X{key}` in logs (e.g., `traceId`).
- **Appender:** where logs go (console, rolling file, async, JSON).
- **Encoder / pattern:** how a line is formatted.
- **Rolling policy:** file rotation by size/time + retention caps.
- **Filter:** include/exclude logs by level/condition.
- **`logback-spring.xml` / `<springProfile>`:** Spring-aware Logback config / profile blocks.
- **Structured logging:** key-value (JSON) logs for aggregation/search.
- **Log aggregation:** shipping logs to ELK/Loki/Datadog for search (12.7).
- **Redaction/masking:** removing/obscuring sensitive data before logging.

---

*This is the note for **Section 9.1 — Logging**.*
*Previous section in roadmap: **8.2 Liquibase**.*
*Next section in roadmap: **9.2 Monitoring & Observability**.*
