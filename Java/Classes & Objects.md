# Classes
A **Java class** is a user-defined blueprint or template used to create objects that share common properties and behaviors. It is the core building block of Object-Oriented Programming (OOP) in Java, acting as a logical entity that defines how its individual instances (objects) look and act.
#### Components of a Java Class
A standard Java class is defined using the `class` keyword and typically contains five core elements:

- **Fields**: Variables that maintain the state of the object.
- **Methods**: Functions that define the logic and actions an object can execute.
- **Constructors**: Special blocks used to allocate memory and initialize new objects.
- **Blocks**: Code segments (static or instance) executed during class loading or object initialization.
- **Nested Classes**: Classes written entirely inside another outer class.

#### Core Syntax and Code Example
The following example defines a blueprint for a `Car` and instantiates it inside a execution class.

```java title:class.java
// The Blueprint Class
class Car {
    // 1. Fields (State)
    String brand;
    String color;

    // 2. Constructor (Initialization)
    Car(String brand, String color) {
        this.brand = brand;
        this.color = color;
    }

    // 3. Method (Behavior)
    void drive() {
        System.out.println("The " + color + " " + brand + " is driving.");
    }
}

// Execution Class
public class Main {
    public static void main(String[] args) {
        // Instantiating objects using the 'new' keyword
        Car car1 = new Car("Tesla", "Red");
        Car car2 = new Car("BMW", "Blue");

        // Invoking methods
        car1.drive(); 
        car2.drive();
    }
}
```

#### Strict Naming & File Rules
When compiling code, the [Java Development Kit (JDK)](https://docs.oracle.com/javase/tutorial/java/javaOO/classes.html) enforces strict organization boundaries:

1. **File Matching**: If a class is declared `public`, the surrounding `.java` filename **must** exactly match that class name.
2. **One Public Limitation**: A single Java source file can contain multiple classes, but **only one** can be marked `public`.
3. **Naming Convention**: Class names should strictly use `PascalCase` (e.g., `ShoppingCart`, `UserAccount`), starting with an uppercase letter.

#### Types of Classes in Java
Java has specific types of classes designed for different architectural needs. 

##### A. Concrete Class
This is your standard, everyday class. It provides full implementations for all its methods and can be instantiated (you can create objects from it using the [[Keywords#4. Object, Class, and Interface Management (9 Keywords)|new]] keyword).

- Example: The `Car` class above is a concrete class.

##### B. Abstract Class (`abstract`)
An abstract class is a restricted class that **cannot be used to create objects directly**. It is meant to be subclassed (inherited from). It can contain both regular methods with code inside them, and "abstract methods" (methods without a body that the child class _must_ complete).

- Use case: Creating a generic `Animal` class where you want `Dog` and `Cat` to inherit from it, but you never want a generic `Animal` object to exist on its own.

##### C. Final Class (`final`)

A final class is the exact opposite of an abstract class: it **cannot be inherited**. Once a class is marked as final, no other class can extend it.

- Use case: Security and immutability. Java's `String` class is a famous example of a final class; you cannot create a custom subclass of `String`.

##### D. Inner / Nested Class
A class declared inside another class.

- Use case: Used to logically group classes that are only used in one place, increasing encapsulation. For example, a `Bank` class might have a private `Vault` class nested inside it.

##### E. Anonymous Class
A temporary class that does not have a name. It is declared and instantiated in a single expression.

- Use case: Often used for "on-the-fly" implementations of interfaces, like creating a quick click-listener for a button in a user interface without writing a whole new standalone class file.

##### F. Enum Class (`enum`)
A special type of class that represents a fixed set of constants (unchangeable variables).

- Use case: Representing distinct, finite things like `DaysOfTheWeek` (MONDAY, TUESDAY...) or `OrderStates` (PENDING, SHIPPED, DELIVERED).

##### G. Record Class (`record`) (Java 14)
A special, concise type of class designed purely to hold immutable data. When you create a record, Java automatically generates the constructor, getters, and standard methods (like `equals()`, `hashCode()`, and `toString()`) behind the scenes.

- Use case: Creating simple data-carrier objects (like returning coordinates `x` and `y` or a `User` profile with a name and email) without writing dozens of lines of boilerplate code.

# Object?
In Java, an **object is a self-contained instance of a class** that combines both data and behavior to represent a real-world entity or abstract concept. While a class acts as a blueprint or template, an object is the physical, runtime entity created from that blueprint that occupies actual space in your computer's memory.

#### The Three Characteristics of an Object
Every object in Java possesses three fundamental features:

- **State**: This represents the data or attributes stored inside the object's variables (known as instance fields).
- **Behavior**: This represents the actions or operations the object can perform, which are defined via its methods.
- **Identity**: This is a unique address or identifier used internally by the Java Virtual Machine (JVM) to distinguish one object from another, even if their data states are identical.

#### Class vs. Object
To make this simpler, think of a **Class** as the architectural blueprint for a house. The blueprint itself isn't a house; it just describes what a house will look like.

An **Object** is the actual physical house built using that blueprint. From one single blueprint (Class), you can build dozens of unique houses (Objects), each with its own paint color or occupant data (State).

| Concept Component      | Class Blueprint                | Object Instance                     |
| ---------------------- | ------------------------------ | ----------------------------------- |
| **Real-world Analogy** | General concept of a "Dog"     | A specific dog named "Tuffy"        |
| **State (Data)**       | Breed, Age, Color variables    | Labrador, 3 years old, Golden       |
| **Behavior (Actions)** | Bark(), Sleep(), Eat() methods | Tuffy barks when a method is called |

#### How Objects Look in Java Code
To create an object in Java, you typically use the `new` keyword. The `new` keyword instructs the JVM to allocate heap memory for the new instance.

```java title:object.java
// 1. Defining the Blueprint (Class)
class Car {
    String brand; // State
    int year;     // State

    void drive() { // Behavior
        System.out.println("The car is driving.");
    }
}

// 2. Creating and using the Object
public class Main {
    public static void main(String[] args) {
        // Instantiating a Car object using the 'new' keyword
        Car myCar = new Car(); 
        
        // Assigning data to the object's state
        myCar.brand = "Toyota";
        myCar.year = 2026;
        
        // Invoking the object's behavior using the dot (.) operator
        myCar.drive(); 
    }
}
```


