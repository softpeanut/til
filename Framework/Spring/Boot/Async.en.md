# Spring Async (@Async)

## Concept

Spring's `@Async` is **an annotation that makes methods execute asynchronously**. When `@Async` is applied to a method, the caller can immediately execute the next code without waiting for the method to complete.

### Key Characteristics

- **Non-blocking Execution**: Calling thread doesn't wait for task completion
- **Thread Pool Based**: Asynchronous tasks execute in a separate thread pool
- **Proxy Based**: Handles asynchronous execution through AOP proxy
- **Return Type Support**: Supports various return types like `void`, `Future`, `CompletableFuture`

---

## Why Needed

### Problems to Solve

In synchronous approach, next tasks cannot proceed until long-running tasks (external API calls, file processing, email sending, etc.) complete. This causes response time delays and resource waste.

### Limitations of Existing Approach

1. **Blocking**: Thread waits until task completion, wasting resources
2. **Response Delay**: Users unnecessarily wait for long tasks
3. **Low Throughput**: Limited number of concurrent requests
4. **Complex Thread Management**: Must create and manage threads manually

### Value Provided

- **Fast Response**: Users receive immediate response while tasks process in background
- **High Throughput**: Process more concurrent requests without blocking
- **Simple Usage**: Implement asynchronous processing with just annotations
- **Efficient Resource Utilization**: Thread pool enables thread reuse

---

## How It Works

### @Async Implementation Registration and Discovery Process

The process until Spring executes `@Async` methods asynchronously:

```
1. @EnableAsync declaration
   ↓
2. AsyncConfigurationSelector activation
   ↓
3. ProxyAsyncConfiguration registration
   ↓
4. AsyncAnnotationBeanPostProcessor
   → Wraps @Async methods with proxy
   ↓
5. AsyncExecutionInterceptor
   → Proxy intercepts actual method calls
   ↓
6. Executor selection (priority order)
   1) AsyncConfigurer.getAsyncExecutor()
   2) Bean named "taskExecutor"
   3) SimpleAsyncTaskExecutor (default)
```

### Spring Boot's Default Executor Configuration

Spring Boot automatically registers `ThreadPoolTaskExecutor` through `TaskExecutionAutoConfiguration`.

**Bean Information:**
- **Bean Name**: `applicationTaskExecutor`
- **Actual Implementation**: `ThreadPoolTaskExecutor`
- **Configuration**: `TaskExecutionProperties` (`spring.task.execution.*`)

### ThreadPoolTaskExecutor Default Settings

| Setting | Default Value | Description | Caution |
|---------|--------------|-------------|---------|
| `corePoolSize` | 8 | Minimum number of threads always maintained | Limits async requests per second per server |
| `maxPoolSize` | `Integer.MAX_VALUE` | Maximum number of threads allowed | Meaningless if queueCapacity is unlimited, OOM risk |
| `queueCapacity` | `Integer.MAX_VALUE` | Size of BlockingQueue storing tasks | OOM risk if queue keeps accumulating |
| `keepAliveSeconds` | 60 | How long to keep inactive threads | Cleanup idle threads |
| `allowCoreThreadTimeOut` | true | Whether to apply timeout to core threads | If false, always maintain corePoolSize |
| `rejectedExecutionHandler` | `CallerRunsPolicy` | Policy when queue is full | 4 policies available |
| `waitForTasksToCompleteOnShutdown` | false | Wait for in-progress tasks on shutdown | Consider with Kubernetes |
| `awaitTerminationSeconds` | 0 | Maximum wait time on shutdown | Set below `terminationGracePeriodSeconds` |

### ThreadPoolTaskExecutor Operation Flow

```
Task submission
    ↓
1. Current thread count < corePoolSize?
   ✅ YES → Create new thread and execute immediately
   ❌ NO → Go to step 2
    ↓
2. Queue has available space?
   ✅ YES → Add task to queue
   ❌ NO → Go to step 3
    ↓
3. Current thread count < maxPoolSize?
   ✅ YES → Create additional thread (up to maxPoolSize)
   ❌ NO → Go to step 4
    ↓
4. RejectedExecutionHandler executes
   - AbortPolicy: Throw RejectedExecutionException
   - CallerRunsPolicy: Calling thread executes directly (backpressure)
   - DiscardPolicy: Discard new task
   - DiscardOldestPolicy: Remove oldest task and retry
```

### Exception Handling

Exceptions in `@Async` methods without return values are passed to `AsyncUncaughtExceptionHandler`.

- **Default Handler**: `SimpleAsyncUncaughtExceptionHandler`
- **Behavior**: Only performs error level logging
- **Customization**: Can be changed by implementing `AsyncConfigurer`

---

## Pitfalls

### 1. OOM Risk with Default Configuration

Default `maxPoolSize` and `queueCapacity` are unlimited, potentially causing out of memory.

```yaml
# application.yml - Safe configuration example
spring:
  task:
    execution:
      pool:
        core-size: 10
        max-size: 20
        queue-capacity: 100
        keep-alive: 60s
```

### 2. Self-Invocation Problem

Calling `@Async` method within the same class doesn't go through proxy, executing synchronously.

```kotlin
// ❌ Wrong usage - Doesn't work asynchronously
@Service
class UserService {
    fun process() {
        sendEmail() // Same class call → Synchronous execution
    }
    
    @Async
    fun sendEmail() {
        // ...
    }
}

// ✅ Correct usage - Call from different bean
@Service
class UserService(
    private val emailService: EmailService
) {
    fun process() {
        emailService.sendEmail() // Different bean call → Asynchronous execution
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

### 3. Exception Handling Required

Exceptions in `@Async` methods without return values are not propagated to caller.

```kotlin
// Exception silently disappears
@Async
fun processAsync() {
    throw RuntimeException("Error") // Default handler only logs
}

// Register exception handler with AsyncConfigurer
@Configuration
@EnableAsync
class AsyncConfig : AsyncConfigurer {
    override fun getAsyncUncaughtExceptionHandler(): AsyncUncaughtExceptionHandler {
        return AsyncUncaughtExceptionHandler { ex, method, params ->
            logger.error("Async error in ${method.name}: ${ex.message}", ex)
            // Additional handling: notifications, monitoring, etc.
        }
    }
}
```

### 4. Graceful Shutdown Configuration

Must safely complete in-progress tasks on application shutdown.

```yaml
spring:
  task:
    execution:
      shutdown:
        await-termination: true
        await-termination-period: 30s # Below Kubernetes terminationGracePeriodSeconds
```

---

## Practical Application

### 1. Basic Usage

```kotlin
@SpringBootApplication
@EnableAsync // Enable async
class Application

@Service
class NotificationService {
    
    // Async without return value
    @Async
    fun sendEmail(to: String, subject: String, body: String) {
        Thread.sleep(3000) // Simulate email sending
        println("Email sent to $to")
    }
    
    // Return Future
    @Async
    fun calculateAsync(value: Int): Future<Int> {
        Thread.sleep(2000)
        return AsyncResult(value * 2)
    }
    
    // Return CompletableFuture (recommended)
    @Async
    fun fetchDataAsync(id: Long): CompletableFuture<User> {
        val user = userRepository.findById(id)
        return CompletableFuture.completedFuture(user)
    }
}
```

### 2. Custom Executor Configuration

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

### 3. Multiple Executors

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
    
    @Async("emailExecutor") // Specify particular Executor
    fun sendEmail(to: String) {
        // Send email
    }
    
    @Async("reportExecutor")
    fun generateReport(userId: Long) {
        // Generate report
    }
}
```

### 4. CompletableFuture Composition

```kotlin
@Service
class OrderService(
    private val paymentService: PaymentService,
    private val inventoryService: InventoryService,
    private val notificationService: NotificationService
) {
    
    fun processOrder(orderId: Long) {
        // Execute multiple async tasks in parallel and compose results
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

### 5. Monitoring Setup

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
    
    @Scheduled(fixedRate = 10000) // Every 10 seconds
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

## References

### Official Documentation
- [Spring Framework - Asynchronous Execution](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async)
- [Spring Boot - Task Execution](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.core.spring.task.execution)

### Related TIL
- [Class Loader](./Class%20Loader.md) - Understanding async proxy creation process
