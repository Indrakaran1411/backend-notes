# ⚡ Backend Scaling & Performance Engineering — Complete In-Depth Notes
### Lecture 21 — Part 1

> **Goal of these notes:** Build a deep intuition for *how systems behave under load*, where bottlenecks actually hide, and how to think clearly when your system starts falling apart at scale.

---

## 🧠 The Core Mental Model

> **Performance is not just about making things fast — it is about understanding what slows things down and why.**

Before learning any scaling technique, you need to understand:
1. **How to measure** performance (not just feel it)
2. **Where bottlenecks actually live** (not where you assume they are)
3. **How resources behave** under increasing load (it is not linear — it is exponential)

---

## 📚 Table of Contents

1. [What Does "Fast" Actually Mean?](#1-what-does-fast-actually-mean)
2. [Latency — The Core Metric](#2-latency--the-core-metric)
3. [Percentiles — P50, P90, P99](#3-percentiles--p50-p90-p99)
4. [Throughput](#4-throughput)
5. [Utilization and the Exponential Curve](#5-utilization-and-the-exponential-curve)
6. [Bottlenecks — Find Before You Fix](#6-bottlenecks--find-before-you-fix)
7. [Profiling](#7-profiling)
8. [Distributed Tracing](#8-distributed-tracing)
9. [Database Performance](#9-database-performance)
   - [N+1 Query Problem](#91-n1-query-problem)
   - [Indexes](#92-indexes)
   - [Connection Pooling](#93-connection-pooling)
10. [Caching](#10-caching)
    - [Cache Invalidation](#101-cache-invalidation)
    - [Local vs Distributed Cache](#102-local-vs-distributed-cache)
    - [Caching Patterns](#103-caching-patterns)
    - [Cache Hit Rate](#104-cache-hit-rate)
11. [Scaling Strategies](#11-scaling-strategies)
    - [Vertical Scaling](#111-vertical-scaling-scaling-up)
    - [Horizontal Scaling](#112-horizontal-scaling-scaling-out)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. What Does "Fast" Actually Mean?

When a user calls your system fast or slow, they are talking about this entire journey:

```
User clicks button
       │
       ▼
Browser sends HTTP request across the internet
       │
       ▼
Your Server receives request
       │
       ▼
Server processes → queries DB → maybe calls external API (e.g. send email)
       │
       ▼
Server sends JSON response back across the internet
       │
       ▼
Browser parses JSON → renders cards on screen
       │
       ▼
User sees result ✅
```

**The total time from click → result appearing on screen = Latency.**

That is what users feel. That is what "fast" and "slow" means in practice.

---

## 2. Latency — The Core Metric

> **Latency** = How long a single request takes from origin to receiving a response.

### Why Latency Varies Per Request

No two requests take the same time. In the real world:

| Reason | Effect |
|--------|--------|
| Request hit a CDN or in-memory cache (Redis) | Fast — maybe 50ms |
| Request hit an idle server | Fast |
| Request hit a busy server processing 50 other requests | Slow — maybe 200ms |
| Request needed a complex DB query | Slow |
| Request had to call an external API | Slow |

### Why Averages Are Useless for Latency

> **Average latency is one of the most misleading numbers in performance engineering.**

**Example:**
- 1,000 requests measured
- Average latency = 100ms ← looks great!
- Reality: 99% of requests finished in 50ms, but **1% took 5 seconds**

**At scale:**
- 1 million requests/day
- 1% = **10,000 users waited 5 seconds** for a response
- The average told you everything was fine

The average **hides the pain of the worst-case users** completely.

> **Rule: Never use averages to measure performance. Always use percentiles.**

---

## 3. Percentiles — P50, P90, P99

Percentiles tell you the *distribution* of latency — how different groups of users actually experience your system.

### The Three Critical Percentiles

| Term | What It Means | Example |
|------|--------------|---------|
| **P50** (50th percentile) | 50% of users experience this latency or less | P50 = 100ms → half your users get ≤100ms |
| **P90** (90th percentile) | 10% of users experience *this much or more* | P90 = 500ms → 10% of users wait ≥500ms |
| **P99** (99th percentile) | 1% of users experience *this much or more* | P99 = 2s → 1% of users wait ≥2 seconds |

### Memory Trick

> **P-something → Subtract from 100 = % of users experiencing that latency or worse**
> - P90 → 100 - 90 = **10%** of users are unhappy
> - P99 → 100 - 99 = **1%** of users are unhappy

### Why Engineers Focus on P99 and P95

**The requests that land in the P99 zone are your most complex workflows:**
- Payment processing
- Purchase confirmation
- Account creation with multiple DB writes
- Any flow that calls external services

**These users are often your highest-value customers** — paying users, users in checkout, users making important decisions. Their experience matters most.

> **Always optimize P99 and P95 first. P50 fixes are nice-to-have. P99 fixes protect your business.**

### Visual Representation

```
Latency
  ^
  │                                        ● P99 (2s)
  │                               ● P95
  │                      ● P90 (500ms)
  │            ● P75
  │   ● P50 (100ms)
  └──────────────────────────────────────►
     50%      75%      90%      95%    99%
              Percentage of users
```

---

## 4. Throughput

> **Throughput** = How many requests your system can handle in a given time period.
> Measured in: **requests per second (RPS)** or requests per minute.

### Why Throughput Matters (Not Just Latency)

A system can look fast at low load but fall apart at high load:

```
10 RPS  → Latency: 50ms   ✅ Great
100 RPS → Latency: 200ms  ✅ Acceptable
1000 RPS → Latency: 2000ms ❌ Broken
```

### Real-World Questions Throughput Answers

- Can our system survive **Black Friday** traffic spikes?
- If we send an email campaign to 500,000 users at once and they all click the link, what happens?
- We got featured on a popular podcast — how many concurrent users can we support?

### Throughput + Latency Together

These two metrics are **always connected**:

```
As throughput increases ──► Latency increases slowly at first
                        ──► Then increases DRAMATICALLY near capacity
```

This is not linear. It is an exponential curve (covered in Section 5).

---

## 5. Utilization and the Exponential Curve

> **Utilization** = The percentage of your system's capacity that is currently in use.

| Utilization | System State |
|-------------|-------------|
| 0% | Idle, doing nothing |
| 50–70% | Healthy, smooth operation |
| 80–90% | Noticeable slowdowns, unpredictable |
| 100% | At the brink of collapse |

### The Most Important (Counter-Intuitive) Relationship

You might expect latency to grow *linearly* with utilization:

```
❌ What you expect:
Latency │         /
        │        /
        │       /
        │      /
        └──────────► Utilization
```

**What actually happens:**

```
✅ Reality:
Latency │                      /
        │                    /
        │                  /
        │               /
        │          __--
        │     __--
        └──────────────────────► Utilization
         0%   50%   80%  90% 100%
```

> **Near 100% utilization, latency grows exponentially — not linearly.**

### The Highway Analogy

| Highway Capacity | What Happens |
|-----------------|--------------|
| 50% full | Cars flow smoothly, easy to overtake |
| 80% full | Noticeable slowdowns, lane changes require thought |
| 90% full | Completely unpredictable — sometimes flows, sometimes jams |
| 100% full | Nobody moves. Complete gridlock. |

**The same thing happens to your backend server.**

### The Practical Rule

> **Never run production systems at 100% utilization. Always keep 20–40% headroom.**

**Why?**
- Traffic comes in **bursts** (not as a steady stream)
- Even if your average load is 50%, a burst can instantly spike you to 150% of capacity
- Without headroom, that burst crashes your system

**Real-world standard: Run at 60–80% utilization, reserve 20–40% for bursts.**

---

## 6. Bottlenecks — Find Before You Fix

> **Bottleneck** = The specific component in your system that is causing slowness.

### The #1 Mistake Engineers Make

When a system is slow, engineers jump to **textbook solutions** without measuring:

```
System is slow
      │
      ▼
"Must be the database" ──► Add Redis caching (1 week of work)
      │
      ▼
System is still slow ❌
      │
      ▼
Actual cause: A synchronous logging function that took 500ms
              (which was never even looked at)
```

**The problem:** You spent a week implementing something that did not fix anything.

### Real Example from the Video

An API `/products/:id` was slow. Engineers assumed it was the database and added caching. After deploying — still slow.

Then they added **timing measurements** in the code:

| Component | Time Taken |
|-----------|-----------|
| Database query | 10ms ✅ Fast |
| Redis cache lookup | 5ms ✅ Fast |
| **Synchronous log write to remote service** | **500ms ❌ THE REAL CULPRIT** |

The logging function was calling a remote service (like Elasticsearch) **synchronously** — meaning the entire API request was paused and waiting for the log write to complete before moving on.

### Fix for This Specific Example

Make the logging **asynchronous** — fire and forget. Log in the background, don't block the request.

```javascript
// ❌ SLOW — synchronous logging blocks the whole request
await logger.write(logData);  // waits 500ms
return response;              // only returns after logging finishes

// ✅ FAST — async logging, request returns immediately
logger.write(logData);        // fires in background, doesn't wait
return response;              // returns immediately
```

### The Golden Rule

> **Never guess which component is causing the bottleneck. Always measure first.**

**Common non-obvious bottlenecks:**
- JSON serialization of a huge response payload
- Calling an external API inside a loop
- Synchronous operations that should be async
- The network transmission itself (response payload too large)
- Database connection setup overhead (not the query itself)

---

## 7. Profiling

> **Profiling** = The practice of measuring exactly where your application spends its time during execution.

### How It Works

A profiler attaches to your running application and records:
- Which functions are executing
- When they started and ended
- How long each function took
- Which functions called which other functions

### Flame Graphs

The raw output of a profiler is overwhelming (thousands of function calls). **Flame graphs** make it readable:

```
┌────────────────────────────────────────────────────┐  ← handleRequest() (wide = slow!)
│  ┌──────────────────────────┐  ┌─────────────────┐ │
│  │     queryDatabase()      │  │  serializeJSON()│ │
│  │  ┌────────┐  ┌─────────┐ │  └─────────────────┘ │
│  │  │ parse()│  │execute()│ │                       │
│  │  └────────┘  └─────────┘ │                       │
│  └──────────────────────────┘                       │
└────────────────────────────────────────────────────┘
```

- **Wider = more time spent** → focus here
- **Stacked = called by the function below**

### Important Limitation of Profilers

Profilers are excellent at measuring **CPU-bound tasks** (actual computation):
- Sorting algorithms
- Data transformation
- Business logic calculations

Profilers are **poor at measuring IO-bound tasks**:
- Database queries (waiting for DB)
- External API calls (waiting for response)
- File reads/writes (waiting for disk)

> **In typical SaaS backend applications, performance problems are almost always IO-bound, not CPU-bound.**

This is why we need a different tool for IO-bound performance issues — **Distributed Tracing**.

---

## 8. Distributed Tracing

> **Distributed Tracing** = Following a single request as it flows through your entire system, recording exactly how long it spent at each component.

### What It Measures

For a request to `GET /products/5`:

```
Request enters system ──────────────────────────► Total: 820ms
│
├── Business logic code ────────────────────── 2ms  ✅
│
├── Redis cache check ───────────────────────── 3ms  ✅
│
├── Database query ──────────────────────────── 800ms ❌ BOTTLENECK
│
└── JSON serialization ──────────────────────── 15ms ✅
```

Now you know: **your database query is the problem.** Everything else is fine.

### Why Distributed Tracing > Profiling for Backends

| | Profiling | Distributed Tracing |
|--|-----------|-------------------|
| Good at | CPU computation | IO waits (DB, APIs, network) |
| Shows | Function call time | Request lifecycle per component |
| Best for | Algorithm optimization | Finding the slow DB query or API call |

### Popular Tools

- **Jaeger** (open source)
- **Zipkin** (open source)
- **Datadog APM** (paid)
- **AWS X-Ray** (if on AWS)
- **OpenTelemetry** (vendor-neutral standard)

> **Rule: Use profiling for CPU-heavy code. Use distributed tracing for IO-heavy backends (which is most web apps).**

---

## 9. Database Performance

Databases are the most common bottleneck in backend applications. They do the hard work:
- Persist data durably to disk
- Handle concurrent reads and writes
- Execute complex queries across billions of rows
- Guarantee consistency under failure

There are three major performance concerns with databases:

---

### 9.1 N+1 Query Problem

#### What It Is

Making **1 query to get N items, then N more queries to get details** about each item.

**Frontend Example (to build intuition):**

You need to display 20 blog posts, each with its author's name.

```javascript
// ❌ N+1 pattern
const posts = await fetch('/api/posts');          // 1 query → 20 posts
for (const post of posts) {
  const author = await fetch(`/api/users/${post.authorId}`); // 20 more queries!
}
// Total: 21 API calls to show 1 page
```

**The Real Problem — it happens at the server/DB level:**

```javascript
// ❌ N+1 at database level (the actual problem)
const posts = await db.select('posts');   // Query 1: get all posts
for (const post of posts) {
  const author = await db.select('users').where({ id: post.authorId }); // Query per post!
}
// For 1000 posts → 1001 database queries
// At 5ms per query → 5,005ms = 5 seconds of latency
```

#### Why It Happens

ORM code looks like normal programming language code, so you don't notice you're firing a query inside a loop.

#### The Fix — Bulk Fetching

```javascript
// ✅ CORRECT — 2 queries total regardless of how many posts
const posts = await db.select('posts');

// Collect all author IDs first
const authorIds = posts.map(post => post.authorId);

// Fetch ALL authors in ONE query
const authors = await db.select('users').whereIn('id', authorIds);

// Join them in memory (in your application code)
const authorMap = authors.reduce((map, author) => {
  map[author.id] = author;
  return map;
}, {});

const postsWithAuthors = posts.map(post => ({
  ...post,
  author: authorMap[post.authorId]
}));
```

**Result:** Always 2 queries, no matter if you have 20 or 20,000 posts.

#### ORM Built-In Solutions

| ORM | Solution |
|-----|---------|
| Django | `select_related()` (foreign keys), `prefetch_related()` (many-to-many) |
| Ruby on Rails | `.includes(:author)` |
| TypeORM | `.leftJoinAndSelect()` |
| Prisma | `include: { author: true }` |
| Drizzle | `.leftJoin()` |

> **Rule: Never fetch related data in a loop. Use JOINs or ORM bulk-fetch methods.**

---

### 9.2 Indexes

#### The Library Analogy

Imagine a library with 1 million books and **no catalog**. A reader asks for all books by "John Green." You have to walk every single shelf and check every single book. That takes **3 days**.

Now imagine a catalog organized alphabetically by author. You look up "Green, John" → get exact shelf locations → fetch all books in **3 minutes**.

**That catalog = a database index.**

#### What an Index Actually Is

An index is a **separate data structure** (usually a B-tree) that maintains a **sorted copy of a column's values** with pointers to the actual rows.

```
Books table (no index):
Row 1: id=1, title="Fault in Our Stars", author_id=3
Row 2: id=2, title="Harry Potter",       author_id=7
Row 3: id=3, title="Looking for Alaska", author_id=3
...
Row 1,000,000: ...

Query: SELECT * FROM books WHERE author_id = 3
→ Must scan ALL 1,000,000 rows (Full Table Scan) ❌ Slow
```

```
Index on author_id:
author_id=1 → [row 45, row 891, row 12034, ...]
author_id=2 → [row 7, row 234, ...]
author_id=3 → [row 1, row 3, row 5672, ...]  ← Jump directly here ✅
author_id=4 → [...]

Query: SELECT * FROM books WHERE author_id = 3
→ Jump to author_id=3 in index → fetch only those rows ✅ Fast
```

#### Performance Impact

| Scenario | Without Index | With Index |
|----------|--------------|-----------|
| 1 million rows, query by author | ~4 seconds (full scan) | ~40ms (index scan) |
| Improvement | — | **100x faster** |

#### The Cost of Indexes

> **Indexes are not free. Every index has two costs.**

**Cost 1: Storage**
- The sorted copy of the column values takes disk space
- Grows proportionally with your table size

**Cost 2: Write Overhead (More Serious)**
- Every `INSERT`, `UPDATE`, or `DELETE` on the table must **also update every index on that table**
- If you index every column → every write operation updates every index → writes become very slow

```
Table with 5 indexes:
INSERT into books ──► Updates books table
                 ──► Updates index 1
                 ──► Updates index 2
                 ──► Updates index 3
                 ──► Updates index 4
                 ──► Updates index 5
= 6 operations instead of 1
```

#### Index Strategy — What to Index

**Always index (automatically):**
- Primary key (`id`) — databases do this for you

**Index when you know it will be queried frequently:**
- Foreign keys used in JOINs (`author_id`, `user_id`)
- Columns used in WHERE clauses for common queries
- Columns used in ORDER BY for frequent sorts

**Don't index:**
- Every column just in case
- Columns that are rarely queried
- Tables that are mostly written to (high write, low read)

#### Composite Index

Index on **multiple columns together** for queries that always filter by both:

```sql
-- If you frequently run:
SELECT * FROM posts WHERE user_id = 5 AND created_at > '2025-01-01'

-- Create a composite index (order matters!):
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at);

-- This index HELPS: WHERE user_id = 5 AND created_at > ...  ✅
-- This index HELPS: WHERE user_id = 5                       ✅ (left prefix)
-- This index DOES NOT HELP: WHERE created_at > ...          ❌ (skips first column)
```

> **Rule: In a composite index, the order matters. It helps queries that use the leftmost columns first.**

#### Covering Index

An index that **contains all columns needed by the query** — the database never needs to touch the actual table at all.

```sql
-- Query only needs department name and ID
SELECT id, name FROM departments WHERE name = 'Engineering';

-- Covering index includes both columns:
CREATE INDEX idx_dept_name_id ON departments(name, id);
-- Database serves the query entirely from the index — fastest possible ✅
```

#### How to Find Which Columns Need Indexing

Use `EXPLAIN ANALYZE` before your query:

```sql
EXPLAIN ANALYZE
SELECT * FROM posts
JOIN users ON posts.author_id = users.id
WHERE posts.created_at > '2025-01-01'
GROUP BY users.id;
```

Look for:
- **`Seq Scan`** (Sequential Scan) → No index being used → Possible candidate for indexing
- **`Index Scan`** → Index is being used ✅
- **`Bitmap Index Scan`** → Index used for range queries ✅

After adding an index, run `EXPLAIN ANALYZE` again and verify it now shows `Index Scan`.

---

### 9.3 Connection Pooling

#### The Hidden Cost of Database Connections

Every time your backend connects to the database, this happens:

```
1. TCP three-way handshake (SYN → SYN-ACK → ACK)
2. Authentication (username/password verification)
3. Encryption negotiation (TLS handshake)
4. Session state setup
5. Database allocates memory for the connection (several MB)
```

All of this takes time and resources — **every single time** you establish a connection.

If your app opens a new connection for every database query and immediately closes it after:

```
Request 1 → Open connection → Query → Close connection → Pay full setup cost
Request 2 → Open connection → Query → Close connection → Pay full setup cost again
Request 3 → Open connection → Query → Close connection → Pay full setup cost again
...
```

At scale with thousands of requests per second, this adds up massively.

#### The Second Problem: Connection Limits

PostgreSQL default: ~100–500 maximum connections (configurable but limited by RAM).

```
Traffic spike → 10,000 concurrent requests
Each request opens a new connection
→ Exhausts Postgres connection limit of 500
→ New connection requests get rejected
→ Your application crashes ❌
```

#### The Fix: Connection Pooling

A **connection pool** maintains a set of pre-opened, reusable connections:

```
Application Server
       │
       ▼
  ┌──────────────────┐
  │  Connection Pool  │  ← Maintains 20 open connections
  │  ● ● ● ● ●       │
  │  ● ● ● ● ●       │
  │  ● ● ● ● ●       │
  │  ● ● ● ● ●       │
  └──────────────────┘
       │
       ▼
   PostgreSQL DB
```

**How it works:**
1. Request needs a DB query → **borrows** a connection from the pool
2. Runs the query
3. **Returns** the connection to the pool (connection stays open)
4. Next request reuses the same connection — no setup cost

#### Internal vs External Pooling

**Internal Pooling** — each server instance manages its own pool:

```javascript
// Example: pg pool in Node.js
const pool = new Pool({
  max: 20,            // max 20 connections in this pool
  idleTimeoutMillis: 30000,
});
```

**Problem with internal pooling at scale:**
```
3 server instances × 20 connections each = 60 total connections to DB
→ Fine if DB limit is 100

Traffic spike → Kubernetes adds 10 more instances
→ 13 instances × 20 = 260 connections
→ Exceeds DB limit of 200 → DB crashes ❌
```

**External Pooling (PgBouncer for PostgreSQL)** — a single shared pool for all server instances:

```
Server Instance 1 ─┐
Server Instance 2 ─┤──► PgBouncer Pool (250 connections) ──► PostgreSQL DB
Server Instance 3 ─┤                    (centralized)
Server Instance 4 ─┘
```

- All instances share the **same centralized pool**
- You never accidentally exceed DB connection limits
- Traffic spikes trigger more server instances but they all share the same external pool

> **Rule: For production systems with horizontal scaling, always use an external pooler like PgBouncer (for Postgres). Internal pooling is fine for development and single-server setups.**

---

## 10. Caching

> **Core idea:** Store the result of an expensive operation so the next request can get it instantly without redoing the work.

```
Without cache:
Request → Complex DB Query (800ms) → Response ❌ Slow

With cache:
Request → Check Redis (5ms) → Cache HIT → Response ✅ Fast
                           → Cache MISS → DB Query (800ms) → Store in Redis → Response
```

**With just this one change, you can reduce latency from 800ms to 5ms for repeated requests.**

---

### 10.1 Cache Invalidation

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

**The core problem:** When the underlying data changes in your database, the cached version becomes **stale** (outdated). You need to invalidate (delete/update) the cache entry — but this is harder than it sounds.

Cache exists at multiple layers:
```
User's Browser Cache
       ↓
CDN Cache (Cloudflare, etc.)
       ↓
Reverse Proxy Cache
       ↓
Application Cache (Redis)
       ↓
Database
```

When data changes, you need to invalidate across **all these layers** consistently.

#### Two Invalidation Strategies

**Strategy 1: Time-Based Expiration (TTL)**

Set an automatic expiry time when storing in cache:

```javascript
// Cache expires automatically after 5 minutes
redis.set('user:123:profile', JSON.stringify(userData), 'EX', 300); // 300 seconds
```

| Pros | Cons |
|------|------|
| Simple to implement | Hard to pick the right TTL |
| Automatic, no manual work | Risk of serving stale data until TTL expires |
| Works well for slowly changing data | Too short TTL → high cache miss rate |

**How to pick TTL:** Depends on how frequently the data changes and how much staleness is acceptable.
- User profile → maybe 5 minutes is fine
- Stock price → maybe 1 second
- Static content (blog post) → maybe 24 hours

---

**Strategy 2: Event-Based Invalidation**

Explicitly delete the cache entry whenever the underlying data changes:

```javascript
// When user updates their profile:
async function updateUserProfile(userId, newData) {
  // 1. Update the database
  await db.update('users').set(newData).where({ id: userId });

  // 2. Immediately delete the cache entry
  await redis.del(`user:${userId}:profile`);
  // Next GET request will find no cache → fetch fresh from DB → re-cache
}
```

| Pros | Cons |
|------|------|
| Data is never stale | Must remember to invalidate at every update point |
| No need to guess the right TTL | If you forget one update path → stale data |

> **Best practice:** Use **event-based** for data you know when it changes. Use **TTL** as a safety net on top of that.

---

### 10.2 Local vs Distributed Cache

**Local Cache** — stored in your server's own memory:

```javascript
// Simple in-memory cache (Map)
const localCache = new Map();
localCache.set('user:123', userData);
```

**Problem:** If you have 10 server instances, each has its **own separate cache**:

```
Server 1: cache { user:123 = {name: "Alice"} }
Server 2: cache { user:123 = {name: "Alice"} }
...
Server 10: cache { user:123 = {name: "Alice"} }

User updates name to "Alicia" → Server 1's cache is invalidated
Server 2 through 10 still serve the old "Alice" ❌ Cache inconsistency
```

**Distributed Cache** — an external shared service (Redis, Memcached, Valkey):

```
Server 1 ─┐
Server 2 ─┤──► Redis (single shared cache) ──► {user:123 = {name: "Alice"}}
Server 3 ─┘

User updates name → Redis cache invalidated → ALL servers see fresh data ✅
```

| | Local Cache | Distributed Cache |
|--|-------------|-------------------|
| Speed | ~1ms (in memory) | ~5–50ms (network roundtrip) |
| Consistency | ❌ Inconsistent across servers | ✅ Consistent everywhere |
| Survives server restart | ❌ No | ✅ Yes |
| Best for | Single-server setups | Horizontally scaled apps |

**Tiered Caching (Best of Both Worlds):**

```
Request
   │
   ▼
Local Cache (in-memory, ~1ms)
   │ miss
   ▼
Redis Distributed Cache (~10ms)
   │ miss
   ▼
Database (~100-800ms)
```

Keep the **hottest / most frequently accessed** data in local cache. Everything else in Redis.

---

### 10.3 Caching Patterns

Three standard patterns for *when and how* to cache:

#### Pattern 1: Cache-Aside (Lazy Loading) ✅ Most Common

The application code manages the cache manually. Cache is populated on demand.

```
GET /users/123
       │
       ▼
Check Redis: "user:123" exists? ──YES──► Return cached data ✅ (fast)
       │
      NO
       │
       ▼
Query Database → Get user data
       │
       ▼
Store in Redis (with TTL)
       │
       ▼
Return response
```

```javascript
async function getUserProfile(userId) {
  // 1. Check cache first
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // 2. Cache miss — go to database
  const user = await db.findOne('users', { id: userId });

  // 3. Store in cache for next time
  await redis.set(`user:${userId}`, JSON.stringify(user), 'EX', 300);

  return user;
}
```

**When data is updated:** Delete the cache entry (`redis.del(...)`)

**Best for:** Read-heavy workloads where data doesn't change too often.

---

#### Pattern 2: Write-Through

Every write operation updates **both the database and the cache simultaneously**.

```javascript
async function updateUserProfile(userId, newData) {
  // Write to both at the same time
  await Promise.all([
    db.update('users').set(newData).where({ id: userId }),
    redis.set(`user:${userId}`, JSON.stringify(newData), 'EX', 300)
  ]);
}
```

| Pros | Cons |
|------|------|
| Cache always fresh — no stale data | Writes are slightly slower (two operations) |
| No cache misses on reads | Wastes cache space if data is rarely read after write |

---

#### Pattern 3: Write-Behind (Write-Back)

Write to cache **immediately** (fast), then **asynchronously** write to database in the background.

```javascript
async function updateUserProfile(userId, newData) {
  // 1. Write to cache instantly (very fast ~5ms)
  await redis.set(`user:${userId}`, JSON.stringify(newData));

  // 2. Asynchronously queue the DB write (don't await it)
  queueDatabaseWrite(userId, newData);

  return { success: true }; // Return immediately
}
```

| Pros | Cons |
|------|------|
| Fastest write response time | Risk of data loss if cache crashes before DB write |
| User gets instant feedback | Cache and DB temporarily inconsistent |
| Handles write bursts gracefully | More complex to implement |

**Best for:** High-write workloads where some data loss is acceptable (analytics, counters, logs).

---

### 10.4 Cache Hit Rate

> **Cache Hit Rate** = % of requests that successfully get their data from the cache (vs. having to go to the database).

```
Hit Rate = Cache Hits / Total Requests × 100
```

| Cache Hit Rate | Interpretation |
|---------------|----------------|
| 90%+ | Excellent — cache is working perfectly |
| 70–90% | Good — some room for improvement |
| 50–70% | Mediocre — review your caching strategy |
| < 50% | Poor — your cache is barely helping |

### Factors That Affect Cache Hit Rate

**1. TTL (Time-to-Live)**
- Too short TTL → items expire before they're accessed again → low hit rate
- Too long TTL → risk of stale data
- Find the sweet spot based on your data's change frequency

**2. Cache Size**
- Larger cache = more items can be stored = higher hit rate
- When cache is full, old items get evicted (LRU = Least Recently Used by default)
- Small cache → frequent evictions → items expire before next access

**3. Understanding User Access Patterns**
- If you don't know which data your users access most frequently, your caching strategy will be inefficient
- Use analytics and distributed tracing to understand access patterns
- Cache what users actually request most — not what you assume they request

---

## 11. Scaling Strategies

### 11.1 Vertical Scaling (Scaling Up)

> **Add more power to the same machine.**

```
Before:              After:
┌──────────────┐     ┌──────────────────────┐
│  Server      │     │  Server (upgraded)   │
│  4 cores     │ ──► │  16 cores            │
│  8 GB RAM    │     │  64 GB RAM           │
│  100 GB SSD  │     │  1 TB NVMe SSD       │
└──────────────┘     └──────────────────────┘
```

**Hardware knobs you can turn:**

| Resource | What You Upgrade | Effect |
|----------|-----------------|--------|
| **CPU** | More cores | More concurrent requests processed |
| **RAM** | More memory | Larger in-memory cache, more concurrent processes |
| **Storage** | Larger/faster SSD (NVMe) | Faster disk I/O for DB |
| **Network card** | 10 Gbps card | More network throughput |

#### Advantages

- **Simplicity** — No code changes, no architecture changes
- **No distributed systems complexity** — No load balancers, no syncing between servers
- **Cost effective at small scale** — One powerful machine < two smaller machines (in cost and maintenance)
- **No state management headaches** — Everything on one machine, no split-brain problems

#### Disadvantages

| Problem | Explanation |
|---------|-------------|
| **Hard ceiling** | Cloud providers have a max instance size. Once you hit it, you're stuck. |
| **Single point of failure** | One machine crashes → your entire service is down |
| **No geographic distribution** | Users in other continents experience high latency |
| **Downtime for upgrades** | Upgrading hardware often requires taking the server offline |

#### When Vertical Scaling Makes Sense

- Early-stage startups (simple, fast, cheap)
- Applications with unpredictable access patterns
- Before you've optimized queries and code
- When your traffic is predictable and manageable

> **Rule: Always optimize your code and queries before scaling. A well-optimized app on a small server beats a poorly optimized app on a huge server.**

---

### 11.2 Horizontal Scaling (Scaling Out)

> **Add more instances of the same server, all working together.**

```
Before:                        After:
┌──────────────┐               ┌──────────────┐
│  Server      │               │  Server 1    │
│  (all traffic│          ┌───►│              │
│   goes here) │          │    └──────────────┘
└──────────────┘  Traffic─┤    ┌──────────────┐
                           ├───►│  Server 2    │
                           │    └──────────────┘
                           │    ┌──────────────┐
                           └───►│  Server 3    │
                                └──────────────┘
```

The request distributor between them = **Load Balancer** (covered in Lecture 21 Part 2).

#### Advantages

| Advantage | Explanation |
|-----------|-------------|
| **No hard ceiling** | Add as many servers as needed — infinitely scalable in theory |
| **Redundancy** | One server dies → traffic redistributed to remaining servers automatically |
| **Geographic distribution** | Place servers in different regions → users connect to nearest server |
| **Elastic scaling** | Auto-scale up during traffic spikes, scale down at night |

#### The Math

```
1 server handles 1,000 RPS
5 servers handle ~5,000 RPS  (theoretical)
10 servers handle ~10,000 RPS (theoretical)
```

> Note: The word "theoretical" — in practice you don't get perfect linear scaling due to coordination overhead, shared database bottlenecks, etc.

#### Disadvantages / New Complexities Introduced

Horizontal scaling doesn't eliminate problems — it **transforms** them into a new set:

| New Problem | What You Need to Solve It |
|-------------|--------------------------|
| How to distribute traffic across servers? | Load Balancer + routing algorithm |
| How do servers stay in sync? | Shared database / Redis / event-driven messaging |
| How to handle server failures automatically? | Health checks + auto-failover |
| How to manage shared state? | Stateless servers + external state stores |
| How to manage 10+ servers instead of 1? | Kubernetes / container orchestration |
| What if the network between servers fails? | CAP theorem, distributed consensus |

### Vertical vs Horizontal — At a Glance

| | Vertical Scaling | Horizontal Scaling |
|--|-----------------|-------------------|
| **Approach** | Bigger machine | More machines |
| **Complexity** | Low — no code changes | High — distributed system complexity |
| **Cost** | Cheaper per unit at small scale | Cheaper per unit at large scale |
| **Ceiling** | Hard hardware limit | Virtually unlimited |
| **Failure Risk** | Single point of failure | Redundant — no single point |
| **Geographic reach** | One location | Multiple regions |
| **Best for** | Small-to-medium scale | Large scale, enterprise |

> **Practical approach used by most companies:**
> 1. Start with vertical scaling (simple, fast)
> 2. Optimize code, queries, and caching first
> 3. Move to horizontal scaling when vertical hits its limits or single point of failure becomes unacceptable

---

## 12. Quick Reference Cheat Sheet

### Performance Metrics

| Metric | Definition | Measured In |
|--------|-----------|-------------|
| **Latency** | Time for one request to complete | milliseconds (ms) |
| **Throughput** | How many requests system handles | requests/second (RPS) |
| **Utilization** | % of system capacity in use | percentage (%) |
| **P50** | 50% of users experience this latency or less | ms |
| **P99** | 1% of users experience this latency or more | ms |
| **Cache Hit Rate** | % of requests served from cache | percentage (%) |

### Key Rules to Remember

| Rule | Reason |
|------|--------|
| **Never use averages for latency** | Hides worst-case users (P99) |
| **Always measure before optimizing** | Prevents wasted effort on the wrong bottleneck |
| **Run at 60–80% utilization max** | Leave headroom for traffic bursts |
| **Never fetch related data in a loop** | N+1 query problem |
| **Don't index every column** | Indexes slow down writes |
| **Use external pooler at scale** | Prevents DB connection exhaustion |
| **Event-based cache invalidation > TTL** | Prevents stale data |
| **Optimize first, scale second** | Scaling a slow app just gives you a big slow app |

### Bottleneck Checklist (When Your System Is Slow)

```
Step 1: Add timing measurements / distributed tracing
         → Find EXACTLY which component is slow

Step 2: Is it the database?
         → Check for N+1 queries
         → Check for missing indexes (EXPLAIN ANALYZE)
         → Check connection pool configuration

Step 3: Is it an external API call?
         → Is it being called synchronously when it should be async?
         → Is it being called in a loop?

Step 4: Is it your application code?
         → Run a profiler → look at flame graph
         → Is JSON serialization on a huge payload the issue?

Step 5: Is the response payload too large?
         → Network transmission is the bottleneck
         → Consider pagination, compression, or GraphQL

Step 6: After identifying the bottleneck → fix THAT SPECIFIC THING
         → Only then consider caching
         → Only then consider scaling
```

---

*These notes cover Lecture 21 Part 1: Backend Scaling & Performance Engineering. Topics include latency, percentiles, throughput, utilization, bottleneck identification, profiling, distributed tracing, N+1 queries, database indexes, connection pooling, caching strategies, and vertical vs horizontal scaling.*

*Lecture 21 Part 2 covers: Load Balancers, CDNs, and further distributed systems concepts.*
