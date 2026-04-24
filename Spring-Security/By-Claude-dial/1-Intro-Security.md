# Spring Security Masterclass — Step 1: Introduction to Spring Security

---

## 1.1 What is Spring Security Internally?

Most tutorials say *"Spring Security is a framework for authentication and authorization."* That's surface level. Let me show you what it **actually is** under the hood.

**Spring Security is fundamentally a chain of Servlet Filters.**

That's it. Before your request ever reaches your `@RestController`, it passes through a series of filters. These filters intercept the request, inspect it, and either allow it to proceed or reject it with a `401`/`403` response.

```
Incoming HTTP Request
        │
        ▼
┌─────────────────────────────┐
│     Servlet Filter Chain    │  ← Spring Security lives HERE
│  ┌─────────────────────┐    │
│  │ SecurityFilter 1    │    │
│  │ SecurityFilter 2    │    │
│  │ SecurityFilter 3    │    │
│  │ ...                 │    │
│  └─────────────────────┘    │
└─────────────────────────────┘
        │
        ▼
  DispatcherServlet
        │
        ▼
  Your @RestController
```

Spring Security inserts its own special filter — called `DelegatingFilterProxy` — into the standard Servlet filter chain. This proxy delegates all work to Spring's internal `FilterChainProxy`, which then runs a list of security-specific filters in order.

**Key insight:** Your application code runs AFTER all security filters pass. If any filter rejects the request, your controller never gets called.

---

## 1.2 Why Security is Needed in REST APIs

REST APIs are stateless and exposed over HTTP. Without security:

| Threat | Example |
|--------|---------|
| Unauthorized access | Anyone can call `/api/admin/users` |
| Data theft | Attacker reads another user's data |
| Privilege escalation | Regular user deletes records |
| Replay attacks | Stolen request reused infinitely |

Unlike a monolith with server-side sessions, **REST APIs have no built-in concept of "who is calling"** — every request must prove its identity.

---

## 1.3 Authentication vs Authorization (Real Definitions)

These two words are used interchangeably by beginners. They are completely different things.

### Authentication — *Who are you?*

Verifying identity. Proving you are who you claim to be.

```
Real world: Showing your passport at the airport
REST API:   Sending username + password → getting a token back
```

### Authorization — *What are you allowed to do?*

Checking permissions after identity is confirmed.

```
Real world: Your boarding pass only lets you into Gate 12, not Gate 5
REST API:   ADMIN can DELETE /users, USER cannot
```

### Concrete Example in a Banking API:

```
Step 1 — Authentication:
  POST /auth/login
  Body: { username: "john", password: "secret" }
  → Server verifies identity, returns JWT token

Step 2 — Authorization:
  GET /api/accounts/transfer (with JWT token)
  → Server checks: is John allowed to transfer money?
  → John has role USER, not ADMIN → allowed for own account only
  → John tries DELETE /api/admin/users → FORBIDDEN (403)
```

---

## 1.4 HTTP Basic vs Bearer Token

### HTTP Basic Authentication

Credentials sent with **every single request** in the `Authorization` header, Base64 encoded.

```
Authorization: Basic am9objpzZWNyZXQ=
                      ↑
               Base64("john:secret")
```

**Problems with Basic Auth for REST APIs:**
- Password travels with every request (even over HTTPS, it's risky)
- Server must validate credentials on every request (hits database every time)
- No expiry — if intercepted, valid forever
- Not suitable for microservices

### Bearer Token (JWT)

Client authenticates once, receives a **token**. That token is sent with subsequent requests.

```
Step 1: POST /auth/login → Server returns token
Step 2: GET /api/data
        Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

**Why Bearer Tokens win for REST APIs:**
- Token is self-contained (carries user info)
- Stateless — server doesn't store sessions
- Can expire (short-lived = safer)
- Works perfectly across microservices

---

## 1.5 Internal Flow: Request → Filters → Authentication → Authorization

This is the most important diagram you'll see in this entire course. Internalize it.

```
HTTP Request: GET /api/orders
Authorization: Bearer <token>
        │
        ▼
┌──────────────────────────────────────────┐
│           DelegatingFilterProxy          │  (Standard Servlet Filter)
└──────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────┐
│            FilterChainProxy              │  (Spring's master filter)
│                                          │
│  Runs filters in this ORDER:             │
│  1. SecurityContextPersistenceFilter     │  ← Loads security context
│  2. UsernamePasswordAuthenticationFilter │  ← Handles form login
│  3. BearerTokenAuthenticationFilter      │  ← Handles JWT (OAuth2)
│  4. ExceptionTranslationFilter           │  ← Converts exceptions to HTTP errors
│  5. AuthorizationFilter                  │  ← Checks permissions (last step)
└──────────────────────────────────────────┘
        │
        ▼ (if all filters pass)
┌──────────────────────────────────────────┐
│          DispatcherServlet               │
│              ↓                           │
│         @RestController                  │
└──────────────────────────────────────────┘
```

### What happens at each stage:

**Stage 1 — Filter extracts credentials:**
The JWT filter reads `Authorization: Bearer <token>` from the header.

**Stage 2 — Authentication object is created:**
Filter creates an `Authentication` object (e.g., `UsernamePasswordAuthenticationToken`) with user details.

**Stage 3 — Authentication is stored in SecurityContext:**
```java
SecurityContextHolder.getContext().setAuthentication(authentication);
```
This is how Spring Security "remembers" who the user is for the rest of this request.

**Stage 4 — AuthorizationFilter checks permissions:**
"Does this authenticated user have the right role/authority to access this endpoint?"

**Stage 5 — Request reaches your controller (or gets rejected).**

---

## Hands-On: Add Spring Security and Observe Default Behavior

### Step 1: Create a Spring Boot project

Add these dependencies to `pom.xml`:

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

### Step 2: Create a simple test controller

```java
// src/main/java/com/example/demo/controller/TestController.java

package com.example.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello, World!";
    }
}
```

### Step 3: Run the application

Look at your console. You will see something like:

```
Using generated security password: 8e24b8c2-4d6b-4a72-b5bc-7a29f45e3d22
```

**What just happened?**

Spring Security auto-configured itself. Without any configuration from you, it:
1. Created an in-memory user with username `user`
2. Generated a random password (printed in console)
3. Protected ALL endpoints by default
4. Enabled HTTP Basic authentication

### Step 4: Test it

Try calling `GET http://localhost:8080/hello` without credentials.

```bash
curl http://localhost:8080/hello
```

**Response:**
```json
{
  "status": 401,
  "error": "Unauthorized"
}
```

Now try with credentials:

```bash
curl -u user:8e24b8c2-4d6b-4a72-b5bc-7a29f45e3d22 http://localhost:8080/hello
```

**Response:** `Hello, World!`

### Step 5: Understand what just happened internally

```
Your request to /hello
        │
        ▼
BasicAuthenticationFilter  ← Reads "user:password" from Authorization header
        │
        ▼
UserDetailsService         ← Looks up "user" in memory
        │
        ▼
BCryptPasswordEncoder      ← Verifies password hash
        │
        ▼ (authenticated)
SecurityContextHolder      ← Stores Authentication object
        │
        ▼
AuthorizationFilter        ← Is authenticated? YES → proceed
        │
        ▼
TestController.hello()     ← Returns "Hello, World!"
```

### Step 6: Key observation about Spring Security's default behavior

Add this to `application.properties` to see what's happening:

```properties
logging.level.org.springframework.security=TRACE
```

Restart and hit the endpoint. You'll see logs like:

```
Security filter chain: [
  DisableEncodeUrlFilter
  WebAsyncManagerIntegrationFilter
  SecurityContextHolderFilter
  HeaderWriterFilter
  CsrfFilter
  LogoutFilter
  BasicAuthenticationFilter
  ...
  AuthorizationFilter
]
```

**This is the actual filter chain running on every request.** We will customize this in Section 6.

---

## Section 1 — Interview Summary

| Question | Answer |
|----------|--------|
| What is Spring Security internally? | A chain of Servlet filters that intercept requests before they reach controllers |
| What is the difference between Authentication and Authorization? | Authentication = verifying identity; Authorization = verifying permissions |
| Why use JWT over Basic Auth for REST APIs? | JWT is stateless, carries user info, can expire, works across microservices |
| What happens when no security config is provided? | Spring Boot auto-configures HTTP Basic auth with an in-memory user |
| Where is authenticated user info stored during a request? | In `SecurityContextHolder` via the `SecurityContext` |
| What rejects an unauthorized request before the controller? | The `AuthorizationFilter` (last in the chain) |

---

## Quick Understanding Check

Before I move to Section 2 (Spring Security Internals), answer these:

**Q1.** If a request has a valid JWT token but the user doesn't have the `ADMIN` role required for an endpoint — which stage fails: Authentication or Authorization? And what HTTP status code would you expect?

**Q2.** Look at this flow:
```
Request → Filter A → Filter B → Filter C → Controller
```
If Filter B rejects the request, does Filter C run? Does the Controller run?

**Q3.** In your hands-on test, you called `/hello` without credentials and got 401. If you called a valid user but they tried to access an endpoint they don't have permission for, what status code would you get — and why is it different from 401? your time answering these. Once you confirm understanding (or ask follow-up questions), I'll move to **Section 2: Spring Security Internals** — where we go deep into `SecurityFilterChain`, `SecurityContext`, `Authentication` object internals, and you'll actually log and trace the filter execution flow.
