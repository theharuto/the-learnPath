# OOD PRINCIPLES — PART 3

## DRY · KISS · YAGNI (Senior-Level Truths)

---

## DRY — Don’t Repeat Yourself

### Common misunderstanding

> DRY means “no duplication”.

This belief **destroys codebases**.

---

### Senior definition (very important)

> DRY means **no duplication of knowledge**, not no duplication of code.

Two pieces of code that *look the same* but mean different things
**must not be forced together**.

---

### Anime Story — *Hunter x Hunter (Nen Types)*

Nen has:

* Enhancement
* Emission
* Manipulation
* Specialization

They may **look similar in practice**,
but they are **fundamentally different powers**.

If Gon tried to “DRY” Nen:

* merge all types
* force one abstraction

Nen would become useless.

---

### Bad DRY (harmful abstraction)

```java
calculateFee(amount, type)
calculateTax(amount, type)
```

You “DRY” them into:

```java
calculateCharge(amount, mode)
```

Now:

* meaning lost
* future rules entangled
* changes break unrelated logic

---

### Good DRY (shared knowledge)

```java
Money applyPercentage(Money amount, Percentage percentage)
```

Used by:

* tax
* discount
* commission

Same **knowledge**, different **policy**.

---

### Senior DRY Rule

| Duplication Type        | Action        |
| ----------------------- | ------------- |
| Same knowledge          | Abstract      |
| Coincidental similarity | Keep separate |
| Shared algorithm        | Utility       |
| Shared policy           | Strategy      |

---

### DRY Failure Signal

> “Let’s reuse this just in case.”

That is **YAGNI speaking in disguise**.

---

## KISS — Keep It Simple, Stupid

### Junior interpretation

> Short code is simple.

Wrong.

---

### Senior definition

> Simple code is code with **the fewest moving parts**
> **that still communicates intent clearly**.

Simplicity is about **cognitive load**, not line count.

---

### Anime Story — *One Punch Man (Saitama)*

Saitama:

* doesn’t use combos
* doesn’t calculate probabilities
* doesn’t build elaborate plans

One punch. Problem solved.

Complex villains lose because:

* they overthink
* they over-design

---

### Bad KISS Violation (Over-engineering)

```java
AbstractOrderProcessorFactoryProviderManager
```

* Impressive
* Unreadable
* Unmaintainable

---

### Good KISS Design

```java
OrderService
OrderRepository
PaymentGateway
```

Clear. Obvious. Boring.

---

### Senior KISS Rules

* Prefer:

  * straightforward conditionals over clever tricks
  * explicit code over magic
* Avoid:

  * clever lambdas
  * meta-programming unless necessary
* Optimize for:

  * the reader under stress
  * the junior maintainer

---

### KISS Failure Signal

> “This is clever.”

Clever code ages badly.

---

## YAGNI — You Aren’t Gonna Need It

This is the **most emotionally difficult principle**.

---

### Junior fear

> “But what if we need it later?”

---

### Senior reality

> You will need something else later.

YAGNI is about **trusting refactoring**.

---

### Anime Story — *Tokyo Revengers*

Takemichi constantly:

* overthinks future timelines
* panics
* reacts emotionally

But success comes when he:

* focuses on **this moment**
* fixes what is actually broken

Speculating too far ahead **creates new problems**.

---

### Bad YAGNI Violation

```java
interface PaymentGateway {
    pay();
    refund();
    partialRefund();
    schedulePayment();
    internationalPay();
}
```

Only `pay()` is needed today.

---

### Good YAGNI Design

```java
interface PaymentGateway {
    pay();
}
```

Add others **when required**.

---

### Senior YAGNI Rule

| Question                               | If answer is NO  |
| -------------------------------------- | ---------------- |
| Is this required now?                  | Don’t build      |
| Do we have tests protecting refactors? | Don’t pre-design |
| Is change cost low?                    | Don’t speculate  |

---

### YAGNI Failure Signal

> “It might be useful someday.”

That’s how frameworks are born inside applications.

---

## Conflict Resolution — When Principles Clash

This is **senior-only territory**.

---

### DRY vs KISS

* DRY too early → abstraction hell
* KISS too far → duplication

**Rule:**
Start with KISS, move to DRY **when duplication stabilizes**.

---

### YAGNI vs OCP

* OCP too early → unnecessary abstractions
* YAGNI too strict → brittle code

**Rule:**
Apply OCP only to **hotspots**.

---

### SOLID vs Speed

* Perfect design slows delivery
* Bad design slows everything later

**Rule:**
Design just enough to move safely.

---

## FINAL SYNTHESIS — Senior Mental Model

> Clean Code → readability
> SOLID → survivability
> DRY → consistency
> KISS → clarity
> YAGNI → discipline

The goal is **not perfection**.
The goal is **controlled evolution**.

---

## One-Line Memory Anchors

| Principle | Remember This               |
| --------- | --------------------------- |
| DRY       | Duplicate knowledge is evil |
| KISS      | Boring code wins            |
| YAGNI     | Future you can refactor     |
| SOLID     | Protect change boundaries   |
