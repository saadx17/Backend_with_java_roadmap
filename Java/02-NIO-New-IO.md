# NIO (New I/O) — Path, Files, Channels & Buffers

> **Phase 1 — Java Language Mastery → 1.9 Java I/O**
> Goal: Master modern Java file I/O — `Path`/`Paths`, the `Files` utility class, channels & buffers, and the WatchService — the cleaner, more powerful APIs that largely replace classic `java.io`.

---

## 0. The Big Picture

**NIO** ("New I/O", `java.nio`, since Java 1.4; **NIO.2** file API since Java 7) modernizes Java I/O with:
- A clean **file system API** (`Path`, `Files`) that replaces clunky `java.io.File`.
- **Channels + Buffers** for efficient, lower-level data transfer.
- **Non-blocking I/O** capabilities (recall Phase 0.1: selectors/epoll, the basis of Netty).

```java
// Classic (verbose):  see previous note (streams + try-with-resources)
// NIO (one line):
List<String> lines = Files.readAllLines(Path.of("data.txt"), StandardCharsets.UTF_8);
String content = Files.readString(Path.of("config.yml"));
```

> For everyday file work, **prefer NIO's `Files`/`Path`** — it's concise, charset-aware, and less error-prone than classic streams.

---

## 1. Path and Paths (Replacing File)

A **`Path`** represents a file/directory location — the modern replacement for the old `java.io.File`.

```java
Path p = Path.of("/home/user/data.txt");        // modern (Java 11+)
Path p2 = Paths.get("/home/user/data.txt");      // older equivalent
Path rel = Path.of("config", "app.yml");         // config/app.yml (OS-independent separator)

// Path operations (these manipulate the path, they don't touch disk):
p.getFileName();        // data.txt
p.getParent();          // /home/user
p.getRoot();            // /
p.toAbsolutePath();     // resolve to absolute
p.resolve("sub/x.txt"); // append a path
p.relativize(other);    // path from one to another
p.normalize();          // remove . and .. segments
```
> A `Path` is just a location descriptor (no disk access until you use `Files`). It handles OS path separators correctly (recall Phase 0.3 paths).

---

## 2. The Files Utility Class (the workhorse)

`java.nio.file.Files` provides **static methods** for nearly every file operation — often one-liners replacing pages of classic stream code.

### 2.1 Reading
```java
String text = Files.readString(Path.of("f.txt"));                  // whole file -> String (Java 11+)
List<String> lines = Files.readAllLines(Path.of("f.txt"));         // all lines -> List
byte[] bytes = Files.readAllBytes(Path.of("img.jpg"));             // binary -> byte[]

// Stream lines lazily (for LARGE files — doesn't load all into memory):
try (Stream<String> lines = Files.lines(Path.of("big.csv"), StandardCharsets.UTF_8)) {
    lines.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}   // close the stream (it holds a file handle) — try-with-resources!
```

### 2.2 Writing
```java
Files.writeString(Path.of("out.txt"), "Hello");                   // write a String (Java 11+)
Files.write(Path.of("out.txt"), List.of("line1", "line2"));       // write lines
Files.write(Path.of("img.jpg"), byteArray);                       // write bytes

// Append instead of overwrite:
Files.writeString(Path.of("log.txt"), "entry\n", StandardOpenOption.APPEND);
```

### 2.3 File operations
```java
Files.exists(path);  Files.notExists(path);
Files.createFile(path);                  Files.createDirectories(Path.of("a/b/c"));
Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dest);                   // move/rename
Files.delete(path);                      // throws if missing
Files.deleteIfExists(path);              // safe delete
Files.size(path);                        // bytes
Files.isDirectory(path);  Files.isReadable(path);  Files.isWritable(path);
```

### 2.4 Walking a directory tree
```java
// List a directory (one level):
try (Stream<Path> entries = Files.list(Path.of("/home/user"))) {
    entries.forEach(System.out::println);
}
// Walk recursively (all depths):
try (Stream<Path> tree = Files.walk(Path.of("/project"))) {
    tree.filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}
// Find with a matcher:
try (Stream<Path> found = Files.find(start, maxDepth,
        (p, attrs) -> attrs.isRegularFile() && p.toString().endsWith(".log"))) {
    found.forEach(System.out::println);
}
```
> `Files.lines`, `Files.list`, `Files.walk`, `Files.find` return **streams that hold file handles** → always use try-with-resources to close them (recall Phase 1.8 + 1.5).

### 2.5 Why Files over old `File`
| Old `java.io.File` | NIO `Files`/`Path` |
|--------------------|--------------------|
| Returns `boolean` on failure (no detail) | Throws informative `IOException` |
| No charset control | Charset-aware (`readString(path, charset)`) |
| Clunky for read/write | One-line read/write |
| Limited metadata/symlink support | Rich attributes, symlinks, file systems |

---

## 3. Channels & Buffers (Lower-Level, High-Performance)

NIO's lower-level model: **Channels** transfer data to/from **Buffers** (rather than streaming one item at a time). This is more efficient for bulk/binary transfer.

### 3.1 The model
```
Channel  <-- reads/writes -->  Buffer  <-- your code accesses -->
(connection to a file/socket)  (a block of memory holding data)
```
- A **`Channel`** (e.g., `FileChannel`, `SocketChannel`) is a bidirectional connection to a data source.
- A **`Buffer`** (e.g., `ByteBuffer`) is a fixed-size memory block you fill/drain.
- Channels read **into** buffers and write **from** buffers (bulk transfer).

### 3.2 ByteBuffer basics
A `ByteBuffer` has three key markers: **position**, **limit**, **capacity**. You **`flip()`** to switch from writing to reading.
```java
try (FileChannel channel = FileChannel.open(Path.of("data.bin"),
        StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    int bytesRead = channel.read(buffer);   // channel fills the buffer (writing into it)
    buffer.flip();                          // switch to read mode (position=0, limit=end)
    while (buffer.hasRemaining()) {
        byte b = buffer.get();              // drain the buffer
    }
    buffer.clear();                         // reset for the next read
}
```
| Buffer method | Purpose |
|---------------|---------|
| `allocate(n)` | Create a buffer of capacity n |
| `put(...)` / `get(...)` | Write into / read from |
| `flip()` | Switch from writing to reading |
| `clear()` / `compact()` | Reset for writing / keep unread + reset |
| `hasRemaining()` / `remaining()` | Iteration helpers |

### 3.3 When to use channels/buffers
- High-performance file copying (`FileChannel.transferTo`).
- Memory-mapped files (`MappedByteBuffer`) for very large files.
- Non-blocking network I/O with **selectors** (the basis of **Netty**, recall Phase 0.1 I/O multiplexing/epoll).
> For most app code, `Files` is enough. Channels/buffers matter for **performance-critical** or **non-blocking network** I/O — which frameworks (Netty, the JDK HTTP client) handle for you.

---

## 4. Blocking vs Non-Blocking NIO (recap from Phase 0.1)

| | Classic `java.io` | NIO channels |
|---|-------------------|--------------|
| Model | Stream-based | Buffer/channel-based |
| Blocking | Always blocking | Can be **non-blocking** (selectors) |
| Thread per connection | Yes (doesn't scale) | One thread, many connections (epoll/`Selector`) |
| Backs | Simple apps | Netty, reactive servers, high-concurrency I/O |

> NIO's **`Selector`** lets one thread monitor many channels (readiness-based, recall Phase 0.1's I/O multiplexing) — solving the C10K problem. You rarely write this directly; **Netty/WebFlux** (Phase 16) build on it.

---

## 5. WatchService (File System Events)

`WatchService` lets you **monitor a directory for changes** (create/modify/delete) instead of polling — event-driven file watching.
```java
WatchService watcher = FileSystems.getDefault().newWatchService();
Path dir = Path.of("/config");
dir.register(watcher,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_MODIFY,
    StandardWatchEventKinds.ENTRY_DELETE);

WatchKey key = watcher.take();          // blocks until an event occurs
for (WatchEvent<?> event : key.pollEvents()) {
    System.out.println(event.kind() + ": " + event.context());
}
key.reset();
```
> Use cases: hot-reloading config files, detecting dropped files in a directory, build tools. (Spring Boot DevTools uses similar mechanisms for restart.)

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using old `java.io.File` for new code | Use `Path`/`Files` |
| Not closing `Files.lines`/`walk`/`list` streams | They hold file handles — try-with-resources |
| `Files.readString`/`readAllLines` on a huge file | Use `Files.lines` (lazy) for large files |
| Forgetting `buffer.flip()` before reading | Flip to switch write→read mode |
| Relying on platform charset | Pass `StandardCharsets.UTF_8` |
| Polling a directory in a loop | Use `WatchService` (event-driven) |
| Writing channels/selectors by hand | Use Netty/framework for non-blocking network I/O |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **File handling (Phase 5.3):** uploads/downloads, reading templates/config — `Files`/`Path` are cleaner than streams.
- **Large file processing (Project 2):** `Files.lines` streams huge CSVs lazily (memory-safe).
- **Non-blocking I/O (Phase 16):** NIO channels/selectors underpin **Netty** → Spring WebFlux, reactive servers, the JDK HTTP client, and async DB drivers (R2DBC).
- **Config hot-reload / file watching:** `WatchService`-style mechanisms.
- **Performance (Phase 13):** memory-mapped files and `transferTo` for high-throughput file ops.
- **Virtual threads (Phase 1.10):** make blocking `Files`/stream code scale without the complexity of raw NIO selectors.

---

## 8. Quick Self-Check Questions

1. What does NIO add over classic `java.io`?
2. What is a `Path`, and how does it differ from the old `File`?
3. How do you read a whole file vs stream a huge file's lines? Why does the difference matter?
4. Why must you close streams returned by `Files.lines`/`walk`?
5. What's the relationship between channels and buffers?
6. What does `ByteBuffer.flip()` do?
7. When would you use channels/selectors instead of `Files`?
8. What is `WatchService` for, and what does it replace?

---

## 9. Key Terms Glossary

- **NIO / NIO.2:** modern Java I/O (`java.nio`) / the file API (Java 7+).
- **`Path` / `Paths` / `Path.of`:** location descriptor replacing `File`.
- **`Files`:** static utility for file operations (read/write/copy/walk).
- **`Files.lines`/`walk`/`list`:** stream-returning file operations (must close).
- **Channel:** a bidirectional connection to a data source (`FileChannel`, `SocketChannel`).
- **Buffer / `ByteBuffer`:** a fixed memory block for bulk data transfer.
- **`flip()` / `clear()` / `position`/`limit`/`capacity`:** buffer state management.
- **`Selector`:** monitors many channels from one thread (non-blocking).
- **MappedByteBuffer:** memory-mapped file access.
- **`WatchService`:** event-driven file system change notifications.

---

*Previous topic: **Byte, Character & Buffered Streams**.*
*Next topic: **Serialization**.*
