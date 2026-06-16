# Common String Methods, Formatting & Text Blocks

> **Phase 1 — Java Language Mastery → 1.4 Strings (Deep Dive)**
> Goal: Master the everyday `String` methods, string formatting (`format()`, `formatted()`, `printf`), and text blocks (Java 13+/15).

---

## 0. The Big Picture

This note covers the **practical toolkit** you'll use daily: methods to inspect, search, extract, and transform strings; ways to format strings with placeholders; and multi-line text blocks. (Remember: every method returns a **new** string — strings are immutable.)

---

## 1. Common String Methods

### 1.1 Inspecting
```java
String s = "Hello, World";
s.length();             // 12
s.isEmpty();            // false  (length == 0)
s.isBlank();            // false  (Java 11+: empty or only whitespace)
s.charAt(0);            // 'H'
s.indexOf("o");         // 4   (first occurrence; -1 if not found)
s.lastIndexOf("o");     // 8
s.contains("World");    // true
s.startsWith("Hello");  // true
s.endsWith("World");    // true
```

### 1.2 Extracting
```java
s.substring(7);          // "World"      (from index 7 to end)
s.substring(0, 5);       // "Hello"      (from 0, up to but NOT including 5)
s.split(", ");           // ["Hello", "World"]  (returns String[])
s.split(",", 2);         // limit the number of splits
"a,b,c".split(",");      // ["a", "b", "c"]
```
> `substring(begin, end)` is **end-exclusive**. An out-of-range index throws `StringIndexOutOfBoundsException`.

### 1.3 Transforming (each returns a NEW string)
```java
s.toUpperCase();         // "HELLO, WORLD"
s.toLowerCase();         // "hello, world"
s.trim();                // removes leading/trailing whitespace (ASCII only)
s.strip();               // Java 11+: Unicode-aware trim (preferred)
s.stripLeading();        // / stripTrailing()
s.replace("o", "0");     // "Hell0, W0rld"  (replaces ALL literal occurrences)
"  a  ".repeat(3);       // Java 11+: repeats the string
"a\nb\nc".lines();       // Java 11+: Stream<String> of lines
```

### 1.4 Comparison (recap from Fundamentals note)
```java
s.equals("Hello, World");           // content equality
s.equalsIgnoreCase("hello, world"); // case-insensitive
s.compareTo("Hello");               // lexicographic ordering
```

### 1.5 Conversion
```java
String.valueOf(42);          // "42"  (int -> String)
Integer.parseInt("42");      // 42    (String -> int; throws NumberFormatException)
s.toCharArray();             // char[] of characters
String.valueOf(charArray);   // char[] -> String
s.getBytes(StandardCharsets.UTF_8);  // byte[] (see encoding note)
new String(bytes, StandardCharsets.UTF_8);  // byte[] -> String
```

### 1.6 `trim()` vs `strip()` (subtle but important)
- `trim()` removes only characters ≤ U+0020 (ASCII space and below) — **legacy**.
- `strip()` (Java 11+) is **Unicode-aware** (handles all Unicode whitespace) — **prefer it**.

### 1.7 Method quick-reference table
| Category | Methods |
|----------|---------|
| Length/empty | `length`, `isEmpty`, `isBlank` |
| Char access | `charAt`, `toCharArray`, `chars()` |
| Search | `indexOf`, `lastIndexOf`, `contains`, `startsWith`, `endsWith`, `matches` |
| Extract | `substring`, `split`, `lines` |
| Transform | `toUpperCase`, `toLowerCase`, `trim`, `strip`, `replace`, `replaceAll`, `repeat` |
| Compare | `equals`, `equalsIgnoreCase`, `compareTo`, `compareToIgnoreCase` |
| Convert | `valueOf`, `getBytes`, `format`, `join` |

---

## 2. String Formatting

### 2.1 `String.format()` and `formatted()`
Build a string from a **format string** with **placeholders** (format specifiers):
```java
String msg = String.format("Name: %s, Age: %d, Score: %.2f", "Sam", 30, 95.5);
// "Name: Sam, Age: 30, Score: 95.50"

String msg2 = "Name: %s, Age: %d".formatted("Sam", 30);  // Java 15+: instance method
```

### 2.2 `printf` (print directly)
```java
System.out.printf("Name: %s, Age: %d%n", "Sam", 30);  // prints, doesn't return
```
> `%n` is a platform-independent newline (preferred over `\n` in format strings).

### 2.3 Common format specifiers
| Specifier | Meaning | Example |
|-----------|---------|---------|
| `%s` | String (any object's `toString`) | `%s` → `Sam` |
| `%d` | Integer (decimal) | `%d` → `30` |
| `%f` | Floating-point | `%.2f` → `95.50` |
| `%e` | Scientific notation | `%e` → `9.55e+01` |
| `%x` / `%o` | Hex / octal | `%x` → `1e` |
| `%b` | Boolean | `%b` → `true` |
| `%c` | Character | `%c` → `A` |
| `%%` | Literal percent sign | `%%` → `%` |
| `%n` | Line separator | (newline) |

### 2.4 Width, precision, flags
```java
String.format("%5d", 42);      // "   42"   (width 5, right-aligned)
String.format("%-5d|", 42);    // "42   |"  (left-aligned)
String.format("%05d", 42);     // "00042"   (zero-padded)
String.format("%.3f", 3.14159);// "3.142"   (precision)
String.format("%,d", 1000000); // "1,000,000" (grouping)
String.format("%+d", 42);      // "+42"     (always show sign)
```

> ⚠️ Format specifier/argument mismatches throw runtime exceptions (`IllegalFormatException`, `MissingFormatArgumentException`). For money, format a `BigDecimal`, don't rely on `%f` rounding for storage.

---

## 3. Text Blocks (Java 13 preview → 15)

A **text block** is a multi-line string literal delimited by **triple quotes** `"""`, eliminating the need for `\n` and escaped quotes. Great for JSON, SQL, HTML, etc.

### 3.1 Basic syntax
```java
String json = """
        {
            "name": "Sam",
            "age": 30
        }
        """;
```
Equivalent old way (ugly):
```java
String json = "{\n    \"name\": \"Sam\",\n    \"age\": 30\n}\n";
```

### 3.2 Rules
- Opens with `"""` followed by a **line break** (the opening line can't have content).
- **Incidental indentation** is stripped automatically based on the **least-indented line** (and the closing `"""` position) — you control the result's indentation by where you place the closing delimiter.
- Quotes don't need escaping inside (only `"""` itself does).
- Newlines are preserved as written.

### 3.3 Common uses
```java
String sql = """
        SELECT id, name, email
        FROM users
        WHERE active = true
        ORDER BY name
        """;

String html = """
        <html>
            <body><h1>Hello</h1></body>
        </html>
        """;
```

### 3.4 Useful escapes (Java 14+)
- `\` at end of line → **suppress the newline** (line continuation):
  ```java
  String s = """
          Hello \
          World""";        // "Hello World" (one line)
  ```
- `\s` → an explicit space (preserved, prevents trailing-space stripping).

### 3.5 Combine with formatting
```java
String body = """
        {
          "user": "%s",
          "id": %d
        }
        """.formatted(name, id);
```

---

## 4. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `substring` index off-by-one (end-exclusive) | Remember `[begin, end)`; guard bounds |
| `replace` vs `replaceAll` confusion | `replace` = literal; `replaceAll` = **regex** (next note) |
| Using `trim()` for Unicode whitespace | Use `strip()` (Java 11+) |
| Format specifier/argument mismatch | Match types & count; test format strings |
| Using `%f` for money rounding/storage | Use `BigDecimal` |
| `split` with a regex-special delimiter (`.`, `|`) | Escape it: `split("\\.")` (next note) |
| Forgetting text block opening newline | `"""` must be followed by a line break |
| Unexpected text block indentation | Position the closing `"""` to control stripping |

---

## 5. Connection to Backend / Spring (Why This Matters Later)

- **Text blocks** make embedded **SQL/JSON/HTML** readable — common in JPA `@Query`, tests, and JSON fixtures (Phase 5, 6).
- **`String.format`/`formatted`** for building messages, logs (but prefer **parameterized logging** in hot paths — Phase 9), and responses.
- **`split`/`substring`/`strip`** are everywhere in parsing request data, headers, CSV (Project 2), and tokens.
- **`replaceAll` is regex-based** — a frequent bug source (next note covers regex).
- **`getBytes`/`new String` with explicit charset** is essential for correct I/O and APIs (encoding note).

---

## 6. Quick Self-Check Questions

1. What's the difference between `isEmpty()` and `isBlank()`? `trim()` and `strip()`?
2. Is `substring(2, 5)` inclusive or exclusive of index 5?
3. What's the difference between `replace` and `replaceAll`?
4. Write a `String.format` that pads an int to width 5 with leading zeros.
5. What does `%n` mean and why prefer it over `\n`?
6. What problem do text blocks solve, and how is incidental indentation handled?
7. How do you suppress a newline inside a text block?
8. Why shouldn't you use `%f` for storing monetary values?

---

## 7. Key Terms Glossary

- **`substring`:** extract a portion (end-exclusive).
- **`split`:** divide a string by a **regex** delimiter into an array.
- **`strip` vs `trim`:** Unicode-aware vs ASCII-only whitespace removal.
- **`replace` vs `replaceAll`:** literal vs regex replacement.
- **Format specifier:** placeholder like `%s`, `%d`, `%.2f`.
- **`String.format` / `formatted` / `printf`:** build/print formatted strings.
- **`%n`:** platform-independent newline in format strings.
- **Text block:** multi-line string literal delimited by `"""`.
- **Incidental indentation:** leading whitespace stripped from text blocks.

---

*Previous topic: **String vs StringBuilder vs StringBuffer**.*
*Next topic: **Regular Expressions**.*
