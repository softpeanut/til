# Decorator Pattern

## Concept

The Decorator Pattern **allows dynamically adding new responsibilities (features) to an object**.

Using composition instead of inheritance, it wraps existing objects and flexibly extends features at runtime. As one of the GoF structural patterns, the key is "**extending features without inheritance**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **OCP** | Add new features without modifying existing code |
| **Composition over Inheritance** | Use composition for flexibility instead of inheritance |
| **Single Responsibility** | Each decorator handles only one feature |
| **Transparency** | Decorator and original object implement the same interface |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Inheritance-based extension - Class explosion
open class Coffee {
    open fun cost(): Int = 3000
}

class MilkCoffee : Coffee() {
    override fun cost(): Int = super.cost() + 500
}

class MilkSugarCoffee : Coffee() {
    override fun cost(): Int = super.cost() + 500 + 200
}

class MilkSugarWhipCoffee : Coffee() {
    override fun cost(): Int = super.cost() + 500 + 200 + 700
}

// Classes grow exponentially as combinations increase
// Milk, Sugar, Whip, Shot, Syrup... combinations = dozens of classes
```

- New class needed for each feature combination (class explosion)
- Cannot add/remove features at runtime
- Deep inheritance hierarchy becomes hard to maintain

### Value Provided

- **Flexible Extension**: Dynamically add/remove features at runtime
- **Fewer Classes**: Solve with decorator chaining instead of combinations
- **Single Responsibility**: Each decorator handles only one feature
- **Preserve Existing Code**: Extend features without modifying original class

---

## How It Works

### Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│                    ┌────────────────────┐                        │
│                    │   <<interface>>    │                        │
│                    │     Component      │                        │
│                    │  + operation()     │                        │
│                    └────────────────────┘                        │
│                              △                                   │
│               ┌──────────────┴──────────────┐                    │
│               │                             │                    │
│      ┌────────┴────────┐         ┌──────────┴──────────┐         │
│      │ ConcreteComponent│        │     Decorator       │         │
│      │  ─────────────  │         │  ─────────────────  │         │
│      │  + operation()  │         │  - component        │◆────────┤
│      └─────────────────┘         │  + operation()      │         │
│                                  └─────────────────────┘         │
│                                           △                      │
│                              ┌────────────┴────────────┐         │
│                              │                         │         │
│                    ┌─────────┴─────────┐     ┌─────────┴─────────┐
│                    │  DecoratorA       │     │  DecoratorB       │
│                    │  + operation()    │     │  + operation()    │
│                    └───────────────────┘     └───────────────────┘
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Component** | Defines base interface |
| **ConcreteComponent** | Original object implementing base functionality |
| **Decorator** | Wraps Component and implements same interface |
| **ConcreteDecorator** | Decorator implementing additional features |

### Basic Implementation

```kotlin
// Component interface
interface Coffee {
    fun cost(): Int
    fun description(): String
}

// ConcreteComponent - Basic coffee
class Espresso : Coffee {
    override fun cost(): Int = 3000
    override fun description(): String = "Espresso"
}

class Americano : Coffee {
    override fun cost(): Int = 3500
    override fun description(): String = "Americano"
}

// Decorator abstract class
abstract class CoffeeDecorator(
    protected val coffee: Coffee
) : Coffee {
    override fun cost(): Int = coffee.cost()
    override fun description(): String = coffee.description()
}

// ConcreteDecorators
class MilkDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
    override fun cost(): Int = super.cost() + 500
    override fun description(): String = "${super.description()} + Milk"
}

class SugarDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
    override fun cost(): Int = super.cost() + 200
    override fun description(): String = "${super.description()} + Sugar"
}

class WhipDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
    override fun cost(): Int = super.cost() + 700
    override fun description(): String = "${super.description()} + Whipped Cream"
}

// Usage - Decorator chaining
val espresso: Coffee = Espresso()
println("${espresso.description()}: ${espresso.cost()}won")
// Espresso: 3000won

val latte: Coffee = MilkDecorator(Espresso())
println("${latte.description()}: ${latte.cost()}won")
// Espresso + Milk: 3500won

val sweetLatte: Coffee = SugarDecorator(MilkDecorator(Espresso()))
println("${sweetLatte.description()}: ${sweetLatte.cost()}won")
// Espresso + Milk + Sugar: 3700won

val fancyCoffee: Coffee = WhipDecorator(SugarDecorator(MilkDecorator(Americano())))
println("${fancyCoffee.description()}: ${fancyCoffee.cost()}won")
// Americano + Milk + Sugar + Whipped Cream: 4900won
```

### Wrapping Structure Visualization

```
┌─────────────────────────────────────────────────────┐
│  WhipDecorator                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  SugarDecorator                             │    │
│  │  ┌───────────────────────────────────────┐  │    │
│  │  │  MilkDecorator                        │  │    │
│  │  │  ┌─────────────────────────────────┐  │  │    │
│  │  │  │  Americano (ConcreteComponent)  │  │  │    │
│  │  │  │  cost() = 3500                  │  │  │    │
│  │  │  └─────────────────────────────────┘  │  │    │
│  │  │  cost() = 3500 + 500 = 4000           │  │    │
│  │  └───────────────────────────────────────┘  │    │
│  │  cost() = 4000 + 200 = 4200                 │    │
│  └─────────────────────────────────────────────┘    │
│  cost() = 4200 + 700 = 4900                         │
└─────────────────────────────────────────────────────┘
```

---

## Pitfalls

### 1. Order-Dependent Results

Results may vary depending on decorator order.

```kotlin
// Discount → Tax vs Tax → Discount
val price = 10000

// 10% discount then 10% tax
val discountFirst = DiscountDecorator(TaxDecorator(BasePrice(price)))
// 10000 * 0.9 * 1.1 = 9900

// 10% tax then 10% discount
val taxFirst = TaxDecorator(DiscountDecorator(BasePrice(price)))
// 10000 * 1.1 * 0.9 = 9900 (same in this case, but may differ with other logic)
```

### 2. Many Decorators Increase Complexity

Too many nested decorators make debugging difficult.

```kotlin
// ❌ Excessive nesting - Hard to debug
val complex = D(C(B(A(component))))

// ✅ Use with Builder pattern for readability
val order = CoffeeBuilder()
    .base(Americano())
    .addMilk()
    .addSugar()
    .addWhip()
    .build()
```

### 3. Duplicate Decorator Application

Same decorator may be applied multiple times unintentionally.

```kotlin
// Double milk added (intentional or mistake?)
val doubleMilk = MilkDecorator(MilkDecorator(Espresso()))
```

### 4. Watch Type Checks

Decorated object is different type from original.

```kotlin
val coffee: Coffee = MilkDecorator(Espresso())

coffee is Espresso  // false - MilkDecorator type
coffee is Coffee    // true - Same interface
```

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Dynamically add/remove features | Stream processing, Logging, Caching |
| Many feature combinations | Beverage options, Pizza toppings |
| Hard to solve with inheritance | Extending final class, Multiple feature combinations |
| Cannot modify existing code | Extending library classes |

### Java I/O Stream Example

```kotlin
// Java's representative Decorator pattern usage
import java.io.*

// Add buffering and line reading to base stream
val reader = BufferedReader(  // Decorator
    InputStreamReader(         // Decorator
        FileInputStream("file.txt")  // ConcreteComponent
    )
)

// Compression + Buffering stream
val outputStream = BufferedOutputStream(  // Decorator
    GZIPOutputStream(                      // Decorator
        FileOutputStream("file.gz")        // ConcreteComponent
    )
)
```

### Spring Environment: Service Layer Decorating

```kotlin
// Component interface
interface UserService {
    fun getUser(id: Long): User
}

// ConcreteComponent - Actual service
@Service
@Primary
class UserServiceImpl(
    private val userRepository: UserRepository
) : UserService {
    override fun getUser(id: Long): User {
        return userRepository.findById(id)
            .orElseThrow { NotFoundException("User not found: $id") }
    }
}

// Decorator - Add caching feature
@Service
class CachingUserServiceDecorator(
    @Qualifier("userServiceImpl")
    private val delegate: UserService,
    private val cache: Cache
) : UserService {
    
    override fun getUser(id: Long): User {
        return cache.get(id) {
            delegate.getUser(id)
        }
    }
}

// Decorator - Add logging feature
@Service
class LoggingUserServiceDecorator(
    @Qualifier("cachingUserServiceDecorator")
    private val delegate: UserService,
    private val logger: Logger
) : UserService {
    
    override fun getUser(id: Long): User {
        logger.info("getUser called with id: $id")
        val result = delegate.getUser(id)
        logger.info("getUser result: $result")
        return result
    }
}
```

### Kotlin Extension Functions Combination

```kotlin
// In Kotlin, can be used with extension functions
interface TextProcessor {
    fun process(text: String): String
}

class BasicProcessor : TextProcessor {
    override fun process(text: String): String = text
}

class UpperCaseDecorator(
    private val processor: TextProcessor
) : TextProcessor {
    override fun process(text: String): String = 
        processor.process(text).uppercase()
}

class TrimDecorator(
    private val processor: TextProcessor
) : TextProcessor {
    override fun process(text: String): String = 
        processor.process(text).trim()
}

// DSL style builder
class TextProcessorBuilder {
    private var processor: TextProcessor = BasicProcessor()
    
    fun uppercase() = apply { processor = UpperCaseDecorator(processor) }
    fun trim() = apply { processor = TrimDecorator(processor) }
    fun build(): TextProcessor = processor
}

fun textProcessor(block: TextProcessorBuilder.() -> Unit): TextProcessor {
    return TextProcessorBuilder().apply(block).build()
}

// Usage
val processor = textProcessor {
    trim()
    uppercase()
}

println(processor.process("  hello world  "))  // "HELLO WORLD"
```

### Before/After Comparison

```kotlin
// ❌ Before: Inheritance-based - Class explosion
open class Notifier {
    open fun send(message: String) { /* Basic notification */ }
}

class EmailNotifier : Notifier() { /* Email */ }
class SlackNotifier : Notifier() { /* Slack */ }
class EmailSlackNotifier : Notifier() { /* Email + Slack */ }
class EmailSlackSmsNotifier : Notifier() { /* Email + Slack + SMS */ }
// ... Class needed for each combination

// ✅ After: Decorator Pattern - Flexible combinations
interface Notifier {
    fun send(message: String)
}

class BasicNotifier : Notifier {
    override fun send(message: String) = println("Basic notification: $message")
}

abstract class NotifierDecorator(
    protected val notifier: Notifier
) : Notifier

class EmailDecorator(notifier: Notifier) : NotifierDecorator(notifier) {
    override fun send(message: String) {
        notifier.send(message)
        println("Email sent: $message")
    }
}

class SlackDecorator(notifier: Notifier) : NotifierDecorator(notifier) {
    override fun send(message: String) {
        notifier.send(message)
        println("Slack sent: $message")
    }
}

class SmsDecorator(notifier: Notifier) : NotifierDecorator(notifier) {
    override fun send(message: String) {
        notifier.send(message)
        println("SMS sent: $message")
    }
}

// Create desired combination at runtime
val allChannels = SmsDecorator(SlackDecorator(EmailDecorator(BasicNotifier())))
allChannels.send("Urgent notification")
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Proxy** | Same interface for **access control** (lazy loading, permission check), Decorator is for **adding features** |
| **Composite** | Represents **part-whole** relationships in tree structure, Decorator is **linear wrapping** |
| **Strategy** | **Replaces** entire algorithm, Decorator **adds** to existing behavior |

### Decorator vs Proxy

| Comparison | Decorator | Proxy |
|------------|-----------|-------|
| Purpose | Add/extend features | Access control |
| Client Awareness | Often directly constructs decorator chain | Unaware of proxy existence |
| Wrapping Count | Multiple chaining possible | Usually one |
| Use Case | Java I/O Stream | Lazy loading, Caching proxy |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Starbuzz Coffee example

### Recommended Sites
- [Refactoring Guru - Decorator Pattern](https://refactoring.guru/design-patterns/decorator)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Composite Pattern](./Composite%20Pattern.en.md)
- [Strategy Pattern](../Behavioral/Strategy%20Pattern.en.md)
