# Distributed Tracing: Following One Request Across 15 Services

You just shipped it. The endpoint works. Then the first complaint rolls in: "the checkout page is slow." You look at the API logs — the request took 3.2 seconds and returned 200. Every service in the chain logged its own little slice, and all of them say "I was fine."

So which one ate the 3 seconds? With per-service logs alone, you're about to spend an afternoon playing detective. Distributed tracing is the fix: one ID that follows a single request through every service, database call, and queue hop, so you can finally see the whole journey instead of isolated snapshots.

We'll cover how traces actually work under the hood, how to wire up OpenTelemetry (the de facto standard), and the sampling decisions you can't skip — because tracing everything is a bill you don't want to pay.

## The Core Idea: One ID, One Journey

A trace is just a tree of **spans**. Each span represents one unit of work — an HTTP request handled, a database query executed, a message published. The first span is the root; everything that happens downstream hangs off it as children.

```
Trace: 4bf92f3577b34da6a3ce929d0e0e4736
└── POST /checkout                     (api-gateway, 3200ms)
    ├── auth-service: verify token     (850ms)
    │   └── redis: GET session         (810ms)
    ├── inventory-service: check stock (1900ms)  ← the culprit
    │   └── postgres: SELECT ...       (1880ms)  ← missing index
    └── payments-service: charge       (420ms)
```

That tree is the whole point. You don't just know the checkout was slow — you know *exactly* which span ate the time, and you can drill into its attributes, logs, and error state. This is what turns "3 seconds, somewhere" into "unindexed query in inventory-service."

The magic ingredient that makes this work across process boundaries is **context propagation**: each service passes the trace ID (plus a bit more) along with the request, so the next service can attach its spans to the same tree.

## The Header That Makes It All Work

If every service used a random ID, you'd have 15 unrelated traces. Propagation is the glue, and the W3C `tracecontext` standard is how it's done in practice. When service A calls service B over HTTP, it sends two headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              │  └─────── trace id ───────┘ └────── span id ──────┘ │
              version                                        trace flags
```

- **trace-id**: the one ID that's identical across all 15 services
- **parent-id**: the span ID of the caller, so the new span knows its parent
- **trace-flags**: `01` = sampled, `00` = not (more on sampling later)

You'll almost never parse these manually — your tracing library does it for you. But you *will* hit the failure mode where propagation breaks, and knowing what the header should look like makes that a 5-minute fix instead of an afternoon.

## Instrumenting Your First Span (OpenTelemetry)

OpenTelemetry is the standard these days: one SDK, and your spans can go to Jaeger, Zipkin, Datadog, Grafana Tempo, or whatever you run. Here's a minimal Node.js setup:

```js
// instrumentation.js — run this once at service startup
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { BatchSpanProcessor } = require('@opentelemetry/sdk-trace-base');

const provider = new NodeTracerProvider();
// Batch spans and ship them in the background — never block requests on tracing
provider.addSpanProcessor(new BatchSpanProcessor(new OTLPTraceExporter({
  url: 'http://otel-collector:4318/v1/traces',
})));

provider.register();
```

Now instrument the actual work. The key habit: **create a span, do the work, end the span** — and always record errors on it:

```js
const { trace } = require('@opentelemetry/api');
const tracer = trace.getTracer('checkout-service');

async function checkStock(productId) {
  // Span auto-inherits the incoming trace context from the request headers
  const span = tracer.startSpan('inventory.checkStock', {
    attributes: { 'product.id': productId },
  });

  try {
    const result = await db.query(
      'SELECT stock FROM products WHERE id = $1', [productId]
    );
    span.setAttribute('db.rows', result.rowCount);
    return result.rows[0];
  } catch (err) {
    // ❌ Swallowing the error loses the trace link to your bug
    // span.setStatus({ code: ERROR }); // ← don't skip this
    throw err;
  } finally {
    span.end(); // always end, even on the error path
  }
}
```

For HTTP frameworks there's usually an auto-instrumentation package that creates the incoming span and extracts the `traceparent` header for you. Use it. Hand-rolling propagation is where people get it wrong.

## The 80/20: Auto-Instrumentation First

The fastest win is enabling auto-instrumentation for your web framework and database client. One line in Node:

```js
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
// Wraps http, express, postgres, redis, etc. — spans appear with zero code changes
provider.register();
```

It's not magic — it monkey-patches your libraries at startup to create spans and propagate context. That gives you 80% of the value (the request tree, the timings, the DB queries) with 20% of the effort.

Use manual spans for the stuff auto-instrumentation can't see:

- Business steps: "charge payment", "send confirmation email"
- Background jobs that don't go through HTTP
- Anything where the timing is the business logic

The trade-off: auto-instrumentation spans are generic and noisy ("http.request" everywhere). Manual spans carry meaning. A healthy setup is mostly auto with a few well-placed manual spans on the parts you actually care about.

## Sampling: You Can't Trace Everything

Here's the uncomfortable part: tracing every request at high traffic is expensive. Every span is CPU, memory, and storage in your tracing backend. At 10k req/s, you're generating millions of spans an hour.

So we sample — keep the interesting traces, drop the rest. The two main strategies:

| Strategy | How it works | Pros | Cons |
|----------|--------------|------|------|
| **Head sampling** | Decide at the *start* (root span) whether to trace | Cheap, simple, consistent | You might drop the rare slow/error trace before it happens |
| **Tail sampling** | Trace everything, *then* decide what to keep | Can keep all errors + slow traces | Expensive — you pay for everything, and it needs a central collector |

The classic default: **head sampling at 5-10%**, plus a rule to always keep traces with errors. That way you get representative data without the bill. Tail sampling is for when you absolutely can't miss the rare 99.9th-percentile slow request — and you have the budget for it.

One trap: if your sampling decision isn't propagated in the `trace-flags` bit, each service decides independently, and you get half-traces — spans with no root, trees that end abruptly. The W3C header exists precisely to avoid this.

## Make Traces Searchable: Attributes, Not Just Trees

A trace tree tells you *where* time went. It doesn't tell you *which* user, *which* order, or *which* product. That's what span attributes are for:

```js
span.setAttribute('user.id', userId);
span.setAttribute('order.id', orderId);
span.setAttribute('http.status_code', 500);
```

These become the filter fields in your tracing UI. Without them, finding a specific trace is like searching for a needle in a haystack where all needles look identical. The rule of thumb: if you'd want to search for it during an incident, make it an attribute.

## Tie Traces to Logs

Spans are great for "where", logs are great for "what exactly happened". Don't keep them separate — inject the trace ID into your log lines:

```js
// ❌ Logs with no trace context
logger.error('Payment failed for user');

// ✅ Now you can jump from the log straight to the trace
logger.error({ trace_id: trace.getActiveSpan()?.spanContext().traceId },
  'Payment failed for user');
```

Most logging libraries have a correlation plugin that does this automatically (and OpenTelemetry's `baggage`/log correlation spec formalizes it). Once it's wired, your workflow becomes: see error in logs → click the trace ID → see the full request journey. That single link saves more incident time than any dashboard.

## What Actually Goes Wrong in Production

Things I've seen break, in order of frequency:

1. **Propagation dropped at a boundary** — a message queue, a gRPC call, or a hand-rolled HTTP client that doesn't forward the headers. Result: orphan spans, "traces" with one span. Fix: check the `traceparent` header at each hop.
2. **Clock skew** — spans depend on timestamps; if service B's clock is 30 seconds ahead of A's, your trace looks like time travel. Fix: NTP everywhere, and don't hand-roll span timing.
3. **Cardinality explosion** — adding `user.id` or `request.url` with a huge URL as attributes on every span. Your tracing backend starts eating disk like crazy. Fix: be deliberate about which attributes you set, and use `span.setAttribute` sparingly on hot paths.
4. **Sampling misconfigured at the edge** — the gateway traces 100%, downstream traces 10%. You get thousands of root spans with no children. Fix: propagate the decision, or set consistent sampling at the gateway.

## Takeaways

- Trace IDs and the W3C `traceparent` header are the glue — verify propagation at every service boundary (HTTP, queues, gRPC) before anything else.
- Turn on auto-instrumentation for your framework and DB client first; add manual spans only where the business logic lives.
- Sample at the head (5-10% is a sane start), always keep errors, and make sure the sampling decision propagates.
- Put searchable attributes on spans — user ID, order ID, status codes — or your traces are unqueryable during an incident.
- Inject the trace ID into your logs so you can go log → trace → fix in one click.
- Instrumentation is code, so it can fail silently: keep an eye on your collector's intake rate. Zero spans ingested usually means propagation or exporter config broke, not that traffic is fine.
