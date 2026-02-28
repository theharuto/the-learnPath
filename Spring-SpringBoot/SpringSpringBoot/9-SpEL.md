1. What SpEL is
2. Where it is used
3. Property access
4. Method calls & calculations
5. Conditional expressions
6. Using SpEL with `@Value`
7. When NOT to use SpEL

---

# 9️⃣ Spring Expression Language (SpEL)

---

# 1️⃣ What is SpEL?

SpEL (Spring Expression Language) is:

> A powerful expression language used to query and manipulate objects at runtime within the Spring container.

It allows:

* Accessing bean properties
* Calling methods
* Performing calculations
* Using conditional logic
* Injecting dynamic values

Syntax:

```text
#{ expression }
```

Important:

* `${}` → property placeholder
* `#{}` → SpEL expression

Do not confuse them.

---

# 2️⃣ Where SpEL Is Used

SpEL appears in:

* `@Value`
* XML configuration (legacy)
* Security expressions
* Conditional bean creation
* Caching expressions

Most common usage:

```java
@Value("#{someBean.someProperty}")
```

---

# 3️⃣ Accessing Bean Properties

Example bean:

```java
@Component
class AppConfig {
    private String name = "SpringApp";

    public String getName() {
        return name;
    }
}
```

Using SpEL:

```java
@Component
class Consumer {

    @Value("#{appConfig.name}")
    private String appName;
}
```

Spring:

* Finds bean `appConfig`
* Calls `getName()`
* Injects value

---

# 4️⃣ Calling Methods & Calculations

SpEL can call methods:

```java
@Value("#{appConfig.name.toUpperCase()}")
private String upperName;
```

It can do calculations:

```java
@Value("#{5 * 10}")
private int result;
```

Injects `50`.

You can combine expressions:

```java
@Value("#{2 + 3 * 4}")
private int value;
```

Result: `14`

Operator precedence applies.

---

# 5️⃣ Conditional Expressions (Ternary)

SpEL supports conditional logic.

Example:

```java
@Value("#{appConfig.name == 'SpringApp' ? 'YES' : 'NO'}")
private String status;
```

Equivalent to Java ternary operator.

This is often used in configuration decisions.

---

# 6️⃣ Accessing Properties and System Values

SpEL can access system properties:

```java
@Value("#{systemProperties['os.name']}")
private String osName;
```

Or environment variables:

```java
@Value("#{systemEnvironment['JAVA_HOME']}")
private String javaHome;
```

---

# 7️⃣ Using SpEL with @Value

There are two different patterns:

---

## A) Property Placeholder (Not SpEL)

```java
@Value("${app.name}")
private String appName;
```

This reads from:

* application.properties
* environment variables
* property sources

No expression evaluation.

---

## B) SpEL Expression

```java
@Value("#{2 * 5}")
private int value;
```

Evaluates expression at runtime.

---

## Combining Both

```java
@Value("#{${app.multiplier} * 2}")
private int computedValue;
```

If:

```properties
app.multiplier=5
```

Result → `10`

This mixes property placeholder with SpEL.

---

# 8️⃣ Accessing Other Beans

You can inject one bean's property into another:

```java
@Value("#{orderService.timeout}")
private int timeout;
```

But be careful:

This creates hidden coupling.

Prefer constructor injection instead.

---

# 9️⃣ Collection & Map Access

SpEL supports collections:

```java
@Value("#{myList[0]}")
private String firstElement;
```

Map access:

```java
@Value("#{myMap['key']}")
private String value;
```

---

# 🔟 When NOT to Use SpEL

SpEL is powerful, but:

❌ Do not use it for business logic
❌ Do not overuse it in configuration
❌ Do not create unreadable expressions
❌ Avoid complex conditional chains

Use it for:

* Simple dynamic values
* Configuration logic
* Security expressions
* Minor calculations

If logic becomes complex → move to Java code.

---

# 1️⃣1️⃣ Mental Model

Think of SpEL as:

```text
Runtime mini-expression engine inside Spring.
```

It evaluates expressions during:

* Bean initialization
* Configuration processing

It is not meant to replace Java logic.

---

# 1️⃣2️⃣ Common Mistakes

❌ Confusing `${}` and `#{}`
❌ Writing complex business logic in SpEL
❌ Creating circular references via SpEL
❌ Overusing SpEL when constructor injection is cleaner

Keep it simple.

---

# 📘 Revision Notes — Topic 9

---

## 📌 Spring Expression Language (SpEL)

### What is SpEL?

An expression language used by Spring to evaluate expressions at runtime.

Syntax:

```
#{ expression }
```

---

### Property Access

```java
@Value("#{bean.property}")
```

---

### Method Calls

```java
@Value("#{bean.method()}")
```

---

### Calculations

```java
@Value("#{5 * 10}")
```

---

### Conditional Expressions

```java
@Value("#{condition ? 'A' : 'B'}")
```

---

### Property Placeholder vs SpEL

* `${}` → Property resolution
* `#{}` → Expression evaluation

---

### Best Practices

* Keep expressions simple
* Avoid business logic in SpEL
* Prefer constructor injection for dependencies
* Use for configuration-level logic only
