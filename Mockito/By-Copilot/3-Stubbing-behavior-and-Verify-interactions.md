# ✅ 5. Stubbing Behavior (IN DEPTH)

***

## 5.1 What is *Stubbing* in Mockito?

### ✅ Simple definition

**Stubbing** means:

> “Telling a mock **how to behave** when a specific method is called.”

Without stubbing:

*   Mockito mocks do nothing
*   They return **default values** (`null`, `0`, `false`)

Stubbing allows you to:

*   Control execution paths
*   Simulate success or failure
*   Isolate business logic

***

## 5.2 `when(...).thenReturn(...)` — defining return behavior

***

### ✅ Basic syntax

```java
when(mock.method(args)).thenReturn(value);
```

### ✅ Meaning (VERY IMPORTANT)

> “When this method is called with these arguments, return this value.”

***

### ✅ Example (most common case)

#### Production code

```java
class UserService {
    private final UserRepository repository;

    UserService(UserRepository repository) {
        this.repository = repository;
    }

    User getUser(Long id) {
        return repository.findById(id);
    }
}
```

#### Test code with stubbing

```java
@Mock
UserRepository repository;

@InjectMocks
UserService userService;

@Test
void shouldReturnUser() {
    User user = new User(1L);

    when(repository.findById(1L)).thenReturn(user);

    User result = userService.getUser(1L);

    assertEquals(1L, result.getId());
}
```

✅ `repository.findById(1L)` does **not** hit a database  
✅ Method returns exactly what you defined

***

### ✅ Argument matching in stubbing

```java
when(repository.findById(anyLong()))
        .thenReturn(user);
```

Meaning:

> “Return `user` for **any** long value.”

⚠️ **Rule (important)**  
If you use **one matcher**, you must use matchers for **all arguments**.

✅ Correct:

```java
when(repo.find(anyLong(), eq("ACTIVE"))).thenReturn(user);
```

❌ Incorrect (interview trap):

```java
when(repo.find(anyLong(), "ACTIVE")).thenReturn(user);
```

***

## 5.3 Throwing exceptions with `thenThrow()`

Stubbing is not just for success — **failure paths are critical**.

***

### ✅ Syntax

```java
when(mock.method()).thenThrow(new RuntimeException());
```

***

### ✅ Example: simulate DB failure

#### Production code

```java
void deleteUser(Long id) {
    repository.deleteById(id);
}
```

#### Test code

```java
@Test
void shouldThrowExceptionWhenDeleteFails() {
    when(repository.deleteById(1L))
        .thenThrow(new RuntimeException("DB is down"));

    assertThrows(RuntimeException.class, () -> {
        userService.deleteUser(1L);
    });
}
```

✅ No real database  
✅ Failure path tested  
✅ Business logic isolated

***

### ✅ When should you use `thenThrow()`?

Use it when:

*   Dependency failures must be handled
*   Testing error scenarios
*   Verifying retry / exception logic

***

## 5.4 `thenAnswer()` — custom dynamic behavior (ADVANCED but IMPORTANT)

***

### ✅ Why `thenAnswer()` exists

`thenReturn()` is **static** (always same value).  
Sometimes you need **dynamic behavior**, like:

*   Return value based on arguments
*   Mutate input
*   Simulate complex logic
*   Count calls

This is where `thenAnswer()` is used.

***

### ✅ Syntax

```java
when(mock.method(any()))
    .thenAnswer(invocation -> {
        // custom logic
        return value;
    });
```

***

### ✅ Example: return argument value

```java
when(repository.save(any(User.class)))
    .thenAnswer(invocation -> invocation.getArgument(0));
```

Meaning:

> “Return whatever user was passed to `save()`.”

***

### ✅ Full example

```java
@Test
void thenAnswerExample() {
    when(repository.save(any(User.class)))
        .thenAnswer(invocation -> invocation.getArgument(0));

    User saved = userService.createUser(new User(99L));

    assertEquals(99L, saved.getId());
}
```

✅ Very useful for:

*   Builders
*   Mappers
*   Save‑and‑return methods

***

### ✅ Key takeaway (INTERVIEW READY)

*   `thenReturn()` → fixed response
*   `thenThrow()` → fixed exception
*   `thenAnswer()` → dynamic logic

***

# ✅ 6. Verifying Interactions (REVISITED & REINFORCED)

You’ve seen verification already — now we **connect it with stubbing properly**.

***

## 6.1 What verification REALLY checks

Verification answers behavioral questions:

*   Was this method called?
*   How many times?
*   With which arguments?
*   Was it *never* called?

Stubbing controls **what happens**  
Verification checks **what happened**

***

## 6.2 `verify()` — ensure method was invoked

***

### ✅ Basic use

```java
verify(repository).findById(1L);
```

Meaning:

> “This method **must have been called at least once**.”

***

### ✅ Full example

```java
@Test
void verifyMethodCall() {
    when(repository.findById(1L)).thenReturn(new User(1L));

    userService.getUser(1L);

    verify(repository).findById(1L);
}
```

✅ Passes if called  
❌ Fails if not called

***

## 6.3 Verifying number of invocations

***

### ✅ Exactly `n` times

```java
verify(repository, times(2)).findById(1L);
```

***

### ✅ Never called

```java
verify(repository, never()).delete(any());
```

***

### ✅ At least once

```java
verify(repository, atLeastOnce()).findById(anyLong());
```

***

### ✅ At least `n` times

```java
verify(repository, atLeast(3)).findById(anyLong());
```

***

## 6.4 Verifying arguments (with matchers)

***

### ✅ Exact argument match

```java
verify(repository).findById(1L);
```

***

### ✅ Flexible match

```java
verify(repository).findById(anyLong());
```

***

### ✅ Combine count + argument

```java
verify(repository, times(1)).findById(eq(1L));
```

✅ This is extremely common in interviews

***

## 6.5 Verifying no further interactions

***

### ✅ `verifyNoMoreInteractions()`

```java
verify(repository).findById(1L);
verifyNoMoreInteractions(repository);
```

Meaning:

> “No other methods should be called on this mock.”

⚠️ **Senior advice**  
Avoid overuse — it makes tests brittle.

***

## ✅ Stubbing vs Verification — CLEAR DIFFERENCE

| Aspect    | Stubbing          | Verification    |
| --------- | ----------------- | --------------- |
| Purpose   | Controls behavior | Checks behavior |
| Focus     | Return values     | Interactions    |
| When used | Before execution  | After execution |
| Keyword   | `when()`          | `verify()`      |

***

## ✅ Interview‑Perfect One‑Line Summary

> **Stubbing defines how mocked methods behave using `when()` variants, while verification checks how mocked dependencies were used using `verify()` and invocation count matchers.**

***

## 🔜 Next Topics (STRONGLY RECOMMENDED)

1️⃣ `doReturn()`, `doThrow()` vs `when()` (VERY IMPORTANT)  
2️⃣ Verifying with `ArgumentCaptor` (more complex cases)  
3️⃣ Spies vs Mocks (CRITICAL interview topic)  
4️⃣ Mockito + Spring Boot (`@MockBean` vs `@Mock`)
