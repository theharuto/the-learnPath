# Dependency Inversion Principle (DIP)

## How a Senior Actually Thinks About It

> **DIP:** High-level policy should not depend on low-level details.
> Both should depend on abstractions.

Forget this sentence for now.
We’ll *derive* it.

---

## Step 1: Start With a Real Requirement (Business Language)

**Requirement (realistic):**

> “When a user places an order, we must charge the payment and store the order.”

That’s it.
No databases. No frameworks. No Stripe. No MySQL.

**Senior mental note:**
👉 This is **business policy**, not infrastructure.

---

## Step 2: Identify What Is *Core* vs *Replaceable*

Now ask **two critical architect questions**:

1. What **must exist for the business to work**?
2. What **might change** without changing business meaning?

### Core (stable)

* Order placement
* Payment success/failure

### Replaceable (volatile)

* Payment provider (Stripe, Razorpay, PayPal)
* Storage (MySQL, MongoDB, S3)

This distinction is **where DIP starts**.

---

## Step 3: The Naive (Wrong) Implementation

This is how most engineers write it initially:

```java
public class OrderService {

    public void placeOrder(Order order) {
        StripePaymentGateway gateway = new StripePaymentGateway();
        gateway.charge(order.getAmount());

        MySqlOrderRepository repository = new MySqlOrderRepository();
        repository.save(order);
    }
}
```

### Why this *feels* okay

* Code is readable
* Works in production
* Simple

### Why this is **architecturally broken**

Ask yourself:

* What happens if payment provider changes?
* What happens if database changes?
* Can I test this without Stripe or MySQL?

**Answer:** You modify `OrderService`.

That’s the violation.

---

## Step 4: Identify the High-Level Policy

Read this method **without thinking about Java**:

```java
placeOrder(order):
    take payment
    persist order
```

That is the **policy**.

**Senior insight:**

> Policy describes *what must happen*, not *how* it happens.

So ask:

> Does the business care *how* payment is charged?

No.

> Does the business care *where* the order is stored?

No.

That means:

* Stripe
* MySQL

are **details**, not policy.

---

## Step 5: The Mental Flip (The “Inversion”)

### Junior thinking

> OrderService uses Stripe

### Senior thinking

> Stripe **serves** OrderService

That mental flip **is Dependency Inversion**.

---

## Step 6: Create the Abstractions (Derived, Not Invented)

We now extract **what the business needs**, not what tech provides.

### Payment — from business POV

```java
public interface PaymentProcessor {
    void charge(Money amount);
}
```

### Storage — from business POV

```java
public interface OrderRepository {
    void save(Order order);
}
```

Notice:

* No Stripe
* No MySQL
* Pure business language

---

## Step 7: High-Level Code After DIP

```java
public class OrderService {

    private final PaymentProcessor paymentProcessor;
    private final OrderRepository orderRepository;

    public OrderService(PaymentProcessor paymentProcessor,
                        OrderRepository orderRepository) {
        this.paymentProcessor = paymentProcessor;
        this.orderRepository = orderRepository;
    }

    public void placeOrder(Order order) {
        paymentProcessor.charge(order.getAmount());
        orderRepository.save(order);
    }
}
```

### What changed?

* OrderService **no longer depends on details**
* It depends on **what it needs**, not how it’s done
* Business logic is now **stable**

This is DIP **in practice**.

---

## Step 8: Low-Level Details Implement the Policy

```java
public class StripePaymentProcessor implements PaymentProcessor {
    public void charge(Money amount) {
        // Stripe API
    }
}
```

```java
public class MySqlOrderRepository implements OrderRepository {
    public void save(Order order) {
        // JDBC / ORM
    }
}
```

### Critical observation

**Dependencies now point inward**:

* Stripe depends on business abstraction
* MySQL depends on business abstraction
* Business depends on nothing concrete

That’s the **inversion**.

---

## Step 9: Real-World Payoff (Why DIP Matters)

### 1. Change Payment Provider

* Zero changes in OrderService
* Only wiring changes

### 2. Testing Becomes Trivial

```java
class FakePaymentProcessor implements PaymentProcessor {
    public void charge(Money amount) {}
}
```

No Stripe.
No network.
No flakiness.

### 3. Architecture Emerges Naturally

* Business core
* Infrastructure shell

This is **Clean Architecture** in embryo.

---

## Anime Analogy — *Code Geass (Perfect DIP Example)*

Lelouch:

* Defines **strategy**
* Doesn’t care who executes it
* Can swap soldiers, weapons, locations

If Lelouch said:

> “Only Suzaku with this sword can execute this plan”

He would be **architecturally doomed**.

**Strategy must not depend on soldiers.**

---

## How to Recognize DIP in the Wild (Checklist)

When reading code, ask:

1. Is business logic importing framework classes?
2. Will a DB change modify business code?
3. Is testing forcing real infrastructure?
4. Are interfaces named after technologies?

If **yes** → DIP violation.

---

## DIP vs Interfaces (Important Clarification)

❌ DIP is **not** “use interfaces everywhere”
❌ DIP is **not** Dependency Injection frameworks

✅ DIP is **about dependency direction**

Frameworks only help *enforce* it.

---

## One-Sentence Mental Model (Memorize This)

> **Policy depends on meaning, not mechanism.**

If you remember this, DIP becomes obvious.

---

## Final Question for You (Think Carefully)

> If tomorrow Stripe shuts down,
> **which classes should change**?

If your answer includes **business logic** → DIP is broken.
