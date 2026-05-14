*The final Keyword*
# What is a constant?
A **constant** is a variable whose value **cannot be changed** after it is assigned.  
Use the `final` keyword to declare a constant.

```
final double PI = 3.14159265358979;
final int MAX_LOGIN_ATTEMPTS = 3;
final String DATABASE_URL = "jdbc:postgresql://localhost:5432/mydb";
```

If you try to change a `final` variable, the compiler gives an error:
```
final int MAX_SIZE = 100;
MAX_SIZE = 200; // ❌ Compile error: cannot assign a value to final variable
```

# Why use constants?
- Prevents accidental changes to important values
- Makes code more readable (named values are clearer than magic numbers)
- Makes code easier to maintain (change the value in one place)

```
// ❌ Magic numbers — what do these mean?
if (attempts > 3) { ... }
double area = 3.14159 * radius * radius;

// ✅ Named constants — clear and maintainable
final int MAX_LOGIN_ATTEMPTS = 3;
final double PI = 3.14159;

if (attempts > MAX_LOGIN_ATTEMPTS) { ... }
double area = PI * radius * radius;
```

# The `static final` Constant.
In real Java applications, constants are usually declared as `static final`:
```
public class AppConfig {
public static final int MAX_CONNECTIONS = 100;
public static final String APP_VERSION = "1.0.0";
public static final double TAX_RATE = 0.18;
}
```

- ==`static`== → belongs to the class, not to any object.
- ==`final`== → cannot be changed after assignment.