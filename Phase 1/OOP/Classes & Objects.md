Everything in this note ties back to the memory model ([[Stack & Heap Memory - Hardware Level|Stack & Heap]]): **references live on the stack, objects live on the heap.**

> Class : Object
> Blueprint : House
> Recipe : Cake
>  `class Car {}` : `new Car()`

## 1. Class Definition
A **class** is a **blueprint/template**, it defines *what an object will look like* (state via fields) and *what it can do* (behavior via methods). It occupies no object memory by itself.

### 1.1 Types of Classes in Java
Java has specific types of classes designed for different architectural needs. 

#### A. Concrete Class
This is your standard, everyday class. It provides full implementations for all its methods and can be instantiated (you can create objects from it using the [[Keywords#4. Object, Class, and Interface Management (9 Keywords)|new]] keyword).

- Example: The `Car` class above is a concrete class.

#### B. Abstract Class (`abstract`)
An abstract class is a restricted class that **cannot be used to create objects directly**. It is meant to be subclassed (inherited from). It can contain both regular methods with code inside them, and "abstract methods" (methods without a body that the child class _must_ complete).

- Use case: Creating a generic `Animal` class where you want `Dog` and `Cat` to inherit from it, but you never want a generic `Animal` object to exist on its own.

#### C. Final Class (`final`)

A final class is the exact opposite of an abstract class: it **cannot be inherited**. Once a class is marked as final, no other class can extend it.

- Use case: Security and immutability. Java's `String` class is a famous example of a final class; you cannot create a custom subclass of `String`.

#### D. Inner / Nested Class
A class declared inside another class.

- Use case: Used to logically group classes that are only used in one place, increasing encapsulation. For example, a `Bank` class might have a private `Vault` class nested inside it.

#### E. Anonymous Class
A temporary class that does not have a name. It is declared and instantiated in a single expression.

- Use case: Often used for "on-the-fly" implementations of interfaces, like creating a quick click-listener for a button in a user interface without writing a whole new standalone class file.

#### F. Enum Class (`enum`)
A special type of class that represents a fixed set of constants (unchangeable variables).

- Use case: Representing distinct, finite things like `DaysOfTheWeek` (MONDAY, TUESDAY...) or `OrderStates` (PENDING, SHIPPED, DELIVERED).

#### G. Record Class (`record`) (Java 14)
A special, concise type of class designed purely to hold immutable data. When you create a record, Java automatically generates the constructor, getters, and standard methods (like `equals()`, `hashCode()`, and `toString()`) behind the scenes.

- Use case: Creating simple data-carrier objects (like returning coordinates `x` and `y` or a `User` profile with a name and email) without writing dozens of lines of boilerplate code.

### 1.2 Anatomy of a class
```java title:class_anatomy.java
public class Car {                 // class declaration
    // 1. Fields (instance variables = state)
    private String model;
    private int speed;

    // 2. Constructor (how to build one)
    public Car(String model) {
        this.model = model;
        this.speed = 0;
    }

    // 3. Methods (behavior)
    public void accelerate(int delta) {
        this.speed += delta;
    }

    public int getSpeed() {
        return speed;
    }
}
```

#### A class can contain:
- **Fields**: Variables that maintain the state of the object.
- **Methods**: Functions that define the logic and actions an object can execute.
- **Constructors**: Special blocks used to allocate memory and initialize new objects.
- **Blocks**: Code segments (static or instance) executed during class loading or object initialization.
- **Nested Classes**: Classes written entirely inside another outer class.

### 1.3 Rules & conventions
- One **public** top-level class per `.java` file, and the file **must** be named after it (`Car.java`).
- Class names use **PascalCase** (`BankAccount`), fields/methods use **camelCase**.
- A class without an explicit constructor gets a [[#3.1 Default (no-arg) constructor|default constructor]].

## 2. Objects (Instantiation)
An **object** (a.k.a. **instance**) is a **concrete thing built from that blueprint**, living on the **heap** at runtime.

```java title:object.java
Car myCar = new Car("Tesla");   // create an instance
```

Breaking this single line into its three parts is essential:
```
Car        myCar      =      new Car("Tesla");
 ^           ^                  ^
 type       reference          object creation expression
 (compile)  variable           (runtime, on the heap)
            (on the stack)
```

1. **`Car`**: the declared **type** of the reference (compile-time contract).
2. **`myCar`**: a **reference variable** stored on the **stack** (inside the current method's frame). It holds the *address* of the object, **not** the object itself.
3. **`new Car("Tesla")`**: allocates a `Car` **object on the heap** and runs its constructor.

A reference is like a *remote control*; the object is the *TV*. Many remotes (references) can point to one TV (object):
```java title:object2.java
Car a = new Car("Tesla");
Car b = a;            // b points to the SAME object (no new object created)
b.accelerate(50);
System.out.println(a.getSpeed()); // 50 -> same object!
```

This is **why `==` compares references (identity)** and `.equals()` compares content, a recurring Java gotcha (covered fully in the Object methods note).

## 3. Constructors
A **constructor** is a special method-like block that runs **once**, at object creation, to initialize the object. Rules:
- Same **name as the class**.
- **No return type** (not even `void`).
- Called automatically by `new`.
- Can be overloaded (multiple constructors with different parameters).

### 3.1 Default (no-arg) constructor
If you write **no constructors at all**, the compiler inserts an implicit **default constructor** with no arguments:
```java title:dog.java
public class Dog {
    String name;          // no constructor written
}
// compiler generates:  public Dog() { super(); }
Dog d = new Dog();        // works
```
⚠️ **Note:** The moment you declare **any** constructor, the compiler **stops** generating the default one. Then `new Dog()` would fail to compile unless you add a no-arg constructor explicitly.

### 3.2 Parameterized constructor
Takes arguments to initialize fields with caller-supplied values:
```java title:dog.java
public class Dog {
    String name;
    int age;

    public Dog(String name, int age) {   // parameterized
        this.name = name;
        this.age = age;
    }
}
Dog d = new Dog("Rex", 3);
```

### 3.3 Copy constructor
Java has **no built-in** copy constructor (unlike C++), but it's a common idiom: a constructor that takes another instance of the same class and copies its state.
```java title:pont.java
public class Point {
    private int x, y;

    public Point(int x, int y) {     // parameterized
        this.x = x;
        this.y = y;
    }

    public Point(Point other) {      // COPY constructor
        this.x = other.x;
        this.y = other.y;
    }
}
Point p1 = new Point(3, 4);
Point p2 = new Point(p1);   // independent copy
```

> **Shallow vs deep copy caution:** if a field is itself a mutable object/reference, a naive copy constructor copies the *reference*, so both objects share that inner object (**shallow copy**). For a fully independent (**deep**) copy, you must copy the nested objects too:
```java title:person.java
public Person(Person other) {
    this.name = other.name;                      // String is immutable -> fine
    this.addresses = new ArrayList<>(other.addresses); // copy the list (defensive)
}
```
(Deep vs shallow is revisited in the `clone()` and immutability notes.)

### 3.4 Constructor overloading
Multiple constructors differing by parameter list, the compiler picks the matching one:
```java title:rectangle.java
public class Rectangle {
    int width, height;
    public Rectangle() { this(1, 1); }            // default size
    public Rectangle(int side) { this(side, side); } // square
    public Rectangle(int width, int height) {     // full control
        this.width = width;
        this.height = height;
    }
}
```

## 4. Constructor Chaining (`this()` and `super()`)
**Constructor chaining** = one constructor calling another, so initialization logic isn't duplicated. Two directions:

### 4.1 `this(...)`
 chain to another constructor in the **same class**.
```java title:user.java
public class User {
    String name;
    String role;

    public User(String name) {
        this(name, "USER");      // delegates to the 2-arg constructor
    }

    public User(String name, String role) {
        this.name = name;
        this.role = role;
    }
}
```
- `this(...)` **must be the first statement** in the constructor.
- Avoids duplicated initialization across overloaded constructors.

### 4.2 `super(...)`
chain to the **parent class** constructor.
```java title:animal.java
class Animal {
    String species;
    Animal(String species) { this.species = species; }
}

class Dog extends Animal {
    String name;
    Dog(String name) {
        super("Canine");   // call parent constructor FIRST
        this.name = name;
    }
}
```
- `super(...)` **must also be the first statement** (so you can't use both `this()` and `super()` in the same constructor).
- If you don't write `super(...)`, the compiler inserts an implicit `super()` (no-arg) call. → If the parent has **no** no-arg constructor, you **must** call `super(args)` explicitly, or it won't compile.

### 4.3 The full chain always reaches `Object`
Every constructor call chains upward until it reaches `java.lang.Object`. Construction then runs **top-down** (parent initialized before child):
```
new Dog("Rex")
   -> Dog(String)
        -> super("Canine")  ==> Animal(String)
                                  -> super()  ==> Object()
   <- Object body runs, then Animal body, then Dog body
```
> **Order guarantee:** parent fields/constructor finish before the child's constructor body runs. This is why a parent constructor must never call an overridable method expecting the child to be fully initialized (a classic bug).

## 5. The `this` Keyword (ALL Uses)
`this` is an implicit reference to **the current object** (the instance on which the method/constructor is running). Five distinct uses:

| # | Use | Example |
|---|-----|---------|
| 1 | **Disambiguate field vs parameter** (shadowing) | `this.name = name;` |
| 2 | **Call another constructor** (chaining) | `this(name, "USER");` |
| 3 | **Pass the current object** as an argument | `register(this);` |
| 4 | **Return the current object** (fluent/builder APIs) | `return this;` |
| 5 | **Refer to the current instance** explicitly (e.g., from inner class) | `OuterClass.this.field` |

```java title:account.java
public class Account {
    private double balance;

    public Account deposit(double balance) {
        this.balance += balance;   // (1) field vs param disambiguation
        return this;               // (4) enable method chaining
    }

    public void audit(Auditor a) {
        a.record(this);            // (3) pass current object
    }
}
// (4) fluent usage:
account.deposit(100).deposit(50).deposit(25);
```

> `this` cannot be used in a **static** context (static methods/blocks) — there's no current instance.

## 6. The `new` Keyword
The **`new` keyword** in Java is a core operator used to **instantiate classes by dynamically allocating memory for new objects on the heap** at runtime. It triggers the object creation lifecycle, initializes instance variables to default values, invokes the targeted constructor, and ultimately returns a memory reference to that fresh object.

#### What is does on the memory?
This is the conceptual heart of the topic. When you execute `new Car("Tesla")`, the JVM performs these steps:
```java title:car.java
Car myCar = new Car("Tesla");
```

**STEP 1:** Class loading check
        Is class Car loaded/linked/initialized? If not, the Class Loader
        loads it (Phase 1.11). Static fields/blocks run ONCE here.

**STEP 2:** Heap allocation
        JVM allocates memory on the HEAP big enough for a Car object
        (object header + all instance fields). Often via a fast bump-pointer
        in a TLAB (Thread-Local Allocation Buffer).

**STEP 3:** Default initialization (zeroing)
        All instance fields are set to their DEFAULT values:
        numbers -> 0/0.0, boolean -> false, references -> null.

**STEP 4:** Constructor execution (top-down)
        super(...) chain runs first, then instance initializer blocks &
        field initializers, then THIS constructor's body. Fields get their
        real values here.

**STEP 5:** Return the reference
        new returns the heap ADDRESS of the new object. It is stored in
        the reference variable 'myCar' on the STACK.

### 6.1 Object header (what else lives in an object)
Beyond your fields, every object carries a small **header**:
- **Mark word**: identity hashcode, GC age, lock state (used by `synchronized`).
- **Klass pointer**: points to the class metadata (so the object knows its type, enables `getClass()`, virtual dispatch).
- (Arrays also store a length.)

### 6.2 Order of initialization (precise sequence within one class)
For a single object construction, members initialize in this order:
1. **Static** fields & **static initializer blocks**; *once*, when the class is first loaded (not per object).
2. Then, per `new`: **instance field initializers** and **instance initializer blocks**, in source order.
3. Then the **constructor body**.

```java
public class Demo {
    static int s = init("static field");          // 1 (once)
    static { System.out.println("static block"); }// 1 (once)

    int i = init("instance field");               // 2 (each new)
    { System.out.println("instance block"); }     // 2 (each new)

    Demo() { System.out.println("constructor"); } // 3 (each new)

    static int init(String s){ System.out.println(s); return 0; }
}
```
With inheritance, this whole sequence interleaves via the `super()` chain (parent statics → parent instance init/ctor → child instance init/ctor).

## 7. Object Lifecycle
**An object's life from birth to reclamation:**

1. CREATION      new -> allocated on heap, constructed, reference returned
2. IN USE        reachable via references; methods called; state mutated
3. UNREACHABLE   no live reference points to it anymore
4. GC ELIGIBLE   the Garbage Collector may reclaim it
5. FINALIZATION  (legacy/deprecated finalize(); avoid) 
6. RECLAIMED     memory freed, returned to the heap

### 7.1 Becoming unreachable (eligible for GC)
```java title:new_car.java
Car c = new Car("A");
c = new Car("B");   // the "A" object is now unreachable -> GC eligible
c = null;           // the "B" object is now unreachable too
```
Java has **no manual `delete`/`free`**, the **[[JVM Architecture#3. Garbage Collection|garbage collector]]** reclaims unreachable objects automatically.

### 7.2 Reachability
An object is "alive" if it's reachable from a **GC root** (e.g., local variables on the stack, static fields, active threads). When the last reference chain to it is gone, it becomes collectible.

### 7.3 `finalize()` - don't use it
Historically objects had a `finalize()` method called before reclamation. It's **deprecated** (unpredictable, slow, can resurrect objects). For cleanup use **`try-with-resources`** + `AutoCloseable` instead (covered in the exception handling / I/O notes).

## 8. Common Pitfalls & Fix

| Pitfall | Fix / Best practice |
|---------|---------------------|
| Forgetting that a custom constructor removes the default one | Add an explicit no-arg constructor if you need one |
| Confusing reference copy with object copy (`b = a`) | Use a copy constructor for an independent object |
| Shallow copy sharing mutable inner objects | Deep-copy nested mutable fields (defensive copying) |
| Calling overridable methods from a constructor | Avoid — child isn't fully initialized yet |
| Using `this` in a static context | Not allowed — no instance exists |
| Relying on `finalize()` for cleanup | Use `try-with-resources` / `AutoCloseable` |
| Memory "leak" via lingering references (e.g., in static collections) | Null out / remove references when done |
| Assuming `new` guarantees an object stays in RAM | It's virtual memory; the heap is GC-managed (Phase 0 link) |

**Best practices:**
- Prefer **constructor injection** of dependencies (sets up clean, testable objects - foreshadows Spring DI in Phase 5).
- Keep constructors **simple** — initialize state, don't do heavy work or I/O.
- Make fields `private` and `final` where possible (immutability — later note).
- Use **constructor chaining** (`this()`) to avoid duplicate initialization logic.

## 9. Connection to Backend / Spring (Why This Matters Later)

- **Dependency Injection** (Spring) is literally *constructor injection* — the framework calls your constructor passing dependencies. Understanding constructors deeply makes Spring intuitive.
- **JPA entities** (Phase 5.4) require a no-arg constructor (Hibernate instantiates them reflectively) — this is exactly the "custom constructor removes the default" rule biting you.
- **Immutability + copy constructors** underpin thread-safe, predictable backend code (Phase 1.10 concurrency).
- **Object lifecycle / GC** awareness is essential for diagnosing memory leaks in long-running servers (Phase 1.11, Phase 13).
- **`new` and the heap** connect straight back to Phase 0: objects live on the shared heap → that's why they need synchronization across threads.

## 10. Quick Self-Check Questions

1. What's the difference between a class and an object? Where does each "live" in memory?
2. In `Car c = new Car();`, what's on the stack and what's on the heap?
3. When does the compiler provide a default constructor — and when does it *stop*?
4. What is a copy constructor, and what's the shallow-vs-deep-copy concern?
5. What are the rules for `this(...)` and `super(...)` placement? Can you use both together?
6. List all five uses of the `this` keyword.
7. Walk through the steps `new` performs in memory.
8. In what order do static blocks, instance blocks, field initializers, and the constructor run?
9. When does an object become eligible for garbage collection?
10. Why should you avoid `finalize()` and calling overridable methods in a constructor?

## 11. Key Terms

- **Class:** blueprint defining state (fields) and behavior (methods).
- **Object / Instance:** a concrete entity created from a class, living on the heap.
- **Reference variable:** a stack-stored value holding an object's address.
- **Instantiation:** creating an object with `new`.
- **Constructor:** special initializer run once at creation; class-named, no return type.
- **Default constructor:** compiler-generated no-arg constructor (only if none declared).
- **Parameterized constructor:** constructor taking arguments.
- **Copy constructor:** constructor that builds a copy from another instance.
- **Constructor chaining:** one constructor invoking another via `this()`/`super()`.
- **`this`:** reference to the current object.
- **`super`:** reference to the parent-class portion / its constructor.
- **`new`:** operator that allocates and constructs an object on the heap.
- **Object header:** per-object metadata (mark word + klass pointer).
- **TLAB:** Thread-Local Allocation Buffer — fast per-thread heap allocation region.
- **Initializer block:** `{ ... }` (instance) or `static { ... }` setup code.
- **Object lifecycle:** creation → use → unreachable → GC-eligible → reclaimed.
- **Reachability / GC root:** basis the GC uses to decide if an object is alive.
- **Garbage Collector (GC):** subsystem that reclaims unreachable objects.