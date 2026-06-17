# Maven: Lifecycle, Plugins & Multi-Module Projects

> **Phase 3 — Build Tools & Dependency Management → 3.1 Maven**
> Goal: Master Maven's build lifecycle (phases & goals), the common plugins, and multi-module project structure (parent POM, inheritance).

---

## 0. The Big Picture

Maven's build is driven by a **standard lifecycle** of phases. Running a phase executes it **and all phases before it**. The actual work is done by **plugins** (their **goals** are bound to phases). For larger systems, **multi-module projects** split code into related sub-projects sharing a parent POM.

```
mvn package  ->  runs validate -> compile -> test -> package  (all preceding phases)
                 (each phase triggers plugin goals: compile, run tests, build the JAR)
```

> Understanding the lifecycle explains *what `mvn install` actually does* and *where* in the build your plugins run. Multi-module structure is how real backends (and microservices monorepos) organize code.

---

## 1. The Three Built-in Lifecycles

Maven has **three** independent lifecycles:
| Lifecycle | Purpose |
|-----------|---------|
| **`clean`** | Removes build output (`target/`) |
| **`default`** (build) | The main build: validate → compile → test → package → install → deploy |
| **`site`** | Generates project documentation/reports |
> The **`default`** lifecycle does the real building. `clean` and `site` are separate (you often run `mvn clean install` = clean lifecycle + default lifecycle).

---

## 2. The Default Lifecycle Phases (Most Important)

The `default` lifecycle has many phases; the **key ones** (in order):
```
validate   -> check the project is correct
compile    -> compile src/main/java -> target/classes
test       -> run unit tests (src/test/java)
package    -> bundle compiled code into a JAR/WAR (target/)
verify     -> run integration tests / checks
install    -> copy the artifact to the LOCAL repo (~/.m2)
deploy     -> upload the artifact to a REMOTE repo (Nexus/Artifactory)
```

### 2.1 The crucial rule: phases run cumulatively
> **Running a phase runs that phase AND every phase before it.** `mvn package` automatically runs `validate`, `compile`, and `test` first. You never need to chain them manually.
```
mvn compile  -> validate, compile
mvn test     -> validate, compile, test
mvn package  -> validate, compile, test, package
mvn install  -> validate, compile, test, package, verify, install
```

### 2.2 Common phase commands
| Command | Does |
|---------|------|
| `mvn validate` | Validate the project structure |
| `mvn compile` | Compile main code |
| `mvn test` | Compile + run unit tests |
| `mvn package` | Build the JAR/WAR (runs tests first) |
| `mvn verify` | Run integration tests / quality checks |
| `mvn install` | Build + install to `~/.m2` (so other local projects can use it) |
| `mvn deploy` | Build + publish to a remote repo (CI/CD — Phase 10.3) |
| `mvn clean install` | The everyday combo: wipe `target/`, then full build + install |

### 2.3 Skipping tests (use sparingly)
```bash
mvn package -DskipTests          # compile tests but DON'T run them
mvn package -Dmaven.test.skip=true   # don't even compile tests (faster, riskier)
```
> ⚠️ Skipping tests defeats their purpose. Use only for quick local iterations — **never** skip tests in CI (Phase 10.3).

---

## 3. Phases, Goals & Plugins

The lifecycle defines **phases**, but the *actual work* is done by **plugin goals** bound to those phases.
```
Phase (compile)  ->  bound to  ->  Plugin goal (maven-compiler-plugin:compile)
Phase (test)     ->  bound to  ->  maven-surefire-plugin:test
Phase (package)  ->  bound to  ->  maven-jar-plugin:jar  (or spring-boot:repackage)
```
- A **plugin** is a collection of **goals** (units of work).
- Goals are **bound to phases** (by default or by your config) → running the phase runs the goal.
- You can also run a goal **directly**: `mvn dependency:tree` (the `tree` goal of the `dependency` plugin — recall previous note).
```
mvn <plugin>:<goal>      # run a specific goal directly
mvn <phase>              # run a phase (and all preceding) -> triggers bound goals
```

---

## 4. Common Plugins

| Plugin | Goal / role | Bound to |
|--------|-------------|----------|
| **maven-compiler-plugin** | Compiles Java (set source/target version) | `compile` |
| **maven-surefire-plugin** | Runs **unit** tests | `test` |
| **maven-failsafe-plugin** | Runs **integration** tests | `verify` |
| **maven-jar-plugin** | Builds the JAR | `package` |
| **maven-shade-plugin** | Builds a **fat/uber JAR** (all deps bundled) | `package` |
| **spring-boot-maven-plugin** | Repackages into an executable Spring Boot fat JAR | `package` |
| **jacoco-maven-plugin** | Code coverage reports (Phase 6) | `test`/`verify` |
| **maven-surefire** vs **failsafe** | Unit vs integration test separation | — |

### 4.1 Configuring a plugin
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>21</source>
                <target>21</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```
> The **compiler plugin** sets which Java version to compile for (or use the `maven.compiler.source/target` properties — Phase 3.1.1). The **Spring Boot plugin** turns your JAR into an executable fat JAR (`java -jar app.jar`) — Phase 5.2.

### 4.2 Fat JAR plugins (shade vs spring-boot)
- **maven-shade-plugin:** merges all dependencies into one "uber JAR" → run anywhere with `java -jar`.
- **spring-boot-maven-plugin:** creates a Spring Boot **executable JAR** with a nested-JAR structure and a launcher (the standard for Spring Boot apps — Phase 5.2, 10).
> A "fat/uber JAR" bundles your code **and** all dependencies into one runnable file — essential for containerized deployment (Phase 10).

---

## 5. Multi-Module Projects

A **multi-module** project splits a large system into multiple related **modules** under one **parent POM** — each module is its own buildable unit, but they share configuration and are built together.

### 5.1 Structure
```
my-system/                  <- parent (aggregator)
├── pom.xml                 <- parent POM (packaging: pom)
├── common/                 <- shared utilities module
│   └── pom.xml
├── service-a/              <- module
│   └── pom.xml
└── service-b/              <- module (may depend on common)
    └── pom.xml
```

### 5.2 The parent (aggregator) POM
The parent has `packaging: pom` and lists its `<modules>`:
```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-system</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>             <!-- parent = pom packaging -->

    <modules>                              <!-- aggregation: build these together -->
        <module>common</module>
        <module>service-a</module>
        <module>service-b</module>
    </modules>

    <!-- Shared config inherited by all children: -->
    <properties>
        <java.version>21</java.version>
    </properties>
    <dependencyManagement>                  <!-- centralize versions (prev note) -->
        ...
    </dependencyManagement>
</project>
```

### 5.3 A child module POM
Each child declares its **parent** and inherits its config:
```xml
<project>
    <parent>                               <!-- inherit from the parent POM -->
        <groupId>com.example</groupId>
        <artifactId>my-system</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>service-a</artifactId>     <!-- inherits groupId & version -->

    <dependencies>
        <dependency>
            <groupId>com.example</groupId>  <!-- depend on a sibling module! -->
            <artifactId>common</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
```

### 5.4 Aggregation vs Inheritance (two distinct concepts)
| Concept | Mechanism | Purpose |
|---------|-----------|---------|
| **Aggregation** | Parent's `<modules>` list | Build all modules together with one command |
| **Inheritance** | Child's `<parent>` element | Children inherit config (versions, properties, plugins) |
> They're often combined (one parent both aggregates *and* is inherited from), but they're separate ideas. **Inheritance** shares config; **aggregation** builds together.

### 5.5 Benefits
- **Shared configuration** (versions, plugins, Java level) defined once in the parent.
- **Consistent dependency versions** across modules (via `dependencyManagement`/BOM — prev note).
- **Modular structure** — clear boundaries (recall modular monolith, Phase 14; this is the build-tool side).
- **Build all at once:** `mvn install` from the parent builds every module in dependency order (Maven topologically sorts them — recall topological sort, Phase 2.2!).
> Multi-module is common for **microservices monorepos**, layered apps (api/service/domain modules), and shared-library + app structures. Maven figures out the correct **build order** automatically (a topological sort of the module dependency graph).

---

## 6. Profiles (Environment-Specific Builds)

A **profile** lets you customize the build for different environments (dev/test/prod) — activating different dependencies, properties, or plugins.
```xml
<profiles>
    <profile>
        <id>prod</id>
        <properties>
            <db.url>jdbc:postgresql://prod-server/db</db.url>
        </properties>
    </profile>
    <profile>
        <id>dev</id>
        <activation><activeByDefault>true</activeByDefault></activation>
        <properties>
            <db.url>jdbc:postgresql://localhost/db</db.url>
        </properties>
    </profile>
</profiles>
```
```bash
mvn package -Pprod      # activate the 'prod' profile
```
> Profiles handle environment differences at **build** time. (Spring Boot also has its own runtime **profiles** — `application-{profile}.yml`, Phase 5.2 — which are usually preferred for app config; Maven profiles are for build-time differences.)

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Chaining phases manually (`mvn compile test package`) | One phase runs all preceding — just `mvn package` |
| Skipping tests in CI | Never (`-DskipTests` only for local quick iteration) |
| Confusing phase vs goal | Phase = lifecycle step; goal = plugin's unit of work |
| Forgetting `clean` (stale `target/`) | Use `mvn clean install` when in doubt |
| Wrong Java version compiled | Configure compiler plugin / `maven.compiler.*` |
| Confusing aggregation and inheritance | Aggregation builds together; inheritance shares config |
| Not building dependent modules first | Maven topo-sorts module order automatically |
| Maven profiles for app config | Prefer Spring profiles for runtime config (Phase 5.2) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **`spring-boot-maven-plugin`** repackages your app into an executable fat JAR (`java -jar`) — the artifact you containerize (Phase 5.2, 10).
- **Lifecycle in CI/CD (Phase 10.3):** pipelines run `mvn clean verify` / `mvn deploy` — building, testing, and publishing artifacts.
- **surefire/failsafe + jacoco** separate unit vs integration tests and measure coverage (Phase 6).
- **Multi-module** structures microservices monorepos and layered apps (Phase 12, 14).
- **Parent POM + BOM** (prev note) keep versions consistent across modules (Spring Boot's parent POM — Phase 5.2).
- **Profiles** handle build-time environment differences (though Spring profiles handle runtime config — Phase 5.2).

---

## 9. Quick Self-Check Questions

1. What are Maven's three lifecycles?
2. List the key phases of the default lifecycle in order.
3. What's the crucial rule about running a phase?
4. What's the relationship between phases, goals, and plugins?
5. Name four common plugins and their roles.
6. What does a fat/uber JAR contain, and which plugins build one?
7. What's the difference between aggregation and inheritance in multi-module projects?
8. How does Maven determine the build order of modules?
9. What are profiles for, and how do they differ from Spring profiles?

---

## 10. Key Terms Glossary

- **Lifecycle:** an ordered set of build phases (`clean`/`default`/`site`).
- **Phase:** a step in a lifecycle (compile, test, package, install, deploy).
- **Cumulative execution:** running a phase runs all preceding phases.
- **Goal:** a unit of work provided by a plugin.
- **Plugin:** a collection of goals (compiler, surefire, spring-boot, etc.).
- **Fat/uber JAR:** a JAR bundling all dependencies (shade / spring-boot plugin).
- **Multi-module project:** related modules under a parent POM.
- **Parent POM / aggregator:** shared-config / module-listing POM (`packaging: pom`).
- **Aggregation:** building modules together (`<modules>`).
- **Inheritance:** children inheriting parent config (`<parent>`).
- **Profile:** an environment-specific build configuration.

---

*Previous topic: **Dependency Management**.*
*This completes **Section 3.1 — Maven**.*
*Next section in roadmap: **3.2 Gradle**.*
