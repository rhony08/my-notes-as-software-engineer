# Calling LLM APIs: Streaming, Timeouts, and Retries

Your first LLM call works in ten lines: send a prompt, get a response, print it. Then you ship it to real users and everything falls apart. The spinner sits for 40 seconds. One flaky upstream turns a chat into a dead page. And your retry logic quietly bills you three times for the same completion.

None of this is about prompting. It's about treating an LLM API like what it actually is: a slow, expensive, occasionally-unavailable network dependency that streams and fails mid-response. That's a distributed systems problem, and it comes with familiar solutions.

## The Three Properties That Make LLM APIs Weird

Most REST APIs are fast, cheap to retry, and atomic — you get the full response or an error, in a few hundred milliseconds. LLM APIs break all three assumptions:

| Property | Normal API | LLM API |
|---|---|---|
| Latency | 50–300 ms | 1–60 s |
| Cost per call | effectively free | metered per token, both directions |
| Response | atomic | streamed token by token |
| Failure mode | fails clean or clean | fails *halfway through* |

That last row is the one people underestimate. A response that streams for 30 seconds and then dies on a transient `503` is neither success nor failure. Your code has to decide what to do with the half-sentence you already showed the user.

## Streaming Is Not Inline, It's a UX Decision

Waiting 30 seconds for a full response feels broken. Streaming the first token in 500 ms feels fast — even though the total time is identical. This is the single highest-leverage change you can make, and it's mostly about *perceived* latency.

The API returns server-sent events (SSE): a sequence of small JSON chunks, each carrying a token fragment.

```python
# ❌ Blocking — user stares at a spinner for 30s
response = client.chat.completions.create(model="gpt-4o", messages=messages)
print(response.choices[0].message.content)

# ✅ Streaming — first token lands in ~500ms
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    stream=True,               # switch the transport to SSE
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:                  # first chunks can be role-only, no content
        print(delta, end="", flush=True)
```

Two gotchas that bite everyone once:

- **The first chunk often has no content.** It carries the role (`assistant`) and nothing else. Check before you print, or you'll wonder why your output starts with an empty line.
- **You have to handle the tail.** The stream ends with `finish_reason` and, crucially, a usage block you must capture for cost tracking. If you `break` out early (user hit "stop"), you still need to know how many tokens you burned.

Streaming costs you one thing: **you can't validate the whole response before showing it.** With structured outputs, that's a real problem — more on that below.

## Timeouts: One Number Is Never Enough

The default timeout in most HTTP clients is "forever," which is how you get requests that hang for minutes during an upstream incident. But a single timeout value fails too — LLM calls have two distinct phases with wildly different timing.

```python
# ❌ One timeout for everything
client = OpenAI(timeout=60)   # too short for big generations, too long to fail fast

# ✅ Split connect from read, and read from total
client = OpenAI(
    timeout=httpx.Timeout(
        connect=5.0,    # TCP+TLS handshake — should be fast, fail fast
        read=60.0,      # time between streamed chunks — this is your real guard
        write=10.0,
        pool=5.0,
    ),
)
```

The key insight: with streaming, `read` is **time-to-next-chunk**, not total duration. A 3-minute generation is fine as long as tokens keep flowing every few hundred milliseconds. A 90-second silence means the connection is dead — kill it. So set `read` to catch stalls (say 30–60s), not to cap total generation time.

Then add a **wall-clock cap** on top, because a stream that dribbles one token every 29 seconds never trips the read timeout but still hangs the user:

```python
import time
deadline = time.monotonic() + 120   # hard ceiling for the whole turn
for chunk in stream:
    if time.monotonic() > deadline:
        stream.close()
        raise TimeoutError("generation exceeded wall-clock budget")
```

## Retries: The Part That Costs (and Duplicates) Money

Retries are where naive implementations go wrong in two directions at once: they retry things that shouldn't be retried, and they blindly re-send things that already succeeded.

**Retry transient failures, never client errors.**

| Error | Retry? | Why |
|---|---|---|
| `429` rate limit | ✅ yes, back off | server said "slow down", it'll clear |
| `500` / `502` / `503` | ✅ yes | transient upstream |
| `408` / connection reset | ✅ yes | network blip |
| `400` bad request | ❌ no | your payload is wrong; retrying won't fix it |
| `401` / `403` | ❌ no | auth problem; retrying hammers a wall |
| `context_length_exceeded` | ❌ no | deterministic; fix the prompt |

Use **exponential backoff with jitter**. Jitter matters more than people think: without it, all your clients retry in lockstep, and you recreate the outage you were recovering from (thundering herd).

```python
import random, time

def with_backoff(fn, max_attempts=4, base=1.0, cap=20.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except (RateLimitError, APIStatusError) as e:
            retryable = isinstance(e, RateLimitError) or (500 <= e.status_code < 600)
            if not retryable or attempt == max_attempts - 1:
                raise
            # honor server's Retry-After if present, else exponential + jitter
            delay = min(cap, base * 2 ** attempt) * (0.5 + random.random())
            time.sleep(delay)
```

That `Retry-After` header (or the SDK's computed `retry_after`) is a gift — the server is telling you exactly how long to wait. Ignore it and you'll just get more `429`s.

### The duplication trap

Here's the subtle one. A stream that dies at token 800 of 1000 has already been *billed*. Retrying the whole request pays for those 800 tokens twice — and if you showed the user the partial text, they now see two overlapping answers. Guardrails:

- **Never retry after you've emitted tokens to the user.** For streaming, a mid-stream failure is a final failure — surface it, offer a "retry" button, let the human decide.
- **Deduplicate with idempotency.** If you have a request ID, send it and let a gateway drop duplicates (many providers accept an idempotency key).
- **Cap the blast radius.** Circuit-break the whole feature after N consecutive failures instead of letting every request retry three times against a dying upstream.

```python
# ❌ Retries a stream that already sent half a response — double billing + garbled UI
for attempt in range(3):
    try:
        stream_response(messages)
        break
    except Exception:
        continue

# ✅ Streaming is one-shot once bytes flow; retries only protect the pre-first-token window
sent_any = False
try:
    for chunk in stream:
        sent_any = True
        render(chunk)
except Exception:
    if not sent_any:
        retry_once()      # safe: user saw nothing yet
    else:
        show_error_with_retry_button()   # let the user re-ask, don't silently duplicate
```

## Putting It Together: A Request That Doesn't Embarrass You

The pattern is: stream for perceived speed, split your timeouts, retry only before the first token, and track usage on every path.

```python
def chat(messages, on_token, budget_s=120):
    def call():
        return client.chat.completions.create(
            model="gpt-4o", messages=messages, stream=True,
            stream_options={"include_usage": True},   # get token counts on the final chunk
        )

    stream = with_backoff(call)          # retries only apply here, pre-first-token
    deadline = time.monotonic() + budget_s
    usage = None
    for chunk in stream:
        if time.monotonic() > deadline:
            stream.close(); raise TimeoutError
        if chunk.usage:
            usage = chunk.usage          # final chunk carries counts
        if chunk.choices and chunk.choices[0].delta.content:
            on_token(chunk.choices[0].delta.content)
    record_cost(usage)                   # always — even on partial/large responses
    return usage
```

## Takeaways

- **Stream always, in user-facing paths.** It's the cheapest perceived-latency win you'll ever get, and it's one flag.
- **Split your timeouts.** Fail the connect fast; set `read` to catch *stalls*, not total duration; add a separate wall-clock cap.
- **Retry `429`/`5xx`/network — never `4xx`.** Use exponential backoff *with jitter*, and honor `Retry-After`.
- **Never retry after tokens have reached the user.** A partial stream is a final failure; let the human re-ask instead of double-billing and garbling the UI.
- **Track usage on every exit path** — success, timeout, early stop. Uninstrumented LLM calls are how bills surprise you.
- **Circuit-break the feature**, not just the request. When the upstream is down, fail fast and cheap instead of retrying into a wall.
