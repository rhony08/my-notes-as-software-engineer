# Structured Outputs and JSON Mode

You ask a model to extract an invoice and return JSON. It gives you this:

```
Sure! Here's the invoice data you asked for:

{
  "vendor": "Acme Corp",
  "total": "$1,240.00",
  "date": "March 3rd, 2026"
}

Let me know if you need anything else!
```

Now your parser crashes. The model was helpful, chatty, and *wrapped your JSON in a sentence*. So you do what everyone does: regex out the `{...}`, strip the fences, add `JSON.parse` in a try/catch, and pray.

That works right up until it doesn't — a trailing comma here, a `"$1,240.00"` where you wanted a number there — and suddenly your "AI feature" is a 500 error in production.

Structured outputs exist to kill that whole class of bug. Instead of coaxing JSON out of a chat model and cleaning it up, you tell the API what shape you want and it *guarantees* that shape.

## The Real Problem Isn't JSON — It's Trust

Parsing the model's text is the symptom. The actual problem is that you can't trust free-text output as a data contract.

```python
# ❌ Hope-driven development: parse, clean, retry, repeat
raw = llm("Extract the vendor and total as JSON")
text = raw.strip().removeprefix("```json").removesuffix("```")  # fragile
try:
    data = json.loads(text)
except json.JSONDecodeError:
    data = json.loads(regex_search(r"\{.*\}", text))  # more fragile
if not isinstance(data.get("total"), (int, float)):
    raise ValueError("Model gave me a string again")   # the classic
```

Every line there is a bug waiting for the next model version, the next temperature bump, the next weird edge case. You've built a parser for a format nobody promised to follow.

Structured outputs flip it: **the constraint moves from your parser into the model's decoder.** If the schema says `total` is a number, you get a number. Not `"$1,240.00"`. Not a paragraph around it. A number.

## Three Levels of "Structured"

Not all approaches are equal. There's a spectrum, and knowing where you sit tells you how much defensive code you still need.

| Approach | How it works | Guarantee | Still needs cleanup? |
|---|---|---|---|
| Prompt-only | "Please respond in JSON" | None | Yes — always |
| JSON mode | API forces *valid* JSON | Valid JSON, any shape | Yes — keys/types can drift |
| Schema-constrained (structured outputs) | API forces JSON matching your schema | Valid JSON **+** your schema | No parsing cleanup |

The middle one trips people up. **JSON mode guarantees syntax, not shape.** Turn it on and the model will always emit parseable JSON — but it might use `"amount"` where you asked for `"total"`, or hand you a stringified number. You've moved the bug from "invalid JSON" to "valid JSON with the wrong fields," which is honestly *harder* to catch.

Schema-constrained output is the one you actually want when a downstream system depends on the data.

## How It Actually Works

The magic isn't a better prompt. It's **constrained decoding** — the API restricts which tokens the model is allowed to generate at each step, based on your schema.

Think of it like autocomplete with guardrails:

```text
Schema:  { "sentiment": "positive" | "negative" | "neutral", "score": number }

Step 1: model must emit '{'
Step 2: model must emit '"sentiment"'
Step 3: model must emit ':' then one of "positive"/"negative"/"neutral"
Step 4: ...and so on until the object closes
```

At each position, tokens that would break the schema are simply masked out of the probability distribution. The model still *chooses* the values, but it can't choose a shape that violates the contract.

That's why it's a hard guarantee rather than a strong suggestion. You can't "convince" a decoder to ignore grammar.

## A Real Example: Extraction You Can Trust

Most SDKs now expose this. The pattern is the same everywhere: define a schema, pass it, parse with confidence.

```python
from pydantic import BaseModel
from openai import OpenAI

class Invoice(BaseModel):
    vendor: str
    total: float
    currency: str
    line_items: list[str]

client = OpenAI()

# ✅ The API constrains generation to match Invoice. No fences, no prose.
response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "Extract the invoice fields."},
        {"role": "user", "content": invoice_text},
    ],
    response_format=Invoice,
)

invoice = response.choices[0].message.parsed   # typed, validated, done
print(invoice.total * 1.1)                       # it's a float, not a string
```

Compare that to the scraping version at the top. No regex, no try/catch, no "wait, is this a string again?" The schema is the contract and the decoder enforces it.

The same idea exists for JSON Schema directly if you're not in Python:

```json
{
  "type": "object",
  "properties": {
    "vendor":   { "type": "string" },
    "total":    { "type": "number" },
    "currency": { "type": "string", "enum": ["USD", "EUR", "GBP"] },
    "line_items": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["vendor", "total", "currency"],
  "additionalProperties": false
}
```

`additionalProperties: false` matters more than people think — it stops the model from inventing extra keys your code doesn't expect.

## When Structured Outputs Bite You

It's not free. A few things to know before you rip out every parser.

**Schema complexity has a cost.** Constrained decoding can be slow to compile for large or deeply nested schemas. The first request against a big schema can add latency while the grammar gets built. If you have a monster object, consider splitting it.

**Strictness can reduce quality slightly.** Forcing a rigid shape sometimes nudges the model into less natural answers — especially for tasks that are better done in prose first. If you need a *reason* and a *verdict*, ask for the reasoning in a separate call, or let it write free-form and extract afterward.

**Not every provider supports it equally.** JSON mode is common; full schema enforcement is newer and varies. Check what your model actually supports before designing around it.

**Enums and unions are your friends.** `"enum": ["approved", "rejected", "needs_review"]` is worth a paragraph of prompt. It removes a whole category of "the model said something close but wrong" bugs.

```python
# ❌ Asking for a free-text category you'll have to normalize
# "Classify as one of: bug, feature, or question"  -> "feature request" 😬

# ✅ Constrain it so there's nothing to normalize
class Triage(BaseModel):
    category: Literal["bug", "feature", "question"]
    priority: int = Field(ge=1, le=5)
```

## Where This Fits in a Pipeline

Structured outputs shine at the *boundaries* — anywhere LLM output feeds into something typed.

- **Extraction** — invoices, resumes, tickets, receipts into your DB
- **Classification** — routing, tagging, sentiment, moderation verdicts
- **Tool arguments** — the args the model sends to your function are just structured output
- **Chaining** — one step's typed output becomes the next step's typed input

If an LLM is generating data for *another program*, it should be structured. If it's generating data for a *human*, leave it free-form.

## Takeaways

- Free-text parsing is a bug factory. If a program consumes the output, make the output a contract.
- JSON mode gives you valid JSON — not the shape you wanted. Don't confuse the two.
- Schema-constrained decoding is a real guarantee, not a strong suggestion. Use it for anything downstream depends on.
- Keep schemas tight: `additionalProperties: false`, enums, and required fields do the work prompts can't.
- Watch the trade-offs — heavy schemas cost compile time, and rigid shapes can hurt output quality on reasoning-heavy tasks.

The rule of thumb I keep landing on: **talk to humans in prose, talk to programs in schemas.**
