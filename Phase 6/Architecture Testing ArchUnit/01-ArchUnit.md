# Architecture Testing (ArchUnit)

> **Phase 6 — Testing → 6.7 Architecture Testing (ArchUnit)**
> Goal: Understand ArchUnit — automated tests that enforce your architecture: layered-architecture rules, package-dependency rules, naming conventions, and cycle detection.

---

## 0. The Big Picture

**ArchUnit** lets you write **tests that enforce architectural rules** — as code, run with your normal test suite (JUnit — Phase 6.2). Instead of hoping the team follows the architecture (layering, package boundaries, naming), you **assert** it automatically — and CI fails if someone violates it.

```
"Controllers must not access repositories directly"  -> written as a TEST -> enforced automatically
```

> Architecture erodes silently as teams grow ("just this once" shortcuts accumulate into a "big ball of mud"). ArchUnit makes architecture **executable and self-enforcing** — a guardrail that catches violations in CI (Phase 10.3) before they become entrenched. It connects to layered/clean architecture (Phase 14).

---

## 1. Why Test Architecture?

| Without architecture tests | With ArchUnit |
|----------------------------|---------------|
| Rules live in docs/wikis (ignored) | Rules are **executable** and enforced |
| Violations slip in unnoticed | CI **fails** on a violation |
| Architecture erodes over time | Architecture stays intact |
| Code review must catch everything | Automated checks free up reviewers |
> ⭐ The core value: **architecture as documentation is aspirational; architecture as tests is enforced.** Diagrams and conventions get violated; a failing build doesn't. ArchUnit turns "we agreed controllers shouldn't call repositories" into a rule that *can't* be broken silently.

---

## 2. How ArchUnit Works

ArchUnit **analyzes your compiled classes** (reflection/bytecode — recall Phase 1.11/1.14) and lets you assert rules about them, as JUnit tests:
```java
@Test
void controllersShouldNotAccessRepositories() {
    JavaClasses classes = new ClassFileImporter().importPackages("com.example");  // import classes

    ArchRule rule = noClasses()                          // assertion: NO classes...
        .that().resideInAPackage("..controller..")       // ...in the controller package
        .should().dependOnClassesThat()                  // ...should depend on...
        .resideInAPackage("..repository..");             // ...the repository package

    rule.check(classes);                                 // fails the test if violated
}
```
> ArchUnit reads your classes' structure (packages, dependencies, names, annotations) and checks **`ArchRule`s** against them. A violated rule **fails the test** — just like any assertion (Phase 6.1). The fluent DSL (`classes().that()...should()...`) reads like English.

---

## 3. Layered Architecture Enforcement

The most common use: enforce the **layered architecture** (Controller → Service → Repository — recall Phase 14) so dependencies only flow the right way:
```java
@Test
void layeredArchitectureIsRespected() {
    layeredArchitecture().consideringAllDependencies()
        .layer("Controller").definedBy("..controller..")
        .layer("Service").definedBy("..service..")
        .layer("Repository").definedBy("..repository..")

        // Define who may access each layer:
        .whereLayer("Controller").mayNotBeAccessedByAnyLayer()           // top layer
        .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")    // only controllers call services
        .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")    // only services call repos
        .check(importedClasses);
}
```
> ⭐ This enforces the **dependency rule** (Phase 14): controllers call services, services call repositories — **never the reverse**, and **never skipping layers** (a controller calling a repository directly). Without this, layers blur over time. ArchUnit makes the layering a hard, automated constraint.

---

## 4. Package Dependency Rules

Enforce which packages may depend on which — preventing forbidden dependencies (e.g., the domain shouldn't depend on the web layer — recall hexagonal/clean architecture, Phase 14):
```java
@Test
void domainShouldNotDependOnInfrastructure() {
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAPackage("..infrastructure..")
        .check(importedClasses);
}

@Test
void servicesShouldOnlyDependOnAllowedPackages() {
    classes()
        .that().resideInAPackage("..service..")
        .should().onlyDependOnClassesThat()
        .resideInAnyPackage("..service..", "..domain..", "..repository..", "java..", "org.springframework..")
        .check(importedClasses);
}
```
> Package rules keep **module boundaries** clean (essential for a modular monolith — Phase 14). E.g., the **domain layer stays pure** (no dependency on web/persistence frameworks) — the heart of hexagonal/clean architecture (Phase 14).

---

## 5. Naming Convention Enforcement

Enforce consistent naming (recall clean code, Phase 14):
```java
@Test
void servicesShouldBeNamedProperly() {
    classes()
        .that().resideInAPackage("..service..")
        .and().areAnnotatedWith(Service.class)       // (recall @Service, Phase 5.1)
        .should().haveSimpleNameEndingWith("Service")
        .check(importedClasses);
}

@Test
void controllersShouldBeAnnotated() {
    classes()
        .that().haveSimpleNameEndingWith("Controller")
        .should().beAnnotatedWith(RestController.class)   // (Phase 5.3)
        .check(importedClasses);
}

@Test
void repositoriesShouldBeInterfaces() {
    classes()
        .that().areAnnotatedWith(Repository.class)
        .should().beInterfaces()                          // Spring Data repos are interfaces (Phase 5.4)
        .check(importedClasses);
}
```
> Naming rules enforce conventions automatically (`*Service`, `*Controller`, `*Repository`) — keeping the codebase consistent and self-documenting (recall stereotype annotations, Phase 5.1; clean naming, Phase 14). New code that breaks the convention fails the build.

---

## 6. Cycle Detection

> ⚠️ **Circular dependencies between packages/modules are a design smell** (tight coupling, hard to understand/test — recall fragile-base-class/coupling discussions, Phase 1.3, and SCCs, Phase 2.2). ArchUnit can **detect cycles** automatically:
```java
@Test
void noCyclicDependenciesBetweenPackages() {
    slices().matching("com.example.(*)..")    // slice by sub-package
        .should().beFreeOfCycles()             // assert NO cycles between them
        .check(importedClasses);
}
```
> Package cycles (package A depends on B, B depends on A — recall cycle detection, Phase 2.2) make code impossible to modularize or reason about. This rule catches them early. (Spring also fails on cyclic *bean* dependencies — recall Phase 5.1.)

---

## 7. Common Rules Summary

ArchUnit ships with pre-built rules and a fluent API for custom ones:
| Rule kind | Example |
|-----------|---------|
| **Layered architecture** | Controller → Service → Repository only |
| **Package dependencies** | Domain must not depend on infrastructure |
| **Naming** | `*Service` classes in the service package |
| **Cycles** | No cyclic package dependencies |
| **Annotations** | All `@Controller`s end with `Controller` |
| **No forbidden APIs** | No use of `System.out.println` / `java.util.Date` / field injection |
```java
// E.g., forbid field injection (enforce constructor injection — Phase 5.1):
@Test
void noFieldInjection() {
    noFields().should().beAnnotatedWith(Autowired.class).check(importedClasses);
}
```
> ArchUnit can encode *any* structural rule your team agrees on (no `System.out`, no field injection, only `@Transactional` on services, etc.) — turning code-review nitpicks into automated checks.

---

## 8. Where ArchUnit Fits

| | Catches |
|---|---------|
| **Unit/integration tests** (6.1-6.6) | **Behavioral** correctness (does it work?) |
| **ArchUnit tests** | **Structural** correctness (is it built right?) |
> ArchUnit tests a **different dimension** than other tests: not "does the code do the right thing?" but "is the code *structured* the right way?" Both matter. ArchUnit is especially valuable on **large teams/codebases** where architecture drift is a real risk — and for **modular monoliths/microservices** (Phase 12/14) where boundaries are critical.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Architecture rules only in docs | Encode them as ArchUnit tests |
| No layering enforcement (drift) | Add `layeredArchitecture()` rules |
| Ignoring package cycles | Add `beFreeOfCycles()` |
| Over-strict rules blocking legitimate work | Keep rules pragmatic; allow exceptions where justified |
| Adding ArchUnit late (many violations) | Add early; fix incrementally |
| Treating it as a replacement for behavioral tests | It's complementary (structure, not behavior) |
| Rules not run in CI | Run them in the build (Phase 10.3) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Enforces layered architecture** (Controller → Service → Repository — Phase 14) — the structure of every Spring app (Phase 5).
- **Protects domain purity** for hexagonal/clean architecture (Phase 14).
- **Detects cyclic dependencies** (package cycles — Phase 2.2; bean cycles — Phase 5.1).
- **Enforces conventions** (naming, constructor injection — Phase 5.1; clean code — Phase 14).
- **Critical for modular monoliths & microservices** (Phase 12/14) — module boundaries stay intact.
- **Runs in CI** (Phase 10.3) — architecture violations fail the build, freeing code review.
- Complements behavioral tests (6.1–6.6) with **structural** tests.

---

## 11. Quick Self-Check Questions

1. What does ArchUnit test, and how does it differ from unit/integration tests?
2. Why test architecture instead of documenting it?
3. How does ArchUnit analyze your code, and how is a rule run?
4. How do you enforce a layered architecture (the dependency rule)?
5. How do you enforce package dependency rules (e.g., domain purity)?
6. How do you enforce naming conventions?
7. Why are package cycles bad, and how does ArchUnit catch them?
8. Where does ArchUnit provide the most value?

---

## 12. Key Terms Glossary

- **ArchUnit:** library for testing architectural rules as code.
- **`ArchRule`:** an architectural assertion (`classes().that()...should()...`).
- **Layered architecture rule:** enforces Controller→Service→Repository dependencies.
- **Package dependency rule:** restricts which packages may depend on which.
- **Naming rule:** enforces class/package naming conventions.
- **Cycle detection (`beFreeOfCycles`):** catches circular package dependencies.
- **`slices()`:** group classes (e.g., by sub-package) for cycle/dependency checks.
- **Structural vs behavioral testing:** how code is built vs what it does.
- **Architecture drift:** gradual erosion of architectural rules.

---

*This is the note for **Section 6.7 — Architecture Testing (ArchUnit)**.*
*This completes **Phase 6 — Testing**.*
*Next: **Phase 7 — API Design & Documentation**.*
