# Abstract Factory Pattern

## 개념

추상 팩토리 패턴은 **서로 관련된 객체들의 군(family)을 생성하는 인터페이스를 제공하는 패턴**이다.

한 번 선택한 제품군은 일관되게 적용되어야 하고, 서로 다른 제품군의 객체가 섞이지 않도록 보장한다. GoF 생성 패턴 중 하나로, "**구체적인 클래스를 지정하지 않고 관련 객체들을 생성**"하는 것이 핵심이다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **제품군 일관성** | 생성되는 객체들이 항상 같은 제품군에 속함을 보장 |
| **캡슐화** | 구체적인 클래스 생성 로직을 팩토리 내부로 숨김 |
| **DIP** | 클라이언트는 추상 인터페이스에만 의존 |
| **OCP** | 새로운 제품군 추가 시 기존 코드 수정 없이 확장 가능 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 직접 생성 - 서로 다른 제품군이 섞일 위험
class UIBuilder {
    fun build(theme: String) {
        val button = if (theme == "dark") DarkButton() else LightButton()
        val checkbox = if (theme == "dark") DarkCheckbox() else LightCheckbox()
        val dialog = if (theme == "dark") LightDialog() else DarkDialog()  // 실수로 잘못된 테마!
        
        // 제품군 불일치 발생
    }
}
```

- 제품군이 섞이면 UI 일관성이 깨짐
- 새로운 제품군(예: Blue 테마) 추가 시 모든 조건문 수정 필요
- 제품 종류가 늘어날수록 실수 가능성 증가

### 제공하는 가치

- **일관성 보장**: 같은 팩토리에서 생성된 객체들은 항상 호환됨
- **제품군 전환 용이**: 팩토리만 교체하면 전체 제품군이 변경됨
- **확장성**: 새로운 제품군 추가가 쉬움
- **결합도 감소**: 클라이언트는 구체 클래스를 알 필요 없음

---

## 동작 원리

### 구조

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

| 구성요소 | 역할 |
|---------|------|
| **AbstractFactory** | 제품군 생성 메서드들을 선언하는 인터페이스 |
| **ConcreteFactory** | 특정 제품군의 객체들을 생성하는 구현체 |
| **AbstractProduct** | 생성될 제품의 공통 인터페이스 |
| **ConcreteProduct** | 특정 제품군에 속하는 실제 제품 구현체 |

### 기본 구현

```kotlin
// AbstractProduct - 제품 인터페이스들
interface Button {
    fun render()
}

interface Checkbox {
    fun render()
}

// ConcreteProduct - Dark 테마 제품군
class DarkButton : Button {
    override fun render() = println("Dark Button")
}

class DarkCheckbox : Checkbox {
    override fun render() = println("Dark Checkbox")
}

// ConcreteProduct - Light 테마 제품군
class LightButton : Button {
    override fun render() = println("Light Button")
}

class LightCheckbox : Checkbox {
    override fun render() = println("Light Checkbox")
}

// AbstractFactory - 팩토리 인터페이스
interface UIComponentFactory {
    fun createButton(): Button
    fun createCheckbox(): Checkbox
}

// ConcreteFactory - Dark 테마 팩토리
class DarkThemeFactory : UIComponentFactory {
    override fun createButton(): Button = DarkButton()
    override fun createCheckbox(): Checkbox = DarkCheckbox()
}

// ConcreteFactory - Light 테마 팩토리
class LightThemeFactory : UIComponentFactory {
    override fun createButton(): Button = LightButton()
    override fun createCheckbox(): Checkbox = LightCheckbox()
}

// 클라이언트 - 추상 인터페이스에만 의존
fun buildUI(factory: UIComponentFactory) {
    val button = factory.createButton()
    val checkbox = factory.createCheckbox()
    
    button.render()
    checkbox.render()
}

// 사용 - 팩토리만 교체하면 전체 제품군이 변경됨
buildUI(DarkThemeFactory())   // Dark Button, Dark Checkbox
buildUI(LightThemeFactory())  // Light Button, Light Checkbox
```

---

## 주의사항

### 1. 단일 객체 생성에는 과설계

관련된 객체 **군**을 생성해야 할 때만 사용한다. 단일 객체 생성에는 Factory Method가 더 적합하다.

```kotlin
// ❌ 과설계 - 제품이 하나뿐인데 Abstract Factory 사용
interface SingleProductFactory {
    fun createProduct(): Product
}

// ✅ Factory Method로 충분
abstract class ProductFactory {
    abstract fun create(): Product
}
```

### 2. 제품 추가 시 인터페이스 변경 필요

새로운 제품 종류(예: Dialog)를 추가하면 모든 팩토리 인터페이스와 구현체를 수정해야 한다.

```kotlin
// 새로운 제품 추가 시
interface UIComponentFactory {
    fun createButton(): Button
    fun createCheckbox(): Checkbox
    fun createDialog(): Dialog  // 모든 ConcreteFactory에 구현 필요
}
```

### 3. Factory Method와의 관계

Abstract Factory 내부에서 각 제품을 생성하는 메서드는 사실상 Factory Method다.

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 관련 객체들의 일관성 필요 | UI 테마, OS별 위젯 |
| 제품군 전환이 필요 | 개발/운영 환경별 설정 |
| 플랫폼 독립적 설계 | 크로스 플랫폼 앱 |
| 프레임워크 확장 포인트 | 플러그인 시스템 |

### Spring 환경: 클라우드 환경별 저장소 전략

```kotlin
// AbstractProduct - 저장소 인터페이스
interface Storage {
    fun save(data: String)
}

interface Cache {
    fun get(key: String): String?
}

// AbstractFactory - 저장소 팩토리 인터페이스
interface StorageFactory {
    fun createStorage(): Storage
    fun createCache(): Cache
}

// ConcreteFactory - AWS 제품군
@Component
@Profile("aws")
class AWSStorageFactory : StorageFactory {
    override fun createStorage(): Storage = S3Storage()
    override fun createCache(): Cache = ElastiCache()
}

// ConcreteFactory - GCP 제품군
@Component
@Profile("gcp")
class GCPStorageFactory : StorageFactory {
    override fun createStorage(): Storage = GcsStorage()
    override fun createCache(): Cache = Memorystore()
}

// 클라이언트 - 추상 인터페이스에만 의존
@Service
class DataService(
    private val storageFactory: StorageFactory
) {
    fun process(data: String) {
        val storage = storageFactory.createStorage()
        val cache = storageFactory.createCache()
        
        storage.save(data)
        // 항상 같은 클라우드 제품군 사용 보장
    }
}
```

**장점**: Profile만 변경하면 전체 클라우드 인프라 제품군이 일괄 교체됨

### Before/After 비교

```kotlin
// ❌ Before: 조건문 기반 - 제품군 섞임 위험
class DatabaseService(private val env: String) {
    fun getConnection(): Connection {
        return if (env == "prod") ProdConnection() else DevConnection()
    }
    
    fun getRepository(): Repository {
        return if (env == "prod") ProdRepository() else DevRepository()
    }
    
    fun getCache(): Cache {
        // 실수로 prod 환경에 dev 캐시 사용 가능
        return if (env == "dev") ProdCache() else DevCache()  // 버그!
    }
}

// ✅ After: Abstract Factory - 제품군 일관성 보장
interface DatabaseFactory {
    fun createConnection(): Connection
    fun createRepository(): Repository
    fun createCache(): Cache
}

class ProdDatabaseFactory : DatabaseFactory {
    override fun createConnection() = ProdConnection()
    override fun createRepository() = ProdRepository()
    override fun createCache() = ProdCache()  // 항상 Prod 제품군
}

class DevDatabaseFactory : DatabaseFactory {
    override fun createConnection() = DevConnection()
    override fun createRepository() = DevRepository()
    override fun createCache() = DevCache()  // 항상 Dev 제품군
}

class DatabaseService(private val factory: DatabaseFactory) {
    fun setup() {
        val conn = factory.createConnection()
        val repo = factory.createRepository()
        val cache = factory.createCache()
        // 제품군 섞임 불가능
    }
}
```

---

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Factory Method** | **단일 객체** 생성을 서브클래스에 위임 |
| **Abstract Factory** | **관련 객체 군** 생성, 여러 Factory Method 포함 |
| **Builder** | 복잡한 객체를 **단계별로** 생성, 생성 과정에 초점 |
| **Prototype** | 기존 객체를 **복사**하여 새 객체 생성 |

### Factory Method vs Abstract Factory

| 비교 항목 | Factory Method | Abstract Factory |
|----------|----------------|------------------|
| 생성 대상 | 단일 제품 | 제품군(여러 관련 제품) |
| 확장 방식 | 서브클래스 추가 | 팩토리 구현체 추가 |
| 복잡도 | 상대적으로 단순 | 구조가 복잡 |
| 사용 시점 | 제품 종류 확장 | 제품군 전환 필요 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 피자 재료 팩토리 예제

### 추천 사이트
- [Refactoring Guru - Abstract Factory](https://refactoring.guru/design-patterns/abstract-factory)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)
- [Factory Method](./Factory%20Method.md)