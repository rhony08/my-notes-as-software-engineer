# Structured Logging Patterns

It's 3am, the pager just went off, and you're staring at a log line that says:

```
User 10452 did something at 2026-08-18 02:58:33 and it failed with error null
```

What did the user do? What failed? Which request was this even part of? You `grep` for more context, find nothing useful, and now you're manually checking timestamps to piece together what happened. This is the real cost of unstructured logging — and it's not paid at write time, it's paid at 3am when your system is on fire and your logs can't answer a single question.

Structured logging fixes this by making every log line machine-readable: instead of a sentence, you get fields. This article covers what structured logging actually looks like, how to propagate context (correlation IDs), how to handle sensitive data, and the mistakes I've seen teams make when they "switch to JSON logging" without changing anything else.

## What "Structured" Actually Means

Unstructured logging is string interpolation:

```javascript
// ❌ A sentence. Good luck querying this.
logger.info(`User ${user.id} created order ${order.id} with total ${order.total}`);

// ✅ Fields. Queryable, filterable, aggregatable.
logger.info({
  event: "order.created",
  userId: user.id,
  orderId: order.id,
  total: order.total,
  currency: order.currency
});
```

The second one isn't just prettier — it changes what you *can do* with the logs. Your log aggregation tool (ELK, Loki, Datadog, whatever) can now:

- Filter by `userId` without regex
- Build a graph of `order.created` events over time
- Alert on `total > 1000` without parsing text
- Correlate across services by a shared field

With plain text, all of that is string matching against a format that drifts every time someone edits a log message.

## Why This Matters

### Debugging Becomes Querying

When something breaks, the difference between:

```
grep "user 10452" app.log
```

and:

```
{userId: "10452"} | sort by timestamp
```

is the difference between hoping and knowing. Structured logs turn "let me search for a fragment of a sentence" into "give me everything that happened for this entity, in order."

### Alerts Stop Being Garbage

Unstructured log alerts are regex against human sentences — fragile and full of false positives. Structured alerts are field conditions: `level = ERROR AND event = payment.failed`. That's a query, not a pattern match.

### Correlation IDs Actually Work

This is the big one for distributed systems. If every service logs its own unstructured lines, tracing a single request across three services is archaeology. With structured logging, you thread a `requestId` (or trace ID) through the whole chain, and every log line from every service carries it. Now the 3am question "what happened to this request?" becomes one query across services.

## Log Levels Done Right

Most teams have levels but use them wrong. The classic failure: everything is `INFO`, including errors, because "we'll filter later." You won't. Here's the level contract that actually holds up:

| Level | Use it for | Example |
|-------|-----------|---------|
| `DEBUG` | Development detail, noisy, off in prod | SQL queries, request bodies |
| `INFO` | Lifecycle events, normal operations | "service started", "order created" |
| `WARN` | Something's off but handled | retry happened, cache miss storm, slow query > threshold |
| `ERROR` | Something failed, needs attention | exception caught, external call failed |
| `FATAL` | Process can't continue | failed to bind port, corrupt config |

The rule I'd enforce: **an ERROR log line must represent a thing a human might need to act on.** If you log every caught exception as ERROR, your ERROR stream becomes noise and real problems drown in it.

```javascript
// ❌ Catching and logging as ERROR — this is a retryable, expected failure
try {
  await sendEmail();
} catch (err) {
  logger.error("email failed", { error: err.message }); // now every blip is "ERROR"
}

// ✅ WARN for handled/retryable, ERROR only when it actually breaks something
try {
  await sendEmail();
} catch (err) {
  logger.warn("email.send_failed, will retry", { error: err.message, attempt });
  // retry logic...
}
```

## Context Propagation: The Part Everyone Skips

Switching to JSON logs is the easy 20%. The hard 80% is making sure every log line has the context that makes it useful. That means request-scoped fields: `requestId`, `userId`, `route`, `durationMs`, `statusCode`.

The pattern in Node (pino):

```javascript
const logger = require("pino")();

app.use((req, res, next) => {
  // child logger inherits parent + adds fields to EVERY log in this request
  req.log = logger.child({
    requestId: req.headers["x-request-id"] || crypto.randomUUID(),
    path: req.path,
    method: req.method
  });
  next();
});

// Anywhere in the handler:
req.log.info({ userId: user.id }, "auth.check"); // includes requestId, path, method automatically
```

Same idea in Go with `slog`:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

func handler(w http.ResponseWriter, r *http.Request) {
    // With() returns a derived logger carrying these fields everywhere
    reqLogger := logger.With(
        "request_id", r.Header.Get("X-Request-Id"),
        "path", r.URL.Path,
    )
    reqLogger.Info("order.created", "order_id", order.ID, "total", order.Total)
}
```

And Python's `structlog`:

```python
import structlog

logger = structlog.get_logger()

def handler(request):
    log = logger.bind(request_id=request.headers.get("X-Request-Id"), path=request.path)
    log.info("order.created", order_id=order.id, total=order.total)
```

One field matters more than all others: **the correlation/trace ID**. In a microservice world, if your logs don't share one ID per request across services, each service's logs are an island. Pass `x-request-id` (or whatever your tracing system uses) between services, and log it everywhere. This single habit saves more debugging time than any other logging practice I know.

## Redacting Sensitive Data

Structured logging makes a new problem easy to create: now it's *trivial* to log the entire request object, including passwords, tokens, and PII. And because it's structured, it gets shipped straight into your log aggregation tool and stored there for months.

```javascript
// ❌ Logging the whole body — credit cards, passwords, tokens, all of it
logger.info("request received", { body: req.body });

// ✅ Explicit allowlist of what you actually need
logger.info("request received", {
  userId: req.body.userId,
  action: req.body.action,
  amount: req.body.amount
});
```

Practical rules:

- **Never log raw passwords, tokens, or full credit card numbers. Ever.**
- Log only the fields you need, not the whole object.
- Use redaction if your logger supports it (pino has `redact`):

```javascript
const logger = require("pino")({
  redact: ["req.headers.authorization", "user.password", "card.number"]
});
```

- Consider masking emails/phones if you store logs long-term (GDPR etc. makes this a compliance issue, not just a taste issue).

## Common Pitfalls

**Logging in hot paths.** `logger.debug()` on every iteration of a million-item loop is a CPU + I/O tax on every request, even if you never read those logs. If a log line wouldn't help you debug, don't emit it. Log what you'd query, not everything you can.

**Logging the same error twice.** Catch → log → rethrow → the top-level handler logs again. Now your error stream shows two ERROR lines per failure, and deduplicating by hand gets old fast. Log at the boundary where you handle the error, not at every frame.

**Giant log lines.** A 2MB log line (hello, serialized response body) breaks every log tool's ingestion pipeline and makes the log useless anyway — nobody reads a 2MB line, and most systems truncate. Keep lines small; if you need the payload, store it elsewhere and log a reference.

**JSON string inside a JSON field.** Don't do `error: JSON.stringify(err)` — that's a string, not a field, and it kills queryability. Log the error as a structured object (`error.message`, `error.stack`) or let the serializer handle it.

## The Honest Trade-offs

Structured logging isn't free, and it's worth being straight about the costs:

- **Human readability.** `{"level":30,"time":...,"msg":"order.created"}` is uglier than `Order 42 created`. Fix: use a pretty-printer in dev (`pino-pretty`, `slog` text handler locally) and JSON in prod. You get both.
- **Slightly more bytes.** JSON has overhead vs short strings. With compression at the transport level (most log shippers do this), the real cost is negligible.
- **Discipline required.** The benefit only materializes if everyone uses consistent field names and the correlation ID consistently. Half-adopted structured logging — some services structured, some not — is nearly as bad as none, because you can't correlate anyway.
- **Migration effort.** Existing unstructured logs won't retroactively become structured. You migrate the important services and leave the rest on the old format until they get touched.

None of these are reasons to skip it — they're the reasons to do it deliberately instead of slapping `JSON.stringify` on everything.

## Takeaways

- Every log line is a list of fields, not a sentence: `event`, IDs, and the values that make the event meaningful.
- Enforce the level contract: ERROR means "a human might need to act", not "I caught an exception".
- Thread a correlation ID through every service and log it on every line — it's the single highest-value field in a distributed system.
- Log the fields you'd query, never whole objects; redact secrets and PII at the logger level, not by hoping.
- Use pretty-printing locally so you get human readability in dev and queryability in prod.
- One JSON logger library doesn't make you "structured" — consistent field naming and context propagation do.
