# 📦 Module: Implementing REST CRUD Operations

---

# 🔹 Big Picture (System Flow)

Think in layers:

```text
Client → Controller → Service → Repository → DB
                      ↑
                  DTO ↔ Entity
```

👉 CRUD is just:

> Controlled state transitions of a resource

---

# 🔹 1. CRUD = State Transitions

| Operation | Meaning        |
| --------- | -------------- |
| CREATE    | Add new state  |
| READ      | Retrieve state |
| UPDATE    | Modify state   |
| DELETE    | Remove state   |

---

---

# 🔹 2. CREATE (POST)

---

## Short Explanation

> Accept input → create new resource → return created state

---

## Mental Model

```text
Client sends data → system creates new record → assigns ID → returns it
```

---

## Controller Example

```java
@PostMapping
public ResponseEntity<ResourceResponse> create(
        @RequestBody CreateResourceRequest request) {

    Resource entity = new Resource();
    entity.setName(request.getName());

    Resource saved = repository.save(entity);

    ResourceResponse response = new ResourceResponse(
        saved.getId(), saved.getName()
    );

    return ResponseEntity
            .status(HttpStatus.CREATED)
            .header("Location", "/resources/" + saved.getId())
            .body(response);
}
```

---

## Internal Flow

```text
JSON → DTO → Entity → save() → DB insert → return entity → DTO → JSON
```

---

## Production Rules

✔ Always return **201 CREATED**
✔ Include **Location header**
✔ Never trust input blindly

---

## Common Mistakes

❌ Returning 200 instead of 201
❌ Not returning created resource
❌ No validation

---

---

# 🔹 3. READ (GET)

---

## Short Explanation

> Fetch resource(s) from system

---

## Single Resource

```java
@GetMapping("/{id}")
public ResponseEntity<ResourceResponse> get(@PathVariable Long id) {

    Resource entity = repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Not found"));

    return ResponseEntity.ok(
        new ResourceResponse(entity.getId(), entity.getName())
    );
}
```

---

## Multiple Resources

```java
@GetMapping
public List<ResourceResponse> list() {
    return repository.findAll().stream()
        .map(e -> new ResourceResponse(e.getId(), e.getName()))
        .toList();
}
```

---

## Production Rules

✔ Use **404 if not found**
✔ Avoid returning full entity graph
✔ Use pagination (later topic)

---

---

# 🔹 4. UPDATE (PUT vs PATCH)

---

## 🔹 PUT (Full Update)

---

### Short Explanation

> Replace entire resource

---

### Mental Model

```text
Existing state → completely overwritten
```

---

### Code

```java
@PutMapping("/{id}")
public ResponseEntity<ResourceResponse> update(
        @PathVariable Long id,
        @RequestBody UpdateRequest request) {

    Resource entity = repository.findById(id)
        .orElseThrow(() -> new RuntimeException());

    entity.setName(request.getName());

    Resource updated = repository.save(entity);

    return ResponseEntity.ok(
        new ResourceResponse(updated.getId(), updated.getName())
    );
}
```

---

## Important Rule

👉 Client must send **full data**

---

---

## 🔹 PATCH (Partial Update)

---

### Short Explanation

> Update only specific fields

---

### Mental Model

```text
Existing state → modify only provided fields
```

---

### Code

```java
@PatchMapping("/{id}")
public ResponseEntity<ResourceResponse> patch(
        @PathVariable Long id,
        @RequestBody UpdateRequest request) {

    Resource entity = repository.findById(id)
        .orElseThrow();

    if (request.getName() != null) {
        entity.setName(request.getName());
    }

    Resource updated = repository.save(entity);

    return ResponseEntity.ok(
        new ResourceResponse(updated.getId(), updated.getName())
    );
}
```

---

## Production Insight

* PATCH is tricky:

  * null vs absent fields problem
  * idempotency concerns

---

---

# 🔹 PUT vs PATCH (CRITICAL DIFFERENCE)

| Feature        | PUT          | PATCH          |
| -------------- | ------------ | -------------- |
| Type           | Full replace | Partial update |
| Idempotent     | ✅ Yes        | ⚠ Depends      |
| Missing fields | Overwritten  | Ignored        |

---

---

# 🔹 5. DELETE

---

## Short Explanation

> Remove resource

---

## Code

```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {

    Resource entity = repository.findById(id)
        .orElseThrow(() -> new RuntimeException());

    repository.delete(entity);

    return ResponseEntity.noContent().build();
}
```

---

## Production Rules

✔ Return **204 NO CONTENT**
✔ Must be idempotent

---

---

# 🔹 6. JpaRepository Integration

---

## What is it?

> Spring Data interface for DB operations

---

## Code

```java
public interface ResourceRepository
        extends JpaRepository<Resource, Long> {
}
```

---

---

## Methods You Get for Free

| Method     | Purpose       |
| ---------- | ------------- |
| save()     | insert/update |
| findById() | fetch by ID   |
| findAll()  | fetch all     |
| delete()   | remove        |

---

---

## Internal Working

```text
Repository method
   ↓
Spring Data proxy
   ↓
Hibernate
   ↓
SQL executed
```

---

---

# 🔹 7. Full CRUD Flow (End-to-End)

---

```text
POST → create → DB insert  
GET → read → DB fetch  
PUT/PATCH → update → DB update  
DELETE → remove → DB delete  
```

---

---

# 🔹 8. Production-Level Concerns

---

## 1. Transaction Management

* Updates must be atomic
* Use `@Transactional` in service layer

---

## 2. Concurrency Issues

* Lost updates
* Use optimistic locking (`@Version`)

---

## 3. Validation Required

* Prevent invalid data

---

## 4. DTO Mapping

* Never expose entity directly

---

## 5. Error Handling

* Return correct status codes

---

---

# 🔹 9. Common Mistakes

---

❌ Using POST for updates
❌ Returning wrong status codes
❌ Not handling null values in PATCH
❌ No exception handling
❌ Direct entity exposure

---

---

# 🔹 10. Final Mental Model

```text
CRUD =
Controlled transitions of resource state
```

---

```text
Controller = API contract
Repository = persistence
DTO = external view
Entity = internal storage
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 REST CRUD Operations (Spring Boot)

---

## 🔹 CRUD Operations

* POST → Create
* GET → Read
* PUT → Full update
* PATCH → Partial update
* DELETE → Remove

---

## 🔹 CREATE (POST)

* Creates new resource
* Returns 201 CREATED
* Include Location header

---

## 🔹 READ (GET)

* Fetch resource(s)
* Return 200 OK
* 404 if not found

---

## 🔹 UPDATE

### PUT

* Full replacement
* Idempotent

### PATCH

* Partial update
* Not always idempotent

---

## 🔹 DELETE

* Removes resource
* Returns 204 NO CONTENT
* Idempotent

---

## 🔹 JpaRepository

* Provides CRUD methods
* save(), findById(), findAll(), delete()

---

## 🔹 Flow

Controller → Service → Repository → DB

---

## 🔹 Best Practices

* Use DTOs
* Validate input
* Handle exceptions
* Use proper status codes

---

## 🔹 Common Mistakes

* Wrong HTTP methods
* Exposing entities
* No validation
* Incorrect status codes

---

## 🔹 Key Takeaways

* CRUD = resource state management
* Follow HTTP semantics
* Use repository for DB operations
* Design APIs carefully
