# RESTful API Design — Idempotency, Conditional Requests, PATCH, Rate Limiting, Async & Bulk

> **Phase 7 — API Design & Documentation → 7.1 RESTful API Design Principles (Part 2 of 2)**
> Goal: Master the advanced mechanics that make REST APIs robust at scale — idempotency keys, filtering/sorting/pagination query params, rate limiting headers, ETags/conditional requests, PATCH formats (JSON Merge Patch & JSON Patch), async APIs (202 + polling), file uploads, and bulk operations.

---

## 0. The Big Picture

Part 1 covered the *shape* of a REST API (resources, methods, status codes, bodies, versioning). Part 2 covers the **operational mechanics** that make APIs survive the real world: unreliable networks (retries → idempotency), large datasets (pagination/filtering), caching/concurrency (ETags), partial edits (PATCH), abuse (rate limiting), long jobs (async), and batch work (bulk).

```
Client retries on timeout ───────────► Idempotency keys (don't double-charge)
Huge collections ────────────────────► Filter + sort + paginate
Caching & concurrency ───────────────► ETags + conditional requests (If-None-Match / If-Match)
Partial edits ───────────────────────► PATCH (JSON Merge Patch / JSON Patch)
Abuse / fairness ────────────────────► Rate limiting (429 + headers)
Long-running work ───────────────────► 202 Accepted + polling / callbacks
Batch work ──────────────────────────► Bulk endpoints (partial success)
```

> Builds on Part 1 (this section), HTTP caching/headers (Phase 0.2), optimistic locking (Phase 4.5 — version columns → ETags), Spring Web MVC (Phase 5.3), and connects forward to rate limiting at the gateway (Phase 12.4), async/messaging (Phase 11), and performance (Phase 13).

---

## 1. Idempotency & Idempotency Keys

**Idempotent** = repeating a request produces the **same result/state** as doing it once (Part 1 §2). GET/PUT/DELETE are *naturally* idempotent. **POST is not** — retrying `POST /payments` could charge twice.

The problem: a client sends `POST /payments`, the network drops the **response** (but the server already processed it). The client times out and **retries** → double charge. 💥

**Solution: idempotency keys.** The client generates a unique key (e.g., a UUID) per logical operation and sends it in a header:
```http
POST /api/v1/payments
Idempotency-Key: 7b3f...-uuid
Content-Type: application/json

{ "orderId": 42, "amount": "59.90" }
```

Server algorithm:
```
1. Look up the Idempotency-Key in a store (e.g., Redis — Phase 4.8, with a TTL).
2. If FOUND  -> return the SAVED response (do NOT re-execute) -> same result.
3. If ABSENT -> execute, then SAVE (key -> response + status) -> return it.
   (Use a lock/atomic insert to handle concurrent retries — Phase 12.6 distributed locking.)
```

| Concern | Guidance |
|---------|----------|
| Where to store keys | Redis (TTL e.g. 24h) or a DB table with a unique constraint |
| Scope | Per endpoint + per client/account |
| Concurrent duplicates | Atomic "insert if absent" or distributed lock (Redisson — Phase 12.6) |
| Response replay | Persist status + body + headers, replay verbatim |
| Key generation | Client-side UUID per logical action (not per HTTP retry) |

> ⭐ Stripe popularized this pattern (`Idempotency-Key` header). It's **essential for any "create" or money-moving POST** in a distributed system where retries happen (Phase 12 — at-least-once delivery, Phase 11). Combine with the **Outbox/Inbox patterns** (Phase 11.4/12.6) for end-to-end exactly-once *effect*.

---

## 2. Filtering, Sorting & Pagination (Query Params)

For collection endpoints, use **query parameters** (not new URIs) to filter, sort, and page. These compose.

```http
GET /api/v1/orders?status=PENDING&minTotal=20&createdAfter=2026-01-01
                  &sort=createdAt,desc&sort=id,asc
                  &page=0&size=20
```

### 2.1 Filtering
| Style | Example | Notes |
|-------|---------|-------|
| **Exact match** | `?status=PENDING&userId=42` | Simplest; maps to `WHERE status=? AND user_id=?` |
| **Operators** | `?minTotal=20&maxTotal=100` or `?total[gte]=20` | Range/comparison filters |
| **Search** | `?q=alice` | Free-text search (→ Elasticsearch, Phase 16.4) |
| **Multi-value** | `?status=PENDING,PAID` | CSV or repeated param |

> ⚠️ **Build filters safely** — bind to typed params and use parameterized queries / JPA Criteria / Specifications (Phase 5.4, Phase 14.1 Specification pattern). **Never** concatenate query params into SQL → SQL injection (Phase 15.1). Whitelist sortable/filterable fields.

### 2.2 Sorting
- `?sort=field,direction` (e.g., `sort=createdAt,desc`); allow multiple `sort` params for tie-breakers.
- ⚠️ **Whitelist** sortable columns — don't let clients sort by arbitrary/unindexed columns (Phase 4.4 — a sort on an unindexed column = full scan + filesort).

### 2.3 Pagination (mechanics)
| Param set | Strategy | When |
|-----------|----------|------|
| `page` + `size` | **Offset** (Part 1 §5.1) | Small/medium data, need page numbers |
| `cursor` (+ `size`) | **Cursor/keyset** (Part 1 §5.2) | Large/infinite/real-time feeds |

```java
// Spring Data binds page/size/sort straight to a Pageable (Phase 5.4):
@GetMapping("/orders")
Page<OrderDto> list(@RequestParam(required=false) Status status, Pageable pageable) {
    return service.search(status, pageable);   // ?page=0&size=20&sort=createdAt,desc auto-bound
}
```
> ⭐ Cap `size` (e.g., max 100) to protect the server. Default sensible `size`. Return total counts only when cheap (counting millions of rows is expensive — Phase 13.1).

---

## 3. ETags & Conditional Requests

An **ETag** (Entity Tag) is an opaque identifier (often a hash or version) for a specific representation of a resource. It enables **caching** (skip transfer if unchanged) and **optimistic concurrency** (prevent lost updates).

```http
GET /api/v1/orders/42
                                            <- 200 OK
                                               ETag: "v7"
                                               { ...order... }
```

### 3.1 Conditional GET (caching → 304)
```http
GET /api/v1/orders/42
If-None-Match: "v7"
                                            <- 304 Not Modified   (empty body — client reuses cache)
```
> Saves bandwidth (Phase 0.2 caching, Phase 13.1 HTTP optimization). If the ETag changed, the server returns 200 + the new body + new ETag.

### 3.2 Conditional update (optimistic concurrency → 412)
```http
PUT /api/v1/orders/42
If-Match: "v7"
{ ...changes... }
                                            <- 200 OK            if current ETag == "v7"
                                            <- 412 Precondition Failed  if it changed meanwhile
```
> ⭐ This prevents the **lost update** problem: two clients both GET v7, both edit, both PUT — the second gets **412** instead of silently overwriting the first. This maps directly to JPA **optimistic locking** (`@Version` — Phase 4.5/5.4): the version column *is* your ETag.

```java
// Spring: ShallowEtagHeaderFilter auto-adds ETags; or set explicitly:
return ResponseEntity.ok().eTag("\"v" + order.getVersion() + "\"").body(dto);
```

| Header | Used with | Meaning |
|--------|-----------|---------|
| `ETag` | response | Version id of this representation |
| `If-None-Match` | GET | "Send body only if ETag differs" → 304 if same |
| `If-Match` | PUT/PATCH/DELETE | "Only modify if ETag matches" → 412 if not |
| `Last-Modified` / `If-Modified-Since` | GET | Time-based alternative to ETags |

---

## 4. Partial Updates — PATCH Formats

PATCH does **partial** updates, but "partial" needs a precise format. Two RFC standards:

### 4.1 JSON Merge Patch (RFC 7386) ⭐ simplest
`Content-Type: application/merge-patch+json`. The body is a *partial object*: present fields are set; **`null` means "delete/clear" the field**; absent fields are untouched.
```http
PATCH /api/v1/users/42
Content-Type: application/merge-patch+json

{ "phone": "+8801...", "middleName": null }   // set phone, clear middleName, leave the rest
```
- ✅ Intuitive, easy for clients.
- ⚠️ **Cannot** explicitly set a field to `null` (null *means delete*). Can't patch array elements precisely (arrays are replaced wholesale).

### 4.2 JSON Patch (RFC 6902) — operation list, more powerful
`Content-Type: application/json-patch+json`. The body is an **array of operations** (`add`, `remove`, `replace`, `move`, `copy`, `test`) with JSON Pointer paths.
```http
PATCH /api/v1/users/42
Content-Type: application/json-patch+json

[
  { "op": "replace", "path": "/phone", "value": "+8801..." },
  { "op": "remove",  "path": "/middleName" },
  { "op": "add",     "path": "/tags/-", "value": "vip" },   // append to array
  { "op": "test",    "path": "/version", "value": 7 }       // precondition (concurrency)
]
```
- ✅ Precise array edits, move/copy, `test` op for preconditions.
- ⚠️ More complex; harder to read; needs a library to apply.

| | JSON Merge Patch (7386) | JSON Patch (6902) |
|--|------------------------|-------------------|
| Shape | Partial object | Array of ops |
| Set null | impossible (null=delete) | `replace` with value `null` |
| Array element edits | replace whole array | precise (`add`/`remove` by index) |
| Readability | high | lower |
| Use when | simple field updates | complex/precise edits |

> In Spring, apply JSON Patch with a library (e.g., `json-patch`/`zjsonpatch`): deserialize to a `JsonNode`, apply ops to the resource's JSON, map back to the entity, validate, save (Phase 5.4). ⚠️ **Re-validate after applying a patch** — clients can patch into an invalid state.

---

## 5. Rate Limiting & Headers

Protect the API from abuse and ensure fair use. When a client exceeds its quota, return **429 Too Many Requests** and communicate limits via headers.

```http
                                            <- 429 Too Many Requests
                                               Retry-After: 30
                                               RateLimit-Limit: 100
                                               RateLimit-Remaining: 0
                                               RateLimit-Reset: 30
```

| Header | Meaning |
|--------|---------|
| `RateLimit-Limit` | Total requests allowed in the window |
| `RateLimit-Remaining` | Requests left in the current window |
| `RateLimit-Reset` | Seconds (or timestamp) until the window resets |
| `Retry-After` | How long to wait before retrying (also used with 503) |

> ⭐ Algorithms: **token bucket**, **leaky bucket**, **fixed/sliding window**. Implement with **Redis** (atomic counters — Phase 4.8) for a distributed limit across instances, or **Bucket4j**, or at the **API Gateway** (Spring Cloud Gateway `RequestRateLimiter` + Redis — Phase 12.4). Also a defense-in-depth security control (Phase 15.1). Clients should honor `Retry-After` with **exponential backoff + jitter** (Phase 12.3).

---

## 6. Async APIs (Long-Running Operations)

Some operations take too long for a synchronous request (report generation, video processing, bulk import). Don't block the connection — return **202 Accepted** and let the client poll or be notified.

### 6.1 202 + polling pattern
```http
POST /api/v1/reports
{ "type": "annual", "year": 2025 }
                                            <- 202 Accepted
                                               Location: /api/v1/reports/jobs/abc123
                                               { "jobId": "abc123", "status": "PENDING" }

GET /api/v1/reports/jobs/abc123
                                            <- 200 OK
                                               { "status": "RUNNING", "progress": 60 }

GET /api/v1/reports/jobs/abc123             (later)
                                            <- 200 OK
                                               { "status": "COMPLETED",
                                                 "result": { "href": "/api/v1/reports/987" } }
```
| Step | Detail |
|------|--------|
| Submit | `POST` → **202** + `Location` of a **status/job resource** |
| Poll | `GET` the job URI → status (`PENDING`/`RUNNING`/`COMPLETED`/`FAILED`) + progress |
| Result | On completion, link to the result resource |
| Alternative to polling | **Webhooks/callbacks** (server calls client), or **WebSocket/SSE** (Phase 16.3) push |

> ⭐ The heavy work runs out-of-band: a `@Async` method (Phase 5.3/5.8), a scheduled job (Phase 5.8), or — best at scale — a **message queue/Kafka consumer** (Phase 11). The job resource is your unit of tracking. This is the foundation of resilient, decoupled backends (Phase 11/12).

---

## 7. File Upload & Download Design

| Concern | Guidance |
|---------|----------|
| **Upload encoding** | `multipart/form-data` (file + metadata fields) for browser/form uploads |
| **Spring** | `@PostMapping(consumes=MULTIPART_FORM_DATA)` + `@RequestPart MultipartFile file` (Phase 5.3) |
| **Large files** | Stream (don't buffer whole file in memory — Phase 1.9 I/O); set max size limits |
| **Very large / direct-to-storage** | **Pre-signed URLs** (S3 — Phase 16.7): API returns a time-limited URL; client uploads straight to object storage, bypassing your server |
| **Resumable / chunked** | `tus` protocol or multipart byte-range chunks for unreliable networks |
| **Validation** | Check type/size; ⚠️ never trust the client `Content-Type`/filename — scan & sniff (Phase 15) |
| **Download** | `Content-Disposition: attachment; filename="..."`; support `Range` requests for resume/streaming |
| **Metadata** | Return a file resource (id, URL, size, checksum) after upload |

> ⭐ For cloud backends, **pre-signed URLs** are the scalable pattern — your app never proxies the bytes (saves bandwidth/CPU — Phase 13.1). Store only metadata + the storage key.

---

## 8. Bulk / Batch Operations

When clients need to create/update/delete many resources at once, a per-item round trip is wasteful. Offer bulk endpoints — but handle **partial success** carefully.

```http
POST /api/v1/orders/bulk
{ "items": [ {...}, {...}, {...} ] }
                                            <- 207 Multi-Status   (per-item outcomes)
                                               { "results": [
                                                   { "index": 0, "status": 201, "id": 51 },
                                                   { "index": 1, "status": 422,
                                                     "error": { "title": "...", "detail": "..." } },
                                                   { "index": 2, "status": 201, "id": 52 }
                                               ] }
```
| Decision | Options |
|----------|---------|
| **All-or-nothing vs partial** | Transactional (Phase 4.5 — all rollback on any failure) **or** best-effort with per-item results |
| **Status code** | `200`/`207 Multi-Status` (mixed outcomes) — return per-item statuses |
| **Size limits** | Cap batch size (e.g., 1000); reject oversized with 413/400 |
| **Idempotency** | Combine with idempotency keys (§1) for safe retries |
| **Async for huge batches** | Switch to the async/job pattern (§6) |

> ⚠️ The hardest part is **partial failure semantics** — decide and **document** (7.2) whether a bad item fails the whole batch (atomic) or just that item (best-effort). For very large batches, prefer **async ingestion** (§6) + a queue (Phase 11).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Retrying POST without idempotency → double effects | Idempotency-Key + server-side dedup store (§1) |
| Building filter/sort SQL by string concat | Parameterized queries / Specifications; whitelist fields (§2, Phase 15.1) |
| Offset pagination on deep pages (slow) | Cursor/keyset pagination (§2.3, Part 1 §5.2) |
| No `size` cap → client requests millions | Cap & default page size |
| Ignoring ETags → lost updates | `If-Match` → 412 on conflict (§3.2, Phase 4.5 `@Version`) |
| PATCH with no defined format | Pick JSON Merge Patch or JSON Patch; document it |
| Forgetting to re-validate after a PATCH | Validate the resulting state (§4.2) |
| Rate limiting without informative headers | Send `RateLimit-*` + `Retry-After` (§5) |
| Blocking the request for long jobs | 202 + job resource + polling/push (§6) |
| Buffering huge uploads in memory | Stream / pre-signed URLs (§7) |
| Bulk endpoint with unclear failure semantics | Define atomic vs partial; use 207 + per-item results (§8) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Idempotency keys (§1)** rely on **Redis** (Phase 4.8) / **distributed locks** (Redisson — Phase 12.6) and pair with **Outbox/Inbox** (Phase 11.4).
- **Filtering/sorting (§2)** uses **JPA Specifications / Criteria** (Phase 5.4, Phase 14.1) and **indexes** (Phase 4.4) for speed.
- **ETags / `If-Match` (§3)** map to **JPA optimistic locking `@Version`** (Phase 4.5/5.4); Spring's `ShallowEtagHeaderFilter`.
- **Rate limiting (§5)** is implemented at the **API Gateway** (Spring Cloud Gateway + Redis — Phase 12.4) and is a **security** control (Phase 15.1).
- **Async APIs (§6)** are powered by `@Async`/scheduling (Phase 5.3/5.8) and **messaging/Kafka** (Phase 11); observable via tracing (Phase 9.2/12.7).
- **File uploads (§7)** use Spring `MultipartFile` (Phase 5.3) and **cloud storage / pre-signed URLs** (Phase 16.7).
- **Bulk (§8)** ties to **transactions** (Phase 4.5/5.5) and async ingestion (Phase 11).
- Everything here is documented in **OpenAPI (7.2)** and shaped by **DTOs (7.3)**.
- Applied in **Project 5 (E-Commerce Backend)** & **Project 7 (Microservices E-Commerce)**.

---

## 11. Quick Self-Check Questions

1. Why is POST not idempotent, and how do idempotency keys fix double-submission? Where do you store the keys?
2. How do filtering, sorting, and pagination compose as query params? How do you keep them safe & fast?
3. Why does deep offset pagination get slow, and what fixes it?
4. What is an ETag? Walk through a conditional GET (304) and a conditional PUT (412).
5. How do ETags relate to JPA optimistic locking (`@Version`)?
6. Compare JSON Merge Patch vs JSON Patch — when use each? What can't Merge Patch do?
7. Which status + headers do you return when rate-limited? Name a rate-limiting algorithm.
8. Describe the 202 + polling async pattern and where the heavy work actually runs.
9. What's the scalable pattern for very large file uploads, and why?
10. How do you handle partial success in a bulk endpoint?

---

## 12. Key Terms Glossary

- **Idempotency key:** client-supplied unique id (header) for safe POST retries.
- **Filtering / sorting / pagination:** query params to narrow, order, and page collections.
- **Cursor (keyset) pagination:** opaque token using `WHERE id > ?` — index-friendly.
- **ETag:** opaque version id of a representation.
- **Conditional request:** `If-None-Match` (→304) / `If-Match` (→412).
- **Optimistic concurrency:** detect concurrent edits via version/ETag (lost-update prevention).
- **JSON Merge Patch (RFC 7386):** partial object; `null` deletes a field.
- **JSON Patch (RFC 6902):** array of `op`/`path`/`value` operations.
- **Rate limiting:** 429 + `RateLimit-*` / `Retry-After`; token/leaky bucket, sliding window.
- **202 Accepted:** request accepted for async processing (+ job/status resource).
- **Pre-signed URL:** time-limited URL for direct client↔storage transfer.
- **Bulk / 207 Multi-Status:** batch endpoint with per-item outcomes.

---

*This is Part 2 of the note for **Section 7.1 — RESTful API Design Principles**.*
*Previous: **7.1 Part 1 — Resources, Methods, Status & Responses** (`01-...`).*
*Next section in roadmap: **7.2 API Documentation (OpenAPI / SpringDoc)**.*
