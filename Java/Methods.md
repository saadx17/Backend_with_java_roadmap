A **method** is a named, reusable block of code that defines an object's (or class's) **behavior**. It can take inputs (**parameters**), perform work, and optionally produce an output (**return value**).

```java title:syntax.java
public int add(int a, int b) {   // method
    return a + b;
}
```

#### Anatomy of a Method
In Java, a method declaration defines the attributes and execution logic of a function within a class. According to the [Oracle Java Tutorials](https://docs.oracle.com/javase/tutorial/java/javaOO/methods.html), a full method declaration consists of **six distinct components** that must appear in a specific order.
```java title:anatomy.java
//  access    return
//  modifier  type    name        parameters
//  ↓         ↓       ↓           ↓
    public    int     add(        int a, int b) {
        return a + b;   // ← return statement
    }

// Full syntax
accessModifier returnType methodName(param1, param2, ...) {
    // method body
    return value;  // if returnType is not void
}
```

| Part | Meaning |
|------|---------|
| **Modifiers** | Access (`public`/`private`/...) + others (`static`, `final`, `abstract`, `synchronized`) |
| **Return type** | Type of value returned, or `void` for none |
| **Name** | Identifier (camelCase, usually a verb: `calculateTotal`) |
| **Parameter list** | Zero or more typed inputs |
| **Body** | The code that runs |

## 1. Instance & Static Methods
This is the most fundamental distinction.

### 1.1 Instance methods
- Belong to an **object (instance)**. Require an object to be called.
- Can access **instance fields** and **other instance methods** (they have a `this` reference).
- Operate on the *specific object's* state.

```java
public class Counter {
    private int count;                 // instance field

    public void increment() {          // instance method
        count++;                       // uses this.count
    }
    public int getCount() { return count; }
}

Counter c = new Counter();
c.increment();    // called ON an object
```

### 1.2 Static methods
- Belong to the **class itself**, not any object. Called on the **class name**.
- **Cannot** access instance fields/methods directly (there is **no `this`**).
- Can only directly use **static fields** and other **static methods**.
- Used for utility/helper functions that don't depend on object state.

```java
public class MathUtils {
    public static int square(int x) {  // static method
        return x * x;
    }
}

int r = MathUtils.square(5);   // called on the CLASS, no object needed
```

### 1.3 Instance Vs. Static Methods

| Aspect | Instance method | Static method |
|--------|-----------------|---------------|
| Belongs to | An object | The class |
| Called via | `object.method()` | `ClassName.method()` |
| Has `this`? | Yes | **No** |
| Can access instance members? | Yes | No (not directly) |
| Can access static members? | Yes | Yes |
| Participates in overriding/polymorphism? | **Yes** (dynamic dispatch) | **No** (hidden, not overridden) |
| Typical use | Behavior tied to object state | Stateless utilities, factories |

> ⚠️ **Static methods are not polymorphic.** If a subclass declares a static method with the same signature, it **hides** (not overrides) the parent's — the version called depends on the *reference type*, decided at compile time. (Detail revisited in the Polymorphism note.)

### 1.4 Common rules & gotchas
- You **can't** use `this` or `super` in a static method.
- Calling a static method via an instance (`obj.staticMethod()`) compiles but is **bad style**, call it via the class.
- `main` is `static` because the JVM calls it **before** any object exists:
  ```java
  public static void main(String[] args) { ... }
  ```

## 2. Method Signatures and Overloading

### 2.1 What is a method signature?
A **method signature** = the method **name + parameter list** (types, number, and order of parameters).

```
add(int, int)        // signature
add(double, double)  // DIFFERENT signature
```

> **The return type is NOT part of the signature.** Neither are parameter *names* or access modifiers. This is why you can't have two methods that differ *only* by return type.

### 2.2 Method overloading
**Overloading** = multiple methods with the **same name** but **different signatures** (different parameter lists) in the same class. The compiler picks which one to call based on the arguments - this is **compile-time (static) polymorphism**.

```java
public class Printer {
    void print(int x)            { System.out.println("int: " + x); }
    void print(double x)         { System.out.println("double: " + x); }
    void print(String x)         { System.out.println("String: " + x); }
    void print(int x, int y)     { System.out.println("two ints"); }
}
```

**Valid overloads differ by:**
- **Number** of parameters
- **Types** of parameters
- **Order** of parameter types

```java
void f(int a, String b) {}
void f(String a, int b) {}   // OK: different order
```

**NOT valid** (won't compile - return type / param name don't count):
```java
int  g(int x) {return x;}
// double g(int x) { return x; }  // ERROR: same signature, only return differs
```

### 2.3 How the compiler resolves an overloaded call
When you call an overloaded method, the compiler chooses in this priority order:
1. **Exact match** of types.
2. **Widening** primitive conversion (e.g., `int` → `long` → `double`).
3. **Autoboxing/unboxing** (e.g., `int` → `Integer`).
4. **Varargs** (last resort).

```java
void m(long x)    { System.out.println("long"); }
void m(Integer x) { System.out.println("Integer"); }
m(5);   // prints "long"  -> widening preferred over autoboxing
```

> This resolution is decided **at compile time** based on the *static (declared) type* of arguments - a key contrast with overriding (runtime). Overloading vs overriding is a classic interview topic.

| | Overloading | Overriding |
|---|-------------|-----------|
| Where | Same class (or inherited) | Subclass redefines parent method |
| Signature | **Different** params | **Same** signature |
| Bound at | Compile time | Runtime (dynamic dispatch) |
| Return type | Can differ | Same or covariant |
| Polymorphism type | Static | Dynamic |

## 3. Varargs (Variable Arguments)
**Varargs** let a method accept a **variable number of arguments** of the same type, using `...`.

```java
public int sum(int... numbers) {     // varargs
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}

sum();              // 0   -> empty
sum(1, 2);          // 3
sum(1, 2, 3, 4, 5); // 15
sum(new int[]{1,2});// 3   -> can also pass an array
```

### 3.1 How it works
- Internally, `int... numbers` is treated as an **array** (`int[]`). The compiler bundles the loose arguments into an array.
- You iterate it like any array; `numbers.length` gives the count.

### 3.2 Rules
1. A method can have **only one** varargs parameter.
2. It **must be the last** parameter:
   ```java
   void log(String level, String... messages) {}   // OK
   // void bad(String... msgs, String level) {}     // ERROR
   ```
3. You can mix fixed + varargs params (fixed ones come first).

### 3.3 Gotchas
- **Ambiguity with overloading:** the compiler prefers a more specific (non-varargs) overload:
  ```java
  void p(int a, int b) {}
  void p(int... a)     {}
  p(1, 2);   // calls p(int,int) -> exact match wins over varargs
  ```
- Passing `null` to varargs can be ambiguous; be explicit if needed.
- Common real uses: `String.format("%s %d", name, age)`, logging APIs, `List.of(...)`, `Arrays.asList(...)`.

## 4. Pass-by-Value
Java is ALWAYS Pass-by-Value.
> **This is one of the most misunderstood topics in Java. Memorize: Java is ALWAYS pass-by-value. There is no pass-by-reference in Java.**

### 4.1 What "pass-by-value" means
When you pass an argument to a method, Java passes a **copy of the value** of that variable. The method works on the copy; changes to the *parameter itself* do not affect the caller's variable.

### 4.2 Primitives - clearly pass-by-value
```java
void modify(int x) {
    x = 99;             // changes only the local copy
}

int a = 5;
modify(a);
System.out.println(a);  // 5  -> unchanged
```
The method got a copy of `5`; reassigning `x` did nothing to `a`.

### 4.3 Objects - the confusing part (still pass-by-value!)
For objects, **the value being copied is the reference (the address), not the object.** So the copy points to the *same object*. This means:
- You **can** mutate the object's state through the copied reference.
- You **cannot** make the caller's variable point to a different object.

```java
void mutate(StringBuilder sb) {
    sb.append(" world");      // mutates the SHARED object -> visible to caller
}
void reassign(StringBuilder sb) {
    sb = new StringBuilder("new");  // reassigns the COPY -> caller unaffected
}

StringBuilder s = new StringBuilder("hello");
mutate(s);
System.out.println(s);   // "hello world"  -> object was mutated

reassign(s);
System.out.println(s);   // "hello world"  -> still! reassign didn't affect caller
```

### 4.4 The mental model (ties to Phase 0 stack/heap)
```
Caller variable 's' (stack) ----> [ StringBuilder object on HEAP ]
                                       ^
Method parameter 'sb' (stack) --------+   (a COPY of the reference)

mutate():   sb.append(...)  -> follows the arrow -> changes the shared heap object
reassign(): sb = new(...)   -> repoints ONLY sb's copy -> 's' still points to original
```

### 4.5 The classic "swap" proof
```java
void swap(Point p1, Point p2) {
    Point temp = p1;
    p1 = p2;       // swaps the COPIES of references
    p2 = temp;
}                  // caller's references are unchanged -> swap "fails"
```
This famous failure is the definitive proof Java is pass-by-value: if it were pass-by-reference, the swap would work.

> ✅ **Summary rule:** You can always **change the contents** of a mutable object passed in, but you can **never reassign** the caller's variable from inside a method.

## 5. Method Return Types

### 5.1 The basics
- Every method declares a **return type**: a primitive, a reference type, or **`void`** (returns nothing).
- A non-`void` method **must** return a value of (or convertible to) that type on **every** code path.
- `return` immediately exits the method (optionally with a value).

```java
public double average(int[] xs) {
    if (xs.length == 0) return 0.0;   // early return
    int sum = 0;
    for (int x : xs) sum += x;
    return (double) sum / xs.length;
}
```

### 5.2 `void` methods
```java
public void printGreeting(String name) {
    System.out.println("Hello, " + name);
    return;   // optional bare 'return' to exit early; no value
}
```

### 5.3 Returning objects, arrays, collections
A method can return any reference type. Best practice: return **empty collections, not null**, to avoid `NullPointerException`:
```java
public List<String> getNames() {
    return List.of();   // empty list, never null
}
```

### 5.4 Covariant return types (preview)
An overriding method may return a **subtype** of the parent's return type:
```java
class Animal { Animal reproduce() { return new Animal(); } }
class Dog extends Animal {
    @Override Dog reproduce() { return new Dog(); }  // covariant: Dog is-a Animal
}
```
(Full coverage in the Polymorphism note.)

### 5.5 `Optional<T>` as a return type (preview)
For "might not have a value" results, prefer returning `Optional<T>` over `null` (Phase 1.8):
```java
public Optional<User> findById(long id) { ... }
```

### 5.6 Multiple "return values"
Java methods return one value. To return several, use a class/record, an array, or a collection:
```java
record MinMax(int min, int max) {}
public MinMax range(int[] xs) { /* ... */ return new MinMax(lo, hi); }
```

## 6. Recursion
**Recursion** = a method that calls **itself** to solve a problem by breaking it into smaller subproblems.

### 6.1 The two essential parts
Every correct recursion needs:
1. **Base case** — a condition that stops the recursion (no further self-call).
2. **Recursive case** — the method calls itself on a *smaller* input, moving toward the base case.

```java
public long factorial(int n) {
    if (n <= 1) return 1;          // BASE CASE
    return n * factorial(n - 1);   // RECURSIVE CASE (smaller n)
}
```

### 6.2 How it works in memory (ties to Phase 0 stack)
Each recursive call pushes a **new stack frame** (its own parameters/locals/return address). They unwind in LIFO order as calls return:
```
factorial(4)
  -> 4 * factorial(3)
        -> 3 * factorial(2)
              -> 2 * factorial(1)
                    -> returns 1        (base case)
              <- returns 2*1 = 2
        <- returns 3*2 = 6
  <- returns 4*6 = 24
```

### 6.3 StackOverflowError
If the base case is missing/never reached, frames pile up until the **thread's stack is exhausted** → `StackOverflowError` (exactly the guard-page mechanism from the Phase 0 stack note):
```java
int bad(int n) { return bad(n + 1); }   // no base case -> StackOverflowError
```

### 6.4 Recursion vs iteration
| | Recursion | Iteration (loops) |
|---|-----------|-------------------|
| Readability | Elegant for tree/graph/divide-and-conquer | Better for simple repetition |
| Memory | Uses call stack (risk of overflow) | Constant stack usage |
| Speed | Method-call overhead | Usually faster |
| Best for | Trees, graphs, backtracking, recursive math | Counting, summing, linear scans |

Any recursion can be rewritten as iteration (sometimes using an explicit stack).

### 6.5 Tail recursion (awareness)
A **tail-recursive** call is the last operation in a method. Some languages optimize this into a loop (constant stack). **Java does NOT perform tail-call optimization** — deep recursion still risks overflow, so convert to iteration for very deep cases.

### 6.6 Types of recursion
- **Direct:** method calls itself.
- **Indirect/mutual:** A calls B, B calls A.
- **Tree recursion:** multiple self-calls per invocation (e.g., naive Fibonacci → exponential time; fix with memoization, Phase 2 DP).

```java
int fib(int n) {                       // tree recursion (inefficient!)
    if (n < 2) return n;               // base cases
    return fib(n-1) + fib(n-2);        // two recursive calls
}
```

> Recursion is foundational for **Phase 2 (DSA)** — tree/graph traversals, backtracking (N-Queens, Sudoku), and divide-and-conquer (merge/quick sort).

## 7. Common Pitfalls & Best Practices

| Pitfall | Fix / Best practice |
|---------|---------------------|
| Trying to overload by return type only | Not allowed — change the parameter list |
| Calling static method via instance | Call via the class name |
| Thinking objects are pass-by-reference | They're pass-by-value of the *reference* |
| Expecting `swap`/reassignment to affect caller | It can't — only object mutation is visible |
| Varargs not last / multiple varargs | Only one, must be last parameter |
| Recursion without a reachable base case | Always ensure progress toward base case |
| Deep recursion causing `StackOverflowError` | Convert to iteration or use explicit stack |
| Returning `null` collections | Return empty collections (`List.of()`) |

**Best practices:**
- Methods should **do one thing** (Single Responsibility — Phase 14 clean code).
- Keep methods **small** and well-named (verbs).
- Prefer **static** for pure, stateless utilities; **instance** for state-dependent behavior.
- Validate parameters early (**fail fast**), e.g., `Objects.requireNonNull(arg)`.

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Static factory methods** (`List.of`, `Optional.of`, builders) are everywhere in idiomatic Java and Spring.
- **Overloading** powers flexible APIs (e.g., Spring's many `RestTemplate`/`JdbcTemplate` overloads).
- **Pass-by-value understanding** prevents subtle bugs when passing mutable DTOs/entities into service methods — and explains why **defensive copying** and immutability matter (Phase 1.10 concurrency).
- **Return types**: returning `Optional<T>` from repositories (Spring Data) is standard; understanding `void` vs returning results shapes clean service APIs.
- **Recursion** underlies tree/graph processing, JSON traversal, and DSA interview prep (Phase 2).

## 9. Quick Self-Check Questions

1. What's the difference between an instance method and a static method? Why can't static methods use `this`?
2. What exactly makes up a method signature? Is the return type part of it?
3. What is overloading, and how does the compiler choose among overloads?
4. What are the rules for varargs? How is `int...` represented internally?
5. State precisely why "Java is always pass-by-value," and what that means for objects.
6. Why does a `swap(a, b)` method fail to swap the caller's references?
7. Can you mutate an object passed into a method? Can you reassign the caller's variable? Why/why not?
8. What two parts must every recursion have? What happens without a base case?
9. Why doesn't Java optimize tail recursion, and what's the practical consequence?

## 10. Key Terms

- **Method:** named reusable block defining behavior.
- **Instance method:** belongs to an object; has `this`; accesses instance state.
- **Static method:** belongs to the class; no `this`; for stateless utilities.
- **Method signature:** method name + parameter types/number/order (NOT return type).
- **Overloading:** same name, different signatures; resolved at compile time.
- **Overriding:** subclass redefines an inherited method; resolved at runtime.
- **Varargs (`...`):** accept a variable number of args; backed by an array; must be last.
- **Pass-by-value:** a copy of the variable's value (the reference, for objects) is passed.
- **Parameter vs argument:** declared placeholder vs actual value passed.
- **Return type / `void`:** type of the produced value / no value.
- **Covariant return type:** override returning a subtype of the parent's return type.
- **Recursion:** a method calling itself.
- **Base case / recursive case:** the stopping condition / the self-call on a smaller input.
- **StackOverflowError:** stack exhausted by too-deep recursion.
- **Tail recursion:** recursive call as the final operation (not optimized by Java).