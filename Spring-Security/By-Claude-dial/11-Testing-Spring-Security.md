# Spring Security Masterclass — Step 11: Testing Spring Security

---

## 11.1 Why Security Testing Is Different

Testing Spring Security requires understanding three distinct testing layers. Most developers only use one and miss critical bugs.

```
Layer 1: Unit Tests
  Test security logic in isolation
  No Spring context
  No HTTP
  Fast — milliseconds
  Tests: JwtService, token generation, claims extraction

Layer 2: Slice Tests (@WebMvcTest)
  Test HTTP layer with security filter chain
  Spring MVC context only — no full application context
  No real database
  Tests: Controller endpoints, 401/403 responses, filter behavior

Layer 3: Integration Tests (@SpringBootTest)
  Test complete flow end to end
  Full Spring context
  Real database (H2 in-memory for tests)
  Tests: Register → Login → Use token → Refresh → Logout
```

Each layer catches different bugs. A bug that passes unit tests often fails at the slice level. A bug that passes slice tests often fails at integration level.

---

## 11.2 Test Dependencies

```xml
<!-- pom.xml — test dependencies -->

<dependencies>

    <!-- Spring Boot Test — includes JUnit 5, Mockito, AssertJ -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Spring Security Test — @WithMockUser, SecurityMockMvcRequestPostProcessors -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- H2 — in-memory database for integration tests -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>

</dependencies>
```

---

## 11.3 Test Configuration

```properties
# src/test/resources/application-test.properties
# This profile is activated for integration tests

# ── Use H2 instead of PostgreSQL ──────────────────────────────────────
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# ── JPA ───────────────────────────────────────────────────────────────
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.show-sql=false

# ── JWT — short expiry for testing token expiration scenarios ──────────
app.jwt.secret=testSecretKeyForTestingPurposesOnlyNotForProduction12345
app.jwt.expiration=3600000
app.jwt.refresh-expiration=604800000

# ── Suppress noise in test output ─────────────────────────────────────
logging.level.org.springframework.security=WARN
logging.level.org.hibernate=WARN
```

---

## 11.4 Layer 1: Unit Tests for `JwtService`

Pure unit tests — no Spring context, no database, no HTTP.

```java
// src/test/java/com/example/demo/security/JwtServiceTest.java

package com.example.demo.security;

import com.example.demo.model.User;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.ExpiredJwtException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.security.core.userdetails.UserDetails;

import java.lang.reflect.Field;
import java.util.Date;

import static org.assertj.core.api.Assertions.*;

class JwtServiceTest {

    private JwtService jwtService;
    private User testUser;

    @BeforeEach
    void setUp() throws Exception {
        jwtService = new JwtService();

        // Inject private fields using reflection
        // We do this because JwtService reads from @Value
        // but there is no Spring context in unit tests
        setField(jwtService, "secretKey",
            "testSecretKeyForTestingPurposesOnlyNotForProduction12345");
        setField(jwtService, "jwtExpiration", 3600000L);

        testUser = new User("john", "hashedPassword",
                            "john@example.com", "ROLE_USER");
    }

    // ── Token Generation ───────────────────────────────────────────────

    @Nested
    @DisplayName("Token Generation")
    class TokenGeneration {

        @Test
        @DisplayName("generates a non-null, non-empty token")
        void generatesToken() {
            String token = jwtService.generateToken(testUser);

            assertThat(token)
                .isNotNull()
                .isNotEmpty()
                // JWT always has exactly two dots (three parts)
                .satisfies(t -> assertThat(t.split("\\.")).hasSize(3));
        }

        @Test
        @DisplayName("token subject equals the username")
        void tokenSubjectIsUsername() {
            String token = jwtService.generateToken(testUser);
            String subject = jwtService.extractUsername(token);

            assertThat(subject).isEqualTo("john");
        }

        @Test
        @DisplayName("two tokens generated at different times are different")
        void tokensAreUnique() throws InterruptedException {
            String token1 = jwtService.generateToken(testUser);
            // Small delay ensures different iat (issued at) timestamp
            Thread.sleep(10);
            String token2 = jwtService.generateToken(testUser);

            assertThat(token1).isNotEqualTo(token2);
        }

        @Test
        @DisplayName("token contains correct expiration time")
        void tokenHasCorrectExpiration() {
            long beforeGeneration = System.currentTimeMillis();
            String token = jwtService.generateToken(testUser);
            long afterGeneration = System.currentTimeMillis();

            Date expiration = jwtService.extractClaim(
                token, Claims::getExpiration
            );

            // Expiration should be approximately (now + 1 hour)
            long expectedExpiry = beforeGeneration + 3600000L;
            long tolerance = 2000L; // 2 seconds tolerance

            assertThat(expiration.getTime())
                .isGreaterThanOrEqualTo(expectedExpiry - tolerance)
                .isLessThanOrEqualTo(afterGeneration + 3600000L + tolerance);
        }
    }

    // ── Token Validation ───────────────────────────────────────────────

    @Nested
    @DisplayName("Token Validation")
    class TokenValidation {

        @Test
        @DisplayName("valid token passes validation for correct user")
        void validTokenPassesValidation() {
            String token = jwtService.generateToken(testUser);

            boolean isValid = jwtService.isTokenValid(token, testUser);

            assertThat(isValid).isTrue();
        }

        @Test
        @DisplayName("token is invalid for a different user")
        void tokenIsInvalidForDifferentUser() {
            String token = jwtService.generateToken(testUser);

            User differentUser = new User("alice", "pass",
                                          "alice@example.com", "ROLE_USER");

            boolean isValid = jwtService.isTokenValid(token, differentUser);

            assertThat(isValid).isFalse();
        }

        @Test
        @DisplayName("tampered token fails validation")
        void tamperedTokenFailsValidation() {
            String token = jwtService.generateToken(testUser);

            // Tamper with the signature (last part after second dot)
            String[] parts = token.split("\\.");
            String tamperedToken = parts[0] + "." + parts[1] + ".invalidsignature";

            // Tampered token should throw an exception or return false
            assertThatThrownBy(() ->
                jwtService.isTokenValid(tamperedToken, testUser)
            ).isInstanceOf(Exception.class);
        }

        @Test
        @DisplayName("expired token throws ExpiredJwtException")
        void expiredTokenThrowsException() throws Exception {
            // Override expiration to -1 ms (already expired)
            setField(jwtService, "jwtExpiration", -1000L);

            String expiredToken = jwtService.generateToken(testUser);

            assertThatThrownBy(() ->
                jwtService.isTokenValid(expiredToken, testUser)
            ).isInstanceOf(ExpiredJwtException.class);
        }

        @Test
        @DisplayName("token signed with different key fails validation")
        void wrongKeyFailsValidation() throws Exception {
            // Generate token with current key
            String token = jwtService.generateToken(testUser);

            // Change the signing key
            setField(jwtService, "secretKey",
                "completelydifferentkeythatwillcauseverification12345");

            // Validation with wrong key should throw
            assertThatThrownBy(() ->
                jwtService.isTokenValid(token, testUser)
            ).isInstanceOf(Exception.class);
        }
    }

    // ── Claims Extraction ──────────────────────────────────────────────

    @Nested
    @DisplayName("Claims Extraction")
    class ClaimsExtraction {

        @Test
        @DisplayName("extracts username from valid token")
        void extractsUsername() {
            String token = jwtService.generateToken(testUser);

            assertThat(jwtService.extractUsername(token))
                .isEqualTo("john");
        }

        @Test
        @DisplayName("extracts expiration date from valid token")
        void extractsExpiration() {
            String token = jwtService.generateToken(testUser);

            Date expiration = jwtService.extractClaim(token, Claims::getExpiration);

            assertThat(expiration).isAfter(new Date());
        }

        @Test
        @DisplayName("extracts issued-at date from valid token")
        void extractsIssuedAt() {
            long before = System.currentTimeMillis();
            String token = jwtService.generateToken(testUser);
            long after = System.currentTimeMillis();

            Date issuedAt = jwtService.extractClaim(token, Claims::getIssuedAt);

            assertThat(issuedAt.getTime())
                .isGreaterThanOrEqualTo(before)
                .isLessThanOrEqualTo(after);
        }
    }

    // ── Helper ─────────────────────────────────────────────────────────

    private void setField(Object target, String fieldName, Object value)
            throws Exception {
        Field field = target.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(target, value);
    }
}
```

---

## 11.5 Layer 2: Slice Tests with `@WebMvcTest`

`@WebMvcTest` loads only the Spring MVC layer — controllers, filters, security config. No service beans, no repositories, no database.

```
@WebMvcTest loads:
  ✓ Controllers
  ✓ SecurityFilterChain
  ✓ JwtAuthenticationFilter
  ✓ ExceptionHandlers
  ✗ Services (must mock with @MockBean)
  ✗ Repositories (must mock with @MockBean)
  ✗ Database connection
```

### Testing `AuthController`:

```java
// src/test/java/com/example/demo/controller/AuthControllerTest.java

package com.example.demo.controller;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.dto.RegisterRequest;
import com.example.demo.security.JwtService;
import com.example.demo.security.UserDetailsServiceImpl;
import com.example.demo.service.AuthService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(AuthController.class)
class AuthControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    // Mock all beans that SecurityConfig and AuthController depend on
    @MockBean private AuthService authService;
    @MockBean private JwtService jwtService;
    @MockBean private UserDetailsServiceImpl userDetailsService;

    private final AuthResponse sampleResponse = new AuthResponse(
        "eyJhbGci.sample.token",
        "550e8400-refresh-token",
        900000L,
        604800000L
    );

    // ── Registration Tests ─────────────────────────────────────────────

    @Nested
    @DisplayName("POST /auth/register")
    class Register {

        @Test
        @DisplayName("returns 201 with tokens on valid registration")
        void successfulRegistration() throws Exception {
            RegisterRequest request = new RegisterRequest(
                "alice", "password123", "alice@example.com"
            );

            // Arrange: mock service to return token pair
            given(authService.register(any(), any()))
                .willReturn(sampleResponse);

            // Act & Assert
            mockMvc.perform(post("/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())           // 201
                .andExpect(jsonPath("$.accessToken").exists())
                .andExpect(jsonPath("$.refreshToken").exists())
                .andExpect(jsonPath("$.tokenType").value("Bearer"))
                .andExpect(jsonPath("$.accessTokenExpiresIn").value(900000));
        }

        @Test
        @DisplayName("returns 400 when username is too short")
        void rejectsShortUsername() throws Exception {
            RegisterRequest request = new RegisterRequest(
                "ab",           // too short — min 3
                "password123",
                "alice@example.com"
            );

            mockMvc.perform(post("/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())        // 400
                .andExpect(jsonPath("$.errors.username").exists());
        }

        @Test
        @DisplayName("returns 400 when password is too short")
        void rejectsShortPassword() throws Exception {
            RegisterRequest request = new RegisterRequest(
                "alice",
                "short",        // too short — min 8
                "alice@example.com"
            );

            mockMvc.perform(post("/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors.password").exists());
        }

        @Test
        @DisplayName("returns 400 when email is invalid")
        void rejectsInvalidEmail() throws Exception {
            RegisterRequest request = new RegisterRequest(
                "alice",
                "password123",
                "not-an-email"  // invalid email format
            );

            mockMvc.perform(post("/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors.email").exists());
        }

        @Test
        @DisplayName("returns 400 when body is missing")
        void rejectsMissingBody() throws Exception {
            mockMvc.perform(post("/auth/register")
                    .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isBadRequest());
        }

        @Test
        @DisplayName("returns 409 when username already exists")
        void rejectsDuplicateUsername() throws Exception {
            RegisterRequest request = new RegisterRequest(
                "existing", "password123", "new@example.com"
            );

            // Simulate service throwing duplicate error
            given(authService.register(any(), any()))
                .willThrow(new IllegalArgumentException(
                    "Username already taken: existing"
                ));

            mockMvc.perform(post("/auth/register")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isConflict())          // 409
                .andExpect(jsonPath("$.message")
                    .value("Username already taken: existing"));
        }
    }

    // ── Login Tests ────────────────────────────────────────────────────

    @Nested
    @DisplayName("POST /auth/login")
    class Login {

        @Test
        @DisplayName("returns 200 with tokens on valid credentials")
        void successfulLogin() throws Exception {
            LoginRequest request = new LoginRequest("john", "secret123");

            given(authService.login(any(), any()))
                .willReturn(sampleResponse);

            mockMvc.perform(post("/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())                // 200
                .andExpect(jsonPath("$.accessToken").exists())
                .andExpect(jsonPath("$.refreshToken").exists());
        }

        @Test
        @DisplayName("returns 401 on bad credentials")
        void badCredentials() throws Exception {
            LoginRequest request = new LoginRequest("john", "wrongpassword");

            given(authService.login(any(), any()))
                .willThrow(new org.springframework.security.authentication
                    .BadCredentialsException("Bad credentials"));

            mockMvc.perform(post("/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isUnauthorized())      // 401
                .andExpect(jsonPath("$.message")
                    .value("Invalid username or password"));
        }

        @Test
        @DisplayName("returns 400 when username is blank")
        void rejectsBlankUsername() throws Exception {
            LoginRequest request = new LoginRequest("", "password123");

            mockMvc.perform(post("/auth/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest());
        }
    }
}
```

---

### Testing `ApiController` — Secured Endpoints:

```java
// src/test/java/com/example/demo/controller/ApiControllerTest.java

package com.example.demo.controller;

import com.example.demo.model.User;
import com.example.demo.security.JwtService;
import com.example.demo.security.UserDetailsServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.BDDMockito.given;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ApiController.class)
class ApiControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean private JwtService jwtService;
    @MockBean private UserDetailsServiceImpl userDetailsService;

    private User johnUser;
    private User adminUser;

    @BeforeEach
    void setUp() {
        johnUser = new User(
            "john", "hashed", "john@example.com", "ROLE_USER"
        );
        // Set ID via reflection or add setId method
        adminUser = new User(
            "admin", "hashed", "admin@example.com", "ROLE_ADMIN"
        );
    }

    // ── Unauthenticated access ─────────────────────────────────────────

    @Nested
    @DisplayName("Unauthenticated Requests")
    class Unauthenticated {

        @Test
        @DisplayName("GET /api/profile returns 401 without token")
        void profileRequiresAuth() throws Exception {
            mockMvc.perform(get("/api/profile"))
                .andExpect(status().isUnauthorized());  // 401
        }

        @Test
        @DisplayName("GET /api/admin/users returns 401 without token")
        void adminRequiresAuth() throws Exception {
            mockMvc.perform(get("/api/admin/users"))
                .andExpect(status().isUnauthorized());  // 401
        }
    }

    // ── @WithMockUser — simplest way to test secured endpoints ─────────

    @Nested
    @DisplayName("With @WithMockUser")
    class WithMockUserTests {

        @Test
        @WithMockUser(username = "john", roles = {"USER"})
        @DisplayName("authenticated user can access /api/profile")
        void profileAccessibleToAuthenticatedUser() throws Exception {
            // @WithMockUser creates a mock Authentication in SecurityContext
            // No JWT processing — bypasses JwtAuthenticationFilter
            // The principal is a Spring Security User object (not our custom User)
            mockMvc.perform(get("/api/whoami"))
                .andExpect(status().isOk());
        }

        @Test
        @WithMockUser(username = "admin", roles = {"ADMIN"})
        @DisplayName("admin can access /api/admin/users")
        void adminCanAccessAdminEndpoint() throws Exception {
            mockMvc.perform(get("/api/admin/users"))
                .andExpect(status().isOk());
        }

        @Test
        @WithMockUser(username = "john", roles = {"USER"})
        @DisplayName("regular user gets 403 on admin endpoint")
        void userCannotAccessAdminEndpoint() throws Exception {
            mockMvc.perform(get("/api/admin/users"))
                .andExpect(status().isForbidden());     // 403
        }
    }

    // ── SecurityMockMvcRequestPostProcessors.user() ────────────────────
    // More control than @WithMockUser — inject our custom User object

    @Nested
    @DisplayName("With Custom User Principal")
    class WithCustomUser {

        @Test
        @DisplayName("profile endpoint returns correct user data")
        void profileReturnsUserData() throws Exception {
            mockMvc.perform(get("/api/profile")
                    // Inject our custom User as the principal
                    // This allows @AuthenticationPrincipal User to work correctly
                    .with(user(johnUser)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.username").value("john"))
                .andExpect(jsonPath("$.email").value("john@example.com"))
                .andExpect(jsonPath("$.role").value("ROLE_USER"));
        }

        @Test
        @DisplayName("whoami returns correct authentication info")
        void whoamiReturnsCorrectInfo() throws Exception {
            mockMvc.perform(get("/api/whoami")
                    .with(user(adminUser)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.username").value("admin"))
                .andExpect(jsonPath("$.isAdmin").value(true))
                .andExpect(jsonPath("$.isAuthenticated").value(true));
        }
    }

    // ── Testing JWT token in Authorization header ──────────────────────

    @Nested
    @DisplayName("With Real JWT Token")
    class WithJwtToken {

        @Test
        @DisplayName("valid JWT in Authorization header grants access")
        void validJwtGrantsAccess() throws Exception {
            String fakeToken = "valid.jwt.token";

            // Mock the JWT service behavior
            given(jwtService.extractUsername(fakeToken))
                .willReturn("john");
            given(userDetailsService.loadUserByUsername("john"))
                .willReturn(johnUser);
            given(jwtService.isTokenValid(fakeToken, johnUser))
                .willReturn(true);

            mockMvc.perform(get("/api/profile")
                    .header("Authorization", "Bearer " + fakeToken))
                .andExpect(status().isOk());
        }

        @Test
        @DisplayName("invalid JWT returns 401")
        void invalidJwtReturns401() throws Exception {
            String invalidToken = "invalid.jwt.token";

            given(jwtService.extractUsername(invalidToken))
                .willThrow(new io.jsonwebtoken.MalformedJwtException(
                    "Malformed JWT"
                ));

            mockMvc.perform(get("/api/profile")
                    .header("Authorization", "Bearer " + invalidToken))
                .andExpect(status().isUnauthorized());
        }

        @Test
        @DisplayName("expired JWT returns 401")
        void expiredJwtReturns401() throws Exception {
            String expiredToken = "expired.jwt.token";

            given(jwtService.extractUsername(expiredToken))
                .willThrow(new io.jsonwebtoken.ExpiredJwtException(
                    null, null, "Token expired"
                ));

            mockMvc.perform(get("/api/profile")
                    .header("Authorization", "Bearer " + expiredToken))
                .andExpect(status().isUnauthorized());
        }
    }
}
```

---

## 11.6 `@WithMockUser` vs `.with(user(...))` vs Real JWT — When to Use Each

```
@WithMockUser(username = "john", roles = {"USER"})
  ✓ Simplest — one annotation
  ✓ Good for testing access control rules
  ✗ Principal is Spring's User, not your custom User class
  ✗ @AuthenticationPrincipal User currentUser will fail (wrong type)
  Use when: testing role-based access (200 vs 403), not inspecting response data

.with(user(yourCustomUserObject))
  ✓ Injects your actual User object as principal
  ✓ @AuthenticationPrincipal User currentUser works correctly
  ✓ Response body contains real user data
  ✗ Slightly more setup
  Use when: testing endpoints that use @AuthenticationPrincipal and return user data

Real JWT token + mocked JwtService
  ✓ Tests the actual JwtAuthenticationFilter processing
  ✓ Catches bugs in filter header parsing
  ✓ Tests error responses for malformed/expired tokens
  ✗ Most setup — must mock JwtService behavior
  Use when: testing the filter chain itself, error handling for bad tokens
```

---

## 11.7 Custom Security Test Annotation

Reduce boilerplate by creating reusable security annotations:

```java
// src/test/java/com/example/demo/security/WithMockJohnUser.java

package com.example.demo.security;

import org.springframework.security.test.context.support.WithMockUser;
import java.lang.annotation.*;

// Reusable annotation for tests that need john (ROLE_USER)
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "john", roles = {"USER"})
public @interface WithMockJohnUser {
}
```

```java
// src/test/java/com/example/demo/security/WithMockAdminUser.java

package com.example.demo.security;

import org.springframework.security.test.context.support.WithMockUser;
import java.lang.annotation.*;

// Reusable annotation for tests that need admin (ROLE_ADMIN)
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "admin", roles = {"ADMIN"})
public @interface WithMockAdminUser {
}
```

Usage is now clean:

```java
@Test
@WithMockJohnUser
void regularUserTest() throws Exception { ... }

@Test
@WithMockAdminUser
void adminUserTest() throws Exception { ... }
```

---

## 11.8 Layer 3: Integration Tests

Full Spring context, real database (H2), real JWT generation. Tests the complete flow from HTTP request to database and back.

```java
// src/test/java/com/example/demo/integration/AuthIntegrationTest.java

package com.example.demo.integration;

import com.example.demo.dto.LoginRequest;
import com.example.demo.dto.RefreshRequest;
import com.example.demo.dto.RegisterRequest;
import com.example.demo.repository.RefreshTokenRepository;
import com.example.demo.repository.UserRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")   // loads application-test.properties
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class AuthIntegrationTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    @Autowired private UserRepository userRepository;
    @Autowired private RefreshTokenRepository refreshTokenRepository;

    // Shared state across ordered tests
    private static String accessToken;
    private static String refreshToken;

    @BeforeEach
    void cleanDatabase() {
        // Clean state before each test
        refreshTokenRepository.deleteAll();
        userRepository.deleteAll();
    }

    // ── Test 1: Register ───────────────────────────────────────────────

    @Test
    @Order(1)
    @DisplayName("1. Register creates user and returns token pair")
    void registerCreatesUserAndReturnsTokens() throws Exception {
        RegisterRequest request = new RegisterRequest(
            "alice", "password123", "alice@example.com"
        );

        MvcResult result = mockMvc.perform(post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.accessToken").isNotEmpty())
            .andExpect(jsonPath("$.refreshToken").isNotEmpty())
            .andExpect(jsonPath("$.tokenType").value("Bearer"))
            .andReturn();

        // Verify user was persisted to database
        assertThat(userRepository.findByUsername("alice")).isPresent();

        // Verify refresh token was persisted to database
        assertThat(refreshTokenRepository.count()).isEqualTo(1);

        // Store tokens for subsequent tests
        String responseBody = result.getResponse().getContentAsString();
        accessToken = extractField(responseBody, "accessToken");
        refreshToken = extractField(responseBody, "refreshToken");
    }

    // ── Test 2: Login ──────────────────────────────────────────────────

    @Test
    @Order(2)
    @DisplayName("2. Login with valid credentials returns token pair")
    void loginWithValidCredentials() throws Exception {
        // First register
        registerUser("bob", "password123", "bob@example.com");

        LoginRequest request = new LoginRequest("bob", "password123");

        MvcResult result = mockMvc.perform(post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").isNotEmpty())
            .andExpect(jsonPath("$.refreshToken").isNotEmpty())
            .andReturn();

        String responseBody = result.getResponse().getContentAsString();
        accessToken = extractField(responseBody, "accessToken");
        refreshToken = extractField(responseBody, "refreshToken");
    }

    // ── Test 3: Access protected endpoint ─────────────────────────────

    @Test
    @Order(3)
    @DisplayName("3. Valid access token grants access to protected endpoint")
    void accessTokenGrantsAccess() throws Exception {
        // Register and login to get token
        String token = registerAndGetAccessToken(
            "carol", "password123", "carol@example.com"
        );

        mockMvc.perform(get("/api/profile")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("carol"))
            .andExpect(jsonPath("$.email").value("carol@example.com"));
    }

    // ── Test 4: No token returns 401 ──────────────────────────────────

    @Test
    @Order(4)
    @DisplayName("4. Missing token returns 401")
    void missingTokenReturns401() throws Exception {
        mockMvc.perform(get("/api/profile"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.status").value(401));
    }

    // ── Test 5: Invalid credentials return 401 ────────────────────────

    @Test
    @Order(5)
    @DisplayName("5. Wrong password returns 401 with generic message")
    void wrongPasswordReturns401() throws Exception {
        registerUser("dave", "password123", "dave@example.com");

        LoginRequest request = new LoginRequest("dave", "wrongpassword");

        mockMvc.perform(post("/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isUnauthorized())
            // Generic message — does not reveal whether user exists
            .andExpect(jsonPath("$.message")
                .value("Invalid username or password"));
    }

    // ── Test 6: Refresh token flow ─────────────────────────────────────

    @Test
    @Order(6)
    @DisplayName("6. Refresh token returns new token pair")
    void refreshTokenReturnsNewPair() throws Exception {
        // Setup
        String[] tokens = registerAndGetBothTokens(
            "eve", "password123", "eve@example.com"
        );
        String oldRefreshToken = tokens[1];

        RefreshRequest request = new RefreshRequest(oldRefreshToken);

        MvcResult result = mockMvc.perform(post("/auth/refresh")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.accessToken").isNotEmpty())
            .andExpect(jsonPath("$.refreshToken").isNotEmpty())
            .andReturn();

        String responseBody = result.getResponse().getContentAsString();
        String newRefreshToken = extractField(responseBody, "refreshToken");

        // New refresh token should be different (rotation)
        assertThat(newRefreshToken).isNotEqualTo(oldRefreshToken);

        // Old refresh token should now be revoked in database
        refreshTokenRepository.findByToken(oldRefreshToken)
            .ifPresent(rt -> assertThat(rt.isRevoked()).isTrue());
    }

    // ── Test 7: Reusing rotated token fails ───────────────────────────

    @Test
    @Order(7)
    @DisplayName("7. Using already-rotated refresh token returns 401")
    void rotatedRefreshTokenIsRejected() throws Exception {
        String[] tokens = registerAndGetBothTokens(
            "frank", "password123", "frank@example.com"
        );
        String originalRefreshToken = tokens[1];

        // First refresh — consumes the token, rotates it
        mockMvc.perform(post("/auth/refresh")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(
                    new RefreshRequest(originalRefreshToken)
                )))
            .andExpect(status().isOk());

        // Second refresh with same token — should fail
        mockMvc.perform(post("/auth/refresh")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(
                    new RefreshRequest(originalRefreshToken)
                )))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.message")
                .value("Refresh token has been revoked — please log in again"));
    }

    // ── Test 8: Logout ─────────────────────────────────────────────────

    @Test
    @Order(8)
    @DisplayName("8. Logout revokes refresh token")
    void logoutRevokesRefreshToken() throws Exception {
        String[] tokens = registerAndGetBothTokens(
            "grace", "password123", "grace@example.com"
        );
        String accessTk = tokens[0];
        String refreshTk = tokens[1];

        // Logout
        mockMvc.perform(post("/auth/logout")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Authorization", "Bearer " + accessTk)
                .content(objectMapper.writeValueAsString(
                    new RefreshRequest(refreshTk)
                )))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.message").value("Logged out successfully"));

        // Verify refresh token is revoked in database
        refreshTokenRepository.findByToken(refreshTk)
            .ifPresent(rt -> assertThat(rt.isRevoked()).isTrue());

        // Verify cannot refresh after logout
        mockMvc.perform(post("/auth/refresh")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(
                    new RefreshRequest(refreshTk)
                )))
            .andExpect(status().isUnauthorized());
    }

    // ── Test 9: Role-based access ──────────────────────────────────────

    @Test
    @Order(9)
    @DisplayName("9. ROLE_USER cannot access admin endpoint (403)")
    void regularUserCannotAccessAdminEndpoint() throws Exception {
        String token = registerAndGetAccessToken(
            "henry", "password123", "henry@example.com"
        );

        mockMvc.perform(get("/api/admin/users")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isForbidden());  // 403 — authenticated but wrong role
    }

    // ── Test 10: Duplicate registration ───────────────────────────────

    @Test
    @Order(10)
    @DisplayName("10. Duplicate username registration returns 409")
    void duplicateUsernameReturns409() throws Exception {
        registerUser("iris", "password123", "iris@example.com");

        // Try to register same username again
        RegisterRequest duplicate = new RegisterRequest(
            "iris", "different123", "different@example.com"
        );

        mockMvc.perform(post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(duplicate)))
            .andExpect(status().isConflict())   // 409
            .andExpect(jsonPath("$.message")
                .value("Username already taken: iris"));
    }

    // ── Helper methods ─────────────────────────────────────────────────

    private void registerUser(String username, String password,
                               String email) throws Exception {
        RegisterRequest request = new RegisterRequest(username, password, email);
        mockMvc.perform(post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated());
    }

    private String registerAndGetAccessToken(String username, String password,
                                              String email) throws Exception {
        RegisterRequest request = new RegisterRequest(username, password, email);
        MvcResult result = mockMvc.perform(post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andReturn();
        return extractField(result.getResponse().getContentAsString(), "accessToken");
    }

    private String[] registerAndGetBothTokens(String username, String password,
                                               String email) throws Exception {
        RegisterRequest request = new RegisterRequest(username, password, email);
        MvcResult result = mockMvc.perform(post("/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andReturn();
        String body = result.getResponse().getContentAsString();
        return new String[]{
            extractField(body, "accessToken"),
            extractField(body, "refreshToken")
        };
    }

    private String extractField(String json, String field) throws Exception {
        return objectMapper.readTree(json).get(field).asText();
    }
}
```

---

## 11.9 Testing `@PreAuthorize` Ownership Rules

```java
// src/test/java/com/example/demo/controller/OwnershipSecurityTest.java

package com.example.demo.controller;

import com.example.demo.model.User;
import com.example.demo.security.JwtService;
import com.example.demo.security.UserDetailsServiceImpl;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(ApiController.class)
class OwnershipSecurityTest {

    @Autowired private MockMvc mockMvc;
    @MockBean private JwtService jwtService;
    @MockBean private UserDetailsServiceImpl userDetailsService;

    @Test
    @DisplayName("user can access their own resource")
    void userCanAccessOwnResource() throws Exception {
        User john = new User("john", "pass", "john@ex.com", "ROLE_USER");
        // john's id is 1

        mockMvc.perform(get("/api/users/1/data")
                .with(user(john)))
            .andExpect(status().isOk());
    }

    @Test
    @DisplayName("user cannot access another user's resource")
    void userCannotAccessOthersResource() throws Exception {
        User john = new User("john", "pass", "john@ex.com", "ROLE_USER");
        // john's id is 1, trying to access id=2

        mockMvc.perform(get("/api/users/2/data")
                .with(user(john)))
            .andExpect(status().isForbidden());  // 403
    }

    @Test
    @DisplayName("admin can access any user's resource")
    void adminCanAccessAnyResource() throws Exception {
        User admin = new User("admin", "pass", "admin@ex.com", "ROLE_ADMIN");

        mockMvc.perform(get("/api/users/1/data")
                .with(user(admin)))
            .andExpect(status().isOk());
    }
}
```

---

## 11.10 Running Tests

```bash
# Run all tests
mvn test

# Run only unit tests (fast)
mvn test -Dtest="JwtServiceTest"

# Run only slice tests
mvn test -Dtest="AuthControllerTest,ApiControllerTest"

# Run only integration tests
mvn test -Dtest="AuthIntegrationTest"

# Run with verbose output
mvn test -Dsurefire.useFile=false

# Run and generate coverage report (requires JaCoCo plugin)
mvn verify
```

### Expected test output:

```
[INFO] Tests run: 8,  Failures: 0, Errors: 0 — JwtServiceTest
[INFO] Tests run: 12, Failures: 0, Errors: 0 — AuthControllerTest
[INFO] Tests run: 5,  Failures: 0, Errors: 0 — ApiControllerTest
[INFO] Tests run: 10, Failures: 0, Errors: 0 — AuthIntegrationTest
[INFO] Tests run: 3,  Failures: 0, Errors: 0 — OwnershipSecurityTest
[INFO] ─────────────────────────────────────────────────────
[INFO] BUILD SUCCESS
[INFO] Total tests run: 38, Failures: 0, Errors: 0, Skipped: 0
```

---

## Section 11 — Interview Summary

| Question | Answer |
|----------|--------|
| What does `@WebMvcTest` load? | MVC layer only — controllers, filters, security config. No services, repositories, or database |
| What does `@SpringBootTest` load? | Full application context — all beans, database, complete filter chain |
| What is the difference between `@WithMockUser` and `.with(user(...))`? | `@WithMockUser` injects Spring's generic `User` as principal. `.with(user(...))` injects your custom object — needed when `@AuthenticationPrincipal` is used |
| Why use `@MockBean` in `@WebMvcTest`? | Services and repositories are not loaded — `@MockBean` creates mocks and registers them in the context |
| What does `@ActiveProfiles("test")` do? | Activates `application-test.properties` — uses H2 instead of PostgreSQL |
| When would you use real JWT tokens in slice tests? | When testing the `JwtAuthenticationFilter` behavior itself — header parsing, expired token handling, malformed token responses |
| What is the purpose of `@TestMethodOrder`? | Controls test execution order — needed when integration tests share state across ordered steps |
| What three test layers should every Spring Security app have? | Unit tests (JwtService), slice tests (@WebMvcTest for filter/controller), integration tests (@SpringBootTest for end-to-end flow) |

---

## Quick Understanding Check

**Q1.** A `@WebMvcTest` for `ApiController` passes all tests. The integration test for the same endpoint fails with 401. Name three possible reasons this discrepancy exists and how you would diagnose each one.

**Q2.** You add a new endpoint:

```java
@GetMapping("/api/reports")
@PreAuthorize("hasRole('ADMIN') or @reportService.isOwner(principal, #reportId)")
public Report getReport(@PathVariable Long reportId,
                        @AuthenticationPrincipal User user) { ... }
```

Write the test method signatures (not implementation) for every security scenario this endpoint has. Name each test and specify which annotation or post-processor you would use.

**Q3.** A colleague says: *"Integration tests are slow. We should just write `@WebMvcTest` tests for everything and skip `@SpringBootTest`."*

Give two concrete examples of bugs that `@WebMvcTest` would miss but `@SpringBootTest` would catch in a Spring Security application.

---

This completes the full Spring Security Masterclass. Here is the complete picture of what you have built:

```
Section 1-3:  Concepts — Authentication vs Authorization, JWT structure, filter chain
Section 4:    Project setup and dependencies
Section 5:    JwtService — token generation and validation
Section 6:    SecurityConfig — filter chain wiring
Section 7:    AuthController — login endpoint
Section 8:    Secured APIs — @PreAuthorize, @AuthenticationPrincipal, 401 vs 403
Section 9:    Database integration — JPA entities, real user persistence
Section 10:   Token refresh and logout — refresh token rotation, revocation
Section 11:   Testing — unit, slice, and integration tests
```
