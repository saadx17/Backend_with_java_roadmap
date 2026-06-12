*Write Once, Run Anywhere.*
# How does Java actually works?
Here is how JAVA works from start to finish:

1. **Source Code (`.java`):** This is the ==human-readable== code you write. It's full of English-like words and Java syntax (like the `Dog` blueprint we discussed earlier).

2. **Compiler (`javac`):** This is Java's proofreader and ==initial translator==. When you command the system to build your program, `javac` reads your `.java` file. If your syntax is perfectly correct, it translates your code into an intermediate format.

3. **Bytecode (`.class`):** This is the ==output from the compiler==. It is not human-readable, but it's also not true machine code yet. It is a **universal, platform-independent** instruction set meant specifically for Java machines.

4. **JVM (Java Virtual Machine):** When you actually ==_run_== your program, the JVM steps in. It loads your universal `.class` bytecode.

5. **Machine Code:** As the program runs, the JVM reads the bytecode and translates it _on the fly_ into the specific ==1s and 0s (machine code)== that your exact computer hardware (whether it's a Windows PC, a Mac, or a Linux server) can execute physically.

# What happens in the code?
**Syntax:**
```java title:Hello.java
public class Hello{
public static void main(String[] args){
System.out.println("Hello, World!"); }
}
```

#### Breaking down EVERY single word

```java title:Hello.java
public class Hello {
```

- `public` → This class is accessible from anywhere.
- `class` → We are defining a class (everything in Java lives inside a class)
- `Hello` → The name of the class (must match the filename: `Hello.java`)
- `{` → Opens the class body.

```java title:Hello.java
public static void main(String[] args) {
```

- `public` → The method is accessible from anywhere (JVM needs to access it)
- `static` → This method belongs to the class itself, not an object. JVM can call it without creating an object.
- `void` → This method returns nothing.
- `main` → Special name — this is the **entry point** of the program. JVM always looks for `main` to start.
- `String[] args` → An array of Strings passed from command line (can be ignored for now)
- `{` → Opens the method body.

```java title:HelloWorld.java
System.out.println("Hello, World!");
```

- `System` → A built-in Java class that provides system-level utilities.
- `out` → A static field in System — represents the standard output stream (your console/terminal)
- `println` → A method that prints text and adds a new line at the end
- `"Hello, World!"` → A String literal - the text to print.
- `;` → Every statement in Java ends with a semicolon.

```java title:Hello.java
  }
}
```

- First `}` → Closes the `main` method.
- Second `}` → Closes the `Hello` class.

| Rule                             | Explanation                                    |
| -------------------------------- | ---------------------------------------------- |
| File name must match class name  | `Hello.java` must contain `public class Hello` |
| Java is case-sensitive           | `Main` ≠ `main`                                |
| Every statement ends with `;`    | `System.out.println("Hi");`                    |
| `main` method is the entry point | JVM always starts from `main`                  |
| Everything lives inside a class  | You cannot write code outside a class          |

#### Full Flow Diagram
```
Hello.java
│
│->    javac Hello.java (Compilation step - done by JDK)
│
▼ Hello.class (bytecode)
│
│->    java Hello (Run step - done by JVM)
│
▼
JVM loads the class
│
▼
JVM finds main() method (Entry point of every Java program)
│
▼
Executes instructions
│
▼
Output: Hello World
```
