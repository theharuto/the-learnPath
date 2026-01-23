## SOLID (O · L · I · D)
---

## O — Open / Closed Principle (OCP)

> Software entities should be **open for extension** but **closed for modification**.

This is **the most abused principle** in interviews and codebases.

---

### What juniors misunderstand

* ❌ “Never modify existing code”
* ❌ “Use inheritance everywhere”
* ❌ “Always add abstractions early”

All wrong.

---

### Senior definition (important)

> You should be able to add **new behavior**
> **without touching stable, tested code paths**.

OCP is about **protecting volatility boundaries**, not freezing code.

---

### Real-world failure mode

In production systems:

* Frequently modified code = bug factory
* Stable code = trust anchor

OCP minimizes **blast radius** of change.

---

### Bad OCP Example (Very Common)

```java
public class DiscountCalculator {

    public double calculate(Order order) {
        if (order.getType() == OrderType.REGULAR) {
            return order.total();
        }
        if (order.getType() == OrderType.PREMIUM) {
            return order.total() * 0.9;
        }
        if (order.getType() == OrderType.VIP) {
            return order.total() * 0.8;
        }
        return order.total();
    }
}
```

### Why this violates OCP

* Every new order type → modify this class
* Risk increases with each condition
* Tests become fragile

---

### OCP-Compliant Design

```java
public interface DiscountPolicy {
    double apply(Order order);
}
```

```java
public class RegularDiscount implements DiscountPolicy {
    public double apply(Order order) {
        return order.total();
    }
}
```

```java
public class PremiumDiscount implements DiscountPolicy {
    public double apply(Order order) {
        return order.total() * 0.9;
    }
}
```

```java
public class DiscountService {
    private final DiscountPolicy policy;

    public DiscountService(DiscountPolicy policy) {
        this.policy = policy;
    }

    public double calculate(Order order) {
        return policy.apply(order);
    }
}
```

Now:

* New discount → new class
* No modification to stable logic
* Behavior extended safely

---

### Anime analogy — *Code Geass*

Lelouch doesn’t rewrite the world.
He **adds new moves without exposing himself**.

OCP is about **strategic extension**, not rigidity.

---

### Senior OCP Warning (Critical)

> If you apply OCP too early, you create **abstraction hell**.

**Rule of thumb:**

* Apply OCP only to **hotspots** (frequently changing behavior)
* Ignore it for stable logic

---

## L — Liskov Substitution Principle (LSP)

> Subtypes must be substitutable for their base types.

This is **the most violated principle in real systems**.

---

### What LSP really means

If `S` is a subtype of `T`, then:

* Any code using `T`
* Should work correctly when given `S`
* **Without knowing it**

If behavior changes → LSP violation.

---

### Classic Broken Example

```java
class Bird {
    void fly() {}
}

class Penguin extends Bird {
    @Override
    void fly() {
        throw new UnsupportedOperationException();
    }
}
```

This **compiles**.
This is **broken design**.

---

### Why this is dangerous

* Client code trusts `Bird`
* Substitution breaks behavior
* Runtime failures appear unexpectedly

---

### Correct LSP Design

```java
interface Bird {}

interface FlyingBird extends Bird {
    void fly();
}

class Sparrow implements FlyingBird {
    public void fly() {}
}

class Penguin implements Bird {
    void swim() {}
}
```

Now:

* No false promises
* Behavior preserved
* Substitution safe

---

### Senior LSP Insight

LSP violations often hide behind:

* `instanceof`
* overridden methods that weaken behavior
* throwing exceptions in subclasses

If you need conditionals based on subtype → **design is wrong**.

---

### Anime analogy — *One Punch Man*

Saitama doesn’t pretend to fight strategically.
He is honest about his capabilities.

Subtypes must **honor contracts**, not fake them.

---

## I — Interface Segregation Principle (ISP)

> Clients should not be forced to depend on methods they do not use.

---

### Common junior mistake

```java
public interface Worker {
    void work();
    void eat();
    void attendMeeting();
}
```

```java
public class RobotWorker implements Worker {
    public void eat() {
        throw new UnsupportedOperationException();
    }
}
```

This is **ISP violation**.

---

### Why this hurts systems

* Unused methods = fake contracts
* Forces meaningless implementations
* Encourages runtime failures

---

### ISP-Compliant Design

```java
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface MeetingParticipant {
    void attendMeeting();
}
```

Classes implement **only what they need**.

---

### Senior ISP Rule

> Interfaces are **promises**, not convenience groupings.

If a client does not use a method, it should not know it exists.

---

### Anime analogy — *Spy x Family*

Anya doesn’t do adult missions.
Forcing her to breaks everything.

Interfaces should fit the actor.

---

## D — Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules.
> Both should depend on abstractions.

This is **where architecture begins**.

---

### Junior misunderstanding

* ❌ “Use interfaces everywhere”
* ❌ “Dependency Injection = DIP”

No.

---

### Senior definition

> Business rules must not change
> because infrastructure changes.

---

### Bad DIP Example

```java
public class OrderService {
    private final MySqlOrderRepository repository = new MySqlOrderRepository();
}
```

Now:

* DB change → service change
* Tests become hard
* System becomes rigid

---

### DIP-Compliant Design

```java
public interface OrderRepository {
    void save(Order order);
}
```

```java
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

Now:

* Business logic independent
* Infrastructure swappable
* Tests easy

---

### Anime analogy — *Code Geass*

Lelouch controls **strategy**, not soldiers’ uniforms.

High-level policy must stay pure.

---

## SOLID — Senior Synthesis

| Principle | Protects Against       |
| --------- | ---------------------- |
| SRP       | Change collision       |
| OCP       | Regression explosion   |
| LSP       | Runtime surprises      |
| ISP       | Fake contracts         |
| DIP       | Architectural rigidity |

SOLID is about **survivability**, not perfection.
