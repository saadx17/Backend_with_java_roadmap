# Regular Expressions (Regex)

> **Phase 1 — Java Language Mastery → 1.4 Strings (Deep Dive)**
> Goal: Master Java regex — `Pattern` and `Matcher`, common patterns, greedy vs lazy quantifiers, capturing & named groups, and lookahead/lookbehind.

---

## 0. The Big Picture

A **regular expression (regex)** is a pattern describing a set of strings — used for **matching, searching, extracting, and replacing** text. Java's regex lives in `java.util.regex` (`Pattern` + `Matcher`), and several `String` methods use regex directly.

```java
"abc123".matches("[a-z]+\\d+");   // true — letters followed by digits
```

> ⚠️ In Java, regex patterns are written as **strings**, so backslashes must be **doubled**: the regex `\d` is written `"\\d"` in Java source.

---

## 1. Pattern and Matcher Classes

The core API: **compile** a pattern once, then **match** against input.

```java
import java.util.regex.*;

Pattern pattern = Pattern.compile("\\d+");        // compile the regex
Matcher matcher = pattern.matcher("abc123def456"); // bind to input

while (matcher.find()) {                           // find each match
    System.out.println(matcher.group());           // "123", then "456"
}
```

### 1.1 Matcher methods
| Method | Meaning |
|--------|---------|
| `matches()` | Does the **entire** input match the pattern? |
| `find()` | Find the **next** match anywhere in the input (call repeatedly) |
| `lookingAt()` | Does the **start** of input match? |
| `group()` | The text of the last match |
| `group(n)` | The text of capturing group *n* |
| `start()` / `end()` | Index positions of the match |
| `replaceAll(repl)` / `replaceFirst(repl)` | Replace matches |

### 1.2 Performance: compile once, reuse
```java
// BAD: recompiles the regex every call
boolean isValid(String s) { return s.matches("\\d+"); }

// GOOD: compile once, reuse the Pattern (thread-safe; Matcher is not)
private static final Pattern DIGITS = Pattern.compile("\\d+");
boolean isValid(String s) { return DIGITS.matcher(s).matches(); }
```
> `Pattern.compile` is relatively expensive. **Compile constant patterns once** (e.g., a `static final` field). `Pattern` is thread-safe; `Matcher` is **not** (create one per use).

### 1.3 String methods that use regex
| Method | Note |
|--------|------|
| `s.matches(regex)` | Whole-string match (implicitly anchored) |
| `s.split(regex)` | Split on a regex (escape special chars!) |
| `s.replaceAll(regex, repl)` | Regex replace all |
| `s.replaceFirst(regex, repl)` | Regex replace first |
> `s.replace(...)` is **literal** (not regex) — easy to confuse with `replaceAll` (recap from methods note).

---

## 2. Common Regex Syntax

### 2.1 Character classes
| Pattern | Matches |
|---------|---------|
| `.` | Any character (except newline by default) |
| `\d` / `\D` | A digit / non-digit |
| `\w` / `\W` | Word char `[a-zA-Z0-9_]` / non-word |
| `\s` / `\S` | Whitespace / non-whitespace |
| `[abc]` | Any one of a, b, c |
| `[^abc]` | Any char **except** a, b, c |
| `[a-z]` | Any char in the range |
| `[a-zA-Z0-9]` | Combined ranges |

### 2.2 Quantifiers (how many)
| Pattern | Matches |
|---------|---------|
| `*` | 0 or more |
| `+` | 1 or more |
| `?` | 0 or 1 (optional) |
| `{n}` | Exactly n |
| `{n,}` | n or more |
| `{n,m}` | Between n and m |

### 2.3 Anchors & boundaries
| Pattern | Matches |
|---------|---------|
| `^` | Start of input (or line in multiline mode) |
| `$` | End of input (or line) |
| `\b` / `\B` | Word boundary / non-boundary |

### 2.4 Groups & alternation
| Pattern | Meaning |
|---------|---------|
| `(...)` | Capturing group |
| `(?:...)` | Non-capturing group |
| `a|b` | a OR b (alternation) |

### 2.5 Example patterns
```java
"\\d{3}-\\d{4}"                       // phone like 123-4567
"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"  // simple email
"^\\d{5}(-\\d{4})?$"                  // US zip (5 or 9 digit)
"https?://\\S+"                       // http or https URL (s is optional)
```

---

## 3. Greedy vs Lazy (Reluctant) Quantifiers

By default, quantifiers are **greedy** — they match **as much as possible**, then backtrack. Adding `?` makes them **lazy** — match **as little as possible**.

```java
String html = "<a><b>";

// GREEDY: <.+>  matches as much as possible
Pattern.compile("<.+>").matcher(html).results()...   // matches "<a><b>" (the whole thing!)

// LAZY: <.+?>  matches as little as possible
Pattern.compile("<.+?>").matcher(html)...            // matches "<a>" then "<b>"
```

| Quantifier | Greedy | Lazy | Possessive |
|------------|--------|------|------------|
| 0+ | `*` | `*?` | `*+` |
| 1+ | `+` | `+?` | `++` |
| 0/1 | `?` | `??` | `?+` |
| n..m | `{n,m}` | `{n,m}?` | `{n,m}+` |

- **Greedy:** grabs everything, backtracks to fit — default.
- **Lazy:** grabs minimum, expands as needed — use when you want the *smallest* match.
- **Possessive (`*+`):** like greedy but **never backtracks** — faster, avoids catastrophic backtracking (advanced).

> Greedy quantifiers on ambiguous input can cause **catastrophic backtracking** — a regex that hangs on certain inputs (a real **ReDoS** security risk — Phase 15). Prefer specific patterns over `.*` where possible.

---

## 4. Capturing Groups

Parentheses `(...)` **capture** the matched substring for later extraction or back-reference.

```java
Pattern p = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");  // date: yyyy-mm-dd
Matcher m = p.matcher("2026-06-12");
if (m.matches()) {
    m.group(0);   // "2026-06-12"  (group 0 = entire match)
    m.group(1);   // "2026"        (first group)
    m.group(2);   // "06"
    m.group(3);   // "12"
}
```
- **Group 0** is always the whole match; groups 1, 2, ... are numbered by opening parenthesis order.
- **Non-capturing group `(?:...)`** groups without capturing (slightly more efficient when you don't need the value).

### 4.1 Back-references
Refer to a previous group within the pattern with `\1`, `\2`, etc. (written `\\1` in Java):
```java
"(\\w+) \\1"     // matches a repeated word, e.g., "hello hello"
```

### 4.2 Replacement with group references
In `replaceAll`, refer to captured groups with `$1`, `$2`:
```java
"2026-06-12".replaceAll("(\\d{4})-(\\d{2})-(\\d{2})", "$3/$2/$1");
// "12/06/2026"  (reorder to dd/mm/yyyy)
```

---

## 5. Named Groups

Instead of remembering group numbers, **name** them with `(?<name>...)` and reference by name — far more readable.

```java
Pattern p = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})");
Matcher m = p.matcher("2026-06-12");
if (m.matches()) {
    m.group("year");   // "2026"
    m.group("month");  // "06"
    m.group("day");    // "12"
}
```
- Reference in replacement with `${name}`:
  ```java
  "2026-06-12".replaceAll("(?<y>\\d{4})-(?<m>\\d{2})-(?<d>\\d{2})", "${d}/${m}/${y}");
  ```
> Named groups make complex patterns self-documenting — prefer them over numbered groups for non-trivial regex.

---

## 6. Lookahead and Lookbehind (Zero-Width Assertions)

These match a position based on what **follows** or **precedes** — **without consuming** characters (zero-width).

| Construct | Name | Matches if... |
|-----------|------|---------------|
| `(?=...)` | Positive lookahead | followed by `...` |
| `(?!...)` | Negative lookahead | **not** followed by `...` |
| `(?<=...)` | Positive lookbehind | preceded by `...` |
| `(?<!...)` | Negative lookbehind | **not** preceded by `...` |

### 6.1 Examples
```java
// Positive lookahead: digits followed by "px" (but don't include "px" in the match)
Pattern.compile("\\d+(?=px)").matcher("12px 30pt")...   // matches "12" only

// Negative lookahead: a word NOT followed by a digit
"\\b\\w+\\b(?!\\d)"

// Positive lookbehind: number preceded by "$"
Pattern.compile("(?<=\\$)\\d+").matcher("$100")...      // matches "100"

// Password rule: at least one digit and one uppercase (lookaheads as constraints)
"^(?=.*\\d)(?=.*[A-Z]).{8,}$"   // 8+ chars, has a digit AND an uppercase
```
> Lookarounds are powerful for **validation constraints** (e.g., password rules) and **extracting** without including delimiters. They don't move the match position.

---

## 7. Useful Pattern Flags

Pass flags to `Pattern.compile(regex, flags)` or inline with `(?i)`:
```java
Pattern.compile("hello", Pattern.CASE_INSENSITIVE);   // or "(?i)hello"
```
| Flag | Effect |
|------|--------|
| `CASE_INSENSITIVE` (`(?i)`) | Ignore case |
| `MULTILINE` (`(?m)`) | `^`/`$` match line starts/ends |
| `DOTALL` (`(?s)`) | `.` also matches newlines |
| `COMMENTS` (`(?x)`) | Allow whitespace/comments in the pattern |
| `UNICODE_CASE` | Unicode-aware case folding |

---

## 8. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Forgetting to double backslashes (`\d` → `"\\d"`) | Always escape in Java strings |
| `split(".")` splitting on everything | `.` is regex "any char" — use `split("\\.")` |
| Confusing `replace` (literal) with `replaceAll` (regex) | Use the right one |
| Recompiling patterns in hot paths | Compile once as `static final` |
| Sharing a `Matcher` across threads | `Matcher` isn't thread-safe; `Pattern` is |
| Catastrophic backtracking / ReDoS | Avoid nested `.*`; use possessive quantifiers / specific patterns |
| Over-relying on regex for email/HTML | Use proper validators/parsers for complex formats |
| Using `group(n)` and miscounting | Use **named groups** |

---

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Input validation** (Phase 5.3): `@Pattern(regexp=...)` Bean Validation, custom validators — emails, phone numbers, usernames.
- **Routing/parsing:** URL path patterns, log parsing, tokenizing.
- **Security (Phase 15):** **ReDoS** (regex denial of service) from catastrophic backtracking is a real vulnerability — keep patterns simple and bounded; never run untrusted regex.
- **Performance (Phase 13):** precompiled `static final Pattern` avoids recompilation in hot endpoints.
- **Data processing** (Project 2): extracting/cleaning fields from CSV/text.
- **Prefer libraries** for emails/URLs/dates where correctness matters — regex is a tool, not always the right one.

---

## 10. Quick Self-Check Questions

1. Why must regex backslashes be doubled in Java source?
2. What's the difference between `matches()`, `find()`, and `lookingAt()`?
3. Why compile a `Pattern` once, and which of `Pattern`/`Matcher` is thread-safe?
4. What's the difference between greedy, lazy, and possessive quantifiers?
5. What is group 0, and how do numbered vs named groups differ?
6. How do you reference captured groups in a `replaceAll` replacement?
7. Explain positive/negative lookahead and lookbehind with an example.
8. Why is `split(".")` a bug, and what causes ReDoS?

---

## 11. Key Terms Glossary

- **Regex:** a pattern describing a set of strings.
- **`Pattern` / `Matcher`:** compiled regex / engine bound to input.
- **Character class:** `[...]`, `\d`, `\w`, `\s`, etc.
- **Quantifier:** `* + ? {n,m}` — repetition count.
- **Greedy / lazy / possessive:** match max / min / max-no-backtrack.
- **Anchor:** `^`, `$`, `\b` — position assertions.
- **Capturing group:** `(...)` — captures matched text.
- **Non-capturing group:** `(?:...)`.
- **Named group:** `(?<name>...)`.
- **Back-reference:** `\1`, `${name}` — reuse a captured group.
- **Lookahead / lookbehind:** `(?=)`, `(?!)`, `(?<=)`, `(?<!)` — zero-width assertions.
- **ReDoS:** denial of service via catastrophic regex backtracking.
- **Flags:** `CASE_INSENSITIVE`, `MULTILINE`, `DOTALL`, etc.

---

*Previous topic: **Common Methods, Formatting & Text Blocks**.*
*Next topic: **Character Encoding (ASCII, UTF-8, UTF-16)**.*
