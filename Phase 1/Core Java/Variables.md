A variable is a named container that **stores data** in memory.

```java title:syntax.java
int age = 25
```

Every variable in Java has:
- A **type** → what kind of data it holds.
- A **name** → how you refer to it.
- A **value** → the actual data stored.

Java has three kinds, distinguished by **where they're declared** and **how long they live**:

| Kind                  | Declared                   | Belongs to            | **Default Value?**                | Lifetime                       | Memory                  |
| --------------------- | -------------------------- | --------------------- | --------------------------------- | ------------------------------ | ----------------------- |
| **Local variable**    | Inside a method/block      | The method invocation | No (Must be initialized manually) | Until block ends               | Stack                   |
| **Instance variable** | In class body (non-static) | Each object           | Yes (`0`, `false`, `null`)        | As long as the object          | Heap (inside object)    |
| **Static variable**   | In class body with static  | The class itself      | Yes (`0`, `false`, `null`)        | As long as the class is loaded | Method area / metaspace |

## 1. Local Variables
Declared **inside a method, constructor, or block**. Used for temporary computation.

```java title:local_variable.java
void calculate() {
    int sum = 0;              // local variable
    for (int i = 0; i < 10; i++) {   // i is local to the for block
        sum += i;
    }
    // i is NOT accessible here
}
```

### 1.1 Key properties
- **No default value** - must be **explicitly initialized** before use (compiler enforces this):
  ```java
  int x;
  // System.out.println(x);  // COMPILE ERROR: might not be initialized
  ```
- **Scoped** to the block where declared.
- Stored on the **stack** → naturally **thread-safe**.
- Not accessible outside their block.

## 2. Instance Variables (Fields)
Declared in the class body **without `static`**. Each **object** gets its **own copy**.

```java title:instance_variable.java
public class Counter {
    private int count;        // instance variable — one per object
    private String name = "default";  // can have an initializer
}

Counter a = new Counter();    // a.count = 0 (default)
Counter b = new Counter();    // b.count is independent of a.count
```

### 2.1 Key properties
- **Auto-initialized** to defaults (`0`, `false`, `null`, etc.) - unlike locals.
- Live on the **heap**, inside the object → exist as long as the object is reachable.
- Each object has an independent copy.
- Initialized via: default → field initializers / instance blocks → constructor (recall Classes & Objects note's init order).

## 3. Static (Class) Variables
Declared with `static`. There is **exactly one copy**, shared by **all** instances and the class itself.

```java title:static_variable.java
public class Counter {
    private static int totalCount = 0;   // ONE shared copy for all objects
    private int id;

    public Counter() {
        totalCount++;          // increments the shared variable
        this.id = totalCount;
    }
    public static int getTotalCount() { return totalCount; }
}

new Counter(); new Counter(); new Counter();
System.out.println(Counter.getTotalCount());  // 3  -> accessed via class name
```

### 3.1 Key properties
- **One copy per class**, not per object - shared state.
- Accessed via the **class name**: `Counter.getTotalCount()` (not via an instance, by convention).
- Initialized when the class is **loaded** (once), via static initializers (recall init order).
- Live in the **method area / metaspace**.
- ⚠️ **Shared mutable static state is a concurrency hazard** - multiple threads modifying a static variable need synchronization.

### 3.2 `static final` constants
In real Java applications, constants are usually declared as `static final`:
```java title:final_static.java
public class AppConfig {
public static final int MAX_CONNECTIONS = 100;
public static final String APP_VERSION = "1.0.0";
public static final double TAX_RATE = 0.18;
}
```

- ==`static`== → belongs to the class, not to any object.
- ==`final`== → cannot be changed after assignment.

## 4. Variable Scope and Lifetime
**Scope** = where a variable is *accessible*. **Lifetime** = how long it *exists in memory*.

```java title:scope_lifetime.java
public class ScopeDemo {
    static int classVar = 1;     // scope: whole class; lifetime: class loaded -> unloaded
    int instanceVar = 2;         // scope: whole class (non-static); lifetime: object's life

    void method(int param) {     // param scope: this method
        int local = 3;           // scope: from here to method end
        if (local > 0) {
            int blockVar = 4;    // scope: only inside this if block
        }
        // blockVar NOT accessible here
    }
}
```

### 4.1 Scope rules
- A variable is accessible only **within the block `{}` where it's declared**, and nested blocks.
- **Variable shadowing:** an inner-scope variable can hide an outer one of the same name (avoid; use `this.field` to disambiguate):
  ```java
  int x = 1;
  void m() {
      int x = 2;          // shadows the field x
      System.out.println(x);        // 2 (local)
      System.out.println(this.x);   // 1 (field)
  }
  ```
- You **cannot** redeclare a variable already in scope.

| Level | Scope | Lifetime |
|-------|-------|----------|
| Static | Class-wide | Class loaded → unloaded |
| Instance | Class-wide (non-static context) | Object created → GC'd |
| Method parameter | The method | Method call |
| Local | Declaring block onward | Block execution |
| Block (loop/if) | That block only | Block execution |

## 5. The `final` Keyword for Variables
`final` makes a variable's value **unchangeable after assignment** - it can be assigned **exactly once**.

```java title:final_keyword.java
final int MAX = 100;
// MAX = 200;            // COMPILE ERROR: cannot reassign a final variable

final List<String> list = new ArrayList<>();
list.add("hi");          // OK! the LIST contents can change...
// list = new ArrayList<>();  // ERROR: ...but the REFERENCE can't be reassigned
```

### 5.1 The crucial nuance: final reference != immutable object
`final` makes the **variable** (the reference/value) constant, **not** the object it points to. A `final` reference can still point to a mutable object whose contents change. For true immutability, the object itself must be immutable.

### 5.2 Where `final` applies
| Use                    | Effect                                                                       |
| ---------------------- | ---------------------------------------------------------------------------- |
| `final` local variable | Assigned once within the method                                              |
| `final` instance field | Must be set by the constructor; never changes after (great for immutability) |
| `final static` field   | A constant (assigned at declaration or static block)                         |
| `final` parameter      | Can't be reassigned inside the method                                        |

### 5.3 Benefits
- **Safety:** prevents accidental reassignment.
- **Immutability:** `private final` fields are the backbone of immutable classes.
- **Thread safety:** `final` fields have special **safe-publication** guarantees in the Java Memory Model (Phase 1.10).
- **Clarity:** signals intent ("this won't change").
- Required for **variable capture** in lambdas/inner classes (must be `final` or *effectively final*, recall Inner Classes note).

## 6. The `var` Keyword (Java 10+ Type Inference)
Introduced in Java 10, the `var` keyword brought a highly requested feature to the language: **Local Variable Type Inference**.
Instead of forcing you to explicitly declare the data type on the left side of the equals sign, `var` tells the Java compiler to look at the value on the right side and figure out the data type for itself.

```java title:var.java
var name = "Sam";                 // inferred as String
var count = 42;                   // inferred as int
var list = new ArrayList<String>(); // inferred as ArrayList<String>
var map = new HashMap<String, List<Integer>>();  // less boilerplate

// name = 5;   // COMPILE ERROR: name is String, type is fixed at inference
```

### 6.1 Rules and restrictions
Because the compiler needs to be 100% certain of the data type, `var` is heavily restricted. You can only use it when the context is perfectly clear.
#### It can ONLY be used for Local Variables
You can only use `var` for variables declared _inside_ a method, in a `for` loop, or in a `try-with-resources` block.
```java title:Player.java
public class Player {
    var maxHealth = 100; // ERROR: Cannot be used for class/instance variables
    
    public void heal(var amount) { // ERROR: Cannot be used for method parameters
        var currentHealth = 50; // VALID: Local variable inside a method
    }
}
```

#### It MUST be initialized immediately
The compiler cannot guess the type if you don't give it a value right away.
```java title:var.java
var playerName; // ERROR: Cannot infer type without an assignment
playerName = "Mario"; 

var currentName = "Luigi"; // VALID
```

#### It CANNOT be assigned to `null`
Because `null` doesn't have a specific type, the compiler doesn't know what kind of reference variable to create.

```java title:null.java
var activePlayer = null; // ERROR: Variable initializer is 'null'
```

#### It requires explicit types for Arrays
You cannot use the array initialization shortcut with `var`.
```java title:var.java
var scores = {10, 20, 30}; // ERROR: Array initializer needs an explicit target-type
var validScores = new int[]{10, 20, 30}; // VALID
```

### 6.2 When to use `var`
The golden rule for `var` is **Readability**. `var` improves readability when the type is evident, and hurts it when the type is hidden behind a method call. Use judgment.

| Use `var` when...                                                  | Avoid `var` when...                                         |
| ------------------------------------------------------------------ | ----------------------------------------------------------- |
| The type is obvious from the right side (`var user = new User();`) | It hurts readability (`var x = getStuff();` - unclear type) |
| It reduces verbose generics (`var map = new HashMap<...>();`)      | The type matters for the reader                             |
| In `for` loops (`for (var e : entries)`)                           | You'd want an interface type (e.g., `List` not `ArrayList`) |

#### Use `var`
Use it when the right side of the assignment makes it blindingly obvious what the data type is. It is fantastic for shortening long object creations or cleaning up `for` loops:
```java title:use_var.java
// Excellent use case
for (var entry : map.entrySet()) { ... }

var stream = new FileInputStream("data.txt");
```

#### Avoid `var`
Avoid it when the method you are calling obscures the return type. If another developer has to guess what data type is coming back, go back to using explicit declarations:
```java title:avoid_var.java
// Bad use case - What is data? A String? A byte array? A custom Object?
var data = processNetworkRequest();
```

## 7. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Using a local variable before initializing it | Always initialize locals before use |
| Thinking `final` makes the object immutable | It only fixes the reference; make the object immutable separately |
| Shared mutable static state across threads | Synchronize or avoid mutable statics |
| Variable shadowing causing confusion | Use distinct names; `this.field` to disambiguate |
| Using `var` where the type is unclear | Write the explicit type for readability |
| Trying `var` on a field or parameter | `var` is local-only |
| Accessing static via instance | Access via the class name |

## 8. Connection to Backend / Spring (Why This Matters Later)

- **`private final` fields + constructor injection** is the idiomatic Spring bean pattern — immutable, testable dependencies (Phase 5).
- **Static mutable state** is an anti-pattern in services (breaks scaling/thread-safety) — prefer Spring-managed singleton beans with controlled state (Phase 1.10, 13).
- **`static final` constants** for config keys, defaults, error codes.
- **`var`** keeps modern service code concise, especially with generics and streams.
- **Variable scope/lifetime** understanding underpins memory-leak diagnosis (long-lived references holding objects — Phase 1.11, 13).

## 9. Quick Self-Check Questions

1. What are the three kinds of variables, and where does each live in memory?
2. Which variables get default values and which don't?
3. How many copies of a static variable exist? How is it accessed?
4. What's the difference between scope and lifetime?
5. What does `final` guarantee — and what does it NOT guarantee?
6. Why is `final` important for immutability and lambdas?
7. What does `var` do? Name two restrictions on it.
8. Is `var` dynamic typing? Explain.

## 10. Key Terms

- **Local variable:** declared in a method/block; on the stack; no default value.
- **Instance variable (field):** per-object state; on the heap; auto-defaulted.
- **Static (class) variable:** one shared copy per class.
- **Scope:** where a variable is accessible.
- **Lifetime:** how long a variable exists in memory.
- **Shadowing:** an inner variable hiding an outer one of the same name.
- **`final`:** assign-once; fixes the variable, not the referenced object.
- **`static final`:** a constant.
- **`var`:** compiler type inference for local variables (Java 10+).
- **Effectively final:** assigned once, never reassigned (capturable by lambdas).
- **Safe publication:** JMM guarantee for `final` fields across threads.