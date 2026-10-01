# LLMOps: Shipping and Maintaining AI Features

You built the demo in an afternoon. The notebook works, the answers look great, and everyone's excited. Then someone asks the only question that matters: "So when does it go to prod?"

And that's where it quietly falls apart. The prompt lives in a Python string. Nobody knows which model version generated last week's results. When quality drops, there's no alert — just a support ticket that says "the bot got dumber." You can't reproduce a bad answer, and you can't roll back a change you can't pin down.

Shipping an AI feature is easy. Keeping it good for the next six months is the actual job. That job has a name — LLMOps — and it's mostly the boring stuff you already do for normal software, applied to a component that's non-deterministic and changes under your feet.

## The demo-to-prod gap

A normal feature is deterministic. Same input, same output, forever. An LLM feature is the opposite: same input, and the output can drift when the *vendor* ships a new model, when someone edits a prompt, or when the data it retrieves changes.

So the whole discipline comes down to one thing: **making a non-deterministic system observable, versioned, and reversible.**

If you can answer these three questions at 3am, you're doing LLMOps:

- What exact prompt, model, and config produced this bad output?
- Is this a regression, or was it always this bad?
- How do I roll it back in the next five minutes?

Most teams can't answer any of them at launch. That's the gap.

## Version everything, including the prompt

The single biggest LLMOps mistake is treating prompts as throwaway strings.

```python
# ❌ The prompt is text in the code. Changing it is a mystery diff.
response = call_llm("You are a helpful support agent. Answer: " + q)

# ✅ The prompt is a versioned artifact with an ID you can log and roll back.
PROMPT = load_prompt("support_agent", version="v7")  # stored in a registry/git
response = call_llm(PROMPT.render(question=q), meta={"prompt": "support_agent@v7"})
```

Once prompts are artifacts, they get everything code already has: review, diffs, rollback, and blame. The prompt is not "just text" — it's your product logic, and it deserves the same treatment as a function.

Same rule for the other moving parts. Log **model name + version**, **temperature and key params**, **retrieval config** (top-k, chunk size), and **tool versions**. When something goes wrong, you want a record of exactly what ran, not a guess.

A useful rule of thumb: if it can affect the output, it should be versioned and logged with the output.

## Evals are your tests

You can't write a normal unit test for "is this answer good?" But you can do the next best thing: keep a dataset of inputs with known-good expectations, and run it on every change.

```python
# A minimal eval as a CI gate. Not perfect judgment — just a tripwire.
def run_evals(dataset, candidate_config):
    results = [score(case, run_model(case.input, candidate_config)) for case in dataset]
    pass_rate = mean(r.passed for r in results)
    if pass_rate < REQUIRED_PASS_RATE:   # fail the build, don't ship
        raise SystemExit(f"Evals regressed: {pass_rate:.0%} < {REQUIRED_PASS_RATE:.0%}")
    return results
```

The point isn't a perfect score. It's catching the case where your "harmless" prompt tweak fixed one thing and broke three others. Evals are a **tripwire**, not a verdict.

Build your dataset from production. Real traffic is the only source of inputs that actually matter. When you find a bad output in the wild, promote it into the eval set so it can never regress silently. That set grows into your most valuable asset.

Two layers that work well together:

- **Assertion evals** — cheap, deterministic checks. Did it return valid JSON? Did it refuse when it should? Does the citation exist in the source?
- **Judged evals** — an LLM or a human rates relevance/correctness on a sample. Slower and fuzzier, but catches things assertions can't.

❌ "It looked good in testing, ship it."
✅ "It passes 94% of our eval set, up from 91%, with no case regressing below the floor."

## Observability: trace the whole request

LLM apps fail in ways logs don't capture. A slow response might be your retriever, a retry storm, or a model that got slower last Tuesday. You need *traces*, not just log lines.

Capture per request, span by span: the prompt version, retrieval results, model + params, token counts, latency (broken into queue/retrieval/generation), cost, and the final output. Then you can answer "why was this slow" or "why was this expensive" without guessing.

```
trace_id=abc123
  ├─ retrieve        (43ms, 6 chunks)
  ├─ llm.generate    (2,140ms, model=gpt-x@2026-08, in=1,820 out=310, $0.0081)
  └─ guardrail.check (8ms, passed)
```

That single trace tells you: the generation dominated latency, it cost about 0.8 cents, and retrieval was fine. Without spans, you'd be reading tea leaves.

The two numbers to watch hardest are **cost** and **latency per request**, broken down by prompt version. Rising cost usually means a prompt grew or retrieval started returning more chunks. Rising latency usually means context is bloating. Both are early warnings you can act on before users notice.

## Guardrails belong in the deploy path, not the prompt

Prompt instructions like "never reveal internal data" are a *hope*, not a control. A determined user can talk the model out of nearly any instruction. Real guardrails are code that sits in front of and behind the model.

```python
# ❌ A guardrail that lives only in the system prompt
system = "Never answer questions about competitors. Be safe."  # bypassable

# ✅ A guardrail that lives in the request path
if not input_guard.allows(user_input):        # block/flag before spending tokens
    return refused_response()
output = call_llm(...)
if not output_guard.allows(output):           # catch PII leaks, jailbreaks, bad format
    return safe_fallback()
```

Put input checks early to save cost (why pay to generate an answer you'll throw away?), and output checks late as the last line of defense. Guardrails are an engineering control with a test suite — see [the guardrails note](../144-guardrails-moderation/) for the specifics. Treat them like auth: assume someone is actively trying to get around them.

## Roll out prompts like you roll out code

A prompt change is a production change. It should ship with the same caution.

- **Canary first.** Send a small slice of traffic to the new prompt. Watch cost, latency, and your eval metrics on live traffic.
- **Keep a kill switch.** You should be able to revert to the previous prompt version in a config change, not a redeploy.
- **A/B when it's about quality.** If "better" is subjective, split traffic and compare judged scores, not vibes.
- **Shadow before you switch.** Run the new prompt on real traffic *without* serving its output, and diff the results. Free signal, zero user risk.

The mindset shift: prompts are config, and config gets gradual rollouts and instant rollback.

## Fallbacks so a bad day doesn't become an outage

LLM providers go down and rate-limit you. If your feature has a single hard dependency on one model, its uptime is now your uptime.

```python
# Degrade instead of fail. A worse answer beats an error page.
try:
    return primary_model(q)
except (RateLimited, Timeout):
    return fallback_model(q)          # smaller/cheaper, still useful
except ProviderDown:
    return cached_or_default(q)       # last-known-good, or a clear refusal
```

Layer it: primary model → cheaper fallback model → cache → graceful degradation → clear error. Each layer is a different trade-off between quality, cost, and availability. See [graceful degradation](../30-graceful-degradation/) for the pattern in full. The point is that "the answer got a bit worse" should never become "the feature is down."

## The LLMOps checklist

| Concern | Demos do | Production does |
|---|---|---|
| Prompt | Hardcoded string | Versioned artifact, reviewed, rollback-able |
| Model | Whatever's default | Pinned version, logged per request |
| Quality | "Looks good" | Eval set in CI, pass-rate gate |
| Debugging | Re-read the code | Full traces: prompt, retrieval, tokens, cost |
| Safety | Prompt instructions | Input/output guardrails as code |
| Rollout | Edit and hope | Canary, shadow, A/B, kill switch |
| Failure | Error page | Fallback chain + graceful degradation |
| Improvement | Manual tweaks | Bad outputs promoted into evals |

## The cost profile nobody scopes for

LLM features have two budgets, and both are recurring:

- **Token cost** — scales with usage. A cache layer and tighter context can cut this dramatically. ([caching and cost control](../146-llm-caching-cost/))
- **Maintenance cost** — this is the one teams forget. Every base-model upgrade reopens the eval question. Every new feature adds prompt surface area. Someone has to own "is it still good?"

Budget for the second one. The demo is a sprint; the maintenance is the rest of the product's life.

## What to take away

LLMOps isn't a tool you install — it's treating a fuzzy, non-deterministic component with the same rigor you'd give a database.

- **Version everything** that can change the output: prompt, model, params, retrieval config.
- **Eval on every change.** A small tripwire beats a big argument.
- **Trace every request.** Cost and latency per prompt version are your early-warning system.
- **Guardrails in code**, not just in the prompt.
- **Roll out and roll back** prompts like deploys — canary, shadow, kill switch.
- **Fallbacks everywhere.** Degrade, don't die.
- **Feed bad outputs back** into your eval set. That's how it gets better instead of just staying the same.

The teams that win with AI features aren't the ones with the cleverest prompt. They're the ones who can change that prompt on a Tuesday, prove it didn't break anything, notice when it does, and roll it back in five minutes. That's the whole discipline — and it's the same discipline you already know.
