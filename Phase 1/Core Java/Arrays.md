An **array** is a **fixed-size**, **ordered** container holding elements of a **single type**, accessed by **zero-based index**. Arrays are the most fundamental data structure - fast, contiguous, and cache-friendly.

```java
int[] numbers = {10, 20, 30};
numbers[0];        // 10  (first element, index 0)
numbers.length;    // 3   (FIELD, not a method — no parentheses!)
```

**Key facts:**
- **Fixed size:** set at creation, **cannot grow/shrink** (use `ArrayList` for dynamic).
- **Zero-indexed:** valid indices are `0` to `length - 1`.
- Arrays are **objects** → they live on the **heap**.
- `length` is a **field**, not a method (`array.length`, not `array.length()`).

In Java, arrays are categorized into two primary types based on their dimensions: 
1. [[#1. Single-Dimensional Arrays|1D Arrays]]
2. [[#2. Multi-Dimensional Arrays|2D Arrays]]

## 1. Single-Dimensional Arrays
A **one-dimensional (1D) array** in Java is a linear data structure that stores a **fixed-size collection of elements** of the **same data type** in contiguous memory locations.

### 1.1 Declaration & creation
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

### 1.2 Default values (auto-initialized)
A new array's elements get the **type's default** (recall Data Types note):
```java
int[]     ints    = new int[3];     // {0, 0, 0}
double[]  dbls    = new double[3];   // {0.0, 0.0, 0.0}
boolean[] bools   = new boolean[3];  // {false, false, false}
String[]  strings = new String[3];   // {null, null, null}  <- references default to null!
```
> ⚠️ A `String[]` (or any object array) is filled with **`null`**, not empty strings — accessing/using them without assigning throws NPE.

### 1.3 Accessing & Modifying Elements
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

### 1.4 Array Iteration
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

### 1.5 Common Array Operations (Manual)
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


## 2. Multi-Dimensional Arrays
A **2D array in Java** is essentially an array of arrays that stores data in a grid structure composed of rows and columns. It is widely used to build mathematical matrices, game boards (like chess or tic-tac-toe), and data spreadsheets.

### 2.1 Declaration & Creation
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

### 2.2 2D Array Iteration
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

### 2.3 2D Array Operations
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

### 2.4 Jagged Arrays (Irregular 2D)
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


## 3. The `Arrays` Utility Class
The primary array utility class in Java is **[`java.util.Arrays`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Arrays.html)**. It belongs to the Java Collections Framework and contains exclusively `static` methods to manipulate, format, search, and copy arrays without manual loops.

```java title:import_class.java
import java.util.Arrays;
// java.util.Arrays has static methods for common operations
```

`java.util.Arrays` provides static helper methods for common array operations:

| Method                              | Purpose                               |
| ----------------------------------- | ------------------------------------- |
| `Arrays.toString(arr)`              | Readable string of a 1D array         |
| `Arrays.deepToString(arr)`          | Readable string of a multi-D array    |
| `Arrays.sort(arr)`                  | Sort in place (ascending)             |
| `Arrays.binarySearch(arr, key)`     | Binary search (array must be sorted!) |
| `Arrays.fill(arr, val)`             | Fill all elements with a value        |
| `Arrays.equals(a, b)`               | Element-wise equality (1D)            |
| `Arrays.deepEquals(a, b)`           | Element-wise equality (nested)        |
| `Arrays.copyOf(arr, newLen)`        | Copy (truncated/padded to newLen)     |
| `Arrays.copyOfRange(arr, from, to)` | Copy a sub-range                      |
| `Arrays.asList(arr)`                | View as a `List` (fixed-size!)        |
| `Arrays.stream(arr)`                | Create a stream (Phase 1.8)           |

### 3.1 Sorting
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

### 3.2Searching
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

### 3.3 Filling & Comparing
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

### 3.4 toString & Printing
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

### 3.5 Other Useful Methods
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

## 4. Array Copying - Shallow vs Deep
> Copying arrays correctly is a frequent source of bugs and connects to the shallow/deep copy lesson from the Encapsulation note.

### 4.1 Ways to copy
```java
int[] src = {1, 2, 3};

int[] c1 = Arrays.copyOf(src, src.length);        // recommended
int[] c2 = src.clone();                            // also works for 1D
int[] c3 = Arrays.copyOfRange(src, 0, 2);          // {1, 2}
int[] c4 = new int[3];
System.arraycopy(src, 0, c4, 0, 3);                // low-level, fast
```

### 4.2 The reference-copy trap
**Assignment does NOT copy an array**, it copies the reference (both point to the same array):
```java
int[] a = {1, 2, 3};
int[] b = a;          // NOT a copy — same array!
b[0] = 99;
System.out.println(a[0]);   // 99  -> a changed too
```

### 4.3 Shallow vs deep for object arrays
For arrays of **objects** (or 2D arrays), `clone()`/`copyOf` make a **shallow copy** - the new array has the **same element references**:
```java
Person[] people = { new Person("A"), new Person("B") };
Person[] copy = people.clone();      // shallow: copy[0] == people[0] (same Person!)
copy[0].setName("X");                // also changes people[0]'s name

int[][] grid = {{1,2},{3,4}};
int[][] shallow = grid.clone();      // shallow: shallow[0] == grid[0] (shared rows!)
shallow[0][0] = 99;                  // grid[0][0] also becomes 99
```
For a **deep copy**, copy each element/row yourself:
```java
int[][] deep = new int[grid.length][];
for (int i = 0; i < grid.length; i++)
    deep[i] = grid[i].clone();       // copy each row independently
```

| Copy method | 1D primitives | Object/2D arrays |
|-------------|---------------|------------------|
| `clone()` / `copyOf` | True copy | **Shallow** (shared elements) |
| Manual element-by-element | n/a | **Deep** (independent) |

## 5. Common Pitfalls

### 5.1 `ArrayIndexOutOfBoundsException`
Accessing an invalid index throws this runtime exception:
```java
int[] a = {1, 2, 3};
a[3];        // ArrayIndexOutOfBoundsException (valid: 0..2)
a[-1];       // also throws — no negative indices
```
- Valid indices: `0` to `length - 1`.
- **Off-by-one** errors (`i <= length` instead of `i < length`) are the usual cause.

### 5.2 `NegativeArraySizeException`
```java
int[] a = new int[-1];   // NegativeArraySizeException
```

### 5.3 NullPointerException
```java
int[] a = null;
a.length;                // NPE — the array reference is null
String[] s = new String[3];
s[0].length();           // NPE — element is null (not assigned)
```

### 5.4 Pitfalls table
| Pitfall | Fix |
|---------|-----|
| `ArrayIndexOutOfBoundsException` | Use `0..length-1`; prefer for-each |
| Printing an array directly | Use `Arrays.toString` / `deepToString` |
| Assignment ≠ copy | Use `clone()`/`copyOf` to actually copy |
| Shallow copy sharing elements | Deep-copy object/2D arrays manually |
| `Arrays.asList(...).add(...)` fails | Wrap in `new ArrayList<>(...)` |
| Forgetting `String[]` defaults to null | Initialize elements before use |
| Needing a dynamic-size array | Use `ArrayList` / collections (Phase 1.6) |
| `binarySearch` on an unsorted array | Sort first |

## 6. Arrays vs ArrayList (preview)

| Aspect | Array | `ArrayList` |
|--------|-------|-------------|
| Size | Fixed | Dynamic (grows) |
| Type | Primitives or objects | Objects only (boxing for primitives) |
| Syntax | `a[0]`, `a.length` | `list.get(0)`, `list.size()` |
| Performance | Slightly faster, less memory | Flexible, richer API |
| Use when | Fixed size, performance-critical, primitives | Size varies, need List operations |

> Arrays are the foundation; `ArrayList` (backed by an array internally) is what you'll use most in application code (Phase 1.6).

## 7. Connection to Backend / Spring (Why This Matters Later)

- Arrays underpin **collections** (`ArrayList`, `HashMap` buckets) and many algorithms (Phase 1.6, 2).
- **Byte arrays (`byte[]`)** are everywhere in I/O, file upload/download, serialization, hashing, and crypto (Phase 1.9, 5, 15).
- **Cache locality** of arrays explains performance differences vs linked structures (Phase 2, 13).
- **Defensive copying of arrays** protects encapsulation when exposing internal data (Encapsulation note) — return a copy, not the internal array.
- `Arrays.stream(...)` bridges arrays into the **Stream API** (Phase 1.8).
- Varargs are arrays under the hood (Methods note).

## 8. Quick Self-Check Questions

1. What are the key properties of a Java array (size, indexing, where it lives)?
2. Why is `length` written without parentheses?
3. What default values do `int[]`, `boolean[]`, and `String[]` get?
4. How are 2D arrays really represented, and what is a jagged array?
5. Name four useful `Arrays` utility methods.
6. Why does `int[] b = a;` not create a copy? How do you actually copy?
7. What's the difference between a shallow and deep copy for a 2D/object array?
8. What causes `ArrayIndexOutOfBoundsException`, and what are the valid indices?
9. When would you use an array vs an `ArrayList`?

## 9. Key Terms Glossary

- **Array:** fixed-size, indexed container of one element type.
- **Index:** zero-based position of an element.
- **`length`:** the array's size (a field, not a method).
- **Default values:** auto-assigned element values (`0`, `false`, `null`, ...).
- **Multi-dimensional array:** an array of arrays.
- **Jagged array:** a 2D array with rows of differing lengths.
- **`Arrays` class:** `java.util.Arrays` utility methods.
- **Shallow copy:** copies the array but shares element references.
- **Deep copy:** copies the array and its nested elements independently.
- **`System.arraycopy`:** fast low-level array copy.
- **`ArrayIndexOutOfBoundsException`:** invalid index access.
- **`NegativeArraySizeException`:** creating an array with negative size.