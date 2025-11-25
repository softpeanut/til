# Coroutine

## Concept

Coroutine is a **suspendable lightweight thread**.

While regular functions must run to completion once started, Coroutines can be suspended during execution and resumed later. They enable efficient asynchronous operations without directly creating OS threads.

### Key Characteristics

- **Lightweight**: Thousands of coroutines can be created with less memory and performance overhead than threads
- **Suspendable**: Work can be suspended without blocking threads through `suspend` functions
- **Structured Concurrency**: Lifecycle is managed through `CoroutineScope`, and child coroutines are automatically cancelled when the scope terminates

---

## Why Needed?

### Problem to Solve

When handling asynchronous operations, directly managing threads or using callback patterns increases complexity and reduces readability.

### Limitations of Existing Approaches

1. **Thread**: High creation cost and context switching overhead
2. **Callback**: Deep nesting leads to Callback Hell
3. **RxJava/Future**: Complex chaining and steep learning curve

### Value Provided

- **Readability**: Code is written like synchronous code but executes asynchronously
- **Efficiency**: Handles numerous concurrent operations with far fewer resources than threads
- **Safety**: Prevents memory leaks and task omissions through structured concurrency

---

## How It Works

### Core Mechanism

Coroutines transform `suspend` functions into state machines through **CPS (Continuation-Passing Style) transformation**. At compile time, functions are split into multiple stages, saving and restoring state at each suspension point.

### Processing Flow

1. **Suspend function call**: Current state is saved in a `Continuation` object
2. **Suspension**: Control is returned and the thread performs other work
3. **Resumption**: Execution continues from the saved `Continuation`

### Key Components

- **Continuation**: Object containing the state and resumption information of a suspension point
- **Dispatcher**: Determines which thread the coroutine executes on (Main, IO, Default, etc.)
- **CoroutineScope**: Scope that manages the lifecycle of coroutines

```kotlin
// Before compilation
suspend fun fetchData(): String {
    delay(1000)
    return "Data"
}

// After compilation (conceptual)
fun fetchData(continuation: Continuation<String>): Any {
    when (continuation.label) {
        0 -> {
            continuation.label = 1
            return delay(1000, continuation) // Returns COROUTINE_SUSPENDED
        }
        1 -> {
            return "Data"
        }
    }
}
```

---

## Pitfalls

### 1. Never Use GlobalScope

Lifecycle cannot be managed, leading to memory leaks.

```kotlin
// ❌ Bad: GlobalScope
GlobalScope.launch {
    // May continue running even after server terminates
    processBackgroundJob()
}

// ✅ Good: Proper scope management
class OrderService {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    fun processOrder(orderId: String) {
        scope.launch {
            // Automatically cancelled when scope is cancelled
            processOrderInternal(orderId)
        }
    }
    
    fun shutdown() {
        scope.cancel() // Cancels all coroutines on service shutdown
    }
}
```

### 2. Blocking Operations on Wrong Dispatcher

Thread pool gets exhausted and other requests cannot be processed.

```kotlin
// ❌ Bad: Blocking I/O on Dispatchers.Default
@GetMapping("/users/{id}")
suspend fun getUser(@PathVariable id: Long): User {
    // Blocking I/O on Dispatchers.Default exhausts thread pool
    val data = readFromDatabase(id)
    return data
}

// ✅ Good: Use Dispatchers.IO for I/O operations
@GetMapping("/users/{id}")
suspend fun getUser(@PathVariable id: Long): User = withContext(Dispatchers.IO) {
    // I/O operations run on Dispatchers.IO
    readFromDatabase(id)
}
```

### 3. Calling Suspend Functions from Regular Functions

`suspend` functions can only be called from within a coroutine scope or another `suspend` function.

```kotlin
// ❌ Bad: Calling suspend from regular function
fun loadData() {
    delay(1000) // Compilation error
}

// ✅ Good: Proper suspend function
suspend fun loadData() {
    delay(1000)
}

// Or
fun loadData() {
    CoroutineScope(Dispatchers.Main).launch {
        delay(1000)
    }
}
```

### Best Practices

- Use appropriate scopes (`viewModelScope`, `lifecycleScope`, etc.)
- Execute I/O operations on `Dispatchers.IO`
- Execute CPU-intensive operations on `Dispatchers.Default`
- Follow Structured Concurrency principles

---

## Practical Application

### Basic Usage

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        delay(1000L) // Suspends for 1 second (not blocking thread)
        println("World!")
    }
    println("Hello,")
}

// Output
// Hello,
// World!
```

### Common Usage Patterns

#### Pattern 1: Parallel Execution

When executing multiple async operations simultaneously and waiting for all results

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { fetchUser() }
    val posts = async { fetchPosts() }
    val notifications = async { fetchNotifications() }
    
    Dashboard(
        user = user.await(),
        posts = posts.await(),
        notifications = notifications.await()
    )
}
```

#### Pattern 2: Cancellable Operations and Resource Cleanup

Coroutines check for cancellation at suspension points and clean up resources

```kotlin
// ❌ Regular function - Not cancellable
fun processLargeFile(filePath: String): Int {
    var totalBytes = 0
    File(filePath).inputStream().use { input ->
        val buffer = ByteArray(8192)
        while (true) {
            val bytesRead = input.read(buffer) // Blocking - Cannot stop while reading chunk
            if (bytesRead == -1) break
            totalBytes += bytesRead
            // Cannot be cancelled - Only thread termination possible
        }
    }
    return totalBytes
}

// ❌ Coroutine but no cancellation points
suspend fun processLargeFile(filePath: String): Int = withContext(Dispatchers.IO) {
    var totalBytes = 0
    File(filePath).inputStream().use { input ->
        val buffer = ByteArray(8192)
        while (true) {
            val bytesRead = input.read(buffer) // Blocking - Still not cancellable
            if (bytesRead == -1) break
            totalBytes += bytesRead
        }
    }
    totalBytes
}

// ✅ Coroutine - Cancellable
suspend fun processLargeFile(filePath: String): Int = withContext(Dispatchers.IO) {
    var totalBytes = 0
    File(filePath).inputStream().use { input ->
        val buffer = ByteArray(8192)
        while (true) {
            ensureActive() // Check cancellation per chunk - Throws CancellationException if cancelled
            val bytesRead = input.read(buffer)
            if (bytesRead == -1) break
            totalBytes += bytesRead
        }
    }
    totalBytes
} // use block automatically calls close() on cancellation

// Usage example
class FileService {
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    
    fun startProcessing(filePath: String): Job {
        return scope.launch {
            try {
                val totalBytes = processLargeFile(filePath)
                println("Processed $totalBytes bytes")
            } catch (e: CancellationException) {
                println("Processing cancelled - resources cleaned up")
                throw e // Always rethrow CancellationException
            }
        }
    }
    
    // Example: Auto-cancel after 5 seconds
    fun startWithTimeout(filePath: String) {
        scope.launch {
            val job = launch { processLargeFile(filePath) }
            delay(5000)
            job.cancel() // Cancel after 5 seconds - Detected at ensureActive()
        }
    }
}
```

#### Pattern 3: Structured Concurrency for Exception Propagation

Handle exceptions so that child coroutine failures propagate to parent

```kotlin
// SupervisorJob: Other children continue even if one child fails
class NotificationService {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    suspend fun sendNotifications(userIds: List<String>) = coroutineScope {
        userIds.map { userId ->
            async {
                try {
                    sendNotification(userId)
                } catch (e: Exception) {
                    logger.error("Failed to send notification to $userId", e)
                    null // Other notifications continue even on failure
                }
            }
        }.awaitAll()
    }
}
```

#### Pattern 4: Understanding Blocking/Non-blocking Combinations

Combinations of asynchronous execution patterns

```kotlin
// Non-blocking + Synchronous
runBlocking {
    delay(100)        // (1) Non-blocking
    test()            // (2) Non-blocking
    println("Done")   // (3) Executes after 1, 2 complete
}
suspend fun test() { delay(200) }

// Non-blocking + Asynchronous
runBlocking {
    val result1 = async { delay(100) }  // (1) Start async
    val result2 = async { delay(200) }  // (2) Start async
    println("Done")                     // (3) Executes first
    result1.await()                     // (4) Wait until 1 completes
    result2.await()                     // (5) Wait until 2 completes
}

// Blocking + Asynchronous
runBlocking {
    val result = launch {
        withContext(Dispatchers.Default) { 
            Thread.sleep(100)  // Blocking but on separate thread
        }
    }
    println("Done")    // (2) Can execute first
    result.join()      // (3) Wait until result completes
}
```

### Before/After Comparison

```kotlin
// ❌ Callback approach
fun fetchData(callback: (Result<Data>) -> Unit) {
    api.getData(object : Callback {
        override fun onSuccess(data: Data) {
            callback(Result.success(data))
        }
        override fun onError(e: Exception) {
            callback(Result.failure(e))
        }
    })
}

// ✅ Coroutine approach
suspend fun fetchData(): Data {
    return api.getData() // Concise and intuitive
}
```

---

## References

### Official Documentation
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)

### Recommended Articles
- [Understanding Kotlin Coroutines](https://www.silica.io/understanding-kotlin-coroutines/5/) - Detailed explanation of Coroutine internals

### Related TIL
- [JVM](../JVM/JVM.md)
