# Lecture 15 — Elasticsearch and Full-Text Search
> Source: [Sriniously — YouTube](https://www.youtube.com/watch?v=estH64OkwxU)  
> Series: Backend from First Principles

---

## What This Lecture Is About

Why does a `LIKE` query in PostgreSQL become painfully slow at scale? Why does Google return relevant results instantly despite searching billions of pages? This lecture answers those questions by explaining how full-text search engines work, what makes Elasticsearch special, and when to use it as a backend engineer.

---

## 1. The Problem — Why SQL LIKE Queries Break at Scale

### The 2005 E-Commerce Story

Imagine you're a backend engineer at a growing e-commerce company in 2005. You have 5,000 products and you write a simple search query:

```sql
SELECT * FROM products
WHERE name ILIKE '%laptop%'
OR description ILIKE '%laptop%';
```

At 5,000 products → returns in 50 milliseconds. Works great.

Company grows → now you have **millions of products**.

Same query → takes **30 seconds**. Customers leave. Manager is angry.

But it's not just speed. Customers also want:
1. **Relevance** — "laptop" should show MacBook Pro first, not laptop bags
2. **Typo tolerance** — "laптop" (typo) should still return laptop results
3. **Speed** — search results in milliseconds, not seconds

None of these are possible with `ILIKE + %wildcard%`.

---

## 2. Why SQL LIKE Queries Are Slow — The Librarian Analogy

Think of your PostgreSQL database as a **librarian** in a huge library.

You ask: "Find me all books about machine learning."

The librarian's approach:
1. Goes to shelf 1 — picks up "Harry Potter" — checks title/content — no match — moves on
2. Goes to shelf 2 — picks up "Game of Thrones" — checks — no match — moves on
3. Eventually finds "Introduction to Machine Learning" — match!
4. Continues through **every single book** in the entire library

This is called a **full table scan** — and it's exactly what `ILIKE '%term%'` does.

**Two fatal flaws:**
1. **Speed:** At 1 million rows, the database must scan every single row character by character. Painfully slow.
2. **No relevance:** Results returned in arbitrary order. A book where "machine learning" appears once in the last page ranks the same as a book titled "Introduction to Machine Learning". Completely wrong.

---

## 3. The Solution — Inverted Index

The revolution came from **flipping the problem**.

Instead of:
> "Go through every document and check if the search term is in it"

What if:
> "Go through every term in every document and record which documents contain it"

This is called an **inverted index**.

### How It Works

When documents are stored, you build an index that maps **each word → list of documents that contain it**:

```
Term        → Documents (and where they appear)
"machine"   → [Introduction to ML (pages 1,15,23), The Machine Age (pages 5,89), Coffee Machine Manual (page 1)]
"learning"  → [Introduction to ML (pages 1,16,24), Learning to Cook (pages 3,7), Deep Learning Fundamentals (pages 2,8,14)]
"deep"      → [Deep Learning Fundamentals (pages 1,3,8)]
"coffee"    → [Coffee Machine Manual (pages 1,5,12)]
```

Now when someone searches "machine learning":
1. Look up "machine" in the index → instant list of documents
2. Look up "learning" in the index → instant list of documents
3. Find documents that contain both → those are the results
4. Rank by relevance (how many times each term appears, where it appears, etc.)

**No scanning every document.** Direct lookup. Extremely fast.

> 💡 This is the same technique Google uses. They pre-build an inverted index of the entire internet before you even type your search.

---

## 4. Elasticsearch — What It Is

Elasticsearch is a **full-text search engine** built on top of **Apache Lucene**.

- **Apache Lucene** is the underlying library that implements the inverted index technology
- **Elasticsearch** is a distributed, production-ready wrapper around Lucene
- Other tools also use Lucene (including PostgreSQL's built-in full-text search feature)

### What Elasticsearch adds on top of inverted index:

1. **Relevance scoring** — doesn't just find matches, ranks them by importance
2. **Typo tolerance** — finds "trending" even when you type "treading"
3. **Field boosting** — match in title counts more than match in description
4. **Distributed** — scales horizontally across many servers (shards)
5. **Real-time** — newly indexed documents are searchable almost instantly
6. **Rich query language** — JSON-based DSL with many search capabilities

---

## 5. Relevance Scoring — BM25 Algorithm

Elasticsearch uses the **BM25 algorithm** to score and rank results. You don't need to understand the math, but understand the factors:

### Factors That Affect a Document's Relevance Score

| Factor | What it is | Effect |
|---|---|---|
| **Term Frequency (TF)** | How often the search term appears in this document | More occurrences → higher score |
| **Document Frequency (DF)** | How common the term is across ALL documents | Very common words ("the", "and") → lower weight |
| **Document Length** | How long the document is | Same term count in shorter doc = more relevant |
| **Field Boosting** | Where the term appears (title vs description vs content) | Title match > description match > content match |

### Field Boosting Example

```
Search: "machine learning"

Document A: Title = "Introduction to Machine Learning"
            → Term in TITLE → HIGH relevance score

Document B: Description = "A book about computers... machine learning mentioned briefly"
            → Term in DESCRIPTION → MEDIUM relevance score

Document C: Content = "...page 347 mentions machine learning once..."
            → Term in CONTENT only → LOW relevance score

Ranking: A first, B second, C third ✅
```

You can also **customise field boosts** in your queries:
```json
{
  "query": {
    "multi_match": {
      "query": "machine learning",
      "fields": ["title^3", "description^2", "content^1"]
    }
  }
}
```
`^3` means "title matches count 3x more than content matches".

---

## 6. Typo Tolerance (Fuzzy Search)

One of Elasticsearch's most loved features.

```
User types: "treading today"
Elasticsearch understands: "trending today" ✅

User types: "laптop"
Elasticsearch returns: laptop results ✅
```

This works through **fuzzy matching** — Elasticsearch calculates the "edit distance" (number of character changes needed to transform one word into another). If within a threshold, it treats them as the same word.

```json
{
  "query": {
    "fuzzy": {
      "title": {
        "value": "laптop",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

`AUTO` means Elasticsearch decides the fuzziness threshold based on word length.

---

## 7. Key Concepts in Elasticsearch

### Terminology (vs SQL)

| SQL | Elasticsearch |
|---|---|
| Database | Index |
| Table | (no equivalent — an index = a table) |
| Row | Document (JSON) |
| Column | Field |
| Schema | Mapping |

### Documents
Everything in Elasticsearch is a **JSON document**:
```json
{
  "id": "123",
  "title": "Introduction to Machine Learning",
  "description": "A comprehensive guide to ML algorithms",
  "content": "Machine learning is a subset of...",
  "published_at": "2024-01-15"
}
```

### Mapping (Schema)
Before indexing, you define a mapping that tells Elasticsearch how to treat each field:
```json
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },         // full-text search (tokenised)
      "sentiment": { "type": "keyword" },  // exact match only
      "published_at": { "type": "date" },  // date operations
      "rating": { "type": "float" }        // numeric operations
    }
  }
}
```

**`text` vs `keyword`:**
- `text` → tokenised, full-text search, analysed for relevance
- `keyword` → exact match only, used for filtering/aggregation (e.g., status = "active")

---

## 8. The Speed Difference — Real Demo Results

The lecture includes a live demo: 50,000 product reviews in both PostgreSQL and Elasticsearch.

Same search query run against both:

| Database | Query Type | Time |
|---|---|---|
| PostgreSQL | `ILIKE '%laptop%'` | **3–7.5 seconds** |
| Elasticsearch | Full-text query | **500 milliseconds** |

**That's 6–15x faster for the same results.**

At 50,000 rows, both return the same number of results. But as data scales to millions of rows, the gap widens dramatically. PostgreSQL's ILIKE performance degrades linearly — Elasticsearch's performance stays nearly constant.

---

## 9. Use Cases for Elasticsearch

### 1. Product / Content Search
Any search box where users type free-form text and expect smart results:
- E-commerce product search (Amazon-style)
- Job search (LinkedIn)
- Article/blog search
- App store search

### 2. Typeahead / Autocomplete
As user types → instant suggestions (powered by Elasticsearch's near-instant response):
```
User types: "mach"
Suggestions: "machine learning", "machine vision", "machining tools"
```

### 3. Log Analytics (ELK Stack)
The most famous use of Elasticsearch in production is the **ELK Stack**:
- **E**lasticsearch — stores and searches logs
- **L**ogstash — collects and processes logs from various sources
- **K**ibana — visualises logs and creates dashboards

Companies use ELK to search through billions of log lines in milliseconds — finding errors, tracing requests, analysing usage patterns.

### 4. Social Media Search
Searching through posts, profiles, comments — full-text with relevance ranking.

---

## 10. Elasticsearch vs PostgreSQL Full-Text Search

PostgreSQL actually has a built-in full-text search feature using `tsvector` and `tsquery` — also based on inverted index.

| | PostgreSQL Full-Text | Elasticsearch |
|---|---|---|
| **Speed** | Fast (but slower than ES at scale) | Very fast |
| **Setup** | Already in your DB — no new infra | Separate service |
| **Typo tolerance** | Limited | Excellent |
| **Relevance scoring** | Basic | Advanced (BM25) |
| **Scalability** | Limited | Highly scalable (distributed) |
| **Aggregations** | Basic | Powerful |
| **Use case** | Small-medium scale, simple search | Large scale, complex search |
| **Operational cost** | Zero (already running) | Additional infrastructure |

### When to Choose What

**Use PostgreSQL Full-Text Search when:**
- Small to medium dataset (< few million rows)
- You don't want another service to manage
- Search requirements are simple
- Your company doesn't already use Elasticsearch

**Use Elasticsearch when:**
- Large dataset (millions+ documents)
- Advanced relevance scoring needed
- Typo tolerance is important
- Your company already runs ELK for logs
- You need complex aggregations and analytics
- Building a search-heavy product

---

## 11. How to Think About It as a Backend Engineer

Sriniously's honest take (from the lecture):

> "Knowing the knowledge of Elasticsearch is not as important as the knowledge of databases. Database knowledge is something you absolutely have to master. But Elasticsearch is something you can get away with just by copy-pasting some snippets from any LLM or any docs."

**Priority as a backend engineer:**
1. **Master databases** — indexes, query optimization, schema design — this is 99% of your daily work
2. **Know Elasticsearch exists** — know WHEN to use it (full-text search, log analytics)
3. **Know how to implement it** — docs + snippets are usually sufficient for most use cases
4. **Deep Elasticsearch optimisation** — only if you're building a search-heavy product

---

## 12. Keeping Data in Sync (ES + PostgreSQL)

In production, you typically have **PostgreSQL as source of truth** and **Elasticsearch as the search layer**.

Common sync strategies:

**1. Sync on Write (Write-Through)**
```javascript
async function createProduct(data) {
  // Step 1: Write to PostgreSQL (source of truth)
  const product = await db.insert('products', data);

  // Step 2: Index in Elasticsearch
  await esClient.index({
    index: 'products',
    id: product.id,
    document: {
      title: product.title,
      description: product.description,
      price: product.price,
    }
  });

  return product;
}
```

**2. Event-Driven Sync (using CDC — Change Data Capture)**
Database emits events on changes → Kafka/queue carries them → consumer updates Elasticsearch. More complex but more reliable at scale.

**3. Periodic Reindex**
Run a script every X minutes/hours that syncs new/changed records from PostgreSQL to Elasticsearch. Simple but introduces delay.

---

## Summary

| Concept | Key Point |
|---|---|
| **Why SQL search fails** | Full table scan — checks every row. No relevance. Breaks at scale. |
| **Inverted index** | Maps terms → documents (instead of scanning documents for terms). Extremely fast. |
| **Elasticsearch** | Built on Apache Lucene's inverted index. Adds relevance scoring, typo tolerance, distributed search. |
| **Relevance scoring** | BM25 algorithm. Considers term frequency, document frequency, document length, field position. |
| **Field boosting** | Title match > description match > content match. Configurable. |
| **Typo tolerance** | Fuzzy matching — finds "trending" even if you type "treading". |
| **`text` vs `keyword`** | text = full-text search. keyword = exact match for filtering. |
| **ELK Stack** | Elasticsearch + Logstash + Kibana — industry standard for log management. |
| **Speed** | ES: ~500ms. PostgreSQL ILIKE: ~7.5s. 15x faster on same 50k records. |
| **When to use ES** | Large datasets, complex search, typo tolerance, your company uses ELK |
| **When to use PG FTS** | Small-medium data, simple search, don't want extra infrastructure |
| **Sync strategy** | PostgreSQL = source of truth. Elasticsearch = search layer. Sync on write or via CDC. |

---

## One-Line Takeaways

> SQL `LIKE` queries scan every row — they're thorough but slow and return results in random order. Elasticsearch uses an inverted index — blazing fast and relevance-ranked.

> Elasticsearch is a specialist tool for search. PostgreSQL is your everyday database. Know which problems each one solves.

> As a backend engineer: master databases deeply, know Elasticsearch well enough to implement it when needed.

---

*Next lecture → Error Handling — Building robust, predictable error responses across your entire backend.*
