# Spring Security: Authentication & Authorization

> **Phase 5 — Spring Framework & Spring Boot → 5.6 Spring Security**
> Goal: Master the core of Spring Security — authentication vs authorization, the security filter chain, UserDetailsService, password encoding, and method/URL-level authorization.

---

## 0. The Big Picture

**Spring Security** protects your application: it verifies **who** a user is (**authentication**) and **what** they're allowed to do (**authorization**). It's built on a **filter chain** (recall filters, Phase 5.3) that intercepts every request before it reaches your controllers.

```
Request -> [ Security Filter Chain: authenticate + authorize ] -> Controller (if allowed)
```

> Security is non-negotiable for any real backend. Spring Security is comprehensive (and complex) — this note covers the core model; the next covers JWT and OAuth2. It connects to HTTP (Phase 0.2 — auth headers, status codes), filters (Phase 5.3), and security fundamentals (Phase 15).

---

## 1. Authentication vs Authorization (the fundamental distinction)

> ⚠️ **Authentication = WHO you are. Authorization = WHAT you can do.** (Recall Phase 0.2 — `401 Unauthorized` = not authenticated; `403 Forbidden` = authenticated but not allowed.)
| | **Authentication (AuthN)** | **Authorization (AuthZ)** |
|---|----------------------------|---------------------------|
| Question | "Who are you?" | "Are you allowed to do this?" |
| Mechanism | Verify credentials (password, token) | Check roles/permissions |
| Failure | `401 Unauthorized` | `403 Forbidden` |
| Example | Logging in | Accessing an admin page |
> This distinction is foundational. **AuthN happens first** (establish identity), then **AuthZ** (check permissions). The `401` vs `403` HTTP codes (Phase 0.2) map exactly to these.

---

## 2. The Security Filter Chain

Spring Security works as a **chain of servlet filters** (recall Phase 5.3 filters) that process every request **before** it reaches your controllers:
```
Request -> [Filter1: extract credentials] -> [Filter2: authenticate]
        -> [Filter3: authorize] -> ... -> Controller (only if all pass)
```
- Each filter handles one concern (extract a token, authenticate, check authorization, CSRF, etc.).
- If authentication/authorization fails, a filter **short-circuits** with `401`/`403` — the request never reaches your controller.
> The filter chain is *the* architecture of Spring Security. Custom authentication (like JWT — next note) means adding a **custom filter** to this chain (a `OncePerRequestFilter` — Phase 5.3).

---

## 3. SecurityFilterChain Configuration

In modern Spring Security (6+), you configure security with a **`SecurityFilterChain` bean** (lambda DSL):
```java
@Configuration
@EnableWebSecurity                              // enable Spring Security web support
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()        // anyone
                .requestMatchers("/api/admin/**").hasRole("ADMIN")    // ADMIN role only
                .requestMatchers(HttpMethod.POST, "/api/users").hasAuthority("user:create")
                .anyRequest().authenticated()                         // everything else: must be authed
            )
            .csrf(csrf -> csrf.disable())          // disable CSRF for stateless APIs (next note)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))  // JWT
            .httpBasic(Customizer.withDefaults()); // (or formLogin, or a JWT filter — next note)
        return http.build();
    }
}
```
| `HttpSecurity` config | Purpose |
|----------------------|---------|
| `authorizeHttpRequests` | URL-based authorization rules (§6) |
| `csrf` | CSRF protection (enable for sessions, disable for stateless APIs — next note) |
| `sessionManagement` | Session policy (STATELESS for JWT — next note) |
| `httpBasic` / `formLogin` / custom filter | The authentication mechanism |
> `@EnableWebSecurity` + a `SecurityFilterChain` bean is the modern config (replacing the old `WebSecurityConfigurerAdapter`). Rules are evaluated **top to bottom** — order matters (put specific rules before `anyRequest()`).

---

## 4. Authentication: UserDetailsService & UserDetails

To authenticate, Spring needs to **load user data** (username, password hash, roles) — via a **`UserDetailsService`**:
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;   // your entity repo (Phase 5.4)
    public CustomUserDetailsService(UserRepository repo) { this.userRepository = repo; }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));   // Optional (Phase 1.8)
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())          // the STORED HASH (never plaintext!)
            .authorities("ROLE_" + user.getRole())     // roles/authorities
            .build();
    }
}
```
| Component | Role |
|-----------|------|
| **`UserDetailsService`** | Loads a user by username (from your DB) |
| **`UserDetails`** | Represents the user (username, password hash, authorities) |
| **`AuthenticationManager`** | Orchestrates authentication |
| **`AuthenticationProvider`** | Performs the actual credential check |
> Spring calls `loadUserByUsername`, gets the stored **password hash**, and compares it against the submitted password using the **`PasswordEncoder`** (§5). You provide the `UserDetailsService` (reading your `User` entity — Phase 5.4); Spring does the comparison.

---

## 5. Password Encoding (NEVER Store Plaintext)

> ⚠️ **NEVER store passwords in plaintext (or with MD5/SHA — they're too fast and crackable). Always hash with a slow, salted algorithm — BCrypt is the standard.** (Recall Phase 15 — OWASP cryptographic failures.)
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();        // slow, salted hashing — the standard
}

// When registering a user — HASH the password:
String hash = passwordEncoder.encode(rawPassword);   // store THIS hash
user.setPasswordHash(hash);

// Spring uses encoder.matches(rawPassword, storedHash) during login (automatically)
```
| Algorithm | Use for passwords? |
|-----------|--------------------|
| **BCrypt** (or Argon2, scrypt) | ✅ Yes — slow + salted (resists brute force) |
| MD5 / SHA-1 / SHA-256 | ❌ No — too fast, no salt → crackable |
| Plaintext | ❌❌ Never |
> ⭐ **BCrypt** is intentionally **slow** and **salted** — each password gets a unique salt, and the cost factor makes brute-forcing expensive (recall Phase 0.1 — fast hashes are *bad* for passwords; you *want* slowness here). Recall passwords should be `char[]` not `String` where possible (Phase 1.4/15). This is one of the most important security rules.

---

## 6. Authorization (URL & Method Level)

### 6.1 URL-based (in the SecurityFilterChain — §3)
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/users/**").hasAnyRole("ADMIN", "USER")
    .anyRequest().authenticated())
```

### 6.2 Method-level security
Enable with `@EnableMethodSecurity`, then annotate methods (recall AOP — Phase 5.1):
```java
@EnableMethodSecurity                              // enable method security
// ...
@PreAuthorize("hasRole('ADMIN')")                  // checked BEFORE the method runs
public void deleteUser(Long id) { ... }

@PreAuthorize("hasAuthority('user:read') and #id == authentication.principal.id")  // SpEL (Phase 5.1)
public UserDto getUser(Long id) { ... }            // can access only your own data

@PostAuthorize("returnObject.ownerId == authentication.principal.id")  // checked AFTER
public Document getDocument(Long id) { ... }
```
| Annotation | When |
|------------|------|
| **`@PreAuthorize`** | Before the method (most common) — uses SpEL |
| **`@PostAuthorize`** | After the method (check the return value) |
| `@PreFilter`/`@PostFilter` | Filter collection arguments/results |
> Method security uses **AOP** (Phase 5.1) and **SpEL** (Phase 5.1) — `@PreAuthorize("hasRole('ADMIN')")`. It's more granular than URL rules and lives next to the business logic. Note: it has the same **proxy gotchas as `@Transactional`** (self-invocation — Phase 5.1/5.5).

---

## 7. RBAC vs Authorities (Roles vs Permissions)

| Concept | Meaning |
|---------|---------|
| **Role** | A coarse group (`ADMIN`, `USER`) — prefixed `ROLE_` internally |
| **Authority/Permission** | A fine-grained action (`user:create`, `order:read`) |
| **RBAC** | Role-Based Access Control — assign roles to users, permissions to roles |
| **ABAC** | Attribute-Based — decisions based on attributes (user, resource, context) |
```java
.hasRole("ADMIN")              // -> checks for authority "ROLE_ADMIN" (Spring adds the prefix)
.hasAuthority("user:delete")   // -> exact authority match (no prefix)
```
> ⚠️ **`hasRole("ADMIN")` checks for the authority `ROLE_ADMIN`** (Spring auto-prefixes `ROLE_`). **Roles** are coarse (admin/user); **authorities/permissions** are fine-grained (`order:cancel`). RBAC (roles) is most common; permission-based access (authorities) is more flexible for complex apps.

---

## 8. Getting the Current User (SecurityContext)

The authenticated user is stored in the **`SecurityContextHolder`** (a `ThreadLocal` — recall Phase 1.10!):
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
// In a controller — inject it directly:
@GetMapping("/me")
public UserDto me(@AuthenticationPrincipal UserDetails user) {   // current user injected
    return userService.findByUsername(user.getUsername());
}
```
> The `SecurityContextHolder` uses a **`ThreadLocal`** (Phase 1.10) — the current user is bound to the request thread. ⚠️ This is why it must be **cleared per request** (Spring does this) and why it's tricky with `@Async`/other threads (recall ThreadLocal pitfalls, Phase 1.10). `@AuthenticationPrincipal` is the clean way to get the current user in controllers. Also used by JPA auditing's `@CreatedBy` (Phase 5.4).

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Storing plaintext/MD5/SHA passwords | Use BCrypt (slow, salted) |
| Confusing 401 and 403 | 401 = not authenticated; 403 = not authorized |
| `hasRole("ROLE_ADMIN")` (double prefix) | Use `hasRole("ADMIN")` (prefix added automatically) |
| Authorization rules in wrong order | Specific rules before `anyRequest()` |
| Method security self-invocation | Same proxy gotcha as `@Transactional` (Phase 5.1) |
| Forgetting CSRF concerns | Enable for sessions; disable for stateless APIs (next note) |
| Returning sensitive data despite auth | Combine with DTOs (Phase 5.3/7) |
| Relying on frontend for security | Always enforce server-side |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **JWT & OAuth2** (next note) build on this — a custom filter authenticates the token, then this authorization model applies.
- **`401`/`403`** map to authentication/authorization (Phase 0.2); handled by `@RestControllerAdvice`/Spring (Phase 5.3).
- **Method security** uses AOP + SpEL (Phase 5.1) — same proxy gotchas (Phase 5.5).
- **`SecurityContextHolder`** (ThreadLocal — Phase 1.10) feeds JPA auditing's `@CreatedBy` (Phase 5.4).
- **BCrypt password hashing** is a core security control (Phase 15 — cryptographic failures).
- **RBAC** secures endpoints in every project (Projects 4, 5, 7: ADMIN/USER/CUSTOMER/VENDOR roles).
- **Filter chain** = filters (Phase 5.3) applied to security.

---

## 11. Quick Self-Check Questions

1. What's the difference between authentication and authorization? Which HTTP codes map to each?
2. How does the security filter chain work?
3. How do you configure security in modern Spring Security?
4. What do `UserDetailsService` and `UserDetails` do?
5. Why must passwords be hashed with BCrypt (not MD5/SHA/plaintext)?
6. What's the difference between URL-based and method-level authorization?
7. What does `hasRole("ADMIN")` actually check?
8. What's the difference between roles and authorities/permissions (RBAC)?
9. How do you get the current authenticated user, and what's the ThreadLocal caveat?

---

## 12. Key Terms Glossary

- **Authentication (AuthN):** verifying identity (→ 401 on failure).
- **Authorization (AuthZ):** verifying permissions (→ 403 on failure).
- **Security filter chain:** filters that authenticate/authorize each request.
- **`SecurityFilterChain` / `HttpSecurity`:** modern security config.
- **`UserDetailsService` / `UserDetails`:** load / represent a user.
- **`AuthenticationManager` / `AuthenticationProvider`:** authentication orchestration.
- **`PasswordEncoder` / BCrypt:** password hashing (slow, salted).
- **`@PreAuthorize` / `@PostAuthorize`:** method-level authorization (AOP + SpEL).
- **Role / authority / RBAC / ABAC:** coarse group / fine permission / access-control models.
- **`SecurityContextHolder` / `@AuthenticationPrincipal`:** current-user storage (ThreadLocal) / injection.

---

*This is the first note of **Section 5.6 — Spring Security**.*
*Next topic: **JWT, OAuth2 & Security Headers**.*
