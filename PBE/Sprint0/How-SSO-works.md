## 1. Same Browser Requirement

Yes, usually **SSO works in the same browser session** because the **authentication session (cookie/token)** is stored in that browser.

Example:

* You log in to **App A**
* Then open **App B** in the **same browser**
* App B detects you are already authenticated and logs you in automatically.

If you open **another browser** or **incognito window**, you usually must log in again because the **session/cookies are not shared**.

---

## 2. Important Point: Token is Usually NOT Stored Directly by App A

In a proper SSO architecture:

* Apps **do not authenticate users themselves**
* They redirect authentication to a **central identity provider (IdP)**.

Common identity providers include:

* Keycloak
* Okta
* Auth0

Protocols used:

* OAuth 2.0
* OpenID Connect
* SAML

---

## 3. Real SSO Flow (Simplified)

### Step 1 – User opens App A

App A sees user is **not authenticated**.

➡ Redirects user to **Identity Provider** (for example **Keycloak**).

---

### Step 2 – Login at Identity Provider

User enters username/password.

Identity provider:

* Creates a **login session**
* Stores it as a **cookie in the browser** for its domain.

---

### Step 3 – Token Issued

The identity provider sends back:

* **JWT access token**
* **ID token**

to **App A**.

App A now allows the user.

---

### Step 4 – User Opens App B

User goes to **App B**.

App B also redirects to the **same Identity Provider**.

BUT:

* Browser **already has the IdP session cookie**
* So the IdP **doesn't ask for login again**

➡ It immediately sends a token to **App B**.

User logs in **without entering credentials again**.

---

## 4. Important Correction to Your Statement

You said:

> App B searches for a specific header in the browser for the login token.

Not exactly.

Browsers **do not store HTTP headers**.

What usually happens is:

* **Cookies** store the login session (for the Identity Provider).
* Tokens (like **JWT**) may be stored in:

  * **cookies**
  * **localStorage**
  * **sessionStorage**

But **SSO mainly works because of the IdP session cookie**, not because App B reads headers from the browser.

---

## 5. Simple Visual Flow

```
User → App A
        ↓
Redirect → Identity Provider (Keycloak)
        ↓
User Login
        ↓
IdP sets login cookie in browser
        ↓
JWT Token → App A
```

Later:

```
User → App B
        ↓
Redirect → Identity Provider
        ↓
IdP sees existing login cookie
        ↓
No login required
        ↓
JWT Token → App B
```

---

## 6. Key Idea

SSO works because:

✅ **Identity Provider session cookie**
❌ Not because apps read browser headers directly

---

✅ **One-line interview answer:**

> In SSO, multiple applications trust a centralized identity provider. Once a user logs in, the identity provider stores a session cookie in the browser, allowing other applications to authenticate the user without requiring another login.

---

If you want, I can also explain **how SSO + JWT works in Spring Boot with Keycloak**, which is a **very common Java microservices interview question**.
