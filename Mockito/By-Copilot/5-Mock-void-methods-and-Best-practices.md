# ✅ 9. Mocking Void Methods in Mockito (IN DEPTH)

***

## 9.1 Why Void Methods Are “Special” in Mockito

### ✅ The core issue

A **void method**:

*   Does **not return anything**
*   Often performs **side effects**, such as:
    *   Saving to DB
    *   Sending email
    *   Writing to a file
    *   Publishing events

Because `when(...).thenReturn(...)` relies on a return value, **it cannot be used with void methods**.

❌ This is illegal:

```java
when(repo.save(entity)).thenReturn(...); // compile-time error if save() is void
```

✅ Mockito provides **`doXxx()` APIs** instead.

***

## 9.2 The `doXxx()` Family (Mental Model)

For **void methods**, Mockito uses:

```java
doSomething().when(mock).voidMethod(args);
```

### ✅ Why this syntax exists

*   `when(mock.method())` **invokes the real method**
*   With void methods, this can:
    *   Trigger real behavior
    *   Throw exceptions
    *   Modify state

So Mockito uses a **safe, non‑invoking syntax**.

***

## 9.3 `doNothing()` – Explicitly do nothing (default but useful)

### ✅ What it means

```java
doNothing().when(mock).voidMethod();
```

> “When this void method is called, do nothing.”

Strictly speaking, **mocks already do nothing by default**, but `doNothing()` is useful for:

*   Readability
*   Spies (very important)
*   Expressing intent

***

### ✅ Example: suppress side effects

#### Production code

```java
class EmailService {
    void sendEmail(String msg) {
        // real email sending
    }
}
```

#### Test code

```java
doNothing().when(emailService).sendEmail(anyString());

service.process(); // will call sendEmail internally
```

✅ No real email  
✅ Test remains isolated

***

## 9.4 `doThrow()` – Testing failure paths for void methods

This is **extremely common in production tests**.

***

### ✅ Syntax

```java
doThrow(new RuntimeException()).when(mock).voidMethod(args);
```

***

### ✅ Example: simulate failure

#### Production code

```java
void deleteUser(Long id) {
    repository.deleteById(id);
}
```

#### Test code

```java
doThrow(new RuntimeException("DB error"))
    .when(repository)
    .deleteById(1L);

assertThrows(RuntimeException.class, () -> {
    userService.deleteUser(1L);
});
```

✅ No real DB  
✅ Failure handling tested  
✅ Correct unit isolation

***

## 9.5 `doAnswer()` – Custom behavior for void methods (ADVANCED)

`doAnswer()` gives **full control**.

***

### ✅ Syntax

```java
doAnswer(invocation -> {
    // custom logic
    return null;
}).when(mock).voidMethod(args);
```

⚠️ Must return `null` because the method is `void`.

***

### ✅ Example: inspect arguments

```java
doAnswer(invocation -> {
    User user = invocation.getArgument(0);
    assertEquals("ACTIVE", user.getStatus());
    return null;
}).when(repository).save(any(User.class));
```

Use this when:

*   You must validate arguments inside void methods
*   You want conditional behavior
*   Side effects must be simulated

***

## 9.6 Void Methods + Spies (VERY IMPORTANT RULE)

When spying real objects:

❌ **Never use**

```java
when(spy.voidMethod()).thenThrow(...);
```

✅ **Always use**

```java
doThrow(...).when(spy).voidMethod();
```

This avoids invoking real logic during stubbing.

***

## ✅ Summary of Void Method Stubbing

| Method        | Purpose           |
| ------------- | ----------------- |
| `doNothing()` | Suppress behavior |
| `doThrow()`   | Simulate failure  |
| `doAnswer()`  | Custom logic      |

***

# ✅ 10. Mockito Best Practices (PRODUCTION + INTERVIEW)

These practices **separate junior tests from senior tests**.

***

## 10.1 Mock Selectively (DON’T mock everything)

### ❌ Bad practice

```java
@Mock
User entity;
```

Entities, DTOs, value objects:

*   Have no external dependencies
*   Are cheap to construct
*   Should be **real objects**

✅ Mock:

*   Repositories
*   External clients
*   Third‑party services

📌 **Interview line**

> *Mock dependencies, not data.*

***

## 10.2 Avoid Over‑Mocking

### ❌ Over‑mocked tests:

*   Verify every method call
*   Know too much about implementation
*   Break easily on refactor

✅ Good tests:

*   Verify **meaningful interactions**
*   Focus on **observable behavior**

***

### ✅ Example

❌ Bad:

```java
verify(repo).findById(1L);
verify(repo).save(any());
verify(repo).flush();
```

✅ Better:

```java
verify(repo).save(any());
```

***

## 10.3 Resetting Mocks (`reset()`) – Use Sparingly

### ✅ What `reset()` does

```java
reset(mock);
```

*   Clears stubbings
*   Clears interaction history

### ⚠️ Best practice

*   **Avoid reusing mocks across tests**
*   Prefer creating a **new test instance each time**

✅ Modern Mockito + JUnit makes reset rarely necessary.

***

## 10.4 Verify Only Meaningful Interactions

Ask yourself:

> “Does this interaction represent business behavior?”

✅ Yes → verify  
❌ No → don’t verify

Examples of meaningful interactions:

*   Sending email
*   Charging payment
*   Saving domain state

Examples to avoid:

*   Getter calls
*   Logging calls
*   Framework internals

***

## 10.5 Prefer `@InjectMocks` with Constructor Injection

### ✅ Why this is recommended

Mockito injection priority:

1.  Constructor
2.  Setter
3.  Field

Constructor injection:

*   Is explicit
*   Prevents `null` dependencies
*   Matches Spring best practices

***

### ✅ Example

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    PaymentService paymentService;

    @InjectMocks
    OrderService orderService;
}
```

No manual wiring  
No setup boilerplate  
Cleaner tests

📌 **Interview line**

> *Using `@InjectMocks` with constructor injection creates maintainable, readable tests aligned with production code.*

***

## ✅ Common Mockito Anti‑Patterns (INTERVIEW TRAPS)

| Anti‑Pattern            | Why Bad            |
| ----------------------- | ------------------ |
| Mocking everything      | Brittle tests      |
| Spying everywhere       | Side effects       |
| Verifying trivial calls | Over‑specification |
| Heavy reset() usage     | Design smell       |
| Mocking entities        | Unnecessary        |

***

## ✅ Final One‑Line Summary (MEMORIZE)

> **Void methods are stubbed using `doXxx()` APIs, and good Mockito tests mock only external dependencies, verify meaningful behavior, avoid over‑specification, and prefer constructor‑based injection with `@InjectMocks`.**

***

## 🔜 Next Mockito Topics (RECOMMENDED NEXT STEPS)

1️⃣ `@Mock` vs `@MockBean` (CRITICAL for Spring Boot)  
2️⃣ Mockito + Spring Boot testing layers  
3️⃣ Strict stubbing & UnnecessaryStubbingException  
4️⃣ Mockito vs Integration Tests (when NOT to mock)
