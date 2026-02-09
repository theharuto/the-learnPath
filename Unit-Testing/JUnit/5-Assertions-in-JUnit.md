# ✅ **Layer 4: Assertions in JUnit 5**

Assertions are the **verification mechanism** in JUnit Jupiter.  
They ensure that your code behaves as expected.  
JUnit Jupiter provides its assertions via **`org.junit.jupiter.api.Assertions`**.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

# **1. Core Assertions in JUnit 5**

JUnit Jupiter includes a wide set of core assertion methods for validating program state.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

### ✔ **assertEquals(expected, actual)**

Checks that values are equal.

```java
assertEquals(10, calculator.add(5, 5));
```

***

### ✔ **assertNotEquals(a, b)**

Ensures values are not equal.

```java
assertNotEquals(5, calculator.add(2, 2));
```

***

### ✔ **assertTrue(condition)**

Ensures a boolean condition is `true`.

```java
assertTrue(user.isActive());
```

***

### ✔ **assertFalse(condition)**

Ensures a boolean condition is `false`.

```java
assertFalse(user.hasExpired());
```

***

### ✔ **assertNull(value)**

Ensures the value is `null`.

```java
assertNull(response.getError());
```

***

### ✔ **assertNotNull(value)**

Ensures the value is not `null`.

```java
assertNotNull(order.getId());
```

All of these assertion types are part of the core JUnit Jupiter assertion set.    [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/)

***

# **2. Combining Assertions with `assertAll()`**

JUnit 5 allows you to group multiple assertions using **`assertAll()`**, which ensures:

*   All assertions run
*   All failures are reported together (not just the first failure)

This is enabled thanks to Java 8 lambda expressions.    [\[youtube.com\]](https://www.youtube.com/watch?v=lj5nnGa_DIw)

### ✔ Example: Multiple Assertions in One Test

```java
@Test
void testUserProfile() {
    User user = new User("Alice", 25, "ACTIVE");

    assertAll(
        () -> assertEquals("Alice", user.getName()),
        () -> assertTrue(user.getAge() > 18),
        () -> assertNotNull(user.getStatus()),
        () -> assertEquals("ACTIVE", user.getStatus())
    );
}
```

This style is cleaner, avoids multiple test methods, and gives fuller error reporting.

***

# **3. UML‑ish Diagram: Assertion Grouping**

A small, meaningful diagram to visualize how `assertAll()` behaves:

            ┌─────────────────────────────┐
            │        assertAll()          │
            ├─────────────┬───────────────┤
            │             │               │
    ┌───────▼──────┐ ┌────▼────────┐ ┌────▼─────────┐
    │ assertEquals │ │ assertTrue   │ │ assertNotNull│
    └──────────────┘ └──────────────┘ └──────────────┘
             │              │               │
        (all executed even if one fails)

This diagram enhances understanding without overdoing visuals.

***

# **4. Full Example Using All Core Assertions**

```java
@Test
void testProductDetails() {

    Product product = new Product("Laptop", 50000, null);

    assertAll(
        () -> assertEquals("Laptop", product.getName()),
        () -> assertNotEquals(40000, product.getPrice()),
        () -> assertTrue(product.getPrice() > 0),
        () -> assertFalse(product.isOutOfStock()),
        () -> assertNull(product.getDiscountCode()),
        () -> assertNotNull(product.getName())
    );
}
```

***

# ⭐ **Summary of Layer 4**

| Concept             | Explanation                                                                                                                                                                                                    |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Core Assertions** | `assertEquals`, `assertNotEquals`, `assertTrue`, `assertFalse`, `assertNull`, `assertNotNull` validate behavior. [\[geekflare.com\]](https://geekflare.com/dev/unit-testing-guide/) |
| **assertAll()**     | Groups multiple assertions; all executed even if one fails. Enabled by Java 8 functional support. [\[youtube.com\]](https://www.youtube.com/watch?v=lj5nnGa_DIw)                  |
| **Use Cases**       | Cleaner validation, improved error reporting, better test readability.                                                                                                                                         |

***

If you're ready, we can continue with:

### **Layer 5: Advanced Assertions & Exception Testing**

*   `assertThrows()`
*   `assertTimeout()`
*   Using AssertJ / Hamcrest vs. JUnit assertions  
    OR

### **Layer 5: Assumptions in JUnit (assumeTrue, assumingThat)**
