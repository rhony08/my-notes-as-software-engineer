# Observability: Metrics, Logs, and Distributed Tracing

Your production server is running. Is it healthy? You check the logs—thousands of lines scrolling by. CPU looks fine. Response times seem okay. But users are complaining about slow checkouts.

Where's the problem? Is it the database? The payment service? A slow third-party API?

Without observability, you're flying blind. With it, you can pinpoint exactly what's wrong in seconds.

## The Three Pillars (And Why They Matter)

**Metrics** tell you *something* is wrong.
**Logs** tell you *what* happened.
**Traces** tell you *where* it happened.

You need all three. Here's why:

| Pillar | Question It Answers | Example Use Case |
|--------|---------------------|------------------|
| Metrics | Is the system healthy? | Alert when error rate > 1% |
| Logs | What went wrong? | Debug why payment failed |
| Traces | Where is the latency? | Find which service is slow |

## Metrics: The Pulse of Your System

Metrics are aggregated numbers—counters, gauges, histograms. They're cheap to store and fast to query, making them perfect for dashboards and alerts.

### Types of Metrics

**Counters** only go up (until reset). Use for: requests served, errors, messages processed.

```javascript
// ✅ Good: Increment a counter
httpRequestsTotal.inc({ method: 'POST', path: '/checkout', status: 200 });

// ❌ Bad: Storing every request timestamp
// This will explode your memory/storage
requestLog.push({ timestamp: Date.now(), path: '/checkout' });
```

**Gauges** go up and down. Use for: current connections, queue depth, memory usage.

```javascript
// Active database connections
dbConnectionsGauge.set(activeConnections);

// Queue size
queueDepthGauge.set(queue.length);
```

**Histograms** track distributions. Use for: latency, request sizes.

```javascript
// ❌ Wrong: Just tracking average latency
avgLatency = totalTime / requestCount; // Averages hide outliers

// ✅ Better: Use histograms to see percentiles
httpDurationHistogram.observe(durationMs);
// Now you can query: p50, p95, p99 latency
```

### What Metrics to Track

The **RED method** for request-driven services:
- **R**ate: Requests per second
- **E**rrors: Failed requests per second
- **D**uration: Latency distribution (p50, p95, p99)

The **USE method** for resources (CPU, memory, disk):
- **U**tilization: How busy is it?
- **S**aturation: How much is queued?
- **E**rrors: Error count

### Code Example: Prometheus Metrics

```javascript
const promClient = require('prom-client');

// Create a Registry
const register = new promClient.Registry();

// Default metrics (CPU, memory, etc.)
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5] // seconds
});

// Middleware to track
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    end({ method: req.method, route: req.path, status_code: res.statusCode });
  });
  next();
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

## Logs: What Happened?

Logs are your detailed record of events. When something breaks at 3 AM, good logs are the difference between a 10-minute fix and a 4-hour debugging session.

### Structured Logging: Non-Negotiable

```javascript
// ❌ NEVER DO THIS: Unstructured logs
console.log(`User ${userId} logged in at ${new Date()}`);
console.log(`Error processing payment: ${err}`);

// ✅ DO THIS: Structured JSON logs
logger.info('User logged in', { 
  userId, 
  timestamp: new Date().toISOString(),
  source: 'auth-service'
});

logger.error('Payment processing failed', { 
  error: err.message,
  stack: err.stack,
  orderId,
  amount,
  source: 'payment-service'
});
```

**Why structured logs?** Because grep is a terrible way to search logs at scale. With JSON:

```bash
# Find all errors in payment service from last hour
jq 'select(.level == "error" and .source == "payment-service")' logs.json

# In Grafana Loki or similar:
{level="error", source="payment-service"} |= "payment"
```

### Log Levels: Use Them Correctly

| Level | When to Use | Example |
|-------|-------------|---------|
| DEBUG | Development details | "Query: SELECT * FROM users WHERE id=?" |
| INFO | Normal operations | "User logged in", "Order completed" |
| WARN | Unexpected but handled | "Retrying connection (attempt 3/5)" |
| ERROR | Failures requiring attention | "Payment gateway timeout" |
| FATAL | System can't continue | "Database connection pool exhausted" |

```javascript
// ❌ Wrong: Everything at INFO level
logger.info('User not found'); // Should be WARN?
logger.info('Database connection failed'); // Should be ERROR?

// ✅ Correct: Appropriate levels
logger.warn('User not found', { userId }); // Something unusual
logger.error('Database connection failed', { error: err.message });
```

### Correlation IDs: Connecting the Dots

In a distributed system, one user request might hit 5 services. How do you trace it end-to-end?

```javascript
// Generate a correlation ID at entry point
const correlationId = req.headers['x-correlation-id'] || uuidv4();

// Pass it through all services
axios.post('http://payment-service/charge', data, {
  headers: { 'x-correlation-id': correlationId }
});

// Include in every log
logger.info('Processing payment', { 
  correlationId, // ← This is the key
  orderId 
});

// Now you can search all logs for this ID
// and see the entire request flow
```

## Distributed Tracing: Following the Request

You've got metrics telling you the checkout API is slow. Logs show no errors. But response times are 3 seconds and customers are angry.

Where's the time going?

Distributed tracing answers this by following a request across all services.

### How Tracing Works

1. **Span**: A single operation (like "call payment API")
2. **Trace**: Collection of spans forming the full request path
3. **Context propagation**: Passing trace ID between services

```
Request ──┬── Auth Service (15ms) ────────────────────────────────┐
          │                                                       │
          ├── User Service (45ms) ─── DB Query (38ms) ──────────┤
          │                                                       │
          ├── Payment Service (2800ms) ──┬── Validate (5ms) ────┤
          │                               │                      │
          │                               ├── Stripe API (2750ms) │ ← HERE'S THE PROBLEM
          │                               │                      │
          │                               └── Update DB (40ms) ──┤
          │                                                       │
          └── Email Service (120ms) ─────────────────────────────┘
          
Total: ~3000ms (most spent waiting on Stripe API)
```

### OpenTelemetry: The Standard

OpenTelemetry is the vendor-neutral standard for traces (and metrics/logs). Use it.

```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

// Set up tracing
const provider = new NodeTracerProvider();
provider.addSpanProcessor(
  new BatchSpanProcessor(
    new JaegerExporter({ endpoint: 'http://jaeger:14268/api/traces' })
  )
);
provider.register();

// In your code
const tracer = opentelemetry.trace.getTracer('checkout-service');

async function processCheckout(order) {
  const span = tracer.startSpan('process-checkout');
  
  span.setAttribute('order.id', order.id);
  span.setAttribute('order.amount', order.amount);
  
  try {
    await validateOrder(order, span); // Pass context
    await chargePayment(order, span);
    await sendConfirmation(order, span);
    span.setStatus({ code: SpanStatusCode.OK });
  } catch (err) {
    span.recordException(err);
    span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
    throw err;
  } finally {
    span.end();
  }
}

async function chargePayment(order, parentSpan) {
  // Create child span
  const span = tracer.startSpan('charge-payment', {
    parent: parentSpan // Links to parent
  });
  
  span.setAttribute('payment.provider', 'stripe');
  
  try {
    const result = await stripe.charges.create({ /* ... */ });
    span.setAttribute('payment.charge_id', result.id);
    return result;
  } finally {
    span.end();
  }
}
```

### Context Propagation Between Services

```javascript
// Service A: Inject trace context into HTTP headers
const { context, propagation } = require('@opentelemetry/api');

async function callServiceB(data) {
  const headers = {};
  
  // Inject trace context into headers
  propagation.inject(context.active(), headers);
  // Headers now contain: traceparent, tracestate
  
  return axios.post('http://service-b/api', data, { headers });
}

// Service B: Extract context from headers
app.use((req, res, next) => {
  const context = propagation.extract(context.active(), req.headers);
  // Now Service B's spans are linked to Service A's trace
  next();
});
```

## Common Anti-Patterns

### ❌ Logging Sensitive Data

```javascript
// NEVER DO THIS
logger.info('User login', { 
  email: user.email,
  password: req.body.password, // 🚨 NO
  creditCard: payment.cardNumber // 🚨 NO
});

// ✅ Redact sensitive fields
logger.info('User login', { 
  email: user.email.replace(/(.{2}).*@/, '$1***@'),
  cardLast4: payment.cardNumber.slice(-4) // Only last 4 digits
});
```

### ❌ High-Cardinality Labels on Metrics

```javascript
// ❌ WRONG: User ID has infinite unique values
httpRequestsTotal.inc({ 
  method: 'GET', 
  userId: req.user.id // ← This will explode your metrics storage
});

// ✅ CORRECT: Low-cardinality labels only
httpRequestsTotal.inc({ 
  method: 'GET', 
  route: '/api/users/:id', // Route pattern, not actual ID
  status: '200'
});
```

### ❌ Sampling Errors at 100%

You want to capture all errors, right? So you set error sampling to 100%. But what happens when your error rate spikes?

```javascript
// ❌ Problem: No sampling for errors
if (isError) {
  logger.error('Request failed', { ...data });
}

// In an outage with 10,000 errors/second, your logging system
// might collapse under the load.

// ✅ Better: Rate-limited error logging
const errorLimiter = new RateLimiter({ tokensPerInterval: 100, interval: 'second' });

if (isError && errorLimiter.tryRemoveTokens(1)) {
  logger.error('Request failed', { 
    ...data,
    errorCount: errorCounter.increment() // Track how many we're not logging
  });
}
```

## Choosing Your Stack

| Component | Options | Notes |
|-----------|---------|-------|
| Metrics | Prometheus + Grafana | Industry standard, self-hosted |
| | Datadog, New Relic | SaaS, expensive at scale |
| Logs | ELK Stack (Elasticsearch, Logstash, Kibana) | Powerful, resource-heavy |
| | Loki + Grafana | Lighter weight, PromQL for logs |
| Traces | Jaeger | Open source, works with OpenTelemetry |
| | Zipkin | Simpler, good for small setups |
| All-in-One | Datadog, New Relic, Honeycomb | Easier setup, vendor lock-in |

### Local Development Setup

```yaml
# docker-compose.yml for local observability
services:
  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml"]
  
  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
  
  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]
  
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports: 
      - "16686:16686" # UI
      - "14268:14268" # HTTP endpoint
```

## The Cost of Observability

Observability isn't free. Be intentional about what you track.

| Data Type | Cost Driver | Mitigation |
|-----------|-------------|------------|
| Metrics | High-cardinality labels | Only low-cardinality labels |
| Logs | Volume | Sample non-errors, structured logs |
| Traces | Sample rate | Sample at 1-10% for success, 100% for errors |

A good rule of thumb: observability infrastructure costs should be 5-15% of your application infrastructure. If it's higher, you're probably over-collecting.

## Quick Setup Checklist

**Metrics (start here):**
- [ ] Track RED metrics (Rate, Errors, Duration) for all HTTP endpoints
- [ ] Track USE metrics (Utilization, Saturation, Errors) for databases
- [ ] Set up alerts for error rate > 1%, p99 latency > threshold
- [ ] Create a Grafana dashboard with these panels

**Logs (next):**
- [ ] Use structured JSON logging
- [ ] Add correlation IDs to all requests
- [ ] Include source, timestamp, level in every log
- [ ] Set up log aggregation (Loki, ELK, or SaaS)

**Traces (when you need them):**
- [ ] Instrument entry points and external calls
- [ ] Propagate context between services
- [ ] Start with 10% sampling, increase for errors
- [ ] Visualize with Jaeger or similar

---

**Bottom line:** Start with metrics. Add structured logs with correlation IDs. Bring in distributed tracing when you have multiple services. You don't need everything on day one—but you'll need it all eventually.