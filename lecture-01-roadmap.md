# Lecture 1 — Roadmap for Backend from First Principles
> Source: [Sriniously — YouTube](https://youtu.be/0Rwb4Xmlcwc?si=r16K8enPmbaYPxi0)  
> Series: Backend from First Principles

---

## What This Lecture Is About

This is the **roadmap video** — no code, no deep dives yet. The goal is to give you the **big picture** of what backend engineering actually is, why learning it from first principles matters, and what all topics you'll cover in this series.

Think of this as the **table of contents** of your entire backend journey — but explained with context so you understand *why* each topic exists.

---

## 1. What is Backend Engineering — The Real Scope

Most beginners think backend = writing CRUD APIs (Create, Read, Update, Delete). That is a tiny slice of what backend actually is.

**Real backend engineering is about building systems that are:**

| Property | What it means |
|---|---|
| **Reliable** | Works correctly even when things go wrong |
| **Scalable** | Handles 10 users and 10 million users |
| **Fault-tolerant** | Doesn't fully crash if one part fails |
| **Maintainable** | Easy to understand, change, and extend over time |
| **Efficient** | Uses resources (CPU, memory, network) wisely |

> 💡 CRUD is the *starting point*, not the destination.

---

## 2. Why Most People Struggle to Learn Backend

Sriniously identifies two root causes:

### Problem 1 — Learning with a Limited Scope
Most people learn from:
- A college course
- A bootcamp
- A single online tutorial

These give you a narrow starting point. You then spend *years* filling in gaps through trial and error or help from senior developers. It's slow and painful.

### Problem 2 — Learning Through a Framework First
Many people start with:
- Express (Node.js)
- Spring Boot (Java)
- Ruby on Rails
- Django / FastAPI (Python)

The problem: **you start seeing every problem through that framework's lens.** You learn *how Express solves it*, not *why the problem exists* or *how it's solved fundamentally*.

**Real example from the video:**  
You worked with Ruby on Rails for years. Your company migrates to Golang for performance. How much of your knowledge transfers?

If you learned Rails tricks → very little transfers.  
If you learned *why Rails does what it does* → almost everything transfers.

> 💡 Frameworks come and go. Fundamentals are forever.

---

## 3. What This Series Covers — Every Topic Explained

Below is every topic in the series with a one-line explanation of *why it exists*. You'll go deep on each in later lectures.

---

### 🔹 High-Level Understanding
Before any code — understand how a request travels from your browser, through the internet, through firewalls, to a backend server on AWS, and how the response comes back.

**You'll learn:**
- How clients and servers communicate
- What happens at each "hop" in the network
- What a request and response actually look like

---

### 🔹 HTTP Protocol
HTTP is the *language* clients and servers speak. Everything in backend runs on top of HTTP.

**You'll learn:**
- Raw HTTP message format
- HTTP headers — request headers, response headers, security headers, representation headers
- HTTP methods — GET, POST, PUT, PATCH, DELETE and their semantics
- CORS — what it is, simple requests vs preflight requests
- HTTP response structure and status codes
- HTTP caching — ETags, max-age headers
- HTTP/1.1 vs HTTP/2 vs HTTP/3 — differences and why they matter
- Content negotiation between client and server
- Persistent connections
- Compression — gzip, deflate, Brotli
- SSL/TLS and HTTPS

---

### 🔹 Routing
Routing = mapping a URL + HTTP method to the correct piece of code.

**You'll learn:**
- How routing connects URLs to server-side logic
- Path parameters vs query parameters
- Types of routes: static, dynamic, nested, hierarchical, wildcard, catch-all, regex-based
- API versioning strategies
- How to deprecate old routes properly
- Route grouping (helps with versioning, permissions, shared middleware)
- Securing routes
- Optimizing route matching performance

---

### 🔹 Serialisation and Deserialisation
Before data can travel over a network, it must be converted to a transferable format. After receiving, it must be converted back.

**You'll learn:**
- What serialisation and deserialisation actually are
- Text formats (JSON, XML) vs binary formats (Protobuf) — performance tradeoffs
- How different languages handle it (Python dicts, Go structs, JS objects)
- JSON deep dive — data types, nested objects, collections
- Common JSON errors — null values, date/timezone issues, missing fields
- Custom serialisation
- Error handling in serialisation
- Security — injection attacks, schema validation before processing
- Performance — compression, eliminating unnecessary fields

---

### 🔹 Authentication and Authorisation
Who are you? What can you do?

**You'll learn:**
- Stateful vs stateless authentication
- Basic auth, Bearer token auth
- Sessions, JWTs, Cookies
- OAuth 2.0 and OpenID Connect (deep dive)
- API keys
- Multi-factor authentication (MFA)
- Salting and hashing — cryptographic techniques
- Authorisation: ACL, RBAC, ABAC
- Security best practices — securing cookies, CSRF, XSS, MITM prevention
- Audit logging — recording auth events for monitoring
- Timing attacks — how attackers exploit response time differences to guess credentials
- Consistent error messages to prevent information leakage

---

### 🔹 Validation and Transformation
Never trust user input. Validate everything at the entry point.

**You'll learn:**
- Types of validation:
  - **Syntactic** — is this string a valid email/phone/date format?
  - **Semantic** — date of birth can't be in the future; age must be 1–120
  - **Type** — is this actually a number? a string? an array?
- Client-side vs server-side validation — why both matter, why server-side is the real security gate
- Fail fast — return early to avoid unnecessary processing
- Transformations — type casting (string query param → number), date format conversion
- Normalisation — lowercase email, trim whitespace, add country code to phone
- Sanitisation — prevent SQL injection
- Complex validations — cross-field (password === confirmPassword), conditional (partnerName required only if married = true)
- Chained validation — lowercase → remove special chars → check length
- Error handling — meaningful messages, aggregating all errors in one response
- Obfuscating errors — "invalid credentials" not "wrong password" (prevents attacks)
- Performance — return early, avoid redundant checks

---

### 🔹 Middlewares
A middleware is a function that runs *between* the request arriving and the handler executing.

**You'll learn:**
- What middleware is and when to use it
- The request cycle role — pre-request and post-response middleware
- Chaining — middleware runs in sequence, passes control via `next()`
- Middleware order matters (auth before route handling, logging first, error handler last)
- Short-circuiting — middleware can stop the chain and return early (e.g., 401 if not authenticated)
- Common middlewares:
  - Security headers (X-Content-Type, Strict-Transport-Security, CSP)
  - CORS headers
  - CSRF protection
  - Rate limiting
  - Authentication
  - Logging and monitoring
  - Error handling (global, consistent API responses)
  - Compression (reduce response body size)
  - Body parsing (JSON, URL-encoded forms, file uploads)
- Performance — keeping middlewares lightweight, correct ordering

---

### 🔹 Request Context
Metadata that flows through the entire lifecycle of a single request — shared across middlewares, controllers, and services.

**You'll learn:**
- What request context is — request-scoped state
- Components: HTTP method, URL, headers, query params, body, user info (from auth middleware), request/trace ID, custom injected data
- Use cases — authentication, rate limiting, tracing, logging
- Connection between middlewares and context
- Timeouts — request timeouts, custom timeouts, cancellation signals
- Best practices — keep it lightweight, clean up after request ends, don't over-rely on it

---

### 🔹 Handlers, Controllers and Services
How to structure code so it doesn't become a giant mess.

**You'll learn:**
- The MVC pattern
- Responsibilities of each layer:
  - Handler/Controller — HTTP in/out only
  - Service — business logic
  - Repository — database
- Centralised error handling in handlers
- Consistent success and error response formats

---

### 🔹 CRUD Deep Dive
Create, Read, Update, Delete — but done *properly*.

**You'll learn:**
- How CRUD maps to HTTP methods and status codes
- Common APIs per method
- Pagination, search, sorting, filtering
- Best practices — strict validation, consistent formatting, limiting payload size, redacting sensitive fields

---

### 🔹 RESTful Architecture and Best Practices
REST is a *style guide* for building web APIs, not a protocol.

**You'll learn:**
- Designing APIs around resources (nouns not verbs)
- Sticking to HTTP semantics
- Filtering, pagination best practices
- Versioning — URI, header, query string, media type versioning
- OpenAPI spec-first design
- Content negotiation
- Exception handling
- Client-side caching support (ETags)
- Optimising large requests and responses

---

### 🔹 Databases
Where your data actually lives.

**You'll learn:**
- Relational (SQL) vs Non-relational (NoSQL) — when to use which
- ACID properties
- CAP theorem
- Basic querying and JOINs
- Schema design and indexing best practices
- Query optimisation, caching, connection pooling
- Constraints, transactions, concurrency
- ORMs — tradeoffs of using vs not using one
- Database migrations

---

### 🔹 Business Logic Layer (BLL)
The heart of your application — the rules and decisions that make your product unique.

**You'll learn:**
- The three layers: Presentation → BLL → Data Access
- Design principles: Separation of Concerns, Single Responsibility, Open/Closed, Dependency Inversion
- Components: Services, Domain Models (User, Order), Business Rules, Business Validation
- Service layer design best practices
- Error propagation from service layer to presentation layer

---

### 🔹 Caching
Store results of expensive operations so you don't repeat them.

**You'll learn:**
- Why caching is different from DB persistence
- Types: in-memory, browser, database caching
- Client-side vs server-side caching
- Caching strategies: Cache-Aside, Write-Through, Write-Behind/Write-Back, Read-Through
- Cache eviction strategies: LRU, LFU, TTL, FIFO
- Cache invalidation — manual, TTL, event-based
- Caching levels: L1 (in-memory, fast, small) and L2 (distributed, slower, large)
- Web app caching — static assets, API response caching via HTTP headers
- Database query caching with Redis (storing results of heavy JOINs)
- Cache hit/miss ratio and how to optimise it

---

### 🔹 Transactional Emails
Emails triggered by user actions — not marketing, but functional.

**You'll learn:**
- Common use cases (welcome, password reset, OTP, order confirmation)
- Anatomy of a transactional email — subject, preheader, body, CTA, footer
- Personalisation with dynamic parameters

---

### 🔹 Task Queuing and Scheduling
Move slow tasks out of the request cycle into the background.

**You'll learn:**
- Use cases — sending emails, image processing, payment webhooks, batch processing, clearing user data
- Scheduling — DB backups, recurring notifications, data sync, maintenance (clearing logs/caches)
- Components of a task queue — producer, queue, consumer, broker, backend
- Task dependencies — chain, parent-child
- Task groups — run multiple tasks concurrently, wait for all
- Error handling and retries in task queues
- Task prioritisation and rate limiting (e.g., payments before notifications)

---

### 🔹 Elasticsearch
Full-text search engine for when SQL `LIKE` queries aren't enough.

**You'll learn:**
- Why Elasticsearch exists and how it works — inverted index, TF-IDF, segments, shards
- Use cases — typeahead, log analytics, social media search, full-text search
- Creating and managing indexes
- Types of searching — basic, full-text, relevance scoring
- Optimisation — text vs keyword fields, analyzers, boosting, pagination
- Advanced patterns — filtering, aggregation, fuzzy search
- Kibana — visualising ES data
- Best practices — explicit field mappings, shard count, batch indexing, avoiding wildcards

---

### 🔹 Error Handling
Every system fails. How you handle failures defines the quality of your backend.

**You'll learn:**
- Error types — syntax, runtime, logical
- Strategies — fail-safe, fail-fast, graceful degradation, prevention
- Practices — catch early, don't swallow errors, custom error types, logging, stack traces
- Global error handlers
- User-facing errors — friendly messages, actionable feedback
- Monitoring — Sentry, ELK stack
- Alerts — email-based, Slack-based

---

### 🔹 Config Management
Keep environment-specific settings out of your code.

**You'll learn:**
- What config management is and why it decouples logic from environment
- Use cases — different DBs per environment, feature flags, secrets management
- Types of config — static (DB URL, API endpoints), dynamic (feature flags, rate limits), sensitive (credentials, tokens)
- Config sources — .env files, JSON, YAML
- Environment variables vs command-line flags vs static files
- Best practices for managing secrets safely

---

### 🔹 Logging, Monitoring and Observability
You can't fix what you can't see.

**You'll learn:**
- Differences between logging, tracing, monitoring, and observability
- Log types — system, application, access, security logs
- Log levels — DEBUG, INFO, WARN, ERROR, FATAL
- Structured vs unstructured logging
- Best practices — centralised logging, log rotation, contextual logs, never log passwords
- Monitoring types — infrastructure, APM (Application Performance Monitoring), uptime
- Tools — Prometheus, Grafana
- Alerts — defining thresholds, avoiding alert fatigue, keeping alerts actionable
- Observability three pillars — Logs, Metrics, Traces
- Security and compliance in log management

---

### 🔹 Graceful Shutdown
Stop your server without breaking things in progress.

**You'll learn:**
- Why graceful shutdown matters
- Use cases — server restarts, cloud scale-in, microservices, long-running jobs
- OS signals — SIGTERM, SIGINT, SIGKILL
- Steps: capture signal → stop accepting requests → finish in-flight requests → close DB connections/files → terminate
- How Kubernetes uses SIGTERM

---

### 🔹 Security
Protect your system and your users.

**You'll learn:**
- Attack types — SQL injection, NoSQL injection, XSS, CSRF, broken auth, insecure deserialisation
- Principles — least privilege, defence in depth, fail secure, separation of duties, security by design
- Input validation and sanitisation
- Rate limiting
- Content Security Policy (CSP)
- CORS
- SameSite cookies
- Monitoring security events

---

### 🔹 Scaling and Performance
Build systems that work under load.

**You'll learn:**
- Performance metrics — response time, resource utilisation, bottleneck identification
- Caching and DB optimisation — avoid N+1 queries, proper JOINs, lazy loading
- Database indexes for read-heavy fields
- Batch processing for large datasets
- Avoiding memory leaks — close file handles, DB connections, clean up long processes
- Reducing network overhead — payload compression, smaller payloads
- Performance testing and profiling
- Best practices — clear code first (no premature optimisation), modular code, graceful degradation under load, offload non-critical tasks to background jobs

---

### 🔹 Concurrency and Parallelism
Do multiple things at once — the right way.

**You'll learn:**
- Concurrency vs parallelism
- Concurrency for I/O-bound tasks
- Parallelism for CPU-bound tasks

---

### 🔹 Object Storage and Large Files
Store files at scale.

**You'll learn:**
- When to use object storage (AWS S3)
- Managing large files with chunking and streaming
- Multipart file uploads

---

### 🔹 Real-Time Backend Systems
Push data to clients as it happens.

**You'll learn:**
- WebSockets
- Server-Sent Events (SSE)
- Pub/Sub architecture

---

### 🔹 Testing and Code Quality
Ship code you can trust.

**You'll learn:**
- Test types — unit, integration, E2E, functional, regression, performance, load/stress, UAT, security
- Test-Driven Development (TDD)
- Automating tests in CI/CD
- Code quality tools — linting, formatting
- Quality metrics:
  - **Cyclomatic complexity** — counts possible paths through a function (lower = simpler)
  - **Maintainability index** — scores how easy the codebase is to maintain

---

### 🔹 12 Factor App
A set of 12 principles for building production-grade, scalable apps.

*(Deep dive in a later lecture)*

---

### 🔹 OpenAPI Standards
A standard way to document and design REST APIs.

**You'll learn:**
- Why OpenAPI exists and its benefits
- Ecosystem — Swagger UI, Codegen, Postman
- Swagger → OpenAPI transition and active versions
- Key concepts — paths, parameters, schemas, request/response definitions
- OpenAPI document structure — metadata, paths, components, security, responses
- New features in OpenAPI 3.0 and 3.1
- API-First Development — write the spec first, then build the API

---

### 🔹 Webhooks
Let other systems notify yours when something happens.

**You'll learn:**
- Use cases — payment notifications, third-party integrations
- API polling vs webhooks (client-initiated vs server-initiated)
- Key components — webhook URL, event triggers, payload, HTTP method, response handling
- Best practices — signature verification, HTTPS, retry logic, logging
- Testing webhooks with ngrok
- Real examples — Stripe, GitHub, Slack, Twilio

---

### 🔹 DevOps for Backend Engineers
Deploy, run, and manage your backend in production.

**You'll learn:**
- CI/CD — continuous integration, delivery, deployment
- DevOps practices — infrastructure as code, config management, version control
- Docker — containerise your app
- Kubernetes — orchestrate containers at scale
- CI/CD pipelines
- Scaling — horizontal vs vertical
- Deployment strategies — blue-green, rolling deployment

---

## 4. The Core Philosophy of This Series

> *"Don't learn Rails. Learn why Rails does what it does."*

Three things Sriniously wants you to walk away with:

1. **Framework-agnostic knowledge** — learn the underlying system, not the tool. Tools change. Principles don't.
2. **The big picture** — understand how all these concepts connect, not just each one in isolation.
3. **First principles thinking** — when you face a new problem, you can reason from fundamentals instead of Googling "how to do X in Express".

---

## 5. How to Use These Notes (and the Series)

- **Watch each lecture** → come back to the corresponding notes file
- **Don't rush** — each topic builds on the previous one
- **Build small prototypes** as you go — learning by doing sticks
- **This roadmap video = your compass** — whenever you feel lost, come back here and see where you are in the journey

---

## Summary — Topics at a Glance

```
Foundation
  ├── High-Level Understanding (how the internet works)
  ├── HTTP Protocol (the language of the web)
  └── Routing (matching URLs to code)

Input Layer
  ├── Serialisation / Deserialisation
  ├── Validation and Transformation
  └── Middlewares + Request Context

Architecture
  ├── Handlers, Controllers, Services
  ├── CRUD Deep Dive
  ├── RESTful Architecture
  └── Business Logic Layer

Data Layer
  ├── Databases (SQL, NoSQL, ACID, CAP)
  ├── Caching (Redis, strategies, eviction)
  └── Elasticsearch (full-text search)

Async Systems
  ├── Task Queuing and Scheduling
  ├── Transactional Emails
  └── Real-Time (WebSockets, SSE, Pub/Sub)

Reliability
  ├── Error Handling
  ├── Graceful Shutdown
  ├── Config Management
  └── Logging, Monitoring, Observability

Performance and Scale
  ├── Scaling and Performance
  ├── Concurrency and Parallelism
  └── Object Storage and Large Files

Production Standards
  ├── Security
  ├── Testing and Code Quality
  ├── 12 Factor App
  ├── OpenAPI Standards
  ├── Webhooks
  └── DevOps for Backend Engineers
```

---

*Next lecture → High-Level Understanding: How a request travels from your browser to a backend server and back.*
