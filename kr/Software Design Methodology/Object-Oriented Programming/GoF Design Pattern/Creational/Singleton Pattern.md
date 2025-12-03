# Singleton Pattern

## 개념

Singleton 패턴은 **클래스의 인스턴스가 오직 하나만 생성되도록 보장**하고, 그 인스턴스에 대한 **전역 접근점을 제공**하는 패턴이다.

GoF 생성 패턴 중 하나로, 애플리케이션 전체에서 단 하나의 객체만 필요한 경우에 사용한다. 핵심은 "**유일한 인스턴스**"와 "**전역 접근**"이다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **단일 인스턴스** | 클래스의 인스턴스는 오직 하나만 존재 |
| **전역 접근** | 어디서든 동일한 인스턴스에 접근 가능 |
| **지연 초기화** | 필요한 시점에 인스턴스 생성 (선택적) |
| **생성 제어** | 외부에서 new로 인스턴스 생성 불가 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 여러 인스턴스가 생성되어 일관성 없는 상태 발생
class DatabaseConnection {
    private var connectionPool: List<Connection> = mutableListOf()
    
    fun getConnection(): Connection {
        // 매번 새로운 인스턴스를 만들면 커넥션 풀이 분산됨
        return connectionPool.firstOrNull() ?: createNewConnection()
    }
}

// 각각 다른 인스턴스 - 커넥션 풀이 공유되지 않음
val db1 = DatabaseConnection()
val db2 = DatabaseConnection()
val db3 = DatabaseConnection()
```

- 리소스를 공유해야 하는 객체가 여러 개 생성됨
- 설정 정보가 인스턴스마다 달라질 수 있음
- 메모리 낭비 및 상태 불일치 발생
- 전역 상태 관리가 어려움

### 제공하는 가치

- **리소스 절약**: 하나의 인스턴스만 생성하여 메모리 효율적
- **상태 일관성**: 모든 클라이언트가 동일한 상태 공유
- **전역 접근**: 어디서든 쉽게 접근 가능
- **생성 제어**: 인스턴스 생성 시점과 방법을 완전히 제어

---

## 동작 원리

### 구조

```
┌─────────────────────────────────────────────────────────────┐
│                         Singleton                           │
├─────────────────────────────────────────────────────────────┤
│  - instance: Singleton          ← 유일한 인스턴스 저장       │
│  - Singleton()                  ← private 생성자            │
├─────────────────────────────────────────────────────────────┤
│  + getInstance(): Singleton     ← 전역 접근점               │
│  + businessLogic()              ← 비즈니스 로직              │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ 호출
                              ▼
┌──────────┐  getInstance()  ┌──────────┐  getInstance()  ┌──────────┐
│ Client A │ ──────────────► │          │ ◄────────────── │ Client B │
└──────────┘                 │ instance │                 └──────────┘
                             │          │
┌──────────┐  getInstance()  │          │  getInstance()  ┌──────────┐
│ Client C │ ──────────────► │          │ ◄────────────── │ Client D │
└──────────┘                 └──────────┘                 └──────────┘
                             동일한 인스턴스 반환
```

### Java 기본 구현

```java
// 가장 기본적인 Singleton (Thread-safe 하지 않음)
public class BasicSingleton {
    private static BasicSingleton instance;
    
    // private 생성자 - 외부에서 new 불가
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
    
    // synchronized로 동기화 - 성능 저하 가능
    public static synchronized ThreadSafeSingleton getInstance() {
        if (instance == null) {
            instance = new ThreadSafeSingleton();
        }
        return instance;
    }
}

// Double-Checked Locking (DCL)
public class DCLSingleton {
    // volatile: 메모리 가시성 보장
    private static volatile DCLSingleton instance;
    
    private DCLSingleton() {}
    
    public static DCLSingleton getInstance() {
        if (instance == null) {                    // 1차 검사 (락 없이)
            synchronized (DCLSingleton.class) {
                if (instance == null) {            // 2차 검사 (락 안에서)
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh Singleton (권장) - Initialization-on-demand holder
public class BillPughSingleton {
    private BillPughSingleton() {}
    
    // static inner class는 외부 클래스 로드 시 로드되지 않음
    // getInstance() 호출 시점에 로드되어 인스턴스 생성
    private static class SingletonHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

// Enum Singleton (가장 안전) - Joshua Bloch 권장
public enum EnumSingleton {
    INSTANCE;
    
    public void businessLogic() {
        // 비즈니스 로직
    }
}
```

### Kotlin 구현

```kotlin
// Kotlin object - 언어 레벨에서 Singleton 지원
object DatabaseConfig {
    var host: String = "localhost"
    var port: Int = 5432
    var database: String = "mydb"
    
    fun getConnectionString(): String {
        return "jdbc:postgresql://$host:$port/$database"
    }
}

// 사용 - 별도의 getInstance() 호출 불필요
val connectionString = DatabaseConfig.getConnectionString()
DatabaseConfig.host = "production-db.example.com"
```

```kotlin
// lazy 초기화가 필요한 경우
class AppConfig private constructor() {
    var appName: String = ""
    var version: String = ""
    var debug: Boolean = false
    
    companion object {
        // by lazy는 thread-safe하고 최초 접근 시 초기화
        val instance: AppConfig by lazy { 
            AppConfig().apply {
                // 초기화 로직
                loadFromFile()
            }
        }
    }
    
    private fun loadFromFile() {
        // 설정 파일에서 로드
        appName = "MyApp"
        version = "1.0.0"
        debug = false
    }
}

// 사용
val config = AppConfig.instance
println(config.appName)  // MyApp
```

---

## 주의사항

### 1. 멀티스레드 환경에서 동기화 필수

```kotlin
// ❌ Race Condition 발생 가능
class UnsafeSingleton {
    companion object {
        private var instance: UnsafeSingleton? = null
        
        fun getInstance(): UnsafeSingleton {
            if (instance == null) {  // Thread A, B 동시 진입 가능
                instance = UnsafeSingleton()  // 인스턴스가 2개 생성될 수 있음
            }
            return instance!!
        }
    }
}

// ✅ Kotlin object 사용 - Thread-safe 보장
object SafeSingleton {
    // Kotlin object는 JVM 레벨에서 thread-safe
}

// ✅ 또는 lazy 사용
class SafeLazySingleton {
    companion object {
        val instance: SafeLazySingleton by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
            SafeLazySingleton()
        }
    }
}
```

### 2. 테스트하기 어려움

```kotlin
// ❌ Singleton 직접 사용 - Mock 불가
class OrderService {
    fun createOrder(order: Order) {
        // Singleton 직접 참조 - 테스트 시 DB 연결됨
        DatabaseConnection.instance.execute("INSERT INTO orders ...")
    }
}

// ✅ 의존성 주입으로 해결
interface DatabaseConnection {
    fun execute(sql: String)
}

object ProductionDatabaseConnection : DatabaseConnection {
    override fun execute(sql: String) {
        // 실제 DB 연결
    }
}

class OrderService(
    private val database: DatabaseConnection  // 주입받음
) {
    fun createOrder(order: Order) {
        database.execute("INSERT INTO orders ...")
    }
}

// 테스트
class OrderServiceTest {
    @Test
    fun `주문 생성 테스트`() {
        val mockDatabase = mockk<DatabaseConnection>()
        val service = OrderService(mockDatabase)
        
        service.createOrder(testOrder)
        
        verify { mockDatabase.execute(any()) }
    }
}
```

### 3. 전역 상태로 인한 숨겨진 의존성

```kotlin
// ❌ 숨겨진 의존성 - 클래스만 보고 의존관계 파악 불가
class PaymentProcessor {
    fun process(payment: Payment) {
        val config = AppConfig.instance  // 숨겨진 의존성
        val logger = Logger.instance     // 숨겨진 의존성
        // ...
    }
}

// ✅ 명시적 의존성
class PaymentProcessor(
    private val config: AppConfig,
    private val logger: Logger
) {
    fun process(payment: Payment) {
        // 의존성이 명시적
    }
}
```

### 4. 단일 책임 원칙 위반 가능

Singleton은 "인스턴스 생성 관리"와 "비즈니스 로직" 두 가지 책임을 가진다.

```kotlin
// ❌ 두 가지 책임이 혼재
object OrderManager {
    // 책임 1: 인스턴스 관리 (object 키워드로 암시적)
    
    // 책임 2: 비즈니스 로직
    fun createOrder() { }
    fun cancelOrder() { }
    fun processPayment() { }
}

// ✅ Spring에서는 프레임워크가 생명주기 관리
@Service  // Spring이 Singleton으로 관리
class OrderManager {
    // 비즈니스 로직만 담당
    fun createOrder() { }
    fun cancelOrder() { }
}
```

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 설정 관리 | 애플리케이션 설정, 환경 변수 |
| 로깅 | 로그 인스턴스 공유 |
| 캐시 | 전역 캐시 매니저 |
| 스레드 풀 | 커넥션 풀, 스레드 풀 |
| 레지스트리 | 객체 레지스트리, 서비스 로케이터 |

### 로깅 시스템 예제

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

// 어디서든 동일한 로거 사용
class UserService {
    fun createUser(name: String) {
        Logger.info("Creating user: $name")
        // 사용자 생성 로직
        Logger.debug("User created successfully")
    }
}

class OrderService {
    fun processOrder(orderId: String) {
        Logger.info("Processing order: $orderId")
        try {
            // 주문 처리 로직
        } catch (e: Exception) {
            Logger.error("Order processing failed", e)
        }
    }
}
```

### 캐시 매니저 예제

```kotlin
object CacheManager {
    private val cache = ConcurrentHashMap<String, CacheEntry>()
    private val defaultTtlMs = 5 * 60 * 1000L  // 5분
    
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

// 사용
class ProductService {
    fun getProduct(id: Long): Product {
        val cacheKey = "product:$id"
        
        // 캐시에서 먼저 조회
        CacheManager.get<Product>(cacheKey)?.let { return it }
        
        // DB에서 조회
        val product = productRepository.findById(id)
        
        // 캐시에 저장
        CacheManager.put(cacheKey, product)
        
        return product
    }
}
```

### Spring 환경: Singleton Scope

Spring에서는 빈이 기본적으로 Singleton으로 관리된다.

```kotlin
// Spring Bean은 기본적으로 Singleton
@Service
class UserService(
    private val userRepository: UserRepository
) {
    // 모든 요청에서 동일한 인스턴스 사용
    fun findById(id: Long): User = userRepository.findById(id).orElseThrow()
}

// Singleton이지만 테스트 가능 - DI 덕분
@SpringBootTest
class UserServiceTest {
    @MockBean
    private lateinit var userRepository: UserRepository
    
    @Autowired
    private lateinit var userService: UserService
    
    @Test
    fun `사용자 조회 테스트`() {
        // Mock 주입된 상태로 테스트 가능
    }
}
```

```kotlin
// 명시적 Scope 지정
@Configuration
class AppConfig {
    
    @Bean
    @Scope("singleton")  // 기본값, 생략 가능
    fun singletonBean(): MyService = MyService()
    
    @Bean
    @Scope("prototype")  // 매번 새 인스턴스
    fun prototypeBean(): MyService = MyService()
}
```

### Before/After 비교

```kotlin
// ❌ Before: 직접 Singleton 패턴 구현
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

// 사용
val pool = ConnectionPool.getInstance()
val conn = pool.getConnection()

// ✅ After: Spring에 위임 + HikariCP 사용
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

// Spring이 Singleton으로 관리, 테스트도 쉬움
@Service
class UserRepository(
    private val dataSource: DataSource  // 주입받음
) {
    fun findById(id: Long): User {
        dataSource.connection.use { conn ->
            // ...
        }
    }
}
```

---

## 구현 방식 비교

| 방식 | Thread-Safe | Lazy | 장점 | 단점 |
|------|-------------|------|------|------|
| **기본 방식** | ❌ | ✅ | 단순함 | 동시성 문제 |
| **synchronized** | ✅ | ✅ | 안전함 | 성능 저하 |
| **DCL** | ✅ | ✅ | 성능 좋음 | 코드 복잡 |
| **Bill Pugh** | ✅ | ✅ | 깔끔, 성능 좋음 | Java 전용 |
| **Enum** | ✅ | ❌ | 직렬화/리플렉션 안전 | 상속 불가 |
| **Kotlin object** | ✅ | ✅ | 간단명료 | Kotlin 전용 |

### Singleton vs Static

| 비교 항목 | Singleton | Static |
|-----------|-----------|--------|
| 상속 | 가능 | 불가능 |
| 인터페이스 구현 | 가능 | 불가능 |
| 지연 초기화 | 가능 | 어려움 |
| 테스트 (Mock) | 가능 (DI 시) | 어려움 |
| 다형성 | 가능 | 불가능 |

---

## 참고 자료

### 서적
- [Effective Java 3/E](https://www.amazon.com/Effective-Java-Joshua-Bloch/dp/0134685997) - Item 3: Enforce the singleton property with a private constructor or an enum type
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Singleton 챕터

### 추천 사이트
- [Refactoring Guru - Singleton Pattern](https://refactoring.guru/design-patterns/singleton)
- [Kotlin Object Declarations](https://kotlinlang.org/docs/object-declarations.html)

### 관련 TIL
- [디자인 패턴이란](../What%20is%20Design%20Pattern.md)
- [Factory Method 패턴](./Factory%20Method%20Pattern.md)
- [Abstract Factory 패턴](./Abstract%20Factory.md)
