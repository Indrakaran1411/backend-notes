# Lecture 10 — Controllers, Services, Repositories, Middlewares and Request Context
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=hyc-7w3pee8)  
> Series: Backend from First Principles

---

## What This Lecture Is About

This lecture covers the internal architecture of a backend server — what actually happens from the moment a request enters your server to the moment a response leaves it. Three closely related topics are covered together: the Handler/Controller → Service → Repository pattern, Middlewares, and Request Context.

---

## 1. The Request Lifecycle Inside the Server

We've covered what happens *outside* the server (HTTP, routing). Now let's look at what happens *inside*.

```
Client sends HTTP Request
        ↓
[Server Entry Point] — OS forwards request to the port your server listens on
        ↓
[Route Matching] — which handler should handle this?
        ↓
[Middlewares] — common operations (auth, logging, CORS, rate limiting)
        ↓
[Handler / Controller] — extract data, validate, transform, call service
        ↓
[Service Layer] — business logic, orchestration
        ↓
[Repository Layer] — database operations
        ↓
[Service returns data to Handler]
        ↓
[Handler sends HTTP Response]
        ↓
[Middlewares] — (post-response: logging, error handling)
        ↓
Client receives HTTP Response
```

---

## 2. Why Three Layers? (Handler → Service → Repository)

You *could* put everything in one function. But that creates:
- Massive, unreadable functions
- Zero reusability
- Impossible to test
- Hard to debug
- Hard to add features

The three-layer pattern is a **design decision for maintainability and scalability**, not a technical requirement.

---

## 3. The Handler / Controller Layer

**Responsibilities:** Everything HTTP-related. It's the boundary between the HTTP world and your application logic.

### What it does (in order):

#### Step 1 — Extract data from the request
Depending on the HTTP method:
```
GET    → extract query parameters
POST   → extract request body (JSON)
PATCH  → extract request body (JSON)
DELETE → extract path parameters (and optionally body)
```

#### Step 2 — Deserialise (Bind)
The request body arrives as JSON (a string). Convert it into your language's native type.

```javascript
// Node.js — usually done automatically by body-parser middleware
const { name, price } = req.body;

// Go — explicit deserialisation required
var book Book
json.NewDecoder(r.Body).Decode(&book)

// Python (FastAPI with Pydantic) — automatic via type hints
async def create_book(body: CreateBookRequest):
    ...
```

If deserialisation fails → return `400 Bad Request` immediately.

#### Step 3 — Validate and Transform
Run the validation + transformation pipeline (covered in Lecture 9).
- If validation fails → return `400 Bad Request` with specific errors
- If validation passes → proceed

#### Step 4 — Call the Service Layer
Pass all the clean, validated, transformed data to the appropriate service method.

```javascript
// Controller calls service
const books = await bookService.getAllBooks({ sort, page, limit });
```

#### Step 5 — Send Response
Based on what the service returned (success or error), send the appropriate HTTP response.

```javascript
// Success
return res.status(200).json({ data: books });

// Error from service
return res.status(404).json({ error: "NOT_FOUND", message: "Book not found" });
```

### What the Controller should NOT do:
- Business logic (that's the service's job)
- Database queries (that's the repository's job)
- Know anything about database schemas

### The Handler in Code (Conceptual)
```javascript
async function createBookHandler(req, res) {
  // Step 1: Extract
  const body = req.body;

  // Step 2: Deserialise (done by middleware in Node.js)

  // Step 3: Validate
  const result = CreateBookSchema.safeParse(body);
  if (!result.success) {
    return res.status(400).json({ error: result.error.format() });
  }

  // Step 4: Call service
  const book = await bookService.createBook(result.data);

  // Step 5: Respond
  return res.status(201).json(book);
}
```

---

## 4. The Service Layer

**Responsibilities:** All business logic. The heart of your application.

### Key Characteristics
- **No HTTP knowledge** — doesn't know about status codes, request/response objects
- **No database knowledge** — doesn't write SQL or use ORM directly
- **Pure logic** — takes data, processes it, returns data
- **Orchestrates** — can call multiple repository methods, send emails, call external APIs, trigger notifications

### The Test of a Good Service Layer
> Look at a service method. Can you tell it's being used in a web API? If yes — it has too much HTTP logic. A service method should look like it could also be called from a CLI script, a background job, or a unit test.

```javascript
// ❌ Bad service — knows about HTTP
async function createBook(req, res) {
  const book = await bookRepo.insert(req.body);
  res.status(201).json(book);
}

// ✅ Good service — pure logic, no HTTP
async function createBook(bookData) {
  const exists = await bookRepo.findByTitle(bookData.title);
  if (exists) throw new ConflictError('Book already exists');

  const book = await bookRepo.insert(bookData);
  await notificationService.notifyNewBook(book);
  return book;
}
```

### What a service can do:
- Call one or more repository methods
- Call external APIs
- Send emails or notifications
- Merge data from multiple sources
- Apply business rules (pricing, permissions, eligibility)
- Throw business errors (`ConflictError`, `NotFoundError`, etc.)

---

## 5. The Repository Layer

**Responsibilities:** Database operations only. Nothing else.

### Single Responsibility Rule
Each repository method does exactly **one thing**:
```javascript
// ✅ Good — one method, one purpose
async function getAllBooks(filters) { ... }
async function getBookById(id) { ... }
async function insertBook(data) { ... }
async function updateBook(id, data) { ... }
async function deleteBook(id) { ... }

// ❌ Bad — one method doing two different things
async function getBook(id = null) {
  if (id) return db.query('SELECT * FROM books WHERE id = $1', [id]);
  return db.query('SELECT * FROM books');
}
```

A repository method should not conditionally return different types of data based on optional parameters. Separate concerns.

### What the Repository does NOT do:
- Business logic
- Send emails
- Make HTTP calls
- Know about HTTP status codes

### Repository in Code (Conceptual)
```javascript
async function getAllBooks({ sort, page, limit }) {
  const offset = (page - 1) * limit;
  return db.query(
    `SELECT * FROM books ORDER BY ${sort} DESC LIMIT $1 OFFSET $2`,
    [limit, offset]
  );
}
```

---

## 6. How the Three Layers Interact

```
Handler receives request
  ↓
Extract + Deserialise + Validate + Transform
  ↓
Handler calls: bookService.createBook(validData)
  ↓
Service applies business logic:
  → calls bookRepo.findByTitle() to check for duplicates
  → calls bookRepo.insert(data) to create the book
  → calls emailService.sendConfirmation(userId)
  → returns the created book
  ↓
Handler receives the book from service
  ↓
Handler sends: res.status(201).json(book)
```

### The Clean Separation
| Layer | Knows about | Does NOT know about |
|---|---|---|
| Handler/Controller | HTTP (req, res, status codes) | Business rules, DB |
| Service | Business rules, orchestration | HTTP, DB schema details |
| Repository | Database schema, queries | HTTP, Business rules |

---

## 7. Middlewares

### What is a Middleware?

A middleware is a function that executes **between** the request arriving and the response being sent. It sits in the "middle" of the request lifecycle.

### What makes middleware different from a normal handler?

A handler receives `(req, res)`.  
A middleware receives `(req, res, next)`.

**`next` is the key.** It's a function provided by the framework that passes execution to the next middleware or handler in the chain.

```javascript
function myMiddleware(req, res, next) {
  // do something with req/res

  // Option 1: pass to next middleware
  next();

  // Option 2: terminate the chain (send response now)
  // res.status(401).json({ error: "Unauthorized" });
}
```

### Why Middlewares Exist

**Problem without middleware:**
```javascript
// Every single handler has to repeat this:
async function getBooksHandler(req, res) {
  const token = req.headers.authorization;
  const user = await verifyToken(token);       // auth check
  if (!user) return res.status(401).json(...); // auth check
  logger.log(req.method, req.path);            // logging
  // ... actual handler logic
}

async function createBookHandler(req, res) {
  const token = req.headers.authorization;
  const user = await verifyToken(token);       // DUPLICATED
  if (!user) return res.status(401).json(...); // DUPLICATED
  logger.log(req.method, req.path);            // DUPLICATED
  // ... actual handler logic
}
// Multiply this by 100+ endpoints...
```

**Solution with middleware:**
```javascript
// Write once:
app.use(authMiddleware);   // runs for EVERY request automatically
app.use(loggerMiddleware); // runs for EVERY request automatically

// Handlers are now clean:
async function getBooksHandler(req, res) {
  // auth and logging already handled upstream
  const books = await bookService.getAllBooks();
  res.json(books);
}
```

---

### Common Middlewares and Their Purpose

#### 1. CORS Middleware
```
Purpose: Add Access-Control-Allow-Origin headers for cross-origin requests
When to terminate: Never (passes through, just adds headers)
```
Executes first — if the origin is blocked, no point running anything else.

#### 2. Security Headers Middleware
```
Purpose: Add Content-Security-Policy, X-Frame-Options, HSTS, etc.
When to terminate: Never (always adds headers and passes through)
```

#### 3. Authentication Middleware
```
Purpose: Verify JWT / session ID, extract user identity
When to terminate: If token missing or invalid → 401 Unauthorized
When to continue: Token valid → attach user info to context → call next()
```
```javascript
async function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  const user = verifyJWT(token, process.env.JWT_SECRET);
  if (!user) return res.status(401).json({ error: 'Unauthorized' });

  req.user = user;  // ← attach to request context
  next();           // ← proceed to next middleware/handler
}
```

#### 4. Rate Limiting Middleware
```
Purpose: Limit requests per IP to prevent abuse and DDoS
When to terminate: If limit exceeded → 429 Too Many Requests
When to continue: Within limit → call next()
```

#### 5. Logging Middleware
```
Purpose: Log every request (method, path, timestamp, response time)
When to terminate: Never (always passes through)
```
```javascript
function loggerMiddleware(req, res, next) {
  const start = Date.now();
  next();  // pass to next
  const duration = Date.now() - start;
  console.log(`${req.method} ${req.path} - ${res.statusCode} (${duration}ms)`);
}
```

#### 6. Global Error Handling Middleware
```
Purpose: Catch ALL unhandled errors from anywhere in the app
When it runs: After everything else — placed LAST in middleware chain
```
```javascript
// Express: error handler has 4 parameters (err, req, res, next)
function globalErrorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;
  const isClientError = statusCode >= 400 && statusCode < 500;

  res.status(statusCode).json({
    error: err.code || 'INTERNAL_ERROR',
    message: isClientError ? err.message : 'Something went wrong',
    requestId: req.id,
  });
}
```

#### 7. Body Parser / Data Binding Middleware
```
Purpose: Parse request body (JSON, form data, multipart) into native object
When to terminate: If JSON is malformed → 400 Bad Request
When to continue: Always (parsed body attached to req.body)
```

#### 8. Compression Middleware
```
Purpose: Gzip/deflate the response body before sending
When it runs: After response is formed, before it's sent
```

---

### Middleware Order — Critical

The order you register middlewares is the order they execute. **Order matters enormously.**

```javascript
// ✅ Correct order
app.use(corsMiddleware);         // 1. Allow cross-origin
app.use(loggerMiddleware);       // 2. Log the incoming request
app.use(bodyParserMiddleware);   // 3. Parse body (must come before auth that reads body)
app.use(authMiddleware);         // 4. Verify identity (needs parsed body)
app.use(rateLimitMiddleware);    // 5. Limit requests per IP
app.use(routes);                 // 6. Route to handlers
app.use(globalErrorHandler);     // 7. Catch all errors (MUST be last)
```

**Why CORS first?**  
If the origin isn't allowed, reject immediately — don't waste time on auth, rate limiting, etc.

**Why error handler last?**  
Errors can happen anywhere — in any middleware or handler. The error handler must be positioned after everything else so it can catch errors from all of them.

**The one-way flow:**
```
Request → MW1 → MW2 → MW3 → Handler → MW4 (error) → Response
                              ↑
                         Can only go forward, never backward
```
If an error occurs in the Handler, it can't be caught by MW1, MW2, or MW3 — only by middlewares that come AFTER it in the chain.

---

## 8. Request Context

### The Problem Without Context

After the auth middleware verifies a JWT and extracts `userId` and `role`, how does the handler access that data without the middleware explicitly passing it as a function argument (which would tightly couple them)?

```javascript
// ❌ Without context — tightly coupled
authMiddleware(req, res, (userId, role) => {  // awkward coupling
  handlerFunction(req, res, userId, role);    // must pass everywhere
});
```

### The Solution — Request Context

A **request-scoped storage** — a key-value store that's:
- Unique per request (each request has its own context)
- Accessible by ALL middlewares and handlers in that request's lifecycle
- Automatically destroyed when the request ends

```javascript
// ✅ With context
function authMiddleware(req, res, next) {
  const user = verifyJWT(token);
  req.user = user;  // store in request context
  next();
}

function createBookHandler(req, res) {
  const userId = req.user.id;    // read from context
  const role = req.user.role;    // read from context
  // no need for middleware to explicitly pass these
}
```

### What Gets Stored in Request Context

| Data | Set by | Used by |
|---|---|---|
| `user.id`, `user.role` | Auth middleware | Handlers, service layer |
| `requestId` (UUID) | Early middleware | All middlewares (for log tracing) |
| `startTime` | Logging middleware | Logging middleware (to calculate duration) |
| DB transaction handle | Transaction middleware | Repository methods |
| Tenant ID | Multi-tenant middleware | All layers |
| Permissions | RBAC middleware | Route-level permission checks |

### The Request ID Use Case

A UUID is generated for each incoming request and stored in context:

```javascript
function requestIdMiddleware(req, res, next) {
  req.id = crypto.randomUUID();  // unique per request
  next();
}
```

Now every middleware and handler can include this ID in logs:
```
[req:abc-123] GET /books — started
[req:abc-123] DB query — SELECT * FROM books
[req:abc-123] GET /books — 200 OK (45ms)
```

When debugging a production issue, you can filter all logs by `req:abc-123` to trace exactly what happened during that specific request — across all services in a microservice architecture.

### Why NOT Just Use a Global Variable?

```javascript
let currentUserId = null;  // ❌ WRONG

function authMiddleware(req, res, next) {
  currentUserId = verifyJWT(token).id;  // overwrites for ALL concurrent requests!
  next();
}
```

A server handles **thousands of concurrent requests**. A global variable would be overwritten by every incoming request. Each request **must have its own isolated context**.

### Security Benefit of Request Context

Always take sensitive data like `userId` from the **request context (from the token)**, not from the **client's payload**:

```javascript
// ❌ Dangerous — client can send any userId
async function createBookHandler(req, res) {
  const userId = req.body.userId;  // client could send someone else's ID!
  await bookRepo.insert({ ...req.body, userId });
}

// ✅ Safe — userId comes from verified JWT
async function createBookHandler(req, res) {
  const userId = req.user.id;  // set by auth middleware from JWT — can't be faked
  await bookRepo.insert({ ...req.body, userId });
}
```

### Cancellation Signals (Advanced)

Request context can also carry cancellation/timeout signals. If the client disconnects before a response is sent, the context can propagate a cancellation signal to all downstream operations — stopping DB queries, external API calls, etc. — instead of letting them run to completion uselessly.

```go
// Go example — context propagation
func createBookHandler(w http.ResponseWriter, r *http.Request) {
  ctx := r.Context()  // request context with cancellation signal

  // If client disconnects, ctx.Done() fires and DB query is cancelled
  book, err := bookRepo.Insert(ctx, bookData)
}
```

---

## 9. The Complete Request Lifecycle — Full Picture

```
HTTP Request arrives
        ↓
[CORS Middleware]          → check origin, add headers, call next()
        ↓
[Logger Middleware]        → log request start, attach requestId, call next()
        ↓
[Body Parser Middleware]   → parse JSON body into req.body, call next()
        ↓
[Auth Middleware]          → verify JWT → attach req.user → call next()
                             OR → 401 Unauthorized (terminate)
        ↓
[Rate Limit Middleware]    → check request count → call next()
                             OR → 429 Too Many Requests (terminate)
        ↓
[Route Matching]           → which handler?
        ↓
[Handler/Controller]
  → Extract data (req.body, req.params, req.query)
  → Validate + Transform
  → Call service
        ↓
[Service Layer]
  → Business logic
  → Call repository / external APIs / send emails
        ↓
[Repository Layer]
  → Database query
  → Return raw data
        ↓
[Service]                  → return processed data to handler
        ↓
[Handler]                  → send HTTP response (200, 201, 204...)
        ↓
[Global Error Handler]     → catches any unhandled error → structured error response
        ↓
HTTP Response sent to client
```

---

## Summary

| Component | Responsibility | Knows about | Does NOT know about |
|---|---|---|---|
| **Handler/Controller** | Extract, validate, call service, respond | HTTP (req, res, status codes) | Business rules, DB |
| **Service** | Business logic, orchestration | Business rules | HTTP, DB schema |
| **Repository** | Database queries only | DB schema, queries | HTTP, business rules |
| **Middleware** | Cross-cutting concerns (auth, logging, CORS) | req, res, next | Specific handler logic |
| **Request Context** | Shared state for a single request | All middlewares and handlers | Nothing (it's storage) |

### Middleware Ordering
```
1. CORS                  (allow/block cross-origin)
2. Logger                (log request)
3. Body Parser           (parse JSON)
4. Authentication        (verify identity)
5. Rate Limiter          (enforce limits)
6. Routes → Handlers     (business logic)
7. Global Error Handler  (catch all errors — ALWAYS LAST)
```

---

## One-Line Takeaways

> Handler extracts and responds. Service processes. Repository queries. Each layer has one job and knows nothing about the others' concerns.

> Middleware = reusable logic that runs for every request (or every route in a group) without duplicating code in every handler.

> Request context = a per-request key-value store that lets all middlewares and handlers share data without tight coupling. Always get userId from context (verified token), never from the client payload.

---

*Next lecture → REST API Design — How to design clean, consistent, intuitive APIs following RESTful standards.*
