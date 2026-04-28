# Spring Security Masterclass — Step 7: Authentication Controller

---

## 7.1 What This Section Builds

So far we have:
- JWT generation and validation (`JwtService`)
- A filter that reads JWT from requests (`JwtAuthenticationFilter`)
- Security configuration wiring everything together

What is missing: **the door where users enter the system**.

Users do not have a JWT yet. They need to prove their identity first — with username and password — and receive a JWT in return. That is the job of the authentication controller.

```
CURRENT STATE:
  User has JWT → can call /api/** ✓
  User has no JWT → cannot access anything

WHAT WE BUILD NOW:
  User sends username + password → POST /auth/login
  Server verifies → returns JWT
  User now has JWT → can call /api/** ✓
```

---

## 7.2 The Authentication Flow — Internals

Before writing code, understand exactly what happens when a login request arrives:

```
POST /auth/login
Body: { "username": "john", "password": "secret123" }

        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│ JwtAuthenticationFilter                                      │
│  → No "Authorization: Bearer" header found                   │
│  → Passes request along unchanged                            │
└──────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│ AuthorizationFilter                                          │
│  → /auth/login matches .requestMatchers("/auth/**").permitAll│
│  → Allowed through without authentication                    │
└──────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│ AuthController.login()                                       │
│  → Receives LoginRequest(username, password)                 │
│  → Calls AuthenticationManager.authenticate()                │
└──────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│ AuthenticationManager (ProviderManager)                      │
│  → Delegates to DaoAuthenticationProvider                    │
└──────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│ DaoAuthenticationProvider                                    │
│  → userDetailsService.loadUserByUsername("john")             │
│       → loads UserDetails from DB/memory                     │
│  → passwordEncoder.matches("secret123", storedHash)          │
│       → verifies BCrypt hash                                 │
│  → SUCCESS → returns authenticated Authentication object     │
│  → FAILURE → throws BadCredentialsException                  │
└──────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────┐
│ AuthController.login() (continues)                           │
│  → Authentication succeeded                                  │
│  → Load UserDetails                                          │
│  → jwtService.generateToken(userDetails)                     │
│  → Returns { "token": "eyJhbGci..." }                        │
└──────────────────────────────────────────────────────────────┘
```

**Key insight:** We do NOT manually verify the password. We hand the credentials to `AuthenticationManager` and let Spring Security's `DaoAuthenticationProvider` handle it. This is the correct approach — never write your own password verification logic.

---

## 7.3 Request and Response Models

```java
// src/main/java/com/example/demo/dto/LoginRequest.java

package com.example.demo.dto;

public class LoginRequest {

    private String username;
    private String password;

    // Default constructor required for JSON deserialization
    public LoginRequest() {}

    public LoginRequest(String username, String password) {
        this.username = username;
        this.password = password;
    }

    public String getUsername() { return username; }
    public String getPassword() { return password; }

    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
}
```

```java
// src/main/java/com/example/demo/dto/AuthResponse.java

package com.example.demo.dto;

public class AuthResponse {

    private String token;
    private String tokenType = "Bearer";
    private long expiresIn; // milliseconds

    public AuthResponse() {}

    public AuthResponse(String token, long expiresIn) {
        this.token = token;
        this.expiresIn = expiresIn;
    }

    public String getToken() { return token; }
    public String getTokenType() { return tokenType; }
    public long getExpiresIn() { return expiresIn; }

    public void setToken(String token) { this.token = token; }
    public void setTokenType(String tokenType) { this.tokenType = tokenType; }
    public void setExpiresIn(long expiresIn) { this.expiresIn = expiresIn; }
}
```

**Why include `tokenType` and `expiresIn` in the response?**

```
tokenType  → Tells the client how to use the token ("Bearer" goes in Authorization header)
expiresIn  → Client can schedule token refresh before expiry
             Without this, client must decode JWT or guess when to refresh
```

This matches the OAuth2 token response format — a standard your API consumers will expect.

---

## 7.4 The Authentication Service

We put authentication logic in a service layer — not directly in the controller. Controllers should only handle HTTP concerns (request/response). Business logic belongs in services.

```java
// src/main/java/com/example/demo/service/AuthService.java

package com.example.demo.service;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.security.JwtService;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.stereotype.Service;

@Service
public class AuthService {

    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;
    private final JwtService jwtService;

    public AuthService(
            AuthenticationManager authenticationManager,
            UserDetailsService userDetailsService,
            JwtService jwtService
    ) {
        this.authenticationManager = authenticationManager;
        this.userDetailsService = userDetailsService;
        this.jwtService = jwtService;
    }

    public AuthResponse login(LoginRequest request) {

        // ── Step 1: Authenticate credentials ──────────────────────────
        // We create an UNAUTHENTICATED token (2-arg constructor)
        // and pass it to AuthenticationManager for verification.
        //
        // If credentials are wrong:
        //   DaoAuthenticationProvider throws BadCredentialsException
        //   Spring Security maps this to 401 Unauthorized
        //
        // If user not found:
        //   UserDetailsService throws UsernameNotFoundException
        //   Spring Security maps this to 401 (does NOT reveal "user not found"
        //   to prevent username enumeration attacks)
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        request.getUsername(),  // principal
                        request.getPassword()   // credentials
                )
        );

        // ── Step 2: Load full UserDetails ─────────────────────────────
        // Authentication succeeded — load the user to generate their token
        // We call loadUserByUsername again because authenticate()
        // does not return UserDetails directly (returns Authentication)
        UserDetails userDetails = userDetailsService
                .loadUserByUsername(request.getUsername());

        // ── Step 3: Generate JWT ───────────────────────────────────────
        String token = jwtService.generateToken(userDetails);

        // ── Step 4: Return response ────────────────────────────────────
        return new AuthResponse(token, jwtService.getExpirationTime());
    }
}
```

### Add `getExpirationTime()` to JwtService:

```java
// Add this method to JwtService.java

public long getExpirationTime() {
    return jwtExpiration;
}
```

---

## 7.5 The Authentication Controller

```java
// src/main/java/com/example/demo/controller/AuthController.java

package com.example.demo.controller;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.service.AuthService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(response);
    }
}
```

**Why is the controller this thin?**

```
Controller responsibility:  HTTP mapping, request parsing, response formatting
Service responsibility:     Business logic, orchestration

If you put authentication logic in the controller:
  → Hard to unit test (HTTP context required)
  → Hard to reuse (what if you add OAuth2 later?)
  → Violates Single Responsibility Principle
```

---

## 7.6 Exception Handling for Authentication Failures

When `AuthenticationManager.authenticate()` fails, Spring throws exceptions. We need to handle them gracefully and return proper JSON responses — not stack traces.

```java
// src/main/java/com/example/demo/exception/GlobalExceptionHandler.java

package com.example.demo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.DisabledException;
import org.springframework.security.authentication.LockedException;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import jakarta.servlet.http.HttpServletRequest;
import java.time.LocalDateTime;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // ── Wrong username or password ─────────────────────────────────────
    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<Map<String, Object>> handleBadCredentials(
            BadCredentialsException ex,
            HttpServletRequest request
    ) {
        return buildErrorResponse(
            HttpStatus.UNAUTHORIZED,
            // IMPORTANT: never say "wrong password" or "user not found"
            // Generic message prevents username enumeration attacks
            "Invalid username or password",
            request.getServletPath()
        );
    }

    // ── Account disabled ───────────────────────────────────────────────
    @ExceptionHandler(DisabledException.class)
    public ResponseEntity<Map<String, Object>> handleDisabled(
            DisabledException ex,
            HttpServletRequest request
    ) {
        return buildErrorResponse(
            HttpStatus.UNAUTHORIZED,
            "Account is disabled",
            request.getServletPath()
        );
    }

    // ── Account locked ─────────────────────────────────────────────────
    @ExceptionHandler(LockedException.class)
    public ResponseEntity<Map<String, Object>> handleLocked(
            LockedException ex,
            HttpServletRequest request
    ) {
        return buildErrorResponse(
            HttpStatus.UNAUTHORIZED,
            "Account is locked",
            request.getServletPath()
        );
    }

    // ── Generic fallback ───────────────────────────────────────────────
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneral(
            Exception ex,
            HttpServletRequest request
    ) {
        return buildErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred",
            request.getServletPath()
        );
    }

    private ResponseEntity<Map<String, Object>> buildErrorResponse(
            HttpStatus status,
            String message,
            String path
    ) {
        Map<String, Object> body = Map.of(
            "timestamp", LocalDateTime.now().toString(),
            "status", status.value(),
            "error", status.getReasonPhrase(),
            "message", message,
            "path", path
        );
        return ResponseEntity.status(status).body(body);
    }
}
```

**Why use a generic error message for bad credentials?**

```
❌ BAD — reveals information:
  "User 'john' not found"    → attacker knows john doesn't exist, tries others
  "Wrong password for john"  → attacker knows john EXISTS, can brute-force password

✅ GOOD — reveals nothing:
  "Invalid username or password"
  → attacker cannot determine if username exists or password was wrong
  → prevents username enumeration attacks
```

---

## 7.7 Updated Project Structure

```
src/main/java/com/example/demo/
│
├── config/
│   ├── ApplicationConfig.java
│   └── SecurityConfig.java
│
├── controller/
│   ├── AuthController.java        ← POST /auth/login
│   └── ApiController.java         ← GET /api/profile, /api/data
│
├── dto/
│   ├── LoginRequest.java          ← Request body model
│   └── AuthResponse.java          ← Response body model
│
├── exception/
│   └── GlobalExceptionHandler.java← Handles auth exceptions
│
├── security/
│   ├── JwtService.java
│   ├── JwtAuthenticationFilter.java
│   ├── JwtAuthenticationEntryPoint.java
│   ├── JwtAccessDeniedHandler.java
│   └── UserDetailsServiceImpl.java
│
└── service/
    └── AuthService.java           ← Authentication business logic
```

---

## Hands-On: Test Using Postman (and curl)

### Step 1: Start the application

Verify `application.properties`:

```properties
app.jwt.secret=mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
app.jwt.expiration=3600000
logging.level.com.example.demo=DEBUG
logging.level.org.springframework.security=DEBUG
```

---

### Test 1: Successful Login

**Postman:**
```
Method: POST
URL:    http://localhost:8080/auth/login
Headers: Content-Type: application/json
Body (raw JSON):
{
    "username": "john",
    "password": "secret123"
}
```

**curl:**
```bash
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john","password":"secret123"}'
```

**Expected Response: 200 OK**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwiaWF0IjoxNzA...",
  "tokenType": "Bearer",
  "expiresIn": 3600000
}
```

**Console logs:**
```
=== JwtAuthenticationFilter triggered for: POST /auth/login ===
No Bearer token found. Passing request along.
Authentication BEFORE filter: null
--- AuthService.login() called for: john
--- DaoAuthenticationProvider verifying credentials...
--- UserDetailsServiceImpl.loadUserByUsername("john")
--- BCrypt password match: true
--- Authentication successful
--- Generating JWT for: john
--- JWT generated successfully
```

---

### Test 2: Wrong Password

```bash
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john","password":"wrongpassword"}'
```

**Expected Response: 401 Unauthorized**
```json
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 401,
  "error": "Unauthorized",
  "message": "Invalid username or password",
  "path": "/auth/login"
}
```

---

### Test 3: User Does Not Exist

```bash
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"nobody","password":"secret123"}'
```

**Expected Response: 401 Unauthorized**
```json
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 401,
  "error": "Unauthorized",
  "message": "Invalid username or password",
  "path": "/auth/login"
}
```

✅ Same message as wrong password — does not reveal whether user exists.

---

### Test 4: Use the Token to Call Protected Endpoint

Copy the token from Test 1, then:

**Postman:**
```
Method:  GET
URL:     http://localhost:8080/api/profile
Headers: Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huI...
```

**curl:**
```bash
# Save token first
TOKEN="eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huI..."

curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer $TOKEN"
```

**Expected Response: 200 OK**
```json
{
  "username": "john",
  "authorities": "[ROLE_USER]",
  "isAuthenticated": true
}
```

---

### Test 5: Call Protected Endpoint Without Token

```bash
curl http://localhost:8080/api/profile
```

**Expected Response: 401 Unauthorized**
```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Full authentication is required to access this resource",
  "path": "/api/profile"
}
```

---

### Test 6: Full End-to-End Flow in One Script

```bash
#!/bin/bash
echo "=== Step 1: Login and get token ==="
RESPONSE=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john","password":"secret123"}')

echo "Login response: $RESPONSE"

# Extract token from response
TOKEN=$(echo $RESPONSE | grep -o '"token":"[^"]*"' | cut -d'"' -f4)
echo "Token: $TOKEN"

echo ""
echo "=== Step 2: Call protected endpoint with token ==="
curl -s http://localhost:8080/api/profile \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

echo ""
echo "=== Step 3: Call protected endpoint without token ==="
curl -s http://localhost:8080/api/profile | python3 -m json.tool
```

---

## 7.8 Postman Collection Setup (Professional Workflow)

In real projects, use Postman environment variables:

```
Environment: "Local Dev"
Variables:
  base_url  = http://localhost:8080
  token     = (empty — filled after login)
```

**Login request — add this to Tests tab:**
```javascript
// Postman Tests tab for POST /auth/login
const response = pm.response.json();
pm.environment.set("token", response.token);
pm.test("Login successful", () => {
    pm.response.to.have.status(200);
    pm.expect(response.token).to.be.a('string');
});
```

**Protected request — Authorization tab:**
```
Type:  Bearer Token
Token: {{token}}    ← automatically uses saved token
```

This eliminates manual copy-pasting of tokens between requests.

---

## 7.9 What Happens Inside AuthenticationManager.authenticate()

This deserves a precise walkthrough because it is a black box to most developers:

```java
authenticationManager.authenticate(
    new UsernamePasswordAuthenticationToken("john", "secret123")
);
```

```
ProviderManager.authenticate(token)
        │
        │ loops through registered providers
        ▼
DaoAuthenticationProvider.supports(UsernamePasswordAuthenticationToken.class)
        │ returns true → this provider handles this token type
        ▼
DaoAuthenticationProvider.authenticate(token)
        │
        ├─ Step 1: retrieveUser("john")
        │     └─ userDetailsService.loadUserByUsername("john")
        │           → returns UserDetails{username=john, password=$2a$12$..., roles=[ROLE_USER]}
        │           → if not found → throws UsernameNotFoundException
        │                           → internally mapped to BadCredentialsException
        │                           (hiding whether user exists)
        │
        ├─ Step 2: preAuthenticationChecks(userDetails)
        │     └─ isAccountNonExpired()  → throws AccountExpiredException if false
        │     └─ isAccountNonLocked()   → throws LockedException if false
        │     └─ isEnabled()            → throws DisabledException if false
        │
        ├─ Step 3: additionalAuthenticationChecks(userDetails, token)
        │     └─ passwordEncoder.matches("secret123", "$2a$12$...")
        │           → if false → throws BadCredentialsException
        │
        ├─ Step 4: postAuthenticationChecks(userDetails)
        │     └─ isCredentialsNonExpired() → throws CredentialsExpiredException if false
        │
        └─ Step 5: createSuccessAuthentication(userDetails, token)
              └─ returns new UsernamePasswordAuthenticationToken(
                     userDetails,             // principal
                     null,                    // credentials (cleared)
                     userDetails.getAuthorities() // roles
                 ) with authenticated = true
```

**This is why you never write password comparison code yourself.** Spring Security handles all edge cases: disabled accounts, locked accounts, expired credentials, password hashing. One `authenticate()` call covers all of it.

---

## Section 7 — Interview Summary

| Question | Answer |
|----------|--------|
| What does `AuthenticationManager.authenticate()` return on success? | An `Authentication` object with `authenticated=true`, credentials cleared, authorities populated |
| What exception is thrown for wrong credentials? | `BadCredentialsException` |
| Why use a generic error message for login failures? | Prevents username enumeration attacks — attacker cannot tell if username exists |
| Why call `loadUserByUsername()` again after `authenticate()`? | `authenticate()` returns `Authentication`, not `UserDetails`. We need `UserDetails` to generate JWT |
| What checks does `DaoAuthenticationProvider` perform? | Credential verification + account enabled/locked/expired/credentials-expired checks |
| Where should authentication logic live — controller or service? | Service layer — controllers handle only HTTP concerns |
| What does `tokenType: "Bearer"` in the response tell the client? | To send the token as `Authorization: Bearer <token>` in subsequent requests |
| Why is `/auth/login` accessible without a token? | `SecurityConfig` has `.requestMatchers("/auth/**").permitAll()` |

---

## Quick Understanding Check

**Q1.** Trace exactly what happens — class by class — when this request arrives:

```
POST /auth/login
Body: { "username": "john", "password": "secret123" }
```

Start from `JwtAuthenticationFilter` and end at the JSON response. Name every class involved.

**Q2.** A developer modifies `AuthService.login()` to do this:

```java
public AuthResponse login(LoginRequest request) {
    UserDetails userDetails = userDetailsService
            .loadUserByUsername(request.getUsername());
    
    if (passwordEncoder.matches(request.getPassword(), userDetails.getPassword())) {
        String token = jwtService.generateToken(userDetails);
        return new AuthResponse(token, jwtService.getExpirationTime());
    }
    throw new BadCredentialsException("Invalid credentials");
}
```

This works. But name THREE things that are wrong or missing compared to using `AuthenticationManager.authenticate()`.

**Q3.** After a successful login, the client receives:

```json
{ "token": "eyJhbGci...", "tokenType": "Bearer", "expiresIn": 3600000 }
```

The client calls `/api/profile` 61 minutes later with that token. Walk through exactly what happens in `JwtAuthenticationFilter` and what the client receives.
