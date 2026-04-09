# 📦 Module: REST Validation (Spring + Hibernate Validator)

---

# 🔹 Big Picture (Story Model)

Think of request flow:

```text
Client → sends data
       ↓
Validation layer (gatekeeper)
       ↓
Controller logic
       ↓
DB
```

👉 Key rule:

> **Invalid data should NEVER reach your business logic**

---

# 🔹 1. What is Validation?

## Short Explanation

> Ensuring incoming data satisfies defined constraints before processing

---

## Mental Model

```text
Input → Check rules → Accept OR Reject
```

---

## Types

* Structural (null, size, format)
* Business (custom rules)

---

---

# 🔹 2. Hibernate Validator (JSR-380 / Jakarta Validation)

---

## What it is

* Standard validation framework
* Spring integrates it automatically via:

```xml
spring-boot-starter-validation
```

---

## Internal Flow

```text
@RequestBody → DTO
   ↓
Validator (Hibernate)
   ↓
If invalid → Exception thrown
```

---

---

# 🔹 3. Core Validation Annotations (MUST KNOW)

---

## `@NotNull`

```java
@NotNull
private String name;
```

✔ Value must not be null
❌ Empty string still allowed

---

---

## `@Size`

```java
@Size(min = 3, max = 10)
private String name;
```

✔ Checks length (String, Collection)

---

---

## `@Email`

```java
@Email
private String email;
```

✔ Valid email format

---

---

## Other Important Ones (Production)

| Annotation          | Purpose                          |
| ------------------- | -------------------------------- |
| `@NotBlank`         | Not null + not empty + no spaces |
| `@NotEmpty`         | Not null + not empty             |
| `@Min` / `@Max`     | Numeric range                    |
| `@Pattern`          | Regex validation                 |
| `@Past` / `@Future` | Date validation                  |

---

## 🔥 Important Distinction

| Annotation | Allows empty string?     |
| ---------- | ------------------------ |
| @NotNull   | ✅ Yes                    |
| @NotEmpty  | ❌ No                     |
| @NotBlank  | ❌ No (also trims spaces) |

👉 Interview trap

---

---

# 🔹 4. DTO Validation Example

---

```java
public class CreateUserRequest {

    @NotNull(message = "Name is required")
    @Size(min = 3, message = "Name must be at least 3 characters")
    private String name;

    @Email(message = "Invalid email format")
    private String email;
}
```

---

---

# 🔹 5. Activating Validation (`@Valid`)

---

## Short Explanation

> Triggers validation on method parameter

---

## Code

```java
@PostMapping
public ResponseEntity<?> create(
        @Valid @RequestBody CreateUserRequest request) {

    return ResponseEntity.ok("Success");
}
```

---

## Internal Working

```text
@RequestBody → DTO
   ↓
@Valid triggers validation
   ↓
If invalid → MethodArgumentNotValidException
```

---

## Important

❌ Without `@Valid` → no validation happens

---

---

# 🔹 6. Validation Failure Flow

---

## What happens?

```text
Invalid request
   ↓
Validation fails
   ↓
Exception thrown
   ↓
Global handler catches
   ↓
Return 400 response
```

---

---

# 🔹 7. Nested Validation

---

## Example

```java
public class Address {

    @NotBlank
    private String city;
}
```

```java
public class UserRequest {

    @Valid
    private Address address;
}
```

---

## Key Point

👉 Must use `@Valid` on nested field

---

---

# 🔹 8. Custom Validation (`@Constraint`)

---

## When needed?

> When built-in annotations are not enough

---

## Example Scenario

* Field must follow custom logic
* Cross-field validation

---

---

## Step 1: Create Annotation

```java
@Constraint(validatedBy = CustomValidator.class)
@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidCode {

    String message() default "Invalid code";

    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

---

---

## Step 2: Create Validator

```java
public class CustomValidator implements ConstraintValidator<ValidCode, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value != null && value.startsWith("ABC");
    }
}
```

---

---

## Step 3: Use It

```java
@ValidCode
private String code;
```

---

---

## Internal Working

```text
Field → Annotation
   ↓
Validator invoked
   ↓
true/false
```

---

---

# 🔹 9. Validation Groups (Advanced)

---

## Use Case

Different validation rules for:

* Create
* Update

---

## Example

```java
public interface CreateGroup {}
public interface UpdateGroup {}
```

```java
@NotNull(groups = UpdateGroup.class)
private Long id;
```

---

## Controller

```java
public ResponseEntity<?> update(
    @Validated(UpdateGroup.class) @RequestBody DTO dto)
```

---

---

# 🔹 10. Production-Level Concerns

---

## 1. Validation ≠ Business Logic

❌ Don’t do:

```java
if (balance < 0)
```

👉 That’s business rule

---

## 2. Fail Fast

* Reject invalid requests early
* Save DB calls

---

## 3. Consistent Error Response

* Always return structured errors

---

## 4. Avoid Over-validation

* Too strict → poor UX
* Too loose → bad data

---

---

# 🔹 11. Performance Considerations

---

* Validation happens per request
* Complex validators → slower
* Nested validation → heavier

👉 Keep it efficient

---

---

# 🔹 12. Common Mistakes

---

❌ Forgetting `@Valid`
❌ Using `@NotNull` instead of `@NotBlank`
❌ Not validating nested objects
❌ Mixing validation with business logic
❌ Not handling validation exceptions

---

---

# 🔹 Final Mental Model

```text
Validation =
Input gatekeeper

DTO → Validate → Process
```

---

---

# 📘 GitHub Notes (Markdown)

---

# 📘 REST Validation (Spring Boot)

---

## 🔹 Validation

* Ensures incoming data is correct
* Prevents invalid data from reaching business logic

---

## 🔹 Hibernate Validator

* Standard validation framework
* Integrated via Spring Boot

---

## 🔹 Core Annotations

* @NotNull → must not be null
* @NotEmpty → not null + not empty
* @NotBlank → not null + not blank
* @Size → length constraint
* @Email → valid email

---

## 🔹 @Valid

* Triggers validation on request body
* Used with @RequestBody

---

## 🔹 Validation Flow

Request → DTO → Validation → Exception → Response

---

## 🔹 Nested Validation

* Use @Valid on nested objects

---

## 🔹 Custom Validation

* Use @Constraint
* Implement ConstraintValidator

---

## 🔹 Validation Groups

* Different rules for different operations

---

## 🔹 Best Practices

* Validate all inputs
* Use DTOs
* Keep validation separate from business logic
* Return structured error responses

---

## 🔹 Common Mistakes

* Missing @Valid
* Wrong annotation usage
* Ignoring nested validation
* Mixing business logic

---

## 🔹 Key Takeaways

* Validation is first defense layer
* Prevents bad data
* Improves API reliability
