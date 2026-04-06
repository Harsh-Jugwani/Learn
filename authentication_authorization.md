# Understanding Authentication and Authorization

## Overview

**Authentication** verifies the identity of a user or service requesting access to a system.
**Authorization** is the prerequisite step before authentication, which determines access levels and permissions.

---

## Table of Contents

1. [Authentication Methods](#authentication-methods)
2. [Authentication Flows](#authentication-flows)
3. [OAuth 2.0 vs OpenID Connect](#oauth-20-vs-openid-connect)
4. [Best Practices](#best-practices)

---

## Authentication Methods

**Authentication** verifies "Who are you?" **Authorization** verifies "What can you access?"

### 1. Basic Authentication
- Sends base64-encoded username:password in every request
- ❌ Unsafe even with HTTPS (credentials repeatedly exposed)
- Use: Simple testing, internal APIs only

### 2. API Key Authentication
- Static token per client, valid indefinitely
- ❌ No expiration, requires database lookup per request
- Use: Server-to-server communication

### 3. Session-Based Authentication
- Server stores session data, client gets session ID in cookie
- ✅ Immediate revocation ❌ Doesn't scale across servers
- Use: Traditional monolithic web applications

### 4. JWT (JSON Web Tokens)
- Self-contained token with digital signature, sent with each request
- ✅ Stateless, scales horizontally, fast verification ⚠️ Delayed revocation
- Use: Microservices, distributed systems, SPAs

### 5. Bearer Token Authentication
- Token sent in Authorization header: `Authorization: Bearer <token>`
- ❌ Token exposed if intercepted (must use HTTPS)
- Use: REST APIs, modern web services (token can be JWT or opaque)

### 6. Digest Authentication
- Hashes credentials with nonce instead of sending plaintext
- ✅ Better than Basic Auth ❌ Rarely used (old, complex)
- Use: Legacy systems, WebDAV

### 7. OAuth 2.0 / OpenID Connect
- Delegated authorization via third-party providers
- ✅ Industry standard, user never shares main password ⚠️ Complex flow
- Use: Social login, third-party integrations

### 8. SAML (Security Assertion Markup Language)
- XML-based authentication, redirect to IdP for login
- ✅ Enterprise standard ❌ Verbose, complex
- Use: Enterprise SSO, identity federation

---

## Authentication Flows

### Flow Diagram: All Authentication Methods

```
CLIENT REQUEST
        │
        ▼
    ┌─────────────┐
    │ No Creds?   │──YES──> 401 Unauthorized (Redirect to login)
    └──NO─┬───────┘
         │
         ▼
    ┌─────────────────────────┐
    │ Authentication Type/    │
    │ Authentication Header   │
    └──┬──┬──┬──┬──┬──┬────────┘
       │  │  │  │  │  │
       │  │  │  │  │  └─────── OAUTH 2.0/OIDC
       │  │  │  │  │          (3rd party provider flow)
       │  │  │  │  │
       │  │  │  │  └────── SAML
       │  │  │  │          (XML assertion, redirect to IdP)
       │  │  │  │
       │  │  │  └──────── BEARER TOKEN
       │  │  │            (Authorization: Bearer <token>)
       │  │  │            ├─ Verify token signature/format
       │  │  │            ├─ Check expiration
       │  │  │            └─ Grant access if valid
       │  │  │
       │  │  └────────── JWT
       │  │              ├─ Extract token
       │  │              ├─ Verify signature
       │  │              ├─ Check expiration
       │  │              └─ Extract claims
       │  │
       │  └──────────── DIGEST AUTH
       │               ├─ Extract username
       │               ├─ Retrieve stored hash
       │               ├─ Compare hash+nonce
       │               └─ Grant if match
       │
       └────────────── BASIC/API KEY/SESSION
                       ├─ Extract credentials/ID
                       ├─ Query database
                       ├─ Hash & compare / Review permissions
                       └─ Grant if valid
```

### JWT Flow: Login to Access

```
1️⃣  LOGIN
    POST /login {username, password}
              ▼
    ✓ Credentials valid → Server creates JWT
    ├─ Header: {alg: HS256, typ: JWT}
    ├─ Payload: {sub: userID, role: "user", exp: now+1h}
    └─ Signature: HMACSHA256(header.payload, SECRET_KEY)
              ▼
    Client receives: eyJhbGc.eyJzdWI.SflKx...
    Stores in localStorage/state
              
2️⃣  API REQUESTS (with token)
    Authorization: Bearer eyJhbGc.eyJzdWI.SflKx...
              ▼
    Server: Verify signature ✓ → Extract claims ✓
    → Check exp ✓ → 200 OK with data
    
    If tampered: Signature mismatch → 401
    If expired: exp < now → 401
```

### Session Flow: Login to Access

```
1️⃣  LOGIN
    POST /login {username, password}
              ▼
    ✓ Credentials valid → Server creates session
    ├─ Session ID: sess_abc123
    ├─ Store in DB: {userID, permissions, created_at}
    └─ Send cookie: Set-Cookie: sid=sess_abc123
              ▼
    Client browser stores cookie automatically

2️⃣  API REQUESTS (with cookie)
    Cookie: sid=sess_abc123
              ▼
    Server: Lookup session in DB ✓
    → Check expiration ✓ → 200 OK with data
    
    If invalid: Session not found → 401
    If expired: created_at + timeout < now → 401
```

### Bearer Token Flow: Generic Token

```
1️⃣  GET TOKEN (via login or OAuth)
    POST /login {username, password}
              ▼
    Server creates token (JWT or opaque)
    Returns: {token: "abc123def456"}

2️⃣  API REQUESTS (with Bearer token)
    Authorization: Bearer abc123def456
              ▼
    Server: Extract token from header
    → Verify signature/format ✓
    → Check expiration ✓
    → 200 OK with data
    
    If invalid: Invalid token → 401
    If expired: exp < now → 401
```

### Digest Auth Flow: Hashed Credentials

```
1️⃣  REQUEST WITHOUT CREDENTIALS
    GET /api/data
              ▼
    Server: No Authorization header
    → Return 401 with: WWW-Authenticate: Digest realm="API"
    
    Include: nonce=random123

2️⃣  CLIENT HASHES CREDENTIALS
    Client receives nonce
    Computes: MD5(username:realm:password)
    Creates digest: MD5(hash:nonce:method:uri)
    
3️⃣  REQUEST WITH DIGEST
    GET /api/data
    Authorization: Digest username="john",
                        realm="API",
                        nonce="random123",
                        uri="/api/data",
                        response="computed_digest"
              ▼
    Server: Retrieve stored user hash
    → Verify digest matches
    → Check nonce validity
    → Grant access if valid

❌ Still sends credentials with every request
❌ MD5 is weak, rarely used today
```

### SAML Flow: Enterprise SSO

```
1️⃣  APP REDIRECT TO IdP
    User clicks "Sign In"
    App: Redirect to SAML IdP (e.g., Okta)
    GET /saml/sso?SAMLRequest=...
              ▼

2️⃣  IdP LOGIN SCREEN
    User logs in at IdP
    IdP: Creates SAML Assertion (XML)
    Contains: UserID, email, roles, signature
              ▼

3️⃣  REDIRECT BACK TO APP WITH ASSERTION
    IdP: POST /saml/acs
    Body: SAML Assertion (base64 encoded)
              ▼

4️⃣  APP VALIDATES ASSERTION
    App: Decode SAML Assertion
    → Verify IdP signature ✓
    → Extract user claims ✓
    → Create app session ✓
    → Redirect to dashboard
    
    If invalid: Assertion doesn't match IdP cert → 401
```

---

## OAuth 2.0 vs OpenID Connect

### OAuth 2.0: Authorization Only

**Purpose**: Let users grant third-party apps access to resources WITHOUT sharing password

**Scenario**: Sign up for app using Google account
- Click "Sign in with Google"
- App redirects to Google login
- You approve permissions (read calendar, contacts, etc.)
- Google gives app an access_token
- App can now read your calendar/contacts
- ✅ You never share Google password with the app

**Use When**: Need to access user's resources (Google Drive, calendar, etc.)

### OAuth 2.0 Flow (Authorization Code)

```
1. USER                      2. APP                   3. GOOGLE
   Click "Sign with Google"      Redirect to           Login & 
                                 Google                 Approve
   ↓                             ↓                      ↓
   GET /login?client_id=X&redirect_uri=Y →  (User enters credentials)
                                              (User clicks "Allow")
   ← Auth Code redirected back
   (code=ABCD123)

4. APP BACKEND               5. GOOGLE
   POST /token               Return tokens
   code=ABCD123
   client_secret=XXXX
   ↓                         ↓
   access_token = AT123
   refresh_token = RT456
   expires_in = 3600

6. APP USES access_token to call Google APIs
   GET /calendar/events
   Authorization: Bearer AT123
   → Returns user's calendar events
```

**Key Points:**
- ✅ User never shares password
- ✅ User can revoke access anytime
- ✅ Temporary access_token (auto-expires)
- ✅ refresh_token renews expired token
- ❌ Doesn't tell app WHO the user is

---

### OpenID Connect (OIDC): Authentication + Authorization

**Purpose**: Adds identity verification ON TOP of OAuth 2.0

**Key Difference**:
- OAuth 2.0 = "App can access your calendar" (permissions)
- OIDC = "App can access your calendar AND I verify who you are" (auth + permissions)

**Returns**: ID Token (JWT) with user identity claims

### OIDC Flow (Same as OAuth 2.0 plus scope=openid)

```
Add to OAuth 2.0 flow:

1. REQUEST
   GET /authorize?client_id=X&scope=openid profile email
   ↑ scope=openid tells Google: "We want identity info"

2. RESPONSE contains:
   {
     "access_token": "AT123",
     "id_token": "eyJhb...",     ← JWT with user identity!
     "token_type": "Bearer",
     "expires_in": 3600
   }

3. ID TOKEN (decoded):
   {
     "iss": "https://accounts.google.com",
     "sub": "user_unique_id",
     "email": "john@example.com",   ← IDENTITY
     "name": "John Doe",
     "picture": "https://...",
     "email_verified": true,
     "iat": 1704067200,
     "exp": 1704070800
   }

APP NOW KNOWS:
  • sub = Who the user is
  • email = User's email
  • name = User's display name
```

---

### OAuth 2.0 vs OIDC: When to Use

| Scenario | Use | Why |
|----------|-----|-----|
| User signs up with Google account | OIDC | Need identity (email, name) + permissions |
| App reads user's Google Drive files | OAuth 2.0 | Only need resource access permission |
| Social login (Facebook, GitHub, Apple) | OIDC | Always need to identify who logged in |
| Microservice calling another microservice | OAuth 2.0 | Service principals, not users |
| Mobile app with multi-provider login | OIDC | Standardizes identity across providers |
| API integration (Stripe charges, Slack messages) | OAuth 2.0 | Only need resource access |

**Simple Rule**: 
- **Need to know WHO the user is?** → Use OIDC
- **Only need to access THEIR resources?** → Use OAuth 2.0
- **In doubt?** → Use OIDC (it includes OAuth 2.0)

---

## Best Practices

1. **Use HTTPS** - Always encrypt credentials and tokens in transit
2. **Store tokens in HTTP-only cookies** - Prevents XSS attacks (JS can't access)
3. **Use short-lived access tokens** - 15-60 minutes reduces compromise risk
4. **Use refresh tokens** - Keep access tokens short, use refresh for renewal
5. **Validate tokens on every request** - Don't trust cached validation
6. **Use strong encryption** - HS256 minimum, RS256 recommended for distributed systems
7. **Never log credentials, passwords, or tokens** - Avoid accidental exposure
8. **Implement rate limiting** - Prevent brute force attacks (max 5 failed logins)
9. **Use industry standards** - OAuth 2.0, OIDC, SAML (never invent custom auth)
10. **Regularly rotate secrets** - Update SECRET_KEY yearly, revoke compromised keys

---

## Summary

Authentication and authorization are complementary security mechanisms:

- **Authentication** answers "Who are you?" and verifies identity
- **Authorization** answers "What can you access?" and grants permissions

Choose the appropriate method based on your system architecture, security requirements, and scalability needs.