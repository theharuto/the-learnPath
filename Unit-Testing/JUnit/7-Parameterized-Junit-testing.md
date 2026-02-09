# ✅ **What Are Parameterized Tests?**

Parameterized tests let you **run the same test method multiple times with different input data**, improving coverage without duplicating code.  
JUnit executes the test once **per input**, injecting the values into method parameters.    [\[baeldung.com\]](https://www.baeldung.com/parameterized-tests-junit-5)

To use parameterized tests, you must include:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-params</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
```

 [\[baeldung.com\]](https://www.baeldung.com/parameterized-tests-junit-5)

***

# ✅ **How Parameterized Tests Work Internally**

1.  Method is annotated with **`@ParameterizedTest`** (not `@Test`).
2.  JUnit locates the **argument provider** annotation (e.g., `@ValueSource`, `@CsvSource`, `@MethodSource`).
3.  JUnit creates a stream of **Arguments** and calls your test method once per entry.
4.  Invocation details like `{index}` and `{arguments}` are handled by the display name patterns.    [\[docs.junit.org\]](https://docs.junit.org/5.3.0/api/org/junit/jupiter/params/ParameterizedTest.html)

***

# 🚀 **Key Parameter Providers (Most Important)**

Let’s go through the three most important ones:

*   `@ValueSource`
*   `@CsvSource`
*   `@MethodSource`

***

# ✔ **1. `@ValueSource` — Simple Literal Inputs**

Used when you need to pass **single primitive/string arguments** into a test.    [\[baeldung.com\]](https://www.baeldung.com/parameterized-tests-junit-5)

### Supports:

*   `int`, `long`, `double`
*   `String`
*   Class literals (for EnumSource-like usage)
*   Does **not** support nulls. (Use `@NullSource` or `@NullAndEmptySource` instead.)    [\[howtodoinjava.com\]](https://howtodoinjava.com/junit5/parameterized-tests/)

### Example

```java
@ParameterizedTest
@ValueSource(strings = {"racecar", "radar", "level"})
void testPalindrome(String word) {
    assertTrue(isPalindrome(word));
}

boolean isPalindrome(String word) {
    return word.equals(new StringBuilder(word).reverse().toString());
}
```

 [\[prgrmmng.com\]](https://prgrmmng.com/parameterized-tests-in-junit-writing-data-driven-tests)

Each value in `ValueSource` becomes one execution of the test.

***

# ✔ **2. `@CsvSource` — Multiple Input Columns**

Use this when the test requires **multiple parameters**.

### Example

```java
@ParameterizedTest
@CsvSource({
    "2, 3, 5",
    "10, 5, 15",
    "7, 8, 15"
})
void testAddition(int a, int b, int expected) {
    Calculator calc = new Calculator();
    assertEquals(expected, calc.add(a, b));
}
```

 [\[prgrmmng.com\]](https://prgrmmng.com/parameterized-tests-in-junit-writing-data-driven-tests)

Each CSV row is one test invocation.

### Notes

*   Values are split on commas.
*   Supports automatic type conversion for primitives and Strings.
*   Limited for complex types — for non‑primitives (List, custom classes), use `@MethodSource`.    [\[stackoverflow.com\]](https://stackoverflow.com/questions/46712485/how-to-pass-a-list-as-a-junit5s-parameterized-test-parameter)

***

# ✔ **3. `@MethodSource` — Fully Custom / Complex Input**

When you need complex objects, lists, enums, or multiple arguments of different types, use `@MethodSource`.

### Key Benefits

*   Completely dynamic — method can compute or fetch data.
*   Supports **any Java type**.
*   Method must be `static` unless test lifecycle is PER\_CLASS.    [\[stackoverflow.com\]](https://stackoverflow.com/questions/46712485/how-to-pass-a-list-as-a-junit5s-parameterized-test-parameter)

### Example

```java
@ParameterizedTest
@MethodSource("generateData")
void testUserData(int id, String name, List<String> tags) {
    assertNotNull(name);
    assertTrue(tags.size() > 0);
}

static Stream<Arguments> generateData() {
    return Stream.of(
        Arguments.of(1, "foo", Arrays.asList("a", "b", "c")),
        Arguments.of(2, "bar", Arrays.asList("x", "y", "z"))
    );
}
```

 [\[stackoverflow.com\]](https://stackoverflow.com/questions/46712485/how-to-pass-a-list-as-a-junit5s-parameterized-test-parameter)

> Important: `@CsvSource` **cannot** convert complex types like `List<String>` – `@MethodSource` is required.    [\[stackoverflow.com\]](https://stackoverflow.com/questions/46712485/how-to-pass-a-list-as-a-junit5s-parameterized-test-parameter)

***

# ⭐ **Other Important Providers (Good to Know)**

(Quick summary, in case you need them later.)

| Provider              | Purpose                                        |
| --------------------- | ---------------------------------------------- |
| `@NullSource`         | Supplies a single `null` argument.             |
| `@EmptySource`        | Supplies an empty String, List, Set, or array. |
| `@NullAndEmptySource` | Combines null + empty.                         |
| `@EnumSource`         | Invokes test once per enum constant.           |
| `@CsvFileSource`      | CSV input from external file.                  |

 [\[howtodoinjava.com\]](https://howtodoinjava.com/junit5/parameterized-tests/)

***

# 📘 **Complete Example Combining All Providers**

```java
class DemoParameterizedTests {

    @ParameterizedTest
    @ValueSource(ints = {1, 3, 5, 7})
    void testOddNumbers(int n) {
        assertTrue(n % 2 != 0);
    }

    @ParameterizedTest
    @CsvSource({
        "'john', 25",
        "'alice', 30",
        "'bob', 40"
    })
    void testNameAndAge(String name, int age) {
        assertNotNull(name);
        assertTrue(age > 0);
    }

    @ParameterizedTest
    @MethodSource("userProvider")
    void testUsers(User user) {
        assertNotNull(user.getName());
        assertTrue(user.getAge() > 18);
    }

    static Stream<Arguments> userProvider() {
        return Stream.of(
            Arguments.of(new User("John", 25)),
            Arguments.of(new User("Alice", 30))
        );
    }
}
```

***

# 🎯 **Summary**

*   Parameterized tests let a single test run with different inputs.
*   **`@ValueSource`** → simple literal values.
*   **`@CsvSource`** → multiple column values for each test call.
*   **`@MethodSource`** → dynamic, complex, or computed input (most flexible).
*   JUnit 5 processes each input as a separate test invocation.    [\[baeldung.com\]](https://www.baeldung.com/parameterized-tests-junit-5), [\[docs.junit.org\]](https://docs.junit.org/5.3.0/api/org/junit/jupiter/params/ParameterizedTest.html)

***
### **Layer 6: Advanced Parameterized Testing**

*   argument converters
*   aggregators
*   `ArgumentsAccessor`
*   `@CsvFileSource`
*   custom providers (`ArgumentsProvider`)
