# Metrics Collection: Pull, Push, and the Cardinality Trap

Your service is running. Uptime is green. Then one Tuesday at 2pm, customers start complaining about slow responses — but your dashboard shows a flat line. Not because nothing happened, but because you never collected the metric that would've shown it. You were monitoring *uptime*, not *health*.

This is the classic failure: metrics only help if you collect the right ones, at the right granularity, in a way your monitoring system can actually handle. Get it wrong and you either fly blind or drown in noise. We'll look at the four metric types you'll actually use, the pull vs push debate (it's not religious — it's practical), and the cardinality trap that quietly OOMs your Prometheus instance.

## The Four Metric Types You'll Actually Use

Every metrics system boils down to a handful of primitives. Learn these and you can read any dashboard on earth:

| Type | What it is | Example | Good for |
|------|-----------|---------|----------|
| **Counter** | Only goes up (or resets) | Total requests, total errors | Rates: req/s, error rate |
| **Gauge** | Goes up *and* down | Current queue depth, CPU usage | Levels you can sample |
| **Histogram** | Counts observations into buckets | Request latency | Percentiles (p50, p99) |
| **Summary** | Client-side quantiles + count | Latency percentiles | When you can't aggregate server-side |

The counter/gauge distinction trips people up constantly. A counter is useless as a point-in-time value — "42,137 total requests" tells you nothing. What you want is the *rate of change*, which your monitoring system computes for you:

```
# ❌ Treating a counter as a gauge
"requests_total is 42137" → meaningless without a time range

# ✅ Asking for a rate over time
rate(requests_total[5m]) → "~120 requests/sec right now"
```

Same raw metric, completely different question. If you ever find yourself plotting a counter directly, you're probably looking at the wrong chart.

## Pull vs Push: Pick by Infrastructure, Not Vibes

There are two dominant collection models, and people argue about them like sports teams. The real answer is: it depends on what you're running.

**Pull (Prometheus-style).** The monitoring system scrapes your app on an interval (`/metrics` endpoint, usually every 15-30s). Your app doesn't care who's watching.

```
# prometheus.yml — the scraper does the work
scrape_configs:
  - job_name: "api"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["api-server:8080"]
```

**Push (StatsD/Graphite-style).** Your app sends metrics to a collector (often over UDP) and forgets about them. The collector aggregates and forwards.

```
// StatsD — fire and forget, UDP means it's cheap
statsd.increment('orders.created');
statsd.timing('checkout.duration', 320); // ms
```

| | Pull (Prometheus) | Push (StatsD) |
|---|---|---|
| Discovery | Scraper must know targets (or use service discovery) | App just needs collector address |
| Short-lived jobs | Painful — job's gone before scrape | Natural fit (batch jobs, lambdas) |
| Firewalls | Needs inbound access to app | Outbound UDP only |
| Availability | Scraper is a SPOF unless HA'd | Collector is a SPOF unless HA'd |
| Debugging | `curl /metrics` anytime | Need a sidecar to inspect |

My rule of thumb: long-lived services that you run yourself → **pull**, because you get the live `/metrics` endpoint for free during debugging. Ephemeral stuff — cron jobs, serverless, batch workers → **push**, because nobody's around to scrape them. Prometheus even has a **Pushgateway** for exactly that case, though it's often misused to push long-lived service metrics, which is a smell.

## The Cardinality Trap

Here's the metric that will silently kill your monitoring stack:

```
http_requests_total{method="GET", path="/users/482913", status="200"}
```

See the problem? `path` is a label, and that label contains a user ID. Every unique user ID creates a *new time series*. Ten thousand users → ten thousand series just for this one metric. Scale that across all your metrics and your Prometheus memory usage explodes while your queries crawl.

**Cardinality** = number of unique label combinations. High cardinality is the #1 way to break a metrics system.

```
# ❌ User-specific values as labels — thousands of series
http_requests_total{user_id="482913", endpoint="/api/orders"}

# ✅ Coarse labels — bounded series count
http_requests_total{endpoint="/api/orders", status="200"}
```

The fix isn't "no labels" — labels are what make metrics useful (grouping by endpoint, status, instance is the whole point). It's *bounded* labels. Endpoints, status codes, regions, instances: all bounded. User IDs, email addresses, IPs, request IDs: all unbounded. Keep unbounded data in *values* (or in logs/traces), never in labels.

Also: be careful with histograms here. Each bucket is a series too, so a histogram with 10 buckets × 3 labels × 4 instances is 120 series for one metric. Fine. But add a high-cardinality label and it's over.

## RED and USE: Two Checklists That Cover Everything

If you don't know *what* to collect, these two frameworks give you a starting point for 90% of services:

**RED** (for request-driven services — APIs, backends):
- **R**ate — requests per second
- **E**rrors — requests failing (as a rate, not just count)
- **D**uration — latency distribution (p50, p95, p99)

**USE** (for resources — databases, queues, servers):
- **U**tilization — % busy (CPU, connections, disk)
- **S**aturation — how much work is *waiting* (queue depth, runnable threads)
- **E**rrors — device errors, dropped connections

The beautiful part: RED tells you the *symptom* ("API is slow"), USE tells you the *cause* ("DB connections saturated"). You need both layers or you'll keep paging yourself with "it's slow" and no idea why.

## Instrumenting an HTTP Server, Properly

Concrete example with the Prometheus Node.js client. The pattern applies to any language:

```js
// metrics.js — the wiring
const client = require('prom-client');

// Default registry collects process metrics (CPU, memory, event loop lag)
client.collectDefaultMetrics({ prefix: 'app_' });

// Counter: total requests. Rate comes later via rate() in queries
const httpRequests = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'], // all bounded — safe labels
});

// Histogram: latency buckets in seconds. Pick buckets around your SLO,
// not a default — if p99 is 300ms, buckets up to 10s waste precision
const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'route'],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

module.exports = { httpRequests, httpDuration };
```

```js
// server.js — instrument at the choke point (middleware), not scattered
const { httpRequests, httpDuration } = require('./metrics');

app.use((req, res, next) => {
  const end = httpDuration.startTimer(); // records duration on end()
  res.on('finish', () => {
    // route from req.route or a matched pattern — NEVER req.url raw,
    // or /users/482913 becomes a label value (cardinality trap!)
    httpRequests.inc({ method: req.method, route: req.route?.path ?? 'unknown', status: res.statusCode });
    end({ method: req.method, route: req.route?.path ?? 'unknown' });
  });
  next();
});

// Expose for scraping
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

The `req.route.path` vs `req.url` detail is the whole article in miniature: a pattern like `/api/orders/:id` is a bounded label, the raw URL is a cardinality bomb.

## Aggregation and the Staleness Gotcha

Once metrics leave your app, someone has to aggregate them. Two common patterns:

- **Agent-based**: a local agent (node_exporter, statsd-exporter) scrapes or receives, then forwards with labels attached (instance, region). Keeps app code clean.
- **OpenTelemetry pipeline**: the OTel SDK collects, an OTel Collector batches/processes/forwards to Prometheus, Jaeger, or whatever you run. More moving parts, but one instrumentation path for metrics *and* traces.

Whatever you pick, remember **staleness**: pull-based systems mark a series stale if no scrape succeeds (Prometheus: ~5 minutes). So if your app crashes, the metric doesn't freeze — it *disappears* from queries. That's a feature (you can alert on `up == 0`), but it surprises people who expect "last known value" behavior from push systems.

Trade-off worth naming: scrape interval vs storage cost. 15s scrapes × 10k series × 15 days retention adds up fast. You can downsample or extend intervals for low-signal metrics — but latency percentiles are *not* low-signal, so don't starve them.

## Actionable Takeaways

- Collect **counters for rates, gauges for levels, histograms for latency** — and never plot a counter without a rate.
- Choose **pull** for long-lived services (free `/metrics` endpoint for debugging), **push** for ephemeral jobs.
- Keep labels **bounded**: route patterns, status codes, instances. Never user IDs, emails, or raw URLs.
- Start with **RED for services, USE for resources** — symptom layer plus cause layer.
- Instrument at **one choke point** (middleware), not scattered through business logic.
- Alert on `up == 0` and error *rates*, not raw counts — your pager will thank you.
