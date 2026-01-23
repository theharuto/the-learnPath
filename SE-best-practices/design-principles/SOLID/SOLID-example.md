## S — Single Responsibility Principle (SRP)

### Principle

> A class should have **one reason to change** (one actor).

---

### Anime Story — *Demon Slayer (Tanjiro)*

Tanjiro has:

* Swordsmanship
* Smell detection
* Emotional empathy

But in battle:

* **He does not try to heal allies**
* **He does not manage logistics**
* **He does not command the Corps**

Those are separate roles.

If Tanjiro tried to:

* fight demons
* heal wounded slayers
* manage mission records

He would fail at all three under pressure.

---

### Code Mapping

**Bad Design (SRP violation)**

```java
class OrderService {
    void processOrder() {}
    void sendEmail() {}
    void logAudit() {}
}
```

**Why this fails**

* Business change → modify
* Compliance change → modify
* Infrastructure change → modify

Multiple actors → instability.

---

**Good Design (SRP compliant)**

```java
OrderService        // business flow
NotificationService // communication
AuditService        // compliance
```

Each changes for **one reason**.

---

### Memory Hook

> Tanjiro fights.
> He doesn’t run the Corps.

---

## O — Open / Closed Principle (OCP)

### Principle

> Open for **extension**, closed for **modification**.

---

### Anime Story — *Code Geass (Lelouch)*

Lelouch **never rewrites the system**.
He:

* adds new strategies
* recruits new allies
* changes outcomes

But:

* the core structure stays intact
* old plans remain valid

If Lelouch rewrote every plan each time:

* chaos
* exposure
* failure

---

### Code Mapping

**Bad Design**

```java
if (type == A) { ... }
else if (type == B) { ... }
else if (type == C) { ... }
```

Each new type → modify core logic.

---

**Good Design**

```java
interface Strategy { execute(); }
```

Add new strategy → **new class**, no modification.

---

### Memory Hook

> Lelouch adds moves.
> He doesn’t rewrite the board.

---

## L — Liskov Substitution Principle (LSP)

### Principle

> Subtypes must be usable **without breaking expectations**.

---

### Anime Story — *One Punch Man (Saitama)*

Everyone expects heroes to:

* fight strategically
* use techniques
* conserve energy

Saitama:

* doesn’t pretend to do that
* doesn’t fake strategy
* doesn’t violate expectations

He is **honest about his behavior**.

Imagine:

* Saitama joins a stealth squad
* punches instantly
* mission fails

That’s an LSP violation.

---

### Code Mapping

**Broken LSP**

```java
class Bird {
    void fly();
}

class Penguin extends Bird {
    void fly() { throw exception; }
}
```

Penguin **lies** about its ability.

---

**Correct Design**

```java
interface FlyingBird { fly(); }
class Sparrow implements FlyingBird {}
class Penguin { swim(); }
```

No false promises.

---

### Memory Hook

> Never pretend to fly if you can’t.

---

## I — Interface Segregation Principle (ISP)

### Principle

> Clients should not depend on methods they don’t use.

---

### Anime Story — *Spy x Family (Anya)*

Anya is:

* a child
* a telepath

She is **not**:

* a spy
* an assassin
* a diplomat

If Loid forced Anya to:

* carry weapons
* attend secret missions
* follow adult protocols

She would:

* fail
* break missions
* create chaos

---

### Code Mapping

**Bad Interface**

```java
interface Agent {
    fight();
    spy();
    negotiate();
}
```

Every class forced to implement everything.

---

**Good Design**

```java
interface Fighter { fight(); }
interface Spy { spy(); }
interface Diplomat { negotiate(); }
```

Classes choose **only what they need**.

---

### Memory Hook

> Don’t give Anya adult responsibilities.

---

## D — Dependency Inversion Principle (DIP)

### Principle

> High-level policy must not depend on low-level details.

---

### Anime Story — *Attack via Code Geass (Lelouch)*

Lelouch:

* defines strategy
* does not care **who fires the gun**
* does not care **what uniform soldiers wear**

If strategy depended on:

* a specific soldier
* a specific weapon

The system would collapse when anything changed.

---

### Code Mapping

**Bad Design**

```java
class OrderService {
    MySqlRepo repo = new MySqlRepo();
}
```

DB change → break business logic.

---

**Good Design**

```java
interface Repo {}
class OrderService {
    Repo repo;
}
```

Business stays pure.

---

### Memory Hook

> Strategy controls details — not the other way around.

---

## SOLID — ONE-LINE MEMORY SUMMARY

| Principle | Anime Memory                      |
| --------- | --------------------------------- |
| SRP       | Tanjiro fights, he doesn’t manage |
| OCP       | Lelouch adds strategies           |
| LSP       | Don’t pretend you can fly         |
| ISP       | Don’t burden Anya                 |
| DIP       | Strategy > soldiers               |

If these images stay in your head, **SOLID becomes instinct**.
