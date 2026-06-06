# What is an Array?
An **array in Java** is a dynamically allocated object that stores a **fixed number of elements of the same data type** in contiguous memory locations.

```
Key facts:
├── Fixed size (can't grow or shrink after creation)
├── Index starts at 0
├── Same data type only
├── Objects in Java (stored in heap)
└── .length property (not a method!)
```

In Java, arrays are categorized into two primary types based on their dimensions: 
1. [[#1D Arrays|Single-Dimensional Arrays]]
2. [[#2D Arrays|Multi-Dimensional Arrays]]

# 1D Arrays
A **one-dimensional (1D) array** in Java is a linear data structure that stores a **fixed-size collection of elements** of the **same data type** in contiguous memory locations.

### Declaration & Creation
1D array can be declared by 3 ways.
```java title:way-one.java
// Way 1: declare then create
int[] arr;              // declaration (no memory yet)
arr = new int[5];       // creation (allocates memory)
```

```java title:way-two.java
// Way 2: declare and create together
int[] arr = new int[5]; // 5 elements, all initialized to 0
```

```java title:way-three.java
// Way 3: declare with values (array literal)
int[] arr = {10, 20, 30, 40, 50};  // size auto-determined
```

```java title:default_values.java
// Default values after new int[n]
int[]     nums    = new int[3];     // [0, 0, 0]
double[]  doubles = new double[3];  // [0.0, 0.0, 0.0]
boolean[] bools   = new boolean[3]; // [false, false, false]
String[]  strings = new String[3];  // [null, null, null]
char[]    chars   = new char[3];    // ['\u0000','\u0000','\u0000']
```

```java title:common_mistakes.java
// Common mistake
int[] arr;
System.out.println(arr[0]);  // compile error, not initialized yet

int[] arr = new int[5];
System.out.println(arr[0]);  // 0 (default value)
```

### Accessing & Modifying Elements
To access and modify a 1D array in Java, you use **square brackets `[]` combined with a zero-based index**. The first element of the array sits at index `0`, and the last element sits at index `array.length - 1`.

```java title:syntax.java
int[] arr = {10, 20, 30, 40, 50};
//  index:   0   1   2   3   4
```

```java title:access.java
// Access
System.out.println(arr[0]);   // 10  (first)
System.out.println(arr[4]);   // 50  (last)
System.out.println(arr[arr.length - 1]); // 50 (last, dynamic)

// 2. Accessing elements
int firstArr = arr[0];   // Accesses 10
int thirdArr = arr[2];   // Accesses 30
```

```java title:modify.java
// Modify
arr[0] = 100;
arr[2] = 999 + 1;
// arr is now {100, 20, 1000, 40, 50}
```

```java title:out-of-bounds.java
// Out of bounds → exception
System.out.println(arr[5]);   // ArrayIndexOutOfBoundsException
System.out.println(arr[-1]);  // ArrayIndexOutOfBoundsException
```

```java title:length.java
// Length
System.out.println(arr.length);    // 5  ← property, no ()
System.out.println(arr.length()); // compile error!
```

### Array Iteration
To iterate through a one-dimensional (1D) array in Java, you can use a **standard `for` loop, an enhanced `for` loop (for-each), a `while` loop, or Java Streams**.
```java title:syntax.java
int[] arr = {10, 20, 30, 40, 50};
```

```java title:loop.java
// for loop (use when you need index)
for (int i = 0; i < arr.length; i++) {
    System.out.println("arr[" + i + "] = " + arr[i]);
}
// arr[0] = 10
// arr[1] = 20 ...
```

```java title:for-each.java
// for-each (cleanest, no index needed)
for (int num : arr) {
    System.out.print(num + " ");   // 10 20 30 40 50
}
```

```java title:while.java
// while loop
int i = 0;
while (i < arr.length) {
    System.out.print(arr[i] + " ");
    i++;
}
```

```java title:reverse.java
// Reverse iteration
for (int j = arr.length - 1; j >= 0; j--) {
    System.out.print(arr[j] + " "); // 50 40 30 20 10
}
```

```java title:array-stream.java
// Arrays.stream (modern)
import java.util.Arrays;
Arrays.stream(arr).forEach(n -> System.out.print(n + " "));
```

#### Common Array Operations (Manual)
**Manual array operations in Java** require iterating through elements using loops and logic rather than relying on built-in utilities like `java.util.Arrays`. Because Java arrays have a **fixed size**, operations like **insertion and deletion** require creating a new array and shifting elements manually.

```java title:syntax.java
int[] arr = {5, 2, 8, 1, 9, 3};
```

```java title:sum-avg.java
// Sum & Average
int sum = 0;
for (int n : arr) {
    sum += n;
}
double avg = (double) sum / arr.length;
System.out.println("Sum: " + sum);        // 28
System.out.println("Average: " + avg);    // 4.666...
```

```java title:min_max.java
// Min & Max
int min = arr[0];
int max = arr[0];
for (int n : arr) {
    if (n < min) min = n;
    if (n > max) max = n;
}
System.out.println("Min: " + min);  // 1
System.out.println("Max: " + max);  // 9
```

```java title:linear_search.java
// Linear Search
int target = 8;
int foundAt = -1;
for (int j = 0; j < arr.length; j++) {
    if (arr[j] == target) {
        foundAt = j;
        break;
    }
}
System.out.println(foundAt != -1 ? "Found at " + foundAt : "Not found");
```

```java title:reverse_array.java
// Reverse array
int left = 0, right = arr.length - 1;
while (left < right) {
    int temp  = arr[left];
    arr[left]  = arr[right];
    arr[right] = temp;
    left++;
    right--;
}
System.out.println(Arrays.toString(arr)); // [3, 9, 1, 8, 2, 5]
```

```java title:count_occur.java
// Count occurrences
int[] data = {1, 2, 3, 2, 1, 2, 4};
int countOf2 = 0;
for (int n : data) {
    if (n == 2) countOf2++;
}
System.out.println("Count of 2: " + countOf2);  // 3
```

### Copying Arrays
```java title:syntax.java
int[] original = {1, 2, 3, 4, 5};
```

```java title:wrong-way.java
// Wrong way (reference copy)
int[] wrong = original;     // both point to SAME array!
wrong[0] = 999;
System.out.println(original[0]);  // 999 ← original changed!
```

```java title:manual_copy.java
// Way 1: manual copy
int[] copy1 = new int[original.length];
for (int i = 0; i < original.length; i++) {
    copy1[i] = original[i];
}
```

```java title:array_copy.java
// Way 2: Arrays.copyOf
int[] copy2 = Arrays.copyOf(original, original.length);
```

```java title:array-copy-range.java
// Way 3: Arrays.copyOfRange
int[] copy3 = Arrays.copyOfRange(original, 1, 4); // index 1 to 3
System.out.println(Arrays.toString(copy3)); // [2, 3, 4]
```

```java title:system-array_copy.java
// Way 4: System.arraycopy
int[] copy4 = new int[original.length];
System.arraycopy(original, 0, copy4, 0, original.length);
//              (src, srcPos, dest, destPos, length)
```

# 2D Arrays
