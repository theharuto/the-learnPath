## 1) The “Must‑Know & Most‑Used” Assertions (JUnit Jupiter)

All JUnit Jupiter assertions are **`static` methods** on `org.junit.jupiter.api.Assertions`. You typically **static‑import** them for readability. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

### Equality / Identity / Nullity

*   **`assertEquals(expected, actual)`** — value equality; many overloads for primitives, objects, with optional message or `Supplier<String>` (lazy). [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions), [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html)
*   **`assertNotEquals(unexpected, actual)`** — inequality. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)
*   **`assertSame(expectedRef, actualRef)`** — **same reference**. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-testing/junit-5-assertions/)
*   **`assertNotSame(unexpectedRef, actualRef)`** — **different reference**. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-testing/junit-5-assertions/)
*   **`assertNull(value)`** / **`assertNotNull(value)`** — null / not‑null checks. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/software-testing/junit-5-assertions/)

### Booleans

*   **`assertTrue(condition)`**, **`assertFalse(condition)`** — boolean guards, also support message or `Supplier<String>`. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

### Collections / Arrays / Text

*   **`assertArrayEquals(expected, actual)`** — deep array equality; overloads for primitive and object arrays (including multi‑dimensional). [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html)
*   **`assertIterableEquals(expected, actual)`** — element‑wise equality for iterables (order‑sensitive). [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)
*   **`assertLinesMatch(expectedLines, actualLines)`** — line‑by‑line comparison with simple pattern matching (great for logs). [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

### Exceptions

*   **`assertThrows(type, executable)`** — expects a specific exception; **returns the thrown instance** so you can assert message/cause. Fails if nothing or a different type is thrown. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions), [\[arhohuttunen.com\]](https://www.arhohuttunen.com/junit-5-expected-exception/)
*   **`assertDoesNotThrow(executable)`** — passes if no exception is thrown; returns the result for `Supplier<T>` form. [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html)

### Timeouts

*   **`assertTimeout(Duration, executable)`** — runs the code **in the calling thread**; waits for it to finish and then fails if the total time exceeded the duration. Safe with thread‑locals. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/timeouts)
*   **`assertTimeoutPreemptively(Duration, executable)`** — runs the code **in a separate thread** and **preemptively aborts** when exceeding duration; can have side effects with thread‑local state. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/timeouts)

### Grouping & Misc

*   **`assertAll(heading?, Executable...)`** — **executes all assertions** and **aggregates failures** (no early exit). Very handy for verifying multiple properties together. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions), [\[baeldung.com\]](https://www.baeldung.com/junit5-assertall-vs-multiple-assertions)
*   **`fail(message?)`** — unconditionally fails; useful in branches that must not be reached. [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html)
*   **`assertInstanceOf(type, obj)` / `assertNotInstanceOf(type, obj)`** — type checks (Jupiter convenience). [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

> Tip: The **official Assertions page** in the User Guide lists these methods and their overloads, and explains lazy messages via `Supplier<String>`. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

***

## 2) How these assertions work **internally** (high‑level mechanics)

### a) Who calls whom in a typical test?

1.  **JUnit Platform Launcher** (used by your IDE/build tool) discovers and runs tests via the **JUnit Jupiter TestEngine**. It reflectively invokes your test method. [\[en.wikipedia.org\]](https://en.wikipedia.org/wiki/Unit_testing)
2.  **Your test method** calls a **static** assertion (e.g., `assertEquals`) from `Assertions`. There is **no instance** involved; it’s a utility class. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)
3.  If the assertion fails, it **throws** an exception (see below). The **Jupiter engine** catches that, marks the test as **failed**, and reports the failure back to the Platform/IDE. [\[en.wikipedia.org\]](https://en.wikipedia.org/wiki/Unit_testing)

### b) What exception type is thrown on failure?

*   By design, JUnit 5 uses **OpenTest4J** as a common error model. Most assertion failures throw **`org.opentest4j.AssertionFailedError`** (or a subclass). This stores **expected/actual** values to help tools render nice diffs. [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html), [\[AssertionF...1.3.0 API)\]](http://ota4j-team.github.io/opentest4j/docs/current/api/org/opentest4j/AssertionFailedError.html)
*   **`assertAll`** aggregates multiple failures into a **`org.opentest4j.MultipleFailuresError`** (heading + list of failures). [\[MultipleFa...1.3.0 API)\]](http://ota4j-team.github.io/opentest4j/docs/current/api/org/opentest4j/MultipleFailuresError.html)

### c) How does `assertAll()` execute lambdas?

*   It accepts `Executable` lambdas, **executes each one**, collects thrown assertion failures, and **throws a single `MultipleFailuresError`** at the end if any failed; otherwise returns normally. [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html), [\[MultipleFa...1.3.0 API)\]](http://ota4j-team.github.io/opentest4j/docs/current/api/org/opentest4j/MultipleFailuresError.html)

### d) How does `assertThrows()` behave?

*   It executes the lambda;
    *   If **no exception** is thrown → throws `AssertionFailedError` (“expected X, but nothing was thrown”).
    *   If **wrong type** is thrown → throws `AssertionFailedError` mentioning mismatch.
    *   If **correct type** is thrown → returns the exception instance for further checks (message/cause). [\[arhohuttunen.com\]](https://www.arhohuttunen.com/junit-5-expected-exception/)

### e) How do timeout assertions work?

*   **`assertTimeout`** runs **in the same thread**; it always lets the block finish, then checks elapsed time and **fails after completion** if it exceeded the duration. Good with frameworks using `ThreadLocal` (e.g., Spring transaction context). [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/timeouts)
*   **`assertTimeoutPreemptively`** runs code **in a separate thread** and **aborts early** if the timeout is hit; **can break** code that relies on thread‑locals or the main test thread identity. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/timeouts)

### f) Lazy failure messages (performance)

*   Many assertions accept `Supplier<String>`; JUnit **evaluates it only on failure**, avoiding cost when the test passes. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

***

## 3) Minimal “who does what” diagram (helpful, not overdone)

    IDE/Build Tool
       │
       │  (JUnit Platform Launcher)
       ▼
    JUnit Platform  ──► JUnit Jupiter TestEngine ──► Your @Test method
                                                  │
                                                  └──► Assertions.<static method>()
                                                        │
                                                        ├─ pass → return
                                                        └─ fail → throw AssertionFailedError/MultipleFailuresError

 [\[en.wikipedia.org\]](https://en.wikipedia.org/wiki/Unit_testing), [\[docs.junit.org\]](https://docs.junit.org/5.1.1/api/org/junit/jupiter/api/Assertions.html)

***

## 4) Quick reference you’ll use daily

```java
// equality / identity / nullity
assertEquals(exp, act);      assertNotEquals(unexp, act);
assertSame(expRef, actRef);  assertNotSame(unexpRef, actRef);
assertNull(x);               assertNotNull(x);

// booleans
assertTrue(condition);       assertFalse(condition);

// arrays / iterables / lines
assertArrayEquals(expArr, actArr);
assertIterableEquals(expIt, actIt);
assertLinesMatch(expLines, actLines);

// exceptions
var ex = assertThrows(MyEx.class, () -> call());
assertDoesNotThrow(() -> callThatShouldNotThrow());

// timeouts
assertTimeout(Duration.ofMillis(200), () -> doWork());
assertTimeoutPreemptively(Duration.ofMillis(200), () -> doWork());

// grouping & misc
assertAll("user",
    () -> assertEquals("alice", user.getName()),
    () -> assertTrue(user.isActive())
);
fail("should not reach here");
```

(All methods are on `org.junit.jupiter.api.Assertions`.) [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/assertions)

***

## 5) Things *not* in JUnit assertions (common confusion)

*   **`assertThat(...)`** fluent style is **not part of JUnit 5 core**—it comes from **AssertJ** or **Hamcrest**. You can use them *with* JUnit 5, but they’re external libraries. (See “Third‑party Assertion Libraries” in the User Guide.) [\[en.wikipedia.org\]](https://en.wikipedia.org/wiki/Unit_testing)

***

## 6) Why this matters for you (dev POV)

*   Understanding that **assertions throw `AssertionFailedError`** and the **engine catches/report it** explains why **any** unhandled exception will fail the test and why grouped failures show up as **one** combined error. [\[en.wikipedia.org\]](https://en.wikipedia.org/wiki/Unit_testing), [\[AssertionF...1.3.0 API)\]](http://ota4j-team.github.io/opentest4j/docs/current/api/org/opentest4j/AssertionFailedError.html), [\[MultipleFa...1.3.0 API)\]](http://ota4j-team.github.io/opentest4j/docs/current/api/org/opentest4j/MultipleFailuresError.html)
*   Knowing the **threading** model of timeout assertions helps prevent flaky tests with frameworks using **`ThreadLocal`**. [\[docs.junit.org\]](https://docs.junit.org/current/writing-tests/timeouts)
