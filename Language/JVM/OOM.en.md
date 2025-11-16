# Out Of Memory (OOM)

## Concept

Out Of Memory is an error that occurs when available memory is insufficient in the JVM or container environment. The JVM manages various memory areas including Heap, Metaspace, Direct Memory, and Native Memory, and different types of OOM errors occur when memory runs out in each area.

## Why Needed

OOM is a critical issue that directly impacts system stability and performance.

- **Failure Prevention**: When OOM occurs, the application terminates abnormally or experiences severe performance degradation
- **Resource Management**: Understanding memory usage patterns and setting appropriate limits ensures stable operations
- **Performance Optimization**: Analyzing OOM causes helps reduce unnecessary memory usage and improve application performance
- **Cost Reduction**: Efficient memory usage saves server resources and reduces operational costs

## How It Works

### OOM Occurrence Mechanism

The JVM and container each manage memory independently, and OOM can occur at both levels.

```
Container Memory Limit (e.g., 1GB)
├── JVM Heap (-Xmx) (e.g., 512MB)
├── Metaspace (Class Metadata)
├── Direct Memory (NIO Buffers)
├── Native Memory (Thread Stack, JNI, etc.)
└── OS Overhead
```

When the container memory limit is exceeded, `Container OOMKilled` occurs. When each memory area within the JVM reaches its limit, the corresponding OOM error for that area occurs.

### OOM Types and Causes

| **OOM Type**                         | **Cause**                                                                                | **Solution**                                                                                                        | **Notes**                                             |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| Container OOMKilled                | - JVM heap setting exceeds container memory limit                                                        | - Limit heap memory to about 60~80% of container<br>    <br>- Configure MaxRAMPercentage (default 25%)                                        | - Not set to 100% to account for native memory             |
| Java heap space                    | - Creating and maintaining many objects<br><br>- Memory leaks (collections, etc.)<br>    <br>- Loading large files/data<br><br>- Inefficient serialization | - Apply data streaming processing (files, json, etc.)<br>    <br>- Check for unnecessary reference retention (collections, caches, etc.)<br>    <br>- Examine HeapDump        |                                                    |
| Metaspace                          | - Dynamically loading many classes (reflection, proxies)<br>    <br>- ClassLoader leaks                                | - Reduce proxies and reflection<br>    <br>- Configure MaxMetaspaceSize (default unlimited)<br>    <br>- Use Elastic Metaspace (default since Java 16) | - Elastic Metaspace immediately returns unused memory            |
| GC overhead limit exceeded         | - Many long-lived objects<br>    <br>- Objects not collected by GC                                       | - Use G1GC<br>    <br>- Reduce caching<br>    <br>- Use UseStringDeduplication (requires G1GC)                                   | - UseStringDeduplication manages duplicate String instances as one |
| unable to create new native thread | - Exceeding OS thread limit during heavy async/parallel processing<br>    <br>- Insufficient native memory due to large stack size          | - Limit maximum number of threads<br>    <br>- Use Virtual Threads<br>    - Or specify stack size per thread (watch for stack overflow)                   | - Virtual Threads don't burden native stack           |
| Direct buffer memory               | - Using direct buffers in Netty/NIO<br>    <br>- Missing explicit deallocation                                 | - Configure MaxDirectMemorySize (default same as Xmx)<br>    <br>- Reuse buffers                                                        | - Direct memory is not GC target, requires manual deallocation            |

## Pitfalls

### 1. Reserve Buffer Space in Container Memory Settings

Do not set the JVM Heap to 100% of container memory. Reserve space for Native Memory (Thread Stack, Direct Memory, JNI, etc.) and OS overhead by limiting to 60~80%.

```bash
# Wrong configuration - Can cause Container OOMKilled
-Xmx1024m  # Container memory is also 1GB

# Correct configuration
-XX:MaxRAMPercentage=75.0  # Use 75% of container memory for Heap
```

### 2. The Essence of GC overhead limit exceeded

This error occurs not simply because memory is insufficient, but when GC consumes more than 98% of CPU time while recovering less than 2% of the Heap. This happens when there are too many live objects in the Old generation or severe memory fragmentation.

**How G1GC solves this:**
- Divides the heap into regions and reduces fragmentation by compacting live objects into other regions
- Efficiently reclaims memory by collecting regions with highest reclaim efficiency first
- Minimizes fragmentation by placing large objects (Humongous) in separate contiguous regions

```bash
# Enable G1GC (default since Java 9)
-XX:+UseG1GC
```

### 3. Overcome Thread Limits with Virtual Threads

Platform Threads have a 1:1 mapping with OS native threads and are subject to OS thread limits. Virtual Threads are lightweight threads scheduled by the JVM, where multiple Virtual Threads run alternately on a few OS threads, making them efficient for I/O-bound workloads.

```kotlin
// Platform Thread - Subject to OS thread limit
repeat(100_000) {
    Thread.ofPlatform().start { /* task */ }
}

// Virtual Thread - Can create many threads without limit
repeat(100_000) {
    Thread.ofVirtual().start { /* task */ }
}
```

## Practical Application

### 1. JVM Memory Configuration in Container Environment

Stable JVM settings when Kubernetes Pod memory limit is 1GB:

```yaml
# Kubernetes Deployment
resources:
  limits:
    memory: "1Gi"
  requests:
    memory: "1Gi"
```

```bash
# JVM Options
-XX:MaxRAMPercentage=75.0         # Heap is 750MB
-XX:+UseG1GC                      # Use G1GC (default since Java 9)
-XX:MaxGCPauseMillis=200          # Target maximum GC pause time
```

### 2. Analyze Heap Dump for Memory Leaks

When `Java heap space` OOM occurs, analyze the Heap Dump to identify the cause.

```bash
# Auto-generate Heap Dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof
```

Use MAT (Memory Analyzer Tool) to analyze the Heap Dump and identify objects consuming the most memory.

```bash
# Example: Objects continuously accumulating in a Collection
Map<String, User> userCache = new ConcurrentHashMap<>();  // Cache keeps growing
// Solution: Use size-limited cache like Caffeine
```

G1GC is the default GC since Java 9 (Serial GC is chosen when CPU has 1 core)

### 3. Limit Metaspace Size

Metaspace has no size limit by default and can grow indefinitely. Applications that heavily use reflection or dynamic proxies should explicitly limit Metaspace size.

```bash
# Limit Metaspace size
-XX:MaxMetaspaceSize=256m
-XX:MetaspaceSize=128m  # Initial size
```

Since Java 16, Elastic Metaspace is enabled by default and immediately returns unused memory.

### 4. Set Up Monitoring Metrics

Continuously monitor memory usage to prevent OOM in advance.

```kotlin
// Spring Boot Actuator + Micrometer
// Check JVM memory metrics
val heapUsed = meterRegistry.gauge("jvm.memory.used", Tags.of("area", "heap"))
val metaspaceUsed = meterRegistry.gauge("jvm.memory.used", Tags.of("id", "Metaspace"))

// Alert when threshold is exceeded
if (heapUsed!! > maxHeap * 0.9) {
    log.warn("Heap memory usage is over 90%")
}
```

## References

### Related TIL
- [JVM.en.md](./JVM.en.md) - JVM memory structure and the role of each area
- [Garbage Collection.en.md](./Garbage%20Collection.en.md) - GC algorithms and tuning methods
- [Analyze Heap Dump.en.md](./Analyze%20Heap%20Dump.en.md) - Heap Dump analysis using MAT
