# Lecture 12 — Mastering Databases with PostgreSQL
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=F7Vwp2Xo5Do)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Databases are one of the most frequent and most important operations in backend engineering. This lecture covers everything: why we need databases, disk vs RAM, DBMS, relational vs non-relational, why PostgreSQL, data types, database design, relationships, migrations, seeding, SQL queries, joins, parameterised queries, indexes, and triggers — all in the context of building a real project management platform.

---

## 1. Why Do We Need Databases?

### Persistence
At the core, a database exists to **persist information across sessions**.

Persistence = storing data so it survives even after the program that created it has been stopped.

Without persistence: every time you open your to-do app, all your tasks are gone. Every login would be a new account. Every order would disappear after the browser tab closes.

> 💡 Persistence is what makes software useful across time and across devices.

### What is a "Database"?

The term is broader than most people think. Any structured persistent storage counts:
- Smartphone contact list
- Browser localStorage / sessionStorage / cookies
- A text file with notes

But in the context of **backend engineering**, "database" specifically means a **disk-based database** — software that stores data on a hard drive or SSD and provides efficient CRUD operations.

---

## 2. Disk vs RAM — Why Databases Use Disk

| Storage | Speed | Cost | Capacity | Used for |
|---|---|---|---|---|
| **RAM** (primary memory) | Very fast | Expensive | Small (8–128 GB typical) | Caching (Redis) |
| **Disk** (secondary memory) | Slower | Cheap | Large (512 GB–2+ TB typical) | Databases (PostgreSQL, MySQL) |

**Why disk for databases?**
- You need more space than RAM can provide affordably
- A fair speed tradeoff — disk is slower, but still fast enough for most database workloads
- Data persists across restarts (RAM is wiped when power is off)

**Why RAM for caching?**
- Ultra-fast access for frequently-read data
- Redis, Memcached store data in RAM
- Trade: more expensive, less space, lost on restart

> 💡 Caches (Redis) = RAM. Databases (PostgreSQL) = Disk. Different tools for different jobs.

---

## 3. DBMS — Database Management System

Just storing data on disk isn't enough. You need software to:
- Organise data efficiently
- Provide fast CRUD operations (Create, Read, Update, Delete)
- Enforce data integrity (the data is always correct and valid)
- Handle security (access control, users, roles)
- Manage concurrency (multiple users modifying data at the same time)

This software is called a **DBMS (Database Management System)**. PostgreSQL, MySQL, MongoDB are all DBMS software.

### Why Not Just Text Files?

Before DBMS existed, people stored data in text files. Problems:

| Problem | Why it's bad |
|---|---|
| **Parsing** | You have to write application code to split lines, find fields — slow and error-prone |
| **No structure** | Can't enforce data types. A number field accepts "hello". |
| **No concurrency** | Two users editing simultaneously = data corruption. Last write wins randomly. |
| **No indexing** | Every lookup scans the entire file sequentially — catastrophically slow at scale |

These exact problems led to the invention of DBMS software.

---

## 4. Relational vs Non-Relational Databases

### Relational (SQL) Databases
- Data stored in **tables with rows and columns**
- **Strict schema** — every table has predefined columns with predefined data types
- Tables **relate to each other** via foreign keys
- Queried using SQL (Structured Query Language)
- Examples: PostgreSQL, MySQL, SQLite, SQL Server

**Key advantage:** Data integrity — the DB enforces types, constraints, and relationships. You can trust the state of your data.

**Best for:** Applications where data accuracy is critical — financial systems, CRMs, e-commerce, user data.

### Non-Relational (NoSQL) Databases
- No fixed schema — each "document" (row equivalent) can have different fields
- Flexible, schema-less
- Examples: MongoDB (documents), Redis (key-value), Cassandra (wide-column)

**Key advantage:** Flexibility and speed of development — push any JSON, don't think about schema.

**Best for:** Prototyping, content management, unstructured data, cases where the schema genuinely can't be defined upfront.

**Key disadvantage:** Data integrity is your responsibility — the DB won't stop you from inserting garbage.

### Real-World Examples

| System | Database choice | Why |
|---|---|---|
| CRM (customer data) | PostgreSQL | Accurate, consistent, relational customer data |
| CMS (blog/content) | MongoDB | Article content is unstructured JSON — variable fields |
| Caching/sessions | Redis | In-memory, ultra-fast, key-value |
| Real-time analytics | InfluxDB / Cassandra | Time-series, high write volume |

---

## 5. Why PostgreSQL

When starting any project, **PostgreSQL is the correct default choice**. Here's why:

| Reason | Detail |
|---|---|
| **Open source and free** | Self-host on your own servers, no licensing costs |
| **SQL standard compliant** | Standard SQL queries work across MySQL, PostgreSQL, etc. Easy to migrate |
| **Extremely extensible** | 1400+ pages of documentation, covers virtually every use case |
| **Reliable and scalable** | Battle-tested by large companies and startups alike |
| **Native JSON support** | `json` and `jsonb` types allow flexible unstructured data — no need for MongoDB for most use cases |

> 💡 PostgreSQL's `jsonb` type means you can have flexible unstructured data AND relational data in the same database. You don't need MongoDB for dynamic data if you're using PostgreSQL.

---

## 6. PostgreSQL Data Types

Understanding which data type to choose is a daily decision as a backend engineer.

### Integer Types
| Type | Max value | Use |
|---|---|---|
| `SMALLINT` | ~32,000 | Small counters |
| `INTEGER` | ~2.1 billion | General purpose IDs and counts |
| `BIGINT` | ~9.2 quintillion | Large IDs, analytics |
| `SERIAL` / `BIGSERIAL` | Auto-incrementing | Auto-increment IDs (BIGSERIAL preferred for production) |

### Decimal/Float Types
| Type | Accuracy | Speed | Use |
|---|---|---|---|
| `DECIMAL` / `NUMERIC` | Exact | Slower | Financial data (price, amount) |
| `REAL` / `DOUBLE PRECISION` / `FLOAT` | Approximate | Faster | Scientific data, sizes, coordinates |

> ⚠️ Always use `DECIMAL` for money/prices. Floating point has different representations across systems — tiny errors can cause significant financial bugs.

### String Types
| Type | Behaviour | Recommendation |
|---|---|---|
| `CHAR(n)` | Pads with spaces to length n | Rarely use — only for fixed-length codes |
| `VARCHAR(n)` | Variable length up to n | Convention from MySQL, has misconceptions in PostgreSQL |
| `TEXT` | Unlimited length | **Use this always in PostgreSQL** |

**Why TEXT over VARCHAR?**
- PostgreSQL documentation explicitly recommends TEXT
- No performance difference between TEXT and VARCHAR in PostgreSQL
- VARCHAR(255) from MySQL conventions has no meaning in PostgreSQL
- If you use VARCHAR(255) and later need more space → migration required. TEXT avoids this.
- Cleaner, simpler migration files — no misleading number

> 💡 Use TEXT for all string fields in PostgreSQL. Enforce length constraints in application code (Zod, Pydantic), not at the DB level.

### Other Important Types
| Type | Use |
|---|---|
| `BOOLEAN` | `true` / `false` |
| `DATE` | Date only (no time) |
| `TIME` | Time only |
| `TIMESTAMP` | Date + time |
| `TIMESTAMPTZ` | Date + time + timezone ← **use this in production** |
| `INTERVAL` | Duration (`10 days`, `1 week`) |
| `UUID` | Universally unique identifier — excellent for primary keys |
| `JSON` | JSON stored as plain text |
| `JSONB` | JSON stored as binary ← **faster queries, use this** |
| `ARRAY` | Array of any type (`integer[]`, `text[]`) |

**JSON vs JSONB:**
- `JSON` = stored as plain text, preserves formatting
- `JSONB` = stored in binary format, faster reads, supports indexing
- **Always use JSONB** unless you specifically need to preserve original JSON formatting

---

## 7. Database Design — The Project Management Platform

The practical section designs a database for a project management tool (similar to Jira/Linear).

### Step 1 — Identify Resources (From Requirements)
From the wireframes and requirements, identify the nouns:
- `organizations`
- `users`
- `projects`
- `tasks`
- `project_members` (linking table)

### Step 2 — Schema Design Principles

**Naming conventions for PostgreSQL:**
- Table names: **plural, lowercase, snake_case** → `users`, `project_members`
- Column names: **lowercase, snake_case** → `full_name`, `created_at`, `owner_id`
- Why? PostgreSQL is case-insensitive by default. CamelCase requires double-quoting everywhere in queries → painful. Stick to snake_case.

**Every table should have:**
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

**NOT NULL is your default:**
> More than 70% of your table columns should have NOT NULL. Null values in unexpected places cause bugs and inconsistencies. Only omit NOT NULL when null genuinely means "not provided/optional".

### Step 3 — Enum Types (Custom Data Types)

For fields with a fixed set of allowed values, create an enum:

```sql
-- Create custom types
CREATE TYPE project_status AS ENUM ('active', 'completed', 'archived');
CREATE TYPE task_status AS ENUM ('pending', 'in_progress', 'completed', 'cancelled');
CREATE TYPE member_role AS ENUM ('owner', 'admin', 'member');
```

**Why enums instead of plain text?**

| Reason | Explanation |
|---|---|
| **Data integrity** | Database rejects any value not in the enum — you can't insert `"archivd"` by typo |
| **Documentation** | Reading the migration, anyone immediately knows what values are allowed |
| **Self-documenting** | Future team members can understand the allowed values without reading all application code |

### Step 4 — Full Schema (Annotated)

```sql
-- ENUM TYPES
CREATE TYPE project_status AS ENUM ('active', 'completed', 'archived');
CREATE TYPE task_status AS ENUM ('pending', 'in_progress', 'completed', 'cancelled');
CREATE TYPE member_role AS ENUM ('owner', 'admin', 'member');

-- USERS TABLE
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         TEXT NOT NULL UNIQUE,   -- unique constraint: no duplicate emails
  full_name     TEXT NOT NULL,
  password_hash TEXT NOT NULL,          -- store hash, never plain text
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- USER PROFILES TABLE (one-to-one with users)
-- Why separate table? Profile data can grow (bio, socials, etc.)
-- and changes more frequently than core auth data
CREATE TABLE user_profiles (
  user_id    UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  -- user_id is BOTH the primary key AND foreign key (enforces 1-to-1)
  avatar_url TEXT,          -- nullable: user may not upload a photo
  bio        TEXT,          -- nullable: user may not write a bio
  phone      TEXT           -- nullable: phone is optional
);

-- PROJECTS TABLE
CREATE TABLE projects (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL,
  description TEXT,         -- nullable: description is optional
  status      project_status NOT NULL DEFAULT 'active',
  owner_id    UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  -- ON DELETE RESTRICT: cannot delete a user who owns projects
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- TASKS TABLE
CREATE TABLE tasks (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  -- ON DELETE CASCADE: deleting a project deletes all its tasks
  title       TEXT NOT NULL,
  description TEXT,
  priority    INTEGER NOT NULL DEFAULT 1,
  CONSTRAINT priority_range CHECK (priority >= 1 AND priority <= 5),
  -- CHECK constraint: priority must be 1-5
  status      task_status NOT NULL DEFAULT 'pending',
  due_date    DATE,
  assigned_to UUID REFERENCES users(id) ON DELETE SET NULL,
  -- ON DELETE SET NULL: if user deleted, task becomes unassigned (not deleted)
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- PROJECT MEMBERS (many-to-many linking table)
CREATE TABLE project_members (
  project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role        member_role NOT NULL DEFAULT 'member',
  PRIMARY KEY (project_id, user_id),  -- composite primary key
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 8. Constraints — Protecting Data Integrity

| Constraint | What it does |
|---|---|
| `PRIMARY KEY` | Uniquely identifies each row. Implies NOT NULL + UNIQUE. Auto-indexed. |
| `NOT NULL` | Field cannot be null. The most common constraint — use it everywhere appropriate. |
| `UNIQUE` | No two rows can have the same value for this field (e.g., email) |
| `CHECK` | Custom condition — value must satisfy an expression (e.g., `priority >= 1 AND priority <= 5`) |
| `FOREIGN KEY` | Value must exist as a primary key in another table. Enforces referential integrity. |

---

## 9. Relationships — The Three Types

### One-to-One
One row in Table A corresponds to exactly one row in Table B.

```
users (1) ←──── (1) user_profiles
```

**Implementation:** The foreign key in Table B is also the primary key.
```sql
CREATE TABLE user_profiles (
  user_id UUID PRIMARY KEY REFERENCES users(id)
  -- user_id is PK + FK → enforces one-to-one
);
```

### One-to-Many
One row in Table A corresponds to many rows in Table B.

```
projects (1) ──── (many) tasks
```

**Implementation:** Table B has a foreign key pointing to Table A's primary key (not the PK of B).
```sql
CREATE TABLE tasks (
  project_id UUID NOT NULL REFERENCES projects(id)
  -- project_id is FK only, not PK → many tasks can share same project_id
);
```

### Many-to-Many
A row in Table A can relate to many rows in Table B, and vice versa.

```
users (many) ────── (many) projects
```

**Implementation:** Create a **linking table** with a **composite primary key** of both foreign keys.
```sql
CREATE TABLE project_members (
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  user_id    UUID NOT NULL REFERENCES users(id)    ON DELETE CASCADE,
  PRIMARY KEY (project_id, user_id)  -- composite PK = unique combination
);
```

The composite primary key ensures a user can't be added to the same project twice.

---

## 10. Referential Integrity — ON DELETE Options

When a row being referenced (e.g., a user) is deleted, what happens to rows that reference it (e.g., tasks)?

| Option | Behaviour | Use when |
|---|---|---|
| `ON DELETE RESTRICT` | Block the deletion. You can't delete user if they own projects. | Critical data — user must clean up before deleting |
| `ON DELETE CASCADE` | Delete all referencing rows too. Deleting project deletes all its tasks. | Child data is useless without parent |
| `ON DELETE SET NULL` | Set the FK field to null. Deleting user → task.assigned_to becomes null | Optional relationship — task still exists, just unassigned |
| `ON DELETE SET DEFAULT` | Set FK to default value. | Rarely used |

---

## 11. Database Migrations

### What Are Migrations?

**Migrations** = versioned SQL files that track every change made to your database schema over time.

```
db/
  migrations/
    001_create_users_table.sql
    002_add_user_profiles.sql
    003_create_projects.sql
    004_add_indexes.sql
```

Each file contains SQL statements. A migration tool (like `dbmate`, `flyway`, `goose`) runs them sequentially and tracks which have been applied.

### Why Migrations Instead of Manual SQL?

| Problem without migrations | Solution with migrations |
|---|---|
| No history of DB changes | Git-tracked SQL files show entire history |
| Can't reproduce DB state | Run all migrations → exact same schema |
| Hard to roll back | Down migrations reverse every change |
| Team sync issues | Everyone runs the same migrations |
| Deployment complexity | CI/CD runs migrations automatically |

### Up vs Down Migrations

```sql
-- ===== UP MIGRATION (apply the change) =====
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  ...
);

-- ===== DOWN MIGRATION (reverse the change) =====
DROP TABLE IF EXISTS users;
```

Run `dbmate up` → applies changes.  
Run `dbmate down` → reverses changes (rollback).

### Schema Version Tracking

Migration tools create a `schema_migrations` table that stores which migrations have been applied. When you run `up`, it only applies unapplied migrations. Prevents running the same migration twice.

---

## 12. Seeding — Test Data

**Seeding** = inserting test data into your database for development/testing purposes.

```sql
-- Example seed: create test users
INSERT INTO users (email, full_name, password_hash) VALUES
  ('alice@test.com', 'Alice Brown', '$2b$12$...'),
  ('bob@test.com', 'Bob Jones', '$2b$12$...');
```

Create a separate migration file for seeding:
```
db/
  migrations/
    001_create_tables.sql
    002_seed_data.sql    ← separate file for test data
```

---

## 13. SQL Queries for Backend APIs

### Get All Users (with Profile) — `GET /api/v1/users`

```sql
SELECT
  u.*,
  to_jsonb(up.*) AS profile    -- embed profile as JSON field
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
-- LEFT JOIN: return user even if they have no profile entry
ORDER BY u.created_at DESC;    -- always sort — DB returns random order without it
```

Key concepts:
- **Table aliases** (`u`, `up`) make queries readable
- **LEFT JOIN** vs **INNER JOIN**: use LEFT when the related record might not exist
- **to_jsonb()**: convert a row to JSON and embed it in the parent row
- **ORDER BY ... DESC**: always sort list queries — DB order is unpredictable otherwise

### Get Single User — `GET /api/v1/users/:userId`

```sql
SELECT
  u.*,
  to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
WHERE u.id = :userId;    -- parameterised query
```

### Create User — `POST /api/v1/users`

```sql
INSERT INTO users (email, full_name, password_hash)
VALUES (:email, :fullName, :passwordHash)
RETURNING *;    -- return the created row immediately
```

### Update User Profile — `PATCH /api/v1/users/:userId`

```sql
UPDATE user_profiles
SET
  bio   = :bio,
  phone = :phone
WHERE user_id = :userId
RETURNING *;
```

Only update fields that were passed — construct the SET clause dynamically in application code.

### Dynamic Query — Filtering, Sorting, Pagination

For list APIs, you construct the query dynamically in your application code:

```javascript
// Pseudocode: build query based on what user passed
let query = `
  SELECT u.*, to_jsonb(up.*) AS profile
  FROM users u
  LEFT JOIN user_profiles up ON u.id = up.user_id
`;

// Filter (only add WHERE if user passed a filter)
if (letter) {
  query += ` WHERE u.full_name ILIKE :letter || '%'`;
}

// Sort (use defaults if not passed)
const sortBy = params.sortBy || 'created_at';
const sortOrder = params.sortOrder || 'DESC';
query += ` ORDER BY u.${sortBy} ${sortOrder}`;

// Pagination
const page = params.page || 1;
const limit = params.limit || 10;
const offset = (page - 1) * limit;
query += ` LIMIT :limit OFFSET :offset`;
```

In actual SQL with all parameters:
```sql
SELECT u.*, to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
WHERE u.full_name ILIKE :letter || '%'   -- optional filter
ORDER BY u.email ASC                      -- dynamic sort
LIMIT :limit OFFSET :offset;              -- pagination
```

**ILIKE** = case-insensitive LIKE. `'J%'` matches "John", "Jane", "JAMES".

---

## 14. Parameterised Queries — SQL Injection Prevention

### What Is SQL Injection?

Without parameterised queries, if you concatenate user input into SQL:
```javascript
// ❌ DANGEROUS — string concatenation
const query = `SELECT * FROM users WHERE email = '${userInput}'`;

// If userInput = "x' OR '1'='1"
// Query becomes: SELECT * FROM users WHERE email = 'x' OR '1'='1'
// Returns ALL users — complete data breach
```

### Parameterised Queries — The Solution

```javascript
// ✅ Safe — parameterised
const query = `SELECT * FROM users WHERE email = $1`;
db.query(query, [userInput]);
// userInput is treated as a STRING, not SQL code
// Malicious SQL is escaped automatically
```

**Rule: Always use parameterised queries for any user-supplied value.** Never concatenate user input into SQL strings.

---

## 15. Database Indexes

### What Is an Index?

An index is a **lookup table** the database maintains alongside your actual data. Like a book's index:
- Book index: "Chapter 4 → page 54" → you jump directly to page 54
- DB index: "email = 'test@test.com' → row at disk location XYZ" → database jumps directly to that row

Without an index → **sequential scan**: check every row one by one. Catastrophic at millions of rows.  
With an index → **direct lookup**: jump straight to the matching row.

### When to Create an Index

Create an index when a column is frequently used in:
1. **WHERE clauses** — `WHERE email = :email`
2. **JOIN conditions** — `ON users.id = tasks.assigned_to`
3. **ORDER BY / sorting** — `ORDER BY created_at DESC`

> ⚠️ Primary keys are automatically indexed. You don't need to manually index them.

### Index Tradeoff

**Benefit:** Dramatically faster reads  
**Cost:** Every INSERT/UPDATE must also update the index → slight write overhead

Evaluate: Is the query called frequently enough that the read speedup outweighs the write overhead?

### Creating Indexes

```sql
-- Index on email (for WHERE user lookup and JOIN conditions)
CREATE INDEX idx_users_email ON users(email);

-- Index on created_at descending (for ORDER BY created_at DESC)
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Index on foreign key (for JOIN operations)
CREATE INDEX idx_tasks_project_id ON tasks(project_id);
CREATE INDEX idx_tasks_assigned_to ON tasks(assigned_to);

-- Index on status (for WHERE status = 'pending' filtering)
CREATE INDEX idx_tasks_status ON tasks(status);
```

### Why Index Foreign Keys?

Foreign keys are NOT automatically indexed in PostgreSQL (unlike primary keys). But they're frequently used in JOIN conditions. Always index foreign keys that are used in JOIN conditions.

---

## 16. Triggers — Automating updated_at

### The Problem

Every time you UPDATE a row, you should set `updated_at` to the current timestamp. But doing this manually in every UPDATE query in your application code is:
- Easy to forget
- Creates inconsistency if one developer forgets

### The Solution — Database Triggers

A **trigger** is a function that automatically runs when a specific database event occurs (INSERT, UPDATE, DELETE).

```sql
-- Step 1: Create the function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Step 2: Attach trigger to each table
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_projects_updated_at
  BEFORE UPDATE ON projects
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Repeat for each table that has updated_at
```

Now whenever any row in `users` or `projects` is updated, `updated_at` is automatically set to the current timestamp — no application code needed.

---

## 17. Complete Database Workflow for a Backend Engineer

```
1. Analyse requirements (wireframes, client discussions)
2. Identify resources (nouns → tables)
3. Design schema (columns, types, constraints, relationships)
4. Write migrations (SQL files in sequential order)
5. Write seed data (test data for development)
6. Write API queries (SELECT with JOINs, INSERT, UPDATE, DELETE)
7. Parameterise all user inputs (prevent SQL injection)
8. Create indexes (WHERE, JOIN, ORDER BY columns)
9. Set up triggers (automated updated_at)
10. Deploy and test
```

---

## Quick Reference — Design Decisions

| Decision | Rule |
|---|---|
| Primary key type | `UUID` with `DEFAULT gen_random_uuid()` |
| String type | Always `TEXT` in PostgreSQL |
| Timestamps | `TIMESTAMPTZ` (includes timezone) |
| NOT NULL | Default yes — only allow null for genuinely optional fields |
| Fixed value sets | Use ENUM types |
| Decimal accuracy | `DECIMAL` for money, `FLOAT` for scientific |
| Foreign keys | Always add + always index |
| Order of list results | Always `ORDER BY created_at DESC` by default |
| Parameterised queries | Always — never concatenate user input |
| Indexes | WHERE + JOIN + ORDER BY columns that are frequently queried |
| updated_at automation | Use triggers |
| Schema changes | Always use migrations, never manual SQL in production |

---

## Summary

| Concept | Key Point |
|---|---|
| **Persistence** | Databases keep data alive after program ends |
| **Disk vs RAM** | Databases = disk (cheap, large). Caches = RAM (fast, expensive) |
| **DBMS** | Software managing CRUD + integrity + security + concurrency |
| **Relational** | Structured tables, strict schema, SQL, relationships via foreign keys |
| **Non-relational** | Flexible schema, no enforced structure, faster to prototype |
| **PostgreSQL** | Open source, SQL-standard, extensible, native JSONB — default choice |
| **Migrations** | Versioned SQL files for schema changes. Up = apply. Down = rollback. |
| **Seeding** | Test data for development environments |
| **Enums** | Enforce allowed values at DB level + document the options |
| **Constraints** | PRIMARY KEY, NOT NULL, UNIQUE, CHECK, FOREIGN KEY — protect data integrity |
| **Relationships** | 1-to-1 (shared PK), 1-to-many (FK in child), many-to-many (linking table) |
| **Referential integrity** | ON DELETE RESTRICT / CASCADE / SET NULL — protect related data |
| **Parameterised queries** | Always use — prevents SQL injection |
| **JOINs** | INNER JOIN (both must exist), LEFT JOIN (keep left even if right is missing) |
| **Indexes** | Lookup tables for fast reads. Index WHERE + JOIN + ORDER BY columns. |
| **Triggers** | Automatic actions on DB events. Used for auto-updating `updated_at`. |

---

## One-Line Takeaways

> Use PostgreSQL as your default. It handles both structured and flexible (JSONB) data. You rarely need MongoDB.

> Always use migrations for schema changes. Never run manual SQL in production.

> Parameterise all user inputs — no exceptions. SQL injection is real and destructive.

> Index what you query. Primary keys are auto-indexed. Foreign keys are not — add them manually.

---

*Next lecture → Business Logic Layer — How to structure the service layer for maintainable, scalable application logic.*
