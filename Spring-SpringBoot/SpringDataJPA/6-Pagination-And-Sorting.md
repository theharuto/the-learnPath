# 📘 Spring Data JPA – Topic 6: Pagination and Sorting

Pagination and sorting allow efficient handling of **large datasets** by retrieving data in **smaller chunks (pages)** instead of loading entire tables into memory.

Example problem:

```text
users table → millions of rows
```

Running:

```java
userRepository.findAll();
```

may load all rows into memory. Pagination solves this by **limiting results per request**.

---

# 1️⃣ Pageable Interface

`Pageable` represents **pagination information**.

It contains:

* Page number
* Page size
* Sorting configuration

Spring Data uses this to generate SQL with **LIMIT and OFFSET**.

Example:

```java
Pageable pageable = PageRequest.of(0, 10);
```

Meaning:

```text
page = 0 (first page)
size = 10 rows
```

Generated SQL (conceptually):

```sql
SELECT * FROM users LIMIT 10 OFFSET 0
```

---

# 2️⃣ Page Interface

When a repository returns `Page<T>`, Spring provides **pagination metadata** along with the data.

Example repository method:

```java
Page<User> findAll(Pageable pageable);
```

Usage:

```java
Pageable pageable = PageRequest.of(0, 10);
Page<User> page = userRepository.findAll(pageable);
```

---

## Page Methods

| Method               | Description                 |
| -------------------- | --------------------------- |
| `getContent()`       | Returns the list of results |
| `getTotalElements()` | Total rows in database      |
| `getTotalPages()`    | Total number of pages       |
| `getNumber()`        | Current page number         |
| `hasNext()`          | Whether next page exists    |
| `isFirst()`          | Whether first page          |
| `isLast()`           | Whether last page           |

Example:

```java
List<User> users = page.getContent();
```

---

# 3️⃣ Creating Pageable Objects

Spring Data provides `PageRequest` implementation.

Example:

```java
Pageable pageable = PageRequest.of(0, 10);
```

Parameters:

```text
PageRequest.of(pageNumber, pageSize)
```

Example:

```java
PageRequest.of(2, 20);
```

Meaning:

```text
3rd page
20 records per page
```

---

# 4️⃣ Sorting with Sort

Spring Data provides `Sort` to define ordering.

Example:

```java
Sort sort = Sort.by("name").ascending();
```

Equivalent SQL:

```sql
ORDER BY name ASC
```

Descending:

```java
Sort sort = Sort.by("name").descending();
```

Equivalent SQL:

```sql
ORDER BY name DESC
```

---

# 5️⃣ Combining Pagination and Sorting

`PageRequest` can include sorting.

Example:

```java
Pageable pageable =
        PageRequest.of(0, 10, Sort.by("name").ascending());
```

Meaning:

```text
Page 0
10 records
Sorted by name ASC
```

Generated SQL (conceptual):

```sql
SELECT * FROM users
ORDER BY name ASC
LIMIT 10 OFFSET 0
```

---

# 6️⃣ Pagination in Repository Query Methods

Derived queries can include `Pageable`.

Example:

```java
Page<User> findByStatus(UserStatus status, Pageable pageable);
```

Usage:

```java
Pageable pageable = PageRequest.of(0, 10);
Page<User> users = userRepository.findByStatus(ACTIVE, pageable);
```

Generated SQL:

```sql
SELECT * FROM users
WHERE status = 'ACTIVE'
LIMIT 10 OFFSET 0
```

---

# 7️⃣ Slice<T> (Lightweight Pagination)

Spring Data also provides `Slice<T>`.

Difference:

| Feature              | Page            | Slice  |
| -------------------- | --------------- | ------ |
| Executes count query | Yes             | No     |
| Provides total pages | Yes             | No     |
| Performance          | Slightly slower | Faster |

Reason:

`Page` executes **two queries**:

```sql
SELECT ... LIMIT ...
SELECT COUNT(*) ...
```

`Slice` executes only **one query**.

---

## Example Using Slice

Repository method:

```java
Slice<User> findByStatus(UserStatus status, Pageable pageable);
```

Usage:

```java
Pageable pageable = PageRequest.of(0, 10);

Slice<User> slice =
        userRepository.findByStatus(ACTIVE, pageable);

List<User> users = slice.getContent();

boolean hasNext = slice.hasNext();
```

Use `Slice` when:

```text
You only need next-page navigation
Total count is unnecessary
```

This improves performance in large datasets.

---

# 8️⃣ How Spring Web Resolves Pageable from URL

Spring MVC automatically converts query parameters into a `Pageable` object using **Spring Data Web Support**.

This works via:

```text
PageableHandlerMethodArgumentResolver
```

which resolves request parameters into `PageRequest`.

---

## Example REST API Request

```text
GET /users?page=0&size=10&sort=name,asc
```

Spring resolves this into:

```java
PageRequest.of(0, 10, Sort.by("name").ascending());
```

---

## Controller Example

```java
@GetMapping("/users")
public Page<User> getUsers(Pageable pageable) {
    return userRepository.findAll(pageable);
}
```

Spring automatically binds:

```text
page → page number
size → page size
sort → sorting configuration
```

---

## Query Parameter Mapping

| Query Parameter | Pageable Property         |
| --------------- | ------------------------- |
| page            | Page number               |
| size            | Page size                 |
| sort=name,asc   | Sorting field + direction |

Multiple sorts are supported:

```text
GET /users?sort=name,asc&sort=id,desc
```

---

# 9️⃣ Example Full Request Flow

```text
Client Request
GET /users?page=1&size=5&sort=name,asc
        ↓
Spring Web Resolver
(PageableHandlerMethodArgumentResolver)
        ↓
Creates PageRequest
PageRequest.of(1,5,Sort.by("name").ascending())
        ↓
Repository Method
        ↓
SQL Query Generated
LIMIT + OFFSET + ORDER BY
        ↓
Database
```

---

# ⚠️ Common Mistakes

### Using `findAll()` on large tables

Loads entire dataset.

---

### Very large page sizes

Example:

```text
size=10000
```

Defeats pagination purpose.

---

### Sorting on non-indexed columns

Causes slow queries.

---

### Pagination without deterministic sorting

Example:

```java
PageRequest.of(0,10);
```

Better:

```java
PageRequest.of(0,10, Sort.by("id"));
```

---

# 🧠 Mental Model

Pagination flow:

```text
Client Request
      ↓
Pageable
      ↓
Repository
      ↓
Generated SQL (LIMIT + OFFSET)
      ↓
Database
```

Sorting adds:

```text
ORDER BY column ASC/DESC
```

---

# ✅ Summary

Pagination and sorting in Spring Data JPA are implemented using:

* `Pageable` → pagination configuration
* `Page<T>` → results + metadata
* `Slice<T>` → lightweight pagination
* `Sort` → ordering results
* `PageRequest` → concrete implementation

Spring MVC automatically converts **HTTP query parameters into Pageable objects**, enabling seamless pagination for REST APIs.
