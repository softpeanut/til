# Composite Pattern

## Concept

The Composite Pattern **composes objects into tree structures to treat individual objects and compositions uniformly**.

Clients can handle single objects (Leaf) and composite objects (Composite) without distinction using the same interface. As one of the GoF structural patterns, the key is "**representing part-whole hierarchies**".

### Core Principles

| Principle | Description |
|-----------|-------------|
| **Single Responsibility** | Both individual and composite objects implement the same interface |
| **Transparency** | Client doesn't need to distinguish between Leaf and Composite |
| **Recursive Composition** | Composite can have Leaf or other Composite as children |
| **Uniform Treatment** | All nodes in tree structure are treated the same way |

---

## Why Needed

### Problem to Solve

```kotlin
// ❌ Different handling per type - Complex and hard to extend
fun calculatePrice(item: Any): Int {
    return when (item) {
        is Product -> item.price
        is Box -> {
            var total = 0
            for (child in item.items) {
                total += when (child) {
                    is Product -> child.price
                    is Box -> calculatePrice(child)  // Recursive call needed
                    else -> 0
                }
            }
            total
        }
        else -> 0
    }
}
```

- Type check logic scattered everywhere
- All conditionals need modification when adding new types
- Code complexity increases with deeper nesting

### Value Provided

- **Simplification**: Handle complex tree structures with simple interface
- **Extensibility**: Easy to add new Leaf/Composite types
- **Consistency**: Client code is simple and consistent
- **Recursive Processing**: Tree traversal logic is naturally implemented

---

## How It Works

### Structure

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

| Component | Role |
|-----------|------|
| **Component** | Defines common interface for Leaf and Composite |
| **Leaf** | Terminal node of tree, has no children |
| **Composite** | Contains and manages child Components, delegates work to children |

### Basic Implementation

```kotlin
// Component interface
interface FileSystemItem {
    fun getSize(): Long
    fun getName(): String
}

// Leaf - File
class File(
    private val name: String,
    private val size: Long
) : FileSystemItem {
    
    override fun getSize(): Long = size
    
    override fun getName(): String = name
}

// Composite - Folder
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
        // Recursively sum all children's sizes
        return children.sumOf { it.getSize() }
    }
    
    override fun getName(): String = name
}

// Usage
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

// Client treats Leaf and Composite uniformly
println(root.getSize())       // 11172 (sum of all file sizes)
println(documents.getSize())  // 3072
println(pictures.getSize())   // 8000
```

### Tree Structure Example

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

## Pitfalls

### 1. Type Safety vs Transparency Tradeoff

Including add/remove in Component increases transparency, but Leaf must implement meaningless methods.

```kotlin
// Transparency first - Include add/remove in Component
interface Component {
    fun operation()
    fun add(component: Component)    // Meaningless in Leaf
    fun remove(component: Component) // Meaningless in Leaf
}

class Leaf : Component {
    override fun operation() { /* ... */ }
    override fun add(component: Component) {
        throw UnsupportedOperationException()  // Or ignore
    }
    override fun remove(component: Component) {
        throw UnsupportedOperationException()
    }
}

// Type safety first - Define add/remove only in Composite
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

### 2. Watch for Circular References

Infinite loops occur if Composite adds itself as a child

```kotlin
// ❌ Circular reference risk
val folder = Folder("test")
folder.add(folder)  // Infinite loop!

// ✅ Prevent circular references
class Folder(private val name: String) : FileSystemItem {
    private val children = mutableListOf<FileSystemItem>()
    
    fun add(item: FileSystemItem) {
        if (item === this) throw IllegalArgumentException("Cannot add self")
        if (containsAncestor(item)) throw IllegalArgumentException("Circular reference")
        children.add(item)
    }
}
```

### 3. Performance Considerations

Deep tree structures with many recursive calls can cause performance degradation. Consider caching if needed.

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

## Practical Application

### Application Scenarios

| Situation | Example |
|-----------|---------|
| Tree/hierarchy representation | File system, Organization chart |
| UI component composition | View hierarchy, Widget tree |
| Menu structure | Menu > Submenu > Menu item |
| Expression representation | Tree structure of operators and operands |

### Order System Example

```kotlin
// Component
interface OrderItem {
    fun getPrice(): Int
    fun getDescription(): String
}

// Leaf - Single product
class Product(
    private val name: String,
    private val price: Int
) : OrderItem {
    
    override fun getPrice(): Int = price
    
    override fun getDescription(): String = "$name: ${price}won"
}

// Composite - Product bundle (combo meal, package, etc.)
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
        return "$name (${(discountRate * 100).toInt()}% discount):\n  $itemDescriptions"
    }
}

// Usage
val burger = Product("Burger", 5000)
val fries = Product("Fries", 2000)
val drink = Product("Cola", 1500)

// Combo meal (10% discount)
val comboMeal = ProductBundle("Burger Combo", 0.1)
comboMeal.add(burger)
comboMeal.add(fries)
comboMeal.add(drink)

// Order includes both single items and combos
val order = ProductBundle("Order")
order.add(comboMeal)
order.add(Product("Ice Cream", 1000))

println(order.getDescription())
println("Total price: ${order.getPrice()}won")

// Output:
// Order (0% discount):
//   Burger Combo (10% discount):
//     Burger: 5000won
//     Fries: 2000won
//     Cola: 1500won
//   Ice Cream: 1000won
// Total price: 8650won
```

### Spring Environment: Permission Hierarchy

```kotlin
// Component - Permission
interface Permission {
    fun hasAccess(resource: String): Boolean
    fun getPermissionName(): String
}

// Leaf - Single permission
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

// Composite - Permission group (Role)
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

// Usage
val adminRole = Role("ADMIN")
adminRole.addPermission(ReadPermission())
adminRole.addPermission(WritePermission())

val userRole = Role("USER")
userRole.addPermission(ReadPermission())

// Client treats single permission and role uniformly
fun checkAccess(permission: Permission, resource: String) {
    if (permission.hasAccess(resource)) {
        println("${permission.getPermissionName()}: $resource access allowed")
    } else {
        println("${permission.getPermissionName()}: $resource access denied")
    }
}

checkAccess(adminRole, "write:users")  // ADMIN: write:users access allowed
checkAccess(userRole, "write:users")   // USER: write:users access denied
```

### Before/After Comparison

```kotlin
// ❌ Before: Type-based branching
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

// ✅ After: Composite Pattern - Uniform interface handling
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

// Client
fun printStructure(item: FileSystemItem) {
    item.print()  // Same call without type distinction
}
```

---

## Similar Patterns Comparison

| Pattern | Difference |
|---------|------------|
| **Decorator** | Dynamically adds features to objects, 1:1 wrapping structure |
| **Composite** | Represents **part-whole** relationships in tree structure, 1:N containment |
| **Chain of Responsibility** | Passes requests through chain, Composite delegates work through tree structure |

### Decorator vs Composite

| Comparison | Decorator | Composite |
|------------|-----------|-----------|
| Purpose | Feature extension | Tree structure representation |
| Structure | 1:1 wrapping | 1:N containment |
| Relationship | Linear chain | Tree hierarchy |
| Use Case | Java I/O Stream | File system |

---

## References

### Books
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124) - Menu system example

### Recommended Sites
- [Refactoring Guru - Composite Pattern](https://refactoring.guru/design-patterns/composite)

### Related TIL
- [What is Design Pattern](../What%20is%20Design%20Pattern.en.md)
- [Strategy Pattern](../Behavioral/Strategy%20Pattern.en.md)
