1. Annotation-based configuration
2. Java-based configuration
3. Mixing both properly (real-world style)
4. Structural best practices

No XML — modern Spring focuses on Java + annotations.

---

# 7️⃣ Bean Configuration

---

# 1️⃣ Annotation-Based Configuration

This is the most common approach in modern Spring applications.

## What It Means

You annotate classes directly:

```java
@Service
class OrderService {}
```

Spring discovers them using **component scanning**.

---

## How It Gets Activated

```java
@Configuration
@ComponentScan("com.example")
class AppConfig {}
```

Spring:

1. Scans package
2. Finds classes annotated with:

   * @Component
   * @Service
   * @Repository
   * @Controller
3. Registers them as beans

---

## When To Use Annotation-Based Config

Use this for:

* Application logic
* Controllers
* Services
* Repositories
* Internal utilities

It keeps configuration minimal and clean.

---

# 2️⃣ Java-Based Configuration

Instead of annotating the class itself, you define beans explicitly using methods.

```java
@Configuration
class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }
}
```

Here:

* The method defines the bean
* The return value becomes the managed object

---

## When To Use Java-Based Configuration

Use @Bean when:

1. You don’t control the class (third-party library)
2. Complex creation logic is required
3. You need conditional or environment-based creation
4. Infrastructure configuration (DB, HTTP client, cache, etc.)

Example:

```java
@Bean
public DataSource dataSource() {
    // custom setup
    return new HikariDataSource();
}
```

You cannot annotate HikariDataSource directly — so you configure it.

---

# 3️⃣ Mixing Configurations (Real-World Practice)

In real applications, you **always mix both**.

Example structure:

```text
com.example
 ├── controller  (@Controller)
 ├── service     (@Service)
 ├── repository  (@Repository)
 ├── config      (@Configuration + @Bean)
```

Application logic → annotation-based
Infrastructure → Java-based

---

## Example Combined Setup

```java
@Configuration
@ComponentScan("com.example")
class AppConfig {

    @Bean
    public ExternalClient externalClient() {
        return new ExternalClient();
    }
}
```

```java
@Service
class OrderService {

    private final ExternalClient externalClient;

    public OrderService(ExternalClient externalClient) {
        this.externalClient = externalClient;
    }
}
```

Spring:

* Scans OrderService
* Registers ExternalClient via @Bean
* Injects automatically

That’s flexible and clean.

---

# 4️⃣ How Spring Sees Configuration

Internally (conceptually):

```text
@Configuration
    ↓
@Bean methods → register definitions

@ComponentScan
    ↓
Find annotated classes → register definitions

All definitions
    ↓
Container builds beans
```

Everything becomes a **bean definition** eventually.

---

# 5️⃣ Common Configuration Patterns

### 1. Central AppConfig

Single configuration class:

```java
@Configuration
@ComponentScan("com.example")
class AppConfig {}
```

Simple apps.

---

### 2. Layered Config

```java
@Configuration
class WebConfig {}

@Configuration
class DataConfig {}

@Configuration
class SecurityConfig {}
```

Better for large applications.

---

### 3. Importing Configurations

```java
@Configuration
@Import({WebConfig.class, DataConfig.class})
class AppConfig {}
```

Allows modular structure.

---

# 6️⃣ Why Java-Based Configuration Is Preferred Over XML

* Type-safe
* Refactor-friendly
* IDE-supported
* Cleaner
* More readable
* Less verbose

Modern Spring avoids XML.

---

# 7️⃣ Configuration Best Practices

1. Keep business logic separate from configuration.
2. Group infrastructure beans inside config packages.
3. Use constructor injection consistently.
4. Avoid mixing too much logic inside @Configuration.
5. Keep configuration deterministic and explicit.

---

# 8️⃣ Common Mistakes

❌ Putting heavy business logic inside @Configuration
❌ Overusing @Bean for everything
❌ Forgetting component scanning base package
❌ Creating circular configuration dependencies

---

# 9️⃣ Mental Model

Think of configuration as:

```text
Application Logic → Annotate classes
Infrastructure → Define via @Bean
Container → Merges both into one graph
```

Spring doesn’t care how beans are defined.
Everything becomes a managed object in the container.

---

# 📘 Revision Notes — Topic 7

---

## 📌 Bean Configuration

### Annotation-Based Configuration

Uses:

* @Component
* @Service
* @Repository
* @Controller

Enabled via:

* @ComponentScan

Best for:

* Application-level classes

---

### Java-Based Configuration

Uses:

* @Configuration
* @Bean

Best for:

* Third-party classes
* Infrastructure setup
* Complex bean creation

---

### Mixing Configurations

Common approach:

* Annotate application classes
* Use @Bean for infrastructure
* Organize configuration into modular classes

---

### Best Practices

* Separate config from business logic
* Keep configuration modular
* Use constructor injection
* Prefer Java config over XML

---

Next topic options:

8️⃣ Spring Profiles
9️⃣ Introduction to Spring Boot
🔟 Small working example tying all core topics together

<details>
  <summary>
    <h1>is @ComponentScan always needed</h1>
  </summary>
  
  Short answer: **No, not always — but usually yes (unless something else is already doing it for you).**

  # 🌱 What `@ComponentScan` Does

In the Spring Framework, annotations like:

* `@Component`
* `@Service`
* `@Repository`
* `@Controller`

do **nothing by themselves**.

They only work if Spring **scans the package** and detects them.

That scanning is enabled by:

```java
@ComponentScan("com.example")
```

Without scanning, Spring will not discover those beans.

---

# 🧠 So Is It Mandatory?

## ✅ Case 1: Plain Spring (No Boot)

If you are using classic Spring (no Boot):

```java
@Configuration
@ComponentScan("com.example")
class AppConfig { }
```

👉 YES — you must add `@ComponentScan`
Otherwise, `@Service`, `@Controller`, etc. will not be registered.

---

## ✅ Case 2: Spring Boot

If you use:

```java
@SpringBootApplication
public class Application { }
```

In Spring Boot, this annotation already includes:

```java
@Configuration
@EnableAutoConfiguration
@ComponentScan
```

So in Spring Boot:

👉 NO — you do NOT need to add `@ComponentScan` manually (in most cases).

It automatically scans the package of the main class and sub-packages.

---

# 📦 Your Example

```java
@Configuration
@ComponentScan("com.example")
class AppConfig {

    @Bean
    public ExternalClient externalClient() {
        return new ExternalClient();
    }
}
```

Here:

* `@Bean` works regardless of component scanning.
* But `@Service`, `@Controller`, etc. require `@ComponentScan`.

Because `@Bean` is explicitly registered inside a `@Configuration` class.

---

# 🔎 Important Rule About Package Structure

Spring scans:

```
Base package + sub-packages
```

So if your main class is in:

```
com.example
```

It will scan:

```
com.example.*
```

But NOT:

```
com.otherpackage
```

If your components are outside the base package, you must explicitly configure scanning.

---

# 🎯 When You Do NOT Need `@ComponentScan`

You don’t need it if:

1. You use Spring Boot with `@SpringBootApplication`
2. All beans are defined manually using `@Bean`
3. You register beans programmatically

---

# 🧠 Mental Model

Think of `@ComponentScan` as:

> 🔎 "Spring, please go search for annotated classes in this package."

If you don’t ask Spring to search, it won’t.

---

# 🚀 Final Answer

| Scenario                              | Need `@ComponentScan`?         |
| ------------------------------------- | ------------------------------ |
| Plain Spring                          | ✅ Yes                          |
| Spring Boot                           | ❌ Usually No                   |
| Only using `@Bean` methods            | ❌ No                           |
| Using `@Service`, `@Controller`, etc. | ✅ Yes (unless Boot auto-scans) |

---

If you'd like, I can also explain:

* Why package placement of your main class matters
* How component scanning works internally
* Or what happens if two beans of same type are found
</details>
