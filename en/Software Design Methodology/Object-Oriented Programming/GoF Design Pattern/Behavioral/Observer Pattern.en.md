# Observer Pattern

## Concept

The Observer Pattern **automatically notifies observers that are watching for state changes in an object**.

It defines a one-to-many (1:N) dependency relationship where when one object (Subject) changes state, all dependent objects (Observers) are automatically notified and updated. As one of the GoF behavioral patterns, the key is "**propagating state changes through loose coupling**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Loose Coupling** | Subject doesn't need to know concrete types of Observers |
| **Open-Closed** | No Subject modification needed when adding new Observers |
| **Unidirectional Dependency** | Observer depends on Subject, Subject only depends on interface |
| **Automatic Notification** | All registered Observers are automatically notified on state change |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Direct calls - Tight coupling
class StockPrice {
    private var price: Int = 0
    
    // Directly call all related objects on price change
    fun updatePrice(newPrice: Int) {
        price = newPrice
        emailService.sendAlert(price)      // Direct dependency on email service
        dashboardService.refresh(price)    // Direct dependency on dashboard service
        tradingBot.analyze(price)          // Direct dependency on trading bot
        // Need to modify here every time new feature is added
    }
}
```

- Adding new interested objects requires modifying existing code (OCP violation)
- Subject must know all dependent objects (tight coupling)
- All dependencies must be set up together for testing

### Value Provided

- **Reduced Coupling**: Subject and Observer can change independently
- **Extensibility**: New Observers can be added/removed anytime
- **Reusability**: Observers can be reused with other Subjects
- **Consistency**: State changes propagate consistently to all interested objects

---

## How It Works

### Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   ┌────────────────────┐              ┌────────────────────┐     │
│   │     Subject        │              │   <<interface>>    │     │
│   │  ─────────────────│              │     Observer       │     │
│   │  - observers: List │──────────────│  + update(state)   │     │
│   │  + attach(observer)│   notifies   └────────────────────┘     │
│   │  + detach(observer)│                        △                │
│   │  + notify()        │               ┌────────┴────────┐       │
│   └────────────────────┘               │                 │       │
│            │                     ┌─────┴─────┐     ┌─────┴─────┐ │
│            │                     │ Observer1 │     │ Observer2 │ │
│   ┌────────┴───────────┐         │           │     │           │ │
│   │  ConcreteSubject   │         └───────────┘     └───────────┘ │
│   │  ─────────────────│                                         │
│   │  - state           │                                         │
│   │  + getState()      │                                         │
│   │  + setState()      │                                         │
│   └────────────────────┘                                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Subject** | Manages Observer list, notifies Observers on state change |
| **ConcreteSubject** | Holds actual state and calls notify() on change |
| **Observer** | Defines update() method to receive state changes |
| **ConcreteObserver** | Receives notifications and performs actual actions |

### Basic Implementation

```kotlin
// Observer interface
interface Observer {
    fun update(price: Int)
}

// Subject interface
interface Subject {
    fun attach(observer: Observer)
    fun detach(observer: Observer)
    fun notify()
}

// ConcreteSubject - Stock price
class StockPrice : Subject {
    private val observers = mutableListOf<Observer>()
    private var price: Int = 0
    
    override fun attach(observer: Observer) {
        observers.add(observer)
    }
    
    override fun detach(observer: Observer) {
        observers.remove(observer)
    }
    
    override fun notify() {
        observers.forEach { it.update(price) }
    }
    
    fun setPrice(newPrice: Int) {
        price = newPrice
        notify()  // Notify all Observers on price change
    }
}

// ConcreteObservers
class EmailAlertObserver : Observer {
    override fun update(price: Int) {
        println("Email alert: Current price ${price}won")
    }
}

class DashboardObserver : Observer {
    override fun update(price: Int) {
        println("Dashboard refresh: ${price}won")
    }
}

class TradingBotObserver : Observer {
    override fun update(price: Int) {
        if (price < 10000) println("Buy signal triggered!")
    }
}

// Usage
val stock = StockPrice()

// Register Observers
stock.attach(EmailAlertObserver())
stock.attach(DashboardObserver())
stock.attach(TradingBotObserver())

// Price change - All Observers automatically notified
stock.setPrice(9500)
// Email alert: Current price 9500won
// Dashboard refresh: 9500won
// Buy signal triggered!
```

### Push vs Pull Model

| Model | Characteristics | Pros/Cons |
|-------|-----------------|-----------|
| **Push** | Subject sends changed data to Observer | Simple but Observer may receive unneeded data |
| **Pull** | Observer retrieves needed data from Subject | Flexible but Observer must know Subject |

```kotlin
// Push model
interface Observer {
    fun update(price: Int, volume: Int, timestamp: Long)
}

// Pull model
interface Observer {
    fun update(subject: StockPrice)  // Pass Subject reference
}

class DashboardObserver : Observer {
    override fun update(subject: StockPrice) {
        val price = subject.getPrice()  // Get only needed data
    }
}
```

---

## Pitfalls

### 1. Watch for Memory Leaks

Memory leaks occur if Observers are registered but never unregistered

```kotlin
// ❌ Missing Observer unregistration
class Activity {
    fun onCreate() {
        stockPrice.attach(this)  // Register
    }
    // Missing detach in onDestroy → Memory leak
}

// ✅ Unregister according to lifecycle
class Activity {
    fun onCreate() {
        stockPrice.attach(this)
    }
    
    fun onDestroy() {
        stockPrice.detach(this)  // Must unregister
    }
}
```

### 2. Watch for Circular Dependencies

Infinite loops can occur if Observer modifies Subject's state within update()

```kotlin
// ❌ Circular call risk
class BadObserver : Observer {
    override fun update(subject: StockPrice) {
        subject.setPrice(subject.getPrice() + 100)  // Calls notify again → Infinite loop
    }
}
```

### 3. Don't Depend on Notification Order

The order of update() calls to Observers is not guaranteed. Avoid logic that depends on order.

### 4. Consider Async Processing

Subject can be blocked if there are many Observers or update() processing takes long.

```kotlin
// ✅ Async notification
class AsyncSubject : Subject {
    private val scope = CoroutineScope(Dispatchers.Default)
    
    override fun notify() {
        observers.forEach { observer ->
            scope.launch {
                observer.update(state)
            }
        }
    }
}
```

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| State change broadcast | Stock prices, Sensor data |
| Event handling systems | GUI events, Notification systems |
| Publish-Subscribe model | Message queues, Event buses |
| Data binding | MVVM ViewModel-View relationship |

### Spring Environment: ApplicationEvent Usage

```kotlin
// Event definition (state)
data class OrderCreatedEvent(
    val orderId: String,
    val amount: Int
) : ApplicationEvent(orderId)

// Subject role - Event publishing
@Service
class OrderService(
    private val eventPublisher: ApplicationEventPublisher
) {
    fun createOrder(request: OrderRequest): Order {
        val order = orderRepository.save(Order(request))
        
        // Publish event - Notifies all registered Listeners
        eventPublisher.publishEvent(OrderCreatedEvent(order.id, order.amount))
        
        return order
    }
}

// Observer role - Event subscription
@Component
class EmailNotificationListener {
    @EventListener
    fun onOrderCreated(event: OrderCreatedEvent) {
        println("Sending order confirmation email: ${event.orderId}")
    }
}

@Component
class InventoryListener {
    @EventListener
    fun onOrderCreated(event: OrderCreatedEvent) {
        println("Processing inventory deduction: ${event.orderId}")
    }
}

@Component
class PointListener {
    @EventListener
    fun onOrderCreated(event: OrderCreatedEvent) {
        println("Accumulating points: ${event.amount * 0.01}")
    }
}
```

**Advantage**: When adding new Listeners, just add `@EventListener`, no OrderService modification needed

### Kotlin Flow Usage

```kotlin
// Use StateFlow as Subject
class StockPriceViewModel {
    private val _price = MutableStateFlow(0)
    val price: StateFlow<Int> = _price.asStateFlow()
    
    fun updatePrice(newPrice: Int) {
        _price.value = newPrice
    }
}

// Observer - Flow collect
class PriceDisplayView(
    private val viewModel: StockPriceViewModel
) {
    suspend fun observe() {
        viewModel.price.collect { price ->
            println("Current price: ${price}won")
        }
    }
}
```

### Before/After Comparison

```kotlin
// ❌ Before: Direct dependency - Tight coupling
class OrderService(
    private val emailService: EmailService,
    private val inventoryService: InventoryService,
    private val pointService: PointService,
    private val analyticsService: AnalyticsService  // Injection needed each time
) {
    fun createOrder(request: OrderRequest) {
        val order = save(request)
        emailService.sendConfirmation(order)
        inventoryService.decrease(order)
        pointService.accumulate(order)
        analyticsService.track(order)  // Code modification each time
    }
}

// ✅ After: Observer Pattern - Loose coupling
class OrderService(
    private val eventPublisher: ApplicationEventPublisher
) {
    fun createOrder(request: OrderRequest) {
        val order = save(request)
        eventPublisher.publishEvent(OrderCreatedEvent(order))
        // No code change needed when adding new Listeners
    }
}
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Mediator** | Mediator coordinates **bidirectional** communication between objects, Observer is **unidirectional** notification |
| **Pub/Sub** | Extension of Observer pattern, message broker (channel) exists in between |
| **Event Sourcing** | Stores state changes as events, Observer focuses on state change notification |

### Observer vs Pub/Sub

| Comparison | Observer | Pub/Sub |
|------------|----------|---------|
| Coupling | Subject manages Observer list | Publisher and Subscriber don't know each other |
| Intermediary | None | Message broker/channel exists |
| Sync/Async | Usually synchronous | Usually asynchronous |
| Use Case | State changes within objects | Distributed systems, Microservices |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Weather Station example

### Recommended Sites
- [Refactoring Guru - Observer Pattern](https://refactoring.guru/design-patterns/observer)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Strategy Pattern](./Strategy%20Pattern.en.md)
