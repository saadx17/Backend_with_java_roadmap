In Java, a **String** is an object that represents a sequence of characters. It is implemented via the `java.lang.String` class and is **immutable**, meaning its value cannot be altered after it is created.
Since String is a class, it comes with many **built-in methods**.  
Remember: Strings are **immutable**, so these methods always return a **new String**.

```
String → sequence of characters
       → Object in Java (not primitive)
       → Stored in HEAP memory
       → IMMUTABLE (cannot be changed after creation)

Special treatment:
├── Has its own literal syntax → "hello"
├── Has a String Pool (memory optimization)
├── Most used class in Java
└── Overloads + operator for concatenation
```
# Methods of Creation
You can instantiate strings using two distinct mechanisms:

- **String Literal**: Created by using double quotes (e.g., `String s = "Hello";`). The JVM checks the **String Constant Pool** first; if the sequence exists, it reuses the reference to conserve heap memory.
- **`new` Keyword**: Created via standard constructor instantiation (e.g., `String s = new String("Hello");`). This explicitly forces the creation of a brand-new object in the regular heap memory, bypassing pool optimization.

# String Immutability
In Java, string immutability means that once a `String` object is created in memory, its content cannot be modified.

Any operation that appears to alter a string (like concatenation, replacement, or converting to uppercase) does not change the original object; instead, it **creates a completely new `String` object** containing the modified value.


Once created, String content CANNOT change, Any "modification" creates a NEW string object.
```java title:string_immu.java
String s = "Hello";
s.toUpperCase();                    // creates new String "HELLO"
System.out.println(s);             // still "Hello" ← unchanged!

// Must reassign to use result
String upper = s.toUpperCase();
System.out.println(upper);         // "HELLO"
System.out.println(s);             // "Hello" ← still unchanged

// Every operation = new object
String a = "Hello";
String b = a.concat(" World");     // new String created
String c = b.replace("World","Java"); // another new String

System.out.println(a);  // "Hello"         ← unchanged
System.out.println(b);  // "Hello World"   ← unchanged
System.out.println(c);  // "Hello Java"    ← new string
```

## Why Immutable?
In Java, **`String` is immutable** because its value cannot be changed after creation, a design choice meant to guarantee **security, memory efficiency, concurrency safety, and performance**. If you alter a string, Java creates an entirely new string object instead of modifying the existing one.

```
Security:
├── Strings used in passwords, URLs, DB connections
└── Immutable = can't be changed after validation

Thread Safety:
├── Multiple threads can share same String safely
└── No synchronization needed

String Pool (Performance):
├── Same literal reused across program
└── Saves memory

HashCode Caching:
└── hashCode calculated once, cached (used in HashMap)
```

## How Immutability Works Internally?
Internally, immutability in Java is not enforced by a special "immutable" memory zone; rather, it is a strict combination of **class design constraints, access modifiers, and Java Virtual Machine (JVM) memory barriers**.

```java title:internal_work.java
// String concatenation in loop → PERFORMANCE TRAP ⚠️
String result = "";
for (int i = 0; i < 5; i++) {
    result += i;    // creates NEW string every iteration!
}
// Creates: "", "0", "01", "012", "0123", "01234"
// 6 objects created → bad for large loops

// Internally:
// result = result + i
// result = new String(result + i)  ← new object each time!

// ✅ Use StringBuilder instead (explained later)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 5; i++) {
    sb.append(i);   // modifies same object
}
String result = sb.toString();  // "01234"
```


