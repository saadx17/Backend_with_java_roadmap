# Integration Testing Strategy

> **Phase 6 — Testing → 6.6 Integration Testing Strategy**
> Goal: Develop a coherent integration-testing strategy — test database choice (H2 vs Testcontainers), test data setup, transactional rollback, full API tests, and mocking external HTTP services (WireMock).

---

## 0. The Big Picture

**Integration tests** verify that **components work together correctly** — controller → service → repository → real database, or your app → an external API. They catch the bugs unit tests (with mocks) can't: wiring issues, SQL/dialect problems, serialization, transactions, and integration with real dependencies.

```
Unit test (Phase 6.3):  one class, mocked dependencies     -> tests logic
Integration test:        real components together (real DB) -> tests they actually fit
```

> This note is about *strategy* — the decisions that make integration tests **trustworthy** (real dependencies) and **maintainable** (clean data, isolation). It builds on Spring Boot testing (Phase 6.5) and the database/HTTP foundations (Phase 4, 0.2).

---

## 1. What Integration Tests Catch (That Unit Tests Don't)

| Bug type | Why unit tests miss it |
|----------|------------------------|
| **SQL/query errors** | Mocked repositories don't run real SQL (Phase 5.4) |
| **DB dialect/type issues** | Mocks don't hit a real database (JSONB, enums — Phase 4.6) |
| **Transaction behavior** | Rollback/propagation only happens with a real DB (Phase 5.5) |
| **Wiring/config issues** | Mocks bypass Spring's actual bean wiring (Phase 5.1) |
| **Serialization** | Real JSON (de)serialization (Phase 5.3) |
| **Security config** | Real filter chain (Phase 5.6) |
> ⭐ Integration tests verify the parts you **can't mock meaningfully** — especially **database access** (does the query actually work against PostgreSQL?). They're slower (Phase 6.1 pyramid → fewer of them) but catch a critical class of bugs.

---

## 2. Test Database Strategy: H2 vs Testcontainers (The Key Decision)

> ⚠️ **The most important integration-testing decision: what database do tests run against?**
| | **H2 (in-memory)** | **Testcontainers (real PostgreSQL)** |
|---|--------------------|---------------------------------------|
| Speed | Very fast (in-memory) | Slower (starts a Docker container) |
| Fidelity | ⚠️ **Different** from PostgreSQL (dialect, types, SQL) | ✅ **Identical** to production |
| Catches dialect bugs | ❌ No | ✅ Yes |
| Requires Docker | No | Yes (Phase 10) |
| JSONB/arrays/full-text (Phase 4.6) | ❌ Not supported | ✅ Supported |

### 2.1 The recommendation
> ⭐ **Use Testcontainers (real PostgreSQL) for integration tests, not H2.** H2 is fast but **lies** — tests can pass on H2 yet fail in production because H2's SQL dialect, types, and behavior differ from PostgreSQL (Phase 4.6). Testing against the **same database as production** is the whole point of integration testing. H2 is acceptable only for the quickest smoke tests where DB fidelity doesn't matter. (Recall Phase 6.5 — `@AutoConfigureTestDatabase(replace = NONE)` to disable the H2 swap.)

### 2.2 Testcontainers setup (recall Phase 6.5)
```java
@SpringBootTest
@Testcontainers
@AutoConfigureTestDatabase(replace = NONE)        // use the real DB
class OrderIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    @DynamicPropertySource
    static void config(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }
}
```
> **Performance tip:** share one container across many test classes (a `static` container + a base class, or Testcontainers' **reusable** mode / Singleton Container pattern) — starting a container per test class is slow. Run migrations (Flyway — Phase 8) against it so the schema matches production.

---

## 3. Test Data Setup

Integration tests need **realistic, controlled** data. Strategies (recall Phase 6.5):
| Approach | How | Best for |
|----------|-----|----------|
| **Object Mother / builders** | `aUser().withEmail("x").build()` (Phase 1.3 builder) | Reusable, readable test objects |
| **`@Sql` scripts** | `@Sql("/data.sql")` runs SQL before a test | Loading fixtures/datasets |
| **Programmatic (`@BeforeEach`)** | Insert via repository/`TestEntityManager` | Per-test setup |
| **Migrations (Flyway)** | The real schema applied to the test DB | Schema parity with prod (Phase 8) |
```java
@Test
@Sql("/test-data/orders.sql")                     // load fixtures before this test
void shouldComputeOrderTotals() { ... }
```
> ⭐ **Object Mother / test data builders** are the cleanest approach — centralized factories that produce **valid** domain objects, customizable per test (`anOrder().withStatus(SHIPPED).build()`). They reduce duplication and survive schema changes. Combine with Testcontainers + Flyway for production-like data on a production-like schema.

---

## 4. Test Isolation: Cleaning State (Critical)

> ⚠️ **Integration tests share a database → they MUST clean up after themselves**, or one test's data pollutes another (breaking isolation — Phase 6.1). Strategies:
| Strategy | How | Caveat |
|----------|-----|--------|
| **Transactional rollback** | `@Transactional` on the test → auto-rollback (Phase 6.5) | ⚠️ Doesn't cover real HTTP-call transactions (`RANDOM_PORT`) |
| **`@AfterEach` cleanup** | `repository.deleteAll()` / truncate after each test | Explicit, always works |
| **Fresh schema/container** | Recreate the DB per test class | Slow but fully isolated |
| **`@Sql` cleanup script** | Run a teardown script after | Explicit |
```java
@AfterEach
void cleanUp() {
    orderRepository.deleteAll();   // clean state for the next test (when rollback doesn't apply)
}
```
> ⚠️ **The transactional-rollback caveat (recall Phase 6.5):** with `@SpringBootTest(RANDOM_PORT)` + real HTTP requests, the request runs in its **own transaction** (separate thread) — the test's rollback won't undo it. In that case, clean up explicitly (`@AfterEach`, truncate). For `@DataJpaTest`/service-level tests, rollback works.

---

## 5. Full API Integration Tests (End-to-End Through HTTP)

Test the **whole flow** — a real HTTP request through the controller, service, repository, to a real DB — verifying the complete request/response cycle (Phase 0.2/5.3):
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserApiIntegrationTest {
    @Container static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    // ... @DynamicPropertySource ...
    @Autowired TestRestTemplate restTemplate;        // a real HTTP client (or WebTestClient)

    @Test
    void shouldCreateAndRetrieveUser() {
        // CREATE (POST)
        var create = restTemplate.postForEntity("/api/users",
            new CreateUserRequest("Alice", "alice@x.com"), UserDto.class);
        assertThat(create.getStatusCode()).isEqualTo(HttpStatus.CREATED);   // 201 (Phase 0.2)
        Long id = create.getBody().id();

        // RETRIEVE (GET) — proves it was actually persisted
        var get = restTemplate.getForEntity("/api/users/" + id, UserDto.class);
        assertThat(get.getBody().name()).isEqualTo("Alice");   // (AssertJ — Phase 6.4)
    }
}
```
> Full API tests exercise **everything together** (routing, validation, serialization, security, transactions, real SQL) — the highest-confidence test that the system works end-to-end. Keep them few (Phase 6.1 pyramid) and focused on critical flows. `TestRestTemplate`/`WebTestClient` make real HTTP calls to the random-port server.

---

## 6. Mocking External HTTP Services (WireMock)

> ⚠️ **Don't call real external services (payment APIs, third-party APIs) in tests** — they're slow, flaky, cost money, and may have rate limits (recall flaky tests, Phase 6.1). Instead, **mock the external HTTP service** with **WireMock** — a fake HTTP server that returns canned responses.
```java
@SpringBootTest
class PaymentIntegrationTest {
    static WireMockServer wireMock = new WireMockServer(8089);   // a fake HTTP server

    @BeforeAll static void start() {
        wireMock.start();
        wireMock.stubFor(post(urlEqualTo("/charges"))            // stub the external endpoint
            .willReturn(aResponse().withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"status\":\"succeeded\",\"id\":\"ch_123\"}")));
    }

    @Test
    void shouldProcessPayment() {
        // your code calls http://localhost:8089/charges -> WireMock responds with the stub
        PaymentResult result = paymentService.charge(request);
        assertThat(result.status()).isEqualTo("succeeded");
    }
}
```
> **WireMock** lets you test your **integration with an external API** without the real service — stubbing responses (success, errors, timeouts, slow responses) to test all paths, including failure handling (resilience — Phase 12.3). Point your WebClient/RestTemplate (Phase 5.9) at WireMock's URL in tests. (It can also *record* real responses to replay later.)

---

## 7. A Layered Integration-Testing Strategy
```
1. UNIT tests (most)        -> services/logic with mocks (Phase 6.3) — fast
2. SLICE tests             -> @WebMvcTest (controllers), @DataJpaTest (repos, Testcontainers) — Phase 6.5
3. INTEGRATION tests       -> @SpringBootTest + Testcontainers (real DB) — full wiring
4. API/E2E tests (few)     -> RANDOM_PORT + real HTTP through the whole stack
5. EXTERNAL deps           -> WireMock for third-party APIs
```
> This mirrors the testing pyramid (Phase 6.1): mostly fast unit tests, a focused set of slice/integration tests against **real** dependencies (Testcontainers/WireMock), and a few full API tests for critical flows. Trustworthy *and* fast.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Testing repositories against H2 | Use Testcontainers (real PostgreSQL) |
| Tests polluting each other's data | Clean state (rollback / `@AfterEach` / fresh schema) |
| Assuming rollback covers `RANDOM_PORT` HTTP calls | It doesn't — clean up explicitly |
| Calling real external APIs in tests | Mock with WireMock |
| Too many slow `@SpringBootTest` tests | Follow the pyramid; prefer slices/unit |
| Starting a container per test class | Share/reuse containers |
| No schema parity (test vs prod) | Run Flyway migrations on the test DB (Phase 8) |
| Duplicated test data setup | Use Object Mother / builders |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Testcontainers** (Phase 6.5) is the backbone — real **PostgreSQL** (Phase 4.6), **Redis** (Phase 4.8), **Kafka** (Phase 11) in tests.
- **Repository tests** (Phase 5.4) need a real DB to catch query/dialect bugs.
- **Transaction tests** (Phase 5.5) need a real DB to verify rollback/propagation.
- **WireMock** mocks external APIs (payment — Projects 5/7; microservice calls — Phase 12) and tests resilience (Phase 12.3).
- **Flyway migrations** (Phase 8) on the test DB ensure schema parity.
- **Projects 4, 5, 7** explicitly require "integration tests with Testcontainers."
- **CI/CD** (Phase 10.3) runs these (needs Docker for Testcontainers).

---

## 10. Quick Self-Check Questions

1. What bugs do integration tests catch that unit tests miss?
2. Why prefer Testcontainers over H2 for repository/integration tests?
3. How do you share a container across test classes for speed?
4. What are the main test-data-setup strategies?
5. Why must integration tests clean up state, and what's the rollback caveat?
6. How do you write a full API integration test through real HTTP?
7. Why shouldn't tests call real external services, and what's the alternative?
8. How does WireMock let you test integration with (and failures of) an external API?

---

## 11. Key Terms Glossary

- **Integration test:** tests components working together (real dependencies).
- **H2 vs Testcontainers:** in-memory (low fidelity) vs real DB in Docker (high fidelity).
- **Testcontainers:** real services (PostgreSQL/Redis/Kafka) in containers for tests.
- **Reusable / singleton container:** sharing a container for speed.
- **Test data setup (Object Mother / `@Sql` / builders):** creating test data.
- **Transactional rollback:** auto-cleanup per test (with caveats).
- **Full API / E2E test:** real HTTP through the whole stack.
- **`TestRestTemplate` / `WebTestClient`:** test HTTP clients.
- **WireMock:** a fake HTTP server for mocking external services.
- **Schema parity:** the test DB matching production (via migrations).

---

*This is the note for **Section 6.6 — Integration Testing Strategy**.*
*Next section in roadmap: **6.7 Architecture Testing (ArchUnit)**.*
