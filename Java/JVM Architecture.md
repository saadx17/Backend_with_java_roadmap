While the JVM handles a lot of complex tasks under the hood, we can break it down into three main subsystems: 
1. The **Class Loader**
2. The **Runtime Memory Area** (which contains the Heap, Stack, etc.)
3. The **Execution Engine**.

```
┌──────────────────────────────────────────────────────────┐
│                        JVM                               │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │            Class Loader Subsystem                │    │
│  │  ┌──────────────┐ ┌───────────┐ ┌─────────────┐  │    │
│  │  │  Bootstrap   │ │Extension  │ │ Application │  │    │
│  │  │ ClassLoader  │ │ClassLoader│ │ ClassLoader │  │    │
│  │  └──────────────┘ └───────────┘ └─────────────┘  │    │
│  └──────────────────────────────────────────────────┘    │
│                          │                               │
│                          ▼                               │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Runtime Memory Areas                │    │
│  │  ┌──────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ │    │
│  │  │  Heap    │ │Stack   │ │  Method│ │   PC     │ │    │
│  │  │(objects) │ │(frames │ │  Area  │ │ Register │ │    │
│  │  │          │ │locals) │ │ (class │ │          │ │    │
│  │  │  GC here │ │        │ │ meta)  │ │          │ │    │
│  │  └──────────┘ └────────┘ └────────┘ └──────────┘ │    │
│  │  ┌─────────────────────────────────────────────┐ │    │
│  │  │         Native Method Stack                 │ │    │
│  │  └─────────────────────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────┘    │
│                          │                               │
│                          ▼                               │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Execution Engine                    │    │
│  │  ┌────────────┐ ┌───────────────┐ ┌──────────┐   │    │
│  │  │Interpreter │ │  JIT Compiler │ │    GC    │   │    │
│  │  │(slow,line  │ │(fast, compiles│ │ (memory  │   │    │
│  │  │  by line)  │ │  hot code)    │ │ cleanup) │   │    │
│  │  └────────────┘ └───────────────┘ └──────────┘   │    │
│  └──────────────────────────────────────────────────┘    │
│                          │                               │
│                          ▼                               │
│  ┌──────────────────────────────────────────────────┐    │
│  │         Native Method Interface (JNI)            │    │
│  │         Native Method Libraries (C/C++)          │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Class Loader Subsystem
Before the JVM can run anything, it needs to load your compiled `.class` files into memory. The Class Loader handles this in three phases:

1. **Loading:** It reads the `.class` file and generates binary data representing the class.

2. **Linking:** It verifies the bytecode for security, allocates memory for class variables, and resolves symbolic references into direct memory references.

3. **Initialization:** It assigns the original values to any static variables and executes static initialization blocks.
#### The Class Loader Hierarchy
##### Bootstrap ClassLoader
- Loads core Java classes (java.lang.*, java.util.* etc.)
- Written in C/C++ - part of JVM itself
- Loads from: $JAVA_HOME/lib/rt.jar (Java 8)
- $JAVA_HOME/jmods/     (Java 9+)

##### Extension ClassLoader
- Loads from: $JAVA_HOME/lib/ext/
- Loads extension/optional libraries
- Parent = Bootstrap

##### Application ClassLoader
- Loads YOUR classes + third-party JARs
- Loads from: classpath (-cp flag, CLASSPATH env)
- Parent = Extension
- Also called: System ClassLoader

##### `classLoader` Example
```java title:classloader.java
// ─── See which ClassLoader loaded a class ─────────────────
public class ClassLoaderDemo {
    public static void main(String[] args) {
        // Bootstrap → null (implemented in C, no Java object)
        System.out.println(String.class.getClassLoader());
        // null  ← Bootstrap ClassLoader

        // Application ClassLoader
        System.out.println(ClassLoaderDemo.class.getClassLoader());
        // jdk.internal.loader.ClassLoaders$AppClassLoader@...

        // Parent delegation chain
        ClassLoader cl = ClassLoaderDemo.class.getClassLoader();
        while (cl != null) {
            System.out.println(cl);
            cl = cl.getParent();
        }
        // AppClassLoader → PlatformClassLoader → null (Bootstrap)
    }
}
```

#### Parent Delegation Model
The **Parent Delegation Model** is ==the JVM's strategy for loading classes==. When a class loader receives a request to load a class, it does not immediately load it. Instead, it delegates the request to its **parent class loader**. The request travels up the hierarchy; if the parent can load it, it does. If not, the request returns to the child.
When ClassLoader gets request to load class:
1. Ask PARENT first → don't load yourself
2. Parent asks ITS parent
3. Goes all the way up to Bootstrap
4. If Bootstrap can't find it → Extension tries
5. If Extension can't find it → Application tries
6. If nobody finds it → ClassNotFoundException

##### Why is it used?
- **Security:** It prevents malicious code from replacing core Java classes (like `java.lang.String`), because core classes are always loaded by the highest-level trusted loader.
- **Consistency:** It ensures that classes are loaded consistently across the JVM, as the parent is always given the first opportunity to load them.
- **No Duplication:** It stops the same class from being loaded multiple times in memory.

##### How the Process Works (Step-by-Step)
1. **Request:** The Application Class Loader is asked to load a class.
2. **Delegate Up:** Before searching its own path, it delegates the request to its parent (the Platform Class Loader).
3. **Delegate to Top:** The Platform Class Loader delegates to its parent (the Bootstrap Class Loader).
4. **Load Attempt:** The Bootstrap Class Loader checks its core libraries. If the class is found, it loads it. If not, the request returns down the chain to the Platform Class Loader.
5. **Local Load:** If neither parent can find the class, the Application Class Loader finally attempts to load it from the application's local classpath.

### Runtime Memory Areas
The Java Virtual Machine (JVM) manages memory during program execution by dividing it into **five distinct Runtime Data Areas**. These areas are broadly classified into two categories:
1. Thread-Shared (accessible by all threads)
2. Thread-Specific (private to an individual thread)

| Attribute                | Thread-Shared Memory (Heap, Metaspace)      | Thread-Specific Memory (Stack, PC, Native Stack)   |
| ------------------------ | ------------------------------------------- | -------------------------------------------------- |
| **Scope**                | Shared by all threads in the process.       | Private to one thread.                             |
| **Garbage Collection**   | Yes. Periodically swept by GC cycles.       | No. Automatically dropped when a method completes. |
| **Primary Failure Type** | `OutOfMemoryError`.                         | `StackOverflowError`.                              |
| **Access Latency**       | Variable; can suffer from CPU cache misses. | Extremely fast.                                    |

#### 1. Thread-Shared
In the Java Virtual Machine, **all threads share a single, global Heap Memory and Metaspace**, allowing them to communicate and exchange data efficiently.

| Memory Area                 | Main Purpose                               | Managed By / Cleanup   |
| --------------------------- | ------------------------------------------ | ---------------------- |
| **Heap Area**               | Stores all objects and arrays              | Garbage Collector      |
| **Method Area (Metaspace)** | Stores class metadata and static variables | Class Loader lifecycle |
##### Heap Memory
JVM heap memory is a shared runtime data area where the JVM allocates memory for all **Java objects, arrays, and instance variables**. Created automatically when the JVM starts up, it serves as the primary playground for dynamic memory allocation and is systematically managed by the Garbage Collector (GC).

- **What it holds:** **Every single Object** you create in Java lives here (e.g., a `new Dog()`, `new Person()`).
- **How it works:** When you use the `new` keyword, the JVM allocates memory for that specific object on the Heap. Unlike the Stack, objects on the Heap are not automatically removed when a method ends; they stay there as long as your program has a reference variable (on the Stack) pointing to them.

**Note:** Since the Heap doesn't automatically empty itself, Java needs a mechanism to clean up objects that are no longer being used. This automatic process is called **Garbage Collection**. The JVM's Garbage collection process runs in the background. It identifies objects on the Heap that are no longer accessible, meaning no variable on any Stack (or any other reference) points to them. It reclaims the memory occupied by these "unreachable" objects, freeing up space for new allocations.

##### Method Area (Metaspace)
This is where the JVM stores class-level data. It holds the **class structure, metadata, static variables, and the runtime constant pool**. Because it stores data for the class itself, there is only one Method Area shared across the entire JVM.
- **What it holds:** stores **class-level metadata** and application blueprints.
- **How it works:** When a class is loaded by the JVM, its structural blueprint is stored in Metaspace.

**Metaspace** is a native memory region in the HotSpot JVM introduced in Java 8 to store **class metadata**. It completely replaced the older **Permanent Generation (PermGen)** space. The most critical difference is its location: while PermGen was part of the contiguous Java Heap and had a fixed maximum size, Metaspace is allocated from **off-heap native memory** and expands dynamically by default based on the available system memory.

#### 2. Thread-Specific
In the Java Virtual Machine, thread-specific memory (also known as thread-private memory) refers to the allocated memory regions that are **strictly isolated and accessible exclusively by a single thread**.

| Memory Area             | Main Purpose                                  | Managed By / Cleanup                  |
| ----------------------- | --------------------------------------------- | ------------------------------------- |
| **JVM Stack Area**      | Stores local variables and method call frames | Automatically on method return        |
| **PC Register**         | Tracks current bytecode instruction address   | Automatically on thread death         |
| **Native Method Stack** | Handles native code (C/C++) instructions      | Automatically on native method return |
##### Stack
Think of the Stack like a stack of plates - Last-In, First-Out ([LIFO](https://www.investopedia.com/terms/l/lifo.asp)). It's a fast, efficient memory area used for **temporary data** and **method execution**.

- **What it holds:** Primitive data types (`int`, `boolean`, `double`, etc.) and the **references (pointers)** to objects that live on the Heap.
- **How it works:** Each time a thread calls a method, a new "stack frame" is pushed onto the stack. This frame contains the method's local variables. When the method finishes running, its frame is "popped" off the stack, immediately freeing that memory.

##### PC Register
The **Program Counter (PC) Register** in the Java Virtual Machine (JVM) is a **thread-private memory area** that stores the address or offset of the **JVM bytecode instruction currently being executed**.

- **What it holds:** PC register in the JVM holds the memory address (or offset) of the **currently executing bytecode instruction**.
- **How it works:** PC reguster is a specialized, high-speed register inside a CPU that **keeps track of memory addresses**. It stores the exact location of the **next instruction** to be executed. It functions as a roadmap, directing the processor to the correct place in the main memory (RAM)

##### Native Method Stack
Sometimes Java needs to interact with hardware or OS-level code written in C or C++ (called Native Methods). This stack operates exactly like the regular Java Stack, but it is dedicated exclusively to managing these native C/C++ method calls.
- **What it holds:** it holds information and state for execution of methods written in languages other than Java (typically C or C++). When a Java program calls a native method using the **Java Native Interface (JNI)**, the thread switches from the standard Java stack to the Native Method Stack.
- **How it works:** By acting as a mechanical bridge between the Java runtime environment and the underlying operating system.

### Execution Engine
Execution Engine takes platform-independent bytecode from the runtime memory areas and translates it into machine-specific native code for the host operating system to execute. It bridges the gap between Java's "Write Once, Run Anywhere" philosophy and physical hardware performance.
The engine relies on three primary elements to execute and optimize bytecode:
```
1. INTERPRETER
   ├── Reads bytecode line by line
   ├── Executes immediately
   ├── Slow for repeated code
   └── Used for code run just once or a few times

2. JIT COMPILER (Just-In-Time)
   ├── Monitors code for "hot spots" (frequently run code)
   ├── Compiles hot bytecode → native machine code
   ├── Native code runs MUCH faster (no interpretation)
   ├── Subsequent calls use compiled native code directly
   └── This is why Java is fast despite being "interpreted"!

3. GARBAGE COLLECTOR
   ├── Automatically frees heap memory
   ├── Finds objects with no references
   └── Multiple algorithms available
```

#### 1. The Interpreter
The **Interpreter** is a core component of the JVM Execution Engine that executes platform-independent bytecode by converting it into machine-specific native code line by line at runtime. It serves as the initial execution mechanism when a Java application starts up, allowing the program to run immediately without waiting for a lengthy ahead-of-time compilation process.

The Interpreter acts as a translation layer between the compiled `.class` bytecode and the underlying physical hardware:

- **Instruction Fetch:** It reads individual bytecode instructions sequentially from the Method Area, relying on the **Program Counter (PC) Register** to track the address of the next instruction.
- **Decoding & Execution:** It decodes each 1-byte OpCode along with its operands, transforms them into corresponding native machine commands, and executes them on the fly.

##### The Efficiency Problem
While the Interpreter enables an **instant application startup**, executing code sequentially comes with a significant performance penalty:

- **Repetitive Translation:** The interpreter does not remember what it has previously translated. If a specific method or loop is called a thousand times, it will blindly translate and execute those exact same lines a thousand times.
- **Slow Overall Execution:** Line-by-line interpretation requires additional processor overhead, making interpreted Java slower than languages compiled directly to native code (like C or C++).

#### 2. JIT
**Just-In-Time** (JIT) compilation is the engine that gives Java its incredible, near-native performance while maintaining its "Write Once, Run Anywhere" portability. It is a dynamic compilation technique performed by the JVM **while your program is running (at runtime).**

To be cross-platform, Java compiles your code not into the binary `1`s and `0`s for your specific computer (AOT - Ahead-of-Time compilation like in C++), but into **bytecodes** (`.class` files). When you run a program, the JVM uses an **Interpreter** to read this universal bytecode one instruction at a time and translate it into machine code. This process works everywhere, but it is relatively **slow**.
This is where JIT comes in. As your program runs, the JVM doesn't just keep translating line by line. It dynamically profiles and monitors your code execution, looking for what it calls **hotspots**:

- **Method Invocation Counters:** How many times has this particular method (like `calculateTax()`) been called?

- **Back-Edge (Loop) Counters:** How many times has this loop iterated?

Once the frequency of a particular method or loop exceeds a conceptual threshold (meaning the code is "hot"), the JVM decides the time spent optimizing this section will be well worth the significant speed boost it will deliver. It activates the **JIT Compiler**.

##### How JIT Works?
###### Source to Bytecode (Compile Time)
`.java` is compiled by `javac` into universal bytecode.

```java title:bytecode.java
// What you write
int sum(int a, int b) {
  return a + b;
}
```

###### Interpreter (Runtime)
The program starts. The Interpreter reads the bytecode, translating it instruction by instruction for execution. It's safe and portable, but slow.
```
// Internal conceptual view
interpreter -> loads 'sum' method bytecode
interpreter -> translates 'iadd' instruction
interpreter -> translates 'iload_1' instruction
... (slow line-by-line execution) ...
```

###### Hotspot Detection (Runtime)
The dynamic profiler notices the `sum` method has been called millions of times. It marks the method as "hot" and passes it to the JIT compiler.

###### JIT Compiler & Optimizations (Runtime)
Instead of translating instruction by instruction, the JIT compiler analyzes the _entire method at once_ and translates it into highly optimized, platform-specific **Native Machine Code** (`1`s and `0`s). It can perform sophisticated optimizations that the initial Interpreter could never do because it only saw one line at a time:

- **Method In lining:** Instead of an expensive call to another method, the JIT may replace the call with the actual method body, saving time.
- **Constant Folding/Propagation:** It pre-calculates expressions with constants (e.g., `5 * 10` becomes `50`).
- **Loop Unrolling:** It unrolls loops to process multiple iterations in a single step.
The optimized native machine code is then stored in a special memory area called the code cache.

###### Native Execution (Runtime)
The next time the `sum` method is called, the JVM completely **bypasses the Interpreter**. It directly runs the pre-compiled, heavily optimized native code from the code cache, rivaling or even exceeding pre-compiled languages in raw speed. This is why you will often notice that a Java program actually gets _faster_ as it continues to run and "warms up."
This interactive visualization lets you explore this transformation dynamically, watching the Interpreter identify a hotspot and witnessing how JIT compilation and common optimizations lead to a massive leap in execution speed.

#### 3. Garbage Collection
The **Garbage Collector (GC)** is a core component of the JVM's **Execution Engine**, operating as a background daemon thread that automatically manages memory by reclaiming heap space from unreachable objects. It eliminates the need for manual memory deallocation, protecting applications from memory leaks and `OutOfMemoryError` failures.

##### How the Garbage Collector Works
The Execution Engine executes garbage collection using a three-phase operational workflow:

###### 1. Marking (Liveness Analysis)
The GC identifies which objects in the heap are still in use and which are abandoned. It starts from a set of active references known as **GC Roots** (such as local variables on thread stacks, active thread objects, and static variables) and traces the object graph.

- **Live Objects**: Accessible from any GC Root.
- **Dead Objects**: Unreachable from the GC Roots, making them eligible for collection.

###### 2. Sweeping or Copying
Once dead objects are identified, the GC reclaims their memory space.

- **Sweeping**: Clears dead objects, leaving the remaining objects in place. This can lead to memory fragmentation.
- **Copying**: Moves live objects to a totally separate, clean block of memory, automatically wiping the old block to leave it completely empty.

###### 3. Compacting
To combat memory fragmentation caused by sweeping, the GC shifts all surviving objects to the beginning of the heap space. This creates a large, continuous block of free memory, allowing for much faster allocation of future objects via simple pointer bumping.

## JVM Tuning Flags
JVM tuning flags are parameters used to optimize Java applications by controlling memory allocation, garbage collection (GC) behavior, and system resource utilization. Proper configuration prevents crashes, reduces latency, and boosts throughput.

### How to do tuning?
To perform Java Virtual Machine (JVM) tuning, you must ==**pass specific flags to the Java command line** when launching your application== to optimize memory management, control the Garbage Collector (GC), and generate diagnostic logs.

```java title:example.java
# ─── Real production example ──────────────────────────────
java \
  -Xms512m \
  -Xmx2g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -Dspring.profiles.active=prod \
  -Dfile.encoding=UTF-8 \
  -jar myapp.jar
```

#### 1. Memory and Heap Sizing

- `-Xms`: Sets the **initial heap size** (e.g., `-Xms512m`). Prevents latency spikes caused by the JVM needing to expand the heap during application runtime.
- `-Xmx`: Sets the **maximum heap size** (e.g., `-Xmx4g`). Crucial for preventing `OutOfMemoryError` (OOM), but avoid making it too large, as it can lead to longer GC pause times.
- `-XX:MaxMetaspaceSize`: Limits memory for class definitions and metadata (e.g., `-XX:MaxMetaspaceSize=512m`). Prevents unbounded native memory leaks.

#### 2. Container and Cloud Settings

- `-XX:+UseContainerSupport`: (Enabled by default in modern Java versions). Tells the JVM to respect memory and CPU limits enforced by Docker or Kubernetes, preventing OOM kills.
- `-XX:MaxRAMPercentage`: Sets the maximum percentage of container/host RAM that the JVM should use for the heap (e.g., `-XX:MaxRAMPercentage=75.0`). Safely scales the heap if the container memory limit changes.

#### 3. Garbage Collection (GC) Selection

- `-XX:+UseG1GC`: The default GC for most modern enterprise applications. It balances throughput and latency.
- `-XX:+UseZGC`: Ideal for applications requiring ultra-low latency. It minimizes GC pause times regardless of the heap size.
- `-XX:MaxGCPauseMillis`: Sets a target for the maximum amount of time the JVM will pause execution to run a GC cycle (e.g., `-XX:MaxGCPauseMillis=200`).

#### 4. Diagnostics and Monitoring

- `-XX:+HeapDumpOnOutOfMemoryError`: Automatically generates a heap dump file if the application crashes with an Out Of Memory error.
- `-XX:HeapDumpPath=/path/to/dumps`: Specifies the directory where the heap dump file should be saved.
- `-Xlog:gc*`: Enables detailed garbage collection logging for performance analysis.

# How code executes in Java Ecosystem

```java title:syntax_myApp.java
// STEP 1: You write MyApp.java (source code)
// ─────────────────────────────────────────
public class MyApp {
    public static void main(String[] args) {
        int x = 5;
        int y = 10;
        System.out.println(x + y);
    }
}
```

```
STEP 2: javac MyApp.java → MyApp.class (bytecode)
─────────────────────────────────────────
javac reads .java → parses → checks syntax
→ type checking → generates bytecode

Bytecode (simplified view of MyApp.class):
0: bipush 5        (push 5 onto stack)
2: istore_1        (store in variable x)
3: bipush 10       (push 10 onto stack)
5: istore_2        (store in variable y)
6: getstatic       (get System.out)
9: iload_1         (load x)
10: iload_2        (load y)
11: iadd           (add them)
12: invokevirtual  (call println)
15: return
```

```
STEP 3: java MyApp → JVM loads & runs
─────────────────────────────────────────
a) ClassLoader loads MyApp.class
b) Bytecode verifier checks it's safe
c) Interpreter starts executing bytecode
d) JIT monitors → compiles hot methods to native
e) GC manages memory throughout
f) Program exits → JVM shuts down
```
