# Testing Fundamentals

> **Phase 6 — Testing → 6.1 Testing Fundamentals**
> Goal: Understand *why* and *how* to test — the testing pyramid, TDD/BDD, the AAA pattern, test isolation/determinism, and code coverage vs mutation testing.

---

## 0. The Big Picture

**Testing** is writing code that verifies your code works correctly — automatically, repeatably, and before users find the bugs. It's a **non-negotiable** professional skill: untested code is legacy code the moment it's written.

```
Write code -> write tests that prove it works -> tests run on every change (CI) -> confidence to change/refactor
```

> Tests give you **confidence to change code without breaking things** (the foundation of refactoring and continuous delivery — Phase 10.3). This phase builds on everything: you'll test the Java (Phase 1), the services and APIs (Phase 5), and the database access (Phase 4).

---

## 1. Why Test? (The Payoff)

| Benefit | Why |
|---------|-----|
| **Catch bugs early** | Cheaper to fix in dev than in production (fail fast — Phase 1.5) |
| **Enable refactoring** | Tests catch regressions → change code fearlessly |
| **Living documentation** | Tests show how code is *meant* to be used |
| **Design feedback** | Hard-to-test code is usually badly designed (tight coupling) |
| **Confidence to deploy** | Green tests = a safety net for CI/CD (Phase 10.3) |
> ⭐ The deepest benefit: tests let you **change code safely**. Without them, every change risks breaking something invisible — and fear of change leads to rot. **Testable code is well-designed code** (DI/loose coupling — Phase 5.1 — makes testing easy; if testing is hard, your design is likely wrong).

---

## 2. The Testing Pyramid

The **testing pyramid** guides *how many* of each test type to write — many fast low-level tests, few slow high-level ones:
```
        /\        E2E / UI tests        (few — slow, brittle, expensive)
       /  \
      /----\      Integration tests     (some — test components together, e.g., with a DB)
     /      \
    /--------\    Unit tests            (MANY — fast, isolated, cheap)
```
| Layer | Tests | Speed | Quantity |
|-------|-------|-------|----------|
| **Unit** | One class/method in isolation (mocks for dependencies) | **Fast** (ms) | **Many** (most of your tests) |
| **Integration** | Multiple components together (real DB, Spring context) | Slower (seconds) | Some |
| **End-to-End (E2E)** | The whole system through the UI/API | **Slow** (minutes) | Few |
> ⚠️ **Write many fast unit tests, fewer integration tests, and very few E2E tests.** The **anti-pattern is the "ice cream cone"** (mostly slow E2E tests) — slow feedback, brittle, hard to debug. Fast unit tests give quick feedback on every change; integration/E2E catch the gaps where components meet.

---

## 3. Unit vs Integration Tests

| | **Unit test** | **Integration test** |
|---|---------------|----------------------|
| Tests | One class/method in isolation | Multiple components working together |
| Dependencies | **Mocked** (Phase 6.3) | **Real** (DB, Spring context — Phase 6.5/6.6) |
| Speed | Fast (no I/O) | Slower (I/O, startup) |
| Catches | Logic bugs | Wiring/integration bugs |
| Example | `UserService` with a mock repo | `UserController` → service → real DB |
> A **unit test** isolates the unit under test by **mocking its dependencies** (e.g., test `UserService` with a fake `UserRepository`). An **integration test** uses the real collaborators (e.g., a real database via Testcontainers — Phase 6.5/6.6). You need both: units verify logic fast; integration verifies the pieces fit.

---

## 4. TDD (Test-Driven Development)

**TDD** = write the **test first**, then the code to make it pass. The cycle is **Red → Green → Refactor**:
```
1. RED:      write a failing test for the next bit of behavior
2. GREEN:    write the MINIMUM code to make it pass
3. REFACTOR: clean up the code (tests still green) -> repeat
```
| Benefit of TDD | Why |
|----------------|-----|
| Forces clear requirements | You define expected behavior before coding |
| Guarantees test coverage | Every line exists to pass a test |
| Better design | Writing the test first surfaces awkward APIs |
| Confidence | You always have a passing test suite |
> TDD is a discipline, not a requirement — but the **Red-Green-Refactor** mindset (write a test, make it pass, improve) is invaluable even when applied loosely. It keeps code testable and focused. (You don't have to do strict TDD, but you should *always* test.)

---

## 5. BDD (Behavior-Driven Development)

**BDD** extends TDD by focusing on **behavior** described in business-readable terms — often the **Given-When-Then** structure:
```
GIVEN a user with an empty cart       (the initial context/state)
WHEN they add a product               (the action)
THEN the cart contains one item       (the expected outcome)
```
> BDD encourages tests that read like specifications of *behavior* (not implementation details). Tools like Cucumber use Given-When-Then literally; but even in plain JUnit, **structuring tests as Given-When-Then** (≈ the AAA pattern, §6) makes them clearer and behavior-focused. Mockito's BDD style (`given().willReturn()` — Phase 6.3) supports this.

---

## 6. The AAA Pattern (Arrange-Act-Assert)

Every good test has three clear sections — **Arrange, Act, Assert** (a.k.a. Given-When-Then):
```java
@Test
void shouldCalculateTotalWithTax() {
    // ARRANGE (Given): set up the test data & objects
    Order order = new Order(List.of(new Item("book", BigDecimal.valueOf(100))));
    TaxCalculator calc = new TaxCalculator(0.10);

    // ACT (When): perform the action under test (one thing!)
    BigDecimal total = calc.calculateTotal(order);

    // ASSERT (Then): verify the outcome
    assertThat(total).isEqualByComparingTo("110.00");
}
```
| Section | Purpose |
|---------|---------|
| **Arrange** (Given) | Set up inputs, mocks, and state |
| **Act** (When) | Invoke the method under test (usually one call) |
| **Assert** (Then) | Verify the result / behavior |
> ⭐ **Structure every test as AAA** — it makes tests readable and focused. **One logical assertion concept per test** (test one behavior); a clear test name (§7) describes *what* it verifies. Avoid mixing multiple acts/scenarios in one test.

---

## 7. Good Test Practices

| Practice | Why |
|----------|-----|
| **Descriptive names** | `shouldThrowWhenAgeIsNegative` — the name documents the behavior |
| **One behavior per test** | A failure points to exactly one cause |
| **AAA structure** | Readable, clear sections |
| **Test behavior, not implementation** | Tests survive refactoring; don't assert internal details |
| **Fast** | Unit tests in milliseconds (run them constantly) |
| **No logic in tests** | Tests should be simple/obvious (logic in tests = bugs in tests) |
> Test the **public behavior** (what the method *does*), not the **implementation** (how it does it). Tests coupled to internals break on every refactor — defeating their purpose.

---

## 8. Test Isolation & Determinism (Critical)

> ⚠️ **Tests must be isolated and deterministic** — each runs independently and produces the **same result every time**.
| Property | Meaning |
|----------|---------|
| **Isolated** | No test depends on another's state/order; each sets up its own data |
| **Deterministic** | Same input → same result, always (no randomness/time/network flakiness) |
| **Repeatable** | Passes locally, in CI, on any machine |
> ⚠️ **Flaky tests** (sometimes pass, sometimes fail) are worse than no tests — they erode trust and get ignored. Common flakiness causes: shared mutable state between tests, dependence on test execution order, `Thread.sleep` for timing, real time/date (`LocalDate.now()`), random data, real network calls, and unclean DB state. **Fix flakiness by isolating tests** (fresh state each time — `@BeforeEach`, transactional rollback — Phase 6.5), **mocking external dependencies** (Phase 6.3), and **injecting a `Clock`** instead of using `now()`.

---

## 9. Code Coverage & Mutation Testing

### 9.1 Code coverage (JaCoCo)
**Code coverage** measures the **percentage of code executed by tests** (lines, branches). **JaCoCo** is the standard Java tool (integrates with Maven/Gradle — Phase 3).
```
80% line coverage = your tests execute 80% of the code's lines.
```
> ⚠️ **Coverage is a useful metric but a dangerous target.** High coverage means code *ran* during tests — **not** that it was *verified*. You can have 100% coverage with **zero assertions** (worthless tests). Aim for meaningful tests with good coverage (~80% is a common target), but **don't game the number** — coverage measures *execution*, not *correctness*.

### 9.2 Mutation testing (PITest)
**Mutation testing** goes further: it **deliberately introduces bugs ("mutants")** into your code (e.g., flips `>` to `<`, removes a line) and checks whether your tests **catch them**. If a test still passes with a mutant, your tests are weak.
```
PITest mutates code -> runs your tests -> a "killed" mutant = a test caught it (good);
a "surviving" mutant = your tests missed a bug (gap!).
```
> Mutation testing measures **test *quality*** (do your assertions actually catch bugs?), not just coverage. It's the real test of whether your tests are meaningful. (Slower than coverage; run periodically, not on every build.)

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| No tests ("I'll add them later") | Test as you go; untested = legacy |
| Mostly E2E tests (ice cream cone) | Follow the pyramid (mostly unit) |
| Flaky tests | Isolate, mock externals, inject `Clock` |
| Tests depending on order/shared state | Make each test independent |
| Testing implementation, not behavior | Assert outcomes, not internals |
| Chasing 100% coverage with weak tests | Meaningful assertions; consider mutation testing |
| Tests with their own logic/loops | Keep tests simple and obvious |
| `Thread.sleep` for timing | Use awaitility / proper synchronization |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Unit tests** of services use **Mockito** (Phase 6.3) — DI (Phase 5.1) makes this easy.
- **Integration tests** use **Spring Boot Test + Testcontainers** (Phase 6.5/6.6) against real PostgreSQL/Redis (Phase 4).
- **JUnit 5** (Phase 6.2) + **AssertJ** (Phase 6.4) are the tools.
- **CI/CD** (Phase 10.3) runs the whole suite on every push — never skip tests in CI (Phase 3.1.3).
- **JaCoCo coverage + quality gates** (SonarQube — Phase 10.3) enforce standards.
- **TDD/AAA** keep code testable (well-designed, loosely coupled — Phase 5.1/14).
- **ArchUnit** (Phase 6.7) tests architecture itself.

---

## 12. Quick Self-Check Questions

1. Why is testing essential, and what's the deepest benefit?
2. What is the testing pyramid, and what's the "ice cream cone" anti-pattern?
3. What's the difference between unit and integration tests?
4. What is the TDD cycle (Red-Green-Refactor)?
5. What is BDD's Given-When-Then, and how does it relate to AAA?
6. What are the three sections of the AAA pattern?
7. Why must tests be isolated and deterministic? What causes flaky tests?
8. What does code coverage measure — and what does it NOT? How does mutation testing differ?

---

## 13. Key Terms Glossary

- **Unit / integration / E2E test:** isolated unit / components together / whole system.
- **Testing pyramid:** many unit, some integration, few E2E.
- **TDD (Red-Green-Refactor):** write a failing test, make it pass, refactor.
- **BDD / Given-When-Then:** behavior-focused testing structure.
- **AAA (Arrange-Act-Assert):** the structure of a test.
- **Test isolation / determinism:** independent, repeatable tests.
- **Flaky test:** intermittently failing (untrustworthy) test.
- **Code coverage / JaCoCo:** % of code executed by tests.
- **Mutation testing / PITest:** introduces bugs to measure test quality.

---

*This is the note for **Section 6.1 — Testing Fundamentals**.*
*Next section in roadmap: **6.2 JUnit 5 (Jupiter)**.*
