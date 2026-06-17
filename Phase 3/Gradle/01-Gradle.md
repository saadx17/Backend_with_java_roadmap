# Gradle

> **Phase 3 — Build Tools & Dependency Management → 3.2 Gradle (Know it)**
> Goal: Understand Gradle at a working level — Groovy vs Kotlin DSL, configurations (dependencies), tasks & plugins, the wrapper, and multi-project builds — and how it compares to Maven.

---

## 0. The Big Picture

**Gradle** is the other major JVM build tool (alongside Maven — section 3.1). It does the same core jobs — compile, test, package, manage dependencies — but uses a **programmatic build script** (Groovy or Kotlin) instead of Maven's declarative XML, and adds **incremental builds** and **build caching** for speed.

```
Maven:  declarative XML (pom.xml)        -> rigid, predictable
Gradle: code-based DSL (build.gradle)    -> flexible, faster (incremental + caching)
```

> The roadmap says **"know it"** — most Spring/enterprise backends use **Maven**, but **Android** and many modern/open-source projects use **Gradle**. You should be able to read a Gradle build, add dependencies, and run tasks. This note maps Gradle concepts onto what you already know from Maven (3.1).

---

## 1. Maven vs Gradle (the comparison)

| Aspect | **Maven** | **Gradle** |
|--------|-----------|------------|
| Build file | `pom.xml` (XML, declarative) | `build.gradle` (Groovy) / `build.gradle.kts` (Kotlin) |
| Style | Declarative | Programmatic (real code) |
| Flexibility | Lower (convention-bound) | Higher (scriptable) |
| Speed | Slower | **Faster** (incremental builds, build cache, daemon) |
| Learning curve | Easier (more rigid) | Steeper (more powerful) |
| Verbosity | More verbose (XML) | More concise |
| Lifecycle | Fixed phases | Configurable **task graph** (DAG) |
| Common in | Enterprise/Spring backends | Android, modern/OSS projects |
> Both resolve dependencies from the **same repositories** (Maven Central — recall 3.1) and produce the same artifacts (JAR/WAR). The difference is the build *model*: Maven's fixed lifecycle vs Gradle's flexible task graph.

---

## 2. Groovy DSL vs Kotlin DSL

Gradle build scripts come in two languages:
| | **Groovy DSL** | **Kotlin DSL** |
|---|----------------|----------------|
| File | `build.gradle` | `build.gradle.kts` |
| Language | Groovy (dynamic) | Kotlin (statically typed) |
| Pros | Concise, common in older projects | **Type-safe, IDE autocomplete, compile-time checks** |
| Cons | No type checking; weaker IDE help | Slightly more verbose |
> **Kotlin DSL (`.kts`) is increasingly preferred** for new projects — you get IDE autocompletion and compile-time error checking on the build script itself. Groovy DSL is still very common. Examples below show both where useful.

### 2.1 A basic build.gradle (Groovy)
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()                      // where to fetch dependencies (recall 3.1)
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'   // GAV (recall 3.1)
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### 2.2 The same in Kotlin DSL (build.gradle.kts)
```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.0"
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    runtimeOnly("org.postgresql:postgresql")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```
> Notice: dependencies are still declared by **GAV** (`group:artifact:version`), repositories are still **Maven Central** — the *concepts* from 3.1 carry over; only the syntax changes.

---

## 3. Configurations (Gradle's Version of Scopes)

In Gradle, dependencies are grouped into **configurations** — the equivalent of Maven's **scopes** (recall 3.1), but more granular:
| Gradle configuration | Maven scope equivalent | Meaning |
|----------------------|------------------------|---------|
| **`implementation`** | `compile` | Needed to compile & run; **not exposed to consumers** |
| **`api`** | `compile` (exported) | Like implementation, but **exposed transitively** to consumers (library authors) |
| **`compileOnly`** | `provided` | Compile only, not at runtime |
| **`runtimeOnly`** | `runtime` | Runtime only (e.g., JDBC driver) |
| **`testImplementation`** | `test` | Test compile & run (JUnit, etc. — Phase 6) |
| **`testRuntimeOnly`** | `test` (runtime) | Test runtime only |
| **`annotationProcessor`** | (annotation processing) | Compile-time processors (Lombok, MapStruct — Phase 1.13) |

### 3.1 implementation vs api (the key Gradle distinction)
> **`implementation`** hides the dependency from consumers of *your* library (it's an internal detail); **`api`** exposes it transitively (consumers also get it). Using `implementation` by default → **faster builds** (changes don't force recompiling everything downstream) and better encapsulation. This is a Gradle improvement over Maven's all-transitive `compile` scope.
```groovy
dependencies {
    api 'com.google.guava:guava:33.0.0'          // consumers of MY lib also see Guava
    implementation 'org.apache.commons:commons-lang3:3.14.0'  // internal only — hidden
}
```
> For **applications** (not libraries), use `implementation` for almost everything. `api` matters mainly when *publishing a library* others depend on.

---

## 4. Tasks & the Task Graph

Where Maven has fixed **lifecycle phases** (3.1), Gradle has **tasks** — units of work organized into a **directed acyclic graph (DAG)** of dependencies (recall DAGs/topological sort, Phase 2.2!). Gradle figures out which tasks to run and in what order.
```
build  depends on  -> assemble + check
assemble  -> jar
check  -> test
jar  -> compileJava + processResources
(Gradle topologically sorts the task graph and runs only what's needed)
```

### 4.1 Common tasks (map to Maven phases)
| Gradle task | Maven equivalent | Does |
|-------------|------------------|------|
| `gradle build` | `mvn package` (+verify) | Compile, test, assemble |
| `gradle compileJava` | `mvn compile` | Compile main code |
| `gradle test` | `mvn test` | Run unit tests |
| `gradle jar` / `bootJar` | `mvn package` | Build the (Spring Boot) JAR |
| `gradle clean` | `mvn clean` | Delete `build/` |
| `gradle bootRun` | `spring-boot:run` | Run the Spring Boot app |
| `gradle publish` | `mvn deploy` | Publish to a repository |
| `gradle dependencies` | `mvn dependency:tree` | Show the dependency tree |
| `gradle tasks` | — | List available tasks |
```bash
gradle build              # or ./gradlew build (use the wrapper — §6)
gradle clean test
gradle dependencies       # inspect the dependency graph (debugging — recall 3.1)
```
> ⚠️ Gradle's output goes to **`build/`** (not Maven's `target/`) — also gitignored (Phase 0.4).

### 4.2 Custom tasks (the programmatic power)
Because the build script is **code**, you can define custom tasks easily — a key Gradle advantage:
```groovy
task hello {
    doLast {
        println 'Hello from a custom Gradle task!'
    }
}
```
> This scriptability is why Gradle is more flexible than Maven — but also why builds can become complex/unpredictable if overused. Maven's rigidity is sometimes a feature.

### 4.3 Incremental builds & caching (Gradle's speed advantage)
- **Incremental builds:** Gradle tracks task inputs/outputs and **skips tasks whose inputs haven't changed** (marked `UP-TO-DATE`).
- **Build cache:** reuses outputs from previous builds (even across machines/CI).
- **Gradle daemon:** a background process that stays warm (avoids JVM startup — recall Phase 1.11 warm-up) → faster repeated builds.
> These make Gradle noticeably **faster** than Maven on large/repeated builds — a major reason big projects (and Android) choose it.

---

## 5. Plugins

Like Maven, Gradle uses **plugins** to add functionality (compilation, Spring Boot, etc.):
```groovy
plugins {
    id 'java'                                              // core Java compilation/testing
    id 'org.springframework.boot' version '3.2.0'         // Spring Boot (bootJar, bootRun)
    id 'io.spring.dependency-management' version '1.1.4'  // BOM-style version management
}
```
| Plugin | Role |
|--------|------|
| `java` | Java compilation, testing, JAR |
| `application` | Runnable app (`run` task) |
| `org.springframework.boot` | Spring Boot fat JAR + `bootRun` (Phase 5.2) |
| `io.spring.dependency-management` | Spring's BOM-style version management (recall BOM, 3.1) |
| `jacoco` | Code coverage (Phase 6) |
> The Spring Boot Gradle plugin + dependency-management plugin together provide the same **BOM-managed versions** you get from Maven's parent POM (3.1) — declare dependencies without versions.

---

## 6. The Gradle Wrapper (gradlew)

Just like the Maven Wrapper (recall 3.1), the **Gradle Wrapper** (`gradlew` / `gradlew.bat`) pins a specific Gradle version per project — committed to the repo so everyone and CI builds identically.
```bash
./gradlew build           # uses the project's pinned Gradle version (no install needed)
./gradlew --version
```
> **Always use `./gradlew`** (not a globally installed `gradle`) — it guarantees reproducible builds across machines and CI (Phase 0.4 reproducibility, Phase 10). Commit `gradlew`, `gradlew.bat`, and `gradle/wrapper/` to git.

---

## 7. Multi-Project Builds

Gradle's equivalent of Maven multi-module (recall 3.1). A **root project** with sub-**projects** sharing configuration via `settings.gradle`.
```
my-system/
├── settings.gradle          <- declares the sub-projects (like parent's <modules>)
├── build.gradle             <- root build (shared config)
├── common/
│   └── build.gradle
├── service-a/
│   └── build.gradle
└── service-b/
    └── build.gradle
```
```groovy
// settings.gradle — lists the projects (aggregation)
rootProject.name = 'my-system'
include 'common', 'service-a', 'service-b'
```
```groovy
// service-a/build.gradle — depend on a sibling project
dependencies {
    implementation project(':common')      // project dependency (like Maven sibling module)
}
```
- **`settings.gradle`** declares which projects exist (Gradle's aggregation mechanism).
- Sub-projects can **depend on each other** via `project(':name')`.
- Shared config can be applied to all subprojects from the root build script.
> Same idea as Maven multi-module (parent POM + modules) — split a system into related, separately-buildable projects with shared config and inter-project dependencies. Gradle builds them in dependency order (topological sort — Phase 2.2).

---

## 8. When to Use Gradle vs Maven

| Choose **Maven** when... | Choose **Gradle** when... |
|--------------------------|---------------------------|
| You want simplicity & convention | You need build flexibility/customization |
| Standard enterprise/Spring backend | Large project where build speed matters |
| Team prefers declarative XML | Android, or a modern/OSS project already on Gradle |
| Predictability over speed | Incremental builds & caching are valuable |
> **For most Spring backends, Maven is the common default** (and what this roadmap's projects use). But you'll encounter Gradle frequently — being able to read/run a Gradle build is the "know it" goal. The *concepts* (dependencies by GAV, repositories, plugins, wrapper, multi-project) are the same; only the syntax and execution model differ.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `api` for everything in an app | Use `implementation` (faster, encapsulated) |
| Installing Gradle globally | Use the wrapper (`./gradlew`) |
| Confusing configurations with Maven scopes | They map closely (`implementation`≈`compile`, etc.) |
| Over-scripting the build (complex Groovy) | Keep it simple; prefer convention |
| Forgetting `build/` is the output dir | Gitignore it (not `target/`) |
| Expecting Maven's fixed phases | Gradle uses a task DAG instead |
| Not knowing Groovy vs Kotlin DSL | `.gradle` = Groovy, `.gradle.kts` = Kotlin |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Spring Initializr** (start.spring.io) offers **both** Maven and Gradle — you'll see/choose both (Phase 5.2).
- **`bootJar`/`bootRun`** (Spring Boot Gradle plugin) build/run the app — the artifact you containerize (Phase 10).
- **Configurations** keep test/runtime deps separated (Phase 6) just like Maven scopes.
- **Wrapper + reproducible builds** matter in CI/CD and Docker (Phase 10).
- **Multi-project builds** for microservices monorepos / layered apps (Phase 12, 14).
- **Incremental builds & caching** speed up CI pipelines on large codebases (Phase 13).
- **Knowing both tools** makes you adaptable across teams/projects.

---

## 11. Quick Self-Check Questions

1. What are the main differences between Maven and Gradle?
2. What's the difference between the Groovy and Kotlin DSL, and which is increasingly preferred?
3. How do Gradle "configurations" map to Maven "scopes"?
4. What's the difference between `implementation` and `api`, and why does it matter?
5. How does Gradle's task model differ from Maven's lifecycle?
6. What makes Gradle faster (name three mechanisms)?
7. What does the Gradle Wrapper do, and why use it?
8. How do multi-project builds work (settings.gradle, project dependencies)?
9. When would you choose Gradle over Maven?

---

## 12. Key Terms Glossary

- **Gradle:** flexible, code-based JVM build tool.
- **Groovy DSL / Kotlin DSL:** `build.gradle` / `build.gradle.kts` script languages.
- **Configuration:** a dependency grouping (Gradle's "scope").
- **`implementation` / `api`:** internal vs transitively-exposed dependency.
- **`runtimeOnly` / `compileOnly` / `testImplementation`:** runtime/compile/test configs.
- **Task / task graph (DAG):** unit of work / dependency-ordered set of tasks.
- **Incremental build / build cache / daemon:** Gradle's speed mechanisms.
- **`bootJar` / `bootRun`:** Spring Boot Gradle tasks.
- **Gradle Wrapper (`gradlew`):** pins the Gradle version per project.
- **`settings.gradle` / multi-project build:** declares & aggregates sub-projects.
- **`build/`:** Gradle's output directory (gitignored).

---

*This is the note for **Section 3.2 — Gradle**.*
*This completes **Phase 3 — Build Tools & Dependency Management**.*
*Next: **Phase 4 — Databases**.*
