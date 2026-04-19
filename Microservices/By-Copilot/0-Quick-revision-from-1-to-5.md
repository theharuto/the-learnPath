# 📘 Microservices – Complete Revision Notes

*(Interview + Production Ready)*

***

## 1️⃣ What Are Microservices?

### ✅ Definition

**Microservices Architecture** is an architectural style where:

*   An application is built as **a collection of small, independent services**
*   Each service:
    *   Runs in its **own process**
    *   Is **independently deployable**
    *   Focuses on **one business capability**

### ✅ Core Characteristics

*   Loosely coupled
*   Independently deployable
*   Decentralized data
*   Communication via HTTP or messaging
*   Designed for failure

### ✅ Benefits

*   Scalability (scale one service, not whole system)
*   Fault isolation
*   Faster deployments
*   Team autonomy

### ❌ Challenges

*   Distributed system complexity
*   Network failures
*   Observability
*   Data consistency
*   Operational overhead

***

## 2️⃣ Monolith vs Microservices

| Aspect         | Monolith      | Microservices      |
| -------------- | ------------- | ------------------ |
| Deployment     | Single unit   | Per service        |
| Scaling        | Whole app     | Per service        |
| Failure        | App-wide      | Isolated           |
| Complexity     | Low initially | High operationally |
| Team structure | Centralized   | Decentralized      |

📌 **Senior insight**:

> *Start with a well‑structured monolith, extract microservices when scale demands it.*

***

## 3️⃣ Service Registry & Discovery (Eureka)

### ✅ The Problem

Service instances:

*   Change IP/port dynamically
*   Scale up/down
*   Restart frequently

➡️ Hard‑coding URLs **does not work**

***

### ✅ Service Registry

A **central directory** that stores:

*   Service name
*   Instance IPs and ports
*   Health information

### ✅ Service Discovery

The process where:

*   A service finds another service **dynamically at runtime**

***

## ✅ Netflix Eureka

### Components

*   **Eureka Server** → registry
*   **Eureka Client** → registers + discovers services

### How It Works

1.  Service starts → registers with Eureka
2.  Sends heartbeats every \~30s
3.  Clients fetch **full registry**
4.  Registry cached **locally in memory**
5.  Calls made using cached data

📌 **Important**

*   Eureka is **client‑side discovery**
*   No routing
*   No retry logic

***

### ✅ Self‑Preservation Mode

Eureka **stops evicting instances** when many heartbeats are lost suddenly  
(to avoid mass deletion during network partitions)

✅ Prevents cascade failures  
❌ Allows stale entries temporarily

***

## 4️⃣ Feign Client – Inter‑Service Communication

### ✅ What Is Feign?

**Feign** is a **declarative REST client**:

*   HTTP calls are written as **Java interface methods**
*   Spring generates implementation at runtime

***

### How Feign Works (Runtime)

1.  Method call on Feign interface
2.  Proxy intercepts the call
3.  Builds HTTP request
4.  Uses service name → Eureka
5.  LoadBalancer selects instance
6.  HTTP call executed
7.  JSON → Java object (deserialization)

✅ No `RestTemplate`  
✅ No manual JSON parsing

***

### ✅ Key Annotations

*   `@EnableFeignClients`
*   `@FeignClient(name = "SERVICE-NAME")`
*   Spring MVC annotations (`@GetMapping`, `@RequestBody`, etc.)

***

### ✅ Feign Characteristics

*   Blocking (thread waits)
*   Synchronous
*   Integrates with:
    *   Eureka
    *   Spring Cloud LoadBalancer
*   Retry **is NOT enabled by default**

***

## 5️⃣ API Gateway – Spring Cloud Gateway

### ✅ What Is an API Gateway?

A **single entry point** that:

*   Receives **all client requests**
*   Routes them to backend services
*   Applies cross‑cutting concerns

***

### ✅ Why API Gateway Exists

Without Gateway:

*   Clients must know many service URLs
*   Security duplicated
*   Hard to evolve system

***

### ✅ Spring Cloud Gateway

#### Built On

*   Spring WebFlux
*   Project Reactor
*   Non‑blocking, reactive model

✅ Essential because gateway handles **all traffic**

***

### ✅ Core Concepts

#### 🔹 Route

Defines:

*   Where request goes

#### 🔹 Predicate

Defines:

*   When route applies  
    (Path, Method, Header, etc.)

#### 🔹 Filter

Defines:

*   What happens to request/response

***

### ✅ Built‑in Filters

*   Add/Remove headers
*   Logging
*   Rate limiting
*   Authentication
*   Path rewriting
*   Retry (explicit)

***

### ✅ Request Lifecycle

    Client
     → Gateway
       → Predicates
       → Pre Filters
         → LoadBalancer
           → Service Instance
       → Post Filters
     → Client

📌 Gateway **does not contain business logic**

***

## 6️⃣ Spring Cloud LoadBalancer

### ✅ What Is Load Balancing?

Distributing traffic across **multiple instances** of the **same service**

***

### ✅ Client‑Side Load Balancing

*   **Client chooses instance**
*   No central load balancer

Spring Cloud LoadBalancer is **client‑side**

***

### ✅ Core Components

#### 1️⃣ ServiceInstance

Represents one service instance (IP + port)

#### 2️⃣ ServiceInstanceListSupplier

Supplies available instances (from Eureka)

#### 3️⃣ ReactorLoadBalancer

Chooses **one instance** using an algorithm

***

### ✅ Default Algorithm

*   **Round‑Robin**

Other options:

*   Random
*   Custom strategies

***

### ✅ `lb://SERVICE-NAME`

Used in:

*   Gateway
*   Feign
*   WebClient

Meaning:

> Resolve service name using LoadBalancer + Eureka

***

## 7️⃣ Retry – WHO does WHAT? (VERY IMPORTANT)

### ❌ What Does NOT Retry

*   Eureka ❌
*   ReactorLoadBalancer ❌
*   LoadBalancer alone ❌

➡️ LoadBalancer **only selects instance**

***

### ✅ Where Retry Happens

| Layer                | Retry?       | Notes               |
| -------------------- | ------------ | ------------------- |
| Feign Retryer        | ✅ (optional) | Disabled by default |
| Gateway Retry Filter | ✅ (optional) | Explicit config     |
| Spring Retry         | ✅            | For client calls    |

📌 Each retry:

*   Calls LoadBalancer again
*   Gets a different instance

***

### ✅ Internal Retry Flow

    Attempt 1 → Instance A → fails
    Retry
    Attempt 2 → Instance B → success

***

## 8️⃣ Key Production Insights (INTERVIEW GOLD)

✅ LoadBalancer ≠ Retry  
✅ Gateway ≠ LoadBalancer  
✅ Feign ≠ Gateway  
✅ Eureka ≠ Health checker

***

### Common Interview Traps

❌ “LoadBalancer retries automatically” → **WRONG**  
✅ Retry must be configured explicitly

❌ “Eureka always knows real instance health” → **WRONG**  
✅ Heartbeat‑based, eventually consistent

***

## 9️⃣ Mental Model Summary (One‑Liners)

*   **Eureka** → “Who exists?”
*   **LoadBalancer** → “Who should I try?”
*   **Feign** → “Make HTTP feel like method call”
*   **Gateway** → “Edge traffic controller”
*   **Retry** → “Try again when failure occurs”

***

## 🔟 What We Would Learn Next (Natural Continuation)

1.  Circuit Breaker (Resilience4j)
2.  Retry vs Circuit Breaker (CRITICAL)
3.  Distributed Tracing (Zipkin / Micrometer)
4.  Kubernetes vs Spring Cloud (real‑world shift)

***

### ✅ Final Advice

> If you understand **roles and boundaries** of Eureka, LoadBalancer, Feign, Gateway, and Retry —  
> you are already **thinking like a senior backend engineer**.

If you want, I can:

*   Convert this into **interview Q\&A**
*   Add **architecture diagrams**
*   Or move to **Resilience4j next**
