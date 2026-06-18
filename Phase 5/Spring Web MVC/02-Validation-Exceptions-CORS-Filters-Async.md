# Spring Web MVC: Validation, Exception Handling, CORS, Filters & Async

> **Phase 5 — Spring Framework & Spring Boot → 5.3 Spring Web MVC**
> Goal: Master the rest of MVC — Bean Validation, global exception handling (ProblemDetail), CORS, interceptors & filters, async controllers, and file upload/download.

---

## 0. The Big Picture

Beyond controllers (previous note), a robust API needs: **validated input**, **consistent error responses**, **CORS** for browser access, **filters/interceptors** for cross-cutting request handling, and support for **async** and **file** operations. This note completes the MVC toolkit.

---

## 1. Bean Validation (JSR 380)

**Bean Validation** declaratively validates request data using annotations — so you reject bad input *before* it reaches your business logic (fail fast — Phase 1.5). Add `starter-validation` (Phase 5.2).

### 1.1 Validation annotations on the DTO
```java
public record CreateUserRequest(
    @NotBlank String name,                               // not null & not empty/whitespace
    @Email @NotBlank String email,                       // valid email format
    @Min(0) @Max(150) int age,                           // numeric range
    @Size(min = 8, max = 100) String password,           // length
    @Pattern(regexp = "\\+?[0-9]{10,15}") String phone,  // regex (recall Phase 1.4)
    @NotNull @Past LocalDate birthDate                   // date in the past
) {}
```
| Annotation | Validates |
|------------|-----------|
| `@NotNull` / `@NotBlank` / `@NotEmpty` | not null / not blank string / non-empty collection |
| `@Size(min, max)` | string/collection length |
| `@Min` / `@Max` / `@Positive` | numeric bounds |
| `@Email` | email format |
| `@Pattern(regexp)` | regex match (Phase 1.4) |
| `@Past` / `@Future` | date constraints |

### 1.2 Triggering validation with @Valid
```java
@PostMapping
public UserDto create(@RequestBody @Valid CreateUserRequest req) {   // @Valid triggers it
    return userService.create(req);   // only runs if validation passes
}
```
> ⚠️ **`@Valid` triggers validation** — if it fails, Spring throws `MethodArgumentNotValidException` **before** your method body runs (fail fast). Without `@Valid`, the annotations are ignored! Handle the exception globally (§2) to return a clean `400 Bad Request`.

### 1.3 @Validated, groups & custom validators
- **`@Validated`** (class-level) enables validation on `@RequestParam`/`@PathVariable` and validation **groups** (validate different rules in different contexts).
- **Custom validators:** implement `ConstraintValidator` for domain rules:
```java
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target(FIELD) @Retention(RUNTIME)
public @interface UniqueEmail { String message() default "Email already exists"; ... }
// + a ConstraintValidator<UniqueEmail, String> implementation (recall Phase 1.13 custom annotations)
```
> Custom validators = custom annotation (Phase 1.13) + a `ConstraintValidator` — for business rules like "email must be unique." `BindingResult` (a method param after `@Valid`) lets you inspect errors manually instead of throwing.

---

## 2. Exception Handling (Consistent Error Responses)

A robust API returns **consistent, structured error responses** — not raw stack traces. Spring offers controller-level and global handling.

### 2.1 @ExceptionHandler (controller-level)
```java
@RestController
public class UserController {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handle(UserNotFoundException e) {
        return ResponseEntity.status(404).body(e.getMessage());   // local to this controller
    }
}
```

### 2.2 @RestControllerAdvice (global — preferred)
**`@RestControllerAdvice`** centralizes exception handling across **all** controllers — the standard approach (recall AOP cross-cutting, Phase 5.1):
```java
@RestControllerAdvice                          // global exception handler for all controllers
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException e) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
        pd.setTitle("Resource Not Found");
        return pd;   // -> 404 with a structured body
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)   // validation failures (§1)
    public ProblemDetail handleValidation(MethodArgumentNotValidException e) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);   // 400
        pd.setTitle("Validation Failed");
        pd.setProperty("errors", e.getBindingResult().getFieldErrors().stream()
            .collect(toMap(FieldError::getField, FieldError::getDefaultMessage)));
        return pd;
    }

    @ExceptionHandler(Exception.class)         // catch-all (last resort) -> 500
    public ProblemDetail handleGeneric(Exception e) {
        log.error("Unexpected error", e);      // log it (don't leak details to the client — Phase 1.5/15)
        return ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```
> ⭐ **`@RestControllerAdvice` is the standard for API error handling** — define error responses once, globally (embodying "log OR rethrow" — Phase 1.5). It maps exceptions → HTTP status codes (Phase 0.2). Recall **unchecked exceptions** (Phase 1.5) are the idiomatic way to signal API errors here.

### 2.3 ProblemDetail (RFC 7807) — the standard error format
**`ProblemDetail`** (Spring 6+) is the standardized error-response format (RFC 7807) — a consistent JSON shape for errors:
```json
{
  "type": "about:blank",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User 42 not found",
  "instance": "/api/users/42"
}
```
> Using **ProblemDetail** gives clients a **predictable error structure** (type, title, status, detail) instead of ad-hoc formats — a REST best practice (Phase 7). ⚠️ **Never leak stack traces or internal details** to clients (security — Phase 15); log them server-side, return a clean message.

---

## 3. CORS (Cross-Origin Resource Sharing)

**CORS** (recall Phase 0.2 in depth) controls whether browser JavaScript from a **different origin** (e.g., a React frontend at `localhost:3000`) can call your API (at `localhost:8080`). You configure the server to *allow* specific origins.

### 3.1 Per-controller
```java
@CrossOrigin(origins = "https://app.example.com")
@RestController
public class UserController { ... }
```

### 3.2 Global CORS config (preferred)
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")     // which origins (NOT "*" with credentials)
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```
> ⚠️ Recall (Phase 0.2): **CORS is browser-enforced, not API security** — it controls what *browser JS* may read, not what reaches your server. Don't use `allowedOrigins("*")` with `allowCredentials(true)` (not allowed; insecure). Configure CORS so your frontend can call the API, but rely on **authentication** (Phase 5.6) for actual security.

---

## 4. Interceptors & Filters (Cross-Cutting Request Handling)

Both intercept requests for cross-cutting concerns (logging, auth, timing), but at different levels:
| | **Filter** | **Interceptor** |
|---|------------|-----------------|
| Level | Servlet (before Spring MVC) | Spring MVC (after routing decided) |
| Scope | All requests (even non-MVC) | MVC handler requests |
| Access to | Raw request/response | Handler method, ModelAndView |
| Use for | Auth (JWT — Phase 5.6), CORS, logging, compression | MVC-specific pre/post handling |

### 4.1 Filter (OncePerRequestFilter)
```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {   // runs once per request
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        long start = System.currentTimeMillis();
        chain.doFilter(req, res);                       // continue the chain (call the next filter/servlet)
        log.info("{} {} -> {} ({}ms)", req.getMethod(), req.getRequestURI(),
                 res.getStatus(), System.currentTimeMillis() - start);
    }
}
```
> **Filters** are where **JWT authentication** lives (Phase 5.6 — a custom filter validates the token before the request reaches controllers). `OncePerRequestFilter` guarantees one execution per request. Use `@Order` to control filter order.

### 4.2 Interceptor (HandlerInterceptor)
```java
@Component
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        // runs BEFORE the controller; return false to block the request
        return true;
    }
    // also: postHandle (after controller, before view), afterCompletion (after everything)
}
```
| Interceptor hook | When |
|------------------|------|
| `preHandle` | Before the controller (can block) |
| `postHandle` | After the controller, before the response is written |
| `afterCompletion` | After the complete request (cleanup) |

---

## 5. Async Controllers

For long-running operations, return **asynchronously** so the servlet thread isn't blocked while waiting (recall blocking I/O / thread-per-request, Phase 0.1/1.10):
```java
@GetMapping("/slow")
public CompletableFuture<Result> async() {            // returns a CompletableFuture (Phase 1.10)
    return CompletableFuture.supplyAsync(() -> slowOperation());
}

@GetMapping("/stream")
public SseEmitter stream() { ... }                    // Server-Sent Events (push updates)
```
| Return type | Use |
|-------------|-----|
| `Callable<T>` | Run on a separate thread, free the servlet thread |
| `DeferredResult<T>` | Complete the response later (from another thread) |
| `CompletableFuture<T>` | Async composition (Phase 1.10) |
| `SseEmitter` / `StreamingResponseBody` | Server-Sent Events / streaming |
> Async controllers free the request thread during waits → better throughput for I/O-bound work. ⭐ **Virtual threads** (`spring.threads.virtual.enabled=true` — Phase 1.10/5.2) achieve similar scalability with simple blocking code, and are often the easier modern choice.

---

## 6. File Upload & Download

### 6.1 Upload (MultipartFile)
```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) throws IOException {   // multipart/form-data
    String name = file.getOriginalFilename();
    long size = file.getSize();
    byte[] bytes = file.getBytes();           // or file.getInputStream() (recall Phase 1.9 I/O)
    storageService.store(file);
    return "uploaded: " + name;
}
```
> Configure limits in `application.yml` (`spring.servlet.multipart.max-file-size`). ⚠️ **Validate uploads** (size, type) and never trust the filename (security — Phase 15).

### 6.2 Download
```java
@GetMapping("/download/{id}")
public ResponseEntity<Resource> download(@PathVariable Long id) {
    Resource file = storageService.load(id);
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + file.getFilename() + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(file);
}
// For large files, use StreamingResponseBody to stream without loading all into memory (Phase 1.9)
```

---

## 7. WebMvcConfigurer (Customizing MVC)

`WebMvcConfigurer` is the hook to customize MVC — register interceptors, CORS, message converters, argument resolvers, formatters:
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override public void addInterceptors(InterceptorRegistry r) { r.addInterceptor(new AuthInterceptor()); }
    @Override public void addCorsMappings(CorsRegistry r) { ... }            // §3
    // also: addFormatters, configureMessageConverters, addArgumentResolvers
}
```

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Forgetting `@Valid` (validation ignored) | Add `@Valid` to trigger validation |
| Returning raw exceptions/stack traces | Use `@RestControllerAdvice` + ProblemDetail; log internally |
| Inconsistent error formats | Standardize on ProblemDetail (RFC 7807) |
| `allowedOrigins("*")` + credentials | Not allowed; scope origins explicitly |
| Thinking CORS secures the API | It's browser-side; use auth (Phase 5.6) |
| Validation logic in the service only | Validate at the boundary (`@Valid`) too |
| Loading large files fully into memory | Stream them (`StreamingResponseBody`, Phase 1.9) |
| Blocking I/O on the servlet thread | Use async / virtual threads |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Validation + exception handling** = robust APIs (Phase 7 API design).
- **ProblemDetail (RFC 7807)** is the standard error format (Phase 7).
- **JWT auth filter** (Phase 5.6) is a `OncePerRequestFilter`.
- **CORS** lets your frontend (React/Angular) call the API (recall Phase 0.2).
- **Custom validators** = custom annotations + `ConstraintValidator` (Phase 1.13).
- **Rate limiting / request logging** (Project 4) often via filters/interceptors.
- **File upload/download** (Projects 4, 5) — images, attachments (S3/local).
- **Async/virtual threads** for scalability (Phase 1.10, 13, 16).

---

## 10. Quick Self-Check Questions

1. How do you validate request data, and what does `@Valid` do?
2. How do you create a custom validator?
3. What's the difference between `@ExceptionHandler` and `@RestControllerAdvice`?
4. What is ProblemDetail, and why use it?
5. Why is CORS needed, and what's the credentials+`*` pitfall? Does CORS secure your API?
6. What's the difference between a filter and an interceptor? Where does JWT auth go?
7. When and how do you handle requests asynchronously?
8. How do you handle file upload and download?

---

## 11. Key Terms Glossary

- **Bean Validation (JSR 380):** declarative input validation (`@NotNull`, etc.).
- **`@Valid` / `@Validated`:** trigger validation / enable groups & param validation.
- **`ConstraintValidator`:** custom validation logic.
- **`@ExceptionHandler` / `@RestControllerAdvice`:** controller / global exception handling.
- **`ProblemDetail` (RFC 7807):** standardized error response format.
- **CORS:** browser-enforced cross-origin access control.
- **Filter (`OncePerRequestFilter`):** servlet-level request interception.
- **Interceptor (`HandlerInterceptor`):** MVC-level pre/post handling.
- **Async controller (`Callable`/`DeferredResult`/`CompletableFuture`):** non-blocking responses.
- **`MultipartFile`:** uploaded file.
- **`StreamingResponseBody` / `SseEmitter`:** streaming / server-sent events.
- **`WebMvcConfigurer`:** MVC customization hook.

---

*Previous topic: **Controllers, Requests & Responses**.*
*This completes **Section 5.3 — Spring Web MVC**.*
*Next section in roadmap: **5.4 Spring Data JPA**.*
