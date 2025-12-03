# Facade Pattern

## 개념

퍼사드 패턴은 **복잡한 서브시스템에 대한 단순화된 인터페이스를 제공하는 패턴**이다.

여러 클래스로 구성된 복잡한 시스템을 하나의 통합된 인터페이스 뒤에 감추어, 클라이언트가 서브시스템의 복잡성을 알 필요 없이 쉽게 사용할 수 있게 한다. GoF 구조 패턴 중 하나로, "**복잡성을 숨기고 단순함을 제공**"하는 것이 핵심이다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **단순화** | 복잡한 서브시스템을 단순한 인터페이스로 래핑 |
| **결합도 감소** | 클라이언트와 서브시스템 간의 의존성 최소화 |
| **계층화** | 시스템을 계층으로 분리하여 관리 용이성 향상 |
| **캡슐화** | 서브시스템의 내부 구현을 외부로부터 숨김 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 클라이언트가 모든 서브시스템을 직접 호출 - 복잡하고 결합도 높음
class OrderController {
    fun placeOrder(request: OrderRequest) {
        // 재고 확인
        val inventoryService = InventoryService()
        if (!inventoryService.checkStock(request.productId, request.quantity)) {
            throw InsufficientStockException()
        }
        
        // 결제 처리
        val paymentService = PaymentService()
        val paymentResult = paymentService.processPayment(request.paymentInfo)
        if (!paymentResult.success) {
            throw PaymentFailedException()
        }
        
        // 주문 생성
        val orderService = OrderService()
        val order = orderService.createOrder(request)
        
        // 배송 요청
        val shippingService = ShippingService()
        shippingService.scheduleDelivery(order)
        
        // 알림 발송
        val notificationService = NotificationService()
        notificationService.sendOrderConfirmation(order)
        
        // 포인트 적립
        val pointService = PointService()
        pointService.accumulatePoints(request.userId, order.totalAmount)
    }
}
```

- 클라이언트가 모든 서브시스템의 세부 사항을 알아야 함
- 서브시스템 변경 시 클라이언트 코드도 수정 필요
- 동일한 로직이 여러 클라이언트에 중복될 수 있음
- 호출 순서나 예외 처리가 클라이언트마다 다를 수 있음

### 제공하는 가치

- **단순한 인터페이스**: 복잡한 작업을 하나의 메서드로 제공
- **결합도 감소**: 클라이언트는 Facade만 알면 됨
- **유지보수성**: 서브시스템 변경이 클라이언트에 영향 없음
- **일관성**: 비즈니스 로직이 한 곳에서 관리됨

---

## 동작 원리

### 구조

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

| 구성요소 | 역할 |
|---------|------|
| **Facade** | 서브시스템들을 조합하여 단순한 인터페이스 제공 |
| **Subsystem** | 실제 기능을 구현한 클래스들, Facade를 통해 간접 호출됨 |
| **Client** | Facade를 통해 서브시스템 사용 |

### 기본 구현

```kotlin
// Subsystem 클래스들
class InventoryService {
    fun checkStock(productId: String, quantity: Int): Boolean {
        println("재고 확인: $productId, 수량: $quantity")
        return true
    }
    
    fun reserveStock(productId: String, quantity: Int) {
        println("재고 예약: $productId")
    }
}

class PaymentService {
    fun processPayment(amount: Int, paymentMethod: String): PaymentResult {
        println("결제 처리: ${amount}원, 방식: $paymentMethod")
        return PaymentResult(success = true, transactionId = "TXN-123")
    }
}

class ShippingService {
    fun calculateShippingCost(address: String): Int {
        println("배송비 계산: $address")
        return 3000
    }
    
    fun scheduleDelivery(orderId: String, address: String) {
        println("배송 예약: 주문 $orderId → $address")
    }
}

class NotificationService {
    fun sendEmail(email: String, message: String) {
        println("이메일 발송: $email")
    }
    
    fun sendSms(phone: String, message: String) {
        println("SMS 발송: $phone")
    }
}

// Facade - 복잡한 서브시스템을 단순화
class OrderFacade(
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService,
    private val shippingService: ShippingService,
    private val notificationService: NotificationService
) {
    fun placeOrder(order: OrderRequest): OrderResult {
        // 1. 재고 확인 및 예약
        if (!inventoryService.checkStock(order.productId, order.quantity)) {
            return OrderResult(success = false, message = "재고 부족")
        }
        inventoryService.reserveStock(order.productId, order.quantity)
        
        // 2. 배송비 계산
        val shippingCost = shippingService.calculateShippingCost(order.address)
        val totalAmount = order.price * order.quantity + shippingCost
        
        // 3. 결제 처리
        val paymentResult = paymentService.processPayment(totalAmount, order.paymentMethod)
        if (!paymentResult.success) {
            return OrderResult(success = false, message = "결제 실패")
        }
        
        // 4. 배송 예약
        val orderId = "ORD-${System.currentTimeMillis()}"
        shippingService.scheduleDelivery(orderId, order.address)
        
        // 5. 알림 발송
        notificationService.sendEmail(order.email, "주문이 완료되었습니다: $orderId")
        notificationService.sendSms(order.phone, "주문 완료: $orderId")
        
        return OrderResult(success = true, orderId = orderId, message = "주문 완료")
    }
}

// 사용 - 클라이언트는 Facade만 알면 됨
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
    address = "서울시 강남구",
    paymentMethod = "CARD",
    email = "user@example.com",
    phone = "010-1234-5678"
))

println(result)  // OrderResult(success=true, orderId=ORD-xxx, message=주문 완료)
```

---

## 주의사항

### 1. God Object 주의

Facade가 너무 많은 책임을 가지면 유지보수가 어려워진다.

```kotlin
// ❌ 모든 기능을 하나의 Facade에 집중
class ApplicationFacade {
    fun placeOrder() { /* ... */ }
    fun cancelOrder() { /* ... */ }
    fun processRefund() { /* ... */ }
    fun manageInventory() { /* ... */ }
    fun generateReport() { /* ... */ }
    fun sendNotification() { /* ... */ }
    // 계속 증가...
}

// ✅ 도메인별로 Facade 분리
class OrderFacade { /* 주문 관련 */ }
class InventoryFacade { /* 재고 관련 */ }
class ReportFacade { /* 리포트 관련 */ }
```

### 2. Facade는 선택 사항

Facade는 서브시스템에 대한 **추가적인** 인터페이스다. 필요하면 클라이언트가 서브시스템을 직접 사용할 수 있어야 한다.

```kotlin
// 일반적인 경우 Facade 사용
orderFacade.placeOrder(request)

// 특수한 경우 서브시스템 직접 사용 가능
paymentService.processRefund(transactionId)
```

### 3. 서브시스템 변경 시 Facade도 수정 필요

서브시스템 내부 변경은 숨길 수 있지만, 인터페이스 변경 시 Facade 수정이 필요하다.

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 복잡한 라이브러리 래핑 | 영상 인코딩, PDF 생성 |
| 레거시 시스템 통합 | 구형 시스템에 현대적 인터페이스 제공 |
| 마이크로서비스 집계 | 여러 서비스 호출을 하나로 통합 |
| 설정 간소화 | 복잡한 초기화 과정을 단순화 |

### 영상 변환 예제

```kotlin
// 복잡한 영상 처리 서브시스템
class VideoFile(val filename: String)

class Codec {
    fun decode(file: VideoFile): ByteArray {
        println("디코딩: ${file.filename}")
        return ByteArray(0)
    }
}

class AudioMixer {
    fun extractAudio(data: ByteArray): ByteArray {
        println("오디오 추출")
        return ByteArray(0)
    }
    
    fun mixAudio(audio: ByteArray, background: ByteArray): ByteArray {
        println("오디오 믹싱")
        return ByteArray(0)
    }
}

class VideoEncoder {
    fun encode(video: ByteArray, audio: ByteArray, format: String): ByteArray {
        println("인코딩: $format 형식")
        return ByteArray(0)
    }
}

class BitrateReader {
    fun read(data: ByteArray): Int {
        println("비트레이트 분석")
        return 1500
    }
}

// Facade - 복잡한 영상 변환을 단순화
class VideoConverterFacade {
    private val codec = Codec()
    private val audioMixer = AudioMixer()
    private val encoder = VideoEncoder()
    private val bitrateReader = BitrateReader()
    
    fun convert(filename: String, format: String): ByteArray {
        println("=== 영상 변환 시작: $filename → $format ===")
        
        val file = VideoFile(filename)
        val decodedData = codec.decode(file)
        
        val bitrate = bitrateReader.read(decodedData)
        println("원본 비트레이트: $bitrate kbps")
        
        val audio = audioMixer.extractAudio(decodedData)
        val result = encoder.encode(decodedData, audio, format)
        
        println("=== 변환 완료 ===")
        return result
    }
}

// 사용 - 복잡한 과정을 한 줄로
val converter = VideoConverterFacade()
converter.convert("video.avi", "mp4")
```

### Spring 환경: 서비스 계층 Facade

```kotlin
// 서브시스템 서비스들
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

// Facade - 마이페이지에 필요한 모든 정보를 한 번에 조회
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

// Controller - Facade 하나만 의존
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

### Before/After 비교

```kotlin
// ❌ Before: 컨트롤러가 모든 서비스를 직접 호출
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
        // ... 10개 이상의 서비스 호출
    }
}

// ✅ After: Facade를 통한 단순화
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
    // ... 다른 서비스들
) {
    @Transactional
    fun processCheckout(request: CheckoutRequest): CheckoutResponse {
        // 모든 복잡한 로직을 여기서 조율
        // 트랜잭션, 에러 처리, 롤백 등도 한 곳에서 관리
    }
}
```

---

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Adapter** | 호환되지 않는 인터페이스를 **변환**, Facade는 복잡성을 **단순화** |
| **Mediator** | 객체들 간의 **양방향 통신**을 조율, Facade는 **단방향**으로 서브시스템 호출 |
| **Proxy** | 동일 인터페이스로 **접근 제어**, Facade는 새로운 **단순화된 인터페이스** 제공 |

### Facade vs Adapter

| 비교 항목 | Facade | Adapter |
|----------|--------|---------|
| 목적 | 복잡성 숨기기 | 인터페이스 변환 |
| 대상 | 여러 클래스로 구성된 서브시스템 | 단일 클래스/인터페이스 |
| 인터페이스 | 새로운 단순한 인터페이스 정의 | 기존 인터페이스에 맞춤 |
| 사용 시점 | 복잡한 시스템 래핑 | 호환성 문제 해결 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 홈시어터 예제

### 추천 사이트
- [Refactoring Guru - Facade Pattern](https://refactoring.guru/design-patterns/facade)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)
- [Adapter Pattern](./Adapter%20Pattern.md)
- [Composite Pattern](./Composite%20Pattern.md)
