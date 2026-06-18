# DTO Pattern & MapStruct

> **Phase 7 — API Design & Documentation → 7.3 DTO Pattern and MapStruct**
> Goal: Understand *why* DTOs exist and how to separate **Request** vs **Response** DTOs from entities, then map between layers cleanly with **MapStruct** — `@Mapper`, field mapping, nested objects, `@AfterMapping`/`@BeforeMapping`, and Spring integration.

---

## 0. The Big Picture

A **DTO (Data Transfer Object)** is a simple, purpose-built object that carries data **across a boundary** — here, between your **API layer** (the JSON contract from 7.1/7.2) and your **domain/persistence layer** (JPA entities — Phase 5.4). DTOs are the public *shape* of your API; entities are your internal *storage* model. **They must be decoupled.**

```
 HTTP request (JSON)                                    HTTP response (JSON)
        │                                                       ▲
        ▼                                                       │
  CreateOrderRequest (DTO) ──map──► Order (entity) ──map──► OrderResponse (DTO)
        │                              │                         ▲
   validated (Phase 5.3)         persisted via JPA          built from entity
                                  (Phase 5.4 / DB Phase 4)
                                       MapStruct does the mapping (this note)
```

> This is the glue between **7.1** (API design), **7.2** (where DTOs become OpenAPI schemas), and the persistence layers (Phase 5.4 entities, Phase 4 DB). It applies the separation-of-concerns and layered-architecture ideas you'll formalize in Phase 14.

---

## 1. Why DTOs? (Never Expose Entities)

Returning/accepting JPA **entities** directly is one of the most common backend mistakes. DTOs solve real problems:

| Problem with exposing entities | How DTOs fix it |
|--------------------------------|-----------------|
| **Over-exposure / security** — leaks `passwordHash`, internal flags, audit fields (Phase 15) | DTO exposes only intended fields |
| **Tight coupling** — DB schema changes break the API contract (7.1 §8) | DTO insulates the contract from the schema |
| **Lazy-loading / serialization issues** — `LazyInitializationException`, infinite recursion on bidirectional relations (Phase 5.4) | DTO is a flat, detached, serialization-safe object |
| **Mass-assignment vulnerability** — client sets fields it shouldn't (e.g., `role`, `id`) (Phase 15.1) | Request DTO only includes editable fields |
| **Mismatch of shapes** — API needs computed/combined/renamed fields | DTO can flatten, rename, compute, aggregate |
| **Validation rules differ from persistence** | Validate the Request DTO (Phase 5.3) independently |
| **Different views for different consumers** | Multiple DTOs per entity (summary vs detail) |

> ⭐ **Rule: the web layer speaks DTOs; only the persistence layer touches entities.** This is the single most important habit for a maintainable, secure Spring backend. (The `@Schema` you added in 7.2 documents the DTO, not the entity.)

---

## 2. Request DTO vs Response DTO

Use **separate DTOs for input and output** — they have different fields, validation, and lifecycles.

```java
// REQUEST DTO — what the client may SEND (input). Validated (Phase 5.3).
// Note: NO id (server assigns), NO audit/createdAt, NO status (server controls).
public record CreateOrderRequest(
    @NotNull Long customerId,
    @NotEmpty @Valid List<OrderItemRequest> items
) {}

// RESPONSE DTO — what the client RECEIVES (output).
// Includes server-controlled fields: id, status, timestamps, computed total, links.
public record OrderResponse(
    Long id,
    Long customerId,
    String status,
    String total,
    Instant createdAt,
    List<OrderItemResponse> items
) {}
```

| Aspect | Request DTO | Response DTO |
|--------|-------------|--------------|
| Direction | Client → server (input) | Server → client (output) |
| Validation | ✅ `@NotNull`, `@Size`... (Phase 5.3) | ❌ (not validated) |
| Server-managed fields (`id`, `createdAt`, `status`) | ❌ omit (prevent mass-assignment) | ✅ include |
| Sensitive fields (`passwordHash`) | maybe `password` (write-only) | ❌ never |
| Often used for | POST/PUT/PATCH bodies | GET responses & 201/200 bodies |

> ⭐ **Java `record`s are ideal for DTOs** (Phase 1.3) — immutable, concise, value-based `equals`/`hashCode`. Consider distinct DTOs per use case: `OrderSummaryResponse` (list view) vs `OrderDetailResponse` (detail view) to avoid over-fetching. ⚠️ Never reuse one "god DTO" for everything — that re-creates entity-exposure problems.

---

## 3. The Mapping Problem → MapStruct

You constantly convert entity ↔ DTO. Doing it by hand is tedious and error-prone:
```java
// Manual mapping — boilerplate, easy to forget a field, runtime-only:
OrderResponse dto = new OrderResponse(
    order.getId(), order.getCustomer().getId(), order.getStatus().name(),
    order.getTotal().toPlainString(), order.getCreatedAt(),
    order.getItems().stream().map(this::toItemDto).toList());
```

**MapStruct** generates this mapping code **at compile time** from interfaces you declare — type-safe, fast (plain method calls, **no reflection**), and verified by the compiler.

| Mapper option | Mechanism | Speed | Safety |
|---------------|-----------|-------|--------|
| **Manual** | Hand-written | Fast | Error-prone, verbose |
| **MapStruct** ⭐ | **Compile-time** generated code | **Fastest** (plain code) | Compiler-checked; unmapped fields warned |
| ModelMapper / Dozer | **Runtime reflection** | Slower | Errors only at runtime |

> ⭐ MapStruct wins because it's **compile-time**: if a field can't be mapped, you get a **build warning/error** (not a production NPE), and there's **zero reflection overhead** (Phase 1.14 — reflection is slow). Generated code lives in `target/generated-sources`.

### 3.1 Setup (Maven — Phase 3)
```xml
<dependency>
  <groupId>org.mapstruct</groupId>
  <artifactId>mapstruct</artifactId>
  <version>1.6.2</version>
</dependency>
<!-- annotation processor that GENERATES the impl at compile time -->
<build><plugins><plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <annotationProcessorPaths>
      <path><groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>1.6.2</version></path>
      <!-- ⚠️ if using Lombok, order matters: lombok, then lombok-mapstruct-binding, then mapstruct -->
    </annotationProcessorPaths>
  </configuration>
</plugin></plugins></build>
```
> ⚠️ **Lombok + MapStruct gotcha:** MapStruct must run *after* Lombok generates getters/setters. Add `lombok-mapstruct-binding` and order the processors correctly, or fields appear "unmapped". (Records sidestep much of this.)

---

## 4. `@Mapper` & Basic Field Mapping

Declare an interface; MapStruct generates the implementation.
```java
@Mapper(componentModel = "spring")        // generate a Spring @Component (§7)
public interface OrderMapper {

    OrderResponse toResponse(Order entity);          // entity -> response DTO
    Order toEntity(CreateOrderRequest request);       // request DTO -> entity

    List<OrderResponse> toResponseList(List<Order> entities);   // collections handled automatically
}
```
MapStruct **maps fields with matching names/types automatically** and converts compatible types (e.g., `int`↔`Integer`, `String`↔enum, dates).

### 4.1 Renamed / mismatched fields → `@Mapping`
```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    @Mapping(source = "customer.id",   target = "customerId")    // flatten nested → flat
    @Mapping(source = "createdInstant",target = "createdAt")     // rename
    @Mapping(target = "total", expression = "java(entity.getTotal().toPlainString())")  // custom expr
    @Mapping(target = "internalNotes", ignore = true)            // explicitly skip a field
    OrderResponse toResponse(Order entity);

    @Mapping(target = "id", ignore = true)            // server assigns id (don't map from request)
    @Mapping(target = "status", constant = "PENDING") // set a constant
    @Mapping(target = "createdAt", expression = "java(java.time.Instant.now())")
    Order toEntity(CreateOrderRequest request);
}
```
| `@Mapping` attribute | Use |
|----------------------|-----|
| `source` / `target` | Map between differently-named fields |
| `source="a.b.c"` | **Flatten** a nested property |
| `constant` | Set a literal value |
| `expression="java(...)"` | Inline Java for computed values |
| `defaultValue` / `defaultExpression` | Fallback when source is null |
| `ignore = true` | Don't map this target (e.g., server-controlled `id`) |
| `dateFormat` / `numberFormat` | Format conversions |
| `qualifiedByName` | Use a specific named helper method |

> ⭐ Set `@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR)` to **fail the build** if any target field is left unmapped — turning forgotten fields into compile errors (great safety net, complements 7.2's contract checks).

---

## 5. Nested Objects & Type Conversions

MapStruct maps nested DTOs/entities automatically **if it can find (or generate) a sub-mapper**.
```java
@Mapper(componentModel = "spring",
        uses = { OrderItemMapper.class })       // reuse another mapper for nested types
public interface OrderMapper {
    OrderResponse toResponse(Order entity);     // items: List<OrderItem> -> List<OrderItemResponse>
}                                               // delegated to OrderItemMapper automatically
```
- **Collections** (`List`/`Set`/`Map`) are mapped element-by-element automatically.
- **Enums** map by name; customize with `@ValueMapping`/`@EnumMapping`.
- **Custom type conversions** (e.g., `Instant` ↔ `String`, `BigDecimal` ↔ `String`) via a method MapStruct discovers, or `@Mapping(... dateFormat=...)`.
- Use `uses = {...}` to compose mappers (one per aggregate — keeps mappers small).

---

## 6. `@BeforeMapping` & `@AfterMapping` (Hooks)

When automatic mapping isn't enough (post-processing, conditional logic, enriching the target), use lifecycle hooks.
```java
@Mapper(componentModel = "spring")
public abstract class OrderMapper {          // abstract class (so you can add helper methods/state)

    public abstract OrderResponse toResponse(Order entity);

    @AfterMapping                            // runs AFTER the generated field mapping
    protected void addComputedFields(Order entity, @MappingTarget OrderResponse.OrderResponseBuilder dto) {
        dto.itemCount(entity.getItems().size());        // enrich the target
    }

    @BeforeMapping                           // runs BEFORE mapping (e.g., normalize/guard)
    protected void normalize(CreateOrderRequest request) {
        // pre-processing / validation hooks
    }
}
```
| Hook | When it runs | Typical use |
|------|--------------|-------------|
| **`@BeforeMapping`** | Before field mapping | Normalize/validate source, init target |
| **`@AfterMapping`** | After field mapping | Compute derived fields, enrich, post-process |
| **`@MappingTarget`** | param annotation | The instance being populated (or its builder) |

> ⭐ Use `@AfterMapping` for **derived/computed** response fields (counts, formatted labels, HATEOAS links — 7.1 §6) that aren't simple field copies. Declare the mapper as an **abstract class** (instead of interface) when you need helper methods or injected dependencies.

---

## 7. Spring Integration

`@Mapper(componentModel = "spring")` makes the generated implementation a **Spring `@Component`**, so you inject it like any bean (Phase 5.1 — DI):
```java
@Service
@RequiredArgsConstructor                       // constructor injection (Phase 5.1)
class OrderService {
    private final OrderRepository repo;        // Phase 5.4
    private final OrderMapper mapper;          // injected MapStruct bean

    OrderResponse create(CreateOrderRequest req) {
        Order entity = mapper.toEntity(req);    // DTO -> entity
        Order saved  = repo.save(entity);       // persist (Phase 5.4 / Phase 4)
        return mapper.toResponse(saved);        // entity -> response DTO
    }
}
```
- The mapper can itself **inject other Spring beans** (e.g., a service to resolve a `customerId` → `Customer`) via `@Mapping(... qualifiedByName=...)` + an injected helper (use abstract-class mapper).
- ⚠️ Where mapping belongs: keep it at the **service** or a dedicated mapping step — controllers receive/return DTOs (7.1); services orchestrate entity ↔ DTO via the mapper. Don't leak entities out of the service layer.

---

## 8. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Returning/accepting JPA entities directly | Always use DTOs (§1) |
| One "god DTO" for input + output | Separate Request vs Response DTOs (§2) |
| Server-managed fields in Request DTO (mass-assignment) | Omit `id`/`status`/`role`; `ignore` on map (§2/§4, Phase 15.1) |
| Using runtime mappers (ModelMapper) in hot paths | Prefer compile-time MapStruct (no reflection) (§3) |
| Lombok fields "unmapped" by MapStruct | Add `lombok-mapstruct-binding`; order processors (§3.1) |
| Silently dropping fields | `unmappedTargetPolicy = ERROR` (§4) |
| `LazyInitializationException` mapping lazy relations | Fetch needed data (join fetch/EntityGraph — Phase 5.4) before mapping |
| Mapping inside the controller | Map in the service; controller stays DTO-only (§7) |
| Reusing entity validation for API input | Validate the Request DTO (Phase 5.3) |
| Bidirectional-relation infinite recursion in JSON | DTOs break the cycle (flat shape) (§1) |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **DTOs are the API contract** designed in **7.1** and documented (`@Schema`) in **7.2**.
- **Request DTO validation** uses Bean Validation (Phase 5.3) → 400/422 + `ProblemDetail` (7.1).
- **Entities & relationships** (Phase 5.4) are the mapping *source/target*; DTOs avoid lazy-loading/serialization traps.
- **Security (Phase 15.1)** — DTOs prevent over-exposure & mass-assignment.
- **MapStruct** is compile-time (contrast reflection — Phase 1.14) and a Spring bean (DI — Phase 5.1).
- **Layered/Hexagonal architecture (Phase 14.4)** formalizes this boundary (the DTO/mapper is the adapter edge).
- **Microservices (Phase 12.2):** DTOs are also the wire format for inter-service REST/gRPC; generated clients (7.2) use them.
- Applied across **Projects 4, 5, 7** (Task Management & E-Commerce backends).

---

## 10. Quick Self-Check Questions

1. What is a DTO, and what boundary does it sit on?
2. List four concrete problems caused by exposing JPA entities directly.
3. Why separate Request and Response DTOs? What fields belong in each?
4. How does omitting fields from a Request DTO prevent the mass-assignment vulnerability?
5. Why is MapStruct preferred over ModelMapper/Dozer? What does "compile-time" buy you?
6. How does `@Mapping` handle renamed fields, nested flattening, constants, and ignored fields?
7. How are nested objects and collections mapped? What does `uses` do?
8. When would you use `@BeforeMapping` vs `@AfterMapping`?
9. What does `componentModel = "spring"` do, and where should mapping happen?
10. What is the Lombok + MapStruct ordering gotcha, and how do you catch unmapped fields at build time?

---

## 11. Key Terms Glossary

- **DTO (Data Transfer Object):** purpose-built object carrying data across a boundary.
- **Request DTO / Response DTO:** validated input model / output model.
- **Mass-assignment:** client setting fields it shouldn't (mitigated by minimal Request DTOs).
- **MapStruct:** compile-time, type-safe bean-mapping code generator.
- **`@Mapper`:** declares a mapper interface/abstract class (`componentModel="spring"`).
- **`@Mapping`:** field-level mapping config (`source`/`target`/`constant`/`expression`/`ignore`).
- **`uses`:** compose/reuse other mappers for nested types.
- **`@BeforeMapping` / `@AfterMapping` / `@MappingTarget`:** lifecycle hooks / the target being populated.
- **`unmappedTargetPolicy`:** warn/error on unmapped fields.
- **Annotation processor:** compiler plugin that generates the mapper impl (no runtime reflection).

---

*This is the note for **Section 7.3 — DTO Pattern and MapStruct**.*
*Previous section in roadmap: **7.2 API Documentation (OpenAPI / SpringDoc)**.*
*This completes **Phase 7 — API Design & Documentation**.*
*Next: **Phase 8 — Database Migration (Flyway / Liquibase)**.*
