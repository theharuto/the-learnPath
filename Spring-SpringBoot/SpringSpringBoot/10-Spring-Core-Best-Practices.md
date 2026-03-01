# 10️⃣ Spring Core Best Practices

---

# 1️⃣ Designing Loosely Coupled Architectures

Loose coupling is not about using `@Autowired`.
It’s about **architectural boundaries**.

---

## ✅ 1. Depend on Interfaces, Not Implementations

Bad:

```java
@Service
class OrderService {

    private final StripePaymentService stripePaymentService;

    public OrderService(StripePaymentService stripePaymentService) {
        this.stripePaymentService = stripePaymentService;
    }
}
```

Good:

```java
@Service
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Why?

* You can swap implementation
* You can mock easily
* Profiles work cleanly

---

## ✅ 2. Constructor Injection Only (Default Rule)

Why:

* Makes dependencies explicit
* Prevents partially constructed objects
* Encourages immutability
* Simplifies testing

Avoid:

* Field injection
* Hidden dependencies

---

## ✅ 3. Keep Layers Clean

Classic layered structure:

```text
Controller → Service → Repository
```

Rules:

* Controller should not call repository directly
* Repository should not contain business logic
* Service contains business rules

Spring helps enforce this using stereotypes:

* @Controller
* @Service
* @Repository

But architecture discipline is your responsibility.

---

## ✅ 4. Keep Beans Stateless

Singleton is default scope.

Singleton + mutable state = concurrency bugs.

Bad:

```java
@Service
class CounterService {
    private int counter;
}
```

In web apps → multiple threads → race conditions.

Keep services stateless.

---

# 2️⃣ Writing Testable & Maintainable Code

Spring should make testing easier, not harder.

---

## ✅ 1. Constructor Injection Enables Unit Testing

Example:

```java
class OrderService {

    private final PaymentService paymentService;

    OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Unit test:

```java
PaymentService mock = Mockito.mock(PaymentService.class);
OrderService service = new OrderService(mock);
```

No Spring context required.

This is clean architecture.

---

## ✅ 2. Do Not Put Logic in Configuration

Bad:

```java
@Bean
public PaymentService paymentService() {
    if (something) {
        return new A();
    }
    return new B();
}
```

Business decisions do not belong in configuration.

Use:

* Profiles
* Conditional configuration

---

## ✅ 3. Avoid Overusing @Value

Bad:

```java
@Value("${api.key}")
private String apiKey;
```

Better:

* Use constructor injection for config properties
* Group properties logically
* Avoid scattering property access

---

## ✅ 4. Avoid Hard Container Coupling

Do not do this:

```java
ApplicationContext context = ...
context.getBean(...)
```

Except in bootstrap code.

Beans should not manually fetch other beans.

That defeats DI.

---

# 3️⃣ Avoiding Common Pitfalls

Now the dangerous part.

---

## ❌ Circular Dependencies

Example:

```text
A depends on B
B depends on A
```

Constructor injection → startup failure.

Why this happens:

* Poor responsibility separation
* Bidirectional logic

Fix:

* Extract shared dependency
* Redesign boundaries
* Introduce mediator/service layer

Never “fix” with field injection.

That hides bad design.

---

## ❌ Excessive Dependencies

If your constructor looks like:

```java
public OrderService(A a, B b, C c, D d, E e, F f)
```

You have a design smell.

Rules of thumb:

* > 4 dependencies → re-evaluate responsibility
* Large dependency graph → high coupling
* Hard to test → design problem

Split responsibilities.

---

## ❌ Overusing @Lazy to Fix Design Problems

Using `@Lazy` to solve circular dependencies is a red flag.

It hides architectural issues.

Use lazy only for:

* Expensive resources
* Rarely used components

Not for broken design.

---

## ❌ Putting State in Singleton Beans

This causes:

* Thread-safety issues
* Race conditions
* Data corruption

If state is required:

* Use prototype scope
* Or manage state outside singleton service

---

## ❌ Business Logic in Controllers

Controller should:

* Validate input
* Call service
* Return response

Not contain domain logic.

---

# 4️⃣ Dependency Management Principles

---

## 1. Prefer Explicit Wiring Over Magic

Use:

* Constructor injection
* Clear interfaces
* Qualifiers when necessary

Avoid:

* Ambiguous beans
* Over-reliance on autowiring without understanding

---

## 2. Keep Configuration Modular

Split:

* DataConfig
* WebConfig
* SecurityConfig

Instead of one giant AppConfig.

---

## 3. Fail Fast

Let application fail at startup if configuration is wrong.

Do not defer errors to runtime.

Spring’s eager singleton behavior helps here.

---

# 5️⃣ Senior-Level Rules of Thumb

Memorize these.

---

### Rule 1:

Spring manages infrastructure.
You manage architecture.

---

### Rule 2:

Constructor injection is default.
Setter injection is exceptional.

---

### Rule 3:

If you need @Lazy to solve design — redesign.

---

### Rule 4:

Services should be stateless.

---

### Rule 5:

Profiles control environment configuration, not business logic.

---

### Rule 6:

If a class has too many dependencies, it has too many responsibilities.

---

# 6️⃣ Mental Model of Good Spring Code

```text
Interfaces define contracts
Services implement business rules
Repositories handle persistence
Configuration wires infrastructure
Container manages lifecycle
```

If your code respects this separation, Spring stays simple.

---

# 📘 Revision Notes — Topic 10

---

## 📌 Spring Core Best Practices

### Loose Coupling

* Depend on interfaces
* Use constructor injection
* Keep layers clean
* Keep beans stateless

---

### Testability

* Constructor injection enables easy unit testing
* Avoid container access inside beans
* Separate business logic from configuration

---

### Avoid Pitfalls

* Avoid circular dependencies
* Avoid excessive constructor dependencies
* Do not hide bad design with @Lazy
* Avoid mutable state in singleton beans

---

### Architecture Principles

* Clear separation of concerns
* Fail fast on misconfiguration
* Keep configuration modular
* Use profiles for environment differences only
