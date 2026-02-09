# ✅ **1. Why Exception Testing Matters**

Exception testing ensures your code **fails correctly** when given invalid inputs or when preconditions are violated.  
It verifies **error handling paths**, not just success paths.    [\[tutorialpedia.org\]](https://www.tutorialpedia.org/blog/how-do-you-assert-that-a-certain-exception-is-thrown-in-junit-tests/)

***

# ✅ **2. Core API: `assertThrows()`**

JUnit 5 introduces a modern and clear API for exception testing: **`assertThrows()`**.  
This API replaces outdated JUnit 4 approaches like `@Test(expected=...)` or `ExpectedException @Rule`.    [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception)

### **Syntax**

```java
<T extends Throwable> T assertThrows(Class<T> expectedType, Executable executable)
```

Explanation:

*   `expectedType` — type of exception you expect.
*   `executable` — lambda containing code expected to throw the exception.

When the exception is thrown:

*   The method **returns the caught exception**, allowing further validation (message, cause).    [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception)

If no exception (or wrong type) is thrown:

*   JUnit fails the test.

***

# ✅ **3. Basic Example — Verifying an Exception Is Thrown**

From Baeldung’s official example:    [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception)

```java
@Test
void whenExceptionThrown_thenAssertionSucceeds() {
    Exception exception = assertThrows(
        NumberFormatException.class,
        () -> Integer.parseInt("1a")
    );

    String expected = "For input string";
    assertTrue(exception.getMessage().contains(expected));
}
```

What happens:

*   `Integer.parseInt("1a")` throws `NumberFormatException`.
*   `assertThrows()` catches it and returns the exception instance.
*   You assert on the message for additional validation.

***

# ✅ **4. Inheritance Behavior**

If you expect a **superclass**, any subclass exception also passes.

Example (also from Baeldung):    [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception)

```java
@Test
void whenDerivedExceptionThrown_thenAssertionSucceeds() {
    assertThrows(
        RuntimeException.class,
        () -> Integer.parseInt("1a")
    );
}
```

`NumberFormatException` extends `RuntimeException`, so this passes.

***

# ✅ **5. Exact Exception Type: `assertThrowsExactly()`**

If you want to ensure **only a specific type** is thrown (no subclass allowed), use:

```java
assertThrowsExactly(ExpectedExceptionClass.class, () -> code);
```

Example:    [\[howtodoinjava.com\]](https://howtodoinjava.com/junit5/expected-exception-example/)

```java
RuntimeException thrown = assertThrowsExactly(
    RuntimeException.class,
    () -> { throw new IllegalArgumentException("expected"); }  // FAILS: subclass thrown
);
```

This fails because `IllegalArgumentException` ≠ `RuntimeException`.

***

# ✅ **6. Testing No Exceptions: `assertDoesNotThrow()`**

Useful for verifying functions that **must not** throw errors.

Example:    [\[howtodoinjava.com\]](https://howtodoinjava.com/junit5/expected-exception-example/)

```java
assertDoesNotThrow(() -> {
    // safe code that should not throw
});
```

***

# ✅ **7. Realistic Example — Testing Invalid Input Handling**

Imagine a `BankAccount` class:

```java
@Test
void withdrawalMoreThanBalance_shouldThrowException() {
    BankAccount acc = new BankAccount(9);

    NotEnoughFundsException ex =
        assertThrows(NotEnoughFundsException.class, () -> acc.withdraw(10));

    assertTrue(ex.getMessage().contains("Balance")); 
}
```

This pattern is exactly how Baeldung demonstrates exception testing.    [\[baeldung.com\]](https://www.baeldung.com/junit)

***

# ✅ **8. Code Pattern: “Act and Assert in One Block”**

Thanks to lambdas, you wrap the failing operation directly inside `assertThrows()`:

```java
assertThrows(MyException.class, () -> myObj.doSomething());
```

This avoids JUnit 4’s old `@Rule` or messy try‑catch blocks.    [\[stackoverflow.com\]](https://stackoverflow.com/questions/40268446/how-to-assert-an-exception-is-thrown-with-junit-5)

***

# ✅ **9. Why JUnit 5’s Approach Is Better**

Compared to JUnit 4:

*   No shared-state `ExpectedException` rule (which caused flaky tests).    [\[javathinking.com\]](https://www.javathinking.com/blog/junit-5-how-to-assert-an-exception-is-thrown/)
*   Exception assertion stays close to the actual code throwing it (clearer Act–Assert separation).
*   You can assert:
    *   Type
    *   Message
    *   Cause
    *   Multiple exceptions in one test (though generally discouraged)    [\[javathinking.com\]](https://www.javathinking.com/blog/junit-5-how-to-assert-an-exception-is-thrown/)

***

# 📘 **10. Summary Table**

| Goal                          | JUnit 5 API                     | Notes                                                                                                                                               |
| ----------------------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ensure exception is thrown    | `assertThrows()`                | Allows subclass matches. [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception)                            |
| Ensure exact type is thrown   | `assertThrowsExactly()`         | No subclass allowed. [\[howtodoinjava.com\]](https://howtodoinjava.com/junit5/expected-exception-example/)                   |
| Ensure no exception is thrown | `assertDoesNotThrow()`          | Useful for confirming safe code paths. [\[howtodoinjava.com\]](https://howtodoinjava.com/junit5/expected-exception-example/) |
| Assert message/cause          | Use returned exception instance | Supports fine-grained checks. [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception)                       |

***

# 🎯 **Final Takeaway**

Use **`assertThrows()`** whenever you need to assert that certain inputs cause errors.  
Use **`assertThrowsExactly()`** when you must enforce a stricter exception type.  
Use **`assertDoesNotThrow()`** for safety checks.

JUnit 5 makes exception testing:

*   **simple**
*   **readable**
*   **precise**
*   **far better** than JUnit 4’s old approaches.    [\[baeldung.com\]](https://www.baeldung.com/junit-assert-exception), [\[javathinking.com\]](https://www.javathinking.com/blog/junit-5-how-to-assert-an-exception-is-thrown/)

***
### **Layer 7: Assumptions in JUnit 5 (assumeTrue, assumingThat)**

or

### **Layer 7: Writing Custom Exceptions & Testing Them**
