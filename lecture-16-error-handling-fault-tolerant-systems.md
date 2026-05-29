# Lecture 16 — Error Handling and Building Fault Tolerant Systems
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=8NaM_9aKS24)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Errors are not exceptional events — they are a guaranteed, normal part of running any backend system. This lecture is about **mindset and philosophy**, not frameworks or tools. It covers the types of errors you will encounter, strategies to prevent them, how to respond when they happen, and the security implications of error handling.

> *"The question is not whether errors will happen but how you will handle them when they actually do."*

---

## 1. The Five Types of Errors

### Type 1 — Logic Errors (The Most Dangerous)

**What they are:** Code that runs fine but produces incorrect results.

**Why they're dangerous:** The app doesn't crash. There are no exceptions. Everything looks normal. But the results are wrong — and the damage accumulates silently for weeks or months.

**Real example:** An e-commerce platform accidentally applies a discount twice, resulting in negative shipping costs. No 500 errors anywhere. But the platform loses money on every single order.

**Common causes:**
- Misunderstood requirements from a client/PM discussion
- Incorrect algorithm implementation (off-by-one, wrong formula)
- Unexpected user behaviour / edge cases not considered

**Why they're hard to catch:**
- No crash → no alert
- No test covering that edge case → passes all checks
- Users may not report it — especially if it benefits them (like a discount)

> 💡 Logic errors in payment workflows, discount calculations, or security checks can corrupt data and drain revenue for months before anyone notices.

---

### Type 2 — Database Errors

Database errors can bring your entire system down because most backend apps depend completely on their database.

#### 2a — Connection Errors
Your backend can't talk to the database.

Causes:
- Network is down
- Database server is overloaded
- Connection pool exhausted (you've used all available TCP connections)

**Connection pooling:** Instead of opening a new TCP connection for every request (expensive — requires handshake), apps maintain a pool of open connections. If the pool is exhausted → new requests can't connect → 500 errors everywhere.

When this happens: frontend shows empty screens, 500s cascade everywhere.

#### 2b — Constraint Violations
Your operation breaks a database rule.

Types:
| Violation | Example |
|---|---|
| **Unique constraint** | Inserting a user with an email that already exists |
| **Foreign key violation** | Creating an order with a customer_id that doesn't exist in customers table |
| **NOT NULL violation** | Leaving a required field empty |
| **Check constraint** | Setting priority to 99 when constraint says 1-5 |

Not all of these can be caught by your validation layer (e.g., unique email — only the DB knows if it's unique). These must be caught and converted to user-friendly messages in your error handler.

#### 2c — Query Errors
Malformed SQL: typos in table names, accessing tables that don't exist, overly complex queries that time out.

#### 2d — Deadlocks
Two database operations each waiting for the other to release a lock → circular dependency → both hang. Particularly tricky. Requires careful transaction ordering.

---

### Type 3 — External Service Errors

Modern SaaS apps depend on external services: payment processors (Stripe), email (Resend, Mailgun), auth (Auth0, Clerk), storage (S3), AI (OpenAI). Every external dependency is a **point of failure you don't control**.

#### Causes of external service failures:

**Network issues:**
- Connection timeouts
- DNS failures
- Network partitions

**Rate limiting (429):**
External services limit how many requests you can make. If your platform has a logic bug that calls an email API 1000x/minute, or if you genuinely have a traffic spike — you'll start getting 429 errors. 

**Solution: Exponential backoff**
```
Get 429 → wait 1 minute → retry
Still 429 → wait 2 minutes → retry
Still 429 → wait 4 minutes → retry
...continuing until success or max retries
```

**Service outages:**
AWS, GCP, email providers — all go down sometimes. You have no control. 

**Solution: Fallbacks**
- If Redis goes down → fall back to in-memory cache
- If email service goes down → queue for retry (task queue)
- If primary auth service goes down → have a backup plan

---

### Type 4 — Input Validation Errors

**What they are:** Users send bad data that doesn't meet your system's requirements.

**Why they matter:** This is your first line of defense. Catching bad data at the entry point before it reaches your business logic or database.

Types of validation:
| Type | Example |
|---|---|
| **Format** | Is this string a valid email? Valid phone number? Valid date format? |
| **Range** | Is age between 1-120? Is string length ≤ 500 chars? Does array have 1-100 items? |
| **Required fields** | Are all mandatory fields present? |

**These are the easiest errors to handle** — you define the rules, you enforce them at the entry point, you send a 400 with a clear message. Unlike logic errors (hard to detect) or external service errors (no control), validation errors are entirely within your control.

---

### Type 5 — Configuration Errors

**What they are:** Missing or incorrect environment variables / config settings that cause unexpected behavior — usually when moving between environments (dev → staging → production).

**Common scenario:** Developer adds `OPENAI_API_KEY` to `.env` for development. Forgets to add it to production secrets. Deployment succeeds. Three weeks later, a user triggers a feature that needs it → 500 error at runtime.

**Two scenarios:**

| Scenario | What happens | Severity |
|---|---|---|
| **Validate config at startup** | App fails to start → previous deployment stays running (blue-green) | ✅ Good |
| **Don't validate at startup** | App starts fine, error only hits when the affected feature is triggered | ❌ Bad |

> 💡 **Best practice:** Validate ALL required config variables before your server starts. If any are missing → refuse to start with a clear error message. This is the "fail fast" principle.

```javascript
// At startup (before server.listen)
const requiredVars = ['DATABASE_URL', 'JWT_SECRET', 'OPENAI_API_KEY'];
for (const varName of requiredVars) {
  if (!process.env[varName]) {
    throw new Error(`Missing required environment variable: ${varName}`);
  }
}
```

---

## 2. Prevention — Finding Errors Before They Spread

> **The best error handling starts before errors happen.**

### Health Checks

Expose a `/health` or `/status` endpoint:

```
GET /health → 200 OK  ← server is running
GET /health → 500     ← something is wrong
```

But a basic health check only tells you if the server process is running. You need to verify it's actually working.

**Levels of health checks:**

| Level | What to check |
|---|---|
| **Server health** | Is the HTTP server responding? Returns 200? |
| **Database health** | Can we connect? Can we run a representative query? Is query performance normal? |
| **External service health** | Payment processor: run a test transaction. Email: send a test to internal address. Auth: generate and validate a test token. |
| **Core functionality** | Are all required config vars present? Are required caches populated? Are internal data structures consistent? |

> 💡 Don't just check if services are running. Check if they're doing their job correctly.

---

### Monitoring and Observability

Track what's happening inside your system in real time.

**Don't just track error rates. Also track:**
- **Performance metrics** — response time degradation is an early warning sign before actual failures
- **HTTP error rates** — per endpoint, not just globally
- **Database query times** — sudden slowdowns indicate problems
- **External service failure rates** — are third-party APIs starting to fail?
- **Business metrics** — a sudden drop in successful transactions indicates a problem even if error rates look normal

**Structured logging:**
```json
{
  "level": "error",
  "requestId": "abc-123",
  "userId": "usr_42",
  "error": "UNIQUE_CONSTRAINT_VIOLATION",
  "table": "users",
  "timestamp": "2024-09-23T14:30:00Z"
}
```

JSON logs → parseable by Grafana/Loki → searchable, alertable.

---

## 3. Response Strategies

### Recoverable vs Non-Recoverable Errors

| Type | Strategy |
|---|---|
| **Recoverable** (email timeout, DB connection exhausted, temporary network issue) | Retry with exponential backoff. Don't overwhelm already-stressed systems. |
| **Non-recoverable** (payment data corrupted, critical service permanently down) | Containment + graceful degradation. Use cached data. Disable non-essential features. Provide alternative functionality. |

**⚠️ Warning about retries:** Your retry logic should not add more stress to an already-stressed system. Implement circuit breakers — if a service is failing consistently, stop retrying for a cooldown period instead of hammering it.

---

## 4. Global Error Handling — The Final Safety Net

This is the most important error handling strategy in any backend app. Set it up once. It pays dividends immediately and indefinitely.

### The Architecture

```
Request
  ↓
Routing layer → finds the right handler
  ↓
Handler → deserialise + validate + call service
  ↓
Service → business logic + call repositories
  ↓
Repository → database queries

Any layer can throw an error ↑
All errors bubble up ↑
↓
Global Error Handler Middleware (LAST in middleware chain)
  ↓
Reads the error type → generates appropriate response → sends to client
```

### How Error Bubbling Works

**JavaScript/Python (exception-based):**
```javascript
// Repository throws
throw new DatabaseError('Unique constraint violation on users.email');

// Service doesn't catch it → it bubbles up
// Handler doesn't catch it → it bubbles up
// Global error handler catches it
app.use((err, req, res, next) => {
  // Identify error type and respond appropriately
});
```

**Go (return-based):**
```go
// Repository returns error
func insertBook(book Book) error {
  err := db.Exec(query, book)
  return err  // return up
}

// Service returns it up
func createBook(book Book) error {
  return bookRepo.insertBook(book)  // return up
}

// Handler returns it up to middleware
```

### Error Type → HTTP Response Mapping

| Error type | HTTP response |
|---|---|
| Validation error (name too long, invalid email) | `400 Bad Request` + specific field errors |
| Unique constraint violation (duplicate email) | `400 Bad Request` + "Email already exists" |
| No rows found (book ID 123 doesn't exist) | `404 Not Found` + "Book with ID 123 not found" |
| Foreign key violation (author ID doesn't exist) | `404 Not Found` + "Author with this ID not found" |
| Unhandled/unknown error | `500 Internal Server Error` + "Something went wrong" (generic!) |

### Two Advantages of Global Error Handling

1. **Robustness** — You can never "forget" to handle an error in one repository method, because all errors eventually reach the same global handler
2. **No redundancy** — Error handling logic lives in one place, not duplicated across 50 repository methods

---

## 5. Error Response Structure

Consistent error response format:

```json
{
  "code": 400,
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": [
    { "field": "name", "message": "Name cannot exceed 500 characters" },
    { "field": "email", "message": "Invalid email format" }
  ]
}
```

For 500 errors (always generic):
```json
{
  "code": 500,
  "error": "INTERNAL_SERVER_ERROR",
  "message": "Something went wrong"
}
```

---

## 6. Security — What NOT to Expose in Error Messages

### 1. Never Leak Internal Details

If you forward raw database errors to the client:
```json
// ❌ NEVER do this
{
  "error": "ERROR: duplicate key value violates unique constraint 'users_email_key' on table 'users' in schema 'public'"
}
```

This tells an attacker:
- Your table name (`users`)
- Your constraint name (`users_email_key`)
- Your schema name (`public`)

They can use this to craft more targeted SQL injection attacks.

**Always translate DB errors into user-friendly messages:**
```json
// ✅ Correct
{
  "code": 400,
  "error": "CONFLICT",
  "message": "An account with this email already exists"
}
```

### 2. Authentication Error Messages — The User Enumeration Attack

**The attack:** An attacker tries different emails with various passwords. Detailed error messages help them:

```
"User with this email does not exist"  ← attacker knows email doesn't exist → try next email
"Your password is incorrect"            ← attacker knows THIS email EXISTS → now brute-force the password
```

With detailed messages, the attacker can:
1. Enumerate valid email addresses (get a list of your users)
2. Focus brute-force password attacks only on confirmed valid accounts

**The fix:** Always return the same generic message regardless of what failed:
```json
// ✅ Correct for all auth failures
{
  "code": 401,
  "error": "UNAUTHORIZED",
  "message": "Invalid email or password"
}
```

The user doesn't know if their email was wrong or their password was wrong. Neither does the attacker.

### 3. Never Log Sensitive Data

Logs are not as private as you think. Major data breaches frequently involve **leaked log files**.

**Never log:**
- Passwords (even hashed)
- Credit card numbers
- API keys / tokens
- Full email addresses (use user ID instead)
- Any PII that's not strictly necessary for debugging

**What to log instead:**
```javascript
// ❌ Dangerous
logger.error('Auth failed', { email: 'user@example.com', password: 'attempted_password' });

// ✅ Safe
logger.error('Auth failed', { userId: 'usr_42', requestId: 'abc-123' });
```

Always log **user ID** + **correlation/request ID** — enough to debug, no sensitive data.

---

## 7. Error Propagation and Error Boundaries

### Error Propagation (Bubbling)

Errors should propagate upward through layers, gaining more context at each level:

```
Repository error: "PG error: 23505 unique_violation on users.email"
    ↓ bubble up
Service adds context: "Failed to create user - email conflict"
    ↓ bubble up
Handler/Middleware translates: "400 - Email already taken"
```

Each layer adds business context without exposing internal details.

### Error Boundaries

In microservice architectures, an error in one service should not cascade and take down other services.

Strategies:
- **Separate processes** — services run independently; one crashing doesn't kill others
- **Timeouts** — if Service A calls Service B and B hangs, A should time out and handle the failure rather than hanging indefinitely
- **Message queues** — async communication (RabbitMQ, SQS) means a bug in the consumer doesn't affect the producer

---

## 8. Summary — The Fault-Tolerant Mindset

| Principle | Practice |
|---|---|
| **Errors are inevitable** | Design for failure, not against it |
| **Fail fast** | Validate config at startup. Reject bad data immediately. |
| **Detect early** | Health checks, monitoring, performance metrics |
| **Recoverable errors** | Retry with exponential backoff |
| **Non-recoverable errors** | Graceful degradation, fallbacks, containment |
| **Global error handler** | One place for all error handling logic |
| **Bubble errors up** | From repository → service → handler → global middleware |
| **Never expose internals** | Translate DB errors, use generic 500 messages |
| **Auth errors are generic** | "Invalid email or password" — always, regardless of what failed |
| **Never log sensitive data** | User ID only, never email/password/tokens/card numbers |

---

## One-Line Takeaways

> The question isn't whether errors will happen. It's whether you were ready when they did.

> The best error handling starts before errors happen — validate config at startup, implement health checks, monitor performance degradation.

> Global error handling is a one-time investment that pays immediately and indefinitely. Set it up first.

> Generic error messages protect your users. Specific error messages help attackers.

---

*Next lecture → Config Management — How to manage environment variables and configuration across development, staging, and production.*
