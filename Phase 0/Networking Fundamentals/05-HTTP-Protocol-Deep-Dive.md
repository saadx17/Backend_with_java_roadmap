# HTTP Protocol Deep Dive

> **Phase 0 — Computer Science Fundamentals → 0.2 Networking Fundamentals**
> Goal: Master HTTP — request/response structure, methods, status codes (every common one), headers, and cookies/sessions. This is the protocol your entire backend career is built on.

---

## 0. The Big Picture

**HTTP (HyperText Transfer Protocol)** is the **Application-layer (OSI L7)** protocol of the web. It's a **request–response**, **text-based**, **stateless** protocol: a client sends a **request**, the server returns a **response**. It runs over **TCP** (and TLS for HTTPS).

```
Client  --- HTTP Request -->  Server
Client  <-- HTTP Response --  Server
```

> **Stateless:** each request is independent — the server doesn't inherently remember previous requests. State (logins, carts) is layered on top via **cookies/sessions/tokens** (§6).

As a backend engineer, you will read, write, debug, and design around HTTP **every single day** — this is arguably the most important note in section 0.2.

---

## 1. HTTP Request Structure

An HTTP request has four parts:
```
GET /api/users/42?verbose=true HTTP/1.1      <- 1. Request line
Host: api.example.com                         <- 2. Headers
Authorization: Bearer eyJhbGc...
Content-Type: application/json
Accept: application/json
                                              <- 3. Blank line (separates headers/body)
{ "note": "optional request body" }           <- 4. Body (for POST/PUT/PATCH)
```

| Part | Contains |
|------|----------|
| **Request line** | Method + path (+ query string) + HTTP version |
| **Headers** | Key-value metadata about the request |
| **Blank line** | Separates headers from body |
| **Body** | Optional payload (JSON, form data, file) |

- **Query string** (`?verbose=true`): parameters appended to the URL.
- The **path** identifies the resource; the **method** says what to do with it.

---

## 2. HTTP Response Structure

```
HTTP/1.1 200 OK                               <- 1. Status line
Content-Type: application/json                 <- 2. Headers
Content-Length: 51
Cache-Control: no-cache
                                              <- 3. Blank line
{ "id": 42, "name": "Sam" }                    <- 4. Body
```

| Part | Contains |
|------|----------|
| **Status line** | HTTP version + status code + reason phrase |
| **Headers** | Metadata about the response |
| **Blank line** | Separator |
| **Body** | The returned content (JSON, HTML, etc.) |

---

## 3. HTTP Methods (Verbs)

The method states the **intent**. Two key properties:
- **Safe:** doesn't modify server state (read-only).
- **Idempotent:** repeating it has the same effect as doing it once.

| Method | Purpose | Safe? | Idempotent? | Body? |
|--------|---------|:-----:|:-----------:|:-----:|
| **GET** | Retrieve a resource | ✅ | ✅ | No |
| **POST** | Create a resource / submit data | ❌ | ❌ | Yes |
| **PUT** | Replace a resource entirely | ❌ | ✅ | Yes |
| **PATCH** | Partially update a resource | ❌ | ❌* | Yes |
| **DELETE** | Remove a resource | ❌ | ✅ | Sometimes |
| **HEAD** | Like GET but headers only (no body) | ✅ | ✅ | No |
| **OPTIONS** | Ask what methods/capabilities are allowed (CORS preflight) | ✅ | ✅ | No |

\* PATCH *can* be idempotent depending on the payload, but isn't guaranteed.

### 3.1 Why idempotency matters
If a **PUT** or **DELETE** times out, the client can **safely retry** — repeating it won't cause harm. **POST** is *not* idempotent → retrying may create duplicates (hence **idempotency keys** for payments — Phase 7/12).

### 3.2 PUT vs PATCH vs POST
- **POST `/users`** → create a new user (server assigns ID).
- **PUT `/users/42`** → replace user 42 entirely (send the whole object).
- **PATCH `/users/42`** → update *part* of user 42 (e.g., just the email).

---

## 4. HTTP Status Codes (Know Every Common One)

The 3-digit code's first digit gives the category:
```
1xx  Informational   (rarely seen directly)
2xx  Success
3xx  Redirection
4xx  Client error    (the request is wrong)
5xx  Server error    (the server failed)
```

### 4.1 1xx — Informational
| Code | Meaning |
|------|---------|
| `100 Continue` | Server got headers; client may send the body |
| `101 Switching Protocols` | Upgrading (e.g., to WebSocket) |

### 4.2 2xx — Success
| Code | Meaning |
|------|---------|
| `200 OK` | Standard success (GET/PUT/PATCH) |
| `201 Created` | Resource created (POST) — include `Location` header |
| `202 Accepted` | Accepted for async processing (not done yet) |
| `204 No Content` | Success, no body (common for DELETE) |

### 4.3 3xx — Redirection
| Code | Meaning |
|------|---------|
| `301 Moved Permanently` | Resource moved for good (update bookmarks) |
| `302 Found` | Temporary redirect |
| `304 Not Modified` | Cached version is still valid (conditional GET) |
| `307 / 308` | Temporary / Permanent redirect, **preserving the method** |

### 4.4 4xx — Client Errors (the request is the problem)
| Code | Meaning |
|------|---------|
| `400 Bad Request` | Malformed request / validation failure |
| `401 Unauthorized` | **Not authenticated** (no/invalid credentials) |
| `403 Forbidden` | Authenticated but **not allowed** |
| `404 Not Found` | Resource doesn't exist |
| `405 Method Not Allowed` | Wrong HTTP method for this resource |
| `409 Conflict` | State conflict (e.g., duplicate, version mismatch) |
| `422 Unprocessable Entity` | Well-formed but semantically invalid (validation) |
| `429 Too Many Requests` | Rate limit exceeded |

> ⚠️ **401 vs 403:** `401` = "I don't know who you are" (authentication); `403` = "I know who you are, but you can't do this" (authorization). A classic interview distinction.

### 4.5 5xx — Server Errors (the server failed)
| Code | Meaning |
|------|---------|
| `500 Internal Server Error` | Generic server failure (unhandled exception) |
| `502 Bad Gateway` | Upstream server returned an invalid response |
| `503 Service Unavailable` | Server overloaded / down for maintenance |
| `504 Gateway Timeout` | Upstream server didn't respond in time |

> **Rule of thumb:** 4xx = *client* should fix the request; 5xx = *server* is at fault. Returning the right code is core to good API design (Phase 7).

---

## 5. HTTP Headers

**Headers** are key-value metadata sent with requests/responses. Essentials:

### 5.1 Common request headers
| Header | Purpose |
|--------|---------|
| `Host` | Target domain (required in HTTP/1.1) |
| `Authorization` | Credentials (e.g., `Bearer <token>`, `Basic <base64>`) |
| `Content-Type` | Format of the request body (`application/json`) |
| `Accept` | Formats the client can handle in the response |
| `User-Agent` | Client software identity |
| `Cookie` | Cookies sent back to the server |
| `Accept-Encoding` | Supported compression (`gzip`, `br`) |
| `If-None-Match` / `If-Modified-Since` | Conditional requests (caching) |

### 5.2 Common response headers
| Header | Purpose |
|--------|---------|
| `Content-Type` | Format of the response body |
| `Content-Length` | Body size in bytes |
| `Cache-Control` | Caching directives (`no-cache`, `max-age=3600`) |
| `Set-Cookie` | Tell the client to store a cookie |
| `Location` | URL for redirects / created resources (201) |
| `ETag` | Version identifier for caching/conditional requests |
| `Access-Control-Allow-Origin` | CORS permission (later note) |
| `WWW-Authenticate` | How to authenticate (with 401) |

### 5.3 Content-Type (a.k.a. MIME type)
Tells the receiver how to interpret the body:
| Value | Meaning |
|-------|---------|
| `application/json` | JSON (the standard for REST APIs) |
| `application/x-www-form-urlencoded` | HTML form data |
| `multipart/form-data` | File uploads |
| `text/html`, `text/plain` | HTML / plain text |
| `application/octet-stream` | Arbitrary binary |

---

## 6. Cookies and Sessions (Adding State to a Stateless Protocol)

HTTP is **stateless**, but apps need to remember users (logins, carts). Two main approaches:

### 6.1 Cookies
Small key-value data the server asks the browser to store and send back on every request:
```
Server response:  Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
Client request:   Cookie: sessionId=abc123
```
Important cookie attributes:
| Attribute | Effect |
|-----------|--------|
| `HttpOnly` | JS can't read it → mitigates XSS theft (Phase 15) |
| `Secure` | Only sent over HTTPS |
| `SameSite` | Restricts cross-site sending → mitigates CSRF |
| `Max-Age` / `Expires` | Lifetime |
| `Domain` / `Path` | Scope |

### 6.2 Session-based vs token-based state
| Approach | How it works | Trade-off |
|----------|-------------|-----------|
| **Server-side sessions** | Server stores session data; client holds a `sessionId` cookie | Simple; but server must store state (sticky sessions / shared store — Phase 13) |
| **Token-based (JWT)** | Client holds a self-contained signed token; server stores nothing | Stateless, scalable; but revocation is harder (Phase 5.6) |

> Modern stateless APIs often use **JWT bearer tokens** (in the `Authorization` header) instead of session cookies — better for horizontal scaling (Phase 5.6, 13).

---

## 7. HTTP Versions (brief awareness)
| Version | Key feature |
|---------|-------------|
| HTTP/1.1 | Persistent connections (keep-alive), text-based, one request at a time per connection (head-of-line blocking) |
| HTTP/2 | **Multiplexing** many requests over one TCP connection, binary framing, header compression |
| HTTP/3 | Runs over **QUIC (UDP)** — faster setup, no TCP head-of-line blocking |

> Keep-alive and HTTP/2 multiplexing reduce the cost of TCP/TLS handshakes (recall TCP note) — important for performance (Phase 13).

---

## 8. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| Using `200 OK` for everything | Use correct codes (201, 204, 400, 404, etc.) |
| Confusing 401 and 403 | 401 = not authenticated; 403 = not authorized |
| Using GET to modify data | GET is safe/read-only; use POST/PUT/PATCH/DELETE |
| Putting sensitive data in URLs/query strings | They get logged/cached — use headers/body + HTTPS |
| Assuming HTTP remembers you | It's stateless — state is via cookies/tokens |
| POST is idempotent | It's not — retries can duplicate |
| Returning 200 with an error in the body | Use the proper status code |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Spring MVC (Phase 5.3)** maps directly to this: `@GetMapping`/`@PostMapping`, `@RequestBody`, `@RequestParam`, `ResponseEntity`, `@ResponseStatus` — all HTTP concepts.
- **Status codes** are central to REST API design and error handling (ProblemDetail — Phase 5.3, 7).
- **Headers** drive content negotiation, auth (`Authorization`), caching (`ETag`/`Cache-Control`), and CORS (Phase 5, 7, 13).
- **Cookies vs JWT** is a core auth design decision (Phase 5.6).
- **Idempotency** underpins safe retries and payment APIs (Phase 7, 12).
- **HTTP/2 & keep-alive** inform performance tuning (Phase 13).
- **`curl`** (section 0.3) lets you craft/inspect raw HTTP — an everyday debugging tool.

---

## 10. Quick Self-Check Questions

1. What are the four parts of an HTTP request and response?
2. Why is HTTP called "stateless," and how is state added?
3. Define "safe" and "idempotent." Which methods are which?
4. What's the difference between PUT, PATCH, and POST?
5. Give the meaning of: 201, 204, 301, 400, 401, 403, 404, 409, 429, 500, 502, 503.
6. What's the precise difference between 401 and 403?
7. Name five common headers and what they do.
8. What do `HttpOnly`, `Secure`, and `SameSite` cookie attributes protect against?
9. What's the difference between server-side sessions and JWT tokens?

---

## 11. Key Terms Glossary

- **HTTP:** stateless request–response application protocol of the web.
- **Request/response line:** method+path+version / version+status+reason.
- **Method (verb):** GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS.
- **Safe / idempotent:** read-only / same effect when repeated.
- **Status code:** 3-digit result (1xx–5xx).
- **Header:** key-value metadata on a request/response.
- **Content-Type / MIME type:** body format indicator.
- **Query string:** `?key=value` URL parameters.
- **Stateless:** server doesn't inherently remember prior requests.
- **Cookie:** client-stored data resent to the server.
- **Session / JWT:** server-side state vs self-contained token.
- **Keep-alive / multiplexing:** connection reuse / concurrent requests (HTTP/1.1, HTTP/2).

---

*Previous topic: **TCP vs UDP**.*
*Next topic: **HTTPS & the TLS Handshake**.*
