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

# String Methods
In Java, **[String methods](https://www.geeksforgeeks.org/java/java-string-methods/)** are built-in functions provided by the `java.lang.String` class to perform actions on text. Because strings in Java are **immutable** (cannot be changed after creation), every method that modifies a string actually returns a brand-new string object.
## Inspection Methods
**Inspection methods of a String in Java** are built-in functions belonging to the `java.lang.String` class that allow you to read, search, or evaluate metadata about text without changing the original string. Unlike modification methods, these operations return primitives, booleans, or specific character components.

#### `length()`
Number of characters.
```java title:length.java
String name = "Ahmed";
System.out.println(name.length()); // 5

String empty = "";
System.out.println(empty.length()); // 0

String sentence = "Hello World"; System.out.println(sentence.length()); // 11 (space counts!)
```

#### `charAt(index)`
Character at a position. Returns the **char** at the given index (0-based).
```java title:charAt.java
String name = "Ahmed";
//             01234 ← indexes

System.out.println(name.charAt(0)); // A
System.out.println(name.charAt(2)); // m
System.out.println(name.charAt(4)); // d

// name.charAt(5) → ❌ StringIndexOutOfBoundsException
```

#### `indexOf()`
Find position of a character or String. Returns the **index** of the first occurrence.  Returns **-1** if not found.
```java title:indexOf.java
String text = "Hello World";
System.out.println(text.indexOf('o')); // 4 (first 'o')
System.out.println(text.indexOf('o', 5)); // 7 (search starting from index 5)
System.out.println(text.indexOf("World")); // 6
System.out.println(text.indexOf('z')); // -1 (not found)
System.out.println(text.indexOf("xyz")); // -1 (not found)
```

#### `contains()`
Check if String contains a value. Returns `boolean` true if found, false if not.
```java title:contains.java
String email = "ahmed@gmail.com";
System.out.println(email.contains("@")); // true
System.out.println(email.contains("gmail")); // true
System.out.println(email.contains("yahoo")); // false
```

#### `startsWith()` & `endsWith()`
checks whether a string begins or ends with a specific sequence of characters. Both methods are **case-sensitive** and return a boolean value (`true` or `false`).
```java title:startsWith_endsWith.java
System.out.println(s.startsWith("Hello")); // true
System.out.println(s.endsWith("!"));       // true
System.out.println(s.startsWith("World", 7)); // true (from index 7)
```

#### `isEmpty()` & `isBlank()`
`isEmpty()` checks if the string length is exactly 0, while `isBlank()` checks if the string is empty or contains only whitespace characters.
```java title:isEmpty_isBlank.java
System.out.println("".isEmpty());          // true
System.out.println("  ".isEmpty());        // false (has spaces)
System.out.println("  ".isBlank());        // true  (Java 11+)
System.out.println("hi".isEmpty());        // false
```

## Transformation Methods
In Java, string transformation can refer specifically to the modern **`transform()` method** introduced in Java 12, or generally to standard text manipulation methods. Because strings are **immutable**, all of these methods return a new string instance or a completely new object type without modifying the original text.

#### `toUpperCase()` & `toLowerCase()`
Changes the casing of text. Because strings in Java are immutable, these methods do not change the original string; instead, they return a brand new string with the altered case.
```java title:up_low_case.java
String name = "Ahmed Ali";
System.out.println(name.toUpperCase()); // "AHMED ALI"
System.out.println(name.toLowerCase()); // "ahmed ali"
// Useful for case-insensitive comparison
String input = "YES";
if (input.toLowerCase().equals("yes")) { System.out.println("User said yes");
}
```

#### `trim()` & `strip()`
Removes leading and trailing whitespace from a string, but they handle different types of whitespace characters based on Unicode standards.
```java title:Trim_Strip.java
System.out.println(s.trim());          // "Hello, World!" (removes spaces)
System.out.println(s.strip());         // "Hello, World!" (Java 11+, Unicode aware)
System.out.println(s.stripLeading());  // "Hello, World!  "
System.out.println(s.stripTrailing()); // "  Hello, World!"
```

#### `replace()`
Replace characters or substrings.
```java title:replace.java
String text = "Hello World";
System.out.println(text.replace('l', 'x')); // "Hexxo Worxd"
System.out.println(text.replace("World", "Java")); // "Hello Java"
System.out.println(text.replace("l", "LL")); // "HeLLLLo WorLLd"

// Original is unchanged!
System.out.println(text); // "Hello World"
```

#### `substring()`
Extract part of a String.
Two versions:
- `substring(startIndex)` → from startIndex to end

- `substring(startIndex, endIndex)` → from startIndex up to (not including) endIndex.
```java title:substring.java
String text = "Hello World";
//             01234567890

System.out.println(text.substring(6)); // "World" (from index 6 to end)

System.out.println(text.substring(0, 5)); // "Hello" (0 to 4, not including 5)

System.out.println(text.substring(6, 11)); // "World" (6 to 10, not including 11)
```

#### `concat()`
Appends a specified string to the end of an existing string and returns a brand-new string object.
```java title:concat.java
String hello = "Hello";
String world = " World";
System.out.println(hello.concat(world));  // "Hello World"
System.out.println(hello + world);        // "Hello World" (easier)
```

## Splitting & Joining
In Java, you can split a string into an array using the **`String.split()`** method and merge them back together using the static **`String.join()`** method.

```java title:splitting.java
// split
String csv = "Alice,Bob,Charlie,Dave";
String[] names = csv.split(",");
System.out.println(Arrays.toString(names)); // [Alice, Bob, Charlie, Dave]
System.out.println(names[0]);               // Alice
System.out.println(names.length);           // 4

// Split with limit
String[] limited = csv.split(",", 2);  // max 2 parts
System.out.println(Arrays.toString(limited)); // [Alice, Bob,Charlie,Dave]

// Split on whitespace
String sentence = "Hello World Java";
String[] words = sentence.split("\\s+");  // one or more spaces
System.out.println(Arrays.toString(words)); // [Hello, World, Java]

// Split on multiple delimiters
String data = "one,two;three|four";
String[] parts = data.split("[,;|]");
System.out.println(Arrays.toString(parts)); // [one, two, three, four]
```

```java title:joining.java
// join
String joined = String.join(", ", "Alice", "Bob", "Charlie");
System.out.println(joined);  // Alice, Bob, Charlie

String[] arr = {"one", "two", "three"};
String result = String.join(" - ", arr);
System.out.println(result);  // one - two - three

// join with List
List<String> list = List.of("a", "b", "c");
System.out.println(String.join("|", list));  // a|b|c
```

## Conversion Methods
In Java, conversion methods involving strings broadly fall into three categories: **converting other data types to a `String`**, **converting a `String` to other data types**, and **modifying case within a `String`**.

```java title:conversion.java
// String → other types
int    num  = Integer.parseInt("42");
double d    = Double.parseDouble("3.14");
long   l    = Long.parseLong("123456789");
boolean b   = Boolean.parseBoolean("true");

// Other types → String
String s1 = String.valueOf(42);          // "42"
String s2 = String.valueOf(3.14);        // "3.14"
String s3 = String.valueOf(true);        // "true"
String s4 = String.valueOf('A');         // "A"
String s5 = Integer.toString(42);        // "42"
String s6 = "" + 42;                     // "42" (implicit)

// char array ↔ String
char[] chars = {'H', 'e', 'l', 'l', 'o'};
String str   = new String(chars);        // "Hello"
char[] back  = str.toCharArray();        // back to char[]

// String → char array iteration
String word = "Hello";
for (char c : word.toCharArray()) {
    System.out.print(c + " ");  // H e l l o
}
```

## String.format & printf
In Java, `String.format()` and `System.out.printf()` use identical formatting rules, but **`String.format()` creates and returns a new string** without printing it, while **`System.out.printf()` prints the formatted text directly to the console output**.

#### `String.format()`
The method in Java is a static utility used to create a formatted string by substituting placeholders (format specifiers) with variable values. It returns a new string rather than modifying the original one or printing it directly.
```java title:string_format.java
// String.format
String name  = "Alice";
int    age   = 25;
double score = 95.678;

String formatted = String.format("Name: %s, Age: %d, Score: %.2f",
                                  name, age, score);
System.out.println(formatted);
// Name: Alice, Age: 25, Score: 95.68


// Format specifiers
// %s  → String
// %d  → integer
// %f  → float/double
// %.2f→ float with 2 decimal places
// %n  → newline
// %b  → boolean
// %c  → char
// %10s→ right-align in 10 chars
// %-10s → left-align in 10 chars


// Formatted table 
System.out.printf("%-15s %5s %8s%n", "Name", "Age", "Score");
System.out.printf("%-15s %5d %8.2f%n", "Alice",   25, 95.67);
System.out.printf("%-15s %5d %8.2f%n", "Bob",     30, 87.33);
System.out.printf("%-15s %5d %8.2f%n", "Charlie", 22, 91.50);

// Name              Age    Score
// Alice              25    95.67
// Bob                30    87.33
// Charlie            22    91.50


// Java 15+ formatted() instance method
String result = "Hello %s!".formatted("World");
System.out.println(result);  // Hello World!
```

