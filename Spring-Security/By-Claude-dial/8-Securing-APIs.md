# Spring Security Masterclass — Step 8: Securing APIs

---

## 8.1 What This Section Covers

We have authentication working. Users can log in and get a JWT. Now we build the actual protected API surface — correctly.

This section covers four distinct problems:

```
Problem 1: Different endpoints need different access levels
           /api/users     → any authenticated user
           /api/admin/**  → only ADMIN role
           /api/own/**    → only the owner of the resource

Problem 2: Getting the current user cleanly inside a controller
           Without: SecurityContextHolder.getContext()... (verbose, untestable)
           With:    @AuthenticationPrincipal (clean, testable)

Problem 3: Method-level security
           URL rules cover coarse-grained access
           Method rules cover fine-grained business logic access

Problem 4: Returning correct responses
           200 with data     → authenticated, authorized
           401 Unauthorized  → not authenticated
           403 Forbidden     → authenticated but wrong role
```

---

## 8.2 Understanding 401 vs 403 — The Most Confused Status Codes

Most developers get these wrong. Get this right once and never confuse them again:

```
401 Unauthorized (misleading name — should be "Unauthenticated")
  Meaning: "I don't know who you are."
  Cause:   No token, expired token, invalid token
  Fix:     Log in and get a valid token

403 Forbidden
  Meaning: "I know who you are, but you cannot do this."
  Cause:   Valid token, but insufficient role/permission
  Fix:     Need a different account with higher privileges
```

```
Scenario A:
  Request: GET /api/admin/users (no Authorization header)
  Filter:  No token → no Authentication set
  Auth:    Anonymous user hits protected endpoint
  Result:  401 Unauthorized ← "Who are you?"

Scenario B:
  Request: GET /api/admin/users
  Header:  Authorization: Bearer <valid john token, ROLE_USER only>
  Filter:  Token valid → Authentication set for john with ROLE_USER
  Auth:    john authenticated, but /api/admin/** requires ROLE_ADMIN
  Result:  403 Forbidden ← "I know you're john, but you can't do this"
```

Spring Security handles this distinction automatically through `ExceptionTranslationFilter`:

```
AuthenticationException (not authenticated) → 401 → AuthenticationEntryPoint
AccessDeniedException   (not authorized)    → 403 → AccessDeniedHandler
```

---

## 8.3 URL-Level Security — requestMatchers Rules

### Building a realistic authorization ruleset:

```java
// SecurityConfig.java — updated authorizeHttpRequests section

.authorizeHttpRequests(auth -> auth

    // ── Public endpoints ───────────────────────────────────────────
    // No token required
    .requestMatchers("/auth/**").permitAll()
    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/actuator/health", "/actuator/info").permitAll()

    // ── Role-based endpoint protection ────────────────────────────
    // Only users with ROLE_ADMIN can access admin endpoints
    // Note: hasRole("ADMIN") automatically checks for "ROLE_ADMIN"
    //       hasAuthority("ROLE_ADMIN") checks the exact string
    .requestMatchers("/api/admin/**").hasRole("ADMIN")

    // ── Multiple roles allowed ─────────────────────────────────────
    // Either ADMIN or MODERATOR can access moderation endpoints
    .requestMatchers("/api/moderate/**").hasAnyRole("ADMIN", "MODERATOR")

    // ── HTTP method specific rules ─────────────────────────────────
    // GET requests to /api/products are public (anyone can view)
    // POST/PUT/DELETE require authentication (only logged-in users modify)
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
    .requestMatchers(HttpMethod.POST, "/api/products/**").authenticated()
    .requestMatchers(HttpMethod.PUT, "/api/products/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasRole("ADMIN")

    // ── Catch-all ──────────────────────────────────────────────────
    // Everything not matched above requires authentication
    .anyRequest().authenticated()
)
```

### `hasRole` vs `hasAuthority` — Critical Distinction:

```java
// Spring stores roles with "ROLE_" prefix internally
// When you call .roles("ADMIN") → stored as "ROLE_ADMIN"

.hasRole("ADMIN")
// → checks if authorities contain "ROLE_ADMIN"
// → Spring adds "ROLE_" prefix automatically
// → Use this when working with roles defined via .roles() or @PreAuthorize

.hasAuthority("ROLE_ADMIN")
// → checks if authorities contain exactly "ROLE_ADMIN"
// → No prefix added — you supply the full string
// → Use this for custom permissions like "READ_PRIVILEGE", "WRITE_PRIVILEGE"

// These are EQUIVALENT:
.hasRole("ADMIN")
.hasAuthority("ROLE_ADMIN")

// These are NOT equivalent:
.hasRole("ROLE_ADMIN")       // ❌ checks for "ROLE_ROLE_ADMIN" — bug!
.hasAuthority("ADMIN")       // ❌ checks for "ADMIN" (no prefix) — likely not found
```

---

## 8.4 `@AuthenticationPrincipal` — The Clean Way to Get Current User

### The wrong way — verbose and hard to test:

```java
// ❌ Wrong — couples your controller to Spring Security internals
@GetMapping("/api/profile")
public ResponseEntity<ProfileResponse> getProfile() {
    
    // Verbose, imports SecurityContextHolder everywhere
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    
    // Unsafe cast — could throw ClassCastException
    UserDetails userDetails = (UserDetails) auth.getPrincipal();
    
    String username = userDetails.getUsername();
    // ...
}
```

### The right way — `@AuthenticationPrincipal`:

```java
// ✅ Correct — Spring injects principal directly into method parameter
@GetMapping("/api/profile")
public ResponseEntity<ProfileResponse> getProfile(
        @AuthenticationPrincipal UserDetails userDetails
) {
    // userDetails is injected automatically — no casting, no SecurityContextHolder
    String username = userDetails.getUsername();
    // ...
}
```

**How does `@AuthenticationPrincipal` work internally?**

```
Spring MVC processes method parameters before calling the controller method.
When it sees @AuthenticationPrincipal:
  → calls SecurityContextHolder.getContext().getAuthentication()
  → calls authentication.getPrincipal()
  → casts to the parameter type (UserDetails in this case)
  → injects it as the method parameter

You get the same result but:
  ✓ No SecurityContextHolder calls in your code
  ✓ Easy to unit test (just pass a mock UserDetails)
  ✓ Clean, readable code
```

---

## 8.5 Building a Complete Protected API

### First, create a proper `User` entity

For `@AuthenticationPrincipal` to give us full user data (not just username), we need a custom `UserDetails` implementation:

```java
// src/main/java/com/example/demo/model/User.java

package com.example.demo.model;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

// Implements UserDetails so it can be used directly as the principal
// In Section 9, this becomes a JPA entity with @Entity
public class User implements UserDetails {

    private Long id;
    private String username;
    private String password;
    private String email;
    private String role;  // "ROLE_USER" or "ROLE_ADMIN"
    private boolean enabled;

    // ── Constructors ───────────────────────────────────────────────────
    public User() {}

    public User(Long id, String username, String password,
                String email, String role) {
        this.id = id;
        this.username = username;
        this.password = password;
        this.email = email;
        this.role = role;
        this.enabled = true;
    }

    // ── UserDetails interface implementation ───────────────────────────

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // Convert role string to GrantedAuthority
        // SimpleGrantedAuthority wraps a role string
        return List.of(new SimpleGrantedAuthority(role));
    }

    @Override
    public String getPassword() { return password; }

    @Override
    public String getUsername() { return username; }

    // Account status checks — all true for now
    // Section 9 (database) will make these dynamic
    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return enabled; }

    // ── Custom getters (not in UserDetails interface) ──────────────────
    // These are accessible when you @AuthenticationPrincipal User user

    public Long getId() { return id; }
    public String getEmail() { return email; }
    public String getRole() { return role; }

    // ── Setters ────────────────────────────────────────────────────────
    public void setId(Long id) { this.id = id; }
    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
    public void setEmail(String email) { this.email = email; }
    public void setRole(String role) { this.role = role; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
}
```

### Update `UserDetailsServiceImpl` to return our `User`:

```java
// src/main/java/com/example/demo/security/UserDetailsServiceImpl.java

package com.example.demo.security;

import com.example.demo.model.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(12);

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        // Hardcoded users — Section 9 replaces this with DB lookup
        return switch (username) {
            case "john" -> new User(
                1L, "john",
                encoder.encode("secret123"),
                "john@example.com",
                "ROLE_USER"
            );
            case "admin" -> new User(
                2L, "admin",
                encoder.encode("admin123"),
                "admin@example.com",
                "ROLE_ADMIN"
            );
            default -> throw new UsernameNotFoundException(
                "User not found: " + username
            );
        };
    }
}
```

---

### Build the complete API controller:

```java
// src/main/java/com/example/demo/controller/ApiController.java

package com.example.demo.controller;

import com.example.demo.model.User;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class ApiController {

    // ── Endpoint 1: Current user's own profile ─────────────────────────
    // Any authenticated user can see their own profile
    @GetMapping("/profile")
    public ResponseEntity<Map<String, Object>> getProfile(
            @AuthenticationPrincipal User currentUser
    ) {
        // Spring injects the authenticated User directly
        // We can access ALL User fields — not just username
        return ResponseEntity.ok(Map.of(
            "id",          currentUser.getId(),
            "username",    currentUser.getUsername(),
            "email",       currentUser.getEmail(),
            "role",        currentUser.getRole(),
            "authorities", currentUser.getAuthorities().toString()
        ));
    }

    // ── Endpoint 2: Public data ────────────────────────────────────────
    // permitAll in SecurityConfig — no @AuthenticationPrincipal needed
    @GetMapping("/public/products")
    public ResponseEntity<List<Map<String, Object>>> getProducts() {
        List<Map<String, Object>> products = List.of(
            Map.of("id", 1, "name", "Widget A", "price", 9.99),
            Map.of("id", 2, "name", "Widget B", "price", 19.99)
        );
        return ResponseEntity.ok(products);
    }

    // ── Endpoint 3: Admin only — URL level security ────────────────────
    // SecurityConfig: .requestMatchers("/api/admin/**").hasRole("ADMIN")
    @GetMapping("/admin/users")
    public ResponseEntity<List<Map<String, Object>>> getAllUsers(
            @AuthenticationPrincipal User currentUser
    ) {
        // We know currentUser is ADMIN — enforced by SecurityConfig
        List<Map<String, Object>> users = List.of(
            Map.of("id", 1, "username", "john", "role", "ROLE_USER"),
            Map.of("id", 2, "username", "admin", "role", "ROLE_ADMIN")
        );
        return ResponseEntity.ok(users);
    }

    // ── Endpoint 4: Owner-only access — method level security ──────────
    // Any user can try this endpoint, but they can only access their own data
    // @PreAuthorize handles the ownership check
    @GetMapping("/users/{userId}/data")
    @PreAuthorize("#userId == #currentUser.id or hasRole('ADMIN')")
    public ResponseEntity<Map<String, Object>> getUserData(
            @PathVariable Long userId,
            @AuthenticationPrincipal User currentUser
    ) {
        // Reached only if: userId matches currentUser.id OR user is ADMIN
        return ResponseEntity.ok(Map.of(
            "userId",    userId,
            "data",      "Private data for user " + userId,
            "accessedBy", currentUser.getUsername()
        ));
    }

    // ── Endpoint 5: Echo who is making the request ─────────────────────
    @GetMapping("/whoami")
    public ResponseEntity<Map<String, Object>> whoAmI(
            @AuthenticationPrincipal User currentUser
    ) {
        return ResponseEntity.ok(Map.of(
            "username",        currentUser.getUsername(),
            "isAdmin",         currentUser.getRole().equals("ROLE_ADMIN"),
            "isAuthenticated", true
        ));
    }
}
```

---

## 8.6 Method-Level Security — `@PreAuthorize`

URL rules in `SecurityConfig` are coarse-grained. `@PreAuthorize` gives you fine-grained control directly on the method.

### Enable it in SecurityConfig:

```java
// Already added in Section 6 — verify it is there
@EnableMethodSecurity   // enables @PreAuthorize, @PostAuthorize, @Secured
public class SecurityConfig { ... }
```

### `@PreAuthorize` expressions:

```java
// ── Simple role check ──────────────────────────────────────────────────
@PreAuthorize("hasRole('ADMIN')")
public void adminOnlyMethod() { ... }

// ── Multiple roles ─────────────────────────────────────────────────────
@PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
public void moderationMethod() { ... }

// ── Ownership check using method parameters ────────────────────────────
// #userId refers to the method parameter named userId
// #currentUser refers to the method parameter named currentUser
@PreAuthorize("#userId == #currentUser.id or hasRole('ADMIN')")
public void getUserData(Long userId, 
                        @AuthenticationPrincipal User currentUser) { ... }

// ── Access Authentication object directly ──────────────────────────────
// principal refers to the Authentication.getPrincipal()
@PreAuthorize("principal.username == #username")
public void getByUsername(String username) { ... }

// ── Complex business rule ──────────────────────────────────────────────
@PreAuthorize("hasRole('USER') and #amount <= 1000 or hasRole('ADMIN')")
public void transfer(BigDecimal amount) { ... }

// ── Using a custom method from a Spring bean ───────────────────────────
// Calls permissionService.canEdit(currentUser, resourceId) at runtime
@PreAuthorize("@permissionService.canEdit(principal, #resourceId)")
public void editResource(Long resourceId) { ... }
```

### `@PostAuthorize` — check AFTER method runs:

```java
// Run the method, then check if result is accessible by the caller
// If check fails → 403 is returned AFTER method executed
// Use carefully — method side effects already happened

@PostAuthorize("returnObject.username == principal.username or hasRole('ADMIN')")
public User findUser(Long id) {
    return userRepository.findById(id).orElseThrow();
    // Result returned only if caller is the user themselves or ADMIN
}
```

### Where to use URL rules vs `@PreAuthorize`:

```
URL Rules (SecurityConfig)          @PreAuthorize (method level)
───────────────────────────────     ──────────────────────────────────────
Coarse-grained access control       Fine-grained access control
Entire URL pattern                  Individual method
Role-based                          Role + data-based
Applied before filter chain ends    Applied when method is called
Easier to audit (all in one place)  More expressive (SpEL expressions)
Good for: /admin/**, /public/**     Good for: ownership, dynamic rules
```

**Best practice:** Use URL rules for broad role-based access. Use `@PreAuthorize` for business-logic-driven access rules.

---

## 8.7 Complete SecurityConfig With All Rules

```java
// src/main/java/com/example/demo/config/SecurityConfig.java

package com.example.demo.config;

import com.example.demo.security.JwtAuthenticationEntryPoint;
import com.example.demo.security.JwtAccessDeniedHandler;
import com.example.demo.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;
    private final JwtAuthenticationEntryPoint authEntryPoint;
    private final JwtAccessDeniedHandler accessDeniedHandler;

    public SecurityConfig(
            JwtAuthenticationFilter jwtAuthFilter,
            AuthenticationProvider authenticationProvider,
            JwtAuthenticationEntryPoint authEntryPoint,
            JwtAccessDeniedHandler accessDeniedHandler
    ) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.authenticationProvider = authenticationProvider;
        this.authEntryPoint = authEntryPoint;
        this.accessDeniedHandler = accessDeniedHandler;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http
            .csrf(AbstractHttpConfigurer::disable)

            .authorizeHttpRequests(auth -> auth

                // Public — no authentication needed
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()

                // HTTP method specific
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()

                // Role-based
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/moderate/**").hasAnyRole("ADMIN", "MODERATOR")

                // Catch-all
                .anyRequest().authenticated()
            )

            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )

            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            .authenticationProvider(authenticationProvider)

            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## Hands-On: Test All Scenarios

### Setup: Get tokens for both users

```bash
# Get john's token (ROLE_USER)
JOHN_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john","password":"secret123"}' \
  | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

# Get admin's token (ROLE_ADMIN)
ADMIN_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' \
  | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

echo "John's token acquired: ${JOHN_TOKEN:0:20}..."
echo "Admin's token acquired: ${ADMIN_TOKEN:0:20}..."
```

---

### Test 1: Profile endpoint — authenticated user

```bash
curl -s http://localhost:8080/api/profile \
  -H "Authorization: Bearer $JOHN_TOKEN" | python3 -m json.tool
```

```json
{
  "id": 1,
  "username": "john",
  "email": "john@example.com",
  "role": "ROLE_USER",
  "authorities": "[ROLE_USER]"
}
```
✅ 200 — `@AuthenticationPrincipal` gave us full User object

---

### Test 2: Admin endpoint — with user token (should get 403)

```bash
curl -s http://localhost:8080/api/admin/users \
  -H "Authorization: Bearer $JOHN_TOKEN" | python3 -m json.tool
```

```json
{
  "status": 403,
  "error": "Forbidden",
  "message": "You do not have permission to access this resource",
  "path": "/api/admin/users"
}
```
✅ 403 — john is authenticated but lacks ROLE_ADMIN

---

### Test 3: Admin endpoint — with admin token (should succeed)

```bash
curl -s http://localhost:8080/api/admin/users \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

```json
[
  { "id": 1, "username": "john", "role": "ROLE_USER" },
  { "id": 2, "username": "admin", "role": "ROLE_ADMIN" }
]
```
✅ 200 — admin has ROLE_ADMIN, access granted

---

### Test 4: Ownership check — user accessing own data

```bash
# john (id=1) accessing userId=1 → should succeed
curl -s http://localhost:8080/api/users/1/data \
  -H "Authorization: Bearer $JOHN_TOKEN" | python3 -m json.tool
```

```json
{
  "userId": 1,
  "data": "Private data for user 1",
  "accessedBy": "john"
}
```
✅ 200 — john accessing his own data, `@PreAuthorize` allows it

---

### Test 5: Ownership check — user accessing another's data

```bash
# john (id=1) trying to access userId=2 → should fail with 403
curl -s http://localhost:8080/api/users/2/data \
  -H "Authorization: Bearer $JOHN_TOKEN" | python3 -m json.tool
```

```json
{
  "status": 403,
  "error": "Forbidden",
  "message": "You do not have permission to access this resource",
  "path": "/api/users/2/data"
}
```
✅ 403 — `@PreAuthorize("#userId == #currentUser.id or hasRole('ADMIN')")` failed for john trying to access userId=2

---

### Test 6: Admin accessing another user's data

```bash
# admin accessing userId=1 → should succeed (ADMIN override)
curl -s http://localhost:8080/api/users/1/data \
  -H "Authorization: Bearer $ADMIN_TOKEN" | python3 -m json.tool
```

```json
{
  "userId": 1,
  "data": "Private data for user 1",
  "accessedBy": "admin"
}
```
✅ 200 — admin has `hasRole('ADMIN')` which satisfies the `or` condition

---

### Test 7: No token on protected endpoint

```bash
curl -s http://localhost:8080/api/profile | python3 -m json.tool
```

```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Full authentication is required to access this resource",
  "path": "/api/profile"
}
```
✅ 401 — no token means unauthenticated

---

### Complete Test Summary Table

| Endpoint | User | Expected | Reason |
|----------|------|----------|--------|
| `GET /api/profile` | john | 200 | Authenticated user |
| `GET /api/profile` | no token | 401 | Not authenticated |
| `GET /api/admin/users` | john | 403 | Missing ROLE_ADMIN |
| `GET /api/admin/users` | admin | 200 | Has ROLE_ADMIN |
| `GET /api/users/1/data` | john (id=1) | 200 | Owner access |
| `GET /api/users/2/data` | john (id=1) | 403 | Not owner, not admin |
| `GET /api/users/1/data` | admin | 200 | Admin override |
| `GET /public/products` | no token | 200 | Public endpoint |

---

## 8.8 Accessing Current User in Service Layer

Sometimes you need the current user inside a service, not just a controller. Do NOT pass `UserDetails` as a method parameter down through every service. Use `SecurityContextHolder` in the service:

```java
// src/main/java/com/example/demo/service/SecurityUtils.java

package com.example.demo.service;

import com.example.demo.model.User;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

@Component
public class SecurityUtils {

    /**
     * Get the currently authenticated User.
     * Throws IllegalStateException if no user is authenticated.
     * Call this only from secured endpoints.
     */
    public User getCurrentUser() {
        Authentication authentication = 
            SecurityContextHolder.getContext().getAuthentication();

        if (authentication == null || !authentication.isAuthenticated()) {
            throw new IllegalStateException("No authenticated user found");
        }

        return (User) authentication.getPrincipal();
    }

    /**
     * Get current username without casting to User.
     * Safe to call even from partially secured contexts.
     */
    public String getCurrentUsername() {
        Authentication authentication = 
            SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication == null) {
            return "anonymous";
        }
        
        return authentication.getName();
    }

    /**
     * Check if the current user has a specific role.
     */
    public boolean currentUserHasRole(String role) {
        return getCurrentUser()
            .getAuthorities()
            .stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
    }
}
```

```java
// Usage in any service:
@Service
public class OrderService {

    private final SecurityUtils securityUtils;

    public OrderService(SecurityUtils securityUtils) {
        this.securityUtils = securityUtils;
    }

    public List<Order> getMyOrders() {
        User currentUser = securityUtils.getCurrentUser();
        // fetch orders for currentUser.getId()
        return orderRepository.findByUserId(currentUser.getId());
    }
}
```

**Why is `SecurityContextHolder` safe to use in services?**

```
Spring Security stores Authentication in a ThreadLocal by default.
ThreadLocal = each thread has its own isolated copy.
Your HTTP request is processed on one thread from start to finish.
The Authentication set by JwtAuthenticationFilter is visible
to every class called on that same thread — including services.
No need to pass Authentication as a parameter.
```

---

## Section 8 — Interview Summary

| Question | Answer |
|----------|--------|
| What is the difference between 401 and 403? | 401 = not authenticated (who are you?). 403 = authenticated but not authorized (you can't do this) |
| What does `@AuthenticationPrincipal` do? | Injects the current authenticated principal into the controller method parameter automatically |
| What is the difference between `hasRole()` and `hasAuthority()`? | `hasRole("ADMIN")` checks for "ROLE_ADMIN" (prefix added). `hasAuthority("ROLE_ADMIN")` checks exact string |
| What does `@EnableMethodSecurity` enable? | Enables `@PreAuthorize`, `@PostAuthorize`, `@Secured` on methods |
| What does `@PreAuthorize("#userId == #currentUser.id")` mean? | Method parameter `userId` must equal `currentUser.id` — `#` references method parameters by name |
| When would you use `@PostAuthorize`? | When you need to check the return value against the current user — runs AFTER method executes |
| Why is `SecurityContextHolder` safe to use in services? | Spring Security uses `ThreadLocal` storage — Authentication is available throughout the entire request thread |
| What rule ordering mistake causes `anyRequest()` to block public endpoints? | Putting `anyRequest().authenticated()` before specific `permitAll()` rules — first match wins |

---

## Quick Understanding Check

**Q1.** You have this method:

```java
@GetMapping("/api/orders/{orderId}")
@PreAuthorize("#orderId == #currentUser.id or hasRole('ADMIN')")
public Order getOrder(
    @PathVariable Long orderId,
    @AuthenticationPrincipal User currentUser
) {
    return orderService.findById(orderId);
}
```

A request comes in: `GET /api/orders/5` with a valid JWT for john (id=1, ROLE_USER).

Walk through exactly:
- What value does `#orderId` have?
- What value does `#currentUser.id` have?
- What does the `@PreAuthorize` expression evaluate to?
- What does the client receive?

**Q2.** You discover that `@PreAuthorize` is silently not working — all requests pass through regardless of roles. What is the most likely cause and how do you fix it?

**Q3.** A developer argues: *"We don't need `@PreAuthorize` for ownership checks. We can just load the resource in the service and check if the current user owns it, then throw a 403 manually."*

Compare the two approaches. When is `@PreAuthorize` better? When is the manual check better? Give one concrete example where manual checking is the only option.
