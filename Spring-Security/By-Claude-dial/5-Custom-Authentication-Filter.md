# Spring Security Masterclass — Step 5: Custom Authentication Filter

---

## 5.1 Why We Need a Custom Filter

Recall the flow from Section 2:

```
Request → Filter Chain → Controller
```

Spring Security has no built-in filter that knows how to handle YOUR JWT format, YOUR secret key, and YOUR user loading logic.

We need to teach Spring Security how to:

```
1. Intercept every incoming request
2. Look for "Authorization: Bearer <token>" header
3. Extract the token
4. Validate it using JwtService
5. Load the user from UserDetailsService
6. Set Authentication in SecurityContextHolder
7. Let the request continue
```

This is exactly what our custom filter does. Without it, Spring Security has no idea who the user is — every request looks anonymous.

---

## 5.2 OncePerRequestFilter — The Right Base Class

Spring provides several filter base classes. We use `OncePerRequestFilter`.

**Why `OncePerRequestFilter` specifically?**

```
Standard Servlet Filter:
  → Can be called multiple times per request
  → Spring internally forwards requests (error handling, async)
  → Your filter might execute 2-3 times for one HTTP request
  → You'd validate JWT multiple times — wasteful and error-prone

OncePerRequestFilter:
  → Guarantees exactly ONE execution per HTTP request
  → Handles internal forwards/includes automatically
  → Correct choice for security filters
```

```java
// What OncePerRequestFilter does internally:
public final void doFilter(ServletRequest request, ...) {
    
    String attributeName = getAlreadyFilteredAttributeName();
    
    if (request.getAttribute(attributeName) != null) {
        // Already ran for this request — skip
        chain.doFilter(request, response);
        return;
    }
    
    // Mark as already ran
    request.setAttribute(attributeName, Boolean.TRUE);
    
    // Call YOUR implementation exactly once
    doFilterInternal((HttpServletRequest) request, ...);
}
```

You override `doFilterInternal()` — Spring handles the "once" guarantee.

---

## 5.3 The Complete Filter — Built Step by Step

Let us build the filter in layers so every line is clear.

### Layer 1: The skeleton

```java
// src/main/java/com/example/demo/security/JwtAuthenticationFilter.java

package com.example.demo.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        // Our logic goes here
        // At the end, ALWAYS call this to pass request to next filter:
        filterChain.doFilter(request, response);
    }
}
```

**Critical rule:** Always call `filterChain.doFilter(request, response)` at the end — unless you are intentionally stopping the request (like returning a 401 directly). If you forget this call, the request stops at your filter and the controller never executes.

---

### Layer 2: Extract the token from the header

```java
@Override
protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
) throws ServletException, IOException {

    // Step 1: Read the Authorization header
    final String authHeader = request.getHeader("Authorization");

    // Step 2: Validate header format
    // Must start with "Bearer " (note the space)
    if (authHeader == null || !authHeader.startsWith("Bearer ")) {
        // No token present — pass request along
        // AnonymousAuthenticationFilter will handle unauthenticated state
        filterChain.doFilter(request, response);
        return; // Important: stop executing THIS filter's logic
    }

    // Step 3: Extract token (remove "Bearer " prefix — 7 characters)
    final String jwt = authHeader.substring(7);

    // jwt now contains: eyJhbGci...header.payload.signature
    filterChain.doFilter(request, response);
}
```

**Why return after `filterChain.doFilter()` when there is no token?**

```
If there is no Authorization header:
  → This might be a public endpoint (/auth/login, /public/*)
  → We should not block it here
  → Pass it along — AuthorizationFilter will decide if it needs auth
  → If endpoint is public: allowed through
  → If endpoint needs auth: AuthorizationFilter rejects with 401
```

---

### Layer 3: Extract username and validate

```java
@Override
protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
) throws ServletException, IOException {

    final String authHeader = request.getHeader("Authorization");

    if (authHeader == null || !authHeader.startsWith("Bearer ")) {
        filterChain.doFilter(request, response);
        return;
    }

    final String jwt = authHeader.substring(7);
    
    // Step 4: Extract username from token
    // extractUsername() internally calls parseSignedClaims()
    // → verifies signature
    // → verifies not expired
    // → returns subject claim
    final String username = jwtService.extractUsername(jwt);

    // Step 5: Check if username was extracted AND user is not already authenticated
    //
    // Why check SecurityContextHolder?
    // If this request was already authenticated (e.g., by another filter),
    // we should NOT overwrite that authentication.
    // This prevents redundant DB lookups.
    if (username != null && 
        SecurityContextHolder.getContext().getAuthentication() == null) {
        
        // Proceed to load user and set authentication...
    }

    filterChain.doFilter(request, response);
}
```

---

### Layer 4: Load user, validate, set SecurityContext

```java
if (username != null && 
    SecurityContextHolder.getContext().getAuthentication() == null) {

    // Step 6: Load full user details from database
    // This gives us UserDetails with authorities, password, etc.
    UserDetails userDetails = userDetailsService.loadUserByUsername(username);

    // Step 7: Validate token against the loaded user
    // Checks: does token's username match? Is token expired?
    if (jwtService.validateToken(jwt, userDetails)) {

        // Step 8: Create Authentication object
        // UsernamePasswordAuthenticationToken is the standard implementation
        //
        // Constructor used here:
        //   arg1: principal   = UserDetails object
        //   arg2: credentials = null (we don't need password after auth)
        //   arg3: authorities = user's roles/permissions
        //
        // Using the 3-arg constructor sets authenticated = true automatically
        UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(
                        userDetails,           // principal
                        null,                  // credentials (null = already verified)
                        userDetails.getAuthorities() // authorities
                );

        // Step 9: Add request details to authentication
        // This attaches metadata like IP address, session ID
        // ExceptionTranslationFilter and audit logs can use this
        authToken.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request)
        );

        // Step 10: Store Authentication in SecurityContext
        // This is what makes Spring Security consider this request authenticated
        SecurityContextHolder.getContext().setAuthentication(authToken);
    }
}
```

**Why `UsernamePasswordAuthenticationToken` with 3 arguments?**

```java
// 2-argument constructor: authenticated = FALSE
// Used BEFORE verification (just holds credentials to pass to AuthenticationManager)
new UsernamePasswordAuthenticationToken(username, password)

// 3-argument constructor: authenticated = TRUE
// Used AFTER verification (we have confirmed the JWT is valid)
new UsernamePasswordAuthenticationToken(userDetails, null, authorities)
```

This distinction is important. If you use the 2-arg version, Spring Security treats the user as NOT authenticated even though you set it in the context.

---

## 5.4 The Complete Filter

```java
// src/main/java/com/example/demo/security/JwtAuthenticationFilter.java

package com.example.demo.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(JwtAuthenticationFilter.class);

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    // Constructor injection — preferred over @Autowired
    public JwtAuthenticationFilter(
            JwtService jwtService,
            UserDetailsService userDetailsService
    ) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        log.debug("=== JwtAuthenticationFilter triggered for: {} {} ===",
                request.getMethod(), request.getRequestURI());

        // ── Step 1: Extract Authorization header ──────────────────────
        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            log.debug("No Bearer token found. Passing request along.");
            filterChain.doFilter(request, response);
            return;
        }

        // ── Step 2: Extract JWT from header ───────────────────────────
        final String jwt = authHeader.substring(7);
        log.debug("JWT token extracted from header.");

        // ── Step 3: Extract username from JWT ─────────────────────────
        final String username;
        try {
            username = jwtService.extractUsername(jwt);
            log.debug("Username extracted from token: {}", username);
        } catch (Exception e) {
            // Token is malformed, tampered, or expired
            // Log and let the request continue unauthenticated
            // AuthorizationFilter will reject if endpoint needs auth
            log.warn("Failed to extract username from token: {}", e.getMessage());
            filterChain.doFilter(request, response);
            return;
        }

        // ── Step 4: Authenticate if not already done ──────────────────
        if (username != null &&
                SecurityContextHolder.getContext().getAuthentication() == null) {

            // Load user from database / in-memory store
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            log.debug("UserDetails loaded for: {}", username);

            // Validate token against loaded user
            if (jwtService.validateToken(jwt, userDetails)) {

                // Create Authentication token (3-arg = pre-authenticated)
                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                                userDetails,
                                null,
                                userDetails.getAuthorities()
                        );

                // Attach request metadata
                authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request)
                );

                // Set in SecurityContext — request is now authenticated
                SecurityContextHolder.getContext().setAuthentication(authToken);

                log.debug("Authentication set in SecurityContext for user: {}", username);

            } else {
                log.warn("Token validation failed for user: {}", username);
            }
        }

        // ── Step 5: Always continue the filter chain ───────────────────
        filterChain.doFilter(request, response);
    }
}
```

---

## 5.5 The Complete Request Flow With Our Filter

Now let us trace a real request through the entire system:

```
GET /api/orders
Authorization: Bearer eyJhbGci...

        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ SecurityContextHolderFilter                                │
│  → Creates empty SecurityContext for this thread           │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ JwtAuthenticationFilter  (OUR FILTER)                      │
│                                                            │
│  1. authHeader = "Bearer eyJhbGci..."                      │
│  2. jwt = "eyJhbGci..."                                    │
│  3. username = jwtService.extractUsername(jwt) → "john"    │
│  4. SecurityContext.getAuthentication() == null → proceed  │
│  5. userDetails = userDetailsService.loadByUsername("john")│
│  6. jwtService.validateToken(jwt, userDetails) → true      │
│  7. authToken = new UsernamePasswordAuthenticationToken(   │
│         userDetails, null, [ROLE_USER])                    │
│  8. SecurityContext.setAuthentication(authToken)           │
│  9. filterChain.doFilter() → continue                      │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ AnonymousAuthenticationFilter                              │
│  → SecurityContext already has Authentication              │
│  → Skips (does nothing)                                    │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ ExceptionTranslationFilter                                 │
│  → Wraps next filter in try/catch                          │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ AuthorizationFilter                                        │
│  → getAuthentication() → john with ROLE_USER               │
│  → /api/orders requires authentication → john IS auth ✓   │
│  → ALLOWS request through                                  │
└────────────────────────────────────────────────────────────┘
        │
        ▼
    OrderController.getOrders() → 200 OK
```

---

### What happens with an invalid token:

```
GET /api/orders
Authorization: Bearer TAMPERED_TOKEN

        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ JwtAuthenticationFilter                                    │
│  → extractUsername(TAMPERED_TOKEN)                         │
│  → throws SignatureException                               │
│  → caught in catch block                                   │
│  → logs warning                                            │
│  → filterChain.doFilter() → continue WITHOUT setting auth  │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ AnonymousAuthenticationFilter                              │
│  → SecurityContext has NO Authentication                   │
│  → Sets AnonymousAuthenticationToken                       │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ AuthorizationFilter                                        │
│  → getAuthentication() → AnonymousAuthenticationToken      │
│  → /api/orders requires authenticated user                 │
│  → Anonymous is NOT authenticated                          │
│  → throws AuthenticationException                          │
└────────────────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ ExceptionTranslationFilter catches AuthenticationException │
│  → Returns 401 Unauthorized                                │
└────────────────────────────────────────────────────────────┘
```

---

## 5.6 shouldNotFilter — Skip Filter for Certain URLs

Sometimes you want the JWT filter to completely skip public endpoints (like `/auth/login`). There is no point validating a JWT on a login endpoint — there is no token yet.

```java
// Add this method to JwtAuthenticationFilter:

@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    String path = request.getServletPath();
    
    // Skip JWT filter for these public paths
    return path.startsWith("/auth/") ||
           path.startsWith("/public/") ||
           path.equals("/actuator/health");
}
```

When `shouldNotFilter()` returns `true`, `OncePerRequestFilter` skips `doFilterInternal()` entirely for that request.

**Should you use this?**

It is optional but good practice. Without it, your filter runs on `/auth/login`, finds no token, and passes through cleanly anyway. But skipping it explicitly is:
- Slightly more efficient
- More explicit about intent
- Avoids unnecessary log noise

---

## Hands-On: Log When Filter Is Triggered

### Step 1: Minimal Security Config to register the filter

We need a `SecurityFilterChain` to register our filter. We will build the full configuration in Section 6. For now, use this minimal version:

```java
// src/main/java/com/example/demo/config/SecurityConfig.java

package com.example.demo.config;

import com.example.demo.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            
            // Register our filter BEFORE UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### Step 2: Add a test controller

```java
// src/main/java/com/example/demo/controller/ApiController.java

package com.example.demo.controller;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api")
public class ApiController {

    @GetMapping("/hello")
    public Map<String, String> hello() {
        // Read the authenticated user directly from SecurityContext
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        return Map.of(
            "message", "Hello, " + auth.getName() + "!",
            "authorities", auth.getAuthorities().toString()
        );
    }
}
```

### Step 3: Enable debug logging

```properties
# application.properties
app.jwt.secret=mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
app.jwt.expiration=3600000

logging.level.com.example.demo.security=DEBUG
logging.level.org.springframework.security=DEBUG
```

### Step 4: Generate a token to test with

Add a temporary test endpoint (remove in production):

```java
// src/main/java/com/example/demo/controller/AuthController.java

package com.example.demo.controller;

import com.example.demo.security.JwtService;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final JwtService jwtService;

    public AuthController(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    // TEMPORARY — returns a hardcoded token for testing
    // We build the real /login endpoint in Section 7
    @GetMapping("/token")
    public Map<String, String> getTestToken() {
        UserDetails testUser = User.builder()
                .username("john")
                .password("ignored")
                .roles("USER")
                .build();

        String token = jwtService.generateToken(testUser);
        return Map.of("token", token);
    }
}
```

### Step 5: Test the full flow

**Test 1: Get a token**
```bash
curl http://localhost:8080/auth/token
```
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwi..."
}
```

**Test 2: Call protected endpoint WITH token**
```bash
curl http://localhost:8080/api/hello \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwi..."
```
```json
{
  "message": "Hello, john!",
  "authorities": "[ROLE_USER]"
}
```

**Console logs you should see:**
```
=== JwtAuthenticationFilter triggered for: GET /api/hello ===
JWT token extracted from header.
Username extracted from token: john
UserDetails loaded for: john
Authentication set in SecurityContext for user: john
```

**Test 3: Call protected endpoint WITHOUT token**
```bash
curl http://localhost:8080/api/hello
```
```json
{
  "status": 401,
  "error": "Unauthorized"
}
```

**Console logs:**
```
=== JwtAuthenticationFilter triggered for: GET /api/hello ===
No Bearer token found. Passing request along.
```

**Test 4: Call protected endpoint WITH tampered token**
```bash
curl http://localhost:8080/api/hello \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.TAMPERED.signature"
```
```json
{
  "status": 401,
  "error": "Unauthorized"
}
```

**Console logs:**
```
=== JwtAuthenticationFilter triggered for: GET /api/hello ===
JWT token extracted from header.
Failed to extract username from token: JWT signature does not match...
```

---

## Section 5 — Interview Summary

| Question | Answer |
|----------|--------|
| Why extend `OncePerRequestFilter`? | Guarantees the filter runs exactly once per HTTP request, even with internal forwards |
| Why check `SecurityContextHolder.getContext().getAuthentication() == null`? | Avoid overwriting existing authentication and prevent redundant DB lookups |
| Why use the 3-argument `UsernamePasswordAuthenticationToken` constructor? | 3-arg constructor sets `authenticated = true`. 2-arg sets `authenticated = false` |
| Why set credentials to `null` in the authentication token? | Password is not needed after JWT validation — clearing it is a security best practice |
| What does `WebAuthenticationDetailsSource().buildDetails(request)` add? | Attaches request metadata (IP address, session ID) to the Authentication object |
| What happens if you forget `filterChain.doFilter()`? | The request stops at your filter — controller never executes, client hangs |
| What does `shouldNotFilter()` do? | Tells `OncePerRequestFilter` to skip `doFilterInternal()` for specific URL patterns |
| What status code does a tampered token produce and why? | 401 — filter passes unauthenticated, `AuthorizationFilter` rejects, `ExceptionTranslationFilter` maps to 401 |

---

## Quick Understanding Check

**Q1.** Trace exactly what happens — filter by filter — when this request hits your application:

```
GET /api/orders
Authorization: Bearer <valid-token-for-john>
```

But `john` has been deleted from the database since the token was issued. The JWT itself is valid (correct signature, not expired). What happens when `userDetailsService.loadUserByUsername("john")` is called?

**Q2.** A colleague writes this code in the filter:

```java
UsernamePasswordAuthenticationToken authToken =
    new UsernamePasswordAuthenticationToken(
        userDetails.getUsername(),  // just the string, not UserDetails
        null,
        userDetails.getAuthorities()
    );
SecurityContextHolder.getContext().setAuthentication(authToken);
```

This works but has a subtle problem. What is it, and why does it matter when you later call `SecurityContextHolder.getContext().getAuthentication().getPrincipal()` in a controller?

**Q3.** Why do we add `JwtAuthenticationFilter` BEFORE `UsernamePasswordAuthenticationFilter` specifically? What would happen if we added it AFTER `AuthorizationFilter` instead?
