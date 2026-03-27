# 1️⃣ What is a Transaction?

A **transaction** is a group of operations executed as a **single unit of work**.

```text
Either ALL operations succeed
OR everything is rolled back
```

---

## Example

```text
Transfer Money:
1. Debit Account A
2. Credit Account B
```

If step 2 fails:

```text
→ Step 1 must be rolled back
```

---

## ACID Properties (Conceptual)

| Property    | Meaning                              |
| ----------- | ------------------------------------ |
| Atomicity   | All or nothing                       |
| Consistency | Valid data state                     |
| Isolation   | No interference between transactions |
| Durability  | Changes are permanent after commit   |

---

# 2️⃣ @Transactional Annotation

Spring manages transactions using:

```java
@Transactional
```

---

## What It Does

```text
1. Starts transaction
2. Executes method
3. Commits if success
4. Rolls back if exception
```

---

## Example

```java
@Transactional
public void transfer() {
    debit();
    credit();
}
```

---

# 3️⃣ How Spring Implements Transactions

Spring uses **AOP (proxy-based interception)**.

```text
Method Call
   ↓
Spring Proxy
   ↓
Start Transaction
   ↓
Execute Method
   ↓
Commit / Rollback
```

---

## ⚠ Important Limitation

### Self-invocation problem

```java
this.methodB(); ❌
```

Transaction will NOT apply because proxy is bypassed.

---

# 4️⃣ Where to Use @Transactional

---

## ✅ Correct (Service Layer)

```java
@Service
public class UserService {

    @Transactional
    public void createUser() {}
}
```

---

## ❌ Incorrect (Controller Layer)

```java
@RestController
@Transactional ❌
```

Transactions belong to **business logic**, not controllers.

---

# 5️⃣ Default Behavior

```text
Propagation → REQUIRED
Rollback → RuntimeException only
```

---

# 6️⃣ Transaction Propagation

Defines how transactions behave when **nested calls occur**.

---

## REQUIRED (Default)

```text
If transaction exists → join it
Else → create new
```

---

## REQUIRES_NEW

```text
Always create new transaction
Suspend existing one
```

Use case:

```text
Logging, auditing (independent commit)
```

---

## MANDATORY

```text
Must have existing transaction
Else → exception
```

---

## SUPPORTS

```text
Use transaction if exists
Else → run without transaction
```

---

## NOT_SUPPORTED

```text
Always run without transaction
Suspend existing one
```

---

## NEVER

```text
Must NOT run in transaction
Else → exception
```

---

## NESTED (Advanced)

```text
Creates sub-transaction
Supports partial rollback
```

Depends on DB support.

---

# 7️⃣ Propagation Summary

```text
Existing Transaction?

YES → REQUIRED joins
NO  → REQUIRED creates

REQUIRES_NEW → always new
MANDATORY → must exist
SUPPORTS → optional
NOT_SUPPORTED → force no transaction
```

---

# 8️⃣ Rollback Rules

Default:

```text
Rollback only on RuntimeException
```

---

## Example

```java
@Transactional
public void test() throws Exception {
    throw new Exception(); ❌ no rollback
}
```

---

## Force Rollback

```java
@Transactional(rollbackFor = Exception.class)
```

---

# 9️⃣ Read-Only Transactions

```java
@Transactional(readOnly = true)
```

---

## Purpose

```text
Indicates no data modification
```

---

## Benefits

* Skips Hibernate dirty checking
* Improves performance
* Reduces overhead

---

## Example

```java
@Transactional(readOnly = true)
public List<User> getUsers() {
    return userRepository.findAll();
}
```

---

## ⚠ Important

```text
readOnly = true is a hint, not strict enforcement
```

Writes may still happen depending on DB.

---

# 🔟 Common Mistakes

---

## ❌ Missing @Transactional

* Lazy loading errors
* Partial updates
* Data inconsistency

---

## ❌ Using on private methods

Spring AOP won’t intercept → no transaction.

---

## ❌ Self-invocation

```java
this.method() ❌
```

Transaction not applied.

---

## ❌ Overusing REQUIRES_NEW

Can cause **partial commits and inconsistent data**.

---

## ❌ Not using readOnly for queries

Unnecessary overhead.

---

# 🧠 Mental Model

```text
@Transactional
      ↓
Spring Proxy (AOP)
      ↓
TransactionManager
      ↓
EntityManager (JPA)
      ↓
Hibernate
      ↓
Database
      ↓
Commit / Rollback
```

---

# ✅ Summary

Transactions in Spring Data JPA:

* Ensure **data consistency and integrity**
* Managed using `@Transactional`
* Controlled via **propagation strategies**
* Optimized with `readOnly = true`
* Implemented using **Spring AOP + TransactionManager**
