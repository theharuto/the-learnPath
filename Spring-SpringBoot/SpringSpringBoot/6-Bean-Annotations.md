# 6️⃣ Spring Annotations

Spring annotations can be grouped into:

1. Stereotype (Component) Annotations
2. Configuration Annotations
3. Bean Behavior Annotations

---

# 1️⃣ Core Stereotype Annotations

These mark classes as Spring-managed components.

---

## 🔹 @Component

```java
@Component
class UserService {}
```

Meaning:

> Register this class as a Spring bean.

It is the most generic stereotype annotation.

Used when:

* No specific semantic layer applies.

---

## 🔹 @Service

```java
@Service
class OrderService {}
```

Specialization of `@Component`.

Indicates:

* Business logic layer

Functionally same as `@Component`, but semantically clearer.

---

## 🔹 @Repository

```java
@Repository
class UserRepository {}
```

Also extends `@Component`.

Extra benefit:

* Enables exception translation for persistence layer
  (Converts low-level database exceptions into Spring's unified DataAccessException hierarchy)

Used for:

* DAO / database access layer

---

## 🔹 @Controller

```java
@Controller
class UserController {}
```

Marks:

* Web layer component
* Used in Spring MVC

Registers the class as a web controller.

---

## Important Truth

All of these are built on top of:

```text
@Component
```

They exist for:

* Layered architecture clarity
* Tooling support
* Framework-level behavior (like exception translation)

---

# 2️⃣ Configuration Annotations

These control how beans are defined and managed.

---

## 🔹 @Configuration

```java
@Configuration
class AppConfig {}
```

Marks:

* A class that contains bean definitions.

Spring treats this class specially.

Used with:

* @Bean methods

---

## 🔹 @Bean

```java
@Configuration
class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }
}
```

Meaning:

> The return value of this method should be registered as a bean.

Use cases:

* Third-party classes
* Custom construction logic
* Infrastructure configuration

---

## Component vs @Bean — Clear Distinction

| @Component                 | @Bean                            |
| -------------------------- | -------------------------------- |
| Applied to class           | Applied to method                |
| Auto-detected via scanning | Explicit configuration           |
| For your own classes       | For external or complex creation |

---

# 3️⃣ Bean Behavior Annotations

These control how beans behave inside the container.

---

## 🔹 @Scope

Defines how many instances exist.

Example:

```java
@Component
@Scope("prototype")
class MyBean {}
```

Common scopes:

* singleton (default)
* prototype
* request (web)
* session (web)
* application (web)

Use prototype when:

* You need a new instance every time.

Be careful:
Singleton beans must be stateless.

---

## 🔹 @Lazy

Controls initialization timing.

```java
@Component
@Lazy
class HeavyService {}
```

Meaning:

* Do not create this bean at startup.
* Create it only when first requested.

Use case:

* Expensive initialization
* Rarely used components

Can also be used at injection point:

```java
public OrderService(@Lazy PaymentService paymentService) { }
```

This injects a proxy that initializes lazily.

---

# 4️⃣ Putting It Together — Real Structure Example

```java
@Controller
class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

```java
@Service
class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

```java
@Repository
class UserRepository {}
```

```java
@Configuration
class AppConfig {

    @Bean
    public ExternalClient externalClient() {
        return new ExternalClient();
    }
}
```

This is clean layered Spring architecture.

---

# 5️⃣ Common Mistakes

❌ Using @Component everywhere without structure
❌ Putting business logic in @Controller
❌ Using @Bean when @Component would suffice
❌ Ignoring @Scope implications
❌ Overusing @Lazy

---

# 6️⃣ Mental Model

Think of annotations as:

```text
@Component → Register me
@Service → Business layer bean
@Repository → Data layer bean
@Controller → Web layer bean

@Configuration → Configuration class
@Bean → Register this method's return as bean

@Scope → How many instances?
@Lazy → When to create?
```

Nothing magical — just metadata for the container.

---

# 📘 Revision Notes — Topic 6

---

## 📌 Spring Annotations

### Core Stereotypes

* @Component → Generic bean
* @Service → Business logic layer
* @Repository → Persistence layer (with exception translation)
* @Controller → Web layer component

All are specializations of @Component.

---

### Configuration Annotations

* @Configuration → Marks configuration class
* @Bean → Registers method return value as bean

Use @Bean for third-party or custom bean creation.

---

### Bean Behavior

* @Scope → Defines bean scope (singleton, prototype, request, session)
* @Lazy → Delays bean initialization

---

### Best Practices

* Use proper stereotype for clarity
* Keep layers clean
* Prefer singleton + stateless services
* Use @Bean for external integrations
* Avoid unnecessary @Lazy usage

---

Next topic options:

7️⃣ Spring Profiles
8️⃣ Introduction to Spring Boot
9️⃣ Mini project combining everything
🔟 Deep dive into bean lifecycle internals

Tell me the next step.
