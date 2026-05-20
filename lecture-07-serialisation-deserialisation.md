# Lecture 7 — Serialisation and Deserialisation for Backend Engineers
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=vzg90tY3uM0)  
> Series: Backend from First Principles

---

## What This Lecture Is About

You have a JavaScript frontend and a Rust backend. The frontend creates a JavaScript object. The backend expects a Rust struct. These are completely different data types in completely different languages on completely different machines.

How does data get from one to the other without getting lost in translation?

That's what serialisation and deserialisation solve.

---

## 1. The Core Problem

Imagine two machines trying to communicate:

```
Machine A (Client)          Network          Machine B (Server)
JavaScript App         ─────────────────>    Rust App
{ name: "Pravan" }                           ???
```

JavaScript and Rust have fundamentally different type systems:
- JavaScript is **dynamic** — types are inferred at runtime, loosely typed
- Rust is **strictly typed** — every variable has a precise compile-time type

A JavaScript object `{ name: "Pravan", age: 22 }` means nothing to a Rust program. Rust doesn't know what a JavaScript object is. Same problem in reverse — a Rust struct can't be read directly by JavaScript.

**The question:** How do two machines with completely different languages and type systems exchange data and make sense of it?

---

## 2. The Solution — A Common Standard

The answer is elegantly simple: **agree on a common format** that both sides understand. Neither JavaScript nor Rust. A neutral, universal format that any language can convert to and from.

```
JavaScript Object  →  [convert to common format]  →  travels over network
                                                              ↓
                                              [convert from common format]
                                                              ↓
                                                       Rust Struct
```

This is serialisation and deserialisation:

| Term | Direction | What happens |
|---|---|---|
| **Serialisation** | Your data → common format | Convert JS object / Rust struct / Python dict into the agreed standard (e.g., JSON) |
| **Deserialisation** | Common format → your data | Convert received JSON back into your language's native type (JS object, Rust struct, Python dict) |

### In Plain English
- **Serialise** = pack your data into a universal box before sending
- **Deserialise** = unpack the box you received and read its contents in your language

> 💡 Serialisation and deserialisation are not special features — they are a **fundamental necessity** for any two different systems to communicate.

---

## 3. Where Does This Fit in the OSI Model?

You don't need to master the OSI model as a backend engineer, but here's the mental model to have:

```
Client Side:
  Application Layer (Layer 7)  ← JSON lives here — YOUR responsibility
  Presentation Layer
  Session Layer
  Transport Layer (TCP)
  Network Layer (IP)
  Data Link Layer
  Physical Layer               ← 01001010... voltage signals over fibre

Server Side (mirror):
  Physical Layer               ← receives the bits
  ...layers convert back up...
  Application Layer            ← JSON appears here again — YOUR responsibility
```

**Key insight for backend engineers:**

Your responsibility starts and ends at the **application layer**. You send JSON. You receive JSON. Everything between — TCP packets, IP headers, data frames, physical bits — is handled by the network stack and operating system. You don't touch it.

```
Mental model:

Client                                    Server
  ↓                                          ↑
JSON ──── [network magic happens here] ──── JSON

You only care about these two ends.
```

---

## 4. Serialisation Formats — Which One to Use?

There are many serialisation standards. They fall into two broad categories:

### Text-Based Formats
Human-readable. You can open the payload and read it directly.

| Format | Full Name | Primary Use |
|---|---|---|
| **JSON** | JavaScript Object Notation | REST API communication — most common by far |
| **XML** | Extensible Markup Language | Legacy systems, SOAP APIs, some config files |
| **YAML** | YAML Ain't Markup Language | Config files (Docker, Kubernetes, CI/CD pipelines) |

### Binary Formats
Not human-readable, but significantly faster and smaller.

| Format | Primary Use |
|---|---|
| **Protobuf** (Protocol Buffers) | gRPC communication, high-performance microservices |
| **MessagePack** | Binary alternative to JSON |
| **Avro** | Big data (Apache Kafka) |

### Text vs Binary — The Tradeoff

| | Text (JSON) | Binary (Protobuf) |
|---|---|---|
| **Human readable** | ✅ Yes — you can inspect it | ❌ No — looks like garbled bytes |
| **Size** | Larger | Much smaller (2-10x) |
| **Speed** | Slower to parse | Much faster to parse |
| **Debugging** | Easy — paste in browser/Postman | Hard — need special tools |
| **Language support** | Universal | Needs schema + generated code |
| **Best for** | REST APIs, general use | High-throughput systems, gRPC |

> 💡 For this series (and for ~80% of backend work you'll do): **JSON is the standard.** Learn it deeply. You can pick up Protobuf when you need high-performance inter-service communication.

---

## 5. JSON — Deep Dive

JSON = **J**ava**S**cript **O**bject **N**otation

Despite the name, JSON is not limited to JavaScript. It's a language-neutral format used everywhere — REST APIs, config files, log files, databases (PostgreSQL has a JSON column type), and more.

### Why JSON Became the Standard
- Human-readable — you can look at a JSON payload and understand it instantly
- Simple structure — very few rules to learn
- Universally supported — every language has a JSON parser built in or available
- Lightweight compared to XML
- Maps naturally to data structures in most languages

---

### JSON Structure and Rules

```json
{
  "id": 1,
  "name": "Pravan",
  "age": 22,
  "isActive": true,
  "scores": [95, 88, 76],
  "address": {
    "city": "Chennai",
    "country": "India",
    "pinCode": "600036"
  },
  "nickname": null
}
```

**The rules — memorise these:**

| Rule | Example |
|---|---|
| Starts and ends with `{}` (object) or `[]` (array) | `{ }` or `[ ]` |
| **Keys must be strings in double quotes** | `"name"` not `name` |
| Key-value pairs separated by `:` | `"name": "Pravan"` |
| Pairs separated by `,` | `"name": "Pravan", "age": 22` |
| No trailing comma on the last item | `"age": 22` ← no comma |
| No comments allowed | `// this breaks JSON` ← invalid |

---

### JSON Data Types

JSON supports exactly **6 data types**:

| Type | Example | Notes |
|---|---|---|
| **String** | `"Pravan"` | Must use double quotes — single quotes invalid |
| **Number** | `22`, `3.14`, `-5` | No distinction between int and float |
| **Boolean** | `true`, `false` | Lowercase only |
| **Null** | `null` | Represents absence of value |
| **Array** | `[1, 2, 3]` or `["a", "b"]` | Ordered list, can mix types |
| **Object** | `{ "key": "value" }` | Nested JSON object |

**What JSON does NOT support:**
- Dates (stored as strings: `"2024-09-23T14:30:00Z"`)
- Functions
- `undefined`
- BigInt / very large integers (precision loss above `2^53`)
- Single-quoted strings
- Comments

---

### JSON in Different Languages

The same JSON gets deserialised into different native types:

```
JSON string:   '{"name": "Pravan", "age": 22}'

JavaScript:    { name: "Pravan", age: 22 }          ← JS object
Python:        {"name": "Pravan", "age": 22}         ← Python dict
Go:            struct { Name string; Age int }        ← Go struct
Rust:          struct User { name: String, age: u32 } ← Rust struct
Java:          User object with getName(), getAge()   ← Java object
```

Same data. Different representations. JSON is the universal bridge.

---

### Serialisation in Code (Conceptual)

```javascript
// JavaScript — Serialise (object → JSON string)
const user = { name: "Pravan", age: 22 };
const jsonString = JSON.stringify(user);
// Result: '{"name":"Pravan","age":22}'
// This string is what travels over the network

// JavaScript — Deserialise (JSON string → object)
const received = '{"id":1,"name":"Pravan"}';
const user = JSON.parse(received);
// Result: { id: 1, name: "Pravan" }
// Now you can use user.id, user.name normally
```

```python
# Python — Serialise
import json
user = {"name": "Pravan", "age": 22}
json_string = json.dumps(user)
# Result: '{"name": "Pravan", "age": 22}'

# Python — Deserialise
received = '{"id": 1, "name": "Pravan"}'
user = json.loads(received)
# Result: {'id': 1, 'name': 'Pravan'}
```

---

## 6. The Full Flow — Client to Server and Back

Let's trace a complete request and response through the serialisation lens:

```
Step 1 — Client prepares data (JavaScript)
  const newBook = { title: "Clean Code", author: "Robert Martin", price: 499 };

Step 2 — Client SERIALISES (object → JSON string)
  JSON.stringify(newBook)
  → '{"title":"Clean Code","author":"Robert Martin","price":499}'

Step 3 — JSON string is sent over HTTP
  POST /api/books HTTP/1.1
  Content-Type: application/json

  {"title":"Clean Code","author":"Robert Martin","price":499}

Step 4 — Network transmission
  JSON string → TCP packets → IP packets → bits → optical fibre
  [you don't care about any of this]
  bits → IP packets → TCP packets → JSON string

Step 5 — Server DESERIALISES (JSON string → server's native type)
  Rust:   let book: Book = serde_json::from_str(&body)?;
  Python: book = json.loads(request.body)
  Node:   const book = JSON.parse(req.body)  (or auto-parsed by framework)

Step 6 — Server processes the data
  Validates it, saves to database, applies business logic

Step 7 — Server SERIALISES the response (native type → JSON string)
  Rust:   serde_json::to_string(&response)?
  Python: json.dumps(response)
  Node:   res.json(response)  (framework does this automatically)

Step 8 — JSON response travels back over HTTP
  HTTP/1.1 201 Created
  Content-Type: application/json

  {"id":42,"title":"Clean Code","author":"Robert Martin","price":499}

Step 9 — Client DESERIALISES the response
  const book = await response.json();
  // Now: book.id = 42, book.title = "Clean Code", etc.

Step 10 — Client renders the data in the UI ✅
```

---

## 7. Common JSON Gotchas

Things that trip up developers when working with JSON:

### 1. Dates
JSON has no date type. Dates are stored and transmitted as strings.
```json
{ "createdAt": "2024-09-23T14:30:00Z" }
```
You must parse this string back into a Date object on both ends. Timezone handling is a common source of bugs.

### 2. Large Integers
JavaScript uses 64-bit floating point for all numbers. Integers larger than `2^53 - 1` (9007199254740991) lose precision.
```json
{ "id": 9999999999999999 }
// JavaScript reads this as: 10000000000000000 ← WRONG
```
Fix: Send large IDs as strings: `{ "id": "9999999999999999" }`

### 3. `undefined` Is Not Serialised
```javascript
const obj = { name: "Pravan", age: undefined };
JSON.stringify(obj)
// Result: '{"name":"Pravan"}'
// age is completely dropped!
```

### 4. Circular References Crash Serialisation
```javascript
const a = {};
const b = { ref: a };
a.ref = b;  // circular!
JSON.stringify(a); // Throws: TypeError: circular structure
```

### 5. The Difference Between `null` and Missing Field
```json
{ "nickname": null }    // field exists, value is null
{ }                     // field doesn't exist at all
```
These are different. Treat them differently in your code.

---

## 8. When to Use Binary Formats (Protobuf)

JSON is perfect for most use cases. But there are scenarios where binary formats like **Protobuf** are worth the extra setup:

| Scenario | Use JSON | Use Protobuf |
|---|---|---|
| REST API between frontend and backend | ✅ | ❌ |
| Public API that developers will use | ✅ | ❌ |
| Microservice-to-microservice (internal) | ✅ (fine) | ✅ (better) |
| High-throughput data pipeline | ❌ | ✅ |
| Real-time systems with millions of messages/sec | ❌ | ✅ |
| Mobile apps on slow networks | Consider | ✅ (smaller payloads) |

The cost of Protobuf: you need to define a `.proto` schema file, generate code from it, and both client and server need the same schema. More setup, but much faster and smaller.

---

## 9. Key Terms to Remember

| Term | Definition |
|---|---|
| **Serialisation** | Converting your language's native data type → common format (JSON) for transmission |
| **Deserialisation** | Converting received common format (JSON) → your language's native data type |
| **JSON** | The most common text-based serialisation standard for HTTP APIs |
| **Protobuf** | Binary serialisation format — faster and smaller than JSON, used in gRPC |
| **Content-Type: application/json** | HTTP header that tells the receiver the body is JSON |
| **Schema** | The agreed-upon structure of data — what fields exist, what types they have |

---

## Summary

| Concept | Key Point |
|---|---|
| **Why it exists** | JavaScript and Rust (and every other language) have incompatible types — a common format is needed |
| **Serialise** | Convert your data → JSON before sending |
| **Deserialise** | Convert received JSON → your native type |
| **JSON** | The universal standard — human-readable, universally supported, 6 data types |
| **JSON rules** | Keys in double quotes (strings only), 6 value types, no trailing commas, no comments |
| **Your responsibility** | Application layer only — you deal with JSON. Everything below is the network's job. |
| **Text vs Binary** | JSON = readable, universal. Protobuf = fast, small, complex. Use JSON by default. |
| **Gotchas** | Dates are strings, large ints lose precision, undefined is dropped, circular refs crash |

---

## One-Line Takeaway

> Serialisation is how two different systems speak the same language — they agree on a common format (JSON), convert to it before sending, and convert from it after receiving.

---

*Next lecture → Authentication and Authorisation — How backends verify who you are and what you're allowed to do.*
