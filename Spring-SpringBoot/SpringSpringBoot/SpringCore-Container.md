# 3️⃣ Spring Core Container

---

# 1️⃣ IoC Overview — How Spring Manages Objects

## First, the Problem

In plain Java:

```java
class OrderService {
    private PaymentService paymentService = new PaymentService();
}
```

You:

* Create objects
* Wire dependencies
* Manage lifecycle

This scales poorly.

---

## What IoC Means in Spring

**Inversion of Control (IoC)** means:

> The framework controls object creation and lifecycle instead of your code.

Instead of:

```java
new PaymentService();
```

Spring does:

* Create PaymentService
* Create OrderService
* Inject PaymentService into OrderService
* Manage both

You only declare relationships.

---

## What the IoC Container Actually Does

At startup:

1. Scans classes / configuration
2. Registers beans
3. Creates objects
4. Injects dependencies
5. Manages lifecycle

Think of it as:

```text
Spring Container = Smart Object Factory + Dependency Graph Manager
```

---

# 2️⃣ BeanFactory vs ApplicationContext

This confusion must be crystal clear.

---

## BeanFactory

### What it is

* The basic IoC container
* Minimal implementation

Responsibilities:

* Create beans
* Inject dependencies
* Manage scopes

### Behavior

* Lazy initialization by default
* No enterprise features

Use case today?

Almost none directly.

Mostly used internally by Spring.

---

## ApplicationContext

### What it is

* Advanced IoC container
* Built on top of BeanFactory

Adds:

* Eager singleton creation
* Event publishing
* AOP integration
* Environment profiles
* Resource loading
* Annotation processing

In real apps:

> ApplicationContext is the container you use.

Spring Boot always creates an ApplicationContext.

---

## Clear Comparison

| Feature             | BeanFactory | ApplicationContext   |
| ------------------- | ----------- | -------------------- |
| Basic DI            | ✅           | ✅                    |
| Lazy by default     | ✅           | ❌ (eager singletons) |
| Events              | ❌           | ✅                    |
| AOP                 | ❌           | ✅                    |
| Annotation scanning | ❌           | ✅                    |
| Production ready    | ❌           | ✅                    |

---

### Mental Shortcut

BeanFactory = engine
ApplicationContext = full car

---

# 3️⃣ Component Scanning

Now we move to how beans are discovered.

---

## What is Component Scanning?

Instead of manually defining beans:

```java
@Bean
public UserService userService() {
    return new UserService();
}
```

Spring can scan packages and auto-detect components.

---

## @Component

```java
@Component
class UserService {}
```

This tells Spring:

> Register this class as a bean.

---

## @Service

```java
@Service
class OrderService {}
```

Same as @Component, but semantically indicates:

> Business service layer

---

## @Repository

```java
@Repository
class UserRepository {}
```

Also a specialization of @Component.

Extra benefit:

* Exception translation for persistence layer.

---

## Important Truth

All of these:

```text
@Component
@Service
@Repository
```

Are functionally component stereotypes.

Difference is semantic + tooling support.

---

## How Scanning Works (High-Level)

1. Spring starts
2. Looks at base package
3. Finds classes annotated with component annotations
4. Registers them as beans

No deep reflection discussion yet.

---

# 4️⃣ Configuration Annotations

Now let’s see how we define beans manually.

---

## @Configuration

```java
@Configuration
class AppConfig {}
```

Marks class as configuration source.

It tells Spring:

> This class contains bean definitions.

---

## @Bean

Inside @Configuration:

```java
@Configuration
class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new PaymentService();
    }
}
```

Spring registers the return value as a bean.

Equivalent to:

```text
Create PaymentService
Manage it as a bean
```

---

## When Do We Use @Bean?

Use @Bean when:

* You don’t control the class (third-party library)
* You need custom construction logic
* You need conditional configuration
* You want explicit bean creation

---

## Component vs @Bean — Difference

| Aspect     | @Component       | @Bean                      |
| ---------- | ---------------- | -------------------------- |
| Applied to | Class            | Method                     |
| Discovery  | Component scan   | Explicit configuration     |
| Use case   | Your own classes | Third-party / custom logic |

---

# 5️⃣ Big Picture Flow

Here’s how it connects:

```text
@Configuration
@ComponentScan
     ↓
Find @Component classes
     ↓
Register beans
     ↓
ApplicationContext
     ↓
Inject dependencies
     ↓
App ready
```

---

# 6️⃣ Real-World Mental Model

In a typical Spring app:

* Controllers → @RestController
* Services → @Service
* Repositories → @Repository
* Config → @Configuration + @Bean

All managed by ApplicationContext.

You rarely touch BeanFactory directly.

---

# 7️⃣ Common Beginner Mistakes

❌ Thinking @Component and @Bean are interchangeable
❌ Not understanding container difference
❌ Assuming Spring creates objects magically
❌ Forgetting base package scanning

Correct understanding:

Spring builds a container, registers beans, and wires dependencies systematically.

---

# 📘 Revision Notes — Topic 3

---

## 📌 Spring Core Container

### IoC (Inversion of Control)

Spring container manages:

* Object creation
* Dependency injection
* Lifecycle management

Developers declare relationships; Spring handles instantiation.

---

### BeanFactory

* Basic IoC container
* Lazy initialization by default
* Minimal features

---

### ApplicationContext

* Advanced container built on BeanFactory
* Eager singleton creation
* Supports AOP, events, profiles
* Used in production applications

---

### Component Scanning

Spring scans packages for:

* @Component
* @Service
* @Repository

Registers detected classes as beans.

---

### @Configuration and @Bean

* @Configuration → marks configuration class
* @Bean → defines a bean via method

Used for:

* Third-party libraries
* Custom bean creation
* Explicit configuration

---

### Mental Model

Spring Container = IoC engine
ApplicationContext = Production-ready container
Beans = Managed objects
