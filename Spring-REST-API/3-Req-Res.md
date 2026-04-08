# 📦 Module: Request & Response Handling (Spring REST)

---

# 🔹 Big Picture (Story Model)

Think of your API like a **pipeline**:

```text
Incoming HTTP Request
   ↓
Deserialize (JSON → Java)
   ↓
Controller Method
   ↓
Process (service layer)
   ↓
Serialize (Java → JSON/XML)
   ↓
HTTP Response (status + headers + body)
```

👉 You control:

* Input mapping
* Output structure
* HTTP semantics

---

# 🔹 1. Handling Request Body (`@RequestBody`)

---

## Short Explanation

> Converts HTTP request body (JSON/XML) → Java object

---

## Mental Model

```text
Client sends structured data
→ Spring parses it
→ You get a ready object
```

---

## Code

```java
@PostMapping("/resources")
public Resource create(@RequestBody Resource resource) {
    return resource;
}
```

---

## Input Example

```json
{
  "id": 1,
  "name": "sample"
}
```

---

## Internal Working (CRITICAL)

```text
HTTP Request Body
   ↓
HttpMessageConverter
   ↓
Jackson (ObjectMapper)
   ↓
Java Object
```

---

## Important Details (from Spring internals)

* Uses **HttpMessageConverter**
* Default JSON converter → `MappingJackson2HttpMessageConverter`
* Uses **Jackson ObjectMapper**

---

## Edge Cases

### 1. Missing `@RequestBody`

```java
public Resource create(Resource resource)
```

→ ❌ Object = null

---

### 2. Invalid JSON

```json
{ "id": "abc" }
```

→ ❌ `HttpMessageNotReadableException`

---

### 3. Partial Data

```json
{ "name": "A" }
```

→ Object created but missing fields = null

---

## Production Insight

* Never trust input
* Always validate (next module)
* Handle malformed requests gracefully

---

---

# 🔹 2. Returning JSON / XML (Serialization)

---

## Short Explanation

> Converts Java object → HTTP response body (JSON/XML)

---

## Mental Model

```text
Controller returns object
→ Spring converts it
→ Sends as JSON/XML
```

---

## Code

```java
@GetMapping("/{id}")
public Resource get() {
    return new Resource(1, "data");
}
```

---

## Output (JSON)

```json
{
  "id": 1,
  "name": "data"
}
```

---

## Internal Working

```text
Java Object
   ↓
HttpMessageConverter
   ↓
Jackson
   ↓
JSON Response
```

---

https://github.com/theharuto/the-learnPath/blob/main/Spring-REST-API/Misc-Return-JSON-or-XML-based-on-Client-req.md

## XML Support

If configured:

```text
Accept: application/xml
```

→ Spring can return XML (via Jackson XML or JAXB)

---

## Production Insight

* JSON = default (most used)
* XML only when required (legacy systems)

---

## Common Mistakes

❌ Returning entities directly (security risk)
❌ Exposing internal fields
❌ Large nested objects → performance issues

---

---

# 🔹 3. `ResponseEntity` (Full Control)

---

## Short Explanation

> Wrapper to control:

* Status code
* Headers
* Body

---

## Mental Model

```text
Instead of:
return data

You say:
return response(status + headers + body)
```

---

## Code

```java
@GetMapping("/{id}")
public ResponseEntity<Resource> get() {
    Resource res = new Resource(1, "data");
    return ResponseEntity.ok(res);
}
```

---

## Common Variants

```java
return ResponseEntity.ok(data);                 // 200
return ResponseEntity.status(201).body(data);  // 201 Created
return ResponseEntity.noContent().build();     // 204
return ResponseEntity.notFound().build();      // 404
```

---

## Internal Working

* Spring sees `ResponseEntity`
* Extracts:

  * status
  * headers
  * body
* Builds HTTP response manually

---

## Production Insight (VERY IMPORTANT)

Without `ResponseEntity`:

```java
return data;
```

→ Always returns **200 OK**

❌ Wrong for:

* errors
* creation
* deletion

---

## Correct Usage

| Scenario    | Status |
| ----------- | ------ |
| Success     | 200    |
| Created     | 201    |
| Deleted     | 204    |
| Not found   | 404    |
| Bad request | 400    |

---

---

# 🔹 4. Adding Custom Headers

---

## Short Explanation

> Add metadata to HTTP response

---

## Mental Model

```text
Response = body + headers + status
```

---

## Code

```java
@GetMapping("/{id}")
public ResponseEntity<Resource> get() {
    return ResponseEntity.ok()
        .header("X-Custom-Header", "value")
        .body(new Resource(1, "data"));
}
```

---

## Common Headers

| Header        | Purpose         |
| ------------- | --------------- |
| Content-Type  | Response format |
| Authorization | Security        |
| Cache-Control | Caching         |
| Location      | Resource URL    |

---

## Example: Resource Creation

```java
@PostMapping
public ResponseEntity<Resource> create(@RequestBody Resource resource) {
    return ResponseEntity
        .status(201)
        .header("Location", "/resources/" + resource.getId())
        .body(resource);
}
```

---

## Production Insight

Headers are used for:

* Caching
* Security
* Pagination metadata
* Rate limiting

---

---

# 🔹 5. Full Request Lifecycle (End-to-End)

---

## Flow

```text
Client sends HTTP request
   ↓
DispatcherServlet
   ↓
HandlerMapping
   ↓
Controller method
   ↓
@RequestBody → object conversion
   ↓
Business logic
   ↓
Return object / ResponseEntity
   ↓
HttpMessageConverter
   ↓
JSON/XML response
```

---

---

# 🔹 Production-Level Thinking

---

## 1. Always Control Status Codes

❌ Default 200 everywhere → bad API
✔ Use `ResponseEntity`

---

## 2. Input Validation Missing (Danger)

* Wrong data enters system
* Leads to bugs + corruption

---

## 3. Large Payload Handling

* Big JSON → memory + latency issues
* Use pagination / streaming

---

## 4. Error Handling (Critical)

Without it:

* Stack traces leak
* Bad UX

---

---

# 🔹 Common Mistakes

1. Not using `ResponseEntity`
2. Returning wrong status codes
3. Not validating request body
4. Exposing internal models
5. Ignoring headers

---

---

# 🔹 Final Mental Model

```text
Request Handling = Deserialization
Response Handling = Serialization + HTTP control
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 Request & Response Handling (Spring REST)

---

## 🔹 Request Handling

### @RequestBody

* Converts JSON → Java object
* Uses Jackson internally
* Requires valid JSON

---

## 🔹 Response Handling

* Java object → JSON
* Handled by HttpMessageConverter

---

## 🔹 ResponseEntity

* Full control over response
* Controls:

  * Status
  * Headers
  * Body

---

### Common Status Usage

* 200 → OK
* 201 → Created
* 204 → No Content
* 400 → Bad Request
* 404 → Not Found

---

## 🔹 Custom Headers

* Add metadata to response
* Example:

  * Location
  * Cache-Control
  * Custom headers

---

## 🔹 Internal Flow

Request → Deserialize → Controller → Serialize → Response

---

## 🔹 Best Practices

* Always use proper status codes
* Validate request body
* Keep responses clean
* Use headers when needed

---

## 🔹 Common Mistakes

* Always returning 200
* Missing @RequestBody
* Not handling invalid JSON
* Exposing internal data

---

## 🔹 Key Takeaways

* Request = JSON → Object
* Response = Object → JSON
* ResponseEntity gives full control
* HTTP semantics matter
