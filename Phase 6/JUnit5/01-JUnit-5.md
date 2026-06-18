# JUnit 5 (Jupiter)

> **Phase 6 — Testing → 6.2 JUnit 5 (Jupiter)**
> Goal: Master JUnit 5 — the test annotations & lifecycle, assertions, assumptions, parameterized & dynamic tests, conditional execution, and extensions.

---

## 0. The Big Picture

**JUnit 5** (a.k.a. **Jupiter**) is the standard testing framework for Java — it provides the structure to write, organize, and run tests. You annotate methods as tests, set up/tear down state, and assert expected outcomes (recall AAA, Phase 6.1).

```java
@Test
void shouldAddTwoNumbers() {
    assertEquals(5, calculator.add(2, 3));   // a JUnit test
}
```

> JUnit 5 is what `spring-boot-starter-test` brings in (Phase 5.2). It's modular: **Jupiter** (the new API), **Vintage** (run old JUnit 4 tests), and the **Platform** (the test engine). This note covers writing tests with Jupiter.

---

## 1. Basic Test Anatomy

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    @Test                                    // marks a test method
    void shouldAddTwoNumbers() {
        Calculator calc = new Calculator();  // Arrange
        int result = calc.add(2, 3);         // Act
        assertEquals(5, result);             // Assert (Phase 6.1 AAA)
    }
}
```
- Test classes/methods don't need to be `public` (unlike JUnit 4).
- Method names describe the behavior (Phase 6.1) — `@DisplayName` adds readable descriptions (§2).

---

## 2. Core Annotations

| Annotation | Purpose |
|------------|---------|
| **`@Test`** | Marks a test method |
| **`@DisplayName("...")`** | A human-readable test name (for reports) |
| **`@Disabled("reason")`** | Skip a test (with a reason) |
| **`@BeforeEach`** | Run **before each** test (set up fresh state — isolation, Phase 6.1) |
| **`@AfterEach`** | Run **after each** test (cleanup) |
| **`@BeforeAll`** | Run **once before all** tests (static; expensive shared setup) |
| **`@AfterAll`** | Run **once after all** tests (static; shared cleanup) |
| **`@Nested`** | Group related tests in an inner class |
| **`@Tag("...")`** | Categorize tests (e.g., "slow", "integration") for filtering |
| **`@Order` / `@TestMethodOrder`** | Control execution order (use sparingly!) |
| **`@RepeatedTest(n)`** | Run a test n times |
| **`@Timeout`** | Fail if the test exceeds a duration |

### 2.1 Lifecycle example
```java
class UserServiceTest {
    private UserService service;

    @BeforeAll                               // once, before everything (static)
    static void initAll() { /* expensive shared setup */ }

    @BeforeEach                              // before EACH test -> fresh, isolated state (Phase 6.1)
    void setUp() { service = new UserService(...); }

    @Test @DisplayName("returns the user when found")
    void findsUser() { ... }

    @AfterEach                               // after each test (cleanup)
    void tearDown() { ... }
}
```
> ⚠️ **`@BeforeEach` runs before *every* test → ensures isolation** (each test gets fresh state — Phase 6.1). `@BeforeAll`/`@AfterAll` run **once** and must be `static` (or use `@TestInstance(PER_CLASS)`) — use for *expensive* shared setup only (a shared resource), since shared mutable state risks coupling tests.

### 2.2 @Nested (grouping)
```java
class OrderServiceTest {
    @Nested
    @DisplayName("when the cart is empty")
    class EmptyCart {
        @Test void rejectsCheckout() { ... }
    }
    @Nested
    @DisplayName("when the cart has items")
    class WithItems {
        @Test void allowsCheckout() { ... }
    }
}
```
> `@Nested` organizes tests by scenario/context — improving readability (maps to BDD's "given a context" — Phase 6.1). Reports show a nested tree.

---

## 3. Assertions

**Assertions** verify outcomes — the "Assert" of AAA (Phase 6.1). JUnit's built-in assertions:
```java
assertEquals(expected, actual);              // equality
assertEquals(expected, actual, "message");   // with a failure message
assertNotEquals(a, b);
assertTrue(condition);  assertFalse(condition);
assertNull(obj);  assertNotNull(obj);
assertSame(a, b);  assertNotSame(a, b);      // reference identity (==, recall Phase 1.3)
assertArrayEquals(expected, actual);
```

### 3.1 assertThrows (testing exceptions)
```java
@Test
void shouldThrowWhenAgeNegative() {
    var ex = assertThrows(IllegalArgumentException.class,   // expected exception type
        () -> user.setAge(-1));                             // the code that should throw
    assertEquals("age must be >= 0", ex.getMessage());     // verify the message
}
```
> `assertThrows` verifies a method **throws** the expected exception (recall exceptions, Phase 1.5) — and returns it so you can assert on its message/cause. Essential for testing error paths.

### 3.2 assertAll (group assertions)
```java
@Test
void verifyUser() {
    assertAll("user",                        // all assertions run, even if some fail
        () -> assertEquals("Alice", user.getName()),
        () -> assertEquals(30, user.getAge()),
        () -> assertNotNull(user.getEmail())
    );
}
```
> `assertAll` runs **all** grouped assertions and reports *all* failures together (a normal failed assertion stops the test at that point). Useful for verifying multiple properties of one object.

### 3.3 assertTimeout
```java
assertTimeout(Duration.ofSeconds(1), () -> service.fastOperation());   // fail if it takes > 1s
```
> **Note:** JUnit's assertions are basic. **AssertJ (Phase 6.4)** provides far more readable, fluent assertions (`assertThat(...)`) — most teams prefer it. JUnit assertions are fine for simple cases.

---

## 4. Assumptions (Conditional Test Execution)

**Assumptions** **abort** a test (mark it skipped, not failed) if a condition isn't met — for tests that only make sense in certain environments:
```java
@Test
void onlyOnCI() {
    assumeTrue("CI".equals(System.getenv("ENV")));   // skip (not fail) if not on CI
    // ... test runs only when the assumption holds ...
}
assumeFalse(condition);
assumingThat(condition, () -> { /* run these assertions only if condition is true */ });
```
> ⚠️ **Assumption vs assertion:** a failed **assertion** = test **fails**; a failed **assumption** = test is **skipped** (aborted). Use assumptions for environment-dependent tests (e.g., "skip if not on Linux / not on CI / external service unavailable").

---

## 5. Parameterized Tests (Same Test, Many Inputs)

**Parameterized tests** run the **same test logic** with **different inputs** — avoiding copy-paste. Use `@ParameterizedTest` + a source:
```java
@ParameterizedTest
@ValueSource(ints = {2, 4, 6, 100})           // run once per value
void shouldBeEven(int number) {
    assertTrue(number % 2 == 0);
}

@ParameterizedTest
@CsvSource({                                   // multiple args per run (comma-separated)
    "2, 3, 5",
    "10, 20, 30",
    "-1, 1, 0"
})
void shouldAdd(int a, int b, int expected) {
    assertEquals(expected, calc.add(a, b));
}

@ParameterizedTest
@EnumSource(Status.class)                       // run once per enum constant (Phase 1.3)
void shouldHandleAllStatuses(Status status) { ... }

@ParameterizedTest
@MethodSource("provideTestCases")               // arguments from a method (complex objects)
void test(Order order, BigDecimal expected) { ... }
static Stream<Arguments> provideTestCases() {
    return Stream.of(Arguments.of(order1, total1), Arguments.of(order2, total2));  // (Phase 1.8 streams)
}
```
| Source | Provides |
|--------|----------|
| `@ValueSource` | A single array of values (ints, strings...) |
| `@CsvSource` | Multiple comma-separated args per run |
| `@CsvFileSource` | Args from a CSV file |
| `@EnumSource` | Each enum constant (Phase 1.3) |
| `@MethodSource` | Args from a method returning `Stream<Arguments>` |
| `@NullSource` / `@EmptySource` | null / empty inputs (edge cases — Phase 2.1) |
> ⭐ Parameterized tests are excellent for **edge cases and boundary testing** (recall Phase 2.1 — empty, null, negative, boundaries). One test method, many scenarios — far cleaner than duplicating tests. `@MethodSource` handles complex object arguments.

---

## 6. Dynamic Tests (@TestFactory)

**Dynamic tests** are generated **at runtime** (vs `@Test` which is fixed at compile time) — for cases where the test cases aren't known until runtime:
```java
@TestFactory
Stream<DynamicTest> dynamicTests() {
    return Stream.of("racecar", "level", "noon")
        .map(word -> DynamicTest.dynamicTest("palindrome: " + word,
            () -> assertTrue(isPalindrome(word))));   // a test generated per word
}
```
> Use dynamic tests when test cases are computed/loaded at runtime. Most of the time, **parameterized tests are simpler** — reach for `@TestFactory` only when you genuinely need runtime generation.

---

## 7. Conditional Execution

Run tests only in certain conditions (OS, Java version, environment):
```java
@Test @EnabledOnOs(OS.LINUX)                    // only on Linux (recall Phase 0.3)
void linuxOnly() { ... }

@Test @EnabledOnJre(JRE.JAVA_21)                // only on Java 21
void java21Only() { ... }

@Test @EnabledIfEnvironmentVariable(named = "ENV", matches = "CI")   // only on CI
void ciOnly() { ... }

@Test @EnabledIf("customCondition")             // custom method-based condition
void conditional() { ... }
```
> Similar to assumptions (§4), but **declarative** (annotations) and evaluated *before* the test runs. Useful for OS/JRE/environment-specific tests.

---

## 8. Extensions (@ExtendWith)

**Extensions** are JUnit 5's plug-in mechanism — they add cross-cutting behavior to tests (the replacement for JUnit 4 runners/rules). You activate them with `@ExtendWith`:
```java
@ExtendWith(MockitoExtension.class)             // integrate Mockito (Phase 6.3)
class UserServiceTest { ... }

@ExtendWith(SpringExtension.class)              // integrate Spring (Phase 6.5 — @SpringBootTest includes it)
class IntegrationTest { ... }
```
| Common extension | Adds |
|------------------|------|
| `MockitoExtension` | Mockito mock injection (`@Mock`/`@InjectMocks` — Phase 6.3) |
| `SpringExtension` | Spring TestContext (Phase 6.5) |
| Testcontainers | Container lifecycle (Phase 6.5/6.6) |
| Custom extensions | Your own setup/teardown logic via lifecycle callbacks |
> Extensions hook into the test lifecycle (before/after callbacks, parameter resolution). `@SpringBootTest` and `@WebMvcTest` (Phase 6.5) include `SpringExtension` automatically. You can write custom extensions for reusable test infrastructure.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Relying on test execution order | Tests must be independent (Phase 6.1) |
| Shared mutable state across tests | Reset in `@BeforeEach` |
| `@BeforeAll`/`@AfterAll` not static | Make them static (or `@TestInstance(PER_CLASS)`) |
| Duplicating tests for different inputs | Use parameterized tests |
| Confusing assumption (skip) with assertion (fail) | Assumption aborts; assertion fails |
| Only JUnit assertions for complex objects | Use AssertJ (Phase 6.4) |
| Testing exceptions with try/catch | Use `assertThrows` |
| `@Disabled` left forever | Remove or fix disabled tests |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **`spring-boot-starter-test`** (Phase 5.2) bundles JUnit 5, Mockito, AssertJ.
- **`@ExtendWith(MockitoExtension.class)`** for unit tests (Phase 6.3).
- **`@SpringBootTest`/`@WebMvcTest`/`@DataJpaTest`** build on JUnit 5 (Phase 6.5).
- **Parameterized tests** for validating many inputs (DTO validation — Phase 5.3; edge cases — Phase 2.1).
- **`@BeforeEach`** ensures test isolation (Phase 6.1) — critical with shared DB state (Phase 6.5/6.6).
- **`@Tag`** to separate fast unit tests from slow integration tests in CI (Phase 10.3).
- **AssertJ** (Phase 6.4) is the preferred assertion library on top of JUnit.

---

## 11. Quick Self-Check Questions

1. What does `@Test` do, and how do test classes differ from JUnit 4?
2. What's the difference between `@BeforeEach` and `@BeforeAll`?
3. How do you test that code throws an exception?
4. What does `assertAll` do differently from sequential assertions?
5. What's the difference between an assumption and an assertion?
6. What are parameterized tests, and name three argument sources?
7. When use dynamic tests vs parameterized tests?
8. What are extensions, and name two common ones?

---

## 12. Key Terms Glossary

- **JUnit 5 / Jupiter:** the standard Java testing framework / its API.
- **`@Test` / `@DisplayName` / `@Disabled`:** test method / readable name / skip.
- **`@BeforeEach`/`@AfterEach`/`@BeforeAll`/`@AfterAll`:** lifecycle hooks.
- **`@Nested`:** group related tests.
- **Assertion (`assertEquals`/`assertThrows`/`assertAll`):** verify outcomes.
- **Assumption (`assumeTrue`):** abort (skip) a test if a condition fails.
- **Parameterized test (`@ParameterizedTest` + `@ValueSource`/`@CsvSource`/`@MethodSource`):** same test, many inputs.
- **Dynamic test (`@TestFactory`):** runtime-generated tests.
- **Conditional execution (`@EnabledOnOs`, etc.):** environment-based gating.
- **Extension (`@ExtendWith`):** JUnit 5 plug-in mechanism.

---

*This is the note for **Section 6.2 — JUnit 5 (Jupiter)**.*
*Next section in roadmap: **6.3 Mockito**.*
