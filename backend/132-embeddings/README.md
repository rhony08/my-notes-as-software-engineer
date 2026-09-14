# Embeddings: Turning Text into Vectors

You add a search box. Someone types "my card got charged twice" and gets zero results — because your docs say "duplicate transaction" and "billing dispute." The words don't match, so a keyword search shrugs. But a human instantly sees the two mean the same thing.

That gap — between *what words say* and *what they mean* — is exactly what embeddings close. They turn text into numbers positioned so that similar meanings land close together. Once you've got that, search, clustering, dedup, recommendations, and the retrieval half of every RAG system all fall out of the same primitive.

## The Core Idea

An embedding is a fixed-length list of floats — a vector — that represents a piece of text.

```python
"duplicate transaction"  → [0.021, -0.44, 0.19, ..., 0.07]   # 1536 numbers
"charged twice"          → [0.030, -0.41, 0.22, ..., 0.09]   # 1536 numbers
"banana bread recipe"    → [-0.51, 0.88, -0.13, ..., 0.62]   # 1536 numbers
```

Two things to internalize:

1. **The numbers are meaningless on their own.** You can't read vector position 47 and learn anything. Meaning lives in the *relationship between vectors*, not in any single number.
2. **The length is fixed per model.** `text-embedding-3-small` always returns 1536 dims, whether you feed it one word or a paragraph. That's what makes them cheap to store in a column and compare at scale.

The magic is that a model was trained to place semantically similar text near each other. So "duplicate transaction" and "charged twice" end up as neighbors, and "banana bread recipe" ends up in a completely different neighborhood.

## Similarity: How You Measure "Close"

Since meaning is distance, you need a distance function. In practice it's almost always **cosine similarity** — the cosine of the angle between two vectors.

```python
import numpy as np

def cosine_similarity(a, b):
    # 1.0 = identical direction, 0.0 = unrelated, -1.0 = opposite
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

cosine_similarity(embed("duplicate transaction"), embed("charged twice"))
# → 0.87  ← close, these are about the same thing

cosine_similarity(embed("duplicate transaction"), embed("banana bread recipe"))
# → 0.11  ← far apart, unrelated
```

Why cosine and not straight-line (Euclidean) distance? Because cosine ignores magnitude and only cares about *direction*. Two sentences about the same topic can have quite different lengths and therefore different vector magnitudes — but their direction stays similar. Cosine shrugs off that noise.

| Metric | Formula intuition | Use when |
|---|---|---|
| Cosine similarity | Angle between vectors | Text embeddings, most cases |
| Dot product | Angle + magnitude | When vectors are already normalized |
| Euclidean (L2) | Straight-line distance | Geospatial, when magnitude matters |

**Shortcut:** if you normalize every vector to unit length *once at write time*, then cosine similarity collapses to a plain dot product — cheaper to compute. Most vector databases do this for you automatically.

```python
# ❌ Normalizing on every comparison — wasted work in a hot loop
def sim(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# ✅ Normalize once when you store, then dot product is enough
def normalize(v):
    return v / np.linalg.norm(v)

doc_vec = normalize(embed(doc))          # store this normalized
score = float(np.dot(normalize(embed(query)), doc_vec))  # just a dot product
```

## Generating Embeddings in Practice

You don't train these — you call an API or run a small local model. The workflow is boring on purpose: chunk text, embed, store, compare.

```python
from openai import OpenAI
client = OpenAI()

def embed(texts: list[str], model="text-embedding-3-small") -> list[list[float]]:
    # Always batch. One call for N texts beats N calls for one text.
    resp = client.embeddings.create(model=model, input=texts)
    return [d.embedding for d in resp.data]

# Batch a whole document set in a single request
vectors = embed(["duplicate transaction", "charged twice", "banana bread"])
```

Two rules that save you money and pain:

- **Batch aggressively.** Embedding APIs price per token, but per-request latency and rate limits punish one-at-a-time loops. Send 100–500 texts per call.
- **Cache by content hash.** The same string embedded twice should never hit the API twice.

```python
import hashlib

cache = {}  # back this with Redis in production

def embed_cached(text: str, model="text-embedding-3-small"):
    key = hashlib.sha256(f"{model}:{text}".encode()).hexdigest()
    if key not in cache:
        cache[key] = embed([text], model=model)[0]
    return cache[key]
```

**When you re-embed, you re-do everything.** If embeddings are already in your store and you switch models, caching won't save you — vectors from different models live in entirely different coordinate spaces and aren't comparable. More on that below.

## Choosing a Model (and Its Dimension)

| Model | Dims | Notes |
|---|---|---|
| `text-embedding-3-small` | 1536 | Cheap, fast, great default |
| `text-embedding-3-large` | 3072 | Better quality, ~6x the cost |
| `text-embedding-3-small` (shortened) | 512–1024 | Truncate dims to cut storage; small quality dip |
| `bge-*` / `e5-*` / `all-MiniLM` | 384–1024 | Open-weight, run locally, no per-call cost |

The trade you're making: **higher dimensions capture more nuance but cost more to store and compare, and slow down search.** A 3072-dim float32 vector is ~12KB per chunk. A million chunks is 12GB before any index overhead. Halving the dimensions roughly halves the bill — and the newer OpenAI models let you shrink the vector after the fact, which is a genuinely useful knob.

Local open-weight models are the right call when you have privacy constraints, huge volume, or you're fine trading some quality for zero marginal cost. Just remember the model that made the vectors has to be the same model that embeds the queries.

## The Failure Modes That Bite

### Mixing models (or versions)

```python
# ❌ Stored with model v1, querying with model v2 — silently garbage results
stored = embed(docs, model="text-embedding-ada-002")   # 1536 dims
query  = embed(["charged twice"], model="text-embedding-3-small")  # also 1536!
# Same shape, different universe. Similarities are now meaningless.
similarity = cosine_similarity(stored[0], query[0])  # confidently wrong
```

This is the nastiest bug because it *works* — no error, no shape mismatch if the dims happen to line up, just quietly bad answers. **Pin the model name with the version, store it alongside the vectors, and treat a model change as a full re-index migration.**

### Comparing raw scores across queries

Cosine scores aren't calibrated across different queries. "0.82" might be a great match for one query and mediocre for another. Don't hardcode a global threshold — tune it on real data, per use case, and remember the distribution shifts with your model.

### Forgetting symmetric vs asymmetric

Some models are trained for **symmetric** tasks (finding similar sentences) and some for **asymmetric** ones (short query → long passage). Using the wrong one is like asking a question in a language the model doesn't speak. If your model has separate query/document prefixes (e5 and bge do), using them correctly is worth several points of recall:

```python
# ❌ e5 models expect these prefixes; omitting them hurts retrieval
doc_vec   = embed(["duplicate transaction explanation"])

# ✅ Match how the model was trained
doc_vec   = embed(["passage: duplicate transaction explanation"])
query_vec = embed(["query: charged twice"])
```

### Thinking embeddings understand everything

Embeddings capture *topical/semantic* similarity — not facts, not negation, not exact numbers. "The server is up" and "the server is down" can score surprisingly close because the topic is identical. If you need precision on numbers, names, or `NOT` logic, embeddings alone will let you down. That's why keyword search still earns its place next to them (and why hybrid search exists).

## Practical Rules of Thumb

- **Normalize once at write time**, then use dot product — cheaper comparisons and one less thing per query.
- **Batch your embedding calls and cache by hash.** Both are easy wins, both are routinely skipped.
- **Store the model name *and* version with every vector.** A model swap is a re-index, not a config flip.
- **Tune similarity thresholds on real data**, not vibes. The right cutoff is per-use-case.
- **Respect symmetric vs asymmetric** — use the prefixes and query/passage split the model expects.
- **Don't ask embeddings to do keyword's job.** For exact terms, IDs, and negation, pair them with lexical search.

Get comfortable with embeddings and a lot of "AI features" stop being mysterious. Search that understands paraphrases, dedup that catches near-identical tickets, clustering that discovers topics nobody labeled — it's all the same move: turn text into vectors, put them near their meaning, then measure the distance.
