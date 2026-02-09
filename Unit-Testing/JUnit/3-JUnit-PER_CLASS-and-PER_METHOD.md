## 1️⃣ Default behavior: `PER_METHOD`

By default, JUnit 5 uses:

```java
@TestInstance(Lifecycle.PER_METHOD) // default
```

**Meaning:**

* JUnit creates a **new instance of your test class for each test method**.
* That’s why `@BeforeEach` and `@AfterEach` can safely run without shared state issues.
* But **non-static `@BeforeAll` and `@AfterAll` are not allowed** because they would need a class instance that exists **before any test method**.

Example:

```java
class MyTest {
    int counter = 0;

    @BeforeEach
    void increment() {
        counter++;
    }

    @Test
    void test1() { System.out.println(counter); } // prints 1
    @Test
    void test2() { System.out.println(counter); } // prints 1 again, new instance
}
```

---

## 2️⃣ `PER_CLASS` lifecycle

When you write:

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
```

**Meaning:**

* JUnit creates **only one instance of the test class for all test methods**.
* All test methods share the same object → fields can maintain state across tests.
* Allows **non-static `@BeforeAll` and `@AfterAll`** because the single instance already exists.

Example:

```java
import org.junit.jupiter.api.*;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class MyTest {

    int counter = 0;

    @BeforeAll
    void beforeAll() { System.out.println("Before all tests"); }

    @BeforeEach
    void increment() { counter++; }

    @Test
    void test1() { System.out.println(counter); } // prints 1
    @Test
    void test2() { System.out.println(counter); } // prints 2, same instance
}
```

Output:

```
Before all tests
1
2
```

---

## 3️⃣ When to use `PER_CLASS`

* When you need **shared state between test methods**.
* When you want to **avoid static methods** for `@BeforeAll` and `@AfterAll`.
* Useful for expensive setup/teardown that you only want to run once.

⚠ Caveat: Be careful with shared mutable state — tests can affect each other. JUnit encourages **stateless tests**.

---

### TL;DR

| Lifecycle            | Number of instances | @BeforeAll/@AfterAll | Shared state               |
| -------------------- | ------------------- | -------------------- | -------------------------- |
| PER_METHOD (default) | One per test method | Must be static       | No                         |
| PER_CLASS            | One for whole class | Can be non-static    | Yes, shared across methods |

---
