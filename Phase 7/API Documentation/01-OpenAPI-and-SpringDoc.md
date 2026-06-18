# API Documentation — OpenAPI & SpringDoc

> **Phase 7 — API Design & Documentation → 7.2 API Documentation**
> Goal: Document REST APIs with the **OpenAPI Specification 3.x**, auto-generate interactive docs using **SpringDoc OpenAPI** (Swagger UI), enrich them with annotations (`@Tag`, `@Operation`, `@Parameter`, `@ApiResponse`, `@Schema`), document security schemes, and export a spec file for clients & contract testing.

---

## 0. The Big Picture

An API is only as useful as it is **understandable**. **OpenAPI** is a language-agnostic, machine-readable **contract** (a YAML/JSON document) describing every endpoint, parameter, request/response schema, status code, and auth scheme of your REST API (designed in 7.1). From that single contract you get: interactive docs (**Swagger UI**), client SDK generation, mock servers, and contract tests.

```
Your Spring controllers (Phase 5.3)
        │  SpringDoc scans them at runtime
        ▼
  OpenAPI document  (/v3/api-docs  → JSON/YAML)   ◄── the machine-readable CONTRACT
        │
        ├──► Swagger UI  (/swagger-ui.html)  — interactive, "try it out" docs
        ├──► Client SDK generation (openapi-generator) — typed clients in any language
        ├──► Mock servers / Postman import
        └──► Contract testing (consumer/provider — Phase 6)
```

> This documents the API designed in **7.1** (resources, methods, status codes, `ProblemDetail`, pagination) and the **DTOs** from **7.3** (which become the schemas). It builds on Spring Web MVC (Phase 5.3) and Spring Security (Phase 5.6 — security schemes).

### Design-first vs code-first
| Approach | How | Pros |
|----------|-----|------|
| **Code-first** ⭐ (SpringDoc default) | Write controllers/DTOs → generate the spec | Fast; spec always matches code |
| **Design-first** | Hand-write the OpenAPI YAML → generate server stubs/clients | Contract agreed before coding; great for teams/parallel work |

> SpringDoc primarily enables **code-first**. Many teams do a hybrid: design-first to agree the contract, then keep it in sync via SpringDoc + checks.

---

## 1. The OpenAPI Specification (3.x)

OpenAPI (formerly **Swagger** spec; **OpenAPI 3.0/3.1** are current) is a structured document with these top-level sections:

| Section | Contains |
|---------|----------|
| `openapi` | Spec version (`3.0.3` / `3.1.0`) |
| `info` | Title, description, version, contact, license |
| `servers` | Base URLs (dev/staging/prod) |
| `paths` | Each endpoint → method → params, requestBody, responses |
| `components` | Reusable **schemas**, parameters, responses, **securitySchemes**, examples |
| `security` | Global security requirements |
| `tags` | Logical groupings of operations |

```yaml
openapi: 3.0.3
info:
  title: Orders API
  version: 1.0.0
  description: Manage customer orders.
servers:
  - url: https://api.example.com/api/v1
paths:
  /orders/{id}:
    get:
      tags: [Orders]
      summary: Get an order by id
      operationId: getOrder
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer, format: int64 }
      responses:
        '200':
          description: The order
          content:
            application/json:
              schema: { $ref: '#/components/schemas/OrderResponse' }
        '404':
          description: Not found
          content:
            application/problem+json:
              schema: { $ref: '#/components/schemas/ProblemDetail' }   # RFC 7807 (7.1)
components:
  schemas:
    OrderResponse:
      type: object
      properties:
        id:     { type: integer, format: int64, example: 42 }
        status: { type: string, enum: [PENDING, PAID, CANCELLED] }
        total:  { type: string, example: "59.90" }
      required: [id, status, total]
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT      # (Phase 5.6)
security:
  - bearerAuth: []
```

> ⭐ **`$ref`** lets you define a schema once in `components` and reference it everywhere (DRY). **OpenAPI 3.1** aligns fully with **JSON Schema** and supports `type: [string, "null"]` for nullables (3.0 used `nullable: true`).

---

## 2. SpringDoc OpenAPI Setup

**SpringDoc** scans your Spring controllers/DTOs at runtime and serves both the OpenAPI JSON and Swagger UI. Add one dependency:

```xml
<!-- Spring Boot 3.x / Spring MVC (Phase 3 Maven) -->
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.6.0</version>
</dependency>
```
> ⚠️ Use the **`springdoc-openapi v2.x`** line for **Spring Boot 3 / Jakarta** (the old `springfox` library is dead and the 1.x line was for Boot 2). For WebFlux (Phase 16.1) use `...-webflux-ui`.

Out of the box you get:
| URL | Serves |
|-----|--------|
| `/v3/api-docs` | OpenAPI document as **JSON** |
| `/v3/api-docs.yaml` | OpenAPI document as **YAML** |
| `/swagger-ui.html` (→ `/swagger-ui/index.html`) | **Swagger UI** (interactive) |

Common config (`application.yml` — Phase 5.2):
```yaml
springdoc:
  swagger-ui:
    path: /swagger-ui.html
    operationsSorter: method
    tagsSorter: alpha
  api-docs:
    path: /v3/api-docs
  packages-to-scan: com.example.api      # limit scanning
  paths-to-match: /api/**
```

> ⚠️ **Security:** in production, **lock down or disable** Swagger UI / `api-docs` (it reveals your whole API surface). Restrict by profile (`@Profile`/`springdoc.api-docs.enabled=false`), network, or auth (Phase 5.6/15). It's a great dev/staging tool but an info-disclosure risk if public unintentionally.

### 2.1 Global metadata bean
```java
@Configuration
class OpenApiConfig {
    @Bean
    OpenAPI apiInfo() {
        return new OpenAPI()
            .info(new Info()
                .title("Orders API")
                .version("1.0.0")
                .description("Manage customer orders.")
                .contact(new Contact().name("Platform Team").email("api@example.com"))
                .license(new License().name("Apache 2.0")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components().addSecuritySchemes("bearerAuth",
                new SecurityScheme().type(SecurityScheme.Type.HTTP)
                    .scheme("bearer").bearerFormat("JWT")));   // (Phase 5.6)
    }
}
```

---

## 3. Documenting Operations with Annotations

SpringDoc infers a lot automatically (paths, methods, params, DTO schemas). Use **`io.swagger.v3.oas.annotations`** annotations to enrich what it can't infer.

```java
@RestController
@RequestMapping("/api/v1/orders")
@Tag(name = "Orders", description = "Create, read, and manage orders")   // group in the UI
class OrderController {

    @Operation(                                                  // describe the operation
        summary = "Get an order by id",
        description = "Returns a single order. Requires the ORDER_READ scope.")
    @ApiResponses({                                             // document each status (7.1)
        @ApiResponse(responseCode = "200", description = "Order found",
            content = @Content(schema = @Schema(implementation = OrderResponse.class))),
        @ApiResponse(responseCode = "404", description = "Order not found",
            content = @Content(mediaType = "application/problem+json",
                schema = @Schema(implementation = ProblemDetail.class)))   // RFC 7807 (7.1)
    })
    @GetMapping("/{id}")
    OrderResponse get(
        @Parameter(description = "Order id", example = "42")     // describe a parameter
        @PathVariable Long id) {
        return service.get(id);
    }
}
```

| Annotation | Where | Purpose |
|------------|-------|---------|
| **`@Tag`** | class/method | Group operations under a heading in the UI |
| **`@Operation`** | method | `summary`, `description`, `operationId` of the endpoint |
| **`@Parameter`** | param | Describe a path/query/header param (`description`, `example`, `required`) |
| **`@ApiResponse(s)`** | method | Document a response status + its schema/media type |
| **`@RequestBody`** (swagger) | param | Describe the request body + examples |
| **`@Content` / `@Schema`** | nested | Tie a response/param to a media type & model |
| **`@Hidden`** | method/class | Exclude from the docs |

> ⭐ Document **every status code** you designed in 7.1 (200/201/400/401/403/404/409/422...) and point error responses at the **`ProblemDetail`** schema. `operationId` should be unique & stable — client generators use it as the method name.

### 3.1 Documenting the DTO schemas (`@Schema`)
Annotate the **DTOs from 7.3** so the generated models are rich:
```java
@Schema(description = "Request to create an order")
public record CreateOrderRequest(
    @Schema(description = "Customer id", example = "42", requiredMode = REQUIRED)
    @NotNull Long customerId,                                  // Bean Validation (Phase 5.3)

    @Schema(description = "Line items", minLength = 1)
    @NotEmpty List<OrderItemDto> items
) {}
```
> ⭐ **Bean Validation annotations are read by SpringDoc** — `@NotNull`, `@Size`, `@Min/@Max`, `@Pattern`, `@Email` (Phase 5.3) automatically become schema constraints (`required`, `minLength`, `minimum`, `pattern`...) in the spec. So validating your input *also* documents it. Enums become `enum` lists; `@Schema(example=...)` adds examples that show up in Swagger UI's "try it out".

---

## 4. Documenting Security Schemes

Tell consumers (and Swagger UI's "Authorize" button) how to authenticate. Common schemes for backends (Phase 5.6):

```java
@SecurityScheme(name = "bearerAuth", type = SecuritySchemeType.HTTP,
                scheme = "bearer", bearerFormat = "JWT")        // JWT bearer (Phase 5.6)
@Configuration
class SecurityDocsConfig {}

// Apply per-operation or globally:
@Operation(security = { @SecurityRequirement(name = "bearerAuth") })
@GetMapping("/{id}") OrderResponse get(...) { ... }
```

| Scheme `type` | For |
|---------------|-----|
| `http` + `scheme: bearer` (`bearerFormat: JWT`) | **JWT/bearer tokens** (Phase 5.6) |
| `http` + `scheme: basic` | HTTP Basic auth |
| `apiKey` (in header/query/cookie) | **API keys** (Phase 15.2) |
| `oauth2` (flows: authorizationCode, clientCredentials...) | **OAuth2/OIDC** (Phase 5.6) |
| `openIdConnect` | OIDC discovery URL |

> ⭐ Once declared, Swagger UI shows an **"Authorize"** button — paste a token there and the UI sends `Authorization: Bearer ...` on "try it out" calls, so you can test secured endpoints (Phase 5.6) live.

---

## 5. Generating & Using the Spec File

The OpenAPI document is an artifact you can **export, version, and share**.

```bash
# Fetch the live spec (app running) — JSON or YAML:
curl http://localhost:8080/v3/api-docs       > openapi.json
curl http://localhost:8080/v3/api-docs.yaml  > openapi.yaml
```

**Maven build-time generation** (no manual curl; great for CI — Phase 10.3):
```xml
<plugin>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-maven-plugin</artifactId>  <!-- starts the app, dumps the spec -->
</plugin>
```

What you do with the spec file:
| Use | Tool |
|-----|------|
| **Generate typed clients/SDKs** (Java, TS, Python...) | `openapi-generator-cli` |
| **Generate server stubs** (design-first) | `openapi-generator` |
| **Mock server** | Prism, Microcks |
| **Import to API platform** | Postman, Insomnia, API gateways (Phase 12.4) |
| **Contract / diff testing in CI** | `openapi-diff` (fail the build on breaking changes — 7.1 §8 versioning) |
| **Publish docs portal** | Redoc, Stoplight |

> ⭐ **Commit the generated `openapi.yaml` and diff it in CI** — a build that changes the contract in a breaking way (removed field/endpoint) can **fail the pipeline** (Phase 10.3), enforcing the versioning discipline from 7.1 §8. This turns "documentation" into an enforced **contract**.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using dead `springfox` / SpringDoc 1.x on Boot 3 | Use `springdoc-openapi v2.x` (Jakarta) |
| Swagger UI exposed publicly in prod | Disable/secure it by profile/network/auth (§2) |
| Documenting only the 200 case | Document all status codes + `ProblemDetail` errors (7.1) |
| Hand-writing constraints already in `@Valid` | Reuse Bean Validation annotations — SpringDoc reads them (§3.1) |
| No examples → empty "try it out" | Add `@Schema(example=...)`/`example` on params |
| Spec drifts from code | Code-first + generate spec in CI; diff for breaking changes |
| Exposing entities as schemas | Document **DTOs** (7.3), not JPA entities |
| Forgetting `operationId` stability | Set stable `operationId` (client method names depend on it) |
| Not documenting auth | Declare `securitySchemes` so "Authorize" works (§4) |
| Duplicated inline schemas | Reuse via `components` + `$ref` |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Documents 7.1's design** (URIs, methods, status codes, `ProblemDetail`, pagination) and **7.3's DTOs** (which become schemas).
- **Bean Validation (Phase 5.3)** annotations auto-enrich schemas (§3.1) — validate once, document for free.
- **Spring Security (Phase 5.6)** → security schemes (JWT/OAuth2/API keys) + Swagger "Authorize".
- **CI/CD (Phase 10.3)** generates the spec and runs **contract/diff checks** to guard against breaking changes (7.1 §8).
- **API Gateway (Phase 12.4)** can import the spec for routing/validation; **client SDKs** are generated for inter-service calls (Phase 12.2 — Feign/WebClient).
- **Testing (Phase 6):** the spec underpins **contract testing** (consumer-driven).
- Applied in **Project 4 (Task Management REST API)** & **Project 5/7 (E-Commerce)** — "document the API with OpenAPI/Swagger".

---

## 8. Quick Self-Check Questions

1. What is OpenAPI, and what artifacts can you derive from one spec document?
2. What are the top-level sections of an OpenAPI 3.x document?
3. Which SpringDoc dependency is correct for Spring Boot 3, and what URLs does it expose?
4. Why and how should you secure/disable Swagger UI in production?
5. What do `@Tag`, `@Operation`, `@Parameter`, `@ApiResponse`, and `@Schema` each do?
6. How do Bean Validation annotations relate to the generated schema?
7. How do you document a JWT bearer security scheme so "Authorize" works?
8. How do you export the spec, and what can you do with the file in CI?
9. Code-first vs design-first — what's the difference and which does SpringDoc favor?
10. How can the spec enforce the versioning discipline from 7.1?

---

## 9. Key Terms Glossary

- **OpenAPI Specification (OAS 3.0/3.1):** machine-readable API contract (YAML/JSON).
- **Swagger / Swagger UI:** the original name of the spec / the interactive docs UI.
- **SpringDoc OpenAPI:** library that generates OpenAPI + Swagger UI from Spring code.
- **`/v3/api-docs` / `/swagger-ui.html`:** spec endpoint / interactive UI.
- **`components` / `$ref`:** reusable definitions / references (DRY schemas).
- **`@Tag` / `@Operation` / `@Parameter` / `@ApiResponse` / `@Schema`:** doc enrichment annotations.
- **securityScheme:** declared auth method (bearer JWT, basic, apiKey, oauth2).
- **operationId:** unique stable id for an operation (client method name).
- **Code-first / design-first:** generate spec from code vs write spec first.
- **openapi-generator:** generates clients/server stubs from a spec.
- **Contract / diff testing:** fail CI on breaking spec changes.

---

*This is the note for **Section 7.2 — API Documentation (OpenAPI / SpringDoc)**.*
*Previous section in roadmap: **7.1 RESTful API Design Principles**.*
*Next section in roadmap: **7.3 DTO Pattern and MapStruct**.*
