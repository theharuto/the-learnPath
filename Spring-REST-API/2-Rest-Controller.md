# 📦 Module: Spring REST Controllers (Deep Understanding)

---

# 🔹 Big Picture First (Story Style)

Imagine your system as a **processing machine**:

```text
Input (HTTP Request)
    ↓
Controller (entry gate)
    ↓
Processing layers (service, DB, etc.)
    ↓
Output (HTTP Response)
```

👉 The **controller is NOT the brain**
👉 It is the **gatekeeper / translator**

---

# 🔹 1. What is `@RestController`?

## Core Idea

> Marks a class as a component that handles HTTP requests and returns data directly (JSON)

---

## Mental Model

Think:

```text
Client sends request → Controller receives → returns data (not HTML)
```

---

## Code

```java
@RestController
@RequestMapping("/resources")
public class ResourceController {

    @GetMapping("/{id}")
    public Resource getResource(@PathVariable int id) {
        return new Resource(id, "sample");
    }
}
```

---

## What Actually Happens Internally (IMPORTANT)

When request comes:

```text
HTTP Request
   ↓
DispatcherServlet (central router)
   ↓
Find matching controller method
   ↓
Execute method
   ↓
Return Java object
   ↓
Jackson converts → JSON
   ↓
HTTP Response sent
```

---

## Why `@RestController` Matters

Without it:

```java
@Controller
```

Spring assumes:

```text
Return value = View name (HTML)
```

👉 That’s **wrong for APIs**

---

## Production Insight

* Always use `@RestController` for APIs
* Avoid mixing view-based controllers and REST controllers

---

---

# 🔹 2. URL Mapping (`@RequestMapping`, `@GetMapping`, etc.)

---

## Core Idea

> Map HTTP request (URL + method) → Java method

---

## Mental Model

```text
Incoming request:
GET /resources/10

System asks:
"Who handles this?"

→ Finds matching method
```

---

## Base Mapping

```java
@RequestMapping("/resources")
```

Now all endpoints start with:

```text
/resources
```

---

## Method-Level Mapping

```java
@GetMapping("/{id}")
```

Full path:

```text
GET /resources/{id}
```

---

## Shortcut vs Full Form

### Full form

```java
@RequestMapping(value="/{id}", method=RequestMethod.GET)
```

### Shortcut

```java
@GetMapping("/{id}")
```

✔ Always prefer shortcut

---

## Production Insight

### Bad API

```text
POST /getResource
```

### Good API

```text
GET /resources/{id}
```

👉 HTTP method must define behavior—not URL name

---

---

# 🔹 3. `@PathVariable` (Data from URL)

---

## Core Idea

> Extract dynamic value from URL path

---

## Mental Model

```text
URL: /resources/10
→ 10 is part of resource identity
→ Bind it to method parameter
```

---

## Code

```java
@GetMapping("/{id}")
public Resource get(@PathVariable int id) {
    return new Resource(id, "data");
}
```

---

## Internal Working

* Spring parses URL
* Matches `{id}`
* Uses **argument resolver**
* Converts string → int

---

## Edge Case

```java
@GetMapping("/{id}")
public Resource get(@PathVariable int id)
```

Request:

```text
/resources/abc
```

→ ❌ Type mismatch → 400 error

---

## Production Insight

Use `@PathVariable` when:

* Identifying **specific resource**

---

---

# 🔹 4. `@RequestParam` (Data from Query)

---

## Core Idea

> Extract values from query parameters

---

## Mental Model

```text
GET /resources?page=2&size=10
→ These are filters / modifiers
```

---

## Code

```java
@GetMapping
public List<Resource> list(
    @RequestParam int page,
    @RequestParam int size
) {
    return List.of();
}
```

---

## Difference vs PathVariable

| Use Case               | Annotation   |
| ---------------------- | ------------ |
| Resource identity      | PathVariable |
| Filtering / pagination | RequestParam |

---

## Optional Params

```java
@RequestParam(defaultValue = "0") int page
```

---

## Edge Case

Missing required param:

```text
GET /resources
```

→ ❌ 400 Bad Request

---

## Production Insight

* Always provide defaults for optional params
* Validate values (negative page, huge size)

---

---

# 🔹 5. `@RequestBody` (JSON → Object)

---

## Core Idea

> Convert request JSON into Java object

---

## Mental Model

```text
Client sends JSON
→ Spring converts it
→ You get ready-to-use object
```

---

## Code

```java
@PostMapping
public Resource create(@RequestBody Resource resource) {
    return resource;
}
```

---

## Request Example

```json
{
  "id": 1,
  "name": "sample"
}
```

---

## Internal Working (VERY IMPORTANT)

```text
HTTP Request Body
   ↓
HttpMessageConverter
   ↓
Jackson (JSON parser)
   ↓
Java Object created
```

---

## Failure Scenarios

### 1. Missing `@RequestBody`

```java
public Resource create(Resource resource)
```

→ ❌ resource = null

---

### 2. Invalid JSON

```json
{ "id": "abc" }
```

→ ❌ parsing error → 400

---

### 3. Missing fields

→ Object created but fields = null

---

## Production Insight

* Always validate request body (we’ll cover later)
* Never trust incoming JSON blindly

---

---

# 🔹 Putting Everything Together

---

## Example API

```java
@RestController
@RequestMapping("/resources")
public class ResourceController {

    @GetMapping("/{id}")
    public Resource get(@PathVariable int id) {
        return new Resource(id, "data");
    }

    @GetMapping
    public List<Resource> list(@RequestParam int page) {
        return List.of();
    }

    @PostMapping
    public Resource create(@RequestBody Resource resource) {
        return resource;
    }
}
```

---

## Full Flow

```text
Client → HTTP Request
       ↓
DispatcherServlet
       ↓
HandlerMapping
       ↓
Controller method
       ↓
Business logic (service layer ideally)
       ↓
Return object
       ↓
Jackson → JSON
       ↓
Response
```

---

---

# 🔹 Production-Level Thinking

---

## 1. Controllers must be THIN

### ❌ Bad

```java
@GetMapping
public void process() {
    // DB calls
    // business logic
}
```

---

### ✔ Good

```java
@GetMapping
public Resource get() {
    return service.get();
}
```

---

## 2. Validation is missing here (danger)

* Invalid data enters system
* Leads to:

  * DB corruption
  * runtime errors

---

## 3. Error Handling (not yet covered)

Without proper handling:

* Users get raw exceptions
* Poor API experience

---

## 4. Scalability Behavior

With 1000+ users:

* Controllers must be:

  * Stateless
  * Fast
* Heavy logic → move to service layer

---

---

# 🔹 Common Mistakes (Real World)

1. Mixing controller + business logic
2. Using wrong HTTP methods
3. Forgetting `@RequestBody`
4. Not handling invalid input
5. Returning internal entities directly

---

---

# 🔹 Final Mental Model (Very Important)

Think of controller as:

```text
Translator Layer

HTTP → Java → HTTP
```

NOT:

```text
Business Logic Layer ❌
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 Spring REST Controllers

---

## 🔹 Role of Controller

* Entry point for HTTP requests
* Maps URL → method
* Converts request/response

---

## 🔹 @RestController

* Combines @Controller + @ResponseBody
* Returns JSON instead of views

---

## 🔹 Request Mapping

### Base Mapping

* @RequestMapping("/resources")

### Method Mapping

* @GetMapping
* @PostMapping
* @PutMapping
* @DeleteMapping

---

## 🔹 @PathVariable

* Extracts values from URL
* Used for resource identification

Example:
GET /resources/{id}

---

## 🔹 @RequestParam

* Extracts query parameters
* Used for filtering/pagination

Example:
GET /resources?page=1

---

## 🔹 @RequestBody

* Converts JSON → Java object
* Uses Jackson internally

---

## 🔹 Internal Flow

Request → DispatcherServlet → Controller → Object → JSON → Response

---

## 🔹 Best Practices

* Keep controllers thin
* Use service layer for logic
* Follow REST conventions
* Validate inputs

---

## 🔹 Common Mistakes

* Using @Controller instead of @RestController
* Missing @RequestBody
* Mixing business logic
* Poor API design

---

## 🔹 Key Takeaways

* Controller = translation layer
* Spring handles routing and conversion
* Proper annotations are critical
