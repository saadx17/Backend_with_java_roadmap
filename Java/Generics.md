[Generics](https://www.geeksforgeeks.org/java/generics-in-java/) in Java refer to parameterized types that allow writing code which works with multiple data types using a single class, interface, or method. They improve reusability and ensure type safety at compile time.

#### Why we use Generics?
- Before Generics, Java collections like ArrayList or HashMap could store any type of object, everything was treated as an Object. It had some problems.
- If you added a String to a List, Java didn’t remember its type. You had to manually cast it when retrieving. If the type was wrong, it caused a runtime error.
- With Generics, you can specify the type the collection will hold like `ArrayList<String>`. Now, Java knows what to expect and it checks at compile time, not at runtime.

### 1. Naming Conventions (The Standard)
When building generic classes or methods, industry standards dictate using single uppercase letters to represent the unknown type. This makes the code instantly readable.

- **`T`** - Type (The most common, e.g., `class Box<T>`)

- **`E`** - Element (Used extensively by the Java Collections Framework, e.g., `List<E>`)

- **`K`** - Key (Used in Maps, e.g., `Map<K, V>`)

- **`V`** - Value (Used in Maps)

- **`N`** - Number

### 2. Creating Generic Classes and Methods
You can apply generics at the class level or the method level.

**Generic Class:** The generic type is declared after the class name and is available to all non-static methods inside the class.
```java title:genericsClass.java
public class ApiResponse<T> {
    private int statusCode;
    private T payload; // The payload can be a User, a String, or a List

    public T getPayload() { return payload; }
    public void setPayload(T payload) { this.payload = payload; }
}
```


**Generic Method:** Sometimes you don't need the whole class to be generic, just a single method. The generic type `<T>` must be declared _before_ the return type.
```java title:genericsMethod.java
public class ArrayUtil {
    // This method can take an array of any object type and swap elements
    public static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
}
```

### 3. Bounded Type Parameters (Constraining the Unknown)
Often, you want to use generics, but you need to guarantee that the type has certain behaviors. For example, if you write a sorting algorithm, you need to ensure the generic type can be compared.

You use the `extends` keyword (which in the context of generics means either `extends` a class or `implements` an interface).
```java title:generics.java
// T must be a subclass of Number AND implement Comparable
public <T extends Number & Comparable<T>> void processNumber(T num) {
    System.out.println(num.intValue()); // Safe because T is guaranteed to be a Number
}
```

### 4. Wildcards and the PECS Rule (Interview Goldmine)
When you are designing APIs that accept collections, you use Wildcards (`?`) to provide flexibility. The absolute most important rule in Java API design is **PECS: Producer Extends, Consumer Super**.

1. **`? extends T` (Upper Bound):** The collection is a **Producer** of data. You want to _read_ `T` from it.
- **Rule:** You can read elements (you know they will be `T` or a subclass of `T`). You **CANNOT add anything** to this list (except `null`) because the compiler doesn't know the _exact_ subclass the list holds.

- _Example:_ A method calculating the sum of a list. `public double sum(List<? extends Number> list)`

2. **`? super T` (Lower Bound):** The collection is a **Consumer** of data. You want to _write_ `T` into it.

- **Rule:** You can safely add instances of `T` (or its subclasses). However, when you read from it, you are only guaranteed to get an `Object`.

- _Example:_ A method that adds default users to a list. `public void addDefaults(List<? super User> list)`

3. **`?` (Unbounded Wildcard):** Equivalent to `? extends Object`. Used when you only care about the collection itself (like `list.size()`) and not the types inside it.

### 5. The Hard Restrictions (What You Cannot Do)
Because of **Type Erasure** (the compiler erasing generics at runtime to maintain backward compatibility), there are strict limitations. Put these in your notes:

1. **Cannot instantiate Generic Types with Primitives:** You must use Wrappers. (e.g., `List<int>` is illegal; use `List<Integer>`).

2. **Cannot instantiate Generic Objects:** `T instance = new T();` is illegal because at runtime, `T` is erased to `Object`. The JVM doesn't know what constructor to call.

3. **Cannot declare Static Generic Fields:** `private static T info;` is illegal. Static fields are shared across all instances of a class, but `T` belongs to a specific instance's declaration (e.g., `Box<String>` vs `Box<Integer>`).

4. **Cannot instantiate Generic Arrays:** `T[] array = new T[10];` and `List<String>[] array = new List<String>[10];` are illegal due to how arrays enforce type safety at runtime vs how generics enforce it at compile time.