# Backend From First Principles — Complete Notes
> Based on Sriniously's YouTube Playlist  
> GitHub-ready | Simple language | Every topic covered

---

## Table of Contents

1. [High-Level Understanding](#1-high-level-understanding)
2. [HTTP Protocol](#2-http-protocol)
3. [Routing](#3-routing)
4. [Serialisation and Deserialisation](#4-serialisation-and-deserialisation)
5. [Authentication and Authorisation](#5-authentication-and-authorisation)
6. [Validation and Transformation](#6-validation-and-transformation)
7. [Middlewares](#7-middlewares)
8. [Request Context](#8-request-context)
9. [Handlers, Controllers, and Services](#9-handlers-controllers-and-services)
10. [CRUD Deep Dive](#10-crud-deep-dive)
11. [RESTful Architecture and Best Practices](#11-restful-architecture-and-best-practices)
12. [Databases](#12-databases)
13. [Business Logic Layer (BLL)](#13-business-logic-layer-bll)
14. [Caching](#14-caching)
15. [Transactional Emails](#15-transactional-emails)
16. [Task Queuing and Scheduling](#16-task-queuing-and-scheduling)
17. [Elasticsearch](#17-elasticsearch)
18. [Error Handling](#18-error-handling)
19. [Config Management](#19-config-management)
20. [Logging, Monitoring and Observability](#20-logging-monitoring-and-observability)
21. [Graceful Shutdown](#21-graceful-shutdown)
22. [Security](#22-security)
23. [Scaling and Performance](#23-scaling-and-performance)
24. [Concurrency and Parallelism](#24-concurrency-and-parallelism)
25. [Object Storage and Large Files](#25-object-storage-and-large-files)
26. [Real-Time Backend Systems](#26-real-time-backend-systems)
27. [Testing and Code Quality](#27-testing-and-code-quality)
28. [12 Factor App](#28-12-factor-app)
29. [OpenAPI Standards](#29-openapi-standards)
30. [Webhooks](#30-webhooks)
31. [DevOps for Backend Engineers](#31-devops-for-backend-engineers)

---

## 1. High-Level Understanding

### What is a Backend?
The backend is the part of a software system that runs on a **server** — it stores data, runs business logic, and sends responses to clients (browsers, mobile apps, etc.).

Think of a restaurant:
- **Client (browser/app)** = customer placing an order
- **Backend server** = kitchen that receives and processes it
- **Database** = pantry/storage where ingredients are kept

### Layers of a Backend System
```
Client
  ↓ HTTP Request
[Server / API Layer]  ← handles routing, auth, validation
  ↓
[Business Logic Layer]  ← decisions, rules, processing
  ↓
[Data Layer]  ← database, cache, file storage
  ↑
[Server / API Layer]  ← formats and returns response
  ↑ HTTP Response
Client
```

### Why "First Principles"?
Using frameworks (Express, FastAPI, NestJS) without understanding the fundamentals means:
- You can't debug properly when things go wrong
- You can't switch stacks easily
- You can't design scalable systems

Understanding the *why* behind every abstraction = true backend engineer.

---

## 2. HTTP Protocol

### What is HTTP?
HTTP (HyperText Transfer Protocol) is the **language** that clients and servers use to talk to each other over the internet. Every web request you make uses HTTP.

### How it Works — Request/Response Cycle
```
Browser sends Request:
  GET /users HTTP/1.1
  Host: api.example.com
  Authorization: Bearer <token>

Server sends Response:
  HTTP/1.1 200 OK
  Content-Type: application/json

  [{"id": 1, "name": "Pravan"}]
```

### Anatomy of an HTTP Request
| Part | Example | Purpose |
|---|---|---|
| Method | GET, POST, PUT, DELETE | What action to do |
| URL/Path | `/api/users/1` | Which resource |
| Headers | `Authorization: Bearer ...` | Metadata about the request |
| Body | `{"name": "Pravan"}` | Data being sent (POST/PUT) |
| Query Params | `?page=1&limit=10` | Filters/options in the URL |

### HTTP Methods (Verbs)
| Method | Meaning | Safe? | Idempotent? |
|---|---|---|---|
| GET | Read data | ✅ | ✅ |
| POST | Create data | ❌ | ❌ |
| PUT | Replace data | ❌ | ✅ |
| PATCH | Partially update | ❌ | ✅ |
| DELETE | Remove data | ❌ | ✅ |

> **Idempotent** = calling it multiple times gives the same result. e.g., DELETE /users/1 ten times still results in user 1 being deleted.

### HTTP Status Codes
| Range | Meaning | Examples |
|---|---|---|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Server error | 500 Internal Server Error, 503 Service Unavailable |

### HTTP Versions
- **HTTP/1.1** — text-based, one request per TCP connection at a time (keep-alive helps)
- **HTTP/2** — binary, multiplexing (multiple requests over one connection), header compression
- **HTTP/3** — built on QUIC (UDP), even faster for unreliable networks

### Headers You'll See Often
```
Content-Type: application/json   → tells the receiver what format the body is
Accept: application/json         → tells the server what format the client wants
Authorization: Bearer <token>    → carries auth credentials
Cache-Control: no-cache          → caching instructions
CORS headers                     → control cross-origin access
```

---

## 3. Routing

### What is Routing?
Routing is how your server decides **which code to run** based on the incoming URL and HTTP method.

```
GET  /users       → list all users
GET  /users/:id   → get one user
POST /users       → create a user
PUT  /users/:id   → update a user
DELETE /users/:id → delete a user
```

### Route Parameters vs Query Parameters
```
Route param:   /users/:id     → /users/42       (part of the path)
Query param:   /users?page=2  → ?page=2&limit=5 (optional filters)
```

### How Routing Works Internally
The server keeps a **routing table** — a list of `(method, path) → handler` mappings. When a request comes in:
1. Parse the method and path
2. Match it against the routing table (exact match, then regex/wildcards)
3. Extract route params (`:id` → `42`)
4. Call the matched handler function

### Route Grouping / Prefixes
Good APIs group related routes under a prefix:
```
/api/v1/users     → user routes
/api/v1/products  → product routes
/api/v1/orders    → order routes
```
This makes versioning easy (`/v1/`, `/v2/`).

### Wildcard and Catch-All Routes
```
/*         → matches anything (useful for SPAs, 404 handlers)
/files/*   → matches any path under /files/
```

---

## 4. Serialisation and Deserialisation

### Simple Analogy
You want to send a package. You **pack** it (serialise) into a box, send it, and the receiver **unpacks** it (deserialise).

- **Serialise** = convert your in-memory object (like a Python dict or JS object) into a format that can be sent over the wire (like JSON or binary).
- **Deserialise** = convert the received data back into an in-memory object.

### JSON (Most Common Format)
```json
// JavaScript object (in memory)
{ name: "Pravan", age: 22 }

// Serialised to JSON string (over the wire)
'{"name":"Pravan","age":22}'

// Deserialised back to object
{ name: "Pravan", age: 22 }
```

### Why This Matters
- HTTP bodies are raw **bytes/strings**. Your language doesn't know it's JSON — you have to tell it.
- You must `JSON.parse()` (deserialise) incoming data before using it.
- You must `JSON.stringify()` (serialise) your data before sending it.

### Other Formats
| Format | Use case |
|---|---|
| JSON | Default for REST APIs — human-readable |
| XML | Legacy systems, SOAP APIs |
| Protobuf | gRPC, high-performance systems — binary, fast, small |
| MessagePack | Binary JSON alternative |
| FormData | File uploads, HTML forms |

### Gotchas
- Dates in JSON are just strings — you have to parse them manually.
- Circular references crash JSON serialisation.
- Big integers can lose precision in JSON (use strings for large IDs).

---

## 5. Authentication and Authorisation

### The Difference (Easy to Confuse!)
| Term | Question it answers | Example |
|---|---|---|
| **Authentication (AuthN)** | *Who are you?* | Logging in with username/password |
| **Authorisation (AuthZ)** | *What can you do?* | Only admins can delete users |

### Common Authentication Methods

#### 1. Session-Based Auth (Traditional)
```
1. User logs in → server creates a session, stores it in DB
2. Server sends back a session ID (in a cookie)
3. Client sends the cookie with every request
4. Server looks up the session ID in DB to verify
```
- ✅ Easy to invalidate (just delete session from DB)
- ❌ Doesn't scale well (server must store all sessions)

#### 2. JWT (JSON Web Tokens) — Most Common Today
```
1. User logs in → server creates a signed JWT token
2. Server sends token to client
3. Client stores it (localStorage or cookie) and sends with every request
4. Server verifies the signature — no DB lookup needed!
```

**JWT Structure:**
```
header.payload.signature

eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiI0MiJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```
- **Header** — algorithm used (e.g., HS256)
- **Payload** — the data (userId, role, expiry) — **NOT encrypted, just base64**
- **Signature** — server verifies this to ensure token wasn't tampered with

> ⚠️ Don't put passwords or sensitive data in JWT payload — it's readable by anyone!

**Access Token vs Refresh Token:**
- **Access token** — short-lived (15 mins), used for API calls
- **Refresh token** — long-lived (7 days), used to get a new access token

#### 3. OAuth 2.0 — "Login with Google/GitHub"
You delegate authentication to a trusted third party:
```
1. User clicks "Login with Google"
2. Google verifies the user
3. Google sends your app an access token
4. Your app uses the token to get user info
```

#### 4. API Keys
- Simple string: `X-API-Key: abc123`
- Used for server-to-server communication, not user auth
- Easy to implement but harder to scope/revoke

### Authorisation Patterns
| Pattern | How it works |
|---|---|
| **RBAC** (Role-Based) | Users have roles (admin, user, moderator). Roles have permissions. |
| **ABAC** (Attribute-Based) | Access based on attributes (user.department == resource.department) |
| **ACL** (Access Control List) | Per-resource permission list (this file can be read by user IDs: 1, 2, 5) |

---

## 6. Validation and Transformation

### Why Validate?
Never trust user input. Someone can send you:
- Missing required fields
- Wrong data types (`"age": "abc"`)
- Malicious strings (`'; DROP TABLE users; --`)
- Excessively large payloads

### Where to Validate
```
Client → [Validation Layer] → Business Logic → Database
```
Validate as early as possible — at the **input boundary** before anything else runs.

### What to Validate
- **Presence** — required fields exist
- **Type** — `age` is a number, `email` is a string
- **Format** — email looks like `x@y.z`, phone has 10 digits
- **Range** — age is between 0 and 120
- **Uniqueness** — email not already in DB (validate in DB layer)

### Validation Libraries
| Language | Library |
|---|---|
| Node.js | Zod, Joi, class-validator |
| Python | Pydantic (FastAPI uses this), Marshmallow |
| Go | go-playground/validator |

### Example with Zod (Node.js)
```js
import { z } from 'zod';

const UserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().min(0).max(120),
});

const result = UserSchema.safeParse(req.body);
if (!result.success) {
  return res.status(400).json({ errors: result.error.format() });
}
```

### Transformation
After validation, you often **transform** data:
- Trim whitespace from strings
- Lowercase email addresses
- Parse date strings to Date objects
- Convert IDs from string to integer

```js
// Transform: trim and lowercase email
const email = req.body.email.trim().toLowerCase();
```

---

## 7. Middlewares

### What is Middleware?
Middleware is a function that sits **between the request coming in and the response going out**. It can read, modify, or stop the request.

Think of it like airport security:
```
[Request] → [Logger] → [Auth Check] → [Rate Limiter] → [Handler] → [Response]
```
Each "gate" is middleware. If one fails, the request stops there.

### Middleware Pattern
```js
// Express-style middleware
function myMiddleware(req, res, next) {
  // Do something with the request
  console.log(`${req.method} ${req.path}`);
  
  // Call next() to pass to the next middleware
  next();
  
  // OR send a response to stop the chain
  // res.status(401).json({ error: 'Unauthorized' });
}
```

### Common Types of Middleware
| Type | What it does |
|---|---|
| **Logger** | Logs every request (method, path, response time) |
| **Auth Middleware** | Verifies JWT token, attaches user to request |
| **Rate Limiter** | Limits requests per IP (e.g., 100 req/min) |
| **CORS** | Adds headers to allow cross-origin requests |
| **Body Parser** | Parses JSON/form body into usable object |
| **Error Handler** | Catches errors from all routes in one place |
| **Compression** | Compresses response with gzip |
| **Helmet** | Sets security headers |

### Middleware Order Matters!
```
app.use(logger);      // 1. Log the request
app.use(bodyParser);  // 2. Parse body so we can read it
app.use(authCheck);   // 3. Check auth (needs parsed body)
app.use(routes);      // 4. Run the actual route handler
app.use(errorHandler); // 5. Catch any errors
```
If you put `authCheck` before `bodyParser`, it won't be able to read the token from the body.

---

## 8. Request Context

### What is Request Context?
Request context is a way to **pass data through the entire lifecycle of a request** without explicitly passing it as function arguments.

For example, once auth middleware verifies a JWT token, it extracts the user. How does the service layer know who the user is? Through request context.

### Without Context (Messy)
```js
// You'd have to pass user everywhere manually
function getUserOrders(userId) { ... }
function calculateDiscount(userId) { ... }
// Every function needs userId passed in
```

### With Context (Clean)
```js
// Middleware sets it once
req.user = { id: 42, role: 'admin' };

// Any handler/service can access it
const user = req.user;
```

### What Gets Stored in Context?
- Authenticated user info (`userId`, `role`)
- Request ID (for tracing logs across services)
- Locale/language
- Database transaction (for transactional operations)
- Tenant ID (for multi-tenant apps)

### Context in Different Frameworks
| Framework | Context mechanism |
|---|---|
| Express (Node) | `req` object — attach data to it |
| FastAPI (Python) | `Request` object, or dependency injection |
| Go | `context.Context` passed as first argument |

---

## 9. Handlers, Controllers, and Services

### The Problem
If you dump all your code into route handlers, you get:
- Massive, unreadable functions
- Impossible to test
- Code duplication

### The Solution: Layered Architecture

```
Request → Handler/Controller → Service → Repository → Database
```

### Handler / Controller
- **Responsibility**: Parse request, call the right service, send response
- **Knows about**: HTTP (req, res)
- **Does NOT know about**: Database, business rules

```js
// Handler — thin, just HTTP stuff
async function createUserHandler(req, res) {
  const userData = req.body; // already validated
  const user = await userService.createUser(userData);
  res.status(201).json(user);
}
```

### Service
- **Responsibility**: Business logic — the actual rules of your application
- **Knows about**: Business rules
- **Does NOT know about**: HTTP, database details

```js
// Service — business logic
async function createUser(userData) {
  if (await userExists(userData.email)) {
    throw new ConflictError('Email already taken');
  }
  const hashedPassword = await hashPassword(userData.password);
  return userRepository.save({ ...userData, password: hashedPassword });
}
```

### Repository
- **Responsibility**: Database operations (CRUD)
- **Knows about**: Database
- **Does NOT know about**: Business rules, HTTP

```js
// Repository — pure DB operations
async function save(user) {
  return db.query('INSERT INTO users ...', [user.email, user.password]);
}
```

### Why This Separation?
- **Testability**: You can test services without spinning up an HTTP server
- **Reusability**: Services can be used by REST APIs, gRPC, background jobs, etc.
- **Clarity**: Each layer has one job

---

## 10. CRUD Deep Dive

### What is CRUD?
CRUD = **Create, Read, Update, Delete** — the four basic operations on any data.

| Operation | HTTP Method | SQL | Example |
|---|---|---|---|
| Create | POST | INSERT | Create a new user |
| Read | GET | SELECT | Get user by ID |
| Update | PUT / PATCH | UPDATE | Update user's name |
| Delete | DELETE | DELETE | Remove a user |

### PUT vs PATCH
- **PUT** = Replace the entire resource. If you don't send a field, it gets removed/set to null.
- **PATCH** = Partially update. Only update the fields you send.

```
PUT /users/1  → { "name": "Pravan", "email": "p@p.com", "age": 22 }  (must send everything)
PATCH /users/1 → { "name": "Pravan" }  (only update name)
```

### Soft Delete vs Hard Delete
- **Hard delete**: Row is permanently removed from DB (`DELETE FROM users WHERE id=1`)
- **Soft delete**: Row stays but is marked as deleted (`UPDATE users SET deleted_at = NOW() WHERE id=1`)

Why soft delete?
- Audit trail — you know *when* something was deleted
- Recovery — easy to undo
- Referential integrity — other records may reference it

```sql
-- Soft delete pattern
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL;

-- "Delete"
UPDATE users SET deleted_at = NOW() WHERE id = 1;

-- Queries must filter it out
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Pagination
Never return all rows — databases can have millions of records.

**Offset Pagination** (simple, common):
```
GET /users?page=2&limit=10
SELECT * FROM users LIMIT 10 OFFSET 10;
```
⚠️ Problem: slow on large datasets because DB still scans all rows up to the offset.

**Cursor Pagination** (recommended for large data):
```
GET /users?cursor=lastSeenId&limit=10
SELECT * FROM users WHERE id > lastSeenId LIMIT 10;
```
✅ Fast even on millions of rows.

---

## 11. RESTful Architecture and Best Practices

### What is REST?
REST (Representational State Transfer) is a set of **conventions** for designing web APIs. It's not a protocol — just a style guide.

### Core Principles of REST
1. **Stateless** — Each request must contain all info needed. Server doesn't remember previous requests.
2. **Resource-based** — URLs identify *things* (nouns), not *actions* (verbs).
3. **Uniform Interface** — Use HTTP methods consistently.
4. **Client-Server** — Client and server are independent. They only communicate via the API.

### URL Design — Use Nouns, Not Verbs
```
❌ Bad:
GET /getUsers
POST /createUser
DELETE /deleteUser/1

✅ Good:
GET    /users
POST   /users
DELETE /users/1
```

### Nested Resources
```
GET /users/1/orders       → all orders for user 1
GET /users/1/orders/5     → order 5 belonging to user 1
POST /users/1/orders      → create an order for user 1
```

### Versioning
Always version your API so you can release breaking changes without breaking existing clients:
```
/api/v1/users  → old version still works
/api/v2/users  → new version with changes
```

### Response Format — Be Consistent
```json
// Success
{
  "data": { "id": 1, "name": "Pravan" },
  "message": "User created successfully"
}

// Error
{
  "error": "VALIDATION_ERROR",
  "message": "Email is required",
  "details": [{ "field": "email", "issue": "required" }]
}
```

### HATEOAS (Advanced)
HATEOAS = Hypermedia as the Engine of Application State. Responses include links to related actions:
```json
{
  "id": 1,
  "name": "Pravan",
  "_links": {
    "self": "/users/1",
    "orders": "/users/1/orders",
    "delete": "/users/1"
  }
}
```

---

## 12. Databases

### Relational (SQL) Databases
**Examples**: PostgreSQL, MySQL, SQLite

Data is stored in **tables with rows and columns**. Tables relate to each other via foreign keys.

```sql
-- Users table
id | name   | email
1  | Pravan | p@p.com

-- Orders table
id | user_id | total
1  | 1       | 500
```

**Strengths**: ACID transactions, complex queries with JOINs, strong schema enforcement

**When to use**: Most applications — user data, orders, financial records

### Non-Relational (NoSQL) Databases
**Examples**: MongoDB (documents), Redis (key-value), Cassandra (wide-column), Neo4j (graph)

**When to use**:
- MongoDB: Unstructured or schema-less data, content management
- Redis: Caching, sessions, leaderboards (in-memory, ultra fast)
- Cassandra: Time-series data, logs, write-heavy workloads

### ACID Properties (Critical Concept)
| Property | Meaning |
|---|---|
| **Atomicity** | A transaction is all-or-nothing. If one step fails, all steps roll back. |
| **Consistency** | DB goes from one valid state to another. Constraints are never violated. |
| **Isolation** | Concurrent transactions don't interfere with each other. |
| **Durability** | Once committed, data survives crashes. |

### Indexes
An index is a **data structure that speeds up reads** at the cost of slower writes and more disk space.

```sql
-- Without index: full table scan (slow)
SELECT * FROM users WHERE email = 'p@p.com';

-- Create index
CREATE INDEX idx_users_email ON users(email);

-- With index: direct lookup (fast)
```

**When to index**: Columns you frequently search/filter/join on. Don't over-index — every index slows down writes.

### N+1 Problem
A common performance trap:
```
// Get 100 users → 1 query
// For each user, get their orders → 100 queries
// Total: 101 queries! ← N+1 problem

// Fix: Use a JOIN or eager loading
SELECT users.*, orders.* FROM users JOIN orders ON users.id = orders.user_id;
// 1 query instead of 101
```

### Migrations
A migration is a **versioned script that changes your database schema**. Instead of manually editing the DB, you write migrations that can be run (and rolled back).

```
migrations/
  001_create_users_table.sql
  002_add_email_to_users.sql
  003_create_orders_table.sql
```

---

## 13. Business Logic Layer (BLL)

### What is it?
The BLL is where your application's **rules and decisions** live. This is the heart of your backend.

Examples of business logic:
- "A user can't place an order if they have an unpaid invoice"
- "Discount of 10% applies if order total > ₹1000"
- "An admin can see all users, a regular user can only see themselves"

### BLL vs Other Layers
```
Controller  → only knows HTTP
BLL/Service → only knows business rules
Repository  → only knows database operations
```

### Why Separate the BLL?
- You can reuse business logic in REST, gRPC, CLI, background jobs — without duplication
- Easy to test: call the service function directly in unit tests
- Non-technical stakeholders can understand the service layer (the logic is expressed clearly)

### Domain-Driven Design (Intro)
In large systems, the BLL is organised around **domains** — areas of your business:
```
/src
  /users          → user domain (register, login, profile)
  /orders         → order domain (place order, cancel, track)
  /payments       → payment domain (charge, refund)
  /notifications  → notification domain
```

Each domain is self-contained with its own service, repository, and models.

---

## 14. Caching

### Why Cache?
Reading from a database on every request is slow and expensive. Cache stores the result of expensive operations so you can return it instantly next time.

```
Without cache: Request → DB query (10ms) → Response
With cache:    Request → Cache hit (0.1ms) → Response ← 100x faster!
```

### Cache Strategies

#### 1. Cache-Aside (Most Common)
```
1. Request comes in
2. Check cache — if HIT: return cached data
3. If MISS: fetch from DB, store in cache, return data
```

#### 2. Write-Through
- Every write goes to cache AND DB at the same time
- Cache is always fresh, but writes are slower

#### 3. Write-Back (Write-Behind)
- Write to cache immediately
- DB is updated asynchronously later
- Faster writes, but risk of data loss if cache crashes

### Cache Invalidation
The hardest problem in caching! When does cache become stale?

Strategies:
- **TTL (Time-To-Live)**: Cache expires after X seconds automatically
- **Explicit invalidation**: When data changes, delete the cache key
- **Cache versioning**: Change the key name when data changes

```js
// Set with TTL of 5 minutes
await redis.set('user:42', JSON.stringify(user), 'EX', 300);

// Invalidate when user is updated
await redis.del('user:42');
```

### What to Cache?
✅ Good to cache:
- User sessions
- Product listings that don't change often
- Computed results (e.g., leaderboards)
- API responses from external services

❌ Don't cache:
- Financial/payment data that must always be fresh
- User-specific sensitive data (or cache with user-scoped keys)

### Redis — The Standard Cache
Redis is an **in-memory key-value store**. It's the go-to for caching in production.

```
SET user:42 '{"name":"Pravan"}' EX 300   → store for 5 min
GET user:42                               → fetch
DEL user:42                               → delete
```

---

## 15. Transactional Emails

### What are Transactional Emails?
These are emails triggered by **user actions** — not marketing blasts:
- Welcome email after signup
- Password reset link
- Order confirmation
- Email verification OTP
- Payment receipt

### Why NOT Send from Your Backend Directly?
Sending email directly via SMTP from your server is unreliable:
- ISPs will mark you as spam
- No delivery tracking
- Handling bounces is complex

**Use an Email Service Provider (ESP)**:
| Provider | Notes |
|---|---|
| SendGrid | Very popular, generous free tier |
| AWS SES | Cheapest at scale |
| Mailgun | Developer-friendly |
| Resend | Modern, easy API |
| Postmark | Best deliverability for transactional |

### How It Works
```
User signs up
  → Backend generates a verification token
  → Stores token in DB with expiry
  → Calls email service API to send email
  → Email contains: https://yourapp.com/verify?token=abc123
  → User clicks link → backend verifies token → marks user as verified
```

### Email Templates
Use templating engines to generate HTML emails:
```html
<h1>Hello, {{name}}!</h1>
<p>Click below to verify your email:</p>
<a href="{{verificationUrl}}">Verify Email</a>
```

### Security Considerations
- Tokens should be **random and unguessable** (use crypto.randomBytes)
- Set an **expiry** (e.g., 24 hours for verification, 15 minutes for password reset)
- Use **HTTPS links** only
- Invalidate the token after use (one-time use)

---

## 16. Task Queuing and Scheduling

### Why Queues?
Some tasks are **too slow** to do during an HTTP request:
- Sending emails (takes 100-500ms)
- Generating PDF reports
- Processing video uploads
- Sending notifications to 10,000 users

If your request handler waits for all this, the client waits 5-10 seconds. That's bad.

**Solution**: Put the task in a queue and return immediately. A background worker picks it up.

```
Request → Handler → Add to Queue → Return 202 Accepted
                                       ↓
                              Worker picks it up later
                              and processes the task
```

### Message Queues vs Task Queues
- **Message Queue** (RabbitMQ, Kafka): Low-level, raw messages between services
- **Task Queue** (BullMQ, Celery): Higher-level, designed for background jobs with retry, delays, priority

### BullMQ (Node.js) Example
```js
// Producer — add job to queue
await queue.add('send-welcome-email', { userId: 42, email: 'p@p.com' });

// Worker — process jobs from queue
worker.process('send-welcome-email', async (job) => {
  await emailService.sendWelcome(job.data.email);
});
```

### Scheduled Jobs (Cron Jobs)
Run tasks on a schedule:
```
"0 9 * * *"  → Every day at 9 AM
"*/5 * * * *" → Every 5 minutes
"0 0 * * 1"  → Every Monday at midnight
```

Use cases: Daily reports, clearing expired tokens, retry failed payments, cleanup old logs.

### Important Queue Concepts
| Concept | What it means |
|---|---|
| **Retry** | If a job fails, try again N times |
| **Dead Letter Queue** | Failed jobs after max retries go here for inspection |
| **Priority** | High-priority jobs processed before low-priority |
| **Delay** | Schedule job to run after X seconds |
| **Concurrency** | How many jobs to process simultaneously |

---

## 17. Elasticsearch

### What is Elasticsearch?
Elasticsearch (ES) is a **full-text search engine** built on Apache Lucene. It's used when you need:
- Search with typo tolerance (fuzzy matching)
- Search across multiple fields
- Ranked results by relevance
- Filtering + full-text search combined (e.g., Swiggy: "biryani near me, under ₹200")

SQL databases are terrible at this — `LIKE '%biryani%'` is slow and can't rank results.

### Core Concepts
| Concept | SQL Equivalent |
|---|---|
| Index | Table |
| Document | Row |
| Field | Column |
| Mapping | Schema |

### How It Works
ES tokenises text into individual words (tokens) and builds an **inverted index**:
```
"chicken biryani" → ["chicken", "biryani"]

Inverted index:
"chicken" → [doc1, doc5, doc12]
"biryani" → [doc1, doc3, doc7]

Search "biryani" → instantly find [doc1, doc3, doc7]
```

### Common Use Cases
- E-commerce product search
- Log search (ELK stack: Elasticsearch + Logstash + Kibana)
- Auto-complete
- Recommendation engines

### Sync Strategy: DB + ES
Keep your main database as the **source of truth** and sync to Elasticsearch:
```
Write to PostgreSQL → trigger → sync to Elasticsearch
Search queries     → Elasticsearch
Data retrieval     → PostgreSQL
```

---

## 18. Error Handling

### Why Error Handling Matters
Without proper error handling:
- Unhandled errors crash your server
- Stack traces leak to the client (security risk!)
- Debugging is impossible

### Types of Errors
| Type | Example | Response |
|---|---|---|
| Validation Error | Missing required field | 400 Bad Request |
| Auth Error | Invalid/expired token | 401 Unauthorized |
| Permission Error | Can't access another user's data | 403 Forbidden |
| Not Found | User ID doesn't exist | 404 Not Found |
| Conflict | Email already registered | 409 Conflict |
| Rate Limit | Too many requests | 429 Too Many Requests |
| Server Error | DB connection failed | 500 Internal Server Error |

### Custom Error Classes
```js
class AppError extends Error {
  constructor(message, statusCode, errorCode) {
    super(message);
    this.statusCode = statusCode;
    this.errorCode = errorCode;
  }
}

class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(message) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}
```

### Global Error Handler Middleware
```js
// Catch ALL errors in one place
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  
  // Don't expose stack trace in production!
  res.status(statusCode).json({
    error: err.errorCode || 'INTERNAL_ERROR',
    message: process.env.NODE_ENV === 'production' 
      ? 'Something went wrong' 
      : err.message,
  });
});
```

### Error vs Exception
- **Expected errors**: User enters wrong password, resource not found → handle gracefully, return 4xx
- **Unexpected exceptions**: DB crashed, null pointer → log it, return 500, alert engineers

### Never Do This
```js
// ❌ Never swallow errors silently
try {
  doSomething();
} catch (e) {
  // nothing here — error disappears forever
}
```

---

## 19. Config Management

### The Problem
Your app needs different settings in different environments:
- **Development**: local DB, debug logging, no rate limits
- **Staging**: staging DB, moderate logging
- **Production**: production DB, minimal logging, strict security

Hard-coding these values is a disaster:
```js
// ❌ Never do this
const dbUrl = 'postgres://prod-db.internal:5432/myapp';
```

### The Solution: Environment Variables
```bash
# .env file (never commit to Git!)
DATABASE_URL=postgres://localhost:5432/myapp_dev
JWT_SECRET=supersecretkey
PORT=3000
NODE_ENV=development
```

```js
// Access in code
const dbUrl = process.env.DATABASE_URL;
```

### Best Practices
1. **Never commit `.env` to Git** — add it to `.gitignore`
2. **Commit `.env.example`** — shows what variables are needed (without real values)
3. **Validate config on startup** — crash early if required variables are missing
4. **Use secrets managers in production** — AWS Secrets Manager, Vault, GCP Secret Manager

### Config Validation on Startup
```js
// Fail fast — if this isn't set, the app shouldn't start
const requiredVars = ['DATABASE_URL', 'JWT_SECRET', 'PORT'];
for (const varName of requiredVars) {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
}
```

### Centralised Config Module
```js
// config.js — one place to read all config
export const config = {
  db: {
    url: process.env.DATABASE_URL,
    maxConnections: parseInt(process.env.DB_MAX_CONN || '10'),
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: '15m',
  },
  port: parseInt(process.env.PORT || '3000'),
};
```

---

## 20. Logging, Monitoring and Observability

### The Three Pillars of Observability

| Pillar | What it is | Example tools |
|---|---|---|
| **Logs** | Text records of what happened | Winston, Pino, structlog |
| **Metrics** | Numbers about system health | Prometheus, Datadog |
| **Traces** | Request journey across services | Jaeger, Zipkin, OpenTelemetry |

### Logging

**Structured Logging** — log as JSON, not plain text:
```json
// ❌ Plain text (hard to query)
"User 42 logged in at 14:32:01"

// ✅ Structured (queryable, filterable)
{
  "level": "info",
  "message": "User logged in",
  "userId": 42,
  "timestamp": "2024-09-23T14:32:01Z",
  "requestId": "abc-123"
}
```

**Log Levels** (most → least verbose):
```
DEBUG   → detailed dev info (disabled in production)
INFO    → normal events (user logged in, order placed)
WARN    → something unexpected but not breaking
ERROR   → something failed (DB timeout, unhandled exception)
FATAL   → system is going down
```

**Never log**:
- Passwords, tokens, credit card numbers
- PII (Personally Identifiable Information) without masking

### Metrics
Track numbers about your system's health:
- Request count per second
- Average response time (p50, p95, p99 latencies)
- Error rate (% of 5xx responses)
- CPU/memory usage
- DB connection pool usage

**Prometheus** + **Grafana** is the standard open-source stack.

### Request ID / Correlation ID
Give every request a unique ID. Include it in all logs for that request. Now you can trace a single request through all your logs:
```
requestId: "abc-123" appears in:
  - Incoming request log
  - DB query log
  - Email sent log
  - Response log
```

### Alerting
Set up alerts for:
- Error rate > 1%
- p99 latency > 2 seconds
- Server CPU > 85%
- Disk usage > 80%

---

## 21. Graceful Shutdown

### What is it?
When you deploy a new version or restart a server, you need to **stop the old process gracefully** — meaning:
1. Stop accepting new requests
2. Wait for in-flight requests to finish
3. Close DB connections, flush logs
4. Then shut down

Without graceful shutdown:
- Active requests get abruptly terminated → clients see errors
- DB connections are left dangling
- Messages in queue get lost

### How it Works
```js
// Listen for shutdown signals
process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down...');
  
  // Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Wait for in-flight requests (e.g., 30s timeout)
  await new Promise(resolve => setTimeout(resolve, 30000));
  
  // Close DB connections
  await db.disconnect();
  
  console.log('Graceful shutdown complete');
  process.exit(0);
});
```

### OS Signals
| Signal | Meaning |
|---|---|
| `SIGTERM` | Polite shutdown request (Kubernetes sends this) |
| `SIGINT` | Ctrl+C in terminal |
| `SIGKILL` | Force kill — cannot be caught |

Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30s), then sends `SIGKILL`.

---

## 22. Security

### OWASP Top 10 — Most Common Web Vulnerabilities

#### 1. SQL Injection
Attacker injects SQL into your query:
```sql
-- Input: email = "' OR '1'='1"
SELECT * FROM users WHERE email = '' OR '1'='1'; -- Returns all users!

-- Fix: Use parameterised queries
SELECT * FROM users WHERE email = $1  -- pass email as parameter
```

#### 2. XSS (Cross-Site Scripting)
Attacker injects malicious JavaScript:
```
// User submits: <script>document.cookie</script> as their name
// If you render it without escaping → script runs in victims' browsers

// Fix: Escape HTML on output, use Content-Security-Policy header
```

#### 3. Broken Authentication
- Weak passwords allowed
- No rate limiting on login → brute force possible
- JWT secrets not rotated

**Fix**: Enforce strong passwords, rate limit login, use short JWT expiry.

#### 4. Sensitive Data Exposure
- Storing passwords in plain text (use bcrypt!)
- Sending SSN/credit card numbers in responses
- Stack traces in production error messages

#### 5. CORS Misconfiguration
```js
// ❌ Too permissive
app.use(cors({ origin: '*' }));

// ✅ Only allow your frontend
app.use(cors({ origin: 'https://yourapp.com' }));
```

### Security Headers (Helmet)
```js
app.use(helmet()); // Sets many security headers automatically
```
Includes: `X-Frame-Options`, `X-Content-Type-Options`, `Content-Security-Policy`, etc.

### Rate Limiting
Prevent brute force and DDoS:
```js
// Allow max 100 requests per 15 minutes per IP
const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 });
app.use('/api', limiter);
```

### HTTPS
Always use HTTPS in production. HTTP sends data in plain text — anyone on the network can read it (passwords, tokens, etc.).

### Password Hashing
```js
// Never store plain text passwords!
const hashedPassword = await bcrypt.hash(plainPassword, 12); // 12 = cost factor

// Verify
const isValid = await bcrypt.compare(inputPassword, hashedPassword);
```

---

## 23. Scaling and Performance

### Vertical vs Horizontal Scaling
| Type | How | Example |
|---|---|---|
| **Vertical (Scale Up)** | Bigger machine (more CPU/RAM) | 2 vCPU → 8 vCPU |
| **Horizontal (Scale Out)** | More machines | 1 server → 10 servers |

Horizontal scaling is more resilient (no single point of failure) and cheaper at large scale.

### Load Balancers
Distribute incoming requests across multiple server instances:
```
Clients → [Load Balancer] → [Server 1]
                         → [Server 2]
                         → [Server 3]
```

Strategies: Round-robin, least connections, IP hash (stickiness).

**Problem with horizontal scaling**: Sessions! If user's session is stored in Server 1's memory, Server 2 won't have it. 

**Solution**: Store sessions in Redis (shared between all servers).

### Database Scaling
- **Read replicas**: One primary DB (writes), multiple replicas (reads)
- **Connection pooling**: Reuse DB connections instead of creating new ones for every request
- **Sharding**: Split data across multiple DBs (by user ID range, region, etc.)

### Performance Optimisations
1. **Database indexes** — fastest win
2. **Caching** — Redis for frequent reads
3. **Async processing** — queues for slow tasks
4. **CDN** — serve static files from edge locations near the user
5. **Compression** — gzip API responses
6. **Connection pooling** — don't create a new DB connection per request
7. **Avoid N+1 queries** — use JOINs or batch loading

### Bottleneck Identification
```
Profile → Find the slow part → Fix it → Repeat

Common bottlenecks:
- Missing DB index
- N+1 query
- No caching on expensive query
- Blocking I/O in Node.js event loop
- Too many external API calls in series (parallelise!)
```

---

## 24. Concurrency and Parallelism

### Definitions
- **Concurrency**: Dealing with multiple tasks *at the same time* (not necessarily simultaneously). Task-switching.
- **Parallelism**: Actually running multiple tasks *simultaneously* on multiple CPU cores.

Analogy: A chef cooking 3 dishes:
- **Concurrent**: One chef, switches between dishes while one is boiling
- **Parallel**: Three chefs, each cooking one dish at the same time

### Node.js Event Loop
Node.js is **single-threaded but non-blocking** using an event loop:
```
[Event Loop]
  - Execute JS code
  - Wait for I/O (network, disk) ← doesn't block! registers a callback
  - When I/O completes → run callback
```

This is why you should **never do CPU-heavy work** in Node.js without worker threads — it blocks the event loop and no other requests can be served.

### Async/Await
```js
// ❌ Sequential (waits for each)
const user = await getUser(id);       // 50ms
const orders = await getOrders(id);   // 50ms
// Total: 100ms

// ✅ Parallel (both run simultaneously)
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
// Total: 50ms
```

### Race Conditions
When two concurrent operations read and modify the same data, results can be unpredictable:
```
Thread 1: Read balance = 1000
Thread 2: Read balance = 1000
Thread 1: Write balance = 1000 - 500 = 500
Thread 2: Write balance = 1000 - 500 = 500  ← Should be 0!
```

**Fix**: Use DB transactions with proper isolation levels, or locks.

### Go Concurrency (Goroutines)
Go has lightweight goroutines for true concurrency:
```go
go func() {
  // Runs concurrently
}()

// Communicate via channels
ch <- result  // send
result := <-ch // receive
```

---

## 25. Object Storage and Large Files

### The Problem with Storing Files in Your DB or Server
- Binary files in DB are huge → DB gets slow
- Files on server disk → lost when server restarts (in containers)
- Can't scale horizontally (each server has different files)

### Object Storage
**S3 (AWS)**, **GCS (Google Cloud)**, **MinIO (self-hosted open-source)** — store files as objects with a unique key.

```
Bucket: my-app-uploads
Objects:
  users/42/profile.jpg
  orders/100/invoice.pdf
  products/5/image.png
```

### Upload Flow (Presigned URLs — Best Practice)
Don't make files go through your backend:
```
1. Client asks your backend for a presigned upload URL
2. Backend generates a presigned URL from S3 (valid for 15 min)
3. Client uploads directly to S3 using that URL
4. Client notifies backend of successful upload
5. Backend saves the S3 key/URL in DB
```

Benefits:
- Your server doesn't handle large file bytes → saves bandwidth + memory
- Direct client→S3 upload is faster

### Handling Large Files
- **Chunked upload**: Split large files into chunks, upload separately, reassemble
- **Multipart upload**: S3 natively supports this for large files (>5MB)
- **Streaming**: Process file as a stream instead of loading whole thing into memory

---

## 26. Real-Time Backend Systems

### When Do You Need Real-Time?
- Chat applications
- Live notifications
- Collaborative editing (Google Docs-style)
- Live sports scores
- Stock price tickers

HTTP is request-response — client must ask for data. Real-time means **server pushes data to client**.

### WebSockets
Full-duplex, persistent connection between client and server:
```
Client: "I want to connect" → Upgrade request
Server: "OK, upgrading"     → 101 Switching Protocols

Now both sides can send messages at any time:
Server → Client: "New message from Pravan"
Client → Server: "Seen"
```

Good for: Chat, live updates, collaborative apps.

### Server-Sent Events (SSE)
One-way push from server to client over HTTP:
```
Client subscribes: GET /events
Server keeps connection open and pushes:
  data: {"type":"notification","message":"Order shipped"}
  data: {"type":"price_update","item":"BTC","price":90000}
```

Good for: Notifications, live feeds, progress updates.

### Long Polling (Older Approach)
Client sends request → server holds it open until there's data → responds → client immediately sends another request.

Not as efficient as WebSockets/SSE but works everywhere.

### Scaling Real-Time (Pub/Sub)
Problem: If you have 10 servers, a message received by Server 1 won't reach clients connected to Server 2.

**Solution: Redis Pub/Sub or a message broker**:
```
Client A (on Server 1) sends message
  → Server 1 publishes to Redis channel
  → All servers subscribed to that channel receive it
  → Server 2 pushes to its connected clients
```

---

## 27. Testing and Code Quality

### Why Test?
- Catch bugs before they reach production
- Refactor confidently (tests tell you if you broke something)
- Living documentation (tests show how code is supposed to behave)

### Types of Tests (Testing Pyramid)
```
         /\
        /E2E\        ← Few, slow, expensive
       /------\
      /Integr. \     ← Medium
     /----------\
    / Unit Tests  \  ← Many, fast, cheap
   /--------------\
```

#### Unit Tests
Test a single function/module in isolation. Mock external dependencies.
```js
// Test that calculateDiscount works correctly
test('10% discount when order > 1000', () => {
  expect(calculateDiscount(1500)).toBe(150);
});
```

#### Integration Tests
Test multiple components working together (e.g., service + DB):
```js
// Test that createUser service actually saves to DB
test('createUser saves user to DB', async () => {
  const user = await userService.createUser({ email: 'test@test.com' });
  const found = await db.findUserByEmail('test@test.com');
  expect(found).not.toBeNull();
});
```

#### End-to-End (E2E) Tests
Test the full system via HTTP requests:
```js
// Actual HTTP call to your running server
test('POST /users creates a user', async () => {
  const res = await request(app).post('/users').send({ email: 'test@test.com' });
  expect(res.status).toBe(201);
  expect(res.body.data.email).toBe('test@test.com');
});
```

### TDD (Test-Driven Development)
Write the test first, then write the code to make it pass:
```
1. Write failing test (Red)
2. Write minimal code to pass it (Green)
3. Refactor (Refactor)
```

### Code Quality Tools
| Tool | Purpose |
|---|---|
| ESLint / Flake8 | Linting (catch bad patterns) |
| Prettier / Black | Auto-formatting |
| TypeScript | Static type checking |
| Husky | Run checks before git commit |
| SonarQube | Code quality analysis |

---

## 28. 12 Factor App

A methodology for building **scalable, maintainable, production-ready** applications. Each factor is a best practice:

| # | Factor | Simple explanation |
|---|---|---|
| 1 | **Codebase** | One codebase, tracked in git, many deploys |
| 2 | **Dependencies** | Declare all dependencies explicitly (package.json, requirements.txt) |
| 3 | **Config** | Config in environment variables, not code |
| 4 | **Backing Services** | DB, cache, queue — treat them as attached resources (swappable via URL) |
| 5 | **Build, Release, Run** | Separate build → release → run stages |
| 6 | **Processes** | App is stateless. No memory between requests. Share state via DB/cache. |
| 7 | **Port Binding** | App exposes itself on a port (self-contained, no web server dependency) |
| 8 | **Concurrency** | Scale by running more processes, not bigger ones |
| 9 | **Disposability** | Fast startup, graceful shutdown. App can be killed anytime. |
| 10 | **Dev/Prod Parity** | Keep dev, staging, and prod as similar as possible |
| 11 | **Logs** | Treat logs as event streams. Write to stdout. Infrastructure handles the rest. |
| 12 | **Admin Processes** | Run admin tasks (migrations, one-off scripts) as one-off processes |

---

## 29. OpenAPI Standards

### What is OpenAPI?
OpenAPI (formerly Swagger) is a **standard way to document your REST API** in a machine-readable format (YAML or JSON).

It describes:
- All your endpoints
- Request/response formats
- Authentication requirements
- Example values

### Why Use It?
- Auto-generate interactive documentation (Swagger UI)
- Auto-generate client SDKs in any language
- Use as a contract between frontend and backend teams
- Validate requests/responses automatically

### Example OpenAPI Spec (simplified)
```yaml
openapi: "3.0.0"
info:
  title: "My API"
  version: "1.0.0"
paths:
  /users:
    post:
      summary: "Create a user"
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                email:
                  type: string
                  format: email
      responses:
        "201":
          description: "User created"
        "400":
          description: "Validation error"
```

### Code-First vs Design-First
- **Design-First**: Write the OpenAPI spec first, then implement the API from it
- **Code-First**: Write the code, auto-generate the spec from annotations/decorators

FastAPI (Python) auto-generates OpenAPI docs from your code — both approaches in one!

---

## 30. Webhooks

### What are Webhooks?
A webhook is a way for one system to **notify another system** about events in real-time, by making an HTTP POST request.

Traditional: *You* ask the API "did anything happen?" (polling)
Webhooks: *They* tell you when something happens (push)

### Real-World Examples
- Stripe sends a webhook to your server when a payment succeeds
- GitHub sends a webhook when someone pushes code
- Shopify sends a webhook when an order is placed

### How Webhooks Work
```
1. You register your webhook URL with the provider
   e.g., "Send events to: https://myapp.com/webhooks/stripe"

2. Event happens at provider (payment succeeds)

3. Provider POSTs to your URL:
   POST https://myapp.com/webhooks/stripe
   {
     "event": "payment.succeeded",
     "amount": 5000,
     "userId": "cust_42"
   }

4. Your server processes it, returns 200 OK

5. If you return non-200 → provider retries (usually with backoff)
```

### Webhook Security — Verify Signatures!
Anyone can POST to your webhook URL. Verify it's actually from the provider:
```js
// Stripe sends a signature in the header
const signature = req.headers['stripe-signature'];
const event = stripe.webhooks.constructEvent(req.body, signature, webhookSecret);
// Throws if signature is invalid
```

### Idempotency
Webhooks can be delivered **more than once** (retries). Handle duplicates:
```js
// Store processed event IDs
if (await eventAlreadyProcessed(event.id)) {
  return res.status(200).send('Already processed');
}
await markEventProcessed(event.id);
// Now process the event
```

### Building Outgoing Webhooks (You as the Sender)
If you're building a platform that sends webhooks to customers:
- Let customers register URLs via API
- Queue webhook delivery (use a job queue)
- Retry on failure with exponential backoff
- Store delivery logs
- Sign payloads with HMAC

---

## 31. DevOps for Backend Engineers

### Why Should Backend Engineers Know DevOps?
Modern backend engineers are expected to be able to deploy, monitor, and maintain their own services. "It works on my machine" is not acceptable.

### Docker
Package your app and all its dependencies into a **container** — runs the same everywhere.

```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node", "src/index.js"]
```

```bash
docker build -t my-api .         # Build image
docker run -p 3000:3000 my-api   # Run container
```

### Docker Compose
Run your app + DB + Redis together:
```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://db:5432/myapp
    depends_on: [db, redis]

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7
```
```bash
docker compose up  # Starts everything
```

### CI/CD Pipeline
Automate build, test, and deploy on every push to main:
```
Push to GitHub
  → CI runs tests (Jest, pytest)
  → If tests pass → build Docker image
  → Push image to registry
  → CD deploys new image to server
  → Health check passes → deployment done
```

Tools: GitHub Actions, GitLab CI, CircleCI, Jenkins.

### Kubernetes (K8s) — Basics
Kubernetes orchestrates many containers across many machines:
- **Pod**: Smallest unit — one or more containers
- **Deployment**: Manages pods, handles rolling updates, self-healing
- **Service**: Stable network endpoint (load balancer) for a deployment
- **Ingress**: HTTP routing rules (like: domain → service)
- **ConfigMap/Secret**: Config and secrets injected into pods

### Environment Promotion
```
Developer's laptop → Dev → Staging → Production

Each environment:
  - Has its own DB, configs
  - Runs the same Docker image (built once!)
  - Accessed via different URLs
```

### Infrastructure as Code (IaC)
Define your infrastructure in code:
- **Terraform**: Provision cloud resources (servers, DBs, networking)
- **Helm**: Package Kubernetes deployments
- **Ansible**: Configure servers

### Monitoring in Production
Every production service needs:
- Health check endpoint: `GET /health` → `{ "status": "ok" }`
- Metrics: Prometheus scraping `/metrics`
- Dashboards: Grafana
- Alerts: PagerDuty/Opsgenie when things break
- Centralized logs: ELK stack or Loki + Grafana

---

## Quick Reference — HTTP Status Codes Cheat Sheet

| Code | Name | When to use |
|---|---|---|
| 200 | OK | Successful GET/PUT/PATCH |
| 201 | Created | Successful POST (resource created) |
| 204 | No Content | Successful DELETE (nothing to return) |
| 400 | Bad Request | Validation error, malformed request |
| 401 | Unauthorized | Not logged in / invalid token |
| 403 | Forbidden | Logged in but no permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate (email already exists) |
| 422 | Unprocessable Entity | Semantic validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Bug on the server |
| 503 | Service Unavailable | Server overloaded or in maintenance |

---

## Quick Reference — Choosing a Database

| Need | Use |
|---|---|
| General purpose, relational data | PostgreSQL |
| Simple/lightweight embedded DB | SQLite |
| Flexible JSON documents | MongoDB |
| Caching, sessions, real-time | Redis |
| Full-text search | Elasticsearch |
| Time-series data (metrics, logs) | InfluxDB / TimescaleDB |
| Graph relationships | Neo4j |
| High-write, distributed | Cassandra |

---

*Notes prepared from the Sriniously "Backend from First Principles" playlist.*  
*Language: Simple, no-fluff explanations. GitHub-ready.*
