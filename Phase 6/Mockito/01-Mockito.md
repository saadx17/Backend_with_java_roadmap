# Mockito

> **Phase 6 — Testing → 6.3 Mockito**
> Goal: Master Mockito — creating mocks (`@Mock`/`@InjectMocks`), stubbing, verification, argument matchers/captors, spies, mocking statics, and the BDD style.

---

## 0. The Big Picture

**Mockito** is the standard Java **mocking** framework. A **mock** is a fake implementation of a dependency that you **control** — so you can test a class **in isolation** (a unit test, Phase 6.1) without its real collaborators (database, network, other services).

```
Test UserService in isolation:  replace its real UserRepository with a MOCK
  -> you control what the mock returns -> test ONLY the service's logic
```

> Mocking is what makes **unit tests** possible (recall Phase 6.1 — unit tests mock dependencies). It's enabled by **DI** (Phase 5.1 — because the service takes its dependencies via the constructor, you can inject mocks). This is *why* constructor injection and "program to interfaces" matter.

---

## 1. Why Mock? (Isolation)

To unit-test `UserService`, you don't want to hit a real database (slow, requires setup, non-deterministic — Phase 6.1). Instead, give it a **mock** `UserRepository` that returns exactly what you tell it:
```java
// Without mocking: needs a real DB (integration test — Phase 6.6)
// With mocking: control the dependency -> test ONLY the service's logic, fast & isolated
```
| Benefit | Why |
|---------|-----|
| **Isolation** | Test one unit; failures point to that unit (Phase 6.1) |
| **Speed** | No real I/O (DB/network) → fast (Phase 6.1) |
| **Determinism** | You control responses → no flakiness (Phase 6.1) |
| **Test error paths** | Make the mock throw → test exception handling (Phase 1.5) |
> Mocking lets you simulate **any** scenario — including ones hard to reproduce with real dependencies (a DB timeout, a specific exception, an empty result).

---

## 2. Creating Mocks

### 2.1 With annotations (preferred)
```java
@ExtendWith(MockitoExtension.class)          // enable Mockito (recall extensions, Phase 6.2)
class UserServiceTest {

    @Mock                                    // a mock of this dependency
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks                             // inject the @Mocks into this object's constructor
    private UserService userService;         // the class under test

    @Test
    void test() { ... }                      // userService has the mock repo & email injected
}
```
| Annotation | Effect |
|------------|--------|
| **`@Mock`** | Creates a mock of a dependency |
| **`@InjectMocks`** | Creates the object under test and injects the `@Mock`s into it |
| `@ExtendWith(MockitoExtension.class)` | Wires it all up (Phase 6.2) |
> ⭐ `@InjectMocks` injects the mocks via the **constructor** (recall constructor injection, Phase 5.1 — which is *why* this works cleanly). It tries constructor → setter → field injection.

### 2.2 Programmatically
```java
UserRepository repo = mock(UserRepository.class);   // create a mock directly
UserService service = new UserService(repo);         // inject it manually
```

---

## 3. Stubbing (Defining Mock Behavior)

**Stubbing** tells a mock what to return/do when called. By default, mocks return "empty" values (`null`, `0`, empty collections, `Optional.empty()`).
```java
// when/thenReturn — return a value:
when(userRepository.findById(1L)).thenReturn(Optional.of(new User("Alice")));  // (Optional — Phase 1.8)

// thenThrow — simulate an error (test the error path — Phase 1.5):
when(userRepository.findById(99L)).thenThrow(new EntityNotFoundException());

// thenAnswer — dynamic response based on arguments:
when(userRepository.save(any(User.class))).thenAnswer(inv -> inv.getArgument(0));  // return what was saved

// Multiple calls — different results in sequence:
when(counter.next()).thenReturn(1, 2, 3);   // returns 1, then 2, then 3 on successive calls
```
| Stub | Use |
|------|-----|
| `when(x).thenReturn(v)` | Return a value |
| `when(x).thenThrow(e)` | Throw an exception |
| `when(x).thenAnswer(...)` | Compute the response dynamically |

### 3.1 doReturn/doThrow (for void methods & spies)
```java
doThrow(new RuntimeException()).when(emailService).send(any());   // stub a VOID method
doNothing().when(emailService).send(any());                       // explicitly do nothing
```
> ⚠️ For **void methods** (and spies — §6), you can't use `when(...)` (the method returns nothing) — use the **`doThrow`/`doNothing`/`doReturn` ... `.when(mock)`** syntax instead.

---

## 4. Verification (Checking Interactions)

**Verification** checks that the mock was **called as expected** — useful when the behavior under test is an *interaction* (e.g., "the email service was called once"), not just a return value.
```java
verify(emailService).send(any());                       // called exactly once (default)
verify(emailService, times(2)).send(any());             // called exactly twice
verify(emailService, never()).send(any());              // never called
verify(emailService, atLeast(1)).send(any());           // at least once
verify(emailService, atMost(3)).send(any());
verify(userRepository).save(user);                      // called with this exact argument
verifyNoInteractions(emailService);                     // no method was ever called
verifyNoMoreInteractions(userRepository);               // no calls beyond those already verified
```
| Verification | Checks |
|--------------|--------|
| `verify(mock).method()` | Called exactly once |
| `times(n)` / `never()` / `atLeast(n)` / `atMost(n)` | Call count |
| `verifyNoInteractions(mock)` | Never touched |
> Verification tests **behavior** (did the right interaction happen?). ⚠️ **Don't over-verify** — verifying every call couples the test to implementation (recall Phase 6.1 — test behavior, not implementation). Verify the *important* interactions (e.g., "the order was saved," "the notification was sent").

---

## 5. Argument Matchers & ArgumentCaptor

### 5.1 Argument matchers
Match arguments flexibly when stubbing/verifying:
```java
when(repo.findByStatus(any(Status.class))).thenReturn(List.of());   // any Status
verify(repo).save(any(User.class));                                  // any User
when(repo.findById(eq(1L))).thenReturn(...);                         // exactly 1L
verify(service).process(argThat(order -> order.getTotal() > 100));   // custom predicate
```
| Matcher | Matches |
|---------|---------|
| `any()` / `any(Type.class)` | Any value of the type |
| `eq(value)` | Exactly that value |
| `anyString()`, `anyLong()`, etc. | Any of that type |
| `argThat(predicate)` | A custom condition (Phase 1.8 predicate) |
> ⚠️ **If you use a matcher for one argument, you must use matchers for ALL arguments** in that call (`verify(s).m(eq(1), any())`, not `verify(s).m(1, any())`) — a common Mockito error.

### 5.2 ArgumentCaptor (capture & assert arguments)
**`ArgumentCaptor`** captures the actual argument a mock was called with, so you can assert on it:
```java
@Captor ArgumentCaptor<User> userCaptor;        // or ArgumentCaptor.forClass(User.class)

@Test
void shouldSaveUserWithCorrectData() {
    userService.register("Alice", "alice@x.com");
    verify(userRepository).save(userCaptor.capture());   // capture the saved User
    User saved = userCaptor.getValue();
    assertThat(saved.getName()).isEqualTo("Alice");      // assert on it (AssertJ — Phase 6.4)
    assertThat(saved.getEmail()).isEqualTo("alice@x.com");
}
```
> `ArgumentCaptor` is the way to verify **what** was passed to a mock (not just *that* it was called) — e.g., "the user saved had the right email," "the event published had the right fields." Very useful for testing transformations.

---

## 6. Spies (Partial Mocks)

A **spy** wraps a **real object** — by default it calls the **real methods**, but you can stub specific ones. (A mock fakes everything; a spy is real except where you override.)
```java
@Spy
private List<String> spyList = new ArrayList<>();   // a real ArrayList, spied on

@Test
void test() {
    spyList.add("a");                  // calls the REAL add (the list actually changes)
    assertEquals(1, spyList.size());    // real behavior
    doReturn(100).when(spyList).size(); // stub ONLY size()
    assertEquals(100, spyList.size());  // now stubbed
}
```
> ⚠️ **Spies are a code smell if overused** — wanting to mock part of a class often means the class does too much (Single Responsibility — Phase 14). Prefer mocks. Use spies for legacy code or when you genuinely need real behavior plus one stubbed method. (Note: stub spies with `doReturn().when()`, not `when()`.)

---

## 7. Mocking Static Methods

Historically Mockito couldn't mock statics; now `mockStatic` can (use sparingly):
```java
try (MockedStatic<LocalDate> mocked = mockStatic(LocalDate.class)) {
    mocked.when(LocalDate::now).thenReturn(LocalDate.of(2026, 1, 1));   // control "now"
    // ... test code that calls LocalDate.now() ...
}   // the static mock is scoped to this block
```
> ⚠️ **Needing to mock statics is usually a design smell** — static dependencies are hard to test (recall Phase 1.3 — static methods aren't injectable). The better fix is to **inject a `Clock`** (or wrap the static in an injectable bean) rather than mock `LocalDate.now()` (recall flaky-time tests, Phase 6.1). `mockStatic` is an escape hatch for code you can't change.

---

## 8. BDD Mockito Style (given/when/then)

Mockito offers a **BDD-style** API (recall BDD Given-When-Then, Phase 6.1) — `given().willReturn()` reads more like a specification:
```java
import static org.mockito.BDDMockito.*;

@Test
void shouldReturnUser() {
    // GIVEN
    given(userRepository.findById(1L)).willReturn(Optional.of(user));
    // WHEN
    User result = userService.getById(1L);
    // THEN
    assertThat(result).isEqualTo(user);          // (AssertJ — Phase 6.4)
    then(userRepository).should().findById(1L);  // verify, BDD-style
}
```
| Standard Mockito | BDD Mockito |
|------------------|-------------|
| `when(x).thenReturn(v)` | `given(x).willReturn(v)` |
| `verify(mock).method()` | `then(mock).should().method()` |
> BDD-style aligns with the Given-When-Then structure (Phase 6.1) — purely stylistic, but many teams prefer it for readability. Functionally identical to standard Mockito.

---

## 9. A Complete Unit Test Example
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository userRepository;
    @Mock EmailService emailService;
    @InjectMocks UserService userService;

    @Test
    @DisplayName("registers a user and sends a welcome email")
    void shouldRegisterUser() {
        // ARRANGE (Phase 6.1 AAA)
        var request = new RegisterRequest("Alice", "alice@x.com");
        given(userRepository.existsByEmail("alice@x.com")).willReturn(false);
        given(userRepository.save(any(User.class))).willAnswer(inv -> inv.getArgument(0));

        // ACT
        User result = userService.register(request);

        // ASSERT
        assertThat(result.getName()).isEqualTo("Alice");
        then(emailService).should().sendWelcome("alice@x.com");   // verify the email was sent
    }

    @Test
    void shouldThrowWhenEmailExists() {
        given(userRepository.existsByEmail("alice@x.com")).willReturn(true);
        assertThatThrownBy(() -> userService.register(new RegisterRequest("Alice", "alice@x.com")))
            .isInstanceOf(EmailAlreadyExistsException.class);     // test the error path (Phase 1.5)
        then(emailService).should(never()).sendWelcome(any());    // email NOT sent
    }
}
```

---

## 10. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Over-mocking (mocking everything) | Mock only external dependencies; don't mock value objects |
| Over-verifying (testing implementation) | Verify important interactions only (Phase 6.1) |
| Mixing matchers and raw values in one call | Use matchers for all args or none |
| `when()` on a void method/spy | Use `doThrow`/`doNothing`/`doReturn().when()` |
| Mocking statics frequently | Inject a `Clock`/wrapper instead (design smell) |
| Spies everywhere | Prefer mocks; spies hint at design issues |
| Mocking what you should integration-test | Some things need real (DB → Phase 6.6) |
| Not resetting mocks between tests | `MockitoExtension` handles it per test (isolation, Phase 6.1) |

---

## 11. Connection to Backend / Spring (Why This Matters Later)

- **Unit-testing services** (Phase 5.1): mock the repository, test the business logic in isolation (Phase 6.1).
- **DI enables mocking** (Phase 5.1) — constructor injection + `@InjectMocks`. This is *why* good design = testable.
- **`@MockBean`** (Phase 6.5) is Spring's way to put a Mockito mock into the application context.
- **Testing error paths** (Phase 1.5): `thenThrow` to simulate DB failures, timeouts, etc.
- **AssertJ** (Phase 6.4) pairs with Mockito for fluent assertions.
- **BDD style** aligns with Given-When-Then (Phase 6.1).
- **Don't mock everything** — integration tests (Phase 6.6) verify real DB/wiring that mocks can't.

---

## 12. Quick Self-Check Questions

1. What is a mock, and why does it enable unit testing?
2. How does DI (Phase 5.1) make mocking possible?
3. What do `@Mock` and `@InjectMocks` do?
4. How do you stub a method to return a value? To throw? To handle a void method?
5. What is verification, and what's the danger of over-verifying?
6. What's the matchers rule (mixing matchers and raw values)?
7. What is an `ArgumentCaptor` for?
8. What's the difference between a mock and a spy? When is a spy a smell?
9. Why is needing `mockStatic` often a design problem?

---

## 13. Key Terms Glossary

- **Mock:** a controllable fake of a dependency.
- **Mockito:** the standard Java mocking framework.
- **`@Mock` / `@InjectMocks`:** create mocks / inject them into the unit under test.
- **Stubbing (`when/thenReturn`, `doThrow`):** define mock behavior.
- **Verification (`verify`, `times`, `never`):** check interactions.
- **Argument matcher (`any`, `eq`, `argThat`):** flexible argument matching.
- **`ArgumentCaptor`:** capture and assert on call arguments.
- **Spy:** a partial mock (real object with selective stubbing).
- **`mockStatic`:** mock static methods (use sparingly).
- **BDD Mockito (`given`/`then`):** Given-When-Then style API.

---

*This is the note for **Section 6.3 — Mockito**.*
*Next section in roadmap: **6.4 AssertJ (Fluent Assertions)**.*
