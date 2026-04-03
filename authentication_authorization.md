# Understanding Authentication and Authorization

## Overview

**Authentication** verifies the identity of a user or service requesting access to a system.
**Authorization** is the prerequisite step before authentication, which determines access levels and permissions.

---

## Table of Contents

1. [Fundamentals of Authentication](#fundamentals-of-authentication)
2. [Basic Authentication Methods](#basic-authentication-methods)
3. [Token-Based Authentication](#token-based-authentication)
4. [Internal Workings - JWT Deep Dive](#internal-workings---jwt-deep-dive)
5. [Other Algorithm Internal Workings](#other-algorithm-internal-workings)
6. [OAuth 2.0 and OpenID Connect](#oauth-20-and-openid-connect)
7. [Single Sign-On and Identity Protocols](#single-sign-on-and-identity-protocols)
8. [Authentication Flow Diagram](#authentication-flow-diagram)
9. [Why Modern Methods Are Better](#why-modern-methods-are-better)

---

## Fundamentals of Authentication

### Key Concepts

- **Authentication**: Verifies the identity of a user or service requesting access to a system
- **Authorization**: Determines access levels and permissions for verified identities
- **Common Configurations**: Exist between authentication methods, which verify identity and authorization frameworks that grant access
- **Unauthorized Requests**: Receive a 401 response; valid identities proceed to authorization

### The Authentication Question

Authentication answers: **"Who is trying to access the system?"**

---

## Basic Authentication Methods

### 1. Basic Authentication

- **How it works**: Uses base64-encoded username and password in headers
- **Transport**: Sent with HTTPS credentials in every request
- **Security**: Vulnerable unless used with HTTPS
- **Use case**: Simple APIs requiring basic security

```
Authorization: Basic base64(username:password)
```

### 2. Digest Authentication

- **How it works**: Improves on basic by hashing credentials with MD5
- **Advantage**: Still encrypted, but less exposed than basic auth
- **Status**: Still outdated and rarely used

### 3. API Key Authentication

- **How it works**: Unique key per client sent with requests
- **Scopes**: Keys can be embedded with usage scopes and limitations
- **Risks if compromised**: Exposed keys present significant security risks; no native expiration
- **Use case**: Server-to-server communication, internal APIs

### 4. Session-Based Authentication

- **How it works**: User logs in; server creates a session stored in memory
- **Storage**: Session data remains in memory, restricted to databases and internal caches
- **Scalability**: Less scalable for distributed systems
- **Common Use**: Traditional web applications

---

## Token-Based Authentication

### JWT (JSON Web Tokens)

- **Format**: Signed JSON objects containing user data and claims
- **Structure**: Stateless and self-contained
- **Advantages**: Reduces server load, enables distributed systems
- **Usage**: Modern web and mobile applications

### Access Tokens

- **Purpose**: Short-lived tokens used to access APIs
- **Lifespan**: Seconds to minutes
- **Security**: Limits damage from token compromise

### Refresh Tokens

- **Purpose**: Long-lived tokens used to obtain new access tokens
- **Storage**: Stored securely in HTTP-only cookies
- **Security**: Prevents XSS attacks by restricting JavaScript access

### Bearer Authentication

- **Pattern**: Possession of the token grants access to resources
- **Format**: Not tied to specific cryptographic methods
- **Use case**: REST APIs and modern web services

---

## Internal Workings - JWT Deep Dive

### JWT Structure

JWT consists of three parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI1NDMyMTAwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

[HEADER].[PAYLOAD].[SIGNATURE]
```

### Part 1: Header (JSON encoded in Base64)

```json
{
  "alg": "HS256",      // Algorithm used for signing
  "typ": "JWT"         // Token type
}
```

**Base64 Encoded:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

### Part 2: Payload (JSON encoded in Base64)

```json
{
  "sub": "5432100",              // Subject (user ID)
  "name": "John Doe",            // User name
  "email": "john@example.com",   // Email claim
  "iat": 1516239022,             // Issued at (timestamp)
  "exp": 1516242622,             // Expiration time
  "aud": "myapp.com",            // Audience
  "custom_claim": "value"        // Custom claims
}
```

**Base64 Encoded:** `eyJzdWIiOiI1NDMyMTAwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ`

### Part 3: Signature (Cryptographic Hash)

The signature is created by:

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  "your-secret-key"
)
```

**Result:** `SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`

### Can Users Tamper with JWT? NO! Here's Why:

```
┌─────────────────────────────────────────────────────────┐
│              JWT Tampering Protection                   │
└─────────────────────────────────────────────────────────┘

SCENARIO: User intercepts JWT and tries to change it

Original JWT:
  Header: {alg: HS256}
  Payload: {role: "user", id: 123}
  Signature: abc123xyz (computed with SERVER's SECRET_KEY)
  ✓ Valid signature matches payload


User intercepts and modifies:
  Header: {alg: HS256}
  Payload: {role: "admin", id: 123}  ← CHANGED
  Signature: abc123xyz (unchanged)
  ✗ Signature now INVALID!


Why? WITHOUT the SECRET_KEY:
  • User CANNOT recompute HMACSHA256(header.payload, SECRET_KEY)
  • Only the server knows the SECRET_KEY
  • Even if user tries to fake a new signature:
    - Probability of guessing correct hash: 1 in 2^256
    - Essentially impossible


SERVER VERIFICATION PROCESS:
  1. Receives: {header.payload.abc123xyz}
  2. Extracts: header and payload
  3. Computes: expected_sig = HMACSHA256(header.payload, SERVER_SECRET_KEY)
  4. Compares: expected_sig vs abc123xyz
  5. Result: ❌ MISMATCH → 401 Unauthorized
```

### JWT Flow Diagram: Creation to Verification

```
── JW T A U T H E N T I C AT I O N F L O W ──

1️⃣  USER LOGIN
   ┌───────────────────┐
   │ POST /login       │
   │ username: john    │
   │ password: pass123 │
   └────────┬──────────┘
            │
            ▼
   💾 SERVER SIDE (Has SECRET_KEY in vault)
   ┌────────────────────────────────────────┐
   │ ✓ Verify credentials against database  │
   │ ✓ Credentials valid                    │
   │                                        │
   │ Create JWT Payload:                    │
   │ {                                      │
   │   "sub": "john",      // user id       │
   │   "role": "user",     // permissions   │
   │   "email": "j@ex",    // claims        │
   │   "iat": 1704067200,  // issued now    │
   │   "exp": 1704070800   // expires 1h    │
   │ }                                      │
   │                                        │
   │ Compute Signature:                     │
   │ sig = HMACSHA256(                      │
   │   header + "." + payload,              │
   │   SECRET_KEY                           │
   │ )                                      │
   └──────────┬─────────────────────────────┘
              │
              ▼
   ┌┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┐
   │ JWT Response:                         │
   │ eyJhbGc.eyJzdWI.SflKxw                │
   │ [HEADER].[PAYLOAD].[SIGNATURE]        │
   └┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┘
              │
              ▼
   📱 CLIENT
   • Stores JWT in localStorage
   • Includes with every API request:
     Authorization: Bearer eyJhbGc.eyJzdWI.SflKxw


2️⃣  CLIENT API REQUEST
   ┌─────────────────────────────────┐
   │ GET /api/profile                │
   │ Authorization: Bearer eyJhbGc...│
   └────────┬────────────────────────┘
            │
            ▼
   💾 SERVER VERIFICATION
   ┌────────────────────────────────────────┐
   │ Receive: eyJhbGc.eyJzdWI.SflKxw        │
   │                                        │
   │ Step 1: PARSE JWT                      │
   │ • Split by dots into 3 parts           │
   │ • Header: eyJhbGc                      │
   │ • Payload: eyJzdWI                     │
   │ • Signature: SflKxw                    │
   │                                        │
   │ Step 2: VERIFY SIGNATURE               │
   │ • Compute: expected = HMACSHA256(      │
   │     header+"."+payload,                │
   │     SECRET_KEY                         │
   │   )                                    │
   │ • Compare: expected vs SflKxw          │
   │ ✓ MATCH! → Continue                   │
   │                                        │
   │ Step 3: CHECK EXPIRATION               │
   │ • Extract "exp": 1704070800            │
   │ • Compare with current_time            │
   │ ✓ Not expired → Continue               │
   │                                        │
   │ Step 4: EXTRACT CLAIMS                 │
   │ • Decode payload                       │
   │ • Get: role="user", sub="john"         │
   │ ✓ Valid user → Grant access            │
   └────────┬────────────────────────────────┘
            │
            ▼
   ┌────────────────────────┐
   │ 200 OK                 │
   │ {profile_data}         │
   │ Include new token:     │
   │ X-New-Token: eyJ...    │
   └────────────────────────┘


OR if verification fails:

Tampered Token:
  Signature doesn't match → ❌ 401 Unauthorized
  Token expired → ❌ 401 Token Expired
  Token malformed → ❌ 400 Bad Request
```

---

## Other Algorithm Internal Workings

### Session-Based Authentication (Legacy Approach)

```
── S E S S I O N - B A S E D F L O W ──

1️⃣  USER LOGIN
   Client: POST /login (user, pass)
   ┌────────────────────────────────────┐
   │ SERVER                             │
   │ • Verify credentials ✓             │
   │ • Generate Session ID:             │
   │   sess_a1b2c3d4e5f6g7h8            │
   │                                    │
   │ • Store in SERVER MEMORY:          │
   │ {                                  │
   │   "sess_a1b2c3d4e5f6g7h8": {       │
   │     "user_id": 123,                │
   │     "username": "john",            │
   │     "role": "user",                │
   │     "created_at": 1704067200       │
   │   }                                │
   │ }                                  │
   │                                    │
   │ • Send to client via HTTP cookie   │
   │   Set-Cookie:                      │
   │   session=sess_a1b2c3d4e5f6g7h8    │
   └────────┬─────────────────────────────┘
            │
            ▼
   📱 CLIENT
   Browser stores cookie automatically
   (HTTP-Only cookie = safer, can't access via JS)


2️⃣  SUBSEQUENT API REQUEST
   Browser automatically sends:
   Cookie: session=sess_a1b2c3d4e5f6g7h8
   ┌────────────────────────────────────┐
   │ SERVER                             │
   │ Lookup in SESSION_STORE:           │
   │                                    │
   │ if (sess_a1b2c3d4e5f6g7h8 exists)  │
   │   • Extract user_id = 123          │
   │   • Check permissions              │
   │   • Grant access ✓                 │
   │ else                               │
   │   • Session invalid/expired        │
   │   • Return 401 Unauthorized ✗      │
   └────────────────────────────────────┘


❌ DISADVANTAGES:
• Must store every session on server → Memory intensive
• Database/cache lookup on every request → Slow
• Doesn't scale across multiple servers (sessions stuck on 1 server)
• Can't use with stateless microservices
• Must implement cache invalidation
• Requires Redis/Memcached for distributed systems
```

### Basic Authentication (Credentials)

```
── B A S I C A U T H F L O W ──

EVERY REQUEST includes credentials:

Client Request:
GET /api/data
Authorization: Basic dGVzdDpwYXNzd29yZA==
                      ↑ Base64("test:password")

┌────────────────────────────────────┐
│ SERVER                             │
│ • Decode Base64:                   │
│   dGVzdDpwYXNzd29yZA== → test:pass  │
│                                    │
│ • Query database:                  │
│   SELECT * FROM users              │
│   WHERE username = 'test'           │
│                                    │
│ • Hash submitted password:          │
│   bcrypt("password") vs stored_hash │
│                                    │
│ • If match → Grant access ✓        │
│ • Else → 401 Unauthorized ✗        │
└────────────────────────────────────┘


❌ CRITICAL SECURITY ISSUES:
✗ Credentials sent with EVERY request
✗ Even over HTTPS, credentials repeatedly exposed
✗ Base64 is ENCODING, not ENCRYPTION
✗ Can be easily decoded: echo "dGVzdDpwYXNzd29yZA==" | base64 -d
✗ If token leaked, attacker has permanent access
✗ No expiration time
✗ No token revocation mechanism
✗ Server must hash & verify on every request (slow)
✗ Credentials stored in multiple places (browser, network, server)
```

### API Key Authentication (Static Token)

```
── A P I K E Y F L O W ──

1️⃣  KEY GENERATION (One-time setup)
   Admin creates in dashboard:
   API-Key: sk_live_51234567890abc...
   
   Server stores in database:
   {
     "api_key": "sk_live_51234567890abc...",
     "user_id": 123,
     "permissions": ["read", "write"],
     "created_at": 1704067200,
     "expires_at": null  ← NO EXPIRATION!
   }


2️⃣  CLIENT REQUESTS (Every time, forever)
   GET /api/data
   Authorization: ApiKey sk_live_51234567890abc...


3️⃣  SERVER VALIDATES
   ┌────────────────────────────────────┐
   │ • Lookup key in database ✓         │
   │   (Could be slow with many keys)   │
   │                                    │
   │ • Compare: sk_live... == stored_key│
   │   ✓ Match → Grant access           │
   │   ✗ No match → 401 Unauthorized    │
   │                                    │
   │ • Extract permissions from record  │
   └────────────────────────────────────┘


❌ MAJOR LIMITATIONS:
✗ No expiration → If leaked, compromised forever
✗ Static → Same key used for years
✗ No refresh mechanism
✗ Limited claims (just API key + permissions)
✗ Requires database lookup (slow at scale)
✗ Manual revocation (doesn't auto-expire)
✗ Can't track token usage patterns
✗ Difficult to rotate securely
```

---

## OAuth 2.0 and OpenID Connect

### OAuth 2.0

- **Purpose**: Authorization framework, not an authentication method
- **Function**: Grants apps access to user resources without revealing identity
- **Scope**: Enables apps to verify user identity securely

### OpenID Connect (OIDC)

- **Purpose**: Adds authentication on top of OAuth 2.0
- **Return Value**: Returns ID tokens with user identity information
- **Enhancement**: Handles both identity verification and permission granting

### Key Clarifications

- OAuth 2.0 handles permissions
- OpenID Connect handles user identity verification
- Together, they provide secure authentication and authorization

---

## Single Sign-On and Identity Protocols

### Single Sign-On (SSO)

- **User Experience**: Allows one login to access multiple services
- **Implementation**: Pattern enabling seamless access across services
- **Benefit**: Improved user experience without repeated logins

### SAML (Security Assertion Markup Language)

- **Format**: XML-based assertion protocol
- **Usage**: Widely used in enterprise and legacy systems
- **Function**: Redirects user to login and returns identity assertion

### OpenID Connect

- **Standard**: Modern, JSON-based alternative to SAML
- **Alternatives**: Used by Google, Microsoft, and other major providers
- **Advantage**: More developer-friendly than SAML

### SSO Implementation

- **Cookies and Sessions**: Enable seamless access across services after initial login
- **Security**: Sessions and cookies provide secure continuation of user identity

---

## Authentication Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   User Request                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │  Step 1: AUTHENTICATION              │
        │  Verify "Who are you?"               │
        │  (Identity Verification)             │
        └──────────────┬───────────────────────┘
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
      ✓ Valid           ✗ Invalid
      Credentials       Credentials
            │                     │
            │                     ▼
            │              401 Unauthorized
            │              Response
            │
            ▼
  ┌──────────────────────────────────┐
  │  Step 2: AUTHORIZATION           │
  │  Verify "What can you access?"   │
  │  (Permission Verification)       │
  └───────────┬──────────────────────┘
              │
    ┌─────────┴──────────┐
    ▼                    ▼
✓ Permitted         ✗ Denied
Resources           Resources
    │                    │
    │                    ▼
    │            403 Forbidden
    │            Response
    │
    ▼
Access Granted
✓ 200 OK with Resource
```

---

## Authentication Methods Comparison

| Method | Security | Scalability | Use Case | Complexity |
|--------|----------|-------------|----------|-----------|
| Basic Auth | Low | Low | Simple APIs | Low |
| Digest Auth | Medium | Low | Legacy Systems | Medium |
| API Key | Medium | High | Internal APIs | Low |
| Session-Based | High | Low | Monolithic Apps | Medium |
| JWT/Token | High | High | Distributed Systems | High |
| OAuth 2.0 | High | High | Third-party Apps | Very High |
| SSO | High | High | Enterprise | Very High |

---

## Best Practices

1. **Always use HTTPS** for credential transmission
2. **Store sensitive tokens** in HTTP-only cookies to prevent XSS attacks
3. **Use short-lived access tokens** to minimize compromise risk
4. **Implement refresh tokens** for extended sessions
5. **Validate tokens** on every request
6. **Use industry standards** (OAuth 2.0, OpenID Connect, SAML)
7. **Never log credentials** or tokens in plain text
8. **Implement rate limiting** to prevent brute force attacks
9. **Use strong encryption** for stored data
10. **Regularly rotate secrets** and update libraries

---

## Decision Tree

```
Is user attempting access?
├─ No credentials provided?
│  └─> Redirect to login (401 Unauthorized)
│
├─ Simple internal API?
│  └─> Use API Keys or JWT
│
├─ Traditional web app?
│  └─> Use Session-Based Authentication
│
├─ Distributed/Microservices?
│  └─> Use JWT/Token-Based Auth
│
├─ Third-party app integration?
│  └─> Use OAuth 2.0 + OpenID Connect
│
└─ Enterprise environment?
   └─> Use SSO with SAML or OIDC
```

---

## Why Modern Methods Are Better

### JWT vs. Session-Based

| Aspect | Session-Based | JWT |
|--------|---------------|-----|
| **Storage** | Server (memory/database) | Client-stored, stateless |
| **Scalability** | ❌ Doesn't scale - session affinity required | ✅ Scales infinitely - no server state |
| **Microservices** | ❌ Requires session sharing between services | ✅ Each service can verify independently |
| **Database Hits** | ❌ Lookup required per request | ✅ No lookup - signature verified locally |
| **Performance** | ❌ Slower (DB/cache round-trip) | ✅ Faster (cryptographic verification) |
| **Token Revocation** | ✅ Immediate (delete from store) | ⚠️ Delayed (until token expires) |
| **Security** | ✅ All data server-side | ✅ Tamper-proof (cryptographic signature) |

**Example:** Handling 1M concurrent users
```
Session-Based:
  • Must store 1M sessions in memory: ~500MB - 1GB RAM
  • Every request hits database/cache
  • Cache cluster complexity, sync issues
  • Session loss if server crashes

JWT:
  • No server storage needed
  • Verify locally with cryptographic key
  • Horizontal scaling: add more servers, no sync needed
  • Server crash = no impact on token validity
```

### JWT vs. Basic Authentication

| Aspect | Basic Auth | JWT |
|--------|-----------|-----|
| **Credentials Sent** | Every request | Once at login, token thereafter |
| **Exposure** | Continuously exposed | Protected by expiration |
| **Man-in-the-Middle** | ❌ Can intercept credentials | ✅ Can't hijack session (token expires) |
| **Storage** | Multiple locations (risky) | Single token |
| **Revocation** | Immediate (change password) | Delayed (until expiry) |
| **Claim Data** | No claims | Rich claims included |

**Security Comparison:**
```
Basic Auth Attack:
  Attacker intercepts HTTPS → Gets credentials
  Can use forever → Permanent access ❌

JWT Attack:
  Attacker intercepts JWT → Gets token
  Token expires in 15 minutes
  Then only valid refresh token helps (harder to get) ✅
```

### JWT vs. API Keys

| Aspect | API Key | JWT |
|--------|---------|-----|
| **Expiration** | ❌ None (manually revoke) | ✅ Auto-expires |
| **Lock-in** | ❌ Can't revoke without blocking user | ✅ Clean rotation via new token |
| **Claims** | Limited (just permissions) | Rich (user data, roles, scopes) |
| **Refresh** | ❌ Manual key rotation | ✅ Automatic refresh token flow |
| **Verification** | ❌ DB lookup (slow) | ✅ Cryptographic verification (fast) |
| **Token Size** | Small | Larger (contains more data) |

**Real-World Scenario - Key Rotation:**
```
API Key:
  OLD_KEY: sk_live_abc123 (active, leaked in logs)
  Problem: Can't revoke without breaking user's integration
  ❌ Forced to keep active key exposed

JWT:
  OLD_TOKEN: expires in 5 minutes anyway
  User gets NEW_TOKEN automatically
  OLD_TOKEN becomes invalid soon
  ✅ Leaked token has limited exposure window
```

### Why Tampering Is Impossible with JWT

```
┌─────────────────────────────────────────────────────┐
│         Why JWT Signatures Matter                   │
└─────────────────────────────────────────────────────┘

Attacker's View:
  JWT: eyJhbGc.eyJzdWI6Im5vcm1hbCJ9.abc123
        │        │                  │
        └──────┬─────────────────────┘
               │ What attacker sees:
               │ • Header is readable (Base64)
               │ • Payload is readable (Base64)
               │ • Signature: abc123
               │
               │ Attacker changes:
               │ Payload: role = admin
               │ New payload: eyJzdWI6ImFkbWluIn0
               │
               │ But now: signature INVALID
               │ Can't compute valid signature WITHOUT
               │ the SECRET_KEY!


Mathematical Proof:
  Signature = HMACSHA256(header.payload, SECRET_KEY)
  
  Attacker knows:
    • header (readable)
    • old_payload (readable)
    • old_signature (readable)
    • new_payload (they created)
  
  Attacker DOESN'T know:
    • SECRET_KEY (hidden on server only)
  
  To forge signature, attacker must:
    new_signature = HMACSHA256(header.new_payload, ???)
    
  Options:
    1. Guess SECRET_KEY: ~2^256 tries (impossible in lifetime)
    2. Use wrong key: Signature won't match (caught)
    3. Create fake signature: ~2^256 tries (impossible)
  
  Result: ❌ Forged token always rejected


Server's Verification:
  received_sig = abc123
  computed_sig = HMACSHA256(header.new_payload, SECRET_KEY)
  
  if received_sig != computed_sig:
    return 401 Unauthorized
```

### Performance Comparison

**Request Latency for 1M Requests:**

```
Session-Based Auth:
  ┌─ Parse cookie (1ms)
  ├─ Database lookup (5-10ms) ← Big cost!
  ├─ Fetch user permissions (2-5ms) ← Another lookup!
  └─ Grant access (1ms)
  TOTAL: 10-20ms per request

JWT Auth:
  ┌─ Extract token (0.1ms)
  ├─ Verify signature (0.5ms) ← Cryptographic, fast
  ├─ Check expiration (0.1ms)
  ├─ Decode claims (0.1ms)
  └─ Grant access (0.1ms)
  TOTAL: 1-2ms per request
  
  ✅ JWT is 5-10x FASTER than session-based!
```

**System Load:**

```
Session-Based (1M concurrent users):
  • Session storage: 1GB+ memory
  • Cache hits/misses: 85% hit rate typical
  • DB connections: 100+ required
  • Network traffic: Session data back-and-forth
  • CPU: Hashing passwords on every request
  • Cost: Expensive cache infrastructure

JWT (1M concurrent users):
  • Session storage: Zero
  • Cache: Not needed for verification
  • DB connections: Zero
  • Network traffic: Just verify local signature
  • CPU: Cryptographic verification
  • Cost: Cheap infrastructure, horizontal scaling
```

### Scalability Matrix

```
Server Configuration Comparison:

Session-Based with 100 Servers:
  ┌──────────────┐
  │ Server Pool  │
  ├──────────────┤
  │ Server 1 ─┐  │
  │ Server 2 ─┼─ Shared Session Store (Redis)
  │ Server 3 ─┘  │
  │ ...          │ Problem: All requests hit central point
  │ Server 100   │ Creates bottleneck!
  └──────────────┘

JWT with 100 Servers:
  ┌──────────────┐
  │ Server Pool  │
  ├──────────────┤
  │ Server 1 ────── Verify JWT locally
  │ Server 2 ────── Verify JWT locally
  │ Server 3 ────── Verify JWT locally
  │ ...            Each server independent!
  │ Server 100 ──── Verify JWT locally
  └──────────────┘
  
  ✅ Perfect horizontal scaling!
```

### Real-World Example: API Rate Limiting

```
SCENARIO: Protect against brute force - allow 100 requests/hour per user

Session-Based Implementation:
  1. User makes request
  2. Server looks up session
  3. Query rate limit counter from cache
  4. Increment counter: SELECT rate + 1
  5. Update cache: SET rate = new_value
  6. Check if > 100: IF counter > 100 RETURN 429
  7. Process request
  
  Challenges:
    ❌ Multiple DB hits per request
    ❌ Race conditions with concurrent requests
    ❌ Cache coherency issues
    ❌ Performance degrades under load

JWT Implementation:
  1. User receives JWT with: {sub: "user123", iat: 1704067200}
  2. User makes request with JWT
  3. Server extracts: user_id from JWT (local, no DB)
  4. Query rate limiter for user_id (single lookup)
  5. Check limit
  6. Process request
  
  Benefits:
    ✅ Minimal DB hits
    ✅ No race conditions
    ✅ Scales horizontally
    ✅ Faster response time
```

---

## Summary

Authentication and authorization are complementary security mechanisms:

- **Authentication** answers "Who are you?" and verifies identity
- **Authorization** answers "What can you access?" and grants permissions

Choose the appropriate method based on your system architecture, security requirements, and scalability needs.

