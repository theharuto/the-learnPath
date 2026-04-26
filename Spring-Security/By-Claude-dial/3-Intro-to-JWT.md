# Spring Security Masterclass — Step 3: Introduction to JWT

---

## 3.1 What is JWT — Really?

JWT stands for **JSON Web Token**. But forget the name for a second.

Here is the core problem JWT solves:

```
Client authenticates → Server needs to remember "this person is logged in"

Option 1 (Sessions):  Server stores session in memory/database
                       Client sends session ID in cookie
                       Server looks up session on every request
                       → STATEFUL (server must remember)

Option 2 (JWT):       Server creates a signed token with user info embedded
                       Client stores the token
                       Client sends token on every request
                       Server just VERIFIES the signature — no DB lookup needed
                       → STATELESS (server remembers nothing)
```

JWT is not encrypted by default. It is **signed**. The difference is critical:

```
Encrypted  = nobody can READ the content without the key
Signed     = anybody can READ the content, but nobody can TAMPER with it
             without invalidating the signature
```

This means: **never put sensitive data in a JWT** (passwords, credit cards, SSN). The payload is just Base64 encoded — anyone can decode it.

---

## 3.2 JWT Structure — Dissected

A JWT looks like this:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqb2huIiwicm9sZXMiOlsiUk9MRV9VU0VSIl0sImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAzNjAwfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Three parts separated by dots:

```
HEADER.PAYLOAD.SIGNATURE
```

Let me decode each part:

---

### Part 1: Header

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

Base64Url decode this → you get:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

| Field | Meaning |
|-------|---------|
| `alg` | Algorithm used to sign the token (HS256 = HMAC-SHA256) |
| `typ` | Type of token — always "JWT" |

**Common algorithms:**

```
HS256  → HMAC with SHA-256 (symmetric — same secret key to sign and verify)
RS256  → RSA with SHA-256   (asymmetric — private key signs, public key verifies)
ES256  → ECDSA with SHA-256 (asymmetric — more efficient than RSA)
```

For microservices: `RS256` is preferred because the auth server signs with private key, and each microservice only needs the public key to verify. No secret key shared across services.

For our implementation: We'll use `HS256` (simpler, fine for single service).

---

### Part 2: Payload (Claims)

```
eyJzdWIiOiJqb2huIiwicm9sZXMiOlsiUk9MRV9VU0VSIl0sImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAzNjAwfQ
```

Base64Url decode this → you get:

```json
{
  "sub": "john",
  "roles": ["ROLE_USER"],
  "iat": 1700000000,
  "exp": 1700003600
}
```

These are called **claims**. Two types:

**Registered Claims (standard, predefined):**

| Claim | Full Name | Meaning |
|-------|-----------|---------|
| `sub` | Subject | Who the token is about (usually username or user ID) |
| `iat` | Issued At | When token was created (Unix timestamp) |
| `exp` | Expiration | When token expires (Unix timestamp) |
| `iss` | Issuer | Who issued the token (e.g., "my-auth-service") |
| `aud` | Audience | Who the token is intended for |
| `jti` | JWT ID | Unique ID for the token (for revocation) |

**Custom Claims (you define these):**

```json
{
  "sub": "john",
  "roles": ["ROLE_USER", "ROLE_ADMIN"],
  "userId": 42,
  "email": "john@example.com",
  "exp": 1700003600
}
```

**⚠️ Remember:** This payload is just Base64 encoded. Anyone who has the token can decode and read it. Do NOT put passwords, secrets, or sensitive PII here.

---

### Part 3: Signature

```
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

This is the most important part. Here's exactly how it's created:

```
SIGNATURE = HMAC_SHA256(
    Base64Url(header) + "." + Base64Url(payload),
    SECRET_KEY
)
```

In plain English:

```
1. Take the header (encoded)
2. Add a dot
3. Add the payload (encoded)
4. Hash the whole thing using your secret key
5. That hash IS the signature
```

**Why this makes tampering impossible:**

```
Attacker receives JWT:
  header.payload.signature

Attacker decodes payload:
  { "sub": "john", "roles": ["ROLE_USER"] }

Attacker modifies payload:
  { "sub": "john", "roles": ["ROLE_ADMIN"] }  ← changed!

Attacker re-encodes payload and reassembles JWT:
  header.MODIFIED_payload.signature  ← OLD signature still there

Server verifies:
  Recalculates: HMAC_SHA256(header + "." + MODIFIED_payload, SECRET_KEY)
  = completely different hash
  ≠ old signature
  → REJECTED ✓
```

The attacker cannot forge a valid signature without knowing the secret key.

---

## 3.3 Why JWT is Stateless

This is fundamental. Understand this deeply.

### Session-based (Stateful):

```
Login Request:
  Client → POST /login (username, password)
  Server → validates credentials
  Server → creates session: { sessionId: "abc123", userId: 42, roles: ["ROLE_USER"] }
  Server → stores session in memory or Redis
  Server → returns cookie: JSESSIONID=abc123

Every subsequent request:
  Client → GET /api/orders (cookie: JSESSIONID=abc123)
  Server → looks up "abc123" in session store
  Server → finds: { userId: 42, roles: ["ROLE_USER"] }
  Server → processes request

Problems:
  ✗ Server must store session state
  ✗ Multiple servers need shared session store (Redis/DB)
  ✗ Horizontal scaling is complicated
  ✗ Session store becomes a single point of failure
```

### JWT-based (Stateless):

```
Login Request:
  Client → POST /login (username, password)
  Server → validates credentials
  Server → creates JWT: { sub: "john", roles: ["ROLE_USER"], exp: ... }
  Server → signs JWT with secret key
  Server → returns JWT to client
  Server → STORES NOTHING

Every subsequent request:
  Client → GET /api/orders (Authorization: Bearer <jwt>)
  Server → reads JWT
  Server → verifies signature (math operation, no DB)
  Server → reads claims: { sub: "john", roles: ["ROLE_USER"] }
  Server → processes request

Benefits:
  ✓ Server stores zero state
  ✓ Any server in your cluster can verify any token independently
  ✓ Perfect for microservices — each service verifies independently
  ✓ No shared session store needed
```

### Visual comparison for microservices:

```
SESSION-BASED (problematic):
                    ┌─────────────┐
Request ──────────→ │  Service A  │ ──→ Must hit Redis to verify session
Request ──────────→ │  Service B  │ ──→ Must hit Redis to verify session
Request ──────────→ │  Service C  │ ──→ Must hit Redis to verify session
                    └─────────────┘
                           │
                    ┌──────▼──────┐
                    │    Redis    │ ← Single point of failure + latency
                    └─────────────┘

JWT-BASED (scalable):
                    ┌─────────────┐
Request+JWT ──────→ │  Service A  │ ──→ Verifies JWT locally (no network call)
Request+JWT ──────→ │  Service B  │ ──→ Verifies JWT locally (no network call)
Request+JWT ──────→ │  Service C  │ ──→ Verifies JWT locally (no network call)
                    └─────────────┘
                    No shared state needed ✓
```

---

## 3.4 Complete JWT Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION PHASE                      │
└─────────────────────────────────────────────────────────────┘

1. Client sends credentials:
   POST /auth/login
   Body: { "username": "john", "password": "secret123" }

2. Server validates credentials against DB

3. Server generates JWT:
   Header:  { "alg": "HS256", "typ": "JWT" }
   Payload: { "sub": "john", "roles": ["ROLE_USER"], 
              "iat": 1700000000, "exp": 1700003600 }
   Signs with SECRET_KEY → produces signature

4. Server returns:
   { "token": "eyJhbGci...SflKxw" }

5. Client STORES token (localStorage, memory, etc.)

┌─────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION PHASE                       │
└─────────────────────────────────────────────────────────────┘

6. Client sends request with token:
   GET /api/orders
   Authorization: Bearer eyJhbGci...SflKxw

7. Server's JWT filter intercepts:
   a. Extracts token from Authorization header
   b. Splits into header.payload.signature
   c. Recalculates expected signature using SECRET_KEY
   d. Compares with received signature → match ✓
   e. Checks exp claim → not expired ✓
   f. Extracts "sub": "john" and "roles": ["ROLE_USER"]
   g. Creates Authentication object
   h. Puts in SecurityContextHolder

8. AuthorizationFilter checks:
   Does john with ROLE_USER have access to GET /api/orders?
   YES → proceed

9. Controller executes, returns response
```

---

## 3.5 JWT Pitfalls You Must Know

| Pitfall | Problem | Solution |
|---------|---------|----------|
| Long expiry times | Stolen token valid for weeks | Use short expiry (15-60 min) + refresh tokens |
| Sensitive data in payload | Anyone can decode payload | Only store non-sensitive identifiers |
| Weak secret key | Brute-forceable | Use 256-bit+ cryptographically random key |
| No token revocation | Can't invalidate a stolen token | Use short expiry or maintain blocklist |
| Storing JWT in localStorage | XSS attack can steal token | Use httpOnly cookies or memory storage |
| `alg: none` attack | Attacker sets algorithm to none | Always validate the algorithm explicitly |

---

## Hands-On: Decode a JWT Manually

### Step 1: Generate a sample JWT

Go to [jwt.io](https://jwt.io) and create a token with this payload:

```json
{
  "sub": "john",
  "roles": ["ROLE_USER"],
  "iat": 1700000000,
  "exp": 1700003600
}
```

You'll get something like:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJqb2huIiwicm9sZXMiOlsiUk9MRV9VU0VSIl0sImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAzNjAwfQ.8Th0L6KPeq4hXBi3BolBSH6cBHJHnVGkwRqpAMNlr-A
```

### Step 2: Decode it manually in Java

Create a simple main class (no Spring needed):

```java
// src/main/java/com/example/demo/util/JwtDecoder.java

package com.example.demo.util;

import java.util.Base64;

public class JwtDecoder {

    public static void main(String[] args) {
        
        String jwt = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" +
                     ".eyJzdWIiOiJqb2huIiwicm9sZXMiOlsiUk9MRV9VU0VSIl0sImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAzNjAwfQ" +
                     ".8Th0L6KPeq4hXBi3BolBSH6cBHJHnVGkwRqpAMNlr-A";

        // Split into three parts
        String[] parts = jwt.split("\\.");

        System.out.println("=== JWT DECODED ===\n");

        // Decode Header (part 0)
        String header = new String(Base64.getUrlDecoder().decode(parts[0]));
        System.out.println("HEADER:");
        System.out.println(header);

        // Decode Payload (part 1)
        String payload = new String(Base64.getUrlDecoder().decode(parts[1]));
        System.out.println("\nPAYLOAD:");
        System.out.println(payload);

        // Signature (part 2) — binary, don't try to decode as string
        System.out.println("\nSIGNATURE (raw Base64Url):");
        System.out.println(parts[2]);
        System.out.println("(This is a cryptographic hash — not human-readable)");

        System.out.println("\n=== KEY OBSERVATIONS ===");
        System.out.println("1. Header and Payload are just Base64 encoded — ANYONE can read them");
        System.out.println("2. Signature requires SECRET_KEY to verify or forge");
        System.out.println("3. Without the secret key, you cannot create a valid signature");
    }
}
```

**Output:**

```
=== JWT DECODED ===

HEADER:
{"alg":"HS256","typ":"JWT"}

PAYLOAD:
{"sub":"john","roles":["ROLE_USER"],"iat":1700000000,"exp":1700003600}

SIGNATURE (raw Base64Url):
8Th0L6KPeq4hXBi3BolBSH6cBHJHnVGkwRqpAMNlr-A
(This is a cryptographic hash — not human-readable)

=== KEY OBSERVATIONS ===
1. Header and Payload are just Base64 encoded — ANYONE can read them
2. Signature requires SECRET_KEY to verify or forge
3. Without the secret key, you cannot create a valid signature
```

### Step 3: Prove tampering breaks the token

```java
// Add this to your main method:

System.out.println("\n=== TAMPERING DEMONSTRATION ===");

// Modify the payload — change role to ADMIN
String maliciousPayload = Base64.getUrlEncoder()
    .withoutPadding()
    .encodeToString("{\"sub\":\"john\",\"roles\":[\"ROLE_ADMIN\"],\"iat\":1700000000,\"exp\":1700003600}"
    .getBytes());

String tamperedJwt = parts[0] + "." + maliciousPayload + "." + parts[2];

System.out.println("Original JWT  : " + jwt);
System.out.println("Tampered JWT  : " + tamperedJwt);
System.out.println("\nDecode tampered payload:");
String tamperedDecoded = new String(Base64.getUrlDecoder().decode(maliciousPayload));
System.out.println(tamperedDecoded);
System.out.println("\nThe payload shows ROLE_ADMIN now...");
System.out.println("BUT the signature no longer matches the modified payload!");
System.out.println("Server will reject this token during signature verification.");
```

**This proves:** You can read the payload, you can even change it, but the signature will mismatch and the server will reject it.

---

## Section 3 — Interview Summary

| Question | Answer |
|----------|--------|
| What are the three parts of a JWT? | Header (algorithm), Payload (claims), Signature (tamper-proof hash) |
| Is JWT encrypted? | No — it is signed (Base64 encoded). Anyone can read the payload. |
| Why is JWT stateless? | All user info is embedded in the token. Server verifies signature mathematically, no DB/session lookup needed. |
| What makes JWT tamper-proof? | The signature is an HMAC hash of header+payload using a secret key. Changing payload invalidates signature. |
| What is the `sub` claim? | Subject — typically the username or user ID |
| What is the `exp` claim? | Expiration time as Unix timestamp — server rejects expired tokens |
| Why is JWT good for microservices? | Each service independently verifies the token without shared session storage |
| What should you NEVER put in JWT payload? | Passwords, secrets, sensitive PII — payload is publicly readable |
| What is the `alg: none` attack? | Attacker removes signature and sets algorithm to "none" — always validate algorithm explicitly |

---

## Quick Understanding Check

**Q1.** A JWT has this payload:
```json
{ "sub": "alice", "roles": ["ROLE_ADMIN"], "exp": 1700003600 }
```
An attacker intercepts this token. Can they:
- (a) Read the payload and know Alice is an ADMIN?
- (b) Change the payload to make themselves an ADMIN?
- (c) Use Alice's valid token before expiry?

Explain your answer for each.

**Q2.** Your team is debating: *"Should we store the user's current account balance in the JWT payload so we don't need a DB call?"*

What is your response as a security-conscious engineer? Give two specific reasons.

**Q3.** You have three microservices: `order-service`, `payment-service`, `inventory-service`. A user logs in via `auth-service` and gets a JWT signed with `RS256`.

- What does `RS256` mean in this context?
- Which key does `auth-service` use?
- Which key do the three microservices use to verify?
- Why is this better than `HS256` for microservices?
