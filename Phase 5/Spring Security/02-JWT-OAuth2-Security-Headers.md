# Spring Security: JWT, OAuth2 & Security Headers

> **Phase 5 — Spring Framework & Spring Boot → 5.6 Spring Security**
> Goal: Master JWT authentication (structure, access/refresh tokens, custom filter, revocation), OAuth 2.0 / OpenID Connect, CSRF, security headers, and getting the current user.

---

## 0. The Big Picture

Modern APIs are usually **stateless** — instead of server-side sessions (recall Phase 0.2), the client carries a **token** proving its identity on every request. **JWT** is the dominant token format; **OAuth 2.0** is the framework for delegated authorization (e.g., "Login with Google"). This note builds on the authentication/authorization core (previous note).

```
Login -> server issues a JWT -> client sends it in Authorization: Bearer <token> on each request
Server validates the token (no session lookup needed) -> stateless
```

---

## 1. Stateless Auth: Sessions vs Tokens (recall Phase 0.2)

| | **Session-based** | **Token-based (JWT)** |
|---|-------------------|------------------------|
| State | Server stores session; client holds a `sessionId` cookie | Client holds a self-contained token; server stores **nothing** |
| Scaling | Needs sticky sessions or a shared store (Redis — Phase 4.8) | **Stateless** → any server handles any request |
| Mechanism | Cookie | `Authorization: Bearer <token>` header |
| Revocation | Easy (delete the session) | Harder (token is valid until expiry — §4) |
> **Stateless token auth** scales horizontally (no shared session state — recall Phase 0.2 LB statelessness, Phase 13) → the default for REST APIs and microservices. The trade-off is **revocation** is harder (§4).

---

## 2. JWT Structure

A **JWT (JSON Web Token)** is a self-contained, **signed** token with three Base64-encoded parts separated by dots:
```
eyJhbGc...header.eyJzdWI...payload.SflKxw...signature
       ^header        ^payload         ^signature
```
| Part | Contains |
|------|----------|
| **Header** | Algorithm & token type (`{"alg":"HS256","typ":"JWT"}`) |
| **Payload (claims)** | Data: `sub` (user), `roles`, `exp` (expiry), `iat` (issued-at), custom claims |
| **Signature** | A cryptographic signature over header+payload, using a secret/key |
```json
// Payload (claims) example:
{ "sub": "alice@x.com", "roles": ["USER"], "exp": 1735689600, "iat": 1735686000 }
```
> ⚠️ **The payload is signed, NOT encrypted** — anyone can decode and read it (Base64 is not encryption!). **Never put secrets in a JWT payload.** The **signature** lets the server verify the token wasn't tampered with (using its secret key — recall asymmetric/symmetric crypto, Phase 0.2 TLS). If the signature is valid, the server trusts the claims **without a database lookup** (that's what makes it stateless).

---

## 3. Access & Refresh Token Flow

Two tokens balance security and usability:
| Token | Lifetime | Purpose |
|-------|----------|---------|
| **Access token** | **Short** (e.g., 15 min) | Sent on every request; grants access |
| **Refresh token** | **Long** (e.g., 7 days) | Used to get a new access token (not sent on normal requests) |
```
1. Login -> server returns { accessToken (15min), refreshToken (7days) }
2. Client sends the ACCESS token on each request (Authorization: Bearer ...)
3. Access token expires (401) -> client sends the REFRESH token to /refresh
4. Server validates the refresh token -> issues a new access token
5. (Logout / compromise -> revoke the refresh token)
```
> ⭐ **Short-lived access tokens limit damage if stolen** (it expires fast); the **long-lived refresh token** (stored more securely, e.g., httpOnly cookie) gets new access tokens without re-login. This is the standard pattern (recall short-lived-token best practice, Phase 0.2 cookies/HTTPS).

---

## 4. Custom JWT Filter

Spring doesn't include JWT out of the box — you add a **custom filter** to the security chain (recall filters, Phase 5.3; the filter chain, previous note):
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {   // runs once per request (Phase 5.3)
    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            if (jwtService.isValid(token)) {                 // verify signature + expiry
                var auth = jwtService.toAuthentication(token);  // build Authentication from claims
                SecurityContextHolder.getContext().setAuthentication(auth);  // mark as authenticated
            }
        }
        chain.doFilter(req, res);                            // continue the chain
    }
}
// Register it in the SecurityFilterChain (before the username/password filter):
// http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```
> The flow: extract the `Bearer` token → validate its signature & expiry → set the authenticated user in the `SecurityContextHolder` (the ThreadLocal — previous note). After this, the URL/method authorization rules (previous note) apply normally. Libraries like **jjwt** or **Nimbus** handle token creation/validation.

---

## 5. Token Revocation Strategies

> ⚠️ **JWTs can't be "deleted" — they're valid until they expire.** This is the main JWT downside (vs sessions). Strategies:
| Strategy | How |
|----------|-----|
| **Short expiry** | Keep access tokens short-lived (limits the window) |
| **Blacklist** | Store revoked token IDs in Redis (Phase 4.8) until they expire |
| **Refresh-token rotation** | Invalidate the refresh token on use/logout |
| **Token versioning** | Include a version claim; bump it server-side to invalidate all of a user's tokens |
> A pure stateless JWT can't be revoked before expiry — so for logout/compromise you need a **blacklist** (Redis — Phase 4.8) or short expiry + refresh rotation. (Project 4: "token revocation strategies" + "scheduled cleanup of expired tokens" — Phase 5.8.)

---

## 6. OAuth 2.0 & OpenID Connect

**OAuth 2.0** is a framework for **delegated authorization** — letting an app access resources on a user's behalf **without** handling their password (e.g., "Login with Google/GitHub"). **OpenID Connect (OIDC)** adds an **identity** layer (authentication) on top of OAuth2.
| | OAuth 2.0 | OpenID Connect |
|---|-----------|----------------|
| Purpose | Authorization (access to resources) | Authentication (who the user is) |
| Adds | — | An `id_token` (a JWT with identity claims) |

### 6.1 Grant types (flows)
| Grant | Use |
|-------|-----|
| **Authorization Code + PKCE** | Web/mobile apps (the standard, secure flow) |
| **Client Credentials** | Service-to-service (no user — machine-to-machine) |
| (Implicit, Password — **deprecated/avoid**) | Legacy |
```
Authorization Code flow ("Login with Google"):
1. User clicks "Login with Google" -> redirected to Google
2. User authenticates with Google; Google redirects back with an auth CODE
3. Your server exchanges the code (+ secret) for tokens
4. Your app gets an id_token (who) + access_token (what)
```

### 6.2 Spring Security OAuth2
| Role | Spring support |
|------|----------------|
| **OAuth2 Client** | Your app logs users in via Google/GitHub/Keycloak (`spring-boot-starter-oauth2-client`) |
| **Resource Server** | Your API validates incoming OAuth2/JWT tokens (`spring-boot-starter-oauth2-resource-server`) |
```java
// Resource server (validate JWTs from an OAuth2 provider):
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```
> **Use OAuth2/OIDC** for "Login with X" (delegated identity) and for integrating with identity providers (**Keycloak**, **Okta**, Auth0). For service-to-service, use the **Client Credentials** grant. Spring Security has first-class support for both client and resource-server roles. (Awareness-level for now; deep in real projects.)

---

## 7. CSRF Protection

**CSRF (Cross-Site Request Forgery)** tricks a logged-in user's browser into making unwanted requests using their **cookies** (recall Phase 0.2/15).
> ⚠️ **CSRF only matters for cookie-based (session) auth.** For **stateless JWT APIs** (token in the `Authorization` header, not a cookie), CSRF is **not a threat** → it's standard to **disable CSRF** for such APIs:
```java
http.csrf(csrf -> csrf.disable());   // OK for stateless token-based APIs
```
| Auth type | CSRF protection |
|-----------|-----------------|
| Cookie/session-based | **Enable** CSRF (Spring's default) |
| Stateless JWT (header) | **Disable** CSRF (not applicable) |
> The reasoning: CSRF works because browsers **auto-send cookies**. A JWT sent via the `Authorization` header is *not* auto-sent by the browser, so an attacker can't forge it → CSRF doesn't apply. (Don't blindly disable CSRF for session-based apps!)

---

## 8. Security Headers

Spring Security adds protective **HTTP headers** (recall Phase 0.2 headers) to defend against common attacks:
| Header | Protects against |
|--------|------------------|
| **`Content-Security-Policy` (CSP)** | XSS (controls allowed sources — Phase 15) |
| **`X-Frame-Options`** | Clickjacking (prevents framing) |
| **`Strict-Transport-Security` (HSTS)** | Downgrade attacks (forces HTTPS — recall Phase 0.2 TLS) |
| `X-Content-Type-Options: nosniff` | MIME-sniffing |
```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
    .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true)));
```
> Spring Security sets sensible defaults for many of these. **HSTS forces HTTPS** (recall Phase 0.2 — tokens/cookies must travel over HTTPS only). **CSP** mitigates XSS (Phase 15). These are defense-in-depth (Phase 15).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Putting secrets in the JWT payload | It's readable (Base64, not encrypted) — never |
| Long-lived access tokens | Keep them short; use refresh tokens |
| Assuming JWTs can be revoked instantly | They can't — use blacklist/short expiry |
| Sending tokens over HTTP | HTTPS only (HSTS) |
| Disabling CSRF for cookie-based auth | Only disable for stateless token APIs |
| Storing JWTs in localStorage (XSS risk) | Prefer httpOnly cookies for refresh tokens |
| Using deprecated OAuth2 grants (implicit/password) | Use Authorization Code + PKCE |
| Not validating the token signature | Always verify signature + expiry |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **JWT filter** = a custom `OncePerRequestFilter` (Phase 5.3) in the security chain (previous note).
- **Stateless tokens** enable horizontal scaling (Phase 0.2/13) — no shared session state.
- **Token revocation/blacklist** uses **Redis** (Phase 4.8); cleanup via **`@Scheduled`** (Phase 5.8 — Project 4).
- **OAuth2/OIDC** integrates with identity providers (Keycloak/Okta) — common in microservices (Phase 12).
- **API Gateway** validates JWTs centrally (Phase 12.4).
- **Security headers + HTTPS/TLS** (Phase 0.2) are defense-in-depth (Phase 15 OWASP).
- **Projects 4, 5, 7** use JWT auth with roles (Phase 5.6 core).

---

## 11. Quick Self-Check Questions

1. What's the difference between session-based and stateless token auth? Why does JWT scale better?
2. What are the three parts of a JWT? Is the payload encrypted?
3. Why use both access and refresh tokens, and what lifetimes?
4. How does a custom JWT filter authenticate a request?
5. Why can't JWTs be revoked instantly, and what are the strategies?
6. What's the difference between OAuth 2.0 and OpenID Connect? Name two grant types.
7. When should you disable CSRF, and why?
8. Name three security headers and what they protect against.

---

## 12. Key Terms Glossary

- **JWT:** signed, self-contained token (header.payload.signature).
- **Claims:** the JWT payload data (`sub`, `roles`, `exp`).
- **Access / refresh token:** short-lived access / long-lived renewal token.
- **Bearer token:** token sent in `Authorization: Bearer ...`.
- **Stateless auth:** no server-side session (scales horizontally).
- **Token revocation / blacklist:** invalidating a token before expiry.
- **OAuth 2.0 / OpenID Connect:** delegated authorization / identity layer.
- **Grant types (Authorization Code + PKCE, Client Credentials):** OAuth2 flows.
- **OAuth2 Client / Resource Server:** login-via-provider / token-validating API.
- **CSRF:** cross-site request forgery (cookie-based threat).
- **Security headers (CSP/HSTS/X-Frame-Options):** browser-side protections.

---

*Previous topic: **Authentication & Authorization**.*
*This completes **Section 5.6 — Spring Security**.*
*Next section in roadmap: **5.7 Spring Cache Abstraction**.*
