# Spring Core: Aspect-Oriented Programming (AOP)

> **Phase 5 — Spring Framework & Spring Boot → 5.1 Spring Core Fundamentals**
> Goal: Master AOP — aspects, join points, advice, pointcuts, weaving; the advice types; pointcut expressions; how Spring's proxy-based AOP works (JDK vs CGLIB); and the self-invocation problem.

---

## 0. The Big Picture

**Aspect-Oriented Programming (AOP)** lets you separate **cross-cutting concerns** — logic that applies across many methods/classes (logging, transactions, security, caching) — from your business code. Instead of scattering the same code everywhere, you define it **once** in an **aspect** and apply it declaratively.

```
Cross-cutting concern (e.g., logging) is needed in MANY methods.
Without AOP: copy-paste logging into every method (scattered, duplicated).
With AOP:    define a logging ASPECT once -> applied automatically where you specify.
```

> AOP is *how* Spring implements `@Transactional` (Phase 5.5), `@Cacheable` (Phase 5.7), and method security (`@PreAuthorize`, Phase 5.6) — they all work via AOP behind the scenes. Understanding AOP demystifies these annotations.

---

## 1. The Problem: Cross-Cutting Concerns

Some concerns cut **across** many parts of your code — they don't belong to one class:
| Cross-cutting concern | Appears in... |
|-----------------------|---------------|
| Logging | Every service method |
| Transactions | Every data-modifying method (Phase 5.5) |
| Security checks | Many controller/service methods (Phase 5.6) |
| Caching | Many read methods (Phase 5.7) |
| Performance monitoring | Many methods (Phase 9) |
| Exception handling/auditing | Throughout |
```java
// WITHOUT AOP — the same boilerplate scattered everywhere (mixed with business logic):
public Order createOrder(...) {
    log.info("entering createOrder");          // logging concern
    tx.begin();                                 // transaction concern
    checkPermission();                          // security concern
    try {
        // ... ACTUAL business logic (buried) ...
        tx.commit();
    } catch (Exception e) { tx.rollback(); throw e; }
    log.info("exiting createOrder");
}
```
> This "tangling" (concerns mixed into business logic) and "scattering" (the same concern repeated everywhere) is what AOP eliminates — keeping business methods focused on business logic.

---

## 2. AOP Terminology (Core Concepts)

| Term | Meaning |
|------|---------|
| **Aspect** | A module bundling a cross-cutting concern (e.g., a `LoggingAspect`) |
| **Join point** | A point in program execution where an aspect can apply (in Spring: a **method execution**) |
| **Advice** | The action taken by an aspect at a join point (the code to run — §3) |
| **Pointcut** | An expression selecting *which* join points the advice applies to (§4) |
| **Weaving** | Linking aspects to the target code (in Spring: at runtime, via proxies — §5) |
| **Target** | The bean being advised |
```
Aspect = Advice (what to do) + Pointcut (where to do it)
```
> Mnemonic: a **pointcut** says *where* (which methods), **advice** says *what* (the action), an **aspect** bundles them, and **weaving** wires it in. In Spring, the join point is always a **method execution** on a Spring bean.

---

## 3. Advice Types (When the Advice Runs)

**Advice** is the code that runs; the type determines *when* relative to the target method:
| Advice | Runs... |
|--------|---------|
| **`@Before`** | Before the method executes |
| **`@After`** | After the method (finally — runs whether it succeeds or throws) |
| **`@AfterReturning`** | After the method **returns successfully** (can access the return value) |
| **`@AfterThrowing`** | After the method **throws an exception** |
| **`@Around`** | **Wraps** the method — runs before AND after; can modify args/result/skip the call (most powerful) |

### 3.1 Example aspect
```java
@Aspect                                  // marks this as an aspect
@Component                               // make it a bean (so Spring manages it)
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")   // pointcut (§4)
    public void logBefore(JoinPoint jp) {
        log.info("Calling: {}", jp.getSignature().getName());
    }

    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))",
                    returning = "result")
    public void logResult(JoinPoint jp, Object result) {
        log.info("{} returned {}", jp.getSignature().getName(), result);
    }

    @Around("execution(* com.example.service.*.*(..))")   // the most powerful
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();        // CALL the actual method (you control this!)
        log.info("{} took {}ms", pjp.getSignature().getName(),
                 System.currentTimeMillis() - start);
        return result;
    }
}
```
> **`@Around`** is the most powerful (and the one used by `@Transactional`, `@Cacheable`): it **wraps** the method via `ProceedingJoinPoint.proceed()` — you decide whether/when to call the real method, can modify arguments and the return value, and handle exceptions. (E.g., `@Transactional` is essentially "begin tx → proceed() → commit/rollback" as `@Around` advice.)

---

## 4. Pointcut Expressions (Where Advice Applies)

A **pointcut** selects which methods the advice targets, using an expression language:
```java
execution(* com.example.service.*.*(..))
   |       |              |       |   |
 designator returntype  package class.method (args)
```
| Pointcut designator | Matches |
|---------------------|---------|
| **`execution(...)`** | Method executions matching a signature pattern (most common) |
| **`@annotation(...)`** | Methods annotated with a specific annotation (e.g., a custom `@Loggable`) |
| **`within(...)`** | Methods within certain classes/packages |
| **`bean(...)`** | Methods on a specific bean by name |

### 4.1 Examples
```java
"execution(* com.example.service..*.*(..))"        // all methods in service package (& sub)
"execution(public * *(..))"                         // all public methods
"execution(* *..*Service.*(..))"                    // any method in a *Service class
"@annotation(com.example.Loggable)"                 // methods annotated @Loggable
"within(com.example.service.*)"                     // methods within the service package
```
### 4.2 Custom annotation + AOP (a powerful pattern)
```java
// Define a marker annotation (recall Phase 1.13):
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.METHOD)
public @interface Loggable {}

// Aspect targets methods annotated with it:
@Around("@annotation(Loggable)")                    // applies only where @Loggable is present
public Object log(ProceedingJoinPoint pjp) throws Throwable { ... }

// Usage — opt in per method:
@Loggable
public void importantOperation() { ... }
```
> Combining a **custom annotation (Phase 1.13)** with an `@annotation` pointcut is the idiomatic way to make cross-cutting behavior **opt-in** (e.g., `@RateLimited`, `@Audited`, `@Loggable`) — exactly how `@Transactional`/`@Cacheable` are wired (annotation + AOP + reflection — Phase 1.13/1.14).

---

## 5. How Spring AOP Works: Proxies

> **Spring AOP is proxy-based.** When a bean has advice applied, Spring wraps it in a **proxy** object. Callers get the **proxy**, not the real bean — the proxy runs the advice (before/around/etc.) and then delegates to the real method.
```
Caller -> [ PROXY ] -> (runs advice: logging/tx/security) -> [ real bean method ]
The container injects the PROXY wherever the bean is needed.
```

### 5.1 JDK dynamic proxy vs CGLIB
Spring creates the proxy one of two ways (recall reflection/dynamic proxies, Phase 1.14, and bytecode manipulation, Phase 1.11):
| | **JDK dynamic proxy** | **CGLIB** |
|---|----------------------|-----------|
| How | Implements the bean's **interfaces** (`java.lang.reflect.Proxy`) | Subclasses the bean's **class** (bytecode generation) |
| Requires | The bean to implement an interface | No interface needed |
| Spring default | If the bean has an interface | If no interface (or forced) |
| Limitation | Only interface methods are advised | Can't proxy `final` classes/methods |
> Spring Boot uses **CGLIB by default** (proxies the class, so it works without interfaces). Either way, the proxy intercepts calls to apply advice. (CGLIB uses bytecode manipulation — recall Phase 1.11; JDK proxies use reflection — Phase 1.14.)

---

## 6. The Self-Invocation Problem (CRITICAL Gotcha)

> ⚠️ **Because Spring AOP works via proxies, advice only applies to calls that go *through* the proxy — i.e., calls from *other* beans. A method calling another method on `this` (self-invocation) bypasses the proxy, so the advice does NOT run.**
```java
@Service
public class OrderService {

    @Transactional                         // advice via the PROXY
    public void outer() {
        inner();                           // ❌ calls inner() on `this` — bypasses the proxy!
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void inner() {                  // its @Transactional is IGNORED when called from outer()!
        // ... runs in outer()'s transaction, NOT a new one
    }
}
```
> This is the **#1 Spring AOP gotcha** and a frequent bug: a `@Transactional` (or `@Cacheable`, `@Async`) annotation on a method called from *within the same class* **silently doesn't work** — the internal call doesn't pass through the proxy. (Recall Phase 1.3 — this was previewed in the "self-invocation problem" of inheritance/methods.)

### 6.1 Fixes for self-invocation
| Fix | How |
|-----|-----|
| **Move the method to another bean** | The call then goes through that bean's proxy (cleanest) |
| **Inject self** | Inject the bean into itself and call via the injected proxy |
| **`AopContext.currentProxy()`** | Get the proxy explicitly (requires `exposeProxy=true`) |
> ⚠️ Also: AOP-based annotations work **only on Spring-managed beans** and **public methods** (CGLIB can't advise `private`/`final`). `@Transactional` on a private method, or on a non-bean, silently does nothing.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Self-invocation (internal call) bypassing advice | Call via another bean / injected self |
| `@Transactional`/`@Cacheable` on a private method | They only work on public bean methods (proxy) |
| AOP annotations on non-Spring objects | Only beans are proxied |
| `final` class/method blocking CGLIB | Don't make advised classes/methods `final` |
| Overly broad pointcuts | Scope pointcuts precisely (performance, surprises) |
| Heavy logic in `@Around` without `proceed()` | Remember to call `proceed()` or the method never runs |
| Assuming advice runs on every call | Only on proxy-routed calls |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **`@Transactional` (Phase 5.5)** is `@Around` advice — and the **self-invocation problem** explains why nested `@Transactional` sometimes "doesn't work."
- **`@Cacheable`/`@CacheEvict` (Phase 5.7)** are AOP-based — same proxy/self-invocation rules.
- **Method security `@PreAuthorize`/`@PostAuthorize` (Phase 5.6)** uses AOP.
- **`@Async` (Phase 1.10/5)** uses AOP (and suffers the same self-invocation gotcha).
- **Custom cross-cutting annotations** (`@RateLimited`, `@Audited`, `@Loggable`) = custom annotation (Phase 1.13) + AOP + reflection (Phase 1.14).
- **Distributed scheduling (`@Scheduled`)**, retry, circuit breakers (Resilience4j, Phase 12.3) — all annotation + AOP.
- Understanding proxies explains many "why didn't my annotation work?" mysteries.

---

## 9. Quick Self-Check Questions

1. What problem does AOP solve (cross-cutting concerns)?
2. Define aspect, join point, advice, pointcut, and weaving.
3. Name the advice types and when each runs. Which is most powerful and why?
4. What does a pointcut expression like `execution(* com.example.service.*.*(..))` mean?
5. How do you make cross-cutting behavior opt-in with a custom annotation?
6. How does Spring AOP work (proxies)? JDK proxy vs CGLIB?
7. What is the self-invocation problem, and why does it happen?
8. How do you fix self-invocation, and what other limitations does proxy-based AOP have?

---

## 10. Key Terms Glossary

- **AOP:** modularizing cross-cutting concerns.
- **Cross-cutting concern:** logic spanning many methods (logging, tx, security).
- **Aspect:** a module of cross-cutting code (advice + pointcut).
- **Join point:** a point where advice can apply (method execution in Spring).
- **Advice:** the action run (`@Before`/`@After`/`@AfterReturning`/`@AfterThrowing`/`@Around`).
- **Pointcut:** expression selecting join points (`execution(...)`, `@annotation(...)`).
- **Weaving:** linking aspects to code (runtime proxies in Spring).
- **Proxy:** the wrapper object that runs advice.
- **JDK dynamic proxy / CGLIB:** interface-based / subclass-based proxying.
- **Self-invocation problem:** internal calls bypass the proxy (advice skipped).
- **`ProceedingJoinPoint.proceed()`:** invoke the real method in `@Around`.

---

*Previous topic: **Bean Scopes, Lifecycle, Configuration & Profiles**.*
*This completes **Section 5.1 — Spring Core Fundamentals**.*
*Next section in roadmap: **5.2 Spring Boot**.*
