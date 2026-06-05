**In enterprise backend systems, you will almost never use `Scanner` to read from `System.in`.**
Backend applications run on cloud servers (like **AWS** or **Azure**) inside containers. There is no user sitting at a keyboard to type into a console. Instead, inputs arrive as HTTP requests (JSON payloads) or messages from queues (like **Kafka**).

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

# What is a `Scanner`?
The **`Scanner` class** in Java is a built-in utility found in the `java.util` package that is used to **read and parse text input** from various sources like the keyboard, files, or strings.
Scanner is a class in java.util package.
```
It reads input from:
├── Keyboard (System.in)  ← most common
├── File
├── String
└── Network stream

You must IMPORT it before using.
```

**Syntax:**
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

##### Reading Different Data Types
```java title:scan.java
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        // ─── Reading primitives ───────────────────────────────
        System.out.print("Enter integer: ");
        int age = sc.nextInt();

        System.out.print("Enter double: ");
        double price = sc.nextDouble();

        System.out.print("Enter long: ");
        long bigNum = sc.nextLong();

        System.out.print("Enter float: ");
        float f = sc.nextFloat();

        System.out.print("Enter boolean: ");
        boolean b = sc.nextBoolean();   // accepts "true" or "false"

        // ─── Reading Strings ──────────────────────────────────
        System.out.print("Enter a word: ");
        String word = sc.next();        // reads ONE word (stops at space)

        System.out.print("Enter a sentence: ");
        sc.nextLine();                  // ⚠️ flush leftover newline (explained below)
        String sentence = sc.nextLine(); // reads FULL line including spaces

        System.out.println("Age: "      + age);
        System.out.println("Price: "    + price);
        System.out.println("Word: "     + word);
        System.out.println("Sentence: " + sentence);

        sc.close();
    }
}
```

# The `nextLine()`
```java title:nextLine.java
Scanner sc = new Scanner(System.in);

// User types: 25 and hits ENTER
// Buffer contains: "25\n"
int age = sc.nextInt();
// nextInt() reads "25" but leaves "\n" in buffer

// ❌ PROBLEM
String name = sc.nextLine();
// nextLine() reads the leftover "\n" immediately
// name = "" (empty string!) user never got to type

// ✅ SOLUTION → consume the leftover newline first
int age = sc.nextInt();
sc.nextLine();           // flush the leftover \n
String name = sc.nextLine();  // NOW it waits for user input

// Example:
System.out.print("Age: ");
int age = sc.nextInt();
sc.nextLine();                        // flush

System.out.print("Full Name: ");
String name = sc.nextLine();          // works correctly ✅

System.out.println(age + " " + name);
```

# Reading Multiple Values
```java title:scanner.java
Scanner sc = new Scanner(System.in);

// ─── On separate lines ────────────────────────────────────
System.out.println("Enter 3 numbers:");
int a = sc.nextInt();   // user types: 10 ENTER
int b = sc.nextInt();   // user types: 20 ENTER
int c = sc.nextInt();   // user types: 30 ENTER
System.out.println(a + b + c);  // 60

// ─── On same line (space separated) ──────────────────────
// User types: 10 20 30  (all on one line)
int x = sc.nextInt();  // reads 10
int y = sc.nextInt();  // reads 20
int z = sc.nextInt();  // reads 30
// Scanner handles spaces automatically ✅

// ─── Read into array ──────────────────────────────────────
System.out.print("How many numbers? ");
int n = sc.nextInt();
int[] arr = new int[n];

for (int i = 0; i < n; i++) {
    arr[i] = sc.nextInt();
}

// print array
for (int num : arr) {
    System.out.print(num + " ");
}
```

# Reading Until End of Input
```java title:scanner.java
Scanner sc = new Scanner(System.in);

// ─── hasNext() → check if more tokens exist ───────────────
while (sc.hasNextInt()) {
    int num = sc.nextInt();
    System.out.println("Read: " + num);
}

// ─── hasNextLine() → check if more lines exist ────────────
while (sc.hasNextLine()) {
    String line = sc.nextLine();
    System.out.println("Line: " + line);
}

// Common in competitive programming:
// reads until no more input (EOF)
Scanner sc = new Scanner(System.in);
int sum = 0;
while (sc.hasNextInt()) {
    sum += sc.nextInt();
}
System.out.println("Sum: " + sum);
```

# Scanner from String (Useful for Parsing)

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

# Scanner from File
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

# Input Validation


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

# Complete Example
```java title:example.java
import java.util.Scanner;

public class StudentGradeCalculator {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);

        System.out.print("Enter student name: ");
        String name = sc.nextLine();

        System.out.print("How many subjects? ");
        int count = sc.nextInt();

        double total = 0;
        for (int i = 1; i <= count; i++) {
            System.out.print("Enter score " + i + ": ");
            total += sc.nextDouble();
        }

        double average = total / count;
        String grade = (average >= 90) ? "A" :
                       (average >= 80) ? "B" :
                       (average >= 70) ? "C" :
                       (average >= 60) ? "D" : "F";

        System.out.println("─────────────────────");
        System.out.println("Student : " + name);
        System.out.println("Average : " + average);
        System.out.println("Grade   : " + grade);

        sc.close();
    }
}

// Sample run:
// Enter student name: Alice
// How many subjects? 3
// Enter score 1: 85
// Enter score 2: 90
// Enter score 3: 78
// ─────────────────────
// Student : Alice
// Average : 84.33333333333333
// Grade   : B
```