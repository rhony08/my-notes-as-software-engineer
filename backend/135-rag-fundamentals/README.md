# RAG: Retrieval-Augmented Generation from Scratch

Ask an LLM about your company's internal docs and it will confidently invent an answer. It doesn't know your data — it knows the internet up to its training cutoff, compressed into weights. Fine-tuning is one fix, but it's slow, expensive, and it forgets the moment your docs change.

RAG is the other fix, and for most "the model should know about *our* stuff" problems, it's the right one. Instead of baking knowledge into the model, you fetch the relevant chunks at query time and hand them to the model as context. The model stays general; your data stays fresh in a database you control.

Most RAG articles jump straight to a framework. That hides the mechanics — and the mechanics are where RAG fails. So let's build the naive version first, watch it break, then fix it piece by piece.

## The Whole Idea in One Diagram

```text
Indexing (offline):   docs → chunk → embed → store vectors
Query   (online):     question → embed → search top-k → stuff into prompt → LLM → answer
```

That's it. Two pipelines: one that prepares your knowledge, one that uses it. Everything that goes wrong lives in the details of these five steps.

## Step 1: Chunk the Documents

You can't embed a 40-page PDF as one vector — the meaning gets averaged into mush, and it won't fit in the prompt anyway. So you split documents into chunks small enough to be specific, large enough to carry meaning.

The classic mistake is fixed-size splitting:

```python
# ❌ Splits mid-sentence, mid-thought — retrieval returns half-answers
chunks = [text[i:i+500] for i in range(0, len(text), 500)]
```

Chunk on **structure**, not character count. Split by paragraph or heading, then merge small pieces up to a target size with overlap so a sentence cut at a boundary still appears whole in at least one chunk.

```python
# ✅ Paragraph-aware, with overlap so boundaries don't lose context
def chunk_text(text, target=800, overlap=100):
    paras = [p.strip() for p in text.split("\n\n") if p.strip()]
    chunks, buf = [], ""
    for p in paras:
        if len(buf) + len(p) > target and buf:
            chunks.append(buf)
            buf = buf[-overlap:] + "\n\n" + p   # carry tail into next chunk
        else:
            buf = (buf + "\n\n" + p).strip()
    if buf:
        chunks.append(buf)
    return chunks
```

Two rules of thumb worth internalizing:

- **Chunk size is a trade-off, not a setting.** Small chunks (200–400 tokens) give precise retrieval but lose context. Large chunks (1000+) keep context but dilute the embedding and waste prompt space. Start around 500–800 tokens and tune against your eval set.
- **Store metadata with every chunk.** Source file, heading path, page number, timestamp. You'll need it to cite the answer and to filter retrieval later. Retrofitting metadata is miserable.

## Step 2: Embed the Chunks (and the Question)

An embedding model turns text into a vector — a point in a few-hundred-dimensional space where similar meanings sit close together. Embed each chunk once, store the vectors; embed the user's question at query time with the **same model**, then find the closest chunk vectors.

```python
# ❌ One model to embed, another to search — vectors aren't comparable across models
index_vectors = embed_model_a(chunks)
query_vector = embed_model_b(question)
```

**The embedding model for indexing and querying must be identical.** Vectors from different models live in different spaces; comparing them is meaningless. This one bites people who change models and forget to re-index everything.

```python
# ✅ Same model both sides, batched, normalized
def embed(texts, model="text-embedding-3-small"):
    resp = client.embeddings.create(input=texts, model=model)
    return [d.embedding for d in resp.data]

chunk_vectors = embed(chunks)      # do this once, offline
```

Also: pick your embedding model and **stick with it as a project decision**. It determines your vector dimensions, your cost, and — because it's trained differently — what "similar" even means. Re-embedding millions of chunks isn't free, so changing it later is a migration, not a config tweak.

## Step 3: Store and Search the Vectors

For a small corpus you can brute-force cosine similarity in memory. For anything real, use a vector store — pgvector if you already run Postgres, a dedicated one (Qdrant, Pinecone, Weaviate) if you need scale or filtered search.

```python
import numpy as np

def cosine_top_k(query_vec, chunk_vecs, chunks, k=5):
    q = np.array(query_vec)
    M = np.array(chunk_vecs)
    sims = (M @ q) / (np.linalg.norm(M, axis=1) * np.linalg.norm(q) + 1e-10)
    top = np.argsort(sims)[::-1][:k]
    return [chunks[i] for i in top]
```

The `k` you retrieve is another trade-off: too few and the answer isn't in the context; too many and you bury the signal in noise (and pay for the tokens). Five is a reasonable default to start.

**Similarity is not relevance.** The top-5 nearest chunks are the most *semantically similar* to the question, which is not always the same as the most *useful*. A question like "what's our refund policy?" may retrieve three chunks that all mention "refund" but only one with the actual policy. That gap is why reranking exists — we'll get there.

## Step 4: Stuff the Context and Prompt the Model

Now you build the prompt. The retrieved chunks become context; the question becomes the instruction. The whole game here is **grounding**: convincing the model to answer only from what you gave it.

```python
# ❌ Vague prompt invites hallucination when the context is thin
prompt = f"Context:\n{context}\n\nQuestion: {question}\nAnswer:"

# ✅ Explicit grounding + a way to abstain
prompt = f"""Answer the question using ONLY the context below.
If the context does not contain the answer, say "I don't have that information."
Cite the source file for each claim.

Context:
{context}

Question: {question}
Answer:"""
```

That "I don't have that information" escape hatch is not politeness — it's a **correctness feature**. Without it, models prefer to answer, and a plausible-sounding wrong answer is worse than a refusal in almost every business context.

Keep the retrieved chunks attributable:

```python
# Put source labels in the context so the model can cite, and you can audit
context = "\n\n".join(f"[{c['source']}]\n{c['text']}" for c in retrieved)
```

## Step 5: Wire It Together

```python
def rag_answer(question, k=5):
    q_vec = embed([question])[0]
    hits = vector_store.search(q_vec, k=k)          # list of {text, source, score}
    context = "\n\n".join(f"[{h['source']}]\n{h['text']}" for h in hits)
    return llm.complete(build_prompt(question, context)), hits
```

Ship that and it demos beautifully. Then real usage exposes the cracks.

## Where Naive RAG Breaks (and What to Do)

| Symptom | Root cause | Fix |
|---|---|---|
| Answer is missing info that's clearly in the docs | retrieved chunks were relevant-looking but wrong | **hybrid search** (keyword + vector), then **rerank** |
| Confident answer with no source support | weak grounding; no abstain path | stricter prompt, require citations, validate |
| Right doc, wrong section | retrieval matched topic, not the specific question | smaller chunks, or **parent-document retrieval** (search small, return big) |
| Works for exact phrases, fails for synonyms | keyword-only search | add vector search (or vice versa — combine them) |
| "What did we change last week?" returns stale info | index is a snapshot | scheduled re-indexing or CDC on the source |

The highest-leverage upgrades, in order:

1. **Hybrid search + reranking.** Retrieve more candidates than you need (say 25) with both keyword and vector search, then have a smaller cross-encoder model rerank them down to the best 5. This single step fixes more RAG failures than any prompt tweak.
2. **Metadata filtering.** "Only search docs from the last 90 days" or "only this customer's files" — filter at query time, not after. It shrinks the search space and prevents cross-contamination.
3. **Query rewriting.** User questions are messy. A cheap LLM pass to rewrite the question (or generate 2–3 sub-questions) before retrieval often dramatically improves recall.

## RAG Is a Recall Problem, Not a Generation Problem

The mental shift that makes debugging RAG tractable: **the LLM is almost never the failure point.** If the answer isn't in the retrieved context, no amount of prompt engineering will conjure it. So evaluate retrieval separately from generation.

```python
# ❌ You can't tell if the failure was retrieval or the model
assert "42" in final_answer

# ✅ Measure each stage, so you fix the right one
recall = hits_contain_ground_truth(retrieved, gold_doc_ids)
grounded = answer_supported_by_context(answer, retrieved)
```

If recall is low, fix chunking, embeddings, or search. If recall is high but the answer is wrong, fix the prompt or the model. Conflating these two is the number one time sink in RAG debugging.

## Takeaways

- **RAG = retrieve, then generate.** Keep knowledge in your database, not the model's weights — it stays fresh, cheap, and auditable.
- **Chunk on structure, with overlap, and always store metadata.** Size is a trade-off between precision and context; start at 500–800 tokens.
- **Same embedding model for indexing and queries.** Changing it means re-indexing the whole corpus — treat it as a migration, not a setting.
- **Ground the prompt and give the model an abstain path.** "I don't have that information" prevents the worst failure mode: a confident hallucination.
- **Hybrid search + reranking beats prompt tweaking** for retrieval quality. Retrieve wide, rerank narrow.
- **Debug retrieval and generation separately.** RAG failures are usually recall failures, and you can only see that if you measure the stages independently.
