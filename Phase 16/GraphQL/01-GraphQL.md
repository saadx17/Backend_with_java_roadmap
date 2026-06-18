# GraphQL

> **Phase 16 — Advanced Topics → 16.2 GraphQL**
> Goal: Understand GraphQL — its query language and schema, how it differs from REST, queries/mutations/subscriptions, resolvers, the N+1 problem (DataLoader), and Spring for GraphQL.

---

## 0. The Big Picture

**GraphQL** is a query language and runtime for APIs where the **client specifies exactly what data it needs** in a single request, against a strongly-typed **schema**. It addresses two REST pain points (Phase 7): **over-fetching** (getting fields you don't need) and **under-fetching** (needing many round-trips to assemble a screen).

```
   REST:  GET /users/1  +  GET /users/1/orders  +  GET /orders/*/items   (multiple round-trips)
   GraphQL:  POST /graphql  { user(id:1){ name orders{ total items{ name } } } }   (ONE request, exact shape)
```

> An alternative/complement to REST (Phase 7) and gRPC (12.2). Built on a typed schema (like OpenAPI — 7.2). Integrated via **Spring for GraphQL**; shares concerns with DTOs (7.3), N+1 (5.4/13.1), security (15), and can be reactive (16.1).

---

## 1. GraphQL vs REST ⭐

| | **REST** (Phase 7) | **GraphQL** |
|--|--------------------|-------------|
| Endpoints | Many (one per resource) | **One** (`/graphql`) |
| Data shape | Server-defined (fixed) | **Client-defined** (exact fields) |
| Over/under-fetching | Common | Avoided |
| Versioning | URL/header (7.1 §8) | Evolve schema (deprecate fields) |
| Caching | HTTP/ETags easy (7.1 §3) | Harder (POST, single endpoint) |
| Learning curve | Low | Higher |
| File upload / simple CRUD | Natural | Awkward |
| Best for | Simple/public APIs, caching | Complex/flexible client needs, aggregating data, mobile |
> ⭐ **GraphQL excels when diverse clients need different data shapes** (mobile vs web), or you aggregate many sources (a BFF — 12.2). ⚠️ It's **not a REST replacement** — REST is simpler, caches better (CDN/ETags), and fits simple CRUD. Many systems use both. GraphQL also moves complexity to the server (resolvers, N+1, security).

---

## 2. The Schema (SDL)

GraphQL is **schema-first**: you define types, queries, mutations in **SDL** (Schema Definition Language) — a strongly-typed contract (like OpenAPI — 7.2).
```graphql
type User {
  id: ID!                       # ! = non-null
  name: String!
  email: String!
  orders: [Order!]!            # a list of Orders (relationship)
}
type Order { id: ID!  total: Float!  items: [Item!]! }
type Item { name: String!  price: Float! }

type Query {                    # read operations
  user(id: ID!): User
  orders(status: String): [Order!]!
}
type Mutation {                 # write operations
  placeOrder(input: PlaceOrderInput!): Order!
}
type Subscription {             # real-time stream (over WebSocket — 16.3)
  orderUpdated(id: ID!): Order!
}
```
| Element | Meaning |
|---------|---------|
| **Type** | An object with typed fields (maps to your domain — 12.1) |
| **Scalars** | `Int`, `Float`, `String`, `Boolean`, `ID` (+ custom) |
| **`!` / `[]`** | Non-null / list |
| **Query / Mutation / Subscription** | The three root operation types |
| **Input type** | Argument object for mutations |

---

## 3. Queries, Mutations, Subscriptions

| Operation | Purpose | REST analogy |
|-----------|---------|--------------|
| **Query** ⭐ | Read data (you pick fields) | GET |
| **Mutation** | Create/update/delete | POST/PUT/PATCH/DELETE |
| **Subscription** | Real-time push stream | WebSocket/SSE (16.3) |
```graphql
# Query — request exactly these fields
query { user(id: "1") { name orders { total } } }

# Mutation
mutation { placeOrder(input: { customerId: "1", items: [...] }) { id status } }
```
> ⭐ One **query** can fetch a deeply nested graph in a single round-trip (the headline benefit). **Subscriptions** push updates over WebSocket (16.3). All go to the **single `/graphql` endpoint** via POST.

---

## 4. Resolvers & the N+1 Problem ⭐

A **resolver** is the server function that fetches the data for a field. Spring for GraphQL maps schema fields to controller methods.
```java
@Controller
class UserGraphQlController {
    @QueryMapping
    User user(@Argument String id) { return userService.find(id); }       // resolves Query.user

    @SchemaMapping(typeName = "User", field = "orders")                     // resolves User.orders
    List<Order> orders(User user) { return orderService.findByUser(user.getId()); }
}
```
### 4.1 The N+1 problem ⭐ (GraphQL's classic trap)
If a query asks for 100 users and each user's orders, a naive resolver runs **1 query for users + 100 queries for orders** = N+1 (recall JPA N+1 — 5.4/13.1).
```
   query { users { orders { ... } } }   →  1 (users) + N (orders per user)  💥
```
**Fix: DataLoader** — batches and caches per-request: collect all the order requests, fetch them in **one** batched query.
```java
@BatchMapping                          // Spring for GraphQL batches automatically
Map<User, List<Order>> orders(List<User> users) {
    return orderService.findByUsers(users);   // ONE query for all users' orders
}
```
> ⭐ **N+1 is the #1 GraphQL performance pitfall** because clients can request arbitrary nesting. **DataLoader / `@BatchMapping`** batches resolver calls within a request → turns N+1 into 1+1. Always batch nested resolvers. (Same root cause and fix mindset as JPA N+1 — 5.4/13.1.)

---

## 5. Spring for GraphQL & Security Concerns

### 5.1 Spring for GraphQL
```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-graphql</artifactId></dependency>
```
- Schema in `src/main/resources/graphql/*.graphqls`; `@QueryMapping`/`@MutationMapping`/`@SchemaMapping`/`@BatchMapping` controllers; built-in **GraphiQL** UI for exploring; works with WebMVC or **WebFlux** (16.1).

### 5.2 GraphQL-specific security ⚠️
| Concern | Mitigation |
|---------|-----------|
| **Malicious deep/complex queries** ⭐ | Limit query **depth** & **complexity**; reject expensive queries (DoS — 15.1) |
| **N+1 as a DoS vector** | Batching + complexity limits (§4) |
| **Introspection exposure** | Disable schema introspection in production (info disclosure — 15.1) |
| **Authorization per field** | Enforce authZ at resolver/field level (15.2 — not just at the endpoint) |
| **Rate limiting** | Harder than REST (one endpoint) — limit by query cost (7.1 §5) |
| **Errors** | Don't leak internals in GraphQL error responses (15.1) |
> ⚠️ ⭐ GraphQL's flexibility is also a **security risk**: a client can craft a deeply nested, expensive query to DoS you. **Always enforce query depth/complexity limits**, disable introspection in prod, and do **field-level authorization** (15.2). The single endpoint also complicates per-operation rate limiting (12.4).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Treating GraphQL as a REST replacement | Use where flexible data shapes help; REST elsewhere (§1) |
| Naive resolvers → N+1 | DataLoader / `@BatchMapping` (§4) |
| No query depth/complexity limits | Enforce limits (DoS) (§5.2) |
| Introspection enabled in prod | Disable it (§5.2) |
| AuthZ only at the endpoint | Field/resolver-level authorization (§5.2, 15.2) |
| Expecting easy HTTP caching | Caching is harder; use persisted queries/app caches (§1) |
| Over-fetching via lazy JPA in resolvers | Fetch what's needed; batch (§4, 5.4) |
| Ignoring error/format handling | Structured GraphQL errors, no leaks (§5.2) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- Alternative/complement to **REST** (Phase 7) and **gRPC** (12.2); a typed contract like **OpenAPI** (7.2).
- **Spring for GraphQL**; can run reactive on **WebFlux** (16.1); **subscriptions** over **WebSocket** (16.3).
- **N+1** mirrors JPA N+1 (5.4) and performance (13.1); maps to domain types (12.1) and DTOs (7.3).
- **Security**: depth/complexity limits, field authZ, introspection (15.1/15.2); rate limiting (7.1/12.4).
- Optional/advanced; could feature in **Project 5/7** for flexible client APIs.

---

## 8. Quick Self-Check Questions

1. What two REST problems does GraphQL address, and how?
2. Compare GraphQL and REST — when use each?
3. What is the SDL schema, and what are the three root operation types?
4. What's the difference between a query, mutation, and subscription?
5. What is a resolver, and how does Spring for GraphQL map them?
6. What is the GraphQL N+1 problem, and how does DataLoader/`@BatchMapping` fix it?
7. Why is GraphQL's flexibility a security/DoS risk, and what limits help?
8. Why disable introspection in production?
9. Why is authorization needed at the field level, not just the endpoint?
10. Why is HTTP caching harder with GraphQL?

---

## 9. Key Terms Glossary

- **GraphQL:** client-specified query language + runtime over a typed schema.
- **Schema / SDL:** the typed contract / its definition language.
- **Query / Mutation / Subscription:** read / write / real-time operations.
- **Resolver:** server function that fetches a field's data.
- **Over-fetching / under-fetching:** too much data / too many round-trips (REST issues).
- **N+1 problem / DataLoader / `@BatchMapping`:** nested-resolver explosion / batching fix.
- **Introspection:** querying the schema itself (disable in prod).
- **Query depth/complexity limits:** DoS protection.
- **Spring for GraphQL / GraphiQL:** Spring integration / explorer UI.

---

*This is the note for **Section 16.2 — GraphQL**.*
*Previous section in roadmap: **16.1 Reactive Programming**.*
*Next section in roadmap: **16.3 WebSocket**.*
