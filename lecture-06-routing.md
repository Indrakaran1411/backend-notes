# Lecture 6 — What is Routing in Backend? How Requests Find Their Way Home
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=SubuU1iOC2s)  
> Series: Backend from First Principles

---

## What This Lecture Is About

In the last lecture we covered HTTP methods — the **"what"** of a request (what do you want to do?). This lecture covers routing — the **"where"** of a request (where do you want to do it?).

Together, HTTP method + route = a complete instruction to the server.

---

## 1. What is Routing?

Routing is the mechanism that **maps a URL path + HTTP method to a specific piece of server-side logic** (called a Handler).

### The Simple Formula
```
HTTP Method  +  URL Path  =  Unique Route  →  Handler
   (what)         (where)
```

### Real Example
```
GET  /api/books  →  "Fetch all books" handler
POST /api/books  →  "Create a new book" handler
```

Both use the same path `/api/books` but the **method makes them different routes**. They never clash.

### What the Server Does Internally
When a request arrives, the server does two checks:
1. What is the **method**? (GET, POST, PUT...)
2. What is the **path**? (/api/books, /api/users/123...)

It combines these two and looks up the matching handler in its **routing table** (a list of all registered routes). The matched handler runs, executes business logic, hits the database, and returns a response.

> 💡 Think of routing like a post office directory. The method is the type of mail (letter, parcel, courier). The path is the address. Together they determine exactly which desk handles it.

---

## 2. Static Routes

The simplest type of route. The path is **fixed and never changes**.

```
GET  /api/books     → always fetches books
POST /api/books     → always creates a book
GET  /api/products  → always fetches products
```

**Why "static"?**  
The string `/api/books` is constant. Every request to this path looks exactly the same. No variables, no dynamic parts.

### Characteristics
- Same path → always goes to the same handler
- Always returns the same *type* of data (though the actual data may differ)
- Simple to define, simple to understand

---

## 3. Dynamic Routes (Path Parameters)

Sometimes you need to target **a specific resource** — not all books, but book with ID 42. That's where dynamic routes come in.

### Example
```
GET /api/users/123     → fetch user with ID 123
GET /api/users/456     → fetch user with ID 456
GET /api/users/999     → fetch user with ID 999
```

These are all different requests but they all go to the **same handler** — because the server registers the route pattern as:

```
GET /api/users/:id     → "fetch one user" handler
```

The `:id` is a **placeholder** — it matches any string in that position. When the request comes in, the server extracts the actual value (`123`, `456`, `999`) and makes it available to the handler.

### The `:id` Convention
You'll see this across every language and framework:

```
Node.js (Express):    GET /api/users/:id
Go (Chi/Gin):         GET /api/users/{id}
Python (FastAPI):     GET /api/users/{id}
Ruby (Rails):         GET /api/users/:id
```

The exact symbol differs but the concept is identical: a named slot that captures a dynamic value from the URL.

### Important: Route params are always strings
Even if you send `/api/users/123`, the `123` is received as the **string** `"123"`, not the number `123`. You must convert it in your handler if you need a number.

### Why Dynamic Routes?
They make APIs human-readable and semantically meaningful:

```
GET /api/users/123         → "Give me the user whose ID is 123"
GET /api/users/123/posts   → "Give me the posts belonging to user 123"
```

You can read the URL and immediately understand what it's asking for. This is the core idea behind RESTful API design.

---

## 4. Path Parameters vs Query Parameters

These are the two ways to pass data to the server in a GET request. They look similar but serve very different purposes.

---

### Path Parameters
- Part of the URL path itself (after a `/`)
- Used to **identify a specific resource**
- Semantically meaningful — they're part of the resource address

```
GET /api/users/123
           ↑
       path parameter
       (identifies WHICH user)
```

---

### Query Parameters
- Appended after a `?` in the URL
- Key-value pairs separated by `&`
- Used to **modify or filter** the request — not identify the resource

```
GET /api/books?page=2&limit=10&sort=title&order=asc
              ↑
        query parameters
        (HOW to return the books, not WHICH books)
```

---

### Why Not Put Everything in Path Parameters?

Let's say you want to search for books. Could you do:
```
GET /api/search/some+value+user+typed
```
Technically yes. But:
- It's ugly and hard to maintain
- It defeats the semantic purpose of path parameters (identifying resources)
- Adding multiple filters becomes impossible to read: `/api/books/page2/limit10/sortTitle/asc`

Query parameters solve this cleanly:
```
GET /api/books?page=2&limit=10&sort=title&order=asc
```

**Rule of thumb:**
- **Path param** → "Which resource?" (user ID, product ID, post ID)
- **Query param** → "How do you want it?" (filters, pagination, sorting, search terms)

---

### Pagination Example (Common Real-World Use)

A typical paginated API response:

**Request:**
```
GET /api/books?page=2&limit=20
```

**Response:**
```json
{
  "data": [...20 books...],
  "metadata": {
    "total": 100,
    "currentPage": 2,
    "totalPages": 5,
    "limit": 20
  }
}
```

The client uses the metadata to know:
- There are 100 books total
- I'm on page 2
- There are 5 pages total
- To get page 3: `GET /api/books?page=3&limit=20`

For the first request, the client didn't need to send `page=1` — the server defaults to page 1 if not specified.

---

### Common Query Parameter Uses
| Use Case | Example |
|---|---|
| Pagination | `?page=2&limit=20` |
| Search | `?query=javascript` |
| Sorting | `?sort=price&order=asc` |
| Filtering | `?category=fiction&inStock=true` |
| Date range | `?from=2024-01-01&to=2024-12-31` |

---

## 5. Nested Routes

Nested routes reflect **hierarchical relationships** between resources. They're not a new type of route — they're a pattern you'll see everywhere.

### The Pattern
```
/api/users                    → all users
/api/users/:userId            → one specific user
/api/users/:userId/posts      → all posts of that user
/api/users/:userId/posts/:postId → one specific post of that user
```

### Why Nesting?

Each level adds a layer of meaning:

```
GET /api/users/123
→ "Give me user 123"

GET /api/users/123/posts
→ "Give me ALL posts belonging to user 123"

GET /api/users/123/posts/456
→ "Give me the SPECIFIC post with ID 456 that belongs to user 123"
```

Each of these is a **different route** in the server — each mapped to a different handler. Even though they share a prefix (`/api/users/123`), they are completely separate routes.

### In the Server (Conceptually)
```
GET  /api/users            → handler: getAllUsers
GET  /api/users/:id        → handler: getUserById
GET  /api/users/:id/posts  → handler: getPostsByUser
GET  /api/users/:id/posts/:postId → handler: getPostById
```

### How Deep Should You Nest?
General industry rule: **don't nest more than 2-3 levels deep**. Beyond that it becomes hard to read and maintain.

```
✅ Good:
GET /api/users/:id/posts/:postId

❌ Too deep (avoid):
GET /api/users/:id/posts/:postId/comments/:commentId/likes/:likeId
```

If you find yourself nesting this deep, consider flattening the API:
```
GET /api/comments/:commentId/likes
```

---

## 6. Route Versioning and Deprecation

### The Problem
You have a working API that returns data in a certain format. Your frontend team depends on it. Now requirements change and you need to change the response format. If you just change it, you break all existing clients.

### The Solution — Version Your Routes
```
GET /api/v1/products  → old format
GET /api/v2/products  → new format
```

### Real Example from the Lecture

**V1 response:**
```json
{
  "data": [
    { "id": 1, "name": "Book A", "price": 299 }
  ]
}
```

**V2 response (changed field name from `name` to `title`):**
```json
{
  "data": [
    { "id": 1, "title": "Book A", "price": 299 }
  ]
}
```

This seems trivial but in real systems, version changes can include:
- Completely different response structures
- New required/optional fields
- Changes to authentication mechanisms
- Changes to pagination format
- Different error response shapes

### The Deprecation Workflow
```
Step 1: Release V2 alongside V1
         → Both /v1/products and /v2/products work

Step 2: Announce to frontend/mobile teams:
         "V1 is deprecated. Migrate to V2 by [date]."

Step 3: Give teams a migration window (weeks or months)

Step 4: Remove V1

Step 5: Optionally rename V2 → V1 (or keep V2 prefix)
```

### Benefits of Versioning
- **No breaking changes** — old clients keep working during migration
- **Clear intent** — V1 vs V2 clearly communicates there's a new version
- **Controlled rollout** — you decide when to retire old versions
- **Multiple clients** — web app uses V2, older mobile app still uses V1

### Versioning Strategies
| Strategy | Example | Notes |
|---|---|---|
| **URI versioning** (most common) | `/api/v1/users` | Clear, visible, easy to route |
| **Header versioning** | `API-Version: 1` in header | Cleaner URLs, harder to test in browser |
| **Query param versioning** | `/api/users?version=1` | Simple but can conflict with other query params |
| **Media type versioning** | `Accept: application/vnd.api.v1+json` | Most RESTful, most complex |

---

## 7. Catch-All Routes

### The Problem
What happens when a client requests a route your server doesn't handle?

```
GET /api/v3/products   → Server has no handler for this
```

Without a catch-all, the server returns an empty or confusing response.

### The Solution — Wildcard Route
```
/*   → catch-all handler → "Route not found"
```

This is registered **last** — after all specific routes. Any request that doesn't match any known route falls through to here.

```
Server routing logic (order matters!):
  1. GET  /api/v1/products  → products handler ✅
  2. GET  /api/v2/products  → products v2 handler ✅
  3. GET  /api/users/:id    → user handler ✅
  ...all other routes...
  999. /*                   → catch-all handler → "Route not found" ✅
```

### What the Catch-All Returns
```json
{
  "error": "NOT_FOUND",
  "message": "The route /api/v3/products does not exist",
  "statusCode": 404
}
```

This is far better than:
- An empty response
- A server crash
- An HTML error page in a JSON API

> 💡 Always register a catch-all route. It improves the developer experience for anyone consuming your API.

---

## 8. How the Server Matches Routes — Under the Hood

When a request arrives, the server runs through its routing table in order:

```
Incoming: GET /api/users/123

Routing table:
  GET  /api/books        → no match
  POST /api/books        → no match (wrong method)
  GET  /api/users        → no match (path doesn't match exactly)
  GET  /api/users/:id    → ✅ MATCH! id = "123"
  ...
```

The server stops at the first match. This is why **order of route registration matters** — more specific routes should be registered before more generic ones.

```
⚠️ Wrong order:
  GET /api/users/*      → catches everything first!
  GET /api/users/profile → never reached

✅ Correct order:
  GET /api/users/profile  → specific route first
  GET /api/users/:id      → dynamic route second
  GET /api/users/*        → wildcard last
```

---

## 9. Route Grouping (Bonus Concept)

Related routes are often grouped together with a shared prefix. This is done for:
- **Organisation** — all user routes together, all product routes together
- **Shared middleware** — apply auth middleware only to a group of routes
- **Versioning** — group all v1 routes under `/api/v1`

```
Router: /api/v1
  GET    /users           → list users
  POST   /users           → create user
  GET    /users/:id       → get user
  PATCH  /users/:id       → update user
  DELETE /users/:id       → delete user

Router: /api/v2
  GET    /users           → list users (new format)
  POST   /users           → create user (new format)
```

---

## Complete Mental Model — All Route Types Together

```
Static Route:
  GET /api/books
      └── fixed path, always the same handler

Dynamic Route:
  GET /api/users/:id
                 └── :id captures any value (123, 456, etc.)

Nested Route:
  GET /api/users/:userId/posts/:postId
                  └── userId    └── postId (two dynamic params)

Query Parameters:
  GET /api/books?page=2&limit=10&sort=price
                 └─────────────────────────── not part of the route
                                               passed separately as key-value pairs

Versioned Routes:
  GET /api/v1/products   → old format
  GET /api/v2/products   → new format

Catch-All Route:
  /*   → handles anything that didn't match above → 404
```

---

## Quick Reference — Path Param vs Query Param

| | Path Parameter | Query Parameter |
|---|---|---|
| **Location** | In the URL path after `/` | After `?` in the URL |
| **Syntax** | `/users/:id` or `/users/{id}` | `?key=value&key2=value2` |
| **Purpose** | Identify a specific resource | Filter, sort, paginate, search |
| **Required?** | Yes (part of the route) | Usually optional |
| **Example** | `/api/users/123` | `/api/books?page=2&sort=price` |
| **Semantic meaning** | "Which one?" | "How do you want it?" |

---

## Summary

| Concept | What it is | Example |
|---|---|---|
| **Routing** | Maps method + path → handler | `GET /api/users` → list users |
| **Static Route** | Fixed path, no variables | `/api/books` |
| **Dynamic Route** | Path with variable slot | `/api/users/:id` |
| **Path Parameter** | Dynamic value in the path | `/api/users/123` → id = "123" |
| **Query Parameter** | Key-value pairs after `?` | `?page=2&limit=10` |
| **Nested Route** | Hierarchical resource path | `/api/users/:id/posts/:postId` |
| **Route Versioning** | Multiple API versions | `/api/v1/` and `/api/v2/` |
| **Deprecation** | Retire old version with notice | Announce → migrate window → remove |
| **Catch-All Route** | Handle unknown routes | `/* → 404 handler` |

---

## One-Line Takeaways

> HTTP method = **what** you want to do. Route = **where** you want to do it. Together they form a unique address to a specific piece of server logic.

> Path params = "which resource". Query params = "how do you want it".

> Always version your APIs before making breaking changes. Always register a catch-all route.

---

*Next lecture → Serialisation and Deserialisation — How data transforms before and after travelling over the network.*
