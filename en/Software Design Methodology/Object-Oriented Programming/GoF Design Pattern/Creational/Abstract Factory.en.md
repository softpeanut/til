# Abstract Factory Pattern

## Concept

The Abstract Factory Pattern **provides an interface for creating families of related objects**.

Once a product family is selected, it must be consistently applied, ensuring that objects from different product families are not mixed. As one of the GoF creational patterns, the key is "**creating related objects without specifying concrete classes**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Product Family Consistency** | Ensures created objects always belong to the same product family |
| **Encapsulation** | Hide concrete class creation logic inside factory |
| **DIP** | Client depends only on abstract interfaces |
| **OCP** | Extend without modifying existing code when adding new product families |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Direct creation - Risk of mixing product families
class UIBuilder {
    fun build(theme: String) {
        val button = if (theme == "dark") DarkButton() else LightButton()
        val checkbox = if (theme == "dark") DarkCheckbox() else LightCheckbox()
        val dialog = if (theme == "dark") LightDialog() else DarkDialog()  // Wrong theme by mistake!
        
        // Product family mismatch occurs
    }
}
```

- UI consistency breaks if product families mix
- Adding new product family (e.g., Blue theme) requires modifying all conditionals
- Mistake probability increases as product types grow

### Value Provided

- **Consistency Guaranteed**: Objects created from same factory are always compatible
- **Easy Product Family Switching**: Switching factory changes entire product family
- **Extensibility**: Easy to add new product families
- **Reduced Coupling**: Client doesn't need to know concrete classes

---

## How It Works

### Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌────────────────────┐              ┌────────────────────┐     │
│   │  <<interface>>     │              │  <<interface>>     │     │
│   │  AbstractFactory   │              │  AbstractProductA  │     │
│   │  + createProductA()│──────────────│  + operationA()    │     │
│   │  + createProductB()│              └────────────────────┘     │
│   └────────────────────┘                        △                │
│            △                           ┌────────┴────────┐       │
│   ┌────────┴────────┐                  │                 │       │
│   │                 │            ┌─────┴─────┐     ┌─────┴─────┐ │
│ ┌─┴───────────┐ ┌───┴─────────┐  │ProductA1  │     │ProductA2  │ │
│ │ConcreteFactory1│ConcreteFactory2│ │(Family1) │     │(Family2) │ │
│ └─────────────┘ └─────────────┘  └───────────┘     └───────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **AbstractFactory** | Interface declaring product family creation methods |
| **ConcreteFactory** | Implementation that creates objects of a specific product family |
| **AbstractProduct** | Common interface for products to be created |
| **ConcreteProduct** | Actual product implementation belonging to specific product family |

### Basic Implementation

```kotlin
// AbstractProduct - Product interfaces
interface Button {
    fun render()
}

interface Checkbox {
    fun render()
}

// ConcreteProduct - Dark theme product family
class DarkButton : Button {
    override fun render() = println("Dark Button")
}

class DarkCheckbox : Checkbox {
    override fun render() = println("Dark Checkbox")
}

// ConcreteProduct - Light theme product family
class LightButton : Button {
    override fun render() = println("Light Button")
}

class LightCheckbox : Checkbox {
    override fun render() = println("Light Checkbox")
}

// AbstractFactory - Factory interface
interface UIComponentFactory {
    fun createButton(): Button
    fun createCheckbox(): Checkbox
}

// ConcreteFactory - Dark theme factory
class DarkThemeFactory : UIComponentFactory {
    override fun createButton(): Button = DarkButton()
    override fun createCheckbox(): Checkbox = DarkCheckbox()
}

// ConcreteFactory - Light theme factory
class LightThemeFactory : UIComponentFactory {
    override fun createButton(): Button = LightButton()
    override fun createCheckbox(): Checkbox = LightCheckbox()
}

// Client - Depends only on abstract interfaces
fun buildUI(factory: UIComponentFactory) {
    val button = factory.createButton()
    val checkbox = factory.createCheckbox()
    
    button.render()
    checkbox.render()
}

// Usage - Changing factory changes entire product family
buildUI(DarkThemeFactory())   // Dark Button, Dark Checkbox
buildUI(LightThemeFactory())  // Light Button, Light Checkbox
```

---

## Pitfalls

### 1. Over-engineering for Single Object Creation

Use only when creating **families** of related objects. Factory Method is more appropriate for single object creation.

```kotlin
// ❌ Over-engineering - Abstract Factory for single product
interface SingleProductFactory {
    fun createProduct(): Product
}

// ✅ Factory Method is sufficient
abstract class ProductFactory {
    abstract fun create(): Product
}
```

### 2. Interface Changes When Adding Products

Adding new product type (e.g., Dialog) requires modifying all factory interfaces and implementations.

```kotlin
// When adding new product
interface UIComponentFactory {
    fun createButton(): Button
    fun createCheckbox(): Checkbox
    fun createDialog(): Dialog  // Must be implemented in all ConcreteFactories
}
```

### 3. Relationship with Factory Method

Each method that creates products inside Abstract Factory is essentially a Factory Method.

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Consistency needed for related objects | UI themes, OS-specific widgets |
| Product family switching needed | Dev/Prod environment configurations |
| Platform-independent design | Cross-platform apps |
| Framework extension points | Plugin systems |

### Spring Environment: Cloud Environment Storage Strategy

```kotlin
// AbstractProduct - Storage interfaces
interface Storage {
    fun save(data: String)
}

interface Cache {
    fun get(key: String): String?
}

// AbstractFactory - Storage factory interface
interface StorageFactory {
    fun createStorage(): Storage
    fun createCache(): Cache
}

// ConcreteFactory - AWS product family
@Component
@Profile("aws")
class AWSStorageFactory : StorageFactory {
    override fun createStorage(): Storage = S3Storage()
    override fun createCache(): Cache = ElastiCache()
}

// ConcreteFactory - GCP product family
@Component
@Profile("gcp")
class GCPStorageFactory : StorageFactory {
    override fun createStorage(): Storage = GcsStorage()
    override fun createCache(): Cache = Memorystore()
}

// Client - Depends only on abstract interface
@Service
class DataService(
    private val storageFactory: StorageFactory
) {
    fun process(data: String) {
        val storage = storageFactory.createStorage()
        val cache = storageFactory.createCache()
        
        storage.save(data)
        // Always guaranteed to use same cloud product family
    }
}
```

**Advantage**: Just changing Profile switches entire cloud infrastructure product family

### Before/After Comparison

```kotlin
// ❌ Before: Conditional-based - Risk of product family mixing
class DatabaseService(private val env: String) {
    fun getConnection(): Connection {
        return if (env == "prod") ProdConnection() else DevConnection()
    }
    
    fun getRepository(): Repository {
        return if (env == "prod") ProdRepository() else DevRepository()
    }
    
    fun getCache(): Cache {
        // Accidentally using dev cache in prod environment possible
        return if (env == "dev") ProdCache() else DevCache()  // Bug!
    }
}

// ✅ After: Abstract Factory - Product family consistency guaranteed
interface DatabaseFactory {
    fun createConnection(): Connection
    fun createRepository(): Repository
    fun createCache(): Cache
}

class ProdDatabaseFactory : DatabaseFactory {
    override fun createConnection() = ProdConnection()
    override fun createRepository() = ProdRepository()
    override fun createCache() = ProdCache()  // Always Prod product family
}

class DevDatabaseFactory : DatabaseFactory {
    override fun createConnection() = DevConnection()
    override fun createRepository() = DevRepository()
    override fun createCache() = DevCache()  // Always Dev product family
}

class DatabaseService(private val factory: DatabaseFactory) {
    fun setup() {
        val conn = factory.createConnection()
        val repo = factory.createRepository()
        val cache = factory.createCache()
        // Product family mixing impossible
    }
}
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Factory Method** | Delegates **single object** creation to subclasses |
| **Abstract Factory** | Creates **related object families**, contains multiple Factory Methods |
| **Builder** | Creates complex objects **step by step**, focuses on creation process |
| **Prototype** | Creates new objects by **copying** existing ones |

### Factory Method vs Abstract Factory

| Comparison | Factory Method | Abstract Factory |
|------------|----------------|------------------|
| Creation Target | Single product | Product family (multiple related products) |
| Extension Method | Add subclasses | Add factory implementations |
| Complexity | Relatively simple | Complex structure |
| When to Use | Product type extension | Product family switching needed |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Pizza ingredient factory example

### Recommended Sites
- [Refactoring Guru - Abstract Factory](https://refactoring.guru/design-patterns/abstract-factory)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Factory Method](./Factory%20Method.en.md)
