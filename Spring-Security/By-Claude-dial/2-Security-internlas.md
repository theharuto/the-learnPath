# Spring Security Masterclass — Step 2: Spring Security Internals

---

## 2.1 Security Filter Chain (Deep Explanation)

Let me be precise about something most tutorials get wrong.

There is **not one** filter chain. There can be **multiple** filter chains, and Spring Security picks the right one based on the request URL pattern.

```
Incoming Request: GET /api/orders
                        │
                        ▼
            ┌───────────────────────┐
            │   FilterChainProxy    │  ← Master controller
            │                       │
            │  Chain 1: /public/**  │  ← Maybe no security
            │  Chain 2: /api/**     │  ← Full JWT security
            │  Chain 3: /admin/**   │  ← Extra strict security
            └───────────────────────┘
                        │
              Picks FIRST matching chain
                        │
                        ▼
              Runs ALL filters in that chain
```

**Key Rule:** `FilterChainProxy` matches the request URL against each chain in order and uses the **first match only**. Remaining chains are ignored.

In Spring Security 6, you define each chain as a `SecurityFilterChain` bean.

---

### The Default Filter Chain — What's Inside?

When you add Spring Security with zero configuration, you get these filters in this exact order:

```
 1.  DisableEncodeUrlFilter
 2.  WebAsyncManagerIntegrationFilter
 3.  SecurityContextHolderFilter          ← IMPORTANT
 4.  HeaderWriterFilter
 5.  CsrfFilter                           ← IMPORTANT
 6.  LogoutFilter
 7.  UsernamePasswordAuthenticationFilter ← IMPORTANT
 8.  DefaultLoginPageGeneratingFilter
 9.  DefaultLogoutPageGeneratingFilter
10.  BasicAuthenticationFilter            ← IMPORTANT
11.  RequestCacheAwareFilter
12.  SecurityContextHolderAwareRequestFilter
13.  AnonymousAuthenticationFilter        ← IMPORTANT
14.  ExceptionTranslationFilter           ← IMPORTANT
15.  AuthorizationFilter                  ← IMPORTANT (LAST)
```

Let me explain the ones that actually matter for JWT-based APIs:

---

### Filter 3: SecurityContextHolderFilter

**Job:** At the start of every request, loads the `SecurityContext` (who is logged in). At the end of the request, clears it.

```
Request starts  → Loads SecurityContext into SecurityContextHolder
Request ends    → Clears SecurityContext (thread cleanup)
```

**Why clearing matters:** Spring uses a thread-per-request model. If you don't clear the context, the next request handled by the same thread would inherit the previous user's identity. That would be catastrophic.

```java
// What this filter does internally (simplified):
SecurityContext context = securityContextRepository.loadContext(request);
SecurityContextHolder.setContext(context);
try {
    chain.doFilter(request, response); // rest of filters run here
} finally {
    SecurityContextHolder.clearContext(); // ALWAYS clears, even on exception
}
```

---

### Filter 5: CsrfFilter

**Job:** Protects against Cross-Site Request Forgery attacks.

**For REST APIs with JWT:** We disable this. Here's why:

CSRF attacks work by tricking a browser into sending requests using the browser's stored cookies. If your API uses JWT in the `Authorization` header (not cookies), CSRF is not a threat — browsers cannot add custom headers cross-origin.

```
CSRF relevant:   Cookie-based sessions (browser auto-sends cookies)
CSRF irrelevant: JWT in Authorization header (browser won't add this automatically)
```

We will explicitly disable CSRF in Section 6.

---

### Filter 13: AnonymousAuthenticationFilter

**Job:** If no authentication has been set by previous filters, this filter sets an **anonymous** authentication object.

```java
// What it does:
if (SecurityContextHolder.getContext().getAuthentication() == null) {
    // No one authenticated yet — set anonymous user
    SecurityContextHolder.getContext().setAuthentication(
        new AnonymousAuthenticationToken("key", "anonymousUser", 
            List.of(new SimpleGrantedAuthority("ROLE_ANONYMOUS")))
    );
}
```

**Why this matters:** The `AuthorizationFilter` (filter 15) always has an `Authentication` object to work with — either a real user or an anonymous one. This is how Spring Security cleanly handles "not logged in" cases.

---

### Filter 14: ExceptionTranslationFilter

**Job:** Catches security exceptions and converts them into proper HTTP responses.

```
AccessDeniedException    → 403 Forbidden
AuthenticationException  → 401 Unauthorized (redirects to login or returns 401)
```

This filter sits just before `AuthorizationFilter` specifically so it can catch whatever `AuthorizationFilter` throws.

```
ExceptionTranslationFilter
        │
        ▼ (calls next filter)
AuthorizationFilter ──throws AccessDeniedException──→ caught by ExceptionTranslationFilter
                                                               │
                                                               ▼
                                                        Returns 403 response
```

---

### Filter 15: AuthorizationFilter

**Job:** The final gatekeeper. Checks if the authenticated user (or anonymous user) has permission to access the requested URL.

This is the filter that enforces your rules like:
- `/api/**` requires authentication
- `/admin/**` requires `ROLE_ADMIN`
- `/public/**` is open to everyone

---

## 2.2 What is SecurityContext?

`SecurityContext` is a simple container that holds one thing: the current `Authentication` object.

```java
public interface SecurityContext {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

`SecurityContextHolder` is the class that stores the `SecurityContext` for the current thread.

```
Thread 1 (Request from User John):
    SecurityContextHolder → SecurityContext → Authentication(john, ROLE_USER)

Thread 2 (Request from User Admin):
    SecurityContextHolder → SecurityContext → Authentication(admin, ROLE_ADMIN)

Thread 3 (Unauthenticated request):
    SecurityContextHolder → SecurityContext → AnonymousAuthentication
```

Each thread has its own isolated `SecurityContext`. This is achieved using `ThreadLocal` internally.

```java
// Internally, SecurityContextHolder uses ThreadLocal:
private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();
```

**ThreadLocal** means each thread has its own copy of the variable. Thread 1's security context never leaks into Thread 2.

### Accessing SecurityContext in your code

You can read the current user anywhere in your application:

```java
// Get current authenticated user
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
String username = authentication.getName();
Collection<? extends GrantedAuthority> roles = authentication.getAuthorities();
```

This works in any `@Service`, `@Repository`, or `@Controller` — because it reads from the current thread's context.

---

## 2.3 What is the Authentication Object?

`Authentication` is an interface. It represents the "who is this person and what can they do" concept.

```java
public interface Authentication extends Principal {
    
    // The user's roles/permissions (e.g., ROLE_USER, ROLE_ADMIN)
    Collection<? extends GrantedAuthority> getAuthorities();
    
    // The credentials (password, token) — cleared after authentication
    Object getCredentials();
    
    // Extra details (IP address, session ID, etc.)
    Object getDetails();
    
    // The principal — usually a UserDetails object or username string
    Object getPrincipal();
    
    // Has this Authentication been verified?
    boolean isAuthenticated();
    
    void setAuthenticated(boolean isAuthenticated);
}
```

### Authentication Lifecycle

```
BEFORE authentication:
  Authentication object created with:
    - principal   = username (String)
    - credentials = password (String)
    - authenticated = false

AFTER authentication (AuthenticationManager verifies):
  Authentication object updated with:
    - principal   = UserDetails object (full user info)
    - credentials = null (cleared for security)
    - authorities = [ROLE_USER, ROLE_ADMIN]
    - authenticated = true
```

### Most Common Implementation: UsernamePasswordAuthenticationToken

```java
// Before authentication (we create this with credentials):
Authentication request = new UsernamePasswordAuthenticationToken(
    "john",    // principal (username)
    "secret"   // credentials (password)
);

// After authentication (Spring creates this after verifying):
Authentication result = new UsernamePasswordAuthenticationToken(
    userDetails,           // principal (full UserDetails object)
    null,                  // credentials (CLEARED)
    userDetails.getAuthorities()  // roles granted
);
```

---

## 2.4 How a Request Flows Through Filters (Complete Picture)

Let me trace an actual JWT request end-to-end so you can visualize the entire flow.

```
Request: GET /api/orders
Headers: Authorization: Bearer eyJhbGci...
         
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Filter 3: SecurityContextHolderFilter                   │
│   → Creates empty SecurityContext for this thread       │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Filter 5: CsrfFilter                                    │
│   → DISABLED (we disable it in config)                  │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Our Custom JwtAuthenticationFilter  ← WE BUILD THIS     │
│   1. Reads "Authorization: Bearer eyJhbGci..."         │
│   2. Extracts and validates the JWT                     │
│   3. Extracts username from JWT payload                 │
│   4. Loads UserDetails from database                    │
│   5. Creates Authentication object                      │
│   6. Sets it in SecurityContextHolder                   │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Filter 13: AnonymousAuthenticationFilter                │
│   → SecurityContext already has Authentication          │
│   → Skips (does nothing)                                │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Filter 14: ExceptionTranslationFilter                   │
│   → Wraps the next filter call in try/catch             │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ Filter 15: AuthorizationFilter                          │
│   → Reads Authentication from SecurityContextHolder     │
│   → Checks: does this user have access to /api/orders?  │
│   → YES → proceeds                                      │
└─────────────────────────────────────────────────────────┘
         │
         ▼
    DispatcherServlet
         │
         ▼
    OrderController.getOrders()
         │
         ▼
    Response: 200 OK + JSON data
```

---

## 2.5 Where Does Authentication Actually Happen?

This confuses many developers. Let me be precise.

**Filters do NOT authenticate by themselves.** They delegate to `AuthenticationManager`.

```
Filter
  │
  │ creates Authentication request object
  │ calls:
  ▼
AuthenticationManager.authenticate(authRequest)
  │
  │ delegates to:
  ▼
AuthenticationProvider
  │
  ├── DaoAuthenticationProvider (username/password against DB)
  ├── JwtAuthenticationProvider (JWT token)
  ├── LdapAuthenticationProvider (LDAP)
  └── ... (you can write custom ones)
  │
  ▼
Returns authenticated Authentication object (or throws exception)
```

### AuthenticationManager Interface

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) 
        throws AuthenticationException;
}
```

The standard implementation is `ProviderManager`, which loops through all registered `AuthenticationProvider`s until one can handle the authentication type.

```java
// ProviderManager internally (simplified):
for (AuthenticationProvider provider : providers) {
    if (provider.supports(authentication.getClass())) {
        return provider.authenticate(authentication); // delegates here
    }
}
throw new ProviderNotFoundException("No provider found");
```

**For JWT:** We will bypass `AuthenticationManager` in our custom filter. We'll validate the JWT ourselves and directly set the `Authentication` in `SecurityContextHolder`. This is the standard approach for JWT.

---

## Hands-On: Log the Filter Execution Flow

### Step 1: Add this to `application.properties`

```properties
logging.level.org.springframework.security=TRACE
logging.level.org.springframework.security.web.FilterChainProxy=DEBUG
```

### Step 2: Create a custom filter to observe positioning

```java
// src/main/java/com/example/demo/filter/LoggingFilter.java

package com.example.demo.filter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class LoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // BEFORE other security filters process the request
        System.out.println("=== REQUEST ENTERING FILTER ===");
        System.out.println("URL: " + request.getRequestURI());
        System.out.println("Method: " + request.getMethod());

        Authentication authBefore = SecurityContextHolder.getContext().getAuthentication();
        System.out.println("Authentication BEFORE: " + authBefore);

        // Pass request to next filter in chain
        filterChain.doFilter(request, response);

        // AFTER all security filters have processed
        Authentication authAfter = SecurityContextHolder.getContext().getAuthentication();
        System.out.println("Authentication AFTER: " + authAfter);
        System.out.println("Response Status: " + response.getStatus());
        System.out.println("=== REQUEST LEAVING FILTER ===");
    }
}
```

### Step 3: Create a minimal Security Configuration to register the filter

```java
// src/main/java/com/example/demo/config/SecurityConfig.java

package com.example.demo.config;

import com.example.demo.filter.LoggingFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final LoggingFilter loggingFilter;

    public SecurityConfig(LoggingFilter loggingFilter) {
        this.loggingFilter = loggingFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Add our logging filter BEFORE UsernamePasswordAuthenticationFilter
            .addFilterBefore(loggingFilter, UsernamePasswordAuthenticationFilter.class)
            
            // For now, require authentication for all requests
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            
            // Keep HTTP Basic so we can test
            .httpBasic(basic -> {});

        return http.build();
    }
}
```

### Step 4: Run and observe

Hit `GET http://localhost:8080/hello` with correct credentials:

```bash
curl -u user:<generated-password> http://localhost:8080/hello
```

**You'll see in console:**

```
=== REQUEST ENTERING FILTER ===
URL: /hello
Method: GET
Authentication BEFORE: null          ← No auth yet (our filter runs first)

=== REQUEST LEAVING FILTER ===
Authentication AFTER: UsernamePasswordAuthenticationToken [
    Principal=org.springframework.security.core.userdetails.User [
        Username=user, 
        Password=[PROTECTED], 
        Authorities=[ROLE_USER]
    ], 
    Credentials=[PROTECTED],   ← Already cleared!
    Authenticated=true, 
    Authorities=[ROLE_USER]
]
Response Status: 200
```

**What this proves:**
1. Before `BasicAuthenticationFilter` runs → `Authentication` is `null`
2. After `BasicAuthenticationFilter` runs → `Authentication` is fully populated
3. Our filter runs BEFORE `BasicAuthenticationFilter` (as we configured)
4. Credentials are cleared after authentication (security best practice)

### Step 5: Observe what happens with a bad request

```bash
curl http://localhost:8080/hello   # No credentials
```

**Console output:**

```
=== REQUEST ENTERING FILTER ===
Authentication BEFORE: null
=== REQUEST LEAVING FILTER ===
Authentication AFTER: null          ← Still null, anonymous filter ran but...
Response Status: 401                ← AuthorizationFilter rejected it
```

---

## Critical Concept: Filter Order Matters

```java
// Adding filter BEFORE a specific filter:
.addFilterBefore(myFilter, UsernamePasswordAuthenticationFilter.class)

// Adding filter AFTER a specific filter:
.addFilterAfter(myFilter, UsernamePasswordAuthenticationFilter.class)

// Adding filter AT the same position:
.addFilterAt(myFilter, UsernamePasswordAuthenticationFilter.class)
```

For our JWT implementation, we'll add our custom JWT filter **before** `UsernamePasswordAuthenticationFilter` — so our filter runs first, validates the token, and sets the `Authentication` in context. By the time `AuthorizationFilter` runs, the user is already authenticated.

---

## Section 2 — Interview Summary

| Question | Answer |
|----------|--------|
| What is `FilterChainProxy`? | Spring Security's master filter that selects and runs the correct `SecurityFilterChain` |
| What is `SecurityContextHolder`? | A `ThreadLocal`-based holder for the current request's `SecurityContext` |
| Why is `SecurityContext` cleared after each request? | Thread pool reuse — clearing prevents previous user's context leaking into next request |
| What does `AnonymousAuthenticationFilter` do? | Sets an anonymous `Authentication` if no filter has authenticated the request yet |
| What does `ExceptionTranslationFilter` do? | Converts `AccessDeniedException` → 403, `AuthenticationException` → 401 |
| Who actually verifies credentials? | `AuthenticationProvider` (via `AuthenticationManager`), NOT the filter itself |
| What is `UsernamePasswordAuthenticationToken`? | The most common `Authentication` implementation for username/password and JWT |

---

## Quick Understanding Check

Answer these before we proceed to JWT:

**Q1.** You have two `SecurityFilterChain` beans. Chain 1 matches `/public/**` and Chain 2 matches `/**`. A request comes in for `/public/info`. Which chain handles it, and does Chain 2 ever run for this request?

**Q2.** Look at this code:
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```
If you call this line inside a `@Service` method during a request, what will it return for:
- (a) An authenticated user
- (b) An unauthenticated request that passed through `AnonymousAuthenticationFilter`

**Q3.** Why does our `LoggingFilter` show `Authentication BEFORE: null` but `Authentication AFTER: <user object>`? Which specific filter caused this change?
