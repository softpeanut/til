# Facade Pattern

## Concept

The Facade Pattern **provides a simplified interface to a complex subsystem**.

It hides the complexities of a system composed of multiple classes behind a single unified interface, allowing clients to use the system easily without knowing its internal complexity. As one of the GoF structural patterns, the key is "**hiding complexity and providing simplicity**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Simplification** | Wrap complex subsystem with simple interface |
| **Reduced Coupling** | Minimize dependencies between client and subsystem |
| **Layering** | Separate system into layers for easier management |
| **Encapsulation** | Hide internal implementation of subsystem from outside |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Client directly calls all subsystems - Complex and high coupling
class OrderController {
    fun placeOrder(request: OrderRequest) {
        // Check inventory
        val inventoryService = InventoryService()
        if (!inventoryService.checkStock(request.productId, request.quantity)) {
            throw InsufficientStockException()
        }
        
        // Process payment
        val paymentService = PaymentService()
        val paymentResult = paymentService.processPayment(request.paymentInfo)
        if (!paymentResult.success) {
            throw PaymentFailedException()
        }
        
        // Create order
        val orderService = OrderService()
        val order = orderService.createOrder(request)
        
        // Request shipping
        val shippingService = ShippingService()
        shippingService.scheduleDelivery(order)
        
        // Send notification
        val notificationService = NotificationService()
        notificationService.sendOrderConfirmation(order)
        
        // Accumulate points
        val pointService = PointService()
        pointService.accumulatePoints(request.userId, order.totalAmount)
    }
}
```

- Client must know details of all subsystems
- Client code must be modified when subsystem changes
- Same logic may be duplicated across multiple clients
- Call order and exception handling may differ between clients

### Value Provided

- **Simple Interface**: Provide complex operations as single method
- **Reduced Coupling**: Client only needs to know Facade
- **Maintainability**: Subsystem changes don't affect clients
- **Consistency**: Business logic managed in one place

---

## How It Works

### Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────┐                                                       │
│   │ Client  │                                                       │
│   └────┬────┘                                                       │
│        │                                                            │
│        ▼                                                            │
│   ┌─────────────────────────────────────────────────────────┐       │
│   │                      Facade                              │       │
│   │  ─────────────────────────────────────────────────────  │       │
│   │  + simpleOperation()                                    │       │
│   └──────────────────────────┬──────────────────────────────┘       │
│                              │                                      │
│              ┌───────────────┼───────────────┐                      │
│              │               │               │                      │
│              ▼               ▼               ▼                      │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│   │ Subsystem A  │  │ Subsystem B  │  │ Subsystem C  │              │
│   │  + methodA() │  │  + methodB() │  │  + methodC() │              │
│   └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                     │
│                        Subsystem                                    │
└─────────────────────────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Facade** | Combines subsystems to provide simple interface |
| **Subsystem** | Classes implementing actual functionality, called indirectly through Facade |
| **Client** | Uses subsystem through Facade |

### Basic Implementation

```kotlin
// Subsystem classes
class InventoryService {
    fun checkStock(productId: String, quantity: Int): Boolean {
        println("Checking stock: $productId, quantity: $quantity")
        return true
    }
    
    fun reserveStock(productId: String, quantity: Int) {
        println("Reserving stock: $productId")
    }
}

class PaymentService {
    fun processPayment(amount: Int, paymentMethod: String): PaymentResult {
        println("Processing payment: ${amount}won, method: $paymentMethod")
        return PaymentResult(success = true, transactionId = "TXN-123")
    }
}

class ShippingService {
    fun calculateShippingCost(address: String): Int {
        println("Calculating shipping cost: $address")
        return 3000
    }
    
    fun scheduleDelivery(orderId: String, address: String) {
        println("Scheduling delivery: Order $orderId → $address")
    }
}

class NotificationService {
    fun sendEmail(email: String, message: String) {
        println("Sending email: $email")
    }
    
    fun sendSms(phone: String, message: String) {
        println("Sending SMS: $phone")
    }
}

// Facade - Simplifies complex subsystem
class OrderFacade(
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService,
    private val shippingService: ShippingService,
    private val notificationService: NotificationService
) {
    fun placeOrder(order: OrderRequest): OrderResult {
        // 1. Check and reserve stock
        if (!inventoryService.checkStock(order.productId, order.quantity)) {
            return OrderResult(success = false, message = "Insufficient stock")
        }
        inventoryService.reserveStock(order.productId, order.quantity)
        
        // 2. Calculate shipping cost
        val shippingCost = shippingService.calculateShippingCost(order.address)
        val totalAmount = order.price * order.quantity + shippingCost
        
        // 3. Process payment
        val paymentResult = paymentService.processPayment(totalAmount, order.paymentMethod)
        if (!paymentResult.success) {
            return OrderResult(success = false, message = "Payment failed")
        }
        
        // 4. Schedule delivery
        val orderId = "ORD-${System.currentTimeMillis()}"
        shippingService.scheduleDelivery(orderId, order.address)
        
        // 5. Send notifications
        notificationService.sendEmail(order.email, "Your order is complete: $orderId")
        notificationService.sendSms(order.phone, "Order complete: $orderId")
        
        return OrderResult(success = true, orderId = orderId, message = "Order complete")
    }
}

// Usage - Client only needs to know Facade
val facade = OrderFacade(
    InventoryService(),
    PaymentService(),
    ShippingService(),
    NotificationService()
)

val result = facade.placeOrder(OrderRequest(
    productId = "PROD-001",
    quantity = 2,
    price = 10000,
    address = "123 Main St",
    paymentMethod = "CARD",
    email = "user@example.com",
    phone = "010-1234-5678"
))

println(result)  // OrderResult(success=true, orderId=ORD-xxx, message=Order complete)
```

---

## Pitfalls

### 1. Watch for God Object

Facade becomes hard to maintain if it has too many responsibilities.

```kotlin
// ❌ All functionality concentrated in one Facade
class ApplicationFacade {
    fun placeOrder() { /* ... */ }
    fun cancelOrder() { /* ... */ }
    fun processRefund() { /* ... */ }
    fun manageInventory() { /* ... */ }
    fun generateReport() { /* ... */ }
    fun sendNotification() { /* ... */ }
    // Keeps growing...
}

// ✅ Separate Facades by domain
class OrderFacade { /* Order related */ }
class InventoryFacade { /* Inventory related */ }
class ReportFacade { /* Report related */ }
```

### 2. Facade is Optional

Facade is an **additional** interface to the subsystem. Clients should be able to use subsystems directly when needed.

```kotlin
// Normal cases use Facade
orderFacade.placeOrder(request)

// Special cases can use subsystem directly
paymentService.processRefund(transactionId)
```

### 3. Facade Needs Modification When Subsystem Changes

Internal subsystem changes can be hidden, but interface changes require Facade modification.

---

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Wrapping complex libraries | Video encoding, PDF generation |
| Legacy system integration | Provide modern interface to old systems |
| Microservice aggregation | Combine multiple service calls into one |
| Configuration simplification | Simplify complex initialization process |

### Video Conversion Example

```kotlin
// Complex video processing subsystem
class VideoFile(val filename: String)

class Codec {
    fun decode(file: VideoFile): ByteArray {
        println("Decoding: ${file.filename}")
        return ByteArray(0)
    }
}

class AudioMixer {
    fun extractAudio(data: ByteArray): ByteArray {
        println("Extracting audio")
        return ByteArray(0)
    }
    
    fun mixAudio(audio: ByteArray, background: ByteArray): ByteArray {
        println("Mixing audio")
        return ByteArray(0)
    }
}

class VideoEncoder {
    fun encode(video: ByteArray, audio: ByteArray, format: String): ByteArray {
        println("Encoding: $format format")
        return ByteArray(0)
    }
}

class BitrateReader {
    fun read(data: ByteArray): Int {
        println("Analyzing bitrate")
        return 1500
    }
}

// Facade - Simplifies complex video conversion
class VideoConverterFacade {
    private val codec = Codec()
    private val audioMixer = AudioMixer()
    private val encoder = VideoEncoder()
    private val bitrateReader = BitrateReader()
    
    fun convert(filename: String, format: String): ByteArray {
        println("=== Starting video conversion: $filename → $format ===")
        
        val file = VideoFile(filename)
        val decodedData = codec.decode(file)
        
        val bitrate = bitrateReader.read(decodedData)
        println("Original bitrate: $bitrate kbps")
        
        val audio = audioMixer.extractAudio(decodedData)
        val result = encoder.encode(decodedData, audio, format)
        
        println("=== Conversion complete ===")
        return result
    }
}

// Usage - Complex process in one line
val converter = VideoConverterFacade()
converter.convert("video.avi", "mp4")
```

### Spring Environment: Service Layer Facade

```kotlin
// Subsystem services
@Service
class UserService(private val userRepository: UserRepository) {
    fun findById(id: Long): User = userRepository.findById(id).orElseThrow()
    fun updateLastLogin(id: Long) { /* ... */ }
}

@Service
class CartService(private val cartRepository: CartRepository) {
    fun getCart(userId: Long): Cart = cartRepository.findByUserId(userId)
    fun clearCart(userId: Long) { /* ... */ }
}

@Service  
class CouponService(private val couponRepository: CouponRepository) {
    fun getAvailableCoupons(userId: Long): List<Coupon> = 
        couponRepository.findAvailableByUserId(userId)
    fun applyCoupon(couponId: Long, orderId: Long) { /* ... */ }
}

@Service
class RecommendationService {
    fun getRecommendations(userId: Long): List<Product> { /* ... */ }
}

// Facade - Retrieve all information needed for My Page at once
@Service
class MyPageFacade(
    private val userService: UserService,
    private val cartService: CartService,
    private val couponService: CouponService,
    private val recommendationService: RecommendationService
) {
    fun getMyPageData(userId: Long): MyPageResponse {
        val user = userService.findById(userId)
        val cart = cartService.getCart(userId)
        val coupons = couponService.getAvailableCoupons(userId)
        val recommendations = recommendationService.getRecommendations(userId)
        
        userService.updateLastLogin(userId)
        
        return MyPageResponse(
            user = user,
            cartItemCount = cart.items.size,
            couponCount = coupons.size,
            recommendations = recommendations.take(5)
        )
    }
}

// Controller - Depends on only one Facade
@RestController
@RequestMapping("/api/mypage")
class MyPageController(
    private val myPageFacade: MyPageFacade
) {
    @GetMapping
    fun getMyPage(@AuthenticationPrincipal userId: Long): MyPageResponse {
        return myPageFacade.getMyPageData(userId)
    }
}
```

### Before/After Comparison

```kotlin
// ❌ Before: Controller directly calls all services
@RestController
class CheckoutController(
    private val cartService: CartService,
    private val inventoryService: InventoryService,
    private val pricingService: PricingService,
    private val couponService: CouponService,
    private val paymentService: PaymentService,
    private val orderService: OrderService,
    private val shippingService: ShippingService,
    private val notificationService: NotificationService,
    private val pointService: PointService
) {
    @PostMapping("/checkout")
    fun checkout(@RequestBody request: CheckoutRequest): CheckoutResponse {
        val cart = cartService.getCart(request.userId)
        val inventory = inventoryService.checkAll(cart.items)
        val price = pricingService.calculate(cart)
        // ... 10+ service calls
    }
}

// ✅ After: Simplified through Facade
@RestController
class CheckoutController(
    private val checkoutFacade: CheckoutFacade
) {
    @PostMapping("/checkout")
    fun checkout(@RequestBody request: CheckoutRequest): CheckoutResponse {
        return checkoutFacade.processCheckout(request)
    }
}

@Service
class CheckoutFacade(
    private val cartService: CartService,
    private val inventoryService: InventoryService,
    // ... other services
) {
    @Transactional
    fun processCheckout(request: CheckoutRequest): CheckoutResponse {
        // All complex logic coordinated here
        // Transaction, error handling, rollback all managed in one place
    }
}
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Adapter** | **Converts** incompatible interfaces, Facade **simplifies** complexity |
| **Mediator** | Coordinates **bidirectional communication** between objects, Facade calls subsystem **unidirectionally** |
| **Proxy** | **Controls access** with same interface, Facade provides new **simplified interface** |

### Facade vs Adapter

| Comparison | Facade | Adapter |
|------------|--------|---------|
| Purpose | Hide complexity | Convert interface |
| Target | Subsystem composed of multiple classes | Single class/interface |
| Interface | Define new simple interface | Conform to existing interface |
| When to Use | Wrapping complex systems | Solving compatibility issues |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Home Theater example

### Recommended Sites
- [Refactoring Guru - Facade Pattern](https://refactoring.guru/design-patterns/facade)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Adapter Pattern](./Adapter%20Pattern.en.md)
- [Composite Pattern](./Composite%20Pattern.en.md)
