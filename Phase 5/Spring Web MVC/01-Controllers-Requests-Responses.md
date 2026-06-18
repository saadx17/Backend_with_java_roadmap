# Spring Web MVC: Controllers, Requests & Responses

> **Phase 5 — Spring Framework & Spring Boot → 5.3 Spring Web MVC**
> Goal: Master REST API development with Spring MVC — the DispatcherServlet flow, controller annotations, request handling, response handling, and Jackson JSON.

---

## 0. The Big Picture

**Spring Web MVC** is the framework for building **web applications and REST APIs** in Spring. It maps incoming **HTTP requests** (recall Phase 0.2 HTTP) to your **controller methods**, handles serialization (JSON ↔ Java objects), and produces **HTTP responses**.

```
HTTP request -> DispatcherServlet -> your @RestController method -> HTTP response (JSON)
```

> This is where everything connects: HTTP (Phase 0.2), beans/DI (Phase 5.1), and Spring Boot's auto-configured `starter-web` (Phase 5.2). REST API development is the core of most backend work.

---

## 1. The DispatcherServlet (Request Flow)

The **`DispatcherServlet`** is the **front controller** — a single servlet that receives **all** HTTP requests and routes them to the right handler.
```
1. HTTP request arrives -> DispatcherServlet (the front controller)
2. HandlerMapping: which controller method handles this URL+method?
3. The controller method runs (your code)
4. The return value is converted to the response body (Jackson -> JSON)
5. HTTP response sent back to the client
```
> Spring Boot auto-configures the `DispatcherServlet` (Phase 5.2) — you never set it up manually. You just write controllers; it routes requests to them. (It's the **Front Controller pattern** — Phase 14.)

---

## 2. Controllers

A **controller** is a bean (recall Phase 5.1) that handles HTTP requests. For REST APIs, use **`@RestController`**:
```java
@RestController                          // = @Controller + @ResponseBody (returns data, not views)
@RequestMapping("/api/users")            // base path for all methods in this class
public class UserController {
    private final UserService userService;
    public UserController(UserService userService) {   // constructor injection (Phase 5.1)
        this.userService = userService;
    }
    // ... handler methods ...
}
```
| Annotation | Use |
|------------|-----|
| **`@Controller`** | Returns view names (server-rendered HTML — Thymeleaf) |
| **`@RestController`** | **Returns data** (JSON) — for REST APIs (`@Controller` + `@ResponseBody`) |
| **`@RequestMapping`** | Maps a base URL path (class or method level) |
> For REST APIs, **always use `@RestController`** — its methods' return values are serialized to the response body (JSON) automatically, rather than resolved as view names.

---

## 3. Mapping HTTP Methods

Each HTTP method (recall Phase 0.2 — GET/POST/PUT/PATCH/DELETE) maps to a controller-method annotation:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping                          // GET /api/users -> list
    public List<UserDto> getAll() { ... }

    @GetMapping("/{id}")                 // GET /api/users/42 -> one
    public UserDto getById(@PathVariable Long id) { ... }

    @PostMapping                         // POST /api/users -> create
    public UserDto create(@RequestBody @Valid CreateUserRequest req) { ... }

    @PutMapping("/{id}")                 // PUT /api/users/42 -> replace
    public UserDto update(@PathVariable Long id, @RequestBody @Valid UpdateUserRequest req) { ... }

    @PatchMapping("/{id}")               // PATCH /api/users/42 -> partial update
    public UserDto patch(@PathVariable Long id, @RequestBody Map<String, Object> updates) { ... }

    @DeleteMapping("/{id}")              // DELETE /api/users/42 -> remove
    public void delete(@PathVariable Long id) { ... }
}
```
| Annotation | HTTP method | Semantics (Phase 0.2) |
|------------|-------------|------------------------|
| `@GetMapping` | GET | Read (safe, idempotent) |
| `@PostMapping` | POST | Create (not idempotent) |
| `@PutMapping` | PUT | Replace (idempotent) |
| `@PatchMapping` | PATCH | Partial update |
| `@DeleteMapping` | DELETE | Remove (idempotent) |
> These map directly to the HTTP semantics from Phase 0.2 — use the right verb for the operation (REST design — Phase 7).

---

## 4. Request Handling (Getting Data In)

Spring binds parts of the HTTP request to method parameters via annotations:
| Annotation | Binds from | Example |
|------------|------------|---------|
| **`@PathVariable`** | URL path segment | `/users/{id}` → `id` |
| **`@RequestParam`** | Query string | `?page=2&size=10` → `page`, `size` |
| **`@RequestBody`** | Request body (JSON) | POST/PUT payload → a Java object |
| **`@RequestHeader`** | An HTTP header | `Authorization` header |
| **`@CookieValue`** | A cookie | session cookie |
```java
@GetMapping("/{id}")
public UserDto get(@PathVariable Long id) { ... }              // /api/users/42

@GetMapping
public Page<UserDto> list(
        @RequestParam(defaultValue = "0") int page,            // ?page=0
        @RequestParam(defaultValue = "20") int size,           // ?size=20
        @RequestParam(required = false) String status) { ... } // ?status=ACTIVE (optional)

@PostMapping
public UserDto create(@RequestBody CreateUserRequest req) { ... }  // JSON body -> object

@GetMapping("/me")
public UserDto me(@RequestHeader("Authorization") String token) { ... }
```
> **`@RequestBody`** triggers JSON deserialization (Jackson, §6) — the request body becomes a Java object. **`@PathVariable`** for resource identity (`/users/{id}`), **`@RequestParam`** for filters/options/pagination (recall query strings, Phase 0.2). Spring auto-converts types (String → Long, etc.).

---

## 5. Response Handling (Getting Data Out)

### 5.1 Returning objects (auto-serialized to JSON)
With `@RestController`, the return value is serialized to the response body:
```java
@GetMapping("/{id}")
public UserDto getById(@PathVariable Long id) {
    return userService.findById(id);     // -> serialized to JSON, 200 OK
}
```

### 5.2 ResponseEntity (full control over the response)
**`ResponseEntity<T>`** lets you set the **status code, headers, and body** explicitly (recall status codes, Phase 0.2):
```java
@PostMapping
public ResponseEntity<UserDto> create(@RequestBody @Valid CreateUserRequest req) {
    UserDto created = userService.create(req);
    return ResponseEntity
        .status(HttpStatus.CREATED)                          // 201 Created (Phase 0.2)
        .header("Location", "/api/users/" + created.id())    // Location header
        .body(created);
}

@GetMapping("/{id}")
public ResponseEntity<UserDto> get(@PathVariable Long id) {
    return userService.findOptional(id)                      // recall Optional, Phase 1.8
        .map(ResponseEntity::ok)                             // 200 with body
        .orElse(ResponseEntity.notFound().build());          // 404 (Phase 0.2)
}
```

### 5.3 @ResponseStatus (set the status declaratively)
```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)      // returns 201 instead of the default 200
public UserDto create(@RequestBody CreateUserRequest req) { ... }
```
> ⚠️ **Use the correct HTTP status codes** (Phase 0.2): `201 Created` for POST (with a `Location` header), `200 OK` for GET/PUT, `204 No Content` for DELETE, `404` when not found. Returning `200` for everything is poor API design (Phase 7). Use **`ResponseEntity`** when you need control; return the object directly for simple `200 OK` cases.

### 5.4 Content negotiation
Spring picks the response format based on the `Accept` header (recall Phase 0.2). For REST APIs it's almost always **JSON** (`application/json`) via Jackson — auto-configured by `starter-web` (Phase 5.2).

---

## 6. Jackson JSON Processing

**Jackson** is the library that converts Java objects ↔ JSON (auto-configured by Spring Boot — recall Phase 1.9: prefer JSON over Java serialization). Customize mapping with annotations:
| Annotation | Effect |
|------------|--------|
| **`@JsonProperty("name")`** | Map a field to a different JSON key |
| **`@JsonIgnore`** | Exclude a field from JSON (recall `transient`, Phase 1.9) |
| **`@JsonInclude(NON_NULL)`** | Omit null fields from output |
| **`@JsonFormat`** | Format dates/numbers (e.g., date pattern) |
| **`@JsonSerialize`/`@JsonDeserialize`** | Custom (de)serializers |
| **`@JsonTypeInfo`/`@JsonSubTypes`** | Polymorphic types (recall sealed types, Phase 1.3) |
```java
public record UserDto(
    Long id,
    @JsonProperty("full_name") String name,         // JSON key: "full_name"
    @JsonIgnore String password,                     // never serialize the password! (Phase 15)
    @JsonFormat(pattern = "yyyy-MM-dd") LocalDate birthDate
) {}
```
### 6.1 ObjectMapper configuration
```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
    return builder -> builder
        .serializationInclusion(JsonInclude.Include.NON_NULL)   // omit nulls globally
        .modulesToInstall(new JavaTimeModule());                // Java 8 date/time support
}
```
> ⚠️ **`@JsonIgnore` sensitive fields** (passwords, tokens) so they never leak into responses (Phase 15) — or better, use **DTOs** (Phase 7) that simply don't include them. **Records work great as DTOs** (Phase 1.3/1.9) — Jackson serializes/deserializes them cleanly. Spring Boot configures the `JavaTimeModule` so `LocalDate`/`Instant` serialize properly.

---

## 7. A Complete Controller Example
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    public UserController(UserService userService) { this.userService = userService; }

    @GetMapping
    public Page<UserDto> list(@RequestParam(defaultValue = "0") int page,
                              @RequestParam(defaultValue = "20") int size) {
        return userService.findAll(PageRequest.of(page, size));   // pagination (Phase 5.4)
    }

    @GetMapping("/{id}")
    public UserDto get(@PathVariable Long id) {
        return userService.findById(id);   // throws if not found -> 404 via @ControllerAdvice (next note)
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto create(@RequestBody @Valid CreateUserRequest req) {   // @Valid -> validation (next note)
        return userService.create(req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)   // 204
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using `@Controller` for a REST API | Use `@RestController` |
| Returning `200` for everything | Use correct status codes (201/204/404...) |
| Exposing entities directly in responses | Use DTOs (Phase 7) — decouple & hide fields |
| Serializing passwords/secrets | `@JsonIgnore` or omit from DTOs (Phase 15) |
| Confusing `@PathVariable` and `@RequestParam` | Path = identity; param = query options |
| Forgetting `@RequestBody` for JSON input | Required to deserialize the body |
| Field injection in controllers | Constructor injection (Phase 5.1) |
| Business logic in controllers | Keep controllers thin; delegate to services |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **The entry point of every request** — controllers connect HTTP (Phase 0.2) to your service layer (Phase 5.1).
- **DTOs + Jackson** (Phase 7) decouple your API from entities and control what's exposed (security — Phase 15).
- **Validation + exception handling** (next note) make robust APIs.
- **Pagination** (`Page<T>` — Phase 5.4) for list endpoints.
- **REST API design** (Phase 7): proper verbs, status codes, resource naming.
- **Security** (Phase 5.6) protects these endpoints; **CORS** (next note) allows frontend access.
- **Keep controllers thin** — they orchestrate; services hold business logic (Phase 14 layered architecture).

---

## 10. Quick Self-Check Questions

1. What is the DispatcherServlet, and what's the request flow?
2. What's the difference between `@Controller` and `@RestController`?
3. Map each HTTP verb to its Spring annotation.
4. What do `@PathVariable`, `@RequestParam`, and `@RequestBody` bind?
5. When use `ResponseEntity` vs returning an object directly?
6. Why and how do you set the correct HTTP status code?
7. What is Jackson, and how do you customize JSON mapping / hide a field?
8. Why use DTOs instead of returning entities?

---

## 11. Key Terms Glossary

- **Spring MVC:** the web/REST framework in Spring.
- **DispatcherServlet:** front controller routing all requests.
- **`@RestController` / `@Controller`:** REST data / view controllers.
- **`@RequestMapping` / `@GetMapping` etc.:** URL + HTTP-method mapping.
- **`@PathVariable` / `@RequestParam` / `@RequestBody`:** path / query / body binding.
- **`ResponseEntity`:** full control of status/headers/body.
- **`@ResponseStatus`:** declarative status code.
- **Content negotiation:** choosing the response format (usually JSON).
- **Jackson:** Java↔JSON library.
- **`@JsonProperty`/`@JsonIgnore`/`@JsonFormat`:** JSON mapping annotations.
- **DTO:** Data Transfer Object (decouples API from entities).

---

*This is the first note of **Section 5.3 — Spring Web MVC**.*
*Next topic: **Validation, Exception Handling, CORS, Filters & Async**.*
