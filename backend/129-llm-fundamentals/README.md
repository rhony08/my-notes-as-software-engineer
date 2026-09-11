# How LLMs Work: A Developer's Mental Model

You've built software your whole career on one assumption: if you give a program the same input, you get the same output. Then you call an LLM and ask "what's the capital of France?" twice, and you get two slightly different sentences. Nothing's broken. You just moved from deterministic computation into something else — and if you keep treating it like a normal API, you'll ship bugs that look like randomness.

You don't need to understand transformers to build with LLMs. You don't need to know about attention heads or backpropagation any more than you need to know how a CPU fabricates transistors to write a `for` loop. What you *do* need is a working mental model — a small set of abstractions that explains the behavior you'll actually observe: why the same prompt gives different answers, why the output drifts off the rails, why it can't do math, and why "the model knows X" is a claim you should be suspicious of.

Here's the model I use. It's not precise enough to train a model on, but it's precise enough to debug one.

## The Whole Model Is a Next-Token Predictor

Strip away everything and an LLM does exactly one thing: **given a sequence of tokens, predict the next one.**

That's it. That's the whole trick.

When you send a prompt, the model doesn't "answer" it. It appends tokens one at a time, each time asking "what's most likely to come next?" — and it does this in a loop until it emits a stop token.

```
You: "The capital of France is"
Model: → " Paris"          (most likely next token)
Model: → "."                (given "...is Paris")
Model: → [STOP]
```

Everything an LLM appears to do — reasoning, coding, translation, holding a conversation — is this one operation applied repeatedly. The "intelligence" is emergent from a model that got really, really good at guessing the next token across a massive range of text.

This single fact explains most of the weirdness you'll hit in production:

- **It predicts plausibility, not truth.** A sentence can be perfectly fluent and factually wrong. The token "Paris" is likely after "capital of France is," but a confidently-wrong token is just as likely after a question the model has no good data for.
- **It has no memory of your conversation.** Each request is a fresh sequence of tokens. "Memory" is an illusion you create by re-sending the whole conversation every time.
- **It can't do steps inherently.** It computes one token at a time with no scratchpad unless you make it write one out.

## Tokens, Not Words

The model doesn't see characters or words. It sees **tokens** — chunks of text roughly 3–4 characters on average in English.

```
"Hello, world!" → ["Hello", ",", " world", "!"]
"unbelievable"  → ["un", "believ", "able"]
```

Why you should care:

- **You're billed per token**, input and output, and output usually costs more.
- **Context limits are token limits**, not word limits. "128k context" sounds enormous until you realize a big JSON payload can eat it fast.
- **Tokenization is why the model is bad at spelling and character-level tasks.** Ask it how many "r"s are in "strawberry" and it's operating on `["straw", "berry"]` — the letters aren't even visible to it.

## The Weights Are Frozen — Everything You Change Is in the Prompt

Here's the mental model that matters most for engineering: **the model's weights don't change when you use it.**

When you call `gpt-4o` or Claude or Llama, you're running inference against a frozen snapshot of parameters trained months ago. You cannot update it by chatting. You cannot teach it your API by giving it examples in one request — those examples exist only for that one request.

So there are exactly two levers you control:

1. **The context window** — what you put in the prompt right now (instructions, examples, retrieved documents, conversation history).
2. **The generation parameters** — temperature, top-p, max tokens, stop sequences.

Everything else — fine-tuning, RAG, tool use, agents — is machinery built *around* those two levers. Remember this and half the "how do I make the model remember X?" questions answer themselves.

## Temperature: The Dial That Controls Randomness

Temperature controls how the model picks among likely next tokens.

- **temperature = 0** → always take the single most likely token. As deterministic as an LLM gets.
- **temperature = 1** → sample proportional to the model's probabilities. Varied, creative, sometimes unhinged.

```
temperature=0.0  "The function returns the sum of the two numbers."
temperature=0.2  "The function returns the sum of the two numbers."
temperature=1.0  "The function adds the two numbers together and hands back the result."
```

**Temperature 0 is not the same as deterministic.** Even at 0, floating-point non-determinism on GPUs, batching, and provider-side changes can give you slightly different outputs. If your test asserts an exact string match on an LLM response, it *will* flake.

Rule of thumb:

| Task | Temperature |
|------|-------------|
| Extraction, classification, code gen | 0 – 0.2 |
| Summarization, Q&A | 0.2 – 0.5 |
| Brainstorming, creative writing | 0.7 – 1.0 |

## The Context Window Is the Model's Entire Working Memory

Everything the model "knows" in a given moment lives in one flat sequence of tokens: system prompt, conversation history, retrieved docs, your question. If it's not in that sequence, the model has no access to it.

This has sharp consequences:

- **Bigger context ≠ better recall.** Models attend unevenly across long contexts. Cramming 100k tokens of docs in often performs *worse* than retrieving the 5 most relevant chunks. More context dilutes attention.
- **Position matters.** Information at the very start and very end of the context tends to get used more reliably than stuff buried in the middle — the classic "lost in the middle" effect.
- **Cost and latency scale with it.** Every token in the window gets reprocessed (or at least billed) on every call.

```
❌  Dump all 400 support articles into the prompt "just in case"
✅  Retrieve the 3 relevant articles, put them near the end, right before the question
```

## Why It "Hallucinates" (and Why That Word Is Misleading)

The model is optimized to produce *plausible* continuations. When it hasn't seen a fact, the most plausible-looking continuation is often a confident fabrication — fake but fluent.

It's not lying. Lying requires knowing the truth and choosing otherwise. The model has no truth-checking step; it has a probability distribution over the next token, and "a plausible-sounding citation" is frequently the highest-probability completion of "According to a 2019 study by...".

What actually reduces hallucination:

- **Grounding**: put the real source material in the context and instruct the model to answer *only* from it.
- **Retrieval (RAG)**: fetch relevant documents and inject them, instead of trusting the model's memory.
- **Tools**: give it a calculator, a search API, a database — let it fetch facts instead of recalling them.
- **Structured output**: force it into a schema so it can't ramble into a fake paragraph.

None of these *eliminate* hallucination. They reduce the surface area. Plan for it, don't assume it's gone.

## What This Model Gets You

If you internalize the next-token predictor framing, you'll stop making a specific class of mistakes:

- You'll stop expecting determinism from a sampling process.
- You'll stop expecting the model to remember anything you didn't re-send.
- You'll stop trusting fluent output as correct output.
- You'll stop blaming "the model" for behavior that's actually your prompt and context.

And you'll start designing with the right questions: *What do I put in the context? What do I retrieve? What do I verify? What do I do when it's wrong?*

That's the jump from "calling an AI API" to engineering an AI feature.

---

## Takeaways

- An LLM is a next-token predictor, full stop. Everything it appears to do is that one operation in a loop. Understanding this explains most surprising behavior.
- It predicts plausibility, not truth. Fluent and correct are independent properties.
- Input is tokens, not words (~3–4 characters each). You're billed per token and limited by tokens, and character-level tasks are genuinely hard for it.
- Weights are frozen at inference time. You only control the context window and generation parameters — everything else (RAG, tools, fine-tuning) is built around those.
- `temperature=0` reduces variance but doesn't guarantee determinism. Never assert exact string equality on LLM output.
- The context window is the model's whole working memory. More context usually isn't better; relevant, well-positioned context is.
- Hallucination isn't a bug you patch away; it's a property you design around with grounding, retrieval, tools, and validation.
- Design the loop as if the model will sometimes be wrong. Because it will.
