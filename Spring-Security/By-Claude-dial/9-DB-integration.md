# Spring Security Masterclass — Step 9: Database Integration

---

## 9.1 What This Section Builds

Every previous section used hardcoded users in a `switch` statement. That is not a real application. This section replaces it entirely.

```
BEFORE (hardcoded):
  loadUserByUsername("john") → switch case → new User(...)

AFTER (database):
  loadUserByUsername("john") → SELECT * FROM users WHERE username = ?
                             → map ResultSet to User entity
                             → return UserDetails

NEW CAPABILITIES:
  POST /auth/register  → hash password → INSERT INTO users → return JWT
  POST /auth/login     → SELECT user → verify BCrypt → return JWT
  Users persist across restarts
  Usernames are unique (enforced at DB level)
  Passwords never stored in plaintext
```

---

## 9.2 Dependencies

Add to `pom.xml`:

```xml
<dependencies>

    <!-- Spring Data JPA — repository pattern, entity management -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL driver — connects Java to PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- H2 — in-memory database for tests only -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Validation — @NotBlank, @Email, @Size annotations -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

</dependencies>
```

---

## 9.3 Database Configuration

```properties
# src/main/resources/application.properties

# ── JWT Configuration ──────────────────────────────────────────────────
app.jwt.secret=mNXq3z7Kp1rYvWwJ2bFhDcEoGsAtUiLm9nPeRkVjHyI=
app.jwt.expiration=3600000

# ── PostgreSQL Connection ──────────────────────────────────────────────
spring.datasource.url=jdbc:postgresql://localhost:5432/securitydemo
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.datasource.driver-class-name=org.postgresql.Driver

# ── JPA / Hibernate ────────────────────────────────────────────────────
# create-drop: creates schema on startup, drops on shutdown
# update:      updates schema without dropping data
# validate:    validates schema without changing it
# none:        does nothing (production setting)
spring.jpa.hibernate.ddl-auto=create-drop

# Show SQL in logs — useful during development
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Database dialect — tells Hibernate which SQL flavour to use
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect

# ── Logging ────────────────────────────────────────────────────────────
logging.level.com.example.demo=DEBUG
logging.level.org.springframework.security=DEBUG
```

### Start PostgreSQL with Docker (simplest approach):

```bash
docker run --name security-demo-db \
  -e POSTGRES_DB=securitydemo \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  -d postgres:15
```

Verify it is running:

```bash
docker ps
# Should show security-demo-db running

docker exec -it security-demo-db psql -U postgres -d securitydemo -c "\dt"
# Shows tables after first app startup
```

---

## 9.4 The User Entity

This replaces the plain `User` class from Section 8. It is now a JPA entity — mapped to a database table.

```java
// src/main/java/com/example/demo/model/User.java

package com.example.demo.model;

import jakarta.persistence.*;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.time.LocalDateTime;
import java.util.Collection;
import java.util.List;

@Entity
@Table(
    name = "users",
    uniqueConstraints = {
        // Database-level unique constraint on username
        // Prevents duplicate usernames even under concurrent inserts
        @UniqueConstraint(columnNames = "username"),
        @UniqueConstraint(columnNames = "email")
    }
)
public class User implements UserDetails {

    // ── Primary Key ────────────────────────────────────────────────────
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    // IDENTITY = database auto-increment (SERIAL in PostgreSQL)
    // Hibernate does not generate the ID — the DB does on INSERT
    private Long id;

    // ── Columns ────────────────────────────────────────────────────────
    @Column(nullable = false, unique = true)
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    private String username;

    @Column(nullable = false)
    @NotBlank(message = "Password is required")
    // No @Size here — stored as BCrypt hash (always 60 chars)
    private String password;

    @Column(nullable = false, unique = true)
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;

    @Column(nullable = false)
    // Stores "ROLE_USER" or "ROLE_ADMIN"
    // In production: separate Role entity with @ManyToMany
    // We keep it simple for this course
    private String role;

    @Column(nullable = false)
    private boolean enabled = true;

    @Column(nullable = false)
    private boolean accountNonLocked = true;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // ── JPA Lifecycle Callbacks ────────────────────────────────────────
    // Called automatically by JPA before INSERT
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    // Called automatically by JPA before UPDATE
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // ── Constructors ───────────────────────────────────────────────────
    // JPA requires a no-arg constructor
    protected User() {}

    // Builder-style constructor for creating new users
    public User(String username, String password, String email, String role) {
        this.username = username;
        this.password = password;
        this.email = email;
        this.role = role;
    }

    // ── UserDetails Implementation ─────────────────────────────────────
    // These methods are called by Spring Security during authentication

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority(role));
    }

    @Override
    public String getPassword() { return password; }

    @Override
    public String getUsername() { return username; }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return accountNonLocked; }

    @Override
    public boolean isEnabled() { return enabled; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    // ── Custom Getters ─────────────────────────────────────────────────
    public Long getId() { return id; }
    public String getEmail() { return email; }
    public String getRole() { return role; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }

    // ── Setters ────────────────────────────────────────────────────────
    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
    public void setEmail(String email) { this.email = email; }
    public void setRole(String role) { this.role = role; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public void setAccountNonLocked(boolean accountNonLocked) {
        this.accountNonLocked = accountNonLocked;
    }
}
```

**What JPA annotations mean here:**

```
@Entity          → Hibernate manages this class as a database table
@Table           → customizes table name and constraints
@Id              → marks the primary key field
@GeneratedValue  → tells JPA how to generate the primary key value
@Column          → customizes column properties (name, nullable, unique)
@PrePersist      → method called by JPA BEFORE first INSERT
@PreUpdate       → method called by JPA BEFORE every UPDATE
```

---

## 9.5 The User Repository

```java
// src/main/java/com/example/demo/repository/UserRepository.java

package com.example.demo.repository;

import com.example.demo.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Spring Data JPA generates the SQL from the method name:
    // SELECT * FROM users WHERE username = ?
    Optional<User> findByUsername(String username);

    // SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // SELECT COUNT(*) > 0 FROM users WHERE username = ?
    boolean existsByUsername(String username);

    // SELECT COUNT(*) > 0 FROM users WHERE email = ?
    boolean existsByEmail(String email);
}
```

**How Spring Data JPA generates queries from method names:**

```
findByUsername(String username)
  → "find" + "By" + "Username"
  → SELECT * FROM users WHERE username = ?

findByEmailAndEnabled(String email, boolean enabled)
  → SELECT * FROM users WHERE email = ? AND enabled = ?

findByRoleOrderByCreatedAtDesc(String role)
  → SELECT * FROM users WHERE role = ? ORDER BY created_at DESC

existsByUsername(String username)
  → SELECT COUNT(*) > 0 FROM users WHERE username = ?
```

No SQL needed. No implementation class needed. Spring Data JPA generates everything at startup.

---

## 9.6 Updated `UserDetailsServiceImpl`

Replace the hardcoded switch statement with a real database lookup:

```java
// src/main/java/com/example/demo/security/UserDetailsServiceImpl.java

package com.example.demo.security;

import com.example.demo.repository.UserRepository;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepository userRepository;

    public UserDetailsServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Transactional(readOnly = true)
    // readOnly = true → hint to JPA: no writes happening
    // Hibernate skips dirty checking → slightly faster
    // Database may use read replica if configured
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        // Single database query
        // SELECT * FROM users WHERE username = ?
        return userRepository
                .findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException(
                    "User not found: " + username
                ));
        // Our User entity implements UserDetails
        // so it can be returned directly — no mapping needed
    }
}
```

**Before vs After:**

```
BEFORE (hardcoded):
  loadUserByUsername("john")
    → switch("john") → case "john" → new User(...)
    → O(1) but fake

AFTER (database):
  loadUserByUsername("john")
    → SELECT * FROM users WHERE username = 'john'
    → Hibernate maps ResultSet to User entity
    → return User
    → Real data, persists across restarts
```

---

## 9.7 Registration Flow

### Request/Response DTOs:

```java
// src/main/java/com/example/demo/dto/RegisterRequest.java

package com.example.demo.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public class RegisterRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    private String username;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    @NotBlank(message = "Email is required")
    @Email(message = "Must be a valid email")
    private String email;

    // Constructors
    public RegisterRequest() {}

    public RegisterRequest(String username, String password, String email) {
        this.username = username;
        this.password = password;
        this.email = email;
    }

    // Getters and Setters
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public String getEmail() { return email; }
    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
    public void setEmail(String email) { this.email = email; }
}
```

### Updated `AuthService` with registration:

```java
// src/main/java/com/example/demo/service/AuthService.java

package com.example.demo.service;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.dto.RegisterRequest;
import com.example.demo.model.User;
import com.example.demo.repository.UserRepository;
import com.example.demo.security.JwtService;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;

    public AuthService(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder,
            JwtService jwtService,
            AuthenticationManager authenticationManager,
            UserDetailsService userDetailsService
    ) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.jwtService = jwtService;
        this.authenticationManager = authenticationManager;
        this.userDetailsService = userDetailsService;
    }

    // ── Registration ───────────────────────────────────────────────────
    @Transactional
    public AuthResponse register(RegisterRequest request) {

        // Step 1: Check username uniqueness
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new IllegalArgumentException(
                "Username already taken: " + request.getUsername()
            );
        }

        // Step 2: Check email uniqueness
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new IllegalArgumentException(
                "Email already registered: " + request.getEmail()
            );
        }

        // Step 3: Hash the password
        // NEVER store plaintext passwords
        // BCryptPasswordEncoder.encode() generates a new salt each time
        // The salt is embedded in the hash — matches() extracts it automatically
        String hashedPassword = passwordEncoder.encode(request.getPassword());

        // Step 4: Create and save user
        User user = new User(
            request.getUsername(),
            hashedPassword,         // hashed — never plaintext
            request.getEmail(),
            "ROLE_USER"             // default role for new registrations
        );

        User savedUser = userRepository.save(user);
        // @Transactional ensures: if save fails, nothing is committed

        // Step 5: Generate JWT for immediate use
        // User is registered AND logged in — no second login step needed
        String token = jwtService.generateToken(savedUser);

        return new AuthResponse(token, jwtService.getExpirationTime());
    }

    // ── Login ──────────────────────────────────────────────────────────
    @Transactional(readOnly = true)
    public AuthResponse login(LoginRequest request) {

        // Step 1: Verify credentials via AuthenticationManager
        // Internally: loads user from DB, verifies BCrypt hash
        // Throws BadCredentialsException on failure
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        // Step 2: Load user to generate token
        UserDetails userDetails = userDetailsService
            .loadUserByUsername(request.getUsername());

        // Step 3: Generate and return JWT
        String token = jwtService.generateToken(userDetails);
        return new AuthResponse(token, jwtService.getExpirationTime());
    }
}
```

**Why is `@Transactional` important for registration?**

```
Without @Transactional:
  userRepository.save(user) runs in its own transaction
  If something fails AFTER save but BEFORE method returns
  → user is saved but the response fails
  → Inconsistent state

With @Transactional:
  The entire method runs in ONE transaction
  If ANY exception occurs → entire transaction rolls back
  → user NOT saved → database stays clean
  → No orphaned records
```

---

## 9.8 Updated Auth Controller

```java
// src/main/java/com/example/demo/controller/AuthController.java

package com.example.demo.controller;

import com.example.demo.dto.AuthResponse;
import com.example.demo.dto.LoginRequest;
import com.example.demo.dto.RegisterRequest;
import com.example.demo.service.AuthService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    // ── Register ───────────────────────────────────────────────────────
    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(
            @Valid @RequestBody RegisterRequest request
            // @Valid triggers validation annotations on RegisterRequest
            // If validation fails → MethodArgumentNotValidException
            // → GlobalExceptionHandler catches → 400 Bad Request
    ) {
        AuthResponse response = authService.register(request);
        return ResponseEntity
            .status(HttpStatus.CREATED)  // 201 Created — user was created
            .body(response);
    }

    // ── Login ──────────────────────────────────────────────────────────
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(
            @Valid @RequestBody LoginRequest request
    ) {
        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(response);  // 200 OK
    }
}
```

---

## 9.9 Handling Validation Errors

Add validation error handling to `GlobalExceptionHandler`:

```java
// Add to GlobalExceptionHandler.java

import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;

// ── Validation failures (@Valid) ───────────────────────────────────────
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, Object>> handleValidationErrors(
        MethodArgumentNotValidException ex,
        HttpServletRequest request
) {
    // Collect all field validation errors into a map
    Map<String, String> fieldErrors = new HashMap<>();
    ex.getBindingResult()
      .getAllErrors()
      .forEach(error -> {
          String fieldName = ((FieldError) error).getField();
          String message = error.getDefaultMessage();
          fieldErrors.put(fieldName, message);
      });

    Map<String, Object> body = Map.of(
        "timestamp", LocalDateTime.now().toString(),
        "status", 400,
        "error", "Bad Request",
        "message", "Validation failed",
        "errors", fieldErrors,
        "path", request.getServletPath()
    );

    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(body);
}

// ── Duplicate username/email ───────────────────────────────────────────
@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<Map<String, Object>> handleIllegalArgument(
        IllegalArgumentException ex,
        HttpServletRequest request
) {
    return buildErrorResponse(
        HttpStatus.CONFLICT,   // 409 Conflict — resource already exists
        ex.getMessage(),
        request.getServletPath()
    );
}
```

---

## 9.10 Data Initialization — Creating an Admin User

We need an admin user for testing. Use Spring's `ApplicationRunner`:

```java
// src/main/java/com/example/demo/config/DataInitializer.java

package com.example.demo.config;

import com.example.demo.model.User;
import com.example.demo.repository.UserRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

@Component
public class DataInitializer implements ApplicationRunner {

    private static final Logger log = LoggerFactory.getLogger(DataInitializer.class);

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public DataInitializer(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder
    ) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public void run(ApplicationArguments args) {

        // Create admin user if not already present
        if (!userRepository.existsByUsername("admin")) {
            User admin = new User(
                "admin",
                passwordEncoder.encode("admin123"),
                "admin@example.com",
                "ROLE_ADMIN"
            );
            userRepository.save(admin);
            log.info("Admin user created");
        }

        // Create a test user
        if (!userRepository.existsByUsername("john")) {
            User john = new User(
                "john",
                passwordEncoder.encode("secret123"),
                "john@example.com",
                "ROLE_USER"
            );
            userRepository.save(john);
            log.info("Test user 'john' created");
        }

        log.info("Data initialization complete. Users in DB: {}",
            userRepository.count());
    }
}
```

**Why `ApplicationRunner` instead of `@PostConstruct`?**

```
@PostConstruct:
  Runs during bean creation — before Spring context fully starts
  Database might not be ready
  Transactions might not work correctly

ApplicationRunner.run():
  Runs AFTER Spring context is fully started
  Database is connected and ready
  All beans are initialized
  Transactions work correctly
  Correct choice for data seeding
```

---

## 9.11 Complete Database Flow Diagram

```
POST /auth/register
{ "username": "alice", "password": "pass1234", "email": "alice@ex.com" }

        │
        ▼
AuthController.register(@Valid RegisterRequest)
  │
  ├── @Valid triggers: username not blank ✓, password ≥ 8 chars ✓, email valid ✓
  │
  ▼
AuthService.register(request)
  │
  ├── userRepository.existsByUsername("alice")
  │     → SELECT COUNT(*) FROM users WHERE username = 'alice'
  │     → 0 → username available ✓
  │
  ├── userRepository.existsByEmail("alice@ex.com")
  │     → SELECT COUNT(*) FROM users WHERE email = 'alice@ex.com'
  │     → 0 → email available ✓
  │
  ├── passwordEncoder.encode("pass1234")
  │     → BCrypt.hash("pass1234", salt)
  │     → "$2a$12$X9xV8Y..." (60-char hash)
  │
  ├── new User("alice", "$2a$12$X9xV8Y...", "alice@ex.com", "ROLE_USER")
  │
  ├── userRepository.save(user)
  │     → INSERT INTO users (username, password, email, role, enabled, ...)
  │     →  VALUES ('alice', '$2a$12$X9xV8Y...', 'alice@ex.com', 'ROLE_USER', true, ...)
  │     → RETURNING id → user.id = 3
  │
  ├── jwtService.generateToken(savedUser)
  │     → sub: "alice", iat: now, exp: now+1h
  │     → sign with HMAC-SHA256
  │     → "eyJhbGci..."
  │
  └── return AuthResponse("eyJhbGci...", 3600000)

        │
        ▼
201 Created
{
  "token": "eyJhbGci...",
  "tokenType": "Bearer",
  "expiresIn": 3600000
}
```

---

## Hands-On: Complete Test Sequence

### Start the application and verify database:

```bash
mvn spring-boot:run

# In another terminal, verify table was created:
docker exec -it security-demo-db \
  psql -U postgres -d securitydemo \
  -c "SELECT id, username, email, role, enabled FROM users;"
```

Expected output:
```
 id | username |       email        |   role    | enabled
----+----------+--------------------+-----------+---------
  1 | admin    | admin@example.com  | ROLE_ADMIN| t
  2 | john     | john@example.com   | ROLE_USER | t
(2 rows)
```

---

### Test 1: Register a new user

```bash
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "pass1234",
    "email": "alice@example.com"
  }' | python3 -m json.tool
```

**Expected: 201 Created**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600000
}
```

Verify in database:
```bash
docker exec -it security-demo-db \
  psql -U postgres -d securitydemo \
  -c "SELECT id, username, email, role FROM users;"
```
```
 id | username |        email         |   role
----+----------+----------------------+-----------
  1 | admin    | admin@example.com    | ROLE_ADMIN
  2 | john     | john@example.com     | ROLE_USER
  3 | alice    | alice@example.com    | ROLE_USER
```

---

### Test 2: Try to register duplicate username

```bash
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "different1",
    "email": "different@example.com"
  }' | python3 -m json.tool
```

**Expected: 409 Conflict**
```json
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 409,
  "error": "Conflict",
  "message": "Username already taken: alice",
  "path": "/auth/register"
}
```

---

### Test 3: Register with invalid data (validation)

```bash
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "ab",
    "password": "short",
    "email": "not-an-email"
  }' | python3 -m json.tool
```

**Expected: 400 Bad Request**
```json
{
  "timestamp": "2024-01-15T10:30:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "errors": {
    "username": "Username must be 3-50 characters",
    "password": "Password must be at least 8 characters",
    "email": "Must be a valid email"
  },
  "path": "/auth/register"
}
```

---

### Test 4: Login with registered user

```bash
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pass1234"}' \
  | python3 -m json.tool
```

**Expected: 200 OK**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600000
}
```

---

### Test 5: Use alice's token to access protected endpoint

```bash
ALICE_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pass1234"}' \
  | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

curl -s http://localhost:8080/api/profile \
  -H "Authorization: Bearer $ALICE_TOKEN" | python3 -m json.tool
```

**Expected: 200 OK**
```json
{
  "id": 3,
  "username": "alice",
  "email": "alice@example.com",
  "role": "ROLE_USER",
  "authorities": "[ROLE_USER]"
}
```

---

### Test 6: Verify password is hashed in database

```bash
docker exec -it security-demo-db \
  psql -U postgres -d securitydemo \
  -c "SELECT username, password FROM users WHERE username = 'alice';"
```

```
 username |                           password
----------+--------------------------------------------------------------
 alice    | $2a$12$X9xV8YmK3...long BCrypt hash...
```

✅ Never plaintext — always BCrypt hash starting with `$2a$12$`

---

### Test 7: Restart app — data persists

```bash
# Stop the application (Ctrl+C)
# Restart it
mvn spring-boot:run

# Login again — alice still exists
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pass1234"}'
```

✅ Login succeeds — data survived restart because it is in PostgreSQL, not memory.

---

## Section 9 — Interview Summary

| Question | Answer |
|----------|--------|
| Why does `User` implement `UserDetails` directly? | Eliminates a mapping layer — JPA entity serves directly as the Spring Security principal |
| Why use `@GeneratedValue(strategy = GenerationType.IDENTITY)`? | Delegates ID generation to the database (PostgreSQL SERIAL) — no application-side ID generation |
| What does `@PrePersist` do? | JPA calls this method automatically before the first INSERT — used to set `createdAt` |
| Why is `@Transactional` on `register()`? | Wraps all DB operations in one transaction — rolls back everything if any step fails |
| Why check `existsByUsername` before saving? | Provides a clear error message. Without it, a unique constraint violation gives a cryptic DB error |
| What does `spring.jpa.hibernate.ddl-auto=create-drop` do? | Creates schema on startup, drops on shutdown — for development only, never production |
| Why use `ApplicationRunner` for data seeding rather than `@PostConstruct`? | `ApplicationRunner` executes after the full Spring context starts, ensuring DB and transactions are ready |
| How does Spring Data JPA generate SQL from method names? | Parses the method name at startup — `findByUsername` → `SELECT * FROM users WHERE username = ?` |

---

## Quick Understanding Check

**Q1.** A user registers successfully. Ten minutes later they try to log in and get 401. The password is correct. Walk through every possible cause — naming the exact class and method where each failure would occur.

**Q2.** You have this in `UserDetailsServiceImpl`:

```java
return userRepository
    .findByUsername(username)
    .orElseThrow(() -> new UsernameNotFoundException(
        "User not found: " + username
    ));
```

During login, Spring Security catches `UsernameNotFoundException` internally and converts it to `BadCredentialsException` before throwing to the caller. Why does Spring do this conversion? What security property does it protect?

**Q3.** A developer suggests storing the role as a separate `@Entity` with `@ManyToMany` between `User` and `Role`. Sketch what that would look like. What are the advantages over storing role as a plain `String` column? What complexity does it add?
