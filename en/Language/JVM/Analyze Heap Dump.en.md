# Heap Dump Analysis

## Concept

A Heap Dump is a **snapshot of JVM heap memory state saved to a file at a specific point in time**.

It is used to analyze application memory usage, object reference relationships, and causes of memory leaks. It contains information about all objects in the heap and their reference structures.

### Key Characteristics

- **Point-in-time Snapshot**: Completely captures heap memory state at a specific moment
- **Memory Diagnosis**: Can analyze the cause of OOM (Out Of Memory) occurrences
- **Object Tracking**: Can identify which objects consume the most memory

---

## Why Needed?

### Problem to Solve

When memory leaks or OOM occur in a running application, logs alone cannot identify the cause. We need to know which objects are occupying memory and why they are not being garbage collected.

### Limitations of Existing Approaches

1. **Log Analysis**: Can see memory usage but cannot identify specific object information
2. **Monitoring Tools**: Show real-time memory usage but cannot analyze past points in time
3. **Speculation-based Debugging**: Difficult to find memory leak causes by just looking at code

### Value Provided

- **Accurate Diagnosis**: Directly examine actual objects and reference relationships in memory
- **Solving Unreproducible Issues**: Can analyze issues that only occur in production
- **Evidence-based Optimization**: Memory optimization based on data rather than speculation

---

## How It Works

### Heap Dump Generation Method

The JVM serializes all object information in heap memory and saves it to a file. STW (Stop The World) occurs during generation, temporarily pausing the application.

### Generation Methods

```bash
# 1. Manual generation with jmap command
jmap -dump:live,format=b,file=heap.hprof <PID>

# 2. Automatic generation on OOM (JVM option)
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/path/to/dumps \
     -jar application.jar

# 3. Using jcmd command
jcmd <PID> GC.heap_dump /path/to/heap.hprof
```

### File Format

Heap Dump files have a `.hprof` extension and are stored in binary format. File size is proportional to heap memory usage and can reach several GB.

---

## Pitfalls

### 1. Caution When Generating in Production

STW occurs during Heap Dump generation, stopping the application.

```bash
# ❌ Bad: Generating carelessly during peak time
jmap -dump:format=b,file=heap.hprof <PID>  # May cause service interruption

# ✅ Good: Set auto-generation option or choose off-peak time
-XX:+HeapDumpOnOutOfMemoryError  # Auto-generate only on OOM
```

### 2. Check Disk Space

Heap Dump files are very large, and generation fails if disk space is insufficient.

```bash
# ✅ Check disk space before generation
df -h /path/to/dumps

# ✅ Compress or delete old files if needed
gzip old-heap.hprof
```

### 3. Potential for Sensitive Information

Heap Dumps contain all data that was in memory, so sensitive information like passwords and API keys may be exposed.

```bash
# ✅ Delete immediately after analysis
rm -f heap.hprof

# ✅ Do not share externally
# Analyze Heap Dumps only internally and do not transmit externally
```

### Best Practices

- Always configure OOM auto-dump option (`-XX:+HeapDumpOnOutOfMemoryError`) in production
- Specify Heap Dump save path to a location with sufficient disk space
- Delete files immediately after analysis for security
- Practice generating/analyzing dumps regularly in test environments

---

## Practical Application

### Basic Analysis Process

Analysis using MAT (Memory Analyzer Tool)

```bash
# 1. Download and install MAT
# https://eclipse.dev/mat/downloads.php

# 2. Open Heap Dump file
# File > Open Heap Dump > Select heap.hprof
```

### Common Analysis Patterns

#### Pattern 1: Quick Diagnosis with Leak Suspects Report

Using MAT's automatic analysis feature

```
1. Open Heap Dump file
2. "Leak Suspects Report" runs automatically
3. Check objects consuming most memory in Suspect section
4. Click "See stacktrace" to track object creation location
```

**Check Points**:
- Review Problem Suspects 1, 2, 3... in order
- Find suspicious classes in Accumulated Objects by Class
- Check why objects aren't GC'd in Shortest Paths to GC Roots

#### Pattern 2: Memory Occupancy Analysis with Dominator Tree

Find objects consuming the most memory.

```
1. Open Dominator Tree view
2. Sort by Retained Heap
3. Track reference relationships of top objects
4. Check references with "List objects" > "with outgoing references"
```

**Example**:
```
com.example.Cache @ 0x7f8a1234
  └─ Retained Heap: 500 MB
     └─ java.util.HashMap @ 0x7f8a5678
        └─ Entry[] (size: 1,000,000)  // This is the problem!
```

#### Pattern 3: Instance Check by Class with Histogram

Analyze instance count and memory usage per class

```
1. Open Histogram view
2. Sort by "Shallow Heap" or "Retained Heap"
3. Right-click on suspicious class
4. Select "List objects" > "with incoming references"
5. Trace back to see what's referencing it
```

#### Pattern 4: Search Specific Objects with OQL

Search objects with SQL-like queries

```sql
-- Find all instances of a specific class
SELECT * FROM com.example.User

-- Find objects meeting specific conditions
SELECT * FROM java.lang.String s WHERE s.count > 1000

-- Find large collections
SELECT * FROM java.util.ArrayList a WHERE a.size > 10000
```

### Real Analysis Example

```
Problem: OOM occurred with java.lang.OutOfMemoryError: Java heap space

Analysis Process:
1. Open heap.hprof file with MAT
2. Check Leak Suspects Report
   → Problem Suspect 1: com.example.CacheManager (Retained: 3.2GB)
3. Check CacheManager in Dominator Tree
   → 1 million entries exist in HashMap
4. Search com.example.CacheEntry in Histogram
   → 1 million instances exist, 3KB each
5. Check Path to GC Roots
   → Referenced by static field, cannot be GC'd
   
Cause: Static Map in CacheManager grows indefinitely
Solution: Change to LRU cache + add maximum size limit
```

### Before/After Comparison

```kotlin
// ❌ Problem code: Cache grows indefinitely
object CacheManager {
    private val cache = mutableMapOf<String, Data>()
    
    fun put(key: String, data: Data) {
        cache[key] = data  // Keeps accumulating
    }
}

// ✅ Improved code: LRU cache with size limit
class CacheManager(private val maxSize: Int = 10000) {
    private val cache = object : LinkedHashMap<String, Data>(
        maxSize, 0.75f, true
    ) {
        override fun removeEldestEntry(
            eldest: MutableMap.MutableEntry<String, Data>?
        ): Boolean {
            return size > maxSize  // Automatically removes old entries
        }
    }
    
    fun put(key: String, data: Data) {
        cache[key] = data
    }
}
```

---

## References

### Official Documentation
- [Eclipse Memory Analyzer (MAT)](https://eclipse.dev/mat/)
- [Oracle JVM Troubleshooting Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/)

### Recommended Articles
- [Heap Dump Analysis Guide](https://incheol-jung.gitbook.io/docs/q-and-a/java/heap-dump-feat.-oom) - How to analyze Heap Dumps when OOM occurs
- [Practical MAT Usage](https://shonm.tistory.com/646) - Detailed guide on using MAT tool (Korean)
- [Solving OOM with Heap Dump](https://blog.yevgnenll.me/posts/heap-dump-out-of-memory) - Analysis process based on real cases (Korean)

### Related TIL
- [JVM](./JVM.md)
- [Garbage Collection](./Garbage%20Collection.md)
- [OOM](./OOM.md)
