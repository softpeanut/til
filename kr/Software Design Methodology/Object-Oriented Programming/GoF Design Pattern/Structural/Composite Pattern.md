# Composite Pattern

## 개념

컴포지트 패턴은 **객체들을 트리 구조로 구성하여 개별 객체와 복합 객체를 동일하게 다룰 수 있게 하는 패턴**이다.

클라이언트가 단일 객체(Leaf)와 복합 객체(Composite)를 구분하지 않고 동일한 인터페이스로 처리할 수 있다. GoF 구조 패턴 중 하나로, "**부분-전체 계층 구조를 표현**"하는 것이 핵심이다.

### 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **단일 책임** | 개별 객체와 복합 객체 모두 동일한 인터페이스 구현 |
| **투명성** | 클라이언트는 Leaf와 Composite를 구분할 필요 없음 |
| **재귀적 구성** | Composite는 자식으로 Leaf 또는 다른 Composite를 가질 수 있음 |
| **일관된 처리** | 트리 구조의 모든 노드를 동일한 방식으로 처리 |

---

## 왜 필요한가

### 해결하려는 문제

```kotlin
// ❌ 타입별로 다르게 처리 - 복잡하고 확장 어려움
fun calculatePrice(item: Any): Int {
    return when (item) {
        is Product -> item.price
        is Box -> {
            var total = 0
            for (child in item.items) {
                total += when (child) {
                    is Product -> child.price
                    is Box -> calculatePrice(child)  // 재귀 호출 필요
                    else -> 0
                }
            }
            total
        }
        else -> 0
    }
}
```

- 타입 체크 로직이 곳곳에 산재
- 새로운 타입 추가 시 모든 조건문 수정 필요
- 중첩 구조가 깊어질수록 코드 복잡도 증가

### 제공하는 가치

- **단순화**: 복잡한 트리 구조를 단순한 인터페이스로 다룸
- **확장성**: 새로운 Leaf/Composite 타입 추가가 쉬움
- **일관성**: 클라이언트 코드가 단순하고 일관됨
- **재귀적 처리**: 트리 순회 로직이 자연스럽게 구현됨

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
│      │      Leaf       │         │     Composite       │         │
│      │  ─────────────  │         │  ─────────────────  │         │
│      │  + operation()  │         │  - children: List   │         │
│      └─────────────────┘         │  + operation()      │         │
│                                  │  + add(component)   │         │
│                                  │  + remove(component)│         │
│                                  └─────────────────────┘         │
│                                           │                      │
│                                           │ contains             │
│                                           ▼                      │
│                                    ┌──────────────┐              │
│                                    │  Component   │              │
│                                    └──────────────┘              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

| 구성요소 | 역할 |
|---------|------|
| **Component** | Leaf와 Composite의 공통 인터페이스 정의 |
| **Leaf** | 트리의 말단 노드, 자식을 가지지 않음 |
| **Composite** | 자식 Component들을 포함하고 관리, 자식에게 작업 위임 |

### 기본 구현

```kotlin
// Component 인터페이스
interface FileSystemItem {
    fun getSize(): Long
    fun getName(): String
}

// Leaf - 파일
class File(
    private val name: String,
    private val size: Long
) : FileSystemItem {
    
    override fun getSize(): Long = size
    
    override fun getName(): String = name
}

// Composite - 폴더
class Folder(
    private val name: String
) : FileSystemItem {
    
    private val children = mutableListOf<FileSystemItem>()
    
    fun add(item: FileSystemItem) {
        children.add(item)
    }
    
    fun remove(item: FileSystemItem) {
        children.remove(item)
    }
    
    override fun getSize(): Long {
        // 재귀적으로 모든 자식의 크기 합산
        return children.sumOf { it.getSize() }
    }
    
    override fun getName(): String = name
}

// 사용
val root = Folder("root")
val documents = Folder("documents")
val pictures = Folder("pictures")

documents.add(File("resume.pdf", 1024))
documents.add(File("report.docx", 2048))

pictures.add(File("photo1.jpg", 5000))
pictures.add(File("photo2.jpg", 3000))

root.add(documents)
root.add(pictures)
root.add(File("readme.txt", 100))

// 클라이언트는 Leaf와 Composite를 구분하지 않고 동일하게 처리
println(root.getSize())       // 11172 (모든 파일 크기 합계)
println(documents.getSize())  // 3072
println(pictures.getSize())   // 8000
```

### 트리 구조 예시

```
root (Folder)
├── documents (Folder)
│   ├── resume.pdf (File: 1024)
│   └── report.docx (File: 2048)
├── pictures (Folder)
│   ├── photo1.jpg (File: 5000)
│   └── photo2.jpg (File: 3000)
└── readme.txt (File: 100)
```

---

## 주의사항

### 1. 타입 안정성 vs 투명성 트레이드오프

Component에 add/remove를 포함하면 투명성은 높지만, Leaf에서 의미 없는 메서드를 구현해야 한다.

```kotlin
// 투명성 우선 - Component에 add/remove 포함
interface Component {
    fun operation()
    fun add(component: Component)    // Leaf에서는 의미 없음
    fun remove(component: Component) // Leaf에서는 의미 없음
}

class Leaf : Component {
    override fun operation() { /* ... */ }
    override fun add(component: Component) {
        throw UnsupportedOperationException()  // 또는 무시
    }
    override fun remove(component: Component) {
        throw UnsupportedOperationException()
    }
}

// 타입 안정성 우선 - Composite에만 add/remove 정의
interface Component {
    fun operation()
}

class Composite : Component {
    private val children = mutableListOf<Component>()
    fun add(component: Component) { children.add(component) }
    fun remove(component: Component) { children.remove(component) }
    override fun operation() { /* ... */ }
}
```

### 2. 순환 참조 주의

Composite가 자기 자신을 자식으로 추가하면 무한 루프 발생

```kotlin
// ❌ 순환 참조 위험
val folder = Folder("test")
folder.add(folder)  // 무한 루프!

// ✅ 순환 참조 방지
class Folder(private val name: String) : FileSystemItem {
    private val children = mutableListOf<FileSystemItem>()
    
    fun add(item: FileSystemItem) {
        if (item === this) throw IllegalArgumentException("Cannot add self")
        if (containsAncestor(item)) throw IllegalArgumentException("Circular reference")
        children.add(item)
    }
}
```

### 3. 성능 고려

깊은 트리 구조에서 재귀 호출이 많으면 성능 저하 가능. 필요시 캐싱 고려.

```kotlin
class Folder(private val name: String) : FileSystemItem {
    private var cachedSize: Long? = null
    
    override fun getSize(): Long {
        return cachedSize ?: children.sumOf { it.getSize() }.also { cachedSize = it }
    }
    
    fun invalidateCache() {
        cachedSize = null
    }
}
```

---

## 실전 적용

### 적용 시나리오

| 상황 | 예시 |
|------|------|
| 트리/계층 구조 표현 | 파일 시스템, 조직도 |
| UI 컴포넌트 구성 | View 계층, 위젯 트리 |
| 메뉴 구조 | 메뉴 > 서브메뉴 > 메뉴 아이템 |
| 수식 표현 | 연산자와 피연산자의 트리 구조 |

### 주문 시스템 예제

```kotlin
// Component
interface OrderItem {
    fun getPrice(): Int
    fun getDescription(): String
}

// Leaf - 단일 상품
class Product(
    private val name: String,
    private val price: Int
) : OrderItem {
    
    override fun getPrice(): Int = price
    
    override fun getDescription(): String = "$name: ${price}원"
}

// Composite - 상품 묶음 (세트 메뉴, 패키지 등)
class ProductBundle(
    private val name: String,
    private val discountRate: Double = 0.0
) : OrderItem {
    
    private val items = mutableListOf<OrderItem>()
    
    fun add(item: OrderItem) {
        items.add(item)
    }
    
    override fun getPrice(): Int {
        val total = items.sumOf { it.getPrice() }
        return (total * (1 - discountRate)).toInt()
    }
    
    override fun getDescription(): String {
        val itemDescriptions = items.joinToString("\n  ") { it.getDescription() }
        return "$name (${(discountRate * 100).toInt()}% 할인):\n  $itemDescriptions"
    }
}

// 사용
val burger = Product("햄버거", 5000)
val fries = Product("감자튀김", 2000)
val drink = Product("콜라", 1500)

// 세트 메뉴 (10% 할인)
val comboMeal = ProductBundle("버거 세트", 0.1)
comboMeal.add(burger)
comboMeal.add(fries)
comboMeal.add(drink)

// 주문에 단품과 세트 모두 포함
val order = ProductBundle("주문")
order.add(comboMeal)
order.add(Product("아이스크림", 1000))

println(order.getDescription())
println("총 가격: ${order.getPrice()}원")

// 출력:
// 주문 (0% 할인):
//   버거 세트 (10% 할인):
//     햄버거: 5000원
//     감자튀김: 2000원
//     콜라: 1500원
//   아이스크림: 1000원
// 총 가격: 8650원
```

### Spring 환경: 권한 체계

```kotlin
// Component - 권한
interface Permission {
    fun hasAccess(resource: String): Boolean
    fun getPermissionName(): String
}

// Leaf - 단일 권한
@Component
class ReadPermission : Permission {
    override fun hasAccess(resource: String): Boolean = 
        resource.startsWith("read:")
    
    override fun getPermissionName(): String = "READ"
}

@Component
class WritePermission : Permission {
    override fun hasAccess(resource: String): Boolean = 
        resource.startsWith("write:")
    
    override fun getPermissionName(): String = "WRITE"
}

// Composite - 권한 그룹 (역할)
class Role(
    private val name: String
) : Permission {
    
    private val permissions = mutableListOf<Permission>()
    
    fun addPermission(permission: Permission) {
        permissions.add(permission)
    }
    
    override fun hasAccess(resource: String): Boolean = 
        permissions.any { it.hasAccess(resource) }
    
    override fun getPermissionName(): String = name
}

// 사용
val adminRole = Role("ADMIN")
adminRole.addPermission(ReadPermission())
adminRole.addPermission(WritePermission())

val userRole = Role("USER")
userRole.addPermission(ReadPermission())

// 클라이언트는 단일 권한이든 역할이든 동일하게 처리
fun checkAccess(permission: Permission, resource: String) {
    if (permission.hasAccess(resource)) {
        println("${permission.getPermissionName()}: $resource 접근 허용")
    } else {
        println("${permission.getPermissionName()}: $resource 접근 거부")
    }
}

checkAccess(adminRole, "write:users")  // ADMIN: write:users 접근 허용
checkAccess(userRole, "write:users")   // USER: write:users 접근 거부
```

### Before/After 비교

```kotlin
// ❌ Before: 타입별 분기 처리
fun printStructure(item: Any, indent: String = "") {
    when (item) {
        is File -> println("$indent${item.name}")
        is Folder -> {
            println("$indent${item.name}/")
            item.children.forEach { child ->
                printStructure(child, "$indent  ")
            }
        }
    }
}

// ✅ After: Composite 패턴 - 동일한 인터페이스로 처리
interface FileSystemItem {
    fun print(indent: String = "")
}

class File(private val name: String) : FileSystemItem {
    override fun print(indent: String) {
        println("$indent$name")
    }
}

class Folder(private val name: String) : FileSystemItem {
    private val children = mutableListOf<FileSystemItem>()
    
    override fun print(indent: String) {
        println("$indent$name/")
        children.forEach { it.print("$indent  ") }
    }
}

// 클라이언트
fun printStructure(item: FileSystemItem) {
    item.print()  // 타입 구분 없이 동일하게 호출
}
```

---

## 유사 패턴 비교

| 패턴 | 차이점 |
|------|--------|
| **Decorator** | 객체에 동적으로 기능 추가, 1:1 감싸기 구조 |
| **Composite** | 트리 구조로 **부분-전체** 관계 표현, 1:N 포함 구조 |
| **Chain of Responsibility** | 요청을 체인으로 전달, Composite는 트리 구조로 작업 위임 |

### Decorator vs Composite

| 비교 항목 | Decorator | Composite |
|----------|-----------|-----------|
| 목적 | 기능 확장 | 트리 구조 표현 |
| 구조 | 1:1 래핑 | 1:N 포함 |
| 관계 | 선형 체인 | 트리 계층 |
| 사용 예 | Java I/O Stream | 파일 시스템 |

---

## 참고 자료

### 도서
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - 메뉴 시스템 예제

### 추천 사이트
- [Refactoring Guru - Composite Pattern](https://refactoring.guru/design-patterns/composite)

### 관련 TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.md)
- [Strategy Pattern](../Behavioral/Strategy%20Pattern.md)
