# ✅ 3. Creating Mocks in Mockito

***

## 3.1 What does “creating a mock” actually mean?

When we say **“Mockito creates a mock”**, this is what really happens:

> Mockito creates a **runtime-generated fake object** that:
>
> *   Implements (or subclasses) the given class
> *   Does **NOT** execute real logic
> *   Returns default values unless explicitly stubbed
> *   Records all method calls for verification

📌 Internally, Mockito uses **Byte Buddy** to generate these fake implementations at runtime.

***

## 3.2 Creating mocks PROGRAMMATICALLY (Mockito.mock)

This is the **most basic and fundamental API**.

### ✅ Syntax

```java
SomeClass mock = Mockito.mock(SomeClass.class);
```

or (most commonly used):

```java
import static org.mockito.Mockito.*;

SomeClass mock = mock(SomeClass.class);
```

***

### ✅ Example (very simple)

```java
List<String> list = Mockito.mock(List.class);

list.add("A");
list.get(0);
```

✅ No real `ArrayList`  
✅ No internal data storage  
✅ Mockito just **records interactions**

***

### ✅ What happens if you call methods on a mock?

By default, Mockito returns **default values**:

| Return type | Default value |
| ----------- | ------------- |
| int         | 0             |
| boolean     | false         |
| object      | null          |
| collection  | null          |

Example:

```java
List<String> list = mock(List.class);
System.out.println(list.size()); // 0
System.out.println(list.get(0)); // null
```

📌 This is intentional — mocks do **nothing** unless told to do so.

***

## 3.3 Why programmatic mock creation is important

Using `Mockito.mock()` is useful when:

✅ You need **manual control**  
✅ You are not using annotations  
✅ You want to create mocks dynamically  
✅ You are outside JUnit (utility tests)

But in **real projects**, we usually prefer **annotations**, because they are cleaner and less error‑prone.

***

# ✅ 4. Mockito Annotations (VERY IMPORTANT)

Mockito annotations make tests:

*   Cleaner
*   Shorter
*   More readable
*   Easier to maintain

They eliminate repeated calls to `Mockito.mock()`.

***

## 4.1 `@Mock` — Create mock objects

***

### ✅ What `@Mock` does

```java
@Mock
UserRepository repository;
```

Means:

> “Create a Mockito mock of `UserRepository`  
> and assign it to this field.”

This is **exactly equivalent to**:

```java
UserRepository repository = Mockito.mock(UserRepository.class);
```

But handled automatically by Mockito.

***

### ✅ Full example

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository repository;

    @Test
    void simpleMockTest() {
        when(repository.findById(1L)).thenReturn(new User());
        verify(repository).findById(1L);
    }
}
```

***

### ✅ Important: `@Mock` alone is not enough

Mockito must **initialize annotations**, otherwise mocks will be `null`.

✅ JUnit 5:

```java
@ExtendWith(MockitoExtension.class)
```

✅ JUnit 4:

```java
@RunWith(MockitoJUnitRunner.class)
```

This step is **mandatory**.

***

## 4.2 `@InjectMocks` — Inject mocks into the class under test

This annotation is ***extremely important*** and often misunderstood.

***

### ✅ What `@InjectMocks` does

```java
@InjectMocks
UserService userService;
```

Means:

> “Create a real instance of `UserService`  
> and inject available Mockito mocks into it.”

***

### ✅ Injection rules (VERY IMPORTANT)

Mockito injects mocks using the following order:

1.  **Constructor injection** (highest priority)
2.  **Setter injection**
3.  **Field injection** (last resort)

No Spring context involved.  
No `@Autowired`.  
No magic.

***

### ✅ Example (COMMON SCENARIO)

#### Production code:

```java
class UserService {

    private final UserRepository repository;

    UserService(UserRepository repository) {
        this.repository = repository;
    }

    void process(Long id) {
        repository.findById(id);
    }
}
```

***

#### Test code:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository repository;

    @InjectMocks
    UserService userService;

    @Test
    void verifyInjection() {
        userService.process(1L);
        verify(repository).findById(1L);
    }
}
```

✅ Mockito creates:

*   Mock `UserRepository`
*   Real `UserService`
*   Injects mock into constructor

***

### ✅ What `@InjectMocks` does NOT do

❌ It does NOT create mocks  
❌ It does NOT work like Spring DI  
❌ It does NOT scan application context

It only works with **Mockito-managed objects**.

***

## 4.3 `@Captor` — Capture method arguments (SUPER IMPORTANT)

### ✅ Why `@Captor` exists

What if you want to check:

*   What object was passed?
*   What value was inside the argument?

You **cannot do this** with `verify()` alone.

This is where `@Captor` comes in.

***

### ✅ What `@Captor` does

```java
@Captor
ArgumentCaptor<User> userCaptor;
```

Means:

> “Create an object that can capture arguments passed
> to mocked methods for later inspection.”

***

### ✅ Example (classic interview question)

#### Production code:

```java
class UserService {

    private final UserRepository repository;

    UserService(UserRepository repository) {
        this.repository = repository;
    }

    void saveUser(Long id) {
        repository.save(new User(id));
    }
}
```

***

#### Test code using `@Captor`:

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
    void captureArgumentExample() {
        userService.saveUser(10L);

        verify(repository).save(userCaptor.capture());

        User capturedUser = userCaptor.getValue();
        assertEquals(10L, capturedUser.getId());
    }
}
```

✅ We verified:

*   Method was called
*   Inspected the actual object passed
*   Checked internal state

***

### ✅ When to use `@Captor`

Use `@Captor` when:

*   Method argument is complex
*   You want to verify **state**, not just interaction
*   Object is constructed inside method (not passed in)

***

## 4.4 Relationship between `@Mock`, `@InjectMocks`, `@Captor`

| Annotation     | Purpose                          |
| -------------- | -------------------------------- |
| `@Mock`        | Create fake dependency           |
| `@InjectMocks` | Create real class + inject mocks |
| `@Captor`      | Capture and inspect arguments    |

***

## ✅ Putting it all together (FULL EXAMPLE)

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    PaymentService paymentService;

    @Captor
    ArgumentCaptor<Order> orderCaptor;

    @InjectMocks
    OrderService orderService;

    @Test
    void completeFlowTest() {
        orderService.placeOrder(99L);

        verify(paymentService).pay(orderCaptor.capture());
        assertEquals(99L, orderCaptor.getValue().getId());
    }
}
```

This test:
✅ Uses mocks  
✅ Injects mocks  
✅ Verifies interaction  
✅ Verifies argument state

***

## ✅ Interview‑Ready One‑Line Summary

*   **`Mockito.mock()`** → manual mock creation
*   **`@Mock`** → automatic mock creation
*   **`@InjectMocks`** → real object with mocks injected
*   **`@Captor`** → capture and inspect method arguments

> Mockito annotations are convenience tools that remove boilerplate while preserving full control over unit test behavior.

***

## 🔜 Next Topics (STRONGLY RECOMMENDED)

1️⃣ `when().thenReturn()` vs `doReturn()`  
2️⃣ Mocking exceptions (`thenThrow`)  
3️⃣ Spies vs Mocks (CRITICAL topic)  
4️⃣ Mockito with Spring Boot (`@MockBean` vs `@Mock`)
