# Lecture 14 — Task Queues and Background Jobs
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=r-nQsyguU1Y)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Some operations are too slow or too unreliable to run inside an HTTP request. This lecture covers background jobs — what they are, why they exist, how task queues work internally, the different types of tasks, and the design considerations you need to know when building scalable backends.

---

## 1. What is a Background Task?

A background task is **any piece of code that runs outside of the request-response lifecycle**.

```
Client → Request → Server → Response → Client
                     ↑
                     └── This is the request-response lifecycle
                         Background tasks run OUTSIDE of this
```

Key characteristics:
- Does not need to complete immediately
- Not blocking the client's response
- Can run asynchronously in a separate process
- Does not have to be synchronous

---

## 2. The Problem — Why We Need Background Tasks

### Example: User Signs Up

In a typical signup flow:
1. User enters email, name, password
2. Frontend makes API call to backend
3. Backend validates data, hashes password, creates user in DB
4. Backend must send a verification email to the user

The email sending step involves:
- Constructing an HTML template
- Making an API call to a third-party email provider (Resend, Mailgun, Brevo, etc.)
- Waiting for that provider to respond

### The Problem with Doing It Synchronously

```
Client                        Your Server              Email Provider
  |                               |                          |
  |──── POST /signup ────────────>|                          |
  |                               |── validate, create user  |
  |                               |── call email API ───────>|
  |                               |                          | (might be slow)
  |                               |                          | (might be down)
  |                               |<─ error: service down ───|
  |<── 500 Internal Server Error ─|
```

**Scenario 1:** Email provider is down. If you have no error handling, the entire signup API fails with a 500. The user couldn't sign up because of an unrelated email service failure.

**Scenario 2:** You have error handling, signup succeeds, but you tell the user "we sent you a verification email" — except you never actually did. Bad UX, user can't verify.

**Scenario 3:** Email provider is slow (2-3 seconds response time). Your entire signup API takes 2-3 seconds to respond. Terrible user experience.

---

## 3. The Solution — Offload to a Background Task

```
Client                        Your Server              Task Queue          Email Consumer
  |                               |                          |                    |
  |──── POST /signup ────────────>|                          |                    |
  |                               |── validate, create user  |                    |
  |                               |── push email task ──────>|                    |
  |<── 201 Created ───────────────|                          |── pick up task ───>|
  |  (instant response!)          |                          |                    |── call email API
  |                               |                          |                    |── success/retry
```

**What changes:**
1. Server validates, creates user, generates verification code
2. Packages all email info as JSON → pushes task into queue
3. **Immediately returns 201 to client** — no waiting for email
4. Separately, a background consumer picks up the task and sends the email

**Benefits:**
- ✅ API responds instantly — great UX
- ✅ If email provider is down → task retries automatically with exponential backoff
- ✅ Your main server is not coupled to email provider uptime
- ✅ Email eventually gets delivered without user having to do anything

---

## 4. How Task Queues Work — The Internal Mechanics

### The Three Components

```
[Producer]  ──push task──>  [Queue/Broker]  ──pick task──>  [Consumer/Worker]
(your app)                  (RabbitMQ, Redis,              (separate process)
                             Amazon SQS, etc.)
```

#### Producer
- Your main application code
- Creates a task containing all data the consumer will need
- **Serialises** data to JSON
- **Enqueues** (pushes) the task into the broker
- That's it — its job is done, returns response to client immediately

#### Broker (The Queue)
- Temporarily stores tasks until a consumer is ready
- Technologies: **RabbitMQ**, **Redis Pub/Sub**, **Amazon SQS**, **Bull/BullMQ** (built on Redis)
- Manages task persistence, ordering, and delivery guarantees

#### Consumer / Worker
- Runs in a **separate process** from the main backend
- **Constantly monitors** the queue for new tasks
- **Dequeues** (picks up) tasks when available
- Deserialises JSON back to native format
- Executes the registered handler function
- Sends **acknowledgement** back to queue on success/failure

### Acknowledgement System

```
Consumer picks up task
    ↓
Executes task
    ↓
Task succeeds → sends ACK (acknowledgement) to queue → task removed ✅
    OR
Task fails → sends NACK → queue marks as failed → retry scheduled
    OR
Consumer crashes (no ACK within visibility timeout) → task made available again
```

**Visibility Timeout:** The period during which a task is considered "in progress". If no ACK is received within this window, the queue assumes the consumer crashed/hung and makes the task available for another consumer to pick up. This prevents tasks from being lost.

---

## 5. Retry Mechanism — Exponential Backoff

When a task fails (e.g., email API is down), the queue doesn't just give up. It retries with **exponential backoff**:

```
Task fails (attempt 1) → retry after 1 minute
Task fails (attempt 2) → retry after 2 minutes
Task fails (attempt 3) → retry after 4 minutes
Task fails (attempt 4) → retry after 8 minutes
Task fails (attempt 5) → max retries reached → move to Dead Letter Queue
```

**Why exponential backoff?** Most external service outages last seconds or minutes — not hours. The backoff gives the service time to recover without hammering it with constant retries.

**Dead Letter Queue (DLQ):** After maximum retries, failed tasks are moved here for manual inspection and debugging.

---

## 6. Types of Background Tasks

### Type 1 — One-Off Tasks (Most Common)
Triggered once by a specific event. Execute once and done.

Examples:
- Send verification email after signup
- Send welcome email after email verification
- Send password reset email
- Send push notification when someone messages you
- Process a single uploaded image

### Type 2 — Recurring Tasks (Cron Jobs)
Execute periodically on a defined schedule.

Examples:
- Send daily/weekly/monthly reports to users at midnight
- Delete expired/orphan sessions from DB every month
- Clear old logs or cached data
- Sync data from an external API every hour
- Send "you have pending tasks" reminder emails every morning

```
Cron syntax example:
  "0 0 * * *"     → every day at midnight
  "0 9 * * 1"     → every Monday at 9 AM
  "*/5 * * * *"   → every 5 minutes
```

### Type 3 — Chain Tasks (Parent-Child Dependency)
Tasks that depend on each other. A child task only starts after its parent completes successfully.

**Real example — LMS video upload:**
```
[Video Uploaded to S3]
         ↓
[Task 1: Encode video to multiple resolutions] ← parent
         ↓
    ┌────┴────┐
    ↓         ↓
[Task 2:    [Task 3:
Thumbnail   Generate audio
generation] transcription]
    ↓
[Task 4: Process thumbnails
to multiple resolutions]
```

- Task 2 and Task 3 both depend on Task 1 (encoding must complete first)
- Task 2 and Task 3 can run **concurrently** (they don't depend on each other)
- Task 4 depends on Task 2 (thumbnails must be generated before processing them)

### Type 4 — Batch Tasks
Trigger a large number of tasks at once, or a single task that spawns many sub-tasks.

Examples:
- Delete account: triggers sub-tasks to delete user's projects, assets, profile, etc.
- Send reports to 50,000 users at midnight: creates 50,000 individual "send report" tasks simultaneously
- Bulk email campaign
- Database cleanup affecting millions of rows

**Delete Account Example:**
```
User clicks "Delete Account"
    ↓
API immediately returns 200 "Account deletion in progress"
    ↓
Background task starts:
  → Delete user's projects
  → Delete user's assets (images, files)
  → Delete user's profile
  → Delete orphaned records
  → Send "account deleted" confirmation email
  → Finally: delete the user record
```

Why not do this in the request? Could take 30+ seconds. That would time out or block the response.

---

## 7. Popular Libraries/Frameworks

| Language | Library | Notes |
|---|---|---|
| Node.js | **BullMQ** | Most popular, built on Redis, full-featured |
| Python | **Celery** | The standard, supports Redis/RabbitMQ |
| Go | **Asynq** | Built on Redis, easy to use |
| Any | **Amazon SQS** | Managed, scales globally, no infrastructure |
| Any | **RabbitMQ** | Self-hosted message broker |

---

## 8. Design Considerations at Scale

### 1. Idempotency — Design Tasks to Be Safe to Retry

If a task fails halfway through and is retried from scratch, it should not cause unintended side effects.

**Bad example:**
```
Delete account task:
  Step 1: Delete projects ✅
  Step 2: Delete assets ✅
  Step 3: Delete profile ✅
  Step 4: Call billing API to cancel subscription ❌ (fails)
  
Task retried from scratch:
  Step 1: Delete projects → Error! Projects already deleted!
```

**Good example — use transactions:**
```
Delete account task:
  BEGIN TRANSACTION
    Step 1: Delete projects
    Step 2: Delete assets
    Step 3: Delete profile
    Step 4: Call billing API
  If any step fails → ROLLBACK entire transaction
  
Task retried from scratch:
  BEGIN TRANSACTION
  Step 1: Delete projects ✅ (nothing existed to delete, no error)
  ... continues normally
```

By wrapping operations in a transaction and rolling back on failure, you ensure the task can be safely retried from scratch.

### 2. Robust Error Handling

Everything runs in a separate process with no user watching. You need:
- Try/catch around all external calls
- Detailed error logging (what failed, why, which task ID)
- Proper failure signaling to the queue (NACK, not silent failure)
- Alerts when failure rates spike

```javascript
// BullMQ worker example
worker.process('send-email', async (job) => {
  try {
    await emailService.send(job.data);
    // Queue automatically marks as success
  } catch (error) {
    logger.error('Email task failed', {
      jobId: job.id,
      userId: job.data.userId,
      error: error.message,
    });
    throw error; // Re-throw so queue knows it failed → triggers retry
  }
});
```

### 3. Monitoring
Always have visibility into your queue system:
- Current queue length (how many tasks waiting)
- Processing rate (tasks/second)
- Failure rate
- Average processing time
- Worker health (are consumers up and running?)

Tools: Prometheus + Grafana, BullMQ's built-in dashboard, Flower (for Celery)

**Alert on:**
- Queue length exceeds X (backlog building up)
- Failure rate exceeds Y%
- Worker count drops to 0

### 4. Horizontal Scaling of Consumers

When user base grows → more tasks generated → need more processing power.

Solution: add more consumer nodes. Each consumer independently pulls from the queue.

```
Queue
  ↓
  ├──> Consumer 1 (process A)
  ├──> Consumer 2 (process B)
  └──> Consumer 3 (process C)
```

Design your task system to support this from day one — use a proper broker (Redis/RabbitMQ/SQS) not a simple in-memory solution.

### 5. Rate Limiting for External Services

If your tasks call an external API (email provider, SMS provider, etc.), that service likely has rate limits.

Don't create 50,000 tasks and have all consumers hammer the email API simultaneously. Configure:
- Maximum concurrent workers for a given queue
- Delay between tasks (rate limiting at the consumer level)
- BullMQ: `limiter: { max: 100, duration: 1000 }` (100 tasks per second)

---

## 9. Best Practices

### Keep Tasks Small and Focused
Each task should do **one thing**. Don't combine multiple concerns in a single task.

```
❌ Bad: "process-new-user" task that:
  - Sends welcome email
  - Creates default workspace
  - Triggers onboarding webhook
  - Sends Slack notification to your team
  - Updates analytics

✅ Good: Separate tasks:
  - "send-welcome-email"
  - "create-default-workspace"
  - "trigger-onboarding-webhook"
  Each can be retried independently if one fails
```

### Avoid Long-Running Tasks
If a task takes > 30 seconds, break it into smaller chunks. Long tasks:
- Are harder to retry
- Hold up worker capacity
- Are harder to debug when they fail

### Always Include All Necessary Data in the Task
The consumer runs in a completely separate process with no memory of the original request. Put everything it needs in the task payload.

```javascript
// ❌ Bad — consumer can't access req.user
await queue.add('send-email', {
  emailType: 'verification'
  // Consumer won't know WHO to send to!
});

// ✅ Good — all data included
await queue.add('send-email', {
  emailType: 'verification',
  userId: user.id,
  userEmail: user.email,
  userName: user.fullName,
  verificationToken: token,
  expiresAt: new Date(Date.now() + 15 * 60 * 1000), // 15 min
});
```

### Set Appropriate TTLs and Max Retries
Not all tasks should retry forever. A verification email from yesterday is useless today.

```javascript
await queue.add('send-verification-email', data, {
  attempts: 5,                   // max 5 retries
  backoff: { type: 'exponential', delay: 60000 },  // exponential backoff starting at 1 min
  removeOnComplete: 100,         // keep last 100 completed jobs for debugging
  removeOnFail: 200,             // keep last 200 failed jobs for debugging
});
```

---

## 10. Complete Flow Diagram

```
User signs up on platform
        ↓
POST /api/signup  ←──── request comes in
        ↓
Validate data (email, password, name)
        ↓
Hash password
        ↓
Save user to DB (users table)
        ↓
Generate verification token → save to DB
        ↓
Push to queue: {
  type: "send-verification-email",
  userId: "abc-123",
  userEmail: "user@example.com",
  userName: "Pravan",
  token: "xyz789",
  expiresAt: "2024-09-23T14:45:00Z"
}
        ↓
Return 201 Created to client  ←──── response sent IMMEDIATELY
        ↓ (parallel process)
Consumer picks up task
        ↓
Deserialise JSON → native object
        ↓
Build HTML email template with {name} and {verificationLink}
        ↓
Call Resend/Mailgun API to send email
        ↓
Success → send ACK to queue → task removed
OR
Failure → NACK → retry after 1 min → 2 min → 4 min → 8 min → DLQ
```

---

## Summary

| Concept | Key Point |
|---|---|
| **Background task** | Any code running outside the request-response lifecycle |
| **Why use them** | Avoid blocking APIs on slow/unreliable external services |
| **Producer** | Your app code that creates and enqueues tasks |
| **Broker/Queue** | Temporary storage for tasks (Redis, RabbitMQ, SQS) |
| **Consumer/Worker** | Separate process that picks up and executes tasks |
| **Acknowledgement** | Consumer signals success/failure back to queue |
| **Visibility timeout** | If no ACK within timeout → task re-queued for another consumer |
| **Exponential backoff** | Retry intervals double on each failure (1m → 2m → 4m → 8m) |
| **Dead Letter Queue** | Where tasks go after max retries exceeded |
| **One-off task** | Triggered once by an event (send email) |
| **Recurring task** | On a schedule (daily reports, cleanup jobs) |
| **Chain task** | Parent-child dependency (encode video → generate thumbnail) |
| **Batch task** | Many tasks triggered at once or one task spawning many (delete account) |
| **Idempotency** | Design tasks so re-running from scratch causes no side effects |
| **Keep tasks small** | One concern per task — easier to retry, debug, scale |

---

## Common Background Task Use Cases

| Use Case | Task Type |
|---|---|
| Send verification email | One-off |
| Send password reset email | One-off |
| Process uploaded image (resize, optimise) | One-off |
| Send push notification | One-off |
| Generate and send weekly report | Recurring (cron) |
| Clean up expired sessions | Recurring (cron) |
| Process video (encode → thumbnail → transcription) | Chain |
| Delete user account (cascade delete) | Batch |
| Send reports to 50,000 users at midnight | Batch |

---

## One-Line Takeaways

> Background tasks = work that doesn't need to happen before the HTTP response. Offload everything slow or unreliable to a queue.

> The queue makes your APIs fast and your operations reliable. The consumer does the heavy lifting separately.

> Design every task to be idempotent — assume it will be retried. Don't let a mid-task failure leave your data in a broken state.

---

*Next lecture → Elasticsearch — Full-text search and how it works under the hood.*
