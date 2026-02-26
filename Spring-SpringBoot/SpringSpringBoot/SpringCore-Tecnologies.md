# 1️⃣ Bean

## What is a Bean?

A **Bean** is:

> An object whose lifecycle is managed by the Spring container.

That’s it.

It is not a special JVM object.
It is not a special keyword.
It is just a normal Java object managed by Spring.

Example:

```java
@Component
class UserService {}
```

When Spring starts:

* It creates the object
* It stores it
* It injects dependencies
* It manages its lifecycle

That object is now a **Spring Bean**.

---

## Important Clarification

❌ Bean ≠ Singleton (by definition)

Default scope is singleton, but beans can be:

* Singleton
* Prototype
* Request-scoped
* Session-scoped

So:

> Bean = Managed Object
> Scope defines instance count

---

# 2️⃣ BeanFactory

## What is BeanFactory?

BeanFactory is:

> The core container responsible for creating and managing beans.

It is the lowest-level IoC container in Spring.

Main responsibilities:

* Instantiating beans
* Injecting dependencies
* Managing lifecycle
* Handling scopes

Think of it as:

> The engine that builds objects.

---

## Important Characteristics

* Lazy initialization by default
* Basic dependency injection
* No advanced enterprise features

Rarely used directly in modern apps.

---

# 3️⃣ ApplicationContext

## What is ApplicationContext?

ApplicationContext is:

> An advanced container built on top of BeanFactory.

It extends BeanFactory and adds enterprise features.

Think of it as:

> BeanFactory + enterprise capabilities

---

## What Extra Does It Provide?

* Eager singleton initialization
* Event publishing
* AOP integration
* Internationalization
* Resource loading
* Environment profiles
* Annotation scanning

In real applications:

> You always use ApplicationContext.

Spring Boot always creates an ApplicationContext.

---

## Clear Comparison

| Feature          | BeanFactory | ApplicationContext   |
| ---------------- | ----------- | -------------------- |
| Basic DI         | ✅           | ✅                    |
| Lazy by default  | ✅           | ❌ (eager singletons) |
| Event system     | ❌           | ✅                    |
| AOP support      | ❌           | ✅                    |
| Enterprise ready | ❌           | ✅                    |

---

# 4️⃣ Dependency Injection (DI)

This is the most important concept.

## What is DI?

Dependency Injection means:

> Dependencies are provided to an object instead of the object creating them.

Without DI:

```java
class OrderService {
    private PaymentService paymentService = new PaymentService();
}
```

Tightly coupled.

With DI:

```java
class OrderService {

    private final PaymentService paymentService;

    OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Now the dependency is injected.

Spring provides the dependency.

---

# 5️⃣ Constructor-Based DI

## Example

```java
@Component
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring sees:

* Constructor requires PaymentService
* Finds bean of that type
* Injects it

---

## Why Constructor Injection Is Preferred

1. Enforces immutability (`final`)
2. Ensures object is fully initialized
3. Prevents null dependencies
4. Easier to test
5. Clear dependency contract

Senior rule:

> If a dependency is required → use constructor injection.

---

# 6️⃣ Setter-Based DI

## Example

```java
@Component
class OrderService {

    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring:

* Creates object first
* Calls setter
* Injects dependency

---

## When Setter Injection Makes Sense

* Optional dependencies
* Rare reconfiguration needs

---

## Risks of Setter Injection

* Object can exist in incomplete state
* Harder to reason about immutability
* Potential runtime errors if not injected

---

# 7️⃣ Constructor vs Setter — Practical Comparison

| Aspect              | Constructor DI | Setter DI |
| ------------------- | -------------- | --------- |
| Required dependency | ✅              | ⚠️        |
| Immutability        | ✅              | ❌         |
| Null safety         | ✅              | ❌         |
| Optional dependency | ⚠️             | ✅         |
| Recommended default | ✅              | ❌         |

---

# 8️⃣ What About Field Injection?

```java
@Autowired
private PaymentService paymentService;
```

You will see this a lot.

But:

* Uses reflection
* Breaks immutability
* Harder to test
* Hides dependencies

In professional codebases:

> Constructor injection is the standard.

---

# 9️⃣ Mental Model

Spring DI flow:

```
1. Find BeanDefinition
2. Choose constructor
3. Resolve dependencies by type
4. Instantiate object
5. Inject dependencies
```

Everything revolves around dependency graph resolution.

---

# 🔟 Common Beginner Confusions

❌ BeanFactory and ApplicationContext are same
❌ Singleton means only one instance in JVM
❌ DI and IoC are same (DI is implementation of IoC)
❌ Field injection is best because less code

Correct mental model:

* BeanFactory = basic container
* ApplicationContext = production container
* Constructor DI = safe default

---

# 📘 Revision Notes — Topic 2

---

## 📌 Spring Core Terminologies

### Bean

A Bean is an object managed by the Spring container.

It is created, wired, and maintained by Spring.

Default scope: Singleton (per ApplicationContext).

---

### BeanFactory

* Core IoC container
* Responsible for instantiating and managing beans
* Lazy initialization by default
* Minimal features

---

### ApplicationContext

* Advanced container built on BeanFactory
* Adds AOP, events, profiles, resource loading
* Eager singleton initialization
* Used in all real applications

---

### Dependency Injection (DI)

Dependencies are supplied externally instead of being created internally.

Improves:

* Loose coupling
* Testability
* Maintainability

---

### Constructor-Based DI

* Dependencies passed through constructor
* Preferred approach
* Ensures immutability
* Prevents incomplete objects

---

### Setter-Based DI

* Dependencies injected via setters
* Used for optional dependencies
* Allows partially initialized objects

---

### Best Practice

Use constructor injection for required dependencies.
