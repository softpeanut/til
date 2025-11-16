# JVM (Java Virtual Machine)

## Concept

JVM is a **virtual machine that executes Java bytecode**.

It enables bytecode (.class) compiled from Java source code (.java) to run independently of the operating system. It is the core implementation of the "Write Once, Run Anywhere" philosophy.

### Key Characteristics

- **Platform Independence**: Executes the same bytecode regardless of operating system
- **Automatic Memory Management**: Automatic memory reclamation through GC (Garbage Collection)
- **Runtime Optimization**: Performance optimization during execution through JIT (Just-In-Time) compiler

---

## Why Needed?

### Problem to Solve

Native languages (C/C++) must be compiled separately for each operating system and hardware. Different platforms require recompilation, and developers must manage memory manually.

### Limitations of Existing Approaches

1. **Platform Dependency**: Different executable files needed for each OS
2. **Manual Memory Management**: Direct memory management required through malloc/free, etc.
3. **Lack of Portability**: Recompilation needed for each platform even without code changes

### Value Provided

- **Portability**: Code written once runs on all platforms
- **Stability**: Reduced memory leaks and errors through automatic memory management
- **Productivity**: Developers can focus on business logic

---

## How It Works

### JVM Architecture

JVM consists of three stages: **Class Loader → Runtime Data Area → Execution Engine**.

```
Java Source (.java)
    ↓ javac
Bytecode (.class)
    ↓ Class Loader
Runtime Data Area
    ↓ Execution Engine
Native Machine Code
```

### 1. Class Loader

Reads `.class` files and loads them into JVM memory.

**Loading Process**:
1. **Loading**: Read class file and load into memory
2. **Linking**: Verify bytecode, allocate memory, convert symbolic references to actual references
3. **Initialization**: Initialize static fields

**Class Loader Hierarchy**:
```
BootstrapClassLoader (JVM built-in, java.lang.*, etc.)
    ↓
PlatformClassLoader (JDK standard API)
    ↓
AppClassLoader (Application classes)
    ↓
Custom ClassLoader (User-defined)
```

**Delegation Model**:
- When a child ClassLoader receives a loading request, it first delegates to the parent
- If the parent cannot load, the child attempts to load
- **Purpose**: 
  - Protect core classes from being redefined by applications
  - Ensure classes with the same name share the same definition within the same JVM

### 2. Runtime Data Area

Memory areas used by JVM during program execution.

| Area | Scope | Description |
|------|-------|-------------|
| **PC Register** | Per Thread | Address of currently executing JVM instruction (similar to CPU register) |
| **JVM Stack** | Per Thread | Stores local variables, parameters, return values, etc. when methods are called |
| **Native Method Stack** | Per Thread | Stack for native method calls |
| **Method Area** | JVM-wide | Stores class metadata, constants, static variables |
| **Heap** | JVM-wide | Area where object instances are allocated |

#### Heap Structure (HotSpot JVM)

```
+---------------------------+
|    Young Generation       |
|  +-----+  +----------+    |
|  |Eden |  |Survivor  |    |
|  |     |  | S0 | S1  |    |
|  +-----+  +----------+    |
+---------------------------+
|    Old Generation         |
|  (Long-lived objects)     |
+---------------------------+
|    Metaspace              |
|  (Class metadata)         |
+---------------------------+
```

**Young Generation**:
- **Eden**: Area where objects are initially allocated
  - Referenced objects move to Survivor, others remain as Garbage
  - Eden is cleared when all objects move to Survivor
- **Survivor (S0, S1)**: Area where objects that survived Eden move to
- **Minor GC**: Move from Eden → Survivor and clear Eden

**Old Generation**:
- Objects that survived long in Young Generation move here (Promotion)
- Objects that exceed Age threshold move
- Stores objects that are relatively long-referenced and likely to continue being used
- Card table exists to manage references to Young Generation objects
- **Major GC (Full GC)**: Executed when Old Generation memory is insufficient

**Metaspace (Java 8+)**:
- Replaces Perm area from Java 7 and earlier
- Manages class metadata in OS native memory
- Static variables and constants are managed in Heap

### 3. Execution Engine

Engine that executes bytecode.

**Main Components**:
- **Interpreter**: Interprets and executes bytecode line by line
- **JIT Compiler**: Compiles frequently executed code to native code for performance improvement
- **Garbage Collector**: Automatically reclaims objects no longer in use (see separate document)

---

## Pitfalls

### 1. Memory Leak Prevention

Java provides automatic memory management but memory leaks can still occur.

```java
// ❌ Bad: Continuously adding to static collection
public class Cache {
    private static Map<String, Object> cache = new HashMap<>();
    
    public void add(String key, Object value) {
        cache.put(key, value); // Keeps accumulating, occupies memory if class is not unloaded
    }
}

// ✅ Good: Set size limit
public class Cache {
    private static final int MAX_SIZE = 1000;
    private static Map<String, Object> cache = new LinkedHashMap<>(MAX_SIZE, 0.75f, true) {
        protected boolean removeEldestEntry(Map.Entry eldest) {
            return size() > MAX_SIZE; // Automatically removes old entries
        }
    };
}
```

### 2. Never Use finalize()

The `finalize()` method is unpredictable and causes performance issues.

```java
// ❌ Bad
@Override
protected void finalize() throws Throwable {
    // Resource release can be delayed as GC timing is unknown
    cleanup();
}

// ✅ Good: Use try-with-resources
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // Automatically calls close()
}
```

### Best Practices

- Minimize unnecessary object creation (object pooling, using StringBuilder, etc.)
- Be cautious when using static collections (cause of memory leaks)
- Explicitly release resources with try-with-resources
- Utilize memory profiling tools (VisualVM, JProfiler, etc.)

---

## Practical Application

### JVM Option Configuration

```bash
# Set Heap size
java -Xms2g -Xmx4g MyApp

# Set Stack size
java -Xss1m MyApp

# Generate Heap Dump on OOM
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/dumps MyApp

# Print class loading information
java -verbose:class MyApp
```

### Monitoring Tools

```bash
# 1. jps - Check running Java processes
jps -v

# 2. jinfo - Check JVM configuration
jinfo <PID>

# 3. jmap - Check Heap information
jmap -heap <PID>

# 4. jstack - Thread dump
jstack <PID>

# 5. jcmd - Multi-purpose diagnostic tool
jcmd <PID> VM.version
jcmd <PID> VM.flags
```

### Primitive vs Reference Type

**Primitive Type**:
- Value stored directly in Stack memory
- null not allowed, generics not allowed
- Types: `byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean`

**Reference Type**:
- Reference (address) in Stack, actual object in Heap
- null allowed, generics allowed
- All classes, interfaces, arrays (inherit from `java.lang.Object`)

```java
// Primitive Type
int a = 10;  // 10 stored directly in Stack

// Reference Type
String str = "Hello";  // Reference in Stack, "Hello" object in Heap
```

---

## References

### Official Documentation
- [Oracle JVM Specification](https://docs.oracle.com/javase/specs/jvms/se17/html/)
- [Java Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)

### Related TIL
- [Class Loader](../../Framework/Spring/Boot/Class%20Loader.md)
- [Garbage Collection](./Garbage%20Collection.md)
- [OOM](./OOM.md)
- [Analyze Heap Dump](./Analyze%20Heap%20Dump.md)
