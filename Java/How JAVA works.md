*Write Once, Run Anywhere.*
# How does Java actually works?
Here is how JAVA works from start to finish:

1. **Source Code (`.java`):** This is the ==human-readable== code you write. It's full of English-like words and Java syntax (like the `Dog` blueprint we discussed earlier).

2. **Compiler (`javac`):** This is Java's proofreader and ==initial translator==. When you command the system to build your program, `javac` reads your `.java` file. If your syntax is perfectly correct, it translates your code into an intermediate format.

3. **Bytecode (`.class`):** This is the ==output from the compiler==. It is not human-readable, but it's also not true machine code yet. It is a **universal, platform-independent** instruction set meant specifically for Java machines.

4. **JVM (Java Virtual Machine):** When you actually ==_run_== your program, the JVM steps in. It loads your universal `.class` bytecode.

5. **Machine Code:** As the program runs, the JVM reads the bytecode and translates it _on the fly_ into the specific ==1s and 0s (machine code)== that your exact computer hardware (whether it's a Windows PC, a Mac, or a Linux server) can execute physically.

# What happens in the code?
Let's trace exactly what happens from typing code to seeing output.

1. **How to write source code:**
You create a file called `Hello.java`
**Code:**
```
public class Hello { public static void main(String[] args){ System.out.println("Hello World"); }
}
```

2. **Compilation**
You run the Java compiler:
```
javac Hello.java
```

- `javac` is the **Java Compiler** (part of JDK)
- It reads `Hello.java`
- It checks for syntax errors
- If no errors → it produces `Hello.class`
- `Hello.class` contains **bytecode**

3. **Running:**
You run the program:
```
java Hello
```

4. **Full Flow Diagram:**
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
