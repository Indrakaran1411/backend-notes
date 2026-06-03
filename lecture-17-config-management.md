# Lecture 17 — Production-Grade Configuration Management
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=GR9NtirPXyc)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Most developers think config management = storing database passwords and API keys. That's like saying a car is just about the engine. This lecture covers the full scope of configuration management — what it is, all the types of config, where to store them, why configs differ by environment, and security best practices.

---

## 1. What is Configuration Management?

**Definition:** The systematic approach to organise, store, access, and maintain all the settings of your backend application.

Think of it as the **DNA of your application** — it decides how your code runs in different environments.

### What Most People Think Config Management Is
- Database connection URL/password
- JWT secrets
- API keys for external services

### What Config Management Actually Covers
- How the application starts up
- How it connects to external services
- How it behaves in different environments
- What it logs and at what level
- Where it sends metrics
- Which features are enabled for which users
- Performance tuning parameters
- Business rules and limits

---

## 2. Why Config Management Matters

### The Stakes Are High

A misconfigured **frontend** might show the wrong dialog or redirect to the wrong route. Minor annoyance.

A misconfigured **backend** can:
- Expose customer data
- Process payments incorrectly
- Bring down your entire platform

### Configuration Chaos

Without a systematic approach, you end up with **configuration chaos**:
- Hard-coded values scattered throughout the codebase
- Inconsistent behaviour across environments
- Security vulnerabilities from exposed secrets
- Debugging nightmares — can't reproduce issues because you don't know what config was active when something broke

### The Distributed Systems Challenge

Modern backends don't run in isolation. They're part of distributed systems with:
- Multiple microservices
- Databases, caches (Redis), message queues
- Third-party integrations (auth, email, payments, storage)

Every integration point requires its own configuration. With dozens of services, config management becomes critical infrastructure.

---

## 3. Types of Configuration

Not all configs are equal. They differ in sensitivity, change frequency, and which environments they apply to.

### Type 1 — Application Settings
The most common. Controls how the server itself runs.

| Setting | Example values |
|---|---|
| `PORT` | `3000` (local), `8080` (production) |
| `LOG_LEVEL` | `debug` (local), `info` (production) |
| `CONNECTION_POOL_SIZE` | `10` (local), `50` (production) |
| `REQUEST_TIMEOUT` | `60000` ms (60 seconds) |

**Log level example:** On local, use `debug` to see detailed logs. On production, use `info` — debug logs would flood your expensive production logging service with noise.

**Timeout example:** If you set a 60-second timeout but your AI image generation averages 80 seconds → every request gets a 504 Gateway Timeout. These values must be tuned per use case.

---

### Type 2 — Database Config
Everything your app needs to connect to the database:
- Host, port, username, password, database name → combined into a connection URL
- Query timeout (how long before a query is killed)
- Connection pool parameters

```
DATABASE_URL=postgres://username:password@host:5432/dbname
DB_POOL_MIN=2
DB_POOL_MAX=10
DB_QUERY_TIMEOUT=30000
```

---

### Type 3 — External Service Configs
API keys and connection details for every third-party service:

```
# Email
RESEND_API_KEY=re_xxx

# Payments
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Auth
CLERK_SECRET_KEY=sk_live_xxx

# Storage
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_S3_BUCKET=my-uploads
```

---

### Type 4 — Feature Flags
Dynamically enable or disable features without deploying new code.

**Use case:** You built a new checkout flow. You don't want to roll it out to all users at once. You want to:
- A/B test it (50% of users get new flow, 50% get old)
- Roll it out geographically (US only first)
- Enable it for beta users only

```
NEW_CHECKOUT_ENABLED=true
NEW_CHECKOUT_ROLLOUT_PERCENTAGE=25
NEW_CHECKOUT_REGIONS=US,CA
```

Feature flags allow this without touching application code. Just change the config.

---

### Type 5 — Security Config
```
JWT_SECRET=super-secret-key-min-256-bits
SESSION_SECRET=another-secret
SESSION_TIMEOUT=3600  # seconds
BCRYPT_ROUNDS=12
```

---

### Type 6 — Performance Tuning
Runtime performance parameters:
```
GOMAXPROCS=4           # Go: max CPU cores to use
NODE_CLUSTER_WORKERS=4 # Node: number of cluster workers
REDIS_MAX_RETRIES=3
HTTP_MAX_CONNECTIONS=1000
```

---

### Type 7 — Business Rules
Application-level business logic that you want to control without code changes:
```
MAX_ORDER_AMOUNT=100000        # cents
MAX_CART_ITEMS=50
FREE_SHIPPING_THRESHOLD=5000   # cents
MAX_COUPON_USES_PER_USER=3
```

This is powerful — change a business rule (like max order amount) with a config change, not a deployment.

---

## 4. Config Storage — Where to Keep Configs

The right storage depends on security requirements, environment, and team size.

### Option 1 — Environment Variables (Most Common)

The universal standard. Loaded from `.env` files in development, from the deployment system in production.

```bash
# .env (local development — NEVER commit to Git)
DATABASE_URL=postgres://localhost:5432/myapp_dev
JWT_SECRET=local-dev-secret
PORT=3000
LOG_LEVEL=debug
```

**How it flows in production:**
```
Deployment triggered
    ↓
Fetch secrets from secrets manager (Vault/AWS Parameter Store/etc.)
    ↓
Inject as environment variables into the running container
    ↓
App reads process.env.DATABASE_URL etc. at runtime
```

**Critical rules:**
- `.env` → add to `.gitignore` immediately — never commit
- `.env.example` → commit this (shows what vars are needed, no real values)
- Production secrets → never in `.env` files, always in secrets manager

---

### Option 2 — Config Files (YAML / TOML / JSON)

Good for non-sensitive application settings and self-documenting configurations.

**Why YAML over JSON:**
- YAML supports comments (`# this is why this value is what it is`)
- Comments are essential for team knowledge sharing
- JSON has no comment support

```yaml
# config.yaml
server:
  port: 3000
  timeout: 30s
  
database:
  pool_size: 10
  max_idle: 5

logging:
  level: info  # debug for local development
  format: json
  
features:
  new_checkout: false
  dark_mode: true
```

TOML is also increasingly used (Rust ecosystem, Cargo.toml, etc.).

**Real-world example:** Major open-source Go projects (like authentication providers, Apache projects) use `config.yaml` for their application settings.

---

### Option 3 — Cloud Secrets Managers (Production)

For production secrets: database passwords, API keys, JWT secrets — use a dedicated service.

| Service | Provider |
|---|---|
| **HashiCorp Vault** | Self-hosted or cloud, very flexible |
| **AWS Parameter Store / Secrets Manager** | AWS-native |
| **Azure Key Vault** | Azure-native |
| **Google Secret Manager** | GCP-native |

**Why these are worth it:**
- Encryption at rest — secrets are encrypted when stored
- Encryption in transit — encrypted when fetched
- Access control — fine-grained permissions
- Audit logs — who accessed what, when
- Secret rotation — automated rotation of API keys and passwords
- Already handles all the security details you'd otherwise have to build yourself

---

### Option 4 — Hybrid Strategy (Most Real-World Apps)

In practice, most production backends use multiple sources with a priority order:

```
Priority (highest to lowest):
  1. AWS Parameter Store / Vault (production secrets)
  2. Environment variables (deployment-time injection)
  3. config.yaml (default application settings)
  4. Code defaults (last resort)
```

At startup:
1. Load defaults from `config.yaml`
2. Override with environment variables
3. Override with secrets from secrets manager
4. Final config object is built and validated

```javascript
const config = {
  db: {
    url: process.env.DATABASE_URL || yaml.db.url,
    poolSize: parseInt(process.env.DB_POOL_SIZE || yaml.db.poolSize),
  },
  jwt: {
    secret: process.env.JWT_SECRET,  // required, no default
  },
  port: parseInt(process.env.PORT || '3000'),
};
```

---

## 5. Environment-Specific Configs

Why do configs differ by environment? Because each environment has different priorities:

| Environment | Priority | Example config differences |
|---|---|---|
| **Development** (local) | Developer productivity, fast debugging | `LOG_LEVEL=debug`, `DB_POOL_SIZE=5`, fast timeouts |
| **Testing** (CI/CD) | Automated validation, reproducibility | In-memory DB, test API keys, deterministic settings |
| **Staging** | Mirror production, cost-conscious | `DB_POOL_SIZE=2` (save $), real services but test keys |
| **Production** | Reliability, security, performance | `DB_POOL_SIZE=50`, secrets manager, `LOG_LEVEL=info` |

### Concrete Example — DB Connection Pool Size

```
Development:  DB_POOL_MAX=10   (you're the only one, local machine)
Staging:      DB_POOL_MAX=2    (few users, save cloud cost)
Production:   DB_POOL_MAX=50   (many concurrent users, traffic spikes)
```

Same application code. Different behaviour. No code changes — just config.

> 💡 This is the whole point of config management: **change behaviour without touching code.**

---

## 6. Security Best Practices

### Rule 1 — Never Hardcode Secrets

```javascript
// ❌ NEVER do this
const db = new Client({ password: 'mydbpassword123' });
const stripe = new Stripe('sk_live_actualrealkey');

// ✅ Always use environment variables
const db = new Client({ password: process.env.DB_PASSWORD });
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
```

If a secret is in your code, it's in your git history forever. Even after you delete it. Thousands of developers have accidentally committed secrets and faced breaches.

### Rule 2 — Use a Secrets Manager for Production

Self-hosting your own secret rotation and encryption logic is complex and error-prone. Services like Vault and AWS Parameter Store:
- Encrypt secrets at rest
- Encrypt secrets in transit
- Provide audit logs
- Support automatic rotation
- Handle access control

> **"It's always a good idea to overengineer when it comes to security."** — Sriniously

### Rule 3 — Principle of Least Privilege

Not everyone needs access to everything:

| Role | What they should access |
|---|---|
| Frontend developer | Frontend API URL, public API keys |
| Backend developer | Database URL, Redis, external service API keys |
| DevOps/Infrastructure | Cloud instance credentials, Kubernetes secrets |
| Everyone | Nothing else |

Restrict access by role. Audit who has access. Review regularly.

### Rule 4 — Rotate Secrets Periodically

All secrets should rotate:
- JWT secrets → rotate every 90 days or after suspected leak
- API keys → rotate periodically, immediately after team member leaves
- Database passwords → rotate on a schedule

Secrets managers can automate this. Manual rotation is better than no rotation.

### Rule 5 — Always Validate Your Configs (The Most Important Rule)

> *"If you were to take one thing from this video, this is the most important part. Always validate your configs."*

Loading config without validation:
```javascript
// ❌ Dangerous — no validation
const jwtSecret = process.env.JWT_SECRET;
// If this is undefined, every JWT operation will silently fail
// Or worse, use `undefined` as the secret — a severe security hole
```

Validated config startup:
```javascript
// ✅ Using Zod for TypeScript
import { z } from 'zod';

const configSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  REDIS_URL: z.string().url().optional(),
});

const config = configSchema.parse(process.env);
// If any required variable is missing or invalid → throws at startup
// App never starts in a misconfigured state
```

**Why validate at startup?**
- If validation fails → app refuses to start → previous deployment keeps running (blue-green)
- If you don't validate → app starts, appears healthy, fails only when a specific feature is used → users get 500 errors days or weeks later
- Config bugs are silent killers — they cause unexpected behaviour that's hard to reproduce and debug

---

## 7. The Complete Config Loading Pattern

```javascript
// config.js — load once at startup, use everywhere

import { z } from 'zod';
import yaml from 'js-yaml';
import fs from 'fs';

// Step 1: Load YAML defaults
const yamlConfig = yaml.load(fs.readFileSync('config.yaml', 'utf8'));

// Step 2: Define schema with validation
const schema = z.object({
  port: z.coerce.number().default(yamlConfig.server?.port || 3000),
  logLevel: z.enum(['debug', 'info', 'warn', 'error']).default('info'),

  db: z.object({
    url: z.string().url(),
    poolSize: z.coerce.number().default(10),
  }),

  jwt: z.object({
    secret: z.string().min(32),
    expiresIn: z.string().default('15m'),
  }),

  stripe: z.object({
    secretKey: z.string(),
  }).optional(),
});

// Step 3: Parse and validate — fail fast if anything is wrong
const config = schema.parse({
  port: process.env.PORT,
  logLevel: process.env.LOG_LEVEL,
  db: {
    url: process.env.DATABASE_URL,
    poolSize: process.env.DB_POOL_SIZE,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN,
  },
  stripe: process.env.STRIPE_SECRET_KEY ? {
    secretKey: process.env.STRIPE_SECRET_KEY,
  } : undefined,
});

export default config;
// App fails to start if any required config is missing
```

---

## Summary

| Topic | Key Point |
|---|---|
| **What config management is** | All settings that control app behaviour — not just secrets |
| **Configuration chaos** | Hard-coded values, inconsistent environments, security holes, impossible debugging |
| **Application settings** | Port, log level, pool size, timeouts — per environment |
| **Database config** | Connection URL, pool size, query timeouts |
| **External service config** | API keys for email, payments, auth, storage |
| **Feature flags** | Enable/disable features without code changes. Enable for specific users/regions. |
| **Security config** | JWT secret, session secret, bcrypt rounds |
| **Business rules** | Max order amount, free shipping threshold — configurable without deployment |
| **Environment variables** | Most common storage. `.env` locally, injected by deployment system in production. |
| **YAML/TOML files** | Good for non-sensitive settings. YAML preferred over JSON (supports comments). |
| **Secrets managers** | Vault, AWS Parameter Store, Azure Key Vault — encrypted at rest + in transit |
| **Hybrid strategy** | Secrets manager → env vars → config file → code defaults |
| **Why envs differ** | Dev=productivity, Testing=reproducibility, Staging=mirror prod, Production=reliability |
| **Never hardcode** | Secrets in code = secrets in git history forever |
| **Least privilege** | Each role only accesses configs they need |
| **Rotate secrets** | Regular rotation, immediately on suspected breach |
| **ALWAYS validate** | Use Zod/go-validator at startup. Fail fast if config is wrong. Best thing you can do. |

---

## One-Line Takeaways

> Config management is the DNA of your application — it controls how your code behaves across every environment without changing a single line of code.

> Never hardcode secrets. Never. Not once. Not even in a test file that you "promise will never reach production."

> Validate ALL your configs at startup using a schema validator. This single habit prevents a class of production bugs that are incredibly hard to debug.

---

*Next lecture → Logging, Monitoring and Observability — The three pillars of understanding what's happening inside your production system.*
