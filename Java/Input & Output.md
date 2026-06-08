In Java, **standard console output** is handled using the `System.out` object, while **user input** is most easily processed using the `Scanner` class from the `java.util` package.

Java I/O = how your program communicates with outside world.

```
Input  → reading data INTO  your program
Output → sending data OUT of your program

3 Standard Streams (always available):
├── System.in  → keyboard input  (InputStream)
├── System.out → console output  (PrintStream)
└── System.err → error output    (PrintStream)

Main ways to read input:
├── Scanner       → easy, beginner friendly
├── BufferedReader → faster, more control
└── Console       → secure (passwords)
```

## 1. `System.out` - Output
**`System.out` represents the standard output stream**, which is typically mapped to the console or terminal window. It is a built-in static member variable of the `System` class used primarily to print text, numbers, and variables for debugging or communicating with the user.

```java title:print.java
// print → no newline at end
System.out.print("Hello");
System.out.print(" World");
System.out.print("!");
// Output: Hello World!   (all on one line)
```

```java title:println.java
// println → adds newline at end
System.out.println("Hello");
System.out.println("World");
System.out.println("!");
// Output:
// Hello
// World
// !
```

```java title:println2.java
// println with no argument → just newline
System.out.println("First paragraph");
System.out.println();           // blank line
System.out.println("Second paragraph");
```

```java title:printf.java
// printf → formatted output (no newline by default)
System.out.printf("Hello %s!%n", "World");  // %n = newline
System.out.printf("Age: %d%n", 25);
System.out.printf("Score: %.2f%n", 95.678);
```

### Format Specifiers
In Java, **format specifiers act as placeholders** within formatting methods like `System.out.printf()`, `System.out.format()`, and `String.format()`, indicating how variables should be converted into text.

```java title:basic_specifiers.java
// Basic specifiers
System.out.printf("%s%n",  "Alice");      // String   → Alice
System.out.printf("%d%n",  42);           // integer  → 42
System.out.printf("%f%n",  3.14159);      // float    → 3.141590
System.out.printf("%.2f%n", 3.14159);     // 2 decimal→ 3.14
System.out.printf("%b%n",  true);         // boolean  → true
System.out.printf("%c%n",  'A');          // char     → A
System.out.printf("%n");                  // newline

// Width & Alignment
System.out.printf("%10s%n",  "Alice");    // right align in 10 →      Alice
System.out.printf("%-10s%n", "Alice");    // left  align in 10 → Alice
System.out.printf("%10d%n",  42);         // right align       →         42
System.out.printf("%-10d|%n", 42);        // left  align       → 42        |

// Numbers
System.out.printf("%05d%n",   42);        // zero padded       → 00042
System.out.printf("%+d%n",    42);        // always show sign  → +42
System.out.printf("%,d%n",    1000000);   // thousand sep      → 1,000,000
System.out.printf("%e%n",     123456.789);// scientific        → 1.234568e+05
System.out.printf("%10.2f%n", 3.14);      // width + decimal   →       3.14

// Formatted table example
System.out.printf("%-15s %5s %10s%n", "Name", "Age", "Salary");
System.out.printf("%-15s %5d %10.2f%n", "Alice",   25, 75000.50);
System.out.printf("%-15s %5d %10.2f%n", "Bob",     30, 92000.00);
System.out.printf("%-15s %5d %10.2f%n", "Charlie", 22, 65000.75);

// Name              Age     Salary
// Alice              25   75000.50
// Bob                30   92000.00
// Charlie            22   65000.75
```

### `System.err` - Error
**`System.err`** is a built-in static instance of the `PrintStream` class in Java used specifically to output error messages and diagnostic logging data. While it writes to the console by default just like `System.out`, it routes data through a completely separate stream known as **Standard Error (stderr)**.

```java title:err.java
// System.err → for error messages
// Separate stream from System.out
// Shows in RED in most IDEs

System.out.println("Normal output");    // stdout
System.err.println("Error occurred!");  // stderr (red)

// Real use
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.err.println("Error: " + e.getMessage());  // Error: / by zero
    System.err.printf("Exception type: %s%n",
                       e.getClass().getSimpleName());
}

// Why separate?
// In production → stdout and stderr can be redirected separately
// java MyApp > output.log 2> errors.log
```


## 2. `Scanner`- Input
To use the **scanner class** for user input in Java, you must import the **`java.util.Scanner`** package, create a `Scanner` object pointing to **`System.in`**, and use its built-in methods to capture data.

#### Syntax
```java title:scanner.java
import java.util.Scanner;  // ← MUST import first

public class Main {
    public static void main(String[] args) {

        Scanner sc = new Scanner(System.in);  // attach to keyboard

        // ... read input ...

        sc.close();  // always close when done (good practice)
    }
}
```

#### Reading Different Data Types
**In enterprise backend systems, you will almost never use `Scanner` to read from `System.in`.**
```
┌─────────────────────┬──────────────────────────────────┐
│ Method              │ What it does                     │
├─────────────────────┼──────────────────────────────────┤
│ nextInt()           │ reads integer                    │
│ nextDouble()        │ reads double                     │
│ nextLong()          │ reads long                       │
│ nextFloat()         │ reads float                      │
│ nextBoolean()       │ reads true/false                 │
│ next()              │ reads one word (no spaces)       │
│ nextLine()          │ reads full line with spaces      │
├─────────────────────┼──────────────────────────────────┤
│ hasNextInt()        │ checks if next token is int      │
│ hasNextDouble()     │ checks if next token is double   │
│ hasNextLine()       │ checks if another line exists    │
│ hasNext()           │ checks if any token exists       │
├─────────────────────┼──────────────────────────────────┤
│ useDelimiter(",")   │ split by custom delimiter        │
│ close()             │ close the scanner                │
└─────────────────────┴──────────────────────────────────┘
```

```java title:scanner.java
import java.util.Scanner;

// Attach to keyboard
Scanner sc = new Scanner(System.in);

// Read different types
int    i = sc.nextInt();       // reads integer
double d = sc.nextDouble();    // reads double
long   l = sc.nextLong();      // reads long
float  f = sc.nextFloat();     // reads float
boolean b= sc.nextBoolean();   // reads true/false
String w = sc.next();          // reads ONE word
String line = sc.nextLine();   // reads FULL line

sc.close();
```

#### Scanner from File
```java title:filescanner.java
import java.util.Scanner;
import java.io.File;
import java.io.FileNotFoundException;

public class Main {
    public static void main(String[] args) {
        try {
            File file = new File("data.txt");
            Scanner sc = new Scanner(file);

            while (sc.hasNextLine()) {
                String line = sc.nextLine();
                System.out.println(line);
            }

            sc.close();

        } catch (FileNotFoundException e) {
            System.out.println("File not found: " + e.getMessage());
        }
    }
}
```

#### Scanner from String (Useful for Parsing)

```java title:string_Scanner.java
// Instead of keyboard, scan a String
String data = "John 25 3.14";
Scanner sc = new Scanner(data);

String name   = sc.next();      // "John"
int    age    = sc.nextInt();   // 25
double value  = sc.nextDouble();// 3.14

System.out.println(name + " " + age + " " + value);
```

```java title:CSVLine.java
// Parse CSV line
String csv = "Alice,30,Engineer";
Scanner sc2 = new Scanner(csv).useDelimiter(",");

String personName = sc2.next();   // Alice
int    personAge  = sc2.nextInt();// 30
String role       = sc2.next();   // Engineer

System.out.println(personName + " is " + personAge + " years old");
```

#### Input Validation with Scanner
```java title:inputva_scanner.java
Scanner sc = new Scanner(System.in);

// Check before reading
System.out.print("Enter a number: ");

if (sc.hasNextInt()) {
    int num = sc.nextInt();
    System.out.println("You entered: " + num);
} else {
    System.out.println("That's not a number!");
}

// Loop until valid input
int number = 0;
boolean valid = false;

while (!valid) {
    System.out.print("Enter an integer: ");
    if (sc.hasNextInt()) {
        number = sc.nextInt();
        valid = true;               // exit loop
    } else {
        System.out.println("Invalid! Try again.");
        sc.next();                  // discard bad input
    }
}
System.out.println("Valid number: " + number);
```



## 3. `BufferedReader` - Faster Input
In Java, **`BufferedReader`** is a class used to read text efficiently from a character input stream by buffering characters, making it highly preferred for large data inputs and competitive programming due to its speed.

### Why `BufferedReader`?
For input in Java primarily because it provides **highly efficient data reading, superior performance with large inputs, and native thread safety**. While classes like `Scanner` are easier for beginners, `BufferedReader` is the professional choice for production-grade applications, heavy data files, and competitive programming.

- **Faster Performance**: It minimizes expensive I/O disk or stream actions by fetching data in large memory blocks rather than single characters.
- **Large Memory Capacity**: It comes with a default internal buffer of **8 KB (8192 characters)**, dwarfing the 1 KB buffer size of a standard `Scanner`.
- **No Parsing Overhead**: It treats all incoming data seamlessly as raw character strings without running complex regular expressions behind the scenes.
- **Thread Safety**: Its operations are fully **synchronized**, allowing safe execution across multi-threaded applications.
- **Line-by-Line Reading**: It features a built-in `readLine()` method, simplifying line-by-line file and stream ingestion.

### Basic Usage
```java title:buffer_reader.java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.IOException;

public class Main {
    public static void main(String[] args) throws IOException {

        // Wrap System.in with InputStreamReader, then BufferedReader
        BufferedReader br = new BufferedReader(
                                new InputStreamReader(System.in));

        // Read a line
        System.out.print("Enter your name: ");
        String name = br.readLine();
        System.out.println("Hello, " + name);

        // Read and parse integer
        System.out.print("Enter age: ");
        int age = Integer.parseInt(br.readLine().trim());
        System.out.println("Age: " + age);

        // Read and parse double
        System.out.print("Enter price: ");
        double price = Double.parseDouble(br.readLine().trim());
        System.out.printf("Price: %.2f%n", price);

        br.close();
    }
}
```

### Reading Multiple Values on One Line
```java title:multi_buffer_reader.java
import java.io.*;
import java.util.StringTokenizer;

BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

// User types: "10 20 30"

// Method 1: split
String[] parts = br.readLine().split(" ");
int a = Integer.parseInt(parts[0]);  // 10
int b = Integer.parseInt(parts[1]);  // 20
int c = Integer.parseInt(parts[2]);  // 30

// Method 2: StringTokenizer (fastest for competitive programming)
StringTokenizer st = new StringTokenizer(br.readLine());
int x = Integer.parseInt(st.nextToken());  // 10
int y = Integer.parseInt(st.nextToken());  // 20
int z = Integer.parseInt(st.nextToken());  // 30

// Read array
// First line: n (size)
// Second line: n numbers
int n = Integer.parseInt(br.readLine().trim());
int[] arr = new int[n];
StringTokenizer st2 = new StringTokenizer(br.readLine());
for (int i = 0; i < n; i++) {
    arr[i] = Integer.parseInt(st2.nextToken());
}
System.out.println(Arrays.toString(arr));
```

### Reading File with `BufferedReader`
```java title:syntax.java
import java.io.*;

// Read file line by line
try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {  // null = end of file
        System.out.println(line);
    }
} catch (FileNotFoundException e) {
    System.err.println("File not found: " + e.getMessage());
} catch (IOException e) {
    System.err.println("Error reading: " + e.getMessage());
}

// Java 8+ → read all lines as stream
import java.nio.file.*;

try {
    Files.lines(Path.of("data.txt"))
         .filter(line -> !line.isBlank())
         .map(String::trim)
         .forEach(System.out::println);
} catch (IOException e) {
    System.err.println("Error: " + e.getMessage());
}

// Read all lines as List (Java 11+)
List<String> lines = Files.readAllLines(Path.of("data.txt"));
lines.forEach(System.out::println);
```

### BufferedWriter - Output to File
In Java, the **`BufferedWriter`** class writes text to a character-output stream efficiently by using an internal memory buffer (defaulting to 8192 characters). Instead of triggering a slow disk or network operation for every single character, it saves them up and writes them out in large chunks.

```java title:buffered_writer.java
import java.io.*;

// Write to file
try (BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    bw.write("Hello World");
    bw.newLine();              // platform-independent newline
    bw.write("Second line");
    bw.newLine();
    bw.flush();               // force write to disk
} catch (IOException e) {
    System.err.println("Error writing: " + e.getMessage());
}

// Append to existing file
try (BufferedWriter bw = new BufferedWriter(
                          new FileWriter("output.txt", true))) { // true = append
    bw.write("Appended line");
    bw.newLine();
} catch (IOException e) {
    System.err.println("Error: " + e.getMessage());
}

// PrintWriter (easier, has printf/println)
try (PrintWriter pw = new PrintWriter(new FileWriter("report.txt"))) {
    pw.println("=== Report ===");
    pw.printf("%-15s %5s%n", "Name", "Score");
    pw.printf("%-15s %5d%n", "Alice", 95);
    pw.printf("%-15s %5d%n", "Bob",   87);
} catch (IOException e) {
    System.err.println("Error: " + e.getMessage());
}
```

## 4. `Scanner` vs `BufferedReader`
```java title:scanner_vs_bufferedReader.java
// ════════════════════════════════════════════════
// Reading an integer
// ════════════════════════════════════════════════

// Scanner
Scanner sc = new Scanner(System.in);
int n1 = sc.nextInt();

// BufferedReader
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
int n2 = Integer.parseInt(br.readLine().trim());


// ════════════════════════════════════════════════
// Reading a full line
// ════════════════════════════════════════════════

// Scanner
String line1 = sc.nextLine();

// BufferedReader
String line2 = br.readLine();


// ════════════════════════════════════════════════
// Reading array of integers on one line
// ════════════════════════════════════════════════

// Scanner
int[] arr1 = new int[5];
for (int i = 0; i < 5; i++) arr1[i] = sc.nextInt();

// BufferedReader
int[] arr2 = Arrays.stream(br.readLine().split(" "))
                   .mapToInt(Integer::parseInt)
                   .toArray();
```

### When to Use Which
```
┌──────────────────┬─────────────────┬─────────────────────┐
│                  │ Scanner         │ BufferedReader      │
├──────────────────┼──────────────── ┼─────────────────────┤
│ Ease of use      │ ✅ Very easy    │ ⚠️ More code         │
│ Speed            │ ⚠️ Slower       │ ✅ Much faster       │
│ Type parsing     │ ✅ Built-in     │ ❌ Manual parse      │
│ Large input      │ ❌ Slow         │ ✅ Fast              │
│ File reading     │ ✅ Can do       │ ✅ Better            │
│ String parsing   │ ✅ Good         │ ⚠️ Manual            │
│ Validation       │ ✅ hasNextInt() │ ❌ Try-catch needed  │
│ Competitive prog │ ⚠️ OK for small │ ✅ Preferred         │
│ Production code  │ ✅ Simple tasks │ ✅ File/stream I/O   │
└──────────────────┴─────────────────┴─────────────────────┘
```

##### USE Scanner when,
- Small input, interactive programs
- Learning / assignments
- Input validation needed
- Reading different types easily

##### USE BufferedReader when,
- Large amounts of input
- Competitive programming
- File reading/writing
- Performance matters

## 5. `Console` Class - Secure Input
The **`java.io.Console`** class is a specialized class introduced in Java 6 to handle character-based console input and output. It is specifically designed for secure and synchronized terminal-based operations, such as reading user passwords without echoing characters onto the screen.

```java title:console.java
import java.io.Console;

// For reading passwords (characters hidden)
Console console = System.console();

if (console == null) {
    System.out.println("No console available (running in IDE)");
    // Console is null when running inside most IDEs!
    // Works in terminal only
    return;
}

// Reading normal input
String username = console.readLine("Username: ");

// Reading password (hidden, returns char[]) 
char[] password = console.readPassword("Password: ");

System.out.println("Welcome, " + username);

// ⚠️ Security: clear password from memory after use
java.util.Arrays.fill(password, ' ');

// printf on console
console.printf("Hello %s!%n", username);
```

## I/O Streams Hierarchy
```
java.io
├── InputStream  (bytes in)
│   ├── System.in
│   ├── FileInputStream
│   └── BufferedInputStream
│
├── OutputStream (bytes out)
│   ├── System.out / System.err
│   ├── FileOutputStream
│   └── BufferedOutputStream
│
├── Reader       (chars in)
│   ├── InputStreamReader  ← bridges bytes to chars
│   ├── FileReader
│   └── BufferedReader     ← adds buffering
│
└── Writer       (chars out)
    ├── OutputStreamWriter
    ├── FileWriter
    ├── BufferedWriter     ← adds buffering
    └── PrintWriter        ← adds print/println/printf
```

## Real Example

```java title:example.java
import java.util.*;
import java.io.*;

public class StudentRecordSystem {

    // ─── Fast I/O setup ───────────────────────────────────
    static BufferedReader br = new BufferedReader(
                                new InputStreamReader(System.in));
    static StringBuilder output = new StringBuilder();

    public static void main(String[] args) throws IOException {

        System.out.println("=== Student Record System ===");
        System.out.print("Number of students: ");
        int n = Integer.parseInt(br.readLine().trim());

        String[] names  = new String[n];
        int[][]  scores = new int[n][3];  // 3 subjects
        double[] avg    = new double[n];

        // ─── Input using BufferedReader ───────────────────
        for (int i = 0; i < n; i++) {
            System.out.printf("Student %d name: ", i + 1);
            names[i] = br.readLine().trim();

            System.out.printf("Enter 3 scores for %s: ", names[i]);
            StringTokenizer st = new StringTokenizer(br.readLine());
            int sum = 0;
            for (int j = 0; j < 3; j++) {
                scores[i][j] = Integer.parseInt(st.nextToken());
                sum += scores[i][j];
            }
            avg[i] = sum / 3.0;
        }

        // ─── Output using printf ──────────────────────────
        System.out.println("\n" + "─".repeat(55));
        System.out.printf("%-15s %8s %8s %8s %8s%n",
                          "Name", "Score1", "Score2", "Score3", "Avg");
        System.out.println("─".repeat(55));

        for (int i = 0; i < n; i++) {
            System.out.printf("%-15s %8d %8d %8d %8.2f%n",
                names[i],
                scores[i][0], scores[i][1], scores[i][2],
                avg[i]);
        }

        System.out.println("─".repeat(55));

        // ─── Stats ────────────────────────────────────────
        double classAvg = Arrays.stream(avg).average().orElse(0);
        int topIdx = 0;
        for (int i = 1; i < n; i++) {
            if (avg[i] > avg[topIdx]) topIdx = i;
        }

        System.out.printf("Class Average : %.2f%n", classAvg);
        System.out.printf("Top Student   : %s (%.2f)%n",
                           names[topIdx], avg[topIdx]);

        // ─── Write results to file ─────────────────────────
        try (PrintWriter pw = new PrintWriter(
                              new FileWriter("results.txt"))) {
            pw.println("=== Student Results ===");
            for (int i = 0; i < n; i++) {
                pw.printf("%-15s → %.2f%n", names[i], avg[i]);
            }
            pw.printf("Class Average: %.2f%n", classAvg);
            System.out.println("\nResults saved to results.txt ✅");
        }

        br.close();
    }
}

// Sample run:
// === Student Record System ===
// Number of students: 3
// Student 1 name: Alice
// Enter 3 scores for Alice: 90 85 92
// Student 2 name: Bob
// Enter 3 scores for Bob: 78 82 80
// Student 3 name: Charlie
// Enter 3 scores for Charlie: 95 90 98
// ───────────────────────────────────────────────────────
// Name            Score1   Score2   Score3      Avg
// ───────────────────────────────────────────────────────
// Alice               90       85       92    89.00
// Bob                 78       82       80    80.00
// Charlie             95       90       98    94.33
// ───────────────────────────────────────────────────────
// Class Average : 87.78
// Top Student   : Charlie (94.33)
// Results saved to results.txt
```