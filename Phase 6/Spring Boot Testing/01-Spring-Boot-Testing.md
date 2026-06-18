# Spring Boot Testing

> **Phase 6 — Testing → 6.5 Spring Boot Testing**
> Goal: Master Spring Boot's testing support — `@SpringBootTest`, test slices (`@WebMvcTest` + MockMvc, `@DataJpaTest`), `@MockBean`/`@SpyBean`, test configuration & profiles, and Testcontainers.

---

## 0. The Big Picture

Spring Boot provides testing tools that **load (parts of) the Spring application context** so you can test your beans, controllers, and repositories **as Spring wires them** (not just in isolation). The key skill: load **only what you need** (a "slice") for fast, focused tests.

```
Unit test (Phase 6.3): no Spring, mocks -> fast, tests logic
Slice test:   loads ONE layer (web OR data) -> medium, tests that layer wired
@SpringBootTest: loads the WHOLE context -> slow, tests everything together
```

> This bridges unit tests (Phase 6.3 — mocks, no Spring) and full integration tests (Phase 6.6). It builds on JUnit 5 (Phase 6.2), Mockito (6.3), AssertJ (6.4), and the Spring stack (Phase 5). `spring-boot-starter-test` (Phase 5.2) bundles all of it.

---

## 1. The Spectrum of Spring Tests

| Approach | Loads | Speed | Use for |
|----------|-------|-------|---------|
| **Plain unit test** (Phase 6.3) | Nothing (mocks) | Fastest | Service/business logic in isolation |
| **`@WebMvcTest`** | Just the web layer (controllers) | Fast | Controller tests (with mocked services) |
| **`@DataJpaTest`** | Just the JPA layer (repositories + DB) | Fast | Repository/query tests |
| **`@SpringBootTest`** | The **whole** application context | Slow | Full integration tests (Phase 6.6) |
> ⭐ **Prefer the narrowest slice** for the job — slices start much faster than the full context (recall JVM/Spring startup cost, Phase 1.11/5.2). Use `@SpringBootTest` only for true end-to-end integration (Phase 6.6). Most tests should be plain unit tests (Phase 6.1 pyramid).

---

## 2. @WebMvcTest + MockMvc (Testing Controllers)

**`@WebMvcTest`** loads **only the web layer** (controllers, filters, JSON serialization — Phase 5.3) — not services or repositories. You mock the service layer with `@MockBean` (§5). **MockMvc** simulates HTTP requests *without* starting a real server.
```java
@WebMvcTest(UserController.class)            // load ONLY this controller's web slice
class UserControllerTest {

    @Autowired MockMvc mockMvc;              // simulates HTTP requests
    @MockBean UserService userService;       // the service is MOCKED (not real — §5)

    @Test
    void shouldReturnUser() throws Exception {
        given(userService.findById(1L)).willReturn(new UserDto(1L, "Alice"));  // stub (Phase 6.3)

        mockMvc.perform(get("/api/users/1"))                 // simulate GET /api/users/1
            .andExpect(status().isOk())                       // assert 200 (Phase 0.2)
            .andExpect(jsonPath("$.id").value(1))             // assert JSON body fields
            .andExpect(jsonPath("$.name").value("Alice"));
    }

    @Test
    void shouldReturn400OnInvalidInput() throws Exception {
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"\"}"))                   // invalid (blank name)
            .andExpect(status().isBadRequest());              // validation -> 400 (Phase 5.3)
    }
}
```
| MockMvc piece | Does |
|---------------|------|
| `perform(get/post/put/delete(...))` | Simulate an HTTP request |
| `.contentType(...)` / `.content(...)` | Set the request body (JSON) |
| `andExpect(status()...)` | Assert the HTTP status (Phase 0.2) |
| `andExpect(jsonPath("$.field")...)` | Assert JSON response fields |
| `andExpect(content()...)` | Assert the raw body |
> ⭐ `@WebMvcTest` tests the **controller layer in isolation** — request mapping, validation (Phase 5.3), JSON serialization, status codes, exception handling — with the **service mocked**. MockMvc is fast (no real server/HTTP). `jsonPath` (JSONPath syntax) navigates the JSON response.

### 2.1 Security context in MockMvc
```java
@Test
@WithMockUser(roles = "ADMIN")               // simulate an authenticated ADMIN (Phase 5.6)
void adminCanDelete() throws Exception {
    mockMvc.perform(delete("/api/users/1")).andExpect(status().isNoContent());
}
```
> `@WithMockUser` (from `spring-security-test`) simulates an authenticated user/role (recall Spring Security, Phase 5.6) — so you can test that endpoints are properly secured (401/403 — Phase 0.2/5.6).

---

## 3. @DataJpaTest (Testing Repositories)

**`@DataJpaTest`** loads **only the JPA layer** (entities, repositories — Phase 5.4) — not controllers or services. It's **transactional** (rolls back after each test — §6) and can use a real or in-memory database.
```java
@DataJpaTest                                 // load ONLY the JPA slice
class UserRepositoryTest {

    @Autowired UserRepository userRepository;
    @Autowired TestEntityManager entityManager;   // helper to set up test data

    @Test
    void shouldFindByEmail() {
        // ARRANGE: persist a test entity
        entityManager.persistAndFlush(new User("alice@x.com", "Alice"));

        // ACT
        Optional<User> found = userRepository.findByEmail("alice@x.com");   // (Optional — Phase 1.8)

        // ASSERT
        assertThat(found).isPresent();                                       // (AssertJ — Phase 6.4)
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```
| `@DataJpaTest` feature | Note |
|------------------------|------|
| Loads only JPA components | Fast (no web/service layers) |
| **`TestEntityManager`** | Set up/inspect entities directly in tests |
| **Transactional + rollback** | Each test rolls back → isolation (§6) |
| Default DB | An **embedded H2** unless configured otherwise (⚠️ §3.1) |

### 3.1 H2 vs real database (a key decision)
> ⚠️ **By default, `@DataJpaTest` uses an embedded H2 database** — fast, but **H2 ≠ PostgreSQL** (Phase 4.6). Tests can pass on H2 but **fail on real PostgreSQL** because dialects, types (JSONB, arrays), and SQL features differ. **For trustworthy repository tests, use the real database via Testcontainers** (§7) — disable the H2 replacement with `@AutoConfigureTestDatabase(replace = NONE)`. (Detailed in Phase 6.6.)

---

## 4. @SpringBootTest (Full Integration)

**`@SpringBootTest`** loads the **entire application context** — all your beans, wired as in production. For testing the whole stack together (controller → service → repository → DB).
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)   // start a real server on a random port
@AutoConfigureMockMvc
class UserIntegrationTest {

    @Autowired TestRestTemplate restTemplate;   // a real HTTP client (or @Autowired WebTestClient)
    // ... or @Autowired MockMvc

    @Test
    void fullFlow() {
        var response = restTemplate.postForEntity("/api/users", request, UserDto.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```
| `WebEnvironment` | Behavior |
|------------------|----------|
| `MOCK` (default) | A mock servlet environment (use with MockMvc) |
| `RANDOM_PORT` | Start a **real** embedded server on a random port |
| `DEFINED_PORT` | Real server on the configured port |
| `NONE` | No web environment |
> `@SpringBootTest` is the **heaviest** (loads everything → slow startup) — use it for genuine **end-to-end integration tests** (Phase 6.6), not for unit-level logic. With `RANDOM_PORT`, it starts a real server and you make real HTTP calls (via `TestRestTemplate`/`WebTestClient`).

---

## 5. @MockBean vs @SpyBean

In Spring tests, replace a real bean in the context with a Mockito mock/spy:
| Annotation | Effect |
|------------|--------|
| **`@MockBean`** | Replace a bean with a **Mockito mock** (Phase 6.3) in the context |
| **`@SpyBean`** | Wrap the **real** bean in a Mockito spy (real behavior + selective stubbing — Phase 6.3) |
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockBean UserService userService;       // the real UserService is replaced by a mock
    // -> controller uses the mock; you control its behavior with given()/when()
}
```
> `@MockBean` is how you mock dependencies in **slice/integration** tests (it puts a mock *into* the Spring context). It differs from plain `@Mock` (Phase 6.3 — which doesn't involve Spring). ⚠️ `@MockBean` **replaces** the bean for the whole context → can slow tests (context cache invalidation); prefer plain unit tests where Spring isn't needed.

---

## 6. Test Transactions & Isolation

> ⚠️ **`@DataJpaTest` and `@Transactional` tests roll back after each test by default** — so each test starts with a clean database (isolation — Phase 6.1). Changes are *not* committed.
```java
@SpringBootTest
@Transactional                              // each test rolls back at the end
class ServiceTest {
    @Test void test() {
        // changes here are rolled back -> next test sees a clean state
    }
}
```
- Rollback gives **test isolation** without manual cleanup (recall Phase 6.1 — fresh state per test).
- ⚠️ **Caveat:** with `@SpringBootTest(RANDOM_PORT)` + real HTTP calls, the request runs in a **separate thread/transaction** from the test → the test's transaction rollback may **not** cover it. In that case, clean up data manually (`@Sql`, `deleteAll()` in `@AfterEach`) or use a fresh container per test.

### 6.1 Test data setup
| Approach | How |
|----------|-----|
| **`@Sql`** | Run a SQL script before/after a test (`@Sql("/test-data.sql")`) |
| **Builder pattern / Object Mother** | Helper methods/builders that create test objects (recall builders, Phase 1.3) |
| **`TestEntityManager`** | Persist entities directly (`@DataJpaTest`) |
| **`@BeforeEach`** | Programmatic setup (Phase 6.2) |
> **Object Mother / test data builders** centralize creating valid test objects (`aUser().withEmail("x").build()`) — reducing duplication and making tests readable. `@Sql` is handy for loading fixtures.

---

## 7. Testcontainers (Real Dependencies in Tests)

> ⭐ **Testcontainers** runs **real dependencies** (PostgreSQL, Redis, Kafka) in **Docker containers** during your tests — giving you production-like integration tests instead of fakes (H2) or mocks.
```java
@SpringBootTest
@Testcontainers                              // enable Testcontainers
@AutoConfigureTestDatabase(replace = NONE)   // use the real DB, not H2 (recall §3.1)
class UserIntegrationTest {

    @Container                                // a real PostgreSQL in Docker (Phase 4.6)
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource                    // wire the container's URL into Spring
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test void test() { /* tests run against REAL PostgreSQL */ }
}
```
| Testcontainers piece | Does |
|----------------------|------|
| `@Testcontainers` / `@Container` | Manage container lifecycle (start/stop) |
| `PostgreSQLContainer` / `GenericContainer` / `KafkaContainer` | Spin up real services (Phase 4.6/4.8/11) |
| `@DynamicPropertySource` | Inject the container's connection details into Spring config (Phase 5.1) |
| **Wait strategies** | Wait until the container is ready before running tests |
| **Reusable containers** | Keep containers across runs for speed (dev) |
> ⭐ **Testcontainers is the gold standard for integration testing** — you test against the **same database/broker as production** (recall PostgreSQL-vs-H2, §3.1), catching dialect/feature differences that H2/mocks hide. Requires Docker (Phase 10). Slower than slices but far more trustworthy. (Deep dive in Phase 6.6.)

---

## 8. Test Slices Summary

Spring Boot provides focused slices for each layer:
| Slice annotation | Loads | For |
|------------------|-------|-----|
| `@WebMvcTest` | Web layer (controllers) | Controller tests + MockMvc |
| `@DataJpaTest` | JPA layer (repositories) | Repository/query tests |
| `@JsonTest` | Jackson serialization | JSON (de)serialization tests (Phase 5.3) |
| `@RestClientTest` | REST clients (WebClient/RestTemplate) | Outbound HTTP client tests (Phase 5.9) |
| `@JdbcTest` | Plain JDBC layer | JdbcTemplate tests (Phase 4.7) |
| `@DataRedisTest` | Redis layer | Redis tests (Phase 4.8) |
> Slices load **only the relevant beans** → fast, focused tests. Use the slice that matches what you're testing; reach for `@SpringBootTest` only for full end-to-end (Phase 6.6).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `@SpringBootTest` for everything (slow) | Use slices / plain unit tests where possible |
| `@DataJpaTest` on H2 (passes, but prod differs) | Use Testcontainers + `replace = NONE` |
| Forgetting `@MockBean` for services in `@WebMvcTest` | Mock the service layer |
| Expecting rollback to cover real HTTP-call data | Clean up manually with `RANDOM_PORT` |
| Many distinct test contexts (slow startup) | Reuse context config (Spring caches contexts) |
| `@WithMockUser` missing for secured endpoints | Add it to test authorization (Phase 5.6) |
| Loading the full context to test one query | Use `@DataJpaTest` |
| Not using `@DynamicPropertySource` with Testcontainers | Required to wire the container into Spring |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Controller tests** (`@WebMvcTest` + MockMvc) verify the web layer (Phase 5.3) — mapping, validation, status codes, JSON.
- **Repository tests** (`@DataJpaTest`) verify queries (Phase 5.4) — best against real PostgreSQL (Testcontainers).
- **`@MockBean`** integrates Mockito (Phase 6.3) into the Spring context.
- **`@WithMockUser`** tests Spring Security (Phase 5.6).
- **Testcontainers** is the foundation of integration testing (Phase 6.6) and used in **Projects 4, 5, 7** ("integration tests with Testcontainers").
- **Test profiles** (`@ActiveProfiles("test")` — Phase 5.1) isolate test config.
- **CI/CD** (Phase 10.3) runs these (Docker needed for Testcontainers).

---

## 11. Quick Self-Check Questions

1. What's the spectrum from unit tests to `@SpringBootTest`, and which to prefer?
2. What does `@WebMvcTest` load, and what is MockMvc?
3. How do you assert HTTP status and JSON fields with MockMvc?
4. What does `@DataJpaTest` load, and what's the H2-vs-PostgreSQL caveat?
5. When use `@SpringBootTest`, and what does `RANDOM_PORT` do?
6. What's the difference between `@MockBean` and plain `@Mock`?
7. How does test transaction rollback provide isolation, and what's the `RANDOM_PORT` caveat?
8. What is Testcontainers, and why is it better than H2/mocks for integration tests?

---

## 12. Key Terms Glossary

- **Test slice:** loads only one layer of the context (`@WebMvcTest`, `@DataJpaTest`).
- **`@WebMvcTest` / MockMvc:** web-layer test / simulated HTTP requests.
- **`jsonPath`:** navigate/assert JSON response fields.
- **`@DataJpaTest` / `TestEntityManager`:** JPA-layer test / test data helper.
- **`@SpringBootTest`:** full-context test (`WebEnvironment` options).
- **`@MockBean` / `@SpyBean`:** mock / spy a bean in the Spring context.
- **Transactional rollback:** auto-rollback per test for isolation.
- **`@Sql` / Object Mother / builder:** test data setup approaches.
- **Testcontainers (`@Container`, `@DynamicPropertySource`):** real dependencies in Docker.
- **`@WithMockUser`:** simulate an authenticated user.
- **`@ActiveProfiles`:** activate a test profile.

---

*This is the note for **Section 6.5 — Spring Boot Testing**.*
*Next section in roadmap: **6.6 Integration Testing Strategy**.*
