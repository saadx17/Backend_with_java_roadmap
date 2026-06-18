# Spring Boot

> **Phase 5 — Spring Framework & Spring Boot → 5.2 Spring Boot**
> Goal: Master Spring Boot — auto-configuration, starters, `@SpringBootApplication`, configuration binding, embedded servers, DevTools, Actuator, fat/layered JARs, and GraalVM native images.

---

## 0. The Big Picture

**Spring Boot** makes Spring **easy to use** by eliminating boilerplate configuration. It provides **sensible defaults**, **auto-configuration**, **starter dependencies**, and an **embedded server** — so you go from zero to a running web app in minutes, with a single runnable JAR.

```
Plain Spring:   lots of manual configuration (XML/Java config, server setup, dependency wiring)
Spring Boot:    convention over configuration -> "just works" with minimal setup
```

> Spring Boot is the standard way to build Spring apps today. It builds directly on Spring Core (Phase 5.1 — IoC/DI/beans) and the build tools (Phase 3 — Maven/Gradle starters & BOM). This note covers what Boot adds *on top of* Spring.

---

## 1. What Spring Boot Adds

| Feature | What it does |
|---------|--------------|
| **Auto-configuration** | Configures beans automatically based on the classpath (§2) |
| **Starters** | Curated dependency bundles (§3) |
| **Embedded server** | Tomcat/Jetty/Undertow built-in — no external deploy (§5) |
| **Opinionated defaults** | Sensible config out of the box; override as needed |
| **Actuator** | Production-ready monitoring endpoints (§7) |
| **Executable JAR** | `java -jar app.jar` — self-contained (§8) |
> Spring Boot is "**convention over configuration**" (recall Maven, Phase 3): it makes decisions for you (which server, which JSON mapper, which connection pool) so you write almost no config — but you can override anything.

---

## 2. Auto-Configuration (The Magic)

> **Auto-configuration is Spring Boot's killer feature:** it automatically configures beans based on **what's on the classpath**. Add a dependency → Boot configures it sensibly.
```
Spring Boot sees postgresql.jar + spring-data-jpa on the classpath
  -> auto-configures a DataSource (HikariCP), an EntityManager, a transaction manager, etc.
You just provide the connection URL — Boot wires the rest.
```

### 2.1 How it works (under the hood)
Auto-configuration is built on **conditional beans** (recall `@Conditional`, Phase 5.1):
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)            // only if DataSource is on the classpath
@ConditionalOnMissingBean(DataSource.class)      // only if YOU haven't defined one
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource(...) { ... }     // Boot provides a sensible default
}
```
| Conditional annotation | Activates the config when... |
|------------------------|------------------------------|
| `@ConditionalOnClass` | A class is present on the classpath |
| `@ConditionalOnMissingBean` | You haven't defined that bean yourself |
| `@ConditionalOnProperty` | A property has a certain value |
| `@ConditionalOnBean` | Another bean exists |
> ⭐ The key insight: **auto-config backs off when you define your own bean** (`@ConditionalOnMissingBean`). So you get defaults for free, but **your explicit config always wins**. Auto-configs are registered in `META-INF/spring/...AutoConfiguration.imports` (formerly `spring.factories`).

### 2.2 Seeing/debugging auto-config
```
java -jar app.jar --debug    # prints the "Conditions Evaluation Report":
                             # which auto-configs were applied (positive matches) and skipped (negative)
```
Or the Actuator `/conditions` endpoint (§7). Useful when "why is this bean configured (or not)?"

---

## 3. Starters (Curated Dependencies)

A **starter** is a single dependency that pulls in a **curated, compatible set** of libraries for a feature — so you don't hunt for individual JARs and versions (recall transitive deps & BOM, Phase 3.1.2).
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>   <!-- pulls in Spring MVC, Tomcat, Jackson... -->
</dependency>
```
| Starter | Pulls in (for...) |
|---------|-------------------|
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, Jackson (REST APIs — Phase 5.3) |
| `spring-boot-starter-data-jpa` | Spring Data JPA, Hibernate, transactions (Phase 5.4) |
| `spring-boot-starter-security` | Spring Security (Phase 5.6) |
| `spring-boot-starter-test` | JUnit, Mockito, AssertJ, Spring Test (Phase 6) |
| `spring-boot-starter-actuator` | Production monitoring endpoints (§7) |
| `spring-boot-starter-cache` | Caching abstraction (Phase 5.7) |
| `spring-boot-starter-data-redis` | Redis integration (Phase 4.8) |
| `spring-boot-starter-amqp` | RabbitMQ (Phase 11.3) |
| `spring-boot-starter-validation` | Bean Validation (Phase 5.3) |
> **One starter = everything you need for a feature**, with versions managed by Spring Boot's **BOM** (recall Phase 3.1.2 — the parent POM imports `spring-boot-dependencies`, so you declare starters **without versions**). This eliminates version-conflict headaches.

### 3.1 The parent POM / BOM (recall Phase 3.1)
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>   <!-- manages versions via BOM -->
    <version>3.2.0</version>
</parent>
```
> The Spring Boot parent POM provides the BOM (pins compatible versions of hundreds of libraries — Phase 3.1.2), the `spring-boot-maven-plugin` (fat JAR — §8), and sensible plugin defaults. (Gradle uses the Spring Boot plugin + dependency-management plugin — Phase 3.2.)

---

## 4. @SpringBootApplication & the Main Class

A Spring Boot app starts from a single annotated class:
```java
@SpringBootApplication                       // the one annotation that bootstraps everything
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);   // starts the app + embedded server
    }
}
```
### 4.1 What @SpringBootApplication combines
It's a **meta-annotation** bundling three (recall stereotype annotations, Phase 5.1):
| Included annotation | Does |
|---------------------|------|
| **`@SpringBootConfiguration`** | Marks this as a config class (`@Configuration`) |
| **`@EnableAutoConfiguration`** | Turns on auto-configuration (§2) |
| **`@ComponentScan`** | Scans this package + sub-packages for beans (Phase 5.1) |
> ⚠️ Because `@ComponentScan` starts from the main class's package, **all your beans must live in (sub)packages of the main class** — otherwise they won't be found (recall Phase 5.1 component scanning). A common "bean not found" cause. `SpringApplication.run()` creates the `ApplicationContext` (Phase 5.1), runs auto-config, starts the embedded server, and the app is ready.

---

## 5. Embedded Servers

> Spring Boot **embeds the web server inside your application** — no separate Tomcat install or WAR deployment. The app *is* the server; you run `java -jar app.jar` and it listens on a port (8080 by default — recall Phase 0.2 ports).
| Server | Note |
|--------|------|
| **Tomcat** (default) | Bundled with `starter-web` |
| **Jetty** | Lightweight alternative |
| **Undertow** | High-performance alternative |
```xml
<!-- Swap Tomcat for Undertow (recall exclusions, Phase 3.1.2): -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions><exclusion>...spring-boot-starter-tomcat...</exclusion></exclusions>
</dependency>
<dependency>...spring-boot-starter-undertow...</dependency>
```
> ⭐ This is a major shift from old Java EE (deploy a WAR to an external app server). With Boot, the app is **self-contained** — perfect for **containers** (one JAR → one Docker image → one process, Phase 10) and microservices (each service runs independently — Phase 12). Configure the port via `server.port` (recall config, Phase 5.1).

---

## 6. Configuration Binding & DevTools

### 6.1 Configuration (recall Phase 5.1)
Spring Boot reads `application.yml`/`application.properties`, env vars, etc. (with the precedence from Phase 5.1):
```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb   # auto-config uses this
    username: postgres
  jpa:
    hibernate:
      ddl-auto: validate                          # don't auto-create schema in prod!
```
- **Relaxed binding** (Phase 5.1): `spring.datasource.url` ↔ `SPRING_DATASOURCE_URL` (env var — Phase 0.3) → set config per environment without code changes.

### 6.2 DevTools (faster development)
`spring-boot-devtools` speeds up the dev loop:
| Feature | Effect |
|---------|--------|
| **Automatic restart** | App restarts when classes change (a restart class loader — recall Phase 1.11) |
| **LiveReload** | Browser auto-refreshes |
| **Dev-friendly defaults** | Disables caching, etc. |
> DevTools is a **dev-only** dependency (excluded from the production JAR). It uses a separate class loader to restart quickly without a full JVM restart (recall class loaders, Phase 1.11).

---

## 7. Actuator (Production-Ready Monitoring)

**Spring Boot Actuator** exposes **operational endpoints** for monitoring and managing a running app — health, metrics, info, environment, etc. Add `spring-boot-starter-actuator`.
| Endpoint | Shows |
|----------|-------|
| **`/actuator/health`** | App health (UP/DOWN) — used by load balancers & K8s probes (Phase 0.2/10) |
| **`/actuator/metrics`** | Metrics (memory, CPU, requests) — feeds Prometheus (Phase 9) |
| **`/actuator/info`** | App info (version, build) |
| **`/actuator/env`** | Environment/config properties |
| **`/actuator/loggers`** | View/change log levels at runtime (Phase 9) |
| **`/actuator/prometheus`** | Metrics in Prometheus format (Phase 9.2) |
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus   # which endpoints to expose
```
### 7.1 Custom health indicators
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override public Health health() {
        return checkSomething() ? Health.up().build() : Health.down().build();
    }
}
```
> ⚠️ **Health endpoints power liveness/readiness probes** (recall Phase 0.2 — load balancers route only to healthy instances; K8s restarts unhealthy pods — Phase 10). **Secure Actuator endpoints** — `/env`, `/loggers`, `/heapdump` expose sensitive info; expose only `health` publicly and secure the rest (Phase 5.6/15). Actuator metrics feed **Prometheus/Grafana** (Phase 9.2).

---

## 8. Packaging: Fat JARs & Layered JARs

### 8.1 Fat (executable) JAR
The `spring-boot-maven-plugin` (recall Phase 3.1.3) repackages your app into a single **executable fat JAR** containing your code **and all dependencies** — runnable with `java -jar` (no external server):
```bash
mvn clean package          # produces target/myapp.jar (the fat JAR)
java -jar target/myapp.jar # runs the app + embedded server
```
> The fat JAR bundles everything → one artifact to deploy/containerize (recall Phase 3.1.3). This is what you copy into a Docker image (Phase 10).

### 8.2 Layered JARs (for Docker)
> A naive Docker image re-copies the whole fat JAR on every build → slow, large layers. **Layered JARs** split the JAR into layers by change frequency (dependencies rarely change; your code changes often) → Docker caches the stable layers and only rebuilds your code layer.
```
Layers (least -> most frequently changed):
  dependencies        (rarely change -> cached by Docker)
  spring-boot-loader
  snapshot-dependencies
  application          (your code -> changes often, the only layer rebuilt)
```
> ⭐ Layered JARs make **Docker builds much faster** by maximizing layer caching (recall Docker layers, Phase 10) — only your small `application` layer rebuilds when you change code. Spring Boot supports this out of the box (`extract` mode / Buildpacks).

### 8.3 Building images
- `spring-boot-maven-plugin`'s **buildpacks** (`mvn spring-boot:build-image`) build an optimized OCI image **without a Dockerfile**.
- Or use **Jib** (Google) — also Dockerfile-less (recall Phase 10).

---

## 9. GraalVM Native Image Support

> Spring Boot 3+ supports compiling to a **GraalVM native image** (recall AOT/GraalVM, Phase 1.11/16.5) — an ahead-of-time-compiled native executable.
| | JVM JAR | Native image |
|---|---------|--------------|
| Startup | Seconds (JIT warm-up — Phase 1.11) | **Milliseconds** |
| Memory | Higher | **Much lower** |
| Peak throughput | Higher (JIT optimizes) | Slightly lower (no runtime profiling) |
| Build time | Fast | Slow (AOT compilation) |
| Reflection | Full | **Limited** (needs config — Phase 1.14) |
> Native images give **near-instant startup + tiny memory** → ideal for **serverless/functions** and fast-scaling containers (Phase 10/16). The trade-off: slow builds, and reflection/dynamic features need explicit configuration (Spring's AOT processing generates much of this — recall reflection limits under native, Phase 1.14). For most long-running services, the regular JVM JAR is still fine.

---

## 10. The Spring Boot Workflow
```
1. Generate a project at start.spring.io (Spring Initializr) — pick starters, Maven/Gradle.
2. Add starters for the features you need (web, data-jpa, security...).
3. Configure via application.yml (DB URL, etc.) — auto-config does the rest.
4. Write beans (@Service/@RestController/etc. — Phase 5.1) under the main package.
5. Run: ./mvnw spring-boot:run  (or java -jar after packaging).
6. Monitor via Actuator; deploy as a fat/layered JAR in a container (Phase 10).
```

---

## 11. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Beans outside the main class's package | Place them under it (component scanning) |
| Fighting auto-config | Override by defining your own bean (`@ConditionalOnMissingBean` backs off) |
| `ddl-auto: create`/`update` in production | Use `validate` + migrations (Flyway, Phase 8) |
| Exposing all Actuator endpoints publicly | Expose only `health`; secure the rest |
| DevTools in production | It's dev-only (excluded from the JAR) |
| Naive Docker JAR copy (slow builds) | Use layered JARs/buildpacks |
| Specifying starter versions manually | The BOM manages them (omit versions) |
| Expecting native image to "just work" with reflection | Provide reflection config / use AOT |

---

## 12. Connection to Backend / Spring (Why This Matters Later)

- **Everything in Phase 5** runs on Spring Boot: web (5.3), data (5.4), security (5.6), caching (5.7), scheduling (5.8) — all enabled by starters + auto-config.
- **`@SpringBootApplication`** bootstraps the `ApplicationContext` (Phase 5.1).
- **Embedded server + fat JAR** → containerization (Phase 10) and microservices (Phase 12).
- **Actuator** → monitoring/observability (Prometheus/Grafana, health probes — Phase 9, 10).
- **`ddl-auto: validate` + Flyway** (Phase 8) for safe schema management (recall don't ALTER in prod, Phase 4.2).
- **Config + profiles** (Phase 5.1) for per-environment deployment (Phase 10).
- **Layered JARs/buildpacks** for efficient Docker images (Phase 10).
- **`spring.threads.virtual.enabled=true`** (recall Phase 1.10) for virtual-thread request handling.
- **GraalVM native** (Phase 16.5) for serverless/fast-startup.

---

## 13. Quick Self-Check Questions

1. What does Spring Boot add on top of Spring?
2. What is auto-configuration, and how does it use conditional beans? How does it "back off"?
3. What is a starter, and how does it relate to the BOM?
4. What three annotations does `@SpringBootApplication` combine? Why must beans be under the main package?
5. What is an embedded server, and why is it significant for containers?
6. What does Actuator provide, and why secure its endpoints?
7. What's a fat JAR vs a layered JAR, and why do layered JARs help Docker?
8. What are the trade-offs of a GraalVM native image?

---

## 14. Key Terms Glossary

- **Spring Boot:** opinionated, convention-over-configuration Spring framework.
- **Auto-configuration:** classpath-based automatic bean configuration.
- **`@Conditional...`:** annotations gating auto-config beans.
- **Starter:** a curated dependency bundle for a feature.
- **BOM / parent POM:** manages compatible dependency versions (Phase 3.1).
- **`@SpringBootApplication`:** `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`.
- **Embedded server:** Tomcat/Jetty/Undertow inside the app.
- **DevTools:** dev-time auto-restart/LiveReload.
- **Actuator:** production monitoring/management endpoints.
- **Health indicator:** custom health check.
- **Fat (executable) JAR:** self-contained runnable JAR.
- **Layered JAR:** JAR split into change-frequency layers (Docker caching).
- **Buildpacks / Jib:** Dockerfile-less image builders.
- **GraalVM native image:** AOT-compiled native executable (fast startup, low memory).

---

*This is the note for **Section 5.2 — Spring Boot**.*
*Next section in roadmap: **5.3 Spring Web MVC (REST API Development)**.*
