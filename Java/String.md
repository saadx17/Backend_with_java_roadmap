Since String is a class, it comes with many **built-in methods**.  
Remember: Strings are **immutable**, so these methods always return a **new String**.

##### ==`length()`==
Number of characters.
```
String name = "Ahmed";
System.out.println(name.length()); // 5

String empty = "";
System.out.println(empty.length()); // 0

String sentence = "Hello World"; System.out.println(sentence.length()); // 11 (space counts!)
```

##### ==`charAt(index)`==
Character at a position. Returns the **char** at the given index (0-based).
```
String name = "Ahmed";
//             01234 ← indexes

System.out.println(name.charAt(0)); // A
System.out.println(name.charAt(2)); // m
System.out.println(name.charAt(4)); // d

// name.charAt(5) → ❌ StringIndexOutOfBoundsException
```

##### ==`substring()`==
Extract part of a String.
Two versions:
- `substring(startIndex)` → from startIndex to end
- `substring(startIndex, endIndex)` → from startIndex up to (not including) endIndex

```
String text = "Hello World";
//             01234567890

System.out.println(text.substring(6)); // "World" (from index 6 to end)

System.out.println(text.substring(0, 5)); // "Hello" (0 to 4, not including 5)

System.out.println(text.substring(6, 11)); // "World" (6 to 10, not including 11)
```

##### ==`indexOf()`==
Find position of a character or String. Returns the **index** of the first occurrence.  Returns **-1** if not found.

```
String text = "Hello World";
System.out.println(text.indexOf('o')); // 4 (first 'o')
System.out.println(text.indexOf('o', 5)); // 7 (search starting from index 5)
System.out.println(text.indexOf("World")); // 6
System.out.println(text.indexOf('z')); // -1 (not found)
System.out.println(text.indexOf("xyz")); // -1 (not found)
```

##### ==`contains()`==
Check if String contains a value. Returns `boolean` true if found, false if not.
```
String email = "ahmed@gmail.com";
System.out.println(email.contains("@")); // true
System.out.println(email.contains("gmail")); // true
System.out.println(email.contains("yahoo")); // false
```

##### ==`replace()`==
Replace characters or substrings.
```
String text = "Hello World";
System.out.println(text.replace('l', 'x')); // "Hexxo Worxd"
System.out.println(text.replace("World", "Java")); // "Hello Java"
System.out.println(text.replace("l", "LL")); // "HeLLLLo WorLLd"

// Original is unchanged!
System.out.println(text); // "Hello World"
```

##### ==`toUpperCase()`== and ==`toLowerCase()`==

```
String name = "Ahmed Ali";
System.out.println(name.toUpperCase()); // "AHMED ALI"
System.out.println(name.toLowerCase()); // "ahmed ali"
// Useful for case-insensitive comparison
String input = "YES";
if (input.toLowerCase().equals("yes")) { System.out.println("User said yes");
}
```

##### ==`trim()`==
Remove leading and trailing whitespace.
```
String input = " hello world ";

System.out.println(input.trim()); // "hello world"
System.out.println(input.trim().length()); // 11

// Very useful when processing user input
String username = " Ahmed ";
username = username.trim(); // Clean it up before using
```

**NOTE:** *Java 11+ also has `strip()` which is Unicode-aware and preferred over `trim()`*

##### ==`split()`==
Split String into an array. Splits by a given delimiter, returns `String[]`.
```
String csv = "Ahmed,Ali,25,Engineer";
String[] parts = csv.split(",");

System.out.println(parts[0]); // "Ahmed"
System.out.println(parts[1]); // "Ali"
System.out.println(parts[2]); // "25"
System.out.println(parts[3]); // "Engineer"
System.out.println(parts.length); // 4

// Split by space
String sentence = "Hello World Java";
String[] words = sentence.split(" ");
// words = ["Hello", "World", "Java"]

// Split by multiple spaces or special regex
String data = "one two three";
String[] items = data.split("\\s+"); // one or more whitespace
```

##### ==`equals()`== and ==`equalsIgnoreCase()`==
String comparison. **Always use `.equals()` to compare String content, never `==`.**

```
String a = "hello";
String b = "hello";
String c = "HELLO";

System.out.println(a.equals(b)); // true
System.out.println(a.equals(c)); // false (case-sensitive)
System.out.println(a.equalsIgnoreCase(c)); // true (ignores case)

// Never use == for String content comparison
System.out.println(a == b); // true (only because of String pool, not reliable)

String x = new String("hello");
System.out.println(a == x); // false (different objects)
System.out.println(a.equals(x)); // true (same content) ← correct way
```

##### Other Useful String Methods.
```
String text = " Hello World ";

// Check if empty
System.out.println("".isEmpty()); // true
System.out.println(" ".isEmpty()); // false (has spaces)
System.out.println(" ".isBlank()); // true (Java 11+ - whitespace only)

// starts/ends with
System.out.println("Hello".startsWith("He")); // true
System.out.println("Hello".endsWith("lo")); // true

// String.valueOf() - convert other types to String
String num = String.valueOf(42); // "42"
String dbl = String.valueOf(3.14); // "3.14"
String bool = String.valueOf(true); // "true"

// String concatenation
String firstName = "Ahmed";
String lastName = "Ali";
String fullName = firstName + " " + lastName; // "Ahmed Ali"

// String formatting (cleaner than concatenation)
String formatted = String.format("Name: %s, Age: %d, GPA: %.2f", "Ahmed", 25, 3.85);
System.out.println(formatted); // Name: Ahmed, Age: 25, GPA: 3.85
```

##### Complete String Methods Reference.

|Method|What it does|Returns|
|---|---|---|
|`length()`|Number of characters|`int`|
|`charAt(i)`|Character at index i|`char`|
|`substring(i)`|From index i to end|`String`|
|`substring(i, j)`|From i to j (exclusive)|`String`|
|`indexOf(x)`|First position of x (-1 if not found)|`int`|
|`contains(x)`|Does it contain x?|`boolean`|
|`replace(a, b)`|Replace a with b|`String`|
|`toUpperCase()`|All uppercase|`String`|
|`toLowerCase()`|All lowercase|`String`|
|`trim()`|Remove leading/trailing spaces|`String`|
|`strip()`|Like trim() but Unicode-aware (Java 11+)|`String`|
|`split(x)`|Split by delimiter x|`String[]`|
|`equals(x)`|Content equals? (case-sensitive)|`boolean`|
|`equalsIgnoreCase(x)`|Content equals? (ignore case)|`boolean`|
|`isEmpty()`|Length is 0?|`boolean`|
|`isBlank()`|Only whitespace? (Java 11+)|`boolean`|
|`startsWith(x)`|Starts with x?|`boolean`|
|`endsWith(x)`|Ends with x?|`boolean`|
|`String.valueOf(x)`|Convert x to String|`String`|
|`String.format(...)`|Formatted String|`String`|
