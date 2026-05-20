# Lecture 5 — Understanding HTTP for Backend Engineers
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=a3C1DMswClQ)  
> Series: Backend from First Principles

---

## What This Lecture Is About

HTTP is the **foundation of everything** in backend engineering. Every request your backend ever handles comes in through HTTP. Every response goes out through HTTP. If you don't understand HTTP deeply, you're building on a shaky foundation.

This lecture covers HTTP from scratch — what it is, how messages look, headers, methods, CORS, status codes, caching, content negotiation, compression, persistent connections, large file handling, and SSL/TLS.

---

## 1. The Two Core Ideas Behind HTTP

Everything in HTTP is built on two foundational ideas. Understand these and everything else makes sense.

---

### Idea 1 — Statelessness

**Stateless = the server has no memory of past requests.**

Every single HTTP request is:
- **Independent** — treated as a brand new event
- **Self-contained** — carries ALL the information the server needs to process it (headers, URL, method, auth token, etc.)

After the server responds, it forgets the request ever happened. The next request from the same client is treated as completely unrelated.

**Real example:**  
To access your profile page, the client must send credentials (cookie or token) on EVERY request. The server won't remember you from the last request.

#### Why is statelessness good?

| Benefit | Explanation |
|---|---|
| **Simplicity** | Server doesn't need to store session data — less complexity |
| **Scalability** | Any server can handle any request — no single server needs to "own" a session |
| **Fault tolerance** | If a server crashes, no session is lost — the next request just goes to another server |

#### But wait — what about logins and shopping carts?
HTTP is stateless, but applications need state. So developers implement state management **on top of** HTTP using:
- **Cookies** — small data stored in the browser, sent with every request
- **Sessions** — server-side storage with a session ID sent via cookie
- **Tokens (JWT)** — signed token carrying user info, sent in headers

> 💡 HTTP doesn't remember you. You remind it every time.

---

### Idea 2 — Client-Server Model

HTTP always has exactly two roles:

| Role | Who | What they do |
|---|---|---|
| **Client** | Browser, mobile app, Postman, another server | Initiates the request |
| **Server** | Your backend (Express, FastAPI, Go server, etc.) | Waits for requests, processes them, responds |

**Key rule:** Communication is ALWAYS initiated by the client. The server never spontaneously sends data to the client in standard HTTP. (WebSockets break this rule — covered in a later lecture.)

> 💡 HTTP = always client asks, server answers. Never the other way around.

---

## 2. How HTTP Actually Sends Data — TCP

Before HTTP messages can travel, a connection must be established. HTTP uses **TCP (Transmission Control Protocol)** for this.

Why TCP and not UDP?
- HTTP needs **reliability** — messages must arrive, in order, without loss
- TCP guarantees this. UDP does not.

### The OSI Model (Brief)
```
Layer 7 — Application  ← HTTP lives here (what we work with)
Layer 6 — Presentation
Layer 5 — Session
Layer 4 — Transport    ← TCP lives here
Layer 3 — Network      ← IP lives here
Layer 2 — Data Link
Layer 1 — Physical
```

As backend engineers, we primarily work at **Layer 7 (Application)**. TCP handshakes, IP routing — that's network engineering territory. Good to know it exists, but don't get lost in it.

---

## 3. HTTP Versions — The Evolution

| Version | Key Change |
|---|---|
| **HTTP/1.0** | New TCP connection for every single request — very slow |
| **HTTP/1.1** | Persistent connections — reuse the same TCP connection for multiple requests. Added chunk transfer encoding and better caching |
| **HTTP/2.0** | Multiplexing — multiple requests/responses over ONE connection simultaneously. Binary framing (not text). Header compression (HPACK). Server push |
| **HTTP/3.0** | Built on QUIC protocol (over UDP instead of TCP). Faster connection setup. Lower latency. Better packet loss handling. No head-of-line blocking |

> 💡 HTTP/1.1 is still the most widely used. HTTP/2 is increasingly common. HTTP/3 is the future.

---

## 4. What an HTTP Message Looks Like

There are two types of HTTP messages — requests (client → server) and responses (server → client).

### Request Message Structure
```
POST /api/users HTTP/1.1          ← Method + Resource URL + HTTP Version
Host: api.example.com             ← Headers start here
Content-Type: application/json    ←
Authorization: Bearer abc123      ←
Accept: application/json          ← Headers end here
                                  ← Blank line (signals headers are done)
{"name": "Pravan", "age": 22}     ← Request Body
```

### Response Message Structure
```
HTTP/1.1 201 Created              ← HTTP Version + Status Code + Status Text
Content-Type: application/json    ← Response Headers start
Cache-Control: max-age=10         ←
ETag: "abc123"                    ← Response Headers end
                                  ← Blank line
{"id": 1, "name": "Pravan"}       ← Response Body
```

### Components at a Glance
| Component | In Request | In Response |
|---|---|---|
| HTTP Version | ✅ | ✅ |
| Method (GET, POST...) | ✅ | ❌ |
| Resource URL | ✅ | ❌ |
| Status Code | ❌ | ✅ |
| Headers | ✅ | ✅ |
| Blank Line | ✅ | ✅ |
| Body | Sometimes (POST/PUT/PATCH) | Usually |

---

## 5. HTTP Headers — The Most Important Part

### What are Headers?
Headers are **key-value pairs** that carry metadata about the request or response.

### The Parcel Analogy
When you send a package, you write the recipient's address, phone number, and PIN code **on the outside** of the box — not inside it. Why? Because everyone handling the package (courier, delivery person) needs to read that info quickly without opening it.

HTTP headers work exactly the same way. Metadata about the message goes in headers, not the body — so intermediaries (proxies, load balancers, browsers) can read it without parsing the body.

---

### Types of Headers

#### 1. Request Headers (Client → Server)
Tell the server about the client's environment, preferences, and capabilities.

| Header | Example | Purpose |
|---|---|---|
| `User-Agent` | `Mozilla/5.0...` | Identifies what kind of client is making the request (browser, Postman, mobile app) |
| `Authorization` | `Bearer eyJ...` | Carries authentication credentials (JWT token, API key, etc.) |
| `Accept` | `application/json` | Tells server what format the client wants in response |
| `Accept-Language` | `en-US` | Preferred language for response content |
| `Accept-Encoding` | `gzip, deflate, br` | Compression formats the client can handle |

#### 2. General Headers (Both Request and Response)
Metadata about the message itself — not the content.

| Header | Purpose |
|---|---|
| `Date` | When the message was sent |
| `Cache-Control` | Caching instructions (no-cache, max-age=300, etc.) |
| `Connection` | Whether to keep the connection alive or close it |

#### 3. Representation Headers (About the Body)
Describe the content being transmitted — how to interpret the body.

| Header | Example | Purpose |
|---|---|---|
| `Content-Type` | `application/json` | Format of the body |
| `Content-Length` | `348` | Size of the body in bytes |
| `Content-Encoding` | `gzip` | Compression applied to the body |
| `ETag` | `"abc123"` | Unique identifier for the resource version (used in caching) |

#### 4. Security Headers (Response → Browser)
Tell the browser how to behave to protect the user.

| Header | What it prevents |
|---|---|
| `Strict-Transport-Security (HSTS)` | Forces HTTPS — prevents protocol downgrade attacks |
| `Content-Security-Policy (CSP)` | Restricts which scripts/styles/images can load — prevents XSS |
| `X-Frame-Options` | Prevents your page being embedded in iframes — prevents clickjacking |
| `X-Content-Type-Options` | Stops browser guessing MIME type — prevents MIME sniffing attacks |
| `Set-Cookie: HttpOnly; Secure` | Cookie inaccessible to JS, only sent over HTTPS |

---

### Two Key Properties of Headers

**1. Extensibility** — Headers can be added or customised without changing HTTP itself. Custom headers can be created for specific app needs (e.g., `X-Request-Id`, `X-Tenant-Id`).

**2. Remote Control** — Headers let the client send instructions to the server that influence behaviour:
- "I want JSON, not XML" → `Accept: application/json`
- "Cache this for 5 minutes" → `Cache-Control: max-age=300`
- "Here are my credentials" → `Authorization: Bearer ...`

---

## 6. HTTP Methods

HTTP methods define the **intent** of the request — what action the client wants to perform.

| Method | Intent | Has Body? | Idempotent? |
|---|---|---|---|
| `GET` | Fetch data (never modify anything) | ❌ | ✅ |
| `POST` | Create new data | ✅ | ❌ |
| `PUT` | Completely replace existing data | ✅ | ✅ |
| `PATCH` | Partially update data | ✅ | ✅ |
| `DELETE` | Remove a resource | ❌ | ✅ |
| `OPTIONS` | Ask server about its capabilities | ❌ | ✅ |

### PATCH vs PUT — A Common Confusion

```
User profile: { name: "Pravan", age: 22, email: "p@p.com" }

PATCH /users/1  { "name": "Pravann" }
→ Only name is updated. age and email stay the same.

PUT /users/1  { "name": "Pravann", "age": 22, "email": "p@p.com" }
→ Entire resource is REPLACED with whatever you send.
   If you forget to include email → email becomes null!
```

> 💡 Thumb rule: always use PATCH unless you specifically need full replacement. Many developers wrongly use PUT when they should use PATCH.

---

### Idempotent vs Non-Idempotent

**Idempotent** = calling the same request multiple times produces the same result.

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | ✅ | Fetching the same data 10 times = same data 10 times |
| `PUT` | ✅ | Replacing a resource 10 times with the same data = same final state |
| `DELETE` | ✅ | Deleting something that's already deleted = still deleted |
| `POST` | ❌ | Creating a note 10 times = 10 notes created (different result each time) |

---

## 7. CORS — Cross-Origin Resource Sharing

### The Same-Origin Policy
Browsers enforce a security rule: by default, a web page can only make requests to the **same domain** it was served from.

```
Frontend: example.com           ← can request from example.com
Backend:  api.example.com       ← BLOCKED by default! Different subdomain = different origin
```

**Origin = Protocol + Domain + Port**  
`http://localhost:5173` and `http://localhost:3000` are different origins (different port).

### What is CORS?
CORS (Cross-Origin Resource Sharing) is a mechanism that allows servers to explicitly **permit cross-origin requests** from specific domains.

Without it, browsers block any request where the client and server are on different origins.

---

### Flow 1 — Simple Request

Conditions for a simple request (no preflight needed):
- Method is GET, POST, or HEAD
- No non-standard headers (no `Authorization`, no custom headers)
- Content-Type is `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`

```
1. Browser sends request with Origin header:
   GET /data HTTP/1.1
   Origin: http://example.com      ← browser adds this automatically

2. Server checks if example.com is allowed

3a. If ALLOWED → server responds with:
    Access-Control-Allow-Origin: http://example.com
    (or Access-Control-Allow-Origin: *)
    → Browser lets the response through ✅

3b. If NOT ALLOWED → server responds WITHOUT that header
    → Browser BLOCKS the response ❌
    → You see CORS error in console
```

> ⚠️ The request still reached the server. CORS errors are a BROWSER enforcement. The server processed it — the browser just blocked you from reading the response.

---

### Flow 2 — Preflight Request

A preflight is required when ANY of these is true (it's either/or — one condition is enough):
1. Method is PUT, DELETE, PATCH (not GET/POST/HEAD)
2. Request includes non-simple headers like `Authorization` or custom headers
3. Content-Type is `application/json` (not a "simple" content type)

> 💡 In practice: **almost all API requests trigger a preflight** because we almost always use JSON and Authorization headers.

**How it works:**

```
Step 1 — Browser sends preflight (OPTIONS request):
   OPTIONS /api/users HTTP/1.1
   Host: api.example.com
   Origin: http://example.com
   Access-Control-Request-Method: PUT          ← "Do you allow PUT?"
   Access-Control-Request-Headers: Authorization, Content-Type

Step 2 — Server responds to preflight:
   HTTP/1.1 204 No Content                     ← No body, just headers
   Access-Control-Allow-Origin: http://example.com
   Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   Access-Control-Allow-Headers: Authorization, Content-Type
   Access-Control-Max-Age: 86400               ← Cache this for 24 hours

Step 3 — Browser checks all conditions:
   ✅ Origin allowed
   ✅ PUT method allowed
   ✅ Authorization and Content-Type headers allowed

Step 4 — Browser sends the ACTUAL request
   PUT /api/users/1 HTTP/1.1
   Authorization: Bearer ...
   Content-Type: application/json
   { "name": "Pravan" }

Step 5 — Server processes and responds normally
```

### Access-Control-Max-Age
This header tells the browser: "don't make another preflight for this route for X seconds."

Without it, the browser would fire a preflight before EVERY single API call — wasteful.

---

## 8. HTTP Status Codes

Status codes let clients instantly understand what happened — without parsing the response body.

### The Five Categories

| Range | Category | What it means |
|---|---|---|
| `1xx` | Informational | Request received, keep going |
| `2xx` | Success | Everything worked |
| `3xx` | Redirection | Go somewhere else |
| `4xx` | Client Error | You (the client) did something wrong |
| `5xx` | Server Error | We (the server) messed up |

---

### 1xx — Informational
Rarely used. Two worth knowing:

| Code | Name | When |
|---|---|---|
| `100` | Continue | Server got the headers, send the body (used in large uploads) |
| `101` | Switching Protocols | Upgrading from HTTP to WebSocket |

---

### 2xx — Success

| Code | Name | When to use |
|---|---|---|
| `200` | OK | Successful GET, PUT, PATCH — resource returned |
| `201` | Created | Successful POST — new resource was created |
| `204` | No Content | Successful DELETE, or preflight OPTIONS response — nothing to return |

---

### 3xx — Redirection

| Code | Name | When to use |
|---|---|---|
| `301` | Moved Permanently | Route permanently moved to new URL — use new URL forever |
| `302` | Found (Temp Redirect) | Temporarily at different URL — keep using original URL later |
| `304` | Not Modified | Cached version is still fresh — use it (no body sent) |

**Real example of 301:**
You renamed `/user` to `/person`. Old apps still use `/user`. You add a 301 redirect to forward them to `/person` so nothing breaks.

---

### 4xx — Client Errors (You'll deal with these daily as a backend engineer)

| Code | Name | When to use |
|---|---|---|
| `400` | Bad Request | Invalid data sent — wrong type, missing required field, illogical value |
| `401` | Unauthorized | Not authenticated — token missing, expired, or invalid |
| `403` | Forbidden | Authenticated but not permitted — user A trying to delete user B's data |
| `404` | Not Found | Resource doesn't exist — wrong URL or already deleted |
| `405` | Method Not Allowed | Wrong HTTP method — PUT to a route that only accepts GET |
| `409` | Conflict | Duplicate — creating a folder that already exists |
| `429` | Too Many Requests | Rate limit exceeded — slow down |

**401 vs 403 — Easy to confuse:**
```
401 → "I don't know who you are" (not logged in)
403 → "I know who you are, but you can't do this" (no permission)
```

---

### 5xx — Server Errors

| Code | Name | When |
|---|---|---|
| `500` | Internal Server Error | Unhandled exception, unexpected crash |
| `501` | Not Implemented | Feature planned but not built yet |
| `502` | Bad Gateway | Proxy (nginx) got invalid response from upstream server |
| `503` | Service Unavailable | Server down for maintenance or overloaded |
| `504` | Gateway Timeout | Proxy didn't get response from upstream within timeout |

---

## 9. HTTP Caching

### What is it?
Caching = storing a copy of the response so you don't have to re-download it if it hasn't changed.

**Benefits:**
- Faster load times for the client
- Less bandwidth used
- Less load on the server

---

### How it Works — The Three Key Headers

**Server sends these headers with the response:**

| Header | Example | Meaning |
|---|---|---|
| `Cache-Control` | `max-age=10` | Cache this response for 10 seconds |
| `ETag` | `"3141"` | A unique identifier (hash) of the current version of the resource |
| `Last-Modified` | `Wed, 20 May 2026...` | When the resource was last changed |

**Client sends these on subsequent requests:**

| Header | Example | Meaning |
|---|---|---|
| `If-None-Match` | `"3141"` | "If the ETag is different from this, send me the new version" |
| `If-Modified-Since` | `Wed, 20 May 2026...` | "If it changed after this time, send me the new version" |

---

### The Full Caching Flow

```
Step 1 — First request:
  Client: GET /api/resource
  Server: 200 OK
          Cache-Control: max-age=10
          ETag: "3141"
          Last-Modified: ...
          Body: { data: "..." }

Step 2 — Second request (within 10 seconds):
  Client: GET /api/resource
          If-None-Match: "3141"
          If-Modified-Since: ...

  Server checks: ETag matches? Last-Modified unchanged?
  → YES → Server: 304 Not Modified (NO body sent)
  → Client uses cached version ✅

Step 3 — Resource is updated on server:
  (Someone does a PUT/PATCH — server assigns new ETag "2943")

Step 4 — Client requests again with old ETag "3141":
  Server checks: ETag "3141" ≠ current "2943"
  → Server: 200 OK with new body + new ETag "2943"

Step 5 — Next request with ETag "2943":
  → 304 Not Modified again ✅
```

> 💡 304 responses have NO body — just headers. This saves bandwidth. The client already has the body from the last 200 response.

### Real-World Note
HTTP-based caching (ETags) is powerful but complex to manage correctly on the server. A missed ETag update = clients stuck with stale data.

Modern alternative: **React Query** (client-side) gives the client full control over when to refetch, at what interval, and stale time — more practical for most modern apps.

---

## 10. Content Negotiation

Content negotiation = client and server **agreeing on the best format** to exchange data.

The client uses headers to tell the server its preferences. The server responds in the best matching format.

### Three Types

| Type | Client Header | Example |
|---|---|---|
| **Media Type** | `Accept` | `Accept: application/json` or `Accept: application/xml` |
| **Language** | `Accept-Language` | `Accept-Language: es` (Spanish) or `en` (English) |
| **Encoding** | `Accept-Encoding` | `Accept-Encoding: gzip, deflate, br` |

### How it Works
```
Client: Accept: application/xml
        Accept-Language: es

Server: Sees these headers → responds in XML format, in Spanish

Same endpoint, different response based on client preference ✅
```

---

## 11. HTTP Compression

Why compress? Large responses waste bandwidth and slow down clients.

**Demo from the video:**
- A JSON file with 11,000 entries
- Without compression: **26 MB**
- With gzip compression: **3.8 MB**

That's a **7x reduction in size** for the same data.

### How It Works
```
Client says: Accept-Encoding: gzip, deflate, br
Server compresses response → Content-Encoding: gzip
Browser automatically decompresses it → client gets original data
```

You as a backend engineer typically just enable compression middleware. The browser handles decompression automatically.

Most common compression formats:
- **gzip** — most widely supported, good compression
- **deflate** — older, less common
- **br (Brotli)** — newer, better compression than gzip, increasingly used

---

## 12. Persistent Connections

### HTTP/1.0 Problem
Every request opened a new TCP connection. TCP connection setup (3-way handshake) takes time. For a page with 20 assets = 20 separate connections. Very slow.

### HTTP/1.1 Solution — Persistent Connections
One TCP connection stays open and handles multiple request-response cycles.

```
HTTP/1.0:
[TCP open] → Request 1 → Response 1 → [TCP close]
[TCP open] → Request 2 → Response 2 → [TCP close]
[TCP open] → Request 3 → Response 3 → [TCP close]

HTTP/1.1:
[TCP open]
→ Request 1 → Response 1
→ Request 2 → Response 2
→ Request 3 → Response 3
[TCP close when done]
```

### The `Connection: keep-alive` Header
- In HTTP/1.1, persistent connections are **on by default**
- The `Connection: keep-alive` header can explicitly request it (for older clients)
- Options: `timeout=5` (close after 5s idle), `max=100` (close after 100 requests)
- `Connection: close` explicitly closes after the response — HTTP/1.0 default behaviour

You usually don't need to worry about this — defaults work fine.

---

## 13. Handling Large Requests and Responses

### Sending Large Files to Server — Multipart Requests

When uploading files (images, videos, PDFs), you use **multipart form data** — not JSON.

Why? Binary file data needs to be sent in **parts** with delimiters separating them.

```
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4

------WebKitFormBoundary7MA4     ← delimiter (start)
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

[binary data of the file...]
------WebKitFormBoundary7MA4--   ← delimiter (end)
```

The `boundary` parameter defines the delimiter string that separates parts of the data.

> 💡 For file uploads, always use `multipart/form-data`. Never try to base64-encode files into JSON — it's 33% larger and much slower.

---

### Receiving Large Responses — Streaming / Chunked Transfer

When the server has a very large response (e.g., a 11,000-entry dataset or a live event stream), it doesn't wait until everything is ready — it **streams chunks** progressively.

```
Response headers:
  Content-Type: text/event-stream    ← streaming format
  Connection: keep-alive             ← keep connection open

Server sends:
  [chunk 1]... → client appends it
  [chunk 2]... → client appends it
  [chunk 3]... → client appends it
  ...until complete
```

The client keeps the connection open and processes each chunk as it arrives. This is how real-time data feeds, log streaming, and LLM token streaming work.

---

## 14. SSL, TLS, and HTTPS

Three terms you'll hear constantly:

| Term | What it is |
|---|---|
| **SSL** | Original encryption protocol — now outdated due to security vulnerabilities |
| **TLS** | Modern replacement for SSL — what's actually used today (TLS 1.3 is current recommended) |
| **HTTPS** | HTTP + TLS encryption = HTTP with a security layer |

### How TLS Works (Simplified)
```
1. Client connects to server
2. Server sends its TLS certificate (proves it's really who it says it is)
3. Client verifies certificate
4. Both agree on encryption keys
5. All subsequent communication is encrypted

Without TLS: passwords, tokens, credit cards travel as plain text → anyone on the network can read them
With TLS: everything is encrypted → unreadable to anyone intercepting
```

> 💡 In production, always use HTTPS. Never send credentials over plain HTTP.

As a backend engineer, you usually don't implement TLS yourself — it's handled by infrastructure (nginx, load balancer, cloud provider). But you need to know what it is and why it matters.

---

## Complete HTTP Request Lifecycle — The Full Picture

```
User types URL in browser
  ↓
Browser builds HTTP request message
  ↓
Browser checks: Is this cross-origin? → If yes, send preflight OPTIONS first
  ↓
TCP connection established (3-way handshake)
  ↓
TLS handshake (if HTTPS)
  ↓
HTTP request sent (method + URL + headers + body)
  ↓
Server receives, parses headers and body
  ↓
Server processes request (routing → middleware → handler → service → DB)
  ↓
Server builds response (status code + headers + body)
  ↓
Response sent back over same TCP connection
  ↓
Browser receives response
  ↓
Browser checks: CORS headers present? Cache valid?
  ↓
Response handed to JavaScript / rendered
```

---

## Quick Reference — Most Common Headers

```
Request Headers:
  Authorization: Bearer <token>
  Content-Type: application/json
  Accept: application/json
  Accept-Encoding: gzip, deflate, br
  Accept-Language: en-US
  Origin: https://example.com

Response Headers:
  Content-Type: application/json
  Cache-Control: max-age=300
  ETag: "abc123"
  Access-Control-Allow-Origin: https://example.com
  Strict-Transport-Security: max-age=31536000
  Content-Encoding: gzip
```

---

## Quick Reference — Status Codes Cheat Sheet

```
2xx Success:
  200 OK           → successful GET/PUT/PATCH
  201 Created      → successful POST
  204 No Content   → successful DELETE / preflight

3xx Redirect:
  301 Moved Permanently
  302 Found (temporary)
  304 Not Modified → use cached version

4xx Client Errors:
  400 Bad Request       → invalid data
  401 Unauthorized      → not logged in / bad token
  403 Forbidden         → logged in but no permission
  404 Not Found         → resource doesn't exist
  405 Method Not Allowed
  409 Conflict          → duplicate
  429 Too Many Requests → rate limited

5xx Server Errors:
  500 Internal Server Error → unexpected crash
  502 Bad Gateway
  503 Service Unavailable
  504 Gateway Timeout
```

---

## Summary — What to Remember

| Concept | Key Takeaway |
|---|---|
| **Statelessness** | Server has no memory. Client must send all context every time. |
| **Client-Server** | Client always initiates. Server always responds. |
| **TCP** | HTTP uses TCP for reliable delivery. |
| **Message Structure** | Method + URL + Version + Headers + blank line + Body |
| **Headers** | Key-value metadata. Four types: Request, General, Representation, Security. |
| **Methods** | GET/POST/PUT/PATCH/DELETE — each has a clear semantic intent. |
| **Idempotency** | GET, PUT, DELETE are idempotent. POST is not. |
| **CORS** | Browser policy. Server must explicitly allow cross-origin requests. |
| **Preflight** | OPTIONS request sent before PUT/DELETE/JSON requests to check permissions. |
| **Status Codes** | Universal language. 2xx=success, 4xx=client error, 5xx=server error. |
| **Caching** | ETags + Cache-Control + 304 responses = skip re-downloading unchanged data. |
| **Content Negotiation** | Client declares preferences via Accept headers. Server responds accordingly. |
| **Compression** | gzip reduces response size by up to 7x. Accept-Encoding/Content-Encoding headers. |
| **Persistent Connections** | HTTP/1.1 reuses TCP connections for multiple requests. |
| **Multipart** | For file uploads — binary data sent in parts with boundary delimiter. |
| **Streaming** | Large responses streamed in chunks via text/event-stream. |
| **TLS/HTTPS** | Encrypts all data in transit. Always use in production. |

---

*Next lecture → Routing — How URLs map to server-side logic.*
