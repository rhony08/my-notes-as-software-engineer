# Hybrid Search and Reranking

You shipped vector search. Semantic queries work beautifully — ask "how do I stop my app from crashing when the DB is down" and it nails the circuit-breaker doc. Then someone searches for `ERR_4521` and gets... nothing useful. Or they type `pgvector` and the top hit is a generic paragraph about PostgreSQL. The embedding model doesn't care about rare tokens; it cares about meaning. And exact tokens like error codes, product SKUs, and function names *are* the meaning.

Then there's the other failure. A colleague asks "why is my refund late" and the top result is a doc about sales tax — semantically adjacent, topically wrong. Your retriever fetched the right neighborhood but ranked the wrong house first.

These are two different problems, and neither dense vectors nor keyword search alone fix them. **Hybrid search fixes the recall problem; reranking fixes the ranking problem.** They're the two upgrades that turn a demo RAG pipeline into one people actually trust.

## Dense and Sparse Fail in Opposite Directions

Let's be precise about why a single retriever is never enough.

**Dense (vector) search** embeds the query and documents into the same space and finds nearest neighbors. It's brilliant at paraphrase and intent — "car won't start" matches "vehicle fails to ignite" because the vectors are close. But it's fuzzy on exact strings. Rare tokens get washed out into a mean vector, so `ERR_4521` and `ERR_4531` end up nearly identical.

**Sparse (keyword) search** — think BM25 — scores documents by exact term overlap, weighted by how rare and how repeated each term is. It's surgical on identifiers, names, and jargon. But it has zero notion of meaning. Search "how to fix a broken pipe" and it happily returns the plumbing doc, not the Unix one you wanted.

| | Dense (vectors) | Sparse (BM25) |
|---|---|---|
| Strength | Paraphrase, intent, synonyms | Exact tokens, IDs, names, acronyms |
| Blind spot | Rare/unique strings blur together | No semantic understanding |
| Query "ERR_4521" | Matches vaguely related errors | Matches the exact code |
| Query "car won't start" | Finds "vehicle ignition failure" | Finds docs containing those words |

Notice the blind spots don't overlap. That's the whole reason hybrid works: **the two retrievers make different mistakes, so combining them covers more of the space.** If they failed the same way, fusion would buy you nothing.

## Hybrid Search: Query Twice, Merge Once

Hybrid search runs *both* retrievers for every query, then merges the two ranked lists into one. The run is cheap and the payoff is large.

```text
              ┌─ BM25 (sparse)  → ranked list A
query ────────┤
              └─ ANN  (dense)   → ranked list B
                                     │
                          fuse A + B into one list
                                     │
                                 return top-k
```

The interesting part is the fusion step — how do you combine a BM25 score of `14.2` with a cosine similarity of `0.83`? They live on different scales, so you can't just add them.

### Reciprocal Rank Fusion: The Default

RRF throws away the raw scores and uses only the **rank** of each document in each list. For every doc, you sum `1 / (k + rank)` across the lists it appears in, where `k` is a smoothing constant (usually 60).

```python
def rrf(ranked_lists, k=60, top_n=10):
    scores = {}
    for results in ranked_lists:
        for rank, doc_id in enumerate(results, start=1):
            # A doc ranked #1 in either list gets a big boost.
            # Appearing in BOTH lists beats appearing high in just one.
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)[:top_n]
```

Why it's the sane default:
- **No score normalization needed.** Rank is rank, no matter the scale.
- **Rewards agreement.** A doc in the top 10 of *both* retrievers outranks one that's #1 in one and absent from the other — usually the right call.
- **One parameter** (`k`), and it barely matters. `k=60` is fine almost everywhere.

The trade-off: RRF ignores *how much* better the #1 hit was. If BM25 is wildly confident and dense search is barely above noise, RRF treats them as equal votes. For most workloads that's an acceptable loss; for specialized ones, it isn't.

### Weighted Fusion: When You Want a Dial

If one retriever is genuinely better for your domain, normalize the scores and weight them:

```python
# ❌ Naive add — scales are unrelated, the bigger one silently wins
score = bm25_score + cosine_sim   # 14.2 + 0.83 → BM25 dominates everything

# ✅ Normalize each list to [0,1], then blend deliberately
def weighted_fusion(bm25_scores, dense_scores, alpha=0.5):
    norm = lambda d: {k: v / max(d.values()) for k, v in d.items()}
    b, s = norm(bm25_scores), norm(dense_scores)
    keys = set(b) | set(s)
    return sorted(keys, key=lambda k: alpha * b.get(k, 0) + (1 - alpha) * s.get(k, 0),
                  reverse=True)
```

Now `alpha` is a real knob: crank it toward 1.0 for ID-heavy corpora, toward 0.0 for conversational ones. You just bought tuning work and a normalization scheme to debug — that's the cost of the extra control.

### Wiring It Up (Elasticsearch / OpenSearch)

You don't have to hand-roll fusion. Modern search engines expose hybrid queries with RRF built in:

```json
// Elasticsearch 8.8+ — BM25 + kNN, fused with RRF in one request
{
  "query": {
    "match": { "body": "how do I fix a broken pipe" }
  },
  "knn": {
    "field": "embedding",
    "query_vector": [/* ... */],
    "k": 50,
    "num_candidates": 200
  },
  "rank": { "rrf": { "rank_constant": 60 } },
  "size": 50
}
```

One round trip, both retrievers, fused. This is the fastest path to a hybrid win if your data already lives in a search index.

## Reranking: Two Stages Beat One

Hybrid search fixes what you *retrieve*. Reranking fixes what you *return on top*. It's a second pass — and it exists because of a fundamental trade-off baked into vector search.

### Why Vectors Are Approximate by Design

To make search fast, an embedding model must encode a document **without seeing the query**. Query and doc are embedded independently, then compared. That's called a **bi-encoder**, and it's what makes pre-computation possible: embed all your docs once, then compare against a single query vector at request time.

The cost of that speed: the query never gets to "look at" the document. The model commits to one vector per doc before it knows what you'll ask. Two docs can land close to a query for completely different reasons, and the retriever can't tell.

A **cross-encoder** flips this. It takes the query and a document *together* as a single input and outputs a relevance score, attending across both. It's far more accurate — it reads the pair, not two independent summaries.

```text
Bi-encoder  →  embed(query) · embed(doc)      → fast, pre-computable, approximate
Cross-encoder → model(query, doc) → score      → slow, on-the-fly, precise
```

You can't run a cross-encoder over a million docs — you'd score a million pairs per query. So you don't. You use it to reorder the shortlist.

### The Two-Stage Pattern

```text
retrieve top 100 (cheap, approximate: hybrid BM25 + ANN)
        │
        ▼
rerank that 100 (expensive, precise: cross-encoder)
        │
        ▼
pass top 5 to the LLM
```

Stage one optimizes for **recall** — make sure the right doc is *somewhere* in the top 100. Stage two optimizes for **precision** — make sure the right doc is *first*. This is the single highest-leverage change in most RAG systems, because retrieval recall was probably never your problem; ranking was.

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def two_stage(query, candidates, top_k=5):
    # candidates come from hybrid retrieval — cheap, maybe 50-100 docs
    pairs = [(query, doc.text) for doc in candidates]
    scores = reranker.predict(pairs)          # the query sees each doc now
    ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, _ in ranked[:top_k]]
```

### The Trade-off Is Latency

A cross-encoder is ~10–100× slower per candidate than a dot product, and it runs *per query* — no pre-computation. Reranking 100 docs might add 100–300ms. Reranking 1,000 adds over a second.

So you tune two numbers: how many candidates to retrieve, and how many to return.
- **Retrieve too few** → the right doc never makes the shortlist, and reranking can't rescue it.
- **Retrieve too many** → you pay real latency for candidates that were never going to win.

A reasonable starting point: retrieve 50–100, rerank to 3–5. Then watch your evals — if recall@100 is already ~100%, more candidates won't help; if it's low, reranking can't save you either.

## Late Interaction: The Middle Ground

There's a third option worth knowing. **ColBERT** and friends store one vector *per token* instead of one per document, and score by finding the best match for each query token ("late interaction"). It's more accurate than a single-vector bi-encoder and far faster than a cross-encoder — but it needs much more storage and a specialized index.

| Approach | Accuracy | Speed | Storage | Pre-computable |
|----------|----------|-------|---------|----------------|
| Bi-encoder (vectors) | Low | Fastest | Small | Yes |
| Late interaction (ColBERT) | Medium | Fast | Large | Yes |
| Cross-encoder (reranker) | High | Slowest | N/A | No |

Most teams land on: **hybrid retrieval to get candidates, cross-encoder to order them.** ColBERT is for when you need the accuracy of reranking but can't afford its latency, and you're willing to pay in storage and infrastructure.

## A Realistic Pipeline

Putting it together, from query to LLM prompt:

1. **Dual retrieve** — send the query to BM25 and to vector search in parallel, get ~50 each.
2. **Fuse** — RRF the two lists into one ranked shortlist of ~50–100.
3. **Rerank** — cross-encoder scores the shortlist against the query.
4. **Truncate** — keep the top 3–5, drop the rest.
5. **Generate** — pass the reranked chunks to the LLM.

Each stage narrows and improves. None of them is expensive on its own; the cost is in the *latency you choose to add*. You can ship steps 1–2 today and add 3 when your evals show ranking is the bottleneck.

## How to Tell If It's Working

Don't guess. Measure each stage separately:

- **Recall@k after fusion** — is the right doc in the shortlist at all? If not, the problem is retrieval (dense or sparse quality), not ranking.
- **nDCG / MRR before vs after reranking** — did the reranker move the right doc up? If the ranking doesn't improve, the reranker is the wrong model or the wrong input.
- **Latency budget** — reranking cost must fit your p95 target, not your median.
- **Cost per query** — a rerank call is money on every request, including the ones that don't need it.

Change one stage at a time. Hybrid retrieval and reranking are independent upgrades; adding both at once makes it impossible to tell which one helped.

## Takeaways

- **Dense and sparse fail in opposite directions** — vectors lose exact tokens, BM25 loses meaning. Hybrid covers both blind spots.
- **Fuse with RRF by default.** It's parameter-light, scale-free, and rewards agreement between retrievers.
- **Reranking is a second stage, and it's the biggest quality win in most RAG pipelines.** Retrieve for recall, rerank for precision.
- **Cross-encoders read the query and doc together** — accurate, but per-query and slow. Never run them over your whole corpus.
- **Retrieve 50–100, return 3–5.** Tune those two numbers against your evals, not your intuition.
- **Watch your p95 latency.** Reranking adds real milliseconds on every request — make sure it fits the budget before you ship it.
- **Measure stages separately.** Low recall means fix retrieval; bad ordering means fix reranking. They're different bugs.
