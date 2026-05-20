# Search Engines: Elasticsearch Basics

Your app's SQL database is great for storing orders, users, and transactions. But ask it to do full-text search across millions of documents with relevance scoring, fuzzy matching, and typo tolerance, and it starts wheezing.

```sql
-- ❌ This kills performance on large datasets
SELECT * FROM products WHERE name LIKE '%erephon%';
```

That `LIKE '%term%'` query? Full table scan. Every time. And it doesn't even understand relevance—just returns matching rows in arbitrary order.

That's where Elasticsearch comes in.

---

## What Elasticsearch Actually Is

Elasticsearch is a **distributed search and analytics engine** built on Apache Lucene. Under the hood, it creates **inverted indexes**—think of it like the index at the back of a book, mapping every word to every document that contains it.

```
Document 1: "The quick brown fox"
Document 2: "The lazy dog"

Inverted index:
  "the"    → [doc1, doc2]
  "quick"  → [doc1]
  "brown"  → [doc1]
  "fox"    → [doc1]
  "lazy"   → [doc2]
  "dog"    → [doc2]
```

When you search for "fox," it doesn't scan documents—it checks the index and instantly knows doc1 matches. **O(1) lookup on words instead of O(n) full scans.** That's the whole superpower.

---

## Key Concepts You Need to Know

### Documents & Indices

| SQL Term | Elasticsearch Term |
|----------|-------------------|
| Database | Cluster |
| Table | Index |
| Row | Document |
| Column | Field |
| Schema | Mapping |

A **document** is JSON. An **index** is a collection of documents. That's it.

```json
// An Elasticsearch document
{
  "id": 1,
  "title": "Wireless Bluetooth Headphones",
  "description": "Noise cancelling over-ear headphones with 30hr battery",
  "price": 89.99,
  "category": "electronics",
  "in_stock": true,
  "tags": ["audio", "bluetooth", "wireless"]
}
```

### Mappings (Schema)

Elasticsearch can infer types from your data (dynamic mapping), but you should **always define explicit mappings** for fields that matter:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "standard"
      },
      "description": {
        "type": "text"
      },
      "price": {
        "type": "float"
      },
      "category": {
        "type": "keyword"
      },
      "in_stock": {
        "type": "boolean"
      },
      "tags": {
        "type": "keyword"
      }
    }
  }
}
```

**`text`** vs **`keyword`**: This one trips everyone up.

- `text` → analyzed, split into tokens, good for full-text search
- `keyword` → exact match, good for filters, aggregations, sorting

Use `text` for the description you want to search through. Use `keyword` for the category you want to filter on. Mix them up and you'll wonder why your filters aren't working.

---

## How Searching Works

```json
// Searching for headphones
GET /products/_search
{
  "query": {
    "match": {
      "title": "bluetooth headphones"
    }
  }
}
```

Elasticsearch will:
1. Analyze the query ("bluetooth" + "headphones")
2. Look up both terms in the inverted index
3. Score each document by relevance (TF-IDF or BM25 by default)
4. Return results sorted by score, descending

The response includes a `_score` field—the higher the score, the better the match.

### Fuzzy Search

Users make typos. Elasticsearch handles this with `fuzziness`:

```json
GET /products/_search
{
  "query": {
    "fuzzy": {
      "title": {
        "value": "erephone",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

With `fuzziness: "AUTO"`, it'll match "erephone" against "headphones" by allowing up to 2 character edits (insertions, deletions, substitutions). Your users don't need to spell things perfectly.

### Filters vs Queries

This is the performance optimization most people miss:

```json
// ✅ Use filters for yes/no conditions
// They're cached and don't affect scoring
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "noise cancelling" } }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "lte": 100 } } }
      ]
    }
  }
}
```

**Filters** are binary (yes/no) and **cached in memory**. Queries compute relevance scores. Use `filter` for things like "in stock" or "price range"—you don't need a relevance score for those, and the caching makes subsequent queries blazing fast.

---

## When Elasticsearch Is the Wrong Tool

It's not a silver bullet. Here's where it falls apart:

| Use Case | Why ES Is Wrong | What to Use Instead |
|----------|----------------|-------------------|
| ACID transactions | No transactional guarantees | PostgreSQL |
| Complex joins | Denormalization is the norm | Relational DB |
| Primary data store | Eventual consistency; can lose data | Your main DB |
| Ad-hoc analytics | Better tools exist | Clickhouse, Druid |
| Simple lookups by ID | Overkill | Redis, DynamoDB |

**Real talk:** Elasticsearch should be a **secondary index** to your primary database, not a replacement. Index data into ES for search, but keep the source of truth in PostgreSQL or MySQL. If ES goes down, your app should still work—just without search.

---

## Indexing from Your App

The most common pattern: write to your primary DB, then async-index to Elasticsearch.

```python
# ✅ Async indexing via a background job
def on_product_updated(product):
    # 1. Save to primary database (source of truth)
    db.products.update(product)
    
    # 2. Queue indexing job (async, fire-and-forget)
    index_job.delay({
        "_index": "products",
        "_id": product.id,
        "_source": {
            "title": product.title,
            "description": product.description,
            "price": product.price,
            "category": product.category,
            "in_stock": product.in_stock,
            "tags": product.tags,
            "updated_at": product.updated_at.isoformat()
        }
    })
```

Or use **Change Data Capture (CDC)** tools like Kafka Connect + Debezium to automatically stream DB changes into ES. That way you don't have to instrument every write path.

---

## Trade-offs Worth Knowing

**Indexing latency:** Documents aren't immediately searchable. There's a refresh interval (default 1s) before new data appears in search results. You can force a refresh, but that's expensive and kills write performance.

**Memory hungry:** ES loves RAM. A lot of it. Lucene relies heavily on OS filesystem cache, and ES itself needs heap space for aggregations and indexing buffers. Underprovision memory and you'll hit circuit breakers mid-search.

**Cluster complexity:** A single-node ES is simple. A multi-node cluster with shards, replicas, and rolling restarts? That's a whole ops discipline. You'll need to learn about shard allocation, rebalancing, and split-brain scenarios.

**No auth by default:** Out of the box, ES has no authentication. Put it behind a reverse proxy or use the Elastic Stack's security features (which require a license). Don't expose it directly to the internet.

---

## Actionable Takeaways

- Use Elasticsearch for **full-text search**—not as your primary database
- Always define **explicit mappings** instead of relying on dynamic inference
- Separate **queries** (scoring) from **filters** (cached, binary) for performance
- Index data **async from your primary DB**—never treat ES as source of truth
- Profile your index mappings: `keyword` for exact match, `text` for analyzed search
- Start with `fuzziness: "AUTO"` to handle typos without over-engineering
- Monitor heap usage and index refresh intervals to avoid OOM crashes
