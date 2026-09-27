# Observability for LLM Applications

Your LLM feature is live. Latency looks fine, error rate is near zero, CPU is flat. By every dashboard you already have, the service is healthy.

Then finance asks why the OpenAI bill tripled last week. A support lead forwards a screenshot of a confidently wrong answer. And nobody can explain why the feature "felt worse" yesterday — it turns out someone tweaked the prompt and quietly dropped the system instructions.

Traditional observability can't see any of this. Your HTTP 200 doesn't tell you the model rambled, the prompt grew by 4k tokens, or the answer was fluent garbage. LLMs fail softly, and soft failures are invisible to infra metrics.

That's the gap. LLM observability means watching the things that actually break AI features: cost, latency distribution, prompt/response content, and quality.

## Why Your Existing Dashboards Aren't Enough

| Signal | What your APM shows | What's actually happening |
|--------|---------------------|---------------------------|
| Latency | "p99 = 2.1s" | Streamed token-by-token — time-to-first-token is what users feel |
| Errors | "0% 5xx" | Model returned a valid HTTP 200 with a wrong or refused answer |
| Cost | Nothing | Every retry, bigger context window, and picked model costs money |
| Quality | Nothing | A prompt change silently degraded outputs |

The failure mode is different. A 500 is easy — the user sees an error. A hallucination is silent. The request "succeeded," the user got an answer, and now they trust it less. You can't alert on what you don't capture.

## Instrument the Call, Not Just the Request

The unit of observability changes from "HTTP request" to "LLM call" (and often a tree of them). Capture four things per call:

- **Inputs/outputs** — the prompt, the completion, the model, and temperature
- **Tokens & cost** — prompt tokens, completion tokens, computed cost
- **Timing** — time-to-first-token, total duration, tokens/sec
- **Outcome** — finish reason, retries, error type, and later, quality signals

The good news: OpenTelemetry now has [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/), so this can live in the same tracing backend you already run.

```python
from opentelemetry import trace

tracer = trace.get_tracer("llm.gateway")

def call_llm(model: str, messages: list, **params):
    with tracer.start_as_current_span("llm.chat") as span:
        # span attributes follow the GenAI semconv, so existing tools can read them
        span.set_attribute("gen_ai.system", "openai")
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.usage.input_tokens", usage.prompt_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", usage.completion_tokens)
        # custom attrs for what semconv doesn't cover yet
        span.set_attribute("llm.cost_usd", computed_cost)
        span.set_attribute("llm.ttft_ms", ttft_ms)
        span.set_attribute("llm.finish_reason", choice.finish_reason)
        return response
```

Wrap every call — including the ones inside your retry loop and your agent's tool steps. Half-instrumented LLM apps are worse than none, because you'll draw conclusions from a biased sample.

## The Metrics That Actually Matter

Infra metrics tell you the box is up. These tell you the feature is working:

| Metric | Why it matters | Watch for |
|--------|----------------|-----------|
| **Time-to-first-token (TTFT)** | What users feel before text appears | p95 spikes on long prompts |
| **Total latency** | Full response time | Grows with output length; segment by token count |
| **Cost per request / per user** | Direct money | Creep from bigger prompts & retries |
| **Prompt tokens in** | Context bloat | Slow climb = prompt or history leaking |
| **Tokens/sec** | Provider throughput | Provider degradation before it's visible elsewhere |
| **Retry rate** | Hidden cost & latency | A retry loop quietly doubles your spend |
| **Cache hit rate** | Savings | Drops mean a key regression |
| **Refusal / finish-reason mix** | Silent quality loss | Rising `content_filter` or `length` finishes |

Two of these deserve a habit:

**Segment latency by output length.** A single p99 number is meaningless when one request returns 50 tokens and another returns 2,000. Bucket by completion tokens, then compare.

**Track cost per feature, not per service.** When the AI search box and the AI summarizer share a gateway, a single bill number tells you nothing. Tag calls with a feature or route name so you can attribute spend.

## Logging Prompts and Responses (Carefully)

You can't debug an LLM feature without seeing the actual prompt and output. But raw logging is a privacy and cost landmine.

```python
# ❌ Logs whatever the user pasted — could be passwords, PII, or card numbers
logger.info("llm_call", prompt=messages, completion=text)

# ✅ Redact, then log with structure and enough metadata to reconstruct context
logger.info("llm_call", extra={
    "feature": "support_summarizer",
    "model": model,
    "prompt_redacted": redact_pii(messages),
    "completion_redacted": redact_pii(text),
    "input_tokens": usage.prompt_tokens,
    "output_tokens": usage.completion_tokens,
    "cost_usd": cost,
    "ttft_ms": ttft_ms,
    "duration_ms": duration_ms,
    "trace_id": trace.get_current_span().get_span_context().trace_id,
})
```

Rules that have saved teams a lot of pain:

- **Redact before you write.** Emails, phone numbers, IDs, tokens, card numbers. Run a PII pass on the way in, not after an incident.
- **Sample high-volume paths.** Log 100% of errors and low-confidence calls; sample the boring happy path (say 5–10%). You rarely need every successful FAQ lookup.
- **Set a retention policy.** Prompt logs are user data. Keep them days-to-weeks, not forever, and say so in your privacy policy.
- **Store the trace ID, not the whole conversation, in your app logs.** The full record belongs in your LLM observability store where it can be joined with cost and quality.

## Quality Signals: The Part Infra Can't Give You

A model can be "up" and still be wrong. Quality is a first-class observability signal, and you collect it from three angles:

**1. Implicit signals** — cheap, always on:
- User thumbs up/down, or copy/regenerate/abandon actions
- Refusal rate and finish-reason distribution
- Response format violations (did it break your expected schema?)
- Retrieval hit quality for RAG (did the retrieved chunks actually contain the answer?)

**2. Explicit evals** — offline and online judging against a golden set (covered in the evals note). Run a slice of production traffic through an LLM judge on high-stakes paths.

**3. Drift detection** — the sneaky one. Track prompt version and model version as tags on every call. When quality drops, the first question is "what changed?" If you can't answer it from your traces, you're guessing.

```python
# Tag every call with its version — turns "it feels worse" into a diff
span.set_attribute("app.prompt_version", PROMPT_VERSION)   # e.g. "support-v7"
span.set_attribute("gen_ai.response.model", response.model)  # catches provider swaps
```

Provider-side model updates are real. `gpt-4o` today isn't always bit-identical to `gpt-4o` next month. Logging the exact response model (not just what you requested) is how you catch a silent change.

## Wiring It Together

You don't need to build this from scratch. A few common setups:

| Tool | Sweet spot |
|------|-----------|
| **OpenTelemetry + your APM** | You already have Grafana/Datadog/Honeycomb; you just want LLM spans in it |
| **Langfuse** | Open-source, self-hostable, traces + evals + cost in one |
| **LangSmith** | Tight LangChain/LangGraph integration, strong eval tooling |
| **Helicone** | Proxy-based — drop-in for cost, caching, and rate-limit visibility |
| **Arize Phoenix** | Strong on evals, drift, and RAG debugging |

The pattern worth copying: put a thin **LLM gateway** in front of the provider. Every call goes through it, and it emits the span, computes cost, applies caching, and tags the feature. One place to instrument, one place to enforce. It also means switching providers doesn't mean re-instrumenting everything.

## Trade-offs You Can't Escape

- **Observability vs privacy.** Full prompt logging is the best debugging tool and the biggest liability. Redact by default, sample aggressively, and keep retention short.
- **Cost of observing.** Storing every prompt/response isn't free — often 1–5% of your inference spend. Sample the happy path.
- **Content capture vs metadata.** Metadata is cheap and safe; content is where the answers are. Capture content for errors and sampled success, not everything.
- **Online judges vs latency.** Running an LLM judge on every response adds latency and cost. Reserve it for high-stakes paths; use implicit signals everywhere else.
- **Granularity vs noise.** Trace every tool step and you'll drown. Trace every LLM call at least; go deeper only where agents actually branch.

## When You'd Actually Build This

- Any LLM feature where you're paying per token (i.e., all of them) — you need cost attribution
- Anything user-facing where a wrong answer has real consequences
- Agentic flows, where one user request fans out into a dozen model calls
- The moment you have more than one prompt version in production

A weekend prototype? A `print` and a cost counter is fine. The day real users touch it, instrument the call.

## Takeaways

- Infra metrics can't see LLM failures — the request returns 200 while the answer is wrong, slow, or expensive.
- Instrument the LLM call, not just the HTTP request: prompt, completion, tokens, cost, TTFT, and finish reason.
- Use OpenTelemetry's GenAI semantic conventions so LLM traces live alongside your existing ones.
- Segment latency by output length and track cost per feature — a single p99 and a single bill hide everything.
- Log prompts and responses, but redact PII on the way in, sample the happy path, and set a short retention policy.
- Treat quality as a signal: thumbs, refusals, schema violations, retrieval hits, and sampled LLM judges.
- Tag every call with prompt and model version so "it feels worse" becomes an actual diff.
- Put a thin LLM gateway in front of the provider — one place to emit traces, compute cost, and cache.
- Observability has real cost and privacy trade-offs. Redact, sample, and retain deliberately — not by accident.
