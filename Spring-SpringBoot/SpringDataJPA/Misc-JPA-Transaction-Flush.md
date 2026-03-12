## 1️⃣ Transaction

**Definition**

A **transaction** is a group of database operations that must **either all succeed or all fail**.

Example:

```text
Transfer money
1 Deduct from Account A
2 Add to Account B
```

If step 2 fails → step 1 must rollback.

---

## 2️⃣ ACID Properties

Transactions follow ACID rules:

| Property    | Meaning                        |
| ----------- | ------------------------------ |
| Atomicity   | All operations succeed or none |
| Consistency | Database remains valid         |
| Isolation   | Transactions don't interfere   |
| Durability  | Committed data persists        |

Handled by databases like **PostgreSQL**, **MySQL**.

---

## 3️⃣ Where Transaction Happens

```text
Application
   ↓
Spring
   ↓
Hibernate/JPA
   ↓
JDBC
   ↓
Database ← actual transaction
```

| Layer     | Role                         |
| --------- | ---------------------------- |
| Spring    | Manages transaction boundary |
| Hibernate | Tracks entity changes        |
| Database  | Executes transaction         |

---

## 4️⃣ @Transactional in Spring

In **Spring Framework**:

```java
@Transactional
public void createUser() {
    repository.save(user);
}
```

Spring internally:

```text
1 Start DB transaction
2 Execute method
3 Flush Hibernate changes
4 Commit transaction
```

If exception occurs → **rollback**.

---

# Flush

## 5️⃣ Flush Definition

**Flush = Synchronizing persistence context changes with the database.**

Meaning:

```text
Entities in memory → converted to SQL → sent to DB
```

Used in **Hibernate** / **Jakarta Persistence**.

---

## 6️⃣ Important Point

Flush **does NOT commit the transaction**.

```text
flush → SQL executed
commit → changes permanently saved
```

Rollback can still undo flushed changes.

---

## 7️⃣ When Flush Happens

Automatic flush occurs:

1. **Before transaction commit**
2. **Before executing queries**
3. **When `entityManager.flush()` is called**

---

## 8️⃣ Example Flow

```java
@Transactional
public void saveUser() {
    User u = new User("John");
    entityManager.persist(u);
}
```

Execution:

```text
persist()
↓
Stored in persistence context
↓
flush
↓
INSERT SQL generated
↓
commit
↓
Data permanently saved
```

---

# One-Line Summary

**Transaction**

> Ensures multiple operations execute atomically (all succeed or all fail).

**Flush**

> Synchronizes in-memory entity changes with the database by executing SQL (without committing).

---

✅ **Super Short Memory Trick**

```text
Transaction → controls commit/rollback
Flush → sends SQL to database
```
