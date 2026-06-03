# Wrapper Classes
A [wrapper class](https://www.geeksforgeeks.org/java/wrapper-classes-java/) in Java is a special class that "wraps" or encapsulates a primitive data type into an object.

While primitive types (like `int` or `char`) are efficient for performance, Java is an object-oriented language where many advanced features, such as [[Java Collections Framework|Collections]] and [[Generics]], only work with objects.

```
Primitive    →    Wrapper Class
─────────────────────────────────
byte         →    Byte
short        →    Short
int          →    Integer
long         →    Long
float        →    Float
double       →    Double
char         →    Character
boolean      →    Boolean
```

#### Why Do They Exist?
Primitive types (int, double, char) are NOT objects. Java sometimes REQUIRES objects (Collections, Generics, etc.)
`Solution → Wrapper Classes convert primitives INTO objects`

#### Basic Usage
```java title:wrapper_objects.java
// Creating Wrapper Objects
Integer a = Integer.valueOf(42);    // ✅ Preferred way
Integer b = 42;                     // ✅ Autoboxing (compiler does it for you)

// Getting primitive back
int x = a.intValue();               // Manual unboxing
int y = a;                          // Auto-unboxing (compiler does it)

System.out.println(a);              // 42
System.out.println(x);              // 42
```

#### Autoboxing & Unboxing
```java title:boxing.java
// AUTOBOXING → primitive automatically becomes object
Integer num = 100;       // compiler converts to: Integer.valueOf(100)
Double  d   = 3.14;      // compiler converts to: Double.valueOf(3.14)

// UNBOXING → object automatically becomes primitive
int result = num;        // compiler converts to: num.intValue()
double val = d;          // compiler converts to: d.doubleValue()

// Happens automatically in operations too
Integer x = 10;
Integer y = 20;
int sum = x + y;         // both unboxed, then added → 30
```

#### Where You ARE FORCED to Use Wrapper Classes?
```java title:wrapper_class.java
// ❌ This will NOT compile - primitives not allowed in generics
List<int> numbers = new ArrayList<int>();

// Must use Wrapper
List<Integer> numbers = new ArrayList<Integer>();
List<Double>  prices  = new ArrayList<Double>();

// Works perfectly with autoboxing
numbers.add(1);    // auto-boxed to Integer.valueOf(1)
numbers.add(2);
numbers.add(3);

int first = numbers.get(0);  // auto-unboxed
```

#### Useful Utility Methods
```java title:utilities.java
// ─── Integer Utilities ───────────────────────────────

// String → int conversion
int a = Integer.parseInt("123");        // 123
int b = Integer.parseInt("FF", 16);     // 255 (hex to decimal)

// int → String
String s = Integer.toString(42);        // "42"
String s2 = String.valueOf(42);         // "42"

// Min / Max values
System.out.println(Integer.MAX_VALUE);  // 2147483647
System.out.println(Integer.MIN_VALUE);  // -2147483648

// Compare
int result = Integer.compare(10, 20);   // -1 (10 < 20)

// Binary, Hex, Octal
System.out.println(Integer.toBinaryString(10)); // "1010"
System.out.println(Integer.toHexString(255));   // "ff"
System.out.println(Integer.toOctalString(8));   // "10"

// ─── Double Utilities ────────────────────────────────

double d = Double.parseDouble("3.14");  // String → double
System.out.println(Double.MAX_VALUE);   // 1.7976931348623157E308
System.out.println(Double.isNaN(0.0 / 0.0));      // true
System.out.println(Double.isInfinite(1.0 / 0.0)); // true

// ─── Character Utilities ─────────────────────────────

char c = 'A';
System.out.println(Character.isLetter(c));     // true
System.out.println(Character.isDigit(c));      // false
System.out.println(Character.isWhitespace(' ')); // true
System.out.println(Character.toUpperCase('a')); // A
System.out.println(Character.toLowerCase('A')); // a
System.out.println(Character.isLetterOrDigit('3')); // true
```

**Note:** Always use `.equals()` to compare Wrapper objects. Never use `==` for value comparison.

#### Exceptions

##### `==` vs. `.equals()`
```java title:equal.java
// TRAP: Integer caches values from -128 to 127
Integer a = 127;
Integer b = 127;
System.out.println(a == b);       // ✅ TRUE  (same cached object)
System.out.println(a.equals(b));  // ✅ TRUE

Integer x = 128;
Integer y = 128;
System.out.println(x == y);       // ❌ FALSE! (different objects)
System.out.println(x.equals(y));  // ✅ TRUE
```

**RULE:** Always use `.equals()` to compare Wrapper objects. Never use `==` for value comparison.

##### NullPointerException
```java title:nullpointer.java
// Wrapper can be null, primitive CANNOT
Integer num = null;

// This will CRASH with NullPointerException!
int x = num;              // tries to unbox null → 💥 NPE

// Always check null before unboxing
if (num != null) {
    int x = num;          // safe
}

// Real world example - dangerous!
public int add(Integer a, Integer b) {
    return a + b;    // if a or b is null → NPE at runtime
}
```

#### Examples
```java title:example.java
// ─── Sorting with Wrapper ────────────────────────────
List<Integer> nums = new ArrayList<>(Arrays.asList(5, 2, 8, 1, 9));
Collections.sort(nums);
System.out.println(nums);  // [1, 2, 5, 8, 9]

// ─── HashMap requires objects ────────────────────────
HashMap<String, Integer> scores = new HashMap<>();
scores.put("Alice", 95);    // autoboxed
scores.put("Bob", 87);

int aliceScore = scores.get("Alice");  // auto-unboxed

// ─── Stream with Wrapper ─────────────────────────────
List<Integer> numbers = List.of(1, 2, 3, 4, 5);
int sum = numbers.stream()
                 .mapToInt(Integer::intValue)  // unbox for primitive stream
                 .sum();
System.out.println(sum);  // 15
```

**Performance Note:**
```java title:performance.java
// ❌ BAD - autoboxing in loop = slow, creates many objects
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i;    // unbox sum, add i, rebox result → 1M objects created!
}

// ✅ GOOD - use primitives when performance matters
long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i;    // pure primitive math → fast
}
```

#### Summary
```
┌─────────────────────────────────────────────────────┐
│ USE primitives when → just storing/calculating      │
│ USE wrappers when → Collections, Generics, null     │
│                                                     │
│ Autoboxing → primitive to object (auto)             │
│ Unboxing → object to primitive (auto)               │
│                                                     │
│ ALWAYS use .equals() not == for wrappers            │
│ WATCH OUT for NullPointerException on unboxing      │ └─────────────────────────────────────────────────────┘
```