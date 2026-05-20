# Lecture 8 — Authentication and Authorisation for Backend Engineers
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=A95rliroC8Q)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Every app you've ever logged into uses authentication and authorisation. This lecture covers both — deeply. From the historical evolution of auth, to sessions, JWTs, cookies, stateful vs stateless auth, OAuth 2.0, OpenID Connect, RBAC, and critical security pitfalls like timing attacks and information leakage.

This is one of the most important topics in backend engineering.

---

## 1. The Two Core Questions

These two questions are the foundation of everything in this lecture:

| Concept | Question it answers | Example |
|---|---|---|
| **Authentication (AuthN)** | *Who are you?* | Logging in with email + password |
| **Authorisation (AuthZ)** | *What can you do?* | Only admins can delete users |

> 💡 Authentication happens first. You must know WHO someone is before you can decide WHAT they're allowed to do.

---

## 2. Historical Context — How Auth Evolved

Understanding why things are the way they are makes them stick. Here's the full evolution:

### Pre-Industrial — Implicit Trust
- Identity = recognition by trusted community members
- A village elder could vouch for someone
- Deals sealed with a handshake
- **Problem:** Doesn't scale beyond familiar circles

### Medieval Period — Seals (First Tokens)
- Wax seals with unique patterns attached to documents
- Acted like physical signatures — if you possess the seal, you're authenticated
- **First authentication tokens** — based on something you *have*
- **Problem:** Seals could be forged → first bypass attacks in history
- Led to: watermarks, encrypted codes, early cryptographic thinking

### Industrial Revolution — Passphrases (Shared Secrets)
- Telegraph operators used pre-agreed passphrases
- Static passwords — one fixed string, shared between parties
- **Principle shift:** From something you *have* → something you *know*
- Direct ancestor of modern passwords

### 1961 — MIT and the First Digital Passwords
- MIT's Compatible Time-Sharing System (CTSS) introduced passwords for multi-user computers
- Users could share a computer without seeing each other's data
- **Critical mistake:** Passwords stored in **plain text**
- Someone printed the password file → world's first password breach
- **This incident motivated** the philosophy: never store passwords in plain text
- Led to: **hashing algorithms** — cryptographic transformation of passwords into irreversible fixed-length strings

### 1970s — Asymmetric Cryptography
- Whitfield Diffie and Martin Hellman invented the **Diffie-Hellman key exchange**
- Two parties can establish a shared secret over an untrusted network without ever meeting
- This became the backbone of **all modern authentication protocols**
- Also: PKI (Public Key Infrastructure) systems emerged
- Also: **Kerberos** — ticket-based authentication, a direct precursor to token-based auth

### 1990s — MFA (Multi-Factor Authentication)
- Simple username/password was vulnerable to brute force and dictionary attacks
- MFA combined **three principles**:
  - Something you **know** (password, PIN)
  - Something you **have** (smart card, OTP generator)
  - Something you **are** (fingerprint, retina scan — biometrics)
- Combining layers made attacks exponentially harder

### 21st Century — Modern Auth
- Cloud computing, mobile, API-first architectures
- Required advanced, scalable auth frameworks
- Brought: **OAuth 2.0**, **JWTs**, **OpenID Connect**, Zero Trust Architecture, Passwordless auth (WebAuthn)

### The Future
- **Decentralised Identity** — blockchain-based identity management (experimental)
- **Behavioural Biometrics** — identifying users by how they type, move the mouse, etc.
- **Post-Quantum Cryptography** — current RSA and public-key algorithms will be broken by quantum computers. New algorithms being developed that are quantum-resistant.

---

## 3. Three Core Components of Auth — Sessions, JWTs, Cookies

Before diving into auth types, understand these three building blocks. They appear in every auth discussion.

---

### Sessions

**Why sessions exist:**
HTTP is stateless. Every request is independent — the server has no memory of the last request. This was fine for early static web pages (you just read content). But once web apps became dynamic (shopping carts, logged-in state), statelessness became a problem.

> How can a user stay logged in while navigating between pages if the server forgets them on every request?

Sessions solved this by giving the server **temporary memory** for each user.

**How sessions work:**

```
Step 1 — User logs in:
  Client sends: email + password

Step 2 — Server creates session:
  Server validates credentials
  Server generates a unique session ID (random cryptographic string)
  Server stores: { sessionId: "abc123", userId: 42, role: "admin", cartItems: [...] }
  → Saved in Redis or database

Step 3 — Server sends session ID to client:
  Server sets an HTTP-only cookie: sessionId=abc123
  (HTTP-only = JavaScript cannot read this cookie — security feature)

Step 4 — All subsequent requests:
  Browser automatically sends the cookie with every request
  Server reads sessionId from cookie
  Server looks up "abc123" in Redis → gets userId, role, cart items
  Server knows who you are ✅

Step 5 — Session expiry:
  Sessions have an expiry (e.g., 15 minutes)
  After expiry, server creates a new session
```

**Storage evolution of sessions:**
- Early: file-based sessions on server disk → didn't scale
- Then: database-backed sessions → better but slower
- Modern: **Redis** (in-memory store) → extremely fast lookups, scales well

---

### JWTs (JSON Web Tokens)

**Why JWTs exist:**
By the mid-2000s, web apps were globally distributed. Sessions had problems:
- Storing session data for millions of users = expensive memory
- Distributed servers across regions needed to **sync session data** → latency + complexity
- If Server A created your session, Server B doesn't know about it

JWTs solved this with **statelessness** — all user data is *in the token itself*, not stored on the server.

**JWT Structure:**
A JWT is a base64-encoded string with three parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiI0MiIsInJvbGUiOiJhZG1pbiJ9.SflKxwR...
        ↑                           ↑                              ↑
    HEADER                      PAYLOAD                        SIGNATURE
```

**Part 1 — Header:**
```json
{
  "alg": "HS256",    ← signing algorithm used
  "typ": "JWT"
}
```

**Part 2 — Payload (the data):**
```json
{
  "sub": "42",           ← subject = user's ID (standard field)
  "iat": 1695000000,     ← issued at (timestamp)
  "exp": 1695003600,     ← expiry (timestamp)
  "name": "Pravan",      ← optional: user's name
  "role": "admin"        ← optional: user's role
}
```

**Part 3 — Signature:**
```
HMACSHA256(
  base64(header) + "." + base64(payload),
  SECRET_KEY
)
```

The server signs the token with its **secret key**. If anyone tampers with the payload, the signature won't match when the server re-verifies it → tampered token detected and rejected.

> ⚠️ JWT payload is **base64-encoded, NOT encrypted**. Anyone can decode and read it. Never put passwords, credit cards, or sensitive data in a JWT payload.

**JWT Advantages:**
| Advantage | Explanation |
|---|---|
| **Stateless** | No server-side storage needed — token carries everything |
| **Scalable** | Any server with the secret key can verify any token — no sync needed |
| **Portable** | Can be sent in headers, cookies, URL params — works everywhere |
| **Microservice-friendly** | All services share the secret key and can independently verify users |

**JWT Disadvantages:**
| Disadvantage | Explanation |
|---|---|
| **Can't revoke** | If someone steals your token, they can use it until it expires — you can't invalidate it without changing the secret key (which logs everyone out) |
| **No real-time control** | Server has no visibility into active sessions |

**The Hybrid Approach** (solving revocation):
Combine JWT (stateless verification) with a **blacklist** in Redis:

```
Request arrives with JWT
  ↓
Verify JWT signature ✅
  ↓
Check blacklist in Redis: is this JWT blacklisted? ❌
  ↓
If not blacklisted → user is authenticated ✅
```

When you need to revoke a specific user's access:
```
Add their JWT (or user ID) to the blacklist in Redis
→ Next request: JWT is valid BUT blacklisted → rejected ✅
```

**The trade-off question:**
If you're doing a Redis lookup anyway, why not just use sessions? This is a valid question. For most apps, the recommendation is:
- Use an **external auth provider** (Auth0, Clerk) in production — they handle all these complexities
- Implement your own auth when learning — understand every trade-off
- In production, don't reinvent auth unless you're very confident

---

### Cookies

A cookie is a way for a **server to store a small piece of information in the user's browser**. That information is automatically sent back to the server with every subsequent request.

**How cookies work in auth:**

```
1. User logs in successfully
2. Server sets a cookie:
   Set-Cookie: authToken=eyJ...; HttpOnly; Secure; SameSite=Strict
3. Browser stores this cookie
4. Every subsequent request to that server automatically includes:
   Cookie: authToken=eyJ...
5. Server reads the cookie, extracts the token, verifies the user
```

**Important cookie flags:**
| Flag | What it does | Why important |
|---|---|---|
| `HttpOnly` | JavaScript cannot read this cookie | Prevents XSS attacks from stealing tokens |
| `Secure` | Cookie only sent over HTTPS | Prevents interception over HTTP |
| `SameSite=Strict` | Cookie not sent in cross-site requests | Prevents CSRF attacks |

**Key point:** A cookie set by `api.example.com` is only sent to `api.example.com`. The server can't access cookies set by other servers. This is a browser security guarantee.

---

## 4. Types of Authentication

### Type 1 — Stateful Authentication (Session-Based)

```
Client                           Server                    Redis
  |                                |                          |
  |-- email + password ----------->|                          |
  |                    validates   |                          |
  |                    generates sessionId                    |
  |                                |-- store(sessionId, user)->|
  |<-- Set-Cookie: sessionId=abc --|                          |
  |                                |                          |
  |-- GET /profile (cookie: abc) ->|                          |
  |                                |-- get(abc) ------------->|
  |                                |<-- { userId:42, role } --|
  |<-- 200 { profile data } -------|                          |
```

**Pros:**
- Centralized control — you can see all active sessions
- Easy to revoke — just delete the session from Redis
- Ideal for web apps

**Cons:**
- Needs persistent storage (Redis/DB)
- Harder to scale across distributed servers in different regions
- Session sync adds latency in globally distributed systems

**Best for:** Web apps, SaaS platforms, anything with a browser UI

---

### Type 2 — Stateless Authentication (JWT-Based)

```
Client                           Server
  |                                |
  |-- email + password ----------->|
  |              validates, creates JWT
  |<-- JWT token ------------------|
  |                                |
  |-- GET /profile                 |
  |   Authorization: Bearer JWT -->|
  |              verify JWT with secret key
  |              extract userId, role from payload
  |<-- 200 { profile data } -------|
```

**No database lookup needed for auth**. The token is self-contained.

**Pros:**
- No session storage needed
- Works across distributed servers (all share secret key)
- Great for APIs, mobile apps, microservices

**Cons:**
- Can't revoke tokens until expiry
- Compromise = attacker has access until token expires

**Best for:** APIs, scalable distributed systems, microservices, mobile apps

---

### Type 3 — API Key Authentication

**What it is:** A cryptographically random string that grants programmatic access to a server's API.

**How you get one:** Go to the platform's UI → Settings → Generate API Key → copy the string.

**How it's used:**
```
GET /api/v1/completions
X-API-Key: sk-abc123xyz...
Content-Type: application/json

{ "prompt": "Summarise this paragraph..." }
```

**Primary use case — Machine to Machine (M2M) Communication:**

```
Human interaction:
  User → types credentials → UI → server (needs human involvement)

Machine to Machine:
  Server A → sends API key in header → Server B
  (fully automated, no human involvement)
```

Real example: Your server wants to use OpenAI's API to summarise text. Your server programmatically sends requests to OpenAI's server using an API key. No UI, no login flow — just key + request.

**Advantages:**
- Easy to generate (one click)
- Scoped permissions (this key can only read, not write)
- Expiry dates (key becomes invalid after X days)
- Ideal for developer integrations and third-party access

**Best for:** Server-to-server communication, third-party API access, programmatic access to platforms

---

### Type 4 — OAuth 2.0 and OpenID Connect

#### Why OAuth Exists — The Delegation Problem

As the internet grew, platforms started needing access to each other's resources:
- Travel app needs to scan your Gmail for flight tickets
- Social app wants to import your Google contacts

**Initial (terrible) solution:** Users shared their passwords with third-party apps. This was catastrophic:
- Full account access — no permissions scoping
- Can't revoke access without changing password everywhere
- If one app is breached, your password is exposed everywhere

**OAuth's Solution:** Share **tokens** instead of passwords.

```
Password: gives full access to everything
Token:    gives access to only specific, scoped permissions
```

#### OAuth 2.0 Components

| Component | Who/What | Example |
|---|---|---|
| **Resource Owner** | The user who owns the data | You |
| **Client** | The app requesting access | Facebook wanting your Google contacts |
| **Resource Server** | The server hosting the data | Google's servers |
| **Authorization Server** | The server that issues tokens | Google's OAuth server |

#### OAuth 2.0 Flows (for different app types)

| Flow | Used for | Notes |
|---|---|---|
| **Authorization Code** | Server-side web apps | Most secure, most common |
| **Implicit** | Browser-based apps | Now discouraged — security risks |
| **Client Credentials** | Machine-to-machine | No user involved, just two servers |
| **Device Code** | Smart TVs, CLI tools | Limited input devices |

#### The Problem OAuth 2.0 Didn't Solve

OAuth handles **authorisation** (what can you do) but not **authentication** (who are you). After OAuth flow, you have an access token — but you don't know WHO the user is, just what they're allowed to access.

#### OpenID Connect (OIDC) — Filling the Gap

Built on top of OAuth 2.0. Adds **authentication** by introducing the **ID Token** (a JWT).

```json
// ID Token payload (JWT)
{
  "sub": "user_google_id_12345",    ← unique user identifier
  "iss": "https://accounts.google.com",  ← who issued this token
  "email": "pravan@gmail.com",
  "name": "Pravan",
  "picture": "https://...",
  "iat": 1695000000,
  "exp": 1695003600
}
```

#### The "Sign in with Google" Flow (OIDC)

```
1. You visit a note-taking app
2. You click "Sign in with Google"

3. Note app redirects you to Google's auth server
   → with requested permissions: "read your name and email"

4. You log in with your Google credentials on Google's page
   → (you never give your Google password to the note app!)

5. You grant the permissions

6. Google sends back to the note app:
   → Authorization Code
   → ID Token (JWT with your name, email, Google user ID)

7. Note app uses the Auth Code to get an Access Token
   → Can now access your data on your behalf

8. Note app extracts your identity from the ID Token
   → Creates or finds your account using your Google email
   → Logs you in ✅
```

> 💡 "Sign in with Google/GitHub/Discord" = OpenID Connect under the hood.

---

## 5. When to Use Which Auth Type

| Scenario | Auth Type |
|---|---|
| Web app with browser users | Stateful (sessions) |
| REST API, mobile apps, microservices | Stateless (JWT) |
| Server-to-server, programmatic access | API Keys |
| Social login ("Sign in with Google") | OAuth 2.0 + OpenID Connect |
| Learning / side projects | Implement your own |
| Production systems | Use an auth provider (Auth0, Clerk) |

---

## 6. Authorisation — RBAC (Role-Based Access Control)

**The problem authorisation solves:**

Not all users should have the same permissions. An admin can delete any post. A regular user can only delete their own posts. A moderator can hide posts but not delete them.

**RBAC** assigns **roles** to users, and **permissions** to roles.

### How RBAC Works

```
Roles:
  user      → read, create own notes
  moderator → read, create, hide any note
  admin     → read, create, update, delete any note, access dead zone

A user registers → server assigns role "user"
An admin promotes someone → server assigns role "admin"
```

**In every request:**
```
Request arrives with JWT or session ID
  ↓
Server extracts user ID
  ↓
Server gets user's role (from JWT payload or DB lookup)
  ↓
Route handler checks: does this role have permission for this action?
  ↓
YES → proceed ✅
NO  → 403 Forbidden ❌
```

**Example:**
```
User (role: "user") tries to access /api/admin/dead-zone
  → Server checks: role "user" does NOT have permission for dead zone
  → Response: 403 Forbidden — "You don't have access to this resource"

Admin (role: "admin") tries to access /api/admin/dead-zone
  → Server checks: role "admin" HAS permission for dead zone
  → Response: 200 OK ✅
```

---

## 7. Security: Never Do These Two Things

### Mistake 1 — Sending Specific Error Messages During Auth

**Bad (leaks information to attackers):**
```
"User not found"           ← attacker learns: this email doesn't exist
"Incorrect password"       ← attacker learns: email is valid! Only try passwords now
"Account locked"           ← attacker learns: email is valid AND they've been trying
```

**Why this is dangerous:**
- If "user not found" → attacker skips this email, tries the next
- If "incorrect password" → attacker knows the email is valid, now brute-forces the password

**Always do this instead:**
```json
{ "error": "Authentication failed" }
```

Same generic message for ALL authentication failures. The attacker gets no information about what specifically went wrong.

> This applies specifically to authentication. For other errors (validation, missing fields, etc.) — be as descriptive as you want.

---

### Mistake 2 — Timing Attacks

**How timing attacks work:**

The server's authentication flow has multiple steps:
```
Step 1: Find user by email in DB
Step 2: Check if account is locked
Step 3: Hash the provided password and compare with stored hash
```

If the **email doesn't exist**, the server fails at Step 1 and returns immediately → **fast response** (~50ms).

If the **email exists but password is wrong**, the server goes through all 3 steps including password hashing → **slower response** (~250ms).

An attacker can measure this **time difference**:
- Fast response → "this email doesn't exist, move on"
- Slow response → "this email IS valid, now brute-force the password"

They never even need to get the password right — they can enumerate valid email addresses just by measuring response times.

**How to defend against timing attacks:**

**Method 1 — Constant time comparison:**
Use cryptographically secure constant-time comparison functions for password hashes. These functions take the same amount of time regardless of whether the result is a match or not.

**Method 2 — Simulated delay:**
Add a fixed artificial delay to all authentication responses:
```javascript
// Always wait 200ms before responding, regardless of outcome
await new Promise(resolve => setTimeout(resolve, 200));
return res.status(401).json({ error: "Authentication failed" });
```

Now: valid email + wrong password takes 250ms. Invalid email also takes ~250ms (artificial delay). Attacker can't distinguish them.

---

## 8. How Passwords Are Stored (and Checked)

Passwords are **never stored in plain text**. Here's what actually happens:

**On signup:**
```
User provides password: "mySecretPassword123"
  ↓
Server hashes it:
  bcrypt.hash("mySecretPassword123", saltRounds=12)
  → "$2b$12$abc...xyz" (hash string)
  ↓
Server stores ONLY the hash in DB
(The original password is gone forever — even the server can't recover it)
```

**On login:**
```
User provides password: "mySecretPassword123"
  ↓
Server fetches stored hash from DB: "$2b$12$abc...xyz"
  ↓
Server hashes the provided password WITH THE SAME ALGORITHM
  ↓
Compares the two hashes
  → Match? ✅ Password correct
  → No match? ❌ Wrong password
```

**Key properties of hashing:**
- **One-way** — you can't reverse a hash to get the original password
- **Deterministic** — same input always produces same hash
- **Fixed length** — whether input is 3 chars or 300 chars, output hash is same length
- **Avalanche effect** — tiny change in input = completely different hash

**Salting:**
A salt is a random value added to the password before hashing. Prevents two users with the same password from having the same hash. Also defeats rainbow table attacks.

```
"password" + "randomSalt123" → hash → "$2b$12$randomSalt123..."
```

---

## Summary

### Authentication Types at a Glance
```
Stateful (Sessions)
  → Server stores session in Redis
  → Client gets session ID in HTTP-only cookie
  → Good: easy revocation, real-time control
  → Bad: needs storage, harder to scale

Stateless (JWT)
  → All user data in the token
  → Client sends token in Authorization header
  → Good: scales easily, no storage
  → Bad: hard to revoke before expiry

API Keys
  → Machine-to-machine communication
  → Scoped permissions, expiry dates
  → Good: simple, programmatic, no human flow needed

OAuth 2.0 + OIDC
  → "Sign in with Google/GitHub/Discord"
  → Solves delegation (sharing access without sharing passwords)
  → OIDC adds identity layer on top of OAuth
```

### Quick Decision Guide
| Need | Use |
|---|---|
| Web app login | Stateful sessions |
| API / mobile / microservices | JWT (stateless) |
| Server-to-server access | API keys |
| Social login / third-party | OAuth 2.0 + OIDC |
| Fine-grained permissions | RBAC |

### Security Rules to Never Forget
1. **Never store passwords in plain text** — always hash with bcrypt
2. **Never send specific auth error messages** — always generic "Authentication failed"
3. **Always use constant-time comparisons or artificial delays** — prevent timing attacks
4. **JWT payload is readable by anyone** — never put sensitive data in it
5. **Use HttpOnly + Secure + SameSite cookies** — protect session tokens
6. **In production, use an auth provider** (Auth0, Clerk) — don't reinvent the wheel

---

*Next lecture → Validation and Transformation — How to properly validate and clean data at the entry point of your backend.*
