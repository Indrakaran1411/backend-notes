# Lecture 19 — Graceful Shutdown
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=6rfBgphiCWM)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Imagine a customer is mid-transaction on Amazon — paying for an order — and at that exact moment the server needs to restart for a deployment. What happens to that payment? Does it get lost? Does the customer get charged twice?

This is the problem **graceful shutdown** solves. It's about teaching your backend **good manners** — it doesn't slam the door and leave in the middle of a conversation. It finishes what it's doing, cleans up, and then exits properly.

---

## 1. The Problem — What Happens Without Graceful Shutdown

In a typical deployment scenario:
1. New code is pushed to production
2. A new server starts up with the new code
3. Once the new server is ready → old server must shut down
4. Traffic switches from old to new server

**Without graceful shutdown:** The old server is killed instantly while it's processing 50 concurrent requests. Those requests die mid-execution:
- Payments get dropped (customer charged but order not created)
- Database transactions left in inconsistent state → potential data corruption
- Clients get connection reset errors instead of proper responses
- Background jobs abandoned halfway through

**With graceful shutdown:** When told to stop, the old server:
1. Stops accepting new requests
2. Finishes all requests currently in progress
3. Cleans up resources (DB connections, file handles, etc.)
4. Then exits

---

## 2. Process Lifecycle — The Foundation

Your backend application runs as a **process** inside an operating system.

Every process has a lifecycle:
```
Born → Running → Dead
(starts)  (executes)  (terminates)
```

When the OS needs to stop your process, it doesn't just pull the plug. It follows an established protocol of communication via **signals**.

Signals are an IPC (Inter-Process Communication) mechanism from Unix/Linux operating systems. They're how the OS "talks" to your running process.

> 💡 Why Linux? 99% of cloud deployments use Linux. AWS, GCP, Azure — all default to Linux for server processes.

---

## 3. The Three Signals

### SIGTERM — The Polite Request

```
Signal name: SIGTERM (Signal Terminate)
Meaning: "Please finish up and exit when you're ready"
Who sends it: Deployment systems, Kubernetes, PM2, systemd
Can your app catch it? YES
Can your app ignore it? YES (but shouldn't)
```

This is a **gentle nudge**. Your application receives this signal and has a window of time (typically 30 seconds) to:
1. Stop accepting new requests
2. Finish in-flight requests
3. Clean up resources
4. Exit

Kubernetes sends SIGTERM when it wants to stop a pod. PM2 sends it when stopping a process. This is the "proper" way to ask a process to stop.

---

### SIGINT — The Developer Interrupt

```
Signal name: SIGINT (Signal Interrupt)
Meaning: "I want to stop this process now"
How it's sent: Ctrl+C in the terminal
Who uses it: Developers in local development
Can your app catch it? YES
Can your app ignore it? NO (well, technically yes, but Ctrl+C is already your nuclear option locally)
```

When you press Ctrl+C while your backend is running locally, the terminal sends SIGINT to your process.

**Handle SIGINT exactly the same way as SIGTERM.** The intention is the same — shutdown. Whether it's a human pressing Ctrl+C or Kubernetes sending SIGTERM, you want to go through the same graceful shutdown procedure.

---

### SIGKILL — The Nuclear Option

```
Signal name: SIGKILL (Signal Kill)
Meaning: "You are terminated. NOW."
Who sends it: OS (as a last resort), manual kill -9
Can your app catch it? NO — impossible
Can your app ignore it? NO — impossible
```

SIGKILL **cannot be caught or ignored** by your application. The OS bypasses your process entirely and terminates it immediately.

**Analogy:** SIGTERM is politely clicking "Shut Down" in your OS menu. SIGKILL is physically yanking the power cable out of the wall.

**Why does SIGKILL exist?** If an application receives SIGTERM but doesn't shut down within the timeout window, the orchestration system escalates to SIGKILL. Kubernetes by default waits 30 seconds after SIGTERM, then sends SIGKILL.

**The implication:** If your application doesn't implement graceful shutdown and respond to SIGTERM, it will eventually receive SIGKILL and die messily with:
- In-flight requests dropped
- DB transactions uncommitted
- Resources not cleaned up
- Potential data corruption

---

## 4. The Two Steps of Graceful Shutdown

When your application receives SIGTERM/SIGINT, it should do two things in order:

### Step 1 — Connection Draining (Stop + Finish Requests)

**The restaurant analogy:**
When a restaurant closes at 11 PM:
1. The host stops letting new customers in (stop accepting new connections)
2. Existing customers are told "we're closing in 15 minutes, please finish your meal" (existing requests allowed to complete)
3. When the last customer leaves, close up (process exits)

**For your HTTP backend:**
```
SIGTERM received
    ↓
Stop accepting new HTTP connections
(new requests get connection refused or routed to new server)
    ↓
Let existing in-flight requests complete
    ↓
All requests done → proceed to cleanup
```

**Implementation concept:**
```javascript
// Express/Node.js example (conceptual)
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown');
  
  // 1. Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed — no new connections');
  });
  
  // 2. Give existing requests time to finish
  // Most HTTP server libraries handle this automatically in .close()
  
  // 3. Proceed to cleanup
  await cleanup();
  process.exit(0);
});
```

**For different server types:**

| Server type | What "connection draining" means |
|---|---|
| HTTP server | Stop accepting new requests; let existing requests complete |
| Database server | Stop taking new queries; finish existing transactions; commit or rollback |
| WebSocket server | Notify connected clients you're closing; then close sockets |
| Background job worker | Finish the current job being processed; don't pick up new jobs |

---

### Step 2 — Resource Cleanup

After requests finish, your application must release everything it acquired during its lifetime:

**Resources to clean up:**

| Resource | Why it matters if not cleaned up |
|---|---|
| **Database connections** | Uncommitted transactions → deadlocks, data corruption |
| **Network connections** | OS-level connection limits; orphaned connections waste resources |
| **File handles** | OS limits number of open file handles per process; leaking them → "too many open files" errors |
| **Redis/cache connections** | Same as network connections |
| **Temporary files** | Disk space leak |
| **Background job consumers** | Jobs might be marked "in progress" but never complete |

**Critical rule: Clean up in reverse order of acquisition**

```
Acquisition order:        Cleanup order:
  1. Connect to Redis   ←  4. Close Redis
  2. Connect to DB      ←  3. Close DB
  3. Start HTTP server  ←  2. Stop HTTP server
  4. Start job worker   ←  1. Stop job worker (first)
```

Why reverse? Because later-acquired resources may depend on earlier ones. Closing DB before Redis is safe. Closing the HTTP server before draining its connections could leave requests with no DB to talk to.

---

## 5. The Timeout Problem

You can't wait forever for in-flight requests to finish. What if a request is stuck in an infinite loop? What if a client is slow?

**Solution: Shutdown timeout (typically 30 seconds)**

```
SIGTERM received
    ↓
Stop accepting new connections
    ↓
Start 30-second countdown
    ↓
If all requests finish within 30s → graceful exit ✅
If 30s timeout expires and requests still running → force exit anyway ⚠️
```

**How to choose your timeout:**
- Normal REST API with sub-second responses: 10-30 seconds is more than enough
- Long-running operations (file processing, AI generation): may need longer
- Real-time WebSocket connections: need special consideration

**Kubernetes' default:** 30 seconds between SIGTERM and SIGKILL. Your timeout should be less than 30 seconds so you can exit gracefully before the OS pulls the plug.

---

## 6. The Complete Graceful Shutdown Flow

```
1. Application registers signal handlers at startup
         ↓
2. Normal operation — processing requests
         ↓
3. SIGTERM received (from Kubernetes/PM2/etc.)
   OR SIGINT received (from Ctrl+C)
         ↓
4. Signal handler triggers graceful shutdown:
         ↓
5. STOP: No longer accept new HTTP connections
         ↓
6. WAIT: Allow in-flight requests to complete
         (with 30-second timeout)
         ↓
7. CLEANUP (in reverse acquisition order):
   → Stop background job workers
   → Close HTTP server
   → Commit/rollback open DB transactions
   → Close database connection pool
   → Close Redis connections
   → Close any other external connections
   → Release file handles
         ↓
8. Log "Server exited gracefully"
         ↓
9. process.exit(0)  ← clean exit code
```

---

## 7. What the Code Looks Like (Conceptual)

You don't need to memorise this — every framework has library support. But understanding what's happening helps you implement it correctly.

```go
// Go example (conceptual)
func gracefulShutdown(server *http.Server, db *pgxpool.Pool, redis *redis.Client, jobServer *asynq.Server) {
  
  // Step 1: Register signal handler
  sigChan := make(chan os.Signal, 1)
  signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
  
  // Step 2: Wait for signal
  sig := <-sigChan
  log.Info("Received signal", "type", sig)
  
  // Step 3: Create context with 30-second timeout
  ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
  defer cancel()
  
  // Step 4: Stop accepting new HTTP requests (draining)
  log.Info("Stopping HTTP server")
  server.Shutdown(ctx)   // Waits for in-flight requests to complete
  
  // Step 5: Stop job workers
  log.Info("Stopping job server")
  jobServer.Shutdown()
  
  // Step 6: Close DB
  log.Info("Closing database connections")
  db.Close()
  
  // Step 7: Close Redis
  log.Info("Closing Redis connections")  
  redis.Close()
  
  log.Info("Server exited gracefully")
}
```

**What the logs look like:**
```
[INFO] Server starting on :8080
[INFO] Connected to database
[INFO] Background job server started
[INFO] Server ready

# Ctrl+C pressed...

[INFO] Received signal: interrupt
[INFO] Stopping HTTP server
[INFO] All in-flight requests completed
[INFO] Stopping background job server
[INFO] Waiting for workers to finish...
[INFO] All workers finished
[INFO] Closing database connections
[INFO] Closing Redis connections
[INFO] Server exited gracefully
```

---

## 8. Graceful Shutdown in Different Contexts

### Kubernetes
- Sends SIGTERM when pod needs to stop
- Waits `terminationGracePeriodSeconds` (default: 30s)
- Then sends SIGKILL

```yaml
# kubernetes deployment
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # override default 30s
```

### PM2 (Node.js process manager)
```bash
# PM2 sends SIGTERM, waits, then kills
pm2 stop my-app        # sends SIGTERM
pm2 kill my-app        # sends SIGKILL
```

### Docker
```bash
docker stop container  # sends SIGTERM, waits 10s, then SIGKILL
docker kill container  # sends SIGKILL immediately
```

---

## 9. Framework Support

Almost all major HTTP frameworks have built-in graceful shutdown support — you rarely need to implement this from scratch:

| Framework | Method |
|---|---|
| Express (Node.js) | `server.close(callback)` |
| Fastify (Node.js) | `fastify.close()` |
| Go standard `net/http` | `server.Shutdown(ctx)` |
| Gin (Go) | `server.Shutdown(ctx)` |
| FastAPI (Python) | Uses `uvicorn` with signal handlers |
| Spring Boot (Java) | `server.setGracefulShutdownTimeout()` |

---

## Summary

| Concept | Key Point |
|---|---|
| **What graceful shutdown is** | Teaching your backend to exit properly — finish ongoing work, clean up, then stop |
| **Why it matters** | Prevents data corruption, double-charges, dropped transactions, resource leaks |
| **SIGTERM** | Polite stop request from OS/Kubernetes/PM2. Your app CAN catch and handle this. |
| **SIGINT** | Ctrl+C from developer. Same as SIGTERM — handle identically. |
| **SIGKILL** | Nuclear option. Cannot be caught or ignored. Process dies instantly. |
| **Connection draining** | Stop accepting new requests; let existing ones complete |
| **Resource cleanup** | Close DB connections, Redis, file handles — in reverse order of acquisition |
| **Timeout** | 30 seconds is standard. If requests don't finish, force-exit anyway. |
| **Reverse cleanup order** | Clean up in reverse acquisition order to avoid dependency issues |
| **Framework support** | Most frameworks have built-in support — copy the pattern, don't build from scratch |

---

## One-Line Takeaways

> Graceful shutdown is your backend saying "give me 30 seconds to finish what I'm doing and pack up" instead of slamming the door mid-sentence.

> SIGTERM = please stop. SIGINT = Ctrl+C stop. SIGKILL = die right now (can't be caught). Always handle SIGTERM and SIGINT the same way.

> Two steps: (1) stop accepting new connections, let existing ones finish. (2) clean up resources in reverse order. That's it.

---

*Next lecture → Security — SQL injection, XSS, CSRF, authentication vulnerabilities, and how to defend against them.*
