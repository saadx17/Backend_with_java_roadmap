**Big O notation** is a mathematical tool used in computer science to describe how the execution time or memory storage (space) of an algorithm grows as the size of the input dataset increases. It focuses on the **worst-case scenario**, providing an upper bound on performance trends rather than measuring exact clock speeds or milliseconds, making it completely independent of machine hardware.

## THE BIG O HIERARCHY
The **Big O Hierarchy** ranks the efficiency of algorithms based on how their execution time or space requirements scale relative to an input size (n). Understanding this hierarchy allows developers using Java to choose the correct collection from the **Java Collections Framework** to ensure application scalability.
```
FASTEST                                            SLOWEST
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
```

#### 1. Constant Time: `O(1)`
Does NOT depend on input size n at all. No matter how big n is, same number of operations.

- **Java Collections:** HashMap (`get`/`put`), HashSet (`add`/`contains`), and index-based lookups in ArrayList.

```java title:constant_time.java
int getFirstElement(int[] array) {
    return array[0]; // Constant time lookup
}
```

#### 2. Logarithmic Time: `O(log n)`

The problem size cuts in half with every single step, making it highly efficient for massive datasets.

- **Java Collections:** TreeMap and TreeSet operations (`containsKey`, `add`, `remove`), and PriorityQueue insertion/deletion.

```java title:logarithmic.java
// Java's built-in binary search cuts the search space in half each iteration
int index = Collections.binarySearch(sortedArrayList, targetKey); 
```

#### 3. Linear Time: `O(n)`
Performance drops in direct, a 1-to-1 proportion to the size of the input data.

- **Java Collections:** Searching an unsorted ArrayList or LinkedList (`contains`), or inserting/deleting elements in the middle of an `ArrayList`.

```java title:linear_time.java
boolean findValue(List<Integer> list, int target) {
    for (int num : list) { // Must check every element in worst case
        if (num == target) return true;
    }
    return false;
}
```

#### 4. Linearithmic / Quasilinear Time: `O(n log n)`
This complexity usually represents an `O(log n)` operation performed `O(n)` times.

- **Java Application:** Core sorting mechanisms like `Arrays.sort()` and `Collections.sort()`, which rely on algorithms like Dual-Pivot Quicksort, Timsort, or Mergesort.

```java title:linearithmic.java
List<String> names = new ArrayList<>(List.of("Zack", "Anna", "Ben"));
Collections.sort(names); // Runs in O(n log n) time
```

#### 5. Quadratic Time: `O(n^2)`

The execution time squares relative to the input, becoming highly inefficient for larger apps.

- **Java Frameworks:** Naive nested iterations or sorting algorithms like Bubble Sort or Selection Sort acting on standard arrays.

```java title:quadratic_time.java
void printAllPairs(int[] array) {
    for (int i : array) {         // Runs n times
        for (int j : array) {     // Runs n times per outer iteration
            System.out.println(i + ", " + j);
        }
    }
}
```

#### 6. Exponential Time: `O(2^n)` ⚠️
With exponential growth, the number of operations **doubles** with every single addition to the input size (n).

**Warning:** fibonacci(50) will take days to finish execution!
```java title:exponential_time.java
public class ExponentialTime {
    // Naive recursive Fibonacci algorithm
    public static int fibonacci(int n) {
        if (n <= 1) {
            return n;
        }
        // O(2^n) - Each call spawns two brand new recursive branches
        return fibonacci(n - 1) + fibonacci(n - 2);
    }

    public static void main(String[] args) {
        System.out.println(fibonacci(10)); 
    }
}
```

#### 7. Factorial Time: `O(n!)` ⚠️
This is the slowest, least efficient complexity tier. The algorithm multiplies the number of operations by the new input size at every step `(n * (n-1) * (n-2)...)`

```java title:factorial_time.java
import java.util.ArrayList;
import java.util.List;

public class FactorialTime {
    // Generates all unique ordering permutations of a list
    public static void permute(List<Integer> arr, int k) {
        for (int i = k; i < arr.size(); i++) {
            java.util.Collections.swap(arr, i, k);
            permute(arr, k + 1); // O(n!) - Recursively forks for all n! combinations
            java.util.Collections.swap(arr, i, k);
        }
        if (k == arr.size() - 1) {
            System.out.println(arr);
        }
    }

    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
        // Prints 3! (6) combinations. A list of 20 items would freeze the computer forever.
        permute(list, 0); 
    }
}
```