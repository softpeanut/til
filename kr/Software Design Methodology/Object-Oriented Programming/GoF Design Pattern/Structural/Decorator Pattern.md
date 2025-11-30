# Decorator Pattern

## 개념

데코레이터 패턴은 **객체에 동적으로 새로운 책임(기능)을 추가할 수 있게 하는 패턴**이다.

상속 대신 합성(Composition)을 사용하여 기존 객체를 래핑하고, 런타임에 기능을 유연하게 확장한다. GoF 구조 패턴 중 하나로, "**상속 없이 기능을 확장**"하는 것이 핵심이다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **OCP** | 기존 코드 수정 없이 새로운 기능 추가 가능 |
| **합성 우선** | 상속보다 합성을 사용하여 유연성 확보 |
| **단일 책임** | 각 데코레이터는 하나의 기능만 담당 |
| **투명성** | 데코레이터와 원본 객체가 동일한 인터페이스 구현 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 상속 기반 확장 - 클래스 폭발
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

// 조합이 늘어날수록 클래스가 기하급수적으로 증가
// Milk, Sugar, Whip, Shot, Syrup... 조합 = 수십 개의 클래스
```

- 기능 조합마다 새로운 클래스 필요 (클래스 폭발)
- 런타임에 기능을 추가/제거할 수 없음
- 상속 계층이 깊어지면 유지보수 어려움

### 제공하는 가치

- **유연한 확장**: 런타임에 기능을 동적으로 추가/제거
- **클래스 수 감소**: 조합 대신 데코레이터 체이닝으로 해결
- **단일 책임**: 각 데코레이터가 하나의 기능만 담당
- **기존 코드 보존**: 원본 클래스 수정 없이 기능 확장

---

## 동작 원리

### 구조

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

| 구성요소 | 역할 |
|---------|------|
| **Component** | 기본 인터페이스 정의 |
| **ConcreteComponent** | 기본 기능을 구현한 원본 객체 |
| **Decorator** | Component를 래핑하고 동일한 인터페이스 구현 |
| **ConcreteDecorator** | 추가 기능을 구현한 데코레이터 |

### 기본 구현

```kotlin
// Component 인터페이스
interface Coffee {
    fun cost(): Int
    fun description(): String
}

// ConcreteComponent - 기본 커피
class Espresso : Coffee {
    override fun cost(): Int = 3000
    override fun description(): String = "에스프레소"
}

class Americano : Coffee {
    override fun cost(): Int = 3500
    override fun description(): String = "아메리카노"
}

// Decorator 추상 클래스
abstract class CoffeeDecorator(
    protected val coffee: Coffee
) : Coffee {
    override fun cost(): Int = coffee.cost()
    override fun description(): String = coffee.description()
}

// ConcreteDecorator들
class MilkDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
    override fun cost(): Int = super.cost() + 500
    override fun description(): String = "${super.description()} + 우유"
}

class SugarDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
    override fun cost(): Int = super.cost() + 200
    override fun description(): String = "${super.description()} + 설탕"
}

class WhipDecorator(coffee: Coffee) : CoffeeDecorator(coffee) {
    override fun cost(): Int = super.cost() + 700
    override fun description(): String = "${super.description()} + 휘핑크림"
}

// 사용 - 데코레이터 체이닝
val espresso: Coffee = Espresso()
println("${espresso.description()}: ${espresso.cost()}원")
// 에스프레소: 3000원

val latte: Coffee = MilkDecorator(Espresso())
println("${latte.description()}: ${latte.cost()}원")
// 에스프레소 + 우유: 3500원

val sweetLatte: Coffee = SugarDecorator(MilkDecorator(Espresso()))
println("${sweetLatte.description()}: ${sweetLatte.cost()}원")
// 에스프레소 + 우유 + 설탕: 3700원

val fancyCoffee: Coffee = WhipDecorator(SugarDecorator(MilkDecorator(Americano())))
println("${fancyCoffee.description()}: ${fancyCoffee.cost()}원")
// 아메리카노 + 우유 + 설탕 + 휘핑크림: 4900원
```

### 래핑 구조 시각화

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

## 주의사항

### 1. 데코레이터 순서에 따른 결과 변화

순서에 따라 결과가 달라질 수 있음을 인지해야 한다.

```kotlin
// 할인 → 세금 vs 세금 → 할인
val price = 10000

// 할인 10% 후 세금 10%
val discountFirst = DiscountDecorator(TaxDecorator(BasePrice(price)))
// 10000 * 0.9 * 1.1 = 9900

// 세금 10% 후 할인 10%
val taxFirst = TaxDecorator(DiscountDecorator(BasePrice(price)))
// 10000 * 1.1 * 0.9 = 9900 (이 경우는 동일하지만, 다른 로직에서는 다를 수 있음)
```

### 2. 많은 데코레이터는 복잡성 증가

데코레이터를 너무 많이 중첩하면 디버깅이 어려워진다.

```kotlin
// ❌ 과도한 중첩 - 디버깅 어려움
val complex = D(C(B(A(component))))

// ✅ 빌더 패턴과 함께 사용하여 가독성 향상
val order = CoffeeBuilder()
    .base(Americano())
    .addMilk()
    .addSugar()
    .addWhip()
    .build()
```

### 3. 동일 데코레이터 중복 적용

의도치 않게 같은 데코레이터가 여러 번 적용될 수 있다.

```kotlin
// 우유 두 번 추가 (의도적인지 실수인지?)
val doubleMilk = MilkDecorator(MilkDecorator(Espresso()))
```

### 4. 타입 체크 시 주의

데코레이터로 래핑된 객체는 원본 타입과 다르다.

```kotlin
val coffee: Coffee = MilkDecorator(Espresso())

coffee is Espresso  // false - MilkDecorator 타입
coffee is Coffee    // true - 인터페이스는 동일
```

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 기능을 동적으로 추가/제거 | 스트림 처리, 로깅, 캐싱 |
| 기능 조합이 다양함 | 음료 옵션, 피자 토핑 |
| 상속으로 해결하기 어려움 | final 클래스 확장, 다중 기능 조합 |
| 기존 코드 수정 불가 | 라이브러리 클래스 확장 |

### Java I/O Stream 예제

```kotlin
// Java의 대표적인 데코레이터 패턴 사용 예
import java.io.*

// 기본 스트림에 버퍼링과 라인 읽기 기능 추가
val reader = BufferedReader(  // Decorator
    InputStreamReader(         // Decorator
        FileInputStream("file.txt")  // ConcreteComponent
    )
)

// 압축 + 버퍼링 스트림
val outputStream = BufferedOutputStream(  // Decorator
    GZIPOutputStream(                      // Decorator
        FileOutputStream("file.gz")        // ConcreteComponent
    )
)
```

### Spring 환경: 서비스 계층 데코레이팅

```kotlin
// Component 인터페이스
interface UserService {
    fun getUser(id: Long): User
}

// ConcreteComponent - 실제 서비스
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

// Decorator - 캐싱 기능 추가
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

// Decorator - 로깅 기능 추가
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

### Kotlin 확장 함수와 조합

```kotlin
// Kotlin에서는 확장 함수와 함께 사용 가능
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

// DSL 스타일 빌더
class TextProcessorBuilder {
    private var processor: TextProcessor = BasicProcessor()
    
    fun uppercase() = apply { processor = UpperCaseDecorator(processor) }
    fun trim() = apply { processor = TrimDecorator(processor) }
    fun build(): TextProcessor = processor
}

fun textProcessor(block: TextProcessorBuilder.() -> Unit): TextProcessor {
    return TextProcessorBuilder().apply(block).build()
}

// 사용
val processor = textProcessor {
    trim()
    uppercase()
}

println(processor.process("  hello world  "))  // "HELLO WORLD"
```

### Before/After 비교

```kotlin
// ❌ Before: 상속 기반 - 클래스 폭발
open class Notifier {
    open fun send(message: String) { /* 기본 알림 */ }
}

class EmailNotifier : Notifier() { /* 이메일 */ }
class SlackNotifier : Notifier() { /* 슬랙 */ }
class EmailSlackNotifier : Notifier() { /* 이메일 + 슬랙 */ }
class EmailSlackSmsNotifier : Notifier() { /* 이메일 + 슬랙 + SMS */ }
// ... 조합마다 클래스 필요

// ✅ After: 데코레이터 패턴 - 유연한 조합
interface Notifier {
    fun send(message: String)
}

class BasicNotifier : Notifier {
    override fun send(message: String) = println("기본 알림: $message")
}

abstract class NotifierDecorator(
    protected val notifier: Notifier
) : Notifier

class EmailDecorator(notifier: Notifier) : NotifierDecorator(notifier) {
    override fun send(message: String) {
        notifier.send(message)
        println("이메일 발송: $message")
    }
}

class SlackDecorator(notifier: Notifier) : NotifierDecorator(notifier) {
    override fun send(message: String) {
        notifier.send(message)
        println("슬랙 발송: $message")
    }
}

class SmsDecorator(notifier: Notifier) : NotifierDecorator(notifier) {
    override fun send(message: String) {
        notifier.send(message)
        println("SMS 발송: $message")
    }
}

// 런타임에 원하는 조합 생성
val allChannels = SmsDecorator(SlackDecorator(EmailDecorator(BasicNotifier())))
allChannels.send("긴급 알림")
```

---

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Proxy** | 동일 인터페이스로 **접근 제어** (지연 로딩, 권한 검사), Decorator는 **기능 추가** |
| **Composite** | 트리 구조로 **부분-전체** 관계 표현, Decorator는 **선형 래핑** |
| **Strategy** | 알고리즘 전체를 **교체**, Decorator는 기존 동작에 **추가** |

### Decorator vs Proxy

| 비교 항목 | Decorator | Proxy |
|----------|-----------|-------|
| 목적 | 기능 추가/확장 | 접근 제어 |
| 클라이언트 인지 | 종종 데코레이터 체인을 직접 구성 | 프록시 존재를 모름 |
| 래핑 개수 | 여러 개 체이닝 가능 | 보통 하나 |
| 사용 예 | Java I/O Stream | 지연 로딩, 캐싱 프록시 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 스타버즈 커피 예제

### 추천 사이트
- [Refactoring Guru - Decorator Pattern](https://refactoring.guru/design-patterns/decorator)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)
- [Composite Pattern](./Composite%20Pattern.md)
- [Strategy Pattern](../Behavioral/Strategy%20Pattern.md)
