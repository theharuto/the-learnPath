# 📦 Module 9: Pagination & Sorting (Deep Dive)

---

# 🔹 0. Start From Absolute Basics

## Problem Without Pagination

Imagine your API:

```text
GET /resources
```

Database has:

```text
1,000,000 records
```

---

## What happens?

```text
DB → fetch 1M rows
→ JVM loads into memory
→ JSON conversion
→ send over network
```

💥 Problems:

* High memory usage
* Slow response
* Network congestion
* DB load spike

---

## Core Idea

> Never send all data. Always send **small chunks**.

---

# 🔹 1. What is Pagination?

## Definition

> Breaking large dataset into **smaller pages**

---

## Mental Model

```text
Total data → divided into pages

Page 0 → items 1–10  
Page 1 → items 11–20  
Page 2 → items 21–30
```

---

## Terminology (VERY IMPORTANT)

| Term   | Meaning                        |
| ------ | ------------------------------ |
| page   | page index (0-based in Spring) |
| size   | number of items per page       |
| offset | how many rows to skip          |

---

## Formula

```text
offset = page * size
```

---

## Example

```text
page=2, size=10
offset = 2 * 10 = 20
```

→ Skip 20 records, return next 10

---

---

# 🔹 2. SQL Level Understanding (Critical)

---

## What DB does

```sql
SELECT * FROM resource
LIMIT 10 OFFSET 20;
```

---

## Meaning

* LIMIT → number of rows
* OFFSET → skip rows

---

👉 Spring generates this automatically

---

---

# 🔹 3. Spring Boot Entry Point: `Pageable`

---

## What is `Pageable`?

> Object that carries pagination + sorting info

---

## Fields inside Pageable

* page number
* page size
* sort details

---

## Controller Example

```java
@GetMapping("/resources")
public Page<Resource> getAll(Pageable pageable) {
    return repository.findAll(pageable);
}
```

---

## Request

```text
GET /resources?page=0&size=5
```

---

## Internal Flow

```text
Query params
   ↓
Spring converts → Pageable object
   ↓
Repository uses Pageable
   ↓
DB query generated
```

---

---

# 🔹 4. What is `Page<T>`?

---

## Definition

> A wrapper around paginated result + metadata

---

## Structure

```json
{
  "content": [...],
  "totalElements": 100,
  "totalPages": 20,
  "size": 5,
  "number": 0
}
```

---

## Key Fields

| Field         | Meaning          |
| ------------- | ---------------- |
| content       | actual data      |
| totalElements | total rows in DB |
| totalPages    | total pages      |
| number        | current page     |
| size          | page size        |

---

---

# 🔹 5. Sorting (`Sort`)

---

## Definition

> Defines order of results

---

## Example Request

```text
GET /resources?sort=name,asc
```

---

## Meaning

```text
ORDER BY name ASC
```

---

## Multiple Sorting

```text
GET /resources?sort=name,asc&sort=id,desc
```

---

---

# 🔹 6. Combining Pagination + Sorting

---

## Request

```text
GET /resources?page=1&size=5&sort=name,desc
```

---

## SQL Generated

```sql
SELECT * FROM resource
ORDER BY name DESC
LIMIT 5 OFFSET 5;
```

---

---

# 🔹 7. Repository Integration

---

## Default Method

```java
Page<Resource> findAll(Pageable pageable);
```

---

## Custom Query

```java
Page<Resource> findByName(String name, Pageable pageable);
```

---

## Usage

```java
@GetMapping
public Page<Resource> search(
        @RequestParam String name,
        Pageable pageable) {

    return repository.findByName(name, pageable);
}
```

---

---

# 🔹 8. Pageable Defaults & Customization

---

## Default Values

```text
page = 0
size = 20
```

---

## Custom Defaults

```java
@GetMapping
public Page<Resource> get(
    @PageableDefault(size = 10, sort = "name") Pageable pageable) {
    return repository.findAll(pageable);
}
```

---

---

# 🔹 9. DTO Mapping (VERY IMPORTANT)

---

## Problem

Returning entity:

```java
Page<Resource>
```

❌ Exposes internal data

---

## Solution

```java
Page<ResourceResponse> result = repository.findAll(pageable)
    .map(entity -> new ResourceResponse(
        entity.getId(), entity.getName()
    ));
```

---

---

# 🔹 10. `Slice` vs `Page` (Advanced)

---

## Page

* includes total count
* extra query: `COUNT(*)`

---

## Slice

* no total count
* faster

---

## When to use Slice?

* infinite scroll
* large datasets

---

---

# 🔹 11. Performance Problems (Critical)

---

## Problem 1: Large OFFSET

```sql
OFFSET 100000
```

💥 DB scans huge data

---

## Solution: Keyset Pagination

Instead of:

```text
?page=1000
```

Use:

```text
?afterId=1000
```

---

---

## Problem 2: Sorting without index

```sql
ORDER BY name
```

💥 Full table scan

---

## Solution

* Add DB index

---

---

# 🔹 12. Validation & Security

---

## Problem

```text
size=100000
```

💥 DOS attack risk

---

## Solution

* limit size (e.g., max 100)

---

---

# 🔹 13. Custom API Response (Production)

---

## Problem

Default Page response is messy

---

## Better Design

```json
{
  "data": [...],
  "page": 1,
  "size": 5,
  "total": 100
}
```

---

## Implementation

```java
public class PageResponse<T> {

    private List<T> data;
    private int page;
    private int size;
    private long total;
}
```

---

---

# 🔹 14. Edge Cases

---

## 1. Page out of range

→ return empty list

---

## 2. Negative values

→ validate input

---

## 3. Invalid sort field

→ runtime error

---

---

# 🔹 15. Production Best Practices

---

✔ Always paginate
✔ Limit page size
✔ Use sorting
✔ Map to DTO
✔ Index sort columns
✔ Consider Slice for large data

---

---

# 🔹 Final Mental Model

```text
Pagination =
Control data size

Sorting =
Control order

Together =
Efficient + scalable APIs
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 Pagination & Sorting (Spring Boot)

---

## 🔹 Pagination

* Breaks large data into pages
* Uses Pageable

---

## 🔹 Pageable

* page → page number
* size → number of items
* sort → ordering

---

## 🔹 Page

* Contains:

  * content
  * totalElements
  * totalPages

---

## 🔹 Sorting

* ASC / DESC
* Multiple fields supported

---

## 🔹 Combining

GET /resources?page=1&size=5&sort=name,desc

---

## 🔹 Repository

* findAll(Pageable pageable)
* Custom queries support Pageable

---

## 🔹 Slice

* No total count
* Better performance

---

## 🔹 Performance

* Avoid large offsets
* Index sorting fields
* Limit page size

---

## 🔹 Best Practices

* Always paginate
* Use DTO mapping
* Validate inputs
* Optimize queries

---

## 🔹 Key Takeaways

* Pagination improves performance
* Sorting ensures consistency
* Pageable simplifies implementation
* Design for scalability

---

---

If you truly understood this, next step:

👉 **Filtering & Dynamic Queries (Specifications, Criteria API — real senior-level topic)**
