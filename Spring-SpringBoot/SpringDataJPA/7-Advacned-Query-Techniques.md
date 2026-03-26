# 📘 Spring Data JPA – Topic 7: Advanced Query Techniques

As query complexity increases, Spring Data JPA provides multiple approaches:

```text
Simple → Derived Query Methods
Medium → @Query (JPQL / SQL)
Complex/Dynamic → Specifications + Criteria API
```

---

# 1️⃣ Derived Query Methods (Advanced)

Spring Data generates queries from **method names**.

---

## Common Keywords

| Keyword      | Example                | Meaning          |
| ------------ | ---------------------- | ---------------- |
| GreaterThan  | findByAgeGreaterThan   | age > value      |
| LessThan     | findByAgeLessThan      | age < value      |
| Between      | findByAgeBetween       | range            |
| Like         | findByNameLike         | LIKE             |
| Containing   | findByNameContaining   | %value%          |
| StartingWith | findByNameStartingWith | value%           |
| EndingWith   | findByNameEndingWith   | %value           |
| IgnoreCase   | findByNameIgnoreCase   | case-insensitive |

---

## Examples

### Greater Than

```java
List<User> findByAgeGreaterThan(int age);
```

---

### Case-Insensitive Search

```java
List<User> findByNameIgnoreCase(String name);
```

---

### Contains Search

```java
List<User> findByNameContaining(String keyword);
```

Equivalent SQL:

```sql
WHERE name LIKE %keyword%
```

---

### Multiple Conditions

```java
List<User> findByAgeGreaterThanAndStatus(int age, UserStatus status);
```

---

## ⚠ Limitation

Very long method names become unreadable:

```java
findByNameAndStatusAndAgeGreaterThanAndCityAndActive
```

➡ Use `@Query` or Specifications instead.

---

# 2️⃣ Specifications (Dynamic Queries)

Spring Data JPA supports dynamic queries via:

```java
JpaSpecificationExecutor<T>
```

---

## Enable Specifications

```java
public interface UserRepository
        extends JpaRepository<User, Long>,
                JpaSpecificationExecutor<User> {
}
```

---

## What is a Specification?

A Specification represents **one query condition**.

Structure:

```java
Specification<T> = (root, query, cb) -> Predicate
```

| Parameter | Description       |
| --------- | ----------------- |
| root      | Entity reference  |
| query     | Query being built |
| cb        | CriteriaBuilder   |

---

## Example Specifications

### Filter by Status

```java
public static Specification<User> hasStatus(UserStatus status) {
    return (root, query, cb) ->
            cb.equal(root.get("status"), status);
}
```

---

### Filter by Age

```java
public static Specification<User> ageGreaterThan(int age) {
    return (root, query, cb) ->
            cb.greaterThan(root.get("age"), age);
}
```

---

# 3️⃣ Combining Specifications

Specifications can be combined dynamically.

---

## AND Condition

```java
Specification<User> spec =
        Specification.where(hasStatus(ACTIVE))
                     .and(ageGreaterThan(25));
```

---

## OR Condition

```java
Specification<User> spec =
        Specification.where(hasStatus(ACTIVE))
                     .or(ageGreaterThan(25));
```

---

## NOT Condition

```java
Specification<User> spec =
        Specification.not(hasStatus(INACTIVE));
```

---

## Repository Usage

```java
List<User> users = userRepository.findAll(spec);
```

---

# 4️⃣ Dynamic Query Building

Example API:

```text
/users?status=ACTIVE&minAge=25
```

Dynamic Specification:

```java
Specification<User> spec = Specification.where(null);

if (status != null) {
    spec = spec.and(hasStatus(status));
}

if (minAge != null) {
    spec = spec.and(ageGreaterThan(minAge));
}
```

Final query depends on input parameters.

---

# 5️⃣ JPA Criteria API

Specifications are built on top of the **JPA Criteria API**.

---

## Purpose

* Build queries programmatically
* Avoid string-based JPQL
* Provide type safety

---

## Example

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();

CriteriaQuery<User> query = cb.createQuery(User.class);

Root<User> root = query.from(User.class);

query.select(root)
     .where(cb.equal(root.get("status"), "ACTIVE"));
```

Equivalent JPQL:

```text
SELECT u FROM User u WHERE u.status = 'ACTIVE'
```

---

# 6️⃣ When to Use What

| Approach        | Use Case          |
| --------------- | ----------------- |
| Derived Methods | Simple filters    |
| `@Query`        | Medium complexity |
| Specifications  | Dynamic filters   |
| Criteria API    | Low-level control |

---

# 7️⃣ Real-World Example

Search API:

```text
/users/search?name=John&status=ACTIVE&minAge=25
```

Specification:

```java
Specification<User> spec =
        where(nameContains("John"))
        .and(hasStatus(ACTIVE))
        .and(ageGreaterThan(25));
```

Execution:

```java
userRepository.findAll(spec);
```

---

# ⚠ Common Mistakes

### Overusing derived queries

Leads to unreadable method names.

---

### Using Criteria API directly everywhere

Too verbose. Prefer Specifications.

---

### Ignoring database indexes

Dynamic queries often filter multiple fields → can become slow.

---

### Not handling null filters

Leads to incorrect query construction.

---

# 🧠 Mental Model

```text
Simple Query → Derived Methods
        ↓
Moderate Query → @Query
        ↓
Dynamic Query → Specifications
        ↓
Low-level → Criteria API
```

---

# ✅ Summary

Advanced querying in Spring Data JPA includes:

* Derived query methods using naming conventions
* `@Query` for custom JPQL or SQL queries
* Specifications for dynamic query building
* Criteria API for programmatic, type-safe queries

Specifications internally use the **JPA Criteria API** to build queries dynamically.
