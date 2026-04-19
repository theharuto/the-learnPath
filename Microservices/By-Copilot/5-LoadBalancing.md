# 5️⃣ Spring Cloud Load Balancing

*(Foundational → senior‑level mental model)*

***

## LAYER 0 — WHY LOAD BALANCING EXISTS (ROOT PROBLEM)

### ✅ The core reality of microservices

In a real microservices system:

*   **One service is never enough**
*   We run **multiple instances of the same service**

Example:

    USER‑SERVICE instances:
    - 10.0.1.1:8080
    - 10.0.1.2:8080
    - 10.0.1.3:8080

Why?

*   High traffic
*   Fault tolerance
*   Zero‑downtime deployments

This situation is **explicitly described** in Spring Cloud and Baeldung docs as the motivation for load balancing. [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-load-balancer), [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

## LAYER 1 — WHAT IS LOAD BALANCING? (EVERY WORD DEFINED)

### ✅ Load

“Load” means:

> Incoming requests (traffic)

Examples:

*   HTTP requests
*   REST calls
*   Service‑to‑service calls

***

### ✅ Balancing

“Balancing” means:

> Distributing something evenly

***

### ✅ Load Balancing — official meaning

> **The process of distributing network traffic across multiple instances of the same service**    [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-load-balancer)

This definition is used consistently by:

*   Spring Docs
*   Baeldung
*   Cloud computing literature

***

## LAYER 2 — TWO TYPES OF LOAD BALANCING (VERY IMPORTANT)

### 1️⃣ Server‑Side Load Balancing

*   A **central load balancer** sits in front
*   Example: NGINX, AWS ELB

Architecture:

    Client → Load Balancer → Service Instances

***

### 2️⃣ Client‑Side Load Balancing ✅ (SPRING CLOUD MODEL)

> **The client itself decides which service instance to call** [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-load-balancer), [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

Architecture:

    Client → Instance (chosen by client)

📌 **Spring Cloud LoadBalancer is client‑side**

***

## LAYER 3 — WHAT IS SPRING CLOUD LOADBALANCER?

### ✅ Official Spring definition

> **Spring Cloud provides its own client‑side load‑balancer abstraction and implementation** [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

This means:

*   Spring provides a **library**
*   It runs **inside your application**
*   It decides **which instance to call**

***

## LAYER 4 — WHY SPRING CLOUD LOADBALANCER WAS CREATED

### ✅ Historical context

*   Netflix Ribbon (older solution) is **deprecated**
*   Spring introduced **Spring Cloud LoadBalancer** as its replacement

Baeldung explicitly states this transition. [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-load-balancer)

***

## LAYER 5 — CORE COMPONENTS (VERY IMPORTANT)

Spring Cloud LoadBalancer consists of **three critical building blocks**.

***

### ✅ 1️⃣ Service Instance

A **ServiceInstance** represents:

*   One running instance
*   Has:
    *   Service ID
    *   Host
    *   Port
    *   Metadata

Returned from:

*   Eureka
*   Consul
*   Other discovery mechanisms [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

### ✅ 2️⃣ ServiceInstanceListSupplier

Definition:

> A component that **supplies the list of available service instances**    [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

Important facts:

*   Fetches instances from **service discovery**
*   Reactive
*   Continuously updated

You can think of it as:

> “The list of possible targets”

***

### ✅ 3️⃣ ReactorLoadBalancer

Definition:

> A component that **chooses one instance from the list** [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

This is where:

*   Algorithms live
*   Decisions are made

***

## LAYER 6 — LOAD BALANCING ALGORITHMS

Spring Cloud LoadBalancer supports multiple strategies.

### ✅ Default: Round‑Robin

> Each request goes to the next instance in sequence [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

Example:

    Request 1 → instance A
    Request 2 → instance B
    Request 3 → instance C
    Request 4 → instance A

✅ Simple  
✅ Fair  
✅ Stateless

***

### ✅ Random

> Picks a random instance for each request [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

✅ Less predictable  
❌ Not evenly distributed under some conditions

***

### ✅ Custom strategies

Spring allows:

*   Custom algorithms
*   Weighted decisions
*   Health‑aware logic

Documented in Spring Docs using `@LoadBalancerClient`. [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

## LAYER 7 — HOW LOAD BALANCING ACTUALLY WORKS (RUNTIME FLOW)

Let’s trace a **real call**.

### Example:

```http
GET http://USER‑SERVICE/users/1
```

### Step‑by‑step

1.  Client wants to call `USER‑SERVICE`
2.  LoadBalancer queries **ServiceInstanceListSupplier**
3.  Instances are fetched from **Eureka**
4.  LoadBalancer applies algorithm (round‑robin)
5.  One instance is selected
6.  HTTP call is made to selected instance

✅ Eureka tells *what exists*  
✅ LoadBalancer decides *which one to use*

 [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html), [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-load-balancer)

***

## LAYER 8 — `lb://<SERVICE‑NAME>` (EXAMPLE EXPLAINED LETTER BY LETTER)

### ✅ Example used in API Gateway

```yaml
uri: lb://USER‑SERVICE
```

***

### ✅ `lb`

*   Stands for **Load Balancer**
*   Tells Spring:
    > “Resolve this via Spring Cloud LoadBalancer”

***

### ✅ `://`

*   URI protocol separator

***

### ✅ `USER‑SERVICE`

*   Logical service ID
*   Must match:

```properties
spring.application.name=USER‑SERVICE
```

✅ No IP  
✅ No port  
✅ Fully dynamic

Documented in Spring Cloud Gateway + LoadBalancer integration. [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

## LAYER 9 — INTEGRATION POINTS (VERY IMPORTANT)

Spring Cloud LoadBalancer integrates with:

### ✅ Spring Cloud Gateway

*   Gateway uses `lb://`
*   LoadBalancer resolves instance [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

### ✅ OpenFeign

*   Feign automatically uses LoadBalancer
*   No extra config required

 [\[javaguides.net\]](https://www.javaguides.net/2024/05/spring-cloud-loadbalancer-example.html)

***

### ✅ WebClient

*   Uses `@LoadBalanced WebClient.Builder`
*   LoadBalancer intercepts requests [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

## LAYER 10 — EAGER vs LAZY LOADING (SENIOR‑LEVEL)

### ✅ Default: Lazy

*   LoadBalancer context created **on first request**

### ✅ Optional: Eager Loading

*   Context created on **startup**

Config:

```properties
spring.cloud.loadbalancer.eager-load.clients[0]=USER-SERVICE
```

Why this matters:

*   Avoids first‑request latency

 [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)

***

## LAYER 11 — WHAT LOAD BALANCER DOES **NOT** DO

❌ It does NOT:

*   Handle authentication
*   Do rate limiting
*   Serve as gateway
*   Store state

✅ It ONLY:

*   Chooses service instance

***

## FINAL MENTAL MODEL (READ TWICE)

> **Spring Cloud LoadBalancer is a client‑side mechanism that, using service discovery, selects one healthy instance of a target service (by default using round‑robin) and routes the outgoing request to it, enabling fault‑tolerant and scalable microservice communication without hard‑coded addresses.**

***

## ✅ Authoritative Sources Used

*   Spring Cloud LoadBalancer Documentation [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)
*   Baeldung: Spring Cloud LoadBalancer Guide [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-load-balancer)

***

### 🔥 NEXT ABSOLUTELY IMPORTANT TOPICS (CHOOSE ONE)

1️⃣ **Feign + LoadBalancer internal call flow**  
2️⃣ **Gateway vs LoadBalancer vs Eureka (roles compared)**  
3️⃣ **Custom load‑balancing strategy (step‑by‑step)**  
4️⃣ **Why Kubernetes can replace LoadBalancer + Eureka**
