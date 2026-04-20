# 📦 Module 11: Spring REST Best Practices

---

# 🔹 0. Big Picture (Mental Model)

Think of your API as:

```text id="1"
A contract between systems
```

👉 If design is bad:

* Clients struggle
* Bugs increase
* Scaling becomes painful

---

# 🔹 1. Resource-Oriented Design (CORE PRINCIPLE)

---

## ❌ Wrong Thinking

```text id="2"
/getUser
/createOrder
/deleteItem
```

👉 Action-based (RPC style)

---

## ✅ Correct Thinking

```text id="3"
/users
/orders
/items
```

👉 Resource-based

---

## Rule

> URL = **noun (resource)**
> HTTP method = **action**

---

## Example

| Operation | Endpoint           |
| --------- | ------------------ |
| Get all   | GET /users         |
| Get one   | GET /users/{id}    |
| Create    | POST /users        |
| Update    | PUT /users/{id}    |
| Delete    | DELETE /users/{id} |

---

---

# 🔹 2. Proper Use of HTTP Methods

---

## GET

* Read-only
* No side effects
* Idempotent

---

## POST

* Create resource
* Not idempotent

---

## PUT

* Full update
* Idempotent

---

## PATCH

* Partial update

---

## DELETE

* Remove resource
* Idempotent

---

---

## 🔥 Critical Rule

```text id="4"
GET must NEVER modify data
```

---

## ❌ Anti-pattern

```text id="5"
GET /users/delete/10
```

💥 Dangerous:

* Browsers may cache GET
* Crawlers may trigger it

---

---

# 🔹 3. URI Design Best Practices

---

## 1. Use plural nouns

```text id="6"
/users ✔
/user ❌
```

---

## 2. Hierarchical structure

```text id="7"
/users/{id}/orders
```

---

## 3. Avoid verbs

```text id="8"
/createUser ❌
/users ✔
```

---

## 4. Use lowercase + hyphens

```text id="9"
/user-profiles ✔
```

---

---

# 🔹 4. Status Codes (Correct Usage)

---

| Scenario     | Status |
| ------------ | ------ |
| Success GET  | 200    |
| Created      | 201    |
| Deleted      | 204    |
| Not found    | 404    |
| Bad input    | 400    |
| Server error | 500    |

---

## 🔥 Anti-pattern

```text id="10"
Always returning 200 ❌
```

---

---

# 🔹 5. Idempotency (VERY IMPORTANT)

---

## Definition

> Multiple identical requests → same result

---

## Examples

| Method | Idempotent? |
| ------ | ----------- |
| GET    | ✅           |
| PUT    | ✅           |
| DELETE | ✅           |
| POST   | ❌           |

---

## Why it matters

```text id="11"
Retries happen in real systems
```

---

---

# 🔹 6. API Versioning

---

## Why needed

* Avoid breaking clients

---

## Common Approaches

### URL Versioning

```text id="12"
/v1/users
/v2/users
```

---

### Header Versioning

```text id="13"
Accept: application/v1+json
```

---

## Production Insight

👉 URL versioning is most common

---

---

# 🔹 7. Pagination, Filtering, Sorting (Recap)

---

## Standard pattern

```text id="14"
/users?page=0&size=10&sort=name,asc
```

---

## Rule

* Never return full dataset

---

---

# 🔹 8. Error Handling Best Practices

---

## Always return structured error

```json id="15"
{
  "status": 404,
  "message": "Resource not found"
}
```

---

## Avoid

❌ Stack traces
❌ Internal exceptions

---

---

# 🔹 9. DTO Usage (Recap)

---

## Rule

```text id="16"
Never expose entities directly
```

---

## Why

* Security
* Flexibility
* Performance

---

---

# 🔹 10. Richardson Maturity Model (Applied)

---

## Level 0

```text id="17"
/api
body: { action: "getUser" }
```

❌ Not REST

---

## Level 1

```text id="18"
/users
```

✔ Resources exist

---

## Level 2 (MOST IMPORTANT)

```text id="19"
GET /users
POST /users
PUT /users/{id}
```

✔ Proper HTTP usage

---

## Level 3 (HATEOAS)

```json id="20"
{
  "id": 1,
  "links": [
    { "rel": "self", "href": "/users/1" }
  ]
}
```

---

## Production Reality

👉 Most systems stop at Level 2

---

---

# 🔹 11. Common Anti-Patterns (CRITICAL)

---

## 1. Action-based URLs

```text id="21"
/getUsers ❌
```

---

## 2. Misusing HTTP methods

```text id="22"
GET /deleteUser ❌
```

---

## 3. No pagination

```text id="23"
GET /users → returns all ❌
```

---

## 4. Returning 200 for everything

---

## 5. Exposing internal models

---

---

# 🔹 12. Production-Level Design Rules

---

## Rule 1: Consistency

```text id="24"
/users
/orders
/products
```

---

## Rule 2: Predictability

* Same pattern everywhere

---

## Rule 3: Simplicity

* Avoid over-engineering

---

## Rule 4: Scalability

* Pagination
* Filtering

---

---

# 🔹 13. Real System Thinking

---

When designing API, always ask:

```text id="25"
1. What is the resource?
2. What operation?
3. What response?
4. What errors?
5. How will it scale?
```

---

---

# 🔹 Final Mental Model

```text id="26"
REST API Design =
Resources + HTTP semantics + consistency
```

---

```text id="27"
Good API =
Easy to understand + hard to misuse
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 Spring REST Best Practices

---

## 🔹 Resource Design

* Use nouns, not verbs
* Example: /users

---

## 🔹 HTTP Methods

* GET → read
* POST → create
* PUT → full update
* PATCH → partial update
* DELETE → remove

---

## 🔹 URI Design

* Use plural nouns
* Use hierarchy
* Avoid verbs

---

## 🔹 Status Codes

* 200 OK
* 201 CREATED
* 204 NO CONTENT
* 400 BAD REQUEST
* 404 NOT FOUND

---

## 🔹 Idempotency

* GET, PUT, DELETE → idempotent
* POST → not idempotent

---

## 🔹 Versioning

* /v1/users
* /v2/users

---

## 🔹 Pagination

* Use page, size, sort

---

## 🔹 Error Handling

* Return structured errors
* Avoid exposing internal details

---

## 🔹 DTO Usage

* Never expose entities
* Use DTOs for API

---

## 🔹 Richardson Model

* Level 2 is industry standard

---

## 🔹 Common Mistakes

* Using verbs in URLs
* Misusing HTTP methods
* No pagination
* Always returning 200

---

## 🔹 Key Takeaways

* Design APIs carefully
* Follow REST principles
* Maintain consistency
* Optimize for scalability

---
👉 **Filtering & Dynamic Queries (Specifications, Criteria API — real senior backend skill)**
