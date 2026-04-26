# Spring Security Masterclass — Step 4: Generating and Validating JWT

---

## 4.1 Setting Up the JJWT Library

We use the **jjwt** library by io.jsonwebtoken. It is the industry standard for JWT in Java.

**⚠️ Version Pitfall — Read This First:**

```
jjwt < 0.10  → Old API (JwtBuilder, parser() method)  — DO NOT USE
jjwt >= 0.11 → New API (completely redesigned)         — USE THIS
jjwt 0.12.x  → Latest, Spring Boot 3.x compatible     — WE USE THIS
```

If you find tutorials using `Jwts.parser()` without a builder pattern, or `Keys.hmacShaKeyFor()` with a String directly — they are using the old deprecated API. We use the current API exclusively.

### Add dependencies to `pom.xml`:

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

    <!-- JJWT API — the interfaces you code against -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>

    <!-- JJWT Implementation — runtime, not compile time -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>

    <!-- JJWT Jackson — for JSON serialization of claims -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**Why three separate jars?**

```
jjwt-api      → Interfaces + classes you reference in code (compile scope)
jjwt-impl     → Actual implementation (runtime only — you never import from here)
jjwt-jackson  → Uses Jackson to serialize/deserialize JWT claims as JSON
```

This separation means you code against the API, and the implementation can change without affecting your code.

---

## 4.2 The Signing Key — Most Critical Concept

Before writing any JWT code, understand the signing key deeply.

### What is the signing key?

For `HS256` (HMAC-SHA256), the signing key is a **secret byte array** used to:
1. **Sign** the token when generating
2. **Verify** the signature when validating

Same key does both — this is symmetric cryptography.

### Key Requirements for HS256:

```
Minimum key length for HS256 = 256 bits = 32 bytes

If your key is shorter → jjwt 0.12.x throws WeakKeyException
If your key is too short → signature is cryptographically weak
```

### How to generate a secure key:

**Option 1: Let jjwt generate one (best for development):**
```java
SecretKey key = Jwts.SIG.HS256.key().build();
// Prints a base64 encoded key you can store in config
String base64Key = Base64.getEncoder().encodeToString(key.getEncoded());
System.out.println(base64Key);
```

**Option 2: Generate via command line (best for production):**
```bash
# Generates a 256-bit (32-byte) random key, base64 encoded
openssl rand -base64 32
# Output example: mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
```

**Option 3: Hard-code for learning (NEVER in production):**
```java
// At least 32 characters for 256-bit minimum
String secret = "mySecretKeyThatIsAtLeast32CharactersLong!";
```

### Store the key in `application.properties`:

```properties
# application.properties
app.jwt.secret=mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
app.jwt.expiration=3600000
# expiration is in milliseconds: 3600000 = 1 hour
```

**⚠️ Production rule:** Never commit secrets to Git. Use environment variables or a secrets manager (AWS Secrets Manager, HashiCorp Vault).

---

## 4.3 Understanding What We Are Building

Before writing code, see the complete picture:

```
JwtService
    │
    ├── generateToken(UserDetails)     → String (JWT)
    │       │
    │       ├── builds claims (sub, iat, exp)
    │       ├── signs with SECRET_KEY
    │       └── returns compact JWT string
    │
    ├── validateToken(token, UserDetails) → boolean
    │       │
    │       ├── extractUsername(token) → must match UserDetails.username
    │       └── isTokenExpired(token)  → must be false
    │
    ├── extractUsername(token)         → String
    │       └── extracts "sub" claim
    │
    └── extractExpiration(token)       → Date
            └── extracts "exp" claim
```

---

## 4.4 Building the JWT Service

```java
// src/main/java/com/example/demo/security/JwtService.java

package com.example.demo.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Base64;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class JwtService {

    // Injected from application.properties
    @Value("${app.jwt.secret}")
    private String secretKey;

    @Value("${app.jwt.expiration}")
    private long jwtExpiration;

    // ================================================================
    // PUBLIC API
    // ================================================================

    /**
     * Generate a JWT token for a given user.
     * Uses username as subject, adds no extra claims.
     */
    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }

    /**
     * Generate a JWT token with extra custom claims.
     * Use this when you want to embed roles, userId, etc.
     */
    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return buildToken(extraClaims, userDetails, jwtExpiration);
    }

    /**
     * Validate token against a specific user.
     * Returns true only if:
     *   1. Token username matches the UserDetails username
     *   2. Token is not expired
     */
    public boolean validateToken(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername())) && !isTokenExpired(token);
    }

    /**
     * Extract the username (subject) from the token.
     */
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    // ================================================================
    // PRIVATE INTERNALS
    // ================================================================

    /**
     * Core token builder.
     * This is where the JWT is actually assembled and signed.
     */
    private String buildToken(
            Map<String, Object> extraClaims,
            UserDetails userDetails,
            long expiration
    ) {
        return Jwts.builder()
                // Custom claims FIRST (so standard claims below can override if needed)
                .claims(extraClaims)

                // Standard claims
                .subject(userDetails.getUsername())           // "sub" claim
                .issuedAt(new Date(System.currentTimeMillis())) // "iat" claim
                .expiration(new Date(System.currentTimeMillis() + expiration)) // "exp" claim

                // Sign with our secret key using HS256
                .signWith(getSigningKey())

                // Serialize to compact JWT string: header.payload.signature
                .compact();
    }

    /**
     * Generic claim extractor using a Function resolver.
     * This pattern lets us reuse one method to extract ANY claim.
     *
     * Example:
     *   extractClaim(token, Claims::getSubject)     → username
     *   extractClaim(token, Claims::getExpiration)  → expiry date
     */
    private <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    /**
     * Parse the JWT and extract ALL claims.
     * This is where signature verification happens.
     *
     * If the signature is invalid → JwtException thrown
     * If the token is expired    → ExpiredJwtException thrown
     */
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                // Tell the parser which key to use for verification
                .verifyWith(getSigningKey())
                .build()
                // Parse and verify in one step
                .parseSignedClaims(token)
                // Get the payload (claims)
                .getPayload();
    }

    /**
     * Check if the token's expiration date is before now.
     */
    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    /**
     * Extract the expiration date from the token.
     */
    private Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    /**
     * Build the SecretKey object from our Base64-encoded secret string.
     *
     * Keys.hmacShaKeyFor() requires raw bytes.
     * We decode the Base64 string back to bytes first.
     *
     * jjwt 0.12.x will throw WeakKeyException if bytes < 32 (256 bits).
     */
    private SecretKey getSigningKey() {
        byte[] keyBytes = Base64.getDecoder().decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

---

## 4.5 Deep Dive: Every Line Explained

### Why `Function<Claims, T> claimsResolver`?

```java
// Without this pattern, you'd need separate methods:
private String extractSubject(String token) { ... }
private Date extractExpiration(String token) { ... }
private String extractIssuer(String token) { ... }
// Lots of repetition

// WITH the Function pattern — one method handles all:
extractClaim(token, Claims::getSubject)    // extracts username
extractClaim(token, Claims::getExpiration) // extracts expiry
extractClaim(token, Claims::getIssuer)     // extracts issuer

// You can even extract custom claims:
extractClaim(token, claims -> claims.get("userId", Long.class))
```

### What happens inside `extractAllClaims()`?

```
token = "eyJhbGci...header.eyJzdWIi...payload.SflKxw...signature"
                                │
                                ▼
Jwts.parser()
    .verifyWith(signingKey)     ← registers the key for verification
    .build()                    ← creates the JwtParser
    .parseSignedClaims(token)   ← THIS does the heavy lifting:
                                    1. Splits token into 3 parts
                                    2. Decodes header → checks algorithm
                                    3. Recalculates expected signature
                                    4. Compares with received signature
                                    5. If mismatch → throws JwtException
                                    6. Checks exp claim → if expired → throws ExpiredJwtException
                                    7. Returns the verified Jws<Claims> object
    .getPayload()               ← extracts the Claims map from verified token
```

### Why `Base64.getDecoder().decode(secretKey)` and not just `.getBytes()`?

```java
// WRONG — common beginner mistake:
byte[] keyBytes = secretKey.getBytes(); 
// If secretKey = "mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI="
// .getBytes() gives you the bytes of the BASE64 STRING (44 bytes of ASCII chars)
// This is NOT the same as the original 32 bytes the key represents

// CORRECT:
byte[] keyBytes = Base64.getDecoder().decode(secretKey);
// This decodes Base64 back to the original 32 raw bytes
// These 32 bytes = 256 bits = valid HS256 key
```

---

## 4.6 Adding UserDetails — Required for validateToken

`JwtService.validateToken()` takes a `UserDetails` parameter. We need a `UserDetailsService` to load user data. Let us create a minimal in-memory one for now (we will replace this with a real database version later):

```java
// src/main/java/com/example/demo/security/UserDetailsServiceImpl.java

package com.example.demo.security;

import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        // Hardcoded for now — Section 7 replaces this with real DB
        if ("john".equals(username)) {
            return User.builder()
                    .username("john")
                    // BCrypt hash of "secret123"
                    .password(new BCryptPasswordEncoder().encode("secret123"))
                    .roles("USER")
                    .build();
        }

        if ("admin".equals(username)) {
            return User.builder()
                    .username("admin")
                    .password(new BCryptPasswordEncoder().encode("admin123"))
                    .roles("USER", "ADMIN")
                    .build();
        }

        throw new UsernameNotFoundException("User not found: " + username);
    }
}
```

---

## 4.7 application.properties

```properties
# application.properties

# JWT Configuration
app.jwt.secret=mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
app.jwt.expiration=3600000

# Show security debug logs
logging.level.org.springframework.security=DEBUG
```

---

## Hands-On: Generate and Validate Token in Code

Create a test class to see JWT generation and validation in action **before** wiring it into Spring Security:

```java
// src/test/java/com/example/demo/security/JwtServiceTest.java

package com.example.demo.security;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Base64;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

class JwtServiceTest {

    private JwtService jwtService;
    private UserDetails userDetails;

    @BeforeEach
    void setUp() {
        jwtService = new JwtService();

        // Inject values manually (normally Spring does this via @Value)
        // Use reflection or a setter — let's use a helper approach:
        setField(jwtService, "secretKey",
                Base64.getEncoder().encodeToString(
                    "mySecretKeyThatIsAtLeast32CharactersLong!!".getBytes()
                ));
        setField(jwtService, "jwtExpiration", 3600000L);

        // Create a test user
        userDetails = User.builder()
                .username("john")
                .password("doesNotMatterForTokenTests")
                .roles("USER")
                .build();
    }

    @Test
    void shouldGenerateToken() {
        String token = jwtService.generateToken(userDetails);

        System.out.println("=== GENERATED TOKEN ===");
        System.out.println(token);
        System.out.println();

        // Decode and print manually
        String[] parts = token.split("\\.");
        System.out.println("HEADER  : " + new String(Base64.getUrlDecoder().decode(parts[0])));
        System.out.println("PAYLOAD : " + new String(Base64.getUrlDecoder().decode(parts[1])));
        System.out.println("SIG     : " + parts[2]);

        assertNotNull(token);
        assertEquals(3, parts.length); // Must have 3 parts
    }

    @Test
    void shouldExtractUsername() {
        String token = jwtService.generateToken(userDetails);
        String extractedUsername = jwtService.extractUsername(token);

        System.out.println("Extracted username: " + extractedUsername);
        assertEquals("john", extractedUsername);
    }

    @Test
    void shouldValidateToken() {
        String token = jwtService.generateToken(userDetails);
        boolean isValid = jwtService.validateToken(token, userDetails);

        System.out.println("Token valid: " + isValid);
        assertTrue(isValid);
    }

    @Test
    void shouldRejectTamperedToken() {
        String token = jwtService.generateToken(userDetails);
        String[] parts = token.split("\\.");

        // Tamper with the payload
        String maliciousPayload = Base64.getUrlEncoder()
                .withoutPadding()
                .encodeToString("{\"sub\":\"admin\",\"roles\":[\"ROLE_ADMIN\"]}".getBytes());

        String tamperedToken = parts[0] + "." + maliciousPayload + "." + parts[2];

        System.out.println("Testing tampered token...");

        // Should throw an exception during parsing
        assertThrows(Exception.class, () -> {
            jwtService.extractUsername(tamperedToken);
        });

        System.out.println("Tampered token correctly rejected!");
    }

    @Test
    void shouldRejectWrongUser() {
        String token = jwtService.generateToken(userDetails);

        // Create a different user
        UserDetails anotherUser = User.builder()
                .username("alice")
                .password("pass")
                .roles("USER")
                .build();

        boolean isValid = jwtService.validateToken(token, anotherUser);

        System.out.println("Token valid for wrong user: " + isValid);
        assertFalse(isValid); // john's token must not validate for alice
    }

    @Test
    void shouldGenerateTokenWithExtraClaims() {
        // Add custom claims — roles, userId
        java.util.Map<String, Object> extraClaims = new java.util.HashMap<>();
        extraClaims.put("userId", 42);
        extraClaims.put("roles", List.of("ROLE_USER"));

        String token = jwtService.generateToken(extraClaims, userDetails);
        String[] parts = token.split("\\.");

        System.out.println("Token with extra claims:");
        System.out.println(new String(Base64.getUrlDecoder().decode(parts[1])));

        // Verify username still extractable
        assertEquals("john", jwtService.extractUsername(token));
    }

    // Helper to inject @Value fields in tests
    private void setField(Object target, String fieldName, Object value) {
        try {
            var field = target.getClass().getDeclaredField(fieldName);
            field.setAccessible(true);
            field.set(target, value);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

### Expected Output:

```
=== GENERATED TOKEN ===
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwiaWF0IjoxNzAwMDAwMDAwLCJleHAiOjE3MDAwMDM2MDB9.xxxxx

HEADER  : {"alg":"HS256"}
PAYLOAD : {"sub":"john","iat":1700000000,"exp":1700003600}
SIG     : xxxxx

Extracted username: john
Token valid: true
Testing tampered token...
Tampered token correctly rejected!
Token valid for wrong user: false

Token with extra claims:
{"userId":42,"roles":["ROLE_USER"],"sub":"john","iat":1700000000,"exp":1700003600}
```

---

## 4.8 Exception Handling Reference

When working with JWT, these are the exceptions you will encounter:

```java
try {
    Claims claims = Jwts.parser()
            .verifyWith(signingKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();

} catch (ExpiredJwtException e) {
    // Token expired — exp claim is in the past
    // Common case — handle gracefully, return 401

} catch (UnsupportedJwtException e) {
    // Token format not supported
    // e.g., you got an unsigned token but expected signed

} catch (MalformedJwtException e) {
    // Token string is not valid JWT format
    // e.g., missing a dot separator, corrupt string

} catch (SignatureException e) {
    // Signature verification failed
    // Token was tampered with OR wrong secret key used

} catch (IllegalArgumentException e) {
    // Token string is null or empty

} catch (JwtException e) {
    // Parent of all JWT exceptions — catch-all fallback
}
```

In production, you catch these in your JWT filter and return `401 Unauthorized` for all of them.

---

## Section 4 — Interview Summary

| Question | Answer |
|----------|--------|
| What library do we use for JWT in Java? | `io.jsonwebtoken` (jjwt), version 0.12.x for Spring Boot 3.x |
| What is the minimum key size for HS256? | 256 bits (32 bytes) — jjwt throws `WeakKeyException` if smaller |
| What does `parseSignedClaims()` do internally? | Splits token, recalculates signature, compares, checks expiry, returns claims |
| What exception is thrown for expired tokens? | `ExpiredJwtException` |
| What exception is thrown for tampered tokens? | `SignatureException` |
| Why decode Base64 before calling `Keys.hmacShaKeyFor()`? | The config stores key as Base64 string — we need the original raw bytes |
| What is the `sub` claim set to? | The username (`userDetails.getUsername()`) |
| How do you add custom claims to a token? | Pass a `Map<String, Object>` to `generateToken(extraClaims, userDetails)` |

---

## Quick Understanding Check

**Q1.** You receive a token and call `jwtService.validateToken(token, userDetails)`. It returns `false` but throws no exception. What are the two possible reasons?

**Q2.** A developer writes this code:
```java
byte[] keyBytes = secretKey.getBytes();
SecretKey key = Keys.hmacShaKeyFor(keyBytes);
```
The secret in `application.properties` is:
```
app.jwt.secret=mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
```
What is wrong with this? What does `.getBytes()` return vs `Base64.getDecoder().decode()`?

**Q3.** You want to embed the user's role list in the JWT so your microservices don't need to query the database for roles on every request. Show how you would call `generateToken` to include a `roles` claim. Then show how you would extract that custom claim back.
