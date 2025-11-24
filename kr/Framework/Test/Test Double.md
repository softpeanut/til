# Test Double

## 개념

Test Double은 **테스트에서 실제 객체를 대신하는 모든 종류의 가짜 객체**를 총칭하는 용어입니다. 영화 촬영에서 배우를 대신하는 스턴트 더블(Stunt Double)에서 유래한 개념으로, Gerard Meszaros가 그의 저서 "xUnit Test Patterns"에서 정립했습니다.

### 왜 Test Double을 사용하는가?

1. **외부 의존성 격리**: 데이터베이스, 외부 API, 파일 시스템 등 외부 시스템과의 의존성을 제거
2. **테스트 속도 향상**: 실제 구현보다 빠르게 동작하여 빠른 피드백 제공
3. **테스트 안정성**: 외부 환경 변화에 영향받지 않는 안정적인 테스트 작성
4. **엣지 케이스 테스트**: 실제 환경에서 재현하기 어려운 에러 상황을 쉽게 시뮬레이션
5. **독립적인 단위 테스트**: 테스트 대상 코드만 집중해서 검증 가능

---

## Test Double의 종류

### 1. Dummy

**인스턴스화는 되지만 실제로 사용되지 않는 객체**

- **목적**: 메서드 시그니처를 만족시키기 위해 전달되는 객체
- **특징**: 
  - 실제로 호출되거나 사용되지 않음
  - 파라미터 자리를 채우는 용도
  - 가장 단순한 형태의 Test Double

```kotlin
// Dummy 예시
interface EmailService {
    fun send(email: String, message: String)
}

class DummyEmailService : EmailService {
    override fun send(email: String, message: String) {
        // 아무것도 하지 않음
    }
}

@Test
fun `사용자 생성 테스트`() {
    val emailService = DummyEmailService() // 실제로 이메일 전송은 테스트하지 않음
    val userService = UserService(emailService)
    
    val user = userService.createUser("test@example.com")
    
    assertThat(user.email).isEqualTo("test@example.com")
}
```

---

### 2. Stub

**미리 정의된 답변을 반환하도록 프로그래밍된 객체**

- **목적**: 테스트 중 특정 입력에 대해 예측 가능한 출력을 제공
- **특징**:
  - 상태 기반 검증(State Verification)에 사용
  - 호출에 대한 검증 없이 단순히 값만 반환
  - 테스트 시나리오에 필요한 데이터를 제공

```kotlin
// Stub 예시
interface UserRepository {
    fun findById(id: Long): User?
}

class StubUserRepository : UserRepository {
    override fun findById(id: Long): User? {
        // 항상 고정된 값 반환
        return if (id == 1L) {
            User(1L, "test@example.com", "Test User")
        } else {
            null
        }
    }
}

@Test
fun `사용자 조회 테스트`() {
    val repository = StubUserRepository()
    val service = UserService(repository)
    
    val user = service.getUser(1L)
    
    assertThat(user?.name).isEqualTo("Test User")
}

// MockK를 사용한 Stub
@Test
fun `MockK Stub 예시`() {
    val repository = mockk<UserRepository>()
    every { repository.findById(1L) } returns User(1L, "test@example.com", "Test User")
    
    val service = UserService(repository)
    val user = service.getUser(1L)
    
    assertThat(user?.name).isEqualTo("Test User")
}
```

---

### 3. Fake

**실제 동작하는 구현을 가지고 있지만, 프로덕션에는 적합하지 않은 단순화된 객체**

- **목적**: 실제와 유사하게 동작하지만 테스트에 최적화된 경량 구현 제공
- **특징**:
  - 실제 로직을 가지고 있음
  - 프로덕션 환경에서는 사용할 수 없음 (성능, 기능 제한 등)
  - 통합 테스트나 복잡한 상호작용 테스트에 유용

```kotlin
// Fake 예시
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<Long, User>()
    private var currentId = 1L
    
    override fun save(user: User): User {
        val id = currentId++
        val savedUser = user.copy(id = id)
        users[id] = savedUser
        return savedUser
    }
    
    override fun findById(id: Long): User? {
        return users[id]
    }
    
    override fun findAll(): List<User> {
        return users.values.toList()
    }
}

@Test
fun `사용자 저장 및 조회 테스트`() {
    val repository = FakeUserRepository() // 메모리 기반 간단한 구현
    val service = UserService(repository)
    
    val user = service.createUser("test@example.com", "Test User")
    val foundUser = service.getUser(user.id!!)
    
    assertThat(foundUser).isEqualTo(user)
}
```

**실제 사용 예시**:
- 인메모리 데이터베이스 (H2, SQLite)
- 인메모리 캐시 구현
- 로컬 파일 시스템 대신 메모리 기반 파일 시스템

---

### 4. Mock

**호출에 대한 기대값을 명세하고, 해당 명세대로 동작했는지 검증하는 객체**

- **목적**: 객체 간 상호작용(Interaction) 검증
- **특징**:
  - 행위 기반 검증(Behavior Verification)에 사용
  - 메서드 호출 여부, 호출 횟수, 인자값, 호출 순서 등을 검증
  - 가장 많이 사용되는 Test Double 유형

```kotlin
// Mock 예시
interface EmailService {
    fun send(email: String, subject: String, body: String)
}

class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService
) {
    fun registerUser(email: String, name: String) {
        val user = userRepository.save(User(email = email, name = name))
        emailService.send(
            email = user.email,
            subject = "환영합니다!",
            body = "${user.name}님, 회원가입을 환영합니다."
        )
    }
}

@Test
fun `사용자 등록 시 환영 이메일을 발송한다`() {
    val userRepository = mockk<UserRepository>()
    val emailService = mockk<EmailService>(relaxed = true)
    
    every { userRepository.save(any()) } returns User(1L, "test@example.com", "Test User")
    
    val service = UserService(userRepository, emailService)
    service.registerUser("test@example.com", "Test User")
    
    // 상호작용 검증
    verify(exactly = 1) {
        emailService.send(
            email = "test@example.com",
            subject = "환영합니다!",
            body = "Test User님, 회원가입을 환영합니다."
        )
    }
}

@Test
fun `호출 순서 검증`() {
    val emailService = mockk<EmailService>(relaxed = true)
    val smsService = mockk<SmsService>(relaxed = true)
    
    val service = NotificationService(emailService, smsService)
    service.sendNotifications("test@example.com", "01012345678")
    
    verifyOrder {
        emailService.send(any(), any(), any())
        smsService.send(any(), any())
    }
}
```

---

### 5. Spy

**실제 객체를 감싸서 일부 메서드는 실제로 동작하고, 필요한 부분만 재정의하여 검증하는 객체**

- **목적**: 실제 구현을 사용하면서 호출 여부도 검증
- **특징**:
  - 실제 메서드를 실행
  - 일부 메서드만 선택적으로 재정의 가능
  - 실제 객체의 동작을 추적하고 검증

```kotlin
// Spy 예시
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService
) {
    fun createUser(email: String, name: String): User {
        val user = userRepository.save(User(email = email, name = name))
        sendWelcomeEmail(user)
        return user
    }
    
    protected fun sendWelcomeEmail(user: User) {
        emailService.send(user.email, "환영합니다!", "가입을 환영합니다.")
    }
}

@Test
fun `Spy를 사용한 실제 메서드 호출 검증`() {
    val userRepository = mockk<UserRepository>()
    val emailService = mockk<EmailService>(relaxed = true)
    
    every { userRepository.save(any()) } returns User(1L, "test@example.com", "Test User")
    
    val service = spyk(UserService(userRepository, emailService))
    
    service.createUser("test@example.com", "Test User")
    
    // 실제 메서드가 호출되었는지 검증
    verify { service.sendWelcomeEmail(any()) }
}

@Test
fun `Spy로 일부 메서드만 재정의`() {
    val calculator = spyk<Calculator>()
    
    // add는 실제 구현 사용, multiply만 재정의
    every { calculator.multiply(any(), any()) } returns 100
    
    assertThat(calculator.add(1, 2)).isEqualTo(3) // 실제 동작
    assertThat(calculator.multiply(5, 6)).isEqualTo(100) // 재정의된 동작
}
```

---

## Test Double 선택 가이드

### 언제 무엇을 사용해야 할까?

| Test Double | 사용 시기 | 예시 상황 |
|------------|---------|---------|
| **Dummy** | 파라미터로 전달되지만 사용되지 않을 때 | 인터페이스 구현이 필요하지만 테스트에서 사용하지 않는 경우 |
| **Stub** | 특정 값을 반환하도록 제어하고 싶을 때 | Repository에서 특정 데이터를 반환하도록 설정 |
| **Fake** | 실제와 유사하게 동작하는 경량 구현이 필요할 때 | 인메모리 데이터베이스, 로컬 파일 시스템 |
| **Mock** | 객체 간 상호작용을 검증하고 싶을 때 | 메서드 호출 여부, 횟수, 순서 검증 |
| **Spy** | 실제 객체를 사용하면서 일부만 제어하고 싶을 때 | 레거시 코드 테스트, 일부 메서드만 검증 필요 |

### Stub vs Mock 구분하기

많은 개발자들이 Stub과 Mock을 혼동합니다. 핵심 차이는:

- **Stub**: "이 메서드가 호출되면 이 값을 반환해" (상태 검증)
- **Mock**: "이 메서드가 정확히 이렇게 호출되었는지 확인해" (행위 검증)

```kotlin
// Stub: 값만 반환
every { repository.findById(1L) } returns user
val result = service.getUser(1L)
assertThat(result).isEqualTo(user) // 결과값 검증

// Mock: 호출 검증
every { emailService.send(any(), any(), any()) } just runs
service.registerUser("test@example.com", "Test")
verify { emailService.send("test@example.com", any(), any()) } // 호출 검증
```

---

## Best Practices

### 1. 과도한 Mocking 지양

```kotlin
// ❌ 나쁜 예: 모든 것을 Mock
@Test
fun badTest() {
    val a = mockk<ClassA>()
    val b = mockk<ClassB>()
    val c = mockk<ClassC>()
    val d = mockk<ClassD>()
    // 너무 많은 Mock은 테스트를 취약하게 만듦
}

// ✅ 좋은 예: 필요한 것만 Mock
@Test
fun goodTest() {
    val externalService = mockk<ExternalService>() // 외부 의존성만 Mock
    val service = UserService(externalService) // 실제 객체
}
```

### 2. Fake 우선 고려

가능하다면 Mock보다 Fake를 사용하는 것이 더 나은 테스트를 만듭니다.

```kotlin
// Fake는 실제 동작을 시뮬레이션하므로 리팩토링에 강함
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<Long, User>()
    // 실제와 유사한 동작 구현
}
```

### 3. 검증은 명확하게

```kotlin
// ❌ 너무 엄격한 검증
verify(exactly = 1) {
    emailService.send(
        email = "test@example.com",
        subject = "환영합니다!",
        body = "정확히 이 텍스트여야 함" // 사소한 변경에 테스트 깨짐
    )
}

// ✅ 핵심만 검증
verify {
    emailService.send(
        email = "test@example.com",
        subject = any(),
        body = match { it.contains("환영") } // 핵심 내용만 검증
    )
}
```

### 4. 테스트가 구현 세부사항에 의존하지 않도록

```kotlin
// ❌ 구현 세부사항에 의존
verify { internalHelper.doSomething() } // private 메서드 검증

// ✅ 공개 API와 결과에 집중
val result = service.process()
assertThat(result.status).isEqualTo(Success)
```

---

## 참고 자료

- Martin Fowler - [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- Gerard Meszaros - xUnit Test Patterns
- [MockK Documentation](https://mockk.io/)