# Spring Scheduling

> **Phase 5 — Spring Framework & Spring Boot → 5.8 Spring Scheduling**
> Goal: Master scheduled tasks in Spring — `@EnableScheduling`/`@Scheduled` (fixedRate, fixedDelay, cron) — and distributed scheduling with ShedLock.

---

## 0. The Big Picture

**Scheduling** runs code **automatically on a schedule or interval** — periodic jobs like cleanups, report generation, data syncs, and health checks. Spring's `@Scheduled` makes this a one-annotation feature (recall cron, Phase 0.3 — this is the in-app equivalent).

```
@Scheduled(cron = "0 0 2 * * *")        // run every day at 2 AM
public void nightlyCleanup() { ... }
```

> Scheduled tasks are common in real backends (token cleanup, sending digests, recalculating stats). This builds on cron (Phase 0.3) and the executor framework (Phase 1.10 — scheduling runs on a thread pool).

---

## 1. Enabling Scheduling

Enable with `@EnableScheduling`, then annotate methods with `@Scheduled`:
```java
@Configuration
@EnableScheduling                       // turn on scheduling support
public class SchedulingConfig {}

@Component
public class CleanupTasks {
    @Scheduled(fixedRate = 60000)       // every 60 seconds
    public void task() { ... }
}
```
> `@Scheduled` methods must be in **Spring beans** (Phase 5.1), take **no arguments**, and usually return `void`. Spring runs them on a scheduled executor (Phase 1.10) — by default a single thread (§4).

---

## 2. Scheduling Types

`@Scheduled` supports three timing modes (recall `ScheduledExecutorService`, Phase 1.10):
| Attribute | Behavior |
|-----------|----------|
| **`fixedRate`** | Run every N ms, measured from the **start** of each run (regardless of duration) |
| **`fixedDelay`** | Run N ms **after the previous run finishes** |
| **`cron`** | Run on a cron schedule (specific times) |
```java
@Scheduled(fixedRate = 5000)            // start a new run every 5s (even if the last is still running*)
public void poll() { ... }

@Scheduled(fixedDelay = 5000)           // wait 5s AFTER each run completes, then run again
public void process() { ... }

@Scheduled(fixedDelay = 5000, initialDelay = 10000)   // wait 10s before the first run
public void delayed() { ... }

@Scheduled(cron = "0 0 2 * * *")        // every day at 2:00 AM (cron — §3)
public void nightly() { ... }
```

### 2.1 fixedRate vs fixedDelay (the key distinction)
```
fixedRate=5s:   |run|---wait---|run|---wait---|run|     (every 5s from each START)
fixedDelay=5s:  |run|--5s--|run|--5s--|run|              (5s GAP after each finishes)
```
> ⚠️ **`fixedRate`** triggers on a fixed interval from the start (good for steady polling); **`fixedDelay`** ensures a gap *between* runs (good when runs vary in length and shouldn't overlap/pile up). *Note: by default `@Scheduled` tasks run on a single thread, so a long `fixedRate` task delays the next (no overlap) — §4.

---

## 3. Cron Expressions

Spring's `cron` uses a **6-field** format (recall the 5-field Unix cron, Phase 0.3 — Spring adds **seconds**):
```
@Scheduled(cron = "second minute hour day-of-month month day-of-week")
                    0      0      2      *           *      *           -> 2:00:00 AM daily
```
| Field | Range |
|-------|-------|
| Second | 0–59 |
| Minute | 0–59 |
| Hour | 0–23 |
| Day of month | 1–31 |
| Month | 1–12 |
| Day of week | 0–7 (0/7 = Sunday) |
```java
@Scheduled(cron = "0 0 2 * * *")         // 2 AM every day
@Scheduled(cron = "0 */15 * * * *")      // every 15 minutes
@Scheduled(cron = "0 0 9 * * MON-FRI")   // 9 AM on weekdays
@Scheduled(cron = "0 0 0 1 * *")         // midnight on the 1st of each month
@Scheduled(cron = "${app.cleanup.cron}") // from config (SpEL placeholder — Phase 5.1)
@Scheduled(cron = "0 0 3 * * *", zone = "America/New_York")  // with a timezone (recall Phase 4.1!)
```
> ⚠️ Spring's cron has **6 fields** (with seconds) — the Unix cron from Phase 0.3 has **5** (no seconds). Watch the difference. Externalize the cron to config (`${...}` — Phase 5.1) so it's tunable without recompiling. Specify a **`zone`** for timezone correctness (recall Phase 4.1 — UTC vs local time).

---

## 4. Thread Pool for Scheduling

> ⚠️ **By default, all `@Scheduled` tasks run on a SINGLE thread.** If one task runs long, it **delays** all others (they queue). For concurrent scheduled tasks, configure a pool (recall thread pools, Phase 1.10):
```java
@Configuration
public class SchedulingConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);                 // 5 threads -> tasks can run concurrently
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```
Or simply: `spring.task.scheduling.pool.size=5` in `application.yml`.
> A single thread means long-running or many tasks block each other — a common surprise. Size the pool for your number of concurrent tasks (recall pool sizing, Phase 1.10/13).

---

## 5. The Distributed Scheduling Problem (ShedLock)

> ⚠️ **In a multi-instance deployment (multiple app instances behind a load balancer — Phase 0.2/13), a `@Scheduled` task runs on EVERY instance — so it executes N times instead of once!**
```
3 app instances, each with @Scheduled nightly cleanup:
  -> the cleanup runs 3 TIMES at 2 AM (once per instance) -> duplicate work / bugs!
```
This is a serious problem for tasks that should run **exactly once** (sending emails, billing, cleanup). The solution: **distributed locking**.

### 5.1 ShedLock
**ShedLock** ensures a scheduled task runs on **only one instance at a time** by using a shared lock (in the DB, Redis, etc. — recall distributed locks, Phase 4.8/12.6):
```java
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "nightlyCleanup",          // a named lock in a shared store
               lockAtMostFor = "10m",            // safety: release after 10m if the holder dies
               lockAtLeastFor = "1m")            // hold at least 1m (prevent rapid re-run)
public void nightlyCleanup() { ... }
```
> ShedLock acquires a **shared lock** before running — only the instance that gets the lock executes; the others skip. `lockAtMostFor` is a safety net (release if an instance crashes mid-task — recall deadlock/lock-timeout, Phase 4.5). This is essential for scheduled tasks in **clustered/multi-instance** deployments (Phase 10/12). (Alternatives: a dedicated scheduler service, Quartz clustering.)

---

## 6. Scheduling vs Other Async Mechanisms

Don't confuse scheduling with related Spring features:
| Feature | Purpose |
|---------|---------|
| **`@Scheduled`** | Run a task on a **time schedule/interval** |
| **`@Async`** (Phase 1.10) | Run a method **asynchronously** (on another thread, on demand) |
| **Message queues** (Phase 11) | Process work from a queue (event-driven) |
| **Batch jobs** (Spring Batch) | Large-scale batch processing |
> `@Scheduled` = "do X at time T / every N." `@Async` = "do X in the background now." For heavy/distributed background processing, use **message queues** (Phase 11) rather than scheduling.

---

## 7. Common Use Cases
| Task | Schedule |
|------|----------|
| Clean up expired tokens/sessions | Nightly cron (Project 4!) |
| Send digest emails / notifications | Daily/weekly cron |
| Recompute stats / refresh materialized views (Phase 4.6) | Periodic |
| Sync data from an external system | fixedDelay/cron |
| Health checks / heartbeats | fixedRate |
| Retry failed operations | fixedDelay |

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Scheduled task running on every instance | Use ShedLock (distributed lock) |
| Single-thread default blocking tasks | Configure a scheduling pool |
| Confusing fixedRate and fixedDelay | Rate = from start; delay = gap after finish |
| Spring cron (6 fields) vs Unix cron (5) | Spring adds seconds |
| Hardcoding the schedule | Externalize to config (`${...}`) |
| Ignoring timezones | Set `zone`; store/compute in UTC (Phase 4.1) |
| Long task with no error handling | Wrap in try/catch — an uncaught exception can stop future runs |
| Heavy distributed processing via @Scheduled | Use message queues (Phase 11) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Project 4: "scheduled cleanup of expired tokens"** — `@Scheduled` + ShedLock + JWT (Phase 5.6).
- **Distributed scheduling (ShedLock)** uses distributed locks (Phase 4.8 Redis / 12.6 Redisson) — essential in multi-instance deployments (Phase 10/12).
- **Cron** builds on Phase 0.3 (with the 6-field difference).
- **Thread pool sizing** (Phase 1.10/13) for concurrent tasks.
- **Refreshing materialized views/caches** (Phase 4.6, 5.7) on a schedule.
- **`@Async`** (Phase 1.10) is the on-demand counterpart.
- **Message queues** (Phase 11) for scalable background processing beyond scheduling.

---

## 10. Quick Self-Check Questions

1. How do you enable and define a scheduled task?
2. What's the difference between `fixedRate`, `fixedDelay`, and `cron`?
3. How does Spring's cron format differ from Unix cron (Phase 0.3)?
4. Why do scheduled tasks run on a single thread by default, and how do you change it?
5. What problem occurs with `@Scheduled` in a multi-instance deployment?
6. What is ShedLock, and how does it solve it?
7. What does `lockAtMostFor` protect against?
8. When use scheduling vs `@Async` vs message queues?

---

## 11. Key Terms Glossary

- **`@EnableScheduling` / `@Scheduled`:** enable / define scheduled tasks.
- **`fixedRate` / `fixedDelay` / `cron`:** interval-from-start / gap-after-finish / cron schedule.
- **`initialDelay`:** delay before the first run.
- **Cron expression (6-field):** second minute hour day month day-of-week.
- **`zone`:** timezone for the schedule.
- **Scheduling thread pool:** lets tasks run concurrently.
- **Distributed scheduling:** ensuring a task runs once across instances.
- **ShedLock / `@SchedulerLock`:** distributed lock for scheduled tasks.
- **`lockAtMostFor` / `lockAtLeastFor`:** lock safety bounds.

---

*This is the note for **Section 5.8 — Spring Scheduling**.*
*Next section in roadmap: **5.9 Spring WebClient**.*
