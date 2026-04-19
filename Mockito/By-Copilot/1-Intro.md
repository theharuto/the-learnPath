 ✅ 1. Introduction to Mockito

## What is Mockito?

**Mockito** is a **Java mocking framework** used for **unit testing** that allows you to:

*   Create **fake objects (mocks)** at runtime
*   Replace **real dependencies** (DB, API, filesystem, message broker)
*   Test a class **in isolation**, focusing only on its logic

**Official definition (Mockito docs):**

> *Mockito enables mock creation, stubbing, and verification, so you can write clean and readable unit tests without depending on real implementations* [\[site.mockito.org\]](https://site.mockito.org/), [\[javadoc.io\]](https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html)

**Baeldung definition:**

> *Mockito is used to simulate the behavior of real objects in controlled ways, enabling fast, isolated, and deterministic unit tests* [\[docs.spring.io\]](https://docs.spring.io/spring-framework/reference/index.html)

***

## ✅ Why “mocking” is necessary (real problem)

In real applications, classes depend on:

*   Databases
*   REST APIs
*   File systems
*   Message queues
*   Other services

These dependencies are:

*   ❌ Slow
*   ❌ Non‑deterministic
*   ❌ Hard to control
*   ❌ External to business logic

👉 **Unit testing should not depend on them.**

Mockito solves this by **replacing dependencies with test doubles**.

***

# ✅ 2. Why Use Mockito?

### Core reasons (production + interview)

| Problem           | Without Mockito          | With Mockito     |
| ----------------- | ------------------------ | ---------------- |
| External DB       | Real connection required | Fake repository  |
| API call          | Network dependency       | Stubbed response |
| Test speed        | Slow                     | Fast             |
| Test stability    | Flaky                    | Deterministic    |
| Failure debugging | Hard                     | Easy             |

**Baeldung explicitly recommends Mockito to isolate unit tests from infrastructure**. [\[docs.spring.io\]](https://docs.spring.io/spring-framework/reference/index.html)

***

## ✅ Example: Without Mockito (BAD UNIT TEST)

```java
@Test
void testCreateOrder() {
    OrderService service = new OrderService(new RealDatabase());
    service.createOrder();
}
```

Problems:

*   Requires real DB
*   Slow
*   Fails if DB is down
*   ❌ This is NOT a unit test

***

## ✅ Same Example: With Mockito (PROPER UNIT TEST)

```java
@Mock
OrderRepository orderRepository;

@InjectMocks
OrderService orderService;

@Test
void testCreateOrder() {
    when(orderRepository.save(any())).thenReturn(new Order());
    orderService.createOrder();
    verify(orderRepository).save(any());
}
```

✔️ No DB  
✔️ Fast  
✔️ Deterministic  
✔️ Pure business logic test

***

# ✅ 3. Key Features of Mockito

According to the **Mockito official docs and Baeldung**, core features are: [\[site.mockito.org\]](https://site.mockito.org/), [\[bing.com\]](https://bing.com/search?q=Mockito+official+documentation+mocking+framework), [\[docs.spring.io\]](https://docs.spring.io/spring-framework/reference/index.html)

***

## 3.1 Mocking

### Mocking = creating fake objects at runtime

```java
Repository repo = mock(Repository.class);
```

or using annotations:

```java
@Mock
Repository repo;
```

✅ No real implementation  
✅ No constructor execution  
✅ No DB / network calls

***

## 3.2 Stubbing

### Stubbing = defining mock behavior

```java
when(repo.findById(1L)).thenReturn(user);
```

Meaning:

> *When this method is called with these arguments, return this value.*

Stubs:

*   Control **inputs**
*   Control **execution path**

***

## 3.3 Verification

### Verification = checking interactions

```java
verify(repo).findById(1L);
```

Meaning:

> *Was this method called? How many times? With which arguments?*

Mockito allows:

*   `times(n)`
*   `never()`
*   `atLeastOnce()`

***

## 3.4 Simplifying Complex Tests

Mockito:

*   Removes setup noise
*   Eliminates manual stub classes
*   Keeps tests **short and intention‑revealing**

This is why Mockito is the **most popular Java mocking framework**. [\[site.mockito.org\]](https://site.mockito.org/)

***

# ✅ 4. Basic Concepts – Mocks vs Stubs (VERY IMPORTANT)

This topic is **frequently misunderstood in interviews**.

***

## What are Test Doubles?

**Test Double** = a generic term for any fake object used in tests:

*   Mock
*   Stub
*   Fake
*   Spy

***

## ✅ Stub (Definition)

> A **stub** provides **predefined responses** to method calls and is used for **state verification**.

**Key properties (Baeldung + Software Testing literature):**

*   Returns fixed values
*   Does **not verify interactions**
*   Cannot fail tests on its own [\[bing.com\]](https://bing.com/search?q=Mocks+vs+Stubs+software+testing+explanation), [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-testing/stubs-vs-mocks/)

### Stub Example

```java
when(paymentService.pay()).thenReturn("SUCCESS");
```

Here:

*   We **only care about output**
*   We do NOT care how many times `pay()` was called

***

## ✅ Mock (Definition)

> A **mock** verifies **how a dependency is used**, not just what it returns.

**Key properties:**

*   Can stub behavior
*   Can verify:
    *   method calls
    *   call count
    *   arguments
*   Test can fail if expectations are not met [\[bing.com\]](https://bing.com/search?q=Mocks+vs+Stubs+software+testing+explanation), [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-testing/stubs-vs-mocks/)

### Mock Example

```java
verify(emailService).sendWelcomeMail(user);
```

Here:

*   The **interaction itself is important**
*   Business correctness depends on the call happening

***

## ✅ Mock vs Stub — Interview‑Ready Comparison

| Aspect                 | Stub           | Mock                  |
| ---------------------- | -------------- | --------------------- |
| Purpose                | Control output | Verify behavior       |
| Focus                  | State          | Interaction           |
| Can fail test directly | ❌ No           | ✅ Yes                 |
| Typical usage          | External data  | External side‑effects |

***

# ✅ 5. When Should You Use Mockito?

### ✅ Use mocking when:

*   Code depends on **external systems**
*   Behavior is **non‑deterministic**
*   Test should focus **only on class logic**
*   You want **fast, isolated unit tests**

Official Mockito docs explicitly highlight isolation as the primary goal. [\[site.mockito.org\]](https://site.mockito.org/), [\[bing.com\]](https://bing.com/search?q=Mockito+official+documentation+mocking+framework)

***

### ❌ Do NOT use Mockito when:

*   Testing pure POJOs
*   Testing integration flows
*   Testing framework behavior itself

***

# ✅ 6. Interview Gold – One‑Line Summaries

*   **Mockito** → isolate business logic
*   **Stub** → control returned data
*   **Mock** → verify collaboration
*   **Unit test** → fast, isolated, deterministic
*   **Integration test** → real components

***

## ✅ Final Mental Model

> **Mockito allows you to replace real dependencies with programmable fake objects so that unit tests verify business logic and interactions without relying on external systems.**

***

### ✅ Sources Used (Authoritative)

*   Mockito Official Documentation [\[site.mockito.org\]](https://site.mockito.org/), [\[javadoc.io\]](https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/org/mockito/Mockito.html)
*   Baeldung: Introduction to Mockito [\[docs.spring.io\]](https://docs.spring.io/spring-framework/reference/index.html)
*   Software Testing: Mocks vs Stubs [\[bing.com\]](https://bing.com/search?q=Mocks+vs+Stubs+software+testing+explanation), [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-testing/stubs-vs-mocks/)

***

### 🔜 Next Mockito Topics (Recommended Order)

1.  `@Mock`, `@InjectMocks`, `@ExtendWith(MockitoExtension.class)`
2.  `when().thenReturn()` vs `doReturn()`
3.  Verifying calls (`verify`, `times`, `never`)
4.  Mockito with Spring Boot (`@MockBean`)
5.  Spies vs Mocks (VERY IMPORTANT)
