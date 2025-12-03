# Singleton Pattern

## Concept

The Singleton Pattern **ensures that a class has only one instance** and provides a **global access point** to that instance.

As one of the GoF creational patterns, it's used when only one object is needed throughout the entire application. The key principles are "**unique instance**" and "**global access**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Single Instance** | Only one instance of the class exists |
| **Global Access** | Same instance accessible from anywhere |
| **Lazy Initialization** | Instance created when needed (optional) |
| **Creation Control** | Cannot create instance with new from outside |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Multiple instances created, inconsistent state occurs
class DatabaseConnection {
    private var connectionPool: List<Connection> = mutableListOf()
    
    fun getConnection(): Connection {
        // Creating new instance each time distributes connection pools
        return connectionPool.firstOrNull() ?: createNewConnection()
    }
}

// Each different instance - connection pool not shared
val db1 = DatabaseConnection()
val db2 = DatabaseConnection()
val db3 = DatabaseConnection()
```

- Objects that should share resources are created multiple times
- Configuration may differ between instances
- Memory waste and state inconsistency occurs
- Global state management becomes difficult

### Value Provided

- **Resource Savings**: Memory efficient with only one instance
- **State Consistency**: All clients share the same state
- **Global Access**: Easy access from anywhere
- **Creation Control**: Complete control over when and how instance is created

---

## How It Works

### Structure

```
┌─────────────────────────────────────────────────────────────┐
│                         Singleton                           │
├─────────────────────────────────────────────────────────────┤
│  - instance: Singleton          ← Stores unique instance    │
│  - Singleton()                  ← private constructor       │
├─────────────────────────────────────────────────────────────┤
│  + getInstance(): Singleton     ← Global access point       │
│  + businessLogic()              ← Business logic            │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ calls
                              ▼
┌──────────┐  getInstance()  ┌──────────┐  getInstance()  ┌──────────┐
│ Client A │ ──────────────► │          │ ◄────────────── │ Client B │
└──────────┘                 │ instance │                 └──────────┘
                             │          │
┌──────────┐  getInstance()  │          │  getInstance()  ┌──────────┐
│ Client C │ ──────────────► │          │ ◄────────────── │ Client D │
└──────────┘                 └──────────┘                 └──────────┘
                             Returns same instance
```

### Java Basic Implementation

```java
// Most basic Singleton (Not thread-safe)
public class BasicSingleton {
    private static BasicSingleton instance;
    
    // private constructor - cannot use new from outside
    private BasicSingleton() {}
    
    public static BasicSingleton getInstance() {
        if (instance == null) {
            instance = new BasicSingleton();
        }
        return instance;
    }
}

// Thread-safe Singleton (synchronized)
public class ThreadSafeSingleton {
    private static ThreadSafeSingleton instance;
    
    private ThreadSafeSingleton() {}
    
    // Synchronized - may cause performance degradation
    public static synchronized ThreadSafeSingleton getInstance() {
        if (instance == null) {
            instance = new ThreadSafeSingleton();
        }
        return instance;
    }
}

// Double-Checked Locking (DCL)
public class DCLSingleton {
    // volatile: ensures memory visibility
    private static volatile DCLSingleton instance;
    
    private DCLSingleton() {}
    
    public static DCLSingleton getInstance() {
        if (instance == null) {                    // 1st check (without lock)
            synchronized (DCLSingleton.class) {
                if (instance == null) {            // 2nd check (inside lock)
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh Singleton (Recommended) - Initialization-on-demand holder
public class BillPughSingleton {
    private BillPughSingleton() {}
    
    // Static inner class is not loaded when outer class is loaded
    // Loaded when getInstance() is called, creating instance
    private static class SingletonHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

// Enum Singleton (Safest) - Recommended by Joshua Bloch
public enum EnumSingleton {
    INSTANCE;
    
    public void businessLogic() {
        // Business logic
    }
}
```

### Kotlin Implementation

```kotlin
// Kotlin object - Language-level Singleton support
object DatabaseConfig {
    var host: String = "localhost"
    var port: Int = 5432
    var database: String = "mydb"
    
    fun getConnectionString(): String {
        return "jdbc:postgresql://$host:$port/$database"
    }
}

// Usage - No separate getInstance() call needed
val connectionString = DatabaseConfig.getConnectionString()
DatabaseConfig.host = "production-db.example.com"
```

```kotlin
// When lazy initialization is needed
class AppConfig private constructor() {
    var appName: String = ""
    var version: String = ""
    var debug: Boolean = false
    
    companion object {
        // by lazy is thread-safe and initializes on first access
        val instance: AppConfig by lazy { 
            AppConfig().apply {
                // Initialization logic
                loadFromFile()
            }
        }
    }
    
    private fun loadFromFile() {
        // Load from config file
        appName = "MyApp"
        version = "1.0.0"
        debug = false
    }
}

// Usage
val config = AppConfig.instance
println(config.appName)  // MyApp
```

---

## Pitfalls

### 1. Synchronization Required in Multi-threaded Environment

```kotlin
// ❌ Race Condition possible
class UnsafeSingleton {
    companion object {
        private var instance: UnsafeSingleton? = null
        
        fun getInstance(): UnsafeSingleton {
            if (instance == null) {  // Thread A, B can enter simultaneously
                instance = UnsafeSingleton()  // Two instances may be created
            }
            return instance!!
        }
    }
}

// ✅ Use Kotlin object - Thread-safe guaranteed
object SafeSingleton {
    // Kotlin object is thread-safe at JVM level
}

// ✅ Or use lazy
class SafeLazySingleton {
    companion object {
        val instance: SafeLazySingleton by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
            SafeLazySingleton()
        }
    }
}
```

### 2. Difficult to Test

```kotlin
// ❌ Direct Singleton usage - Cannot mock
class OrderService {
    fun createOrder(order: Order) {
        // Direct Singleton reference - connects to DB in tests
        DatabaseConnection.instance.execute("INSERT INTO orders ...")
    }
}

// ✅ Solve with Dependency Injection
interface DatabaseConnection {
    fun execute(sql: String)
}

object ProductionDatabaseConnection : DatabaseConnection {
    override fun execute(sql: String) {
        // Actual DB connection
    }
}

class OrderService(
    private val database: DatabaseConnection  // Injected
) {
    fun createOrder(order: Order) {
        database.execute("INSERT INTO orders ...")
    }
}

// Test
class OrderServiceTest {
    @Test
    fun `create order test`() {
        val mockDatabase = mockk<DatabaseConnection>()
        val service = OrderService(mockDatabase)
        
        service.createOrder(testOrder)
        
        verify { mockDatabase.execute(any()) }
    }
}
```

### 3. Hidden Dependencies Due to Global State

```kotlin
// ❌ Hidden dependencies - Cannot identify dependencies from class alone
class PaymentProcessor {
    fun process(payment: Payment) {
        val config = AppConfig.instance  // Hidden dependency
        val logger = Logger.instance     // Hidden dependency
        // ...
    }
}

// ✅ Explicit dependencies
class PaymentProcessor(
    private val config: AppConfig,
    private val logger: Logger
) {
    fun process(payment: Payment) {
        // Dependencies are explicit
    }
}
```

### 4. Possible Single Responsibility Principle Violation

Singleton has two responsibilities: "instance creation management" and "business logic".

```kotlin
// ❌ Two responsibilities mixed
object OrderManager {
    // Responsibility 1: Instance management (implicit via object keyword)
    
    // Responsibility 2: Business logic
    fun createOrder() { }
    fun cancelOrder() { }
    fun processPayment() { }
}

// ✅ In Spring, framework manages lifecycle
@Service  // Spring manages as Singleton
class OrderManager {
    // Only responsible for business logic
    fun createOrder() { }
    fun cancelOrder() { }
}
```

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Configuration Management | Application settings, environment variables |
| Logging | Shared log instance |
| Cache | Global cache manager |
| Thread Pool | Connection pool, thread pool |
| Registry | Object registry, service locator |

### Logging System Example

```kotlin
object Logger {
    enum class Level { DEBUG, INFO, WARN, ERROR }
    
    private var minLevel: Level = Level.INFO
    private val dateFormat = SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
    
    fun setLevel(level: Level) {
        minLevel = level
    }
    
    fun debug(message: String) = log(Level.DEBUG, message)
    fun info(message: String) = log(Level.INFO, message)
    fun warn(message: String) = log(Level.WARN, message)
    fun error(message: String, throwable: Throwable? = null) {
        log(Level.ERROR, message)
        throwable?.printStackTrace()
    }
    
    private fun log(level: Level, message: String) {
        if (level.ordinal >= minLevel.ordinal) {
            val timestamp = dateFormat.format(Date())
            val threadName = Thread.currentThread().name
            println("[$timestamp] [$threadName] ${level.name}: $message")
        }
    }
}

// Use same logger from anywhere
class UserService {
    fun createUser(name: String) {
        Logger.info("Creating user: $name")
        // User creation logic
        Logger.debug("User created successfully")
    }
}

class OrderService {
    fun processOrder(orderId: String) {
        Logger.info("Processing order: $orderId")
        try {
            // Order processing logic
        } catch (e: Exception) {
            Logger.error("Order processing failed", e)
        }
    }
}
```

### Cache Manager Example

```kotlin
object CacheManager {
    private val cache = ConcurrentHashMap<String, CacheEntry>()
    private val defaultTtlMs = 5 * 60 * 1000L  // 5 minutes
    
    data class CacheEntry(
        val value: Any,
        val expireAt: Long
    )
    
    fun <T> get(key: String): T? {
        val entry = cache[key] ?: return null
        
        return if (System.currentTimeMillis() < entry.expireAt) {
            @Suppress("UNCHECKED_CAST")
            entry.value as T
        } else {
            cache.remove(key)
            null
        }
    }
    
    fun put(key: String, value: Any, ttlMs: Long = defaultTtlMs) {
        cache[key] = CacheEntry(
            value = value,
            expireAt = System.currentTimeMillis() + ttlMs
        )
    }
    
    fun remove(key: String) {
        cache.remove(key)
    }
    
    fun clear() {
        cache.clear()
    }
    
    fun size(): Int = cache.size
}

// Usage
class ProductService {
    fun getProduct(id: Long): Product {
        val cacheKey = "product:$id"
        
        // Check cache first
        CacheManager.get<Product>(cacheKey)?.let { return it }
        
        // Query from DB
        val product = productRepository.findById(id)
        
        // Store in cache
        CacheManager.put(cacheKey, product)
        
        return product
    }
}
```

### Spring Environment: Singleton Scope

In Spring, beans are managed as Singletons by default.

```kotlin
// Spring Bean is Singleton by default
@Service
class UserService(
    private val userRepository: UserRepository
) {
    // Same instance used for all requests
    fun findById(id: Long): User = userRepository.findById(id).orElseThrow()
}

// Singleton but testable - thanks to DI
@SpringBootTest
class UserServiceTest {
    @MockBean
    private lateinit var userRepository: UserRepository
    
    @Autowired
    private lateinit var userService: UserService
    
    @Test
    fun `user lookup test`() {
        // Can test with mock injected
    }
}
```

```kotlin
// Explicit Scope specification
@Configuration
class AppConfig {
    
    @Bean
    @Scope("singleton")  // Default, can be omitted
    fun singletonBean(): MyService = MyService()
    
    @Bean
    @Scope("prototype")  // New instance each time
    fun prototypeBean(): MyService = MyService()
}
```

### Before/After Comparison

```kotlin
// ❌ Before: Implement Singleton pattern directly
class ConnectionPool private constructor() {
    private val connections = mutableListOf<Connection>()
    
    companion object {
        @Volatile
        private var instance: ConnectionPool? = null
        
        fun getInstance(): ConnectionPool {
            return instance ?: synchronized(this) {
                instance ?: ConnectionPool().also { instance = it }
            }
        }
    }
    
    fun getConnection(): Connection {
        return connections.firstOrNull { it.isAvailable() }
            ?: createNewConnection()
    }
}

// Usage
val pool = ConnectionPool.getInstance()
val conn = pool.getConnection()

// ✅ After: Delegate to Spring + Use HikariCP
@Configuration
class DataSourceConfig {
    @Bean
    fun dataSource(): DataSource {
        return HikariDataSource().apply {
            jdbcUrl = "jdbc:postgresql://localhost:5432/mydb"
            username = "user"
            password = "password"
            maximumPoolSize = 10
        }
    }
}

// Spring manages as Singleton, easy to test
@Service
class UserRepository(
    private val dataSource: DataSource  // Injected
) {
    fun findById(id: Long): User {
        dataSource.connection.use { conn ->
            // ...
        }
    }
}
```

---

## Implementation Comparison

| Method | Thread-Safe | Lazy | Pros | Cons |
|--------|-------------|------|------|------|
| **Basic** | ❌ | ✅ | Simple | Concurrency issues |
| **synchronized** | ✅ | ✅ | Safe | Performance degradation |
| **DCL** | ✅ | ✅ | Good performance | Complex code |
| **Bill Pugh** | ✅ | ✅ | Clean, good performance | Java only |
| **Enum** | ✅ | ❌ | Serialization/Reflection safe | Cannot inherit |
| **Kotlin object** | ✅ | ✅ | Simple and clear | Kotlin only |

### Singleton vs Static

| Comparison | Singleton | Static |
|------------|-----------|--------|
| Inheritance | Possible | Not possible |
| Interface Implementation | Possible | Not possible |
| Lazy Initialization | Possible | Difficult |
| Testing (Mock) | Possible (with DI) | Difficult |
| Polymorphism | Possible | Not possible |

---

## References

### Books
- [Effective Java 3/E](https://www.amazon.com/Effective-Java-Joshua-Bloch/dp/0134685997) - Item 3: Enforce the singleton property with a private constructor or an enum type
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Singleton Chapter

### Recommended Sites
- [Refactoring Guru - Singleton Pattern](https://refactoring.guru/design-patterns/singleton)
- [Kotlin Object Declarations](https://kotlinlang.org/docs/object-declarations.html)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Factory Method Pattern](./Factory%20Method.en.md)
- [Abstract Factory Pattern](./Abstract%20Factory.en.md)
