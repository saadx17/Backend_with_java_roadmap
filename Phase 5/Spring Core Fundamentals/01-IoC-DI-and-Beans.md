# Spring Core: IoC, Dependency Injection & Beans

> **Phase 5 — Spring Framework & Spring Boot → 5.1 Spring Core Fundamentals**
> Goal: Master the heart of Spring — Inversion of Control (IoC), the IoC container, Dependency Injection (DI) and its types, beans, and component scanning.

---

## 0. The Big Picture

The **core idea of Spring** is **Inversion of Control (IoC)**: instead of your code creating and wiring its own dependencies (`new`), a **container** creates objects (**beans**) and **injects** their dependencies for you. This is **Dependency Injection (DI)**.

```
Without Spring:  UserService service = new UserService(new UserRepository(new DataSource(...)));
With Spring:     the container builds & wires everything; you just ask for UserService
```

> Everything in Spring builds on this. Understanding IoC/DI deeply makes the entire framework intuitive. It's the practical application of "program to interfaces" and "favor composition over inheritance" (recall Phase 1.3) at framework scale.

---

## 1. Inversion of Control (IoC)

**IoC** means **inverting the control** of object creation and lifecycle: your code no longer controls *when* and *how* its collaborators are created — the **framework** does.
```
Traditional control:  your code calls libraries, creates its own objects (you're in control)
Inverted control:     the framework creates your objects & calls your code (it's in control)
                      ("Don't call us, we'll call you" — the Hollywood Principle)
```
| Without IoC | With IoC |
|-------------|----------|
| `new UserRepository()` everywhere | The container creates it once |
| Objects manage their own dependencies | Dependencies are injected |
| Tightly coupled (hard to test/swap) | Loosely coupled (easy to test/swap) |
> IoC is the *principle*; **Dependency Injection (DI)** is the most common *implementation* of it. The benefit: **loose coupling** — your classes depend on abstractions (interfaces), and the container provides concrete implementations (recall Phase 1.3 polymorphism/DI connection).

---

## 2. The IoC Container & ApplicationContext

The **IoC container** is the Spring component that creates, configures, wires, and manages **beans** (the objects it controls). In Spring, the container is the **`ApplicationContext`**.
```
ApplicationContext (the container):
  1. Reads configuration (annotations / Java config)
  2. Creates beans
  3. Injects their dependencies (wires them together)
  4. Manages their lifecycle (init -> use -> destroy)
```
### 2.1 ApplicationContext types
| Type | Use |
|------|-----|
| `AnnotationConfigApplicationContext` | Java/annotation-based config |
| `ClassPathXmlApplicationContext` | Legacy XML config |
| (Spring Boot) auto-created | You almost never instantiate it yourself |
> In **Spring Boot** (Phase 5.2), the `ApplicationContext` is created automatically at startup — you rarely touch it directly. `ApplicationContext` extends the older `BeanFactory` with more features (events, i18n, AOP integration).

---

## 3. Beans

A **bean** is an object **created and managed by the Spring container**. Your services, repositories, controllers, and configuration objects are all beans.
```
Bean = an object whose lifecycle (creation, wiring, destruction) Spring controls.
```
> Not every object is a bean — only the ones you register with Spring (via stereotype annotations or `@Bean` methods — §4). Entities/DTOs are usually *not* beans (they're created per request/row, not managed singletons).

---

## 4. Declaring Beans

Two main ways to tell Spring "manage this as a bean":

### 4.1 Stereotype annotations (component scanning)
Annotate a class so Spring auto-detects and registers it:
```java
@Component          // generic Spring-managed component
@Service            // a service-layer component (business logic)
@Repository         // a data-access component (also translates DB exceptions)
@Controller         // a web controller (MVC)
@RestController      // @Controller + @ResponseBody (REST APIs — Phase 5.3)
public class UserService { ... }
```
| Annotation | Semantic meaning | Special behavior |
|------------|------------------|------------------|
| **`@Component`** | Generic bean | The base; others are specializations |
| **`@Service`** | Business-logic layer | Semantic only (readability) |
| **`@Repository`** | Data-access layer | **Translates DB exceptions** to Spring's `DataAccessException` (recall Phase 1.5 — unchecked!) |
| **`@Controller` / `@RestController`** | Web layer | Handles HTTP requests (Phase 5.3) |
> These are all **`@Component` under the hood** — the specializations add semantic clarity (which layer) and sometimes behavior (`@Repository`'s exception translation). Use the most specific one for the role.

### 4.2 `@Bean` methods in a `@Configuration` class
For beans you can't annotate (third-party classes) or that need custom construction:
```java
@Configuration                              // a source of bean definitions
public class AppConfig {
    @Bean                                   // the return value becomes a bean
    public RestClient restClient() {
        return RestClient.builder().baseUrl("https://api.example.com").build();
    }
    @Bean
    public ObjectMapper objectMapper() {    // configure a third-party object as a bean
        return new ObjectMapper().findAndRegisterModules();
    }
}
```
> Use **stereotype annotations** for *your* classes, and **`@Bean` methods** for *third-party* objects or beans needing custom setup. The method name becomes the bean name by default.

### 4.3 Component scanning
`@ComponentScan` tells Spring **where to look** for stereotype-annotated classes. In Spring Boot, `@SpringBootApplication` (Phase 5.2) includes it, scanning the main class's package and sub-packages automatically.
```java
@ComponentScan("com.example")   // scan this package (and sub-packages) for components
```
> ⚠️ This is why your beans must be in (sub)packages of the main application class — otherwise they won't be scanned/registered. A common "why isn't my bean found?" cause.

---

## 5. Dependency Injection (DI)

**DI** = the container **supplies a bean's dependencies** rather than the bean creating them. Three types of injection:
| Type | How | Recommendation |
|------|-----|----------------|
| **Constructor injection** | Dependencies as constructor parameters | ✅ **Preferred** |
| **Setter injection** | Dependencies via setter methods | For optional dependencies |
| **Field injection** | Directly on fields (`@Autowired`) | ❌ **Avoid** |

### 5.1 Constructor injection (PREFERRED)
```java
@Service
public class UserService {
    private final UserRepository repository;          // final = immutable, guaranteed set
    private final EmailService emailService;

    // Spring injects these via the constructor (no @Autowired needed for a single constructor):
    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```
> ⭐ **Constructor injection is the recommended approach** (and Spring's default). Benefits:
> - **Immutability:** fields can be `final` (recall Phase 1.2) — set once, never null.
> - **Required dependencies are explicit:** the object can't be created without them (fail fast — Phase 1.5).
> - **Testability:** easy to pass mocks in tests (`new UserService(mockRepo, mockEmail)`) — no Spring needed (Phase 6).
> - **No `@Autowired` needed** for a single constructor (since Spring 4.3).
> This is "favor composition over inheritance" + "program to interfaces" in action (Phase 1.3) — the service depends on the `UserRepository` *interface*, injected by the container.

### 5.2 Setter injection
```java
@Service
public class UserService {
    private EmailService emailService;
    @Autowired
    public void setEmailService(EmailService emailService) {   // optional dependency
        this.emailService = emailService;
    }
}
```
Use for **optional** dependencies (can be re-set, can be null).

### 5.3 Field injection (AVOID)
```java
@Service
public class UserService {
    @Autowired private UserRepository repository;   // ❌ avoid
}
```
> ⚠️ **Avoid field injection.** It can't be `final` (mutable), **hides dependencies** (hard to see what a class needs), makes **testing harder** (you need reflection or Spring to inject), and risks `NullPointerException` if used before injection. It's concise but problematic. Use **constructor injection** instead.

### 5.4 DI comparison
| | Constructor | Setter | Field |
|---|-------------|--------|-------|
| `final`/immutable | ✅ | ❌ | ❌ |
| Required deps enforced | ✅ | ❌ | ❌ |
| Easy unit testing | ✅ | ✅ | ❌ |
| Hides dependencies | No | No | **Yes** |
| Best for | **Required deps (default)** | Optional deps | Avoid |

---

## 6. Resolving Ambiguity (@Qualifier, @Primary)

When **multiple beans** of the same type exist, Spring can't decide which to inject → error. Resolve it:
```java
public interface PaymentProcessor { ... }

@Service class StripeProcessor implements PaymentProcessor { ... }
@Service class PayPalProcessor implements PaymentProcessor { ... }   // two candidates!

// Option 1: @Primary marks the default
@Service @Primary
public class StripeProcessor implements PaymentProcessor { ... }

// Option 2: @Qualifier picks a specific one at the injection point
@Service
public class CheckoutService {
    public CheckoutService(@Qualifier("payPalProcessor") PaymentProcessor processor) { ... }
}
```
| Annotation | Effect |
|------------|--------|
| **`@Primary`** | Marks one bean as the default choice |
| **`@Qualifier("beanName")`** | Selects a specific bean by name |
> This leverages polymorphism (Phase 1.3): you inject the **interface**, and Spring picks the implementation — `@Qualifier`/`@Primary` disambiguate. You can also inject **all** implementations as a `List<PaymentProcessor>` (great for the Strategy pattern — Phase 14).

---

## 7. Other Wiring Annotations (awareness)
| Annotation | Purpose |
|------------|---------|
| `@Autowired` | Mark a dependency for injection (optional on constructors) |
| `@Lazy` | Create the bean only when first needed (not at startup) |
| `@DependsOn` | Force another bean to be created first |
| `@Order` | Order beans in an injected collection |
| `@Conditional` / `@ConditionalOn...` | Create a bean only if a condition holds (basis of auto-config — Phase 5.2) |
| `@Value` / `@ConfigurationProperties` | Inject config values (next note) |

---

## 8. Why DI Matters (The Payoff)

| Benefit | Explanation |
|---------|-------------|
| **Loose coupling** | Depend on interfaces; swap implementations without changing code (Phase 1.3) |
| **Testability** | Inject mocks in unit tests (Phase 6) — no real DB/network needed |
| **Single responsibility** | Classes focus on logic, not on creating dependencies |
| **Configuration flexibility** | Different beans per environment (profiles — next note) |
| **Reusability** | Components are decoupled and composable |
> DI is *the* reason Spring apps are testable and maintainable. A service that takes its `UserRepository` via the constructor can be unit-tested with a mock repository — no database required (Phase 6 mocking).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Field injection (`@Autowired` on fields) | Use constructor injection |
| `new`-ing dependencies inside a bean | Inject them (let the container wire) |
| Beans outside the scanned package | Place them under the main class's package |
| Multiple beans of one type → ambiguity error | Use `@Primary` or `@Qualifier` |
| Putting `@Bean` methods outside `@Configuration` | Use a `@Configuration` class |
| Making entities/DTOs beans | They aren't beans (created per request/row) |
| Confusing `@Component` specializations | Use `@Service`/`@Repository`/`@Controller` for the role |
| Circular dependencies | Refactor (often a design smell — Spring may fail to start) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Everything in Spring is built on IoC/DI:** controllers, services, repositories are all beans wired together.
- **Spring Boot (Phase 5.2)** auto-configures beans via `@Conditional` — the same container, automated.
- **Spring Data JPA (Phase 5.4)** repositories are beans (Spring generates the implementation, injects it).
- **Testing (Phase 6):** constructor injection makes mocking trivial (`@Mock`/`@InjectMocks`).
- **AOP (next note):** Spring wraps beans in proxies for `@Transactional`/`@Cacheable` — only works on managed beans.
- **Strategy pattern:** inject `List<Interface>` or use `@Qualifier` to pick implementations (Phase 14).
- **Profiles + `@Value`** (next note) configure beans per environment.

---

## 11. Quick Self-Check Questions

1. What is Inversion of Control, and how does DI implement it?
2. What is the IoC container / `ApplicationContext`?
3. What is a bean, and how do you declare one (two ways)?
4. What's special about `@Repository` vs `@Service` vs `@Component`?
5. What is component scanning, and why must beans be under the main package?
6. Name the three DI types. Why is constructor injection preferred?
7. Why avoid field injection (give three reasons)?
8. How do you resolve ambiguity when multiple beans of one type exist?
9. How does DI make code testable?

---

## 12. Key Terms Glossary

- **IoC (Inversion of Control):** the framework controls object creation/lifecycle.
- **DI (Dependency Injection):** supplying a bean's dependencies externally.
- **IoC container / `ApplicationContext`:** Spring's bean manager.
- **Bean:** a Spring-managed object.
- **Stereotype annotations (`@Component`/`@Service`/`@Repository`/`@Controller`):** declare beans by role.
- **`@Bean` / `@Configuration`:** declare beans via methods.
- **Component scanning / `@ComponentScan`:** auto-detect annotated beans.
- **Constructor / setter / field injection:** DI types (constructor preferred).
- **`@Autowired`:** marks an injection point.
- **`@Primary` / `@Qualifier`:** resolve bean ambiguity.
- **Loose coupling:** depending on abstractions, not concretions.

---

*This is the first note of **Section 5.1 — Spring Core Fundamentals**.*
*Next topic: **Bean Scopes, Lifecycle, Configuration & Profiles**.*
