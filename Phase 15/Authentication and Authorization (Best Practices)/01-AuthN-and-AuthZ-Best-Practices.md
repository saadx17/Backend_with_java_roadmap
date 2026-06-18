# Authentication & Authorization Best Practices

> **Phase 15 — Security → 15.2 AuthN/AuthZ Best Practices**
> Goal: Secure identity and access — password storage (BCrypt), multi-factor authentication, JWT security, OAuth2/OIDC, API keys, and access control models (RBAC vs ABAC).

---

## 0. The Big Picture

**Authentication (authN)** = *who are you?* (proving identity). **Authorization (authZ)** = *what are you allowed to do?* (enforcing permissions). These map to OWASP **A07** (auth failures) and **A01** (broken access control) — two of the most damaging risk categories (15.1). This section is the *best-practices* layer on top of the Spring Security mechanics from Phase 5.6.

```
   AuthN: "Prove you're Alice"  →  credentials / token / MFA
   AuthZ: "Alice can read orders but not delete users"  →  roles/permissions/policies
   401 = not authenticated   ·   403 = authenticated but not authorized  (Phase 7.1)
```

> Builds on Spring Security (Phase 5.6 — filters, `SecurityFilterChain`, JWT/OAuth2), web security (15.1), and HTTP status semantics (7.1). Stateless tokens enable scalability (13.3). Secrets/keys tie to data protection (15.3) and config/Vault (12.5).

---

## 1. Password Storage ⭐ (never store plaintext)

☠️ **Never** store passwords in plaintext or with fast/general hashes (MD5/SHA-256). Use a **slow, salted, adaptive password hashing** function designed to resist brute-force.
| Algorithm | Note |
|-----------|------|
| **BCrypt** ⭐ | Spring Security default; salted, adaptive (work factor); battle-tested |
| **Argon2** | Modern winner (memory-hard); strong choice |
| **scrypt / PBKDF2** | Also acceptable; PBKDF2 is FIPS-friendly |
| ❌ MD5 / SHA-1 / SHA-256 | **Too fast** → brute-forceable; never for passwords |
```java
// Spring Security (Phase 5.6)
PasswordEncoder encoder = new BCryptPasswordEncoder(12);  // work factor (cost) — tune for ~hundreds of ms
String hash = encoder.encode(rawPassword);                // salt is generated & embedded automatically
boolean ok  = encoder.matches(rawPassword, hash);         // verify

// Or DelegatingPasswordEncoder: stores {bcrypt}... prefix → can upgrade algorithms over time
```
| Concept | Why |
|---------|-----|
| **Salt** | Random per-password → identical passwords hash differently; defeats rainbow tables (auto with BCrypt) |
| **Work factor / cost** | Makes each guess slow; raise it as hardware improves |
| **Adaptive** | Tunable cost over time |
| **Pepper** (optional) | Secret added before hashing, stored separately (defense in depth — 15.3) |
> ⭐ **BCrypt (or Argon2) with a tuned cost factor** is the standard. The point is to be **deliberately slow** so an attacker who steals the DB can't crack hashes quickly. Also: enforce strong password policies, check against breached-password lists, and **never log passwords** (9.1/15.1).

---

## 2. Multi-Factor Authentication (MFA)

MFA requires **two or more factors** from different categories — so a stolen password alone isn't enough.
| Factor type | Examples |
|-------------|----------|
| **Something you know** | Password, PIN |
| **Something you have** | Phone (**TOTP** app), security key (FIDO2/WebAuthn), hardware token |
| **Something you are** | Biometrics (fingerprint, face) |
| Common impl | **TOTP** (time-based one-time code, e.g., Google Authenticator), push approval, **passkeys/WebAuthn** ⭐ |
> ⭐ MFA is one of the **highest-impact** security controls — it stops most credential-theft/phishing attacks. ⚠️ **SMS OTP is weak** (SIM-swap/interception) — prefer **TOTP apps** or **passkeys (WebAuthn/FIDO2)**. Offer MFA especially for privileged accounts.

---

## 3. JWT Security ⭐

(Recall JWT mechanics — Phase 5.6.) A **JWT** is a signed token (`header.payload.signature`) carrying claims; it enables **stateless** auth (no server session → scales — 13.3). But it has sharp edges:
| Best practice | Why |
|---------------|-----|
| **Strong signature** (RS256/ES256 or strong HS256 secret) | Prevent forgery |
| ⚠️ **Reject `alg: none`** & validate the algorithm | Classic JWT bypass attack |
| **Validate signature, expiry (`exp`), issuer (`iss`), audience (`aud`)** | Don't trust unverified tokens |
| **Short-lived access tokens** (mins) + **refresh tokens** | Limit damage of a leaked token |
| **Don't put secrets/PII in the payload** | JWT payload is **base64, not encrypted** — anyone can read it! |
| **Store safely on the client** | `HttpOnly`+`Secure`+`SameSite` cookie (XSS/CSRF — 15.1) or secure storage; not plain localStorage |
| **Revocation strategy** | JWTs are stateless → can't easily revoke; use short expiry + a denylist/token-version for logout |
```
   header.payload.signature   ← payload is READABLE (base64). Sign, don't trust blindly. Don't store secrets in it.
```
> ⭐ The biggest JWT mistakes: (1) **not validating** (or accepting `alg:none`), (2) putting **sensitive data in the readable payload**, (3) **long-lived** tokens you can't revoke. Use **short access tokens + refresh tokens**, validate everything, and have a logout/revocation plan. Spring Security validates JWTs as an OAuth2 **Resource Server** (Phase 5.6, gateway — 12.4).

---

## 4. OAuth2 & OpenID Connect (OIDC) ⭐

**OAuth2** is a **delegated authorization** framework — let an app access resources on a user's behalf **without** sharing the password (e.g., "Sign in with Google"). **OIDC** adds an **authentication** layer (identity) on top of OAuth2.
| Term | Meaning |
|------|---------|
| **Resource Owner** | The user |
| **Client** | The app requesting access |
| **Authorization Server** | Issues tokens (Keycloak, Auth0, Okta, Google) |
| **Resource Server** | The API that accepts the access token (your Spring app — 5.6) |
| **Access token / Refresh token** | Short-lived access / longer-lived renewal |
| **ID token (OIDC)** | A JWT proving *who* the user is |
| **Scopes** | What the token is allowed to do |
| **Grant types/flows** | **Authorization Code + PKCE** (web/mobile) ⭐, **Client Credentials** (service-to-service — 12.2) |
```
   Authorization Code + PKCE (the secure default for user login):
   User → Auth Server (login + consent) → code → Client exchanges code (+PKCE) → tokens → call API
```
> ⭐ Use a **dedicated, vetted Authorization Server** (Keycloak/Auth0/Okta) — **don't roll your own** identity provider. For SPAs/mobile use **Authorization Code + PKCE** (not the deprecated Implicit flow); for **service-to-service**, **Client Credentials** (12.2). Your Spring services are **Resource Servers** validating the tokens (Phase 5.6, gateway centralizes it — 12.4).

---

## 5. API Keys

For **machine-to-machine** access (third-party integrations, simple service auth) where full OAuth2 is overkill.
| API key practice | Note |
|------------------|------|
| Treat as a **secret** | Store hashed (like passwords — §1); transmit over TLS only |
| Scope & rate-limit per key | Least privilege; throttle (15.1/12.4) |
| Rotate & revoke | Support key rotation; expire compromised keys |
| Identify, don't fully secure | API keys identify the caller but are weaker than OAuth2 (no user context, easily leaked) |
> ⚠️ API keys are **convenient but weak** — they're long-lived bearer secrets often leaked in code/logs (15.1). Prefer **OAuth2 Client Credentials** for service auth where possible; if using API keys, **hash them, scope them, rotate them, and rate-limit**.

---

## 6. Authorization Models: RBAC vs ABAC ⭐

| | **RBAC (Role-Based)** ⭐ | **ABAC (Attribute-Based)** |
|--|--------------------------|----------------------------|
| Decision based on | **Roles** (ADMIN, USER, MANAGER) | **Attributes** (user dept, resource owner, time, location) |
| Example | "ADMINs can delete users" | "A user can edit a doc IF they own it AND it's not locked" |
| Pros | Simple, common, easy to reason about | Fine-grained, context-aware, flexible |
| Cons | Coarse; "role explosion" for nuanced rules | Complex to implement/audit |
| Spring | `@PreAuthorize("hasRole('ADMIN')")` | `@PreAuthorize("#doc.owner == authentication.name")` |
```java
@PreAuthorize("hasRole('ADMIN')")                       // RBAC (Phase 5.6)
public void deleteUser(Long id) { ... }

@PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")   // ABAC-style (ownership)
public Order getOrder(Long orderId) { ... }
```
> ⭐ **RBAC for coarse permissions; ABAC for fine-grained, contextual rules** (ownership, attributes). Most apps use **RBAC + a sprinkle of ABAC** (ownership checks). ⚠️ **A01 Broken Access Control (15.1) is the #1 risk** — the classic bug is **IDOR** (Insecure Direct Object Reference): `GET /orders/42` returns *anyone's* order because you forgot to check the requester owns it. **Always enforce authZ on every request**, ideally close to the data, and **never trust client-supplied ids/roles**.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Plaintext / fast-hash passwords | BCrypt/Argon2 with salt + cost (§1) |
| SMS-only MFA | Prefer TOTP/passkeys (§2) |
| Not validating JWT / accepting `alg:none` | Validate sig/exp/iss/aud; reject none (§3) |
| Secrets/PII in JWT payload | Payload is readable — keep it minimal (§3) |
| Long-lived, unrevocable tokens | Short access + refresh + denylist (§3) |
| Rolling your own auth server | Use Keycloak/Auth0/Okta (§4) |
| Implicit flow for SPAs | Authorization Code + PKCE (§4) |
| API keys in code/logs, unhashed | Hash, scope, rotate, rate-limit (§5, 15.1) |
| Role explosion / missing checks | RBAC + ABAC; enforce authZ everywhere (§6) |
| **IDOR** (no ownership check) | Verify the caller owns/can access the resource (§6, A01) |
| Trusting client-supplied role/id | Derive identity from the validated token (§6) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Implements on **Spring Security** (Phase 5.6 — `PasswordEncoder`, JWT/OAuth2 Resource Server, `@PreAuthorize`).
- Addresses OWASP **A07/A01** (15.1); **401/403** semantics (7.1).
- **JWT** enables **stateless** scaling (13.3); centralized validation at the **gateway** (12.4).
- **OAuth2 Client Credentials** for **inter-service** auth (12.2); secrets via **Vault** (12.5).
- **Keys/tokens** are secrets → **data protection** (15.3); **dependency security** for auth libs (15.4).
- **Rate limiting** protects login (15.1/12.4); **MFA/passkeys** reduce account takeover.
- Critical across **Projects 4/5/7**.

---

## 9. Quick Self-Check Questions

1. AuthN vs authZ, and which HTTP status maps to each failure?
2. Why never store plaintext or SHA-256 passwords, and what makes BCrypt/Argon2 right?
3. What are salt, work factor, and pepper?
4. What is MFA, the factor categories, and why avoid SMS OTP?
5. List the key JWT validation/security practices. Why not put secrets in the payload?
6. Why use short access tokens + refresh tokens, and how do you handle revocation?
7. OAuth2 vs OIDC; what are the main roles and which flows for which clients?
8. Why not roll your own auth server, and why Authorization Code + PKCE?
9. When are API keys appropriate, and how do you secure them?
10. RBAC vs ABAC, and what is IDOR / broken access control (and how to prevent it)?

---

## 10. Key Terms Glossary

- **Authentication / Authorization:** who you are / what you may do.
- **BCrypt / Argon2 / salt / work factor / pepper:** password hashing essentials.
- **MFA / TOTP / WebAuthn (passkeys):** multi-factor auth methods.
- **JWT / claims / `alg:none`:** signed token / payload data / a forgery attack to reject.
- **Access vs refresh token:** short-lived vs renewal token.
- **OAuth2 / OIDC:** delegated authorization / identity layer.
- **Authorization Server / Resource Server / Client:** OAuth2 roles.
- **Grant flows (Authorization Code + PKCE / Client Credentials):** user login / service auth.
- **API key:** machine-to-machine bearer secret.
- **RBAC / ABAC:** role-based / attribute-based access control.
- **IDOR / Broken Access Control:** missing ownership/permission checks (A01).

---

*This is the note for **Section 15.2 — AuthN/AuthZ Best Practices**.*
*Previous section in roadmap: **15.1 Web Security Fundamentals**.*
*Next section in roadmap: **15.3 Data Protection**.*
