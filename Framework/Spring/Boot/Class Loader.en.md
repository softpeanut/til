# Class Loader

## Concept

Class Loader is **the mechanism by which the JVM loads class files (.class) into memory**. When a Java application runs, it doesn't load all classes at once but dynamically loads them when needed.

### Key Characteristics

- **Dynamic Loading**: Classes are loaded when actually needed
- **Hierarchical Structure**: Class Loaders have a parent-child hierarchical relationship
- **Delegation Model**: Follows the Parent Delegation Model, delegating class loading requests to parent
- **Namespace Separation**: Each Class Loader has an independent namespace to prevent class conflicts

---

## Why Needed

### Problems to Solve

Loading all classes at application startup causes memory waste and long startup time. Also, conflicts can occur when classes with the same name exist in multiple locations.

### Limitations of Existing Approach

1. **Memory Waste**: Loading all classes including unused ones
2. **Long Startup Time**: Waiting until all classes are loaded
3. **Class Conflicts**: Unable to distinguish classes with identical names
4. **Security Issues**: Difficulty separating untrusted code from system code

### Value Provided

- **Efficient Memory Usage**: Save memory by loading only necessary classes
- **Fast Startup Time**: Load minimal classes at application startup
- **Modularization**: Prevent class conflicts with independent namespaces per Class Loader
- **Security**: Enhance security by separating system and application classes
- **Hot Swap**: Quick feedback by reloading classes during development

---

## How It Works

### Class Loader Hierarchy

JVM's Class Loader consists of a 3-tier hierarchy.

```
Bootstrap ClassLoader (null)
    ↓
Platform ClassLoader (PlatformClassLoader)
    ↓
Application ClassLoader (AppClassLoader)
    ↓
Custom ClassLoader (e.g., RestartClassLoader)
```

**Role of Each Class Loader:**

1. **Bootstrap ClassLoader**
   - Implemented in JVM's native code, not a Java object (shown as null)
   - Loads JDK core libraries (`java.lang.*`, `java.util.*`, etc.)
   - rt.jar and core libraries in `$JAVA_HOME/lib` directory

2. **Platform ClassLoader** (Before Java 9: Extension ClassLoader)
   - Loads Java platform extension libraries
   - Classes in `$JAVA_HOME/lib/ext` or `java.ext.dirs` path
   - Standard extension libraries like JDBC drivers (`javax.*`)

3. **Application ClassLoader** (AppClassLoader)
   - Loads classes from application classpath
   - Application code written by developers
   - External library JAR files

4. **Custom ClassLoader**
   - User-defined Class Loader for special purposes
   - Spring Boot Devtools' RestartClassLoader (Hot Reload)
   - Application server's WAR isolation

### Parent Delegation Model

When a class loading request occurs, it's processed in the following order.

```
1. Class loading request occurs
   ↓
2. Check if already loaded by current Class Loader
   ↓
3. Delegate to parent Class Loader (recursively up to Bootstrap)
   ↓
4. If parent fails to load, current Class Loader attempts to load
   ↓
5. Load succeeds or ClassNotFoundException thrown
```

**Operation Flow:**

```kotlin
// Example: Loading String class
AppClassLoader.loadClass("java.lang.String")
    → PlatformClassLoader.loadClass("java.lang.String")
        → BootstrapClassLoader.loadClass("java.lang.String")
            → ✅ Bootstrap loads successfully (rt.jar)
```

**Benefits:**
- **Security**: Load core system classes first to protect from malicious code
- **Consistency**: Always use parent's loaded version even if same class exists in multiple locations
- **Prevent Duplication**: Don't reload already loaded classes

### Class Loader in Spring Boot

Spring Boot applications primarily use AppClassLoader, and RestartClassLoader is added when devtools is enabled.

**AppClassLoader Loading Order:**

1. **Parent Delegation**: PlatformClassLoader → BootstrapClassLoader
2. **Application Classes**: `build/classes/kotlin/main` directory
3. **Library JARs**: `libs/*.jar` or dependency JAR files

**RestartClassLoader (Spring Boot Devtools):**
- Load only application code with RestartClassLoader
- Load libraries with AppClassLoader (retained on restart)
- Recreate only RestartClassLoader on code changes for fast restart

---

## Pitfalls

### 1. Understand Priority When Classes Are Duplicately Defined

Due to parent delegation model, parent loads first even if same-named classes exist in multiple locations.

```kotlin
// Attempting to redefine kotlin.String
package kotlin

class String {
    // Redefined content
}

// Actually loaded class:
// → java.lang.String loaded by BootstrapClassLoader
// → Redefined class is never used
```

**How to Check ClassLoader:**

```kotlin
@RestController
class ClassLoaderController {
    
    @GetMapping("/classloader")
    fun checkClassLoader(): Map<String, Any?> {
        return mapOf(
            "String" to String::class.java.classLoader, // null (Bootstrap)
            "DataSource" to javax.sql.DataSource::class.java.classLoader, // Platform
            "RestController" to org.springframework.web.bind.annotation.RestController::class.java.classLoader, // App
            "MyClass" to this::class.java.classLoader // App or Restart
        )
    }
}
```

### 2. Beware of Class Loading Conflicts

If the same class is loaded by different Class Loaders, they're recognized as different classes.

```kotlin
// User loaded by ClassLoader A
val user1 = classLoaderA.loadClass("com.example.User").newInstance()

// User loaded by ClassLoader B
val user2 = classLoaderB.loadClass("com.example.User").newInstance()

// ClassCastException thrown!
// user1 and user2 are recognized as different classes
```

### 3. Potential Memory Leaks

If Class Loader is not removed, all loaded classes remain in memory.

```kotlin
// Cause of Metaspace OOM
// - Dynamically created Class Loaders not released
// - Proxy classes created by Reflection keep increasing
// - Class unloading doesn't occur

// Solutions:
// 1. Explicitly release Class Loader references
// 2. Set MaxMetaspaceSize
// 3. Reduce unnecessary dynamic class creation
```

---

## Practical Application

### 1. Verify Class Loading Order

Check which Class Loader actually loaded the classes.

```kotlin
@RestController
class ClassLoaderTestController {
    
    @GetMapping("/test-classloader")
    fun testClassLoader(): Map<String, String> {
        val results = mutableMapOf<String, String>()
        
        // 1. kotlin.String - Loaded by Bootstrap (null)
        results["kotlin.String"] = String::class.java.classLoader?.toString() ?: "BootstrapClassLoader"
        
        // 2. javax.sql.DataSource - Loaded by Platform
        results["javax.sql.DataSource"] = javax.sql.DataSource::class.java.classLoader.toString()
        
        // 3. Spring library - Loaded by App
        results["RestController"] = org.springframework.web.bind.annotation.RestController::class.java.classLoader.toString()
        
        // 4. User-defined class - Loaded by App or Restart
        results["MyController"] = this::class.java.classLoader.toString()
        
        return results
    }
}
```

**Expected Output:**
```json
{
  "kotlin.String": "BootstrapClassLoader",
  "javax.sql.DataSource": "jdk.internal.loader.ClassLoaders$PlatformClassLoader@xxx",
  "RestController": "org.springframework.boot.loader.LaunchedURLClassLoader@xxx",
  "MyController": "org.springframework.boot.devtools.restart.classloader.RestartClassLoader@xxx"
}
```

### 2. Class Redefinition Experiment

Check what happens when redefining a class in the same package path.

```kotlin
// 1. Attempting to redefine javax.sql.DataSource
package javax.sql

interface DataSource {
    fun customMethod(): String
}

// 2. Verify actual loading
@Service
class DataSourceTest(
    private val dataSource: javax.sql.DataSource // Which DataSource will be injected?
) {
    fun test() {
        println(dataSource::class.java.classLoader)
        // PlatformClassLoader - Standard library loaded, not redefined class
    }
}
```

**Result:**
- `javax.sql.DataSource`: Loaded by PlatformClassLoader (standard library takes priority)
- Redefined class is never used

### 3. Utilize Spring Boot Devtools

Implement fast restart with RestartClassLoader.

```kotlin
// build.gradle.kts
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}

// application.yml
spring:
  devtools:
    restart:
      enabled: true
      additional-paths: src/main/kotlin
      exclude: static/**,public/**
```

**How It Works:**
```
Change detected
    ↓
Recreate only RestartClassLoader
    ↓
Reload only application code
    ↓
Libraries retained with existing AppClassLoader (fast restart)
```

### 4. Debug Class Loading

Check class loading process with logs.

```bash
# JVM option
-verbose:class
# or
-XX:+TraceClassLoading

# Example output:
[Loaded java.lang.String from /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home/lib/modules]
[Loaded javax.sql.DataSource from /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home/lib/modules]
[Loaded com.example.MyController from file:/app/build/classes/kotlin/main/]
```

---

## References

### Official Documentation
- [Oracle ClassLoader Documentation](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassLoader.html)
- [JVM Specification - Loading](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-5.html)

### Recommended Articles
- [Understanding Class Loader](https://kkang-joo.tistory.com/10)
- [Spring Boot Devtools RestartClassLoader](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools.restart)

### Related TIL
- [JVM](../../../Language/JVM/JVM.md) - JVM architecture and Class Loader location