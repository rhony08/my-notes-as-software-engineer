# Vector Databases: pgvector, Pinecone, and Qdrant

Your first RAG demo probably looked like this: a list of embeddings in memory, a `for` loop computing cosine similarity, and a `sorted()` call. It works beautifully — on 500 chunks. Then real data shows up.

At a million vectors, that loop reads gigabytes and burns seconds of CPU per query. At ten million, the process dies. You didn't hit a bug; you hit the wall that every vector search eventually hits: **brute-force comparison doesn't scale**.

A vector database is the layer that makes "find the nearest points" fast, durable, and filterable. The question isn't *whether* you need one — it's *which* one. Let's cut through the marketing and look at what actually matters.

## What a Vector DB Actually Does

Strip away the branding and a vector database does three things your list-in-memory cannot:

1. **Approximate nearest neighbor (ANN) search** — instead of comparing against every vector, it navigates a smart index to find *very likely* nearest neighbors in milliseconds.
2. **Persistence and scale** — vectors live on disk, survive restarts, and grow past RAM.
3. **Filtered search** — "similar to this, but only where `tenant_id = 42` and `lang = 'en'`."

That second and third point are where naive solutions quietly fall apart. Cosine similarity is easy. Doing it correctly *and* quickly *and* per-tenant is not.

### The Index Is the Product

The heart of any vector DB is its ANN index. Two dominate:

| Index | How it works | Trade-off |
|-------|-------------|-----------|
| **HNSW** | Builds a hierarchical graph; greedy-walks toward the query | Excellent recall + speed, but memory-hungry and slow to build |
| **IVF** | Clusters vectors into buckets; searches only nearby buckets | Cheaper memory, faster build, slightly lower recall |

Most systems expose a knob like `ef_search` or `nprobe` that trades **recall for latency**. Turn it up and you find more true neighbors but answer slower. This is the single most important tuning dial you'll touch, and it's the reason "approximate" is in the name — you're choosing how much recall you're willing to give up for speed.

```python
# The mental model for every ANN tuning knob
# ef_search=16  -> fast, ~90% recall, good for autocomplete
# ef_search=128 -> slower, ~99% recall, good for RAG grounding
# The "right" value comes from your eval set, not from a blog post
```

## The Real Decision: Managed vs Self-Hosted

Before you pick a product, pick a category. This choice affects your bill, your on-call rotation, and how locked-in you are.

**Managed (Pinecone, Weaviate Cloud, Qdrant Cloud):** You get an endpoint and an API key. They handle sharding, replication, failover, upgrades. You pay a premium and your data lives with a vendor.

**Self-hosted (pgvector, Qdrant OSS, Milvus):** You run it. Cheap at scale, full control, no data egress. You also own every 3am page.

There's no universally right answer. The mistake is choosing managed "to move fast" for a workload that's really just 200k vectors next to data you already have in Postgres.

## pgvector: Boring in the Best Way

If you already run Postgres, `pgvector` is the pragmatic default. Vectors become just another column type, and — this is the key part — they live next to your relational data, so filtering is free and consistent.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
  id        bigserial PRIMARY KEY,
  tenant_id bigint NOT NULL,
  content   text   NOT NULL,
  embedding vector(1536)          -- matches your embedding model's dims
);

-- HNSW index: the thing that makes this fast
CREATE INDEX ON documents
  USING hnsw (embedding vector_cosine_ops);
```

Querying is SQL you already know, and filtering is a `WHERE` clause — not a separate "metadata filter" language you have to learn:

```sql
-- ✅ Similar docs, scoped to one tenant, in one consistent query
SELECT id, content
FROM documents
WHERE tenant_id = 42
ORDER BY embedding <=> '[0.013, -0.021, ...]'::vector   -- <=> = cosine distance
LIMIT 5;
```

**Where it shines:** tens of millions of vectors or fewer, data that's already in Postgres, and workloads where consistency between your vectors and your rows matters (it does, more than you think — see below).

**Where it hurts:** you're now running ANN queries on the same box that serves transactions. Under load, a heavy vector search can starve your OLTP queries. At very large scale, dedicated engines win.

## Qdrant: The Engineering-First Middle Ground

Qdrant is an open-source vector engine built *only* for vectors. It's self-hostable with a managed option, written in Rust, and unusually good at filtered search — the case that trips up naive implementations.

Its standout feature is **payload-aware filtering done during the search**, not after it. This sounds like a detail. It isn't:

```python
from qdrant_client import QdrantClient, models

client = QdrantClient(url="http://localhost:6333")

client.create_collection(
    collection_name="docs",
    vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE),
)

client.search(
    collection_name="docs",
    query_vector=query_embedding,
    query_filter=models.Filter(          # filter applied *inside* the index walk
        must=[
            models.FieldCondition(key="tenant_id", match=models.MatchValue(value=42)),
            models.FieldCondition(key="lang", match=models.MatchValue(value="en")),
        ]
    ),
    limit=5,
)
```

Why does "inside" matter? Because of the classic post-filter trap:

```python
# ❌ Post-filtering: fetch 100 neighbors, then throw away 95 that fail the filter
results = db.search(query, limit=100)
filtered = [r for r in results if r.tenant_id == 42]   # maybe 3 survive — useless

# ✅ Filter-aware search: the engine only considers vectors that match
results = db.search(query, filter={"tenant_id": 42}, limit=5)  # all 5 are usable
```

With multi-tenant data, post-filtering means your "top 5" can silently come back as one result, or zero. Qdrant (and pgvector's planner, to a degree) solve this by pushing the filter into the index traversal.

## Pinecone: Managed and Carefree

Pinecone is fully managed, serverless, and API-driven. You don't think about indexes, nodes, or shards — you create an index and upsert vectors.

```python
from pinecone import Pinecone

pc = Pinecone(api_key="...")
index = pc.Index("docs")

index.upsert(vectors=[
    ("doc-1", embedding, {"tenant_id": 42, "lang": "en"}),
])

results = index.query(
    vector=query_embedding,
    top_k=5,
    filter={"tenant_id": {"$eq": 42}},
    include_metadata=True,
)
```

**What you're buying:** zero ops, predictable scale-up, and sane defaults you don't have to tune. **What you're paying for:** per-request/usage pricing that gets expensive at volume, vendor lock-in on both API and data, and less control over index internals.

It's genuinely great for small teams with no infra appetite, or workloads that spike unpredictably and you don't want to capacity-plan for.

## Choosing Without Regret

| You have... | Reach for | Why |
|-------------|-----------|-----|
| Postgres, <10M vectors | **pgvector** | No new infra, filters are just SQL, one source of truth |
| Multi-tenant, heavy filtering | **Qdrant** | Filter-aware search avoids post-filter traps |
| No infra team, spiky load | **Pinecone** | Zero ops, managed scaling |
| 100M+ vectors, cost-sensitive | **Self-hosted Milvus/Qdrant** | Managed pricing becomes brutal at scale |

Two tiebreakers people forget:

- **Dimension churn.** If your embedding model might change, dimension is baked into the schema/index. Plan for re-indexing — changing models means re-embedding *everything*. Pick a DB that makes full re-creation cheap.
- **Consistency.** If your vectors can drift from the source rows (deleted doc, but the vector lingers), you get confidently wrong answers. This is a strong argument for pgvector when your data already lives in Postgres — one transaction, no drift.

## What Breaks in Production

**1. Recall you never measured.** You set `ef_search` to a default, shipped, and quietly serve mediocre results. Build a small labeled set (`question → correct chunk`) and measure recall at your latency budget. Two hours of this beats a week of guessing.

**2. The cost model surprise.** Embedding is cheap; *re-embedding a million docs* after a model upgrade is not. Budget for it, and store your embedding model name + version with each vector so you know what's stale.

**3. Filtering that silently lies.** As shown above, post-filtered "top-k" degrades quietly under selective filters. Test with your *most restrictive* tenant, not your demo data.

**4. Forgetting to normalize.** Cosine vs dot vs euclidean give different rankings. If your embedding model expects cosine, use cosine — don't let the DB default decide your quality.

## Takeaway

Start boring. If your data is in Postgres, `pgvector` will carry you further than you expect and keeps vectors consistent with reality. Reach for Qdrant when filtering is heavy and correctness of "top-k under a filter" matters. Go Pinecone when you'd rather buy the problem than run it — and you're okay paying for that.

Whatever you pick: your vector database is only as good as the recall you actually measure. Pick the index, set the tuning knob, and let your eval set — not the vendor's benchmark — make the final call.
