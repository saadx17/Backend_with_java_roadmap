# The Java Data Triangle
To understand the connection between [[#Collections Framework|Collections]], [[Wrapper Classes]] & [[Generics]], you must look at how data travels from your keyboard into the [[JVM Architecture|JVM]]'s memory.

**1. Collections Need Objects (The Foundation)** The Java Collections Framework (`ArrayList`, `HashMap`, etc.) was designed to hold generic pointers to memory. Therefore, the underlying data structure of an `ArrayList` is an `Object[]` array. Collections **cannot** store raw primitive data (`int`, `double`, `boolean`) because primitives are not Objects; they do not have memory references, methods, or class metadata.

**2. Wrapper Classes Bridge the Gap (The Translator)** Because Collections refuse to accept primitives, **Wrapper Classes** (`Integer`, `Double`, `Boolean`) act as the necessary translators. They take a raw, 4-byte primitive `int` and "wrap" it inside a fully-fledged Object on the JVM heap. Through **Autoboxing**, the compiler does this for you automatically so that your primitive data is acceptable to the Collection.

**3. Generics Enforce the Rules (The Bouncer)** Because Collections use an `Object[]` array under the hood, they will happily accept _any_ Object. You could accidentally put a `String` and a `Double` into the same list. **Generics** (`<Integer>`) act as a strict compile-time contract. They force the compiler to check that the Object being added (which was just autoboxed by the Wrapper class) matches the specific type the Collection expects.

#### The Pipeline in Action
When you write this single line of code, all three systems trigger in sequence:

```java title:dataTriangle.java
List<Integer> scores = new ArrayList<>();
scores.add(95); 
```

Behind the scenes, the compiler translates `scores.add(95);` into this pipeline:

1. **The Raw Data:** You provide `95` (a primitive `int`).

2. **The Wrapper (Autoboxing):** The compiler injects `Integer.valueOf(95)` to convert the primitive into an Object.

3. **The Generic Contract:** The compiler verifies that the resulting `Integer` object matches the `<Integer>` generic tag on your list. (If you tried to add `95.5`, this step would throw a compile error).

4. **The Collection Storage (Type Erasure):** The code compiles successfully. At runtime, the JVM erases the `<Integer>` tag, looks at the underlying `Object[]` array inside the `ArrayList`, and saves the memory reference of your `Integer` object into index 0.


# Collections Framework
A [java collections](https://www.w3schools.com/JAVA/java_collections.asp) is a unified architecture used to store, organize, and manipulate groups of data as a single unit. It provides ready-made data structures (like lists, queues, and maps) and built-in algorithms (for sorting and searching), saving developers from having to build them from scratch.

![[collection-framework.png]]

#### 1. The Core Hierarchy
The framework splits into two distinct trees:

- **`Collection` Interface:** Stores individual elements. Splits into `List` (ordered, allows duplicates), `Set` (unordered, unique elements), and `Queue` (processing order).
- **`Map` Interface:** Stores Key-Value pairs. It does _not_ inherit from `Collection`, but it is heavily integrated into the framework.

#### 2. ArrayList Internals (Dynamic Arrays)
An `ArrayList` is a wrapper around a standard, fixed-size primitive array of Objects (`Object[]`).

- **Default Capacity:** Starts at 10 (lazily initialized upon the first `add()` operation).
 
- **The Resize Mechanic:** When you try to add the 11th element, Java cannot simply "stretch" the array in memory. It must create a brand-new array.

- **Growth Factor:** It grows by 50%. The underlying bitwise shift is $newCapacity = oldCapacity + (oldCapacity \gg 1)$. It copies the old elements into the new array using the highly optimized `System.arraycopy()` native method.

- **Industry Trap:** Resizing a 10-element array is fast. Resizing a 10-million-element array is an expensive CPU operation that blocks the thread. **Pro-tip:** If you know you are fetching 5,000 records from a database, initialize the list with that capacity: `List<User> users = new ArrayList<>(5000);`.

#### 3. HashMap Internals
If you master `HashMap`, you understand hashing, bitwise operations, linked lists, and trees. A `HashMap` is an array of "buckets" (Nodes).

- **Hashing & Indexing:** When you call `put(key, value)`, Java calculates the key's `hashCode()`. To figure out which array index (bucket) this entry goes into, Java uses a bitwise AND operation rather than the slower modulo operator: $index = (n - 1) \ \& \ hash$ (where $n$ is the array length).

- **Collisions:** If two different keys resolve to the exact same bucket index, a collision occurs. Java handles this by chaining them together in a Singly Linked List at that bucket.

- **Treeification (Java 8+):** If a single bucket gets too many collisions (8 or more elements) and the total table size is at least 64, that Linked List transforms into a **Red-Black Tree**. This degrades the search time gracefully from $O(N)$ (scanning a list) to $O(\log N)$ (searching a tree), protecting your backend from Denial of Service (DoS) attacks via engineered hash collisions.

- **Resizing & Load Factor:** The default capacity is 16, and the default load factor is 0.75. When the map becomes 75% full (12 elements), it doubles its array size to 32 and recalculates the index for every single element (a massive performance hit called "rehashing").

#### 4. Sets and Backing Maps
A common misconception is that a `HashSet` is a unique data structure. Under the hood, **a `HashSet` is quite literally just a `HashMap`**.

When you add a value to a `HashSet`, Java places your value as the _key_ in a `HashMap`, and puts a dummy static Object as the _value_. This is why `HashSet` guarantees uniqueness—keys in a map must be unique.

#### 5. Thread Safety and Concurrency

Neither `ArrayList` nor `HashMap` are thread-safe. If multiple threads write to a standard `HashMap` simultaneously, it can corrupt the data structure.

- **Legacy approaches:** `Vector` and `Hashtable` synchronize every single method. This locks the entire collection during a read or write, creating a massive bottleneck in web servers.

- **Modern approach:** `ConcurrentHashMap`. Instead of locking the whole map, it uses **Lock Striping** (in Java 7) and Compare-And-Swap (CAS) operations with node-level locks (in Java 8+). This means Thread A can write to Bucket 2 while Thread B writes to Bucket 8 simultaneously without blocking each other.

