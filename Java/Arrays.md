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
A **2D array in Java** is essentially an array of arrays that stores data in a grid structure composed of rows and columns. It is widely used to build mathematical matrices, game boards (like chess or tic-tac-toe), and data spreadsheets.

### Declaration & Creation
You can create a 2D array in Java using two different approaches: **dynamic initialization** (setting the size first) or **static initialization** (setting the values immediately).

2D array = array of arrays. Think of it as a TABLE (rows × columns).

```java title:size.java
// Create with size
int[][] matrix = new int[3][4];  // 3 rows, 4 columns
//  all initialized to 0
```

```java title:values.java
// Create with values
int[][] matrix = {
    {1, 2, 3},     // row 0
    {4, 5, 6},     // row 1
    {7, 8, 9}      // row 2
};

// Visual representation
//        col0  col1  col2
// row0  [  1    2    3  ]
// row1  [  4    5    6  ]
// row2  [  7    8    9  ]
```

```java title:access_elements.java
// Access elements
System.out.println(matrix[0][0]);  // 1  (row 0, col 0)
System.out.println(matrix[1][1]);  // 5  (row 1, col 1)
System.out.println(matrix[2][2]);  // 9  (row 2, col 2)
```

```java title:dimensions.java
// Dimensions
System.out.println(matrix.length);     // 3 (rows)
System.out.println(matrix[0].length);  // 3 (columns in row 0)
```

### 2D Array Iteration
The most common way to iterate through a 2D array in Java is by using **nested for loops**. Because a 2D array is an "array of arrays" in Java, you use an outer loop to cycle through the rows and an inner loop to cycle through the elements (columns) of each row.

```java title:syntax.java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};
```

```java title:nested_for.java
// Nested for loop
for (int i = 0; i < matrix.length; i++) {          // rows
    for (int j = 0; j < matrix[i].length; j++) {   // columns
        System.out.print(matrix[i][j] + "\t");
    }
    System.out.println();
}
// 1  2  3
// 4  5  6
// 7  8  9
```

```java title:nested_for-each.java
// Nested for-each
for (int[] row : matrix) {
    for (int val : row) {
        System.out.print(val + "\t");
    }
    System.out.println();
}
```

```java title:print_as_grid.java
// Print as grid
for (int[] row : matrix) {
    System.out.println(Arrays.toString(row));
}
// [1, 2, 3]
// [4, 5, 6]
// [7, 8, 9]
```

### 2D Array Operations
In Java, a **two-dimensional (2D) array** is an "array of arrays" structured in a grid format with rows and columns.

```java title:syntax.java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};
```

```java title:sum_of_all.java
// Sum of all elements
int total = 0;
for (int[] row : matrix) {
    for (int val : row) {
        total += val;
    }
}
System.out.println("Total: " + total);  // 45
```

```java title:sum_of_each.java
// Sum of each row
for (int i = 0; i < matrix.length; i++) {
    int rowSum = 0;
    for (int val : matrix[i]) rowSum += val;
    System.out.println("Row " + i + " sum: " + rowSum);
}
// Row 0 sum: 6
// Row 1 sum: 15
// Row 2 sum: 24
```

```java title:diagonal_sum.java
// Diagonal sum (only for square matrix)
int diagSum = 0;
for (int i = 0; i < matrix.length; i++) {
    diagSum += matrix[i][i];   // [0][0], [1][1], [2][2]
}
System.out.println("Diagonal: " + diagSum);  // 15 (1+5+9)
```

```java title:transpose.java
// Transpose (rows become columns)
int rows = matrix.length;
int cols = matrix[0].length;
int[][] transposed = new int[cols][rows];

for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++) {
        transposed[j][i] = matrix[i][j];
    }
}
// original:    transposed:
// 1 2 3        1 4 7
// 4 5 6        2 5 8
// 7 8 9        3 6 9
```

```java title:matrix_multipic.java
// Matrix multiplication
int[][] A = {{1, 2}, {3, 4}};
int[][] B = {{5, 6}, {7, 8}};
int[][] C = new int[2][2];

for (int i = 0; i < 2; i++) {
    for (int j = 0; j < 2; j++) {
        for (int k = 0; k < 2; k++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
// C = {{19, 22}, {43, 50}}
```

### Jagged Arrays (Irregular 2D)
A **jagged array** (or ragged array) in Java is an **array of arrays where each row can have a different length**. Unlike standard 2D arrays that form a rigid, rectangular grid, jagged arrays allow you to allocate only the exact amount of memory needed for each row.

```java title:syntax.java
// Rows can have DIFFERENT lengths

int[][] jagged = new int[3][];  // 3 rows, columns not set
jagged[0] = new int[2];         // row 0 has 2 columns
jagged[1] = new int[4];         // row 1 has 4 columns
jagged[2] = new int[3];         // row 2 has 3 columns
```

```java title:values.java
// With values
int[][] triangle = {
    {1},
    {2, 3},
    {4, 5, 6},
    {7, 8, 9, 10}
};

for (int[] row : triangle) {
    for (int val : row) {
        System.out.print(val + " ");
    }
    System.out.println();
}
// 1
// 2 3
// 4 5 6
// 7 8 9 10
```

```java title:pascal.java
// Use case: Pascal's triangle
int n = 5;
int[][] pascal = new int[n][];

for (int i = 0; i < n; i++) {
    pascal[i] = new int[i + 1];
    pascal[i][0] = 1;
    pascal[i][i] = 1;
    for (int j = 1; j < i; j++) {
        pascal[i][j] = pascal[i-1][j-1] + pascal[i-1][j];
    }
}

for (int[] row : pascal) {
    System.out.println(Arrays.toString(row));
}
// [1]
// [1, 1]
// [1, 2, 1]
// [1, 3, 3, 1]
// [1, 4, 6, 4, 1]
```

# Arrays Utility Class
The primary array utility class in Java is **[`java.util.Arrays`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Arrays.html)**. It belongs to the Java Collections Framework and contains exclusively `static` methods to manipulate, format, search, and copy arrays without manual loops.

```java title:import_class.java
import java.util.Arrays;
// java.util.Arrays has static methods for common operations
```

### Sorting
To sort an array in Java, the easiest and most efficient way is to use the built-in **`Arrays.sort()`** method from the **`java.util.Arrays`** class.

```java title:sorting.java
int[] arr = {5, 2, 8, 1, 9, 3};

// Sort ascending (default)
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));  // [1, 2, 3, 5, 8, 9]

// Sort range only
int[] arr2 = {5, 2, 8, 1, 9, 3};
Arrays.sort(arr2, 1, 4);               // sort index 1 to 3 only
System.out.println(Arrays.toString(arr2)); // [5, 1, 2, 8, 9, 3]

// Sort descending (needs Integer[] not int[])
Integer[] arr3 = {5, 2, 8, 1, 9, 3};
Arrays.sort(arr3, (a, b) -> b - a);   // reverse comparator
System.out.println(Arrays.toString(arr3)); // [9, 8, 5, 3, 2, 1]

// Sort 2D array by first column
int[][] data = {{3, 1}, {1, 4}, {2, 2}};
Arrays.sort(data, (a, b) -> a[0] - b[0]);
// {{1,4}, {2,2}, {3,1}}
```

### Searching
In Java, you can search an array using four primary approaches: **`Arrays.binarySearch()`** for sorted arrays, a standard **for loop** for manual linear search, **Java Streams** for a modern functional style, or converting the array into a **`List`** to check for membership.

```java title:searching.java
// Binary search → array MUST be sorted first!
int[] arr = {1, 2, 3, 5, 8, 9};

int idx = Arrays.binarySearch(arr, 5);
System.out.println(idx);   // 3 (found at index 3)

int missing = Arrays.binarySearch(arr, 7);
System.out.println(missing);  // negative → not found
// negative value = -(insertion point) - 1

// Search in range
int idx2 = Arrays.binarySearch(arr, 0, 4, 3); // search index 0-3
System.out.println(idx2);  // 2
```

### Filling & Comparing
In Java, **filling and comparing arrays** is best handled using the built-in utility methods from the **`java.util.Arrays`** class. Avoid using the `==` operator for comparison, as it only checks if two arrays share the exact same memory location rather than checking their actual elements.

```java title:fill.java
// Fill
int[] arr = new int[5];
Arrays.fill(arr, 7);
System.out.println(Arrays.toString(arr));  // [7, 7, 7, 7, 7]

// Fill range only
Arrays.fill(arr, 1, 4, 99);               // fill index 1-3 with 99
System.out.println(Arrays.toString(arr));  // [7, 99, 99, 99, 7]
```

```java title:compare.java
// Equals (compare contents)
int[] a = {1, 2, 3};
int[] b = {1, 2, 3};
int[] c = {1, 2, 4};

System.out.println(a == b);              // false (different objects)
System.out.println(Arrays.equals(a, b)); // true  (same content)
System.out.println(Arrays.equals(a, c)); // false

// Deep equals (for 2D arrays)
int[][] m1 = {{1, 2}, {3, 4}};
int[][] m2 = {{1, 2}, {3, 4}};

System.out.println(Arrays.equals(m1, m2));      // false (shallow)
System.out.println(Arrays.deepEquals(m1, m2));  // true 
```

### toString & Printing
To print a human-readable array in Java, you must use **`Arrays.toString()`** for 1D arrays or **`Arrays.deepToString()`** for multidimensional arrays.

Calling `.toString()` directly on a Java array object (like `myArray.toString()`) or trying to print it directly via `System.out.println(myArray)` will only print its **memory address / class hashcode** (e.g., `[I@15db9742`). This happens because Java arrays inherit the default fallback `toString()` implementation from the base `Object` class instead of overriding it.

```java title:toString.java
int[]     arr1 = {1, 2, 3, 4, 5};
int[][]   arr2 = {{1, 2}, {3, 4}};
int[][][] arr3 = {{{1, 2}, {3, 4}}, {{5, 6}, {7, 8}}};

// 1D array
System.out.println(arr1);               // [I@1b6d3586 ← useless!
System.out.println(Arrays.toString(arr1));     // [1, 2, 3, 4, 5]

// 2D array
System.out.println(Arrays.toString(arr2));     // [[I@...] ← useless!
System.out.println(Arrays.deepToString(arr2)); // [[1, 2], [3, 4]]

// 3D array
System.out.println(Arrays.deepToString(arr3)); // [[[1, 2], [3, 4]], [[5, 6], [7, 8]]]
```
**NOTE:** A **3D array in Java** is essentially an array of 2D arrays, organized visually like a cube or a data structure with layers, rows, and columns.

### Other Useful Methods
```java title:useful_methods.java
// copyOf
int[] original = {1, 2, 3, 4, 5};

int[] shorter = Arrays.copyOf(original, 3);       // [1, 2, 3]
int[] longer  = Arrays.copyOf(original, 8);       // [1, 2, 3, 4, 5, 0, 0, 0]
int[] range   = Arrays.copyOfRange(original, 1, 4); // [2, 3, 4]

// Stream from array (modern Java)
int[] arr = {1, 2, 3, 4, 5};

int sum = Arrays.stream(arr).sum();
System.out.println(sum);  // 15

int max = Arrays.stream(arr).max().getAsInt();
System.out.println(max);  // 5

int min = Arrays.stream(arr).min().getAsInt();
System.out.println(min);  // 1

double avg = Arrays.stream(arr).average().getAsDouble();
System.out.println(avg);  // 3.0

long count = Arrays.stream(arr).filter(n -> n > 2).count();
System.out.println(count);  // 3
```


# Arrays Class Cheatsheet
```
┌──────────────────────────────┬─────────────────────────┐
│ Method                       │ What it does            │
├──────────────────────────────┼─────────────────────────┤
│ Arrays.sort(arr)             │ sort ascending          │
│ Arrays.sort(arr, from, to)   │ sort range              │
│ Arrays.binarySearch(arr, k)  │ find index of k         │
│ Arrays.fill(arr, val)        │ fill with value         │
│ Arrays.fill(arr, f, t, val)  │ fill range              │
│ Arrays.equals(a, b)          │ compare 1D arrays       │
│ Arrays.deepEquals(a, b)      │ compare 2D arrays       │
│ Arrays.toString(arr)         │ print 1D array          │
│ Arrays.deepToString(arr)     │ print 2D array          │
│ Arrays.copyOf(arr, len)      │ copy with new length    │
│ Arrays.copyOfRange(a, f, t)  │ copy range              │
│ Arrays.stream(arr)           │ convert to stream       │
└──────────────────────────────┴─────────────────────────┘
```