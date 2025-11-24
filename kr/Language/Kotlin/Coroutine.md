
# Coroutine

## 개념

Coroutine은 **중단 가능한(suspendable) 경량 쓰레드**이다.

일반 함수는 시작하면 끝까지 실행되어야 하지만, Coroutine은 실행 중 일시 중단(suspend)했다가 나중에 재개할 수 있다. OS 쓰레드를 직접 생성하지 않고도 비동기 작업을 효율적으로 처리할 수 있다.

### 핵심 특징

- **경량성**: 수천 개의 코루틴을 생성해도 쓰레드보다 메모리와 성능 부담이 적다
- **중단 가능**: `suspend` 함수를 통해 쓰레드를 블로킹하지 않고 작업을 일시 중단할 수 있다
- **구조화된 동시성**: `CoroutineScope`로 생명주기를 관리하며, Scope가 종료되면 하위 코루틴도 자동으로 취소된다

---

## 왜 필요한가?

### 해결하려는 문제

비동기 작업을 처리할 때 쓰레드를 직접 관리하거나 콜백 패턴을 사용하면 복잡도가 증가하고 가독성이 떨어진다.

### 기존 방식의 한계

1. **쓰레드(Thread)**: 생성 비용이 크고 컨텍스트 스위칭 오버헤드가 발생한다
2. **콜백(Callback)**: 중첩이 깊어지면 콜백 지옥(Callback Hell)이 발생한다
3. **RxJava/Future**: 체이닝이 복잡하고 러닝 커브가 높다

### 제공하는 가치

- **가독성**: 동기 코드처럼 작성하면서 비동기로 실행된다
- **효율성**: 쓰레드보다 훨씬 적은 리소스로 수많은 동시 작업을 처리한다
- **안전성**: 구조화된 동시성으로 메모리 누수와 작업 누락을 방지한다

---

## 동작 원리

### 핵심 메커니즘

Coroutine은 **CPS(Continuation-Passing Style) 변환**을 통해 `suspend` 함수를 상태 머신으로 변환한다. 컴파일 시점에 함수가 여러 단계로 분할되며, 각 중단 지점마다 상태를 저장하고 복원한다.

### 처리 과정

1. **suspend 함수 호출**: 현재 상태를 `Continuation` 객체에 저장
2. **중단**: 제어권을 반환하고 쓰레드는 다른 작업 수행
3. **재개**: 저장된 `Continuation`에서 이어서 실행

### 주요 컴포넌트

- **Continuation**: 중단된 지점의 상태와 재개 정보를 담는 객체
- **Dispatcher**: 코루틴이 실행될 쓰레드를 결정 (Main, IO, Default 등)
- **CoroutineScope**: 코루틴의 생명주기를 관리하는 범위

```kotlin
// 컴파일 전
suspend fun fetchData(): String {
    delay(1000)
    return "Data"
}

// 컴파일 후 (개념적)
fun fetchData(continuation: Continuation<String>): Any {
    when (continuation.label) {
        0 -> {
            continuation.label = 1
            return delay(1000, continuation) // COROUTINE_SUSPENDED 반환
        }
        1 -> {
            return "Data"
        }
    }
}
```

---

## 주의사항

### 1. GlobalScope 사용 금지

생명주기 관리가 불가능하여 메모리 누수가 발생한다.

```kotlin
// ❌ 잘못된 예
GlobalScope.launch {
    // 앱이 종료되어도 계속 실행될 수 있음
    fetchData()
}

// ✅ 올바른 예
viewModelScope.launch {
    // ViewModel 생명주기에 맞춰 자동 취소
    fetchData()
}
```

### 2. 블로킹 작업을 Main 쓰레드에서 실행

UI가 멈추는 원인이 된다.

```kotlin
// ❌ 잘못된 예
viewModelScope.launch {
    val data = readFromFile() // 블로킹 I/O
}

// ✅ 올바른 예
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) {
        readFromFile() // I/O 쓰레드에서 실행
    }
}
```

### 3. suspend 함수를 일반 함수에서 호출

`suspend` 함수는 코루틴 스코프나 다른 `suspend` 함수 내에서만 호출 가능하다.

```kotlin
// ❌ 잘못된 예
fun loadData() {
    delay(1000) // 컴파일 에러
}

// ✅ 올바른 예
suspend fun loadData() {
    delay(1000)
}

// 또는
fun loadData() {
    CoroutineScope(Dispatchers.Main).launch {
        delay(1000)
    }
}
```

### Best Practices

- 적절한 Scope 사용 (`viewModelScope`, `lifecycleScope` 등)
- I/O 작업은 `Dispatchers.IO`에서 실행
- CPU 집약적 작업은 `Dispatchers.Default`에서 실행
- 구조화된 동시성(Structured Concurrency) 원칙 준수

---

## 실전 적용

### 기본 사용법

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        delay(1000L) // 1초 동안 일시 중단 (쓰레드 블로킹 아님)
        println("World!")
    }
    println("Hello,")
}

// 결과
// Hello,
// World!
```

### 주요 사용 패턴

#### 패턴 1: 병렬 실행

여러 비동기 작업을 동시에 실행하고 모든 결과를 기다릴 때

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

#### 패턴 2: 순차 실행

이전 작업의 결과가 필요한 경우

```kotlin
suspend fun processOrder(orderId: String) {
    val order = fetchOrder(orderId)        // 1. 주문 조회
    val payment = processPayment(order)     // 2. 결제 처리
    val shipment = createShipment(payment)  // 3. 배송 생성
    notifyUser(shipment)                    // 4. 사용자 알림
}
```

#### 패턴 3: 타임아웃 처리

장시간 실행되는 작업에 제한 시간 설정

```kotlin
suspend fun fetchWithTimeout() {
    try {
        withTimeout(5000) { // 5초 제한
            fetchData()
        }
    } catch (e: TimeoutCancellationException) {
        println("작업 시간 초과")
    }
}
```

#### 패턴 4: 블로킹/논블로킹 조합 이해

비동기 실행 패턴의 조합

```kotlin
// 논블로킹 + 동기
runBlocking {
    delay(100)        // (1) 논블로킹
    test()            // (2) 논블로킹
    println("Done")   // (3) 1, 2가 모두 끝난 후 실행
}
suspend fun test() { delay(200) }

// 논블로킹 + 비동기
runBlocking {
    val result1 = async { delay(100) }  // (1) 비동기 시작
    val result2 = async { delay(200) }  // (2) 비동기 시작
    println("Done")                     // (3) 먼저 실행
    result1.await()                     // (4) 1이 끝날 때까지 대기
    result2.await()                     // (5) 2가 끝날 때까지 대기
}

// 블로킹 + 비동기
runBlocking {
    val result = launch {
        withContext(Dispatchers.Default) { 
            Thread.sleep(100)  // 블로킹하지만 별도 쓰레드
        }
    }
    println("Done")    // (2) 먼저 실행 가능
    result.join()      // (3) result가 끝날 때까지 대기
}
```

### Before/After 비교

```kotlin
// ❌ 콜백 방식
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

// ✅ 코루틴 방식
suspend fun fetchData(): Data {
    return api.getData() // 간결하고 직관적
}
```

---

## 참고 자료

### 공식 문서
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)

### 추천 아티클
- [Understanding Kotlin Coroutines](https://www.silica.io/understanding-kotlin-coroutines/5/) - Coroutine 동작 원리 상세 설명

### 관련 TIL
- [JVM](../JVM/JVM.md)