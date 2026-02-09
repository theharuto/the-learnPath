# ✅ **Layer 3: Deep Dive into JUnit Jupiter**

JUnit **Jupiter** = the **API + programming model** you use to *write* tests in JUnit 5.  
It includes annotations, lifecycle rules, assertions, assumption mechanisms, and extension APIs.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

# **1. Core JUnit Jupiter Annotations**

Here are the foundational annotations used in almost every JUnit 5 test class.

***

## ✔ **@Test**

Marks a method as a test case.

*   Replaces JUnit 4’s `@Test`.
*   Test methods must not be private or static.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

### Example

```java
@Test
void shouldAddNumbers() {
    assertEquals(4, 2 + 2);
}
```

***

## ✔ **@BeforeEach**

Runs **before each** test method.  
Used for per‑test setup.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

Example: open DB connection, create fresh objects, etc.

***

## ✔ **@AfterEach**

Runs **after each** test.  
Used for cleanup.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

## ✔ **@BeforeAll**

Runs **once before all tests** in the class.

*   Must be `static` unless the test instance lifecycle is `PER_CLASS`.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

## ✔ **@AfterAll**

Runs **once after all tests**.  
Used to clean final resources.

*   Must be `static` unless lifecycle is `PER_CLASS`.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

## ✔ **@Disabled**

Skips a test method or entire test class.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

# **2. UML‑ish Diagram: Test Lifecycle Flow**

A simple, intuitive diagram that actually helps visualize ordering:

                   ┌──────────────────┐
                   │   @BeforeAll      │  (runs once)
                   └─────────┬────────┘
                             │
            ┌────────────────┴────────────────┐
            │                                 │
     ┌──────▼──────┐                   ┌──────▼──────┐
     │ @BeforeEach │                   │ @BeforeEach │   (runs before each test)
     └──────┬──────┘                   └──────┬──────┘
            │                                 │
       ┌────▼────┐                        ┌────▼────┐
       │  @Test  │                        │  @Test  │  (multiple test methods)
       └────┬────┘                        └────┬────┘
            │                                 │
     ┌──────▼──────┐                   ┌──────▼──────┐
     │ @AfterEach │                   │ @AfterEach │   (runs after each test)
     └──────┬──────┘                   └──────┬──────┘
            │                                 │
            └────────────────┬───────────────┘
                             │
                   ┌─────────▼────────┐
                   │    @AfterAll      │  (runs once)
                   └──────────────────┘

This reflects the true JUnit Jupiter lifecycle.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

# **3. Full Example Combining Lifecycle Annotations**

```java
class CalculatorTest {

    @BeforeAll
    static void initAll() {
        System.out.println("Runs once before all tests");
    }

    @BeforeEach
    void init() {
        System.out.println("Runs before each test");
    }

    @Test
    void testAdd() {
        assertEquals(5, 2 + 3);
    }

    @Test
    void testMultiply() {
        assertEquals(10, 2 * 5);
    }

    @AfterEach
    void tearDown() {
        System.out.println("Runs after each test");
    }

    @AfterAll
    static void tearDownAll() {
        System.out.println("Runs once after all tests");
    }
}
```

***

# **4. Assertions in JUnit Jupiter**

JUnit Jupiter provides a rich set of assertions through the `org.junit.jupiter.api.Assertions` class.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

### ✔ Common Assertions

| Assertion                             | Usage                                       |
| ------------------------------------- | ------------------------------------------- |
| `assertEquals(expected, actual)`      | Compares expected and actual values         |
| `assertNotEquals(a, b)`               | Ensures values differ                       |
| `assertTrue(condition)`               | Ensures condition is true                   |
| `assertFalse(condition)`              | Ensures condition is false                  |
| `assertThrows(exception, executable)` | Ensures execution throws expected exception |

***

### ✔ Example with Multiple Assertions

```java
@Test
void testUserProfile() {
    assertAll(
        () -> assertEquals("John", user.getName()),
        () -> assertTrue(user.getAge() > 18),
        () -> assertNotNull(user.getId())
    );
}
```

Assertion grouping is enabled by Java 8 lambda support.    [\[youtube.com\]](https://www.youtube.com/watch?v=lj5nnGa_DIw)

***

# **5. Order of Execution (Lifecycle Priority)**

By default, JUnit uses the following execution order:

1.  `@BeforeAll`
2.  Repeated for each test:
    *   `@BeforeEach`
    *   `@Test`
    *   `@AfterEach`
3.  `@AfterAll`

This ordering is defined by JUnit Jupiter’s lifecycle rules.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

If a test is annotated with `@Disabled`, the entire lifecycle for that test is skipped.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

# **6. Minimal UML-ish Class Structure**

A simple class-to-annotation relationship diagram to show structure:

             ┌─────────────────────────────┐
             │        <<TestClass>>        │
             │       CalculatorTest        │
             ├─────────────────────────────┤
             │  @BeforeAll initAll()       │
             │  @BeforeEach init()         │
             │  @Test testAdd()            │
             │  @Test testMultiply()       │
             │  @AfterEach tearDown()      │
             │  @AfterAll tearDownAll()    │
             └─────────────────────────────┘

This clarifies which annotations belong on which methods.

***

# ⭐ **Summary — What You Should Remember from Layer 3**

| Concept                 | Meaning                                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------- |
| **Annotations**         | Control how tests run before/after each test or class                                         |
| **Lifecycle**           | JUnit Jupiter strictly orders execution: BeforeAll → BeforeEach → Test → AfterEach → AfterAll |
| **Assertions**          | Validate expected behavior with a rich API                                                    |
| **UML diagrams**        | Help visualize lifecycle and test class design                                                |
| **JUnit Jupiter focus** | Clean, modern Java 8+ test-writing model                                                      |

***

If you want, the next layer can be:

### **Layer 4: Advanced JUnit Jupiter Features**

*   Dynamic tests
*   Parameterized tests
*   Test instance lifecycle (`PER_METHOD` vs `PER_CLASS`)
*   Nested tests
*   Assumptions (`assumeTrue`, `assumingThat`)
*   Extensions
