# Strategy Pattern

## 개념

전략 패턴은 **알고리즘(행위)을 객체로 캡슐화하여 런타임에 동적으로 교체할 수 있게 하는 패턴**이다.

동일한 문제를 해결하는 여러 알고리즘이 존재할 때, 이를 인터페이스로 추상화하고 실행 시점에 구체적인 구현을 선택할 수 있다. GoF 행위 패턴 중 하나로, "**변하는 것을 캡슐화하라**"는 원칙을 따른다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **캡슐화** | 변경 가능한 알고리즘을 별도 객체로 분리 |
| **합성 우선** | 상속보다 합성(Composition)을 사용하여 유연성 확보 |
| **OCP** | 새로운 전략 추가 시 기존 코드 수정 없이 확장 가능 |
| **SRP** | 각 전략은 하나의 알고리즘만 책임 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 조건문 기반 분기 - 새로운 결제 수단 추가마다 코드 수정 필요
fun pay(type: String, amount: Int) {
    when (type) {
        "kakao" -> println("카카오페이 결제: $amount")
        "card" -> println("신용카드 결제: $amount")
        "naver" -> println("네이버페이 결제: $amount") // 추가할 때마다 수정
        // ...계속 늘어남
    }
}
```

- 조건문이 길어지면 가독성과 유지보수성 저하
- 새로운 정책 추가 시 기존 코드 수정 필요 (OCP 위반)
- 알고리즘 테스트가 어려움

### 제공하는 가치

- **확장성**: 새로운 전략 추가 시 기존 코드 수정 불필요
- **테스트 용이성**: 각 전략을 독립적으로 테스트 가능
- **유연성**: 런타임에 전략 교체 가능
- **단일 책임**: 각 전략이 하나의 알고리즘만 담당

---

## 동작 원리

### 구조

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

| 구성요소 | 역할 |
|---------|------|
| **Strategy** | 알고리즘의 공통 인터페이스 정의 |
| **ConcreteStrategy** | 실제 알고리즘 구현체 |
| **Context** | 전략을 사용하는 실행 환경, 전략 참조를 보유 |

### 기본 구현

```kotlin
// Strategy 인터페이스
interface PaymentStrategy {
    fun pay(amount: Int)
}

// ConcreteStrategy 구현체들
class KakaoPayStrategy : PaymentStrategy {
    override fun pay(amount: Int) {
        println("카카오페이 결제: ${amount}원")
    }
}

class CreditCardStrategy : PaymentStrategy {
    override fun pay(amount: Int) {
        println("신용카드 결제: ${amount}원")
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

// 사용
val processor = PaymentProcessor(KakaoPayStrategy())
processor.process(10000)  // 카카오페이 결제: 10000원

processor.changeStrategy(CreditCardStrategy())
processor.process(20000)  // 신용카드 결제: 20000원
```

---

## 주의사항

### 1. 전략 선택 로직의 위치

전략을 선택하는 로직이 어딘가에는 존재해야 한다. Context가 직접 선택하거나, 외부(Factory, DI Container)에서 주입받아야 한다.

```kotlin
// ❌ Context 내부에서 조건문으로 선택 - 전략 패턴의 의미 퇴색
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

// ✅ 외부에서 전략 주입
class PaymentProcessor(private val strategy: PaymentStrategy) {
    fun process(amount: Int) = strategy.pay(amount)
}
```

### 2. 클래스 폭발 주의

전략마다 클래스를 만들면 클래스 수가 급증할 수 있다. Kotlin에서는 함수 타입을 활용하여 완화할 수 있다.

### 3. 단순한 경우에는 과도한 적용 금지

전략이 1-2개이고 변경 가능성이 낮다면 단순한 조건문이 더 나을 수 있다.

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 다양한 알고리즘 선택 | 정렬 알고리즘, 압축 방식 |
| 정책 변경이 잦은 시스템 | 할인 정책, 배송비 계산 |
| 조건에 따른 행위 분기 | 결제 수단, 인증 방식 |
| 테스트 대체가 필요한 경우 | 외부 API 호출을 Mock으로 대체 |

### Spring 환경: Map 기반 DI 활용

```kotlin
// 전략 인터페이스
interface NotificationSender {
    fun send(message: String)
}

// 구현체를 Bean으로 등록 (key는 Bean 이름)
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

// Map으로 모든 전략 주입받기
@Service
class NotificationService(
    private val strategies: Map<String, NotificationSender>
) {
    fun notify(type: String, message: String) {
        strategies[type]?.send(message)
            ?: error("지원되지 않는 타입: $type")
    }
}

// 사용
notificationService.notify("email", "Hello")  // Email: Hello
notificationService.notify("sms", "Hi")       // SMS: Hi
```

**장점**: 새로운 전략 추가 시 구현체만 `@Service`로 등록하면 자동으로 Map에 포함됨

### Kotlin 함수형 스타일

인터페이스 + 클래스 대신 **함수 타입**을 전략으로 사용하면 더 간결하다.

```kotlin
// 함수 타입을 전략으로 정의
typealias PricingStrategy = (Int) -> Int

// 전략들 (람다로 정의)
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

// 사용
val calculator = PriceCalculator(defaultPricing)
println(calculator.calculate(10000))  // 10000

calculator.change(vipPricing)
println(calculator.calculate(10000))  // 8000
```

### Before/After 비교

```kotlin
// ❌ Before: 조건문 기반
class DiscountService {
    fun calculate(type: String, price: Int): Int {
        return when (type) {
            "vip" -> (price * 0.8).toInt()
            "student" -> (price * 0.9).toInt()
            "event" -> (price * 0.7).toInt()  // 이벤트 추가마다 수정
            else -> price
        }
    }
}

// ✅ After: 전략 패턴
interface DiscountStrategy {
    fun apply(price: Int): Int
}

class VipDiscount : DiscountStrategy {
    override fun apply(price: Int) = (price * 0.8).toInt()
}

class StudentDiscount : DiscountStrategy {
    override fun apply(price: Int) = (price * 0.9).toInt()
}

// 새 할인 정책 추가 시 클래스만 추가하면 됨
class EventDiscount : DiscountStrategy {
    override fun apply(price: Int) = (price * 0.7).toInt()
}

class DiscountService(private val strategy: DiscountStrategy) {
    fun calculate(price: Int) = strategy.apply(price)
}
```

---

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Template Method** | 상속 기반으로 알고리즘 골격은 고정, 일부 단계만 오버라이드 |
| **State** | 내부 상태에 따라 전략이 **자동으로** 변경됨 (Context가 아닌 State가 전환 주도) |
| **Command** | 요청 자체를 객체화 (실행 취소/로깅 등 목적), Strategy는 알고리즘 교체 목적 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 전략 패턴을 첫 번째로 소개

### 추천 사이트
- [Refactoring Guru - Strategy Pattern](https://refactoring.guru/design-patterns/strategy)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)