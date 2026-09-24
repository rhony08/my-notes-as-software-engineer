# Evaluating LLM Outputs: Evals That Mean Something

You tweaked the system prompt. It feels better. The three examples you tried by hand all came back clean. You ship it.

Two weeks later someone files a ticket: "the assistant gave a customer the wrong refund policy." You re-read your prompt, shrug, and tweak it again. This is how most teams "evaluate" LLMs — vibes, a handful of cherry-picked examples, and hope.

The problem isn't that the model is unreliable. It's that **you have no way to know when it changes**. Swap a model version, edit a prompt, bump the temperature, and everything downstream shifts — with nothing to tell you whether you got better or worse.

Evals are how you turn "I think it's better" into "I can prove it's better, and I'll know the moment it isn't."

## Why Evals Feel Hard (and Why They Aren't)

People avoid evals because they imagine a research project: golden datasets, human raters, statistical significance testing. That's one end of the spectrum. The other end is a 30-line script that catches the regressions that actually bite you.

The trick is to stop asking "is this model good?" — a question with no bottom — and start asking **"did this change break the thing I care about?"** That's a question a small, boring test suite answers perfectly.

Three levels, from cheapest to fanciest:

| Level | What it is | Catches | Cost to build |
|---|---|---|---|
| Assertion evals | Hard checks on exact properties | Crashes, format drift, obvious wrongness | Minutes |
| Reference evals | Compare output to a known-good answer | Wrong facts, missed fields, bad classifications | Hours |
| Model-graded evals | An LLM scores open-ended output against a rubric | Tone, helpfulness, reasoning quality | Hours + API spend |

Start at the top. Most teams ship with zero and think they need the bottom.

## Level 1: Assertions — The Stuff You Should Never Ship Without

Before you wonder whether the answer is *good*, check whether it's *valid*. A shocking number of production incidents are just "the output stopped being parseable."

```python
def test_returns_valid_json(res):
    # If this fails, everything downstream is broken. Cheapest possible eval.
    data = json.loads(res)          # raises if not JSON
    assert "summary" in data
    assert isinstance(data["summary"], str)

def test_no_refund_advice_on_non_refund_ticket(res):
    # Policy guardrail — a hard "never" rule, checked with a plain substring match.
    assert "you are eligible for a refund" not in res.lower()
```

These look trivial. They are. That's the point — they cost nothing, run on every commit, and they catch the regressions that hurt most: the model wrapping JSON in prose again, leaking a forbidden phrase, ignoring your word limit.

**If your LLM output feeds into code, you already need assertion evals.** They're just unit tests for stochastic functions.

## Level 2: Reference Evals — Compare Against a Known-Good Answer

For tasks with a *right answer* — classification, extraction, routing, ranking — you don't need a model to grade anything. You need a labeled set and a scoring function.

```python
# The "dataset" is just a list of inputs + expected outputs. Ten is enough to start.
cases = [
    {"ticket": "My card was charged twice",   "expected": "billing"},
    {"ticket": "App crashes when I upload",   "expected": "bug"},
    {"ticket": "How do I export my data?",    "expected": "question"},
]

def accuracy(predict_fn, cases):
    correct = 0
    for c in cases:
        # Constrain the output so scoring is a string compare, not fuzzy matching.
        got = predict_fn(c["ticket"])   # returns one of billing/bug/question
        correct += got == c["expected"]
    return correct / len(cases)

# ✅ One number you can watch over time.
print(f"{accuracy(classify, cases):.0%}")   # start here, not at "perfect"
```

A real eval harness — [promptfoo](https://www.promptfoo.dev/), `deepeval`, or your own `pytest` file — just runs this across many cases and diffs the score when you change the prompt. Ten labeled examples beat zero, every time. You can grow the set as you find failures: **every production bug becomes a test case.** That's the single highest-leverage habit in this whole article.

For open-ended answers where there's no single string, use similarity or a reference fact-list:

```python
# ❌ Exact match on prose fails on every paraphrase
# assert output == "The refund window is 30 days."

# ✅ Check that required facts survived, ignoring wording
REQUIRED_FACTS = ["30 days", "original payment method"]
def covers_facts(output, facts=REQUIRED_FACTS):
    return all(f.lower() in output.lower() for f in facts)

assert covers_facts("Refunds are processed within 30 days to your original payment method.")
```

## Level 3: Model-Graded Evals — When "Right" Is Fuzzy

Tone, helpfulness, coherence, "did it follow the instruction" — no string comparison captures these. So you use a *different* LLM call as a judge, scoring against a rubric.

```python
RUBRIC = """Score the answer 1-5 on each dimension. Return JSON.
- grounded: every claim is supported by the provided context (no invented facts)
- helpful: directly answers the user's question
- safe: no prohibited advice or policy violations
"""

def grade(question, answer, context):
    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": RUBRIC},
            {"role": "user", "content": f"Q: {question}\nContext: {context}\nAnswer: {answer}"},
        ],
        response_format={"type": "json_object"},
    )
    return json.loads(resp.choices[0].message.content)
```

Two rules keep LLM judges honest:

- **Give them a rubric, not a request to "be fair."** Vague judges are noise.
- **Ask for structured scores, not a paragraph.** You can't average an essay.

The big caveat: **LLM judges have known biases.** They prefer longer answers, favor their own model family, and are swayed by presentation. Treat the score as directionally useful, not ground truth. And grade with a *different* model than the one you're evaluating — using the same model to grade itself is marking your own homework.

## The Trap: Eval Sets That Overfit to Themselves

Here's where eval suites quietly die. You tune your prompt against the same 20 examples you score on. The score climbs to 100%, you feel great, and the feature is no better in the real world. You've just overfit to your test set — same sin as training on your validation data.

The fix is boring and works:

- **Hold out a hidden set.** Tune on 80%, check against the 20% you didn't touch. Only trust that number.
- **Pull cases from production**, not just from your imagination. Real user inputs are weirder than anything you'll invent.
- **Add failures as you find them.** Every complaint, every bad output becomes a permanent regression test.
- **Version your evals with your prompts.** A score means nothing without knowing which prompt and model produced it.

```text
prompt v1 + gpt-4o-mini  -> accuracy 0.82
prompt v2 + gpt-4o-mini  -> accuracy 0.91   ✅ keep
prompt v2 + gpt-4o       -> accuracy 0.93   (worth 3x the cost? your call)
```

That table is the entire point. It turns prompt engineering into engineering: a change, a measurement, a decision.

## What to Actually Measure

Not everything. Pick the metrics that map to things users care about:

- **Task success rate** — did it do the job? (accuracy, pass rate)
- **Schema validity** — does it still parse? (the assertion eval)
- **Groundedness** — did it invent facts? (the RAG killer)
- **Refusal / safety** — did it say something it shouldn't?
- **Latency and cost** — quality means nothing if it's too slow or too expensive

Wire the cheap ones into CI so every prompt change runs them automatically. A red build that says "groundedness dropped from 0.95 to 0.71" is worth more than a week of manual spot-checks.

## Takeaways

- If you can't measure a change, you can't know if it helped. Vibes are not a strategy.
- Start with assertion evals — format, guardrails, hard rules. They're unit tests and they're free.
- Use reference evals wherever there's a right answer. Ten labeled cases is a real start.
- Reserve LLM judges for the fuzzy stuff, and always give them a rubric — then remember they're biased.
- Never tune on the same cases you score on. Hold out a set, or you're just memorizing.
- Turn every production failure into a permanent test case. That's how the suite earns its keep.

The teams that ship reliable AI features aren't the ones with the smartest prompts. They're the ones who can answer "is it better?" in one command.
