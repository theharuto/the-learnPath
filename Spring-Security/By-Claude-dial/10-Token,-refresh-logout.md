# Spring Security Masterclass — Step 10: Token Refresh and Logout

---

## 10.1 The Problem This Section Solves

Short-lived JWTs are a security requirement. If a token is stolen, it should expire quickly. But short expiry creates a usability problem:

```
Token expires every 15 minutes
  → User must log in every 15 minutes
  → Terrible user experience

Token expires every 30 days
  → User rarely logs in
  → Stolen token valid for 30 days
  → Catastrophic if compromised

The solution: Two tokens working together

Access Token:
  Short-lived (15 minutes)
  Sent with every API request
  Stored in memory (JavaScript variable)
  If stolen → expires in 15 minutes

Refresh Token:
  Long-lived (7 days)
  Sent ONLY to /auth/refresh endpoint
  Stored in HttpOnly cookie or secure storage
  Used ONLY to get new access tokens
  Can be revoked in the database
```

This is the industry standard pattern used by every serious API.

---

## 10.2 The Token Pair Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Complete Token Lifecycle                      │
└─────────────────────────────────────────────────────────────────┘

1. LOGIN
   Client → POST /auth/login (username + password)
   Server → returns access_token (15 min) + refresh_token (7 days)

2. NORMAL API USAGE
   Client → GET /api/data (Authorization: Bearer access_token)
   Server → validates access_token → returns data
   (Repeat until access_token expires)

3. ACCESS TOKEN EXPIRES
   Client → GET /api/data (Authorization: Bearer expired_token)
   Server → 401 Unauthorized (token expired)

4. REFRESH
   Client → POST /auth/refresh (refresh_token in body or cookie)
   Server → validates refresh_token against database
          → generates new access_token
          → returns new access_token (optionally new refresh_token)

5. LOGOUT
   Client → POST /auth/logout (refresh_token)
   Server → deletes refresh_token from database
          → future refresh attempts fail
          → access_token still valid until it expires naturally

6. REFRESH TOKEN EXPIRES OR IS REVOKED
   Client → POST /auth/refresh
   Server → 401 (refresh_token not found or expired)
   Client → must log in again
```

---

## 10.3 The Refresh Token Entity

Refresh tokens must be stored in the database. This is what enables revocation — something impossible with stateless JWTs alone.

```java
// src/main/java/com/example/demo/model/RefreshToken.java

package com.example.demo.model;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // The token value — a random UUID, NOT a JWT
    // Stored as plain text because we look it up by exact match
    // No need to hash (it is random and unguessable)
    // Some implementations hash it for extra security — we keep it simple
    @Column(nullable = false, unique = true)
    private String token;

    // Which user this refresh token belongs to
    // Many refresh tokens can belong to one user
    // (user logged in from multiple devices)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // When this refresh token expires
    // Checked on every refresh attempt
    @Column(nullable = false)
    private Instant expiresAt;

    // Whether this token has been explicitly revoked
    // Used for logout and security events
    @Column(nullable = false)
    private boolean revoked = false;

    // When this token was created — for audit trail
    @Column(nullable = false)
    private Instant createdAt;

    // Device/client info — optional but useful for security dashboards
    // "Chrome on Windows", "iOS App", etc.
    @Column
    private String deviceInfo;

    // ── JPA Lifecycle ──────────────────────────────────────────────────
    @PrePersist
    protected void onCreate() {
        createdAt = Instant.now();
    }

    // ── Constructors ───────────────────────────────────────────────────
    protected RefreshToken() {}

    public RefreshToken(String token, User user,
                        Instant expiresAt, String deviceInfo) {
        this.token = token;
        this.user = user;
        this.expiresAt = expiresAt;
        this.deviceInfo = deviceInfo;
    }

    // ── Business logic methods ─────────────────────────────────────────

    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }

    public boolean isValid() {
        return !revoked && !isExpired();
    }

    public void revoke() {
        this.revoked = true;
    }

    // ── Getters ────────────────────────────────────────────────────────
    public Long getId() { return id; }
    public String getToken() { return token; }
    public User getUser() { return user; }
    public Instant getExpiresAt() { return expiresAt; }
    public boolean isRevoked() { return revoked; }
    public Instant getCreatedAt() { return createdAt; }
    public String getDeviceInfo() { return deviceInfo; }

    // ── Setters ────────────────────────────────────────────────────────
    public void setRevoked(boolean revoked) { this.revoked = revoked; }
}
```

**Why UUID instead of JWT for refresh tokens?**

```
JWT refresh token:
  ✓ Self-contained (contains user info)
  ✗ Cannot be revoked without a denylist
  ✗ If secret key rotates, all refresh tokens invalidated
  ✗ Larger payload transmitted every refresh

UUID refresh token:
  ✓ Revocable — delete from database
  ✓ Key rotation does not affect existing tokens
  ✓ Small — just a random string
  ✓ No sensitive data in the token itself
  ✗ Requires database lookup on every refresh (acceptable)
```

---

## 10.4 The Refresh Token Repository

```java
// src/main/java/com/example/demo/repository/RefreshTokenRepository.java

package com.example.demo.repository;

import com.example.demo.model.RefreshToken;
import com.example.demo.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

@Repository
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    // Find by token value — used during refresh and logout
    Optional<RefreshToken> findByToken(String token);

    // Find all tokens for a user — used to show active sessions
    List<RefreshToken> findByUser(User user);

    // Find all valid (non-revoked, non-expired) tokens for a user
    List<RefreshToken> findByUserAndRevokedFalseAndExpiresAtAfter(
        User user, Instant now
    );

    // Revoke all tokens for a user — used when password changes
    // or when security event detected
    @Modifying
    @Query("UPDATE RefreshToken rt SET rt.revoked = true WHERE rt.user = :user")
    void revokeAllUserTokens(User user);

    // Delete expired tokens — called by scheduled cleanup job
    @Modifying
    @Query("DELETE FROM RefreshToken rt WHERE rt.expiresAt < :now")
    void deleteAllExpiredTokens(Instant now);

    // Count active sessions for a user — security dashboard
    @Query("SELECT COUNT(rt) FROM RefreshToken rt " +
           "WHERE rt.user = :user AND rt.revoked = false " +
           "AND rt.expiresAt > :now")
    long countActiveTokens(User user, Instant now);
}
```

---

## 10.5 Updated DTOs

```java
// src/main/java/com/example/demo/dto/AuthResponse.java
// Updated to include both tokens

package com.example.demo.dto;

public class AuthResponse {

    private String accessToken;
    private String refreshToken;
    private String tokenType = "Bearer";
    private long accessTokenExpiresIn;   // milliseconds
    private long refreshTokenExpiresIn;  // milliseconds

    public AuthResponse() {}

    public AuthResponse(String accessToken, String refreshToken,
                        long accessTokenExpiresIn, long refreshTokenExpiresIn) {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
        this.accessTokenExpiresIn = accessTokenExpiresIn;
        this.refreshTokenExpiresIn = refreshTokenExpiresIn;
    }

    // Getters
    public String getAccessToken() { return accessToken; }
    public String getRefreshToken() { return refreshToken; }
    public String getTokenType() { return tokenType; }
    public long getAccessTokenExpiresIn() { return accessTokenExpiresIn; }
    public long getRefreshTokenExpiresIn() { return refreshTokenExpiresIn; }

    // Setters
    public void setAccessToken(String accessToken) { this.accessToken = accessToken; }
    public void setRefreshToken(String refreshToken) { this.refreshToken = refreshToken; }
    public void setTokenType(String tokenType) { this.tokenType = tokenType; }
    public void setAccessTokenExpiresIn(long v) { this.accessTokenExpiresIn = v; }
    public void setRefreshTokenExpiresIn(long v) { this.refreshTokenExpiresIn = v; }
}
```

```java
// src/main/java/com/example/demo/dto/RefreshRequest.java

package com.example.demo.dto;

import jakarta.validation.constraints.NotBlank;

public class RefreshRequest {

    @NotBlank(message = "Refresh token is required")
    private String refreshToken;

    public RefreshRequest() {}
    public RefreshRequest(String refreshToken) { this.refreshToken = refreshToken; }

    public String getRefreshToken() { return refreshToken; }
    public void setRefreshToken(String refreshToken) {
        this.refreshToken = refreshToken;
    }
}
```

---

## 10.6 The Refresh Token Service

```java
// src/main/java/com/example/demo/service/RefreshTokenService.java

package com.example.demo.service;

import com.example.demo.model.RefreshToken;
import com.example.demo.model.User;
import com.example.demo.repository.RefreshTokenRepository;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.UUID;

@Service
public class RefreshTokenService {

    // Refresh token TTL — 7 days in milliseconds
    @Value("${app.jwt.refresh-expiration:604800000}")
    private long refreshTokenExpiration;

    private final RefreshTokenRepository refreshTokenRepository;

    public RefreshTokenService(RefreshTokenRepository refreshTokenRepository) {
        this.refreshTokenRepository = refreshTokenRepository;
    }

    // ── Create a new refresh token ─────────────────────────────────────
    @Transactional
    public RefreshToken createRefreshToken(User user, String deviceInfo) {

        RefreshToken refreshToken = new RefreshToken(
            // UUID.randomUUID() generates a cryptographically random token
            // Format: "550e8400-e29b-41d4-a716-446655440000"
            // 122 bits of entropy — practically unguessable
            UUID.randomUUID().toString(),
            user,
            Instant.now().plusMillis(refreshTokenExpiration),
            deviceInfo
        );

        return refreshTokenRepository.save(refreshToken);
    }

    // ── Validate a refresh token ───────────────────────────────────────
    @Transactional(readOnly = true)
    public RefreshToken validateRefreshToken(String token) {

        RefreshToken refreshToken = refreshTokenRepository
            .findByToken(token)
            .orElseThrow(() -> new RefreshTokenException(
                "Refresh token not found — please log in again"
            ));

        // Check if explicitly revoked (logout, password change, security event)
        if (refreshToken.isRevoked()) {
            throw new RefreshTokenException(
                "Refresh token has been revoked — please log in again"
            );
        }

        // Check if expired naturally
        if (refreshToken.isExpired()) {
            // Clean up expired token
            refreshTokenRepository.delete(refreshToken);
            throw new RefreshTokenException(
                "Refresh token has expired — please log in again"
            );
        }

        return refreshToken;
    }

    // ── Revoke a specific token (logout) ───────────────────────────────
    @Transactional
    public void revokeToken(String token) {
        refreshTokenRepository
            .findByToken(token)
            .ifPresent(rt -> {
                rt.revoke();
                refreshTokenRepository.save(rt);
            });
    }

    // ── Revoke all tokens for a user (logout everywhere) ──────────────
    @Transactional
    public void revokeAllUserTokens(User user) {
        refreshTokenRepository.revokeAllUserTokens(user);
    }

    // ── Scheduled cleanup — runs daily at midnight ─────────────────────
    // Removes expired tokens to prevent table bloat
    // @Scheduled requires @EnableScheduling in a config class
    @Scheduled(cron = "0 0 0 * * *")
    @Transactional
    public void cleanupExpiredTokens() {
        refreshTokenRepository.deleteAllExpiredTokens(Instant.now());
    }

    // ── Custom exception ───────────────────────────────────────────────
    public static class RefreshTokenException extends RuntimeException {
        public RefreshTokenException(String message) {
            super(message);
        }
    }
}
```

---

## 10.7 Updated `AuthService`

```java
// src/main/java/com/example/demo/service/AuthService.java
// Full updated version with refresh token support

package com.example.demo.service;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.dto.RefreshRequest;
import com.example.demo.dto.RegisterRequest;
import com.example.demo.model.RefreshToken;
import com.example.demo.model.User;
import com.example.demo.repository.UserRepository;
import com.example.demo.security.JwtService;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import jakarta.servlet.http.HttpServletRequest;

@Service
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;
    private final RefreshTokenService refreshTokenService;

    public AuthService(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder,
            JwtService jwtService,
            AuthenticationManager authenticationManager,
            UserDetailsService userDetailsService,
            RefreshTokenService refreshTokenService
    ) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.jwtService = jwtService;
        this.authenticationManager = authenticationManager;
        this.userDetailsService = userDetailsService;
        this.refreshTokenService = refreshTokenService;
    }

    // ── Register ───────────────────────────────────────────────────────
    @Transactional
    public AuthResponse register(RegisterRequest request,
                                 HttpServletRequest httpRequest) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new IllegalArgumentException(
                "Username already taken: " + request.getUsername());
        }
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new IllegalArgumentException(
                "Email already registered: " + request.getEmail());
        }

        User user = new User(
            request.getUsername(),
            passwordEncoder.encode(request.getPassword()),
            request.getEmail(),
            "ROLE_USER"
        );
        User savedUser = userRepository.save(user);

        return buildAuthResponse(savedUser, httpRequest);
    }

    // ── Login ──────────────────────────────────────────────────────────
    @Transactional
    public AuthResponse login(LoginRequest request,
                              HttpServletRequest httpRequest) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        User user = (User) userDetailsService
            .loadUserByUsername(request.getUsername());

        return buildAuthResponse(user, httpRequest);
    }

    // ── Refresh ────────────────────────────────────────────────────────
    @Transactional
    public AuthResponse refresh(RefreshRequest request,
                                HttpServletRequest httpRequest) {

        // Step 1: Validate the refresh token (checks DB, expiry, revoked)
        RefreshToken refreshToken = refreshTokenService
            .validateRefreshToken(request.getRefreshToken());

        // Step 2: Get the user associated with this refresh token
        User user = refreshToken.getUser();

        // Step 3: Revoke the used refresh token
        // This implements "refresh token rotation":
        // each use generates a new refresh token and invalidates the old one
        // If an attacker steals a refresh token and uses it,
        // the legitimate user's next refresh will fail → alert detected
        refreshToken.revoke();

        // Step 4: Generate new token pair
        return buildAuthResponse(user, httpRequest);
    }

    // ── Logout (single device) ─────────────────────────────────────────
    @Transactional
    public void logout(RefreshRequest request) {
        // Revoke this specific refresh token
        // The access token will expire naturally (no revocation possible for JWT)
        refreshTokenService.revokeToken(request.getRefreshToken());
    }

    // ── Logout everywhere (all devices) ───────────────────────────────
    @Transactional
    public void logoutAll(User currentUser) {
        refreshTokenService.revokeAllUserTokens(currentUser);
    }

    // ── Private: Build the complete auth response ──────────────────────
    private AuthResponse buildAuthResponse(User user,
                                           HttpServletRequest httpRequest) {
        // Generate access token (JWT — short lived)
        String accessToken = jwtService.generateToken(user);

        // Create refresh token (UUID — stored in DB — long lived)
        String deviceInfo = extractDeviceInfo(httpRequest);
        RefreshToken refreshToken = refreshTokenService
            .createRefreshToken(user, deviceInfo);

        return new AuthResponse(
            accessToken,
            refreshToken.getToken(),
            jwtService.getExpirationTime(),
            // 7 days in milliseconds
            604800000L
        );
    }

    // Extract device info from request headers for audit trail
    private String extractDeviceInfo(HttpServletRequest request) {
        String userAgent = request.getHeader("User-Agent");
        return userAgent != null
            ? userAgent.substring(0, Math.min(userAgent.length(), 200))
            : "Unknown device";
    }
}
```

### Refresh Token Rotation — Why It Matters:

```
WITHOUT rotation:
  Attacker steals refresh token
  Attacker keeps using it indefinitely (until 7-day expiry)
  Legitimate user has no idea

WITH rotation:
  Day 1: Legitimate user has refresh_token_A
  Day 2: Attacker steals refresh_token_A
  Day 3: Legitimate user refreshes → uses refresh_token_A
          → gets refresh_token_B
          → refresh_token_A is now revoked
  Day 4: Attacker tries to use refresh_token_A
          → REJECTED (revoked)
          → Attacker is locked out
  Day 4: Legitimate user refreshes → uses refresh_token_B → works
```

---

## 10.8 Updated Auth Controller

```java
// src/main/java/com/example/demo/controller/AuthController.java

package com.example.demo.controller;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.dto.RefreshRequest;
import com.example.demo.dto.RegisterRequest;
import com.example.demo.model.User;
import com.example.demo.service.AuthService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(
            @Valid @RequestBody RegisterRequest request,
            HttpServletRequest httpRequest
    ) {
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(authService.register(request, httpRequest));
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(
            @Valid @RequestBody LoginRequest request,
            HttpServletRequest httpRequest
    ) {
        return ResponseEntity.ok(authService.login(request, httpRequest));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(
            @Valid @RequestBody RefreshRequest request,
            HttpServletRequest httpRequest
    ) {
        return ResponseEntity.ok(authService.refresh(request, httpRequest));
    }

    // Logout from current device only
    @PostMapping("/logout")
    public ResponseEntity<Map<String, String>> logout(
            @Valid @RequestBody RefreshRequest request
    ) {
        authService.logout(request);
        return ResponseEntity.ok(Map.of(
            "message", "Logged out successfully"
        ));
    }

    // Logout from ALL devices — requires authentication
    @PostMapping("/logout-all")
    public ResponseEntity<Map<String, String>> logoutAll(
            @AuthenticationPrincipal User currentUser
    ) {
        authService.logoutAll(currentUser);
        return ResponseEntity.ok(Map.of(
            "message", "Logged out from all devices"
        ));
    }
}
```

---

## 10.9 The JWT Revocation Problem — And Real Solutions

This is the most important conceptual section. Understand it deeply.

```
THE FUNDAMENTAL PROBLEM:

JWT is stateless by design.
Once issued, it is valid until expiry.
There is no "cancel" button.

Scenario:
  User's JWT is stolen (XSS, network sniff, etc.)
  User reports account compromise
  You want to immediately revoke the token
  You CANNOT — the JWT is self-validating
  The attacker keeps access until expiry

  If expiry is 15 minutes  → attacker has 15 minutes max
  If expiry is 24 hours    → attacker has up to 24 hours
  If expiry is 30 days     → catastrophic
```

### Solution 1: Short Access Token Expiry (Minimum viable)

```
Access token: 15 minutes
Refresh token: 7 days (stored in DB, revocable)

On compromise:
  Revoke refresh token → attacker cannot get new access tokens
  Wait 15 minutes → old access token expires naturally
  Damage window: 15 minutes maximum

Trade-off: Acceptable for most applications
```

### Solution 2: JWT Denylist (Full revocation)

```java
// Store revoked JWTs in Redis until their natural expiry
// Check Redis on every request

@Service
public class TokenDenylistService {

    // Redis stores: key="jwt:jti:abc123", value="revoked", TTL=token remaining lifetime
    private final RedisTemplate<String, String> redisTemplate;

    public TokenDenylistService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void denylistToken(String jti, Duration remainingValidity) {
        // Store until the token would have expired anyway
        // After natural expiry, no need to check denylist
        redisTemplate.opsForValue().set(
            "jwt:denylist:" + jti,
            "revoked",
            remainingValidity
        );
    }

    public boolean isDenylisted(String jti) {
        return redisTemplate.hasKey("jwt:denylist:" + jti);
    }
}
```

This requires:
1. Adding `jti` (JWT ID) claim to every token — a unique identifier per token
2. Checking Redis on every request in `JwtAuthenticationFilter`
3. Running Redis

```java
// Updated JwtService — add jti claim to tokens

public String generateToken(UserDetails userDetails) {
    return Jwts.builder()
        .subject(userDetails.getUsername())
        .id(UUID.randomUUID().toString())  // jti claim — unique token identifier
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + jwtExpiration))
        .signWith(getSigningKey())
        .compact();
}

// Updated JwtAuthenticationFilter — check denylist

String jti = jwtService.extractJti(jwt);
if (tokenDenylistService.isDenylisted(jti)) {
    // Token is revoked — treat as invalid
    filterChain.doFilter(request, response);
    return;
}
```

**Trade-off:** Every request hits Redis. Redis is fast (sub-millisecond), but adds infrastructure dependency and network hop.

### Solution 3: Short-lived tokens, no refresh (Simplest)

```
Access token: 5-15 minutes
No refresh token
User must log in again after expiry

Use when:
  Internal tools where re-login is acceptable
  High-security contexts
  Simplicity is priority
```

### Comparison Table:

```
┌────────────────────┬──────────────┬──────────────┬──────────────┐
│ Approach           │ Revocable?   │ Complexity   │ Performance  │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Short expiry only  │ No (15 min)  │ Low          │ Best         │
│ Refresh tokens     │ Refresh only │ Medium       │ Good         │
│ JWT + Redis        │ Yes          │ High         │ Good (Redis) │
│ Short, no refresh  │ N/A          │ Lowest       │ Best         │
└────────────────────┴──────────────┴──────────────┴──────────────┘
```

**What we implement:** Refresh tokens (Solution 1) — industry standard balance of security and simplicity.

---

## 10.10 Enable Scheduling for Token Cleanup

```java
// src/main/java/com/example/demo/config/SchedulingConfig.java

package com.example.demo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableScheduling;

@Configuration
@EnableScheduling
// Enables @Scheduled annotations across the application
// Without this, @Scheduled methods are ignored silently
public class SchedulingConfig {
    // No code needed — annotation does the work
}
```

Add refresh token expiry to `application.properties`:

```properties
# Refresh token expiry — 7 days in milliseconds
app.jwt.refresh-expiration=604800000
```

---

## 10.11 Updated Security Configuration

Add `/auth/refresh` and `/auth/logout` to public endpoints:

```java
.authorizeHttpRequests(auth -> auth
    // All auth endpoints are public
    // /auth/login, /auth/register, /auth/refresh, /auth/logout
    // Note: /auth/logout-all requires authentication (needs @AuthenticationPrincipal)
    .requestMatchers("/auth/login",
                     "/auth/register",
                     "/auth/refresh",
                     "/auth/logout").permitAll()

    // logout-all requires valid access token to identify the user
    .requestMatchers("/auth/logout-all").authenticated()

    .requestMatchers("/public/**").permitAll()
    .requestMatchers("/actuator/health").permitAll()
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)
```

---

## 10.12 Handle Refresh Token Exception

Add to `GlobalExceptionHandler`:

```java
import com.example.demo.service.RefreshTokenService.RefreshTokenException;

@ExceptionHandler(RefreshTokenException.class)
public ResponseEntity<Map<String, Object>> handleRefreshTokenException(
        RefreshTokenException ex,
        HttpServletRequest request
) {
    return buildErrorResponse(
        HttpStatus.UNAUTHORIZED,
        ex.getMessage(),
        request.getServletPath()
    );
}
```

---

## Hands-On: Complete Token Lifecycle Test

### Step 1: Login and capture both tokens

```bash
LOGIN_RESPONSE=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john","password":"secret123"}')

echo $LOGIN_RESPONSE | python3 -m json.tool

ACCESS_TOKEN=$(echo $LOGIN_RESPONSE \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

REFRESH_TOKEN=$(echo $LOGIN_RESPONSE \
  | grep -o '"refreshToken":"[^"]*"' | cut -d'"' -f4)

echo "Access token:  ${ACCESS_TOKEN:0:30}..."
echo "Refresh token: $REFRESH_TOKEN"
```

**Expected response:**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
  "refreshToken": "550e8400-e29b-41d4-a716-446655440000",
  "tokenType": "Bearer",
  "accessTokenExpiresIn": 900000,
  "refreshTokenExpiresIn": 604800000
}
```

---

### Step 2: Use access token

```bash
curl -s http://localhost:8080/api/profile \
  -H "Authorization: Bearer $ACCESS_TOKEN" | python3 -m json.tool
```

```json
{
  "id": 2,
  "username": "john",
  "email": "john@example.com",
  "role": "ROLE_USER"
}
```
✅ Access token works

---

### Step 3: Refresh tokens

```bash
REFRESH_RESPONSE=$(curl -s -X POST http://localhost:8080/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH_TOKEN\"}")

echo $REFRESH_RESPONSE | python3 -m json.tool

NEW_ACCESS_TOKEN=$(echo $REFRESH_RESPONSE \
  | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

NEW_REFRESH_TOKEN=$(echo $REFRESH_RESPONSE \
  | grep -o '"refreshToken":"[^"]*"' | cut -d'"' -f4)

echo "New access token:  ${NEW_ACCESS_TOKEN:0:30}..."
echo "New refresh token: $NEW_REFRESH_TOKEN"
```

**Expected:** New access token and new refresh token (rotation).

---

### Step 4: Verify old refresh token is revoked

```bash
# Try to use the OLD refresh token — should fail
curl -s -X POST http://localhost:8080/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH_TOKEN\"}" | python3 -m json.tool
```

```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Refresh token has been revoked — please log in again",
  "path": "/auth/refresh"
}
```
✅ Old token rejected — rotation working correctly

---

### Step 5: Verify in database

```bash
docker exec -it security-demo-db \
  psql -U postgres -d securitydemo \
  -c "SELECT id, token, revoked, expires_at FROM refresh_tokens;"
```

```
 id |               token                | revoked |       expires_at
----+------------------------------------+---------+------------------------
  1 | 550e8400-e29b-41d4-a716-446655440 | t       | 2024-01-22 10:30:00
  2 | 7b9d3f12-ab4c-4891-b234-987654321 | f       | 2024-01-22 10:30:01
```

✅ Token 1 (old): `revoked = true`
✅ Token 2 (new): `revoked = false`

---

### Step 6: Logout

```bash
curl -s -X POST http://localhost:8080/auth/logout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $NEW_ACCESS_TOKEN" \
  -d "{\"refreshToken\":\"$NEW_REFRESH_TOKEN\"}" | python3 -m json.tool
```

```json
{
  "message": "Logged out successfully"
}
```

---

### Step 7: Verify refresh fails after logout

```bash
curl -s -X POST http://localhost:8080/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$NEW_REFRESH_TOKEN\"}" | python3 -m json.tool
```

```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Refresh token has been revoked — please log in again",
  "path": "/auth/refresh"
}
```
✅ Refresh token revoked by logout — cannot get new tokens

---

### Step 8: Access token still works briefly after logout

```bash
# Access token is still valid (JWT is stateless — no revocation)
# This is the inherent JWT trade-off
curl -s http://localhost:8080/api/profile \
  -H "Authorization: Bearer $NEW_ACCESS_TOKEN" | python3 -m json.tool
```

```json
{
  "id": 2,
  "username": "john",
  "email": "john@example.com",
  "role": "ROLE_USER"
}
```

⚠️ This is expected and by design. The access token expires in 15 minutes. The refresh token is revoked, so no new access tokens can be obtained. Damage window is bounded.

---

## Section 10 — Interview Summary

| Question | Answer |
|----------|--------|
| Why use short-lived access tokens? | Limits damage window if a token is stolen — attacker access is time-bounded |
| Why store refresh tokens in the database? | Enables revocation — impossible with stateless JWT alone |
| What is refresh token rotation? | Each refresh operation invalidates the old refresh token and issues a new one — detects stolen tokens |
| Can you revoke a JWT access token immediately? | Not without a denylist (e.g., Redis). JWT is stateless — valid until expiry by design |
| What happens to the access token after logout? | It remains valid until natural expiry — logout only revokes the refresh token |
| Why use UUID for refresh tokens instead of JWT? | UUIDs are revocable (stored in DB), smaller, contain no sensitive data, unaffected by key rotation |
| What is the `jti` claim used for? | JWT ID — unique identifier per token, used to denylist specific tokens in Redis |
| What does `@EnableScheduling` do? | Activates `@Scheduled` method processing — without it, cleanup jobs never run |
| Why implement refresh token rotation even though it adds complexity? | Provides detection of token theft — using a rotated token triggers failure for the real user |

---

## Quick Understanding Check

**Q1.** A user logs in from three different devices — phone, laptop, and tablet. Each device gets its own refresh token. The user's phone is stolen.

Walk through exactly what the user should do and what happens in the database:
- Which endpoint do they call?
- What SQL runs?
- Which tokens are affected?
- Can the thief still use the phone's refresh token after this action?
- Can the thief use the phone's current access token?

**Q2.** Your security team detects suspicious activity — a user's refresh token was used from an unusual IP. You implement refresh token rotation. Explain precisely how rotation helps detect this situation in the future, and what limitation it has if the attacker uses the stolen token before the legitimate user does.

**Q3.** A developer proposes this logout implementation:

```java
@PostMapping("/logout")
public ResponseEntity<?> logout(
        @AuthenticationPrincipal User user
) {
    // Just tell the client to delete their token
    return ResponseEntity.ok(Map.of("message", "Delete your token"));
}
```

List every security problem with this approach compared to our implementation. Be specific about what an attacker can do that they could not do with our implementation.

---
