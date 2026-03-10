# 1. Spring Bean Lifecycle (Very Important)

Most developers only know:

```
@PostConstruct
@PreDestroy
```

But the **actual lifecycle is bigger**.

### Full lifecycle flow

```
1. Spring container starts
2. Bean Definition loaded
3. Bean instantiated
4. Dependencies injected
5. BeanNameAware
6. BeanFactoryAware
7. BeanPostProcessor (before init)
8. @PostConstruct
9. InitializingBean.afterPropertiesSet()
10. custom init method
11. BeanPostProcessor (after init)
12. Bean ready for use
13. @PreDestroy
14. DisposableBean.destroy()
15. custom destroy method
```

---

### Example

```java
@Component
class PaymentService {

    @PostConstruct
    public void init() {
        System.out.println("Bean initialized");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("Bean destroyed");
    }
}
```

---

### Production Insight

Use lifecycle hooks for:

* cache initialization
* loading configuration
* opening connections
* validating dependencies

Avoid heavy logic here.

---

### Interview Trap

Question:

**When does @PostConstruct execute?**

Correct answer:

```
After dependency injection but before bean is ready.
```

---

# 2. Spring Proxy & AOP Internals

Spring **does not modify your classes directly**.

Instead it creates **proxy objects**.

Example:

```
UserService
```

Spring creates:

```
UserServiceProxy
```

Call flow:

```
Controller
   ↓
Proxy
   ↓
Logging / Transaction
   ↓
Actual method
```

---

### Two Proxy Types

#### 1. JDK Dynamic Proxy

Used when **interface exists**

Example:

```
interface UserService
class UserServiceImpl
```

Proxy created for interface.

---

#### 2. CGLIB Proxy

Used when **no interface**

Spring creates **subclass dynamically**.

```
class UserService
```

Proxy:

```
class UserService$$EnhancerBySpring
```

---

### Production Rule

Prefer **interface-based design**.

Why?

* better testing
* proxy friendly
* clean architecture

---

# 3. @Transactional Internals (Critical Concept)

Most developers think:

```
@Transactional = magic
```

Actually it works using **AOP Proxy**.

Flow:

```
Controller
   ↓
Transaction Proxy
   ↓
Begin Transaction
   ↓
Service Method
   ↓
Commit / Rollback
```

---

### Example

```java
@Transactional
public void transfer() {
    debit();
    credit();
}
```

If `credit()` fails:

```
rollback transaction
```

---

### Interview Trap 1

**Self Invocation Problem**

Example:

```java
public void methodA(){
    methodB();
}

@Transactional
public void methodB(){}
```

Problem:

```
No transaction
```

Why?

Because **methodB is called internally**, proxy not used.

Correct flow must be:

```
External call → Proxy → Method
```

---

### Interview Trap 2

@Transactional works only on:

```
public methods
```

Why?

Spring proxies intercept **public methods**.

---

### Production Rule

Always place `@Transactional` in:

```
Service layer
```

Never in:

```
Controller
Repository
```

---

# 4. Component Scanning

Spring finds beans automatically using **component scanning**.

Example:

```java
@ComponentScan("com.project")
```

Spring scans:

```
@Component
@Service
@Repository
@Controller
```

---

### What actually happens?

Spring:

1. scans packages
2. reads class metadata
3. registers bean definitions
4. creates beans later

---

### Production Rule

Always structure packages like:

```
com.project

   controller
   service
   repository
   config
   model
```

So scanning works properly.

---

# 5. Circular Dependency (Common Production Bug)

Example:

```
ServiceA -> depends on ServiceB
ServiceB -> depends on ServiceA
```

Code:

```java
@Service
class A {
   @Autowired
   B b;
}

@Service
class B {
   @Autowired
   A a;
}
```

Spring fails:

```
BeanCurrentlyInCreationException
```

---

### Why?

Spring cannot determine **which bean to create first**.

---

### Solution

1️⃣ Redesign architecture
2️⃣ Use `@Lazy` (temporary fix)

```java
@Autowired
@Lazy
B b;
```

But real fix:

```
Break the dependency
```

---

# 6. Spring Context Startup Flow

When application starts:

```
SpringApplication.run()
```

Internally:

### Step 1

Create container:

```
ApplicationContext
```

---

### Step 2

Load configuration

* XML
* Java Config
* annotations

---

### Step 3

Component scan

Find beans.

---

### Step 4

Create singleton beans

Singletons are created **at startup**.

---

### Step 5

Dependency injection

Spring wires beans.

---

### Step 6

Apply BeanPostProcessors

Used by:

* AOP
* Transactions
* Security

---

### Step 7

Application ready

---

### Production Insight

Startup time depends on:

* number of beans
* scanning
* reflection
* proxies

Large enterprise apps can have:

```
2000+ beans
```

---

# 7. Why Spring Boot Exists

Original Spring had too much configuration.

Example (old Spring):

```
XML config
Tomcat setup
DispatcherServlet config
Datasource config
Component scan
View resolver
Transaction manager
```

Very complex.

---

Spring Boot solved this using:

### 1. Auto Configuration

Spring automatically configures:

* database
* web server
* jackson
* logging

---

### 2. Embedded Servers

No need to install Tomcat.

Spring Boot includes:

* Tomcat
* Jetty
* Undertow

Run app like:

```
java -jar app.jar
```

---

### 3. Starter Dependencies

Instead of 10 dependencies:

```
spring-boot-starter-web
```

Includes:

* Spring MVC
* Jackson
* Validation
* Logging
* Tomcat

---

### 4. Production Features

Spring Boot provides:

* health checks
* metrics
* monitoring

via:

```
Spring Boot Actuator
```

---

# Senior Engineer Mental Model

When building a Spring application think:

```
1. Beans
2. Dependencies
3. Proxy layers
4. Transactions
5. Container lifecycle
```

Everything in Spring is basically:

```
Bean + Proxy + Container
```

---

# Most Important Spring Concepts (Priority)

If a junior engineer masters these:

```
1. IoC container
2. Dependency Injection
3. Bean lifecycle
4. AOP proxy
5. Transaction management
6. Component scanning
7. Spring context startup
```

They understand **80% of Spring**.

---

# Real Production Advice (15+ yrs experience)

1️⃣ Always design services **stateless**

2️⃣ Avoid **circular dependencies**

3️⃣ Use **constructor injection**

4️⃣ Keep **transactions at service layer**

5️⃣ Avoid **heavy logic in bean initialization**

6️⃣ Never inject **EntityManager in controller**

7️⃣ Always design with **interfaces**

---

If you want, next I can explain **Spring Boot like a senior architect**, including:

```
Spring Boot Auto Configuration (MOST misunderstood)
@SpringBootApplication breakdown
Spring Boot startup internals
DispatcherServlet flow
Spring MVC architecture
```

These are **EXTREMELY important for interviews and real production debugging.**
