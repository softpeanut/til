# Coroutine vs Thread vs Virtual Thread

## Concept

**Coroutine**, **OS Thread**, and **Virtual Thread** are all methods for handling concurrency, but they differ significantly in how they work and their resource efficiency.

| Category | OS Thread | Virtual Thread | Coroutine |
|----------|-----------|----------------|-----------|
| **Introduced** | Operating System | Java 21 (2023) | Kotlin 1.3 (2018) |
| **Managed By** | OS Kernel | JVM | Kotlin Runtime |
| **Memory/Unit** | ~1MB (fixed stack) | ~1KB initial, grows dynamically | ~few hundred bytes~few KB (state only) |
| **Creation Cost** | High (~1ms) | Low (~1μs) | Very Low (~100ns) |
| **Scheduling** | OS Preemptive | Blocking-detection based (implicit) | Cooperative (suspend points) |
| **Blocking I/O** | Occupies thread | Auto-unmount | Dispatcher switch or async API |

---

## Why Needed

### Limitations of OS Threads

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OS Thread Model                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   1 Request = 1 Thread                                                  │
│                                                                         │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│   │ Thread 1 │ │ Thread 2 │ │ Thread 3 │ │ Thread N │   ...             │
│   │  (~1MB)  │ │  (~1MB)  │ │  (~1MB)  │ │  (~1MB)  │                   │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                   │
│        │            │            │            │                         │
│        ▼            ▼            ▼            ▼                         │
│   ┌─────────────────────────────────────────────────┐                   │
│   │              OS Kernel Scheduler                 │                   │
│   │       (Context switching cost: ~1-10μs)          │                   │
│   └─────────────────────────────────────────────────┘                   │
│                                                                         │
│   Problems:                                                             │
│   • 10,000 requests = 10GB memory required                              │
│   • Context switching overhead                                          │
│   • Thread resources wasted during I/O wait                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```kotlin
// ❌ Limitations of Thread-per-request model
class TraditionalServer {
    private val executor = Executors.newFixedThreadPool(200) // Max 200
    
    fun handleRequest(request: Request) {
        executor.submit {
            // Thread is blocked during I/O operations
            val user = userRepository.findById(request.userId)  // ~50ms wait
            val orders = orderRepository.findByUser(user)       // ~100ms wait
            val payments = paymentClient.getHistory(user)       // ~200ms wait
            
            // Thread occupied for 350ms (almost no actual CPU work)
            process(user, orders, payments)
        }
    }
}

// 1000 concurrent requests → Only 200 processed, 800 wait in queue
```

### Solution Approaches

| Approach | Virtual Thread | Coroutine |
|----------|----------------|-----------|
| **Philosophy** | Use existing blocking code as-is | Convert to async code |
| **Learning Cost** | Low (keep existing code) | Medium (understand suspend functions) |
| **Control Level** | Automatic (JVM managed) | Manual (developer chooses Dispatcher) |
| **Ecosystem** | Requires Java 21+ | Kotlin only |

---

## How It Works

### OS Thread

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OS Thread Scheduling                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Thread A                    Thread B                                  │
│   ┌───────┐                   ┌───────┐                                 │
│   │Running│ ←─ Time slice     │ Wait  │                                 │
│   └───┬───┘    expired        └───────┘                                 │
│       │                                                                 │
│       ▼ Context switch (~1-10μs)                                        │
│   ┌───────┐                   ┌───────┐                                 │
│   │ Wait  │                   │Running│ ←─ Scheduler selects            │
│   └───────┘                   └───┬───┘                                 │
│                                   │                                     │
│   • Preemptive: OS forcibly switches threads                            │
│   • Context save: registers, stack pointer, PC, etc.                    │
│   • Kernel mode transition required                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Virtual Thread (Java 21+)

Virtual Thread scheduling is **neither preemptive nor purely cooperative, but "blocking-detection based"**.

| Scheduling | OS Thread | Virtual Thread | Coroutine |
|------------|-----------|----------------|-----------|  
| **Method** | Preemptive | Blocking-detection based | Cooperative |
| **Switch Timing** | OS forces via time slice | JVM detects blocking API calls | Explicit suspend points |
| **Developer Control** | None | None (implicit) | Yes (explicit) |
| **Yield Required** | No | No | Yes (suspend) |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Virtual Thread Architecture                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Virtual Thread 1   Virtual Thread 2   Virtual Thread 3   ...          │
│   ┌────────────┐     ┌────────────┐     ┌────────────┐                  │
│   │ Task: API  │     │ Task: DB   │     │ Task: File │                  │
│   │ (~KB stack)│     │ (~KB stack)│     │ (~KB stack)│                  │
│   └─────┬──────┘     └─────┬──────┘     └─────┬──────┘                  │
│         │                  │                  │                         │
│         └────────────┬─────┴──────────────────┘                         │
│                      ▼                                                  │
│   ┌─────────────────────────────────────────────────────┐               │
│   │              Carrier Thread Pool                     │               │
│   │        (Platform Threads, ~CPU core count)           │               │
│   │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                │               │
│   │  │  PT1 │ │  PT2 │ │  PT3 │ │  PT4 │                │               │
│   │  └──────┘ └──────┘ └──────┘ └──────┘                │               │
│   └─────────────────────────────────────────────────────┘               │
│                                                                         │
│   Blocking-detection based scheduling:                                  │
│   1. JVM detects blocking API calls (sleep, I/O, lock, etc.)            │
│   2. Virtual Thread automatically "unmounts" from Carrier Thread        │
│   3. Carrier Thread runs other Virtual Threads                          │
│   4. "Mounts" again when blocking completes (may be different Carrier)  │
│                                                                         │
│   ❓ Difference from Cooperative:                                        │
│   • Coroutine: Developer explicitly yields with suspend keyword         │
│   • Virtual Thread: JVM implicitly detects blocking APIs and yields     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
// Java 21 Virtual Thread
public class VirtualThreadExample {
    public static void main(String[] args) throws Exception {
        // Can create 1 million Virtual Threads
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 1_000_000).forEach(i -> {
                executor.submit(() -> {
                    // Use blocking code as-is - JVM optimizes automatically
                    Thread.sleep(Duration.ofSeconds(1));
                    System.out.println("Task " + i);
                    return i;
                });
            });
        }
    }
}

// Spring Boot 3.2+
@RestController
public class UserController {
    
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        // Use existing blocking code as-is
        // Virtual Thread auto-unmounts on blocking
        User user = userRepository.findById(id);  // Blocking OK
        List<Order> orders = orderClient.getOrders(user);  // Blocking OK
        return user;
    }
}

// application.yml
// spring.threads.virtual.enabled=true
```

### Coroutine

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Coroutine Architecture                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Coroutine 1         Coroutine 2         Coroutine 3                   │
│   ┌───────────┐       ┌───────────┐       ┌───────────┐                 │
│   │  suspend  │       │  suspend  │       │  suspend  │                 │
│   │  function │       │  function │       │  function │                 │
│   │(state mch)│       │(state mch)│       │(state mch)│                 │
│   └─────┬─────┘       └─────┬─────┘       └─────┬─────┘                 │
│         │                   │                   │                       │
│         └─────────────┬─────┴───────────────────┘                       │
│                       ▼                                                 │
│   ┌─────────────────────────────────────────────────────┐               │
│   │                  Dispatcher                          │               │
│   │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │               │
│   │  │ Dispatchers  │  │ Dispatchers  │  │ Dispatchers│ │               │
│   │  │   .Default   │  │     .IO      │  │   .Main    │ │               │
│   │  │  (CPU work)  │  │  (I/O work)  │  │ (UI work)  │ │               │
│   │  └──────────────┘  └──────────────┘  └────────────┘ │               │
│   └─────────────────────────────────────────────────────┘               │
│                                                                         │
│   At suspend points:                                                    │
│   1. Save current state to Continuation object (few hundred bytes)      │
│   2. Return thread (other coroutines can run)                           │
│   3. Resume from Continuation when work completes                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```kotlin
// Coroutine - Explicit suspend
@RestController
class UserController(
    private val userService: UserService
) {
    @GetMapping("/user/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        // suspend function - executes asynchronously
        return userService.findById(id)
    }
}

@Service
class UserService(
    private val userRepository: UserRepository,
    private val orderClient: OrderClient
) {
    suspend fun findById(id: Long): User {
        // Optimize with parallel execution
        return coroutineScope {
            val user = async { userRepository.findById(id) }
            val orders = async { orderClient.getOrders(id) }
            
            user.await().apply { 
                this.orders = orders.await() 
            }
        }
    }
}
```

---

## Pitfalls

### 1. Be Careful with synchronized in Virtual Threads

```java
// ❌ Blocking in synchronized block - Carrier Thread gets pinned
public class BadExample {
    private final Object lock = new Object();
    
    public void process() {
        synchronized (lock) {
            // Virtual Thread gets "pinned" to Carrier Thread
            blockingCall();  // Occupies Carrier Thread - other VTs can't run
        }
    }
}

// ✅ Use ReentrantLock
public class GoodExample {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void process() {
        lock.lock();
        try {
            blockingCall();  // Can release Carrier Thread on blocking
        } finally {
            lock.unlock();
        }
    }
}
```

### 2. Be Careful with Blocking Code in Coroutines

When using blocking code in Coroutine, Dispatcher switch is required, but using **async libraries** allows direct usage without switching.

```kotlin
// ❌ Wrong - Blocking on Dispatchers.Default
suspend fun processData(): Data {
    // Starves Default Dispatcher threads
    val result = blockingDatabaseCall()  // Blocking!
    return result
}

// ✅ Method 1: Switch Dispatcher with withContext (when using blocking API)
suspend fun processData(): Data = withContext(Dispatchers.IO) {
    // Run blocking work on I/O Dispatcher
    jdbcTemplate.query("SELECT * FROM users")  // Blocking JDBC
}

// ✅ Method 2: Use async libraries (recommended - no Dispatcher switch needed)
suspend fun processData(): Data {
    // Use non-blocking APIs like R2DBC, WebClient
    // Directly supports suspend functions, no Dispatcher switch needed
    return r2dbcRepository.findById(id).awaitSingle()
}
```

#### Async vs Blocking Library Comparison

| Category | Blocking (Traditional) | Async (Reactive) |
|----------|------------------------|------------------|
| **DB** | JDBC, JPA/Hibernate | R2DBC, Mongo Reactive |
| **HTTP Client** | RestTemplate, OkHttp | WebClient, Ktor Client |
| **File I/O** | java.io.* | kotlinx-coroutines-io |
| **Dispatcher Switch** | Required (Dispatchers.IO) | Not Required |
| **Thread Efficiency** | Occupied during blocking | Fully non-blocking |

```kotlin
// Example: WebClient (Spring WebFlux) - No Dispatcher switch needed
@Service
class UserService(
    private val webClient: WebClient
) {
    // Async API can be called directly in suspend function
    suspend fun fetchUser(id: Long): User {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User::class.java)
            .awaitSingle()  // Mono -> suspend conversion
    }
}

// Example: R2DBC - No Dispatcher switch needed
@Repository
interface UserRepository : CoroutineCrudRepository<User, Long> {
    // Auto-generated as suspend function
    suspend fun findByEmail(email: String): User?
}

@Service
class UserService(
    private val userRepository: UserRepository
) {
    suspend fun findUser(email: String): User? {
        // No withContext needed - already non-blocking
        return userRepository.findByEmail(email)
    }
}
```

### 3. Be Careful with ThreadLocal

```kotlin
// Both Virtual Thread / Coroutine need caution

// ❌ ThreadLocal - Problems with reused threads
val threadLocal = ThreadLocal<User>()

suspend fun process() {
    threadLocal.set(currentUser)
    delay(100)  // May resume on different thread
    val user = threadLocal.get()  // May be different value or null
}

// ✅ Coroutine - Use CoroutineContext
val userContext = object : CoroutineContext.Key<UserElement> {}
class UserElement(val user: User) : CoroutineContext.Element {
    override val key = userContext
}

suspend fun process() {
    val user = coroutineContext[userContext]?.user
}

// ✅ Virtual Thread - ScopedValue (Java 21 Preview)
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

void process() {
    ScopedValue.runWhere(CURRENT_USER, user, () -> {
        // Can safely access user
    });
}
```

### 4. When to Choose What?

| Situation | Recommended | Reason |
|-----------|-------------|--------|
| Existing Java blocking code | Virtual Thread | Minimal code changes |
| New Kotlin project | Coroutine | Finer control, language integration |
| CPU-intensive work | Regular Thread | Context switching becomes overhead |
| Very many concurrent connections | Coroutine | Best memory efficiency |
| Simple web server | Virtual Thread | Apply with just configuration |

---

## Practical Application

### Performance Comparison

```kotlin
// Simulating 10,000 concurrent tasks

// OS Thread
fun threadTest() {
    val executor = Executors.newFixedThreadPool(200)
    val latch = CountDownLatch(10_000)
    
    repeat(10_000) {
        executor.submit {
            Thread.sleep(100)  // Simulate I/O wait
            latch.countDown()
        }
    }
    latch.await()  // ~5 seconds (10000/200 * 100ms)
}

// Virtual Thread (Java 21)
fun virtualThreadTest() {
    val executor = Executors.newVirtualThreadPerTaskExecutor()
    val latch = CountDownLatch(10_000)
    
    repeat(10_000) {
        executor.submit {
            Thread.sleep(100)  // Blocking but efficient
            latch.countDown()
        }
    }
    latch.await()  // ~120-200ms (including scheduling overhead)
}

// Coroutine
fun coroutineTest() = runBlocking {
    val jobs = List(10_000) {
        launch {
            delay(100)  // Non-blocking wait
        }
    }
    jobs.joinAll()  // ~105-150ms (fastest)
}
```

### Memory Usage Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│        Memory for 10,000 Concurrent Tasks (simple tasks)                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   OS Thread      │████████████████████████████████████│  ~10GB          │
│                  │ (1MB × 10,000, fixed stack)        │                 │
│                                                                         │
│   Virtual Thread │████                                │  ~50-100MB      │
│                  │ (initial ~1KB, grows with depth)  │                 │
│                                                                         │
│   Coroutine      │██                                  │  ~10-20MB       │
│                  │ (few hundred bytes~KB, state only)│  ✅ Most efficient│
│                                                                         │
│   * Virtual Thread memory increases with deeper call stacks             │
│   * Coroutine uses CPS transformation, stores only state - most efficient│
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Spring Boot Integration

```kotlin
// Spring Boot 3.2+ with Virtual Thread
// application.yml
// spring:
//   threads:
//     virtual:
//       enabled: true

// Use existing code as-is - Virtual Thread applied automatically
@Service
class UserService(
    private val userRepository: UserRepository
) {
    fun findById(id: Long): User {
        return userRepository.findById(id).orElseThrow()
    }
}
```

```kotlin
// Spring WebFlux + Coroutine
@RestController
class UserController(
    private val userService: UserService
) {
    @GetMapping("/users/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        return userService.findById(id)
    }
    
    @GetMapping("/users/{id}/dashboard")
    suspend fun getDashboard(@PathVariable id: Long): Dashboard = coroutineScope {
        val user = async { userService.findById(id) }
        val orders = async { orderService.findByUserId(id) }
        val recommendations = async { recommendationService.getFor(id) }
        
        Dashboard(
            user = user.await(),
            orders = orders.await(),
            recommendations = recommendations.await()
        )
    }
}
```

### Before/After Comparison

```kotlin
// ❌ Before: Traditional Thread Pool approach
@Configuration
class ThreadPoolConfig {
    @Bean
    fun taskExecutor(): TaskExecutor {
        val executor = ThreadPoolTaskExecutor()
        executor.corePoolSize = 50
        executor.maxPoolSize = 200
        executor.queueCapacity = 1000
        // Requests beyond 200 wait in queue
        return executor
    }
}

// ✅ After: Virtual Thread (just change config)
// application.yml
// spring.threads.virtual.enabled: true
// No code changes needed!

// ✅ After: Coroutine (finer control)
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryClient: InventoryClient,
    private val paymentClient: PaymentClient
) {
    suspend fun processOrder(request: OrderRequest): OrderResult = coroutineScope {
        // Parallel validation
        val inventoryCheck = async { inventoryClient.check(request.items) }
        val userValidation = async { validateUser(request.userId) }
        
        inventoryCheck.await()
        userValidation.await()
        
        // Sequential processing
        val payment = paymentClient.process(request.payment)
        orderRepository.save(Order.from(request, payment))
    }
}
```

---

## Coroutine + Virtual Thread Combination

### Why Combine?

Coroutine and Virtual Thread are **complementary**. Replacing Coroutine's `Dispatchers.IO` with Virtual Thread maximizes blocking I/O handling efficiency.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Existing Coroutine + Thread Pool                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Coroutine 1 ──► withContext(Dispatchers.IO) ──► Thread Pool (limited) │
│   Coroutine 2 ──► withContext(Dispatchers.IO) ──► max(64, CPU cores)    │
│   ...                                                                   │
│   Coroutine N ──► withContext(Dispatchers.IO) ──► ⚠️ Thread exhaustion  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    Coroutine + Virtual Thread                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Coroutine 1 ──► withContext(VT Dispatcher) ──► Virtual Thread 1       │
│   Coroutine 2 ──► withContext(VT Dispatcher) ──► Virtual Thread 2       │
│   ...                                                                   │
│   Coroutine N ──► withContext(VT Dispatcher) ──► Virtual Thread N       │
│                                                    ✅ Unlimited scaling  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Virtual Thread Dispatcher Setup

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.Executors

// Create Virtual Thread based Dispatcher (manage as singleton)
object VirtualThreadDispatcher {
    val dispatcher: CoroutineDispatcher by lazy {
        Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
    }
}

// Or define as extension property (lazy to create only once)
@OptIn(ExperimentalCoroutinesApi::class)
val Dispatchers.VT: CoroutineDispatcher by lazy {
    Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
}
```

### Practical Application Example

```kotlin
// Old approach: Using Dispatchers.IO
suspend fun fetchDataOld(): Data = withContext(Dispatchers.IO) {
    // Runs on max(64, CPU cores) thread pool - bottleneck with many concurrent requests
    blockingHttpCall()
}

// Improved approach: Using Virtual Thread
suspend fun fetchDataNew(): Data = withContext(VirtualThreadDispatcher.dispatcher) {
    // Runs on Virtual Thread - efficient even when blocking
    blockingHttpCall()
}
```

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val legacyPaymentClient: LegacyPaymentClient,  // Blocking API
    private val inventoryClient: InventoryClient           // Blocking API
) {
    private val vtDispatcher = Executors.newVirtualThreadPerTaskExecutor()
        .asCoroutineDispatcher()
    
    suspend fun processOrder(request: OrderRequest): OrderResult = coroutineScope {
        // Parallel execution + Virtual Thread for blocking handling
        val payment = async(vtDispatcher) {
            legacyPaymentClient.process(request.payment)  // Blocking OK
        }
        
        val inventory = async(vtDispatcher) {
            inventoryClient.reserve(request.items)  // Blocking OK
        }
        
        // Maintain Coroutine's structured concurrency
        val paymentResult = payment.await()
        val inventoryResult = inventory.await()
        
        // Non-blocking work uses default Dispatcher
        orderRepository.save(Order.from(request, paymentResult, inventoryResult))
    }
}
```

### Spring Boot 3.2+ Integration

```kotlin
// application.yml
// spring:
//   threads:
//     virtual:
//       enabled: true

@Configuration
class CoroutineConfig {
    
    // Virtual Thread based Dispatcher Bean
    @Bean
    fun virtualThreadDispatcher(): CoroutineDispatcher {
        return Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
    }
}

@Service
class UserService(
    private val userRepository: UserRepository,
    private val legacyUserClient: LegacyUserClient,
    @Qualifier("virtualThreadDispatcher")
    private val vtDispatcher: CoroutineDispatcher
) {
    suspend fun enrichUser(id: Long): EnrichedUser = coroutineScope {
        // DB query - suspend function (non-blocking)
        val user = userRepository.findById(id)
        
        // Legacy API call - blocking so use Virtual Thread
        val legacyData = async(vtDispatcher) {
            legacyUserClient.fetchAdditionalData(id)
        }
        
        EnrichedUser(user, legacyData.await())
    }
}
```

### When to Use the Combination?

| Situation | Recommended Approach |
|-----------|---------------------|
| Pure non-blocking code | `Dispatchers.Default` or suspend functions |
| Blocking I/O (DB, HTTP) | `withContext(vtDispatcher)` |
| CPU-intensive work | `Dispatchers.Default` |
| Legacy blocking libraries | Virtual Thread Dispatcher |
| File I/O | Virtual Thread Dispatcher |

```kotlin
// Optimal combination example
suspend fun processRequest(request: Request): Response = coroutineScope {
    // 1. Non-blocking - Default Dispatcher
    val validated = validateRequest(request)
    
    // 2. Blocking I/O - Virtual Thread
    val externalData = withContext(vtDispatcher) {
        legacyClient.fetch(validated.id)  // Blocking API
    }
    
    // 3. CPU computation - Default Dispatcher
    val processed = withContext(Dispatchers.Default) {
        heavyComputation(externalData)
    }
    
    // 4. Non-blocking save
    repository.save(processed)
}
```

### Pitfalls

```kotlin
// ❌ Don't create Virtual Thread Dispatcher every time
suspend fun badExample() {
    withContext(Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()) {
        // Creates new Executor every time - resource waste
    }
}

// ✅ Reuse as singleton
object Dispatchers {
    val VT: CoroutineDispatcher by lazy {
        Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
    }
}

suspend fun goodExample() {
    withContext(Dispatchers.VT) {
        // Executor reused
    }
}
```

```kotlin
// ❌ Be careful with close() timing
class MyService : AutoCloseable {
    private val vtExecutor = Executors.newVirtualThreadPerTaskExecutor()
    private val vtDispatcher = vtExecutor.asCoroutineDispatcher()
    
    override fun close() {
        vtDispatcher.close()  // Close Dispatcher
        vtExecutor.close()    // Close Executor
    }
}
```

---

## Comparison Summary

### Characteristics Comparison

| Characteristic | OS Thread | Virtual Thread | Coroutine |
|----------------|-----------|----------------|-----------|
| Creation Cost | High (~1ms) | Low (~1μs) | Very Low (~100ns) |
| Memory | ~1MB/thread | ~10KB/thread | ~1KB/coroutine |
| Max Count | ~thousands | ~millions | ~millions |
| Blocking Handling | Occupies thread | Auto-unmount | Dispatcher switch |
| Learning Curve | Low | Low | Medium |
| Language Support | All languages | Java 21+ | Kotlin |
| Code Style | Blocking | Blocking | suspend |

### Selection Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Technology Selection Flowchart                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Q: What language?                                                     │
│   ├── Java → Q: Is Java version 21 or higher?                           │
│   │          ├── Yes → Virtual Thread recommended                       │
│   │          └── No → CompletableFuture or RxJava                       │
│   │                                                                     │
│   └── Kotlin → Q: Need fine-grained async control?                      │
│                ├── Yes → Coroutine                                      │
│                └── No → Virtual Thread also possible (JVM 21+)          │
│                                                                         │
│   Q: Lots of existing blocking code?                                    │
│   ├── Yes → Virtual Thread (minimize code changes)                      │
│   └── No → Coroutine (more efficient)                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## References

### Official Documentation
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Spring Framework Virtual Threads](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-task-executor-virtual-threads)

### Recommended Articles
- [Virtual Threads: A Game-Changer for Concurrency](https://blogs.oracle.com/javamagazine/post/java-virtual-threads)
- [Kotlin Coroutines vs Java Virtual Threads](https://kt.academy/article/coroutines-vs-virtual-threads)

### Related TIL
- [Coroutine](./Coroutine.en.md)
- [JVM](../JVM/JVM.en.md)
