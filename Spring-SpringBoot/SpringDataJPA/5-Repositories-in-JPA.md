# 📘 Spring Data JPA – Topic 5: Repositories

Repositories in Spring Data JPA provide an abstraction over the **data access layer (DAO layer)**.

Instead of writing DAO implementations manually, you define **repository interfaces**, and Spring automatically generates their implementation at runtime.

Example:

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Spring creates the implementation using **dynamic proxy mechanisms** and internally uses **JPA's EntityManager**.

---

# 1️⃣ Repository Interfaces

Spring Data provides multiple repository interfaces.

| Repository                   | Purpose                                     |
| ---------------------------- | ------------------------------------------- |
| `CrudRepository`             | Basic CRUD operations                       |
| `PagingAndSortingRepository` | CRUD + pagination and sorting               |
| `JpaRepository`              | Full JPA functionality (most commonly used) |

---

## CrudRepository

Provides basic database operations.

Example:

```java
public interface UserRepository extends CrudRepository<User, Long> {
}
```

Key methods:

```java
save(S entity)
findById(ID id)
findAll()
delete(T entity)
deleteById(ID id)
count()
existsById(ID id)
```

Example usage:

```java
userRepository.save(user);
userRepository.findById(1L);
userRepository.deleteById(1L);
```

---

## PagingAndSortingRepository

Adds pagination and sorting features.

Example:

```java
public interface UserRepository
        extends PagingAndSortingRepository<User, Long> {
}
```

Methods added:

```java
findAll(Pageable pageable)
findAll(Sort sort)
```

Used when handling **large datasets**.

---

## JpaRepository

Most commonly used repository interface.

Example:

```java
public interface UserRepository
        extends JpaRepository<User, Long> {
}
```

Additional features:

```java
flush()
saveAndFlush()
deleteAllInBatch()
deleteInBatch()
```

Typical real-world choice:

```text
JpaRepository is usually the default repository type.
```

---

# 2️⃣ Default CRUD Methods

Spring Data repositories automatically include basic operations.

---

## Save Entity

```java
userRepository.save(user);
```

Behavior:

* Inserts a new record
* Updates existing entity if already persisted

JPA determines whether to **INSERT or UPDATE** based on entity state.

---

## Find by ID

```java
Optional<User> user = userRepository.findById(1L);
```

Return type:

```text
Optional<T>
```

Helps avoid `NullPointerException`.

---

## Find All

```java
List<User> users = userRepository.findAll();
```

Fetches all records from the table.

⚠ Be careful with large tables.

---

## Delete Operations

Examples:

```java
userRepository.delete(user);
userRepository.deleteById(id);
userRepository.deleteAll();
```

---

# 3️⃣ Derived Query Methods

Spring Data JPA can generate queries automatically from **method names**.

Example:

```java
List<User> findByName(String name);
```

Spring converts this to JPQL:

```text
SELECT u FROM User u WHERE u.name = :name
```

---

## Common Query Keywords

| Keyword        | Example             |
| -------------- | ------------------- |
| `findBy`       | findByEmail         |
| `countBy`      | countByStatus       |
| `existsBy`     | existsByEmail       |
| `deleteBy`     | deleteByStatus      |
| `findBy...And` | findByNameAndStatus |
| `findBy...Or`  | findByNameOrEmail   |

---

## Multiple Conditions

Example:

```java
List<User> findByNameAndStatus(String name, UserStatus status);
```

Generated query:

```text
WHERE name = ? AND status = ?
```

---

# 4️⃣ `@Query` Annotation

When queries become complex, you can define them manually.

---

## JPQL Query Example

```java
@Query("SELECT u FROM User u WHERE u.email = :email")
User findUserByEmail(String email);
```

Important rule:

JPQL works with **entities and fields**, not tables.

---

## Native SQL Query

You can run raw SQL queries.

```java
@Query(value = "SELECT * FROM users WHERE email = :email",
       nativeQuery = true)
User findByEmailNative(String email);
```

Use native queries only when necessary.

---

# 5️⃣ Query Parameters

Named parameters example:

```java
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);
```

Spring maps method parameters to query parameters.

---

# 6️⃣ Projection Queries -- using @Param or Positional arguments

Used when we only need **specific fields**, not the entire entity.

Example projection interface:

```java
public interface UserNameOnly {

    String getName();

}
```

Repository method:

```java
List<UserNameOnly> findByStatus(UserStatus status);
```

Spring returns **only the required fields**.

---

# 7️⃣ DTO-Based Queries

DTOs allow mapping query results to custom objects.

Example DTO:

```java
public class UserDTO {

    private String name;
    private String email;

    public UserDTO(String name, String email) {
        this.name = name;
        this.email = email;
    }
}
```

Repository query:

```java
@Query("SELECT new com.example.UserDTO(u.name, u.email) FROM User u")
List<UserDTO> findUserDetails();
```

Useful for **optimized read operations**.

---

# 8️⃣ When to Use Each Query Approach

| Approach        | Use Case                     |
| --------------- | ---------------------------- |
| Derived Methods | Simple queries               |
| JPQL (`@Query`) | Complex entity-based queries |
| Native SQL      | Database-specific queries    |
| Projection      | Fetch limited fields         |
| DTO Queries     | Optimized API responses      |

---

# 🧠 Mental Model

Repository layer flow:

```text
Repository Interface
        ↓
Spring Proxy Implementation
        ↓
JPA EntityManager
        ↓
Hibernate ORM
        ↓
SQL Query
        ↓
Database
```

Repositories **delegate persistence operations to JPA/Hibernate internally**.

---

# ⚠️ Common Mistakes

### Using `findAll()` on large tables

Can load huge datasets into memory.

---

### Extremely long derived method names

Example:

```java
findByNameAndEmailAndStatusAndCreatedDate
```

Better to use `@Query`.

---

### Using native SQL unnecessarily

Breaks database portability.

---

### Returning entities when only few fields are needed

Prefer **projections or DTO queries**.

---

# ✅ Summary

Spring Data repositories:

* Automatically implement DAO logic
* Provide built-in CRUD methods
* Support query derivation from method names
* Allow custom JPQL or native queries
* Support projections and DTO-based results

They simplify **database interaction while still using JPA and Hibernate internally**.
