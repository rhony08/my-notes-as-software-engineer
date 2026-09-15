# Vector Search: Finding Nearest Neighbors at Scale

Embeddings get you halfway there: every document is now a point in space, and "similar meaning" means "close together." The other half is answering the actual query — *give me the K nearest points to this one* — fast, with millions of points, while users wait.

That's the whole ballgame. Storing vectors is easy. Searching them at scale is where the interesting engineering lives.

## The Naive Answer (and Why It Doesn't Scale)

The obvious approach is a brute-force scan: compare the query against every vector, keep the top K.

```python
import numpy as np

def brute_force_search(query, matrix, k=5):
    # matrix: (n_docs, dims), all rows normalized to unit length
    scores = matrix @ query                 # one dot product per doc
    idx = np.argpartition(-scores, k)[:k]   # top-k without a full sort
    return idx[np.argsort(-scores[idx])]
```

This is *exact* — you can't beat it on recall. The problem is cost. A single 1536-dim comparison is ~1536 multiply-adds, and you're doing one per document, per query.

| Corpus size | Dims | Comparisons per query | Rough brute-force time |
|---|---|---|---|
| 10k | 1536 | 15M | ~10–30 ms |
| 1M | 1536 | 1.5B | ~1–3 s |
| 100M | 1536 | 153B | minutes |

So brute force is fine for a demo, fine for <100k vectors with tight caching, and a non-starter beyond that. Real systems use **approximate nearest neighbor (ANN)** search: give up a little accuracy to gain orders of magnitude in speed.

## The Trade-off You're Actually Managing

Everything in vector search is a dial between three things: **recall, latency, and memory**.

- **Recall** — of the true top-K neighbors, how many did you actually return? `recall@10 = 0.95` means you found 95% of the real top 10.
- **Latency** — how long a query takes.
- **Memory** — how much RAM the index eats.

You can't max all three. ANN indexes let you slide along this curve — more recall costs speed or RAM, less RAM costs recall. Tuning is picking your point on that curve, not hunting for a magic setting.

## Similarity Metrics: Pick One and Be Consistent

Before indexes, the metric. This is the part people get subtly wrong.

```python
# All three are equivalent IF vectors are normalized to unit length:
#   cosine_similarity(a, b) == dot_product(a, b) == 1 - (L2_distance(a, b) ** 2) / 2
```

| Metric | Measures | Typical use |
|---|---|---|
| Cosine similarity | Angle between vectors | Text embeddings (direction, not length) |
| Inner/dot product | Angle × magnitude | Models trained for it (some recommenders) |
| Euclidean (L2) | Straight-line distance | Image/generic, normalized comparisons |
| Jaccard / Hamming | Set overlap / bit differences | Binary vectors, hashes, dedup |

The critical rule: **the metric must match how the embedding model was trained and how the index was built.** An HNSW index built for cosine is not a dot-product index. Swapping metrics after the fact doesn't quietly work — it silently degrades recall.

```python
# ❌ Store raw vectors, then query with a different metric each time
index = build_index(vectors, metric="l2")
results = index.search(query, metric="cosine")   # mismatch → garbage neighbors

# ✅ Decide the metric once, normalize for it, and keep it fixed
vectors = normalize(vectors)         # makes cosine == dot product
index = build_index(vectors, metric="dot")
```

## ANN Indexes: The Main Families

### HNSW — the graph you'll reach for first

Hierarchical Navigable Small World builds a multi-layer proximity graph. Search walks greedily from a coarse top layer down to a fine bottom layer, hopping toward the query. It's the default in most vector DBs because it nails the recall/latency curve.

```python
# Conceptual HNSW params
HNSW(
    M=16,                # neighbors per node — higher = better recall, more RAM
    ef_construction=200, # build-time search width — better graph, slower build
    ef_search=64,        # query-time search width — the main latency/recall dial
)
```

`ef_search` is the knob you tune at runtime: bump it for more recall, drop it for lower latency. No rebuild needed.

### IVF — cluster first, scan a few buckets

Inverted File index runs k-means to bucket vectors into `nlist` cells. At query time you only scan the `nprobe` closest cells.

```python
IVF(nlist=4096, nprobe=16)   # nprobe: buckets to scan; higher = better recall, slower
```

Cheap on memory and great for very large corpora, but recall is sensitive to whether your query's true neighbors happen to sit in an adjacent cell. Retrain when the data drifts.

### Product Quantization — trade accuracy for RAM

PQ compresses each vector into a short code by splitting it into subvectors and quantizing each against a learned codebook. A 1536-dim float32 vector (~6KB) shrinks to ~64 bytes — a 100x cut.

| Index | Recall | Latency | Memory | Sweet spot |
|---|---|---|---|---|
| Brute force | 100% | Very high | High | <100k vectors |
| HNSW | High | Low | High | Default for most apps |
| IVF | Medium–High | Low | Medium | Huge corpora, memory-tight |
| IVF + PQ | Medium | Low | Very low | Billions of vectors |

## Filtering: The Trap Nobody Warns You About

Real queries aren't just "nearest neighbors" — they're "nearest neighbors *where tenant_id = 42 and created_at > last week*." How you combine the vector search with the metadata filter changes everything.

- **Post-filtering** — search vectors first, then drop rows that fail the filter. Fast, but if the filter is selective you can end up with almost nothing after dropping.
- **Pre-filtering** — apply the filter first, then search. Accurate, but if the filter leaves a tiny candidate set you've thrown away the index's whole advantage.

```python
# ❌ Post-filter: ask for top 10, then filter — may return zero rows
hits = index.search(query, k=10)
results = [h for h in hits if h.metadata["tenant_id"] == 42]   # often empty

# ✅ Over-fetch, or use an index that filters during traversal
hits = index.search(query, k=200, filter={"tenant_id": 42})    # DB filters as it walks
```

The fix in practice: use a vector store with **filter-aware traversal** (HNSW with filter support, or filtering baked into the graph walk), or over-fetch aggressively and rely on the filter. Ask how your database handles this *before* it bites you in production.

## Measuring Recall (Because You Can't Tune What You Don't Measure)

You can't improve recall without knowing it. Build a small ground-truth set: for a sample of queries, compute the true top-K with brute force, then compare against your ANN index.

```python
def recall_at_k(index, queries, matrix, k=10):
    hits = 0
    for q in queries:
        approx = set(index.search(q, k))
        exact  = set(brute_force_search(q, matrix, k))
        hits += len(approx & exact) / k
    return hits / len(queries)

# Target: 0.95+ for search UX, 0.99+ for anything with money or safety at stake
```

pgvector and Redis both expose recall-relevant knobs (`ef_search`, `nprobe`) — sweep them against your ground truth and pick the cheapest point that clears your bar.

## Practical Rules of Thumb

- **Brute force under ~100k vectors.** Don't add an ANN index you don't need.
- **Decide the metric once** and make it match the model and index. Normalize when you want cosine == dot.
- **Start with HNSW**, using `ef_search` as your runtime latency/recall dial.
- **Reach for IVF / IVF-PQ** only when memory or corpus size forces it.
- **Plan for filtered search early** — post-filtering is the most common "works in dev, empty in prod" bug.
- **Measure recall@K against brute-force ground truth** on a sample; tune on data, not vibes.
- **Re-index when the model changes** — an index is tied to the embedding space that built it.

Get the metric right, pick an index that fits your memory budget, and tune recall against real ground truth — and "semantic search" stops being a research project and becomes a boring, reliable piece of infrastructure.
