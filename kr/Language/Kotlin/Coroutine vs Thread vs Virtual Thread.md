# Coroutine vs Thread vs Virtual Thread

## 개념

**Coroutine**, **OS Thread**, **Virtual Thread**는 모두 동시성을 처리하는 방법이지만, 동작 방식과 리소스 효율성에서 큰 차이가 있다.

| 구분 | OS Thread | Virtual Thread | Coroutine |
|------|-----------|----------------|-----------|
| **도입** | 운영체제 | Java 21 (2023) | Kotlin 1.3 (2018) |
| **관리 주체** | OS 커널 | JVM | Kotlin 런타임 |
| **메모리/개** | ~1MB (고정 스택) | ~1KB 초기, 동적 증가 | ~수백 바이트~수 KB (상태만) |
| **생성 비용** | 높음 (~1ms) | 낮음 (~1μs) | 매우 낮음 (~100ns) |
| **스케줄링** | OS 선점형 | 블로킹 감지 기반 (암시적) | 협력형 (suspend point) |
| **블로킹 I/O** | 스레드 점유 | 자동 언마운트 | Dispatcher 전환 또는 비동기 API |

---

## 왜 필요한가?

### OS Thread의 한계

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OS Thread 모델                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   요청 1개 = 스레드 1개                                                  │
│                                                                         │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│   │ Thread 1 │ │ Thread 2 │ │ Thread 3 │ │ Thread N │   ...             │
│   │  (~1MB)  │ │  (~1MB)  │ │  (~1MB)  │ │  (~1MB)  │                   │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                   │
│        │            │            │            │                         │
│        ▼            ▼            ▼            ▼                         │
│   ┌─────────────────────────────────────────────────┐                   │
│   │              OS Kernel Scheduler                 │                   │
│   │         (컨텍스트 스위칭 비용: ~1-10μs)          │                   │
│   └─────────────────────────────────────────────────┘                   │
│                                                                         │
│   문제점:                                                               │
│   • 10,000 요청 = 10GB 메모리 필요                                      │
│   • 컨텍스트 스위칭 오버헤드                                            │
│   • I/O 대기 시간 동안 스레드 자원 낭비                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```kotlin
// ❌ Thread-per-request 모델의 한계
class TraditionalServer {
    private val executor = Executors.newFixedThreadPool(200) // 최대 200개
    
    fun handleRequest(request: Request) {
        executor.submit {
            // I/O 작업 동안 스레드가 블로킹됨
            val user = userRepository.findById(request.userId)  // ~50ms 대기
            val orders = orderRepository.findByUser(user)       // ~100ms 대기
            val payments = paymentClient.getHistory(user)       // ~200ms 대기
            
            // 총 350ms 동안 스레드 점유 (실제 CPU 작업은 거의 없음)
            process(user, orders, payments)
        }
    }
}

// 동시 요청 1000개 → 200개만 처리, 800개는 큐에서 대기
```

### 해결 방향

| 접근 방식 | Virtual Thread | Coroutine |
|-----------|----------------|-----------|
| **철학** | 기존 블로킹 코드 그대로 사용 | 비동기 코드로 전환 |
| **학습 비용** | 낮음 (기존 코드 유지) | 중간 (suspend 함수 이해 필요) |
| **제어 수준** | 자동 (JVM이 관리) | 수동 (개발자가 Dispatcher 선택) |
| **생태계** | Java 21+ 필요 | Kotlin 전용 |

---

## 동작 원리

### OS Thread

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OS Thread 스케줄링                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Thread A                    Thread B                                  │
│   ┌───────┐                   ┌───────┐                                 │
│   │ 실행  │ ←─ 타임슬라이스    │ 대기  │                                 │
│   └───┬───┘    만료           └───────┘                                 │
│       │                                                                 │
│       ▼ 컨텍스트 스위칭 (~1-10μs)                                       │
│   ┌───────┐                   ┌───────┐                                 │
│   │ 대기  │                   │ 실행  │ ←─ 스케줄러 선택                 │
│   └───────┘                   └───┬───┘                                 │
│                                   │                                     │
│   • 선점형(Preemptive): OS가 강제로 스레드 전환                          │
│   • 컨텍스트 저장: 레지스터, 스택 포인터, PC 등                          │
│   • 커널 모드 전환 필요                                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Virtual Thread (Java 21+)

Virtual Thread의 스케줄링은 **선점형도 아니고 순수 협력형도 아닌 “블로킹 감지 기반”** 방식이다.

| 스케줄링 | OS Thread | Virtual Thread | Coroutine |
|----------|-----------|----------------|-----------|  
| **방식** | 선점형 | 블로킹 감지 기반 | 협력형 |
| **전환 시점** | OS가 타임슬라이스로 강제 | JVM이 블로킹 API 호출 감지 | 명시적 suspend point |
| **개발자 제어** | 없음 | 없음 (암시적) | 있음 (명시적) |
| **yield 필요** | 불필요 | 불필요 | 필요 (suspend) |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Virtual Thread 구조                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Virtual Thread 1   Virtual Thread 2   Virtual Thread 3   ...          │
│   ┌────────────┐     ┌────────────┐     ┌────────────┐                  │
│   │ Task: API  │     │ Task: DB   │     │ Task: File │                  │
│   │ (~KB 스택) │     │ (~KB 스택) │     │ (~KB 스택) │                  │
│   └─────┬──────┘     └─────┬──────┘     └─────┬──────┘                  │
│         │                  │                  │                         │
│         └────────────┬─────┴──────────────────┘                         │
│                      ▼                                                  │
│   ┌─────────────────────────────────────────────────────┐               │
│   │              Carrier Thread Pool                     │               │
│   │          (Platform Thread, ~CPU 코어 수)             │               │
│   │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                │               │
│   │  │  PT1 │ │  PT2 │ │  PT3 │ │  PT4 │                │               │
│   │  └──────┘ └──────┘ └──────┘ └──────┘                │               │
│   └─────────────────────────────────────────────────────┘               │
│                                                                         │
│   블로킹 감지 기반 스케줄링:                                             │
│   1. JVM이 블로킹 API 호출을 감지 (sleep, I/O, lock 등)              │
│   2. Virtual Thread가 Carrier Thread에서 자동 "언마운트"            │
│   3. Carrier Thread는 다른 Virtual Thread 실행                        │
│   4. 블로킹 완료 시 다시 "마운트" (다른 Carrier Thread일 수 있음)    │
│                                                                         │
│   ❓ 협력형과의 차이:                                                   │
│   • Coroutine: 개발자가 suspend 키워드로 명시적 yield             │
│   • Virtual Thread: JVM이 블로킹 API를 암시적으로 감지하여 yield    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```java
// Java 21 Virtual Thread
public class VirtualThreadExample {
    public static void main(String[] args) throws Exception {
        // 100만 개의 Virtual Thread 생성 가능
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 1_000_000).forEach(i -> {
                executor.submit(() -> {
                    // 블로킹 코드 그대로 사용 - JVM이 자동으로 최적화
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
        // 기존 블로킹 코드 그대로 사용
        // Virtual Thread가 블로킹 시 자동으로 언마운트
        User user = userRepository.findById(id);  // 블로킹 OK
        List<Order> orders = orderClient.getOrders(user);  // 블로킹 OK
        return user;
    }
}

// application.yml
// spring.threads.virtual.enabled=true
```

### Coroutine

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Coroutine 구조                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Coroutine 1         Coroutine 2         Coroutine 3                   │
│   ┌───────────┐       ┌───────────┐       ┌───────────┐                 │
│   │ suspend   │       │ suspend   │       │ suspend   │                 │
│   │ 함수 실행 │       │ 함수 실행 │       │ 함수 실행 │                 │
│   │(상태머신) │       │(상태머신) │       │(상태머신) │                 │
│   └─────┬─────┘       └─────┬─────┘       └─────┬─────┘                 │
│         │                   │                   │                       │
│         └─────────────┬─────┴───────────────────┘                       │
│                       ▼                                                 │
│   ┌─────────────────────────────────────────────────────┐               │
│   │                  Dispatcher                          │               │
│   │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │               │
│   │  │ Dispatchers  │  │ Dispatchers  │  │ Dispatchers│ │               │
│   │  │   .Default   │  │     .IO      │  │   .Main    │ │               │
│   │  │ (CPU 연산)   │  │  (I/O 작업)  │  │ (UI 작업)  │ │               │
│   │  └──────────────┘  └──────────────┘  └────────────┘ │               │
│   └─────────────────────────────────────────────────────┘               │
│                                                                         │
│   suspend 지점에서:                                                     │
│   1. 현재 상태를 Continuation 객체에 저장 (수백 바이트)                 │
│   2. 스레드 반환 (다른 코루틴 실행 가능)                                │
│   3. 작업 완료 시 Continuation에서 재개                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

```kotlin
// Coroutine - 명시적 suspend
@RestController
class UserController(
    private val userService: UserService
) {
    @GetMapping("/user/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        // suspend 함수 - 비동기적으로 실행
        return userService.findById(id)
    }
}

@Service
class UserService(
    private val userRepository: UserRepository,
    private val orderClient: OrderClient
) {
    suspend fun findById(id: Long): User {
        // 병렬 실행으로 최적화
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

## 주의사항

### 1. Virtual Thread에서 synchronized 주의

```java
// ❌ synchronized 블록에서 블로킹 - Carrier Thread가 고정됨 (Pinning)
public class BadExample {
    private final Object lock = new Object();
    
    public void process() {
        synchronized (lock) {
            // Virtual Thread가 Carrier Thread에 "pinned" 됨
            blockingCall();  // Carrier Thread 점유 - 다른 VT 실행 불가
        }
    }
}

// ✅ ReentrantLock 사용
public class GoodExample {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void process() {
        lock.lock();
        try {
            blockingCall();  // 블로킹 시 Carrier Thread 반환 가능
        } finally {
            lock.unlock();
        }
    }
}
```

### 2. Coroutine에서 블로킹 코드 주의

Coroutine에서 블로킹 코드를 사용하면 Dispatcher 전환이 필요하지만, **비동기 라이브러리**를 사용하면 전환 없이 바로 사용할 수 있다.

```kotlin
// ❌ 잘못된 예 - Dispatchers.Default에서 블로킹
suspend fun processData(): Data {
    // Default Dispatcher의 스레드 고갈
    val result = blockingDatabaseCall()  // 블로킹!
    return result
}

// ✅ 방법 1: withContext로 Dispatcher 전환 (블로킹 API 사용 시)
suspend fun processData(): Data = withContext(Dispatchers.IO) {
    // I/O Dispatcher에서 블로킹 작업 실행
    jdbcTemplate.query("SELECT * FROM users")  // 블로킹 JDBC
}

// ✅ 방법 2: 비동기 라이브러리 사용 (권장 - Dispatcher 전환 불필요)
suspend fun processData(): Data {
    // R2DBC, WebClient 등 논블로킹 API 사용
    // suspend 함수를 직접 지원하므로 Dispatcher 전환 불필요
    return r2dbcRepository.findById(id).awaitSingle()
}
```

#### 비동기 vs 블로킹 라이브러리 비교

| 구분 | 블로킹 (전통적) | 비동기 (Reactive) |
|------|------------------|---------------------|
| **DB** | JDBC, JPA/Hibernate | R2DBC, Mongo Reactive |
| **HTTP Client** | RestTemplate, OkHttp | WebClient, Ktor Client |
| **파일 I/O** | java.io.* | kotlinx-coroutines-io |
| **Dispatcher 전환** | 필요 (Dispatchers.IO) | 불필요 |
| **스레드 효율** | 블로킹 중 점유 | 완전 논블로킹 |

```kotlin
// 예시: WebClient (Spring WebFlux) - Dispatcher 전환 불필요
@Service
class UserService(
    private val webClient: WebClient
) {
    // 비동기 API는 suspend 함수에서 바로 호출 가능
    suspend fun fetchUser(id: Long): User {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User::class.java)
            .awaitSingle()  // Mono -> suspend 변환
    }
}

// 예시: R2DBC - Dispatcher 전환 불필요
@Repository
interface UserRepository : CoroutineCrudRepository<User, Long> {
    // suspend 함수로 자동 생성
    suspend fun findByEmail(email: String): User?
}

@Service
class UserService(
    private val userRepository: UserRepository
) {
    suspend fun findUser(email: String): User? {
        // withContext 불필요 - 이미 논블로킹
        return userRepository.findByEmail(email)
    }
}
```

### 3. ThreadLocal 사용 시 주의

```kotlin
// Virtual Thread / Coroutine 모두 주의 필요

// ❌ ThreadLocal - 재사용되는 스레드에서 문제
val threadLocal = ThreadLocal<User>()

suspend fun process() {
    threadLocal.set(currentUser)
    delay(100)  // 다른 스레드에서 재개될 수 있음
    val user = threadLocal.get()  // 다른 값이거나 null일 수 있음
}

// ✅ Coroutine - CoroutineContext 사용
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
        // 안전하게 user 접근 가능
    });
}
```

### 4. 언제 무엇을 선택할까?

| 상황 | 추천 | 이유 |
|------|------|------|
| 기존 Java 블로킹 코드 | Virtual Thread | 코드 변경 최소화 |
| 새로운 Kotlin 프로젝트 | Coroutine | 더 세밀한 제어, 언어 통합 |
| CPU 집약적 작업 | 일반 Thread | 컨텍스트 스위칭이 오히려 오버헤드 |
| 매우 많은 동시 연결 | Coroutine | 메모리 효율 최고 |
| 간단한 웹 서버 | Virtual Thread | 설정만으로 적용 가능 |

---

## 실전 적용

### 성능 비교

```kotlin
// 10,000개 동시 작업 시뮬레이션

// OS Thread
fun threadTest() {
    val executor = Executors.newFixedThreadPool(200)
    val latch = CountDownLatch(10_000)
    
    repeat(10_000) {
        executor.submit {
            Thread.sleep(100)  // I/O 대기 시뮬레이션
            latch.countDown()
        }
    }
    latch.await()  // 약 5초 (10000/200 * 100ms)
}

// Virtual Thread (Java 21)
fun virtualThreadTest() {
    val executor = Executors.newVirtualThreadPerTaskExecutor()
    val latch = CountDownLatch(10_000)
    
    repeat(10_000) {
        executor.submit {
            Thread.sleep(100)  // 블로킹이지만 효율적
            latch.countDown()
        }
    }
    latch.await()  // 약 120~200ms (스케줄링 오버헤드 포함)
}

// Coroutine
fun coroutineTest() = runBlocking {
    val jobs = List(10_000) {
        launch {
            delay(100)  // 논블로킹 대기
        }
    }
    jobs.joinAll()  // 약 105~150ms (가장 빠름)
}
```

### 메모리 사용량 비교

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    10,000개 동시 작업 시 메모리 (단순 작업 기준)        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   OS Thread      │████████████████████████████████████│  ~10GB          │
│                  │ (1MB × 10,000, 고정 스택)          │                 │
│                                                                         │
│   Virtual Thread │████                                │  ~50~100MB      │
│                  │ (초기 ~1KB, 호출 깊이에 따라 증가) │                 │
│                                                                         │
│   Coroutine      │██                                  │  ~10~20MB       │
│                  │ (수백 바이트~수 KB, 상태만 저장)   │  ✅ 가장 효율적 │
│                                                                         │
│   * 호출 스택이 깊어지면 Virtual Thread 메모리 사용량 증가              │
│   * Coroutine은 CPS 변환으로 상태만 저장하여 메모리 효율 최고           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Spring Boot 적용

```kotlin
// Spring Boot 3.2+ with Virtual Thread
// application.yml
// spring:
//   threads:
//     virtual:
//       enabled: true

// 기존 코드 그대로 사용 - 자동으로 Virtual Thread 적용
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

### Before/After 비교

```kotlin
// ❌ Before: 전통적인 Thread Pool 방식
@Configuration
class ThreadPoolConfig {
    @Bean
    fun taskExecutor(): TaskExecutor {
        val executor = ThreadPoolTaskExecutor()
        executor.corePoolSize = 50
        executor.maxPoolSize = 200
        executor.queueCapacity = 1000
        // 200개 초과 요청은 큐에서 대기
        return executor
    }
}

// ✅ After: Virtual Thread (설정만 변경)
// application.yml
// spring.threads.virtual.enabled: true
// 코드 변경 없음!

// ✅ After: Coroutine (더 세밀한 제어)
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryClient: InventoryClient,
    private val paymentClient: PaymentClient
) {
    suspend fun processOrder(request: OrderRequest): OrderResult = coroutineScope {
        // 병렬 검증
        val inventoryCheck = async { inventoryClient.check(request.items) }
        val userValidation = async { validateUser(request.userId) }
        
        inventoryCheck.await()
        userValidation.await()
        
        // 순차 처리
        val payment = paymentClient.process(request.payment)
        orderRepository.save(Order.from(request, payment))
    }
}
```

---

## Coroutine + Virtual Thread 조합

### 왜 조합하는가?

Coroutine과 Virtual Thread는 **상호 보완적**이다. Coroutine의 `Dispatchers.IO`를 Virtual Thread로 대체하면 블로킹 I/O 처리 효율이 극대화된다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    기존 Coroutine + Thread Pool                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Coroutine 1 ──► withContext(Dispatchers.IO) ──► Thread Pool (제한적)  │
│   Coroutine 2 ──► withContext(Dispatchers.IO) ──► max(64, CPU코어수)    │
│   ...                                                                   │
│   Coroutine N ──► withContext(Dispatchers.IO) ──► ⚠️ 스레드 고갈 가능   │
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
│                                                    ✅ 무제한 확장 가능   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Virtual Thread Dispatcher 설정

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.Executors

// Virtual Thread 기반 Dispatcher 생성 (싱글톤으로 관리)
object VirtualThreadDispatcher {
    val dispatcher: CoroutineDispatcher by lazy {
        Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
    }
}

// 또는 확장 프로퍼티로 정의 (lazy로 한 번만 생성)
@OptIn(ExperimentalCoroutinesApi::class)
val Dispatchers.VT: CoroutineDispatcher by lazy {
    Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
}
```

### 실전 적용 예제

```kotlin
// 기존 방식: Dispatchers.IO 사용
suspend fun fetchDataOld(): Data = withContext(Dispatchers.IO) {
    // 기본 max(64, CPU코어수) 스레드 풀에서 실행 - 많은 동시 요청 시 병목
    blockingHttpCall()
}

// 개선된 방식: Virtual Thread 사용
suspend fun fetchDataNew(): Data = withContext(VirtualThreadDispatcher.dispatcher) {
    // Virtual Thread에서 실행 - 블로킹해도 효율적
    blockingHttpCall()
}
```

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val legacyPaymentClient: LegacyPaymentClient,  // 블로킹 API
    private val inventoryClient: InventoryClient           // 블로킹 API
) {
    private val vtDispatcher = Executors.newVirtualThreadPerTaskExecutor()
        .asCoroutineDispatcher()
    
    suspend fun processOrder(request: OrderRequest): OrderResult = coroutineScope {
        // 병렬 실행 + Virtual Thread로 블로킹 처리
        val payment = async(vtDispatcher) {
            legacyPaymentClient.process(request.payment)  // 블로킹 OK
        }
        
        val inventory = async(vtDispatcher) {
            inventoryClient.reserve(request.items)  // 블로킹 OK
        }
        
        // Coroutine의 구조화된 동시성 유지
        val paymentResult = payment.await()
        val inventoryResult = inventory.await()
        
        // 논블로킹 작업은 기본 Dispatcher 사용
        orderRepository.save(Order.from(request, paymentResult, inventoryResult))
    }
}
```

### Spring Boot 3.2+ 통합

```kotlin
// application.yml
// spring:
//   threads:
//     virtual:
//       enabled: true

@Configuration
class CoroutineConfig {
    
    // Virtual Thread 기반 Dispatcher Bean
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
        // DB 조회 - suspend 함수 (논블로킹)
        val user = userRepository.findById(id)
        
        // 레거시 API 호출 - 블로킹이므로 Virtual Thread 사용
        val legacyData = async(vtDispatcher) {
            legacyUserClient.fetchAdditionalData(id)
        }
        
        EnrichedUser(user, legacyData.await())
    }
}
```

### 언제 조합을 사용할까?

| 상황 | 권장 방식 |
|------|----------|
| 순수 논블로킹 코드 | `Dispatchers.Default` 또는 suspend 함수 |
| 블로킹 I/O (DB, HTTP) | `withContext(vtDispatcher)` |
| CPU 집약적 작업 | `Dispatchers.Default` |
| 레거시 블로킹 라이브러리 | Virtual Thread Dispatcher |
| 파일 I/O | Virtual Thread Dispatcher |

```kotlin
// 최적의 조합 예시
suspend fun processRequest(request: Request): Response = coroutineScope {
    // 1. 논블로킹 - 기본 Dispatcher
    val validated = validateRequest(request)
    
    // 2. 블로킹 I/O - Virtual Thread
    val externalData = withContext(vtDispatcher) {
        legacyClient.fetch(validated.id)  // 블로킹 API
    }
    
    // 3. CPU 연산 - Default Dispatcher
    val processed = withContext(Dispatchers.Default) {
        heavyComputation(externalData)
    }
    
    // 4. 논블로킹 저장
    repository.save(processed)
}
```

### 주의사항

```kotlin
// ❌ Virtual Thread Dispatcher를 매번 생성하지 않기
suspend fun badExample() {
    withContext(Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()) {
        // 매번 새 Executor 생성 - 리소스 낭비
    }
}

// ✅ 싱글톤으로 재사용
object Dispatchers {
    val VT: CoroutineDispatcher by lazy {
        Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()
    }
}

suspend fun goodExample() {
    withContext(Dispatchers.VT) {
        // Executor 재사용
    }
}
```

```kotlin
// ❌ close() 호출 시점 주의
class MyService : AutoCloseable {
    private val vtExecutor = Executors.newVirtualThreadPerTaskExecutor()
    private val vtDispatcher = vtExecutor.asCoroutineDispatcher()
    
    override fun close() {
        vtDispatcher.close()  // Dispatcher 종료
        vtExecutor.close()    // Executor 종료
    }
}
```

---

## 비교 정리

### 특성 비교

| 특성 | OS Thread | Virtual Thread | Coroutine |
|------|-----------|----------------|-----------|
| 생성 비용 | 높음 (~1ms) | 낮음 (~1μs) | 매우 낮음 (~100ns) |
| 메모리 | ~1MB/thread | ~10KB/thread | ~1KB/coroutine |
| 최대 개수 | ~수천 개 | ~수백만 개 | ~수백만 개 |
| 블로킹 처리 | 스레드 점유 | 자동 언마운트 | Dispatcher 전환 |
| 학습 곡선 | 낮음 | 낮음 | 중간 |
| 언어 지원 | 모든 언어 | Java 21+ | Kotlin |
| 코드 스타일 | 블로킹 | 블로킹 | suspend |

### 선택 가이드

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         기술 선택 플로우차트                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Q: 언어가 무엇인가?                                                   │
│   ├── Java → Q: Java 버전이 21 이상인가?                                │
│   │          ├── Yes → Virtual Thread 권장                              │
│   │          └── No → CompletableFuture 또는 RxJava                     │
│   │                                                                     │
│   └── Kotlin → Q: 비동기 제어가 세밀하게 필요한가?                      │
│                ├── Yes → Coroutine                                      │
│                └── No → Virtual Thread도 가능 (JVM 21+)                 │
│                                                                         │
│   Q: 기존 블로킹 코드가 많은가?                                         │
│   ├── Yes → Virtual Thread (코드 변경 최소화)                           │
│   └── No → Coroutine (더 효율적)                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

### 공식 문서
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Spring Framework Virtual Threads](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-task-executor-virtual-threads)

### 추천 아티클
- [Virtual Threads: A Game-Changer for Concurrency](https://blogs.oracle.com/javamagazine/post/java-virtual-threads)
- [Kotlin Coroutines vs Java Virtual Threads](https://kt.academy/article/coroutines-vs-virtual-threads)

### 관련 TIL
- [Coroutine](./Coroutine.md)
- [JVM](../JVM/JVM.md)
