# Byte Streams, Character Streams & Buffered Streams

> **Phase 1 — Java Language Mastery → 1.9 Java I/O**
> Goal: Master classic Java I/O — the byte stream and character stream hierarchies, buffering for performance, and how it all connects to encoding and resource management.

---

## 0. The Big Picture

**I/O (Input/Output)** is reading data into your program and writing data out — to/from files, the network, the console, memory. Java's classic I/O (`java.io`, since Java 1.0) is built on **streams**: ordered sequences of data you read or write one piece at a time.

```
INPUT:  source (file/socket) --> InputStream/Reader --> your program
OUTPUT: your program --> OutputStream/Writer --> destination (file/socket)
```

This connects directly to **Phase 0.1** (blocking I/O, the user/kernel boundary, DMA) — classic `java.io` is **blocking, stream-based** I/O. NIO (next note) adds non-blocking and channel-based I/O.

> Two parallel hierarchies: **byte streams** (raw binary) and **character streams** (text + encoding). Choosing correctly — and **always closing** resources — is the core skill.

---

## 1. The Two Hierarchies: Bytes vs Characters

| | **Byte streams** | **Character streams** |
|---|------------------|------------------------|
| Unit | Raw **bytes** (8-bit) | **Characters** (text, encoding-aware) |
| Base classes | `InputStream` / `OutputStream` | `Reader` / `Writer` |
| Use for | Binary data: images, audio, files, network, serialized objects | Text: files, logs, JSON, config |
| Encoding | None (raw bytes) | **Applies a charset** (UTF-8, etc.) |

> ⚠️ **Use character streams for text** (they handle encoding correctly — recall Phase 1.4) and **byte streams for binary**. Reading text with a byte stream and converting manually invites mojibake.

---

## 2. Byte Streams (`InputStream` / `OutputStream`)

The root abstract classes for reading/writing **raw bytes**.

### 2.1 The hierarchy
```
InputStream (abstract)
├── FileInputStream          - read bytes from a file
├── ByteArrayInputStream     - read bytes from a byte[] in memory
├── BufferedInputStream      - adds buffering (wraps another stream)
├── DataInputStream          - read primitives (int, double, ...)
└── ObjectInputStream        - read serialized objects (next note)

OutputStream (abstract)
├── FileOutputStream
├── ByteArrayOutputStream
├── BufferedOutputStream
├── DataOutputStream
└── ObjectOutputStream
```

### 2.2 Core methods
| `InputStream` | `OutputStream` |
|---------------|----------------|
| `int read()` — one byte (-1 at end) | `write(int b)` — one byte |
| `int read(byte[] buf)` — into a buffer | `write(byte[] buf)` |
| `close()` | `write(byte[], off, len)`, `flush()`, `close()` |

```java
// Copy a binary file (byte by byte is slow — see buffering §4):
try (InputStream in = new FileInputStream("source.jpg");
     OutputStream out = new FileOutputStream("dest.jpg")) {
    byte[] buffer = new byte[8192];          // read in chunks
    int bytesRead;
    while ((bytesRead = in.read(buffer)) != -1) {
        out.write(buffer, 0, bytesRead);
    }
}   // try-with-resources auto-closes both (recall Phase 1.5)
```

---

## 3. Character Streams (`Reader` / `Writer`)

The root abstract classes for reading/writing **text** — they apply a **character encoding** to convert between bytes and characters.

### 3.1 The hierarchy
```
Reader (abstract)
├── InputStreamReader        - BRIDGE: bytes -> chars (applies a charset)
│   └── FileReader           - convenience for reading text files
├── BufferedReader           - buffering + readLine()
├── CharArrayReader / StringReader

Writer (abstract)
├── OutputStreamWriter       - BRIDGE: chars -> bytes (applies a charset)
│   └── FileWriter
├── BufferedWriter           - buffering + newLine()
├── PrintWriter              - print()/println()/printf()
```

### 3.2 The bridge classes (bytes ↔ chars)
`InputStreamReader`/`OutputStreamWriter` convert between byte streams and character streams using a **charset** — **always specify it explicitly** (recall Phase 1.4 encoding):
```java
Reader reader = new InputStreamReader(new FileInputStream("file.txt"),
                                      StandardCharsets.UTF_8);   // explicit charset!
```
> ⚠️ `FileReader`/`FileWriter` historically used the **platform default charset** (a portability bug — recall Phase 0.3 env). Prefer `new InputStreamReader(..., UTF_8)` or the NIO `Files` methods (next note) for explicit encoding. (Java 11+ `FileReader` can take a charset.)

### 3.3 Reading text line by line
```java
try (BufferedReader br = new BufferedReader(
        new InputStreamReader(new FileInputStream("data.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = br.readLine()) != null) {   // readLine() — the workhorse
        process(line);
    }
}

// Or as a Stream (Java 8+) — pairs with Phase 1.8:
try (BufferedReader br = ...) {
    br.lines().filter(l -> !l.isBlank()).forEach(this::process);
}
```

### 3.4 Writing text
```java
try (PrintWriter pw = new PrintWriter(
        new BufferedWriter(new OutputStreamWriter(
            new FileOutputStream("out.txt"), StandardCharsets.UTF_8)))) {
    pw.println("Hello");
    pw.printf("Count: %d%n", 42);
}
```

---

## 4. Buffered Streams (Performance — Critical)

Reading/writing **one byte/char at a time** hits the underlying source (disk/OS) on **every call** — extremely slow (recall Phase 0.1: each I/O op crosses to the kernel). **Buffered** streams read/write in **large chunks** into an in-memory buffer, drastically reducing system calls.

```java
// SLOW: each read() may hit the disk
InputStream in = new FileInputStream("big.txt");

// FAST: reads a big chunk into a buffer, serves bytes from memory
InputStream in = new BufferedInputStream(new FileInputStream("big.txt"));
```

### 4.1 How buffering helps
```
Unbuffered: read() -> disk, read() -> disk, read() -> disk ...   (1 syscall per byte)
Buffered:   fill 8KB buffer from disk ONCE -> serve 8192 bytes from memory -> refill
```
- A buffer (default ~8KB) means **far fewer** expensive system calls.
- Wrap the raw stream: `new BufferedReader(new FileReader(...))`, `new BufferedOutputStream(new FileOutputStream(...))`.
> **Always wrap file/network streams in a buffered stream** (or use `Files`/NIO which buffers) — it can be 10–100× faster.

### 4.2 The Decorator pattern
Notice the **wrapping**: `BufferedReader(InputStreamReader(FileInputStream(...)))`. This is the **Decorator pattern** (Phase 14) — each layer adds behavior (file access → encoding → buffering → line reading) around the one inside it.
```
FileInputStream   (read bytes from file)
  wrapped by InputStreamReader  (bytes -> chars, charset)
    wrapped by BufferedReader   (buffering + readLine)
```

### 4.3 `flush()` and writing
Buffered **writers** hold data in memory until the buffer fills. **`flush()`** forces buffered data out; **`close()`** flushes then closes. Forgetting to close/flush can lose the last buffered data:
```java
writer.write("important");
writer.flush();   // ensure it's written now
// (close() also flushes — try-with-resources handles this)
```

---

## 5. Standard Streams (Console I/O)

The JVM provides three standard streams (recall Phase 0.3 stdin/stdout/stderr):
| Stream | Type | Purpose |
|--------|------|---------|
| `System.in` | `InputStream` | Standard input |
| `System.out` | `PrintStream` | Standard output |
| `System.err` | `PrintStream` | Standard error |

```java
// Read console input:
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
String line = br.readLine();
// Or the simpler Scanner:
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();

System.out.println("output");
System.err.println("error message");
```

---

## 6. Resource Management — Always Close (recap from Phase 1.5)

Streams are **OS resources** (file handles, sockets). **Not closing them leaks resources** → eventually "too many open files" errors that crash servers. **Always use try-with-resources:**
```java
try (var in = new BufferedReader(new InputStreamReader(
        new FileInputStream("f.txt"), StandardCharsets.UTF_8))) {
    // use in
}   // auto-closed, even on exception (recall AutoCloseable, Phase 1.5)
```
> All streams implement `Closeable`/`AutoCloseable` → use try-with-resources. **Never** rely on `finalize()` (Phase 1.3) or manual `close()` in a `finally` for these.

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Reading text with byte streams | Use character streams (Reader/Writer) for text |
| Relying on the platform default charset | Specify `StandardCharsets.UTF_8` explicitly |
| Unbuffered I/O (byte/char at a time) | Wrap in `Buffered*` (huge speedup) |
| Forgetting to close streams | Use try-with-resources |
| Forgetting to flush a writer before reading output | `flush()` or close it |
| `FileReader`/`FileWriter` for portable encoding | Use `InputStreamReader`/`Files` with a charset |
| Loading a huge file fully into memory | Stream it line-by-line (`readLine`/`lines()`) |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **File upload/download (Phase 5.3):** `MultipartFile`, `Resource`, `StreamingResponseBody` build on these streams.
- **Reading config/templates/files** in services and batch jobs.
- **Project 2 (file processor):** reading large CSVs line-by-line with buffered readers/streams.
- **Resource leaks** (unclosed streams) are a real production crash cause — try-with-resources is mandatory (Phase 13).
- **Encoding correctness** (UTF-8) prevents data corruption in APIs/files (Phase 1.4, 15).
- **NIO `Files`** (next note) often replaces this boilerplate with one-liners.

---

## 9. Quick Self-Check Questions

1. What's the difference between byte streams and character streams, and when use each?
2. Name the four root abstract classes of classic I/O.
3. What do `InputStreamReader`/`OutputStreamWriter` do, and what must you always specify?
4. Why is unbuffered I/O slow, and how does buffering fix it?
5. What design pattern does stream wrapping illustrate?
6. What does `flush()` do, and why can forgetting it lose data?
7. Why must streams always be closed, and what's the right way?
8. How do you read a large text file efficiently without loading it all into memory?

---

## 10. Key Terms Glossary

- **Stream (I/O):** ordered sequence of data read/written piece by piece.
- **Byte stream:** `InputStream`/`OutputStream` — raw bytes.
- **Character stream:** `Reader`/`Writer` — encoding-aware text.
- **`InputStreamReader`/`OutputStreamWriter`:** byte↔char bridges (apply a charset).
- **`BufferedReader`/`BufferedInputStream`:** buffered wrappers (performance).
- **Buffering:** reading/writing in chunks to reduce system calls.
- **Decorator pattern:** wrapping streams to layer behavior.
- **`flush()`:** force buffered output to be written.
- **`PrintWriter`/`PrintStream`:** convenient text output (`println`, `printf`).
- **Standard streams:** `System.in`/`out`/`err`.
- **`Closeable`/`AutoCloseable`:** interfaces enabling try-with-resources.

---

*This is the first note of **Section 1.9 — Java I/O**.*
*Next topic: **NIO (New I/O) — Path, Files, Channels, Buffers**.*
