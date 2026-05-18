While the JVM handles a lot of complex tasks under the hood, we can break it down into three main subsystems: the **Class Loader**, the **Memory Area** (which contains the Heap, Stack, etc.), and the **Execution Engine**.

### Class Loader Subsystem
Before the JVM can run anything, it needs to load your compiled `.class` files into memory. The Class Loader handles this in three phases:

1. **Loading:** It reads the `.class` file and generates binary data representing the class.

2. **Linking:** It verifies the bytecode for security, allocates memory for class variables, and resolves symbolic references into direct memory references.

3. **Initialization:** It assigns the original values to any static variables and executes static initialization blocks.

### Runtime Data Areas (The JVM Memory)
Once the classes are loaded, the JVM allocates memory to run the program. This is divided into five specific regions:

##### 1. Method Area (Shared)
This is where the JVM stores class-level data. It holds the class structure, metadata, static variables, and the runtime constant pool. Because it stores data for the class itself, there is only one Method Area shared across the entire JVM.

##### 2. Stack (Thread-Specific)
Think of the Stack like a stack of plates - Last-In, First-Out ([LIFO](https://www.investopedia.com/terms/l/lifo.asp)). It's a fast, efficient memory area used for **temporary data** and **method execution**.
- **What it holds:** Primitive data types (`int`, `boolean`, `double`, etc.) and the **references (pointers)** to objects that live on the Heap.
- **How it works:** Each time a thread calls a method, a new "stack frame" is pushed onto the stack. This frame contains the method's local variables. When the method finishes running, its frame is "popped" off the stack, immediately freeing that memory.

**Note:** Since the Heap doesn't automatically empty itself like the Stack, Java needs a mechanism to clean up objects that are no longer being used. This automatic process is called **Garbage Collection**. The JVM's GC process runs in the background. It identifies objects on the Heap that are no longer accessible, meaning no variable on any Stack (or any other reference) points to them. It reclaims the memory occupied by these "unreachable" objects, freeing up space for new allocations.

##### 3. Heap (Shared)
The Heap is a large, shared memory area used for **long-term storage**.
- **What it holds:** **Every single Object** you create in Java lives here (e.g., a `new Dog()`, `new Person()`).
- **How it works:** When you use the `new` keyword, the JVM allocates memory for that specific object on the Heap. Unlike the Stack, objects on the Heap are not automatically removed when a method ends; they stay there as long as your program has a reference variable (on the Stack) pointing to them.

##### 4. PC (Program Counter) Register (Thread-Specific)
Because Java is multithreaded, a thread might be paused while the CPU works on something else. The PC Register keeps track of the exact memory address of the instruction that the thread is currently executing, so it knows exactly where to pick up when it resumes.

##### 5. Native Method Stack (Thread-Specific)
Sometimes Java needs to interact with hardware or OS-level code written in C or C++ (called Native Methods). This stack operates exactly like the regular Java Stack, but it is dedicated exclusively to managing these native C/C++ method calls.