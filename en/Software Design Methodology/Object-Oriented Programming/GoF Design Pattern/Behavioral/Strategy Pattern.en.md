# Strategy Pattern

## Concept

The Strategy Pattern **encapsulates algorithms (behaviors) as objects, allowing them to be dynamically interchangeable at runtime**.

When multiple algorithms exist to solve the same problem, they are abstracted through an interface, and the specific implementation can be selected at execution time. As one of the GoF behavioral patterns, it follows the principle of "**encapsulate what varies**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Encapsulation** | Separate changeable algorithms into separate objects |
| **Composition over Inheritance** | Use composition for flexibility instead of inheritance |
| **OCP** | Extend without modifying existing code when adding new strategies |
| **SRP** | Each strategy is responsible for only one algorithm |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Conditional branching - Need to modify code for each new payment method
fun pay(type: String, amount: Int) {
    when (type) {
        "kakao" -> println("KakaoPay payment: $amount")
        "card" -> println("Credit card payment: $amount")
        "naver" -> println("NaverPay payment: $amount") // Modify each time you add
        // ...keeps growing
    }
}
```

- Long conditionals reduce readability and maintainability
- Adding new policies requires modifying existing code (OCP violation)
- Difficult to test algorithms

### Value Provided

- **Extensibility**: No need to modify existing code when adding new strategies
- **Testability**: Each strategy can be tested independently
- **Flexibility**: Strategies can be swapped at runtime
- **Single Responsibility**: Each strategy handles only one algorithm

---

## How It Works

### Structure

```
┌─────────────────────────────────────────────────────┐
│                     Context                         │
│  ┌───────────────────────────────────────────────┐  │
│  │  - strategy: Strategy                         │  │
│  │  + execute()                                  │  │
│  └───────────────────────────────────────────────┘  │
│                        │                            │
│                        ▼ uses                       │
│         ┌──────────────────────────┐                │
│         │    <<interface>>         │                │
│         │       Strategy           │                │
│         │  + algorithm()           │                │
│         └──────────────────────────┘                │
│                   △                                 │
│         ┌─────────┼─────────┐                       │
│         │         │         │                       │
│    ┌────┴────┐ ┌──┴───┐ ┌───┴────┐                  │
│    │Strategy │ │Strategy│ │Strategy│                │
│    │    A    │ │   B    │ │   C    │                │
│    └─────────┘ └────────┘ └────────┘                │
└─────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Strategy** | Defines common interface for algorithms |
| **ConcreteStrategy** | Actual algorithm implementation |
| **Context** | Execution environment that uses strategy, holds strategy reference |

### Basic Implementation

```kotlin
// Strategy interface
interface PaymentStrategy {
    fun pay(amount: Int)
}

// ConcreteStrategy implementations
class KakaoPayStrategy : PaymentStrategy {
    override fun pay(amount: Int) {
        println("KakaoPay payment: ${amount}won")
    }
}

class CreditCardStrategy : PaymentStrategy {
    override fun pay(amount: Int) {
        println("Credit card payment: ${amount}won")
    }
}

// Context
class PaymentProcessor(private var strategy: PaymentStrategy) {
    fun changeStrategy(newStrategy: PaymentStrategy) {
        strategy = newStrategy
    }

    fun process(amount: Int) {
        strategy.pay(amount)
    }
}

// Usage
val processor = PaymentProcessor(KakaoPayStrategy())
processor.process(10000)  // KakaoPay payment: 10000won

processor.changeStrategy(CreditCardStrategy())
processor.process(20000)  // Credit card payment: 20000won
```

---

## Pitfalls

### 1. Strategy Selection Logic Location

Logic to select a strategy must exist somewhere. Either Context selects directly, or it's injected from outside (Factory, DI Container).

```kotlin
// ❌ Selection by conditional in Context - defeats purpose of strategy pattern
class PaymentProcessor {
    fun process(type: String, amount: Int) {
        val strategy = when (type) {
            "kakao" -> KakaoPayStrategy()
            "card" -> CreditCardStrategy()
            else -> throw IllegalArgumentException()
        }
        strategy.pay(amount)
    }
}

// ✅ Strategy injected from outside
class PaymentProcessor(private val strategy: PaymentStrategy) {
    fun process(amount: Int) = strategy.pay(amount)
}
```

### 2. Watch for Class Explosion

Creating a class for each strategy can lead to class proliferation. In Kotlin, this can be mitigated using function types.

### 3. Don't Over-apply to Simple Cases

If there are only 1-2 strategies with low change probability, simple conditionals may be better.

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Various algorithm selection | Sorting algorithms, Compression methods |
| Frequently changing policies | Discount policies, Shipping cost calculation |
| Behavior branching by condition | Payment methods, Authentication methods |
| Test replacement needed | Replace external API calls with Mocks |

### Spring Environment: Map-based DI

```kotlin
// Strategy interface
interface NotificationSender {
    fun send(message: String)
}

// Register implementations as Beans (key is Bean name)
@Service("email")
class EmailNotificationSender : NotificationSender {
    override fun send(message: String) {
        println("Email: $message")
    }
}

@Service("sms")
class SmsNotificationSender : NotificationSender {
    override fun send(message: String) {
        println("SMS: $message")
    }
}

// Inject all strategies via Map
@Service
class NotificationService(
    private val strategies: Map<String, NotificationSender>
) {
    fun notify(type: String, message: String) {
        strategies[type]?.send(message)
            ?: error("Unsupported type: $type")
    }
}

// Usage
notificationService.notify("email", "Hello")  // Email: Hello
notificationService.notify("sms", "Hi")       // SMS: Hi
```

**Advantage**: When adding new strategies, just register implementation with `@Service` and it's automatically included in Map

### Kotlin Functional Style

Using **function types** as strategies instead of interfaces + classes is more concise.

```kotlin
// Define function type as strategy
typealias PricingStrategy = (Int) -> Int

// Strategies (defined as lambdas)
val defaultPricing: PricingStrategy = { price -> price }
val vipPricing: PricingStrategy = { price -> (price * 0.8).toInt() }
val studentPricing: PricingStrategy = { price -> (price * 0.9).toInt() }

// Context
class PriceCalculator(private var strategy: PricingStrategy) {
    fun change(strategy: PricingStrategy) {
        this.strategy = strategy
    }
    
    fun calculate(price: Int) = strategy(price)
}

// Usage
val calculator = PriceCalculator(defaultPricing)
println(calculator.calculate(10000))  // 10000

calculator.change(vipPricing)
println(calculator.calculate(10000))  // 8000
```

### Before/After Comparison

```kotlin
// ❌ Before: Conditional-based
class DiscountService {
    fun calculate(type: String, price: Int): Int {
        return when (type) {
            "vip" -> (price * 0.8).toInt()
            "student" -> (price * 0.9).toInt()
            "event" -> (price * 0.7).toInt()  // Modify each time event is added
            else -> price
        }
    }
}

// ✅ After: Strategy Pattern
interface DiscountStrategy {
    fun apply(price: Int): Int
}

class VipDiscount : DiscountStrategy {
    override fun apply(price: Int) = (price * 0.8).toInt()
}

class StudentDiscount : DiscountStrategy {
    override fun apply(price: Int) = (price * 0.9).toInt()
}

// Just add class when adding new discount policy
class EventDiscount : DiscountStrategy {
    override fun apply(price: Int) = (price * 0.7).toInt()
}

class DiscountService(private val strategy: DiscountStrategy) {
    fun calculate(price: Int) = strategy.apply(price)
}
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Template Method** | Inheritance-based with fixed algorithm skeleton, only certain steps overridden |
| **State** | Strategy changes **automatically** based on internal state (State drives transition, not Context) |
| **Command** | Objectifies requests themselves (for undo/logging), Strategy is for algorithm swapping |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Introduces Strategy Pattern first

### Recommended Sites
- [Refactoring Guru - Strategy Pattern](https://refactoring.guru/design-patterns/strategy)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
