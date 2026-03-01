# 11️⃣ Introduction to Spring Boot and Setup

---

# 1️⃣ What is Spring Boot?

Spring Boot is:

> An opinionated framework built on top of Spring that simplifies configuration and application setup.

Important:

* Spring Boot does NOT replace Spring.
* It builds on Spring Core.
* It automates setup and configuration.

Think of it like this:

```text
Spring = Engine
Spring Boot = Automatic Engine Starter + Default Setup
```

---

# 2️⃣ The Problem Spring Boot Solves

Before Boot, you had to manually:

* Create ApplicationContext
* Configure component scanning
* Define DispatcherServlet
* Configure embedded server
* Setup transaction manager
* Configure data source
* Setup property sources

It was powerful — but verbose.

Boot reduces this to:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

That’s it.

Massive reduction in configuration code.

---

# 3️⃣ How Spring Boot Simplifies Spring

---

## ✅ 1. Auto-Configuration

Spring Boot automatically configures beans based on:

* Classpath
* Existing beans
* Configuration properties

Example:

If Spring Boot sees:

```text
spring-boot-starter-data-jpa
```

It auto-configures:

* DataSource
* EntityManager
* TransactionManager

No manual wiring needed.

But:

You can override everything.

Boot gives defaults, not restrictions.

---

## ✅ 2. Starter Dependencies

Instead of adding many individual dependencies:

Without Boot:

* spring-core
* spring-context
* spring-web
* jackson
* tomcat
* validation
* logging

With Boot:

```xml
spring-boot-starter-web
```

It pulls required dependencies transitively.

Common starters:

* spring-boot-starter-web
* spring-boot-starter-data-jpa
* spring-boot-starter-security
* spring-boot-starter-test

This simplifies dependency management.

---

## ✅ 3. Embedded Servers

No need to deploy WAR files to external Tomcat.

Boot includes embedded:

* Tomcat (default)
* Jetty
* Undertow

Your app becomes:

```text
Self-contained executable JAR
```

Run with:

```bash
java -jar app.jar
```

This is critical for microservices and cloud deployments.

---

## ✅ 4. Convention Over Configuration

Boot assumes sensible defaults.

Example:

* Port 8080
* JSON via Jackson
* Logging via Logback
* Component scan from main package

You configure only when needed.

---

# 4️⃣ Spring Boot Architecture Overview

Let’s visualize.

---

## Boot Application Flow

```text
@SpringBootApplication
        ↓
SpringApplication.run()
        ↓
Create ApplicationContext
        ↓
Apply Auto-Configurations
        ↓
Start Embedded Server
        ↓
Application Ready
```

---

## What @SpringBootApplication Does

It is a meta-annotation combining:

```text
@Configuration
@EnableAutoConfiguration
@ComponentScan
```

So this:

```java
@SpringBootApplication
```

Is equivalent to manually writing:

```java
@Configuration
@EnableAutoConfiguration
@ComponentScan
```

Boot just bundles it.

---

# 5️⃣ Internal Concept — Auto-Configuration

Boot checks:

```text
Is certain class on classpath?
Is certain bean missing?
Are certain properties defined?
```

If yes → configure default bean.

Example:

If `spring-web` exists:

* Configure DispatcherServlet
* Configure Jackson
* Configure error handling

If `spring-data-jpa` exists:

* Configure JPA infrastructure

All conditional.

You can override by defining your own bean.

---

# 6️⃣ What Boot Does NOT Do

Important to understand.

Boot does NOT:

* Change Spring DI principles
* Remove IoC container
* Replace ApplicationContext
* Remove need for good architecture

All core Spring knowledge still applies.

Boot simplifies configuration — not design.

---

# 7️⃣ Setup Overview

Typical Boot project structure:

```text
com.example
 ├── Application.java
 ├── controller
 ├── service
 ├── repository
 └── config
```

Main class:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Component scanning starts from this package downward.

Package structure matters.

---

# 8️⃣ Comparison: Spring vs Spring Boot

| Aspect                | Spring   | Spring Boot     |
| --------------------- | -------- | --------------- |
| Configuration         | Manual   | Auto-configured |
| Dependency management | Manual   | Starters        |
| Server setup          | External | Embedded        |
| Boilerplate           | High     | Minimal         |
| Deployment            | WAR      | Executable JAR  |

---

# 9️⃣ Mental Model Shift

Before Boot:

> You assemble everything manually.

With Boot:

> Boot assembles common infrastructure for you.

But:

You must still understand Spring Core.

Otherwise:

* You won’t debug issues
* You won’t override correctly
* You’ll misuse auto-configuration

---

# 🔟 Common Beginner Mistakes in Boot

❌ Not understanding component scanning base package
❌ Blindly trusting auto-configuration
❌ Overriding beans incorrectly
❌ Putting main class in wrong package
❌ Thinking Boot removes need for DI understanding

---

# 📘 Revision Notes — Topic 11

---

## 📌 Spring Boot Introduction

### What is Spring Boot?

An opinionated framework built on top of Spring that simplifies configuration and setup.

---

### Key Advantages

1. Auto-Configuration
   Automatically configures beans based on classpath and properties.

2. Starter Dependencies
   Simplifies dependency management.

3. Embedded Servers
   Applications run as executable JARs.

4. Convention Over Configuration
   Sensible defaults reduce boilerplate.

---

### @SpringBootApplication

Combines:

* @Configuration
* @EnableAutoConfiguration
* @ComponentScan

---

### Architecture Flow

@SpringBootApplication
→ SpringApplication.run()
→ Create ApplicationContext
→ Apply Auto-Configuration
→ Start Embedded Server

---

### Important Principle

Spring Boot simplifies configuration, but core Spring concepts remain unchanged.
