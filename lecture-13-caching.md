# Lecture 13 — Caching: The Secret Behind It All
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=estH64OkwxU)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Caching is the single most impactful performance technique in backend engineering. This lecture covers what caching is, why it exists, where it's used (from hardware to network to software), how in-memory databases like Redis work, the major caching strategies, eviction policies, and real-world use cases you'll implement as a backend engineer.

---

## 1. What is Caching?

### One-Line Definition
> Caching is a mechanism to decrease the amount of time and effort it takes to perform some amount of work.

### Technical Definition
Caching = keeping a **subset** of data in a location that is:
- **Faster to access** than the primary storage
- **Closer** to where it's needed

Key word: **subset**. You don't cache everything — only the data that's:
- Frequently accessed
- Expensive to compute or fetch
- Not changing rapidly

### The Pattern Behind Every Cache Use Case

Two situations where caching solves a real problem:

| Situation | Problem | Cache Solution |
|---|---|---|
| **Heavy computation** | Algorithm takes seconds, millions of users trigger it | Compute once, cache result, serve cached result instantly |
| **Large data transfer** | Serving terabytes to millions of users globally | Store copies closer to users (CDN / edge servers) |

> 💡 Whenever you see heavy computation or large data delivery repeated frequently — that's where caching belongs.

---

## 2. Real-World Examples — Understanding the Gravity

### Google Search
- Each search query involves crawling, indexing, and ranking billions of web pages — computationally expensive
- "What is the weather today" is searched millions of times daily
- Without caching: every query would trigger full re-computation → massive latency, server overload
- With caching: Google uses a **distributed in-memory caching system** globally. Results are cached. Same queries return instantly from cache.

**Cache hit** → instantly return cached result  
**Cache miss** → run the full algorithm, cache the result, return it

---

### Netflix (CDN)
- Serves terabytes of video content to millions of users worldwide
- A single movie is encoded in multiple resolutions (1080p, 720p, 480p) for different devices/networks
- Problem: All users worldwide requesting from one server in the US = high latency for users in Asia, Europe, Africa

**Solution: CDN (Content Delivery Network)**

```
Originating server (US)
        ↓ (content cached/replicated)
Edge servers in India, Europe, Asia, Africa...
        ↓
User in India requests video
→ Served from nearby Indian edge server (low latency)
Instead of: User in India → US server (high latency)
```

Netflix uses machine learning algorithms to decide **what to pre-cache** at each edge location based on regional viewing trends. They don't cache everything — just the subset most likely to be requested in that region.

---

### Twitter/X (Trending Topics)
- Identifying trending topics requires analyzing millions of real-time tweets using complex ML algorithms — very expensive computation
- Cannot run this algorithm for every user who visits the trending section (billions of users = server crash)

**Solution: Cache the trending topics**
- Every few minutes: run the expensive algorithm → store results in Redis (in-memory cache)
- User visits trending section → instant result from Redis (no re-computation)
- Trends don't change in seconds/minutes — safe to cache for a few minutes

> 💡 The key insight: if the data changes every few seconds, caching is risky. If data is stable for minutes/hours, caching is very safe and very beneficial.

---

## 3. Three Levels of Caching

As a backend engineer, you'll encounter caching at three levels. Most of your day-to-day work is at the software level, but understanding all three gives you the full picture.

---

### Level 1 — Network Level Caching

#### CDN (Content Delivery Network)
Geographically distributed servers that cache and serve content from the location closest to the user.

**Key terms:**
- **Edge server** / **Edge node** — a server geographically close to the user
- **PoP (Point of Presence)** — a region with multiple edge servers
- **Originating server** — the actual source server (e.g., Netflix HQ in US)

**How CDN works:**
```
1. User in Chennai types a URL into browser
2. Browser sends DNS query to resolve the domain
3. CDN's DNS system routes the request to nearest PoP (e.g., Mumbai)
4. Edge server at that PoP checks its cache:
   → Cache HIT:  serve content instantly from edge server ✅
   → Cache MISS: fetch from originating server (US), cache it, serve it
5. All future requests for that content from this region → served from Mumbai edge server
```

**TTL (Time to Live):** CDN caches have an expiry. After TTL expires, the next request fetches fresh content from the originating server. This prevents serving stale/outdated content forever.

**Real-world users of CDN:** Netflix, YouTube, Vercel (edge network), Cloudflare, Akamai, AWS CloudFront.

#### DNS Caching
DNS = Domain Name System. It resolves human-readable domain names (`example.com`) into IP addresses (`93.184.216.34`) so browsers know where to connect.

DNS heavily relies on caching because resolving a domain name involves multiple hops:

```
OS local DNS cache
  ↓ (cache miss)
Browser DNS cache
  ↓ (cache miss)
Recursive resolver (ISP / Google / Cloudflare)
  ↓ (cache miss)
Root server
  ↓
TLD server (.com server)
  ↓
Authoritative name server for example.com
  ↓
Returns IP address
```

Without caching: every request goes through all these hops = slow.  
With caching: each layer caches the result so subsequent requests skip hops.

**Caching layers in DNS:**
1. **OS DNS cache** — your laptop/phone checks this first
2. **Browser DNS cache** — Chrome/Firefox maintain their own
3. **Recursive resolver cache** — your ISP's resolver or Google's 8.8.8.8
4. **Authoritative server** — some also cache

---

### Level 2 — Hardware Level Caching (Brief Overview)

CPU architecture has multiple layers of cache to speed up computation:

```
L1 Cache (fastest, smallest, per-CPU core)
L2 Cache (slightly slower, larger)
L3 Cache (shared across cores)
RAM (Main memory)       ← in-memory databases like Redis live here
Secondary Storage (Disk) ← traditional databases like PostgreSQL live here
```

The reason arrays are fast for sequential access: when you start reading an array, the CPU's predictive algorithms pre-load the entire array into L1/L2 cache, so subsequent elements are retrieved at near-instant speed.

---

### Level 3 — Software Level Caching (Most Important for Backend Engineers)

This is where **Redis**, **Memcached**, and **AWS ElastiCache** live.

---

## 4. In-Memory Databases — Why Redis is Fast

### Disk vs RAM: The Core Tradeoff

| | RAM (Primary Memory) | Disk (Secondary Storage) |
|---|---|---|
| **Speed** | Very fast (electrical signals) | Slower (mechanical/electrical) |
| **Cost** | Expensive | Cheap |
| **Capacity** | Small (8–128 GB typical) | Large (512 GB–multiple TB) |
| **Volatile?** | Yes (data lost on power off) | No (data persists) |

### Why RAM is Faster
- RAM = **Random Access Memory** — any data location can be accessed in constant time using direct memory addresses + electrical signals
- Disk (HDD): mechanical head physically moves to the data location — slow
- Disk (SSD): faster than HDD, but still slower than RAM

### How Redis Uses RAM
Redis stores all data in RAM → reads/writes happen at memory speed.

For persistence (so data survives restarts), Redis also writes to disk in the background. On startup, it loads data from disk back into RAM. So you get both speed (RAM) and persistence (disk).

### In-Memory Key-Value Databases

**Redis** (and Memcached) are called:
- **In-memory** — data stored in RAM (not disk)
- **Key-value** — data organised as `key: value` pairs (not tables/rows)
- **NoSQL** — no strict schema like relational databases

```
Traditional DB (PostgreSQL):
  Table: users | rows | columns | strict schema | disk-based | slower

Redis:
  Key: "user:42"  Value: '{"name":"Pravan","role":"admin"}' | RAM | faster
```

---

## 5. Caching Strategies

### Strategy 1 — Lazy Caching (Cache-Aside) [Most Common]

```
Request comes in
    ↓
Check cache: is the data there?
    ↓ Yes (cache HIT)        ↓ No (cache MISS)
Return data from cache       Fetch from DB
                                ↓
                             Store in cache
                                ↓
                             Return data to client
```

**Characteristics:**
- Cache is only populated when data is actually requested
- You don't proactively pre-load the cache
- Cache starts empty — first requests always go to DB (cache misses)
- Once data is in cache, subsequent requests are fast (cache hits)

**Best for:** Read-heavy workloads with unpredictable access patterns.

---

### Strategy 2 — Write-Through Caching

```
Client writes/updates data (POST/PUT/PATCH)
    ↓
Update primary database  AND  Update cache simultaneously
    ↓
Both DB and cache always have the latest data
```

**Characteristics:**
- Cache is always fresh — never stale
- Every write operation is slower (must write to both DB and cache)
- Good when read-after-write consistency is critical

**Best for:** Systems where you can't afford to serve stale data.

---

### Quick Comparison

| | Lazy Caching | Write-Through |
|---|---|---|
| **Write overhead** | Low | High |
| **Cache freshness** | Can be stale | Always fresh |
| **Initial cache hits** | Low (cold start) | High |
| **Complexity** | Simple | More complex |

---

## 6. Cache Eviction Policies

### The Problem
RAM is finite. Redis will eventually run out of memory. When that happens, you have to decide what to remove to make space for new data.

This decision process is called an **eviction policy**.

### Eviction Strategies

#### 1. No Eviction
Don't remove anything. When memory is full, new writes fail with an error. Rarely useful in production.

#### 2. LRU — Least Recently Used
Remove the key that was accessed **least recently** (longest ago).

```
Keys in cache: key1 (accessed today), key2 (accessed today), 
               key3 (accessed today), key4 (accessed yesterday)
New key (key5) arrives, cache is full.

LRU says: key4 was accessed least recently → evict key4 → add key5
```

**Logic:** If you haven't used data recently, you probably won't use it soon.

#### 3. LFU — Least Frequently Used
Remove the key that has been accessed the **least number of times overall**.

```
Keys in cache:
  key1: accessed 5 times
  key2: accessed 10 times
  key3: accessed 6 times
  key4: accessed 23 times
New key (key5) arrives, cache is full.

LFU says: key1 has lowest frequency (5) → evict key1 → add key5
```

**Logic:** Data accessed rarely is less valuable than data accessed frequently.

#### 4. TTL (Time to Live) Based
Each key is given an expiry time. When the TTL expires, the key is automatically deleted. When cache is full, prioritise removing keys closest to expiry.

```javascript
// Setting a key with TTL of 300 seconds (5 minutes)
await redis.set('user:42', JSON.stringify(user), 'EX', 300);
// After 300 seconds, this key is automatically deleted
```

**Logic:** Stale data is less valuable. Let time decide what to evict.

---

## 7. Real-World Use Cases in Backend Engineering

### Use Case 1 — Caching Expensive Database Queries

**Problem:** A complex SQL query with many JOINs and aggregations is being called on the dashboard page, which millions of users visit. The query takes 500ms and hammers the database.

**Solution:**
```javascript
async function getDashboardStats(orgId) {
  const cacheKey = `dashboard:${orgId}`;

  // Step 1: Check cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);   // ← cache HIT, return instantly

  // Step 2: Cache miss — run the expensive query
  const result = await db.query(`
    SELECT ... FROM projects p
    JOIN tasks t ON p.id = t.project_id
    JOIN ... multiple expensive joins ...
    WHERE p.org_id = $1
    GROUP BY ...
  `, [orgId]);

  // Step 3: Store in cache with 1 hour TTL
  await redis.set(cacheKey, JSON.stringify(result), 'EX', 3600);

  return result;
}
```

**Impact:** 500ms DB query → near 0ms cache hit for all subsequent users.

**Real-world examples:** Amazon caches product details, prices, inventory. LinkedIn caches user feed calculations. Dashboard analytics on any SaaS platform.

---

### Use Case 2 — Session Storage

**Problem:** After authentication, a session token is generated and must be verified on every API request. If stored in PostgreSQL → every request requires a DB lookup → latency + DB load.

**Solution:** Store sessions in Redis:
```javascript
// After successful login
const sessionId = crypto.randomUUID();
await redis.set(
  `session:${sessionId}`,
  JSON.stringify({ userId: 42, role: 'admin' }),
  'EX', 3600   // expire after 1 hour
);

// On every subsequent request (auth middleware)
const session = await redis.get(`session:${req.sessionId}`);
if (!session) return res.status(401).json({ error: 'Unauthorized' });
const { userId, role } = JSON.parse(session);
```

**Why Redis and not PostgreSQL?** Every single API request would be a DB query. Redis lookup takes ~0.1ms. PostgreSQL lookup takes ~5-20ms. At scale, this difference compounds massively.

---

### Use Case 3 — External API Response Caching

**Problem:** Your backend calls a weather API for every user request. The weather API has rate limits (1000 calls/day) and billing per request. 10,000 users = 10,000 API calls/day → bill explodes, rate limit hit.

**Solution:**
```javascript
async function getWeather(city) {
  const cacheKey = `weather:${city}`;

  // Check cache first
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);  // ← serve from cache

  // Cache miss — call external API
  const weather = await weatherAPI.get(city);

  // Cache for 1 hour (weather doesn't change every second)
  await redis.set(cacheKey, JSON.stringify(weather), 'EX', 3600);

  return weather;
}
```

**Result:** 10,000 users per hour → maybe 50 actual external API calls (one per city per hour). 99.5% reduction in external API calls.

---

### Use Case 4 — Rate Limiting

**Problem:** Implement a rate limit of 50 requests per minute per IP address. Must check on every single request.

**Why Redis (not PostgreSQL)?**
- Every API request needs to check and update the counter
- PostgreSQL would add 10-20ms of latency per request + DB overload
- Redis check + increment: ~0.1ms

**Implementation:**
```javascript
async function rateLimitMiddleware(req, res, next) {
  const ip = req.headers['x-forwarded-for'] || req.ip;
  const key = `ratelimit:${ip}`;

  const count = await redis.incr(key);  // increment counter (creates if doesn't exist)

  if (count === 1) {
    // First request this minute — set TTL of 60 seconds
    await redis.expire(key, 60);
  }

  if (count > 50) {
    return res.status(429).json({ error: 'Too many requests' });
  }

  next();
}
```

**How it works:**
- First request: `ratelimit:192.168.1.1` = 1, TTL set to 60 seconds
- Next requests: counter increments (2, 3, 4...)
- After 60 seconds: key expires automatically, counter resets
- If count exceeds 50 within a minute: 429 error

**Why Redis is perfect here:** The counter lives in Redis (in-memory, ~0.1ms), expires automatically via TTL, handles thousands of concurrent requests efficiently.

---

## 8. Cache Invalidation — The Hard Problem

> "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton

Cache invalidation = deciding when to remove or update cached data so clients don't get stale/outdated results.

### Strategies

**1. TTL (Time-Based Invalidation)**
Set an expiry on every cache entry. It auto-deletes after X seconds.
```javascript
redis.set('key', value, 'EX', 300);  // expires in 5 minutes
```
Simple. Predictable. Works when you can tolerate slightly stale data.

**2. Manual Invalidation (Event-Based)**
When data changes, explicitly delete the cached version:
```javascript
// When user updates their profile
async function updateUserProfile(userId, data) {
  await db.update('user_profiles', data, userId);
  await redis.del(`user:${userId}`);  // ← invalidate cache
  // Next request will re-populate from DB
}
```
Keeps cache fresh. More code to write. Easy to forget.

**3. Write-Through (Always Fresh)**
Update cache and DB simultaneously on every write. Cache never becomes stale.
More write overhead but simplest to reason about.

---

## 9. What NOT to Cache

Not everything should be cached:

| Don't cache | Why |
|---|---|
| **Financial/payment data** | Must always be real-time accurate |
| **Inventory counts during flash sales** | Changes every millisecond |
| **Personal sensitive data without scoping** | Privacy risk — never serve User A's cached data to User B |
| **Data that changes every second** | Cache invalidation overhead > benefit |

Cache only when:
- Data is read much more than it's written
- Computation to fetch/calculate is expensive
- You can tolerate slightly stale data (or you invalidate properly)

---

## Quick Reference — Redis Cheatsheet

```bash
# Set a key (no expiry)
SET user:42 '{"name":"Pravan"}'

# Set with TTL (expires in 300 seconds)
SET user:42 '{"name":"Pravan"}' EX 300

# Get a key
GET user:42

# Delete a key
DEL user:42

# Check TTL remaining
TTL user:42

# Increment a counter
INCR ratelimit:192.168.1.1

# Set expiry on existing key
EXPIRE ratelimit:192.168.1.1 60

# Check if key exists
EXISTS user:42

# List all keys matching pattern (caution in production)
KEYS user:*
```

---

## Summary

| Concept | Key Point |
|---|---|
| **What is caching** | Subset of data stored in faster-access location to reduce time and effort |
| **When to use it** | Heavy computation OR large data repeated frequently |
| **CDN** | Cache content at geographically close edge servers to reduce latency |
| **DNS caching** | Multiple layers (OS, browser, resolver) cache IP lookups to skip repeated hops |
| **Hardware cache** | L1/L2/L3 → RAM → Disk. Redis lives in RAM tier. |
| **Why Redis is fast** | Stores data in RAM (not disk). RAM = random access, electrical signals, no mechanical movement. |
| **Cache-Aside** | Check cache → miss → fetch DB → store in cache → return. Lazy population. |
| **Write-Through** | Update DB and cache simultaneously. Always fresh. More write overhead. |
| **LRU** | Evict least recently accessed key when memory is full |
| **LFU** | Evict least frequently accessed key when memory is full |
| **TTL** | Auto-expire keys after a set time. Simple invalidation. |
| **Session storage** | Store auth sessions in Redis for sub-millisecond access vs DB lookup |
| **Query caching** | Cache expensive DB queries. Serve from Redis. Massive performance gain. |
| **API caching** | Cache external API responses. Reduce billing + rate limit exposure. |
| **Rate limiting** | Store per-IP counters in Redis with TTL. Fast check, no DB needed. |

---

## One-Line Takeaways

> Cache = fast access copy of expensive-to-fetch data. It's a subset, not a replacement for your database.

> Redis is fast because it stores data in RAM (random access memory), not on disk. RAM access is near-instant.

> The hardest part of caching isn't setting it up — it's knowing when to invalidate it.

> The two cache strategies: lazy (populate on first request) and write-through (always keep fresh on writes). Use lazy for most cases.

---

*Next lecture → Task Queuing and Scheduling — How to move slow operations out of the request cycle and into the background.*
