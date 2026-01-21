# CLEAN CODE — FULL DEMONSTRATION

### *“The Demon Slayer Payment System”*

### Story Setup

You are building a system for the **Demon Slayer Corps**.

* **Tanjiro** places an order for a sword
* System must:

  * Validate the order
  * Calculate price
  * Save it
  * Handle failures
  * Notify Tanjiro

This system has existed for years. Multiple engineers touched it.
Now **you** inherit it.

---

## STEP 1: The Legacy Code (Dirty but “Working”)

This is the kind of code you *will* see in real companies.

```java
public class OrderProcessor {

    public void process(Order o, boolean f) {
        if (o == null) {
            System.out.println("order null");
            return;
        }

        if (o.getUser() == null || o.getUser().getStatus() != 1) {
            System.out.println("user invalid");
            return;
        }

        double p = 0;
        for (Item i : o.getItems()) {
            p += i.getPrice() * i.getQty();
        }

        if (f) {
            p = p - (p * 0.1);
        }

        try {
            Database.save(o, p);
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("Order processed");
    }
}
```

---

## STEP 2: Identify Clean Code Violations (Senior Lens)

### 1. Naming Problems

* `process` → vague
* `o`, `p`, `f` → no intent
* `1` → magic number

### 2. Function Problems

* Multiple responsibilities:

  * Validation
  * Pricing
  * Discount logic
  * Persistence
  * Logging
* Boolean argument `f` → unclear behavior

### 3. Comments Problem

* Print statements pretending to explain logic
* No business reasoning expressed

### 4. Error Handling

* Catching `Exception`
* Printing stack trace
* Continuing blindly

### 5. Dependency Problems

* Static `Database.save`
* Hard to test
* Hard to replace

This code **works**, but:

* It is fragile
* It is risky
* It does not scale teams

---

## STEP 3: Refactor with Clean Code Principles

We refactor **without changing behavior**.

---

## STEP 4: Clean, Senior-Level Code

### Domain-Driven Naming

```java
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    public OrderService(OrderRepository orderRepository,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }

    public void processOrder(Order order) {
        validateOrder(order);

        Money totalPrice = calculatePrice(order);

        orderRepository.save(order, totalPrice);

        notificationService.notifyOrderProcessed(order);
    }
}
```

---

## STEP 5: Single-Responsibility Functions

```java
private void validateOrder(Order order) {
    if (order == null) {
        throw new InvalidOrderException("Order cannot be null");
    }

    if (!order.getUser().isActive()) {
        throw new InvalidOrderException("User is not active");
    }
}
```

**Why this is clean:**

* Clear intent
* No comments needed
* Fail fast
* Domain language used

---

## STEP 6: Remove Magic Numbers & Clarify Business Rules

```java
private Money calculatePrice(Order order) {
    Money basePrice = sumItemPrices(order);

    if (order.isEligibleForDiscount()) {
        return basePrice.applyDiscount(Discount.TEN_PERCENT);
    }

    return basePrice;
}
```

```java
private Money sumItemPrices(Order order) {
    return order.getItems()
                .stream()
                .map(Item::totalPrice)
                .reduce(Money.ZERO, Money::add);
}
```

---

## STEP 7: Proper Error Handling

```java
public class InvalidOrderException extends RuntimeException {
    public InvalidOrderException(String message) {
        super(message);
    }
}
```

**Senior reasoning:**

* Specific exception
* Meaningful message
* Preserves stack trace
* Part of API contract

---

## STEP 8: Dependency Management (Testable Design)

```java
public interface OrderRepository {
    void save(Order order, Money totalPrice);
}
```

```java
public interface NotificationService {
    void notifyOrderProcessed(Order order);
}
```

Now:

* Easy to mock
* Easy to replace
* Easy to test

---

## STEP 9: Comments — When They Actually Make Sense

```java
// Discount applies only to veteran demon slayers to prevent abuse
if (order.isEligibleForDiscount()) {
    return basePrice.applyDiscount(Discount.TEN_PERCENT);
}
```

This comment explains **WHY**, not **HOW**.
This comment survives refactoring.

---

## STEP 10: Anime Mapping (Memory Anchor)

### Tanjiro = Clean Code

* Calm
* Intentional
* Every move has purpose

### Zenitsu = Dirty Code

* Works under pressure
* Unpredictable
* Hard to rely on

### Muzan = Technical Debt

* Grows silently
* Destroys systems from inside
* Must be confronted early

Clean code is **Tanjiro’s sword form**:

* Disciplined
* Repeatable
* Reliable under stress

---

## STEP 11: What Changed (Senior Summary)

| Aspect       | Before         | After                 |
| ------------ | -------------- | --------------------- |
| Naming       | Cryptic        | Intentional           |
| Functions    | God method     | Single responsibility |
| Comments     | Noise          | Business reasoning    |
| Errors       | Hidden         | Explicit              |
| Dependencies | Static & tight | Inverted & testable   |
| Change risk  | High           | Low                   |

---

## FINAL SENIOR TRUTH

> Clean code is not about making code look nice.
> Clean code is about making **wrong changes hard** and **right changes easy**.

next 

1. Apply this to **a real Java interview question**
2. Write **unit tests (TDD style)** for this code
3. Break this code intentionally and analyze failures
4. Design a **mini clean-architecture project**
