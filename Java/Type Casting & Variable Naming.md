# Type Casting
Type casting is **converting a value from one type to another**. There are 2 types of type casting.
1. Implicit Casting (Widening)
2. Explicit Casting (Narrowing)

##### ==Implicit== Casting
Java **automatically** converts a **smaller type to a larger type**.  
No data is lost. No special syntax needed.

```
byte → short → int → long → float → double (widening direction →)
```

**Syntax:**
```
int myInt = 100;
long myLong = myInt; // int → long (automatic, safe)
double myDouble = myLong; // long → double (automatic, safe)

System.out.println(myInt); // 100
System.out.println(myLong); // 100
System.out.println(myDouble); // 100.0
```

**Another example:**
```
byte b = 42;
short s = b; // byte → short (implicit)
int i = s; // short → int (implicit)
long l = i; // int → long (implicit)
float f = l; // long → float (implicit)
double d = f; // float → double (implicit)
```

##### ==Explicit== Casting
**Manually** converting a **larger type to a smaller type**.  
You must tell Java explicitly. **Data loss can occur.**
```
double → float → long → int → short → byte (narrowing direction ←)
```

**Syntax: `(targetType) value`**
```
double myDouble = 9.99;
int myInt = (int) myDouble; // Explicit cast required 

System.out.println(myDouble); // 9.99
System.out.println(myInt); // 9 ← decimal part is TRUNCATED (not rounded)
```

**More examples:**
```
long bigNumber = 1234567890123L;
int smallNumber = (int) bigNumber; // Data LOSS occurs System.out.println(smallNumber); // Some garbage value (overflow)

double price = 19.99;
int truncated = (int) price;
System.out.println(truncated); // 19 (not 20 — it truncates, not rounds)

// char ↔ int casting
char ch = 'A';
int ascii = (int) ch;
System.out.println(ascii); // 65

int num = 66;
char letter = (char) num;
System.out.println(letter); // B
```

##### Casting Summary:
```
Widening (Implicit) ──────────────────────────────────→
byte → short → int → long → float → double ←────────────────────────────────── Narrowing (Explicit)
```

|Type|Safe?|Syntax|Risk|
|---|---|---|---|
|Implicit (widening)|✅ Yes|Automatic|None|
|Explicit (narrowing)|⚠️ Maybe|`(type) value`|Data loss|

