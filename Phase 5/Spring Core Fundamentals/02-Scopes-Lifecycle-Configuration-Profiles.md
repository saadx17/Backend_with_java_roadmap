# Spring Core: Bean Scopes, Lifecycle, Configuration & Profiles

> **Phase 5 — Spring Framework & Spring Boot → 5.1 Spring Core Fundamentals**
> Goal: Master bean scopes, the bean lifecycle, externalized configuration (`@Value`, `@ConfigurationProperties`, properties vs YAML), profiles, SpEL, and application events.

---

## 0. The Big Picture

Once you can declare and wire beans (previous note), you need to control **how many instances exist** (scopes), **what happens during their life** (lifecycle), and **how to configure them externally** (properties, profiles). This is the configuration backbone of every Spring app.

```
Scope:        how many bean instances? (singleton, prototype, request...)
Lifecycle:    init/destroy hooks
Configuration: externalize values (@Value, @ConfigurationProperties, application.yml)
Profiles:     different config per environment (dev/test/prod)
```

---

## 1. Bean Scopes

A **scope** defines **how many instances** of a bean exist and how long they live.
| Scope | Instances | Lifetime |
|-------|-----------|----------|
| **`singleton`** (default) | **One** per container | Whole application |
| **`prototype`** | **New instance each time** requested | Per request for the bean |
| **`request`** (web) | One per HTTP request | The request |
| **`session`** (web) | One per HTTP session | The session |
| **`application`** (web) | One per ServletContext | The app |
| Custom | User-defined | — |

### 1.1 Singleton (the default — critical to understand)
```java
@Service                            // singleton by default — ONE instance shared everywhere
public class UserService { ... }
```
> ⚠️ **Spring beans are singletons by default** — **one shared instance** handles **all concurrent requests** (each on its own thread — recall Phase 1.10). Therefore **singleton beans MUST be stateless (or thread-safe)** — any mutable instance field shared across requests is a **race condition** (recall Phase 1.10 — "singleton beans are shared across request threads"). This is one of the most important Spring gotchas. Keep beans stateless; for per-request state, use method parameters/local variables.

### 1.2 Prototype
```java
@Service
@Scope("prototype")                 // a NEW instance every time it's injected/requested
public class ShoppingCart { ... }
```
Use when each consumer needs its own instance with state.
> ⚠️ Injecting a prototype into a singleton is tricky — the singleton gets **one** prototype instance at creation (not a new one per call). Use `ObjectProvider`/`@Lookup`/scoped proxies if you truly need a fresh instance each time.

### 1.3 Web scopes
`request`/`session` beans live per HTTP request/session — useful for per-user/per-request state (e.g., a request-scoped context object).

---

## 2. Bean Lifecycle

Spring manages a bean from creation to destruction. The phases:
```
1. Instantiation       (the container creates the bean — constructor)
2. Dependency Injection (wire dependencies — previous note)
3. Initialization callbacks (@PostConstruct / InitializingBean / init-method)
4. Bean is READY and in use
5. Destruction callbacks (@PreDestroy / DisposableBean) — on shutdown
```
### 2.1 Lifecycle hooks
```java
@Service
public class CacheService {
    private Map<String, Object> cache;

    @PostConstruct                  // runs AFTER construction + injection (setup)
    public void init() {
        cache = loadInitialData();  // e.g., warm a cache, open a connection
    }

    @PreDestroy                     // runs BEFORE the bean is destroyed (cleanup)
    public void cleanup() {
        cache.clear();              // release resources, close connections
    }
}
```
| Hook | When | Use |
|------|------|-----|
| **`@PostConstruct`** | After construction & DI | Initialization (warm caches, validate config) |
| **`@PreDestroy`** | Before bean destruction (app shutdown) | Cleanup (close resources) |
> ⚠️ Don't do init logic in the constructor that depends on injected fields — at construction time, **field-injected** dependencies aren't set yet (another reason to prefer constructor injection — previous note). Use `@PostConstruct` for setup that needs the wired dependencies. `@PreDestroy` ties to **graceful shutdown** (recall SIGTERM, Phase 0.3/1.10).

---

## 3. Externalized Configuration

Hardcoding config (URLs, credentials, settings) is bad — externalize it so the **same code** runs in different environments with different config (12-factor — recall Phase 0.3 env vars).

### 3.1 Property sources
Spring Boot reads config from `application.properties` or `application.yml` (and env vars, command-line args, etc.):
```properties
# application.properties
app.name=MyApp
app.max-users=1000
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
```
```yaml
# application.yml (equivalent, hierarchical — preferred for nested config)
app:
  name: MyApp
  max-users: 1000
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
```
| | `.properties` | `.yml` |
|---|---------------|--------|
| Format | flat key=value | hierarchical (indented) |
| Best for | simple config | nested config (preferred for structure) |
| Lists/maps | awkward | natural |

### 3.2 `@Value` (inject a single property)
```java
@Service
public class AppService {
    @Value("${app.name}")                       // inject a property value
    private String appName;

    @Value("${app.max-users:500}")              // with a default (500 if missing)
    private int maxUsers;

    // (Better via constructor for testability:)
    public AppService(@Value("${app.name}") String appName) { ... }
}
```

### 3.3 `@ConfigurationProperties` (type-safe, grouped — preferred)
For a group of related properties, bind them to a typed object (cleaner than many `@Value`s):
```java
@ConfigurationProperties(prefix = "app")
public record AppProperties(String name, int maxUsers, String version) {}   // (record — Phase 1.3)
// binds app.name, app.max-users, app.version

// usage:
@Service
public class AppService {
    private final AppProperties props;
    public AppService(AppProperties props) { this.props = props; }   // inject the typed config
}
```
| | `@Value` | `@ConfigurationProperties` |
|---|----------|----------------------------|
| Scope | single property | a group of related properties |
| Type safety | string-based | **strongly typed** (validated) |
| Relaxed binding | no | yes (`maxUsers` ↔ `max-users` ↔ `MAX_USERS`) |
| Best for | one-off values | structured config (preferred) |
> **Prefer `@ConfigurationProperties`** for grouped config — it's type-safe, supports validation (`@Validated`), and **relaxed binding** (matches `max-users`, `maxUsers`, `MAX_USERS` env var — recall Phase 0.3 env vars). Records make immutable config (Phase 1.3).

### 3.4 Configuration precedence (important)
Spring Boot reads config from many sources in a **priority order** (higher overrides lower):
```
1. Command-line args              (highest)
2. Environment variables / system properties
3. application-{profile}.yml      (profile-specific)
4. application.yml                 (default)
5. defaults in code               (lowest)
```
> ⚠️ This precedence is why **environment variables override `application.yml`** — the 12-factor way to configure per environment without changing files. E.g., set `SPRING_DATASOURCE_URL` as an env var in production (Docker/K8s — Phase 10) to override the dev value in `application.yml`. **Never hardcode secrets** — inject them via env vars / a secrets manager (Phase 15).

---

## 4. Profiles (Environment-Specific Config)

A **profile** is a named set of config/beans active in a specific environment (dev, test, prod). Lets you run the **same code** with different config/beans per environment.

### 4.1 Profile-specific config files
```
application.yml           # common config (always loaded)
application-dev.yml       # active only when the 'dev' profile is on
application-prod.yml      # active only when the 'prod' profile is on
```
```yaml
# application.yml
spring:
  profiles:
    active: dev           # which profile(s) to activate (often set via env var instead)
```
Activate via env var (preferred for prod): `SPRING_PROFILES_ACTIVE=prod` (recall Phase 0.3).

### 4.2 Profile-specific beans
```java
@Bean
@Profile("dev")
public DataSource devDataSource() { return new H2DataSource(); }   // in-memory DB for dev

@Bean
@Profile("prod")
public DataSource prodDataSource() { return new HikariDataSource(...); }   // real DB for prod
```
> Profiles let you, e.g., use an in-memory database in dev/test (Phase 6) and PostgreSQL in prod, or enable verbose logging in dev only. Activate the profile via `SPRING_PROFILES_ACTIVE` (env var — Phase 10) rather than hardcoding it.

---

## 5. Spring Expression Language (SpEL)

**SpEL** is a small expression language usable in annotations — for dynamic values, conditions, and references:
```java
@Value("#{systemProperties['user.name']}")     // a SpEL expression (#{...})
private String osUser;

@Value("#{${app.max-users} * 2}")               // compute from a property
private int doubled;

@Scheduled(cron = "${app.cron}")                // property placeholder
// @PreAuthorize("hasRole('ADMIN')")             // SpEL in security (Phase 5.6)
```
> `${...}` = property placeholder; `#{...}` = SpEL expression. SpEL appears in `@Value`, security expressions (`@PreAuthorize` — Phase 5.6), caching keys (`@Cacheable(key=...)` — Phase 5.7), and more. Awareness-level — you'll use simple SpEL often.

---

## 6. Application Events

Spring has a built-in **event/listener** mechanism for **decoupled** communication between beans (the Observer pattern — Phase 14):
```java
// 1. Define an event (a record):
public record UserRegisteredEvent(Long userId, String email) {}

// 2. Publish it:
@Service
public class UserService {
    private final ApplicationEventPublisher publisher;
    public UserService(ApplicationEventPublisher publisher) { this.publisher = publisher; }
    public void register(User user) {
        // ... save user ...
        publisher.publishEvent(new UserRegisteredEvent(user.getId(), user.getEmail()));
    }
}

// 3. Listen for it (in a different, decoupled bean):
@Component
public class WelcomeEmailListener {
    @EventListener                              // handles the event
    public void onUserRegistered(UserRegisteredEvent event) {
        sendWelcomeEmail(event.email());
    }
    // @Async @EventListener -> handle asynchronously (on another thread — Phase 1.10)
    // @TransactionalEventListener -> handle only after the transaction commits
}
```
> Events **decouple** the publisher from handlers — `UserService` doesn't know who reacts to registration (could be email, analytics, audit). This is the in-process version of event-driven architecture (Phase 11). `@TransactionalEventListener` (fire only after commit) is especially useful with `@Transactional` (Phase 5.5).

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Mutable state in a singleton bean | Keep beans stateless (thread-safety! Phase 1.10) |
| Init logic in constructor needing injected fields | Use `@PostConstruct` |
| Hardcoding config/secrets | Externalize (`@ConfigurationProperties`, env vars) |
| Many `@Value` for related config | Use `@ConfigurationProperties` (type-safe) |
| Hardcoding the active profile | Set `SPRING_PROFILES_ACTIVE` via env var |
| Injecting a prototype into a singleton expecting fresh instances | Use `ObjectProvider`/scoped proxy |
| Secrets in `application.yml` (committed!) | Use env vars / secrets manager (Phase 15) |
| Forgetting config precedence | Env vars override YAML — know the order |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Stateless singletons** are essential for thread-safe, scalable services (Phase 1.10, 13).
- **`@ConfigurationProperties` + env vars** is the 12-factor config approach for Docker/K8s (Phase 10) — recall Phase 0.3 env vars.
- **Profiles** separate dev/test/prod config and beans (Phase 6 testing, Phase 10 deployment).
- **`@PostConstruct`/`@PreDestroy`** + graceful shutdown (SIGTERM, Phase 0.3/10) for clean startup/cleanup.
- **Application events** are the in-process precursor to messaging/event-driven architecture (Phase 11) and the Outbox pattern (Phase 12).
- **SpEL** powers security (`@PreAuthorize`, Phase 5.6) and cache keys (Phase 5.7).
- **Never commit secrets** in config (Phase 15, recall Phase 0.4 .gitignore).

---

## 9. Quick Self-Check Questions

1. What's the default bean scope, and why must such beans be stateless?
2. When would you use prototype vs singleton scope?
3. What are `@PostConstruct` and `@PreDestroy` for, and why not use the constructor?
4. What's the difference between `@Value` and `@ConfigurationProperties`? Which is preferred?
5. Why is `.yml` often preferred over `.properties`?
6. What is configuration precedence, and why do env vars override YAML?
7. How do profiles work, and how do you activate one?
8. What are application events for, and what pattern do they implement?

---

## 10. Key Terms Glossary

- **Bean scope:** how many instances exist (singleton/prototype/request/session).
- **Singleton (default):** one shared instance — must be stateless.
- **Bean lifecycle:** instantiate → inject → init → use → destroy.
- **`@PostConstruct` / `@PreDestroy`:** init / cleanup hooks.
- **Externalized configuration:** config outside code (`application.yml`, env vars).
- **`@Value`:** inject a single property (`${...}`).
- **`@ConfigurationProperties`:** type-safe, grouped config binding.
- **Relaxed binding:** matching `max-users`/`maxUsers`/`MAX_USERS`.
- **Configuration precedence:** order config sources override each other.
- **Profile (`@Profile`):** environment-specific config/beans.
- **SpEL:** Spring Expression Language (`#{...}`).
- **Application event / `@EventListener`:** decoupled in-process pub/sub.

---

*Previous topic: **IoC, DI & Beans**.*
*Next topic: **Aspect-Oriented Programming (AOP)**.*
