# 🧠 Concurrency & Parallelism: IO Bound vs CPU Bound
### Deep Notes for Backend Engineers — Simple English, Real Examples

---

## 📌 Table of Contents
1. [Why Every Backend Needs Concurrency](#1-why-every-backend-needs-concurrency)
2. [The Waste Problem — CPU Sitting Idle](#2-the-waste-problem--cpu-sitting-idle)
3. [IO Bound vs CPU Bound](#3-io-bound-vs-cpu-bound)
4. [Concurrency vs Parallelism — The Core Difference](#4-concurrency-vs-parallelism--the-core-difference)
5. [Threads — How They Work](#5-threads--how-they-work)
6. [Thread Overhead — Why Too Many Threads is Bad](#6-thread-overhead--why-too-many-threads-is-bad)
7. [Event Loop Model](#7-event-loop-model)
8. [Async / Await — How It Actually Works](#8-async--await--how-it-actually-works)
9. [Go Routines — Virtual Threads](#9-go-routines--virtual-threads)
10. [Race Conditions — Shared State Problems](#10-race-conditions--shared-state-problems)
11. [Solutions to Race Conditions](#11-solutions-to-race-conditions)
12. [Final Summary Cheat Sheet](#12-final-summary-cheat-sheet)

---

## 1. Why Every Backend Needs Concurrency

**Every backend server must handle multiple requests at the same time.**

Imagine a pizza shop that serves only one customer at a time. 1000 people are waiting. Disaster. Same with your server.

- A web server that handles **one request at a time** means all other users wait or get an error.
- In production, thousands of users send requests simultaneously.
- **Without concurrency, your server is useless at scale.**

> 🍕 **Analogy:** A restaurant kitchen with only one chef who finishes one dish completely before starting the next. No real kitchen works this way.

---

## 2. The Waste Problem — CPU Sitting Idle

**A CPU doing nothing is wasted money and wasted potential.**

### Numbers to Remember:
- A modern CPU can execute **~3 billion instructions per second** = **3 million instructions per millisecond**
- A typical database query on a **local network** takes **1–2 ms**
- A database in a **different availability zone** takes **20–30 ms**
- A database in a **different region** takes **90–100 ms**

### The Math of Waste:
If your server waits 100ms for a DB response and does nothing:
- It could have executed **300 million instructions**
- Instead it executed **zero**

### Real API Call Breakdown:
| Activity | Time |
|---|---|
| 5 external calls (DB + APIs) × avg 50ms each | 250ms |
| Actual CPU processing | 10ms |
| **Total time** | **260ms** |
| **CPU idle %** | **~95%** |

**The server's CPU is idle 95% of the time.** Concurrency solves this by using those idle periods for other tasks.

---

## 3. IO Bound vs CPU Bound

This is **the most important concept** in this entire topic.

### **IO Bound** — Waiting for something external
The CPU is just sitting there waiting. The bottleneck is Input/Output.

**Examples:**
- Database queries (waiting for DB to respond)
- External API calls (calling Stripe, SendGrid, etc.)
- File system reads/writes (uploading/downloading files)
- Network requests
- Cache reads (Redis)
- Logging to standard output

> 💡 **70% or more of a typical backend application's time is IO bound.**

### **CPU Bound** — Actually doing heavy computation
The CPU is actively crunching numbers. The bottleneck is processing power.

**Examples:**
- Image processing (matrix multiplication)
- Video encoding/rendering
- Encryption / JWT verification
- ML model inference
- Large data transformations

> 💡 **Simple things like JSON parsing or validation are technically CPU bound, but they finish in 1–2ms and are not a real bottleneck.**

### Quick Decision Rule:
```
Is the code WAITING for something external?  → IO Bound → Use Concurrency
Is the code DOING heavy computation itself?   → CPU Bound → Use Parallelism
```

---

## 4. Concurrency vs Parallelism — The Core Difference

People confuse these two constantly. Here's the simple truth:

### **Parallelism = Doing multiple things AT THE SAME MOMENT**
- Requires multiple CPU cores
- 2 cores = 2 tasks running literally simultaneously
- One CPU core can only execute **one instruction at one moment** — this is a hardware constraint

### **Concurrency = Dealing with multiple things at once (but not necessarily at the same moment)**
- Can work with a **single CPU core**
- About **structuring your program** to start, pause, and resume tasks
- Feels like doing multiple things, but zoomed in — one thing runs at a time

> 🎭 **Theatre Analogy:**
> - **Parallelism** = Two plays happening on two different stages at the exact same time
> - **Concurrency** = One actor switching between two roles mid-scene, so fast it feels simultaneous

### Visual Timeline Example (1 CPU Core, 2 Requests):

```
Time (ms):    0    5    10   15   20   25   30   35   40   45   50   55   60
              |    |    |    |    |    |    |    |    |    |    |    |    |

Request A:   [CPU] [---- WAITING FOR DB ----]              [CPU][DONE]
Request B:        [-------- CPU ----------] [-- WAITING FOR DB --][CPU][DONE]
```

- **Request A** used CPU for 5ms, then waited for DB
- **Request B** immediately got CPU while A was waiting
- **Neither wasted CPU time**
- Both were "in progress" at the same time → **Concurrency**
- But only ONE had the CPU at any moment → **Not Parallelism**

---

## 5. Threads — How They Work

**A thread is an independent unit of execution managed by your Operating System.**

### What the OS Creates for Each Thread:
1. **Stack** — Tracks function calls and local variables (like a call history log)
2. **Instruction Pointer** — Marks exactly where in the code the thread currently is
3. **Internal data structures** — OS bookkeeping to manage the thread's state

### Simple Code Example (Python):
```python
import threading

def handle_request(request):
    user = db.query("SELECT * FROM users WHERE id = ?", request.user_id)
    # ^ Thread BLOCKS here waiting for DB response
    # OS switches to another thread while waiting
    return user
```
When the thread hits `db.query(...)`, it **blocks**. The OS scheduler sees this and switches to another thread that has work to do.

### Thread Scheduling (Preemptive):
- The OS **scheduler** decides which thread runs and when
- Each thread gets a **time slice** (a few milliseconds of CPU time)
- After the time slice, the thread is **paused** whether it likes it or not
- This is called **preemptive scheduling**

> 🚦 **Traffic Light Analogy:** The OS scheduler is the traffic light. Threads are cars. Each car gets a green light for a few seconds, then must stop and let others go.

---

## 6. Thread Overhead — Why Too Many Threads is Bad

Threads are powerful, but they're **expensive**. Three types of overhead:

### 1. Memory Overhead
- Each thread gets its own **stack**
- On Linux, default stack size = **8 MB** (much is virtual, but still)
- Even at 1 MB per thread: **10,000 threads = ~10 GB RAM** just for thread stacks
- Your server also needs memory for app logic, DB connections, caches — it will **crash**

### 2. Creation Overhead
- Creating a thread requires a **system call** to the OS kernel
- Kernel sets up stack, data structures, adds to scheduler
- Takes **microseconds to milliseconds** — adds up under load

### 3. Context Switch Overhead (The Big One)
When switching from Thread A to Thread B, the OS must:
1. **Save** Thread A's CPU registers
2. **Update bookkeeping** for Thread A's state
3. **Select** the next thread (Thread B)
4. **Restore** Thread B's saved registers and state

Each switch = **1–10 microseconds** on modern hardware.

With 1000 threads → constant switching → **milliseconds of wasted CPU** on pure maintenance.

> 🏃 **Marathon Analogy:** Imagine a runner who stops every 10 steps, takes off their shoes, puts on different shoes, then runs 10 more steps. The shoe switching is the context switch — pure overhead, no actual progress.

---

## 7. Event Loop Model

**The event loop uses a SINGLE thread to handle many tasks concurrently.**

### Core Idea:
- Never block the thread
- When a task needs IO, it **registers a callback** and gives up control
- The event loop checks if any IO completed, and if so, runs the callback

### How it Works Step by Step:
```
1. Request A arrives → start parsing → make DB query
2. DB query starts → Request A says "call me back when done" → hands control back
3. Event Loop picks up Request B → start parsing → make DB query
4. DB query starts → Request B says "call me back when done" → hands control back
5. Event Loop runs in a loop, checking: "Is A's DB response ready? Is B's DB response ready?"
6. A's DB responds → run A's callback → serialize JSON → send response
7. B's DB responds → run B's callback → serialize JSON → send response
```

### OS Support for Event Loop:
- **Linux:** `epoll` — monitors thousands of connections efficiently
- **macOS:** `kqueue`
- **Windows:** `IOCP`

These OS tools let the event loop check thousands of pending IO operations in one loop iteration.

### Old-style Callback Code (JavaScript):
```javascript
// Before ES6 — callback hell
function handleRequest(req, res) {
    db.query("SELECT * FROM users WHERE id = ?", req.userId, function(err, user) {
        // This function only runs AFTER the DB responds
        if (err) return res.error(err);
        res.send(user);
    });
}
```

### The Callback Hell Problem:
```javascript
// When you have 3+ async operations:
db.query(q1, function(err, result1) {
    db.query(q2, function(err, result2) {
        apiCall(url, function(err, result3) {
            // Code keeps going right → impossible to read
        });
    });
});
```

This is why **async/await** was invented.

---

## 8. Async / Await — How It Actually Works

**Async/await is just syntactic sugar on top of callbacks. It makes callbacks readable.**

### Modern Code (Same logic, readable):
```javascript
async function handleRequest(req, res) {
    const user = await db.query("SELECT * FROM users WHERE id = ?", req.userId);
    // ^ "await" means: pause this function, give CPU to event loop,
    //   resume HERE when DB responds
    res.send(user);
}
```

### The Key Rules:
- **`async`** marks a function as one that can be paused and resumed
- **`await`** marks the exact pause point — "wait here, give up CPU, come back when done"
- You can **only use `await` inside an `async` function** — because the whole function must be convertible to a state machine

### Under the Hood — State Machine:
The runtime converts your async function into a state machine:

```
State 0: Run synchronous code, start DB query, register callback → go to State 1
State 1: [PAUSED — waiting for DB] ...event loop does other things...
         DB responds → move to State 2
State 2: Process result, run remaining code, return value
```

```javascript
// What you write:
async function fetchUserData(userId) {
    const user = await db.getUser(userId);   // State 0 → 1
    const orders = await db.getOrders(userId); // State 1 → 2
    return { user, orders };
}

// What the runtime sees (simplified):
function fetchUserData(userId) {
    let state = 0, user, orders;
    function step() {
        switch(state) {
            case 0:
                state = 1;
                return db.getUser(userId).then(result => { user = result; step(); });
            case 1:
                state = 2;
                return db.getOrders(userId).then(result => { orders = result; step(); });
            case 2:
                return { user, orders };
        }
    }
    return step();
}
```

### Why You Must NEVER Block the Event Loop:
```javascript
// ❌ BAD — This blocks the event loop for 5 seconds
async function badHandler() {
    const result = heavyImageProcessing(); // Takes 5000ms, no await
    // During these 5 seconds, NO other request can be handled
    return result;
}

// ✅ GOOD — Offload CPU work to worker threads
async function goodHandler() {
    const result = await runInWorkerThread(heavyImageProcessing);
    return result;
}
```

### Event Loop Efficiency vs Threads:
| Feature | Threads | Event Loop |
|---|---|---|
| Memory per task | MB (stack) | KB (callback function) |
| Context switch cost | 1–10 μs per switch | Near zero (no switch) |
| Best for | CPU bound | IO bound |
| Risk | Race conditions, memory | Blocking the loop |

---

## 9. Go Routines — Virtual Threads

**Go takes the best of both worlds: lightweight virtual threads managed by the Go runtime, not the OS.**

### The Problem Go Solves:
- OS threads are expensive → can't create one per request
- Event loop requires never-blocking discipline → complex to write

### Go's Solution:
- **Go routines** = virtual threads (much lighter than OS threads)
- Go's standard HTTP server **creates a new goroutine for every request** (affordable because they're cheap)
- The **Go runtime scheduler** manages goroutines — not the OS

### Go HTTP Server (from source):
```go
// Go's net/http package does this for every request:
go c.serve(connCtx)  // "go" keyword = new goroutine
```

### Go Runtime Scheduler Architecture (M:N Model):
```
OS Threads (M):  [M1]    [M2]    [M3]    [M4]   ← Created once, based on CPU cores
                  |       |       |       |
Go Routines (N): [G1,G2] [G3,G4] [G5]   [G6,G7,G8] ← Many goroutines per thread
```

- **GoMaxProcs** setting = number of OS threads (default = number of CPU cores)
- Each OS thread runs **many goroutines** sequentially
- When a goroutine blocks for IO, Go runtime **pauses it and runs another** on the same OS thread
- **No OS context switch needed** → dramatically faster

### Go Handler Code:
```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // This runs in its own goroutine
    rows, err := db.Query("SELECT * FROM users WHERE id = $1", userID)
    // ^ Goroutine PAUSES here (Go runtime handles it)
    // ^ Go runtime runs OTHER goroutines on this same OS thread
    // ^ When DB responds, this goroutine RESUMES
    
    json.NewEncoder(w).Encode(rows)
}
```

### Why Go Routines Are Lightweight:
| Property | OS Thread | Go Routine |
|---|---|---|
| Stack size | ~8 MB | ~2–8 KB (grows as needed) |
| Creation cost | Microseconds (syscall) | Nanoseconds (runtime call) |
| Switch cost | OS context switch | Pointer switch only |
| Max practical count | ~10,000 | ~Millions |

> 🐝 **Analogy:** OS threads are like hiring full-time employees (expensive, limited). Go routines are like hiring freelancers per task (cheap, unlimited, managed by a smart coordinator).

---

## 10. Race Conditions — Shared State Problems

**When multiple concurrent tasks access and modify the same variable, chaos happens.**

### The Classic Counter Problem (Threading):
Two threads both try to increment a counter from 0 to 2:

```
Timeline:
Thread A: READ counter (value=0) → ADD 1 → (value in register=1) → WRITE 1 to counter
Thread B:              READ counter (value=0) → ADD 1 → (value in register=1) → WRITE 1 to counter

Expected result: counter = 2
Actual result:   counter = 1  ← Thread A's increment was LOST
```

This happens because incrementing is actually **3 steps**:
1. Read current value into register
2. Add 1 to register
3. Write register value back to memory

If two threads interleave between these steps, **one update gets overwritten**.

This is called a **Race Condition** — the result depends on the unpredictable "race" between threads.

### Race Conditions Even Happen with Async/Await (Single Thread!):
```javascript
let balance = 100;

async function withdraw(amount) {
    if (balance >= amount) {      // Step 1: Check
        await processPayment();    // Step 2: IO — gives up control here!
        balance -= amount;         // Step 3: Deduct
    }
}

// Two calls happen "simultaneously":
withdraw(100); // Call 1: Checks balance=100 ✓, starts payment, PAUSES
withdraw(100); // Call 2: Checks balance=100 ✓ (still 100!), starts payment, PAUSES
// Call 1 resumes: balance = 100 - 100 = 0
// Call 2 resumes: balance = 0 - 100 = -100  ← WRONG! We gave away $100 we don't have
```

**The check and the update are not atomic** — another task can sneak in between them.

> 🏦 **Bank Analogy:** Two bank tellers both check your balance at the same second ($100), both approve a $100 withdrawal simultaneously. You end up with -$100.

---

## 11. Solutions to Race Conditions

### Solution 1: Locks / Mutex (Mutual Exclusion)
Only one thread can enter a "locked" section at a time.

```python
import threading

lock = threading.Lock()
counter = 0

def increment():
    global counter
    with lock:          # Acquire lock — only 1 thread enters at a time
        counter += 1    # Safe: no other thread can be here simultaneously
    # Lock is released automatically when the "with" block ends
```

The `with lock` section is called a **critical section** — protected from concurrent access.

**Downside:** If too many threads are waiting for the lock, you create a **bottleneck** (threads queue up).

### Solution 2: Channels (Go's approach)
Instead of sharing a variable, pass messages between goroutines. Only ONE goroutine owns and updates the variable.

```go
// Instead of multiple goroutines writing to a shared counter:
counterCh := make(chan int, 1)  // Channel as a message queue

go func() {
    count := 0
    for msg := range counterCh {
        count += msg  // Only THIS goroutine updates count
    }
}()

// Other goroutines SEND to the channel — they don't update directly
counterCh <- 1  // "Please add 1"
counterCh <- 1  // "Please add 1"
```

> 📬 **Analogy:** Instead of everyone grabbing the same pen to write in a shared notebook (race condition), everyone sends notes to one secretary who does all the writing.

### Solution 3: Atomic Operations
For simple operations like counters, use OS-level atomic instructions:

```python
import threading
counter = 0
lock = threading.Lock()

# Or use atomic integers from libraries — read-modify-write in a single CPU instruction
# Atomic operations cannot be interrupted mid-way
```

### The Golden Rule:
> **"Don't communicate by sharing memory; share memory by communicating."** — Go proverb

---

## 12. Final Summary Cheat Sheet

### Core Concepts:

| Concept | One-Line Definition |
|---|---|
| **IO Bound** | Task spends most time WAITING for external things (DB, API, files) |
| **CPU Bound** | Task spends most time COMPUTING (encryption, image processing) |
| **Concurrency** | Dealing with multiple tasks at once (can be 1 CPU core) |
| **Parallelism** | Doing multiple tasks at the exact same moment (needs multiple cores) |
| **Thread** | OS-managed independent execution unit (heavy, ~8MB stack) |
| **Event Loop** | Single thread, never blocks, uses callbacks to handle IO |
| **async/await** | Syntactic sugar for callbacks, converts function to a state machine |
| **Go Routine** | Lightweight virtual thread, managed by Go runtime (not OS) |
| **Race Condition** | Bug where concurrent access to shared state gives wrong results |
| **Mutex/Lock** | Mechanism to allow only one thread into a critical section at a time |

### When to Use What:

```
┌─────────────────────────────────────────────────────┐
│                    Your Workload                     │
├─────────────────────┬───────────────────────────────┤
│      IO Bound       │         CPU Bound              │
│ (DB, APIs, files)   │ (encryption, video, ML)        │
├─────────────────────┼───────────────────────────────┤
│  Use CONCURRENCY    │   Use PARALLELISM              │
│  - async/await      │   - OS Threads                 │
│  - Event Loop       │   - Multiple CPU Cores         │
│  - Go Routines      │   - Worker Threads             │
│  - Virtual Threads  │   - Go Routines (also good!)   │
└─────────────────────┴───────────────────────────────┘
```

### Programming Language Concurrency Models:

| Language | Concurrency Model | Underlying Mechanism |
|---|---|---|
| **JavaScript / Node.js** | async/await, Promises | Single-threaded Event Loop |
| **Python** | async/await, asyncio | Event Loop |
| **Go** | goroutines | Go Runtime Scheduler (M:N threads) |
| **Java** (modern) | Virtual Threads | JVM-managed lightweight threads |
| **Rust** | async/await | Event Loop (Tokio runtime) |

### The 3 Thread Overheads (Memory Trick: **MCC**):
- **M**emory — ~8MB stack per thread
- **C**reation — syscall required, microseconds to create
- **C**ontext Switch — 1–10μs per switch, pure overhead

### Most Important Things to Remember:
1. **95% of backend time is IO bound** — concurrency is mandatory, not optional
2. **Concurrency ≠ Parallelism** — concurrency is about structure, parallelism is about hardware
3. **Event loop is great for IO, terrible for CPU-heavy work** — never block it
4. **Go routines are ~1000x cheaper than OS threads** — create freely
5. **Shared mutable state = race conditions** — use locks, channels, or atomic operations
6. **async/await is just callbacks** — your function becomes a state machine under the hood

---

*Notes based on "Concurrency & Parallelism: IO Bound vs CPU Bound" — Backend Engineering Fundamentals*
