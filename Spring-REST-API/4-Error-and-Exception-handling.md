# 📦 Module: Error & Exception Handling (Spring REST)

---

# 🔹 Big Picture (Story Model)

Think of your API as:

```text
Request → Processing → Response
```

Now imagine failures:

```text
Request → Processing → ❌ Exception → ???
```

👉 Your job:

> Convert **internal failures → meaningful HTTP responses**

---

# 🔹 1. Types of Errors in REST APIs

---

## 1. Client Errors (4xx)

* Invalid input
* Missing fields
* Wrong format

---

## 2. Server Errors (5xx)

* Null pointer
* DB failure
* Unexpected crash

---

## Mental Model

```text
Client mistake → 4xx
Server mistake → 5xx
```

---

---

# 🔹 2. What Happens Without Handling?

---

## Default Behavior (Spring)

```text
Exception → Spring catches → returns 500
```

Response:

```json
{
  "timestamp": "...",
  "status": 500,
  "error": "Internal Server Error"
}
```

---

## Problem

❌ Not user-friendly
❌ Not consistent
❌ Leaks internal info sometimes

---

---

# 🔹 3. Local Exception Handling (`@ExceptionHandler`)

---

## Short Explanation

> Handles exceptions inside a controller

---

## Code

```java
@RestController
@RequestMapping("/resources")
public class ResourceController {

    @GetMapping("/{id}")
    public Resource get(@PathVariable int id) {
        throw new RuntimeException("Something went wrong");
    }

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handle(RuntimeException ex) {
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ex.getMessage());
    }
}
```

---

## Problem

❌ Repeated in every controller
❌ Not scalable

---

---

# 🔹 4. Global Exception Handling (`@RestControllerAdvice`)

---

## Short Explanation

> Centralized error handling across all controllers

---

## Mental Model

```text
Any exception anywhere
→ handled in one place
```

---

## Code

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handleRuntime(RuntimeException ex) {
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ex.getMessage());
    }
}
```

---

## Internal Working

* Spring scans `@RestControllerAdvice`
* Registers handlers
* Matches exception → method

---

## Production Insight

✔ Always use global handler
✔ Keep controllers clean

---

---

# 🔹 5. Custom Exceptions (VERY IMPORTANT)

---

## Why?

> Generic exceptions = poor API design

---

## Example

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String msg) {
        super(msg);
    }
}
```

---

## Use in Controller

```java
if (resource == null) {
    throw new ResourceNotFoundException("Resource not found");
}
```

---

## Handle Globally

```java
@ExceptionHandler(ResourceNotFoundException.class)
public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ex.getMessage());
}
```

---

---

# 🔹 6. Validation (`@Valid`) – Input Safety

---

## Short Explanation

> Automatically validates request body using annotations

---

## DTO Example

```java
public class CreateRequest {

    @NotNull
    private String name;

    @Size(min = 3)
    private String description;
}
```

---

## Controller

```java
@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody CreateRequest request) {
    return ResponseEntity.ok("Created");
}
```

---

## Internal Working

```text
JSON → DTO
   ↓
Validation (Hibernate Validator)
   ↓
If fail → exception thrown
```

---

## Exception Type

```text
MethodArgumentNotValidException
```

---

---

# 🔹 7. Handling Validation Errors

---

## Global Handler

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, String>> handleValidation(
        MethodArgumentNotValidException ex) {

    Map<String, String> errors = new HashMap<>();

    ex.getBindingResult().getFieldErrors()
        .forEach(error -> errors.put(
            error.getField(),
            error.getDefaultMessage()
        ));

    return ResponseEntity.badRequest().body(errors);
}
```

---

## Response Example

```json
{
  "name": "must not be null",
  "description": "size must be at least 3"
}
```

---

## Production Insight

✔ Always validate input
✔ Return structured error response

---

---

# 🔹 8. Standardizing Error Response (VERY IMPORTANT)

---

## Problem

Different errors → different formats ❌

---

## Solution: Common Error Model

```java
public class ErrorResponse {

    private int status;
    private String message;
    private long timestamp;
}
```

---

## Usage

```java
return ResponseEntity
    .status(HttpStatus.NOT_FOUND)
    .body(new ErrorResponse(404, "Not found", System.currentTimeMillis()));
```

---

## Benefit

* Consistent API
* Easy for frontend
* Better debugging

---

---

# 🔹 9. Mapping Exceptions → HTTP Status

---

## Standard Mapping

| Exception          | Status |
| ------------------ | ------ |
| Validation failure | 400    |
| Resource not found | 404    |
| Unauthorized       | 401    |
| Forbidden          | 403    |
| Conflict           | 409    |
| Unexpected error   | 500    |

---

---

# 🔹 10. Full Flow (Production)

```text
Request
   ↓
Validation (@Valid)
   ↓
Controller
   ↓
Exception thrown?
   ↓
Global handler
   ↓
Custom error response
   ↓
HTTP status + JSON
```

---

---

# 🔹 11. Production-Level Insights

---

## 1. Never expose stack traces

❌ Security risk

---

## 2. Always return meaningful messages

❌ "Something went wrong"
✔ "Resource with id 10 not found"

---

## 3. Log internally, not externally

* Log full exception
* Return clean message

---

## 4. Use structured error format

* Helps frontend
* Helps debugging

---

---

# 🔹 12. Common Mistakes

---

❌ Using try-catch in every controller
❌ Returning 200 for errors
❌ Not validating input
❌ Exposing internal exception messages
❌ No global handler

---

---

# 🔹 Final Mental Model

```text
Exception Handling =
Catch once (globally)
→ Translate to HTTP response
→ Maintain consistency
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 Error & Exception Handling (Spring REST)

---

## 🔹 Types of Errors

* 4xx → client errors
* 5xx → server errors

---

## 🔹 Exception Handling

### @ExceptionHandler

* Handles exceptions locally

### @RestControllerAdvice

* Global exception handling
* Recommended approach

---

## 🔹 Custom Exceptions

* Create domain-specific exceptions
* Map to proper HTTP status

---

## 🔹 Validation

### @Valid

* Validates request body

### Common Annotations

* @NotNull
* @Size
* @Min / @Max

---

## 🔹 Validation Exception

* MethodArgumentNotValidException

---

## 🔹 Error Response

* Use consistent structure
* Include:

  * status
  * message
  * timestamp

---

## 🔹 HTTP Status Mapping

* 400 → bad request
* 404 → not found
* 500 → server error

---

## 🔹 Best Practices

* Use global exception handler
* Validate all inputs
* Return meaningful messages
* Do not expose internal details

---

## 🔹 Key Takeaways

* Exceptions must be translated to HTTP responses
* Centralized handling is essential
* Validation is critical for API safety
* Consistent error format improves usability
