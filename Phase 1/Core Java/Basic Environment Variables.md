**`JAVA_HOME`**, **`PATH`**, and **`Classpath`** are distinct [environment variables](https://www.geeksforgeeks.org/java/setting-environment-java/) used to configure the Java development and runtime environments. Understanding how they differ and when to use them is essential for resolving execution and build errors.
```
┌─────────────────────────────────────────────────────────┐
│  JAVA_HOME                                              │
│  ├── Points to JDK root directory                       │
│  ├── Used by build tools, IDEs, app servers             │
│  └── export JAVA_HOME=/path/to/jdk                      │
│                                                         │
│  PATH                                                   │
│  ├── OS searches these dirs for commands                │
│  ├── Must include $JAVA_HOME/bin                        │
│  └── export PATH=$JAVA_HOME/bin:$PATH                   │
│                                                         │
│  CLASSPATH                                              │
│  ├── JVM searches these for .class files                │
│  ├── Can be dirs, JARs, ZIPs, wildcards                 │
│  ├── java -cp out/:lib/* com.myapp.Main                 │
│  └── Package structure = directory structure            │
└─────────────────────────────────────────────────────────┘
```

#### Why This Matters?
Before writing a single line of Java:
- JVM must be installed
- OS must FIND `java`/`javac` commands
- Java must FIND your `.class` files

These 3 concepts solve exactly that:
- `JAVA_HOME` → where Java is installed
- `PATH` → where OS looks for commands
- Classpath → where Java looks for `.class` files

Benefits:
- Misconfiguration = nothing works.
- Understanding this = you can debug any environment issue.

#### Where Java is Installed?
Location of Java directory in different OS.
###### Typical Installation Paths
```java title:install_path.java
// ─── Windows ──────────────────────────────────────────────
C:\Program Files\Java\jdk-21
C:\Program Files\Eclipse Adoptium\jdk-21.0.1.12-hotspot

// ─── macOS ────────────────────────────────────────────────
/Library/Java/JavaVirtualMachines/jdk-21.jdk/Contents/Home
/usr/local/opt/openjdk@21          # Homebrew
/Users/username/.sdkman/candidates/java/21-open  # SDKMAN

// ─── Linux ────────────────────────────────────────────────
/usr/lib/jvm/java-21-openjdk-amd64   # apt (Ubuntu/Debian)
/usr/lib/jvm/java-21-openjdk         # yum (CentOS/RHEL)
/opt/java/21                          # manual install
```

###### JDK Directory Structure
```
jdk-21/
├── bin/                  ← executables (java, javac, jar...)
│   ├── java              ← JVM launcher
│   ├── javac             ← compiler
│   ├── javadoc           ← documentation generator
│   ├── jar               ← archive tool
│   ├── jdb               ← debugger
│   ├── jps               ← list Java processes
│   ├── jmap              ← memory map
│   └── jstack            ← thread dump
│
├── lib/                  ← JDK libraries
│   ├── ct.sym
│   ├── modules
│   └── src.zip           ← JDK source code!
│
├── include/              ← C headers (for JNI)
├── jmods/                ← Java modules
├── legal/                ← licenses
└── release               ← version info file
```

# `JAVA_HOME`
JAVA_HOME is environment variable pointing to JDK installation root, the "home" directory of your Java installation.

**Why it exists:**
- Build tools (Maven, Gradle) need to find JDK.
- Application servers (Tomcat, JBoss) need it.
- IDEs (IntelliJ, Eclipse) look for it.
- Shell scripts use it.
- Standardized way to say "here is my Java."

```
JAVA_HOME should point to JDK ROOT (not bin folder!)
✅ JAVA_HOME = /usr/lib/jvm/java-21-openjdk-amd64
❌ JAVA_HOME = /usr/lib/jvm/java-21-openjdk-amd64/bin
```

# `PATH`
PATH is environment variable listing directories, where OS looks for executable programs.

### How PATH Works?
```java title:path.java
// View your current PATH
echo $PATH
// /usr/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

// Each directory separated by : (Linux/Mac) or ; (Windows)
// OS checks each directory for the command

// When you type: javac MyApp.java
// OS checks: /usr/local/sbin/javac → not found
//            /usr/local/bin/javac  → not found
//            /usr/bin/javac        → not found
//            /bin/javac            → not found
//            /usr/lib/jvm/java-21/bin/javac → FOUND! runs it.
```

# `Classpath`
Classpath is a list of locations where JVM looks for .class files. Java's equivalent of OS PATH but for class files.

```java title:syntax.java
// When Java code does:
    import com.myapp.models.User;
    User u = new User();
```
JVM needs to FIND User.class → searches Classpath locations.

**Locations can be:**
- Directories    → /home/user/myapp/out/
- JAR files      → /home/user/libs/library.jar
- ZIP files      → /home/user/libs/archive.zip
- Wildcard       → /home/user/libs/*  (all JARs in folder)

### How Classpath Works?
```
Project structure:
myapp/
├── src/
│   └── com/
│       └── myapp/
│           ├── Main.java
│           └── models/
│               └── User.java
│
└── out/                     ← compiled .class files go here
    └── com/
        └── myapp/
            ├── Main.class
            └── models/
                └── User.class

Classpath = myapp/out/

JVM gets request: find com.myapp.models.User
JVM searches:     myapp/out/com/myapp/models/User.class → found!

Package structure MUST match directory structure!
```