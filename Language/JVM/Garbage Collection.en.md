# Garbage Collection

## Concept

Garbage Collection (GC) is a mechanism in the JVM that automatically detects and reclaims memory from objects that are no longer in use. The JVM automatically removes unused objects to prevent memory leaks without requiring developers to explicitly free memory.

## Why Needed

Manual memory management leads to memory leaks and dangling pointer problems due to developer mistakes.

- **Memory Leak Prevention**: Objects no longer in use remaining in memory eventually cause OOM
- **Improved Development Productivity**: Developers can focus on business logic without managing memory deallocation
- **Stability Guarantee**: Eliminates dangling pointer problems that reference already freed memory
- **Automatic Optimization**: The JVM selects and tunes the optimal GC strategy for the runtime environment

## How It Works

### Heap Memory Structure (Hotspot JVM)

The Hotspot JVM manages the Heap in a Generational manner, placing objects in different areas based on their lifetime.

```
Heap Memory
├── Young Generation
│   ├── Eden          : Area where new objects are initially allocated
│   └── Survivor (S0, S1) : Area where objects surviving from Eden temporarily stay
└── Old Generation    : Area where objects that survived long in Young are moved
```

**Young Generation:**
- Eden: The area where new objects are initially allocated when created
- Survivor (S0, S1): The area where objects surviving Minor GC from Eden are moved
- Minor GC: Occurs when Eden fills up, moving live objects to Survivor and clearing Eden

**Old Generation:**
- Promotion: Objects that survive multiple Minor GCs and exceed a certain Age threshold are moved from Young to Old
- Major GC (Full GC): Occurs when Old Generation memory is insufficient, cleaning the entire Heap
- Stores objects that are referenced for a relatively long time and have a high probability of continued use
- Card Table: Manages information about references from Old to Young generation objects

### GC Root Set

GC traverses all Reachable objects starting from the Root Set and reclaims unreachable objects.

**Root Set Conditions:**
1. **Classes**: Static fields and methods of classes maintain references until program termination and are considered GC Roots
   - The JVM stores loaded class information in ClassLoaderData (CLDR) and GC traverses the CLDR chain to extract static field and ConstantPool references

2. **Thread Stacks**: Local variables and parameters of methods are valid during method execution and are considered GC Roots
   - When a thread is suspended, each stack frame is explored and the JIT's OopMap identifies which variables are object references

3. **Active Java Threads**: Until currently executing or waiting threads terminate, all related objects are considered GC Roots
   - Active threads are explored through Thread Roots and object references are extracted from thread locals and stack frames

4. **JNI References**: Objects referenced by native code are managed outside the JVM and are not GC targets
   - Object references are extracted from the Handle Table registered by the JVM

5. **Synchronization Monitor Objects**: Objects used in `synchronized` blocks are excluded from GC while holding locks

```
GC Root Set
    ↓
    ├── Object A (Reachable)
    │   ├── Object B (Reachable)
    │   └── Object C (Reachable)
    │
    └── Object D (Reachable)

Object E (Unreachable) ← Garbage
Object F (Unreachable) ← Garbage
```

### Reference Types

The JVM provides 4 Reference types based on reference strength.

| **Reference Type** | **Description** | **GC Behavior** |
|------------------|---------|-----------|
| Strong Reference | Regular object reference (`Object obj = new Object()`) | Never reclaimed if reachable from Root Set |
| Soft Reference | Used for cache objects | Reclaimed only when memory is low |
| Weak Reference | Weak reference relationship | Always reclaimed in the next GC cycle |
| Phantom Reference | Tracking objects about to be destroyed | Enqueued in reference queue after finalize, before memory reclamation |

**Reference Determination Order:**
1. Strong: Objects reachable from Root Set
2. Soft: Not Strong, but has at least one path through only Soft References
3. Weak: Not Strong or Soft, but has at least one path through only Weak References
4. Phantom: Not Strong, Soft, or Weak. Finalized but memory not yet reclaimed

```kotlin
// Weak Reference usage example
val cache = WeakHashMap<String, User>()
cache["key"] = User("name")  // Weak Reference
// Will be reclaimed in next GC if no other Strong References exist
```

### GC Algorithms

GC uses various algorithms to determine object survival and reclaim memory.

**Major Algorithms:**
- **Mark and Sweep**: Marks reachable objects from Root Set and removes (Sweep) unmarked objects
- **Mark and Compact**: After Mark and Sweep, moves live objects to the front of the Heap to reduce fragmentation
- **Copy/Scavenge**: Copies live objects to another area and completely empties the original area (used in Young Generation)
- **Concurrent Mark/Sweep**: Performs marking and collection concurrently during application execution to reduce STW (Stop-The-World) time

### GC Implementations

Java provides various GC implementations that can be selected based on application characteristics.

#### 1. Serial GC

The simplest approach that performs GC with a single thread.

- **Minor GC**: Performs Mark and Sweep on Young Generation
- **Major GC**: Performs Mark and Sweep and Compact on Old Generation
- **Target**: Environments with 1 CPU core or small Heap size (client applications)
- **Disadvantage**: Long STW time due to single thread operation

```bash
-XX:+UseSerialGC
```

#### 2. Parallel GC

A method that increases throughput by parallel processing Serial GC with multiple threads.

- **Minor GC**: Parallel processing of Young Generation with multiple threads
- **Major GC**: Parallel processing of Old Generation with multiple threads
- **Target**: Batch applications where throughput is important in multi-core environments
- **Disadvantage**: STW time is shorter than Serial GC but still occurs

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=4  # Specify number of GC threads
```

#### 3. CMS GC (Concurrent Mark Sweep)

Minimizes STW time by performing GC concurrently during application execution.

**Major GC Process:**
1. Initial Mark (STW): Mark only objects directly referenced from Root Set
2. Concurrent Mark: Mark entire Old Generation during application execution
3. Remark (STW): Re-mark objects changed during Concurrent Mark
4. Concurrent Sweep: Remove unmarked objects during application execution

- **Target**: Web applications where response time (Latency) is important
- **Disadvantage**: Memory fragmentation occurs as Compact is not performed, and uses more CPU resources

```bash
-XX:+UseConcMarkSweepGC  # Deprecated after Java 9, removed in Java 14
```

#### 4. G1 GC (Garbage First)

Performs GC efficiently by dividing the Heap into Region units.

**Structure:**
```
Heap (Divided into Regions)
├── Eden Region
├── Survivor Region
├── Old Region
├── Humongous Region  : Large objects exceeding 50% of Region size
└── Available/Unused  : Regions not yet used
```

**Operation:**
- **Minor GC**: Copies objects from Eden Region to Survivor or Available Region and empties Eden
- **Mixed GC**: When Minor GC is insufficient, collects Young and some Old Regions together
- **Region Selection Strategy**: Prioritizes collecting Regions with highest GC efficiency (most reclaimable memory)
- **Compact**: Naturally resolves fragmentation by copying only live objects to other Regions

- **Target**: Server applications using large Heap (4GB or more)
- **Advantages**: Controls STW time to predictable levels and resolves fragmentation issues

```bash
-XX:+UseG1GC  # Default GC since Java 9
-XX:MaxGCPauseMillis=200  # Target maximum STW time
```

#### 5. ZGC / Shenandoah GC

Low-latency GC that guarantees millisecond-level short STW even on very large Heap (several TB).

- **ZGC**: Performs most work concurrently using Colored Pointer and Load Barrier
- **Shenandoah GC**: Allows application access to objects even during object movement using Brooks Pointer
- **Target**: Applications requiring very large Heap and extremely short response times
- **STW Time**: Limited to under 10ms

```bash
-XX:+UseZGC        # ZGC (officially supported since Java 15)
-XX:+UseShenandoahGC  # Shenandoah GC
```

## Pitfalls

### 1. Avoid Using finalize()

The `finalize()` method is called before GC reclaims an object, but does not guarantee call timing and causes performance degradation.

```kotlin
// Wrong approach - using finalize()
class Resource {
    protected fun finalize() {
        // Cleanup task - unknown when it will be called
        cleanup()
    }
}

// Correct approach - try-with-resources (Kotlin: use)
class Resource : Closeable {
    override fun close() {
        cleanup()  // Called immediately and explicitly
    }
}

Resource().use { resource ->
    // close() called automatically after use
}
```

### 2. Tune GC Carefully

Default GC settings are efficient enough for most cases. If performance issues are not clearly measured, it's better not to attempt GC tuning.

**Pre-tuning Checklist:**
- Analyze GC logs to measure actual STW time and frequency
- Check for memory leaks with Heap Dump
- Optimize application code (reduce unnecessary object creation)

```bash
# Enable GC logging
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```

### 3. Consider Young/Old Size Ratio

If Young Generation is too small, Minor GC occurs frequently; if too large, Minor GC time becomes long. Generally, about 1/3 of Heap is appropriate.

```bash
# Specify Young Generation size
-XX:NewRatio=2  # Old:Young = 2:1 (Young is 1/3 of total)
-Xmn512m        # Fixed Young Generation size
```

## Practical Application

### 1. GC Selection Criteria

Select appropriate GC based on application characteristics.

| **Application Type** | **Recommended GC** | **Reason** |
|-----------------|-----------|---------|
| Web Application (Large Heap) | G1 GC | Predictable response time, fragmentation resolution |
| Batch Processing (Throughput-focused) | Parallel GC | Fast processing with multiple threads |
| Real-time System (Ultra-low latency) | ZGC / Shenandoah | Millisecond-level STW |
| Small Application (Single core) | Serial GC | Minimal overhead |

```bash
# Spring Boot application example (4GB Heap)
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

### 2. Memory Leak Detection Patterns

Memory leaks can occur even when GC operates normally. Check major patterns.

```kotlin
// 1. Infinitely adding objects to Static Collection
companion object {
    private val cache = mutableListOf<User>()  // Keeps growing
}

// Solution: Use cache with size limit
companion object {
    private val cache = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .build<String, User>()
}

// 2. Missing Thread Local cleanup
private val threadLocal = ThreadLocal<Connection>()

fun process() {
    threadLocal.set(createConnection())
    // ... work
    // threadLocal.remove() missing - leaks when using thread pool
}

// Solution: Always clean up
fun process() {
    try {
        threadLocal.set(createConnection())
        // ... work
    } finally {
        threadLocal.remove()  // Explicit cleanup
    }
}
```

### 3. GC Monitoring Metrics

Continuously monitor GC status in production environments.

```kotlin
// Spring Boot Actuator + Micrometer
@Component
class GcMonitor(private val meterRegistry: MeterRegistry) {
    
    @Scheduled(fixedRate = 60000)  // Every 1 minute
    fun checkGcMetrics() {
        val gcPauseTime = meterRegistry.timer("jvm.gc.pause").totalTime(TimeUnit.MILLISECONDS)
        val gcCount = meterRegistry.counter("jvm.gc.pause").count()
        
        if (gcPauseTime > 1000) {  // STW over 1 second
            log.warn("High GC pause time: ${gcPauseTime}ms")
        }
    }
}
```

**Key Monitoring Metrics:**
- GC Frequency (Minor/Major GC count)
- STW Time (Average/Maximum)
- Heap Usage (Young/Old separately)
- GC Throughput (Application execution time / Total time)

### 4. Heap Dump Analysis

Automatically generate Heap Dump when OOM occurs to analyze the cause.

```bash
# Auto-generate Heap Dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof

# Manually generate Heap Dump
jmap -dump:live,format=b,file=heap.hprof <pid>
```

Open the Heap Dump with MAT (Memory Analyzer Tool) to identify objects consuming the most memory and GC Root paths.

## References

- [Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html) - Oracle GC tutorial
- [Naver D2: Java Garbage Collection](https://d2.naver.com/helloworld/1329)
- [Getting Started with the G1 Garbage Collector](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
- [ZGC Documentation](https://wiki.openjdk.org/display/zgc)

### Related TIL
- [JVM.en.md](./JVM.en.md) - JVM memory structure
- [OOM.en.md](./OOM.en.md) - OOM types and solutions
- [Analyze Heap Dump.en.md](./Analyze%20Heap%20Dump.en.md) - Heap Dump analysis methods
