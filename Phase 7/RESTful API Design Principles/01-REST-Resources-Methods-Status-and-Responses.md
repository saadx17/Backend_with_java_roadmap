# RESTful API Design — Resources, Methods, Status Codes & Responses

> **Phase 7 — API Design & Documentation → 7.1 RESTful API Design Principles (Part 1 of 2)**
> Goal: Design clean REST APIs — resource naming, correct HTTP methods & semantics, status codes, well-structured request/response bodies (incl. RFC 7807 `ProblemDetail`, pagination envelopes, HATEOAS), and versioning strategies.

---

## 0. The Big Picture

**REST** (Representational State Transfer) is an architectural *style* (not a protocol) for designing networked APIs over HTTP. The core idea: model your domain as **resources** (nouns), addressed by **URIs**, and act on them with the **uniform interface** of HTTP **methods** (verbs). The server returns a **representation** (usually JSON) plus a **status code** describing the outcome.

```
        VERB        RESOURCE URI            =  ACTION
        ----        ------------               ------
        GET     /api/v1/orders/42           -> read order 42
        POST    /api/v1/orders              -> create an order
        PUT     /api/v1/orders/42           -> replace order 42
        PATCH   /api/v1/orders/42           -> partially update order 42
        DELETE  /api/v1/orders/42           -> delete order 42
```

> This builds directly on HTTP fundamentals (Phase 0.2 — methods, status codes, headers) and Spring Web MVC (Phase 5.3 — `@RestController`, `@RequestMapping`). The API *contract* you design here is documented in 7.2 (OpenAPI) and shaped with DTOs in 7.3. Part 2 covers the advanced mechanics (idempotency, ETags, PATCH formats, async, bulk).

### REST's 6 architectural constraints (Roy Fielding)
| Constraint | Meaning |
|------------|---------|
| **Client–Server** | Separation of UI from data storage |
| **Stateless** | Each request carries all context; server keeps no session (⭐ enables horizontal scaling — Phase 13.3) |
| **Cacheable** | Responses declare cacheability (ETags/Cache-Control — Part 2) |
| **Uniform Interface** | Resources + standard methods + self-descriptive messages + HATEOAS |
| **Layered System** | Proxies/gateways/LBs can sit between client & server (Phase 12.4) |
| **Code on Demand** (optional) | Server can send executable code (rarely used) |

> ⭐ **Stateless** is the most important for backend scaling: no server-side session means any instance can serve any request → load-balance freely (Phase 13.3). Auth travels per-request via tokens (JWT — Phase 5.6), not sessions.

---

## 1. Resource Naming

Resources are **nouns**, not verbs. The URI identifies *what*; the HTTP method says *what to do*.

| Rule | ✅ Good | ❌ Bad |
|------|--------|-------|
| Use **nouns**, not verbs | `GET /orders` | `GET /getOrders` |
| Use **plural** for collections | `/orders`, `/users` | `/order`, `/user` |
| **Hierarchy** for sub-resources | `/orders/42/items` | `/orderItems?orderId=42` |
| **Lowercase + hyphens** (kebab-case) | `/shipping-addresses` | `/shippingAddresses`, `/shipping_addresses` |
| **No trailing slash** | `/orders` | `/orders/` |
| **No file extensions** | `/orders/42` (+ `Accept` header) | `/orders/42.json` |
| Plurals consistent across API | always plural | mixing singular/plural |

```
/api/v1/users                       # collection of users
/api/v1/users/42                    # a single user (by id)
/api/v1/users/42/orders             # that user's orders (sub-collection)
/api/v1/users/42/orders/7           # one specific order of that user
```

### 1.1 Resource vs sub-resource vs action edge cases
- **Nesting depth:** keep it shallow — ideally **≤ 2 levels** (`/users/42/orders`). Beyond that, prefer a top-level resource with a filter: `/orders?userId=42` (filtering covered in Part 2).
- **Non-CRUD "actions"** (verbs that don't fit a method) — model as a sub-resource or use a controller pattern:
  - `POST /orders/42/cancel` (action as sub-resource) — pragmatic & common.
  - or model the state change: `POST /orders/42/cancellation` (a cancellation resource).
  - ⚠️ Avoid `POST /cancelOrder?id=42` — that's RPC, not REST.
- **Singleton sub-resources** (one per parent): `/users/42/profile`, `/account/settings`.

> ⭐ Good URIs are **discoverable and predictable**. A developer should guess `/users/42/orders` works after seeing `/users`. This consistency is what HATEOAS (§6) formalizes.

---

## 2. HTTP Methods & Their Semantics

| Method | Purpose | Safe? | Idempotent? | Has body? | Typical success |
|--------|---------|:-----:|:-----------:|:---------:|-----------------|
| **GET** | Read a resource/collection | ✅ | ✅ | no | 200 |
| **POST** | Create (or non-idempotent action) | ❌ | ❌ | yes | 201 / 200 |
| **PUT** | Full **replace** (or create at known URI) | ❌ | ✅ | yes | 200 / 204 |
| **PATCH** | **Partial** update | ❌ | ❌* | yes | 200 / 204 |
| **DELETE** | Remove a resource | ❌ | ✅ | optional | 204 / 200 |
| **HEAD** | Like GET but headers only (no body) | ✅ | ✅ | no | 200 |
| **OPTIONS** | Discover allowed methods (CORS preflight — Phase 5.3) | ✅ | ✅ | no | 200 |

- **Safe** = does not modify server state (read-only). GET/HEAD/OPTIONS.
- **Idempotent** = making the same request N times has the **same effect** as once. Critical for retries (Part 2 — networks fail; clients retry).
- *PATCH idempotency depends on the patch format: a JSON Merge Patch setting a field is idempotent; an "increment by 1" op is not.

### 2.1 PUT vs PATCH vs POST (the classic confusion)
```
PUT /users/42      body = {name, email, phone}   -> REPLACE the whole user (omitted fields are cleared!)
PATCH /users/42    body = {phone: "..."}          -> change ONLY phone (other fields untouched)
POST /users        body = {name, email}           -> CREATE a new user; server assigns the id/URI
```
> ⚠️ **PUT replaces the entire resource.** If a client `PUT`s a partial body, omitted fields may be **nulled out**. Use **PATCH** for partial updates (detailed formats in Part 2). Use **POST** to create when the server assigns the identifier; **PUT** to create only when the client controls the URI (`PUT /users/my-chosen-id`).

---

## 3. Status Codes

Return the **most specific** code that fits. (Full list in Phase 0.2.)

### 2xx — Success
| Code | Meaning | Use when |
|------|---------|----------|
| **200 OK** | Generic success | GET success; PUT/PATCH returning the updated body |
| **201 Created** | Resource created | POST created a resource — **include a `Location` header** with the new URI |
| **202 Accepted** | Accepted, processing async | Long-running ops (async API — Part 2) |
| **204 No Content** | Success, no body | DELETE, or PUT/PATCH that returns nothing |

### 3xx — Redirection / caching
| Code | Use |
|------|-----|
| **301 / 308** | Resource permanently moved |
| **304 Not Modified** | Conditional GET — client's cached copy is still valid (ETags — Part 2) |

### 4xx — Client errors (the caller did something wrong)
| Code | Meaning |
|------|---------|
| **400 Bad Request** | Malformed syntax / generic invalid request |
| **401 Unauthorized** | **Not authenticated** (no/invalid credentials — Phase 5.6) |
| **403 Forbidden** | Authenticated but **not allowed** (authorization — Phase 5.6) |
| **404 Not Found** | Resource doesn't exist |
| **405 Method Not Allowed** | Method not supported on this URI |
| **406 Not Acceptable** | Can't produce the `Accept`ed media type |
| **409 Conflict** | State conflict (duplicate, optimistic-lock failure — Phase 4.5) |
| **415 Unsupported Media Type** | Body `Content-Type` not supported |
| **422 Unprocessable Entity** | Syntactically valid but **semantically invalid** (validation failed — Phase 5.3) |
| **429 Too Many Requests** | Rate limit exceeded (Part 2) |

### 5xx — Server errors (your fault)
| Code | Meaning |
|------|---------|
| **500 Internal Server Error** | Unhandled exception (⚠️ never leak stack traces — Phase 15) |
| **502 Bad Gateway** | Upstream returned an invalid response |
| **503 Service Unavailable** | Overloaded / down for maintenance (often with `Retry-After`) |
| **504 Gateway Timeout** | Upstream timed out (Phase 12.3 — timeouts) |

> ⭐ **401 vs 403:** 401 = "I don't know who you are" (authenticate). 403 = "I know who you are, but you can't do this." ⚠️ **400 vs 422:** 400 = the request couldn't be parsed/bound; 422 = it parsed fine but failed business/validation rules. Many APIs use 400 for both — pick one convention and document it (7.2).

---

## 4. Request & Response Body Design

- **Format:** JSON by default. Negotiate via `Accept` / `Content-Type` headers (Phase 0.2/5.3).
- **Field naming:** be consistent — `camelCase` (Java/JS convention) or `snake_case`; pick one. Jackson can map between Java fields and JSON (Phase 5.3).
- **Dates/times:** always **ISO-8601 with timezone** (`2026-06-16T09:30:00Z`) — never ambiguous local formats. Use UTC on the wire.
- **Money:** use strings or integer minor-units (cents) to avoid float rounding (recall floating-point, Phase 0.1) — never `double` for currency.
- **Booleans/enums:** use explicit values; document enum sets (7.2).
- **Nulls vs omission:** decide whether to omit null fields (`@JsonInclude(NON_NULL)`) — document it.
- ⭐ **Use DTOs**, never expose entities directly (the whole point of 7.3).

### 4.1 Consistent envelopes (optional)
Some APIs wrap every response in an envelope; others return the resource directly. Be consistent.
```jsonc
// Direct (common, RESTful):
{ "id": 42, "name": "Alice", "email": "alice@x.com" }

// Enveloped (adds metadata; useful for collections/pagination — §5):
{ "data": { ... }, "meta": { ... }, "links": { ... } }
```

---

## 5. Pagination Response Format

Never return an unbounded collection (could be millions of rows — Phase 4.4). Always paginate. Two common strategies (mechanics/query params in Part 2):

### 5.1 Offset/limit (page-based)
```http
GET /api/v1/orders?page=2&size=20&sort=createdAt,desc
```
```jsonc
{
  "content": [ { "id": 41, ... }, { "id": 42, ... } ],
  "page": {
    "number": 2,
    "size": 20,
    "totalElements": 137,
    "totalPages": 7
  }
}
```
> This mirrors Spring Data's `Page<T>` (Phase 5.4) — Spring serializes `Pageable`/`Page` to roughly this shape. ⭐ As of Spring Boot 3.3+, prefer the **stable `PagedModel`** serialization over the raw `Page` JSON (whose format is not contractually stable).

### 5.2 Cursor/keyset (for large/real-time data)
```jsonc
{
  "content": [ ... ],
  "next_cursor": "eyJpZCI6NDJ9",        // opaque token pointing to the next page
  "has_more": true
}
```
> ⭐ **Cursor pagination scales better** than offset for deep pages — offset `LIMIT 20 OFFSET 100000` makes the DB scan & discard 100k rows (Phase 4.4 indexing). Cursors use a `WHERE id > ?` keyset → index-friendly. Trade-off: can't jump to an arbitrary page. (More in Part 2 & Phase 13.)

---

## 6. HATEOAS (Hypermedia)

**HATEOAS** (Hypermedia As The Engine Of Application State) = responses include **links** to related actions/resources, so clients **discover** what they can do next without hardcoding URIs. It's the "uniform interface" constraint taken to its fullest (Richardson Maturity Model **Level 3**).

```jsonc
{
  "id": 42,
  "status": "PENDING",
  "total": "59.90",
  "_links": {
    "self":   { "href": "/api/v1/orders/42" },
    "items":  { "href": "/api/v1/orders/42/items" },
    "cancel": { "href": "/api/v1/orders/42/cancel" },   // only present while cancellable
    "pay":    { "href": "/api/v1/orders/42/payment" }
  }
}
```

**Richardson Maturity Model** (how "RESTful" an API is):
| Level | Description |
|-------|-------------|
| 0 | Single URI, single method (RPC-over-HTTP / SOAP-ish) |
| 1 | Multiple resources (URIs) |
| 2 | Proper HTTP verbs + status codes ← **most "REST" APIs live here** |
| 3 | Hypermedia (HATEOAS) — fully self-describing |

> Spring HATEOAS (`spring-boot-starter-hateoas`) provides `EntityModel`, `CollectionModel`, `WebMvcLinkBuilder.linkTo(...)`. ⚠️ **In practice, full HATEOAS is rare** — most public APIs (GitHub, Stripe) stop at Level 2 + some links. It adds complexity for clients that often hardcode URLs anyway. Know it; apply it judiciously.

---

## 7. Error Responses — RFC 7807 `ProblemDetail`

Errors should be **machine-readable and consistent**, not bare strings. **RFC 7807 (Problem Details for HTTP APIs)** defines a standard JSON error shape, and Spring (6+/Boot 3+) supports it natively via `ProblemDetail`.

```jsonc
// Content-Type: application/problem+json
{
  "type": "https://api.example.com/problems/insufficient-funds",  // URI identifying the problem type
  "title": "Insufficient funds",            // short, human-readable summary
  "status": 422,                            // mirrors the HTTP status
  "detail": "Account 42 has balance 10.00 but the order total is 59.90.",  // this occurrence
  "instance": "/api/v1/orders/42/payment",  // URI of the specific occurrence
  "errors": [                               // (extension) field-level validation errors
    { "field": "quantity", "message": "must be >= 1" }
  ],
  "traceId": "a1b2c3"                        // (extension) correlate with logs/tracing (Phase 9.2)
}
```

### 7.1 Producing it in Spring
```java
@RestControllerAdvice                                  // global exception handling (Phase 5.3)
class ApiExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    ProblemDetail handleNotFound(EntityNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setType(URI.create("https://api.example.com/problems/not-found"));
        pd.setTitle("Resource not found");
        pd.setProperty("traceId", MDC.get("traceId"));  // extension member (Phase 9.1 MDC)
        return pd;                                       // Spring serializes as application/problem+json
    }
}
```
> ⭐ Spring's `@ControllerAdvice`/`ResponseEntityExceptionHandler` already emit `ProblemDetail` for built-in exceptions when you set `spring.mvc.problemdetails.enabled=true`. Use a **`@RestControllerAdvice`** to centralize all error mapping (recall Phase 5.3 — `@ExceptionHandler`). ⚠️ **Never leak stack traces, SQL, or internal class names** to clients (Phase 15 — info disclosure). Log details server-side with a `traceId`; return a safe message.

---

## 8. API Versioning

APIs evolve; you must change them **without breaking existing clients**. Version when you make a **breaking change** (removing/renaming a field, changing types/semantics). *Adding* optional fields is backward-compatible and needs no version bump.

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URI path** ⭐ | `/api/v1/orders` | Simple, visible, cache/route-friendly | "Pollutes" the URI; v-bump touches all paths |
| **Query param** | `/api/orders?version=1` | Easy default | Easy to forget; messy caching |
| **Custom header** | `X-API-Version: 1` | Clean URIs | Invisible; harder to test in a browser |
| **Accept header (media type)** | `Accept: application/vnd.example.v1+json` | "Purest" REST (content negotiation) | Complex; poor tooling/discoverability |

```java
@RestController
@RequestMapping("/api/v1/orders")     // URI versioning — the pragmatic default
class OrderV1Controller { ... }
```

> ⭐ **URI path versioning (`/v1/`) is the most common & pragmatic** (GitHub, Stripe-ish, most enterprise APIs) — visible, easy to route at the gateway (Phase 12.4) and cache. **Best practice: avoid breaking changes** when you can (additive evolution), and when you must version, **support the old version during a deprecation window** (announce via `Deprecation`/`Sunset` headers). Semantic versioning concepts recall Phase 3 (Maven/Gradle deps).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Verbs in URIs (`/getUsers`, `/createOrder`) | Nouns + HTTP methods |
| Using `200 OK` for everything (even errors) | Use correct status codes (201, 204, 4xx, 5xx) |
| `POST` for reads / `GET` with side effects | GET must be safe; POST creates |
| `PUT` used for partial updates (nulling fields) | Use `PATCH` for partial (Part 2) |
| Returning 201 without a `Location` header | Always set `Location` on create |
| Exposing JPA entities directly | Use DTOs (7.3) |
| Bare-string error bodies | RFC 7807 `ProblemDetail` |
| Leaking stack traces / SQL in errors | Safe message + server-side log + traceId (Phase 15) |
| Unbounded collections | Always paginate (§5) |
| Breaking clients by changing fields silently | Version (§8); additive changes only |
| Mixing singular/plural, camel/kebab in URIs | One consistent convention |
| Local/ambiguous date formats | ISO-8601 UTC |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Spring Web MVC (Phase 5.3)** implements all of this: `@RestController`, `@GetMapping`/`@PostMapping`, `ResponseEntity` (set status + `Location`), `@RequestBody`/`@ResponseBody`, content negotiation.
- **Validation (Phase 5.3)** → 400/422 with field-level errors in `ProblemDetail`.
- **`@RestControllerAdvice`** centralizes RFC 7807 error responses (§7).
- **Spring Data `Page`/`Pageable` (Phase 5.4)** powers pagination (§5); index design (Phase 4.4) makes it fast.
- **Security (Phase 5.6)** drives 401/403 semantics.
- **OpenAPI (7.2)** documents every URI, method, status, and schema you design here.
- **DTOs + MapStruct (7.3)** shape the request/response bodies.
- **API Gateway (Phase 12.4)** routes by version/path; **rate limiting/ETags** (Part 2) live at this layer too.
- Used heavily in **Project 4 (Task Management REST API)** and **Project 5 (E-Commerce Backend)**.

---

## 11. Quick Self-Check Questions

1. What are the 6 REST constraints, and why is *statelessness* key for scaling?
2. Give the resource-naming rules — nouns vs verbs, plural, kebab-case, nesting depth.
3. Which methods are *safe* and which are *idempotent*? Why does idempotency matter for retries?
4. Explain PUT vs PATCH vs POST with an example each.
5. When do you return 201 vs 200 vs 204? 401 vs 403? 400 vs 422?
6. What header must accompany a 201 Created?
7. What is RFC 7807 `ProblemDetail`, and how do you produce it in Spring?
8. Compare offset vs cursor pagination — when is each better?
9. What is HATEOAS / the Richardson Maturity Model, and why is full HATEOAS rare?
10. Compare the four versioning strategies — which is most common and why?

---

## 12. Key Terms Glossary

- **Resource / URI / representation:** the thing, its address, its serialized form (JSON).
- **Uniform interface:** standard methods (GET/POST/PUT/PATCH/DELETE) over resources.
- **Safe method:** read-only (GET/HEAD/OPTIONS).
- **Idempotent method:** N identical requests = same effect as 1 (GET/PUT/DELETE).
- **Status code:** 2xx success, 3xx redirect/cache, 4xx client error, 5xx server error.
- **`Location` header:** URI of a newly created resource (with 201).
- **RFC 7807 / `ProblemDetail`:** standard machine-readable error JSON (`application/problem+json`).
- **Pagination (offset vs cursor):** page/size vs opaque keyset cursor.
- **HATEOAS:** responses embed links to related actions (Richardson Level 3).
- **Richardson Maturity Model:** Levels 0–3 of "RESTfulness".
- **Versioning:** URI / query / header / media-type strategies for evolving APIs.
- **Content negotiation:** `Accept`/`Content-Type` selecting representation/format.

---

*This is Part 1 of the note for **Section 7.1 — RESTful API Design Principles**.*
*Next: **7.1 Part 2 — Idempotency, Conditional Requests, PATCH, Async & Bulk** (`02-...`).*
