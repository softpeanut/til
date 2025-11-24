# Test Double

## Concept

Test Double is **a collective term for all types of fake objects that replace real objects in tests**. The concept originates from Stunt Double in film production and was formalized by Gerard Meszaros in his book "xUnit Test Patterns".

### Why Use Test Doubles?

1. **Isolate External Dependencies**: Remove dependencies on databases, external APIs, file systems, and other external systems
2. **Improve Test Speed**: Operate faster than real implementations, providing quick feedback
3. **Test Stability**: Write stable tests unaffected by external environment changes
4. **Test Edge Cases**: Easily simulate error situations that are difficult to reproduce in real environments
5. **Independent Unit Tests**: Focus on verifying only the code under test

---

## Types of Test Doubles

### 1. Dummy

**An object that is instantiated but never actually used**

- **Purpose**: Passed to satisfy method signatures
- **Characteristics**: 
  - Never actually called or used
  - Used only to fill parameter positions
  - The simplest form of Test Double

```kotlin
// Dummy example
interface EmailService {
    fun send(email: String, message: String)
}

class DummyEmailService : EmailService {
    override fun send(email: String, message: String) {
        // Does nothing
    }
}

@Test
fun `user creation test`() {
    val emailService = DummyEmailService() // Not actually testing email sending
    val userService = UserService(emailService)
    
    val user = userService.createUser("test@example.com")
    
    assertThat(user.email).isEqualTo("test@example.com")
}
```

---

### 2. Stub

**An object programmed to return predefined answers**

- **Purpose**: Provide predictable output for specific input during tests
- **Characteristics**:
  - Used for State Verification
  - Simply returns values without verifying calls
  - Provides data needed for test scenarios

```kotlin
// Stub example
interface UserRepository {
    fun findById(id: Long): User?
}

class StubUserRepository : UserRepository {
    override fun findById(id: Long): User? {
        // Always returns fixed value
        return if (id == 1L) {
            User(1L, "test@example.com", "Test User")
        } else {
            null
        }
    }
}

@Test
fun `user retrieval test`() {
    val repository = StubUserRepository()
    val service = UserService(repository)
    
    val user = service.getUser(1L)
    
    assertThat(user?.name).isEqualTo("Test User")
}

// Stub using MockK
@Test
fun `MockK Stub example`() {
    val repository = mockk<UserRepository>()
    every { repository.findById(1L) } returns User(1L, "test@example.com", "Test User")
    
    val service = UserService(repository)
    val user = service.getUser(1L)
    
    assertThat(user?.name).isEqualTo("Test User")
}
```

---

### 3. Fake

**A simplified object with working implementation but not suitable for production**

- **Purpose**: Provide lightweight implementation that behaves similarly to real one but optimized for testing
- **Characteristics**:
  - Has actual logic
  - Cannot be used in production (performance, feature limitations, etc.)
  - Useful for integration tests or complex interaction tests

```kotlin
// Fake example
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
fun `user save and retrieval test`() {
    val repository = FakeUserRepository() // Simple memory-based implementation
    val service = UserService(repository)
    
    val user = service.createUser("test@example.com", "Test User")
    val foundUser = service.getUser(user.id!!)
    
    assertThat(foundUser).isEqualTo(user)
}
```

**Real Usage Examples**:
- In-memory databases (H2, SQLite)
- In-memory cache implementations
- Memory-based file systems instead of local file systems

---

### 4. Mock

**An object that specifies expectations for calls and verifies whether it behaves according to those specifications**

- **Purpose**: Verify interactions between objects
- **Characteristics**:
  - Used for Behavior Verification
  - Verifies method call occurrence, call count, argument values, call order, etc.
  - Most commonly used Test Double type

```kotlin
// Mock example
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
            subject = "Welcome!",
            body = "Welcome ${user.name}, thank you for joining us."
        )
    }
}

@Test
fun `sends welcome email when registering user`() {
    val userRepository = mockk<UserRepository>()
    val emailService = mockk<EmailService>(relaxed = true)
    
    every { userRepository.save(any()) } returns User(1L, "test@example.com", "Test User")
    
    val service = UserService(userRepository, emailService)
    service.registerUser("test@example.com", "Test User")
    
    // Verify interaction
    verify(exactly = 1) {
        emailService.send(
            email = "test@example.com",
            subject = "Welcome!",
            body = "Welcome Test User, thank you for joining us."
        )
    }
}

@Test
fun `verify call order`() {
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

**An object that wraps a real object, allowing some methods to actually work while redefining only necessary parts for verification**

- **Purpose**: Verify calls while using real implementation
- **Characteristics**:
  - Executes real methods
  - Can selectively redefine only some methods
  - Tracks and verifies real object behavior

```kotlin
// Spy example
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
        emailService.send(user.email, "Welcome!", "Thank you for joining.")
    }
}

@Test
fun `verify real method call using Spy`() {
    val userRepository = mockk<UserRepository>()
    val emailService = mockk<EmailService>(relaxed = true)
    
    every { userRepository.save(any()) } returns User(1L, "test@example.com", "Test User")
    
    val service = spyk(UserService(userRepository, emailService))
    
    service.createUser("test@example.com", "Test User")
    
    // Verify real method was called
    verify { service.sendWelcomeEmail(any()) }
}

@Test
fun `redefine only some methods with Spy`() {
    val calculator = spyk<Calculator>()
    
    // Use real implementation for add, redefine only multiply
    every { calculator.multiply(any(), any()) } returns 100
    
    assertThat(calculator.add(1, 2)).isEqualTo(3) // Real behavior
    assertThat(calculator.multiply(5, 6)).isEqualTo(100) // Redefined behavior
}
```

---

## Test Double Selection Guide

### When to Use What?

| Test Double | When to Use | Example Situation |
|------------|---------|---------|
| **Dummy** | When passed as parameter but not used | Interface implementation needed but not used in test |
| **Stub** | When you want to control specific return values | Configure Repository to return specific data |
| **Fake** | When lightweight implementation behaving similarly to real one is needed | In-memory database, local file system |
| **Mock** | When you want to verify interactions between objects | Verify method call occurrence, count, order |
| **Spy** | When you want to use real object while controlling only some parts | Legacy code testing, need to verify only some methods |

### Distinguishing Stub vs Mock

Many developers confuse Stub and Mock. The key difference is:

- **Stub**: "Return this value when this method is called" (State Verification)
- **Mock**: "Verify this method was called exactly like this" (Behavior Verification)

```kotlin
// Stub: Only returns value
every { repository.findById(1L) } returns user
val result = service.getUser(1L)
assertThat(result).isEqualTo(user) // Verify result value

// Mock: Verify call
every { emailService.send(any(), any(), any()) } just runs
service.registerUser("test@example.com", "Test")
verify { emailService.send("test@example.com", any(), any()) } // Verify call
```

---

## Best Practices

### 1. Avoid Excessive Mocking

```kotlin
// ❌ Bad: Mock everything
@Test
fun badTest() {
    val a = mockk<ClassA>()
    val b = mockk<ClassB>()
    val c = mockk<ClassC>()
    val d = mockk<ClassD>()
    // Too many Mocks make tests fragile
}

// ✅ Good: Mock only what's necessary
@Test
fun goodTest() {
    val externalService = mockk<ExternalService>() // Mock only external dependencies
    val service = UserService(externalService) // Real object
}
```

### 2. Prefer Fakes

When possible, using Fakes creates better tests than Mocks.

```kotlin
// Fake simulates real behavior, making it resistant to refactoring
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<Long, User>()
    // Implement behavior similar to real one
}
```

### 3. Clear Verification

```kotlin
// ❌ Too strict verification
verify(exactly = 1) {
    emailService.send(
        email = "test@example.com",
        subject = "Welcome!",
        body = "Must be exactly this text" // Test breaks on minor changes
    )
}

// ✅ Verify only essentials
verify {
    emailService.send(
        email = "test@example.com",
        subject = any(),
        body = match { it.contains("Welcome") } // Verify only key content
    )
}
```

### 4. Don't Depend on Implementation Details

```kotlin
// ❌ Depends on implementation details
verify { internalHelper.doSomething() } // Verifying private method

// ✅ Focus on public API and results
val result = service.process()
assertThat(result.status).isEqualTo(Success)
```

---

## References

- Martin Fowler - [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
- Gerard Meszaros - xUnit Test Patterns
- [MockK Documentation](https://mockk.io/)
