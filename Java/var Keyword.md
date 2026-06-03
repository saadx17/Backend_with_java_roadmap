# `var` keyword (Java 10+)
Introduced in Java 10, the `var` keyword brought a highly requested feature to the language: **Local Variable Type Inference**.
Instead of forcing you to explicitly declare the data type on the left side of the equals sign, `var` tells the Java compiler to look at the value on the right side and figure out the data type for itself.

#### The Core Concept: Less Boilerplate
Before Java 10, declaring variables with long class names often led to redundant, verbose code.

**The Old Way:**
```
// You have to type "HashMap" and its generics twice
HashMap<String, List<Integer>> userData = new HashMap<String, List<Integer>>();
```

**The New Way with `var`:**
```
// The compiler sees the right side and automatically knows userData is a HashMap
var userData = new HashMap<String, List<Integer>>();
```

**Important:** Java is Still Statically Typed!
Using `var` does **not** turn Java into a dynamically typed language like Python or JavaScript. The type is inferred at _compile-time_, not run-time. Once the compiler assigns a type to a `var`, that variable is locked into that type forever.
```
var score = 100; // The compiler locks "score" in as an int
score = "High Score"; // ERROR: Incompatible types. You cannot put a String into an int.
```

#### The Strict Rules of `var`
Because the compiler needs to be 100% certain of the data type, `var` is heavily restricted. You can only use it when the context is perfectly clear.

###### 1. It can ONLY be used for Local Variables
You can only use `var` for variables declared _inside_ a method, in a `for` loop, or in a `try-with-resources` block.
```
public class Player {
    var maxHealth = 100; // ERROR: Cannot be used for class/instance variables
    
    public void heal(var amount) { // ERROR: Cannot be used for method parameters
        var currentHealth = 50; // VALID: Local variable inside a method
    }
}
```

###### 2. It MUST be initialized immediately
The compiler cannot guess the type if you don't give it a value right away.
```
var playerName; // ERROR: Cannot infer type without an assignment
playerName = "Mario"; 

var currentName = "Luigi"; // VALID
```

###### 3. It CANNOT be assigned to `null`
Because `null` doesn't have a specific type, the compiler doesn't know what kind of reference variable to create.

```
var activePlayer = null; // ERROR: Variable initializer is 'null'
```

###### 4. It requires explicit types for Arrays
You cannot use the array initialization shortcut with `var`.
```
var scores = {10, 20, 30}; // ERROR: Array initializer needs an explicit target-type
var validScores = new int[]{10, 20, 30}; // VALID
```

#### When should you use it?
The golden rule for `var` is **Readability**.
Use it when the right side of the assignment makes it blindingly obvious what the data type is. It is fantastic for shortening long object creations or cleaning up `for` loops:
```
// Excellent use case
for (var entry : map.entrySet()) { ... }

var stream = new FileInputStream("data.txt");
```

Avoid it when the method you are calling obscures the return type. If another developer has to guess what data type is coming back, go back to using explicit declarations:
```
// Bad use case - What is data? A String? A byte array? A custom Object?
var data = processNetworkRequest();
```
