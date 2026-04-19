# 📘 Mockito – Complete Revision Notes

*(Production‑Ready + Interview‑Focused)*

***

## 1️⃣ What is Mockito?

### ✅ Definition

**Mockito** is a **Java mocking framework** used in **unit testing** to:

*   Replace real dependencies with **fake (mocked) objects**
*   Test a class **in isolation**
*   Avoid calling real:
    *   Databases
    *   APIs
    *   File systems
    *   Message queues

### ✅ Core Goal

> **Test business logic without external side effects**

***

## 2️⃣ Why Mockito is Used (VERY IMPORTANT)

### ✅ Problems Mockito Solves

| Without Mockito       | With Mockito        |
| --------------------- | ------------------- |
| Requires DB/API       | No external systems |
| Slow tests            | Fast tests          |
| Flaky tests           | Deterministic tests |
| Hard to isolate logic | Pure unit tests     |

### ✅ Real‑World Principle

> A **unit test must not depend on infrastructure**.

***

## 3️⃣ Test Doubles – Terminology You MUST Know

### ✅ Test Double (Parent Term)

Any fake object used instead of a real dependency.

Includes:

*   Mock
*   Stub
*   Spy
*   Fake

***

### ✅ Mock

*   Verifies **interactions**
*   Can:
    *   Stub behavior
    *   Verify method calls
*   Test can **fail if interaction didn’t happen**

✅ Used for **behavior verification**

***

### ✅ Stub

*   Returns **predefined data**
*   Does NOT verify interactions
*   Cannot fail test by itself

✅ Used for **state/output control**

***

### ✅ Spy

*   Wraps a **real object**
*   Calls **real methods by default**
*   Allows selective mocking

✅ Used for **partial mocking**

***

## 4️⃣ Creating Mocks

### ✅ Programmatic Creation

```java
MyService service = Mockito.mock(MyService.class);
```

*   No real logic
*   No constructor execution
*   Returns default values unless stubbed

***

### ✅ Annotation‑Based (Preferred)

```java
@Mock
UserRepository repository;
```

Equivalent to `Mockito.mock()` but cleaner.

⚠️ Requires:

```java
@ExtendWith(MockitoExtension.class)
```

***

## 5️⃣ Dependency Injection in Tests – `@InjectMocks`

### ✅ What `@InjectMocks` Does

*   Creates **real object** of class under test
*   Injects available mocks into it
*   Injection order:
    1.  Constructor
    2.  Setter
    3.  Field

✅ No Spring context  
✅ No `@Autowired`  
✅ Pure Mockito logic

***

### ✅ Example

```java
@Mock
UserRepository repository;

@InjectMocks
UserService userService;
```

***

## 6️⃣ Stubbing Behavior (CORE CONCEPT)

### ✅ What is Stubbing?

> Defining **how a mock behaves** when a method is called.

***

### ✅ `when().thenReturn()`

```java
when(repo.findById(1L)).thenReturn(user);
```

Meaning:

> When this method is called with these args, return this value.

***

### ✅ `thenThrow()` – Failure Simulation

```java
when(repo.findById(1L))
    .thenThrow(new RuntimeException("DB Error"));
```

✅ Used for testing error paths.

***

### ✅ `thenAnswer()` – Dynamic Behavior

```java
when(repo.save(any()))
    .thenAnswer(invocation -> invocation.getArgument(0));
```

✅ Used when response depends on input.

***

### ✅ Stubbing Rules (IMPORTANT)

*   Stubbing must be done **before invocation**
*   If matchers are used → use matchers for **all arguments**

***

## 7️⃣ Verifying Interactions

### ✅ What Verification Means

> Checking **how dependencies were used**.

Verification answers:

*   Was method called?
*   How many times?
*   With which arguments?

***

### ✅ Basic Verification

```java
verify(repo).findById(1L);
```

✅ Fails if not called.

***

### ✅ Invocation Count

| Method          | Meaning              |
| --------------- | -------------------- |
| `times(n)`      | Exactly n times      |
| `never()`       | Should not be called |
| `atLeastOnce()` | 1 or more            |
| `atLeast(n)`    | n or more            |

```java
verify(repo, times(1)).save(any());
```

***

### ✅ Verify Arguments

```java
verify(repo).findById(anyLong());
verify(repo).findById(eq(1L));
```

***

### ✅ Verify No Extra Calls

```java
verifyNoMoreInteractions(repo);
```

⚠️ Use sparingly (can make tests brittle).

***

## 8️⃣ Capturing Arguments – `ArgumentCaptor`

### ✅ Why ArgumentCaptor?

Use when:

*   Argument is complex
*   You must verify **internal state**, not just interaction

***

### ✅ Example

```java
@Captor
ArgumentCaptor<User> captor;

verify(repo).save(captor.capture());

User user = captor.getValue();
assertEquals(10L, user.getId());
```

✅ Behavior + state verification combined.

***

## 9️⃣ Mocking Void Methods

### ✅ Why Special APIs?

Void methods cannot use `when().thenReturn()`.

Use **`doXxx()` family**.

***

### ✅ Common APIs

| Method        | Purpose           |
| ------------- | ----------------- |
| `doNothing()` | Suppress behavior |
| `doThrow()`   | Simulate failures |
| `doAnswer()`  | Custom logic      |

***

### ✅ Example

```java
doThrow(new RuntimeException())
    .when(repo).deleteById(1L);
```

***

## 🔥 IMPORTANT RULE (Interview Favorite)

❌ Never use `when()` with:

*   Void methods
*   Spies

✅ Always use `doXxx()` for spies and voids.

***

## 🔟 Spies – Partial Mocking

### ✅ What is a Spy?

*   Wraps a real object
*   Real methods run by default

```java
Service spy = spy(new Service());
```

***

### ✅ Stubbing a Spy (VERY IMPORTANT)

❌ Wrong:

```java
when(spy.method()).thenReturn(...);
```

✅ Correct:

```java
doReturn(value).when(spy).method();
```

***

### ✅ When to Use Spies

✅ Legacy code  
✅ Partial override  
❌ Avoid when object has side effects

***

## 1️⃣1️⃣ Mockito Best Practices (INTERVIEW + PROD)

### ✅ Mock Selectively

*   Mock external dependencies
*   Do NOT mock entities, DTOs, value objects

***

### ✅ Avoid Over‑Mocking

*   Don’t verify trivial calls
*   Verify meaningful behavior only

***

### ✅ Prefer Constructor Injection

Use `@InjectMocks` with constructor‑based classes.

***

### ✅ Avoid `reset()` Normally

*   Prefer fresh test instances
*   Reset only when reuse is unavoidable

***

### ✅ Clean Unit Test Checklist

✅ Fast  
✅ Isolated  
✅ Deterministic  
✅ Behavior‑focused  
✅ Minimal verification

***

## 🔑 ONE‑LINE MENTAL MODELS (MEMORIZE)

*   **Mockito** → isolate dependencies
*   **Mock** → verify behavior
*   **Stub** → control output
*   **Spy** → partial real behavior
*   **Captor** → inspect arguments
*   **Unit test** → no infra, no side effects

***

## ✅ FINAL INTERVIEW‑READY SUMMARY

> **Mockito enables true unit testing by replacing external dependencies with programmable test doubles, allowing developers to stub behavior, verify interactions, and test business logic in isolation without relying on real infrastructure.**

***

## 🔜 What’s Next (OPTIONAL BUT POWERFUL)

If you want, we can next cover:

1.  `@Mock` vs `@MockBean` (Spring Boot)
2.  Mockito Strict Stubbing
3.  Mockito with Spring Boot slice tests
4.  Mockito vs Integration Testing (when NOT to mock)
