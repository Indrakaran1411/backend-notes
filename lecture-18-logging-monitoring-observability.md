# Lecture 18 — Logging, Monitoring and Observability
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=5PEuwgLOQQM)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Logging, monitoring, and observability are the practices that let you understand what's happening inside your production systems. This lecture defines all three, explains how they work together, covers implementation details (log levels, structured vs unstructured logging, traces), and shows real tooling used in production.

> **Important framing:** These are practices implemented on a spectrum. No company can say "we have 100% logging coverage." You build towards it incrementally. Don't be intimidated by the breadth of this topic.

---

## 1. Why These Practices Exist

Modern backends run in distributed environments — multiple services, multiple servers, multiple regions, users worldwide. When something goes wrong at 3 AM:

- **Without these practices:** You get an alert that "something is broken" → spend hours digging through raw server logs trying to find what failed, where, and why
- **With these practices:** Alert fires → you see the exact error rate spike → find the relevant logs → follow the trace to the exact function that failed → fix it

These practices turn debugging from detective work into a structured investigation.

---

## 2. The Three Concepts Defined

### Logging
**Recording important events** that happen throughout your application's lifecycle.

Examples of events to log:
- User logs in / logs out
- Database query executed
- External API call made
- Error occurred
- Background job started/completed
- New resource created (business event)

Each log entry includes **metadata**: timestamp, user ID, request ID, latency, function name, status — whatever helps you understand what happened.

> Think of logs as a **journal/diary** your backend keeps. When something goes wrong, you read the journal to understand exactly what happened, when, and why.

---

### Monitoring
**Continuously checking the health and performance of your system** — real-time data (with a 10-15 second delay) and historical trends.

Examples of what to monitor:
- CPU and memory usage of your server
- Number of requests processed per second
- Number of open database connections
- Error rate (% of requests returning non-2xx)
- Response time (average, p95, p99)
- Queue length (for background jobs)

Monitoring gives you **numbers and trends**. It tells you *something is wrong*. But not necessarily *exactly what* or *why*.

> Monitoring is like checking your car's dashboard lights. A warning light tells you there's a problem, but not which specific part failed.

---

### Observability
A system is **observable** if you can determine its internal state by examining its external outputs.

Observability has **three pillars**. A system is only truly observable if all three are implemented:

| Pillar | What it gives you | Question it answers |
|---|---|---|
| **Logs** | Record of events | What happened? |
| **Metrics** | Numbers and patterns | How is the system performing over time? |
| **Traces** | Transaction tracking across components | Where exactly in the system did this happen? |

**The key evolution:** Traditional monitoring tells you *that* something is wrong. Observability (logs + metrics + traces together) tells you *exactly what* is wrong and *where*.

---

## 3. How They Work Together — The Debugging Workflow

```
Alert fires in Slack:
"Error rate exceeded 80% in the last 5 minutes"
         ↓
Go to METRICS dashboard
"Error rate is 82%, spiked at 14:32"
"Which API? → GET /api/todos"
         ↓
Go to LOGS filtered to that time window
"Found: Unauthorized errors + DB connection errors"
"SELECT log entry with error"
         ↓
Follow the TRACE attached to that log
"Request started at auth middleware"
"→ Reached service layer"
"→ Failed at repository layer (DB connection timeout)"
         ↓
Root cause found: database connection pool exhausted
Fix: increase pool size or investigate slow queries
```

This is why all three pillars work together — each one answers a different question, and together they give you a complete picture.

---

## 4. Logging in Depth

### Log Levels

Logs are categorised by severity. Your logging library lets you filter what gets recorded based on the environment.

| Level | When to use | Enabled in |
|---|---|---|
| **DEBUG** | Detailed troubleshooting info — every function call, every variable value | Development only |
| **INFO** | Normal successful operations — user logged in, to-do created, job completed | Development + Production |
| **WARN** | Something unexpected but not breaking — user entered wrong password, retry attempt | Development + Production |
| **ERROR** | Something failed — DB query error, external API failed, validation error | Development + Production |
| **FATAL** | Critical failure — application is shutting down | Development + Production |

**Why different levels?**
- In development: `DEBUG` gives you maximum visibility for troubleshooting
- In production: `DEBUG` would flood your logging service with gigabytes of noise and cost a fortune. Set to `INFO` or higher.

**Environment config controls this:**
```
Development:  LOG_LEVEL=debug
Production:   LOG_LEVEL=info
```

---

### Structured vs Unstructured Logging

#### Unstructured (Plain Text) — For Development
```
2024-09-23 14:32:01 [INFO] User logged in: userId=42
2024-09-23 14:32:03 [ERROR] DB query failed: timeout after 5000ms
2024-09-23 14:32:05 [INFO] Created todo: id=123 title="Buy groceries"
```
Human-readable. Coloured. Easy to spot issues in the terminal. Hard for tools to parse automatically.

#### Structured (JSON) — For Production
```json
{"level":"info","timestamp":"2024-09-23T14:32:01Z","message":"User logged in","userId":42,"requestId":"abc-123","latency":45}
{"level":"error","timestamp":"2024-09-23T14:32:03Z","message":"DB query failed","error":"timeout after 5000ms","query":"SELECT * FROM todos","userId":42,"requestId":"abc-123"}
{"level":"info","timestamp":"2024-09-23T14:32:05Z","message":"Created todo","todoId":123,"title":"Buy groceries","userId":42,"requestId":"abc-123"}
```

**Why JSON in production?**
- Log management tools (ELK stack, Loki, New Relic, Datadog) can automatically parse and index every field
- You can search: "find all logs where userId=42 AND level=error AND timestamp > 2024-09-23"
- Build dashboards and alerts based on specific fields
- If logs are plain text, tools have to do regex matching — slow, error-prone, incomplete

**Rule:** Unstructured logs in development (human-readable), JSON logs in production (machine-parseable).

---

### What to Log

#### Always Log
- All errors (with full context: user ID, request ID, what operation was being performed)
- Important business events (user signed up, order placed, payment processed)
- Authentication events (login, logout, failed login attempts)
- Request start/end with latency
- External service calls (made/failed/timed out)
- Background job start/complete/fail

#### Never Log (Security)
- Passwords (even hashed)
- Credit card numbers
- API keys or tokens
- Full email addresses (use user ID instead)
- Any PII that's not strictly necessary

#### Log Entry Fields to Include
```json
{
  "level": "error",
  "timestamp": "2024-09-23T14:32:01Z",
  "message": "Failed to create todo",
  "requestId": "abc-123",       // trace all logs for this request
  "userId": "usr_42",           // who triggered this
  "operation": "create_todo",   // what was being done
  "error": "unique constraint violation",  // what went wrong
  "latency": 145,               // how long it took
  "env": "production"
}
```

> **requestId / correlationId** is especially important. Every request gets a unique ID. Include it in every log line. When debugging, filter by requestId to see the complete story of that single request.

---

## 5. Metrics

Metrics are **numbers that represent the state and performance of your system** over time.

### Types of Metrics

| Category | Examples |
|---|---|
| **Infrastructure** | CPU usage %, memory usage %, disk I/O |
| **Application performance** | Requests/second, average response time, p95/p99 latency |
| **Error rates** | % requests returning 4xx or 5xx, DB query failure rate |
| **Business metrics** | Orders placed/hour, successful payments, active users |
| **Resource utilisation** | DB connections open, queue length, cache hit rate |

### Why Business Metrics Matter

Error rate can be normal while something is seriously wrong. Example:

```
Error rate: 0.1% (looks fine)
Successful order completions: dropped from 1000/hour to 50/hour

This indicates a serious problem in the payment flow that isn't 
throwing errors — maybe it's silently failing, or maybe it's a
logic bug (see Lecture 16 on Logic Errors).
```

Business metrics catch problems that pure technical metrics miss.

### Metrics Are Different From Logs

| Logs | Metrics |
|---|---|
| Discrete events | Continuous measurements |
| "What happened at 14:32:01" | "What was the request rate over the last hour?" |
| High detail, high storage cost | Aggregated, low storage cost |
| Good for debugging specific incidents | Good for spotting trends, alerting |

---

## 6. Traces

A trace is a **record of a request's journey through all the components of your system**.

### What a Trace Contains

```
Trace ID: trace_abc123

Span 1: HTTP Request received by auth middleware     [0ms - 5ms]
Span 2: JWT token validated                          [1ms - 3ms]  
Span 3: Request reaches handler                      [5ms - 6ms]
Span 4: Validation layer                             [6ms - 8ms]
Span 5: Service method: createTodo()                 [8ms - 145ms]
  Span 6: Check parent-child relationship            [9ms - 20ms]
  Span 7: Database INSERT query                      [20ms - 140ms]  ← slow!
  Span 8: Cache invalidation                         [141ms - 145ms]
Span 9: Response sent                                [145ms - 146ms]

Total: 146ms
Problem: Span 7 (DB insert) took 120ms — unexpectedly slow
```

Without traces, you'd know a request took 146ms — too slow. With traces, you can pinpoint that the database INSERT is the bottleneck.

### Instrumentation

To generate traces, you **instrument** your code — add measurement points at key locations:

```go
// Go example with New Relic
func createTodoService(ctx context.Context, data CreateTodoRequest) (*Todo, error) {
  // Start a segment/span for this function
  txn := newrelic.FromContext(ctx)
  defer txn.StartSegment("create_todo_service").End()
  
  // Add attributes to this span
  txn.AddAttribute("userId", data.UserID)
  txn.AddAttribute("title", data.Title)
  
  // Log the event
  log.Info("Creating todo", "userId", data.UserID, "title", data.Title)
  
  // Database call — also instrumented
  todo, err := todoRepo.Insert(ctx, data)
  if err != nil {
    log.Error("Failed to create todo", "error", err)
    txn.NoticeError(err)
    return nil, err
  }
  
  log.Debug("Todo created", "todoId", todo.ID)
  return todo, nil
}
```

### OpenTelemetry — The Standard

OpenTelemetry (OTel) is an **open standard for instrumentation** — a vendor-neutral set of APIs, SDKs, and tools for collecting logs, metrics, and traces.

- Supported in all major languages (Go, Node.js, Python, Java, etc.)
- Works with any backend (Jaeger, Zipkin, New Relic, Datadog, etc.)
- Write your instrumentation once → send data to any tool

This means you can switch from New Relic to Grafana/Jaeger without rewriting all your instrumentation code.

---

## 7. The Tooling Landscape

### Open Source Stack (Self-Hosted)

```
Logs     → Loki (storage) + Promtail (collection)
Metrics  → Prometheus (collection + storage)
Traces   → Jaeger (collection + storage)
Dashboard → Grafana (visualise everything)
```

**Pros:** Free, powerful, full control, no vendor lock-in  
**Cons:** Requires setup, maintenance, infrastructure expertise, and time

### Proprietary All-in-One Solutions

| Tool | Notes |
|---|---|
| **New Relic** | Full observability platform — logs, metrics, traces, dashboards, alerts |
| **Datadog** | Industry standard, very feature-rich |
| **Dynatrace** | AI-powered observability |
| **Grafana Cloud** | Managed version of the open source stack |

**Pros:** Quick to set up, maintained, support available, less infrastructure to manage  
**Cons:** Cost, vendor lock-in

**When to choose which:**
- Small team or early-stage startup → proprietary (New Relic/Datadog) — get value faster
- Larger team with DevOps expertise → open source stack — more control, lower cost at scale

---

## 8. The Observability Workflow in Practice

### What a Full Observability Pipeline Looks Like

```
Application Code
  ↓ (logs, metrics, traces emitted)
Collection Layer
  → Promtail/Fluent Bit (log collection)
  → OpenTelemetry Collector (metrics + traces)
  ↓
Storage Layer
  → Loki (logs)
  → Prometheus (metrics)  
  → Jaeger (traces)
  ↓
Visualisation + Alerting
  → Grafana Dashboards
  → Alertmanager → Slack/PagerDuty alerts
```

### Setting Up Alerts

Define thresholds:
```
IF error_rate > 5% for 5 minutes → Alert: "Critical: High error rate on API service"
IF p99_latency > 2000ms → Alert: "Warning: Slow response times"
IF db_connections > 45 (of pool size 50) → Alert: "Warning: DB connection pool nearly full"
IF queue_length > 1000 → Alert: "Warning: Task queue backing up"
```

**Alert fatigue warning:** Only create alerts for things that need human action. Too many alerts → engineers start ignoring them. Every alert should be actionable.

---

## 9. Implementation Checklist

### Logging
- [ ] Choose a logging library (Winston/Pino for Node.js, Zap/Logrus for Go, structlog for Python)
- [ ] Configure log levels per environment (debug local, info production)
- [ ] Use unstructured/coloured logs in development
- [ ] Use structured JSON logs in production
- [ ] Include requestId in every log entry (set in middleware, pass via context)
- [ ] Log all errors with full context
- [ ] Log business events at INFO level
- [ ] Never log sensitive data (passwords, tokens, card numbers)

### Metrics
- [ ] Track request count, error rate, latency per endpoint
- [ ] Track business metrics (orders, signups, etc.)
- [ ] Track infrastructure metrics (CPU, memory, DB connections)
- [ ] Set up dashboards in Grafana or your chosen tool
- [ ] Create actionable alerts with appropriate thresholds

### Traces
- [ ] Instrument key functions (service methods, DB queries, external calls)
- [ ] Use OpenTelemetry for vendor-neutral instrumentation
- [ ] Connect traces to logs via traceId/requestId
- [ ] Set up trace visualisation (Jaeger, New Relic APM, Datadog APM)

---

## Summary

| Concept | What it does | Answers |
|---|---|---|
| **Logging** | Records events with metadata | What happened? When? For which user? |
| **Monitoring** | Tracks system health with numbers | Is performance degrading? What are the trends? |
| **Observability** | Logs + Metrics + Traces together | What happened, how is the system, and exactly where did it fail? |

| Log Level | Use for |
|---|---|
| DEBUG | Local dev troubleshooting (disabled in prod) |
| INFO | Normal operations, business events |
| WARN | Unexpected but non-critical (wrong password, retry) |
| ERROR | Failures (DB error, external API failed) |
| FATAL | Critical — app is shutting down |

| Format | When |
|---|---|
| Unstructured (plain text) | Development — human-readable |
| Structured (JSON) | Production — machine-parseable by tools |

| Tool Category | Open Source | Proprietary |
|---|---|---|
| Logs | Loki + Promtail | New Relic, Datadog |
| Metrics | Prometheus | New Relic, Datadog |
| Traces | Jaeger | New Relic APM, Datadog APM |
| Dashboards | Grafana | New Relic, Datadog |
| All-in-one | Grafana Stack | New Relic, Datadog, Dynatrace |

---

## One-Line Takeaways

> Monitoring tells you something is wrong. Observability tells you exactly what is wrong and where.

> Three pillars: Logs = what happened. Metrics = how the system is performing. Traces = which code path the request took.

> In production, always log in JSON. Your log management tools depend on it. In development, log in human-readable format — your sanity depends on it.

> A requestId in every log line is the single most useful debugging tool. Set it once in middleware, include it everywhere.

---

*Next lecture → Graceful Shutdown — How to stop your server without breaking in-flight requests or losing data.*
