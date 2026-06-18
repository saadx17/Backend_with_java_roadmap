# Clean Code

> **Phase 14 — Design Patterns & Architecture → 14.3 Clean Code**
> Goal: Write code that humans can read and change — meaningful naming, small focused functions, comments done right, error handling, DRY/KISS/YAGNI, code smells, and refactoring.

---

## 0. The Big Picture

Code is **read far more often than it's written** (and mostly by future-you). **Clean Code** (Robert C. Martin) is about optimizing for the reader — clarity over cleverness. It's the line-and-function-level complement to SOLID (14.2, class level) and architecture (14.4, system level).

```
   Write once  ──────►  Read & modify many times (you + teammates, for years)
   → optimize for READABILITY and CHANGEABILITY, not for showing off
```

> Clean Code complements SOLID (14.2), is enforceable via SonarQube/linters (10.3) and reviews (Phase 0.4), and makes code testable (Phase 6). ⚠️ It's principles, not dogma — judgment matters.

> ⭐ **"Any fool can write code that a computer can understand. Good programmers write code that humans can understand."** — Martin Fowler.

---

## 1. Meaningful Names ⭐

| Rule | ❌ Bad | ✅ Good |
|------|-------|--------|
| Intention-revealing | `int d;` | `int daysSinceLastLogin;` |
| Pronounceable/searchable | `genymdhms` | `generationTimestamp` |
| No cryptic abbreviations | `calc(x, y)` | `calculateTax(price, rate)` |
| Avoid encodings/Hungarian | `strName`, `iCount` | `name`, `count` |
| Class = noun, method = verb | `class Manage` | `class OrderValidator`, `validate()` |
| Consistent vocabulary | `get/fetch/retrieve` mixed | pick one verb per concept |
| Avoid disinformation/noise | `accountList` (if it's a Set) | `accounts` |
> ⭐ **Naming is the highest-leverage clean-code skill.** A good name removes the need for a comment. Booleans read as predicates (`isActive`, `hasPermission`). The name should answer *why it exists, what it does, how it's used*.

---

## 2. Functions ⭐

| Rule | Why |
|------|-----|
| **Small** | A function should fit on a screen; ideally a few lines |
| **Do one thing** | One level of abstraction (mirrors SRP — 14.2) |
| **Few parameters** | 0–2 ideal; 3 is a lot; >3 → pass an object/record (Phase 1.3) |
| **No side effects** | Don't secretly mutate state the name doesn't imply |
| **No boolean/flag args** | `render(true)` is opaque → split into two methods |
| **Command-Query Separation** | A method either *does* something or *answers* something, not both |
| **Avoid deep nesting** | Guard clauses / early return over nested `if` pyramids |
```java
// ❌ does many things, flag arg, deep nesting
void process(Order o, boolean notify) {
    if (o != null) { if (o.isValid()) { save(o); if (notify) { email(o); } } }
}
// ✅ small, guard clauses, one thing, no flag
void placeOrder(Order order) {
    validate(order);          // throws if invalid (early)
    repository.save(order);
}
```
> ⭐ Small functions with one level of abstraction read like prose. **Guard clauses** (early return) flatten nesting. ⚠️ A function that needs a comment to separate its "sections" should be **several functions**.

---

## 3. Comments — Done Right

| Good comments | Bad comments |
|---------------|--------------|
| *Why* something is done (rationale, trade-off) | Restating *what* the code obviously does |
| Warning of consequences / `// TODO`, `// FIXME` | Commented-out code (delete it — Git remembers, Phase 0.4) |
| Public API docs (Javadoc) | Misleading/outdated comments (worse than none) |
| Legal/regulatory notes | "Changelog" comments (use Git) |
```java
// ❌ noise:  i++; // increment i
// ✅ why:    // Retry once: the payment gateway occasionally returns a transient 503 (see INC-1423)
```
> ⭐ **The best comment is a good name** (§1) or a well-named function — code that explains itself. Comments **rot** (code changes, comments don't) → prefer self-documenting code, reserve comments for *why*, not *what*. ⚠️ **Delete commented-out code** — version control (Phase 0.4) is your history.

---

## 4. Error Handling

(Recall exceptions — Phase 1.5.) Clean error handling separates the happy path from failure handling.
| Rule | Why |
|------|-----|
| **Exceptions over error codes** | Don't pollute return values; keep flow clean (Phase 1.5) |
| **Don't swallow exceptions** | Empty `catch {}` hides bugs — at least log (9.1) or rethrow |
| **Fail fast** | Validate early; throw close to the problem |
| **Meaningful exceptions/messages** | Context for debugging (don't leak secrets — 15.3) |
| **Don't return null** | Use `Optional` (Phase 1.8) / empty collections to avoid NPEs |
| **Clean up resources** | try-with-resources (Phase 1.9) |
> ⭐ In Spring, centralize API error handling in `@RestControllerAdvice` → `ProblemDetail` (Phase 7.1), keeping controllers clean. ⚠️ The worst smell is a **swallowed exception** — a silent `catch` that turns a failure into a mysterious wrong result.

---

## 5. DRY, KISS, YAGNI ⭐

| Principle | Meaning | Watch out |
|-----------|---------|-----------|
| **DRY** (Don't Repeat Yourself) | Every piece of knowledge has one authoritative place | ⚠️ Don't over-DRY — coincidental duplication isn't real duplication (premature abstraction couples unrelated things) |
| **KISS** (Keep It Simple) | Prefer the simplest solution that works | Complexity is a cost, not a virtue |
| **YAGNI** (You Aren't Gonna Need It) | Don't build for imagined future needs | Speculative generality is a smell (§6) |
> ⭐ These balance the patterns/SOLID enthusiasm (14.1/14.2): **the simplest code that solves the real problem wins.** ⚠️ Over-applying DRY ("hyper-DRY") creates wrong abstractions — sometimes a little duplication is cheaper than the wrong shared abstraction. "Duplication is far cheaper than the wrong abstraction" (Sandi Metz).

---

## 6. Code Smells & Refactoring

| Smell | What | Fix |
|-------|------|-----|
| **Long method** | Does too much | Extract methods (§2) |
| **Large class / god object** | Too many responsibilities | Split (SRP — 14.2) |
| **Long parameter list** | >3 params | Parameter object/record |
| **Duplicated code** | Same logic in many places | Extract (DRY §5) |
| **Magic numbers/strings** | Unexplained literals | Named constants/enums |
| **Feature envy** | A method obsessed with another class's data | Move method |
| **Primitive obsession** | `String email`, `BigDecimal money` everywhere | Value objects (Money/Email — 12.1) |
| **Shotgun surgery** | One change touches many files | Consolidate responsibility |
| **Dead code / speculative generality** | Unused / "just in case" | Delete (YAGNI §5) |
| **Deep nesting** | Arrow-shaped code | Guard clauses (§2) |

**Refactoring** = improving code structure **without changing behavior** — done in small, safe steps, **backed by tests** (Phase 6).
```
   Refactor loop:  ensure tests pass → make a small structural change → tests still pass → repeat
```
> ⭐ **Refactor continuously** (the "Boy Scout Rule": leave code cleaner than you found it). ⚠️ **Never refactor without tests** — they're your safety net that behavior didn't change (Phase 6.1 — TDD makes this natural). Tools (IDE refactorings — Phase 1.1) make renames/extracts safe.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Cryptic names | Intention-revealing names (§1) |
| Huge multi-purpose functions | Small, one-thing functions (§2) |
| Comments restating code / commented-out code | Self-documenting code; delete dead code (§3) |
| Swallowed exceptions | Log/rethrow; never silent catch (§4) |
| Returning `null` | `Optional`/empty collections (§4, 1.8) |
| Over-DRY (wrong abstraction) | Tolerate small duplication over bad coupling (§5) |
| Building for imagined futures | YAGNI (§5) |
| Magic numbers/strings | Named constants/enums (§6) |
| Primitive obsession | Value objects (§6, 12.1) |
| Refactoring without tests | Have tests first (§6, Phase 6) |
| Cleverness over clarity | Optimize for the reader (§0) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- Line/function-level complement to **SOLID** (14.2) and **architecture** (14.4).
- Enforced by **SonarQube** quality gates (10.3) and **code review** (Phase 0.4).
- **Error handling** ↔ exceptions (1.5), `Optional` (1.8), `ProblemDetail`/`@RestControllerAdvice` (7.1/5.3).
- **Value objects** ↔ DDD (12.1), records (1.3); **refactoring** ↔ tests (Phase 6, TDD 6.1).
- Makes code **testable** (Phase 6) and **changeable** (supports all later phases).
- Applied across **every Project**.

---

## 9. Quick Self-Check Questions

1. Why optimize code for reading over writing?
2. What makes a good name, and why is naming the highest-leverage skill?
3. List the rules for clean functions (size, one-thing, params, flags, nesting).
4. When are comments good vs bad, and what should you do with commented-out code?
5. Give four clean error-handling rules; why is a swallowed exception the worst?
6. Define DRY, KISS, YAGNI — and how can DRY be over-applied?
7. Name five code smells and their refactorings.
8. What is refactoring, and why must it be backed by tests?
9. What is the Boy Scout Rule?
10. What is primitive obsession, and how do value objects fix it?

---

## 10. Key Terms Glossary

- **Clean Code:** code optimized for human readability and changeability.
- **Intention-revealing name:** a name that explains purpose without comments.
- **Single level of abstraction:** a function works at one conceptual level.
- **Guard clause:** early return to flatten nesting.
- **Command-Query Separation:** a method does or answers, not both.
- **DRY / KISS / YAGNI:** don't repeat / keep simple / don't build unneeded.
- **Code smell:** a surface indicator of a deeper design problem.
- **Primitive obsession / magic number / god class / feature envy:** common smells.
- **Refactoring:** behavior-preserving structural improvement.
- **Boy Scout Rule:** leave code cleaner than you found it.

---

*This is the note for **Section 14.3 — Clean Code**.*
*Previous section in roadmap: **14.2 SOLID Principles**.*
*Next section in roadmap: **14.4 Architecture Patterns**.*
