# Serialization

> **Phase 1 — Java Language Mastery → 1.9 Java I/O**
> Goal: Understand Java serialization — the `Serializable` interface, `serialVersionUID`, `transient`, custom serialization — and crucially, **why Java's built-in serialization is problematic** and what to use instead.

---

## 0. The Big Picture

**Serialization** is converting an object into a **byte stream** (so it can be saved to a file, sent over a network, or cached), and **deserialization** is reconstructing the object from those bytes.

```
Object  --serialize-->  bytes (0x...)  --[store/send]-->  bytes  --deserialize-->  Object
```

Java has **built-in serialization** (`Serializable`), but it's widely considered a **historical mistake** with serious security and maintenance problems (§5). Modern backends almost always use **JSON** (Jackson) or other formats instead.

> Know how Java serialization works (you'll encounter it), but **prefer JSON/Protobuf for real applications.**

---

## 1. The Serializable Interface

To make a class serializable with Java's built-in mechanism, implement the **`Serializable`** marker interface (recall marker interfaces, Phase 1.3 — it has no methods; it just "tags" the class as serializable):
```java
public class User implements Serializable {
    private String name;
    private int age;
    // ...
}
```

### 1.1 Serializing / deserializing
Uses the byte-stream classes `ObjectOutputStream`/`ObjectInputStream` (recall previous note):
```java
// Serialize (write object -> bytes -> file):
try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("user.ser"))) {
    oos.writeObject(user);
}

// Deserialize (file -> bytes -> object):
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("user.ser"))) {
    User user = (User) ois.readObject();   // cast needed (returns Object)
}
```

### 1.2 Rules
- The class **and all its non-transient fields' types** must be `Serializable` (transitively) — else `NotSerializableException`.
- Primitives and many JDK types (`String`, collections, wrappers) are already serializable.
- Static fields are **not** serialized (they belong to the class, not the instance).

---

## 2. serialVersionUID (Version Control for Classes)

`serialVersionUID` is a version number identifying a serializable class. During deserialization, Java checks that the stored object's UID **matches** the current class's UID.
```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;   // explicit version
    private String name;
}
```

### 2.1 Why it matters
- If you **don't** declare it, the compiler **auto-generates** one from the class structure. Any change (add a field, change a method) → **different** auto-generated UID → deserializing old bytes throws **`InvalidClassException`**.
- Declaring it **explicitly** gives you control over compatibility:
  - Keep the **same** UID → old serialized data still deserializes (compatible changes).
  - Change the UID → signal an incompatible version.
> **Always declare `serialVersionUID` explicitly** on `Serializable` classes (or you'll get surprise `InvalidClassException`s when the class evolves). IDEs/`serialver` can generate one.

---

## 3. The `transient` Keyword

Mark a field **`transient`** to **exclude it from serialization** — it won't be written, and on deserialization it gets its default value (`null`/`0`/`false`).
```java
public class User implements Serializable {
    private String name;
    private transient String password;     // NOT serialized (sensitive!)
    private transient Connection dbConn;    // NOT serialized (can't/shouldn't persist)
}
```
### 3.1 When to use transient
| Use `transient` for... | Why |
|------------------------|-----|
| **Sensitive data** (passwords, keys) | Don't write secrets to disk/network (Phase 15) |
| **Non-serializable fields** (DB connections, threads, streams) | They can't be meaningfully serialized |
| **Derived/cached values** | Recompute after deserialization instead of storing |
> After deserialization, transient fields are default-valued — reinitialize them if needed (§4).

---

## 4. Custom Serialization

You can customize how an object is serialized by implementing special private methods (the serialization framework calls them via reflection):
```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String derivedField;

    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();          // serialize normal fields
        // custom logic: e.g., encrypt or write extra data
    }
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();           // deserialize normal fields
        this.derivedField = recompute();   // reinitialize transient/derived fields
    }
}
```
- `Externalizable` is a fully manual alternative (you control all bytes) — rarely used.
- `readResolve()`/`writeReplace()` let you control the actual object returned (e.g., preserve a singleton — recall enum singletons handle this automatically, Phase 1.3).

---

## 5. Why Java Serialization Is Problematic (Critical)

> **Java's built-in serialization is widely regarded as a design mistake. Avoid it for new code.** (*Effective Java* Item 85: "Prefer alternatives to Java serialization.")

| Problem | Explanation |
|---------|-------------|
| **Security vulnerabilities** | Deserializing untrusted data can execute arbitrary code (**deserialization attacks** / "gadget chains") — a major source of real CVEs. **Never deserialize untrusted input.** |
| **Fragile versioning** | Class changes break compatibility (`InvalidClassException`); `serialVersionUID` management is error-prone |
| **Hidden, opaque format** | The binary format is Java-specific, not human-readable, not cross-language |
| **Performance** | Slower and larger than modern formats |
| **Bypasses constructors** | Deserialization creates objects **without calling constructors** → can violate invariants/validation (a security & correctness hazard) |
| **Tight coupling** | Ties your persisted/transmitted data to exact Java class internals |

### 5.1 The security issue in one line
> Deserializing bytes from an untrusted source can let an attacker **run code on your server**. This has caused numerous high-profile breaches. If you must use Java serialization, use **serialization filters** (`ObjectInputFilter`, Java 9+) to whitelist allowed classes — but better, **don't use it for untrusted data at all.**

---

## 6. What to Use Instead

| Format | Library | Best for |
|--------|---------|----------|
| **JSON** | **Jackson** (Spring default), Gson | APIs, config, human-readable data — **the default** |
| **Protocol Buffers** | protobuf | High-performance, cross-language, schema-based (recall Phase 0.2 gRPC) |
| **Avro** | Apache Avro | Big-data, Kafka schemas (Phase 11) |
| **MessagePack / CBOR** | various | Compact binary JSON-like |

```java
// JSON with Jackson (the standard in Spring backends):
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);          // serialize -> JSON
User user = mapper.readValue(json, User.class);          // deserialize <- JSON
```
> In Spring backends, **JSON via Jackson is the default** for request/response bodies and most persistence/caching. It's human-readable, cross-language, safe(r), and well-supported (Phase 5.3). Use Protobuf/Avro for performance-critical inter-service comms (Phase 11, 12).

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Using Java serialization for new apps | Use JSON (Jackson) / Protobuf |
| **Deserializing untrusted data** | Don't — it's a remote-code-execution risk |
| Not declaring `serialVersionUID` | Declare it explicitly to control versioning |
| Serializing sensitive fields (passwords) | Mark them `transient` |
| Forgetting a field's type is non-serializable | `NotSerializableException` — make it transient or serializable |
| Expecting constructors to run on deserialize | They don't — validate in `readObject` if needed |
| Assuming serialized form is portable | It's Java-specific; JSON/Protobuf are cross-language |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **JSON (Jackson)** is *the* serialization in Spring — request/response bodies, configuration, REST APIs (Phase 5.3). Master Jackson, not Java serialization.
- **Caching (Phase 5.7):** Redis/cache values are typically serialized as JSON (or sometimes binary) — avoid Java serialization for cache values where possible.
- **Messaging (Phase 11):** Kafka/RabbitMQ payloads use JSON/Avro/Protobuf with schema registries, not Java serialization.
- **Security (Phase 15):** "Insecure Deserialization" is an OWASP risk — Java serialization of untrusted data is a classic attack vector.
- **`transient`** maps conceptually to `@JsonIgnore` (Jackson) for excluding fields (Phase 5.3).
- **Records/DTOs** (Phase 1.3) serialize cleanly to JSON — the modern data-transfer approach.

---

## 9. Quick Self-Check Questions

1. What are serialization and deserialization?
2. What interface enables Java's built-in serialization, and what kind of interface is it?
3. What is `serialVersionUID`, and why should you declare it explicitly?
4. What does `transient` do, and give two reasons to use it?
5. How do you customize serialization (which methods)?
6. List four reasons Java serialization is considered problematic.
7. Why is deserializing untrusted data dangerous?
8. What should you use instead of Java serialization, and what's the default in Spring?

---

## 10. Key Terms Glossary

- **Serialization / deserialization:** object ↔ byte stream conversion.
- **`Serializable`:** marker interface enabling Java serialization.
- **`ObjectOutputStream`/`ObjectInputStream`:** read/write serialized objects.
- **`serialVersionUID`:** class version identifier for compatibility.
- **`InvalidClassException`:** thrown on UID mismatch.
- **`transient`:** excludes a field from serialization.
- **`writeObject`/`readObject`:** hooks for custom serialization.
- **`Externalizable`:** fully manual serialization control.
- **Deserialization attack:** RCE via deserializing malicious bytes.
- **`ObjectInputFilter`:** whitelist filter for safer deserialization.
- **Jackson / JSON:** the standard serialization in Spring backends.
- **Protobuf / Avro:** efficient, cross-language, schema-based formats.

---

*Previous topic: **NIO (New I/O)**.*
*This completes **Section 1.9 — Java I/O**.*
*Next section in roadmap: **1.10 Java Concurrency**.*
