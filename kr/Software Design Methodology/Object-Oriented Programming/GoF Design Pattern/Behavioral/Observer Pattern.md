# Observer Pattern

## 개념

옵저버 패턴은 **객체의 상태 변화를 관찰하는 옵저버들에게 자동으로 통지하는 패턴**이다.

일대다(1:N) 의존 관계를 정의하여, 하나의 객체(Subject) 상태가 변경되면 그에 의존하는 모든 객체(Observer)들이 자동으로 알림을 받고 갱신된다. GoF 행위 패턴 중 하나로, "**느슨한 결합으로 상태 변화를 전파**"하는 것이 핵심이다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **느슨한 결합** | Subject는 Observer의 구체적인 타입을 알 필요 없음 |
| **개방-폐쇄** | 새로운 Observer 추가 시 Subject 수정 불필요 |
| **단방향 의존** | Observer가 Subject에 의존, Subject는 인터페이스에만 의존 |
| **자동 통지** | 상태 변경 시 등록된 모든 Observer에게 자동 알림 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 직접 호출 - 강한 결합
class StockPrice {
    private var price: Int = 0
    
    // 가격 변경 시 모든 관련 객체를 직접 호출
    fun updatePrice(newPrice: Int) {
        price = newPrice
        emailService.sendAlert(price)      // 이메일 서비스에 직접 의존
        dashboardService.refresh(price)    // 대시보드 서비스에 직접 의존
        tradingBot.analyze(price)          // 트레이딩 봇에 직접 의존
        // 새로운 기능 추가마다 여기를 수정해야 함
    }
}
```

- 새로운 관심 객체 추가 시 기존 코드 수정 필요 (OCP 위반)
- Subject가 모든 의존 객체를 알아야 함 (강한 결합)
- 테스트 시 모든 의존성을 함께 설정해야 함

### 제공하는 가치

- **결합도 감소**: Subject와 Observer가 독립적으로 변경 가능
- **확장성**: 새로운 Observer를 언제든 추가/제거 가능
- **재사용성**: Observer는 다른 Subject에도 재사용 가능
- **일관성**: 상태 변화가 모든 관심 객체에 일관되게 전파

---

## 동작 원리

### 구조

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

| 구성요소 | 역할 |
|---------|------|
| **Subject** | Observer 목록 관리, 상태 변경 시 Observer들에게 통지 |
| **ConcreteSubject** | 실제 상태를 보유하고 변경 시 notify() 호출 |
| **Observer** | 상태 변화를 받을 update() 메서드 정의 |
| **ConcreteObserver** | 통지를 받아 실제 동작 수행 |

### 기본 구현

```kotlin
// Observer 인터페이스
interface Observer {
    fun update(price: Int)
}

// Subject 인터페이스
interface Subject {
    fun attach(observer: Observer)
    fun detach(observer: Observer)
    fun notify()
}

// ConcreteSubject - 주식 가격
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
        notify()  // 가격 변경 시 모든 Observer에게 통지
    }
}

// ConcreteObserver들
class EmailAlertObserver : Observer {
    override fun update(price: Int) {
        println("Email 알림: 현재 가격 ${price}원")
    }
}

class DashboardObserver : Observer {
    override fun update(price: Int) {
        println("대시보드 갱신: ${price}원")
    }
}

class TradingBotObserver : Observer {
    override fun update(price: Int) {
        if (price < 10000) println("매수 신호 발생!")
    }
}

// 사용
val stock = StockPrice()

// Observer 등록
stock.attach(EmailAlertObserver())
stock.attach(DashboardObserver())
stock.attach(TradingBotObserver())

// 가격 변경 - 모든 Observer에게 자동 통지
stock.setPrice(9500)
// Email 알림: 현재 가격 9500원
// 대시보드 갱신: 9500원
// 매수 신호 발생!
```

### Push vs Pull 모델

| 모델 | 특징 | 장단점 |
|------|------|--------|
| **Push** | Subject가 변경된 데이터를 Observer에게 전달 | 간단하지만 Observer가 필요 없는 데이터도 받을 수 있음 |
| **Pull** | Observer가 필요한 데이터를 Subject에서 가져감 | 유연하지만 Observer가 Subject를 알아야 함 |

```kotlin
// Push 모델
interface Observer {
    fun update(price: Int, volume: Int, timestamp: Long)
}

// Pull 모델
interface Observer {
    fun update(subject: StockPrice)  // Subject 참조 전달
}

class DashboardObserver : Observer {
    override fun update(subject: StockPrice) {
        val price = subject.getPrice()  // 필요한 데이터만 가져옴
    }
}
```

---

## 주의사항

### 1. 메모리 누수 주의

Observer를 등록만 하고 해제하지 않으면 메모리 누수 발생

```kotlin
// ❌ Observer 해제 누락
class Activity {
    fun onCreate() {
        stockPrice.attach(this)  // 등록
    }
    // onDestroy에서 detach 누락 → 메모리 누수
}

// ✅ 생명주기에 맞춰 해제
class Activity {
    fun onCreate() {
        stockPrice.attach(this)
    }
    
    fun onDestroy() {
        stockPrice.detach(this)  // 반드시 해제
    }
}
```

### 2. 순환 의존 주의

Observer가 update() 내에서 Subject의 상태를 변경하면 무한 루프 발생 가능

```kotlin
// ❌ 순환 호출 위험
class BadObserver : Observer {
    override fun update(subject: StockPrice) {
        subject.setPrice(subject.getPrice() + 100)  // 다시 notify 호출 → 무한 루프
    }
}
```

### 3. 알림 순서에 의존하지 않기

Observer들의 update() 호출 순서는 보장되지 않는다. 순서에 의존하는 로직은 피해야 한다.

### 4. 비동기 처리 고려

Observer가 많거나 update() 처리가 오래 걸리면 Subject가 블로킹될 수 있다.

```kotlin
// ✅ 비동기 통지
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

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 상태 변화 브로드캐스트 | 주식 가격, 센서 데이터 |
| 이벤트 처리 시스템 | GUI 이벤트, 알림 시스템 |
| 발행-구독 모델 | 메시지 큐, 이벤트 버스 |
| 데이터 바인딩 | MVVM의 ViewModel-View 관계 |

### Spring 환경: ApplicationEvent 활용

```kotlin
// 이벤트 정의 (상태)
data class OrderCreatedEvent(
    val orderId: String,
    val amount: Int
) : ApplicationEvent(orderId)

// Subject 역할 - 이벤트 발행
@Service
class OrderService(
    private val eventPublisher: ApplicationEventPublisher
) {
    fun createOrder(request: OrderRequest): Order {
        val order = orderRepository.save(Order(request))
        
        // 이벤트 발행 - 등록된 모든 Listener에게 통지
        eventPublisher.publishEvent(OrderCreatedEvent(order.id, order.amount))
        
        return order
    }
}

// Observer 역할 - 이벤트 구독
@Component
class EmailNotificationListener {
    @EventListener
    fun onOrderCreated(event: OrderCreatedEvent) {
        println("주문 확인 이메일 발송: ${event.orderId}")
    }
}

@Component
class InventoryListener {
    @EventListener
    fun onOrderCreated(event: OrderCreatedEvent) {
        println("재고 차감 처리: ${event.orderId}")
    }
}

@Component
class PointListener {
    @EventListener
    fun onOrderCreated(event: OrderCreatedEvent) {
        println("포인트 적립: ${event.amount * 0.01}")
    }
}
```

**장점**: 새로운 Listener 추가 시 `@EventListener`만 붙이면 됨, OrderService 수정 불필요

### Kotlin Flow 활용

```kotlin
// StateFlow를 Subject로 활용
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
            println("현재 가격: ${price}원")
        }
    }
}
```

### Before/After 비교

```kotlin
// ❌ Before: 직접 의존 - 강한 결합
class OrderService(
    private val emailService: EmailService,
    private val inventoryService: InventoryService,
    private val pointService: PointService,
    private val analyticsService: AnalyticsService  // 추가할 때마다 주입 필요
) {
    fun createOrder(request: OrderRequest) {
        val order = save(request)
        emailService.sendConfirmation(order)
        inventoryService.decrease(order)
        pointService.accumulate(order)
        analyticsService.track(order)  // 추가할 때마다 코드 수정
    }
}

// ✅ After: Observer 패턴 - 느슨한 결합
class OrderService(
    private val eventPublisher: ApplicationEventPublisher
) {
    fun createOrder(request: OrderRequest) {
        val order = save(request)
        eventPublisher.publishEvent(OrderCreatedEvent(order))
        // 새로운 Listener 추가해도 이 코드는 변경 불필요
    }
}
```

---

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Mediator** | 객체 간 통신을 중재자가 **양방향**으로 조정, Observer는 **단방향** 통지 |
| **Pub/Sub** | Observer 패턴의 확장, 중간에 메시지 브로커(채널)가 존재 |
| **Event Sourcing** | 상태 변경을 이벤트로 저장, Observer는 상태 변경 통지에 집중 |

### Observer vs Pub/Sub

| 비교 항목 | Observer | Pub/Sub |
|----------|----------|---------|
| 결합도 | Subject가 Observer 목록 관리 | Publisher와 Subscriber가 서로 모름 |
| 중개자 | 없음 | 메시지 브로커/채널 존재 |
| 동기/비동기 | 주로 동기 | 주로 비동기 |
| 사용 예 | 객체 내 상태 변화 | 분산 시스템, 마이크로서비스 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 기상 스테이션 예제

### 추천 사이트
- [Refactoring Guru - Observer Pattern](https://refactoring.guru/design-patterns/observer)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)
- [Strategy Pattern](./Strategy%20Pattern.md)
