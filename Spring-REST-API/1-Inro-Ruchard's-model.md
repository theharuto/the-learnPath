# 🔹 Topic 1: What is REST?

## Short Explanation

REST is an architectural style where:

* System exposes **resources**
* Client interacts using **HTTP**
* Each request is **independent**
* Server returns a **representation (usually JSON)**

---

## Behavioral Model (Think Like This)

> “A client sends structured input → system maps it to a resource → system returns current state representation”

Example flow:

```text
Request → Identify resource → Process → Return representation
```

---

## Core REST Principles

---

### 1. Statelessness

## Explanation

* Server does NOT store client context
* Each request must contain all required data

---

## Behavioral Model

```text
Request 1 → processed → forgotten
Request 2 → treated as completely new
```

---

## Why it exists

* Enables **horizontal scaling**
* No session synchronization
* Any server can handle any request

---

## Failure Scenario

If stateful:

```text
Request 1 → Server A (state stored)
Request 2 → Server B → ❌ state missing
```

---

## Common Mistake

❌ Storing request-specific data in memory/session

---

---

### 2. Client-Server Separation

## Explanation

* Client handles UI
* Server handles data + logic
* Both evolve independently

---

## Behavioral Model

```text
Client → sends request
Server → processes + returns data
Client → decides how to display
```

---

## Why it matters

* Decoupling
* Multiple clients (web, mobile) reuse same API

---

---

### 3. Uniform Interface

## Explanation

* Standard way to interact with resources
* Uses HTTP methods + predictable URLs

---

## Behavioral Model

```text
GET    → retrieve state
POST   → create new state
PUT    → replace state
PATCH  → modify part of state
DELETE → remove state
```

---

## Key Rule

> “Same type of request → same kind of behavior”

---

## Common Mistake

❌ Using:

```text
POST /getUser
```

✔ Correct:

```text
GET /users/{id}
```

---

# 🔹 Topic 2: What is a RESTful API?

## Short Explanation

A RESTful API is:

> An API that follows REST principles strictly

---

## Behavioral Model

A RESTful API:

```text
Accepts → structured HTTP request
Processes → based on resource + method
Returns → standardized response (JSON + status code)
```

---

## Characteristics

* Stateless
* Resource-based
* Uses HTTP properly
* Returns meaningful status codes

---

## Non-RESTful API Example

```text
POST /doEverything
```

* No resource concept
* No method semantics

---

## RESTful API Example

```text
GET    /users
POST   /users
PUT    /users/{id}
DELETE /users/{id}
```

---

# 🔹 Topic 3: Spring Support for REST

## Short Explanation

Spring Boot provides:

* Embedded server (Tomcat)
* Request routing
* JSON conversion
* Annotation-based API design

---

## Behavioral Model

```text
HTTP Request
   ↓
DispatcherServlet
   ↓
Controller method
   ↓
Object returned
   ↓
Converted to JSON
   ↓
HTTP Response
```

---

## Key Components

* `DispatcherServlet` → entry point
* `@RestController` → defines endpoints
* `HttpMessageConverter` → JSON conversion

---

---

# 🔹 Topic 4: Setting Up Spring REST

(Based on official Spring guide)

---

## 1. Dependency

### Maven

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

---

## What this gives you

* Spring MVC
* Embedded Tomcat
* Jackson (JSON)
* Validation support

---

## Internal Effect

Adding this dependency:

* Auto-configures web layer
* Registers DispatcherServlet
* Starts server

---

---

## 2. Configuration

### application.properties

```properties
server.port=8081
spring.application.name=demo
```

---

## Behavioral Model

```text
App starts → reads config → configures server
```

---

---

## 3. Embedded Server

## Explanation

Spring Boot runs:

* Embedded **Apache Tomcat**

---

## Flow

```text
Run main() → Spring Boot starts → Tomcat starts → app listens on port
```

---

## Why important

* No external deployment needed
* Faster development
* Production-ready setup

---

---

# 🔹 Topic 5: Richardson Maturity Model

(From Martin Fowler)

---

## Short Explanation

It defines how “RESTful” your API is.

---

## Levels Overview

---

## Level 0 – Single Endpoint (Not REST)

```text
POST /api
body: { action: "getUser" }
```

❌ Everything goes through one endpoint
❌ No HTTP semantics

---

## Level 1 – Resources Introduced

```text
/users
/orders
```

✔ Resources exist
❌ Still using wrong HTTP methods

---

---

## Level 2 – HTTP Methods Used Properly (MOST IMPORTANT)

```text
GET    /users
POST   /users
PUT    /users/{id}
DELETE /users/{id}
```

✔ Proper REST usage
✔ Industry standard

---

---

## Level 3 – HATEOAS (Rare in real world)

Response includes links:

```json
{
  "id": 1,
  "links": [
    { "rel": "self", "href": "/users/1" },
    { "rel": "delete", "href": "/users/1" }
  ]
}
```

---

## Reality Check

* Most systems stop at **Level 2**
* Level 3 is rarely implemented

---

---

# 🔹 Production Insights

---

## 1. REST is about constraints, not just endpoints

Bad APIs:

* Ignore HTTP methods
* Mix responsibilities
* Become unscalable

---

## 2. Statelessness enables scale

Without it:

* Load balancing fails
* Sessions break

---

## 3. Design matters more than framework

Spring won’t fix:

* Bad API design
* Poor resource modeling

---

---

# 🔹 Hands-on Tasks

## Task 1

Design endpoints for:

* Fetch all items
* Fetch one item
* Create item
* Delete item

👉 Only URLs + HTTP methods

---

## Task 2

Identify level:

```text
POST /api
body: { "operation": "deleteUser", "id": 10 }
```

---

## Task 3

Why is Level 2 considered “practically sufficient” in most systems?

---

---

# 🔹 Interview Questions

* What is REST?
* What is statelessness?
* Difference between REST and RESTful?
* What is Richardson Maturity Model?
* Why is Level 3 rarely used?

---

---

# 🔹 GitHub Notes (Markdown)

Here are your clean notes:

---

# 📘 REST & Spring REST - Foundations

---

## 🔹 REST

* Architectural style for APIs
* Resource-based design
* Uses HTTP methods
* Stateless communication

---

## 🔹 REST Principles

### Statelessness

* No server-side session storage
* Each request is independent
* Enables scalability

---

### Client-Server Separation

* Client → UI
* Server → logic + data
* Decoupled systems

---

### Uniform Interface

* Standard HTTP methods:

  * GET, POST, PUT, PATCH, DELETE
* Predictable URL structure

---

## 🔹 RESTful API

* API that follows REST principles
* Uses resources + HTTP semantics
* Returns JSON + proper status codes

---

## 🔹 Spring REST Support

* Embedded Tomcat server
* DispatcherServlet handles requests
* @RestController defines endpoints
* JSON handled via Jackson

---

## 🔹 Spring Setup

### Dependency

* spring-boot-starter-web

### Features

* Auto configuration
* Embedded server
* JSON conversion

---

### Configuration

application.properties:

* server.port
* application settings

---

## 🔹 Embedded Server

* Spring Boot runs Tomcat internally
* No external deployment needed

---

## 🔹 Richardson Maturity Model

### Level 0

* Single endpoint
* No REST principles

---

### Level 1

* Resources introduced

---

### Level 2 (Industry Standard)

* Proper HTTP methods
* Clean REST design

---

### Level 3

* HATEOAS (links in response)
* Rarely used

---

## 🔹 Key Takeaways

* REST = constraints, not just endpoints
* Statelessness enables scaling
* Level 2 REST is sufficient for most systems
* Spring simplifies implementation, not design

---

---

# 🔹 Check Your Understanding

Answer these:

1. Why is statelessness required for load balancing?
2. Why is Level 0 not considered REST?
3. What does Spring Boot auto-configure when you add `spring-boot-starter-web`?
