# AssertJ (Fluent Assertions)

> **Phase 6 — Testing → 6.4 AssertJ (Fluent Assertions)**
> Goal: Master AssertJ — the fluent `assertThat()` API, assertions for strings/numbers/collections/exceptions, recursive comparison, soft assertions, and custom assertions.

---

## 0. The Big Picture

**AssertJ** provides **fluent, chainable assertions** that read like natural language and give far better failure messages than JUnit's built-in assertions (Phase 6.2). It's the **preferred assertion library** for Java tests.

```java
// JUnit:    assertEquals("Alice", user.getName());
// AssertJ:  assertThat(user.getName()).isEqualTo("Alice");   // fluent, readable, chainable
```

> AssertJ is the "Assert" in AAA (Phase 6.1) done well. It's bundled in `spring-boot-starter-test` (Phase 5.2) and pairs with JUnit 5 (Phase 6.2) and Mockito (Phase 6.3). The whole API starts with `assertThat(...)`.

---

## 1. Why AssertJ Over JUnit Assertions?

| | JUnit `assertEquals` | AssertJ `assertThat` |
|---|---------------------|----------------------|
| Readability | `assertEquals(expected, actual)` (which is which?) | `assertThat(actual).isEqualTo(expected)` (natural order) |
| Chaining | No | **Yes** (`.isNotNull().hasSize(3)`) |
| Failure messages | Basic | **Rich** (shows exactly what differed) |
| Discoverability | Limited | IDE autocomplete shows all options after `.` |
| Collections/objects | Awkward | **Powerful** built-in support |
> ⭐ AssertJ's killer features: **fluent chaining** (multiple checks in one statement), **IDE autocomplete** (type `assertThat(x).` and see every assertion), and **detailed failure messages** (it tells you precisely what was wrong). Once you use it, JUnit assertions feel primitive.

---

## 2. Basic Assertions

```java
import static org.assertj.core.api.Assertions.*;   // the single import for everything

assertThat(actual).isEqualTo(expected);
assertThat(value).isNotNull();
assertThat(value).isNull();
assertThat(condition).isTrue();
assertThat(condition).isFalse();
assertThat(a).isNotEqualTo(b);
assertThat(obj).isInstanceOf(User.class);
assertThat(obj).isSameAs(other);            // reference identity (==, recall Phase 1.3)

// Chaining multiple assertions on one value:
assertThat(user)
    .isNotNull()
    .extracting(User::getName)              // extract a property, then assert on it
    .isEqualTo("Alice");
```

---

## 3. String Assertions

```java
assertThat("Hello World")
    .startsWith("Hello")
    .endsWith("World")
    .contains("lo Wo")
    .hasSize(11)
    .isEqualToIgnoringCase("hello world")
    .matches("[A-Za-z ]+");                  // regex (Phase 1.4)

assertThat(text).isBlank();                  // empty or whitespace (Phase 1.4)
assertThat(text).isNotEmpty();
```

---

## 4. Number Assertions

```java
assertThat(42)
    .isGreaterThan(40)
    .isLessThanOrEqualTo(50)
    .isBetween(40, 50)
    .isPositive();

assertThat(3.14).isCloseTo(3.0, within(0.2));   // floating-point tolerance (recall Phase 1.2!)

// For BigDecimal (money — Phase 4.1): compare by VALUE, not equals (scale matters):
assertThat(total).isEqualByComparingTo("110.00");   // 110.00 == 110.0 by value
```
> ⚠️ For **floating-point** comparisons, use `isCloseTo(..., within(...))` (recall Phase 1.2 — floats are imprecise; `0.1+0.2 ≠ 0.3`). For **`BigDecimal`** (money — Phase 4.1), use **`isEqualByComparingTo`** (compares value, ignoring scale) — `isEqualTo` would fail on `110.00` vs `110.0`.

---

## 5. Collection Assertions (Very Powerful)

AssertJ's collection assertions are a major reason to use it:
```java
List<String> names = List.of("Alice", "Bob", "Carol");

assertThat(names)
    .hasSize(3)
    .contains("Alice", "Bob")                // contains these (in any order)
    .containsExactly("Alice", "Bob", "Carol")// exactly these, in THIS order
    .containsExactlyInAnyOrder("Carol", "Alice", "Bob")  // these, any order
    .doesNotContain("Dave")
    .allMatch(name -> name.length() > 2)     // every element (Phase 1.8 predicate)
    .anyMatch(name -> name.startsWith("A"))
    .isSorted();

// Extract a property from each element, then assert on the collection of properties:
assertThat(users)
    .extracting(User::getName)               // -> a list of names
    .containsExactly("Alice", "Bob");

// Extract multiple properties as tuples:
assertThat(users)
    .extracting(User::getName, User::getAge)
    .contains(tuple("Alice", 30), tuple("Bob", 25));

// Filter then assert:
assertThat(users)
    .filteredOn(u -> u.getAge() > 25)
    .hasSize(1);

// Maps:
assertThat(map).containsEntry("key", "value").containsKeys("a", "b").hasSize(2);
```
| Assertion | Checks |
|-----------|--------|
| `contains` / `containsExactly` / `containsExactlyInAnyOrder` | membership / order matters / order doesn't |
| `extracting(fn)` | map elements to a property, then assert |
| `filteredOn(predicate)` | filter, then assert (Phase 1.8) |
| `allMatch` / `anyMatch` / `noneMatch` | predicate over elements |
> ⭐ **`extracting` + `containsExactly`** is the idiomatic way to assert on collections of objects — assert the *relevant fields* without comparing whole objects. `tuple(...)` extracts multiple fields per element.

---

## 6. Exception Assertions

AssertJ's exception testing is cleaner than JUnit's `assertThrows` (Phase 6.2):
```java
assertThatThrownBy(() -> userService.register(invalid))
    .isInstanceOf(ValidationException.class)        // the exception type
    .hasMessage("Email is required")                 // exact message
    .hasMessageContaining("Email");                  // partial message

// Type-specific (more readable):
assertThatExceptionOfType(IllegalArgumentException.class)
    .isThrownBy(() -> user.setAge(-1))
    .withMessage("age must be >= 0");

// Assert NO exception is thrown:
assertThatNoException().isThrownBy(() -> service.safeOperation());

// Assert on the cause (chained exceptions — recall Phase 1.5):
assertThatThrownBy(() -> service.call())
    .hasCauseInstanceOf(SQLException.class);
```
> AssertJ exception assertions read fluently and let you chain message/cause checks (recall exception chaining, Phase 1.5). `assertThatThrownBy` is the common form; you can also assert the cause and root cause.

---

## 7. Recursive Comparison (Comparing Whole Objects)

Comparing two objects field-by-field without relying on `equals()` (useful for entities/DTOs that don't override `equals`, or to ignore certain fields):
```java
assertThat(actualUser)
    .usingRecursiveComparison()                  // compare ALL fields recursively
    .isEqualTo(expectedUser);

// Ignore fields that differ (e.g., generated id, timestamps):
assertThat(actualUser)
    .usingRecursiveComparison()
    .ignoringFields("id", "createdAt")
    .isEqualTo(expectedUser);
```
> ⭐ **`usingRecursiveComparison`** compares objects **field-by-field** (no need for `equals`/`hashCode`) — and lets you **ignore** generated fields (id, timestamps). Extremely useful for asserting that a returned DTO/entity matches an expected one without writing field-by-field assertions or relying on `equals` (recall the JPA equals/hashCode pitfalls, Phase 1.6/5.4).

---

## 8. Soft Assertions (Collect All Failures)

Normally a failed assertion **stops** the test (you don't see later failures). **Soft assertions** collect **all** failures and report them together (like JUnit's `assertAll`, Phase 6.2):
```java
SoftAssertions softly = new SoftAssertions();
softly.assertThat(user.getName()).isEqualTo("Alice");
softly.assertThat(user.getAge()).isEqualTo(30);
softly.assertThat(user.getEmail()).isEqualTo("alice@x.com");
softly.assertAll();                              // reports ALL failures at once

// Or the auto-closing form:
assertSoftly(softly -> {
    softly.assertThat(user.getName()).isEqualTo("Alice");
    softly.assertThat(user.getAge()).isEqualTo(30);
});
```
> Soft assertions are useful when verifying **multiple independent properties** — you see *all* the mismatches in one run instead of fixing them one at a time. (For a single behavior, prefer one focused assertion — Phase 6.1.)

---

## 9. Custom Assertions

For domain types you assert on repeatedly, write a **custom assertion** class for readable, reusable checks:
```java
public class OrderAssert extends AbstractAssert<OrderAssert, Order> {
    public OrderAssert(Order actual) { super(actual, OrderAssert.class); }
    public static OrderAssert assertThat(Order actual) { return new OrderAssert(actual); }

    public OrderAssert isPaid() {
        isNotNull();
        if (actual.getStatus() != Status.PAID)
            failWithMessage("Expected order to be PAID but was %s", actual.getStatus());
        return this;   // return this -> chainable
    }
}
// Usage:
OrderAssert.assertThat(order).isPaid().hasTotal("100.00");   // domain-specific, readable
```
> Custom assertions make tests read in **domain language** (`assertThat(order).isPaid()`) and centralize repeated assertion logic. Worth it for domain types asserted across many tests.

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `isEqualTo` for floats | Use `isCloseTo(..., within(...))` |
| `isEqualTo` for BigDecimal (scale) | Use `isEqualByComparingTo` (Phase 4.1) |
| Field-by-field assertions for whole objects | Use `usingRecursiveComparison` |
| Comparing objects that include generated ids | `ignoringFields("id", "createdAt")` |
| One assertion stops the test (missing later failures) | Use soft assertions for independent checks |
| `containsExactly` when order shouldn't matter | Use `containsExactlyInAnyOrder` |
| Asserting whole objects when only fields matter | Use `extracting` |
| Mixing AssertJ and JUnit assertions randomly | Standardize on AssertJ |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **The standard assertion library** in Spring tests (`spring-boot-starter-test` — Phase 5.2).
- **`extracting` + `containsExactly`** for asserting on lists of entities/DTOs (Phase 5.4).
- **`usingRecursiveComparison`** to compare DTOs/entities without relying on `equals` (recall JPA pitfalls, Phase 5.4).
- **`isEqualByComparingTo`** for money assertions (`BigDecimal` — Phase 4.1, Projects 5/7).
- **`assertThatThrownBy`** for testing error handling (Phase 1.5, custom exceptions, ProblemDetail — Phase 5.3).
- **MockMvc assertions** (Phase 6.5) use a similar fluent style.
- Pairs with **Mockito** (Phase 6.3) — `verify` for interactions, `assertThat` for outcomes.

---

## 12. Quick Self-Check Questions

1. Why use AssertJ over JUnit's built-in assertions?
2. What's the single import, and how does chaining work?
3. How do you assert floating-point and BigDecimal equality?
4. How do you assert a collection contains specific elements (order vs no order)?
5. What does `extracting` do, and why is it useful?
6. How do you test exceptions with AssertJ?
7. What is `usingRecursiveComparison`, and when use it?
8. What are soft assertions, and when are they useful?

---

## 13. Key Terms Glossary

- **AssertJ:** fluent assertion library (`assertThat(...)`).
- **Fluent / chainable:** multiple assertions in one chained statement.
- **`extracting`:** map elements/objects to properties before asserting.
- **`tuple`:** extract multiple fields per element.
- **`filteredOn`:** filter a collection before asserting.
- **`isCloseTo` / `within`:** floating-point tolerance.
- **`isEqualByComparingTo`:** value comparison (BigDecimal).
- **`assertThatThrownBy`:** exception assertion.
- **`usingRecursiveComparison`:** field-by-field object comparison.
- **Soft assertions:** collect all failures before reporting.
- **Custom assertion:** domain-specific reusable assertion class.

---

*This is the note for **Section 6.4 — AssertJ (Fluent Assertions)**.*
*Next section in roadmap: **6.5 Spring Boot Testing**.*
