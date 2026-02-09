Here’s a clear, practical guide to **`@CsvFileSource`** in JUnit 5 with runnable examples.

***

## What is `@CsvFileSource`?

`@CsvFileSource` is a **parameter provider** for JUnit 5 parameterized tests. It lets you **externalize your test data in one or more CSV files** and feeds each row to your test method as arguments—one invocation per row. You enable parameterized tests via the `junit-jupiter-params` artifact and annotate your test with `@ParameterizedTest`. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)

***

## Why use it?

*   **Separation of data from test logic** (easy to extend cases by editing CSV).
*   **Readable, maintainable** data‑driven tests for multiple inputs/outputs.
*   Works with the full parameterized test engine in JUnit Jupiter. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html), [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/test-driven-development-tdd/)

***

## Setup (dependency)

Add **`junit-jupiter-params`** to your test scope (Maven):

```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter-params</artifactId>
  <version>5.11.0</version>
  <scope>test</scope>
</dependency>
```

Parameterized tests in JUnit 5 require this artifact. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)

***

## Project layout (+ sample CSV)

Put your CSV under `src/test/resources`, e.g.:

    your-project/
    └─ src/
       └─ test/
          ├─ java/
          │  └─ com/example/...
          └─ resources/
             └─ data/
                └─ add_cases.csv

**`add_cases.csv`** (with a header you may want to skip):

```csv
a,b,sum
1,2,3
10,5,15
7,8,15
```

***

## Example 1 — Basic usage with classpath resource

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvFileSource;

import static org.junit.jupiter.api.Assertions.assertEquals;

class CalculatorParamCsvFileTest {

    @ParameterizedTest(name = "[{index}] {0} + {1} = {2}")
    @CsvFileSource(resources = "/data/add_cases.csv", numLinesToSkip = 1)
    void addsCorrectly(int a, int b, int expectedSum) {
        int result = a + b;
        assertEquals(expectedSum, result);
    }
}
```

**Notes**

*   `resources` points to a **classpath** resource (`/` is from `src/test/resources`).
*   `numLinesToSkip = 1` **skips the header row**.
*   JUnit converts CSV tokens to parameter types (`int`, `String`, etc.) automatically. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html), [\[monday.com\]](https://monday.com/blog/rnd/test-driven-development-tdd/)

***

## Example 2 — Multiple CSV files (aggregate suites)

You can feed **multiple files** to the same test by providing an array:

```java
@ParameterizedTest(name = "[{index}] {0} + {1} = {2}")
@CsvFileSource(
    resources = {"/data/add_cases.csv", "/data/add_cases_extra.csv"},
    numLinesToSkip = 1
)
void addsFromMultipleFiles(int a, int b, int expected) {
    assertEquals(expected, a + b);
}
```

This is a simple way to **extend your test matrix** without changing code. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)

***

## Example 3 — Using file paths instead of classpath

If you prefer pointing to files directly (e.g., generated CSV under `target/`), you can use the `files` attribute with relative or absolute paths:

```java
@ParameterizedTest
@CsvFileSource(files = "src/test/resources/data/add_cases.csv", numLinesToSkip = 1)
void addsWithFilePath(int a, int b, int expected) {
    assertEquals(expected, a + b);
}
```

This is functionally similar; use whichever fits your build setup. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)

***

## Example 4 — Multiple columns (realistic domain)

```csv
# src/test/resources/data/users.csv
name,age,active
Alice,30,true
Bob,40,false
```

```java
@ParameterizedTest(name = "[{index}] name={0}, age={1}, active={2}")
@CsvFileSource(resources = "/data/users.csv", numLinesToSkip = 1)
void userRows(String name, int age, boolean active) {
    assertEquals(name.length() > 0, true);
    assertEquals(age > 0, true);
    // maybe your domain rule:
    if (active) {
        // additional checks...
    }
}
```

This demonstrates **type conversion** (e.g., `boolean`) and validates that CSV rows map cleanly to parameters. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)

***

## Practical tips & gotchas

*   **Header rows**: use `numLinesToSkip` to skip headers. (Common pattern in examples.) [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)
*   **Type conversion**: `@CsvFileSource` leverages JUnit’s converters for primitives and Strings; for complex objects or collections, prefer **`@MethodSource`**. [\[ibm.com\]](https://www.ibm.com/think/topics/test-driven-development)
*   **Escaping commas / quotes**: If your values contain commas, enclose them in quotes in the CSV; JUnit’s CSV parsing will treat quoted commas as data, not separators. (General guidance in parameterized test resources; `@CsvSource`/`@CsvFileSource` are designed for CSV parsing.) [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)
*   **Display names**: Customize invocation names with `@ParameterizedTest(name = "...")`, using placeholders like `{index}` and `{arguments}` for nicer reporting. (Placeholders are defined for parameterized tests.) [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/test-driven-development-tdd/)
*   **Dependency reminder**: ensure `junit-jupiter-params` is on the classpath; otherwise parameterized annotations won’t be recognized. [\[guru99.com\]](https://www.guru99.com/test-driven-development.html)

***

## When to choose `@CsvFileSource` vs others

*   Use **`@CsvFileSource`** when your test data is **large, tabular, or maintained by non‑devs** (QA/BA) and you want it **outside the source code**. [\[dev.to\]](https://dev.to/jamescantor38/what-is-test-driven-development-a-beginners-guide-3p34)
*   Use **`@CsvSource`** for **small inline** multi‑arg matrices. [\[dev.to\]](https://dev.to/jamescantor38/what-is-test-driven-development-a-beginners-guide-3p34)
*   Use **`@MethodSource`** for **complex types** or generated/computed arguments. [\[ibm.com\]](https://www.ibm.com/think/topics/test-driven-development)

***

If you want, I can:

*   Convert one of your current tests to `@CsvFileSource`,
*   Or show **error‑handling patterns** (e.g., invalid rows, blank fields),
*   Or add a **Gradle** example and CI notes for classpath resources.

<details>
  <summary>
   <h1>
   Display parameters in report -- doubt  
   </h1>
  </summary>

1.  **What does `name = "[{index}] {0} + {1} = {2}"` mean?**
2.  **Why is it not included in every example?** (Is it mandatory or optional?)

Let’s break this down clearly.

***

# ✅ 1. What does `name = ...` in `@ParameterizedTest` do?

JUnit 5 lets you **customize how each invocation of the parameterized test is displayed** in the test runner (IDE, console, CI logs).

This is done using the **`name` attribute** of `@ParameterizedTest`.

According to the JUnit 5 API, `ParameterizedTest` supports **display name patterns** using placeholders such as:

*   `{index}` – invocation number (0‑based or 1‑based depending on config)
*   `{arguments}` – full list of arguments
*   `{0}`, `{1}`, `{2}`, … – individual argument values    [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/test-driven-development-tdd/)

***

# 📌 Example Explained

### Example:

```java
@ParameterizedTest(name = "[{index}] {0} + {1} = {2}")
```

Assume CSV row:

    2,3,5

Then JUnit shows the test invocation like:

    [1] 2 + 3 = 5

Because:

*   `{index}` → 1 (2nd invocation)
*   `{0}` → value of parameter 0 → 2
*   `{1}` → value of parameter 1 → 3
*   `{2}` → value of parameter 2 → 5

***

### Another Example:

```java
@ParameterizedTest(name = "[{index}] name={0}, age={1}, active={2}")
```

With CSV row:

    "Alice",30,true

Display becomes:

    [2] name=Alice, age=30, active=true

Makes debugging easier because you instantly see:

✔ The index  
✔ The actual arguments used  
✔ Clear readable names

***

# ❗ 2. Is `name = ...` required?

**No — it is completely optional.**

The JUnit docs explicitly say it is an **optional element** on the annotation. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/test-driven-development-tdd/)

If you don’t specify it, JUnit uses the **default display name pattern**, which is also defined in the annotation:

    [{index}] {arguments}

So without specifying `name = ...`, a test might appear as:

    [1] [2, 3, 5]

This is less readable than:

    [1] 2 + 3 = 5

➡️ That’s why examples often include it — to improve clarity.  
➡️ But it’s not required for correctness.

***

# 📌 3. Why didn’t I include it in Example 3?

Because **it is not required**, and sometimes examples focus on the main feature (like `@CsvFileSource`) instead of optional niceties.

For teaching clarity:

*   Example 1 & 2 → Showed how `name = ...` makes reporting better.
*   Example 3 → Focused only on **how to load files**, not display formatting.

But YOU CAN absolutely add it:

```java
@ParameterizedTest(name = "[{index}] a={0}, b={1}, expected={2}")
@CsvFileSource(files = "src/test/resources/data/add_cases.csv", numLinesToSkip = 1)
void addsWithFilePath(int a, int b, int expected) {
    assertEquals(expected, a + b);
}
```

***

# ✔ Final Summary

### **`name = ...` is:**

*   **Optional**
*   **Customizes the display names of each parameterized invocation**
*   Uses placeholders `{index}`, `{arguments}`, `{0}`, `{1}`, `{2}`, etc.  
    → These are officially documented placeholders. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-engineering/test-driven-development-tdd/)

### **Why not always used?**

*   It improves readability but is not necessary for test execution.
*   Some examples focus on test mechanics, not labeling.

***

If you want, I can create a **diagram** showing how JUnit maps the parameters to placeholders, or show **how this looks in IntelliJ / VSCode test runner outputs**.

</details>
