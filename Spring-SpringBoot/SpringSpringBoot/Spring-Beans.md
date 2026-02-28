# 1️⃣ Defining Beans

There are **two main ways** you will define beans in modern Spring:

---

## A) Annotation-Based (Component Scanning)

```java
@Component
class UserService {}
```

Or specialized stereotypes:

```java
@Service
@Repository
@Controller
```

Spring scans the package and registers these as beans.

### When to Use

* Your own classes
* Layered architecture (service, repository, controller)

---

## B) Java-Based Configuration (@Configuration + @Bean)

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

* You explicitly define how the bean is created.
* The return value becomes a managed bean.

### When to Use

* Third-party classes (you cannot annotate them)
* Custom construction logic
* Conditional creation
* Infrastructure configuration

---

### Practical Rule

Use:

* `@Component` for application classes
* `@Bean` for external / complex configuration

---

# 2️⃣ Bean Lifecycle

Now we answer:

> What happens to a bean from creation to destruction?

---

## High-Level Lifecycle

```text
1. Container starts
2. Bean is instantiated
3. Dependencies injected
4. Initialization callback
5. Bean is ready
6. Context shutdown
7. Destruction callback
```

That’s the practical lifecycle you must understand.

---

## Initialization — @PostConstruct

```java
@Component
class CacheService {

    @PostConstruct
    public void init() {
        System.out.println("Initializing cache...");
    }
}
```

Runs:

* After dependency injection
* Before the bean is used

Use cases:

* Load cache
* Validate configuration
* Initialize connections

Important:

* Runs once for singleton
* Runs every time for prototype (when created)

---

## Destruction — @PreDestroy

```java
@PreDestroy
public void cleanup() {
    System.out.println("Cleaning resources...");
}
```

Runs:

* When ApplicationContext shuts down

Used for:

* Closing connections
* Releasing resources
* Flushing buffers

Important:

* Not called for prototype beans
* Container does not manage prototype destruction

---

# 3️⃣ Custom Init and Destroy Methods

Instead of annotations, you can configure explicitly:

```java
@Configuration
class AppConfig {

    @Bean(initMethod = "start", destroyMethod = "stop")
    public Server server() {
        return new Server();
    }
}
```

Inside the class:

```java
class Server {

    public void start() {
        System.out.println("Server starting...");
    }

    public void stop() {
        System.out.println("Server stopping...");
    }
}
```

Spring calls:

* `start()` after injection
* `stop()` during shutdown

---

# 4️⃣ BeanPostProcessor (Intermediate Understanding)

This is powerful but keep it simple.

## What Is It?

A BeanPostProcessor allows you to:

> Intercept beans before and after initialization.

Example structure:

```java
@Component
class CustomBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

Spring calls these methods for every bean.

---

## Why It Matters

BeanPostProcessor is how:

* AOP works
* Transactions are applied
* Proxies are created
* Security wrappers are added

You rarely write one manually in beginner stages, but you must know:

> Many Spring features rely on BeanPostProcessor internally.

---

# 5️⃣ Bean Scopes

Scope answers:

> How many instances of this bean exist?

---

## 1. Singleton (Default)

```java
@Scope("singleton")
```

One instance per ApplicationContext.

Important:

* Not JVM singleton
* One per container

Best for:

* Stateless services

---

## 2. Prototype

```java
@Scope("prototype")
```

New instance every time it’s requested.

Important:

* Spring creates it
* But does NOT manage full lifecycle
* @PreDestroy not called

Use case:

* Stateful objects
* Per-use objects

---

## 3. Web Scopes (Web Applications Only)

Available only in web context.

---

### Request Scope

```java
@Scope("request")
```

One bean per HTTP request.

---

### Session Scope

```java
@Scope("session")
```

One bean per HTTP session.

---

### Application Scope

```java
@Scope("application")
```

One per servlet context.

---

# Important Production Advice

Singleton beans:

* Must be stateless
* Must be thread-safe

If you put mutable state inside singleton:

* You create race conditions.

Most real production bugs come from this mistake.

---

# Lifecycle + Scope Interaction (Very Important)

| Scope     | @PostConstruct | @PreDestroy |
| --------- | -------------- | ----------- |
| Singleton | Once           | Yes         |
| Prototype | Every creation | ❌ No        |
| Request   | Per request    | Yes         |
| Session   | Per session    | Yes         |

Prototype beans are not fully managed after creation.

---

# 📘 Revision Notes — Topic 4

---

## 📌 Spring Beans

### Defining Beans

1. Annotation-Based

   * @Component
   * @Service
   * @Repository

2. Java-Based Configuration

   * @Configuration
   * @Bean

Use @Bean for third-party or custom construction logic.

---

### Bean Lifecycle

1. Instantiation
2. Dependency Injection
3. Initialization (@PostConstruct or initMethod)
4. Bean ready
5. Destruction (@PreDestroy or destroyMethod)

---

### BeanPostProcessor

* Allows intercepting beans before/after initialization
* Used internally for AOP, transactions, security

---

### Bean Scopes

1. Singleton (default)
2. Prototype
3. Request (web)
4. Session (web)
5. Application (web)

---

### Best Practices

* Use singleton for stateless services
* Avoid mutable state in singleton beans
* Use constructor injection
* Prototype beans are not fully lifecycle-managed

<details>
  <summary>
    <h1>BeanPostPorcessor</h1>
  </summary>

  # 🌱 First: Where It Fits in Spring

In the Spring Framework, the container:

1. Creates beans
2. Injects dependencies
3. Initializes them
4. Makes them ready for use

A **BeanPostProcessor (BPP)** lets you plug into this lifecycle and modify beans **before and after initialization**.

Think of it as:

> 🏭 A factory inspector that can adjust or wrap every object before it leaves the factory.

---

# 📦 What Is BeanPostProcessor?

It is an interface:

```java
public interface BeanPostProcessor {

    Object postProcessBeforeInitialization(Object bean, String beanName);

    Object postProcessAfterInitialization(Object bean, String beanName);
}
```

You implement this interface to:

* Modify beans
* Wrap them with proxies
* Inject extra behavior
* Log information
* Perform validation

---

# 🔄 When Does It Run?

For each bean, Spring does:

1. Instantiate bean
2. Inject dependencies
3. 🔹 `postProcessBeforeInitialization()`
4. Run `@PostConstruct`
5. 🔹 `postProcessAfterInitialization()`
6. Bean is ready

So BPP runs around the initialization phase.

---

# 🧠 Why Is It Powerful?

Because it applies to **every bean in the container**.

You don’t modify a specific bean — you intercept them globally.

Many core Spring features are implemented using BeanPostProcessor, including:

* `@Autowired`
* `@Transactional`
* `@Async`
* AOP proxies

---

# 🧩 Simple Example

### Custom BeanPostProcessor

```java
@Component
public class MyBeanProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before Init: " + beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After Init: " + beanName);
        return bean;
    }
}
```

Now every bean will print logs when created.

---

# 🎭 Real Power: Wrapping a Bean

In `postProcessAfterInitialization`, you can return:

* The same bean
* A modified bean
* A completely new object
* A proxy

Example idea:

```java
if(bean instanceof PaymentService) {
    return createProxy(bean);
}
```

Now Spring will use your proxy instead of the original bean.

This is how:

* AOP works
* `@Transactional` works

---

# 📌 Important Details

## 1️⃣ It must be a Spring bean

Your processor itself must be registered (usually with `@Component`).

## 2️⃣ Order matters

If multiple BPPs exist, they can implement:

```java
Ordered
```

to control execution order.

## 3️⃣ It works at container level

It does NOT run per method call.
It runs during bean creation.

---

# 🆚 Difference From BeanFactoryPostProcessor

| BeanPostProcessor        | BeanFactoryPostProcessor          |
| ------------------------ | --------------------------------- |
| Works on bean instances  | Works on bean definitions         |
| After object creation    | Before object creation            |
| Can modify actual object | Can modify configuration metadata |

---

# 🎯 Mental Model (Simple)

Imagine Spring is making cars 🚗:

* Bean definition = car blueprint
* Bean = actual car
* BeanPostProcessor = final inspection team that can:

  * Paint it
  * Add tracking system
  * Replace engine
  * Wrap it in protection

---

# 🧠 Why Spring Uses It Internally

For example:

### `@Autowired`

Spring has an internal BeanPostProcessor that:

* Scans fields
* Finds `@Autowired`
* Injects dependencies using reflection

Without BPP, annotation-based configuration wouldn’t work.

---

# 🚀 Summary

BeanPostProcessor:

* Is a lifecycle hook
* Runs before and after initialization
* Can modify or replace beans
* Is used internally for major Spring features
* Works globally across the container

---

next explain:

* How `@Transactional` specifically uses BeanPostProcessor
* The exact bean lifecycle diagram
* Or a deep-dive into proxy creation

</details>
