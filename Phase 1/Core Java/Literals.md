# Literals
In Java, a **literal** is the actual, raw data or fixed value that you type directly into your source code to assign to a variable. You aren't calculating it or pulling it from a database; you are hardcoding the exact value.

#### 1. Integer Literals
By default, any whole number typed into Java is treated as a standard base-10 (decimal) `int`. However, Java allows you to represent whole numbers in several different number systems using specific prefixes.

1. **Decimal (Base 10):** The standard numbers we use every day.
- `int standard = 10;`

2. **Hexadecimal (Base 16):** Starts with `0x` or `0X`. Commonly used in memory addressing and color codes.
- `int hex = 0x1A;` _(1A in hex equals 26 in decimal)_

3. **Binary (Base 2):** Starts with `0b` or `0B`. Useful for low-level bitwise operations.
- `int binary = 0b101;` _(101 in binary equals 5 in decimal)_

4. **The Underscore Feature:** To make large numbers easier to read, Java allows you to place underscores between digits. The compiler completely ignores them.
- `int oneMillion = 1_000_000;`

#### 2. Floating-Point Literals
These are numbers that contain a decimal point. In Java, any decimal number is treated as a `double` (which takes up 64 bits of memory) by default.

1. **Float Literals (`f` or `F`):** If you specifically want to save memory and use a 32-bit `float`, you **must** append an `f` to the end of the number. Otherwise, the compiler will throw an error because it thinks you are trying to cram a massive 64-bit `double` into a smaller 32-bit `float` box.
- `float pi = 3.14f;`

2. **Double Literals:** Because `double` is the default, you can just write the number. (You can append a `d` or `D`, but it's rarely done).
- `double precisePi = 3.14159265359;`

#### 3. Character Literals
A character literal represents a single character and must always be enclosed in **single quotes** (`' '`).

1. **Standard Characters:**
- `char letter = 'A';`

2. **Escape Sequences:** These are special character combinations that start with a backslash (`\`) to represent things you can't easily type, like formatting commands.
- `'\n'` : Newline (drops the cursor to the next line)
- `'\t'` : Tab (inserts a horizontal tab space)
- `'\\'` : Backslash (allows you to print an actual backslash)

3. **Unicode Characters:** You can represent any character from any language using its Unicode value by starting with `\u`.
- `char omega = '\u03A9';` _(Prints the Greek letter Ω)_

#### 4. String Literals
While `String` is technically a reference data type, Java allows you to create them using literals for convenience. A string literal must always be enclosed in **double quotes** (`" "`).
- `String message = "Hello";`

Because it relies on double quotes, if you ever need to print a literal quote inside your text, you must use an escape sequence:
- `String quote = "He said, \"Java is fun!\"";`

#### 5. Boolean Literals
Boolean literals are the simplest. They represent the concept of truth and can only ever be one of two values. They must be entirely lowercase and are not surrounded by quotes.
- `boolean isOnline = true;`
- `boolean hasError = false;`
