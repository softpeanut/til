# Design Pattern

## Concept

A Design Pattern is **a reusable solution to commonly occurring problems in software design**.

In 1994, the Gang of Four (GoF) systematized 23 patterns in their book "Design Patterns: Elements of Reusable Object-Oriented Software", which became widely known. Design patterns are not specific code but **design-level templates** that can be adapted to different situations.

---

## Why Needed

### Problem to Solve

Recurring problems in object-oriented design:
- Complexity in object creation methods
- Difficulty in making changes due to tight coupling between classes
- Need to modify existing code when adding new features

### Value Provided

| Value | Description |
|-------|-------------|
| **Reusability** | Proven designs can be applied repeatedly |
| **Maintainability** | Flexible to changes and easy to extend |
| **Communication** | Common vocabulary for conveying design intent between developers |
| **Quality Improvement** | Apply time-tested best practices |

---

## Types

GoF Design Patterns are classified into **Creational**, **Structural**, and **Behavioral** based on their purpose.

### Creational Patterns

Abstract the object creation process to **hide creation complexity and ensure flexibility**.

| Pattern | Description | Example Use |
|---------|-------------|-------------|
| **Singleton** | Create only one instance of a class and provide global access | Configuration management, Connection pool |
| **Factory Method** | Delegate object creation to subclasses to determine object type | Various document types in document editor |
| **Abstract Factory** | Provide interface for creating families of related objects | UI components per theme |
| **Builder** | Separate complex object construction into steps | Assembling objects with various options |
| **Prototype** | Create new objects by copying existing ones | When object cloning is frequent |

### Structural Patterns

Compose classes or objects to **form larger structures and ensure extensibility**.

| Pattern | Description | Example Use |
|---------|-------------|-------------|
| **Adapter** | Convert incompatible interfaces to work together | Legacy system integration |
| **Bridge** | Separate abstraction and implementation for independent extension | Platform-independent UI |
| **Composite** | Compose objects into tree structures to treat individual and composite objects uniformly | File system, UI components |
| **Decorator** | Dynamically add responsibilities (features) to objects | Java I/O streams |
| **Facade** | Provide simplified interface to complex subsystems | Library wrappers |
| **Flyweight** | Efficiently support many similar objects through sharing | Character rendering, Game objects |
| **Proxy** | Provide surrogate object to control access to another object | Lazy loading, Access control, Caching |

### Behavioral Patterns

Enable flexible interactions through **responsibility distribution and algorithm encapsulation** between objects.

| Pattern | Description | Example Use |
|---------|-------------|-------------|
| **Strategy** | Define algorithm family and make them interchangeable at runtime | Sorting algorithms, Payment methods |
| **Template Method** | Define algorithm skeleton and let subclasses implement certain steps | Framework hook methods |
| **Observer** | Automatically notify dependent objects when object state changes | Event systems, Subscription models |
| **State** | Change behavior based on object's internal state | Order status, Game character states |
| **Command** | Encapsulate requests as objects for parameterization, queuing, logging | Undo/Redo, Macros |
| **Chain of Responsibility** | Compose chain of objects that can handle requests | Event bubbling, Middleware |
| **Mediator** | Encapsulate object interactions through mediator object | Chat rooms, GUI component coordination |
| **Memento** | Save object state and restore to previous state | Undo functionality |
| **Visitor** | Add new operations to object structure without modifying objects | Compiler AST processing |
| **Iterator** | Provide way to sequentially access collection elements | Java Iterator, for-each |
| **Interpreter** | Define and interpret language grammar | SQL parser, Regular expressions |

---

## Pitfalls

### 1. Avoid Over-application

Apply patterns **when there's a problem**. Applying complex patterns to simple problems makes code more complicated.

```
// ❌ Factory pattern for simple object creation
UserFactory.createUser("kim")

// ✅ Direct creation is sufficient
User("kim")
```

### 2. Patterns Are Tools, Not Goals

Don't write code "to apply a pattern". The problem to solve comes first, then choose the appropriate pattern.

### 3. Context Understanding Required

The same pattern may be implemented differently depending on language and framework. Understand the pattern's **intent** and apply it according to the situation.

---

## References

### Books
- [Design Patterns: Elements of Reusable Object-Oriented Software](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612) - GoF Original

### Recommended Sites
- [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns) - Detailed explanations and examples in various languages
