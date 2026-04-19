# ✅ 7. Spying Real Objects (Mockito `spy()`)

***

## 7.1 First: Why do Spies exist?

Up to now, you learned **mocks**.

### ✅ Mock recap

A **mock**:

*   Is a *fake object*
*   Has **no real logic**
*   Returns default values unless stubbed
*   Is used when you want **complete isolation**

But sometimes you want:

> ✅ *Most real behavior*  
> ✅ *Except one or two methods*

That is exactly why **spies** exist.

***

## 7.2 What is a Spy? (precise definition)

### ✅ Official definition (Mockito / Baeldung)

A **spy** is a **partial mock**:

*   It wraps a **real object**
*   Real methods are called **by default**
*   You can selectively **mock specific methods** [\[atlassian.com\]](https://www.atlassian.com/microservices/microservices-architecture/microservices-vs-monolith)

***

## 7.3 How is a Spy different from a Mock?

| Aspect               | Mock           | Spy                      |
| -------------------- | -------------- | ------------------------ |
| Real object exists   | ❌ No           | ✅ Yes                    |
| Real methods invoked | ❌ No           | ✅ Yes (by default)       |
| Use case             | Full isolation | Partial override         |
| Risk                 | Safe           | Can execute side effects |

📌 **Interview one‑liner**

> *Use mocks when you want no real behavior; use spies when you want most real behavior with selective stubbing.*

***

## 7.4 Creating a Spy

### ✅ Using `spy()`

```java
List<String> realList = new ArrayList<>();
List<String> spyList = spy(realList);
```

What happens:

*   `spyList` wraps `realList`
*   Real `ArrayList` methods are executed
*   Calls are tracked by Mockito

***

## 7.5 Spy behavior WITHOUT stubbing

```java
spyList.add("A");
spyList.add("B");

System.out.println(spyList.size()); // 2
```

✅ Real logic executed  
✅ Real data structure modified

This is **not** the case with mocks.

***

## 7.6 Stubbing a specific method on a Spy

Now let’s override **only one method**.

```java
doReturn(100).when(spyList).size();
```

### ✅ Meaning

> “Even though this is a real list, when `size()` is called, return 100.”

***

### ✅ Important rule (VERY IMPORTANT)

❌ **DO NOT USE** `when(spy.method())`

```java
// ❌ DANGEROUS
when(spyList.size()).thenReturn(100);
```

This **invokes the real method during stubbing**, which can:

*   Modify state
*   Throw exceptions
*   Hit DB / filesystem

✅ **ALWAYS use `doReturn()` with spies**

```java
doReturn(100).when(spyList).size();
```

📌 This rule is explicitly highlighted by Baeldung and Mockito docs. [\[atlassian.com\]](https://www.atlassian.com/microservices/microservices-architecture/microservices-vs-monolith)

***

## 7.7 Real‑world spy example (production‑style)

### Production code

```java
class PriceCalculator {
    int basePrice() {
        return 100;
    }

    int totalPrice() {
        return basePrice() + 20;
    }
}
```

***

### Test with Spy

```java
PriceCalculator calculator = spy(new PriceCalculator());

doReturn(200).when(calculator).basePrice();

int result = calculator.totalPrice();

assertEquals(220, result);
```

✅ `totalPrice()` uses real logic  
✅ `basePrice()` is overridden  
✅ Perfect partial mocking use case

***

## 7.8 When SHOULD you use Spies?

✅ Use **spies** when:

*   Legacy code is hard to refactor
*   Only 1–2 methods need isolation
*   Complex real logic should still execute

❌ Avoid spies when:

*   Object talks to DB / network
*   Constructor has side effects
*   You want pure unit isolation

📌 **Senior advice**

> *If you find yourself spying heavily, refactor the code instead.*

***

# ✅ 8. Capturing Arguments (ArgumentCaptor)

This topic is **one of the most asked Mockito interview questions**.

***

## 8.1 Why ArgumentCaptor is needed

Consider this verification:

```java
verify(repo).save(any(User.class));
```

✅ Confirms method was called  
❌ Does NOT tell **what was inside User**

But often you need to verify:

*   ID value
*   Status field
*   Derived calculations
*   Mutated objects

That’s the job of **ArgumentCaptor**.

***

## 8.2 What is ArgumentCaptor?

### ✅ Definition

`ArgumentCaptor` captures **actual arguments passed** to a mock so you can:

*   Inspect internal state
*   Assert values
*   Verify transformations [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/monolithic-vs-microservices-architecture/)

***

## 8.3 Creating ArgumentCaptor

### ✅ Annotation-based (recommended)

```java
@Captor
ArgumentCaptor<User> userCaptor;
```

Mockito initializes it automatically.

***

### ✅ Manual creation (less common)

```java
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
```

***

## 8.4 Basic ArgumentCaptor example (step‑by‑step)

### Production code

```java
class UserService {
    private final UserRepository repository;

    UserService(UserRepository repository) {
        this.repository = repository;
    }

    void register(Long id) {
        repository.save(new User(id, "ACTIVE"));
    }
}
```

***

### Test code

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository repository;

    @InjectMocks
    UserService userService;

    @Captor
    ArgumentCaptor<User> userCaptor;

    @Test
    void captureUserArgument() {
        userService.register(10L);

        verify(repository).save(userCaptor.capture());

        User captured = userCaptor.getValue();
        assertEquals(10L, captured.getId());
        assertEquals("ACTIVE", captured.getStatus());
    }
}
```

✅ Method call verified  
✅ Argument captured  
✅ Internal values asserted

***

## 8.5 Capturing multiple invocations

### When method is called multiple times

```java
verify(repository, times(2)).save(userCaptor.capture());

List<User> users = userCaptor.getAllValues();
```

You can then assert:

```java
assertEquals(2, users.size());
```

***

## 8.6 ArgumentCaptor vs verify(any())

| Aspect          | verify(any()) | ArgumentCaptor  |
| --------------- | ------------- | --------------- |
| Confirms call   | ✅             | ✅               |
| Inspects values | ❌             | ✅               |
| Use case        | Behavior only | Behavior + data |

📌 **Interview line**

> *ArgumentCaptor is used when interaction verification must be combined with state verification.*

***

## 8.7 Common mistakes (INTERVIEW TRAPS)

### ❌ Using captor for simple values

```java
verify(repo).findById(1L);
```

✅ No captor needed here  
Captors are for **complex objects**

***

### ❌ Overusing captors

Captors can make tests:

*   Too detailed
*   Brittle
*   Implementation‑dependent

✅ Use captor **only when data inspection matters**

***

## ✅ Spies vs ArgumentCaptor — Combined Example

```java
Service service = spy(new Service());

doReturn(5).when(service).calculateValue();

service.execute();

verify(repository).save(valueCaptor.capture());
assertEquals(5, valueCaptor.getValue());
```

✅ Spy modifies internal behavior  
✅ ArgumentCaptor inspects interaction

***

## ✅ Interview‑Perfect Summary (Memorize)

*   **Spy** → partial mock that calls real methods by default
*   **`doReturn()`** → mandatory for stubbing spies
*   **ArgumentCaptor** → capture and inspect arguments passed to mocks
*   **Use captors only when verifying object state matters**

***

## 📚 Authoritative Sources

*   Baeldung – *Using Spies in Mockito* [\[atlassian.com\]](https://www.atlassian.com/microservices/microservices-architecture/microservices-vs-monolith)
*   Baeldung – *Mockito ArgumentCaptor* [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/monolithic-vs-microservices-architecture/)
*   Mockito Official Documentation [\[refind.com\]](https://refind.com/publications/martinfowler)

***

## 🔜 Next Topics (Highly Recommended)

1️⃣ `doReturn()` vs `when()` (deep dive)  
2️⃣ Mocking void methods (`doNothing`, `doThrow`)  
3️⃣ Spies vs Mocks – interview comparison  
4️⃣ Mockito + Spring Boot (`@MockBean` vs `@Mock`)
