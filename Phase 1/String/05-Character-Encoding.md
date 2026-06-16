# Character Encoding (ASCII, UTF-8, UTF-16)

> **Phase 1 — Java Language Mastery → 1.4 Strings (Deep Dive)**
> Goal: Understand how characters become bytes — ASCII, Unicode, UTF-8 and UTF-16 — how Java represents strings internally, and how to avoid the encoding bugs that plague real backends.

---

## 0. The Big Picture

Computers store **bytes**, but humans use **characters**. A **character encoding** is the agreed mapping between the two. Getting it wrong produces **mojibake** (garbled text like `Ã©` instead of `é`) — one of the most common, frustrating real-world bugs.

```
Character  --(encoding)-->  Bytes  --(decoding)-->  Character
   'A'      UTF-8           0x41                       'A'
   '€'      UTF-8        0xE2 0x82 0xAC                '€'
```

Two distinct concepts (often confused):
- **Character set (charset):** a catalog mapping characters → numeric **code points** (e.g., Unicode says `'A'` = U+0041, `'€'` = U+20AC).
- **Encoding:** how those code points are turned into **bytes** (e.g., UTF-8, UTF-16).

---

## 1. ASCII

The original 7-bit encoding (1963): **128 characters** (0–127) covering English letters, digits, punctuation, and control characters.

```
'A' = 65 (0x41)    'a' = 97 (0x61)    '0' = 48 (0x30)    space = 32
```
- Fits in **7 bits** (often stored in 1 byte with the top bit 0).
- **Limitation:** no accented letters, no non-Latin scripts, no emoji. Insufficient for a global world.
- **Extended ASCII** (8-bit, e.g., Latin-1/ISO-8859-1) added 128 more characters — still far too few for all languages.

---

## 2. Unicode — The Universal Character Set

**Unicode** is a single character set aiming to include **every character in every language** (plus symbols and emoji). It assigns each character a unique **code point**, written `U+XXXX`.

```
'A'  = U+0041
'é'  = U+00E9
'€'  = U+20AC
'😀' = U+1F600   (emoji — beyond U+FFFF)
```
- Unicode has space for ~1.1 million code points (U+0000 to U+10FFFF).
- **Important:** Unicode defines the *numbers* (code points); it does **not** dictate how to store them as bytes — that's the job of an **encoding** (UTF-8, UTF-16, UTF-32).

### 2.1 BMP and supplementary characters
- **BMP (Basic Multilingual Plane):** U+0000–U+FFFF — covers almost all common characters in 16 bits.
- **Supplementary characters:** U+10000 and above (emoji, rare scripts) — need more than 16 bits.

---

## 3. UTF-8 (the web standard)

**UTF-8** is a **variable-length** encoding: 1 to 4 bytes per character. It's the **dominant encoding** of the web, files, and APIs.

| Code point range | Bytes | Notes |
|------------------|-------|-------|
| U+0000–U+007F (ASCII) | **1 byte** | Identical to ASCII! |
| U+0080–U+07FF | 2 bytes | Latin extras, Greek, Cyrillic, Arabic, Hebrew |
| U+0800–U+FFFF | 3 bytes | Most of the BMP (incl. CJK) |
| U+10000–U+10FFFF | 4 bytes | Emoji, rare scripts |

```
'A'  (U+0041) -> 0x41                 (1 byte)
'é'  (U+00E9) -> 0xC3 0xA9            (2 bytes)
'€'  (U+20AC) -> 0xE2 0x82 0xAC       (3 bytes)
'😀'(U+1F600)-> 0xF0 0x9F 0x98 0x80  (4 bytes)
```

### 3.1 Why UTF-8 won
| Advantage | Explanation |
|-----------|-------------|
| **ASCII-compatible** | ASCII text is valid UTF-8 byte-for-byte (huge for legacy/interop) |
| **Space-efficient** | English/ASCII-heavy text stays 1 byte/char |
| **No byte-order issues** | Byte order is fixed (no BOM needed, unlike UTF-16) |
| **Self-synchronizing** | You can find character boundaries from any position |
| **Universal** | The default for HTTP, JSON, HTML, most files and databases |

> **Default to UTF-8 everywhere** — files, HTTP, database connections, source files. It's the safe, universal choice.

---

## 4. UTF-16 (Java's internal historical encoding)

**UTF-16** uses **2 or 4 bytes** per character: 2 bytes for BMP characters, 4 bytes (a **surrogate pair**) for supplementary characters.

```
'A'  (U+0041) -> 0x0041                  (2 bytes)
'😀'(U+1F600)-> 0xD83D 0xDE00 (surrogate pair, 4 bytes)
```
- BMP characters fit in one 16-bit **code unit**.
- Supplementary characters use **two** 16-bit code units (a high + low **surrogate**).
- Has **endianness** (UTF-16BE vs UTF-16LE) → may need a **BOM** (Byte Order Mark) — a downside vs UTF-8.

---

## 5. How Java Represents Strings Internally

This is where Java's `char` quirk (Phase 1.2) makes sense:

- **`char` is 16 bits** — it holds **one UTF-16 code unit**, not necessarily one full character.
- A `String` is conceptually a sequence of UTF-16 code units.
- **Compact Strings (Java 9+):** internally, `String` stores a **`byte[]`** plus a coder flag. If all characters are Latin-1 (fit in 1 byte), it uses 1 byte/char (saving ~50% memory); otherwise UTF-16 (2 bytes/char). This is transparent to your code.

### 5.1 The supplementary-character gotcha
Because a `char` is only 16 bits, an emoji (supplementary character) occupies **two** `char`s:
```java
String s = "😀";
s.length();             // 2  (!) — two UTF-16 code units, not 1 character
s.charAt(0);            // a lone surrogate, not the emoji
s.codePointCount(0, s.length());  // 1  — the actual number of characters
s.codePoints().count();           // 1
```
> **`String.length()` counts code units (UTF-16), not characters.** For text with emoji/rare scripts, use **code points** (`codePoints()`, `codePointCount`) to count/iterate actual characters.

---

## 6. Encoding/Decoding in Java (always be explicit!)

Converting between `String` and `byte[]` requires a charset. **Always specify it explicitly** — never rely on the platform default.

```java
import java.nio.charset.StandardCharsets;

String s = "café";

// String -> bytes (encoding)
byte[] utf8  = s.getBytes(StandardCharsets.UTF_8);     // explicit — GOOD
byte[] bad   = s.getBytes();                            // platform default — AVOID

// bytes -> String (decoding)
String back = new String(utf8, StandardCharsets.UTF_8); // explicit — GOOD
```

### 6.1 Why the platform default is dangerous
`getBytes()` / `new String(byte[])` (no charset) use the **platform default encoding**, which **varies by OS/locale/JVM**:
- Same code works on your machine (UTF-8) but **corrupts data** on a server defaulting to a different charset → mojibake.
- This is a top cause of "works on my machine" text bugs.

> **Rule: always pass a `Charset` explicitly** (use `StandardCharsets.UTF_8`). Since Java 18, the default file encoding is UTF-8 (JEP 400), but being explicit is still best practice for portability.

### 6.2 Where encoding bites in real systems
- **File I/O:** reading/writing files with the wrong charset (Phase 1.9 — use `Files.readString(path, UTF_8)`).
- **HTTP:** request/response bodies need correct `Content-Type: ...; charset=UTF-8` (Phase 5).
- **Databases:** column/connection charset mismatches (use `utf8mb4` in MySQL for full Unicode incl. emoji).
- **Source files:** compile with `-encoding UTF-8` so literals are read correctly.

---

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Using `getBytes()` / `new String(bytes)` without a charset | Always pass `StandardCharsets.UTF_8` |
| Assuming `length()` == number of characters | Use `codePointCount` for emoji/supplementary chars |
| `charAt` splitting a surrogate pair | Iterate by code points |
| Mojibake from charset mismatch (encode UTF-8, decode Latin-1) | Use the **same** charset on both ends |
| MySQL `utf8` (3-byte) dropping emoji | Use `utf8mb4` |
| Relying on the platform default encoding | Be explicit everywhere |
| Missing `charset=UTF-8` in HTTP headers | Set content type charset |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **APIs (Phase 5):** request/response encoding must be UTF-8 end to end; Spring defaults to UTF-8 but verify `Content-Type` charsets.
- **Databases (Phase 4):** use `utf8mb4`/UTF-8 columns and connection charset, or emoji/non-Latin text corrupts.
- **File processing (Phase 1.9, Project 2):** read/write files with explicit UTF-8 (`Files.readString`, `Files.writeString`).
- **I18N:** supporting global users requires Unicode discipline throughout.
- **Security (Phase 15):** encoding confusion can bypass validation (e.g., overlong UTF-8, homoglyph attacks) — normalize and validate carefully.
- **`char[]` for passwords** (security note) interacts with encoding when hashing — encode explicitly.

---

## 9. Quick Self-Check Questions

1. What's the difference between a character set and an encoding?
2. What does Unicode define, and what does it leave to encodings?
3. Why did UTF-8 become the standard (give three reasons)?
4. How many bytes does UTF-8 use for ASCII vs an emoji?
5. How does Java store strings internally, and what changed in Java 9 (Compact Strings)?
6. Why does `"😀".length()` return 2, and how do you count actual characters?
7. Why is `getBytes()` without a charset dangerous?
8. What is mojibake and what typically causes it?

---

## 10. Key Terms Glossary

- **Character set (charset):** mapping of characters → code points.
- **Encoding:** mapping of code points → bytes.
- **ASCII:** 7-bit encoding, 128 characters.
- **Unicode:** universal character set assigning code points (`U+XXXX`).
- **Code point:** a Unicode character's numeric value.
- **Code unit:** the base storage unit of an encoding (16 bits for UTF-16).
- **UTF-8:** variable-length (1–4 bytes), ASCII-compatible, web standard.
- **UTF-16:** 2 or 4 bytes; Java's `char`/historical internal unit; has endianness.
- **Surrogate pair:** two UTF-16 code units encoding a supplementary character.
- **BMP / supplementary:** ≤ U+FFFF / ≥ U+10000.
- **Compact Strings:** Java 9+ byte[]-backed `String` storage.
- **BOM:** Byte Order Mark (UTF-16 endianness marker).
- **Mojibake:** garbled text from charset mismatch.
- **`StandardCharsets.UTF_8`:** the explicit charset constant to use.

---

*Previous topic: **Regular Expressions**.*
*This completes **Section 1.4 — Strings (Deep Dive)**.*
*Next section in roadmap: **1.5 Exception Handling**.*
