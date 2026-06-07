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

# String Pool
The **Java String Pool** (also known as the **String Constant Pool**) is a dedicated memory region inside the Java Heap memory that stores unique string literals. Its primary purpose is to **optimize memory usage and enhance performance** by reusing existing string objects instead of creating duplicate instances.

## How It Works?
The **Java String Pool** (also known as the String Constant Pool or SCP) is a specialized memory region located within the **Java Heap** that caches string literals to optimize memory allocation and performance. Because `String` objects are immutable in Java, multiple references can safely point to a single instance in the pool without risking unintended data modifications.

```
String Pool (also called String Intern Pool):
├── Special memory area inside HEAP (Java 8+)
├── Stores unique String literals
├── JVM reuses existing strings instead of creating new
└── Saves memory when same string used multiple times
```

```java title:string-pool_works.java
// String literals → use pool
String a = "Hello";   // JVM checks pool → not found → adds "Hello"
String b = "Hello";   // JVM checks pool → found! → reuses same object
String c = "Hello";   // same object again

System.out.println(a == b);  // true  ← same object in pool!
System.out.println(b == c);  // true  ← same object in pool!

// new String() → bypasses pool
String x = new String("Hello");  // forces new object in heap
String y = new String("Hello");  // another new object in heap

System.out.println(x == y);      // false ← different objects
System.out.println(x.equals(y)); // true  ← same content
```

```java title:visuals.java
// Visual
//
//  String Pool         Heap
//  ┌──────────┐       ┌──────────┐
//  │ "Hello"  │◄──a   │ "Hello"  │◄──x
//  │          │◄──b   │ "Hello"  │◄──y
//  │          │◄──c   └──────────┘
//  └──────────┘
//   same object         different objects
```

## intern() Method
The **`public String intern()`** method in Java returns a **canonical representation** of a string object by ensuring it is stored in the **String Constant Pool (SCP)**. It is a memory optimization tool used to eliminate duplicate strings and allow safe `==` reference comparisons.

```java title:intern.java
// intern() → forces string into pool
// if pool has it → return pool reference
// if not → add to pool and return reference

String x = new String("Hello");  // heap object, not in pool
String y = x.intern();           // puts in pool (or gets existing)
String z = "Hello";              // gets from pool

System.out.println(x == y);  // false (x is heap, y is pool)
System.out.println(y == z);  // true  (both from pool)

// Rarely used in practice
// String literals are auto-interned by JVM
```

# String Comparison
To compare strings in Java, **always use the `.equals()` method** instead of the `==` operator if you want to compare their actual text content.

## `==` vs `.equals()` vs `.equalsIgnoreCase()`
In Java, the core difference is that **`==` compares memory references (identity)**, **`.equals()` compares exact text content (case-sensitive)**, and **`.equalsIgnoreCase()` compares text content while ignoring uppercase/lowercase differences**.

| Features             | `==` Operator                   | `.equals()` Method                                | `.equalsIgnoreCase()` Method                      |
| -------------------- | ------------------------------- | ------------------------------------------------- | ------------------------------------------------- |
| **Comparison Type**  | Reference / Memory address      | Content / Values                                  | Content / Values                                  |
| **Case Sensitivity** | Not applicable                  | Strict case-sensitive                             | Case-insensitive                                  |
| **Best Used For**    | Primitive types (`int`, `char`) | Exact matching (Passwords)                        | Flexible matching (Usernames, Emails)             |
| **Null Safety**      | Safe (returns false or true)    | Throws `NullPointerException` if called on `null` | Throws `NullPointerException` if called on `null` |

```java title:dec.java
String a = "hello";
String b = "hello";
String c = new String("hello");
String d = "HELLO";
```

```java title:compare_ref.java
// == compares REFERENCES
System.out.println(a == b);    // true  (both from pool)
System.out.println(a == c);    // false (c is heap object)
```

```java title:equals.java
// .equals() compares CONTENT
System.out.println(a.equals(b));    // true
System.out.println(a.equals(c));    // true
System.out.println(a.equals(d));    // false (case sensitive)
System.out.println(a.equals(null)); // false (no NPE)
```

```java title:equalsIgnoreCase.java
// .equalsIgnoreCase() → ignores case
System.out.println(a.equalsIgnoreCase(d));  // true
System.out.println(a.equalsIgnoreCase("HeLLo")); // true
```

```java title:compareTo.java
// .compareTo() → lexicographic comparison
System.out.println("apple".compareTo("banana")); // negative (a < b)
System.out.println("banana".compareTo("apple")); // positive (b > a)
System.out.println("apple".compareTo("apple"));  // 0 (equal)
```

```java title:sorting.java
// Used for sorting strings alphabetically
String[] fruits = {"Banana", "Apple", "Cherry"};
Arrays.sort(fruits);  // uses compareTo internally
System.out.println(Arrays.toString(fruits)); // [Apple, Banana, Cherry]
```

## Safe Comparison Patterns
```java title:safe_comparison.java
String input = getUserInput();  // might be null

// Dangerous
if (input.equals("admin")) { }     // NPE if input is null!
if (input == "admin") { }          // wrong! reference comparison

// Null safe: literal first (Yoda condition)
if ("admin".equals(input)) { }     // no NPE even if input is null

// Null safe: explicit null check
if (input != null && input.equals("admin")) { }

// Java 7+: Objects.equals()
import java.util.Objects;
if (Objects.equals(input, "admin")) { }  // handles null on both sides
```

