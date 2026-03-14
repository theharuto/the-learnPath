**Query parameters** are placeholders used inside JPQL or SQL queries to safely bind method arguments to query values.

### ✔ Why we use them

*   Prevent SQL Injection (values are safely bound, not string‑concatenated) [\[baeldung.com\]](https://www.baeldung.com/the-persistence-layer-with-spring-data-jpa)
*   Keep queries clean, readable, and reusable
*   Automatically map Java method parameters to query parameters
*   Avoid manual string building and type conversion issues

### ✔ Named Parameters (`:name`)

```java
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);
```

*   `:name` = **placeholder** in JPQL
*   `@Param("name")` binds the method argument to the JPQL placeholder
*   Required whenever you use named parameters in `@Query` [\[docs.spring.io\]](https://docs.spring.io/spring-framework/reference/core/aop/introduction-proxies.html)

### ✔ Positional Parameters (`?1`)

```java
@Query("SELECT u FROM User u WHERE u.email = ?1")
User findByEmail(String email);
```

*   Parameter is bound **by position** → `@Param` not required
*   Supported by Spring Data's query mechanism for JPQL/native queries [\[techoral.com\]](https://techoral.com/spring/spring-proxies.html)

### ✔ Summary

*   Use `@Param("x")` **only when using named parameters** (`:x`).
*   With positional parameters (`?1`, `?2`), `@Param` is **not needed**.
*   Query parameters make queries dynamic, safe, and maintainable.
