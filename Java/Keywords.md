Keywords in Java are **predefined, reserved words that carry a specific meaning to the Java compiler**. Because they form the foundational syntax and structure of the language, you **cannot use them as identifiers** (such as names for variables, methods, classes, or objects).

## Types of Keywords in Java
There are over **50 keywords** categorized by their functional purpose in the language syntax.

### 1. Primitive Data Types (8 Keywords)
These keywords define the basic type of data a variable can hold.

- **`boolean`**: Represents a true or false value.
- **`byte`**: Represents an 8-bit integer.
- **`char`**: Represents a single 16-bit Unicode character.
- **`short`**: Represents a 16-bit integer.
- **`int`**: Represents a 32-bit integer.
- **`long`**: Represents a 64-bit integer.
- **`float`**: Represents a 32-bit single-precision floating-point number.
- **`double`**: Represents a 64-bit double-precision floating-point number.

### 2. Return Type (1 Keyword)

- **`void`**: Used in method declarations to specify that the method does not return any value.

### 3. Access Modifiers (3 Keywords)
These define the visibility and accessibility of classes, methods, and variables.

- **`public`**: The entity is accessible from any other class.
- **`protected`**: The entity is accessible within its own package and by subclasses in other packages.
- **`private`**: The entity is accessible only within the class it is declared in. **Note:** If no modifier is used, it defaults to "package-private", which does not have a specific keyword.

### 4. Object, Class, and Interface Management (9 Keywords)
These are used to structure your object-oriented code.

- **`class`**: Declares a new class.
- **`interface`**: Declares a new interface.
- **`enum`**: Declares an enumeration (a special class representing a fixed set of constants).
- **`extends`**: Indicates that a class is inheriting from a superclass or an interface is inheriting from a superinterface.
- **`implements`**: Indicates that a class is providing the implementation for an interface.
- **`new`**: Used to create a new instance of an object.
- **`this`**: A reference to the current object/instance.
- **`super`**: A reference to the parent (superclass) object.
- **`instanceof`**: Tests whether an object is an instance of a specific class or interface.

### 5. Control Flow & Decision Making (5 Keywords)
These dictate the logical path the program takes.

- **`if`**: Executes a block of code if a condition is true.
- **`else`**: Executes a block of code if the `if` condition is false.
- **`switch`**: Selects one of many code blocks to be executed based on a variable's value.
- **`case`**: Defines a specific block of code inside a `switch` statement.
- **`default`**: Defines the default block of code inside a `switch` statement if no cases match.

### 6. Looping & Loop Control (6 Keywords)
These handle repetitive execution of code.

- **`for`**: A loop that repeats a specific number of times.
- **`while`**: A loop that repeats as long as a condition remains true.
- **`do`**: Used alongside `while` to execute a block of code at least once before checking the condition.
- **`break`**: Immediately exits a loop or switch statement.
- **`continue`**: Skips the current iteration of a loop and moves to the next one.
- **`return`**: Exits a method and (optionally) hands back a value to the caller.

### 7. Exception Handling (5 Keywords)
These safely manage errors that occur during runtime.

- **`try`**: Defines a block of code to be tested for errors.
- **`catch`**: Defines a block of code to handle the error caught in the `try` block.
- **`finally`**: Defines a block of code that will execute regardless of whether an exception was caught or not.
- **`throw`**: Used to explicitly throw a custom or built-in exception.
- **`throws`**: Placed in a method signature to declare that the method might throw one or more exceptions.

### 8. Non-Access Modifiers (7 Keywords)
These change the default behavior of classes, variables, or methods.

- **`static`**: Indicates that a variable or method belongs to the class itself, rather than an instance of the class.
- **`final`**: Indicates that a variable's value cannot be changed, a method cannot be overridden, or a class cannot be inherited.
- **`abstract`**: Used to declare a class that cannot be instantiated, or a method that lacks an implementation (to be filled by subclasses).
- **`synchronized`**: Used in multi-threading to lock a method or block of code so only one thread can access it at a time.
- **`volatile`**: Indicates that a variable's value might be modified asynchronously by different threads, forcing the program to read it directly from main memory.
- **`transient`**: Indicates that a variable should not be serialized (saved) when the object is converted into a byte stream.
- **`native`**: Indicates that a method is implemented in native code (usually C or C++) using JNI (Java Native Interface).

### 9. Package Management (2 Keywords)
- **`package`**: Specifies the namespace/directory for a Java class file.
- **`import`**: Brings classes or entire packages into your current file so you can use them.

### 10. Miscellaneous & Reserved (5 Keywords)

- **`assert`**: Used for debugging purposes to evaluate an expression that you expect to be true.
- **`strictfp`**: Used to restrict floating-point calculations to ensure they produce identical results on any hardware (mostly obsolete since Java 17).
- **`_` (underscore)**: As of Java 9, a single underscore is a reserved keyword and cannot be used as a variable name.
- **`goto`**: Reserved keyword, but strictly **unused** in Java (retained to prevent C/C++ programmers from using it).
- **`const`**: Reserved keyword, but strictly **unused** in Java (use `final` instead).

### Technical Distinctions (Not Keywords, but Reserved)
There are a few words often mistaken for keywords that are technically **Reserved Literals** or **Restricted Identifiers**:

- **Reserved Literals**: `true`, `false`, and `null`. You cannot use them as variable names, but technically they represent constant values, not keywords. 
- **Restricted Identifiers (Modern Java)**: In newer versions of Java (10+), words like `var`, `yield`, `record`, `sealed`, `permits`, and `non-sealed` have been introduced. These act like keywords in specific contexts (e.g., when defining a `record`), but they can still be used as variable names elsewhere in your code to ensure backwards compatibility.