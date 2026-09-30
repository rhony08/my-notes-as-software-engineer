# Fine-tuning vs RAG: Choosing the Right Tool

Your support bot gives a confidently wrong answer about a product your company discontinued last month. You ship a "fix" that weekend — you fine-tune a model on your docs. Three weeks later the pricing page changes again, and your freshly trained model is stale again. Money burned, problem back.

This is the trap most teams fall into: treating fine-tuning and RAG like two ways to do the same thing. They're not. One changes *how* the model behaves. The other changes *what the model knows right now*. Pick wrong and you pay for it twice — once to build it, once to maintain it.

## The mental model that actually helps

Here's the distinction that finally made it click for me:

- **RAG** is an open-book exam. The model sits down with the right pages of your docs and answers from them.
- **Fine-tuning** is studying for the exam. The knowledge moves *into* the model's weights so it doesn't need the book anymore.

Open-book exams are great when the material changes every week. Studying is great when you want the student to answer in a certain *style* or *format* — regardless of what today's material says.

That's the whole game. Everything below is just consequences of that one difference.

## What RAG is actually good at

RAG = retrieve relevant chunks at query time, stuff them into the prompt, let the model answer grounded in them.

```
User question → embed → vector search → top-k chunks → prompt → LLM → answer
```

The killer feature: **change the source docs, and the next query reflects it. No retraining.** Delete a chunk, and the model stops knowing it. That's a property you cannot get from a fine-tuned model without retraining.

```python
# The whole "update" story for RAG in one line:
vector_db.upsert(documents=updated_chunks)  # next query is already accurate
# vs fine-tuning: new dataset → new training run → new eval → new deploy
```

RAG also gives you **citations**. When a chunk says "refunds take 5–7 business days," you can point at it. When a fine-tuned model says it, you have no idea where that came from. For anything compliance-adjacent, this alone can settle the debate.

### When RAG is the wrong tool

RAG is fragile in a few specific ways. If your retriever pulls bad chunks, the model answers badly — garbage in, garbage out, just with more confidence. Chunking is a whole discipline ([we covered that separately](../137-chunking-strategies/)), and it's where most RAG quality is won or lost, not in the model choice.

And RAG can't teach *behavior*. If you need every response in a rigid JSON shape, or in your brand's exact voice, bolting on examples at every request is expensive and inconsistent.

## What fine-tuning is actually good at

Fine-tuning takes a base model and nudges its weights using examples of input → desired output.

The honest truth: **fine-tuning is mostly about form, not facts.** It's the right tool when you want the model to:

- Consistently follow a format (structured outputs, a specific reasoning style)
- Adopt a tone or persona that's hard to prompt reliably
- Shrink a giant prompt — if you're sending 2,000 tokens of instructions every call, fine-tuning can bake those in
- Learn from examples you have but can't describe as rules

```
# What fine-tuning is NOT good at:
# "Teach the model our current pricing."        ❌ changes weekly
# "Teach the model the refund policy."           ❌ needs citations
# "Make the model answer in our support voice."  ✅ stable, behavioral
```

### The cost nobody mentions upfront

Fine-tuning has a real bill attached:

- You need a labeled dataset (hundreds to thousands of good examples — this is usually the actual work)
- Every base-model update from the vendor means re-training to stay current
- You own **evals**: how do you know the new model didn't get worse on something else?
- You can't cite sources, because the knowledge is smeared across the weights
- Debugging is hard — why did it say that? Nobody knows, including the model

If it's drifting, there's no "go fix the doc." You retrain and hope.

## The comparison

| | RAG | Fine-tuning |
|---|---|---|
| Updates | Edit docs, instant | Retrain + redeploy |
| Knowledge type | Facts that change | Behavior, format, style |
| Citations | Yes | No |
| Data needed | Documents | Labeled examples (usually more work) |
| Cost profile | Per-query (retrieval + tokens) | Upfront + per base-model upgrade |
| Debuggability | Inspect retrieved chunks | Opaque |
| Best for | Q&A over a changing corpus | Consistent behavior at scale |

## "Which one?" — a decision that has a boring answer

Before you pick, try the cheap options first, in this order:

1. **Better prompting.** Genuinely, most "we need fine-tuning" problems die here.
2. **RAG, if the problem is knowledge.** If answers are wrong because the model doesn't *know* the current truth, that's retrieval's job.
3. **Fine-tuning, if the problem is behavior.** If answers are wrong because the model won't follow your format or voice *even when it has the right facts*, that's a tuning problem.

And the answer is very often **both**. A common production shape: fine-tune a small model to reliably emit citations and follow output format, then RAG to feed it the current facts. The tuning handles *how*, retrieval handles *what*.

```python
# A realistic "both" pipeline
chunks = retrieve(query)                    # RAG: current, citable facts
response = model.generate(                  # fine-tuned model: format + tone
    system="Answer using only the context. Cite chunk IDs.",
    context=chunks,
    question=query,
)
```

## When to reach for each — practical triggers

Reach for **RAG** when:
- Your content changes more often than you'd redeploy a model
- You need sources you can point at
- You don't have a labeled dataset and don't want to build one
- Access control matters (different users see different documents — filter at retrieval time)

Reach for **fine-tuning** when:
- Your prompts are bloated with instructions you repeat every call
- You need strict output schemas and prompting isn't reliable enough
- Latency/cost matters and a smaller tuned model can replace a bigger prompted one
- You already have high-quality examples from production logs

❌ "We'll fine-tune it to learn our docs."
✅ "We'll RAG over the docs because they change; we'll fine-tune only if the model won't follow our format."

## The thing to actually take away

The two are not competitors. Confusing *knowledge* with *behavior* is what makes teams fine-tune their documentation and then wonder why it's stale in a month.

If the fact might change next Tuesday, it belongs in a retrieval index. If it's about *how the model should always behave*, it belongs in the weights — if prompting genuinely can't get you there.

Start with prompting. Add RAG when the gap is knowledge. Fine-tune only when the gap is behavior and you've ruled out everything cheaper. Most teams that follow that order never need to touch fine-tuning at all — and the ones who do, do it for the right reasons.
