# Spring Async (@Async)

## 개념

Spring의 `@Async`는 **메서드를 비동기로 실행하게 만드는 어노테이션**이다. 메서드에 `@Async`를 붙이면 호출자는 메서드의 완료를 기다리지 않고 즉시 다음 코드를 실행할 수 있다.

### 핵심 특징

- **논블로킹 실행**: 호출 스레드가 작업 완료를 기다리지 않음
- **스레드 풀 기반**: 별도의 스레드 풀에서 비동기 작업 실행
- **프록시 기반**: AOP 프록시를 통해 비동기 실행 처리
- **반환 타입 지원**: `void`, `Future`, `CompletableFuture` 등 다양한 반환 타입

---

## 왜 필요한가

### 해결하려는 문제

동기 방식에서는 긴 작업(외부 API 호출, 파일 처리, 이메일 발송 등)이 완료될 때까지 다음 작업을 진행할 수 없다. 이는 응답 시간 지연과 리소스 낭비를 초래한다.

### 기존 방식의 한계

1. **블로킹**: 작업 완료까지 스레드가 대기하여 리소스 낭비
2. **응답 지연**: 사용자가 불필요하게 긴 작업을 기다림
3. **낮은 처리량**: 동시에 처리 가능한 요청 수 제한
4. **복잡한 스레드 관리**: 직접 스레드를 생성하고 관리해야 함

### 제공하는 가치

- **빠른 응답**: 사용자는 즉시 응답을 받고 백그라운드에서 작업 처리
- **높은 처리량**: 블로킹 없이 더 많은 요청 동시 처리
- **간단한 사용**: 어노테이션만으로 비동기 처리 구현
- **효율적인 리소스 활용**: 스레드 풀로 스레드 재사용

---

## 동작 원리

### @Async 구현체 등록 및 탐색 과정

Spring이 `@Async` 메서드를 비동기로 실행하기까지의 과정은 다음과 같다.

```
1. @EnableAsync 선언
   ↓
2. AsyncConfigurationSelector 활성화
   ↓
3. ProxyAsyncConfiguration 등록
   ↓
4. AsyncAnnotationBeanPostProcessor
   → @Async 메서드를 프록시로 래핑
   ↓
5. AsyncExecutionInterceptor
   → 프록시가 실제 메서드 호출을 인터셉트
   ↓
6. Executor 선택 (우선순위 순)
   1) AsyncConfigurer.getAsyncExecutor()
   2) 빈 이름 "taskExecutor"
   3) SimpleAsyncTaskExecutor (기본값)
```

### Spring Boot의 기본 Executor 설정

Spring Boot는 `TaskExecutionAutoConfiguration`을 통해 자동으로 `ThreadPoolTaskExecutor`를 등록한다.

**빈 정보:**
- **빈 이름**: `applicationTaskExecutor`
- **실제 구현체**: `ThreadPoolTaskExecutor`
- **설정**: `TaskExecutionProperties` (`spring.task.execution.*`)

### ThreadPoolTaskExecutor 기본 설정

| 설정 | 기본값 | 설명 | 주의사항 |
|-----|-------|------|---------|
| `corePoolSize` | 8 | 항상 유지되는 최소 스레드 수 | 서버당 초당 처리 가능한 비동기 요청 제한 |
| `maxPoolSize` | `Integer.MAX_VALUE` | 허용되는 최대 스레드 수 | queueCapacity가 무제한이면 의미 없음, OOM 위험 |
| `queueCapacity` | `Integer.MAX_VALUE` | 작업 보관 BlockingQueue 크기 | 큐가 계속 쌓이면 OOM 위험 |
| `keepAliveSeconds` | 60 | 비활성 스레드 유지 시간 | 유휴 스레드 정리 |
| `allowCoreThreadTimeOut` | true | core 스레드도 timeout 적용 여부 | false면 항상 corePoolSize 유지 |
| `rejectedExecutionHandler` | `CallerRunsPolicy` | 큐가 가득 찼을 때 대응 정책 | 4가지 정책 선택 가능 |
| `waitForTasksToCompleteOnShutdown` | false | 종료 시 진행 중인 작업 대기 여부 | Kubernetes와 함께 고려 |
| `awaitTerminationSeconds` | 0 | 종료 시 최대 대기 시간 | `terminationGracePeriodSeconds` 이하로 설정 |

### ThreadPoolTaskExecutor 동작 흐름

```
작업 제출
    ↓
1. 현재 스레드 수 < corePoolSize?
   ✅ YES → 새 스레드 생성하여 즉시 실행
   ❌ NO → 2단계로
    ↓
2. 큐에 여유 공간 있음?
   ✅ YES → 큐에 작업 추가
   ❌ NO → 3단계로
    ↓
3. 현재 스레드 수 < maxPoolSize?
   ✅ YES → 추가 스레드 생성 (최대 maxPoolSize)
   ❌ NO → 4단계로
    ↓
4. RejectedExecutionHandler 동작
   - AbortPolicy: RejectedExecutionException 발생
   - CallerRunsPolicy: 호출 스레드가 직접 실행 (백프레셔)
   - DiscardPolicy: 새 작업 버림
   - DiscardOldestPolicy: 가장 오래된 작업 제거 후 재시도
```

### 예외 처리

반환값이 없는 `@Async` 메서드의 예외는 `AsyncUncaughtExceptionHandler`로 전달된다.

- **기본 핸들러**: `SimpleAsyncUncaughtExceptionHandler`
- **동작**: error level 로깅만 수행
- **커스터마이징**: `AsyncConfigurer` 구현으로 변경 가능

---

## 주의사항

### 1. 기본 설정의 OOM 위험

기본 `maxPoolSize`와 `queueCapacity`가 무제한이어서 메모리 부족이 발생할 수 있다.

```yaml
# application.yml - 안전한 설정 예시
spring:
  task:
    execution:
      pool:
        core-size: 10
        max-size: 20
        queue-capacity: 100
        keep-alive: 60s
```

### 2. Self-Invocation 문제

같은 클래스 내에서 `@Async` 메서드를 호출하면 프록시를 거치지 않아 동기로 실행된다.

```kotlin
// ❌ 잘못된 사용 - 비동기로 동작하지 않음
@Service
class UserService {
    fun process() {
        sendEmail() // 같은 클래스 내 호출 → 동기 실행
    }
    
    @Async
    fun sendEmail() {
        // ...
    }
}

// ✅ 올바른 사용 - 다른 빈에서 호출
@Service
class UserService(
    private val emailService: EmailService
) {
    fun process() {
        emailService.sendEmail() // 다른 빈 호출 → 비동기 실행
    }
}

@Service
class EmailService {
    @Async
    fun sendEmail() {
        // ...
    }
}
```

### 3. 예외 처리 필수

반환값이 없는 `@Async` 메서드에서 발생한 예외는 호출자에게 전달되지 않는다.

```kotlin
// 예외가 조용히 사라짐
@Async
fun processAsync() {
    throw RuntimeException("Error") // 기본 핸들러는 로깅만 함
}

// AsyncConfigurer로 예외 핸들러 등록
@Configuration
@EnableAsync
class AsyncConfig : AsyncConfigurer {
    override fun getAsyncUncaughtExceptionHandler(): AsyncUncaughtExceptionHandler {
        return AsyncUncaughtExceptionHandler { ex, method, params ->
            logger.error("Async error in ${method.name}: ${ex.message}", ex)
            // 알림 전송, 모니터링 등 추가 처리
        }
    }
}
```

### 4. Graceful Shutdown 설정

애플리케이션 종료 시 진행 중인 작업을 안전하게 완료해야 한다.

```yaml
spring:
  task:
    execution:
      shutdown:
        await-termination: true
        await-termination-period: 30s # Kubernetes terminationGracePeriodSeconds 이하
```

---

## 실전 적용

### 1. 기본 사용법

```kotlin
@SpringBootApplication
@EnableAsync // 비동기 활성화
class Application

@Service
class NotificationService {
    
    // 반환값 없는 비동기
    @Async
    fun sendEmail(to: String, subject: String, body: String) {
        Thread.sleep(3000) // 이메일 발송 시뮬레이션
        println("Email sent to $to")
    }
    
    // Future 반환
    @Async
    fun calculateAsync(value: Int): Future<Int> {
        Thread.sleep(2000)
        return AsyncResult(value * 2)
    }
    
    // CompletableFuture 반환 (권장)
    @Async
    fun fetchDataAsync(id: Long): CompletableFuture<User> {
        val user = userRepository.findById(id)
        return CompletableFuture.completedFuture(user)
    }
}
```

### 2. 커스텀 Executor 설정

```kotlin
@Configuration
@EnableAsync
class AsyncConfig : AsyncConfigurer {
    
    override fun getAsyncExecutor(): Executor {
        val executor = ThreadPoolTaskExecutor()
        executor.corePoolSize = 10
        executor.maxPoolSize = 20
        executor.queueCapacity = 100
        executor.setThreadNamePrefix("async-")
        executor.setRejectedExecutionHandler(ThreadPoolExecutor.CallerRunsPolicy())
        executor.initialize()
        return executor
    }
    
    override fun getAsyncUncaughtExceptionHandler(): AsyncUncaughtExceptionHandler {
        return AsyncUncaughtExceptionHandler { ex, method, params ->
            logger.error("Async exception in ${method.name}", ex)
        }
    }
}
```

### 3. 여러 Executor 사용

```kotlin
@Configuration
@EnableAsync
class AsyncConfig {
    
    @Bean("emailExecutor")
    fun emailExecutor(): Executor {
        val executor = ThreadPoolTaskExecutor()
        executor.corePoolSize = 5
        executor.maxPoolSize = 10
        executor.setThreadNamePrefix("email-")
        executor.initialize()
        return executor
    }
    
    @Bean("reportExecutor")
    fun reportExecutor(): Executor {
        val executor = ThreadPoolTaskExecutor()
        executor.corePoolSize = 2
        executor.maxPoolSize = 5
        executor.setThreadNamePrefix("report-")
        executor.initialize()
        return executor
    }
}

@Service
class NotificationService {
    
    @Async("emailExecutor") // 특정 Executor 지정
    fun sendEmail(to: String) {
        // 이메일 발송
    }
    
    @Async("reportExecutor")
    fun generateReport(userId: Long) {
        // 리포트 생성
    }
}
```

### 4. CompletableFuture 조합

```kotlin
@Service
class OrderService(
    private val paymentService: PaymentService,
    private val inventoryService: InventoryService,
    private val notificationService: NotificationService
) {
    
    fun processOrder(orderId: Long) {
        // 여러 비동기 작업을 병렬로 실행하고 결과 조합
        val paymentFuture = paymentService.processPaymentAsync(orderId)
        val inventoryFuture = inventoryService.reserveItemsAsync(orderId)
        
        CompletableFuture.allOf(paymentFuture, inventoryFuture)
            .thenRun {
                val paymentResult = paymentFuture.join()
                val inventoryResult = inventoryFuture.join()
                
                if (paymentResult && inventoryResult) {
                    notificationService.sendOrderConfirmation(orderId)
                }
            }
    }
}
```

### 5. 모니터링 설정

```kotlin
@Configuration
class ExecutorMonitoringConfig {
    
    @Bean
    fun taskExecutorMetrics(
        meterRegistry: MeterRegistry,
        @Qualifier("applicationTaskExecutor") executor: ThreadPoolTaskExecutor
    ): TaskExecutorMetrics {
        return TaskExecutorMetrics(executor, meterRegistry)
    }
}

class TaskExecutorMetrics(
    private val executor: ThreadPoolTaskExecutor,
    private val meterRegistry: MeterRegistry
) {
    
    @Scheduled(fixedRate = 10000) // 10초마다
    fun recordMetrics() {
        val threadPool = executor.threadPoolExecutor
        
        Gauge.builder("executor.pool.size", threadPool) { it.poolSize.toDouble() }
            .register(meterRegistry)
        
        Gauge.builder("executor.active.count", threadPool) { it.activeCount.toDouble() }
            .register(meterRegistry)
        
        Gauge.builder("executor.queue.size", threadPool) { it.queue.size.toDouble() }
            .register(meterRegistry)
    }
}
```

---

## 참고 자료

### 공식 문서
- [Spring Framework - Asynchronous Execution](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async)
- [Spring Boot - Task Execution](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.core.spring.task.execution)