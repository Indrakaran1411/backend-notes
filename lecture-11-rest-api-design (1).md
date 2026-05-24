# Lecture 11 — Complete REST API Design
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=RG6q57DwV8Y)  
> Series: Backend from First Principles

---

## What This Lecture Is About

API design is something you'll spend a large chunk of your career thinking about. This lecture covers REST API design end-to-end — the history, the theory, and a full practical walkthrough of designing APIs for a real project management platform (organisations, projects, tasks). By the end you'll have a clear, consistent framework for designing any API.

---

## 1. Why REST API Design Confuses People

Even experienced developers get confused by questions like:
- Should the URL segment be `/book` or `/books`?
- Should I use `PUT` or `PATCH` for updates?
- Which HTTP method for a custom action that doesn't fit CRUD?
- What status code do I return for which scenario?

The reason: when these standards were being created, the web looked very different (Multi-Page Applications, not SPAs). The standards were designed for that era, and applying them to modern single-page apps and JSON-heavy APIs requires some interpretation.

**The goal of this lecture:** Extract clear, consistent rules from existing standards so you never have to guess again.

---

## 2. Where REST Came From — Brief History

**1990 — Tim Berners-Lee** creates the World Wide Web and invents in one year:
- URI (Uniform Resource Identifier)
- HTTP protocol
- HTML
- First web server
- First web browser

**The problem:** The web grew exponentially. The original design wasn't built to scale to millions of users.

**1993 — Roy Fielding** (co-founder of Apache HTTP server) proposed a set of architectural constraints to make the web scalable. These constraints are:

| Constraint | What it means |
|---|---|
| **Client-Server** | UI concerns separated from data/business logic. Frontend and backend evolve independently. |
| **Uniform Interface** | Standardised way for all components to communicate. Includes: resource identification, manipulation through representations, self-descriptive messages, HATEOAS. |
| **Layered System** | Each layer only interacts with the immediate layer below. Enables load balancers, proxies, caches — without affecting core functionality. |
| **Cacheable** | Responses must be labelled cacheable or non-cacheable. Reduces server load. |
| **Stateless** | Every request contains all information needed to process it. Server remembers nothing between requests. Any server can handle any request — enables horizontal scaling. |
| **Code on Demand** (optional) | Servers can transfer executable code (e.g., JavaScript) to the client. Rarely used explicitly. |

**2000 — Roy Fielding** named this architectural style in his PhD dissertation: **REST** (Representational State Transfer). This is where "REST API" comes from.

---

## 3. What "REST" Actually Means

| Part | Meaning |
|---|---|
| **Representational** | Resources can be represented in different formats (JSON, XML, HTML) depending on who's asking. Same resource, different representation for different clients. |
| **State** | The current condition/attributes of a resource. A shopping cart's state = its items, quantities, total price. This state can be transferred between client and server. |
| **Transfer** | Movement of resource representations between client and server via HTTP (GET, POST, PUT, DELETE, PATCH). |

Combined: REST is an architectural style where resources have representations, those representations have state, and client-server communication transfers that state via HTTP.

---

## 4. Anatomy of a REST API URL

```
https://api.example.com/v1/books/harry-potter?page=1&limit=10
  ↑         ↑       ↑    ↑   ↑        ↑               ↑
scheme   subdomain domain version resource   slug    query params
```

| Part | Rule |
|---|---|
| **Scheme** | Always `https` in production |
| **Subdomain** | Use `api.` prefix for API endpoints |
| **Versioning** | `/v1/`, `/v2/` in the path — standard practice |
| **Resource** | Always **plural noun** in lowercase — `/books` not `/book` |
| **Slug** | Lowercase, hyphens instead of spaces/underscores — `harry-potter` not `Harry_Potter` |
| **Query params** | For filters, pagination, sorting — after `?` |

### Why Plural?
Even when fetching a single book, the resource is still called `books`:
```
GET /api/v1/books        → list all books
GET /api/v1/books/42     → get book with ID 42 (still plural!)
```
The resource name represents the **collection**. You're accessing one item from the `books` collection, so it stays plural.

### Forward Slash = Hierarchy
Every `/` in a path represents a hierarchical relationship:
```
/api/users/123/posts/456
     ↑      ↑    ↑    ↑
  collection  ID  sub-collection  sub-ID

Meaning: "Post 456 that belongs to User 123"
```

---

## 5. Idempotency — Which Method Does What

**Idempotency** = performing the same operation multiple times has the same effect as performing it once. The server state doesn't keep changing with repeated calls.

| Method | Idempotent? | Why | Use for |
|---|---|---|---|
| `GET` | ✅ Yes | Fetching never changes server state. Fetch 1000 times = same result. | Read data |
| `PUT` | ✅ Yes | Replacing resource with same payload → same final state. | Full resource replacement |
| `PATCH` | ✅ Yes | Updating name from A→B. Calling again B→B. Same state. | Partial update |
| `DELETE` | ✅ Yes | Deleting something already deleted → resource still gone. Second call gets 404 but causes no new change. | Delete resource |
| `POST` | ❌ No | Every call creates a new resource. 1000 POST calls = 1000 new entries. | Create resource, custom actions |

### POST — The Open-Ended Method
POST is not just for creating resources. In REST spec, POST is **open-ended** — it's the catch-all for anything that doesn't fit GET/PUT/PATCH/DELETE.

This means:
- Create resource → POST
- Custom actions (archive, clone, send email) → POST

---

## 6. PUT vs PATCH — The Most Confused Pair

```
User resource: { id: 1, name: "Pravan", email: "p@p.com", role: "admin" }

PATCH /users/1   { "name": "Pravann" }
→ Only name changes. email and role stay as-is. ← PARTIAL update

PUT /users/1     { "name": "Pravann", "email": "p@p.com", "role": "admin" }
→ Entire resource REPLACED with this payload.
   If you forget to include "role" → role becomes null! ← FULL replacement
```

**Thumb rule:** Use `PATCH` almost always. Use `PUT` only when you specifically need to replace the entire resource representation. In modern JSON-based APIs, partial updates (PATCH) are far more common.

---

## 7. Identifying Resources — Where to Start

Before writing any code, before designing any routes, start with the **wireframes or product requirements**.

**Process:**
1. Go through Figma designs / wireframes / client discussions
2. Find all the **nouns** — these are your resources
3. Note them down
4. Design DB schema for them
5. Then design API interface

**Example — Project Management Platform (like Jira/Linear):**

| What you see in the wireframe | Resource (noun) |
|---|---|
| "Create an organization" | `organizations` |
| "Add users to org" | `users` |
| "Create a project" | `projects` |
| "Create tasks in a project" | `tasks` |
| "Organize tasks with tags" | `tags` |

These nouns = your API resources.

---

## 8. Designing the API Interface — Full Walkthrough

The workflow:
```
Wireframes → Identify resources → DB Schema → API Interface Design
```

**Rule:** Design the interface BEFORE coding. Use tools like Insomnia, Postman, or Swagger to design the API shape. Language-agnostic. You're designing contracts, not implementations.

For this walkthrough, we'll design APIs for three resources:
- `organizations`
- `projects`
- `tasks` (same patterns — covered by understanding the first two)

---

### Organisation APIs

#### List All Organisations
```
GET /organizations
```
Returns a **paginated** list.

**Response (200 OK):**
```json
{
  "data": [...organisations...],
  "total": 50,
  "page": 1,
  "totalPages": 5
}
```

---

#### Create Organisation
```
POST /organizations

Body:
{
  "name": "Acme Corp",
  "status": "active",
  "description": "Some description"
}
```

**What NOT to include in the payload:** `id`, `createdAt`, `updatedAt` — these are server-handled fields. Never ask the client to send them.

**Response (201 Created):**
```json
{
  "id": "uuid-here",
  "name": "Acme Corp",
  "status": "active",
  "description": "Some description",
  "createdAt": "2024-09-23T...",
  "updatedAt": "2024-09-23T..."
}
```

> 💡 Note: same URL `/organizations` for both GET (list) and POST (create). The HTTP method differentiates them.

---

#### Get Single Organisation
```
GET /organizations/:id
```
**Response (200 OK):** The organisation object.

**Response (404 Not Found):** If ID doesn't exist.
```json
{ "error": "NOT_FOUND", "message": "Organisation not found" }
```

---

#### Update Organisation
```
PATCH /organizations/:id

Body (partial — only fields you want to update):
{
  "status": "active"
}
```
**Response (200 OK):** Updated organisation object.

---

#### Delete Organisation
```
DELETE /organizations/:id
```
**Response (204 No Content):** Empty body. Success confirmed by 204.

After deletion: `GET /organizations/:id` → 404 Not Found.

---

#### Custom Action — Archive Organisation

**Why not just PATCH status to "archived"?**  
Archiving is a complex operation with side effects:
- Delete or archive all projects under the org
- Send notification emails to all org members
- Archive all tasks in those projects
- Maybe trigger billing changes

This is not a simple field update — it's a **custom action**. Use POST.

```
POST /organizations/:id/archive
```
No body needed (or optional override fields).

**Response (200 OK):** Updated organisation with `status: "archived"`.

> 💡 Custom action URL pattern: `/{resource}/{id}/{action-name}`

---

### Project APIs (Applying the Same Patterns)

```
POST   /projects                    → Create project (201)
GET    /projects                    → List projects (200, paginated)
GET    /projects/:id                → Get single project (200)
PATCH  /projects/:id                → Update project (200)
DELETE /projects/:id                → Delete project (204)
POST   /projects/:id/clone          → Clone project (201 — creates new resource)
```

**Create Project payload:**
```json
{
  "name": "Project Alpha",
  "organizationId": "org-uuid",
  "status": "planned",
  "description": "Some description"
}
```

---

## 9. Pagination — Theory and Implementation

### Why Pagination?
- A database can have millions of records
- Returning all of them at once = massive response payload
- JSON serialisation of thousands of objects is slow
- Client gets overwhelmed, UI is slow

### How it Works
Return a **portion** (page) of data at a time. Client asks for more pages as needed.

**Query parameters:**
```
GET /organizations?page=2&limit=10
```

**Server defaults (critical — never require the client to send obvious params):**
- If `page` not sent → default to `1`
- If `limit` not sent → default to `10` or `20`

**Response shape:**
```json
{
  "data": [...10 organisations...],
  "total": 50,        ← total records in DB (for all pages)
  "page": 2,          ← current page being returned
  "totalPages": 5     ← total pages based on limit
}
```

**totalPages calculation:**
```
totalPages = Math.ceil(total / limit)
50 records ÷ limit 10 = 5 pages
```

**Frontend uses `totalPages` to:**
- Know when to stop fetching (infinite scroll)
- Show "Page 2 of 5" UI
- Disable "Next" button on last page

---

## 10. Sorting

```
GET /organizations?sortBy=name&sortOrder=asc
```

| Parameter | Default | Options |
|---|---|---|
| `sortBy` | `createdAt` | Any field: `name`, `status`, `id`, etc. |
| `sortOrder` | `desc` | `asc` or `desc` |

**Server defaults for sorting:**
- Default sort field: `createdAt`
- Default sort order: `desc` (newest first — most natural for list views)

**Why always have a default sort?** Without it, database returns rows in random order — every API call could return data in a different sequence. Always sort explicitly.

---

## 11. Filtering

```
GET /organizations?status=active
GET /organizations?status=archived
GET /organizations?name=Acme
GET /organizations?status=active&name=Acme   ← multiple filters
```

When no matching results:
```json
{
  "data": [],
  "total": 0,
  "page": 1,
  "totalPages": 0
}
```

**Status code: 200** — NOT 404.

> 💡 Critical rule: In a **list API**, never return 404 for empty results. 404 is only for when a **specific resource** is requested by ID and doesn't exist. An empty list is a valid successful response.

---

## 12. Status Codes — The Complete Guide for API Design

| Scenario | Code | Notes |
|---|---|---|
| Successful GET / PATCH / PUT / custom action | `200 OK` | Default success |
| Resource created (POST) | `201 Created` | Return the newly created object |
| Successful DELETE | `204 No Content` | Empty body |
| Resource not found (GET by ID, PATCH, DELETE) | `404 Not Found` | Only for single resource by ID |
| List API with no results | `200 OK` | Return `{ data: [], total: 0 }` |
| Custom action that creates something (clone) | `201 Created` | Because a new resource was created |
| Custom action that doesn't create | `200 OK` | e.g., archive — updates but doesn't create |
| Validation error | `400 Bad Request` | Missing/invalid fields |
| Not authenticated | `401 Unauthorized` | No/invalid token |
| Not permitted | `403 Forbidden` | Logged in but no access |
| Conflict (duplicate) | `409 Conflict` | e.g., duplicate name with unique constraint |

> ⚠️ Don't blindly assume POST → 201. A POST for a custom action (archive) returns 200. A POST for cloning returns 201 because it creates a new resource. The code depends on what the server actually does.

---

## 13. Custom Actions — The Pattern

When an operation doesn't fit any CRUD category:

```
POST /{resource}/{id}/{action}

Examples:
POST /organizations/:id/archive
POST /projects/:id/clone
POST /users/:id/deactivate
POST /invoices/:id/send
POST /reports/:id/generate
```

**Signs that something is a custom action (not just a PATCH):**
- The operation has multiple side effects beyond updating a field
- Other records are affected
- Emails/notifications are triggered
- The "update" is really an irreversible state transition

---

## 14. Consistency — The Most Important Principle

This is what separates a good API from a frustrating one.

### Naming Conventions
```
✅ Consistent:
  Body field: "description"     (in create-org)
  Body field: "description"     (in create-project)

❌ Inconsistent:
  Body field: "description"     (in create-org)
  Body field: "desc"            (in create-project)
```

### Route Patterns
```
✅ Consistent:
  GET    /organizations         → list
  POST   /organizations         → create
  GET    /organizations/:id     → get one
  PATCH  /organizations/:id     → update
  DELETE /organizations/:id     → delete

  GET    /projects              → list (same pattern!)
  POST   /projects              → create
  ...

❌ Inconsistent:
  GET    /organizations         → list
  POST   /org                   → create (singular? different resource name?)
```

### Why Consistency Matters
When a frontend engineer integrates your `organizations` API, they make assumptions about how your `projects` API works. If you're consistent, those assumptions are correct. If you're not, they'll hit errors, file bug reports, and waste time.

**Consistency reduces integration time, bugs, and confusion — for everyone.**

---

## 15. Sane Defaults

Your API should never make the client send "obvious" information. Set intelligent defaults:

| Scenario | Sane Default |
|---|---|
| Client doesn't send `page` | Default: `1` |
| Client doesn't send `limit` | Default: `10` or `20` |
| Client doesn't send `sortBy` | Default: `createdAt` |
| Client doesn't send `sortOrder` | Default: `desc` |
| Client doesn't send `status` when creating | Default: `"active"` (if business logic allows) |
| Optional description field | Allow creating without it |

> 💡 Only require from the client what is absolutely necessary. Everything else should have a sane default.

---

## 16. Other Best Practices

### Always Use camelCase in JSON
```json
✅ Good:
{
  "organizationId": "abc",
  "createdAt": "2024-09-23",
  "totalPages": 5
}

❌ Bad:
{
  "organization_id": "abc",
  "created_at": "2024-09-23",
  "total_pages": 5
}
```

camelCase is the universal JSON standard. Stick to it in both request payloads and response bodies.

### Never Use Abbreviations
```
✅ description
❌ desc

✅ organizationId
❌ orgId

✅ createdAt
❌ crtAt
```

The person integrating your API doesn't have your context. Abbreviations create guesswork.

### Interactive Documentation — Swagger/OpenAPI
Every production API should have:
- A Swagger (OpenAPI) spec
- An interactive playground where engineers can try out endpoints
- Updated docs whenever the API changes

This is not optional for professional backend engineering. It's part of your deliverable.

### Design Before You Code
```
Wireframes
  ↓
Identify resources (nouns)
  ↓
DB schema design
  ↓
API interface design (Insomnia / Postman / Swagger)  ← language-agnostic
  ↓
Implementation (Node.js / Go / Python / etc.)
```

If you jump straight to coding, you make design decisions under pressure and end up with inconsistent APIs that are hard to change later.

---

## Complete API Design Cheatsheet

```
Resource Naming:
  ✅ Plural nouns:      /organizations /projects /tasks
  ❌ Singular:          /organization /project
  ❌ Verbs:             /getOrg /createProject

URL Structure:
  List:    GET    /{resource}
  Create:  POST   /{resource}
  Get one: GET    /{resource}/:id
  Update:  PATCH  /{resource}/:id
  Delete:  DELETE /{resource}/:id
  Action:  POST   /{resource}/:id/{action}

Status Codes:
  200 → success (GET, PATCH, custom action)
  201 → created (POST that creates)
  204 → success, no content (DELETE)
  400 → bad request (validation error)
  401 → not authenticated
  403 → not permitted
  404 → specific resource not found
  409 → conflict (duplicate)

List API Response:
  {
    "data": [...],
    "total": 50,
    "page": 1,
    "totalPages": 5
  }

  Empty list → 200 + { "data": [], "total": 0 }
  NOT 404!

Create Payload:
  Include:  name, status, description (user-provided fields)
  Exclude:  id, createdAt, updatedAt (server-handled)

Defaults:
  page → 1
  limit → 10 or 20
  sortBy → createdAt
  sortOrder → desc

Conventions:
  JSON fields → camelCase always
  No abbreviations
  Same field name across all resources
  Same route pattern across all resources
```

---

## Summary

| Topic | Key Point |
|---|---|
| **REST** | Representational State Transfer — architectural style with 6 constraints for scalable web |
| **Resources** | Identify from nouns in wireframes/requirements |
| **URL design** | `https://api.domain.com/v1/{plural-resource}/{id}` |
| **HTTP methods** | GET=read, POST=create/custom, PATCH=partial update, PUT=full replace, DELETE=remove |
| **Idempotency** | GET/PUT/PATCH/DELETE are idempotent. POST is not. |
| **Custom actions** | Use POST: `POST /{resource}/{id}/{action}` |
| **Status codes** | 200/201/204 for success; 400/401/403/404/409 for errors |
| **Pagination** | Always paginate list APIs. Return `data`, `total`, `page`, `totalPages`. |
| **Sorting** | Support `sortBy` + `sortOrder`. Default: `createdAt` desc. |
| **Filtering** | Support filter params. Empty list → 200 not 404. |
| **Consistency** | Same naming, same patterns, same conventions across all resources. |
| **Sane defaults** | Never require the client to send obvious params. |
| **Documentation** | Always ship Swagger/OpenAPI docs with your API. |
| **Design first** | Design the interface before writing any code. |

---

## One-Line Takeaways

> REST is about representing resources with consistent interfaces and transferring their state between client and server via HTTP.

> A good API is not clever — it's predictable. Follow the patterns, be consistent, and document everything.

> Design the interface first. Code it second. Never the other way around.

---

*Next lecture → Databases — Relational vs non-relational, ACID, schema design, indexes, ORMs, and migrations.*
