# 2. Service Registry & Service Discovery (Using Eureka Server)

***

## Layer 0 — The Core Problem (Why This Exists)

Before we define anything, understand the **problem** microservices create.

### ❓ What Problem Appears in Microservices?

In microservices:

*   Each service runs **independently**
*   Services can **start, stop, scale up, or move**
*   Their **IP address and port change dynamically**

So a **hard‑coded URL** like:

    http://localhost:8081

❌ **breaks immediately** when:

*   The service restarts
*   A new instance is created
*   Autoscaling happens

This problem is formally described as the **Service Discovery problem** in distributed systems. [\[microservices.io\]](https://microservices.io/patterns/service-registry.html)

***

## Layer 1 — What Is a Service Registry? (Foundation)

### ✅ Service Registry (Definition)

A **Service Registry** is:

> A **central directory (database)** that stores  
> **which services are running**,  
> **where they are running (host + port)**,  
> and **whether they are healthy**  
> . [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/system-design/what-is-service-registry-in-microservices/), [\[microservices.io\]](https://microservices.io/patterns/service-registry.html)

### ✅ Simple Analogy

Think of it like:

*   **Phone Contacts app**
*   You search by **name**, not phone number
*   Phone numbers can change, names don’t

***

## Layer 2 — What Is Service Discovery?

### ✅ Service Discovery (Definition)

Service discovery is:

> The **process of finding a service’s network location dynamically at runtime**,  
> instead of hard‑coding it. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/system-design/what-is-service-registry-in-microservices/)

### ✅ In Simple Steps

1.  A service **registers itself** when it starts
2.  Another service **asks the registry** for its location
3.  The registry responds with **available healthy instances**

***

## Layer 3 — Why Service Discovery Is Essential

Service discovery is **not optional** in real microservices.

### ✅ Key Benefits

#### 1️⃣ Dynamic Scaling

*   New service instances automatically appear
*   Old instances automatically disappear [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/system-design/what-is-service-registry-in-microservices/)

#### 2️⃣ Fault Tolerance

*   Dead instances are removed
*   Calls go only to healthy services    [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

#### 3️⃣ Loose Coupling

*   No service knows another’s IP
*   Communication happens via **logical service names**    [\[microservices.io\]](https://microservices.io/patterns/service-registry.html)

✅ **This is foundational for production systems**

***

## Layer 4 — Client‑Side Service Discovery (Important Concept)

### ✅ Client‑Side Discovery (Definition)

In **client‑side discovery**:

> The **client itself** queries the service registry  
> and decides **which instance to call**. [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-netflix-eureka), [\[microservices.io\]](https://microservices.io/patterns/service-registry.html)

### ✅ Eureka Uses Client‑Side Discovery

Netflix Eureka follows this model:

*   Client fetches the registry
*   Client load‑balances requests

This model is **explicitly stated** in Spring Cloud Netflix docs and Baeldung. [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-netflix-eureka), [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

***

## Layer 5 — What Is Eureka?

### ✅ Eureka (Definition)

**Netflix Eureka** is:

> A **Service Registry and Service Discovery server**  
> originally built by Netflix for large‑scale distributed systems  
> . [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

### ✅ Who Uses It?

*   Netflix (originally)
*   Spring Cloud integrates it as **Spring Cloud Netflix Eureka** [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

***

## Layer 6 — Eureka Architecture (Very Important)

Eureka follows a **Client–Server model**.

### ✅ Main Components

#### 1️⃣ Eureka Server

> A Spring Boot application that:

*   Stores service registry
*   Exposes a dashboard
*   Accepts registrations [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/), [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-netflix-eureka)

#### 2️⃣ Eureka Client

> Any microservice that:

*   Registers itself
*   Discovers other services
*   Sends heartbeats [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-netflix-eureka)

***

## Layer 7 — How Eureka Works (Step by Step)

### ✅ Step 1: Service Registration

When a service starts:

*   It sends:
    *   Service name
    *   IP address
    *   Port
    *   Metadata
        to Eureka Server. [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

***

### ✅ Step 2: Heartbeats

### ✅ Heartbeat (Definition)

A heartbeat is:

> A periodic signal sent to the registry saying  
> “I am still alive”

*   Default interval: \~30 seconds
*   Missing heartbeats → service marked **DOWN**  
    . [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/), [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/system-design/what-is-service-registry-in-microservices/)

***

### ✅ Step 3: Registry Fetch

Clients:

*   Periodically fetch full registry
*   Cache it locally
*   Use it for service calls  
    . [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-netflix-eureka)

✅ This avoids dependency on registry for every request.

***

## Layer 8 — Eureka Self‑Preservation Mode (Senior Concept)

### ✅ Self‑Preservation Mode (Definition)

Eureka **does NOT immediately remove instances**  
when many heartbeats are lost suddenly.

Why?

*   To avoid massive removals during:
    *   Network partitions
    *   Temporary outages

This feature is documented in Netflix Eureka behavior. [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

✅ Seniors must **understand this trade‑off**

***

## Layer 9 — Eureka Server Setup (Spring Official Way)

According to **Spring Docs**:

### ✅ Eureka Server Is Created By:

1.  Spring Boot application
2.  Adding `spring-cloud-starter-netflix-eureka-server`
3.  Annotating with:

<!---->

    @EnableEurekaServer

4.  Disabling self‑registration (because it’s the registry itself)  
    . [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/), [\[baeldung.com\]](https://www.baeldung.com/spring-cloud-netflix-eureka)

***

## Layer 10 — Eureka Client Setup (Conceptual)

A Eureka client:

*   Has `spring.application.name`
*   Knows Eureka Server URL
*   Automatically registers

Spring Cloud provides **auto‑registration** when the dependency is present. [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

***

## Layer 11 — Service Name Matters (Very Important)

### ✅ Service Name

*   Logical identifier of a service
*   Comes from `spring.application.name`
*   Used for discovery, NOT IP address. [\[docs.spring.io\]](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/)

Example:

    ORDER-SERVICE

Clients call:

    http://ORDER-SERVICE/api/orders

✅ **IP resolution happens internally**

***

## Layer 12 — Senior‑Level Understanding

### ✅ Eureka Is Best When:

*   You manage your own infrastructure
*   Not using Kubernetes built‑in discovery
*   Need explicit client‑side control

### ❌ Eureka Is Less Needed When:

*   Running fully on Kubernetes (K8s has its own registry)

Spring itself acknowledges Kubernetes can replace Eureka in many setups  
. [\[learn.microsoft.com\]](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/microservices-addressability-service-registry)

***

## ✅ Final Mental Model (One Sentence)

> **Eureka is a centralized directory that allows microservices to dynamically register themselves and discover each other at runtime without hard‑coded network locations.**

***

### ✅ Authoritative References Used

*   Spring Cloud Netflix Docs
*   Baeldung Eureka Guide
*   microservices.io (Chris Richardson)
*   Netflix OSS Design
