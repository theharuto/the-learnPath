# Spring Security Masterclass — Step 6: Spring Security Configuration

---

## 6.1 Why This Section is Critical

Everything we built so far — `JwtService`, `JwtAuthenticationFilter`, `UserDetailsService` — are individual pieces. They do nothing useful until they are wired together correctly inside a `SecurityFilterChain`.

This section is where most developers make mistakes:
- Wrong filter order
- CSRF not properly disabled
- Sessions not stateless
- `AuthenticationManager` not exposed as a bean
- Circular dependency issues

We will build this correctly, understanding every decision.

---

## 6.2 The Old Way vs The New Way

First, understand why you must never use the old API:

```java
// ❌ OLD WAY — Spring Security 5, WebSecurityConfigurerAdapter
// REMOVED in Spring Security 6. Do not use. Do not follow tutorials that show this.
@Configuration
public class OldSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()      // deprecated
            .antMatchers("/public/**") // deprecated
            .permitAll();
    }
}

// ✅ NEW WAY — Spring Security 6, Component-based
// SecurityFilterChain bean. No inheritance. Pure composition.
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // configure here
        return http.build();
    }
}
```

**Why was `WebSecurityConfigurerAdapter` removed?**

It forced you to inherit from a framework class — violating composition-over-inheritance principle. The new approach uses beans. You define what you need as `@Bean` and Spring wires them together. Much cleaner.

---

## 6.3 Every Bean We Need — And Why

Before writing the config class, understand each bean we need:

```
┌─────────────────────────────────────────────────────────────┐
│                    Beans Required                            │
├─────────────────────────┬───────────────────────────────────┤
│ SecurityFilterChain     │ Defines the filter chain rules     │
│ AuthenticationProvider  │ Knows HOW to authenticate users    │
│ AuthenticationManager   │ Delegates to AuthenticationProvider│
│ PasswordEncoder         │ BCrypt hashing for passwords       │
│ UserDetailsService      │ Loads user from DB/memory          │
└─────────────────────────┴───────────────────────────────────┘
```

### Relationships between these beans:

```
SecurityFilterChain
    │ uses
    ▼
JwtAuthenticationFilter
    │ uses
    ├──────────────────→ JwtService
    └──────────────────→ UserDetailsService
                              │
                              ▼ (used by)
                        AuthenticationProvider
                              │ uses
                              ├──→ UserDetailsService
                              └──→ PasswordEncoder

AuthenticationManager
    │ delegates to
    ▼
AuthenticationProvider
```

---

## 6.4 The ApplicationConfig — Supporting Beans

We separate infrastructure beans from security filter configuration. This avoids circular dependencies and keeps code clean.

```java
// src/main/java/com/example/demo/config/ApplicationConfig.java

package com.example.demo.config;

import com.example.demo.security.UserDetailsServiceImpl;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class ApplicationConfig {

    private final UserDetailsServiceImpl userDetailsServiceImpl;

    public ApplicationConfig(UserDetailsServiceImpl userDetailsServiceImpl) {
        this.userDetailsServiceImpl = userDetailsServiceImpl;
    }

    // ── Bean 1: UserDetailsService ────────────────────────────────────
    // Expose as interface type so Spring Security can find it by type
    @Bean
    public UserDetailsService userDetailsService() {
        return userDetailsServiceImpl;
    }

    // ── Bean 2: PasswordEncoder ───────────────────────────────────────
    // BCrypt is the industry standard for password hashing
    // Strength factor 12 = 2^12 hash iterations (good balance of security/speed)
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    // ── Bean 3: AuthenticationProvider ───────────────────────────────
    // DaoAuthenticationProvider = authenticate via UserDetailsService + PasswordEncoder
    // "Dao" = Data Access Object — it accesses user data via UserDetailsService
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();

        // Tell it WHERE to load users from
        provider.setUserDetailsService(userDetailsServiceImpl);

        // Tell it HOW to verify passwords
        provider.setPasswordEncoder(passwordEncoder());

        return provider;
    }

    // ── Bean 4: AuthenticationManager ────────────────────────────────
    // The /auth/login controller will call this to authenticate username+password
    // We get it from Spring's AuthenticationConfiguration (auto-configured)
    // rather than building it manually — this picks up our AuthenticationProvider
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### Why `DaoAuthenticationProvider`?

```
DaoAuthenticationProvider.authenticate(usernamePasswordToken):
    │
    ├── calls userDetailsService.loadUserByUsername(username)
    │         → gets UserDetails from DB
    │
    ├── calls passwordEncoder.matches(rawPassword, encodedPassword)
    │         → verifies BCrypt hash
    │
    ├── if match → returns authenticated UsernamePasswordAuthenticationToken
    └── if no match → throws BadCredentialsException
```

This is exactly what we need for the `/auth/login` endpoint in Section 7.

### Why expose `AuthenticationManager` as a bean?

Spring Security auto-creates an `AuthenticationManager` internally, but does not expose it as a bean by default. Our `AuthController` (Section 7) needs to inject it to authenticate login requests. Exposing it via `AuthenticationConfiguration.getAuthenticationManager()` is the Spring Security 6 standard approach.

---

## 6.5 The SecurityConfig — The Filter Chain

Now the main configuration:

```java
// src/main/java/com/example/demo/config/SecurityConfig.java

package com.example.demo.config;

import com.example.demo.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
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
@EnableMethodSecurity  // Enables @PreAuthorize, @PostAuthorize (needed for Section 10)
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;

    public SecurityConfig(
            JwtAuthenticationFilter jwtAuthFilter,
            AuthenticationProvider authenticationProvider
    ) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.authenticationProvider = authenticationProvider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http
            // ── 1. CSRF Configuration ─────────────────────────────────────
            // Disable CSRF protection for REST APIs
            //
            // WHY: CSRF attacks exploit browser cookie auto-sending behavior.
            // Our API uses JWT in Authorization header — browsers cannot
            // automatically add custom headers cross-origin.
            // Therefore CSRF protection is unnecessary and would break our API.
            .csrf(AbstractHttpConfigurer::disable)

            // ── 2. Authorization Rules ────────────────────────────────────
            // Define which endpoints need authentication and which are public
            .authorizeHttpRequests(auth -> auth

                // Public endpoints — no token required
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/public/**").permitAll()

                // Actuator health check — public (useful for load balancers)
                .requestMatchers("/actuator/health").permitAll()

                // Everything else requires authentication
                .anyRequest().authenticated()
            )

            // ── 3. Session Management ─────────────────────────────────────
            // STATELESS = Spring Security will NEVER create or use HTTP sessions
            //
            // WHY: JWT is stateless by design. If we allow sessions,
            // Spring might cache authentication in session and bypass
            // our JWT filter on subsequent requests. We want JWT
            // verified on EVERY request.
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // ── 4. Authentication Provider ────────────────────────────────
            // Register our DaoAuthenticationProvider
            // Spring Security uses this when AuthenticationManager.authenticate() is called
            .authenticationProvider(authenticationProvider)

            // ── 5. Custom Filter Registration ────────────────────────────
            // Add our JWT filter BEFORE UsernamePasswordAuthenticationFilter
            //
            // WHY before UsernamePasswordAuthenticationFilter?
            // That filter handles form-based login (username+password in request body).
            // We want JWT validation to happen BEFORE Spring tries to do form login.
            // If JWT is valid, we set auth in context.
            // UsernamePasswordAuthenticationFilter then sees auth already set and skips.
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## 6.6 Deep Dive: Every Configuration Decision

### Decision 1: `csrf(AbstractHttpConfigurer::disable)`

```java
// Verbose lambda version (same thing):
.csrf(csrf -> csrf.disable())

// Method reference version (cleaner):
.csrf(AbstractHttpConfigurer::disable)
```

**When should you NOT disable CSRF?**

```
API uses JWT in Authorization header → DISABLE CSRF (our case)
API uses cookies for authentication  → KEEP CSRF ENABLED
Application serves HTML forms        → KEEP CSRF ENABLED
```

### Decision 2: Order of `requestMatchers` rules

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/auth/**").permitAll()    // Rule 1
    .requestMatchers("/public/**").permitAll()  // Rule 2
    .anyRequest().authenticated()               // Rule 3 — catch-all
)
```

**Rules are evaluated in ORDER. First match wins.**

```
Request: GET /auth/login
  → Matches Rule 1 (/auth/**) → permitAll → ALLOWED
  → Rules 2 and 3 never checked

Request: GET /api/orders
  → Does not match Rule 1
  → Does not match Rule 2
  → Matches Rule 3 (anyRequest) → authenticated → checked
```

**⚠️ Common mistake:** Putting `anyRequest().authenticated()` before specific rules. Everything matches `anyRequest`, so your specific rules never execute.

```java
// ❌ WRONG — anyRequest catches everything first
.authorizeHttpRequests(auth -> auth
    .anyRequest().authenticated()          // catches /auth/login too!
    .requestMatchers("/auth/**").permitAll() // never reached
)

// ✅ CORRECT — specific rules first, catch-all last
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/auth/**").permitAll()
    .anyRequest().authenticated()
)
```

### Decision 3: `SessionCreationPolicy.STATELESS`

```java
.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
)
```

Four possible policies explained:

```
ALWAYS      → Always create a session (bad for APIs)
IF_REQUIRED → Create session if needed (Spring Security default — bad for JWT)
NEVER       → Never CREATE a session, but USE one if it exists (partial stateless)
STATELESS   → Never create OR use sessions (true stateless — what we want)
```

With `STATELESS`:
- No `JSESSIONID` cookie ever created
- No `HttpSession` ever consulted
- Every request must be independently authenticated via JWT
- Perfect horizontal scaling — no sticky sessions needed

### Decision 4: Filter position

```java
.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
```

Why specifically before `UsernamePasswordAuthenticationFilter`?

```
Filter Chain positions (relevant ones):

... → LogoutFilter → [OUR JWT FILTER] → UsernamePasswordAuthenticationFilter → ...

If JWT is valid:
  → Our filter sets Authentication in SecurityContext
  → UsernamePasswordAuthenticationFilter sees auth already set
  → Skips (it only processes form login requests anyway)
  → AnonymousAuthenticationFilter sees auth set → skips
  → AuthorizationFilter sees auth → allows request

If JWT is missing/invalid:
  → Our filter does NOT set auth
  → UsernamePasswordAuthenticationFilter looks for username/password in request body
  → Not found (it is an API call) → skips
  → AnonymousAuthenticationFilter sets anonymous auth
  → AuthorizationFilter → anonymous user hits protected endpoint → 401
```

---

## 6.7 Handling 401 and 403 Responses Correctly

By default, Spring Security returns an HTML error page for 401/403. For REST APIs, we want JSON.

### Custom AuthenticationEntryPoint (handles 401):

```java
// src/main/java/com/example/demo/security/JwtAuthenticationEntryPoint.java

package com.example.demo.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Map;

@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException
    ) throws IOException {

        // Set response as JSON
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401

        // Write JSON error body
        Map<String, Object> body = Map.of(
            "status", 401,
            "error", "Unauthorized",
            "message", authException.getMessage(),
            "path", request.getServletPath()
        );

        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

### Custom AccessDeniedHandler (handles 403):

```java
// src/main/java/com/example/demo/security/JwtAccessDeniedHandler.java

package com.example.demo.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.MediaType;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Map;

@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException
    ) throws IOException {

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_FORBIDDEN); // 403

        Map<String, Object> body = Map.of(
            "status", 403,
            "error", "Forbidden",
            "message", "You do not have permission to access this resource",
            "path", request.getServletPath()
        );

        objectMapper.writeValue(response.getOutputStream(), body);
    }
}
```

### Wire them into SecurityConfig:

```java
// Updated SecurityConfig with exception handling:

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;
    private final JwtAuthenticationEntryPoint authenticationEntryPoint;
    private final JwtAccessDeniedHandler accessDeniedHandler;

    public SecurityConfig(
            JwtAuthenticationFilter jwtAuthFilter,
            AuthenticationProvider authenticationProvider,
            JwtAuthenticationEntryPoint authenticationEntryPoint,
            JwtAccessDeniedHandler accessDeniedHandler
    ) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.authenticationProvider = authenticationProvider;
        this.authenticationEntryPoint = authenticationEntryPoint;
        this.accessDeniedHandler = accessDeniedHandler;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http
            .csrf(AbstractHttpConfigurer::disable)

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )

            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // ── Exception Handling ─────────────────────────────────────
            .exceptionHandling(ex -> ex
                // 401 — not authenticated
                .authenticationEntryPoint(authenticationEntryPoint)
                // 403 — authenticated but not authorized
                .accessDeniedHandler(accessDeniedHandler)
            )

            .authenticationProvider(authenticationProvider)

            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## 6.8 Circular Dependency — A Real Pitfall

**Problem that many developers hit:**

```
SecurityConfig
    depends on → JwtAuthenticationFilter
                    depends on → UserDetailsService

Spring Security auto-configuration
    also depends on → UserDetailsService
                         depends on → PasswordEncoder
                                         depends on → SecurityConfig (sometimes)

Result: Circular dependency → ApplicationContext fails to start
```

**How to avoid it:**

```java
// ❌ PROBLEMATIC: Injecting UserDetailsService directly into SecurityConfig
@Configuration
public class SecurityConfig {
    @Autowired
    private UserDetailsService userDetailsService; // can cause circular dep
}

// ✅ SAFE: Keep SecurityConfig thin — only inject what it directly needs
// Move UserDetailsService, PasswordEncoder, AuthenticationProvider
// into a SEPARATE ApplicationConfig class (what we did in 6.4)
// SecurityConfig only needs: JwtAuthenticationFilter, AuthenticationProvider
```

**The separation pattern:**

```
ApplicationConfig.java  → UserDetailsService, PasswordEncoder,
                          AuthenticationProvider, AuthenticationManager
                          (no dependency on SecurityConfig)

SecurityConfig.java     → SecurityFilterChain
                          (depends on ApplicationConfig beans — safe, one direction)

JwtAuthenticationFilter → JwtService, UserDetailsService
                          (no dependency on SecurityConfig)
```

One-directional dependency graph = no circular dependency.

---

## 6.9 Complete Project Structure So Far

```
src/main/java/com/example/demo/
│
├── config/
│   ├── ApplicationConfig.java      ← PasswordEncoder, AuthProvider, AuthManager
│   └── SecurityConfig.java         ← SecurityFilterChain
│
├── security/
│   ├── JwtService.java             ← Token generation and validation
│   ├── JwtAuthenticationFilter.java← Our custom filter
│   ├── JwtAuthenticationEntryPoint.java ← 401 JSON response
│   ├── JwtAccessDeniedHandler.java ← 403 JSON response
│   └── UserDetailsServiceImpl.java ← Load user by username
│
└── controller/
    ├── AuthController.java         ← /auth/** endpoints (public)
    └── ApiController.java          ← /api/** endpoints (protected)
```

---

## Hands-On: Configure Public vs Protected Endpoints

### Step 1: Add these test endpoints

```java
// src/main/java/com/example/demo/controller/ApiController.java

package com.example.demo.controller;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class ApiController {

    // PUBLIC — no token required
    @GetMapping("/public/info")
    public Map<String, String> publicInfo() {
        return Map.of(
            "message", "This is public — no token needed",
            "status", "open"
        );
    }

    // PROTECTED — valid JWT required
    @GetMapping("/api/profile")
    public Map<String, Object> profile() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return Map.of(
            "username", auth.getName(),
            "authorities", auth.getAuthorities().toString(),
            "isAuthenticated", auth.isAuthenticated()
        );
    }

    // PROTECTED — valid JWT required
    @GetMapping("/api/data")
    public Map<String, String> data() {
        return Map.of(
            "secret", "This is protected data",
            "timestamp", String.valueOf(System.currentTimeMillis())
        );
    }
}
```

### Step 2: Test all scenarios

**Scenario 1: Public endpoint — no token**
```bash
curl http://localhost:8080/public/info
```
```json
{
  "message": "This is public — no token needed",
  "status": "open"
}
```
✅ 200 OK — no token needed

---

**Scenario 2: Protected endpoint — no token**
```bash
curl http://localhost:8080/api/profile
```
```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Full authentication is required to access this resource",
  "path": "/api/profile"
}
```
✅ 401 with clean JSON — our `JwtAuthenticationEntryPoint` responded

---

**Scenario 3: Protected endpoint — with valid token**

First get a token (using our temp endpoint from Section 5):
```bash
TOKEN=$(curl -s http://localhost:8080/auth/token | grep -o '"token":"[^"]*"' | cut -d'"' -f4)
```

Then use it:
```bash
curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer $TOKEN"
```
```json
{
  "username": "john",
  "authorities": "[ROLE_USER]",
  "isAuthenticated": true
}
```
✅ 200 OK — authenticated and responded with user info

---

**Scenario 4: Expired or tampered token**
```bash
curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.INVALID.signature"
```
```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Full authentication is required to access this resource",
  "path": "/api/profile"
}
```
✅ 401 — filter caught the bad token, passed unauthenticated, entry point responded

---

## Section 6 — Interview Summary

| Question | Answer |
|----------|--------|
| Why was `WebSecurityConfigurerAdapter` removed in Spring Security 6? | Violated composition-over-inheritance. Replaced by `SecurityFilterChain` bean approach |
| Why disable CSRF for JWT APIs? | JWT in `Authorization` header cannot be auto-sent by browsers cross-origin. CSRF threat does not apply |
| What does `SessionCreationPolicy.STATELESS` do? | Prevents Spring Security from ever creating or using `HttpSession`. Every request must carry JWT |
| What is `DaoAuthenticationProvider`? | Authenticates users by loading from `UserDetailsService` and verifying password with `PasswordEncoder` |
| Why is rule order important in `authorizeHttpRequests`? | Rules are matched top-down. First match wins. `anyRequest()` must always be last |
| How do you expose `AuthenticationManager` as a bean in Spring Security 6? | Inject `AuthenticationConfiguration` and call `config.getAuthenticationManager()` |
| What handles 401 responses in Spring Security? | `AuthenticationEntryPoint` — triggered by `ExceptionTranslationFilter` on `AuthenticationException` |
| What handles 403 responses in Spring Security? | `AccessDeniedHandler` — triggered by `ExceptionTranslationFilter` on `AccessDeniedException` |
| How do you avoid circular dependency in security config? | Separate infrastructure beans (`ApplicationConfig`) from filter chain config (`SecurityConfig`) |

---

## Quick Understanding Check

**Q1.** You have this configuration:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/**").authenticated()
    .requestMatchers("/auth/**").permitAll()
    .anyRequest().permitAll()
)
```

A request comes in: `GET /api/admin/users` with a valid JWT for a user with only `ROLE_USER`.

Walk through exactly:
- Which rule matches?
- What does Spring Security do?
- What HTTP status does the client receive?
- Which class generates that response?

**Q2.** A developer removes `SessionCreationPolicy.STATELESS` from the config. Everything seems to work. But after the first successful JWT request, subsequent requests to protected endpoints succeed even WITHOUT the `Authorization` header.

Why does this happen? What is Spring Security doing internally?

**Q3.** You need to add a second filter chain specifically for `/actuator/**` endpoints that:
- Requires HTTP Basic auth (not JWT)
- Only allows users with `ROLE_MONITORING`

Sketch the `SecurityFilterChain` bean for this. How does Spring Security know which chain to use for which request?
