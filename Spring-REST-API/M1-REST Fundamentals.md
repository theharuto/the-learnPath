# 📘 Module 1: REST Fundamentals (Spring Boot Backend)

---

## 🔹 What is REST?

**REST (Representational State Transfer)** is an architectural style for designing APIs.

### Core Ideas:

* Everything is a **Resource**
* Each resource is identified by a **URI**
* Communication happens via **HTTP methods**
* APIs must be **Stateless**

---

## 🔹 REST Mental Model

* Client → sends request
* Server → returns representation (usually JSON)
* Each request is **independent**
* No memory of previous interactions

---

## 🔹 REST Principles

### 1. Resource-Oriented Design

* Use **nouns**, not verbs

```
❌ /getUser
✅ /users/{id}
```

---

### 2. Statelessness (CRITICAL)

* Server does NOT store client context
* Each request must contain all required data

#### Why it matters:

* Enables **horizontal scaling**
* Supports **load balancing**
* Improves **fault tolerance**

---

### 3. Uniform Interface (HTTP Methods)

| Method | Purpose          |
| ------ | ---------------- |
| GET    | Fetch resource   |
| POST   | Create resource  |
| PUT    | Replace resource |
| PATCH  | Partial update   |
| DELETE | Remove resource  |

---

### 4. Representation

* Data exchanged as:

  * JSON (standard)
  * XML (rare)

---

## 🔹 Idempotency

### Definition:

> An operation is **idempotent** if repeating it produces the same final state.

---

### HTTP Idempotency Table

| Method | Idempotent? |
| ------ | ----------- |
| GET    | ✅           |
| PUT    | ✅           |
| DELETE | ✅           |
| POST   | ❌           |
| PATCH  | ⚠️ Depends  |

---

### Examples

#### ✅ Idempotent

```
PUT /users/1 { "name": "John" }
DELETE /users/1
```

#### ❌ Not Idempotent

```
POST /users
PATCH /users/1 { "balance": +100 }
```

---

### Key Rule:

> Focus on **final system state**, not response or request repetition

---

## 🔹 Why Idempotency Matters

* Handles **network retries**
* Prevents **duplicate operations**
* Critical for:

  * Payments
  * Orders
  * Distributed systems

---

### Real Solution:

* Use **Idempotency Keys**

```
POST /payments
Idempotency-Key: unique-id
```

---

## 🔹 Stateless vs Stateful Systems

### Stateless (Preferred)

* Any request → any server
* No session storage
* Easy scaling

### Stateful (Problems)

* Requires sticky sessions
* Breaks load balancing
* Loses data on server crash

---

## 🔹 Concurrency Problems

### Lost Update Problem

Occurs when:

* Multiple clients update same resource
* One update overwrites another

---

### Example

Initial:

```
balance = 1000
```

Two updates:

```
+100 → 1100
+200 → 1200
```

Final (wrong):

```
1200 ❌
```

Expected:

```
1300 ✅
```

---

## 🔹 Locking Strategies

### Optimistic Locking

* Uses versioning (`@Version`)
* Detects conflicts during update
* No blocking

#### Pros:

* High performance
* Scalable
* Good for low contention

#### Cons:

* High retries under contention

---

### Pessimistic Locking

* Locks row before update
* Prevents concurrent access

#### Pros:

* Prevents conflicts

#### Cons:

* Blocking
* Deadlocks
* Poor scalability

---

## 🔹 High Contention Problem

If many users update same row:

* Optimistic locking → frequent failures & retries
* System slows down

---

## 🔹 Production-Level Solutions

### 1. Avoid Hot Rows

* Split data (sharding)

---

### 2. Event-Based / Ledger Design (BEST)

```
Store transactions → derive state
```

---

### 3. Queue-Based Processing

* Serialize updates via queue (Kafka, RabbitMQ)

---

### 4. Atomic DB Updates

```
UPDATE balance = balance + 100
```

---

### 5. Selective Pessimistic Locking

* Only for critical sections

---

## 🔹 Key Takeaways

* REST = resource-based, stateless architecture
* Statelessness enables scalability
* Idempotency is critical for retries
* PATCH is not inherently idempotent
* Avoid direct state mutation in concurrent systems
* Fix **system design**, not just locking
* Prefer **event-driven / ledger-based models**

---

## 🔹 Interview Quick Hits

* REST is an **architectural style**, not protocol
* Statelessness → enables horizontal scaling
* Idempotent → same final state after retries
* PUT vs PATCH → full vs partial update
* Optimistic vs Pessimistic → detect vs prevent conflicts
* Lost update problem → major concurrency issue

---

## 🔹 Mental Models

* “Every request is independent”
* “Final state > number of executions”
* “Avoid hot rows”
* “Derive state, don’t mutate blindly”

---
