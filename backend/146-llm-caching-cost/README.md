# Caching and Cost Control for LLMs

Your AI feature works. Users love it. Then the first real invoice lands and someone asks why a "chat feature" costs more than the entire database tier.

Here's the thing about LLM costs: they're not fixed infrastructure. They scale with *tokens*, and tokens scale with every user, every retry, every fat context window, and every model you picked because "better is safer." Nothing in your existing stack throttles this. There's no autoscaler for "we accidentally sent 40k tokens of boilerplate on every request."

Caching is the highest-leverage fix — it turns repeated work into a lookup. But LLM caching isn't one thing. There are at least four layers, and the right mix depends on your traffic shape. Let's go through them, then look at the cost levers beyond caching.

## Where the Money Actually Goes

Before caching anything, know what you're paying for. Most surprises come from these:

| Cost driver | Why it's expensive | Typical culprit |
|-------------|-------------------|-----------------|
| Prompt/completion tokens | Billed per token, input and output | Long system prompts, full chat history resent each call |
| Model choice | Frontier models cost 10–50× small ones | Defaulting everything to the biggest model |
| Retries | Every retry re-charges the full prompt | Timeouts, rate limits, fragile parsing |
| Redundant calls | Identical work done repeatedly | Same FAQs, same summaries, same embeddings |
| Context bloat | Tokens grow with conversation length | Never trimming history |

The uncomfortable truth: a lot of LLM spend is *repeated* work. The same question asked by 200 users. The same document embedded twice. The same system prompt prepended to 10,000 calls a day. That's what caching attacks.

## The Four Layers of LLM Caching

They stack. Each one is cheaper and simpler than the last, and you should reach for them in roughly this order.

| Layer | What it caches | Hit rate | Effort |
|-------|---------------|----------|--------|
| 1. Exact-match response cache | Full request → full response | Low–medium | Low |
| 2. Provider prompt caching | The shared *prefix* of prompts | High (many apps) | Low |
| 3. Semantic cache | Similar-meaning requests | Medium–high | Medium |
| 4. Embedding cache | Vectors for unchanged text | Very high | Low |

### Layer 1: Exact-Match Response Cache

The simplest win. Hash the request (model + messages + params), store the response, return it on a repeat. Excellent for deterministic things — classification, extraction, template-generated copy, and any request with `temperature: 0`.

```python
import hashlib, json, redis

r = redis.Redis()

def cache_key(model: str, messages: list, **params) -> str:
    payload = json.dumps(
        {"model": model, "messages": messages, **params},
        sort_keys=True,  # stable ordering, otherwise the hash is useless
    )
    return "llm:" + hashlib.sha256(payload.encode()).hexdigest()

def cached_completion(model: str, messages: list, ttl: int = 3600, **params):
    key = cache_key(model, messages, **params)
    if (hit := r.get(key)):
        return json.loads(hit)

    resp = call_llm(model, messages, **params)
    r.setex(key, ttl, json.dumps(resp))
    return resp
```

The trap: **don't cache non-deterministic calls.** If `temperature > 0`, two identical prompts can legitimately return different answers, and serving a stale one may be fine — or may be wrong. Decide per use case. Low temperature + stable content = cache it. Creative generation = don't.

### Layer 2: Provider Prompt Caching

This is the one people miss, and it's often the biggest saver. If your prompts share a long prefix — a system prompt, tool definitions, a big document — providers like Anthropic and OpenAI will cache that prefix *on their side* and bill the repeated portion at a fraction of the price.

The catch is you have to **structure prompts so the prefix is stable**. Anything that changes near the top invalidates the cache for everything after it.

```python
# ❌ Timestamp near the top busts the prefix cache on every single call
messages = [
    {"role": "system", "content": f"You are a support agent. Current time: {now()}"},
    {"role": "system", "content": LONG_POLICY_DOC},   # 8k tokens, never changes
    {"role": "user", "content": user_question},
]

# ✅ Stable content first, volatile content last
messages = [
    {"role": "system", "content": LONG_POLICY_DOC},   # 8k tokens, cacheable prefix
    {"role": "system", "content": f"Current time: {now()}"},  # volatile, tiny
    {"role": "user", "content": user_question},
]
```

Same tokens. Very different bill. Order matters more than you'd think.

### Layer 3: Semantic Cache

Exact-match caching fails the moment a user rephrases the same question. "How do I reset my password?" and "I forgot my password, how do I change it?" are the same intent — and a vector cache can catch that.

Embed the incoming query, search for a close-enough previous query, and return its cached answer if the similarity clears a threshold.

```python
def semantic_lookup(query: str, threshold: float = 0.95):
    vec = embed(query)
    match = vector_store.search(vec, top_k=1)
    if match and match.score >= threshold:
        return match.response
    return None
```

**Be careful here.** This is where caching goes from "free money" to "why is the bot giving wrong answers." A threshold that's too loose serves a plausible-but-wrong answer to a subtly different question. That's worse than paying for a call. Start high (0.95+), measure, and tighten if you see bad hits. Also:

- Scope the cache by user/tenant if answers are personalized — otherwise you leak one user's data to another.
- Version the cache key by prompt template, so a prompt change doesn't serve old-template answers.

### Layer 4: Embedding Cache

Embeddings are cheap per call but get expensive at scale, and they're perfectly deterministic. Hash the text, cache the vector, and never pay to embed the same string twice.

```python
def embed_cached(text: str) -> list[float]:
    key = "emb:" + hashlib.sha256(text.encode()).hexdigest()
    if (hit := r.get(key)):
        return json.loads(hit)
    vec = embed_api(text)
    r.set(key, json.dumps(vec))  # no TTL — the text→vector mapping never changes
    return vec
```

For a RAG pipeline that re-ingests the same corpus on every deploy, this alone can cut a big chunk of spend.

## Beyond Caching: The Other Cost Levers

Caching handles repetition. These handle the rest.

**Route to the cheapest model that works.** Don't send everything to the frontier model. A cascade — try the small model, escalate on low confidence — is shockingly effective for classification and extraction.

**Cap the output.** You're billed for completion tokens, and models love to ramble. `max_tokens` and "answer in one sentence" instructions are direct cost cuts.

**Trim the history.** Resending the full chat every turn means turn 20 costs 20× turn 1. Summarize or window older turns.

**Track cost per request, not just total.** If you can't attribute spend to a feature, user, or endpoint, you can't optimize it.

```python
# ❌ Happy path only — you learn about cost spikes from the invoice
resp = client.chat.completions.create(...)

# ✅ Capture tokens and computed cost on every call
resp = client.chat.completions.create(...)
cost = resp.usage.prompt_tokens * IN_PRICE + resp.usage.completion_tokens * OUT_PRICE
metrics.increment("llm.cost_usd", cost, tags={"model": model, "feature": "faq"})
```

## Measuring Whether It's Working

You can't manage what you don't measure. Watch a handful of numbers:

| Metric | Why it matters |
|--------|---------------|
| Cache hit rate (per layer) | Is caching actually firing, or just adding latency? |
| Cost per request | The number that shows up on the invoice |
| Tokens per request (in/out) | Explains *why* cost moved |
| P95 latency with/without cache | Caching should make things faster, not slower |
| Semantic cache false-hit rate | Sampled — catches over-aggressive thresholds |

If hit rate is near zero, your keys are probably too specific (timestamps, request IDs, per-user scoping). That's the most common reason a cache "doesn't work."

## What to Watch Out For

- **Stale answers.** A cache is a bet that the answer hasn't changed. Set TTLs by how volatile the underlying data is — seconds for prices, days for docs.
- **Personalization leaks.** Caching across users is only safe when the answer doesn't depend on who's asking.
- **Cache invalidation on prompt changes.** Version your keys, or a prompt tweak will be silently masked by old responses.
- **Over-caching.** If you cache `temperature > 0` outputs and users expected variety, you've traded cost for a worse product.

## Takeaways You Can Ship Today

1. Log cost, tokens, and cache hits *before* optimizing — you can't see what you don't measure.
2. Add exact-match caching for deterministic calls; it's an afternoon of work.
3. Reorder prompts so stable prefixes come first and enable provider prompt caching.
4. Cache embeddings — deterministic and free to reuse.
5. Add a semantic cache only with a high similarity threshold and per-tenant scoping.
6. Route small tasks to small models, cap `max_tokens`, and trim chat history.
7. Version every cache key by prompt template so changes never serve stale answers.
