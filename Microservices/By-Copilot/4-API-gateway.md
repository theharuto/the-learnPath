# 4️⃣ API Gateway — Spring Cloud Gateway

*(Absolute foundational → senior‑level understanding)*

***

## LAYER 0 — WHY API GATEWAY EXISTS (THE ROOT PROBLEM)

### ✅ What problem appears in Microservices?

You already know this part slightly, but let’s restate it **carefully**.

In a microservices system:

*   You don’t have **one backend**
*   You have **many small backend services**
*   Each service has:
    *   Its own URL
    *   Its own port
    *   Its own authentication needs
    *   Its own rate limits

Example:

    User Service        → http://localhost:8081
    Order Service       → http://localhost:8082
    Payment Service     → http://localhost:8083
    Inventory Service   → http://localhost:8084

### ❌ If clients directly call services

The client must:

*   Know all service URLs
*   Handle security per service
*   Handle retries
*   Handle failures
*   Change code when services move or scale

This creates **tight coupling** and **chaos**.

Spring and Baeldung both describe this as a core anti‑pattern in microservices.

***

## LAYER 1 — WHAT IS AN API GATEWAY? (EVERY WORD EXPLAINED)

### ✅ “API Gateway” — definition (from Spring docs)

> **An API Gateway is a single entry point that routes client requests to appropriate backend services**

Now let’s break **every single word**.

***

### ✅ API

**API** = *Application Programming Interface*

In this context:

*   HTTP APIs
*   REST endpoints
*   URLs like `/users`, `/orders`, `/payments`

***

### ✅ Gateway

A **Gateway** is:

> Something that **sits at the edge** and controls **who goes where and how**

Network analogy:

*   Airport security gate
*   Traffic intersection controller

***

### ✅ Single Entry Point (EXTREMELY IMPORTANT)

**Single Entry Point means:**

✅ Client talks to **one URL only**
❌ Client never talks to services directly

Example:

    Client → https://api.company.com

NOT:

    Client → user-service
    Client → order-service
    Client → payment-service

Spring calls this the **edge service pattern**.

***

### ✅ Routes requests

**Routing** means:

> Deciding **which backend service** should handle a request

Example:

*   `/users/**` → User Service
*   `/orders/**` → Order Service

***

## LAYER 2 — WHERE SPRING CLOUD GATEWAY FITS

### ✅ What is Spring Cloud Gateway?

From **Spring Official Documentation**:

> **Spring Cloud Gateway is a lightweight, reactive API Gateway built on Spring WebFlux and Project Reactor**

Now let’s break *that sentence word by word*.

***

## LAYER 3 — “REACTIVE” (DO NOT SKIP THIS)

### ✅ What does “Reactive” mean?

Reactive ≠ async ≠ multithreading ≠ faster by default  
Reactive means:

> **Non‑blocking, event‑driven request processing**

***

### ✅ Blocking vs Non‑Blocking (VERY IMPORTANT)

**Blocking model**:

*   One request = one thread
*   Thread waits during IO (network calls)

**Non‑blocking model**:

*   Thread does NOT wait
*   Uses events + callbacks
*   Threads are reused

Spring Cloud Gateway is **100% non‑blocking**.  
It is built on **Spring WebFlux**, not Spring MVC.

***

### ✅ Why Gateway must be reactive

An API Gateway:

*   Handles **all incoming traffic**
*   Is the **heaviest component**
*   Must scale massively

Blocking gateways collapse under load.  
Reactive gateways survive.

Baeldung explicitly states this design reason.

***

## LAYER 4 — CORE BUILDING BLOCKS OF SPRING CLOUD GATEWAY

Spring Cloud Gateway has **3 foundational building blocks**.

***

### ✅ 1️⃣ Route (WHAT goes WHERE)

#### ✅ Route — definition

A **Route** defines:

*   **When** a request matches
*   **Where** it should be forwarded

Spring definition:

Conceptually:

    IF request matches condition
    THEN forward to service

***

### ✅ 2️⃣ Predicate (WHEN to apply route)

#### ✅ Predicate — definition

A **Predicate** is:

> A **boolean condition** evaluated on the request

Examples of predicates:

*   Path predicate (`/users/**`)
*   Method predicate (GET, POST)
*   Header predicate
*   Query parameter predicate

If predicate = true → route applies.

Predicate concept is explicitly documented by Spring.

***

### ✅ 3️⃣ Filter (WHAT happens to the request)

#### ✅ Filter — definition

A **Filter** is:

> Logic that **modifies the request or response**

This can happen:

*   Before forwarding the request
*   After receiving the response

Baeldung categorizes filters as:

*   Pre‑filters
*   Post‑filters

***

## LAYER 5 — BUILT‑IN FEATURES (EVERY WORD EXPLAINED)

***

### ✅ Logging

**Logging filter** means:

*   Log request path
*   Log headers
*   Log response status

Purpose:

*   Monitoring
*   Debugging
*   Auditing

Implemented via **GatewayFilter**.

***

### ✅ Rate Limiting

#### ✅ Rate Limiting — definition

> Limiting number of requests a client can make in a time window

Example:

*   100 requests per minute per user

Spring Cloud Gateway provides **built‑in rate limiting** using:

*   Token Bucket algorithm
*   Redis (commonly)

Documented in Spring Gateway docs.

***

### ✅ Authentication

Gateway commonly handles:

*   JWT validation
*   OAuth2 token checks
*   API key validation

Why at gateway?
✅ Avoid duplicate security logic in every service  
✅ Keep services internal and simple

Baeldung explicitly recommends this pattern.

***

## LAYER 6 — API GATEWAY + EUREKA (VERY IMPORTANT)

Spring Cloud Gateway integrates with **Eureka**.

How?

*   Gateway uses **logical service names**
*   Resolves them via Eureka
*   Forwards request dynamically

Example:

    lb://USER-SERVICE

`lb` = load balancer  
Documented in Spring Gateway docs.

***

## LAYER 7 — WHAT API GATEWAY IS NOT

❌ Not a business logic layer  
❌ Not a database  
❌ Not a service aggregator (usually)  
✅ It is an **edge traffic controller**

***

## FINAL MENTAL MODEL (READ THIS TWICE)

> **Spring Cloud Gateway is a reactive, non‑blocking edge service that exposes a single entry point, evaluates request predicates, applies filters such as logging, authentication, and rate limiting, and forwards traffic to backend services discovered dynamically via Eureka.**

***

## ✅ Official Sources Used

*   Spring Cloud Gateway Documentation
*   Baeldung: API Gateway with Spring Cloud Gateway

***

### 🔥 NEXT (STRONGLY RECOMMENDED)

1️⃣ **Full request lifecycle diagram through Gateway**  
2️⃣ **Gateway vs Feign (VERY IMPORTANT DIFFERENCE)**  
3️⃣ **Gateway Filters — one by one, deeply**  
4️⃣ **Why Gateway is reactive but services can be blocking**
