# Factory Method Pattern

## 개념

팩토리 메서드 패턴은 **객체 생성을 서브클래스에 위임하여 생성할 객체의 타입을 결정하는 패턴**이다.

객체를 직접 `new`로 생성하지 않고, 팩토리 메서드를 통해 생성함으로써 생성 로직을 캡슐화한다. GoF 생성 패턴 중 하나로, "**구체적인 클래스가 아닌 인터페이스에 의존하라**"는 원칙을 따른다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **캡슐화** | 객체 생성 로직을 팩토리 메서드 내부로 숨김 |
| **DIP** | 호출부는 구체 클래스가 아닌 인터페이스에 의존 |
| **OCP** | 새로운 제품 추가 시 기존 코드 수정 없이 확장 가능 |
| **SRP** | 생성 책임과 사용 책임을 분리 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 직접 생성 - 구체 클래스에 강하게 결합
class NotificationService {
    fun send(type: String, message: String) {
        val notification = when (type) {
            "email" -> EmailNotification()
            "slack" -> SlackNotification()
            "sms" -> SmsNotification()  // 추가할 때마다 수정
            else -> throw IllegalArgumentException()
        }
        notification.send(message)
    }
}
```

- 새로운 타입 추가 시 기존 코드 수정 필요 (OCP 위반)
- 생성 로직과 사용 로직이 섞여 있음
- 테스트 시 특정 구현체로 대체하기 어려움

### 제공하는 가치

- **결합도 감소**: 호출부는 구체 클래스를 알 필요 없음
- **확장성**: 새로운 제품 추가 시 팩토리만 추가하면 됨
- **테스트 용이성**: Mock 팩토리로 쉽게 대체 가능
- **생성 로직 집중화**: 초기화, 검증, 로깅 등을 한 곳에서 관리

---

## 동작 원리

### 구조

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

| 구성요소 | 역할 |
|---------|------|
| **Product** | 생성될 객체의 공통 인터페이스 |
| **ConcreteProduct** | 실제 생성되는 객체 구현체 |
| **Creator** | 팩토리 메서드를 선언하는 추상 클래스 |
| **ConcreteCreator** | 팩토리 메서드를 구현하여 특정 제품 생성 |

### 기본 구현

```kotlin
// Product 인터페이스
interface Notification {
    fun send(message: String)
}

// ConcreteProduct 구현체들
class EmailNotification : Notification {
    override fun send(message: String) = println("Email: $message")
}

class SlackNotification : Notification {
    override fun send(message: String) = println("Slack: $message")
}

// Creator 추상 클래스
abstract class NotificationFactory {
    // 팩토리 메서드 - 서브클래스에서 구현
    abstract fun create(): Notification

    // 템플릿 메서드 - 생성된 객체를 사용하는 로직
    fun notify(message: String) {
        val notification = create()
        notification.send(message)
    }
}

// ConcreteCreator 구현체들
class EmailNotificationFactory : NotificationFactory() {
    override fun create(): Notification = EmailNotification()
}

class SlackNotificationFactory : NotificationFactory() {
    override fun create(): Notification = SlackNotification()
}

// 사용
val factory: NotificationFactory = EmailNotificationFactory()
factory.notify("Hello")  // Email: Hello

val factory2: NotificationFactory = SlackNotificationFactory()
factory2.notify("Hi from Slack")  // Slack: Hi from Slack
```

---

## 주의사항

### 1. 단순한 경우에는 과설계

생성할 객체 타입이 1-2개이고 변경 가능성이 낮다면 직접 생성이 더 간단하다.

```kotlin
// ❌ 과설계 - 타입이 하나뿐인데 팩토리 패턴 적용
abstract class UserFactory {
    abstract fun create(): User
}
class DefaultUserFactory : UserFactory() {
    override fun create() = User()
}

// ✅ 직접 생성으로 충분
val user = User()
```

### 2. 팩토리 선택 로직의 위치

어떤 팩토리를 사용할지 결정하는 로직이 어딘가에는 필요하다.

```kotlin
// 팩토리 선택을 위한 Simple Factory (정적 메서드)
object NotificationFactoryProvider {
    fun getFactory(type: String): NotificationFactory = when (type) {
        "email" -> EmailNotificationFactory()
        "slack" -> SlackNotificationFactory()
        else -> throw IllegalArgumentException("Unknown type: $type")
    }
}
```

### 3. Abstract Factory와 혼동 주의

| 패턴 | 목적 |
|------|------|
| **Factory Method** | 하나의 객체 생성을 서브클래스에 위임 |
| **Abstract Factory** | 관련된 객체 **군(family)**을 생성하는 인터페이스 제공 |

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 생성할 객체 타입이 다양함 | 문서 편집기의 다양한 문서 타입 |
| 생성 로직에 추가 작업 필요 | 로깅, 검증, 상태 초기화 |
| 프레임워크/라이브러리 설계 | 사용자가 확장할 수 있는 구조 제공 |
| 테스트 대체가 필요한 경우 | Mock 객체 생성 팩토리 |

### Spring 환경: Bean Lookup 기반 Factory

```kotlin
// Product 인터페이스
interface PaymentProcessor {
    fun process(amount: Int)
}

// Bean으로 등록된 ConcreteProduct들
@Service("card")
class CardPaymentProcessor : PaymentProcessor {
    override fun process(amount: Int) = println("Card: ${amount}원")
}

@Service("kakao")
class KakaoPaymentProcessor : PaymentProcessor {
    override fun process(amount: Int) = println("KakaoPay: ${amount}원")
}

// Factory - Map으로 모든 구현체 주입받음
@Service
class PaymentProcessorFactory(
    private val processors: Map<String, PaymentProcessor>
) {
    fun get(type: String): PaymentProcessor =
        processors[type] ?: error("Unsupported type: $type")
}

// 사용
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

**장점**: 새로운 결제 수단 추가 시 `@Service`로 Bean 등록만 하면 자동으로 Factory에 포함됨

### Kotlin 간소화 버전: Companion Object 활용

```kotlin
interface Document {
    fun open()
    
    companion object {
        // 팩토리 메서드를 companion object에 정의
        fun create(type: String): Document = when (type) {
            "pdf" -> PdfDocument()
            "word" -> WordDocument()
            else -> throw IllegalArgumentException("Unknown type: $type")
        }
    }
}

class PdfDocument : Document {
    override fun open() = println("PDF 문서 열기")
}

class WordDocument : Document {
    override fun open() = println("Word 문서 열기")
}

// 사용
val doc = Document.create("pdf")
doc.open()  // PDF 문서 열기
```

### Before/After 비교

```kotlin
// ❌ Before: 직접 생성 - 강한 결합
class ReportService {
    fun generate(type: String): Report {
        val exporter = when (type) {
            "pdf" -> PdfExporter()
            "excel" -> ExcelExporter()
            "csv" -> CsvExporter()  // 추가마다 수정
            else -> throw IllegalArgumentException()
        }
        return exporter.export(data)
    }
}

// ✅ After: 팩토리 메서드 - 느슨한 결합
interface ReportExporter {
    fun export(data: Data): Report
}

abstract class ReportExporterFactory {
    abstract fun create(): ReportExporter
}

class PdfExporterFactory : ReportExporterFactory() {
    override fun create() = PdfExporter()
}

// 새 포맷 추가 시 Factory만 추가
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

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Simple Factory** | 정적 메서드로 객체 생성, 상속 구조 없음 (GoF 패턴 아님) |
| **Abstract Factory** | 관련 객체 **군**을 생성, 여러 팩토리 메서드를 포함 |
| **Builder** | 복잡한 객체를 **단계별로** 생성, 생성 과정 자체에 초점 |
| **Prototype** | 기존 객체를 **복사**하여 새 객체 생성 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 피자 가게 예제로 쉽게 설명

### 추천 사이트
- [Refactoring Guru - Factory Method](https://refactoring.guru/design-patterns/factory-method)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)
- [Abstract Factory](./Abstract%20Factory.md)
- [Strategy Pattern](../Behavioral/Strategy%20Pattern.md)