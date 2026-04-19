# ✅ 2. Introduction to JWT (JSON Web Token) — In Depth

*(Foundational → Production‑Grade Understanding)*

***

## 1️⃣ First: The Core Problem JWT Solves

### ✅ The Reality of HTTP

HTTP is **stateless** by design.

That means:

*   Each request is **independent**
*   Server does **not remember** previous requests automatically

So after login:
❓ *How does the server know who is making the next request?*

***

### ❌ Traditional Solution: Server‑Side Sessions

*   Server stores session data
*   Client stores session ID (cookie)

**Problems in modern systems:**

*   Poor scalability
*   Difficult in microservices
*   Session replication required
*   Load balancer affinity needed

These limitations **directly led to JWT**.

***

## 2️⃣ What Is a JWT? (Official Definition)

### ✅ JWT Definition

A **JSON Web Token (JWT)** is:

> A compact, URL‑safe, self‑contained way to securely transmit information between parties as a JSON object, where the information can be verified and trusted because it is digitally signed. [\[jwt.io\]](https://www.jwt.io/introduction), [\[rfc-editor.org\]](https://www.rfc-editor.org/rfc/rfc7519)

Key words (VERY IMPORTANT):

*   **Compact**
*   **Self‑contained**
*   **Digitally signed**

***

## 3️⃣ Why JWT Is So Powerful

### ✅ Stateless Authentication

JWT carries **all required authentication data inside itself**.

So:

*   No DB lookup per request
*   No session storage
*   Any server can validate it

✅ This is **perfect for microservices**. [\[bing.com\]](https://bing.com/search?q=JSON+Web+Token+stateless+authentication+REST+API), [\[generatorkithub.com\]](https://www.generatorkithub.com/blog/complete-guide-jwt)

***

### ✅ Benefits of JWT in API Security

| Benefit        | Why It Matters    |
| -------------- | ----------------- |
| Stateless      | Scalability       |
| Compact        | Fast transmission |
| Self‑contained | No session store  |
| Signed         | Tamper‑proof      |
| Standardized   | RFC 7519          |

***

## 4️⃣ JWT Structure — The Most Important Part

A JWT has **three parts**, separated by dots (`.`):

    Header.Payload.Signature

Example:

    xxxxx.yyyyy.zzzzz

***

## 5️⃣ JWT Part 1 — Header

### ✅ What the Header Contains

Metadata about the token.

Typical header:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Meaning:

*   `alg` → Signing algorithm
*   `typ` → Token type (JWT)

Header is **Base64Url‑encoded**, NOT encrypted. [\[jwt.io\]](https://www.jwt.io/introduction)

***

## 6️⃣ JWT Part 2 — Payload (Claims)

### ✅ What Are Claims?

Claims are **statements about an entity**, usually the user.

Example payload:

```json
{
  "sub": "123",
  "username": "alice",
  "role": "ADMIN",
  "iat": 1713440000,
  "exp": 1713443600
}
```

***

### ✅ Types of Claims (EXAM + INTERVIEW)

| Type       | Meaning                  |
| ---------- | ------------------------ |
| Registered | Standard (iss, sub, exp) |
| Public     | Shared across systems    |
| Private    | Custom application data  |

Important registered claims:

*   `sub` → Subject (user)
*   `exp` → Expiry time
*   `iat` → Issued at
*   `iss` → Issuer

📌 **Payload is readable by anyone** — it is only Base64 encoded, not encrypted. [\[geeksforgeeks.org\]](https://www.geeksforgeeks.org/web-tech/json-web-token-jwt/), [\[jwt.io\]](https://www.jwt.io/introduction)

***

## 7️⃣ JWT Part 3 — Signature (THE SECURITY CORE)

### ✅ What the Signature Does

The signature:

*   Protects **integrity**
*   Detects **tampering**
*   Proves **authenticity**

***

### ✅ How Signature Is Created

Conceptually:

    HMAC(
      base64(header) + "." + base64(payload),
      secret_key
    )

This ensures:

*   Header + payload cannot be modified
*   Only the server with the secret/private key could sign it [\[jwt.io\]](https://www.jwt.io/introduction), [\[auth0.com\]](https://auth0.com/docs/secure/tokens/json-web-tokens)

***

## 8️⃣ JWT Is NOT Encryption (VERY IMPORTANT)

✅ JWT **does NOT hide data**
❌ JWT **DOES NOT encrypt payload**

Anyone can decode header & payload.

JWT security comes from:

*   Signature verification
*   Expiration
*   Trusted issuer

📌 **Sensitive data should never be inside JWT payload**.

***

## 9️⃣ JWT Authentication Flow (STEP‑BY‑STEP)

This is the **correct mental model**.

***

### ✅ Step 1: User Logs In

    POST /login
    username + password

***

### ✅ Step 2: Server Authenticates User

*   Validates credentials
*   Authentication SUCCESS or FAIL

***

### ✅ Step 3: Server Generates JWT

If successful:

*   Create header
*   Create payload (claims)
*   Sign token with secret/private key

JWT is returned to client. [\[generatorkithub.com\]](https://www.generatorkithub.com/blog/complete-guide-jwt)

***

### ✅ Step 4: Client Stores Token

Common storage:

*   Memory
*   Secure storage
*   HttpOnly cookie

***

### ✅ Step 5: Client Sends JWT on Every Request

Header:

    Authorization: Bearer <JWT>

This is standard Bearer token authentication. [\[auth0.com\]](https://auth0.com/docs/secure/tokens/json-web-tokens)

***

### ✅ Step 6: Server Validates JWT

For every request:

*   Verify signature
*   Check expiration (`exp`)
*   Validate issuer (`iss`)
*   Extract claims

✅ If valid → request allowed  
❌ If invalid → **401 Unauthorized**

***

## 🔁 Full Flow Visualization

    Client → Login → Server
    Client ← JWT ← Server

    Client → API (with Bearer JWT) → Server
    Server → Verify JWT → Allow / Reject

***

## 1️⃣0️⃣ Why JWT Is “Stateless” (CRUCIAL)

### ✅ Stateless Means

The server:

*   Does NOT store session
*   Does NOT remember client
*   Only verifies token

All state lives **inside the token**.

 [\[bing.com\]](https://bing.com/search?q=JSON+Web+Token+stateless+authentication+REST+API), [\[generatorkithub.com\]](https://www.generatorkithub.com/blog/complete-guide-jwt)

***

## 1️⃣1️⃣ HTTP Basic vs JWT Bearer (RECAP)

| Aspect                | HTTP Basic    | JWT Bearer    |
| --------------------- | ------------- | ------------- |
| Credentials sent      | Every request | Only at login |
| Stateless             | Yes           | Yes           |
| Secure                | Weak          | Strong        |
| Used in microservices | Rare          | Standard      |
| Scalability           | Low           | High          |

***

## ✅ Final Mental Model (READ THIS TWICE)

> **JWT turns authentication into a cryptographically verifiable token that carries identity and authorization data, allowing fully stateless, scalable, and secure API authentication without server‑side sessions.**

***

## ✅ One‑Line Interview Answer

> *JWT is a stateless, self‑contained authentication mechanism where user identity and authorization claims are securely encoded and signed inside a token that is validated on each request without maintaining server‑side session state.*

***

## 📚 Authoritative Sources

*   JWT Official Docs (RFC 7519, jwt.io) [\[jwt.io\]](https://www.jwt.io/introduction), [\[rfc-editor.org\]](https://www.rfc-editor.org/rfc/rfc7519)
*   Baeldung: Introduction to JWT [\[baeldung.com\]](https://www.baeldung.com/java-json-web-tokens-jjwt)
*   Auth0 JWT Docs [\[auth0.com\]](https://auth0.com/docs/secure/tokens/json-web-tokens)

***

## 🔜 What We Should Learn Next (VERY IMPORTANT)

1️⃣ JWT vs Sessions  
2️⃣ JWT Expiration & Refresh Tokens  
3️⃣ JWT + Spring Security Filter Chain  
4️⃣ JWT in Microservices (Gateway pattern)
