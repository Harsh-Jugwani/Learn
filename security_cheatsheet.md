# 🔒 Web Security Cheat Sheet (Quick Reference Guide)

A comprehensive guide to protecting web applications from common vulnerabilities and attacks.

---

## 📌 Table of Contents

1. [XSS (Cross-Site Scripting)](#xss-cross-site-scripting)
2. [iFrame Protection & Clickjacking](#iframe-protection--clickjacking)
3. [Security Headers](#security-headers)
4. [HTTPS](#https)
5. [Input Validation & Sanitization](#input-validation--sanitization)
6. [SSRF (Server-Side Request Forgery)](#ssrf-server-side-request-forgery)
7. [Dependency Security](#dependency-security)
8. [Compliance & Regulations](#compliance--regulations)

---

## 🎯 XSS (Cross-Site Scripting)

**Injection-based vulnerability where attackers inject malicious scripts into web pages**

### How It Works

```
Attacker Input: www.example.com?name=<script>alert('XSS')</script>
                                 ↓
Browser renders page with attacker's script
                                 ↓
Script executes in victim's browser (with victim's permissions!)
```

### Attack Vector Examples

| Vector | Example |
|--------|---------|
| **Query Parameters** | `?user=<img src=x onerror="alert('XSS')">` |
| **Form Fields** | `<input value="<script>..."/>` |
| **URL Fragments** | `#redirect=javascript:alert('XSS')` |
| **Cookies** | Malicious value in vulnerable cookie handler |
| **Headers** | User-Agent, Referer, etc. |
| **WebSocket** | Messages containing script tags |

### Possible Impacts

| Impact | Details |
|--------|---------|
| **Session Hijacking** | Steal authentication cookies/tokens |
| **Account Takeover** | Perform actions as the victim |
| **Data Exfiltration** | Send sensitive data to attacker's server |
| **Keylogging** | Capture user keystrokes |
| **Phishing** | Inject fake login forms or popups |
| **Defacement** | Modify website content |
| **Malware Distribution** | Redirect to malicious sites |

### 🧠 Real-World Scenario

```javascript
// VULNERABLE CODE
function renderUserProfile(userName) {
  document.getElementById('profile').innerHTML = 
    `<h1>Welcome, ${userName}!</h1>`;
}

// Attack: userName = "<img src=x onerror='alert(document.cookie)'>"
// Result: Cookie stolen!

// FIXED CODE
function renderUserProfile(userName) {
  const h1 = document.createElement('h1');
  h1.textContent = `Welcome, ${userName}!`;  // Safe - uses text, not HTML
  document.getElementById('profile').appendChild(h1);
}
```

### Mitigation Strategies

#### 1️⃣ Input Handling Best Practices

```bash
✅ Maintain a list of all input sources:
   - Query parameters (?param=value)
   - Form fields (POST body)
   - Headers (User-Agent, etc.)
   - Cookies
   - WebSocket messages
   - localStorage / sessionStorage

✅ Validate input format:
   - Email: Must match email pattern
   - Numbers: No special characters
   - URLs: Known domains only

✅ Whitelist approach: Allow only known-safe input
```

#### 2️⃣ Output Encoding (Escaping)

```javascript
// HTML Entity Encoding
function htmlEncode(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .replace(/\//g, '&#x2F;');
}

// Usage
const userInput = '<img src=x onerror="alert(123)">';
console.log(htmlEncode(userInput));
// Output: &lt;img src=x onerror=&quot;alert(123)&quot;&gt;
```

#### 3️⃣ DOM Manipulation Safety

```javascript
// ❌ DANGEROUS METHODS TO AVOID
element.innerHTML = userInput;        // Executes scripts!
document.write(userInput);            // Executes scripts!
eval(userInput);                      // Executes code!

// ✅ SAFE METHODS TO USE
element.textContent = userInput;      // Treats as plain text
element.innerText = userInput;        // Treats as plain text
element.appendChild(document.createTextNode(userInput));

// ✅ Using DOM APIs safely
const el = document.createElement('div');
el.textContent = userInput;  // Safe!
element.appendChild(el);
```

#### 4️⃣ HTML Sanitization Libraries

```javascript
// Using DOMPurify
import DOMPurify from 'dompurify';

const userContent = `
  <h1>Welcome!</h1>
  <img src=x onerror="alert('XSS')">
  <script>alert('Evil')</script>
`;

const clean = DOMPurify.sanitize(userContent);
// Output: <h1>Welcome!</h1><img src="x">
// (dangerous scripts removed, safe HTML kept)

element.innerHTML = clean;  // Now safe!
```

#### 5️⃣ Content Security Policy (CSP) Headers

```http
# Inline script prevention (most secure)
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' https://trusted-cdn.com;
  object-src 'none';
  base-uri 'self'

# Using nonce for inline scripts (trusted)
Content-Security-Policy:
  script-src 'nonce-{RANDOM_VALUE}'
  
# Server-side
// Generate random nonce per request
const nonce = crypto.randomBytes(16).toString('base64');
res.setHeader('Content-Security-Policy', `script-src 'nonce-${nonce}'`);
res.render('template', { nonce }); // Pass to template

# Template
<script nonce="<%= nonce %>">
  // This inline script will execute (trusted by nonce)
  console.log('Safe inline script');
</script>
```

#### 6️⃣ Framework Auto-Escaping

```html
<!-- React (auto-escapes by default) ✅ -->
{/* This is safe */}
<div>{userInput}</div>  {/* XSS prevention built-in */}

<!-- Vue (auto-escapes by default) ✅ -->
<div>{{ userInput }}</div>  {/* Safe */}

<!-- Angular (auto-escapes by default) ✅ -->
<div [textContent]="userInput"></div>  {/* Safe */}

<!-- Never use these in frameworks -->
<div [innerHTML]="userInput"></div>  {/* Dangerous! */}
<div v-html="userInput"></div>       {/* Dangerous! */}
```

#### 7️⃣ HttpOnly Cookies

```http
# Set cookie with HttpOnly flag
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict

# JavaScript cannot access this cookie (even if XSS occurs)
document.cookie  // sessionId NOT visible!
```

### XSS Prevention Checklist

- [ ] All user input identified and tracked
- [ ] Client-side validation implemented
- [ ] Server-side validation enforced
- [ ] Output encoding applied before rendering
- [ ] No `innerHTML` with user data
- [ ] DOMPurify used for user-generated HTML
- [ ] CSP headers configured with nonce
- [ ] HttpOnly cookies enabled
- [ ] Framework auto-escaping verified
- [ ] Security headers set (see Security Headers section)

---

## 🖼️ iFrame Protection & Clickjacking

**Vulnerability where attacker embeds your site in their page to hijack user interactions**

### How Clickjacking Works

```html
<!-- Attacker's Malicious Page -->
<html>
  <body>
    <button style="opacity: 0; position: absolute; top: 100px; left: 100px;">
      Transparent button over iframe
    </button>
    <iframe src="https://yourbank.com/transfer-money" style="width: 100%; height: 100%;"></iframe>
  </body>
</html>

<!-- User clicks what looks like "Download" but actually clicks "Transfer $1000" on your site -->
```

### Possible Impacts

| Impact | Details |
|--------|---------|
| **Clickjacking** | Trick users into clicking hidden buttons (payment, permissions, etc.) |
| **Session Hijacking** | Perform authenticated actions without user consent |
| **Data Theft** | Overlay transparent input fields over password fields |
| **UI Redressing** | Change appearance of legitimate UI to trick users |

### 🧠 Real-World Scenario

```
Bank Site (yourbank.com)
  └─ Transfer Money button at coords (500, 300)

Attacker's Page (evil.com)
  ├─ <iframe src="yourbank.com" ...>
  └─ Hidden button at (500, 300)
     └─ User clicks "Click to win prize!"
        └─ Actually clicked "Transfer Money" on iframe
           └─ Unauthorized bank transfer! 💰
```

### Mitigation Strategies

#### 1️⃣ X-Frame-Options Header

```http
# Prevent ALL framing (recommended for sensitive pages)
X-Frame-Options: DENY

# Allow framing only from same origin
X-Frame-Options: SAMEORIGIN

# Allow framing from specific domain
X-Frame-Options: ALLOW-FROM https://trusted.com
```

#### 2️⃣ Content Security Policy (CSP)

```http
# Only your domain can embed your site
Content-Security-Policy: frame-ancestors 'self'

# Multiple trusted domains
Content-Security-Policy: frame-ancestors 'self' https://trusted1.com https://trusted2.com

# Prevent embedding completely
Content-Security-Policy: frame-ancestors 'none'
```

#### 3️⃣ Secure Cookie Flags

```http
# Comprehensive cookie security
Set-Cookie: sessionId=abc123; 
            HttpOnly;          # JS cannot access
            Secure;            # HTTPS only
            SameSite=Strict;   # Not sent in cross-site requests
            Path=/;            # Available site-wide
            Domain=.example.com  # All subdomains
```

#### 4️⃣ Frame-Busting JavaScript (Legacy)

```javascript
// Detect if page is being framed (fallback for old browsers)
if (window.self !== window.top) {
  window.top.location = window.self.location;
}

// Better: Use X-Frame-Options instead (this can be bypassed)
// But use as defense-in-depth
```

#### 5️⃣ User Action Confirmation

```javascript
// For critical actions, require additional verification
function transferMoney(amount) {
  // Show native browser confirm dialog (cannot be overlaid)
  const confirmed = confirm(`Transfer $${amount}? This cannot be undone.`);
  
  if (!confirmed) return;
  
  // Require re-authentication
  const password = prompt("Enter your password again:");
  if (!verifyPassword(password)) {
    alert("Invalid password");
    return;
  }
  
  // Now proceed with transfer
  proceedTransfer(amount);
}
```

#### 6️⃣ CAPTCHA for Sensitive Actions

```html
<!-- Require CAPTCHA for sensitive operations -->
<form id="transferForm">
  <input type="number" name="amount" placeholder="Amount">
  <button type="button" onclick="requireCaptcha()">Transfer</button>
</form>

<script>
function requireCaptcha() {
  // Trigger CAPTCHA verification
  // Cannot be automated or bypassed via clickjacking
  grecaptcha.ready(() => {
    grecaptcha.execute('SITE_KEY', {action: 'transfer'})
      .then(token => {
        // Verify token server-side
        submitTransfer(token);
      });
  });
}
</script>
```

### iFrame Protection Checklist

- [ ] X-Frame-Options header set (DENY or SAMEORIGIN)
- [ ] CSP frame-ancestors configured
- [ ] Cookies use HttpOnly, Secure, SameSite=Strict
- [ ] Critical actions require user confirmation
- [ ] CAPTCHA on sensitive operations
- [ ] JavaScript frame-busting code added (as fallback)
- [ ] Regular security testing for clickjacking

---

## 🔐 Security Headers

**HTTP headers that instruct browsers to enforce security policies**

### Complete Security Headers Configuration

```http
# 1. X-Powered-By - Hide server technology
# Remove or override (prevents attackers from targeting known vulnerabilities)
X-Powered-By:                    # Remove this header
# OR set custom value
X-Powered-By: NewTech/2.0

# 2. X-XSS-Protection - Legacy XSS protection
# Disable in modern apps (use CSP instead)
X-XSS-Protection: 0

# 3. Referrer-Policy - Control referrer information
# Prevent leaking sensitive info in referrer URL
Referrer-Policy: no-referrer
# OR
Referrer-Policy: strict-origin-when-cross-origin

# 4. X-Content-Type-Options - Prevent MIME type sniffing
# Force browser to respect declared Content-Type
X-Content-Type-Options: nosniff

# 5. Strict-Transport-Security (HSTS)
# Force HTTPS for all future requests
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
# max-age: 31536000 seconds = 1 year
# includeSubDomains: Apply to all subdomains
# preload: Add to browser preload list (prevents initial HTTP request)

# 6. Permissions-Policy - Restrict browser features
Permissions-Policy: 
  camera=(),
  microphone=(),
  geolocation=(self),
  accelerometer=(),
  gyroscope=()

# 7. Content-Security-Policy - Comprehensive attack prevention
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM}' https://trusted-cdn.com;
  style-src 'self' https://fonts.googleapis.com;
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'self';
  base-uri 'self';
  object-src 'none';
  report-uri /csp-report

# 8. Cross-Origin-Resource-Policy (CORP)
# Prevent cross-origin resource loading
Cross-Origin-Resource-Policy: same-site

# 9. Cross-Origin-Opener-Policy (COOP)
# Isolate browsing context from cross-origin pages
Cross-Origin-Opener-Policy: same-origin

# 10. Cross-Origin-Embedder-Policy (COEP)
# Require CORS headers for cross-origin resources
Cross-Origin-Embedder-Policy: require-corp
```

### Security Headers Quick Reference

| Header | Purpose | Recommended Value |
|--------|---------|-------------------|
| **X-Powered-By** | Hide server info | Remove or customize |
| **X-XSS-Protection** | Legacy XSS filter | `0` (use CSP instead) |
| **Referrer-Policy** | Control referrer info | `no-referrer` or `strict-origin` |
| **X-Content-Type-Options** | Prevent MIME sniffing | `nosniff` |
| **HSTS** | Force HTTPS | `max-age=31536000; includeSubDomains; preload` |
| **Permissions-Policy** | Restrict features | Disable unnecessary features |
| **CSP** | XSS/injection prevention | Comprehensive policy for site |
| **CORP** | Cross-origin resources | `same-site` |
| **COOP** | Browsing context isolation | `same-origin` |
| **COEP** | Cross-origin embedding | `require-corp` |

### Implementation Examples

**Node.js/Express**
```javascript
const helmet = require('helmet');
const app = require('express')();

// Use Helmet to set security headers
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-{random}'"],
      styleSrc: ["'self'", "https://fonts.googleapis.com"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
  referrerPolicy: { policy: "strict-origin-when-cross-origin" },
}));
```

**ASP.NET**
```csharp
// In Startup.cs or Program.cs
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Strict-Transport-Security", 
        "max-age=31536000; includeSubDomains; preload");
    await next();
});
```

---

## 🔒 HTTPS

**Secure protocol using TLS/SSL for encrypted web communication**

### Why HTTPS Matters

| Benefit | Importance |
|---------|-----------|
| **Data Encryption (TLS)** | Intercepted data is unreadable |
| **Authentication** | SSL/TLS cert verifies site ownership |
| **Data Integrity (MAC)** | Checksum ensures data not tampered |
| **Data Privacy** | Prevents eavesdropping & sniffing |
| **Faster Load Times** | HTTP/2 support with TLS |
| **User Trust** | Lock icon builds confidence |
| **Regulatory Compliance** | GDPR, PCI-DSS require it |
| **Phishing Protection** | Valid cert prevents impersonation |
| **SEO Ranking Boost** | Google ranks HTTPS higher |
| **No Browser Warnings** | Avoids "Not Secure" label |

### HTTPS Implementation

```
┌─────────────────────────────────────────────┐
│ Client (Browser)                            │
│                                             │
│  1. GET https://example.com                │
│     └─ TLS Handshake Initiated              │
│                                             │
│  2. Server sends certificate                │
│     └─ Browser verifies certificate         │
│                                             │
│  3. Symmetric key exchange                  │
│     └─ Encryption key established           │
│                                             │
│  4. Encrypted communication                 │
│     └─ All data encrypted & signed          │
└─────────────────────────────────────────────┘
         ↕ (Encrypted Channel)
├─────────────────────────────────────────────┐
│ Server                                      │
│                                             │
│  - Presents SSL/TLS certificate             │
│  - Validates certificate (not self-signed)  │
│  - Negotiates encryption algorithm          │
│  - Maintains encrypted session              │
└─────────────────────────────────────────────┘
```

### Certificate Validation

```javascript
// Client-side considerations
// ❌ Self-signed certificates (BAD for production)
https://self-signed.com  // Shows browser warning

// ✅ Valid signed certificates (GOOD)
https://valid-cert.com   // Green lock icon, trusted

// Certificate must be:
// - Issued by trusted Certificate Authority (CA)
// - Not expired
// - Matches domain name
// - Not revoked (CRL or OCSP check)
```

---

## 📝 Input Validation & Sanitization

**Process of verifying & cleaning all incoming data before processing**

### The Security Layering Model

```
┌────────────────────────────────────┐
│  Browser (Client-Side)             │
│  ├─ Form validation                │
│  ├─ Client-side checks             │
│  └─ UX feedback                    │
│                                    │
│  ⚠️ CAN BE BYPASSED!               │
└────────────────────────────────────┘
              ↓
┌────────────────────────────────────┐
│  NETWORK (Attacker can intercept)  │
│                                    │
│  ⚠️ DATA CAN BE MODIFIED!          │
└────────────────────────────────────┘
              ↓
┌────────────────────────────────────┐
│  SERVER (Mandatory)                │
│  ├─ Validate all inputs again      │
│  ├─ Sanitize & encode              │
│  ├─ Apply business rules           │
│  └─ Secure processing              │
│                                    │
│  ✅ ALWAYS VALIDATE HERE!          │
└────────────────────────────────────┘
```

### Best Practices

#### 1️⃣ Input Sources Inventory

```javascript
// Track ALL possible input sources
const inputSources = {
  queryParams: new URL(window.location).searchParams,
  formFields: document.querySelectorAll('input'),
  headers: {
    'User-Agent': request.headers['user-agent'],
    'Referer': request.headers['referer'],
  },
  cookies: document.cookie,
  websocket: ws.onmessage,
  storage: localStorage,
};
```

#### 2️⃣ Validation Rules by Type

```javascript
// Email validation
function validateEmail(email) {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

// Phone number
function validatePhone(phone) {
  const regex = /^(\+\d{1,3})?\d{10}$/;  // E.164 format
  return regex.test(phone);
}

// URL validation
function validateURL(url) {
  try {
    new URL(url);
    return true;
  } catch {
    return false;
  }
}

// Number range
function validateAge(age) {
  const num = parseInt(age, 10);
  return !isNaN(num) && num >= 0 && num <= 150;
}

// Alphanumeric only
function validateUsername(username) {
  return /^[a-zA-Z0-9_-]{3,20}$/.test(username);
}
```

#### 3️⃣ Server-Side Validation (Node.js)

```javascript
// Using middleware for consistent validation
const { body, validationResult } = require('express-validator');

app.post('/register', [
  // Validate email
  body('email')
    .isEmail()
    .normalizeEmail()
    .toLowerCase(),
  
  // Validate password
  body('password')
    .isLength({ min: 8 })
    .withMessage('Password must be 8+ characters')
    .matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .withMessage('Password must contain uppercase, lowercase, number'),
  
  // Validate username
  body('username')
    .matches(/^[a-zA-Z0-9_-]{3,20}$/)
    .withMessage('Invalid username format'),
  
], (req, res) => {
  // Check for validation errors
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }
  
  // Validation passed, proceed with registration
  registerUser(req.body);
});
```

#### 4️⃣ File Upload Restrictions

```javascript
// Validate file type and size
function validateFileUpload(file) {
  // Allowed MIME types (server-side check)
  const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
  if (!allowedTypes.includes(file.mimetype)) {
    throw new Error('Invalid file type');
  }
  
  // File size limit (5 MB)
  const maxSize = 5 * 1024 * 1024;
  if (file.size > maxSize) {
    throw new Error('File too large');
  }
  
  // Check file magic bytes (signature)
  const fileSignature = file.buffer.slice(0, 4);
  if (!isValidFileSignature(fileSignature)) {
    throw new Error('Invalid file format');
  }
  
  // Store outside web root (not directly accessible)
  const uploadDir = '/secure/uploads/';  // Not /public/uploads/
  fs.writeFileSync(`${uploadDir}${file.filename}`, file.buffer);
}
```

#### 5️⃣ Output Encoding

```javascript
// Encode data before displaying
function encodeOutput(data) {
  return data
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .replace(/\//g, '&#x2F;');
}

// Usage
const userInput = "<img src=x onerror='alert(1)'>";
console.log(encodeOutput(userInput));
// Safe output: &lt;img src=x onerror=&#x27;alert(1)&#x27;&gt;
```

#### 6️⃣ Using Trusted Frameworks

```javascript
// ASP.NET Core - Built-in validation
[HttpPost]
public IActionResult Register([FromBody] RegisterModel model)
{
    if (!ModelState.IsValid) {
        return BadRequest(ModelState);
    }
    // Model automatically validated
}

// Django - Built-in validators
from django import forms

class ContactForm(forms.Form):
    email = forms.EmailField()
    message = forms.CharField(
        max_length=1000,
        validators=[MinLengthValidator(10)]
    )

// Laravel - Validation rules
$validated = $request->validate([
    'email' => 'required|email|unique:users',
    'password' => 'required|min:8|confirmed',
]);
```

### Input Validation Checklist

- [ ] All input sources documented
- [ ] Client-side validation for UX
- [ ] Server-side validation mandatory
- [ ] Whitelist approach (allow good data, not block bad)
- [ ] Regex patterns defined for each input type
- [ ] File upload: type, size, extension validated
- [ ] Output encoded before rendering
- [ ] Trusted framework/library used
- [ ] Error messages don't leak sensitive info
- [ ] Validation tested with attack payloads

---

## 🚀 SSRF (Server-Side Request Forgery)

**Vulnerability where server makes unintended requests, often to internal systems**

### How SSRF Works

```
User Request
  └─ /fetch-url?url=http://example.com
     └─ Server makes GET request to user-provided URL
        
NORMAL CASE:
  Server → https://example.com ✅

ATTACK CASE:
  Server → http://169.254.169.254/latest/meta-data/ (AWS credentials!)
  Server → http://localhost:8080/admin (Internal admin panel!)
  Server → http://192.168.1.1/router-config (Network router!)
```

### Possible Impacts

| Impact | Details |
|--------|---------|
| **Access Internal APIs** | Call internal services not exposed publicly |
| **Steal Cloud Credentials** | AWS: `169.254.169.254/latest/meta-data/iam/security-credentials/` |
| **Network Scanning** | Enumerate internal IPs and services |
| **Internal Data Theft** | Read internal databases, files, config |
| **Admin Panel Access** | Bypass authentication on internal admin pages |
| **Port Scanning** | Determine which ports are open internally |

### 🧠 Real-World Scenario

```
Web Service: https://imageproxy.com/image?imageUrl=...

LEGITIMATE USE:
GET /image?imageUrl=https://cdn.example.com/photo.jpg
Response: Photo downloaded ✅

SSRF ATTACK #1 - AWS Credentials:
GET /image?imageUrl=http://169.254.169.254/latest/meta-data/iam/security-credentials/
Response: AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE
          AWS_SECRET_ACCESS_KEY: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Attacker now has AWS credentials! 💀

SSRF ATTACK #2 - Internal Admin:
GET /image?imageUrl=http://localhost:8080/admin
Response: Admin dashboard HTML
Attacker accesses admin panel! 🔓
```

### Mitigation Strategies

#### 1️⃣ Input Validation & Whitelisting

```javascript
// Whitelist allowed domains only
const ALLOWED_DOMAINS = [
  'cdn.example.com',
  'images.trusted-service.com',
  'api.public-provider.com',
];

function validateImageUrl(urlString) {
  try {
    const url = new URL(urlString);
    
    // Check protocol (HTTPS only)
    if (url.protocol !== 'https:') {
      throw new Error('Only HTTPS URLs allowed');
    }
    
    // Check hostname against whitelist
    if (!ALLOWED_DOMAINS.includes(url.hostname)) {
      throw new Error('Domain not whitelisted');
    }
    
    return url;
  } catch (error) {
    throw new Error(`Invalid URL: ${error.message}`);
  }
}
```

#### 2️⃣ Block Private IP Ranges

```javascript
function isPrivateIP(hostname) {
  // Resolve hostname to IP
  const ip = dns.sync.resolve4(hostname)[0];
  
  // Private IP ranges to block
  const privateRanges = [
    /^127\./,                    // Loopback
    /^10\./,                     // 10.0.0.0/8
    /^172\.(1[6-9]|2[0-9]|3[01])\./, // 172.16.0.0/12
    /^192\.168\./,               // 192.168.0.0/16
    /^169\.254\./,               // Link-local
    /^::1$/,                     // IPv6 loopback
    /^fc00:/,                    // IPv6 private
    /^fe80:/,                    // IPv6 link-local
  ];
  
  return privateRanges.some(range => range.test(ip));
}

// Usage
try {
  const url = new URL(userProvidedURL);
  if (isPrivateIP(url.hostname)) {
    throw new Error('Access to private networks forbidden');
  }
  // Proceed with request
} catch (error) {
  console.error(`SSRF blocked: ${error.message}`);
}
```

#### 3️⃣ Disable Unnecessary Protocols

```javascript
function fetchURL(urlString) {
  const url = new URL(urlString);
  
  // Allow only HTTP and HTTPS
  const allowedProtocols = ['http:', 'https:'];
  if (!allowedProtocols.includes(url.protocol)) {
    throw new Error(`Protocol ${url.protocol} not allowed`);
  }
  
  // Block dangerous protocols
  const blockedProtocols = [
    'file:',     // Local file access
    'ftp:',      // FTP protocol
    'gopher:',   // Gopher protocol
    'tftp:',     // TFTP protocol
    'data:',     // Data URI scheme
    'dict:',     // Dictionary protocol
    'ldap:',     // LDAP protocol
  ];
  
  if (blockedProtocols.includes(url.protocol)) {
    throw new Error(`Protocol ${url.protocol} is blocked`);
  }
  
  return fetch(url);
}
```

#### 4️⃣ Network Segmentation

```
┌─ Internet ────┐
│               │
└─── WAF ───────┘
      ↓
┌─ DMZ (Public Services) ───┐
│  - Web Server              │
│  - API Gateway             │
│  - Image Proxy Service     │
│                            │
│  ⚠️ CAN access internal net│
└─ Firewall ────────────────┘
      ↓
┌─ Internal Network (Private) ────┐
│  - Database                     │
│  - Admin Panel (localhost:8080) │
│  - Redis Cache                  │
│  - AWS Metadata (169.254...)    │
│                                 │
│  ✅ Restricted by firewall      │
└─────────────────────────────────┘
```

**Firewall Rules**
```
DMZ → Internal: DENIED (default deny)
Internal → DMZ: ALLOWED (certain ports only)
DMZ → Internet: ALLOWED (port 443 only via proxy)
DMZ → Metadata Service (169.254.169.254): DENIED
```

#### 5️⃣ Cloud Metadata Protection

```bash
# AWS - Block metadata service access from web servers
# In security group for web servers:
OUTBOUND: 169.254.169.254/32 DENY

# Or use Instance Metadata Service Version 2 (IMDSv2)
# which requires authentication token
curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
```

#### 6️⃣ Request Timeout & Size Limits

```javascript
function fetchURLSafely(urlString, options = {}) {
  const url = new URL(urlString);
  
  // Validate URL first
  if (isPrivateIP(url.hostname)) {
    throw new Error('Private IP access denied');
  }
  
  // Set security timeouts
  const controller = new AbortController();
  const timeoutId = setTimeout(
    () => controller.abort(),
    5000  // 5 second timeout
  );
  
  return fetch(url, {
    signal: controller.signal,
    timeout: 5000,
    size: 10 * 1024 * 1024,  // 10 MB max size
  })
    .finally(() => clearTimeout(timeoutId));
}
```

### SSRF Prevention Checklist

- [ ] Whitelist of allowed domains defined
- [ ] Block private/internal IP ranges
- [ ] Only HTTP/HTTPS protocols allowed
- [ ] URL validation server-side (not client)
- [ ] DNS rebinding protection
- [ ] Request timeouts set (5-10 sec)
- [ ] Response size limits enforced
- [ ] Network segmentation in place
- [ ] Cloud metadata service blocked
- [ ] Security testing performed

---

## 🔧 Dependency Security

**Ensuring all third-party packages are safe and up-to-date**

### The Supply Chain Risk

```
Your Application
  ├─ NPM Package A (latest)
  │   ├─ NPM Package B (v1.0.0 - VULNERABLE!)
  │   └─ NPM Package C (v2.1.0)
  ├─ NPM Package D (outdated, CVE-2023-12345)
  └─ Custom Code

One compromised dependency = Your entire app compromised!
```

### Vulnerability Detection Tools

| Tool | Use Case | Features |
|------|----------|----------|
| **npm audit** | NPM packages | Built-in, quick scan |
| **Dependabot** | GitHub repos | Auto pull requests |
| **Snyk** | Multiple languages | CLI + web dashboard |
| **CodeQL** | Code analysis | Deep static analysis |
| **OWASP Dependency-Check** | Open source | Comprehensive DB |
| **Yarn audit** | Yarn packages | Built-in for Yarn |

### Best Practices

#### 1️⃣ Regular Audits

```bash
# NPM
npm audit                  # Scan for vulnerabilities
npm audit --json           # Machine-readable output
npm audit fix              # Auto-fix vulnerable packages
npm audit fix --force      # Force update even if major version

# Yarn
yarn audit                 # Check for vulnerabilities

# Snyk (comprehensive)
snyk test                  # Scan project
snyk test --severity=high  # Only show high/critical
snyk monitor               # Continuous monitoring
```

#### 2️⃣ Version Pinning & Locking

```json
// package.json
{
  "dependencies": {
    "express": "4.18.2",      // Exact version (pinned)
    "lodash": "~4.17.0",      // Patch updates allowed (~)
    "react": "^18.0.0"        // Minor updates allowed (^)
  }
}

// package-lock.json or yarn.lock
// Locks ALL transitive dependency versions
// ALWAYS commit these files!
```

#### 3️⃣ Trusted Sources Only

```bash
# Use official registries
npm install express           # From npmjs.com (official)

# Verify package before install
npm view express@4.18.2       # Check metadata
npm view express repository   # Check source repo

# Use private registry for internal packages
# Nexus, Artifactory, or AWS CodeArtifact
npm set registry https://private-registry.example.com
```

#### 4️⃣ Remove Unused Dependencies

```bash
# Find unused packages
npx depcheck

# Remove unused package
npm uninstall unused-package

# Small attack surface = Reduced risk
```

#### 5️⃣ Monitor Security Advisories

```bash
# Subscribe to GitHub alerts
# Settings → Security → Dependabot alerts

# Use Snyk for real-time monitoring
snyk monitor               # Check on demand
# OR GitHub Actions automated scanning
```

#### 6️⃣ Secure Build Pipeline

```yaml
# .github/workflows/security.yml (GitHub Actions)
name: Security Scan
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: Run Snyk
        run: npx snyk test
      
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          path: './node_modules'
          format: 'JSON'
          
      - name: Report
        if: failure()
        run: echo "Security issues found - Fix before merge"
```

### Dependency Security Checklist

- [ ] Audit run regularly (daily in CI/CD)
- [ ] Security advisories subscribed
- [ ] Outdated packages updated
- [ ] Unused dependencies removed
- [ ] Dependencies from trusted sources only
- [ ] package-lock.json committed
- [ ] CI/CD blocks on high/critical vulnerabilities
- [ ] Annual security audit scheduled
- [ ] Supply chain risk documented

---

## 📋 Compliance & Regulations

**Industry standards mandatory for secure web applications**

### Major Compliance Frameworks

| Framework | Focus | Penalties |
|-----------|-------|-----------|
| **GDPR** (EU) | User data privacy | €20M or 4% revenue |
| **PCI-DSS** | Payment data | $25K-$100K per month |
| **HIPAA** (US) | Healthcare data | $100-$50K per violation |
| **SOC 2** | Service provider security | Compliance certification |
| **ISO 27001** | Information security | Industry standard cert |
| **CCPA** (California) | User data privacy | $2,500-$7,500 per violation |

### GDPR - General Data Protection Regulation (EU)

**Applies to: Any site with EU users**

```yaml
Key Requirements:
  1. Data Collection: Explicit consent before collecting
  2. Privacy Policy: Clear, accessible, understandable
  3. Storage: Only keep data as long as needed
  4. Security: Encrypt sensitive data
  5. Right to Delete: Users can request data deletion
  6. Data Breaches: Report within 72 hours
  7. DPA: Document data processing activities
  8. Third Parties: Ensure subprocessors are compliant

Penalties:
  - Administrative fines up to €20 million
  - OR 4% of annual global revenue (whichever is higher)
  
Example: 
  Facebook violated GDPR
  Fine: €1.2 billion (2023 decision)
```

### PCI-DSS - Payment Card Industry Data Security Standard

**Applies to: Any site handling payment cards**

```yaml
Key Requirements:
  1. Firewall: Install & maintain firewalls
  2. Encryption: Encrypt card data in transit & at rest
  3. Antivirus: Maintain antivirus software
  4. Updates: Patch systems regularly
  5. Access Control: Restrict card data access
  6. Monitoring: Monitor & log all network access
  7. Security Test: Regular penetration testing
  8. Security Policy: Written security policy

Compliance Levels (based on transaction volume):
  Level 1: > 6 million transactions/year (strictest)
  Level 2: 1-6 million transactions/year
  Level 3: 20K-1 million transactions/year
  Level 4: < 20K transactions/year

Penalties:
  - $25K-$100K per month for non-compliance
  - Lost payment processing ability
```

### HIPAA - Health Insurance Portability (US Healthcare)

**Applies to: Healthcare apps handling patient data**

```yaml
Key Requirements:
  1. Administrative: Written policies & training
  2. Physical: Secure servers, controlled access
  3. Technical: Encryption, audit logs, access controls
  4. Portable Rule: Patient data portability
  5. Privacy Rule: Limit data to minimum needed
  6. Security Rule: Technical safeguards

Protected Health Information (PHI):
  - Patient names, SSN, medical records
  - Insurance info, payment data
  
Penalties:
  - $100 to $50,000 per violation
  - Up to $1.5 million per year for category
  - Criminal penalties up to $250,000
```

### Implementation Checklist

```yaml
# Before launching:
1. Identify applicable regulations
   ├─ Where are users located?
   ├─ What data type stored?
   └─ What industry?

2. Document compliance:
   ├─ Data inventory
   ├─ Processing locations
   ├─ Third-party processors
   └─ Retention policies

3. Technical controls:
   ├─ Encryption (at-rest & in-transit)
   ├─ Access controls (IAM)
   ├─ Audit logging
   ├─ Backup & recovery
   └─ Incident response

4. Legal review:
   ├─ Privacy policy
   ├─ Terms of service
   ├─ Data processing agreement
   └─ Cookie consent

5. Testing:
   ├─ Security audit
   ├─ Penetration test
   ├─ Vulnerability scan
   └─ Compliance verification
```

---

## 📊 Security Testing Checklist

Before deploying to production:

### Automated Tools

- [ ] SAST (Static Application Security Testing)
  - CodeQL, Snyk, SonarQube
- [ ] DAST (Dynamic Application Security Testing)
  - OWASP ZAP, Burp Suite
- [ ] Dependency scanning
  - npm audit, OWASP Dependency-Check
- [ ] Infrastructure scanning
  - Trivy, CloudMapper

### Manual Testing

- [ ] XSS injection attacks
- [ ] SQL injection attempts
- [ ] CSRF token validation
- [ ] Authentication bypass attempts
- [ ] Authorization checks
- [ ] Sensitive data exposure
- [ ] API abuse testing
- [ ] Rate limiting verification

### Security Headers Validation

```bash
# Check all security headers
curl -I https://example.com | grep -E "^(X-|Strict|Content|Referrer)"

# Should see:
# Content-Security-Policy: ...
# X-Content-Type-Options: nosniff
# X-Frame-Options: DENY
# Strict-Transport-Security: ...
# Referrer-Policy: ...
```

---

## 🚀 Quick Reference Matrix

| Attack Type | Mitigation |
|------------|-----------|
| **XSS** | CSP, escape output, DOMPurify, HttpOnly cookies |
| **Clickjacking** | X-Frame-Options, CSP frame-ancestors, SameSite cookies |
| **SSRF** | Whitelist URLs, block private IPs, network segmentation |
| **SQL Injection** | Parameterized queries, input validation, ORM |
| **CSRF** | CSRF tokens, SameSite cookies, Origin header |
| **XXE** | Disable external entities, validate XML |
| **Unvalidated Redirect** | Whitelist redirect URLs, validate domains |

---

## 🎯 Security Best Practices Summary

✅ **Always Do**
- ✓ Validate input on server-side
- ✓ Use HTTPS everywhere
- ✓ Set comprehensive security headers
- ✓ Keep dependencies updated
- ✓ Use HttpOnly cookies
- ✓ Implement CSP
- ✓ Regular security audits
- ✓ Log security events

❌ **Never Do**
- ✗ Trust client-side validation alone
- ✗ Expose server technology
- ✗ Use self-signed SSL certificates
- ✗ Hardcode secrets
- ✗ Use `eval()` or `innerHTML` with user data
- ✗ Ignore security warnings
- ✗ Store sensitive data in localStorage
- ✗ Use GET for sensitive data

---

## 🔐 Your Security Journey

You now have a comprehensive reference covering:
- ✅ Cross-Site Scripting (XSS)
- ✅ iFrame & Clickjacking
- ✅ Security Headers
- ✅ HTTPS/TLS
- ✅ Input Validation
- ✅ Server-Side Request Forgery
- ✅ Dependency Analysis
- ✅ Regulatory Compliance

Stay secure! 🛡️
