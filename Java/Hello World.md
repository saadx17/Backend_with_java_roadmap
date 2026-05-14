# Your First Java Program!

```
public class Hello{
public static void main(String[] args){
System.out.println("Hello, World!"); }
}
```

#### **Breaking down EVERY single word:**

```
public class Hello {
```

- `public` → This class is accessible from anywhere.
- `class` → We are defining a class (everything in Java lives inside a class)
- `Hello` → The name of the class (must match the filename: `Hello.java`)
- `{` → Opens the class body.

```
public static void main(String[] args) {
```

- `public` → The method is accessible from anywhere (JVM needs to access it)
- `static` → This method belongs to the class itself, not an object. JVM can call it without creating an object.
- `void` → This method returns nothing.
- `main` → Special name — this is the **entry point** of the program. JVM always looks for `main` to start.
- `String[] args` → An array of Strings passed from command line (can be ignored for now)
- `{` → Opens the method body.

```
System.out.println("Hello, World!");
```

- `System` → A built-in Java class that provides system-level utilities.
- `out` → A static field in System — represents the standard output stream (your console/terminal)
- `println` → A method that prints text and adds a new line at the end
- `"Hello, World!"` → A String literal — the text to print.
- `;` → Every statement in Java ends with a semicolon.

```
  }
}
```

- First `}` → Closes the `main` method.
- Second `}` → Closes the `Hello` class.

##### Key rules to remember:

| Rule                             | Explanation                                    |
| -------------------------------- | ---------------------------------------------- |
| File name must match class name  | `Hello.java` must contain `public class Hello` |
| Java is case-sensitive           | `Main` ≠ `main`                                |
| Every statement ends with `;`    | `System.out.println("Hi");`                    |
| `main` method is the entry point | JVM always starts from `main`                  |
| Everything lives inside a class  | You cannot write code outside a class          |
