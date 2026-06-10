# ⚡ Backend Scaling & Performance Engineering — Complete In-Depth Notes
### Lecture 21 — Part 2

> **Continuation of Part 1.** This part covers: Statelessness, Load Balancers, Database Scaling, CDNs, Edge Computing, Async Processing, Microservices, and Serverless.

---

## 📚 Table of Contents

1. [Statelessness — The Key to Horizontal Scaling](#1-statelessness--the-key-to-horizontal-scaling)
2. [Load Balancers](#2-load-balancers)
   - [Load Balancing Algorithms](#21-load-balancing-algorithms)
   - [Health Checks](#22-health-checks)
3. [Database Scaling](#3-database-scaling)
   - [Read Replicas](#31-read-replicas)
   - [Sharding (Partitioning)](#32-sharding-partitioning)
   - [Distributed Databases](#33-distributed-databases-modern-approach)
4. [CDN — Content Delivery Networks](#4-cdn--content-delivery-networks)
   - [Why CDNs Exist — The Physics Problem](#41-why-cdns-exist--the-physics-problem)
   - [What to Cache in a CDN](#42-what-to-cache-in-a-cdn)
   - [CDN for Security — DDoS Protection](#43-cdn-for-security--ddos-protection)
5. [Edge Computing](#5-edge-computing)
6. [Asynchronous Processing](#6-asynchronous-processing)
7. [Microservices](#7-microservices)
   - [Monolith vs Microservices](#71-monolith-vs-microservices)
   - [When to Use Microservices](#72-when-to-use-microservices)
8. [Serverless Computing](#8-serverless-computing)
   - [The Traditional Server Model](#81-the-traditional-server-model)
   - [How Serverless Works](#82-how-serverless-works)
   - [Cold Starts](#83-the-cold-start-problem)
   - [When to Use Serverless](#84-when-to-use-serverless)
9. [Final Mental Models — The Big Takeaways](#9-final-mental-models--the-big-takeaways)
10. [Complete Quick Reference Cheat Sheet](#10-complete-quick-reference-cheat-sheet)

---

## 1. Statelessness — The Key to Horizontal Scaling

> **Statelessness is the single most important property that makes horizontal scaling possible.**

### What Does Stateless Mean?

**Stateful server** = a server that holds information exclusive to itself (in its own memory):
```
Instance A: { sessions: [user123_session, user456_session] }  ← only A knows this
Instance B: { sessions: [] }
Instance C: { sessions: [] }
```

**Stateless server** = a server that holds zero information. Any instance can handle any request and produce the same result:
```
Instance A, B, C: {}  ← All empty. All data lives OUTSIDE in shared storage.
```

### Why Stateful Breaks Horizontal Scaling

**Real scenario — Session problem:**

```
Step 1: User logs in → Request goes to Instance A
        Instance A creates session, stores it in A's own memory

Step 2: User makes another request → Load balancer sends it to Instance B
        Instance B checks for session → NOT FOUND → Throws 401 Unauthorized ❌

User is confused: "But I just logged in!"
```

**The fix — Externalize ALL state:**

```
❌ Wrong — storing session in server memory:
Instance A stores → { sessionId: "abc123", userId: 5 }  (local memory)

✅ Correct — storing session in external Redis:
Instance A stores → Redis.set("session:abc123", { userId: 5 })
Instance B reads → Redis.get("session:abc123") → Found! ✅
```

### Everything Must Be Externalized

| Data Type | ❌ Wrong (Local) | ✅ Correct (External) |
|-----------|----------------|----------------------|
| Sessions | Server memory | Redis / Memcached |
| Uploaded files | Server's local disk | S3 / Cloudflare R2 / Azure Blob |
| Database | SQLite file on server | PostgreSQL / MySQL RDS |
| Cache | In-process Map/Dict | Redis |
| Logs | Local log file | CloudWatch / Datadog / Elasticsearch |

> **Rule: In a horizontally scaled architecture, if ANY data lives inside a specific server instance and is not accessible by all other instances — your architecture is broken.**

### The Mental Check

Every time you write code in a horizontally scaled system, ask:
> *"If this server instance died right now and a different instance handled the next request, would anything break?"*

If the answer is yes → you have stateful code that needs to be externalized.

---

## 2. Load Balancers

> **A Load Balancer is the traffic cop of horizontal scaling — it sits in front of all your server instances and decides which instance gets each request.**

### Why Load Balancers Are Mandatory

Without a load balancer, you cannot horizontally scale. You'd have multiple servers but no way to distribute traffic to them.

```
Without LB:                    With LB:
                               
Users → ? → Server 1          Users → Load Balancer → Server 1
       ? → Server 2                                  → Server 2
       ? → Server 3                                  → Server 3
(How does the user know                (LB decides automatically)
 which server to talk to?)
```

### How It Works

```
Step 1: All user requests go to the Load Balancer (single entry point)
Step 2: LB applies its algorithm to pick a server instance
Step 3: LB forwards the request to the chosen server
Step 4: Server processes and sends response back to LB
Step 5: LB forwards the response back to the user
```

---

### 2.1 Load Balancing Algorithms

#### Algorithm 1: Round Robin

Sends requests in a rotating, sequential order:

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  ← cycle repeats
Request 5 → Server B
...
```

**Best for:** All servers have the same capacity AND all requests are roughly the same cost (similar processing time).

**Problem:** Does not consider request complexity. A heavy 2-second request and a light 50ms request are treated identically.

```
LB sends requests blindly in order:
Request (heavy, 2s) → Server A  ← A is now busy for 2 seconds
Request (heavy, 2s) → Server A  ← A gets another! Now overloaded ❌
Request (light, 50ms) → Server B
Request (heavy, 2s) → Server A  ← A might crash now
```

---

#### Algorithm 2: Weighted Round Robin

Same as Round Robin, but servers with more capacity get more requests proportionally:

```
Server A: 8GB RAM, 4 cores   → weight = 2 (gets 2x requests)
Server B: 4GB RAM, 2 cores   → weight = 1
Server C: 4GB RAM, 2 cores   → weight = 1

Distribution:
Request 1 → Server A
Request 2 → Server A  (A gets 2 in a row because weight=2)
Request 3 → Server B
Request 4 → Server C
Request 5 → Server A
Request 6 → Server A
...
```

**Best for:** Heterogeneous servers (different capacities in the same pool).

---

#### Algorithm 3: Least Connections ✅ Smarter

Instead of rotating blindly, the LB checks which server currently has the **fewest active connections** and sends the request there.

```
Why "active connections" matters:
- A heavy request (2s to process) = connection stays open for 2 seconds
- A light request (50ms to process) = connection closes in 50ms
```

**Example:**

```
Time 0: All servers idle (0 connections each)

Request 1 (heavy, 2s) → Server A  (A: 1 active connection)
Request 2 (light, 50ms) → Server B  (B: 1 active connection)
Request 3 (light, 50ms) → Server C  (C: 1 active connection)

Time 0.05s: B and C finished their light requests (0 connections)
            A still processing the heavy request (1 connection)

Request 4 → Server B  ← LB picks B because it has 0 connections
Request 5 → Server C  ← LB picks C because it has 0 connections
(Server A is busy, LB avoids sending more to it) ✅
```

**Best for:** Mixed workloads where request processing time varies significantly.

---

#### Algorithm 4: Weighted Least Connections

Same as Least Connections but servers with more capacity can hold proportionally more connections before being considered "busy."

---

#### Other Algorithms (Brief)

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Least Response Time** | Track which server replies fastest, send more there | Performance-sensitive apps |
| **Resource Based** | Check CPU/RAM usage of each server, route to least loaded | CPU-heavy workloads |
| **IP Hash** | Route the same user IP to the same server always | Session-based apps (less common now) |

---

### 2.2 Health Checks

> **How does the Load Balancer know when a server has crashed?**

Without health checks, a dead server would keep getting requests — causing errors for users.

**How health checks work:**

```
Every second, Load Balancer sends a test request to EVERY server:
GET /health  → Server A: 200 OK ✅
GET /health  → Server B: 200 OK ✅
GET /health  → Server C: 200 OK ✅

Server A crashes...

GET /health  → Server A: No response / 502 Error ❌
              LB immediately blacklists Server A
              All traffic now goes only to B and C

Server A comes back online...

GET /health  → Server A: 200 OK ✅
              LB removes A from blacklist
              Traffic resumes to A, B, and C
```

**Key points about health checks:**
- Test requests are extremely lightweight (just a simple endpoint like `GET /health` returning 200)
- They do not cause significant load on servers
- The moment a server stops responding — it is immediately removed from rotation
- The moment it recovers — it is automatically added back

```javascript
// Your server needs this simple health check endpoint:
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', timestamp: Date.now() });
});
```

---

## 3. Database Scaling

> **Scaling your application servers is relatively easy once you externalize state. The hard part is scaling your database — because databases ARE state.**

Unlike application servers (which are stateless), databases:
- Hold all your data persistently on disk
- Must maintain data consistency across all instances
- Cannot simply be duplicated without coordination

Two primary database scaling techniques: **Read Replicas** and **Sharding**.

---

### 3.1 Read Replicas

#### The Problem

All your application servers now talk to ONE database. As traffic grows, this single database becomes the bottleneck.

```
10 App Servers → All querying → 1 Database = Bottleneck ❌
```

#### The Insight

In most SaaS applications, **70–90% of database operations are READ operations** (SELECT queries):
- Fetching user profile
- Loading blog posts
- Getting product listings
- Showing dashboard data

Only 10–30% are WRITE operations (INSERT, UPDATE, DELETE).

#### How Read Replicas Work

```
                    WRITE operations (INSERT, UPDATE, DELETE)
                    ↓
All App Servers → Primary DB (Master)
                    ↓ Replication (automatic, continuous)
                 ↓             ↓             ↓
           Read Replica 1  Read Replica 2  Read Replica 3
           (India region)  (Japan region)  (EU region)
                    ↑             ↑             ↑
                    READ operations (SELECT) from nearby servers
```

**Rules:**
- **Primary DB:** Handles ALL write operations
- **Read Replicas:** Handle ALL read operations — they are read-only copies
- **Replication:** The primary automatically syncs all changes to replicas (continuously)

**Benefits:**
- Primary DB load drops from 100% → only 30% (only writes)
- Users in different regions get data from a nearby replica → lower latency
- If the primary goes down, a replica can be promoted to primary (failover)

#### The Big Problem: Replication Lag

> **You cannot beat physics.** Data takes time to travel from primary to replica.

**Scenario:**
```
User (India) → Updates name: "Alice" → "Alicia"
             → Write goes to Primary DB in USA
             
Primary DB: name = "Alicia" ✅ (updated)
India Read Replica: name = "Alice" ❌ (not yet synced)

Replication Lag = ~200ms (physical distance USA → India)

User immediately refreshes their profile page:
→ Read request goes to India replica
→ Replica returns: name = "Alice" ← STALE DATA! ❌
→ User sees their name reverted — very confusing
```

This is called **replication lag** — the window of inconsistency between primary and replicas.

#### Solutions for Replication Lag

**Solution 1: Route reads after writes to the primary**
```javascript
async function updateUserName(userId, newName) {
  await primaryDB.update('users').set({ name: newName }).where({ id: userId });
  
  // For the immediate read-after-write, use primary (not replica)
  const updatedUser = await primaryDB.findOne('users', { id: userId });
  return updatedUser;
  // All subsequent reads can use replicas again
}
```

**Solution 2: Track and wait for replication lag**
```javascript
// Before reading from replica, check if replication is complete
const replicationLag = await getReplicationLag(); // e.g., 200ms
if (recentWriteHappened && timeSinceWrite < replicationLag) {
  return primaryDB.query(sql); // Use primary
} else {
  return replicaDB.query(sql); // Safe to use replica
}
```

**Solution 3: Frontend delay**
```javascript
// After a successful update response, wait before re-fetching
async function onUpdateSuccess() {
  showSuccessToast("Profile updated!");
  await sleep(300); // Wait for replication lag
  await fetchUserProfile(); // Now fetch fresh data
}
```

> **The key insight:** Replication lag is a fundamental trade-off of read replicas. You trade perfect consistency for better performance and lower latency. Modern managed databases (RDS, Neon, PlanetScale) handle most of this for you automatically.

---

### 3.2 Sharding (Partitioning)

> **Sharding = physically splitting one massive table into multiple database instances.**

#### When Sharding Becomes Necessary

Imagine an e-commerce platform with billions of orders:

```sql
-- This query on a table with 100 billion rows = very slow, even with indexes
SELECT * FROM orders WHERE user_id = 5;
-- Takes several seconds ❌
```

Two problems:
1. **Query latency** — even indexed queries are slow on billions of rows
2. **Single instance capacity** — one DB server can only hold so much data and handle so many connections

#### How Sharding Works

Split the table across multiple physical database instances based on a **sharding key**:

```
Before sharding:
┌─────────────────────────────────┐
│ orders table (100 billion rows) │  ← One DB instance, too slow
│ Jan, Feb, Mar, Apr, May, Jun... │
└─────────────────────────────────┘

After sharding by date range:
┌─────────────────────┐    ┌─────────────────────┐
│ orders_shard_1      │    │ orders_shard_2      │
│ Jan–Jun (50B rows)  │    │ Jul–Dec (50B rows)  │
│ (DB Instance 1)     │    │ (DB Instance 2)     │
└─────────────────────┘    └─────────────────────┘
```

Each shard has half the data → queries are 2x faster → each instance handles half the connections.

#### Choosing the Sharding Key

The sharding key determines how data is divided. It's one of the most important (and tricky) decisions:

| Sharding Key | Example | Pros | Cons |
|-------------|---------|------|------|
| **Date range** | Jan-Jun vs Jul-Dec | Easy to understand | Uneven if recent data is accessed more |
| **User ID range** | Users 1-500k vs 500k-1M | Even distribution | Complex cross-user queries |
| **Geographic region** | US users vs EU users | Latency benefits | Uneven if regions grow differently |
| **Hash of ID** | hash(userId) % N shards | Very even distribution | Hard to add new shards later |

#### Routing to the Right Shard

Your application needs logic to determine which shard to query:

```javascript
function getShardForQuery(orderId, orderDate) {
  // Route based on date range
  const month = new Date(orderDate).getMonth();
  if (month < 6) {
    return shard1DB; // Jan-Jun
  } else {
    return shard2DB; // Jul-Dec
  }
}

// Usage
const shard = getShardForQuery(orderId, orderDate);
const order = await shard.findOne('orders', { id: orderId });
```

#### Sharding Challenges

| Challenge | Description |
|-----------|-------------|
| **Cross-shard queries** | `SELECT * FROM orders WHERE user_id IN (5, 6, 7)` — what if user 5 is in shard 1 and user 6 is in shard 2? |
| **Rebalancing** | When one shard fills up, moving data to a new shard is complex |
| **Transactions** | A transaction across two shards (distributed transaction) is very complex |
| **Schema changes** | Must apply to ALL shards simultaneously |

---

### 3.3 Distributed Databases (Modern Approach)

Modern managed database services handle sharding, replication, and failover automatically:

| Service | Based On | Notable Feature |
|---------|----------|----------------|
| **PlanetScale** | MySQL (Vitess) | Built-in horizontal sharding |
| **Neon** | PostgreSQL (Rust) | Serverless Postgres, auto-scaling |
| **CockroachDB** | Distributed SQL | Global consistency, auto-sharding |
| **Yugabyte** | PostgreSQL | Distributed ACID transactions |
| **Amazon RDS** | MySQL/Postgres | Managed replication, auto-backups |

> **Practical advice:** Unless you are a database expert, never manage your own database infrastructure. Use a managed database provider. You configure WHAT you want (read replicas in these regions, sharding strategy) — they handle HOW to implement it.

---

## 4. CDN — Content Delivery Networks

> **CDN = a globally distributed network of servers that caches your content close to your users.**

---

### 4.1 Why CDNs Exist — The Physics Problem

**Light travels at ~200,000 km/second through fiber optic cables.** This is the speed of the internet. You cannot go faster.

**Tokyo user → Server in Virginia (US East):**
```
Round trip distance: ~20,000 km
Speed through fiber: ~200,000 km/sec
Minimum round-trip time: 20,000 / 200,000 = 0.1 seconds = 100ms
```

**This 100ms is a hard physics cap.** No optimization in the world can beat it for this user.

And 100ms is just the network travel time. Add:
- Request deserialization (JSON parsing): ~10ms
- Business logic execution: ~5-20ms
- Database query: ~50-100ms
- External API calls: ~200ms
- Response serialization: ~10ms

**Total: ~400-500ms minimum for a Tokyo user hitting a US server.**

**CDN Solution:**

```
Without CDN:
Tokyo User → (20,000 km) → US Server
Latency: ~100ms (just network)

With CDN:
Tokyo User → (50 km) → CDN Node in Tokyo → Cached Response
Latency: ~2-3ms ✅ (50x improvement)
```

CDN nodes are strategically placed **at the edge of the network** — as close to users as possible, in collaboration with Internet Service Providers (ISPs).

---

### 4.2 What to Cache in a CDN

**Static Content (Best candidates):**

| Content | Why CDN-Friendly |
|---------|-----------------|
| JavaScript bundles | Generated at build time, same for all users |
| CSS files | Same for all users, rarely changes |
| HTML files | For SPAs and static sites |
| Images, Videos | Large files, same for all users, rarely changes |
| Fonts | Never changes |

```
Traditional SPA deployment:
User requests your app → Your primary server in US → Serves JS/CSS/HTML

CDN-optimized deployment:
User requests your app → CDN node in their region → Serves JS/CSS/HTML (cached)
Primary server never touched → Much faster + much lower server load ✅
```

**Dynamic Content (Some API responses):**

Not all API responses change frequently. Some are great CDN candidates:

```javascript
// Product catalog — doesn't change every second
// Cache this in CDN for 1 hour ✅
GET /api/products?category=electronics
Cache-Control: public, max-age=3600

// User profile — changes per user, don't cache in CDN
GET /api/users/me
Cache-Control: private, no-cache
```

**CDN Cache Invalidation (Purging):**

When content changes, you need to tell the CDN to delete old cached content:

```javascript
// Using Cloudflare API to purge cache when a blog post is updated
await cloudflare.zones.purgeCache({
  tags: [`blog-post-${postId}`, `user-${authorId}-posts`]
});
// CDN will fetch fresh content from your server on the next request
```

---

### 4.3 CDN for Security — DDoS Protection

**DDoS (Distributed Denial of Service):** An attacker controls thousands of bots worldwide and simultaneously floods your server with traffic until it crashes.

```
Without CDN:
10,000 bots → Your server (crashes or incurs massive cost) ❌

With CDN (Cloudflare):
10,000 bots → Cloudflare CDN (distributed across thousands of nodes globally)
             → Cloudflare detects the attack pattern
             → Cloudflare shows CAPTCHA to suspicious traffic
             → Only legitimate users get through to your server ✅
```

**Why Cloudflare can absorb DDoS attacks:** Their CDN network is enormous (handles petabytes of traffic). An attack that would destroy your server is barely a blip across their global network.

---

## 5. Edge Computing

> **Edge Computing = running actual code/logic at the CDN node level, not just serving static files.**

### Traditional CDN vs Edge Computing

```
Traditional CDN:
User → CDN node → "Do you have file X cached?" 
               → Yes → Send file ✅
               → No  → Fetch from origin server

Edge Computing:
User → CDN node → Run code/logic here
               → Check authentication
               → Route based on user location
               → Return processed response
(All without touching your primary server)
```

### Real Use Cases for Edge Computing

**Use Case 1: Authentication at the Edge**

```
Without edge auth:
Tokyo user → (100ms to US) → Server checks auth → (100ms back) = 200ms for a 401 error ❌

With edge auth:
Tokyo user → (2ms to Tokyo edge) → Edge checks auth → 401 response = 2ms ✅
Primary server never receives unauthorized requests → Saves bandwidth and compute
```

**Use Case 2: Geographic Personalization**

```javascript
// Cloudflare Worker example (runs at edge node)
addEventListener('fetch', event => {
  const country = event.request.cf.country; // "JP" for Japan
  const language = event.request.headers.get('Accept-Language');
  
  if (country === 'JP') {
    // Serve Japanese version of the site immediately from edge
    return serveLocalizedContent('ja');
  }
  // Otherwise, forward to origin server
  return fetch(event.request);
});
```

**Other edge use cases:**
- A/B testing (split users into groups at the edge)
- Input validation (reject malformed requests before they hit your server)
- Rate limiting at the edge
- Request routing (send API requests to different microservices)

### Constraints of Edge Computing

Edge nodes are ISP infrastructure — they are not full data centers:

| Resource | Primary Data Center | Edge Node |
|----------|--------------------|-----------| 
| RAM | 64GB+ | ~128MB–1GB |
| CPU | 32+ cores | 1 core equivalent |
| Storage | Terabytes | Very limited |
| Network protocols | All | Limited (no raw TCP) |
| File system access | Full | None (Cloudflare Workers) |
| Max execution time | Unlimited | 50ms–30 seconds |

**Popular edge computing platforms:**
- **Cloudflare Workers** — uses V8 JavaScript isolates, ~0-5ms cold start
- **Vercel Edge Functions** — based on Cloudflare Workers
- **AWS Lambda@Edge** — runs at CloudFront CDN nodes

> **Rule: Use edge computing for lightweight, latency-sensitive logic (auth checks, routing, personalization). Do NOT try to run your entire application at the edge — resource constraints make this impractical.**

---

## 6. Asynchronous Processing

> **Async processing = "Don't make the user wait for things they don't need to see immediately."**

### The Core Idea

Every operation in your backend falls into one of two categories:

| Type | User needs result instantly? | Example |
|------|---------------------------|---------|
| **Synchronous** | YES | Updating profile name, making a payment, logging in |
| **Asynchronous** | NO | Sending email, processing video, deleting account data |

For async operations: **respond immediately to the user, do the work in the background.**

### Example — Inviting a Team Member

**❌ Synchronous approach (slow):**

```
User clicks "Invite user1@gmail.com"
       │
       ▼
Server: Validate request (~20ms)
       │
       ▼
Server: Save invite to DB (~50ms)
       │
       ▼
Server: Call email API (Resend/SendGrid) (~300ms) ← user waiting...
       │
       ▼
Server: Get success response from email API
       │
       ▼
Server: Send 200 response to user

Total wait: ~400ms ❌
```

**✅ Asynchronous approach (fast):**

```
User clicks "Invite user1@gmail.com"
       │
       ▼
Server: Validate request (~20ms)
       │
       ▼
Server: Save invite to DB (~50ms)
       │
       ▼
Server: Push task to Queue (2ms): { type: "send_email", to: "user1@gmail.com" }
       │
       ▼
Server: IMMEDIATELY send 200 response to user ✅

Total wait: ~72ms ✅

Meanwhile (background, user doesn't wait):
Queue Worker → Picks up task → Calls email API (300ms) → Done silently
```

### How to Implement — Message Queues

```
┌─────────────┐    push task    ┌─────────────┐    consume task    ┌─────────────┐
│   Server    │ ──────────────► │    Queue    │ ─────────────────► │   Worker    │
│ (Producer)  │                 │  (Redis/    │                    │ (Consumer)  │
│             │                 │  RabbitMQ)  │                    │             │
└─────────────┘                 └─────────────┘                    └─────────────┘
     ↑                                                                   │
     │                                                                   │
     └─────────────────────────────────────────────────────────────── Does the work
                                                              (email, resize image, etc.)
```

**Popular Queue Technologies:**

| Technology | Best For |
|-----------|---------|
| **Redis Queue (BullMQ)** | Simple background jobs, Node.js apps |
| **RabbitMQ** | Complex routing, multiple consumers |
| **Apache Kafka** | High-throughput event streaming, millions of events/sec |
| **AWS SQS** | Managed queue, AWS ecosystem |

**Node.js example with BullMQ:**

```javascript
// Producer — in your API handler
import { Queue } from 'bullmq';
const emailQueue = new Queue('emails', { connection: redisConnection });

async function inviteUser(req, res) {
  const { email } = req.body;
  
  // 1. Save to DB (fast)
  await db.insert('invites', { email, status: 'pending' });
  
  // 2. Push email task to queue (very fast, ~2ms)
  await emailQueue.add('send_invite', {
    to: email,
    templateId: 'team_invite',
    data: { workspaceName: req.user.workspace }
  });
  
  // 3. Respond immediately — don't wait for email to send
  res.status(200).json({ message: "Invitation sent!" });
}

// Consumer / Worker — separate process (can be scaled independently)
import { Worker } from 'bullmq';
const worker = new Worker('emails', async (job) => {
  await resend.emails.send({
    to: job.data.to,
    template: job.data.templateId,
    data: job.data.data
  });
}, { connection: redisConnection });
```

### Perfect Candidates for Async Processing

| Operation | Why Async? |
|-----------|-----------|
| **Sending emails** | User doesn't need to see email being sent in real-time |
| **Push notifications** | User doesn't expect instant notification dispatch |
| **Video processing** | Encoding/transcoding takes minutes — user can't wait |
| **Image resizing** | Processing takes time, user just needs upload confirmation |
| **Account deletion** | Deleting millions of rows from 8 tables — user just needs logout |
| **Generating reports** | Complex DB queries for CSV/PDF exports |
| **Webhook deliveries** | Retry logic handled in background |
| **Search index updates** | Elasticsearch indexing after DB changes |

### Workers Can Scale Horizontally Too

```
Traffic spike → More emails/videos to process
             → Add more worker instances
             → They all pull from the same queue
             → Queue empties faster ✅

Low traffic  → Remove worker instances
             → Save money ✅
```

> **Rule: Any operation where the user does not need to see the result immediately is a candidate for async processing. Identify these early — it is one of the easiest and highest-impact performance improvements.**

---

## 7. Microservices

### 7.1 Monolith vs Microservices

**Monolith:** All functionality in one codebase, deployed as one unit.

```
my-app/
├── auth/
│   ├── login.js
│   └── register.js
├── orders/
│   ├── create.js
│   └── list.js
├── payments/
│   └── charge.js
├── notifications/
│   └── send.js
└── index.js  ← Everything runs as ONE process
```

**Microservices:** Each module becomes an independent service with its own codebase, deployment, and database.

```
auth-service/    (own repo, own DB, own deployment)
order-service/   (own repo, own DB, own deployment)
payment-service/ (own repo, own DB, own deployment)
notification-service/ (own repo, own DB, own deployment)
```

### Monolith Advantages

| Advantage | Explanation |
|-----------|-------------|
| **Simple development** | One codebase, one repo, one IDE |
| **Easy to test** | Everything runs in one process — integration tests are straightforward |
| **Easy to deploy** | One deployment = whole app updated |
| **Easy to refactor** | Change a function → see its effect immediately in the same codebase |
| **Low latency** | Function calls between modules — no network overhead |

### Why Microservices Exist — 3 Real Problems of Monoliths

> **Important: Microservices are primarily about scaling TEAMS, not machines.**

**Problem 1: Deployment Dependency**

```
Scenario: 500 developers on the same monolith
- Payments team: Critical bug fix, ready to deploy NOW
- Notifications team: Experimental feature on main branch, NOT ready
- Monolith: Both teams' code is mixed → Payments can't deploy without Notifications

With microservices:
- Payment service: Deploy independently ✅ 
- Notification service: Not affected ✅
```

**Problem 2: Independent Scaling**

```
Monolith: Traffic spike on payment processing
→ Must scale the ENTIRE monolith (authentication, notifications, orders, everything)
→ Wasteful — notifications don't need scaling

Microservices: Traffic spike on payment processing
→ Scale ONLY the payment service ✅
→ Auth, notifications, orders unchanged
→ Cheaper and more targeted
```

**Problem 3: Technology Stack Flexibility**

```
Monolith: Must use ONE language and framework for everything

Microservices:
- Notification service: Node.js (great npm ecosystem for email/push libs)
- Image processing service: Go/Rust (CPU-bound, needs raw performance → 50ms vs 500ms)
- ML inference service: Python (best ML libraries)
→ Each service uses the right tool for the job ✅
```

### Microservices Disadvantages

| Problem | Explanation |
|---------|-------------|
| **Network overhead** | What was a function call (nanoseconds) is now an HTTP/gRPC call (milliseconds) |
| **Failure handling** | Network calls can fail → need retry logic, timeouts, circuit breakers |
| **Distributed debugging** | Trace a bug across 5 services' logs — much harder than one log file |
| **Data consistency** | Each service has its own DB → replication lag and distributed transactions |
| **Operational complexity** | 10 services = 10 deployments, 10 monitoring setups, 10 health checks |
| **Local development** | Running 10 services locally is painful |

---

### 7.2 When to Use Microservices

**Use microservices only when ALL of these are true:**

```
✅ Large team (100+ developers) — clear team boundaries needed
✅ Different services have very different scaling needs
✅ Different parts need different tech stacks
✅ You have DevOps expertise to manage the infrastructure
✅ You have distributed tracing set up
```

**Do NOT use microservices when:**

```
❌ Small team (< 50 developers)
❌ Early stage — requirements changing constantly
❌ No clear service boundaries yet
❌ No DevOps/infrastructure expertise
❌ "Because Netflix does it" is your only reason
```

> **The best approach:** Start with a well-structured monolith. When you hit actual scaling limits or team size justifies it, extract services one at a time. This is called the **Modular Monolith** pattern — structured internally like microservices but deployed as one unit until you're ready to split.

---

## 8. Serverless Computing

### 8.1 The Traditional Server Model

Traditional deployment:
1. Provision a VM from AWS/GCP/DigitalOcean
2. Install operating system (Ubuntu)
3. Configure web server (Nginx, etc.)
4. Deploy your application
5. **Pay 24/7** regardless of traffic

**Problems with this model:**

**Problem 1: Capacity Planning is guesswork**
```
Underprovision:
You chose 4GB RAM. Traffic spike happens.
Server crashes. Users leave. Revenue lost. ❌

Overprovision:
You chose 32GB RAM. Traffic is normal.
You're using 20% of capacity.
You paid for 32GB but needed 8GB → 4x wasted money. ❌
```

**Problem 2: You always pay even at zero traffic**
```
Monday 3 AM: 0 requests per minute
Your server: Still running, still consuming 4GB RAM, still costing money
You pay: Full server cost 24/7 regardless
```

**Partial Solution: Autoscaling**

Autoscaling automatically adds/removes servers based on load:

```
Normal traffic:  2 servers running
Traffic spike:   Autoscaler detects CPU > 70% → Spins up 3 more servers
After spike:     Traffic drops → Autoscaler removes extra servers
```

**Autoscaling limitations:**
- **Boot time:** Starting a new server takes 2-5 minutes (boot OS + start app) — during a sudden spike, this is too slow
- **Still has a minimum:** You always pay for at least 1-2 servers (the minimum)
- **Reactive, not proactive:** By the time it detects high load and spins up servers, users are already experiencing slowness
- **Cost risk:** If max instances set too high, a traffic spike (or attack) can cost $100,000 in one day

---

### 8.2 How Serverless Works

> **Serverless = you only provide code (functions). The provider manages all the machines.**

```
Traditional server:
You manage: OS, runtime, dependencies, scaling, load balancing, 24/7 uptime
You pay: 24/7 for the machine regardless of traffic

Serverless:
You manage: ONLY your code (functions)
Provider manages: Everything else
You pay: ONLY for actual execution time (CPU milliseconds used)
```

**The execution model:**

```
No request pending → No machine exists for your code (zero cost ✅)

Request arrives:
       │
       ▼
API Gateway receives HTTP request
       │
       ▼
Serverless provider spins up a container/isolate
(this takes time — the "cold start")
       │
       ▼
Your function code runs
       │
       ▼
Response sent to user
       │
       ▼
Container stays warm for a few seconds (for the next request)
Then shuts down (zero cost again) ✅
```

**Pricing model:**
```
Traditional server: $100/month for 1 server regardless of usage

Serverless (AWS Lambda example):
- First 1 million requests/month: FREE
- Then: $0.20 per million requests
- Plus: $0.0000166667 per GB-second of execution

Example: 10 million requests, each 100ms, 128MB memory
Cost = (9M × $0.20/1M) + (10M × 0.1s × 0.128GB × $0.0000166667/GB-sec)
     = $1.80 + $2.13 = ~$4/month

vs. traditional server: $50-100/month minimum
```

---

### 8.3 The Cold Start Problem

> **Cold start = the time it takes to spin up a fresh serverless container when there's no warm instance available.**

**Why it happens:**
```
Long idle period → Container is shut down to save resources
New request arrives → Must boot a fresh container:
  1. Allocate a VM/microVM
  2. Start the container/isolate
  3. Load your runtime (Node.js, Python, etc.)
  4. Load your code and dependencies
  5. Finally: Execute your function

All of this = Cold Start Time
```

**Cold start times by platform:**

| Platform | Technology | Cold Start |
|----------|-----------|------------|
| AWS Lambda (Node.js) | Firecracker microVMs | 100-500ms |
| AWS Lambda (Java) | Firecracker + JVM | 1-5 seconds |
| Cloudflare Workers | V8 isolates | 0-5ms ✅ |
| Vercel Edge Functions | V8 isolates | 0-5ms ✅ |

**Why Cloudflare Workers are so fast:**
1. **V8 isolates instead of VMs** — Starting a V8 isolate is like running a web page in a browser tab: nearly instant. No OS boot needed.
2. **JavaScript (interpreted language)** — No compilation step. Code just runs.

**Solutions for cold start:**

```javascript
// Solution 1: Keep-warm pings (hack)
// Set up a cron job to ping your function every 60 seconds
// The existing warm container handles the ping → stays alive
// Downside: Costs money 24/7 (partially defeats serverless benefits)

// Solution 2: Use platforms with near-zero cold starts
// → Cloudflare Workers (V8 isolates, <5ms)
// → Choose Node.js or Python over Java for Lambda
```

---

### 8.4 When to Use Serverless

**Good use cases:**

| Use Case | Why Serverless Fits |
|---------|---------------------|
| **Image/video processing** | Triggered by uploads, not constant — pay only when used |
| **API backends with variable traffic** | Scale to zero at night, scale up during business hours |
| **Event-driven pipelines** | Triggered by DB changes, file uploads, queue messages |
| **Scheduled tasks (cron)** | Run once per day/hour — traditional server is wasteful |
| **Webhook handlers** | Receive occasional webhooks — no need for always-on server |

**Bad use cases:**

| Use Case | Why Serverless Doesn't Fit |
|---------|--------------------------|
| **Banking/payment critical paths** | Cold start latency is unacceptable |
| **WebSocket connections** | Serverless can't maintain persistent connections |
| **Long-running tasks (>15min)** | Lambda max execution time is 15 minutes |
| **High database connection needs** | Each serverless invocation opens a new DB connection — exhausts connection pool quickly (use PgBouncer or serverless DB) |
| **Always-on, high-frequency APIs** | At very high RPS, traditional servers become cheaper |

---

## 9. Final Mental Models — The Big Takeaways

After both lectures, here are the 5 principles to carry with you always:

---

### Principle 1: Always Start With Measurement

> **"Never guess which component is the bottleneck. Always measure."**

Before implementing any solution:
```
Step 1: Is your system actually slow? Measure with real metrics (P99 latency)
Step 2: Which component is slow? Use distributed tracing
Step 3: Why is that component slow? Use EXPLAIN ANALYZE, profiler, logs
Step 4: Now implement the targeted fix
Step 5: Measure again to verify improvement
```

**Tools for measuring:**
- **Prometheus + Grafana** — open source metrics and dashboards
- **New Relic / Datadog** — managed observability (easier to set up)
- **Jaeger / Zipkin** — distributed tracing
- **EXPLAIN ANALYZE** — database query analysis

---

### Principle 2: Prefer Simplicity

> **"Complexity has costs. Every component you add can fail, needs monitoring, and someone must understand it."**

```
For performance: Try these in order (simplest first)
1. Fix the actual code bug / N+1 query (free, no new components)
2. Add a database index (5 minutes, no new components)
3. Add caching (Redis — one new component)
4. Vertical scaling (bigger machine — zero code changes)
5. Horizontal scaling (multiple servers — significant complexity)
6. Microservices (multiple codebases — very high complexity)
```

**Only move to the next level when the simpler solution is genuinely insufficient.**

---

### Principle 3: Scale for YOUR Problem, Not for Netflix

> **"Build for the scale you have, with reasonable headroom for growth."**

```
Day 1: You have 0 users
→ A simple monolith on a small VPS is perfect
→ Microservices on day 1 is premature and harmful

10,000 users: Optimize queries and add caching
→ Probably enough for a long time

100,000 users: Maybe horizontal scaling + CDN
→ Measure first to confirm this is actually needed

1,000,000 users: Now consider read replicas, sharding
→ You now have data to make this decision

Engineering blogs from Netflix/Google are written for their scale.
Your application has its own specific bottlenecks. Measure yours.
```

---

### Principle 4: Implement Observability From Day One

> **"You cannot fix what you cannot see. Measure everything from the start."**

The three pillars of observability:

```
Logs    → What happened? (events, errors, info)
Metrics → How much/how fast? (latency, throughput, error rate)
Traces  → Where exactly? (which component took how long per request)
```

Even at 0 users, have these running. When something breaks at 100,000 users, you'll have months of history to diagnose the problem.

---

### Principle 5: Performance Is a Mindset, Not a Feature

> **"You will build systems, watch them struggle, optimize, and learn. That is the job."**

- No tutorial can predict all performance issues your specific app will have
- Your job is not to prevent all problems — it is to **detect and recover from them quickly**
- The real skill is: **measure → diagnose → fix → verify → repeat**

---

## 10. Complete Quick Reference Cheat Sheet

### Scaling Decision Tree

```
System is slow
       │
       ▼
Measure with distributed tracing
       │
       ├── Database is slow?
       │      ├── N+1 queries? → Fix with JOINs / ORM bulk fetch
       │      ├── Missing indexes? → Add indexes (EXPLAIN ANALYZE)
       │      ├── Too many connections? → Add connection pooling (PgBouncer)
       │      ├── Query still slow after indexes? → Add caching (Redis)
       │      ├── Single DB can't handle load? → Add read replicas
       │      └── Table has billions of rows? → Consider sharding
       │
       ├── Application server is slow?
       │      ├── Code logic slow? → Profile (flame graph), optimize code
       │      ├── External API call slow? → Make it async (message queue)
       │      ├── One server not enough? → Horizontal scaling + load balancer
       │      └── Need to reduce distance to users? → CDN + edge computing
       │
       └── Response delivery slow?
              ├── Large payload? → Pagination, compression, GraphQL
              └── Users far from server? → CDN for static content
```

### Key Terms at a Glance

| Term | Simple Definition |
|------|------------------|
| **Latency** | Time for one request to complete |
| **Throughput** | Requests per second your system handles |
| **P99 latency** | The slowest 1% of requests — what your worst-off users experience |
| **Utilization** | % of system capacity in use — keep at 60-80%, never 100% |
| **Bottleneck** | The specific component causing slowness |
| **Statelessness** | Servers hold no exclusive data — required for horizontal scaling |
| **Load Balancer** | Distributes traffic across multiple server instances |
| **Health Check** | LB pings servers every second to detect failures |
| **Read Replica** | Copy of DB that handles only SELECT queries |
| **Replication Lag** | Delay between primary DB update and replica having the new data |
| **Sharding** | Splitting one massive table across multiple physical DB instances |
| **CDN** | Globally distributed cache nodes close to users |
| **Edge Computing** | Running logic at CDN nodes (not just serving static files) |
| **Async Processing** | Do work in background after responding to user immediately |
| **Message Queue** | Buffer that holds tasks for background workers to process |
| **Monolith** | All code in one deployable unit |
| **Microservices** | Independent services, each its own codebase and deployment |
| **Serverless** | Provider manages servers; you pay only for actual execution time |
| **Cold Start** | Time to boot a fresh serverless container when no warm instance exists |
| **Observability** | Logs + Metrics + Traces = ability to understand system behavior |

### Load Balancer Algorithm Selection

| Situation | Best Algorithm |
|-----------|---------------|
| All servers same size, all requests same cost | Round Robin |
| Different server sizes | Weighted Round Robin |
| Mixed lightweight + heavy requests | Least Connections |
| Maximize performance, have monitoring | Least Response Time |

### Caching vs Async Processing

| Problem | Solution |
|---------|---------|
| Reading the same data repeatedly is slow | Cache it (Redis) |
| Writing triggers slow operations (email, processing) | Async queue (BullMQ) |
| Static files served slowly | CDN |
| DB query is slow | Index + Cache |
| User waits too long for non-critical operations | Message queue |

---

*These notes cover Lecture 21 Part 2: Backend Scaling & Performance Engineering. Topics include Statelessness, Load Balancers, Database Scaling (Read Replicas + Sharding), CDNs, Edge Computing, Async Processing, Microservices vs Monoliths, Serverless Computing, and the final engineering mental models.*

*For Part 1 topics (Latency, Percentiles, Throughput, Utilization, Bottlenecks, Profiling, Distributed Tracing, N+1 Queries, Indexes, Connection Pooling, Caching, Vertical vs Horizontal Scaling), see `lecture-21-backend-engineering-notes.md`.*
