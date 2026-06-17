# Maven: POM & Project Structure

> **Phase 3 — Build Tools & Dependency Management → 3.1 Maven**
> Goal: Understand what Maven is and why it exists, the Project Object Model (`pom.xml`), GAV coordinates, the standard directory layout, and the repository system.

---

## 0. The Big Picture

**Maven** is a **build automation and dependency management** tool for Java. It compiles your code, runs tests, packages artifacts (JAR/WAR), and — crucially — **automatically downloads and manages your dependencies** (third-party libraries) and *their* dependencies.

```
You declare WHAT you need (dependencies, plugins) in pom.xml
Maven figures out HOW to build it (download deps, compile, test, package)
```

> Before Maven, you manually downloaded JARs, managed classpaths, and wrote build scripts. Maven introduced **"convention over configuration"** — a standard project layout and lifecycle so any Maven project builds the same way. It's the most common Java build tool (Gradle being the other — section 3.2).

---

## 1. Why Maven Exists (Problems It Solves)

| Problem (without a build tool) | Maven's solution |
|--------------------------------|------------------|
| Manually downloading JARs + their dependencies | **Automatic dependency resolution** (transitive — next note) |
| Inconsistent project structures | **Standard directory layout** (convention) |
| Custom, ad-hoc build scripts | **Standard build lifecycle** (next note) |
| "Works on my machine" classpath issues | **Reproducible builds** from `pom.xml` |
| Sharing libraries across teams | **Central repository** + private repos |

> Maven's philosophy: **convention over configuration**. Follow the standard layout and you write minimal config; the build "just works." Deviate and you must configure more.

---

## 2. The POM (Project Object Model)

The **`pom.xml`** at the project root is Maven's heart — an XML file declaring everything about your project: its identity, dependencies, build configuration, and plugins.

### 2.1 A minimal pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <!-- GAV coordinates: this project's unique identity -->
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <!-- Properties: reusable values -->
    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- Dependencies: libraries this project needs -->
    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- Build: plugins and build configuration -->
    <build>
        <plugins>
            <!-- e.g., maven-compiler-plugin, spring-boot-maven-plugin -->
        </plugins>
    </build>
</project>
```

### 2.2 The main POM sections
| Section | Purpose |
|---------|---------|
| **GAV** (`groupId`, `artifactId`, `version`) | The project's unique identity (§3) |
| `packaging` | Output type: `jar`, `war`, `pom` (§4) |
| `properties` | Reusable values (versions, encoding, Java version) |
| `dependencies` | Libraries the project needs (next note) |
| `dependencyManagement` | Centralized version control / BOM (next note) |
| `build` | Plugins and build settings (next note) |
| `profiles` | Environment-specific configurations (§6) |
| `modules` | Sub-modules in a multi-module project (next note) |
| `parent` | Inherit config from a parent POM (next note) |

---

## 3. GAV Coordinates (The Identity System)

Every Maven artifact (your project *and* every dependency) is uniquely identified by **GAV**: **G**roupId, **A**rtifactId, **V**ersion.
```
groupId    : com.example          (organization/namespace, reverse-DNS like packages)
artifactId : my-app               (the project/library name)
version    : 1.0.0                (the specific version)
```
```xml
<groupId>org.springframework.boot</groupId>     <!-- the org -->
<artifactId>spring-boot-starter-web</artifactId> <!-- the specific library -->
<version>3.2.0</version>                          <!-- the version -->
```
> GAV is how Maven **finds and uniquely identifies** artifacts in repositories. When you add a dependency, you specify its GAV; Maven downloads exactly that artifact.

### 3.1 Versioning & SNAPSHOT
| Version style | Meaning |
|---------------|---------|
| `1.0.0` | A fixed **release** (immutable — never changes once published) |
| `1.0.0-SNAPSHOT` | A **development** version (mutable — re-downloaded as it changes) |
- **Release versions** are immutable: once `1.0.0` is published, it never changes (reproducibility).
- **SNAPSHOT** versions are work-in-progress: Maven re-checks for updates. Used during active development before a release.
> Follow **Semantic Versioning** (`MAJOR.MINOR.PATCH` — recall Phase 0.4 tags): MAJOR = breaking changes, MINOR = new features, PATCH = bug fixes.

---

## 4. Packaging Types

`<packaging>` defines what Maven produces:
| Packaging | Produces | Use |
|-----------|----------|-----|
| **`jar`** (default) | A `.jar` file | Libraries, Spring Boot apps (fat JAR) |
| **`war`** | A `.war` file | Web apps deployed to an external servlet container |
| **`pom`** | No artifact (just the POM) | Parent POMs / aggregator (multi-module — next note) |
> Most backends today produce a **`jar`** (Spring Boot's executable fat JAR — Phase 5.2) rather than a `war`, since the app embeds its own server (Tomcat).

---

## 5. Standard Directory Layout (Convention)

Maven expects a **standard project structure** — follow it and Maven knows where everything is without configuration:
```
my-app/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/           <- application source code
│   │   │   └── com/example/...
│   │   └── resources/      <- config files (application.yml, etc.)
│   └── test/
│       ├── java/           <- test source code
│       │   └── com/example/...
│       └── resources/      <- test config/fixtures
└── target/                 <- BUILD OUTPUT (compiled classes, JARs) — gitignored!
```
| Directory | Contains |
|-----------|----------|
| `src/main/java` | Production Java code |
| `src/main/resources` | Config, properties, static files (bundled into the JAR) |
| `src/test/java` | Test code (JUnit, etc. — Phase 6) |
| `src/test/resources` | Test-only resources |
| `target/` | Compiled output & packaged artifacts (**never commit** — `.gitignore` it, Phase 0.4) |
> ⚠️ **`target/` is generated** — it's regenerated on every build, so it belongs in `.gitignore` (recall Phase 0.4 — build artifacts don't go in version control). `src/main/resources` files end up *inside* the JAR (on the classpath at runtime — recall Phase 1.1/0.1 classpath).

---

## 6. The Repository System

Maven downloads dependencies from **repositories** — and caches them locally. Three levels:
```
1. LOCAL repository  (~/.m2/repository)   -- your machine's cache
        ^ if not found, check...
2. REMOTE/central    (Maven Central)      -- the public default repo
        ^ or...
3. PRIVATE repos     (Nexus, Artifactory) -- company-internal repos
```

### 6.1 The resolution flow
```
Need a dependency:
  1. Look in the LOCAL repo (~/.m2/repository). Found? Use it.
  2. If not, download from a REMOTE repo (Maven Central by default).
  3. Cache it in the local repo for next time.
```
| Repository | Location | Role |
|------------|----------|------|
| **Local** | `~/.m2/repository` | Your machine's cache (avoids re-downloading) |
| **Central** | repo.maven.apache.org | The public default — most open-source libraries |
| **Remote/Private** | Nexus, Artifactory, GitHub Packages | Company-internal libraries, proxies, caches |

### 6.2 Local repository (`~/.m2`)
- The first time you use a dependency, Maven downloads it to `~/.m2/repository` and reuses it forever (until a new version is needed).
- This is why the **first build is slow** (downloading) and subsequent builds are fast (cached).
- `~/.m2/settings.xml` configures Maven globally (repository URLs, credentials, proxies).

### 6.3 Private repositories (Nexus / Artifactory)
Companies run **private repository managers** (Nexus, Artifactory) to:
- Host **internal** libraries (not public).
- **Proxy/cache** Maven Central (faster, resilient, controllable).
- Enforce **security/license policies** on dependencies.
> (Recall Phase 0.4 — artifact repositories like Nexus/Artifactory store build outputs; CI/CD publishes to them — Phase 10.3.)

---

## 7. Maven Wrapper (mvnw)

The **Maven Wrapper** (`mvnw` / `mvnw.cmd`) is a script committed to your repo that **downloads and uses a specific Maven version** automatically — so everyone (and CI) builds with the *exact same* Maven version, no manual install needed.
```bash
./mvnw clean install      # uses the project's pinned Maven version
./mvnw --version          # shows the wrapped version
```
> **Always use the wrapper** (`./mvnw`) in projects that have it — it guarantees build reproducibility across machines and CI (recall Phase 0.4 reproducibility). Generated by `mvn wrapper:wrapper`; commit `mvnw`, `mvnw.cmd`, and `.mvn/` to git.

---

## 8. Running Maven (basics — full lifecycle in the next note)
```bash
mvn clean              # delete target/
mvn compile            # compile src/main/java
mvn test               # run tests
mvn package            # build the JAR/WAR
mvn install            # install to the local repo (~/.m2)
mvn clean install      # the common combo: clean, then build everything
```

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Committing `target/` to git | Add it to `.gitignore` (Phase 0.4) |
| Not following the standard layout | Use `src/main/java`, `src/test/java`, etc. |
| Hardcoding versions everywhere | Use `properties` / `dependencyManagement` (next note) |
| Using SNAPSHOT versions in releases | Use fixed release versions for reproducibility |
| Installing Maven manually for a team | Use the **Maven Wrapper** (`./mvnw`) |
| Confusing groupId/artifactId | groupId = org/namespace; artifactId = project name |
| Polluting the global Maven (not wrapper) | Pin the version with the wrapper |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Every Spring Boot project** is a Maven (or Gradle) project — `spring-boot-starter-*` dependencies, the `spring-boot-maven-plugin` (Phase 5.2).
- **`pom.xml`** is where you add every library you'll use across the roadmap (Spring, JPA, security, testing).
- **Repository system** matters for CI/CD (caching deps, publishing artifacts to Nexus — Phase 10.3).
- **Maven Wrapper** ensures reproducible builds in Docker/CI (Phase 10).
- **`src/main/resources`** holds `application.yml`, Flyway migrations, templates (Phase 5, 8).
- **Packaging (`jar`)** produces the artifact you'll containerize (Phase 10).

---

## 11. Quick Self-Check Questions

1. What two main things does Maven do, and what's "convention over configuration"?
2. What is the POM, and what are its main sections?
3. What is GAV, and why does Maven need it?
4. What's the difference between a release version and a SNAPSHOT?
5. What do `jar`, `war`, and `pom` packaging produce?
6. Describe the standard Maven directory layout. Why is `target/` gitignored?
7. Explain the three-level repository system and the resolution flow.
8. What is the Maven Wrapper, and why use it?

---

## 12. Key Terms Glossary

- **Maven:** build automation + dependency management tool for Java.
- **Convention over configuration:** standard layout/lifecycle → minimal config.
- **POM (`pom.xml`):** the project's build/dependency descriptor.
- **GAV:** groupId, artifactId, version — an artifact's unique identity.
- **SNAPSHOT:** a mutable development version.
- **Semantic Versioning:** MAJOR.MINOR.PATCH.
- **Packaging (jar/war/pom):** the build output type.
- **Standard directory layout:** `src/main/java`, `src/test/java`, `target/`, etc.
- **Local / central / private repository:** cache / public default / company repos.
- **`~/.m2`:** the local repository + `settings.xml`.
- **Nexus / Artifactory:** private repository managers.
- **Maven Wrapper (`mvnw`):** pins the Maven version for reproducible builds.

---

*This is the first note of **Section 3.1 — Maven**.*
*Next topic: **Dependency Management (scopes, transitive deps, exclusions, BOM)**.*
