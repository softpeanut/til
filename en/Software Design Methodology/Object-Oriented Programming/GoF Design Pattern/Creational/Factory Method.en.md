# Factory Method Pattern

## Concept

The Factory Method Pattern **delegates object creation to subclasses to determine the type of object to create**.

Instead of creating objects directly with `new`, creating them through factory methods encapsulates the creation logic. As one of the GoF creational patterns, it follows the principle of "**depend on interfaces, not concrete classes**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Encapsulation** | Hide object creation logic inside factory method |
| **DIP** | Caller depends on interface, not concrete class |
| **OCP** | Extend without modifying existing code when adding new products |
| **SRP** | Separate creation responsibility from usage responsibility |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Direct creation - Tight coupling to concrete classes
class NotificationService {
    fun send(type: String, message: String) {
        val notification = when (type) {
            "email" -> EmailNotification()
            "slack" -> SlackNotification()
            "sms" -> SmsNotification()  // Modify each time you add
            else -> throw IllegalArgumentException()
        }
        notification.send(message)
    }
}
```

- Adding new types requires modifying existing code (OCP violation)
- Creation logic and usage logic are mixed
- Difficult to replace with specific implementations for testing

### Value Provided

- **Reduced Coupling**: Caller doesn't need to know concrete classes
- **Extensibility**: Just add a factory when adding new products
- **Testability**: Easily replace with Mock factory
- **Centralized Creation Logic**: Manage initialization, validation, logging in one place

---

## How It Works

### Structure

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│    ┌──────────────────┐         ┌──────────────────┐    │
│    │  <<interface>>   │         │  <<abstract>>    │    │
│    │     Product      │◄────────│     Creator      │    │
│    │  + operation()   │ creates │  + factoryMethod()│   │
│    └──────────────────┘         │  + someOperation()│   │
│             △                   └──────────────────┘    │
│             │                            △              │
│    ┌────────┴────────┐          ┌────────┴────────┐     │
│    │                 │          │                 │     │
│ ┌──┴───┐        ┌────┴───┐  ┌───┴────┐      ┌────┴───┐  │
│ │Concrete│      │Concrete│  │Concrete│      │Concrete│  │
│ │ProductA│      │ProductB│  │CreatorA│      │CreatorB│  │
│ └────────┘      └────────┘  └────────┘      └────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Product** | Common interface for objects to be created |
| **ConcreteProduct** | Actual object implementation |
| **Creator** | Abstract class declaring factory method |
| **ConcreteCreator** | Implements factory method to create specific product |

### Basic Implementation

```kotlin
// Product interface
interface Notification {
    fun send(message: String)
}

// ConcreteProduct implementations
class EmailNotification : Notification {
    override fun send(message: String) = println("Email: $message")
}

class SlackNotification : Notification {
    override fun send(message: String) = println("Slack: $message")
}

// Creator abstract class
abstract class NotificationFactory {
    // Factory method - Implemented by subclasses
    abstract fun create(): Notification

    // Template method - Logic using created object
    fun notify(message: String) {
        val notification = create()
        notification.send(message)
    }
}

// ConcreteCreator implementations
class EmailNotificationFactory : NotificationFactory() {
    override fun create(): Notification = EmailNotification()
}

class SlackNotificationFactory : NotificationFactory() {
    override fun create(): Notification = SlackNotification()
}

// Usage
val factory: NotificationFactory = EmailNotificationFactory()
factory.notify("Hello")  // Email: Hello

val factory2: NotificationFactory = SlackNotificationFactory()
factory2.notify("Hi from Slack")  // Slack: Hi from Slack
```

---

## Pitfalls

### 1. Over-engineering for Simple Cases

Direct creation is simpler if there are only 1-2 object types with low change probability.

```kotlin
// ❌ Over-engineering - Factory pattern for single type
abstract class UserFactory {
    abstract fun create(): User
}
class DefaultUserFactory : UserFactory() {
    override fun create() = User()
}

// ✅ Direct creation is sufficient
val user = User()
```

### 2. Factory Selection Logic Location

Logic to decide which factory to use is still needed somewhere.

```kotlin
// Simple Factory for factory selection (static method)
object NotificationFactoryProvider {
    fun getFactory(type: String): NotificationFactory = when (type) {
        "email" -> EmailNotificationFactory()
        "slack" -> SlackNotificationFactory()
        else -> throw IllegalArgumentException("Unknown type: $type")
    }
}
```

### 3. Don't Confuse with Abstract Factory

| Pattern | Purpose |
|---------|---------|
| **Factory Method** | Delegate single object creation to subclasses |
| **Abstract Factory** | Provide interface for creating **family** of related objects |

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Various object types to create | Various document types in document editor |
| Additional work needed during creation | Logging, validation, state initialization |
| Framework/library design | Provide structure users can extend |
| Test replacement needed | Mock object creation factory |

### Spring Environment: Bean Lookup Based Factory

```kotlin
// Product interface
interface PaymentProcessor {
    fun process(amount: Int)
}

// ConcreteProducts registered as Beans
@Service("card")
class CardPaymentProcessor : PaymentProcessor {
    override fun process(amount: Int) = println("Card: ${amount}won")
}

@Service("kakao")
class KakaoPaymentProcessor : PaymentProcessor {
    override fun process(amount: Int) = println("KakaoPay: ${amount}won")
}

// Factory - Inject all implementations via Map
@Service
class PaymentProcessorFactory(
    private val processors: Map<String, PaymentProcessor>
) {
    fun get(type: String): PaymentProcessor =
        processors[type] ?: error("Unsupported type: $type")
}

// Usage
@RestController
class PaymentController(
    private val factory: PaymentProcessorFactory
) {
    @PostMapping("/pay")
    fun pay(@RequestParam type: String, @RequestParam amount: Int) {
        factory.get(type).process(amount)
    }
}
```

**Advantage**: When adding new payment methods, just register Bean with `@Service` and it's automatically included in Factory

### Kotlin Simplified Version: Companion Object

```kotlin
interface Document {
    fun open()
    
    companion object {
        // Factory method defined in companion object
        fun create(type: String): Document = when (type) {
            "pdf" -> PdfDocument()
            "word" -> WordDocument()
            else -> throw IllegalArgumentException("Unknown type: $type")
        }
    }
}

class PdfDocument : Document {
    override fun open() = println("Opening PDF document")
}

class WordDocument : Document {
    override fun open() = println("Opening Word document")
}

// Usage
val doc = Document.create("pdf")
doc.open()  // Opening PDF document
```

### Before/After Comparison

```kotlin
// ❌ Before: Direct creation - Tight coupling
class ReportService {
    fun generate(type: String): Report {
        val exporter = when (type) {
            "pdf" -> PdfExporter()
            "excel" -> ExcelExporter()
            "csv" -> CsvExporter()  // Modify each time
            else -> throw IllegalArgumentException()
        }
        return exporter.export(data)
    }
}

// ✅ After: Factory Method - Loose coupling
interface ReportExporter {
    fun export(data: Data): Report
}

abstract class ReportExporterFactory {
    abstract fun create(): ReportExporter
}

class PdfExporterFactory : ReportExporterFactory() {
    override fun create() = PdfExporter()
}

// Just add Factory when adding new format
class CsvExporterFactory : ReportExporterFactory() {
    override fun create() = CsvExporter()
}

class ReportService(private val factory: ReportExporterFactory) {
    fun generate(): Report {
        val exporter = factory.create()
        return exporter.export(data)
    }
}
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Simple Factory** | Creates objects via static method, no inheritance structure (not GoF pattern) |
| **Abstract Factory** | Creates **family** of related objects, contains multiple factory methods |
| **Builder** | Creates complex objects **step by step**, focuses on creation process |
| **Prototype** | Creates new objects by **copying** existing ones |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Easy explanation with pizza shop example

### Recommended Sites
- [Refactoring Guru - Factory Method](https://refactoring.guru/design-patterns/factory-method)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Abstract Factory](./Abstract%20Factory.en.md)
- [Strategy Pattern](../Behavioral/Strategy%20Pattern.en.md)
