# Docker (Containers for Java Backends)

> **Phase 10 — Containerization & Deployment → 10.1 Docker**
> Goal: Master Docker — containers vs VMs, the architecture, the Dockerfile (all key instructions + `CMD` vs `ENTRYPOINT`), multi-stage builds, core commands, Docker Compose, and **Java-specific best practices** (slim/Alpine, non-root, `MaxRAMPercentage`, layered JARs, PID 1, scanning with Trivy, Jib).

---

## 0. The Big Picture

A **container** packages your app **plus everything it needs to run** (JRE, libraries, config) into one portable, isolated unit that runs identically on your laptop, CI, and production. It kills "works on my machine" (recall 8.1 — same idea for schema). **Docker** is the dominant tool to build, ship, and run containers.

```
   Your code + JRE + deps          →  Docker IMAGE (immutable template, layered)
                                       │  docker run
                                       ▼
                                   CONTAINER (a running instance, isolated process)
   Push to a REGISTRY (Docker Hub / ECR) → pull & run anywhere (CI, K8s — 10.2)
```

> Containers are the deployment unit for **Kubernetes** (10.2) and **CI/CD** (10.3). They build on OS process isolation (Phase 0.3 — namespaces/cgroups), networking (Phase 0.2), and the JVM (Phase 1.11). Your Spring Boot fat JAR (Phase 5.2/3) goes *inside* the image.

---

## 1. Containers vs Virtual Machines

```
   VIRTUAL MACHINES                      CONTAINERS
   ┌──────┬──────┬──────┐                ┌──────┬──────┬──────┐
   │ App  │ App  │ App  │                │ App  │ App  │ App  │
   │ Bins │ Bins │ Bins │                │ Bins │ Bins │ Bins │
   │GuestOS GuestOS GuestOS│             ├──────┴──────┴──────┤
   ├──────┴──────┴──────┤                │   Docker Engine    │
   │     Hypervisor     │                ├────────────────────┤
   ├────────────────────┤                │     Host OS Kernel │  ◄ SHARED kernel
   │     Host OS         │                ├────────────────────┤
   │     Hardware        │                │     Hardware       │
   └────────────────────┘                └────────────────────┘
   Each VM = full guest OS (heavy)        Containers share host kernel (light)
```

| | Virtual Machine | Container |
|--|-----------------|-----------|
| Isolation unit | Full guest OS + virtual hardware | Process(es) sharing host kernel |
| Size | GBs | MBs |
| Startup | Minutes | **Seconds / sub-second** |
| Overhead | High (hypervisor + OS per VM) | Low |
| Isolation strength | Stronger (hardware-level) | Weaker (kernel-shared) but sufficient |
| Built on | Hypervisor | Linux **namespaces** (isolation) + **cgroups** (resource limits) — Phase 0.3 |

> ⭐ Containers are light because they **share the host kernel** — no per-app OS. This is why you can pack many on one host and start them in seconds (great for scaling — Phase 13.3 — and K8s — 10.2). ⚠️ Weaker isolation than VMs means kernel security matters (Phase 15).

---

## 2. Docker Architecture

| Component | Role |
|-----------|------|
| **Docker daemon (`dockerd`)** | Background service that builds/runs/manages containers |
| **Docker client (`docker`)** | CLI that talks to the daemon |
| **Image** | Immutable, **layered** template (read-only) |
| **Container** | A running (writable) instance of an image |
| **Registry** | Stores/distributes images (Docker Hub, AWS ECR, GHCR) |
| **Dockerfile** | The recipe to build an image |
| **Layer** | A filesystem diff; layers are **cached** & shared across images |

> ⭐ **Image layering + caching** is central: each Dockerfile instruction creates a layer; unchanged layers are reused on rebuild (fast) and shared between images (disk savings). Ordering instructions to maximize cache hits is a key skill (§5/§7).

---

## 3. The Dockerfile — Key Instructions

```dockerfile
FROM eclipse-temurin:21-jre-jammy        # base image (JRE 21) — the starting layer
WORKDIR /app                             # set working dir (created if absent)
COPY target/app.jar app.jar             # copy build output into the image
EXPOSE 8080                              # document the port (informational)
ENV JAVA_OPTS=""                         # default env var
ENTRYPOINT ["java","-jar","app.jar"]    # the command that runs (exec form)
```

| Instruction | Purpose |
|-------------|---------|
| **`FROM`** | Base image to build on (every Dockerfile starts here) |
| **`WORKDIR`** | Set the working directory for following instructions |
| **`COPY`** | Copy files from build context into the image |
| **`ADD`** | Like COPY + auto-extract tar / fetch URLs (prefer `COPY` — explicit) |
| **`RUN`** | Execute a command at **build time** (creates a layer; e.g., install packages) |
| **`ENV`** | Set environment variables |
| **`ARG`** | Build-time variable (not present at runtime) |
| **`EXPOSE`** | Document which port the app listens on (doesn't publish it) |
| **`USER`** | Run as a specific (non-root) user (§7) |
| **`VOLUME`** | Declare a mount point for persistent/external data |
| **`HEALTHCHECK`** | Command Docker runs to test container health (maps to Actuator — 9.2) |
| **`CMD`** | Default command/args (overridable at `docker run`) |
| **`ENTRYPOINT`** | The executable that always runs |

### 3.1 `CMD` vs `ENTRYPOINT` ⭐ (classic confusion)
| | `ENTRYPOINT` | `CMD` |
|--|-------------|-------|
| Purpose | The fixed executable | Default args (or default command) |
| Overridable at `docker run`? | Only via `--entrypoint` | Yes, by appending args |
| Common pattern | `ENTRYPOINT ["java","-jar","app.jar"]` | `CMD ["--spring.profiles.active=prod"]` |
```dockerfile
ENTRYPOINT ["java","-jar","app.jar"]   # always runs java -jar app.jar
CMD ["--server.port=8080"]             # default arg, appended; `docker run img --server.port=9090` overrides
```
> ⭐ **Use exec form `["..."]`, not shell form** (`ENTRYPOINT java -jar app.jar`). Shell form wraps your process in `/bin/sh -c`, so the **JVM doesn't get PID 1** and won't receive `SIGTERM` on shutdown → no graceful shutdown (§7). Exec form makes the JVM PID 1.

---

## 4. Multi-Stage Builds ⭐ (essential for Java)

A naive Java image bundles the **entire JDK + Maven/Gradle + source** → huge & insecure. **Multi-stage builds** use one stage to *build* and a second, slim stage to *run* — copying only the artifact.

```dockerfile
# ---- Stage 1: BUILD (has JDK + Maven) ----
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B dependency:go-offline          # cache deps layer (changes rarely) — Phase 3
COPY src ./src
RUN mvn -B clean package -DskipTests       # produce target/app.jar

# ---- Stage 2: RUNTIME (only a JRE) ----
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar    # copy ONLY the jar from the build stage
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```
> ⭐ Result: the final image has **no JDK, no Maven, no source** — just a JRE + your jar → far **smaller** (hundreds of MB → ~200MB) and a **smaller attack surface** (Phase 15). Copying `pom.xml` and resolving deps *before* copying `src` maximizes layer caching (dep layer reused unless `pom.xml` changes).

---

## 5. Essential Commands

| Command | Does |
|---------|------|
| `docker build -t myapp:1.0 .` | Build an image from the Dockerfile in `.` |
| `docker run -p 8080:8080 myapp:1.0` | Run a container, publish host:container port |
| `docker run -d --name api myapp:1.0` | Detached (background), named |
| `docker ps` / `docker ps -a` | List running / all containers |
| `docker logs -f api` | Stream a container's stdout (logs → stdout, 9.1) |
| `docker exec -it api sh` | Shell into a running container |
| `docker stop api` / `docker rm api` | Stop / remove a container |
| `docker images` / `docker rmi` | List / remove images |
| `docker pull` / `docker push` | Get from / send to a registry |
| `docker tag myapp:1.0 reg/myapp:1.0` | Tag for a registry |
| `docker system prune` | Reclaim space (dangling images/containers) |

### 5.1 `.dockerignore`
Like `.gitignore` (Phase 0.4) — exclude `target/`, `.git`, `node_modules`, secrets from the build context → faster builds, smaller context, no leaked secrets (Phase 15).

---

## 6. Docker Compose (Multi-Container Local Dev)

**Compose** defines & runs **multiple containers** together (app + DB + Redis + Kafka) from one `docker-compose.yml` — perfect for local dev & integration tests (Testcontainers uses similar ideas — Phase 6.5).

```yaml
services:
  app:
    build: .                          # build from local Dockerfile
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/app   # "db" = service name (DNS — §8)
    depends_on:
      db: { condition: service_healthy }

  db:
    image: postgres:16                # (Phase 4.6)
    environment:
      POSTGRES_DB: app
      POSTGRES_PASSWORD: secret
    volumes: ["pgdata:/var/lib/postgresql/data"]   # named volume = persistent
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]

  redis:
    image: redis:7                    # (Phase 4.8)

volumes:
  pgdata:
```
```bash
docker compose up -d      # start the whole stack
docker compose down       # stop & remove (add -v to delete volumes)
```
> ⭐ Compose gives each service a **DNS name = its service name** (the app reaches Postgres at host `db`) on a shared network. Great for dev/test; for production orchestration you graduate to **Kubernetes** (10.2).

---

## 7. Java-Specific Best Practices ⭐

| Best practice | Why |
|---------------|-----|
| **Multi-stage build** (§4) | Small image, no JDK/source in runtime (security + size) |
| **Slim base** (`-jre`, `jammy`/`alpine`) | Smaller; ⚠️ **Alpine uses musl libc** — can cause subtle JVM issues; prefer `temurin:21-jre-jammy` or distroless unless you've tested Alpine |
| **Run as non-root** (`USER`) | Limit blast radius if compromised (Phase 15) |
| **`-XX:MaxRAMPercentage`** | ⭐ The JVM must respect the **container's** memory limit, not the host's |
| **Layered JARs** | Split Spring Boot jar into layers for better caching |
| **Make JVM PID 1 (exec form)** | Receives `SIGTERM` → graceful shutdown |
| **Log to stdout** (9.1) | Platform collects logs (12-factor) |
| **Pin image versions** (no `:latest`) | Reproducible builds |
| **Scan images** (Trivy — §7.3) | Catch CVEs (Phase 15.4) |

### 7.1 Container memory & the JVM ⭐ (a critical gotcha)
Modern JVMs are **container-aware** (read cgroup limits — Phase 0.3). But the **default heap is only ~25% of available RAM**. Tune it:
```dockerfile
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","-jar","app.jar"]
```
> ⚠️ ⭐ **`-XX:MaxRAMPercentage` (not `-Xmx`)** lets the heap scale with the container's memory limit (set in K8s — 10.2). Hardcoding `-Xmx` ignores the limit → either OOM-killed (heap too big) or wasted RAM (too small). Also set the K8s memory **limit** so the JVM sees the right ceiling. (Heap/GC details — Phase 1.11.)

### 7.2 Non-root + PID 1
```dockerfile
FROM eclipse-temurin:21-jre-jammy
RUN useradd -r -u 1001 appuser
USER 1001                                # non-root (Phase 15)
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]     # exec form → JVM is PID 1, gets SIGTERM (graceful shutdown)
```
> ⚠️ If your process isn't PID 1 (shell form), `SIGTERM` from `docker stop`/K8s isn't delivered to the JVM → no in-flight request draining (recall readiness/graceful shutdown — 9.2). Exec form fixes this; for complex cases use a tiny init like `tini`.

### 7.3 Layered JARs, Trivy, Jib
- **Spring Boot layered JAR:** split the fat jar into `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application` layers so the (rarely-changing) deps layer is **cached** while only the small app layer rebuilds:
```dockerfile
RUN java -Djarmode=layertools -jar app.jar extract
COPY --from=build dependencies/ ./
COPY --from=build application/ ./        # app layer last (changes most)
```
- **Trivy:** scans images for known **CVEs** in OS packages & Java deps → run in CI (Phase 10.3/15.4): `trivy image myapp:1.0`.
- **Jib (Google):** builds **optimized, layered, reproducible** images **without a Dockerfile and without a Docker daemon** — a Maven/Gradle plugin (Phase 3). Great for CI; handles layering & non-root automatically:
```bash
mvn compile jib:build -Dimage=registry/myapp:1.0
```
> ⭐ **Jib** is a popular, low-friction way to containerize Java apps in CI (no Dockerfile to maintain, daemonless, well-optimized layers). **Distroless** base images (no shell/package manager) shrink size & attack surface further.

---

## 8. Networking (briefly)

| Concept | Note |
|---------|------|
| **Bridge network** (default) | Containers on it can reach each other by **name** (Compose §6) |
| **Port publishing** `-p host:container` | Expose a container port to the host |
| **`EXPOSE`** | Documentation only (doesn't publish) |
| **Host / none networks** | Use host's network / no network |
| **DNS** | Docker provides service-name DNS within a user network |
> Full service-to-service networking & discovery scales up to **Kubernetes Services/DNS** (10.2) and the **API gateway** (Phase 12.4). Container networking builds on Phase 0.2.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Single-stage build (huge image w/ JDK+source) | Multi-stage build (§4) |
| `:latest` tags | Pin versions for reproducibility (§7) |
| Hardcoding `-Xmx`, ignoring container limit | `-XX:MaxRAMPercentage` + set K8s limit (§7.1) |
| Shell-form ENTRYPOINT → JVM not PID 1 | Exec form for graceful shutdown (§3.1/§7.2) |
| Running as root | `USER` non-root (§7.2) |
| Copying `src` before deps (cache busting) | Copy `pom.xml` + resolve deps first (§4) |
| Secrets baked into image/layers | Use env/secrets at runtime; `.dockerignore` (§5.1, Phase 12.5) |
| Alpine causing odd JVM/glibc issues | Prefer `jammy`/distroless unless tested (§7) |
| Not scanning images | Trivy in CI (§7.3, Phase 15.4) |
| Logging to files inside the container | Log to stdout (9.1) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- The **Spring Boot fat/layered JAR** (Phase 5.2/3) is what you containerize.
- **JVM container-awareness & heap** ties to memory/GC (Phase 1.11); `MaxRAMPercentage` + K8s limits (10.2).
- **Graceful shutdown** (PID 1/`SIGTERM`) pairs with readiness probes (9.2) and rolling deploys (10.2).
- **HEALTHCHECK** maps to Actuator health (9.2).
- **Kubernetes** (10.2) runs these images; **CI/CD** (10.3) builds, scans (Trivy), and pushes them.
- **Config/secrets** at runtime (Phase 12.5), not baked in.
- **Security** (Phase 15.4) — slim base, non-root, scanning.
- Foundation for **Project 6 (Deployment Pipeline)** & **Project 7 (Microservices)**.

---

## 11. Quick Self-Check Questions

1. How do containers differ from VMs, and why are they lighter (namespaces/cgroups)?
2. Describe Docker's architecture (daemon, client, image, container, registry).
3. What do `FROM`, `COPY`, `RUN`, `ENV`, `EXPOSE`, `USER`, `ENTRYPOINT` do?
4. `CMD` vs `ENTRYPOINT` — difference and a typical combined pattern? Why exec form?
5. What is a multi-stage build, and why is it essential for Java images?
6. Why order Dockerfile instructions to copy deps before source?
7. Why use `-XX:MaxRAMPercentage` instead of `-Xmx` in a container?
8. Why must the JVM be PID 1, and how does ENTRYPOINT form affect that?
9. What does Docker Compose do, and how do services find each other?
10. What are layered JARs, Trivy, and Jib — and why use each?

---

## 12. Key Terms Glossary

- **Container / image / layer:** running instance / immutable template / cached filesystem diff.
- **Docker daemon / client / registry:** engine / CLI / image store.
- **Namespaces / cgroups:** kernel isolation / resource limits (Phase 0.3).
- **Dockerfile:** image build recipe.
- **`CMD` vs `ENTRYPOINT`:** default args vs fixed executable.
- **Exec vs shell form:** `["..."]` (JVM PID 1) vs `/bin/sh -c` wrapper.
- **Multi-stage build:** build stage (JDK) + slim runtime stage (JRE).
- **`-XX:MaxRAMPercentage`:** heap as a % of the container memory limit.
- **Layered JAR:** Spring Boot jar split into cacheable layers.
- **PID 1 / SIGTERM:** init process / shutdown signal → graceful shutdown.
- **Docker Compose:** multi-container definition for local dev.
- **Trivy:** image vulnerability scanner (CVEs).
- **Jib:** daemonless, Dockerfile-less Java image builder (Maven/Gradle).
- **Distroless / Alpine:** minimal base images.
- **`.dockerignore`:** exclude files from the build context.

---

*This is the note for **Section 10.1 — Docker**.*
*Previous section in roadmap: **9.3 APM**.*
*Next section in roadmap: **10.2 Kubernetes**.*
