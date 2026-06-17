# Maven: Dependency Management

> **Phase 3 — Build Tools & Dependency Management → 3.1 Maven**
> Goal: Master Maven's dependency system — scopes, transitive dependencies & conflict resolution, exclusions, BOMs, and the dependency tree.

---

## 0. The Big Picture

Maven's killer feature is **automatic dependency management**: you declare the libraries you need, and Maven **downloads them *and* their dependencies (transitively)**, resolving version conflicts automatically. This eliminates "JAR hell" — but understanding *how* it resolves things is essential when conflicts arise.

```
You declare:  spring-boot-starter-web
Maven resolves: spring-web, spring-core, jackson, tomcat, logging, ... (dozens, transitively)
```

> Dependency management is where most real Maven problems happen (version conflicts, missing classes — recall `NoClassDefFoundError`, Phase 1.11). This note gives you the tools to understand and fix them.

---

## 1. Declaring Dependencies

A dependency is declared by its GAV (recall previous note) inside `<dependencies>`:
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.1</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```
Maven downloads each from a repository (Phase 3.1.1) and adds it to the **classpath** (recall Phase 1.1/0.1 — the classpath is where the JVM finds classes).

---

## 2. Dependency Scopes

A **scope** controls **when** a dependency is available (compile, test, runtime) and **whether** it's bundled into the final artifact. This keeps the classpath correct and the output lean.

| Scope | Available at... | Bundled in artifact? | Example |
|-------|-----------------|----------------------|---------|
| **`compile`** (default) | compile, test, runtime | ✅ Yes | Most libraries (Spring, etc.) |
| **`provided`** | compile, test (NOT runtime) | ❌ No | Servlet API (the container provides it) |
| **`runtime`** | test, runtime (NOT compile) | ✅ Yes | JDBC drivers (PostgreSQL) |
| **`test`** | test only | ❌ No | JUnit, Mockito (Phase 6) |
| **`system`** | like provided, but you supply the JAR path | ❌ No | (avoid — non-portable) |
| **`import`** | (special) for BOMs | n/a | Importing a BOM (§5) |

### 2.1 Understanding the key scopes
- **`compile`** (default): needed everywhere and shipped. Most dependencies.
- **`provided`**: needed to compile, but supplied by the runtime environment (e.g., the servlet API when deploying a WAR to Tomcat). **Not** bundled → avoids conflicts/bloat.
- **`runtime`**: not needed to compile (you code against an interface), only at runtime. Classic example: a **JDBC driver** (you code against `java.sql.*`, the driver is loaded at runtime — Phase 4.7).
- **`test`**: only for the test classpath (JUnit, Mockito, Testcontainers — Phase 6). Never shipped.
```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>     <!-- driver: not needed at compile, loaded at runtime -->
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>        <!-- testing: never shipped to production -->
</dependency>
```
> Choosing the right scope keeps your artifact **small** and your classpath **correct** — e.g., test libraries don't bloat (or leak into) production.

---

## 3. Transitive Dependencies

A **transitive dependency** is a dependency *of your dependency*. Maven pulls these in **automatically** — you don't list them.
```
You declare:  spring-boot-starter-web
  -> it depends on spring-web (transitive)
       -> which depends on spring-core (transitive)
            -> which depends on spring-jcl (transitive) ...
Maven downloads ALL of them automatically.
```
> This is Maven's biggest convenience — you declare a handful of dependencies and get the whole tree. But it's also where **conflicts** arise (two paths needing different versions of the same library).

---

## 4. Dependency Conflict Resolution

When two transitive paths require **different versions** of the same library, Maven must pick one. It uses **"nearest wins"** (dependency mediation):

### 4.1 The "nearest definition" rule
> Maven chooses the version that is **closest to your project** in the dependency tree (fewest hops). If two are at the same depth, the **first declared** wins.
```
Your project
├── A -> C 1.0          (C is 2 levels deep)
└── B -> D -> C 2.0     (C is 3 levels deep)
Maven picks C 1.0 (nearer to your project).
```

### 4.2 Forcing a version (dependency management — §5, or direct declaration)
To override Maven's choice, **declare the dependency directly** (depth 1 = nearest, always wins) or use `<dependencyManagement>`:
```xml
<!-- Declaring C directly forces YOUR version (it's now the nearest) -->
<dependency>
    <groupId>com.example</groupId>
    <artifactId>C</artifactId>
    <version>2.0</version>
</dependency>
```
> ⚠️ **Version conflicts** are a top cause of `NoSuchMethodError`/`NoClassDefFoundError` (Phase 1.11): code compiled against one version runs against another. Diagnose with the dependency tree (§7).

---

## 5. Dependency Management & BOM

### 5.1 `<dependencyManagement>` — centralize versions
`<dependencyManagement>` declares versions **without** adding the dependencies — children/usages then omit the version (it's inherited). This **centralizes version control** so all modules use consistent versions.
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.1</version>   <!-- version defined ONCE here -->
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <!-- no version! inherited from dependencyManagement -->
    </dependency>
</dependencies>
```
> Key distinction: `<dependencies>` = "I use this." `<dependencyManagement>` = "IF used, use this version." It declares versions/config without forcing the dependency itself.

### 5.2 BOM (Bill of Materials)
A **BOM** is a special POM (packaging `pom`) that defines a **consistent set of dependency versions** that are known to work together. You **import** it to inherit all those versions — no need to specify versions for any of them.
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>     <!-- import the BOM -->
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Now you can omit versions for everything in the Spring Boot BOM: -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- no version — managed by the BOM! -->
    </dependency>
</dependencies>
```
> **Spring Boot's BOM** (`spring-boot-dependencies`) is the canonical example: it pins versions of hundreds of libraries that are tested to work together — so you just declare *what* you need, and the BOM ensures compatible *versions*. (Spring Boot's parent POM imports this BOM — Phase 5.2.) This eliminates most version-conflict headaches.

---

## 6. Excluding Transitive Dependencies

Sometimes a dependency pulls in a transitive dependency you **don't want** (a conflict, a vulnerability, or a duplicate). **Exclude** it:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>  <!-- exclude Tomcat... -->
        </exclusion>
    </exclusions>
</dependency>
<!-- ...then add a different server, e.g., Jetty or Undertow -->
```
**Common exclusion reasons:**
| Reason | Example |
|--------|---------|
| Replace a default | Swap Tomcat → Jetty/Undertow (Phase 5.2) |
| Avoid a conflict | Two libraries pull different versions of a transitive dep |
| Remove a vulnerable transitive dep | Security (Phase 15.4) |
| Avoid logging conflicts | Exclude one SLF4J binding (Phase 9) |
> Exclusions are surgical — use them to prune unwanted transitive dependencies. (Spring Boot starters are designed for this kind of swapping.)

---

## 7. The Dependency Tree (Your Debugging Tool)

> **`mvn dependency:tree` is your #1 tool for diagnosing dependency problems.** It shows the full transitive tree, including which versions won and why.
```bash
mvn dependency:tree
# [INFO] com.example:my-app:jar:1.0.0
# [INFO] +- org.springframework.boot:spring-boot-starter-web:jar:3.2.0:compile
# [INFO] |  +- org.springframework.boot:spring-boot-starter:jar:3.2.0:compile
# [INFO] |  +- ...
# [INFO] |  \- org.springframework:spring-web:jar:6.1.1:compile

mvn dependency:tree -Dincludes=com.fasterxml.jackson.core   # filter to a specific dependency
mvn dependency:analyze     # find unused declared deps & used-but-undeclared deps
```
- Shows the **hierarchy** of all (transitive) dependencies and their resolved versions/scopes.
- Reveals **conflicts** (omitted versions are marked).
- Use it whenever you hit a `NoClassDefFoundError`/`NoSuchMethodError` or a mysterious version issue.
> Pair with IDE tooling (IntelliJ's Maven dependency diagram) for visualization.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Specifying versions everywhere | Use `dependencyManagement` / a BOM |
| Version conflict → `NoClassDefFoundError`/`NoSuchMethodError` | Run `dependency:tree`; force/exclude versions |
| Wrong scope (e.g., test lib at compile) | Use the correct scope (`test`, `runtime`, `provided`) |
| JDBC driver at `compile` scope | Use `runtime` scope |
| Manually adding transitive deps | Maven pulls them automatically |
| Not knowing why a version was chosen | "Nearest wins"; check `dependency:tree` |
| Keeping vulnerable transitive deps | Exclude them / update (Phase 15.4) |
| Declaring deps you don't use | `mvn dependency:analyze` to find them |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Spring Boot starters** (Phase 5.2) bundle curated transitive dependencies — one starter pulls in everything you need for a feature (web, data, security).
- **Spring Boot BOM / parent POM** (Phase 5.2) manages versions of all Spring + common libraries → you rarely specify versions.
- **Scopes** keep test libraries (JUnit/Mockito/Testcontainers — Phase 6) out of production artifacts.
- **Conflict resolution** is essential when integrating many libraries (DB, messaging, security) — `dependency:tree` is a daily debugging tool.
- **Exclusions** to swap embedded servers (Phase 5.2) or remove vulnerable deps (Phase 15.4, dependency scanning).
- **`runtime` scope** for JDBC drivers (Phase 4.7), logging bindings (Phase 9).

---

## 10. Quick Self-Check Questions

1. What is a transitive dependency, and who manages them?
2. Name the dependency scopes and what each controls. Which for a JDBC driver? For JUnit?
3. How does Maven resolve conflicting versions ("nearest wins")?
4. What's the difference between `<dependencies>` and `<dependencyManagement>`?
5. What is a BOM, and how does Spring Boot use one?
6. When and how do you exclude a transitive dependency?
7. What command shows the dependency tree, and when do you use it?
8. Why can a version conflict cause `NoClassDefFoundError` at runtime?

---

## 11. Key Terms Glossary

- **Dependency:** a library your project uses (by GAV).
- **Scope (compile/provided/runtime/test/system/import):** when/where a dependency applies.
- **Transitive dependency:** a dependency of a dependency (auto-resolved).
- **Dependency mediation ("nearest wins"):** how Maven resolves version conflicts.
- **`dependencyManagement`:** centralizes versions without adding deps.
- **BOM (Bill of Materials):** a POM defining a compatible set of versions (imported).
- **Exclusion:** removing an unwanted transitive dependency.
- **`mvn dependency:tree` / `:analyze`:** inspect / audit the dependency graph.
- **JAR hell:** the dependency-conflict chaos Maven prevents.

---

*Previous topic: **POM & Project Structure**.*
*Next topic: **Lifecycle, Plugins & Multi-Module Projects**.*
