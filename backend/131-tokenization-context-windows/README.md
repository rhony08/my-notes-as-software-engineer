# Tokens, Context Windows, and What They Cost You

You ship a chat feature. It works great in testing. Then finance sees the bill: your average request costs 12x what you budgeted, and one customer pasting a long PDF blew past the context limit and returned a hard error instead of a helpful answer.

None of this is a model bug. It's a units problem. You were thinking in characters and words; the model bills and thinks in **tokens**. Once you get that mental model right, latency, cost, and half your mysterious truncation bugs become predictable instead of spooky.

## A Token Is Not a Word

Start here, because everything else depends on it. Models don't read text — they read tokens, which are chunks of text produced by a tokenizer. A token is roughly 3-4 characters of English, or about 0.75 words. But "roughly" is where people get burned.

```
"hello"        → 1 token
" Hello"       → 1 token  (note the leading space — different token!)
"tokenization" → 2 tokens ("token" + "ization")
"😀"           → 2-4 tokens depending on the tokenizer
"SELECT * FROM" → 4 tokens
"日本語"        → often 2-3 tokens per character
```

The two facts that matter for engineering:

1. **Whitespace and casing are part of the token.** `"Hello"` and `" Hello"` are different tokens. This is why weird formatting inflates your count.
2. **Unfamiliar text costs more per character.** English prose is efficient because tokenizers were trained heavily on it. Code, JSON, base64 blobs, and non-Latin scripts are all *less* efficient — you pay more tokens for the same amount of "content."

That last one is the silent budget killer. People estimate a payload by eyeballing word count and end up 40% off.

## Context Window: The Working Memory Cap

The context window is the total number of tokens the model can consider at once. And it's the **whole** conversation — system prompt, chat history, retrieved documents, your new question, *and* the response it's generating. It all shares one budget.

```
┌──────────────────────── context window ────────────────────────┐
│ system prompt │ history │ retrieved docs │ user msg │ OUTPUT  │
│     800       │  4,200  │     6,000      │   250    │ reserve │
└───────────────────────────────────────────────────────────────┘
        everything here competes for the same fixed budget
```

Two consequences that cause real bugs:

**1. You must reserve room for the output.** If your model's window is 128k and you stuff 128k of input in, you've left nothing for the answer. Many APIs will just error; others silently truncate. Always budget output tokens explicitly:

```python
# ❌ Feels fine, breaks at the edges
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
)

# ✅ Leave explicit headroom for the response
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    max_tokens=1000,          # reserve for the answer
)
# And before sending, check: count(messages) + max_tokens <= window
```

**2. "Long context" is not "infinite context."** A bigger window doesn't mean the model pays equal attention to everything in it. Information buried in the middle of a huge prompt is regularly missed — the classic "lost in the middle" effect. A 200k window is a ceiling, not a target.

## Counting Tokens Before You Pay

You can't manage what you don't measure. Don't estimate with character math in production — use a real tokenizer.

```python
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

def count_tokens(text: str) -> int:
    return len(enc.encode(text))

text = "The quick brown fox jumps over the lazy dog."
print(count_tokens(text))   # 9 tokens for 44 chars -> ~4.9 chars/token
```

That ratio (chars per token) is worth computing on *your* real traffic. English prose tends to land around 4; dense JSON or code often drops to 2-3. If you budget with the wrong ratio, your cost model is wrong on day one.

For a quick sanity check across a payload, count the parts separately so you can see where the money is going:

```python
def estimate_request(system, history, docs, question):
    return {
        "system":   count_tokens(system),
        "history":  sum(count_tokens(m["content"]) for m in history),
        "docs":     sum(count_tokens(d) for d in docs),
        "question": count_tokens(question),
    }

# Output tells you instantly whether retrieval or history is the hog
# {'system': 820, 'history': 4300, 'docs': 6100, 'question': 41}
```

Nine times out of ten, that breakdown reveals the docs or the history — not the user's question — is what's eating the window.

## Cost Scales Linearly (and Directionally)

Billing is per token, and **input and output are priced differently**. Output is almost always the more expensive side — often 3-4x input — because generation is more compute-heavy than reading.

| What you send | What it costs |
|---|---|
| Growing chat history every turn | Compounds — turn 20 resends turns 1-19 |
| A 6,000-token doc per request | Pays *every single request* |
| `max_tokens` set absurdly high | Doesn't cost money unless used, but wastes latency headroom |
| Verbose system prompt | Paid on every call, forever |

The compounding one catches everyone. In a naive chat loop, every turn re-sends the entire history, so request N costs roughly the sum of all prior turns:

```python
# ❌ Cost grows quadratically over a conversation
messages.append({"role": "user", "content": new_msg})
messages.append({"role": "assistant", "content": reply})
# next turn resends ALL of it

# ✅ Trim or summarize what you resend
messages = [system] + summarize(messages[1:-4]) + messages[-4:]
#          ^ fixed    ^ collapse old turns        ^ keep recent verbatim
```

A 30-turn conversation can cost **10-50x** a single-turn one. That's not a rounding error — it's the difference between a viable product and one that bleeds money.

### Prompt Caching Changes the Math

If your provider supports prompt caching (Anthropic, OpenAI, Gemini all do variants), a stable prefix — system prompt, instructions, reference docs — is billed at a big discount on repeat calls. This flips the usual advice: a *long, stable* prefix can be cheaper than a short one that changes every call. Keep the volatile stuff (the user's question) at the end so the cacheable prefix stays identical.

## Strategies That Actually Reduce Tokens

| Strategy | Saves | Watch out for |
|---|---|---|
| Summarize old chat turns | Big — kills quadratic growth | Loses detail; summarize, don't delete blindly |
| Retrieve fewer/smaller chunks (RAG) | Big — stops paying for irrelevant docs | Too few chunks → worse answers |
| Trim whitespace, JSON keys, boilerplate | 10-30% on structured payloads | Must keep it parseable |
| Cap `max_tokens` | Latency headroom | Truncated answers if set too low |
| Prompt caching | Large on repeated prefixes | Only helps if prefix is stable |
| Choose a cheaper model for easy tasks | Large | Quality cliff on hard inputs |

The honest trade-off: **every token optimization trades context or quality for money.** The trick isn't minimizing tokens — it's figuring out which tokens are *earning their keep*. A doc chunk that improves the answer is cheap; three duplicate doc chunks are expensive.

## Practical Rules of Thumb

- **Budget output tokens first**, then fill the rest of the window with input. Never fill the window and hope.
- **Measure with the real tokenizer** on real traffic, not character counts. Recheck when your input mix changes.
- **Instrument token usage per request** (most APIs return `usage.prompt_tokens` / `completion_tokens`). Track p50 and p95 — outliers are where the budget dies.
- **Watch chat history growth.** Every turn re-sends prior turns; summarize aggressively.
- **Prefer stable prefixes** if you use prompt caching.
- **Treat the context window as a budget, not a bucket.** More isn't better; relevant is better.

Get the units right and AI features stop feeling like a slot machine. You'll know what a request costs before you send it, why latency spikes, and exactly which line item to attack when the invoice arrives.
