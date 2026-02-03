## 1. Why Factory Pattern Exists

**Problem it solves**

* Object creation logic leaking into business logic
* Tight coupling to concrete classes (`new`)
* Difficulty extending behavior safely
* Violation of **SRP, OCP, DIP**

**Core intent**

> Separate **object creation** from **object usage**

Factory answers:

> “Which object should I give you?”

Factory does **NOT**:

* Execute business logic
* Contain domain behavior

---

## 2. Factory vs Business Logic (Intent Clarity)

### Correct responsibility split

| Role          | Responsibility                    |
| ------------- | --------------------------------- |
| Factory       | Decide **which object to create** |
| Service       | Coordinate workflow               |
| Domain object | Execute behavior                  |

If a factory starts “doing work” → **design is wrong**

---

## 3. The Naive (Wrong) Approach

```java
if (type.equals("EMAIL")) {
    new EmailSender().send();
}
```

**Problems**

* Business logic knows concrete classes
* New type → modify business logic
* Hard to test
* OCP violation

---

## 4. Simple Factory (SRP Improvement, NOT OCP)

```java
public class NotificationSenderFactory {
    public NotificationSender create(String type) {
        switch (type) {
            case "EMAIL": return new EmailSender();
            case "SMS": return new SmsSender();
            default: throw new IllegalArgumentException();
        }
    }
}
```

### What this fixes

* ✅ SRP (creation isolated)
* ✅ Cleaner business code

### What this does NOT fix

* ❌ OCP (switch must change for new type)

---

## 5. IMPORTANT REALIZATION (Your Doubt — CORRECT)

> “Even with a factory, using switch violates OCP”

✅ **Correct**

**Key insight**

> Factory does NOT automatically mean OCP-compliant

Design maturity is **progressive**, not binary.

---

## 6. OCP Requires Identifying the Volatility

Ask:

> What changes most often?

Here:

* Mapping of `type → implementation`

So OCP demands:

> Move this knowledge **out of stable code**

---

## 7. OCP-Compliant Factory (Registry-Based — Production Style)

```java
public class NotificationSenderFactory {

    private final Map<String, NotificationSender> registry = new HashMap<>();

    public void register(String type, NotificationSender sender) {
        registry.put(type, sender);
    }

    public NotificationSender create(String type) {
        NotificationSender sender = registry.get(type);
        if (sender == null) {
            throw new IllegalArgumentException("Unknown type: " + type);
        }
        return sender;
    }
}
```

### Registration (startup / composition root)

```java
factory.register("EMAIL", new EmailSender());
factory.register("SMS", new SmsSender());
```

### Adding new type (OCP satisfied)

```java
factory.register("WHATSAPP", new WhatsAppSender());
```

✅ No modification to factory logic
✅ Open for extension
✅ Closed for modification

---

## 8. Why This Is TRUE OCP

| Approach         | SRP | OCP |
| ---------------- | --- | --- |
| Inline `new`     | ❌   | ❌   |
| Switch factory   | ✅   | ❌   |
| Registry factory | ✅   | ✅   |
| DI container     | ✅   | ✅   |

---

## 9. Factory vs Strategy (Quick Clarity)

| Factory             | Strategy               |
| ------------------- | ---------------------- |
| Chooses **object**  | Chooses **algorithm**  |
| Happens at creation | Happens at execution   |
| One-time decision   | Can change dynamically |

Factory → “Which implementation?”
Strategy → “Which behavior?”

---

## 10. Factory Method vs Registry Factory

### Factory Method (GoF classic)

* New subclass per type
* OCP-safe
* Can cause class explosion

### Registry Factory (Modern, practical)

* Runtime registration
* DI-friendly
* Fewer classes
* Preferred in real systems

---

## 11. Factory + DIP (Very Important)

Good factory design:

* Returns **interfaces**
* Hides concrete implementations
* Keeps business logic independent of details

Bad factory design:

* Returns concrete classes
* Hardcodes tech decisions

---

## 12. Senior Rules of Thumb (Memorize)

* **Factories create, they never act**
* **Switch = embedded knowledge**
* **Registry = injected knowledge**
* **SRP first, OCP later**
* **Do not apply OCP prematurely**
* **Prefer DI-managed factories**

---

## 13. When NOT to Use Factory

❌ Only one implementation
❌ Creation is trivial
❌ No expected variation

Overusing Factory → abstraction bloat

---

## 14. One-Line Mental Model (Lock This In)

> **Factory isolates `new` so business logic never changes when types grow.**

---

## 15. Interview-Ready Summary

* Factory separates creation from usage
* Simple Factory improves SRP, not OCP
* OCP requires removing conditionals
* Registry-based factories are production-grade
* DI containers often eliminate explicit factories
