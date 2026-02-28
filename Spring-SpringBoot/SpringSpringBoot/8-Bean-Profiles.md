1. Why Profiles exist
2. How Spring models environments
3. Activating profiles
4. Using `@Profile` on beans
5. Real-world patterns

---

# 8️⃣ Spring Profiles

---

# 1️⃣ Why Spring Profiles Exist

Real applications run in multiple environments:

* dev
* test
* staging
* prod

Each environment may need:

* Different database
* Different logging level
* Different API keys
* Mock vs real integrations

Without profiles, you would:

* Hardcode environment checks
* Use if/else everywhere
* Maintain multiple config classes
* Risk deploying wrong configuration

Profiles solve this cleanly.

---

# 2️⃣ What is a Spring Profile?

A **Profile** is:

> A named logical group of bean definitions that are activated only in specific environments.

You define profiles like:

```text
dev
test
prod
cloud
docker
```

Spring activates only the beans matching the active profile.

---

# 3️⃣ How Spring Models Environment

Spring has an internal abstraction:

```text
Environment
    ├── Active profiles
    ├── Default profiles
    └── Property sources
```

When the container starts:

1. It reads active profiles
2. Registers only matching beans
3. Skips others

---

# 4️⃣ Using @Profile on Beans

## Example: Dev vs Prod Database

```java
@Configuration
class DataConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new H2DataSource();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        return new PostgreSQLDataSource();
    }
}
```

If `dev` is active:

* Only `devDataSource` is registered.

If `prod` is active:

* Only `prodDataSource` is registered.

The other does not exist in the container.

---

## Profile on Class Level

```java
@Profile("dev")
@Service
class DevPaymentService implements PaymentService {}
```

Spring registers this bean only if `dev` is active.

---

# 5️⃣ Activating Profiles

There are multiple ways.

---

## 1️⃣ Using Property: spring.profiles.active

In application.properties:

```properties
spring.profiles.active=dev
```

Or environment variable:

```bash
SPRING_PROFILES_ACTIVE=prod
```

Or JVM argument:

```bash
-Dspring.profiles.active=prod
```

This is the most common way.

---

## 2️⃣ Programmatically

```java
context.getEnvironment().setActiveProfiles("dev");
```

Rare in production — mostly testing.

---

# 6️⃣ Default Profile

If no profile is active:

Spring uses the `default` profile.

You can define:

```java
@Profile("default")
@Bean
public DataSource defaultDataSource() { ... }
```

Useful fallback.

---

# 7️⃣ Multiple Active Profiles

You can activate multiple:

```properties
spring.profiles.active=dev,cloud
```

Spring will register beans matching either profile.

This is useful when combining concerns:

* dev + docker
* prod + cloud

---

# 8️⃣ Real-World Use Cases

Profiles are commonly used for:

* Database configuration
* External API clients
* Feature toggles
* Caching strategies
* Mock vs real services
* Logging configuration

---

# 9️⃣ Clean Architecture Pattern with Profiles

Example:

```text
PaymentService (interface)
   ├── DevPaymentService   (@Profile("dev"))
   └── ProdPaymentService  (@Profile("prod"))
```

OrderService depends on PaymentService.

Spring injects correct implementation based on active profile.

No if/else in business logic.

Clean separation.

---

# 🔟 Common Mistakes

❌ Using profiles for business logic branching
❌ Overusing profiles for minor config differences
❌ Forgetting to set profile in production
❌ Hardcoding environment checks manually

Profiles should control configuration, not application behavior logic.

---

# 1️⃣1️⃣ Mental Model

Think of profiles as:

```text
Conditional bean registration based on environment.
```

Spring does:

```text
If profile matches → register bean
Else → ignore
```

The bean literally does not exist if profile is inactive.

---

# 📘 Revision Notes — Topic 8

---

## 📌 Spring Profiles

### What is a Profile?

A profile is a named configuration group that allows environment-specific bean registration.

---

### Why Use Profiles?

To manage:

* Different environments (dev, test, prod)
* Different infrastructure configurations
* Mock vs real implementations

---

### Using @Profile

On method:

```java
@Bean
@Profile("dev")
```

On class:

```java
@Profile("prod")
@Service
```

Bean is registered only if the profile is active.

---

### Activating Profiles

Common ways:

* application.properties
  `spring.profiles.active=dev`

* JVM argument
  `-Dspring.profiles.active=prod`

* Environment variable
  `SPRING_PROFILES_ACTIVE=prod`

---

### Multiple Profiles

You can activate multiple profiles at once:

```
spring.profiles.active=dev,cloud
```

---

### Best Practices

* Use profiles for environment configuration
* Avoid business logic branching with profiles
* Keep profile usage clean and structured
* Always verify active profile in production

---

You’ve now covered:

1. Spring introduction
2. Core terminology
3. Core container
4. Beans
5. Dependency Injection
6. Annotations
7. Configuration
8. Profiles

Next step options:

9️⃣ Introduction to Spring Boot
🔟 Mini practical example combining everything
🔁 Quick recap + gap analysis before Boot
