# 🚀 Backend Engineering — Complete In-Depth Notes

> A comprehensive reference guide covering foundational backend concepts, patterns, and best practices. Written in simple English with real examples. Perfect for interviews, revision, and day-to-day reference.

---

## Table of Contents

1. [How Backend Systems Work (Big Picture)](#1-how-backend-systems-work-big-picture)
2. [HTTP Protocol — Deep Dive](#2-http-protocol--deep-dive)
3. [Routing](#3-routing)
4. [Serialization & Deserialization](#4-serialization--deserialization)
5. [Authentication & Authorization](#5-authentication--authorization)
6. [Validation & Transformation](#6-validation--transformation)
7. [Middleware](#7-middleware)
8. [Request Context](#8-request-context)
9. [Handlers & Controllers (MVC)](#9-handlers--controllers-mvc)
10. [Databases](#10-databases)
11. [Business Logic Layer (BLL)](#11-business-logic-layer-bll)
12. [Caching](#12-caching)
13. [Transactional Emails](#13-transactional-emails)
14. [Task Queuing & Scheduling](#14-task-queuing--scheduling)
15. [Elasticsearch](#15-elasticsearch)
16. [Error Handling](#16-error-handling)
17. [Config Management](#17-config-management)
18. [Logging, Monitoring & Observability](#18-logging-monitoring--observability)
19. [Graceful Shutdown](#19-graceful-shutdown)
20. [Security](#20-security)
21. [Scaling & Performance](#21-scaling--performance)
22. [Concurrency & Parallelism](#22-concurrency--parallelism)
23. [Object Storage & Large Files](#23-object-storage--large-files)
24. [Real-Time Systems](#24-real-time-systems)
25. [Testing & Code Quality](#25-testing--code-quality)
26. [12-Factor App Principles](#26-12-factor-app-principles)
27. [OpenAPI Standards](#27-openapi-standards)
28. [Webhooks](#28-webhooks)
29. [DevOps Concepts for Backend Engineers](#29-devops-concepts-for-backend-engineers)

---

## 1. How Backend Systems Work (Big Picture)

### The Journey of a Request

When you type `google.com` in your browser, a LOT happens before you see any result. Here's the simplified flow:

```
Browser → DNS Lookup → Internet → Firewall → Load Balancer → Backend Server → Database → Response
```

**Step-by-step:**

1. **DNS Lookup** — Browser asks "what IP address is google.com?" and gets back something like `142.250.80.46`
2. **TCP Handshake** — Your browser connects to the server (3-way handshake: SYN → SYN-ACK → ACK)
3. **TLS Handshake** — For HTTPS, they exchange certificates to encrypt traffic
4. **HTTP Request** — Browser sends the actual request (GET /search?q=hello)
5. **Firewall** — Server checks: is this request safe? Block known malicious IPs
6. **Load Balancer** — Distributes traffic across multiple servers so no single server is overloaded
7. **Application Server** — Your actual backend code runs here (Node.js, Python, Go, etc.)
8. **Database** — Server fetches or stores data
9. **HTTP Response** — Server sends back data (HTML, JSON, etc.)
10. **Browser Renders** — Browser displays the result

> 💡 **Easy Memory**: Think of it like ordering food at a restaurant — you (client) → waiter (HTTP) → kitchen (backend) → storage room (database) → waiter brings food back (response)

---

## 2. HTTP Protocol — Deep Dive

### What is HTTP?

**HTTP (HyperText Transfer Protocol)** is the language that clients and servers use to talk to each other. Every API call you make uses HTTP.

### Raw HTTP Message

```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json
Accept: application/json
```

### HTTP Methods and When to Use Them

| Method | Purpose | Status Code | Example |
|--------|---------|-------------|---------|
| **GET** | Fetch data | 200 OK | `GET /users/123` |
| **POST** | Create new resource | 201 Created | `POST /users` |
| **PUT** | Replace entire resource | 200 OK | `PUT /users/123` |
| **PATCH** | Update part of a resource | 200 OK | `PATCH /users/123` |
| **DELETE** | Remove a resource | 204 No Content | `DELETE /users/123` |

> 💡 **Easy Memory**: GET = Read, POST = Create, PUT = Replace, PATCH = Update part, DELETE = Remove

### HTTP Headers — The Metadata of Requests

Headers are key-value pairs that carry extra information about the request/response.

**Request Headers:**
```http
Authorization: Bearer <token>       → Who are you?
Content-Type: application/json      → What format am I sending?
Accept: application/json            → What format do I want back?
User-Agent: Mozilla/5.0             → What browser/client am I?
```

**Response Headers:**
```http
Content-Type: application/json      → What format am I sending back?
Cache-Control: max-age=3600         → Cache this for 1 hour
Set-Cookie: session=abc123          → Store this cookie
X-Request-ID: 550e8400-e29b         → Unique ID for this request
```

**Security Headers:**
```http
Strict-Transport-Security           → Always use HTTPS
X-Content-Type-Options: nosniff    → Don't guess content type
Content-Security-Policy            → What scripts/resources can load
X-Frame-Options: DENY              → Don't allow iframe embedding
```

### HTTP Status Codes — What Do They Mean?

```
2xx → Success
  200 OK                → Everything worked
  201 Created           → New resource created
  204 No Content        → Success but no body (e.g., DELETE)

3xx → Redirect
  301 Moved Permanently → URL has changed forever
  302 Found             → Temporary redirect
  304 Not Modified      → Use your cached version

4xx → Client Error (YOU did something wrong)
  400 Bad Request       → Invalid input
  401 Unauthorized      → Not logged in
  403 Forbidden         → Logged in but no permission
  404 Not Found         → Resource doesn't exist
  409 Conflict          → Duplicate resource
  422 Unprocessable     → Validation failed
  429 Too Many Requests → Rate limited

5xx → Server Error (WE did something wrong)
  500 Internal Server Error → Something broke on our end
  502 Bad Gateway           → Upstream server failed
  503 Service Unavailable   → Server overloaded or down
```

### HTTP Versions

| Version | Key Improvement |
|---------|----------------|
| **HTTP/1.1** | Persistent connections (keep-alive), chunked transfer |
| **HTTP/2** | Multiplexing (multiple requests over one connection), header compression, server push |
| **HTTP/3** | Built on UDP (QUIC), faster connection setup, better for mobile/lossy networks |

> 💡 **Easy Memory**: 1.1 = One road, one car at a time. HTTP/2 = One road, many cars simultaneously. HTTP/3 = New faster road (UDP highway)

### CORS (Cross-Origin Resource Sharing)

**Problem**: Browser security blocks `frontend.com` from calling `api.backend.com` by default.

**Solution**: Server adds headers saying "I allow requests from frontend.com"

```http
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
```

**Preflight Request**: For non-simple requests (POST with JSON), browser first sends `OPTIONS` request to check if CORS is allowed, then sends the actual request.

```
Browser → OPTIONS /api/data (preflight)
Server  → "Yes, you're allowed" (200 OK with CORS headers)
Browser → POST /api/data (actual request)
Server  → { data: ... }
```

### HTTP Caching

Caching saves bandwidth and reduces server load by reusing previous responses.

**Cache-Control Header:**
```http
Cache-Control: max-age=3600           → Cache for 1 hour
Cache-Control: no-cache               → Always revalidate
Cache-Control: no-store               → Never cache (sensitive data)
Cache-Control: public                 → Anyone can cache this
Cache-Control: private                → Only the user's browser can cache
```

**ETags (Entity Tags):**
```http
Server Response: ETag: "33a64df551"

Next Request: If-None-Match: "33a64df551"
Server: 304 Not Modified (use your cache!)
```

> 💡 **Easy Memory**: ETag = fingerprint of the data. If fingerprint hasn't changed, use the cached version.

---

## 3. Routing

### What is Routing?

**Routing** is the process of mapping a URL + HTTP method to a specific piece of code (handler/controller).

```
GET  /users       → list all users
GET  /users/:id   → get one user
POST /users       → create a user
PUT  /users/:id   → update a user
DELETE /users/:id → delete a user
```

### Types of Routes

**Static Route** — Exact URL match:
```
GET /about
GET /contact
```

**Dynamic Route** — URL with variable parts:
```
GET /users/:id        → /users/123, /users/456
GET /posts/:slug      → /posts/my-first-post
```

**Nested Route** — Resources within resources:
```
GET /users/:userId/posts         → all posts of a user
GET /users/:userId/posts/:postId → specific post of a user
```

**Wildcard/Catchall Route** — Matches anything:
```
GET /files/*        → matches /files/a/b/c/image.png
```

**Regex Route** — Pattern-based matching:
```
GET /products/[0-9]+   → only numeric IDs
```

### Route Components

```
https://api.example.com/users/123?sort=name&order=asc
                         ├──────┘ └─────────────────┘
                     Path Param        Query Params

/users/:id   → id = "123"   (path parameter)
?sort=name   → sort = "name" (query parameter)
```

### API Versioning

When you need to change your API without breaking existing clients:

```
URI Versioning:    /api/v1/users,  /api/v2/users
Header Versioning: Accept: application/vnd.myapp.v2+json
Query String:      /api/users?version=2
```

> 💡 **Best Practice**: Use URI versioning (`/v1/`, `/v2/`) — it's the most explicit and easiest to understand.

### Route Grouping

Group related routes to share middleware and prefix:

```javascript
// All routes under /api/v1/admin share auth + admin middleware
router.group('/api/v1/admin', [authMiddleware, adminMiddleware], () => {
  router.get('/users', listUsers);
  router.delete('/users/:id', deleteUser);
});
```

---

## 4. Serialization & Deserialization

### What is Serialization?

**Serialization** = Converting your in-memory object → a format that can be sent over the network (JSON, XML, binary)

**Deserialization** = Converting received data (JSON, XML, binary) → your in-memory object

```
Your Object (Python dict / Go struct / JS object)
       ↓  Serialize
{ "name": "Alice", "age": 30 }  ← JSON string (sent over network)
       ↓  Deserialize
Your Object again
```

### Formats

**Text-Based Formats:**

| Format | Pros | Cons |
|--------|------|------|
| **JSON** | Human-readable, universal, easy to debug | Larger size, slower parsing |
| **XML** | Self-describing, supports attributes | Verbose, harder to read |

**Binary Formats:**

| Format | Pros | Cons |
|--------|------|------|
| **Protocol Buffers (protobuf)** | Much smaller size, faster parsing | Not human-readable, needs schema |
| **MessagePack** | Smaller than JSON, no schema needed | Not human-readable |

> 💡 **When to Use What**: Use JSON for public APIs (readability matters). Use Protobuf for internal microservices (performance matters).

### JSON Deep Dive

```json
{
  "name": "Alice",
  "age": 30,
  "isActive": true,
  "address": null,
  "scores": [95, 87, 92],
  "metadata": {
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

**Common JSON Pitfalls:**

```javascript
// ❌ Problem: Date as string — timezone issues
{ "createdAt": "2024-01-15 10:30:00" }

// ✅ Solution: Always use ISO 8601 with timezone
{ "createdAt": "2024-01-15T10:30:00Z" }

// ❌ Problem: Missing field vs null
{ "nickname": null }  // Explicitly null
{}                    // Field missing entirely — handle both cases!

// ❌ Problem: Large numbers lose precision in JavaScript
{ "id": 9007199254740993 }  // JavaScript can't handle this!
// ✅ Solution: Send as string
{ "id": "9007199254740993" }
```

### Custom Serialization

Sometimes you need control over how objects are serialized:

```python
# Python example
class User:
    def to_json(self):
        return {
            "id": str(self.id),           # Convert int to string
            "name": self.name,
            "createdAt": self.created_at.isoformat(),  # Format datetime
            # Don't include: password, internal_flags
        }
```

### Performance: JSON vs Protobuf

```
JSON payload:    { "id": 1, "name": "Alice", "age": 30 }  → ~38 bytes
Protobuf:        same data                                 → ~10 bytes (4x smaller!)
Parsing speed:   Protobuf is ~5-10x faster than JSON
```

---

## 5. Authentication & Authorization

### Authentication vs Authorization

> 💡 **Easy Memory**: 
> - **Authentication** = "Who are you?" (Identity check — like showing your ID)
> - **Authorization** = "What can you do?" (Permission check — like showing your ticket)

```
Authentication: Are you logged in? → Yes/No
Authorization:  Can you delete this post? → Yes/No (depends on role/ownership)
```

### Authentication Types

#### 1. Basic Authentication
```
Authorization: Basic base64(username:password)
```
- Simple but insecure over plain HTTP
- Password sent with every request
- Use only over HTTPS, mostly for internal/dev APIs

#### 2. Session-Based Authentication (Stateful)
```
1. User logs in → Server creates session in DB/Redis
2. Server sends back Session ID in cookie
3. Every request: browser sends cookie → server looks up session
4. Logout: server deletes session
```
- **Pros**: Easy to revoke (just delete session)
- **Cons**: Server must store sessions → harder to scale

#### 3. JWT (JSON Web Tokens) (Stateless)
```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEyM30.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV
```
JWT payload example:
```json
{
  "userId": 123,
  "email": "alice@example.com",
  "role": "admin",
  "iat": 1710000000,   // issued at
  "exp": 1710086400    // expires at (24 hours later)
}
```
- **Pros**: Stateless — no DB lookup needed, easy to scale
- **Cons**: Can't revoke before expiry (unless you maintain a blacklist)

> 💡 **JWT Security**: Never store sensitive data in JWT (it's base64-encoded, not encrypted — anyone can decode the payload!). Use short expiry (15 mins) + refresh tokens.

#### 4. OAuth 2.0 — "Login with Google"

```
1. User clicks "Login with Google"
2. Your app redirects to Google: "Give access to user's email"
3. User approves on Google
4. Google redirects back with authorization_code
5. Your backend exchanges code for access_token + refresh_token
6. Use access_token to call Google APIs
```

Key concepts:
- **Access Token**: Short-lived (15 mins - 1 hour), used to access resources
- **Refresh Token**: Long-lived, used to get new access tokens
- **Scopes**: What permissions are requested (`profile`, `email`, `read:files`)

#### 5. API Keys
```http
X-API-Key: sk_live_abc123xyz
```
- Simple, used for server-to-server communication
- No user identity — just authenticates the application
- Rotate regularly, never commit to code

### Password Security: Hashing & Salting

**NEVER store passwords in plaintext!**

```
Plaintext (NEVER do):  password123
MD5/SHA1 (AVOID):      482c811da5d5b4bc6d497ffa98491e38  → Rainbow table attacks!
Bcrypt with salt:      $2b$12$KIXxLRxX8YWxKJeHHjsVWuvF...  → SAFE ✓
Argon2id (BEST):       $argon2id$v=19$m=65536,t=3...      → GOLD STANDARD ✓
```

**Why Salt?** Two users with the same password get different hashes:
```
alice's password: "hello123" + salt_alice → unique_hash_1
bob's password:   "hello123" + salt_bob   → unique_hash_2
```

**Work Factor**: Argon2id is intentionally slow — makes brute force attacks take years instead of seconds.

### Multi-Factor Authentication (MFA)

```
Factor 1: Something you KNOW    (password)
Factor 2: Something you HAVE    (phone, TOTP app like Authy)
Factor 3: Something you ARE     (fingerprint, face ID)
```

### Authorization: RBAC, ABAC, PBAC

**RBAC (Role-Based Access Control):**
```
User → Roles → Permissions
alice → [admin]     → can do everything
bob   → [editor]    → can create/edit posts
carol → [viewer]    → can only read posts
```

**ABAC (Attribute-Based Access Control):**
```
"Can alice access this document?"
Check: alice.department == document.department AND alice.clearance >= document.sensitivity
```

### Security Best Practices

- **Use generic error messages**: Say "Invalid credentials" NOT "Wrong password" (prevents username enumeration)
- **Rate limit login attempts**: Lock account after 5 failed attempts
- **Timing attacks**: Use constant-time comparison for passwords — variable time reveals info to attackers
- **Audit logging**: Log all auth events — logins, logouts, failures, privilege escalations
- **Secure cookies**: `HttpOnly` (JS can't read), `Secure` (HTTPS only), `SameSite=Strict` (CSRF protection)

---

## 6. Validation & Transformation

### Why Validate?

> **"Never trust user input"** — The cardinal rule of backend development

Client-side validation = UX convenience
Server-side validation = **Security gate** — ALWAYS required

### Types of Validation

#### 1. Type Validation — Is it the right type?
```javascript
// Is 'age' a number?
if (typeof req.body.age !== 'number') throw new Error('age must be a number');

// Is 'tags' an array?
if (!Array.isArray(req.body.tags)) throw new Error('tags must be an array');
```

#### 2. Syntactic Validation — Does it match a format?
```javascript
// Is it a valid email?
/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)  // Must contain @ and .

// Is it a valid phone number?
/^\+?[1-9]\d{9,14}$/.test(phone)

// Is it a valid UUID?
/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-/.test(id)
```

#### 3. Semantic Validation — Does it make business sense?
```javascript
// Date of birth can't be in the future
if (dob > new Date()) throw new Error('DOB cannot be in future');

// Age must be between 1 and 120
if (age < 1 || age > 120) throw new Error('Invalid age');

// End date must be after start date
if (endDate <= startDate) throw new Error('End date must be after start date');
```

#### 4. Relational Validation — Do fields match each other?
```javascript
// Password and confirm password must match
if (password !== confirmPassword) throw new Error('Passwords do not match');

// If married = true, partnerName is required
if (married && !partnerName) throw new Error('Partner name required when married');
```

### Transformations

Convert incoming data to the right format before processing:

```javascript
// Path param comes as string, need a number
const userId = parseInt(req.params.id, 10);
if (isNaN(userId)) throw new Error('Invalid user ID');

// Normalize email (always lowercase)
const email = req.body.email.toLowerCase().trim();

// Convert date string to Date object
const dob = new Date(req.body.dateOfBirth);

// Add country code to phone number
const phone = req.body.phone.startsWith('+') ? req.body.phone : `+91${req.body.phone}`;
```

### Sanitization — Security Against Injection

```javascript
// ❌ DANGEROUS — XSS attack possible
const bio = req.body.bio;  // "<script>steal_cookies()</script>"

// ✅ SAFE — Strip HTML tags
const bio = sanitizeHtml(req.body.bio, { allowedTags: [] });
// Result: "" (script tag removed)
```

### Chain Validation Example

```javascript
// Validate and transform in a pipeline
function validateUsername(input) {
  let value = input;
  value = value.toLowerCase();           // 1. Normalize case
  value = value.replace(/[^a-z0-9_]/, ''); // 2. Remove special chars
  if (value.length < 3) throw Error('Too short');   // 3. Check length
  if (value.length > 20) throw Error('Too long');   // 4. Check max length
  return value;
}
```

### Returning Validation Errors

**❌ Bad**: Return one error at a time — user has to fix and retry multiple times
```json
{ "error": "email is required" }
```

**✅ Good**: Return ALL errors at once
```json
{
  "errors": [
    { "field": "email", "message": "email is required" },
    { "field": "age", "message": "age must be between 18 and 100" },
    { "field": "phone", "message": "invalid phone format" }
  ]
}
```

---

## 7. Middleware

### What is Middleware?

**Middleware** is code that runs in the middle — between the request arriving and the response being sent. Think of it as a conveyor belt of functions that each request must pass through.

```
Request → [Logger] → [Auth] → [Validator] → [Handler] → [Error Handler] → Response
```

Each middleware can:
1. Execute code
2. Modify the request or response
3. Pass control to the next middleware (`next()`)
4. Short-circuit (send response early without calling `next()`)

### Middleware Order Matters!

```javascript
// ✅ CORRECT ORDER
app.use(requestLogger);      // 1. Log request first
app.use(corsMiddleware);     // 2. Handle CORS
app.use(authMiddleware);     // 3. Check authentication
app.use(rateLimiter);        // 4. Rate limit
app.use(jsonBodyParser);     // 5. Parse request body
app.use(router);             // 6. Route to handler
app.use(errorHandler);       // 7. Catch errors LAST
```

### Common Middlewares

#### 1. Logging Middleware
```javascript
function requestLogger(req, res, next) {
  const start = Date.now();
  res.on('finish', () => {
    console.log(`${req.method} ${req.url} ${res.statusCode} ${Date.now() - start}ms`);
  });
  next(); // Pass to next middleware
}
```

#### 2. Authentication Middleware
```javascript
async function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1]; // "Bearer <token>"
  if (!token) return res.status(401).json({ error: 'No token provided' });

  try {
    const user = await verifyJWT(token);
    req.user = user; // Attach user to request context
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

#### 3. Rate Limiting Middleware
```javascript
// Allow max 100 requests per 15 minutes per IP
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,
  message: 'Too many requests, please try again later'
});
app.use('/api/', limiter);
```

#### 4. Error Handling Middleware (Always Last!)
```javascript
// Express: 4 parameters = error handler
function errorHandler(err, req, res, next) {
  console.error(err.stack);
  res.status(err.statusCode || 500).json({
    error: err.message || 'Internal Server Error',
    requestId: req.id
  });
}
app.use(errorHandler); // Must be LAST
```

#### 5. CORS Middleware
```javascript
function corsMiddleware(req, res, next) {
  res.setHeader('Access-Control-Allow-Origin', 'https://myfrontend.com');
  res.setHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE');
  res.setHeader('Access-Control-Allow-Headers', 'Authorization,Content-Type');
  if (req.method === 'OPTIONS') return res.sendStatus(204); // Preflight
  next();
}
```

### Middleware Best Practices

- **Keep middleware lightweight** — don't do heavy DB operations in middleware
- **Order correctly** — logging first, error handler last
- **Use `next(error)` to skip to error handler** instead of nested try/catch
- **Apply middleware selectively** — don't run auth on `/health` or `/public`

---

## 8. Request Context

### What is Request Context?

**Request Context** is a temporary container that holds data for the lifetime of a single request. It lets different parts of your app (middleware, controllers, services) share information without passing it as function parameters.

```
Request arrives → [Auth Middleware adds user to context] 
               → [Controller reads user from context] 
               → [Service reads user from context]
               → Response sent → Context destroyed
```

### What Goes in Request Context?

```javascript
{
  requestId: "550e8400-e29b-41d4-a716",   // Unique ID for tracing
  user: { id: 123, role: "admin" },        // Set by auth middleware
  permissions: ["read:posts", "write:posts"], // Set by auth middleware
  startTime: 1710000000000,                // For measuring duration
  correlationId: "abc-123-def"             // For distributed tracing
}
```

### Context Lifecycle

```javascript
// Middleware sets context
app.use((req, res, next) => {
  req.context = {
    requestId: generateUUID(),
    startTime: Date.now()
  };
  next();
});

// Auth middleware enriches context
app.use(async (req, res, next) => {
  const user = await getUserFromToken(req.headers.authorization);
  req.context.user = user;
  next();
});

// Controller uses context
async function getProfile(req, res) {
  const userId = req.context.user.id; // No need to parse token again!
  const profile = await db.users.findById(userId);
  res.json(profile);
}
```

### Timeouts & Cancellation

```javascript
// Set a request timeout
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000); // 5 second timeout

try {
  const data = await fetchFromExternalAPI(url, { signal: controller.signal });
  return data;
} catch (err) {
  if (err.name === 'AbortError') throw new Error('Request timed out');
  throw err;
} finally {
  clearTimeout(timeout);
}
```

---

## 9. Handlers & Controllers (MVC)

### MVC Pattern

**Model-View-Controller** separates concerns:

```
Request → Controller (decides what to do) → Service (business logic) → Model (data)
                                                                     ↓
Response ← Controller (formats response) ←──────────────────────────
```

- **Model**: Data structure + database interaction
- **View**: Response format (JSON for APIs)
- **Controller**: Handles HTTP request, delegates to service, returns response

### Responsibilities

```javascript
// ❌ BAD: Controller doing everything (fat controller)
async function createUser(req, res) {
  // Validation
  if (!req.body.email.includes('@')) return res.status(400).json({error: 'bad email'});
  // Business logic
  const hashedPassword = await bcrypt.hash(req.body.password, 12);
  const existingUser = await db.users.findOne({email: req.body.email});
  if (existingUser) return res.status(409).json({error: 'email taken'});
  // DB operation
  const user = await db.users.create({...req.body, password: hashedPassword});
  // Sending email
  await emailService.sendWelcomeEmail(user.email);
  res.status(201).json(user);
}

// ✅ GOOD: Thin controller, fat service
async function createUser(req, res) {
  const user = await userService.createUser(req.body); // All logic in service
  res.status(201).json(user);
}
```

### CRUD Operations & HTTP Mapping

```javascript
// List resources
GET /posts → 200 OK + array of posts
// { data: [...posts], pagination: { page: 1, total: 100 } }

// Get single resource
GET /posts/123 → 200 OK + post object
// 404 if not found

// Create resource
POST /posts → 201 Created + new post
// 400 if validation fails, 409 if duplicate

// Update (full replace)
PUT /posts/123 → 200 OK + updated post
// All fields required in body

// Update (partial)
PATCH /posts/123 → 200 OK + updated post
// Only fields to change in body

// Delete
DELETE /posts/123 → 204 No Content
```

### Pagination

```javascript
// Offset-based (simple but slow on large datasets)
GET /posts?page=2&limit=20
// SELECT * FROM posts LIMIT 20 OFFSET 40

// Cursor-based (fast, good for infinite scroll)
GET /posts?cursor=eyJpZCI6MTAwfQ&limit=20
// SELECT * FROM posts WHERE id > 100 LIMIT 20
```

**Response format:**
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 500,
    "nextCursor": "eyJpZCI6MTIwfQ"
  }
}
```

### Filtering & Sorting

```
GET /posts?status=published&authorId=5&sort=createdAt&order=desc
```

```javascript
function buildQuery(filters) {
  const query = {};
  if (filters.status) query.status = filters.status;
  if (filters.authorId) query.authorId = parseInt(filters.authorId);
  return query;
}
```

### Consistent Response Format

Always return the same structure — makes front-end development predictable:

```json
// Success
{
  "success": true,
  "data": { "id": 1, "name": "Alice" },
  "meta": { "requestId": "550e8400" }
}

// Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [{ "field": "email", "message": "Invalid email" }]
  },
  "meta": { "requestId": "550e8400" }
}
```

---

## 10. Databases

### Relational vs Non-Relational

| Feature | **Relational (SQL)** | **Non-Relational (NoSQL)** |
|---------|---------------------|--------------------------|
| Structure | Tables, rows, columns | Documents, key-value, graphs |
| Schema | Rigid, predefined | Flexible, dynamic |
| Relationships | JOINs, foreign keys | Embedded or referenced |
| ACID | Full support | Varies (eventual consistency) |
| Examples | PostgreSQL, MySQL | MongoDB, Redis, Cassandra |
| Best for | Complex queries, transactions | High write volume, flexible schema |

### ACID Properties — Why Transactions Matter

```
A — Atomicity:   All or nothing. Money transfer: debit + credit both happen or neither
C — Consistency: Data always valid. Can't have negative balance
I — Isolation:   Concurrent transactions don't interfere with each other
D — Durability:  Once committed, data survives crashes
```

**Example without transactions (DANGEROUS):**
```sql
-- Transfer $100 from Alice to Bob
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- ✓ Alice debited
-- CRASH HERE! Bob never gets credited. $100 disappears!
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
```

**Example with transaction (SAFE):**
```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- Both happen, or ROLLBACK if either fails
```

### CAP Theorem

In a distributed system, you can only have 2 of these 3:

- **Consistency**: Every read gets the most recent write
- **Availability**: Every request gets a response (not guaranteed to be latest)
- **Partition Tolerance**: System works even when network splits happen

> 💡 Since network partitions are unavoidable, choose between CP (consistent) or AP (available):
> - **CP**: PostgreSQL, HBase — banking, inventory systems
> - **AP**: Cassandra, DynamoDB — social media, analytics

### Indexing — Make Queries Fast

```sql
-- Without index: Full table scan (reads every row) — SLOW for large tables
SELECT * FROM users WHERE email = 'alice@example.com';  -- O(n)

-- With index: B-tree lookup — FAST
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'alice@example.com';  -- O(log n)
```

**When to index:**
- Columns in WHERE clauses
- Foreign keys (JOIN columns)
- Columns used in ORDER BY
- Unique constraints

**When NOT to index:**
- Columns rarely queried
- Small tables (full scan is fine)
- Columns with low cardinality (e.g., boolean column — only 2 values)

### N+1 Query Problem — Common Performance Killer

```javascript
// ❌ N+1: 1 query to get posts + N queries to get each author
const posts = await db.posts.findAll();          // 1 query
for (const post of posts) {
  post.author = await db.users.findById(post.authorId); // N queries!
}

// ✅ Solution: JOIN or eager loading
const posts = await db.posts.findAll({
  include: [{ model: db.users, as: 'author' }]  // 1 query with JOIN
});
```

### ORMs — Pros & Cons

**Pros:**
- Write in your language, not SQL
- Type safety, auto-mapping to objects
- Database migrations
- Protection against SQL injection

**Cons:**
- Can generate inefficient queries
- Hides complexity — need to understand what SQL is generated
- Performance overhead for complex queries

> 💡 **Rule of thumb**: Use ORM for 90% of queries. Drop to raw SQL for complex reporting queries or when you need maximum performance.

### Connection Pooling

Creating a new DB connection for every request is expensive (takes ~50-100ms).
Connection pooling reuses connections:

```javascript
// Configure pool
const pool = new Pool({
  max: 20,        // Maximum connections in pool
  idleTimeoutMillis: 30000,  // Close idle connections after 30s
  connectionTimeoutMillis: 2000  // Fail if no connection available after 2s
});
```

---

## 11. Business Logic Layer (BLL)

### The Three Layers

```
┌──────────────────────────────────────┐
│   PRESENTATION LAYER                 │
│   (Controllers, Routes, Middleware)  │
│   Deals with HTTP requests/responses │
├──────────────────────────────────────┤
│   BUSINESS LOGIC LAYER               │
│   (Services, Domain Models)          │
│   Core rules of your application     │
├──────────────────────────────────────┤
│   DATA ACCESS LAYER                  │
│   (Repositories, ORM queries)        │
│   Talks to the database              │
└──────────────────────────────────────┘
```

### Why Separate Layers?

- **Testability**: Test business logic without HTTP or database
- **Reusability**: Service can be called from REST API, GraphQL, CLI, etc.
- **Maintainability**: Change DB without touching business logic

### Service Layer Example

```javascript
// ❌ BAD: Business logic in controller
async function registerUser(req, res) {
  const { email, password } = req.body;
  if (await db.users.findOne({ email })) {  // Business rule in controller!
    return res.status(409).json({ error: 'Email taken' });
  }
  // ... more logic
}

// ✅ GOOD: Business logic in service
// Controller
async function registerUser(req, res) {
  const user = await userService.register(req.body);  // Delegate
  res.status(201).json(user);
}

// Service (has all the business logic)
class UserService {
  async register({ email, password }) {
    // Business rule: email must be unique
    const existing = await userRepository.findByEmail(email);
    if (existing) throw new ConflictError('Email already taken');

    // Business rule: hash password
    const hashedPassword = await hashPassword(password);

    // Create user
    const user = await userRepository.create({ email, password: hashedPassword });

    // Business rule: send welcome email
    await emailService.sendWelcome(user.email);

    return user;
  }
}
```

### Error Propagation

```
Service throws: ConflictError('Email taken')
           ↓
Controller catches and maps to HTTP:
  ConflictError → 409 Conflict
  NotFoundError → 404 Not Found
  ValidationError → 422 Unprocessable
  AuthError → 401 Unauthorized
```

```javascript
// Custom error types
class NotFoundError extends Error {
  constructor(message) {
    super(message);
    this.name = 'NotFoundError';
    this.statusCode = 404;
  }
}

// Error handler middleware maps them
function errorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;
  res.status(statusCode).json({ error: err.message });
}
```

---

## 12. Caching

### Why Cache?

- **Speed**: Memory access = nanoseconds, DB query = milliseconds
- **Cost**: Fewer DB queries = less DB load = lower infrastructure cost
- **Availability**: Serve cached data even when DB is slow/down

### Types of Caching

```
Level 1 (L1): In-memory cache (same process)
  → Fastest (nanoseconds), lost on restart, limited by RAM
  → Example: Node.js Map, Python dict

Level 2 (L2): Network cache (separate service)
  → Fast (microseconds), shared across servers, persists
  → Example: Redis, Memcached

Browser Cache: Client-side
  → Fastest for client (already on device), saves bandwidth

CDN Cache: Edge servers near users
  → Fast globally, for static assets and public content
```

### Caching Strategies

#### Cache-Aside (Lazy Loading) — Most Common
```
1. App checks cache: Is data there? 
2. YES (Cache HIT) → Return cached data
3. NO (Cache MISS) → Query DB, store in cache, return data
```
```javascript
async function getUser(userId) {
  // 1. Check cache
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached); // Cache hit!

  // 2. Cache miss - query DB
  const user = await db.users.findById(userId);
  
  // 3. Store in cache (expire after 1 hour)
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
  
  return user;
}
```

#### Write-Through
```
Write to cache AND DB simultaneously
→ Data always consistent
→ Extra write latency
```

#### Write-Behind (Write-Back)
```
Write to cache first, DB write happens asynchronously
→ Very fast writes
→ Risk of data loss if cache fails before DB write
```

#### Read-Through
```
App only talks to cache. Cache fetches from DB when missed.
→ App code is simpler
→ Cache handles DB interaction
```

### Cache Eviction Strategies

| Strategy | Description | Use Case |
|----------|-------------|---------|
| **LRU** (Least Recently Used) | Evict the item that hasn't been accessed the longest | General purpose, most popular |
| **LFU** (Least Frequently Used) | Evict the item accessed least often | When usage frequency matters |
| **TTL** (Time To Live) | Evict after a set time | When data freshness is important |
| **FIFO** | Evict oldest inserted item | Simple queue-like caches |

### Cache Invalidation — The Hard Problem

> "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton

```javascript
// When user updates profile, invalidate their cached data
async function updateUser(userId, data) {
  await db.users.update(userId, data);
  await redis.del(`user:${userId}`); // Invalidate cache!
}
```

**Strategies:**
- **Manual invalidation**: Delete cache key when data changes
- **TTL expiry**: Let data expire automatically (accept slight staleness)
- **Event-based**: Publish event when data changes, listener deletes cache

### Cache Hit Ratio

```
Cache Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)

Goal: > 90% hit ratio
Low hit ratio (< 60%) means your caching strategy needs work
```

---

## 13. Transactional Emails

### What are Transactional Emails?

Emails triggered by user actions (NOT marketing spam):
- Welcome email after signup
- Email verification
- Password reset
- Order confirmation
- Payment receipt
- Account alerts

### Email Anatomy

```
Subject:    "Confirm your email address"
Preheader:  "Click the link to verify" (shown in inbox preview)
Header:     Logo + navigation
Body:       Main content
CTA:        "Verify Email" button
Footer:     Unsubscribe, legal info, address
```

### Best Practices

```javascript
// ❌ BAD: Send email synchronously (blocks response)
async function register(userData) {
  const user = await createUser(userData);
  await sendWelcomeEmail(user.email); // This could fail and rollback registration!
  return user;
}

// ✅ GOOD: Queue email asynchronously
async function register(userData) {
  const user = await createUser(userData);
  await emailQueue.push({ type: 'welcome', to: user.email }); // Non-blocking
  return user;
}
```

**Providers**: SendGrid, AWS SES, Mailgun, Postmark — don't build your own SMTP server!

---

## 14. Task Queuing & Scheduling

### Why Use Queues?

Some tasks are too slow for a request-response cycle:
- Processing images/videos
- Sending bulk emails
- Generating PDF reports
- Calling slow third-party APIs
- Clearing user data (involves multiple DB queries)

### Task Queue Architecture

```
Producer → [Queue] → Consumer (Worker)
                     ↕
                  Broker (Redis/RabbitMQ)
```

```
User requests: "Export my data as CSV"
↓
Controller: "OK, I'll do it" → Push job to queue → Return 202 Accepted
↓
Background worker picks up job
↓
Worker generates CSV, stores in S3, emails download link to user
```

### Flow Example with Celery (Python)

```python
# Producer (API code)
@app.post('/export')
def export_data(user_id: int):
    task = generate_csv.delay(user_id)  # Push to queue, don't wait
    return {"taskId": task.id, "status": "processing"}

# Consumer (Worker code)
@celery.task
def generate_csv(user_id: int):
    data = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    csv_path = write_csv(data)
    upload_to_s3(csv_path)
    email_user(user_id, s3_url)
```

### Task Dependencies

```python
# Chain: task2 runs after task1 completes
chain(resize_image.s(image_id), upload_to_cdn.s())

# Group: run tasks concurrently, wait for all
group(send_email.s(user) for user in users)

# Chord: group + callback when all done
chord(group(process_order.s(o) for o in orders))(send_summary_email.s())
```

### Retry Logic

```python
@celery.task(
  max_retries=3,
  default_retry_delay=60,  # Wait 60 seconds before retry
  autoretry_for=(RequestException,)  # Auto-retry on network errors
)
def send_email(to, subject, body):
    email_provider.send(to, subject, body)
```

### Task Prioritization

```python
# High priority queue for payments
payment_queue = Queue('payments', priority=10)

# Normal priority for notifications
notification_queue = Queue('notifications', priority=1)

# Route tasks to appropriate queues
@celery.task(queue='payments')
def process_payment(order_id): ...
```

### Scheduling (Cron Jobs)

```python
# Run every day at 2 AM
@celery.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    sender.add_periodic_task(
        crontab(hour=2, minute=0),
        cleanup_logs.s(),
    )
    sender.add_periodic_task(
        crontab(0, 0, day_of_week='monday'),  # Every Monday
        send_weekly_report.s(),
    )
```

Use cases: DB backups, send reminders, sync external data, clear old logs/sessions.

---

## 15. Elasticsearch

### Why Elasticsearch?

Regular databases are bad at full-text search:
```sql
-- SQL LIKE is slow and limited
SELECT * FROM posts WHERE content LIKE '%best coffee shop%';
-- No relevance ranking, can't handle typos, slow on large datasets
```

Elasticsearch handles:
- Full-text search with relevance ranking
- Fuzzy search (handles typos)
- Autocomplete / type-ahead
- Log analytics (part of ELK stack)
- Faceted search (filters + counts)

### How It Works — Inverted Index

Regular index: `document_id → words`
Inverted index: `word → [document_ids]`

```
Documents:
  Doc 1: "The quick brown fox"
  Doc 2: "The lazy brown dog"
  Doc 3: "Quick brown animals"

Inverted Index:
  "quick"  → [Doc 1, Doc 3]
  "brown"  → [Doc 1, Doc 2, Doc 3]
  "fox"    → [Doc 1]
  "lazy"   → [Doc 2]
  "dog"    → [Doc 2]
  "animals"→ [Doc 3]

Search "brown fox" → Intersection of brown + fox → Doc 1
```

### Relevance Scoring (TF-IDF)

- **TF (Term Frequency)**: How often the term appears in the document
- **IDF (Inverse Document Frequency)**: How rare the term is across all documents

```
Score = TF × IDF

"the" → TF high, IDF low (appears everywhere) → low score
"elasticsearch" → TF medium, IDF high (rare word) → high score
```

### Basic Operations

```bash
# Create an index
PUT /products
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },    // Full-text search
      "price": { "type": "float" },
      "category": { "type": "keyword" }  // Exact match only
    }
  }
}

# Index a document
POST /products/_doc/1
{ "name": "iPhone 15", "price": 999.99, "category": "electronics" }

# Search
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iphone",
      "fields": ["name^2", "description"]  // ^2 boosts name field
    }
  }
}
```

### Fuzzy Search (Handles Typos)

```json
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "iphon",     // Typo: missing 'e'
        "fuzziness": "AUTO"   // Allow 1-2 character differences
      }
    }
  }
}
// Still finds "iPhone" even with the typo!
```

### key vs text Fields

```
"type": "keyword"  → Exact match only, used for filtering, sorting, aggregations
                     "electronics" must match exactly

"type": "text"     → Full-text search, analyzed, broken into tokens
                     "Fast iPhone charger" → ["fast", "iphone", "charger"]
```

---

## 16. Error Handling

### Types of Errors

| Type | Example | When |
|------|---------|------|
| **Syntax Error** | Missing bracket, typo | Caught at compile/parse time |
| **Runtime Error** | Null pointer, divide by zero | During execution |
| **Logic Error** | Wrong business rule | Code runs but produces wrong result |
| **Network Error** | DB connection failed, timeout | External dependency failed |

### Error Handling Strategies

**Fail Fast**: Detect and abort early
```javascript
function divide(a, b) {
  if (b === 0) throw new Error('Cannot divide by zero'); // Fail fast
  return a / b;
}
```

**Fail Safe**: Use default values
```javascript
async function getUserPreferences(userId) {
  try {
    return await db.preferences.findById(userId);
  } catch (err) {
    return DEFAULT_PREFERENCES; // Fail safe — return defaults
  }
}
```

**Graceful Degradation**: Reduced functionality is better than total failure
```javascript
async function getNewsFeed(userId) {
  const [posts, recommendations] = await Promise.allSettled([
    getPosts(userId),
    getRecommendations(userId) // Non-critical
  ]);
  
  return {
    posts: posts.status === 'fulfilled' ? posts.value : [],
    recommendations: recommendations.status === 'fulfilled' ? recommendations.value : []
    // Feed works even if recommendations service is down
  };
}
```

### Custom Error Types

```javascript
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true; // Expected error (vs unexpected bug)
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(errors) {
    super('Validation failed', 422, 'VALIDATION_ERROR');
    this.errors = errors;
  }
}
```

### Global Error Handler

```javascript
// Catches ALL unhandled errors
process.on('uncaughtException', (err) => {
  logger.error('Uncaught Exception', err);
  process.exit(1); // Must restart after uncaught exception
});

process.on('unhandledRejection', (err) => {
  logger.error('Unhandled Rejection', err);
  // Graceful shutdown
  server.close(() => process.exit(1));
});
```

### User-Facing Error Messages

```javascript
// ❌ BAD: Exposes internal details
{ "error": "SELECT * FROM users WHERE id=5 failed: column 'pasword' doesn't exist" }

// ❌ BAD: Too vague
{ "error": "Something went wrong" }

// ✅ GOOD: Informative but safe
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user found with that ID",
    "requestId": "550e8400-e29b-41d4"  // For support/debugging
  }
}
```

---

## 17. Config Management

### Why Config Management?

- **Security**: Keep secrets out of code
- **Flexibility**: Different settings for dev/staging/production
- **Feature Flags**: Enable/disable features without deploying

### Types of Configuration

```
Static Config:    DB_HOST, API_ENDPOINT, PORT — changes rarely
Dynamic Config:   FEATURE_FLAGS, RATE_LIMITS — changes at runtime
Sensitive Config: DB_PASSWORD, API_KEYS, JWT_SECRET — MUST be secret
```

### Sources of Configuration

```
1. Environment Variables     → Most common for containers/cloud
2. .env files               → Local development only (NEVER commit to git!)
3. Config files (YAML/JSON) → For complex non-sensitive config
4. Secrets Manager          → AWS Secrets Manager, HashiCorp Vault (for secrets)
```

### Example .env file

```bash
# .env (Add to .gitignore!)
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=super_secret_key_change_in_production
REDIS_URL=redis://localhost:6379
SENDGRID_API_KEY=SG.xxxxxxxxxxxx
NODE_ENV=development
PORT=3000
```

### Environment Variables vs Config Files

```
Environment Variables:
  ✓ Standard for containers (Docker, Kubernetes)
  ✓ Easy to set in CI/CD pipelines
  ✓ Secure (not written to disk)
  ✗ Hard to manage many variables

Config Files (YAML):
  ✓ Good for complex nested config
  ✓ Easier to read and version
  ✗ Can accidentally commit secrets
```

### Feature Flags

```javascript
const featureFlags = {
  NEW_CHECKOUT_FLOW: process.env.FEATURE_NEW_CHECKOUT === 'true',
  DARK_MODE: process.env.FEATURE_DARK_MODE === 'true',
};

// In code
if (featureFlags.NEW_CHECKOUT_FLOW) {
  return newCheckoutService.process(order);
} else {
  return legacyCheckoutService.process(order);
}
```

---

## 18. Logging, Monitoring & Observability

### The Three Pillars of Observability

```
Logs    → What happened? (events, errors, info messages)
Metrics → How healthy is the system? (CPU, memory, request rate, error rate)
Traces  → Why did it take so long? (distributed request tracing across services)
```

### Logging Levels

```
DEBUG   → Detailed info for debugging (only enable in development)
INFO    → Normal operations (user logged in, request processed)
WARN    → Something might be wrong but app still works
ERROR   → Something failed (DB query failed, external API error)
FATAL   → Critical failure, app might crash
```

> 💡 **Rule**: Never log at DEBUG level in production — too verbose, may expose sensitive data

### Structured vs Unstructured Logging

**Unstructured (bad):**
```
[2024-01-15 10:30:00] User alice logged in from 192.168.1.1
```
Hard to search and parse programmatically!

**Structured (good):**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "info",
  "event": "user.login",
  "userId": 123,
  "email": "alice@example.com",
  "ip": "192.168.1.1",
  "requestId": "550e8400",
  "duration": 45
}
```
Easy to search, filter, and aggregate!

### What to Log (and What NOT to)

```javascript
// ✅ DO LOG
logger.info('User registered', { userId: user.id, email: user.email });
logger.error('Payment failed', { orderId, error: err.message, code: err.code });

// ❌ NEVER LOG (sensitive data)
logger.info('User logged in', { password: req.body.password }); // NEVER!
logger.debug('Request body', { creditCard: req.body.cardNumber }); // NEVER!
```

### Monitoring

**What to Monitor:**
- **Infrastructure**: CPU, memory, disk, network
- **Application**: Request rate, response time, error rate
- **Business**: Orders per minute, active users, conversion rate

**Key Metrics (RED Method):**
```
R — Rate:    How many requests per second?
E — Errors:  How many requests are failing?
D — Duration: How long do requests take (latency)?
```

**Tools:**
- **Prometheus**: Metrics collection (pull-based, time-series DB)
- **Grafana**: Visualization dashboards
- **Alertmanager**: Send alerts to PagerDuty, Slack, email

### Distributed Tracing

For microservices: How do you know which service is slow?

```
Request comes in → Service A → Service B → Service C → Database
                    50ms         200ms        30ms        100ms
                              ← This is the bottleneck!
```

Each service adds a trace ID and span to every request, tools like Jaeger/Zipkin visualize the full trace.

### Alerting Best Practices

- **Alert on symptoms, not causes**: Alert on "error rate > 5%" not "CPU > 80%"
- **Avoid alert fatigue**: Every alert should be actionable
- **Set meaningful thresholds**: Don't alert on occasional blips
- **On-call rotation**: Don't always alert the same person

---

## 19. Graceful Shutdown

### Why Graceful Shutdown?

**Without graceful shutdown:**
```
Kill server → In-flight requests die mid-process
→ Database transactions left open
→ Partial data written → Data corruption!
→ Users see random 500 errors
```

**With graceful shutdown:**
```
Signal received → Stop accepting new requests
              → Wait for in-flight requests to complete
              → Close DB connections
              → Close file handles
              → Exit cleanly
```

### Unix Signals

```
SIGTERM: "Please shut down gracefully" (from Kubernetes, Docker)
SIGINT:  "Ctrl+C pressed" (from terminal)
SIGKILL: "Die immediately, no cleanup" (cannot be caught!)
```

### Implementation

```javascript
const server = app.listen(3000);

function gracefulShutdown(signal) {
  console.log(`Received ${signal}, shutting down gracefully...`);
  
  // Step 1: Stop accepting new connections
  server.close(async () => {
    console.log('HTTP server closed');
    
    // Step 2: Close database connections
    await db.pool.end();
    console.log('Database pool closed');
    
    // Step 3: Close Redis connections
    await redisClient.quit();
    console.log('Redis connection closed');
    
    // Step 4: Exit
    process.exit(0);
  });
  
  // Force shutdown after 30 seconds (in case something hangs)
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

## 20. Security

### OWASP Top 10 — Most Common Web Vulnerabilities

#### 1. SQL Injection

```javascript
// ❌ VULNERABLE: String concatenation
const query = `SELECT * FROM users WHERE email = '${req.body.email}'`;
// Attacker sends: ' OR '1'='1
// Query becomes: SELECT * FROM users WHERE email = '' OR '1'='1'
// Returns ALL users!

// ✅ SAFE: Parameterized query
const query = 'SELECT * FROM users WHERE email = $1';
const result = await db.query(query, [req.body.email]);
```

#### 2. XSS (Cross-Site Scripting)

```javascript
// ❌ VULNERABLE: Rendering user input as HTML
const comment = req.body.comment; // "<script>document.cookie</script>"
res.send(`<div>${comment}</div>`); // Script executes in browser!

// ✅ SAFE: Escape HTML or use template engine with auto-escaping
const safeComment = escapeHtml(req.body.comment);
// "<script>" becomes "&lt;script&gt;" — safe to render
```

#### 3. CSRF (Cross-Site Request Forgery)

```
1. You're logged into bank.com
2. Attacker sends you link to evil.com
3. evil.com has hidden form: POST bank.com/transfer?to=attacker&amount=1000
4. Your browser auto-sends your bank.com cookies with the request!
5. Bank thinks it's you and transfers money!
```

**Prevention: CSRF Tokens**
```javascript
// Server generates unique token per session
const csrfToken = generateRandomToken();
req.session.csrfToken = csrfToken;

// Include in every form/request
res.render('form', { csrfToken });

// Server validates on every POST
if (req.body._csrf !== req.session.csrfToken) {
  return res.status(403).json({ error: 'Invalid CSRF token' });
}
```

#### 4. Broken Authentication

- Using weak passwords (no policy enforcement)
- Not rate limiting login attempts
- Storing passwords in plaintext
- Exposing session tokens in URLs

#### 5. Security Misconfiguration

```javascript
// ❌ Debug mode in production
app.set('debug', true);  // Exposes stack traces to users!

// ❌ Default credentials
// admin/admin, root/root — change defaults!

// ❌ Secrets in code
const API_KEY = "sk_live_abc123"; // NEVER in code!

// ✅ Secrets in environment variables
const API_KEY = process.env.API_KEY;
```

### Defense in Depth

Multiple layers of security — one breach doesn't compromise everything:

```
Layer 1: Input validation (reject bad input)
Layer 2: Authentication (verify identity)
Layer 3: Authorization (check permissions)
Layer 4: Rate limiting (prevent abuse)
Layer 5: Encryption (protect data in transit and at rest)
Layer 6: Audit logging (detect breaches)
Layer 7: Monitoring + alerts (respond to incidents)
```

### Security Principles

- **Least Privilege**: Users/services only get minimum necessary permissions
- **Fail Secure**: When something fails, deny access (not grant it)
- **Defense in Depth**: Multiple security layers
- **Security by Design**: Build security in from the start (not added later)
- **Separation of Duties**: No single person/service can do everything

### Rate Limiting

```javascript
// Login endpoint: max 5 attempts per 15 minutes per IP
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts. Please try again later.',
  standardHeaders: true,
});

app.post('/auth/login', loginLimiter, loginController);
```

---

## 21. Scaling & Performance

### Vertical vs Horizontal Scaling

```
Vertical Scaling (Scale Up):
  Small server → Add more RAM/CPU → Bigger server
  Simple but: expensive, single point of failure, has limits

Horizontal Scaling (Scale Out):
  1 server → Add more servers → Multiple servers behind load balancer
  Complex but: cheaper, no limits, fault tolerant
```

### Identifying Bottlenecks

```
Common bottlenecks:
1. Database: slow queries, missing indexes, connection pool exhausted
2. Memory: leaks, inefficient data structures
3. CPU: heavy computation, no async operations
4. Network: large payloads, too many requests
5. External APIs: slow third-party services
```

### N+1 Query — Already Covered in Databases Section

### Database Optimization

```sql
-- 1. Add indexes on frequently queried columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- 2. Avoid SELECT * — only fetch needed columns
SELECT id, name, email FROM users; -- Not SELECT *

-- 3. Use EXPLAIN to understand query plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 5;
```

### Performance Testing

```bash
# Load testing with k6
k6 run --vus 100 --duration 30s test.js  # 100 virtual users for 30 seconds
```

Metrics to measure:
- **Latency**: p50, p95, p99 response times
- **Throughput**: Requests per second
- **Error rate**: % of failed requests
- **Saturation**: CPU/memory usage under load

### Memory Leak Prevention

```javascript
// ❌ Leak: Adding listeners without removing them
emitter.on('data', processData); // Never removed!

// ✅ Fix: Remove listeners when done
emitter.once('data', processData);  // Auto-removes after first call
// or
const handler = processData;
emitter.on('data', handler);
// Later:
emitter.off('data', handler);

// ❌ Leak: Not closing DB connections
const conn = await db.connect();
await conn.query(sql);
// Forgot conn.release()!

// ✅ Fix: Always release
const conn = await db.connect();
try {
  return await conn.query(sql);
} finally {
  conn.release(); // ALWAYS release
}
```

---

## 22. Concurrency & Parallelism

### The Difference

```
Concurrency: Dealing with multiple things at once
  → Tasks take TURNS (one CPU, multiple tasks)
  → Like a chef switching between multiple dishes
  → Best for I/O bound: DB queries, API calls, file reads

Parallelism: Doing multiple things simultaneously
  → Tasks run SIMULTANEOUSLY (multiple CPUs)
  → Like multiple chefs each cooking a dish
  → Best for CPU bound: image processing, math, encryption
```

### Async/Await for I/O Bound Tasks

```javascript
// ❌ Sequential (slow): Each awaits the previous
const user = await getUser(userId);          // 100ms
const orders = await getOrders(userId);      // 100ms
const preferences = await getPrefs(userId); // 100ms
// Total: 300ms

// ✅ Concurrent: All run simultaneously
const [user, orders, preferences] = await Promise.all([
  getUser(userId),          // ↘
  getOrders(userId),        //  → Run concurrently
  getPrefs(userId)          // ↗
]);
// Total: ~100ms (limited by slowest)
```

### Thread Pools for CPU Bound Tasks

```javascript
// Node.js: Use worker threads for CPU-heavy work
const { Worker } = require('worker_threads');

function generatePDF(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./pdf-generator.js', { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

---

## 23. Object Storage & Large Files

### When to Use Object Storage

- **User uploads**: profile pictures, documents
- **Generated files**: PDFs, reports, exports
- **Media**: videos, audio files
- **Backups**: database backups

AWS S3, Google Cloud Storage, Azure Blob Storage are common choices.

### Streaming Large Files

```javascript
// ❌ BAD: Load entire file into memory
const file = fs.readFileSync('huge-video.mp4'); // 2GB file = 2GB RAM!
res.send(file);

// ✅ GOOD: Stream file in chunks
const stream = fs.createReadStream('huge-video.mp4', { highWaterMark: 64 * 1024 }); // 64KB chunks
stream.pipe(res); // Sends data chunk by chunk
```

### Multipart Upload

For large files, split into chunks and upload in parallel:

```javascript
// AWS S3 Multipart Upload
const multipart = await s3.createMultipartUpload({ Bucket, Key }).promise();

// Upload 5MB chunks in parallel
const parts = await Promise.all(chunks.map((chunk, i) =>
  s3.uploadPart({
    Body: chunk,
    PartNumber: i + 1,
    UploadId: multipart.UploadId
  }).promise()
));

// Complete upload
await s3.completeMultipartUpload({ Parts: parts }).promise();
```

### Presigned URLs — Secure Temporary Access

```javascript
// Generate a URL that allows upload for 15 minutes (without needing AWS credentials)
const presignedUrl = s3.getSignedUrl('putObject', {
  Bucket: 'my-bucket',
  Key: `uploads/${userId}/${filename}`,
  Expires: 900  // 15 minutes
});

// Return URL to frontend — they upload directly to S3 (bypasses your server!)
return { uploadUrl: presignedUrl };
```

---

## 24. Real-Time Systems

### WebSockets — Bi-directional Communication

```
HTTP: Client → Server (one direction, new connection each time)
WebSocket: Client ↔ Server (two-way, persistent connection)
```

```javascript
// Server
const wss = new WebSocketServer({ port: 8080 });
wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    // Broadcast to all connected clients
    wss.clients.forEach(client => client.send(message));
  });
});

// Client
const ws = new WebSocket('ws://localhost:8080');
ws.send(JSON.stringify({ type: 'chat', message: 'Hello!' }));
ws.onmessage = (event) => console.log(JSON.parse(event.data));
```

**Use cases**: Chat apps, live notifications, collaborative editing, live sports scores

### Server-Sent Events (SSE) — One-way Push

```javascript
// Server pushes updates to client (client can't send messages back)
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  
  const interval = setInterval(() => {
    res.write(`data: ${JSON.stringify({ price: getStockPrice() })}\n\n`);
  }, 1000);
  
  req.on('close', () => clearInterval(interval));
});
```

**Use cases**: Live price updates, progress bars, notifications

### Pub/Sub Architecture

```
Publisher (service) → [Message Broker: Redis/Kafka] → Subscriber (multiple services)
```

```javascript
// Publisher
redis.publish('order:created', JSON.stringify({ orderId: 123 }));

// Subscriber 1: Send email
redis.subscribe('order:created', (message) => {
  const order = JSON.parse(message);
  sendConfirmationEmail(order);
});

// Subscriber 2: Update inventory
redis.subscribe('order:created', (message) => {
  const order = JSON.parse(message);
  updateInventory(order.items);
});
```

---

## 25. Testing & Code Quality

### Types of Tests

```
Unit Tests:          Test one function in isolation (fast, cheap)
Integration Tests:   Test multiple components together (medium)
E2E Tests:          Test entire flow from API to DB (slow, expensive)
Performance Tests:   Test under load (k6, JMeter)
Security Tests:     Test for vulnerabilities (OWASP ZAP)
```

### Testing Pyramid

```
       /\
      /  \   E2E Tests (few, expensive)
     /────\
    /      \  Integration Tests (some)
   /────────\
  /          \ Unit Tests (many, cheap)
 /────────────\
```

> 💡 **Rule**: Most tests should be unit tests. They're fast, cheap, and run on every commit.

### Unit Test Example

```javascript
// Testing a service function (no HTTP, no DB)
describe('UserService.calculateAge', () => {
  it('should return correct age', () => {
    const dob = new Date('1990-01-15');
    const age = userService.calculateAge(dob);
    expect(age).toBe(34); // assuming year 2024
  });
  
  it('should throw for future date', () => {
    const futureDate = new Date('2090-01-15');
    expect(() => userService.calculateAge(futureDate)).toThrow('DOB cannot be future');
  });
});
```

### Test-Driven Development (TDD)

```
Red   → Write a failing test
Green → Write minimum code to pass the test
Refactor → Clean up code while keeping tests green

Repeat!
```

### Code Quality Metrics

- **Cyclomatic Complexity**: Number of possible paths through code (keep < 10)
- **Test Coverage**: % of code covered by tests (aim for > 80%)
- **Code Duplication**: DRY — Don't Repeat Yourself

### CI/CD Integration

```yaml
# GitHub Actions
on: [push, pull_request]
jobs:
  test:
    steps:
      - run: npm install
      - run: npm run lint          # Check code style
      - run: npm run test          # Run tests
      - run: npm run test:coverage # Check coverage > 80%
```

---

## 26. 12-Factor App Principles

A methodology for building scalable, maintainable, cloud-ready apps:

| Factor | Principle | Example |
|--------|-----------|---------|
| **1. Codebase** | One codebase, many deployments | Single Git repo |
| **2. Dependencies** | Explicitly declare all dependencies | package.json, requirements.txt |
| **3. Config** | Store config in environment | .env files, not hardcoded |
| **4. Backing Services** | Treat services as attached resources | DB, Redis connected via URL |
| **5. Build/Release/Run** | Strictly separate build and run stages | Docker build → tag → deploy |
| **6. Processes** | Execute app as stateless processes | No local session storage |
| **7. Port Binding** | Export services via port binding | App listens on $PORT |
| **8. Concurrency** | Scale via process model | Multiple worker processes |
| **9. Disposability** | Fast startup, graceful shutdown | < 5 second startup |
| **10. Dev/Prod Parity** | Keep dev and prod as similar as possible | Same Docker containers everywhere |
| **11. Logs** | Treat logs as event streams | stdout/stderr → log aggregator |
| **12. Admin Processes** | Run admin tasks as one-off processes | Database migrations |

> 💡 **Key takeaway**: Stateless processes + config in environment + explicit dependencies = easy to scale and deploy anywhere

---

## 27. OpenAPI Standards

### What is OpenAPI?

A standard way to describe your REST API in a machine-readable format (YAML or JSON). Previously called Swagger.

**Benefits:**
- Auto-generate documentation (Swagger UI)
- Auto-generate client SDKs in any language
- API testing tools (Postman import)
- API validation

### OpenAPI Spec Structure

```yaml
openapi: 3.1.0
info:
  title: My API
  version: 1.0.0

paths:
  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
      required: [id, name, email]
```

### API-First Development

Write the OpenAPI spec BEFORE writing code:
1. Design the API contract
2. Share with frontend team — they can mock APIs immediately
3. Generate server stubs from spec
4. Implement the stubs
5. Use spec to validate responses automatically

---

## 28. Webhooks

### Webhooks vs Polling

```
Polling (Pull):
  Client → "Any updates?" → Server → "No"
  Client → "Any updates?" → Server → "No"
  Client → "Any updates?" → Server → "Yes! Here's data"
  → Wasteful, lots of unnecessary requests

Webhooks (Push):
  Event happens on Server → Server calls your URL → "Here's the data"
  → Efficient, real-time, no polling needed
```

### How Webhooks Work

```
1. You register a URL with the service: POST https://stripe.com/webhooks
   { "url": "https://yourapp.com/webhooks/stripe", "events": ["payment.completed"] }

2. Payment happens on Stripe

3. Stripe POSTs to your URL:
   POST https://yourapp.com/webhooks/stripe
   { "event": "payment.completed", "data": { "amount": 9999, "orderId": 123 } }

4. Your server processes the event and responds 200 OK
```

### Webhook Security — Always Verify Signatures!

```javascript
app.post('/webhooks/stripe', (req, res) => {
  const signature = req.headers['stripe-signature'];
  const secret = process.env.STRIPE_WEBHOOK_SECRET;
  
  try {
    // Verify the webhook came from Stripe (not an attacker)
    const event = stripe.webhooks.constructEvent(req.body, signature, secret);
    
    if (event.type === 'payment_intent.succeeded') {
      await fulfillOrder(event.data.object);
    }
    
    res.json({ received: true }); // Respond quickly!
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }
});
```

### Webhook Best Practices

- **Respond fast** (within 5 seconds): Process async, respond immediately
- **Idempotency**: Webhooks may be sent multiple times — handle duplicates
- **Retry logic**: If you return non-200, provider will retry (implement deduplication)
- **Always use HTTPS**
- **Log all incoming webhooks** for debugging

---

## 29. DevOps Concepts for Backend Engineers

### CI/CD Pipeline

```
Developer pushes code
    ↓
CI (Continuous Integration):
  → Run tests
  → Run linter
  → Check code coverage
  → Build Docker image
    ↓
CD (Continuous Delivery):
  → Push image to registry
  → Deploy to staging
  → Run smoke tests
    ↓
CD (Continuous Deployment):
  → Auto-deploy to production (if all tests pass)
```

### Docker

```dockerfile
# Dockerfile
FROM node:20-alpine          # Base image
WORKDIR /app
COPY package*.json ./
RUN npm ci                   # Install dependencies
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]    # Start command
```

```bash
docker build -t myapp:1.0 .         # Build image
docker run -p 3000:3000 myapp:1.0   # Run container
docker-compose up                    # Run multi-container setup
```

### Kubernetes (K8s) — Container Orchestration

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3          # Run 3 instances
  template:
    containers:
    - name: myapp
      image: myapp:1.0
      resources:
        limits:
          cpu: "500m"
          memory: "512Mi"
      readinessProbe:  # K8s checks this before routing traffic
        httpGet:
          path: /health
          port: 3000
```

Kubernetes handles:
- Auto-restart crashed containers
- Rolling updates (zero downtime)
- Scaling based on load
- Service discovery

### Deployment Strategies

| Strategy | How | Risk | Use Case |
|----------|-----|------|---------|
| **Blue-Green** | Run new version alongside old, switch traffic | Low | Critical services |
| **Rolling** | Gradually replace old instances with new | Medium | Most services |
| **Canary** | Route 5% of traffic to new version, increase if stable | Low | High-risk changes |

```
Blue-Green:
[v1 → v1 → v1]  ← All traffic
[v2 → v2 → v2]  ← New version ready
Switch traffic instantly → [v2 → v2 → v2]
```

### Infrastructure as Code (IaC)

Instead of clicking in AWS console, define infrastructure in code:

```hcl
# Terraform — creates an AWS EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name = "production-web-server"
  }
}
```

**Benefits:**
- Version controlled (git)
- Reproducible environments
- Easy disaster recovery
- Audit trail of changes

---

## 🔑 Quick Reference Cheat Sheet

### HTTP Status Codes to Memorize
```
200 OK, 201 Created, 204 No Content
400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Validation, 429 Too Many Requests
500 Server Error, 502 Bad Gateway, 503 Unavailable
```

### Security Checklist
```
□ Parameterized queries (prevent SQL injection)
□ Input sanitization (prevent XSS)
□ CSRF tokens on state-changing requests
□ Rate limiting on auth endpoints
□ Passwords hashed with Argon2id/bcrypt
□ JWT with short expiry + refresh tokens
□ HTTPS everywhere
□ Security headers (CSP, HSTS, X-Frame-Options)
□ Secrets in environment variables, not code
□ Audit logging for auth events
```

### Performance Checklist
```
□ Database indexes on query columns
□ No N+1 queries (use JOINs/eager loading)
□ Caching for frequent read operations
□ Connection pooling configured
□ Async operations for I/O tasks
□ Pagination for list endpoints
□ Response compression (gzip)
```

### API Design Checklist
```
□ Consistent response format
□ Meaningful error codes
□ API versioning (/v1/, /v2/)
□ Authentication required on protected routes
□ Rate limiting
□ Pagination for list endpoints
□ OpenAPI spec documented
□ Input validation on all endpoints
```

---

## 📚 Further Reading

- **OWASP Top 10**: https://owasp.org/Top10/ — The definitive security threats list
- **PortSwigger Web Security Academy**: https://portswigger.net/web-security — Free security labs
- **12 Factor App**: https://12factor.net — Cloud-native app methodology
- **Designing Data-Intensive Applications** by Martin Kleppmann — Best backend book
- **Clean Architecture** by Robert Martin — Software design principles

---

*Made with ❤️ — Star this repo if it helped you!*
