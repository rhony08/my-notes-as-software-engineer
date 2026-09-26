# Guardrails and Output Moderation

You shipped an LLM feature. It worked great in the demo. Then a user asked it to "roleplay as a hacker explaining how to bypass the paywall," and it happily obliged. Or it leaked the system prompt. Or it generated something that made legal call you.

Input filtering (the prompt injection stuff) stops some of that. But it's not enough. The model can still produce outputs you never wanted, from toxic text to fabricated citations to code that quietly exfiltrates data.

That's the gap guardrails fill: a layer around the model that constrains what goes in, what comes out, and what the app is allowed to do with it.

## The Three Places Guardrails Live

Most people think of moderation as one thing. It's really three, and they fail independently:

| Layer | What it does | Typical tooling |
|-------|--------------|-----------------|
| **Input guardrails** | Filter, classify, or rewrite the prompt before it hits the model | Regex, classifiers, prompt-injection detectors, PII scrubbers |
| **Output guardrails** | Validate the model's response before users see it | Moderation APIs, schema validation, PII detection, claim checks |
| **Action guardrails** | Constrain what tools/agents are allowed to do | Allowlists, scoped credentials, dry-run mode, human approval |

Skipping the output layer is the most common mistake. We validate inputs carefully and then trust whatever the model says. Don't.

## Output Moderation: Cheap Check, Big Payoff

Most providers ship a moderation endpoint. It's fast, separate from your main model, and genuinely useful as a first pass.

```python
from openai import OpenAI

client = OpenAI()

def moderate(text: str) -> tuple[bool, dict]:
    """Returns (is_flagged, categories). Fails open or closed? See below."""
    resp = client.moderations.create(
        model="omni-moderation-latest",
        input=text,
    )
    result = resp.results[0]
    flagged = {
        name: score
        for name, score in result.category_scores
        if getattr(result.categories, name)  # only the ones that tripped
    }
    return result.flagged, flagged
```

Two things to get right here:

**1. It's a filter, not a verdict.** A moderate call returns scores per category (harassment, self-harm, violence, etc.). Your policy decision is separate. A support bot might tolerate "frustrated" language that a kids' app should block outright. Set your own thresholds.

**2. Decide fail-open vs fail-closed.** If the moderation API times out, do you show the response or block it?

```python
# ❌ Fail-open on everything — a moderation outage becomes a free-for-all
flagged, _ = moderate(text)

# ✅ Fail-closed for high-risk surfaces, fail-open for low-risk
try:
    flagged, cats = moderate(text)
except TimeoutError:
    if surface in ("public_posting", "health_advice"):
        return BLOCKED_FALLBACK  # be safe when we can't check
    flagged = False            # chat-with-your-notes can proceed
```

Fail-closed sounds safer, but it means a third-party outage takes down your feature. Pick per surface, and log every fail-open decision so you can audit it later.

## What Moderation APIs Miss

Provider moderation is trained on generic safety categories. It won't catch:

- **Fabricated facts** — a confident, wrong citation. Not toxic, just false.
- **PII leakage** — the model echoing another user's data it saw in context.
- **Off-brand tone** — a bank bot making jokes about overdrafts.
- **Business-logic violations** — promising refunds your policy doesn't allow.

For those you need your own checks. Some are cheap:

```python
# ❌ Regex for anything mentioning "refund" nukes legitimate answers
if "refund" in text.lower():
    block()

# ✅ Only block commitments, not mentions
COMMITMENT_PATTERNS = [
    r"\byour refund (has been|will be) processed\b",
    r"\bi('ve| have) (approved|issued) your refund\b",
]
if any(re.search(p, text, re.I) for p in COMMITMENT_PATTERNS):
    escalate_to_human(text)
```

The regex approach is brittle and you'll maintain it forever. That trade-off is real. It's still better than letting the model make financial promises on your behalf.

## Structured Output as a Guardrail

If the model must return a fixed shape, validate it and reject anything that doesn't conform. This is the most reliable guardrail you'll ever write because it's deterministic.

```python
from pydantic import BaseModel, Field, ValidationError

class SupportReply(BaseModel):
    message: str = Field(max_length=600)
    category: str = Field(pattern="^(billing|technical|general)$")
    needs_human: bool

def validate(raw: str) -> SupportReply | None:
    try:
        return SupportReply.model_validate_json(raw)
    except ValidationError as e:
        log.warning("schema_violation", errors=e.errors())
        return None  # caller decides: retry, fallback, or escalate
```

Schema validation catches prompt-injection attempts that try to change your output format, truncation glitches, and models "helpfully" adding extra fields. Retry once on failure; if it fails again, fall back to a safe canned response rather than retrying forever.

## Guardrails for Agents: The Part That Actually Scares People

When the model can call tools, output text isn't the risk. Actions are. An agent that can `send_email`, `delete_record`, or `execute_sql` is a prompt injection away from disaster.

```python
# ❌ The model decides everything; one bad prompt and it's game over
def run_tool(name: str, args: dict):
    return TOOLS[name](**args)

# ✅ Constrain what's callable, and gate the dangerous stuff
TOOL_ALLOWLIST = {"search_docs", "get_order_status"}  # read-only defaults
DANGEROUS = {"send_email", "issue_refund", "delete_record"}

def run_tool(name: str, args: dict, user, confirmed: bool):
    if name not in TOOL_ALLOWLIST and name not in DANGEROUS:
        raise ValueError(f"tool {name} not permitted")
    if name in DANGEROUS:
        # scope to the user, require confirmation, and audit
        args = enforce_ownership(args, user)
        if not confirmed:
            return request_human_confirmation(name, args)
        audit.log(user, name, args)
    return TOOLS[name](**args)
```

The principle: **least privilege for tools, scoped credentials, human in the loop for anything destructive.** Treat the model like an untrusted intern with access to production — because functionally, it is.

## Layering: Cheap Checks First

Don't run an expensive LLM-based judge on every request. Order your checks from cheapest to most expensive, and short-circuit:

```python
def guarded_response(prompt, user):
    if not input_allowed(prompt):            # regex/blocklist — microseconds
        return refusal()
    if pii_detector.find(prompt):            # local DL model — ~10ms
        prompt = pii_detector.redact(prompt)
    raw = call_llm(prompt)
    if not schema_ok(raw):                   # deterministic — <1ms
        return fallback()
    flagged, cats = moderate(raw)            # provider API — ~100ms
    if flagged:
        return refusal()
    if high_stakes(user.surface):            # LLM judge — ~1s, only where it counts
        if not llm_judge(raw).is_acceptable:
            return escalate_to_human(raw)
    return raw
```

A stacking approach keeps your p50 latency low while still applying the heavy checks where mistakes are expensive. In practice very few requests ever reach the last gate.

## Trade-offs You Can't Escape

- **Latency vs safety.** Every guardrail adds time. Front-load deterministic checks; reserve model-based judging for high-stakes paths.
- **False positives vs false negatives.** A strict blocklist will annoy legit users. Track your false-positive rate and tune. A guardrail that blocks 5% of valid requests will get disabled by your own team.
- **Deterministic vs learned.** Regex and schema checks are predictable but brittle. Classifiers are flexible but can be fooled. Use both, in layers.
- **Fail-open vs fail-closed.** Safety says close. Availability says open. Decide per surface and log the decision.

## When You'd Actually Build This

- Any user-facing feature where output goes public (posts, comments, generated marketing copy)
- Anything touching money, health, legal, or children
- Agents with write access to real systems
- Regulated industries where "the AI said it" isn't a defense

If you're prototyping internally and outputs go nowhere real, you can start light. The moment real users see it, add the output layer.

## Takeaways

- Guardrails are three layers: input, output, and actions. Most teams forget output and actions.
- Use provider moderation as a cheap first pass, but set your own thresholds per surface and decide fail-open vs fail-closed deliberately.
- Moderation APIs won't catch fabrication, PII, tone, or business-logic violations — write targeted checks for those.
- Schema validation is the most reliable guardrail you can have; use it whenever output must be structured.
- For agents, constrain tools with allowlists, scoped credentials, and human confirmation for anything destructive.
- Order checks cheapest-first and short-circuit, so safety doesn't kill your latency.
- Every guardrail trades false positives for safety. Track and tune, or your own team will turn it off.
