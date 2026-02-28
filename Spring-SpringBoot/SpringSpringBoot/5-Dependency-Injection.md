1. What DI really means
2. Constructor vs Setter injection (real comparison)
3. @Autowired — how it works conceptually
4. @Qualifier — resolving conflicts

---

# 1️⃣ What is Dependency Injection (DI)?

Dependency Injection means:

> An object receives its dependencies from an external source instead of creating them itself.

Without DI:

```java
class OrderService {
    private PaymentService paymentService = new PaymentService();
}
```

Problems:

* Tight coupling
* Hard to test
* Hard to replace implementation
* No flexibility

With DI:

```java
class OrderService {

    private final PaymentService paymentService;

    OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Now:

* OrderService doesn’t decide which PaymentService to use.
* Spring injects it.
* You can replace it easily.

This enables:

* Loose coupling
* Testability
* Clean architecture

---

# 2️⃣ Constructor vs Setter Injection

Let’s compare properly.

---

## ✅ Constructor Injection (Recommended Default)

```java
@Component
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring:

* Sees constructor
* Resolves PaymentService bean
* Injects it during creation

### Why This Is Preferred

1. Enforces required dependencies
2. Object is fully initialized
3. Supports immutability (`final`)
4. Easier to test (no reflection)
5. Thread-safe when stateless

Senior-level rule:

> If dependency is mandatory → use constructor injection.

---

## ⚠️ Setter Injection

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

### When It Makes Sense

* Optional dependency
* Rare configuration adjustments

### Risks

* Object may exist in partially initialized state
* Harder to reason about invariants
* No immutability

---

## Clear Comparison

| Feature             | Constructor | Setter |
| ------------------- | ----------- | ------ |
| Required dependency | ✅           | ⚠️     |
| Immutability        | ✅           | ❌      |
| Null safety         | ✅           | ❌      |
| Optional dependency | ❌           | ✅      |
| Recommended default | ✅           | ❌      |

---

# 3️⃣ Autowiring with @Autowired

## What Does @Autowired Do?

It tells Spring:

> Inject a matching bean here automatically.

Spring resolves dependencies:

* By type (default)
* By name (if necessary)
* Using qualifiers (if conflicts exist)

---

## Constructor Autowiring

```java
@Component
class OrderService {

    private final PaymentService paymentService;

    @Autowired
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

In modern Spring:

* If there is only one constructor, `@Autowired` is optional.

---

## Setter Autowiring

```java
@Autowired
public void setPaymentService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

---

## Field Autowiring (Not Recommended)

```java
@Autowired
private PaymentService paymentService;
```

Why avoid:

* Uses reflection
* Harder to test
* Hides dependencies
* Breaks immutability

Professional codebases prefer constructor injection.

---

# 4️⃣ What Happens Internally (High-Level)

When Spring creates a bean:

1. Looks at constructor
2. Identifies required parameters
3. Searches container for matching bean type
4. Injects resolved dependency

This is called:

> Dependency resolution

---

# 5️⃣ What If Multiple Beans Exist?

This is where things get interesting.

Example:

```java
@Component
class PaypalPaymentService implements PaymentService {}
```

```java
@Component
class StripePaymentService implements PaymentService {}
```

Now:

```java
@Component
class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring will throw:

```
NoUniqueBeanDefinitionException
```

Because:

* Two beans match the type PaymentService.

Spring cannot guess.

---

# 6️⃣ Resolving Conflicts with @Qualifier

## Using @Qualifier

```java
@Component
class OrderService {

    private final PaymentService paymentService;

    public OrderService(
        @Qualifier("paypalPaymentService") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring now:

* Matches by bean name
* Injects correct implementation

---

## Custom Qualifier Names

```java
@Component
@Qualifier("paypal")
class PaypalPaymentService implements PaymentService {}
```

Then:

```java
public OrderService(@Qualifier("paypal") PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

Cleaner and more explicit.

---

# 7️⃣ Alternative: @Primary

Another way to resolve conflicts:

```java
@Component
@Primary
class PaypalPaymentService implements PaymentService {}
```

Now:

* If multiple beans exist
* Spring chooses the one marked @Primary

Use:

* When one implementation is default

Use @Qualifier:

* When you want explicit control

---

# 8️⃣ Practical Decision Rules

If:

* One implementation → no qualifier needed
* Multiple implementations → use @Qualifier
* One default among many → use @Primary

---

# 9️⃣ Common DI Mistakes

❌ Using field injection everywhere
❌ Forgetting qualifier when multiple beans exist
❌ Putting state inside singleton beans
❌ Thinking DI is magic — it’s just type resolution

---

# 🔟 Mental Model

Think of Spring DI as:

```text
Bean A requires Bean B
   ↓
Container looks for matching type
   ↓
If one match → inject
If multiple → need qualifier or primary
If none → error
```

It’s systematic, not magical.

---

# 📘 Revision Notes — Topic 5

---

## 📌 Dependency Injection (DI)

### What is DI?

Dependencies are provided externally instead of being created inside the class.

Improves:

* Loose coupling
* Testability
* Maintainability

---

### Constructor Injection

* Dependencies passed via constructor
* Recommended default
* Ensures immutability
* Guarantees fully initialized object

---

### Setter Injection

* Dependencies injected via setter method
* Used for optional dependencies
* Allows partial initialization

---

### @Autowired

* Enables automatic dependency resolution
* Injection by type
* Optional on single constructor

---

### Resolving Multiple Beans

If multiple implementations exist:

* Use @Qualifier for explicit selection
* Use @Primary for default selection

---

### Best Practices

* Prefer constructor injection
* Avoid field injection
* Keep singleton beans stateless
* Use qualifiers when multiple beans exist

---

Next topic options:

6️⃣ Spring Profiles
7️⃣ Introduction to Spring Boot
8️⃣ Small hands-on example tying everything together
9️⃣ Deep dive into DI edge cases (optional dependencies, circular deps)

Choose your direction.
