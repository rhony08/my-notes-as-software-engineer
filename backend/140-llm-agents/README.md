# Building LLM Agents

Function calling lets a model ask you to run a tool. That's one round trip: ask, call, answer. Use it to build a "chat with your docs" bot and you're fine.

Then someone asks for something that takes *five* steps, and the whole thing collapses. "Check last week's refunds, flag any over $500, and draft an email to each customer." One tool call won't cut it. The model has to decide what to do first, look at the result, decide what's next, and keep going until it's done.

That's an agent. And most teams build one by wrapping their function-calling loop in `while True:` — then spend the next month discovering why that's a bad idea.

## The Difference Between a Tool Call and an Agent

Worth being precise, because "agent" gets slathered over everything.

| | Tool call (single turn) | Agent (multi-turn) |
|---|---|---|
| Turns | One model → one tool → answer | Loop until a stop condition |
| Who decides the path | You, in code | The model, at runtime |
| Failure mode | Wrong tool / bad args | Runs forever, loops, goes off-script |
| Debugging | Watch the response | Replay the whole trace |
| Cost | Predictable | Anything from 1 to 50 calls |

An agent is **an LLM running a loop where it chooses its own next action**. The moment the model controls *how many times* and *in what order* tools fire, you've left the safe predictability of a single function call and entered a search problem — one your bill pays for.

```python
# ❌ Not an agent — just a pipeline. The path is fixed in code.
summary = llm("summarize", doc)
tags = llm("tag", summary)
post = llm("draft", summary, tags)

# ✅ An agent — the model picks the path, including how long it runs.
while not done:
    response = llm(messages, tools=TOOLS)
    if response.tool_calls:
        messages += run_tools(response.tool_calls)   # decide + act
    else:
        done = True                                   # model chose to stop
```

That `while` loop is the whole idea. Everything hard about agents is about keeping it from going wrong.

## The Anatomy of an Agent Loop

Almost every agent has the same four parts. Skip one and it breaks in a specific, predictable way.

```text
┌─────────────────────────────────────────────────────────┐
│  1. CONTEXT      system prompt + tools + memory + task   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. DECIDE       model reasons and picks the next step   │
│                  (a tool call, a sub-answer, or "done")  │
└─────────────────────────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
      tool call(s)                 final answer
              │                       │
              ▼                       ▼
┌──────────────────────┐      return to user
│  3. ACT + OBSERVE     │
│  run tool, append     │
│  result to history    │
└──────────────────────┘
              │
              ▼
┌──────────────────────┐
│  4. STOP CHECK        │──── loop back to 2, or stop
│  budget? done? stuck? │
└──────────────────────┘
```

### 1. Context — what the model can see

The model is stateless. Every turn, you rebuild its entire world: the system prompt, the tool definitions, the conversation so far, and whatever memory you retrieved. Get this wrong and the agent "forgets" mid-task.

### 2. Decide — reasoning before acting

Modern models can produce a short reasoning step before the tool call. It's tempting to skip it for speed. Don't — it's also your best debugging artifact when the agent does something dumb.

### 3. Act + observe — the ground truth comes back

The tool result is appended as a message the model reads next turn. This is where reality enters the loop. **Never let the model's *guess* stand in for a tool result.**

### 4. Stop — the most-skipped part

An agent without a stop condition is a money fire. You need at least three brakes: a max-step budget, a "no progress" detector, and a deterministic done signal. More on that below.

## ReAct: The Pattern Under Most Agents

The "reason → act → observe" rhythm has a name: **ReAct** (Reasoning + Acting). You don't need a framework to implement it. You need a system prompt that asks for a thought before each action, and a loop that feeds observations back.

```python
SYSTEM_PROMPT = """
You are a research assistant. Work in short steps.

For each step:
1. THINK: state what you know and what you still need (one or two sentences).
2. ACT: call exactly one tool, or give a final answer.
3. After each tool result, THINK again before acting.

Rules:
- Never invent a tool result. If you didn't call a tool, you don't know the answer.
- If a tool fails twice, stop and report the failure instead of retrying forever.
- When the task is complete, reply with FINAL: <answer> and stop calling tools.
"""
```

That last rule matters more than it looks. Models are agreeable — they'll happily keep working forever if you let them. You have to tell them what *done* looks like.

A real trace ends up looking like this:

```text
THINK: I need the refund total for last week. I'll query the database.
ACT:   query_refunds(start="2026-09-14", end="2026-09-20")
OBS:   43 refunds, total $12,840, 5 over $500

THINK: Got the data. The 5 large refunds need customer emails.
ACT:   get_customers(refund_ids=[1002, 1017, 1033, 1041, 1058])
OBS:   5 customers with verified emails

THINK: I have everything. Drafting emails and finishing.
FINAL: Drafted 5 emails for the flagged refunds. [summary...]
```

Read that top to bottom and you can debug it. That's the test: **if you can't read the trace and explain every decision, you can't operate the agent.**

## Memory: The Part Everyone Gets Wrong

An agent's "memory" is really three different things, and conflating them causes most of the confusion.

| Memory type | What it holds | Where it lives | Failure if missing |
|---|---|---|---|
| Short-term | This task's steps | The message history | Agent forgets what it just did |
| Working | Scratchpad / plan | A note in the prompt | Loses the thread on long tasks |
| Long-term | Facts across sessions | A vector store / DB | Re-asks the user the same things |

The trap: **message history grows every step.** After 20 tool calls you're re-sending 20 tool results on every new request. Cost climbs quadratically, latency climbs linearly, and eventually you blow the context window mid-task.

The fix is compression, not more context:

```python
# ❌ Let history grow forever — slow, expensive, will hit the limit
messages.append(tool_result)

# ✅ Keep old results as summaries, recent ones verbatim
def compact(messages, keep_recent=6):
    old, recent = messages[:-keep_recent], messages[-keep_recent:]
    summary = llm(f"Summarize these steps in <=3 bullets each:\n{old}")
    return [system] + [summary_message(summary)] + recent
```

Summarize the past, keep the present exact. Don't summarize a tool result the model is *actively* reasoning about — it'll hallucinate the details.

## Prompting an Agent Is Not Prompting a Chatbot

A chatbot answers in one shot. An agent takes many shots, so tiny prompt differences compound over ten steps into wildly different behavior.

Three things that matter far more for agents than for chat:

**1. Explicit stop conditions.** Tell it when to quit and how to signal it.

```text
# ❌ Vague — the model will "decide" and often decides to keep going
Keep working until the task is complete.

# ✅ Explicit — a signal you can parse and budget you can enforce
You have at most 8 tool calls. When done, respond with "FINAL:" and nothing else.
If you cannot complete the task in 8 calls, respond with "BLOCKED: <reason>".
```

**2. One tool per step (at first).** Parallel tool calls are fast but make traces impossible to reason about while you're building. Start serial, add parallelism once it's reliable.

**3. Tell it how to fail.** Agents given no failure protocol will retry a broken tool until the budget runs out. Give them an exit.

```text
If a tool returns an error, retry ONCE with corrected arguments.
If it fails again, stop and report what failed — do not keep retrying.
```

## The Failure Modes You'll Actually Hit

These aren't hypothetical. They're the ones that show up in week one.

| Failure | What it looks like | The fix |
|---|---|---|
| Infinite loop | Same tool, same args, forever | Hash `(tool, args)`; stop on repeat |
| Thrashing | Alternates between two tools | Cap total steps; detect no-progress |
| Hallucinated action | Says it did something it didn't | Require a tool result before claims |
| Wrong tool | Picks `search` when it needs `lookup` | Tighten tool descriptions |
| Context blowout | Works until step 15, then errors | Compact history |
| Silent cost runaway | One task burns $4 | Hard token/call budget |

The wrong-tool problem is worth dwelling on, because it's not the model's fault. Tool selection is **retrieval over your tool descriptions**. Vague descriptions get vague choices.

```python
# ❌ The model can't tell these apart
{"name": "search", "description": "Search for information"}

# ✅ Description says when to use it AND when not to
{
  "name": "search_knowledge_base",
  "description": (
    "Search internal documents for policy or product info. "
    "Use for facts that change rarely. Do NOT use for live data "
    "like orders or inventory — use get_order instead."
  )
}
```

Add a negative example ("do NOT use for...") and wrong-tool rates drop fast. Every tool description is a prompt.

## Budgets: The Agent's Only Real Safety Net

You can't unit-test a model's judgment. You *can* cap what it's allowed to spend. Do this on day one, not after the first surprise invoice.

```python
MAX_STEPS = 8           # hard ceiling on loop iterations
MAX_TOKENS = 50_000     # total across the whole task
MAX_WALL_CLOCK = 60     # seconds; kill the task if it exceeds this

def run_agent(task, tools):
    steps, spent = 0, 0
    messages = [system_prompt(task), user_message(task)]
    seen = set()

    while steps < MAX_STEPS:
        response = llm(messages, tools=tools)
        spent += response.usage.total_tokens
        if spent > MAX_TOKENS:
            return "BLOCKED: token budget exceeded"

        if not response.tool_calls:          # model chose to finish
            return response.content

        for call in response.tool_calls:
            key = (call.name, json.dumps(call.args, sort_keys=True))
            if key in seen:                  # loop detector
                return "BLOCKED: repeated action, no progress"
            seen.add(key)
            messages.append(run_tool_safely(call))

        steps += 1

    return "BLOCKED: step budget exceeded"   # always return, never hang
```

Notice every exit path returns something. An agent that can throw and leave the user hanging is worse than one that says "I got stuck." Fail loudly, fail short, fail cheap.

## When NOT to Build an Agent

The honest rule: **most problems people reach for an agent to solve are just a fixed pipeline.**

Agents pay for themselves only when the *path* is genuinely unpredictable — you can't know the steps until you see the data. If the steps are knowable, write them down. A pipeline is faster, cheaper, testable, and won't randomly decide to do something creative.

| Use a fixed pipeline when... | Use an agent when... |
|---|---|
| The steps are known in advance | The steps depend on what you find |
| Determinism matters | Exploration is the point |
| You need to unit-test behavior | You need to handle open-ended input |
| Cost must be predictable | Some variance is acceptable |
| "Summarize → tag → store" | "Investigate and report" |

For "summarize this doc," a pipeline wins every time. For "figure out why this customer's account is broken," the investigation path is a real search problem — that's where an agent earns its keep.

## Takeaways

- **An agent is a loop, not a magic model.** The model chooses the path; your code owns the guardrails.
- **ReAct is just think → act → observe.** You don't need a framework. You need a system prompt that asks for a thought and a loop that feeds results back.
- **Always cap three things:** max steps, total tokens, wall-clock time. Do it before you launch, not after.
- **Detect loops by hashing `(tool, args)`.** Same call twice with no new result means you're stuck — bail.
- **Compact long histories.** Summarize old steps, keep recent tool results verbatim.
- **Tool descriptions are prompts.** Add "use this for X, do NOT use for Y" and watch error rates drop.
- **Reach for a pipeline first.** Only build an agent when the steps genuinely can't be known ahead of time.
- **If you can't read the trace and explain every decision, you can't operate it.** Instrument before you scale.
