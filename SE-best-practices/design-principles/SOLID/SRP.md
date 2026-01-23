# OBJECT-ORIENTED DESIGN PRINCIPLES

**From Clean Code to Scalable Design**

---

## 0. First: Correct Mental Model (Very Important)

### Junior misconception

> SOLID, DRY, KISS, YAGNI are rules to follow.

Wrong.

### Senior reality

> These principles are **forces** you balance, not laws you obey.

Violating them blindly is bad.
Following them blindly is **worse**.

Your goal is:

* minimize **change cost**
* minimize **blast radius**
* maximize **clarity of responsibility**

---

## How This Module Connects to Clean Code

| Clean Code       | OOD Principles |
| ---------------- | -------------- |
| Small functions  | SRP            |
| Clear naming     | SRP, DRY       |
| Few side effects | SRP, DIP       |
| Fewer comments   | KISS           |
| Easy refactoring | SOLID          |
| Easy deletion    | YAGNI          |

If Clean Code answers **“Is this readable?”**
OOD answers **“Is this survivable?”**

# CHUNK 1 — SOLID PRINCIPLES (Foundations)

**References (conceptual anchors):**

* Baeldung
* OODesign

SOLID is not about Java.
SOLID is about **change vectors**.

---

## S — Single Responsibility Principle (SRP)

> A class should have **one reason to change**.

This sentence is deceptively simple and widely misunderstood.

---

### What SRP is NOT

* ❌ “A class should do only one thing”
* ❌ “A class should have only one method”
* ❌ “Small classes are always good”

These lead to **over-fragmentation**.

---

### What SRP REALLY means (Senior definition)

> A class should serve **one actor** (stakeholder / concern).

If two different people ask for changes in the same class → SRP violation.

---

### Real-world actor examples

| Actor      | Change Reason        |
| ---------- | -------------------- |
| Business   | Pricing rules        |
| Compliance | Auditing             |
| Ops        | Logging / monitoring |
| UI         | Presentation format  |

If one class changes for multiple actors → **it will rot**.

---

### Anime analogy — *Attack via Demon Slayer*

Think of **Tanjiro’s sword forms**:

* Each form has one purpose
* Mixing forms mid-fight causes failure

Classes that mix concerns fail under pressure.

---

## SRP Violation — Realistic Example

```java
public class OrderService {

    public void processOrder(Order order) {
        validate(order);
        calculatePrice(order);
        saveToDatabase(order);
        sendEmail(order);
        logAudit(order);
    }
}
```

### Why juniors think this is fine

* Reads clean
* Method names look good
* “Everything related to order is here”

### Why seniors reject this

This class changes when:

* business rules change
* database changes
* email system changes
* audit rules change

**Multiple actors → SRP violation**

---

## SRP-Compliant Design (Clean + OOD)

```java
public class OrderService {

    private final OrderValidator validator;
    private final PricingService pricingService;
    private final OrderRepository repository;
    private final NotificationService notificationService;
    private final AuditService auditService;

    public void process(Order order) {
        validator.validate(order);
        Money price = pricingService.calculate(order);
        repository.save(order, price);
        notificationService.notify(order);
        auditService.record(order);
    }
}
```

### Why this is SRP-compliant

* `OrderService` orchestrates (one reason to change)
* Each dependency serves one actor
* Changes are localized

This is **clean code elevated to design**.

---

## Senior SRP Heuristics (Rules of Thumb)

* If a class has:

  * `print`, `save`, `calculate`, `validate` together → 🚩
* If tests for a class:

  * require unrelated mocks → 🚩
* If two teams argue over a class → 🚩

SRP violations show up **socially before technically**.

---

## Common SRP Interview Trap

> “SRP means one responsibility.”

Ask back:

> “One responsibility for *which actor*?”

That answer separates seniors from juniors.

---

## SRP vs Clean Code (Key Connection)

| Clean Code       | SRP                  |
| ---------------- | -------------------- |
| Small functions  | Focused classes      |
| Clear naming     | Clear responsibility |
| Easy refactoring | Isolated change      |

Clean code makes SRP **possible**.
SRP makes clean code **sustainable**.
