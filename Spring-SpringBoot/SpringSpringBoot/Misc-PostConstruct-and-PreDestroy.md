# 🎯 Real-World Example: Email Notification Service

Imagine you’re building a system that:

* Connects to an external email server at startup
* Sends emails during runtime
* Closes connection gracefully at shutdown

That’s a perfect use case for:

* `@PostConstruct`
* `@PreDestroy`

---

# 🏗 Step 1 — The Service

```java
package com.example.notification;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    private boolean connected = false;

    @PostConstruct
    public void init() {
        System.out.println("Connecting to email server...");
        connected = true;
    }

    public void sendEmail(String message) {
        if (!connected) {
            throw new IllegalStateException("Email server not connected!");
        }
        System.out.println("Sending email: " + message);
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Closing email server connection...");
        connected = false;
    }
}
```

---

# 🧠 What Happens Internally?

When Spring starts:

```text
1. Create EmailService instance
2. Inject dependencies (if any)
3. Call @PostConstruct method
4. Bean ready for use
```

When application shuts down:

```text
1. Call @PreDestroy method
2. Remove bean from container
```

---

# 🏃 Step 2 — Bootstrapping (Without Boot)

```java
package com.example;

import com.example.notification.EmailService;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.example")
class AppConfig {}

public class MainApp {

    public static void main(String[] args) {

        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(AppConfig.class);

        EmailService emailService = context.getBean(EmailService.class);
        emailService.sendEmail("Welcome!");

        context.close(); // triggers @PreDestroy
    }
}
```

---

# 🖥 Expected Console Output

```text
Connecting to email server...
Sending email: Welcome!
Closing email server connection...
```

This shows lifecycle clearly.

---

# 🔎 Lifecycle Timeline Explained Simply

---

## 1️⃣ Instantiation

Spring does:

```java
new EmailService();
```

---

## 2️⃣ Dependency Injection

If there were dependencies, they would be injected here.

---

## 3️⃣ @PostConstruct

Spring calls:

```java
init();
```

This is where:

* Expensive setup
* Resource allocation
* Validation
* Cache loading

should happen.

---

## 4️⃣ Runtime Usage

Your service is fully initialized.

---

## 5️⃣ @PreDestroy

When container shuts down:

Spring calls:

```java
cleanup();
```

Used for:

* Closing DB connections
* Releasing sockets
* Flushing logs
* Cleaning threads

---

# ⚠ Important Real-World Rules

---

## Rule 1: @PostConstruct is NOT Constructor

Constructor:

* Object creation
* No injected dependencies guaranteed

@PostConstruct:

* Runs after injection
* Bean fully wired

So if your logic depends on injected fields → use @PostConstruct.

---

## Rule 2: @PreDestroy Only Works for Managed Beans

Spring must manage the lifecycle.

If you manually create object:

```java
new EmailService();
```

@PostConstruct and @PreDestroy will NOT run.

---

## Rule 3: Prototype Scope Does NOT Call @PreDestroy

If:

```java
@Scope("prototype")
```

Spring does not track destruction.

You must clean it manually.

---

# 🧠 Where Lifecycle Fits in Bigger Picture

Simplified Spring lifecycle:

```text
Instantiate
    ↓
Inject dependencies
    ↓
@PostConstruct
    ↓
Bean ready
    ↓
@PreDestroy (on shutdown)
```

Everything you do should respect this flow.

---

# ❌ Common Beginner Mistakes

---

## ❌ Doing heavy work in constructor

Bad:

```java
public EmailService() {
    connectToServer();
}
```

Constructor should be lightweight.

Use @PostConstruct for heavy initialization.

---

## ❌ Forgetting to close context

If you never call:

```java
context.close();
```

@PreDestroy won’t run.

Spring Boot handles shutdown automatically.

---

## ❌ Putting business logic in lifecycle methods

Lifecycle methods are for:

* Infrastructure setup
* Resource management

Not business logic.

---

# 🏢 Real Production Use Cases

@PostConstruct:

* Warm up cache
* Validate configuration
* Load ML model
* Preload static data

@PreDestroy:

* Close DB pool
* Stop background threads
* Shutdown scheduler
* Flush metrics

---

# 📘 Clean Revision Notes

---

## 📌 Spring Bean Lifecycle (@PostConstruct & @PreDestroy)

### Lifecycle Order

1. Instantiate bean
2. Inject dependencies
3. Call @PostConstruct
4. Bean ready
5. Call @PreDestroy (on shutdown)

---

### @PostConstruct

* Runs after dependency injection
* Used for initialization logic
* Executes once for singleton beans

---

### @PreDestroy

* Runs before bean destruction
* Used for cleanup
* Not called for prototype beans

---

### Best Practices

* Keep constructors lightweight
* Use @PostConstruct for setup
* Use @PreDestroy for cleanup
* Do not put business logic in lifecycle hooks
* Ensure container shutdown to trigger cleanup

---

# 🔥 Senior-Level Understanding

Think of lifecycle methods as:

> Infrastructure hooks in object lifetime.

They exist so your application can behave predictably at startup and shutdown.

---
* Deep dive into full bean lifecycle including BeanPostProcessor
* Or convert this example into Spring Boot version
* Or explore circular dependencies and lifecycle interaction
