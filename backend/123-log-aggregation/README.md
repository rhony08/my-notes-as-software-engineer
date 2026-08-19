# Log Aggregation: Stop Grepping 50 Servers

Your app has a bug. Not a fun bug — the kind where the error only happens on the third retry, in production, on one of the forty instances behind the load balancer. You SSH into a box, `tail -f /var/log/app.log`, and... nothing. The error was on a different instance. So you SSH into the next one. And the next one. Forty boxes later you've spent an hour and learned that the log line you need exists somewhere, probably, but you have no idea where.

This is the moment log aggregation stops being a "nice-to-have DevOps thing" and becomes the difference between a 10-minute fix and a lost afternoon. Structured logging (from yesterday's notes) gives you queryable fields — but fields only help if all your logs are in one place you can actually query. This article covers how log aggregation pipelines work, the agent vs library shipping debate, how to pick a backend (ELK, Loki, the SaaS options), and the failure modes that bite everyone.

## What Aggregation Actually Is

Aggregation is a pipeline with four stages:

```
apps → agents → transport → storage → query
 │       │          │          │        │
 └ emits ┘ collects ┘ ships    ┘ indexes ┘ search/alert
```

The goal: every log line from every service, container, and host ends up in one queryable store, with the least possible latency and the least possible data loss. Sounds simple. The details are where it gets spicy.

## The First Decision: How Logs Get From Your App to the Store

You've basically got three shipping models, and teams argue about them like sports teams.

### 1. Library shipping — the app logs directly to the aggregator

```javascript
// ❌ Every app opens its own connection to Elasticsearch and writes directly
const { Client } = require("@elastic/elasticsearch");
const es = new Client({ node: "https://logs.internal:9200" });

es.index({ index: "app-logs", document: { level: "error", msg: "boom" } });
```

This works at small scale and is a disaster at any real scale:

- Every app now needs credentials, network access, and TLS config for your log store.
- A log store outage now takes down your *application* (or you buffer in-process and leak memory).
- You've coupled your app to one specific vendor. Switch from ELK to Loki? Rewrite the logging layer.
- High-volume apps hammer the cluster with per-line HTTP calls.

**When it's actually fine:** small internal tools, cron jobs, serverless functions that can't run an agent. For everything else, prefer an agent.

### 2. Agent shipping — a sidecar daemon collects and forwards

The app writes plain lines to stdout or a file. An agent (Filebeat, Fluent Bit, Vector, Promtail) tails that output and ships it.

```yaml
# fluent-bit.conf — the app doesn't know or care where logs go
[INPUT]
    Name          tail
    Path          /var/log/app/*.log
    Tag           app.*

[OUTPUT]
    Name          http
    Match         app.*
    Host          loki.internal
    Port          3100
    URI           /loki/api/v1/push
    Format        json
```

This decouples everything: the app writes logs like a normal app, the agent handles batching, retries, backpressure, and credential management. Rotate the agent, change the backend, add a second destination — none of it touches application code.

**The cost:** one more moving part per host, and agent config drift across environments. That's a manageable price for not coupling your app to your log vendor.

### 3. Push vs pull

Most setups are **push** — agents shove logs at the collector. Loki's Promtail is the notable **pull** alternative: the collector scrapes logs from targets, similar to how Prometheus scrapes metrics. Pull centralizes auth (the collector holds credentials, not every agent) but requires targets to be reachable, which gets awkward with short-lived containers and serverless. Push is the pragmatic default; pull shines in locked-down environments where agents can't have egress.

## Choosing the Backend

This is where people get religious. Here's the honest comparison:

| | ELK / OpenSearch | Loki | Managed SaaS (Datadog, Grafana Cloud, Splunk) |
|---|---|---|---|
| Storage model | Full-text index (Lucene) | Labels + compressed chunks, indexed lightly | Whatever the vendor does |
| Strengths | Powerful queries, aggregations, any-field filtering | Cheap at scale, native Grafana integration, log→metric correlation | Zero ops, alerts + dashboards built in, one bill |
| Weaknesses | Expensive to run well (memory hungry), ops burden | Queries on non-label fields are slower (grep over chunks) | Per-GB pricing gets brutal, data leaves your infra |
| Best for | Deep log analysis, teams that want full control | High-volume infra logs, already on Grafana/Prometheus | Small teams that can't staff an ops rotation |

The short version: **Elasticsearch gives you the best query engine and the biggest bill (in RAM and ops); Loki gives you 80% of the value at a fraction of the cost if you already live in Grafana; SaaS is the "I have better things to do" option until the invoice scares you.**

My actual advice for a backend team: if you're starting from zero today, seriously consider Loki + Grafana — the log/metric correlation is huge, and the label-based model forces you to think about structure instead of just dumping everything into an index.

## What Breaks in Practice

Aggregation pipelines fail in predictable ways. Here's the list I wish someone had handed me.

### Log rotation

The app writes to `app.log`, and `logrotate` renames it to `app.log.1` every night. A naive agent that opened the file once and kept reading keeps tailing the *rotated* file — which stops growing — while the new file's logs silently vanish from the pipeline.

The fix is agents that handle rotation natively (Filebeat, Fluent Bit tail input, and Promtail all do), plus one config knob you should never forget:

```yaml
# fluent-bit.conf — without this, rotated files can be re-read or missed
[INPUT]
    Name          tail
    Path          /var/log/app/*.log
    # remember where we were across restarts
    DB            /var/log/fluent-bit.db
    # don't re-ship old content on restart
    Ignore_Older  24h
```

### Multi-line logs

Stack traces and pretty-printed JSON span multiple lines. Default line-based parsing will shred them into garbage entries, and your `error.stack` becomes a pile of unparseable fragments. Every serious agent has a multiline parser:

```yaml
[PARSER]
    Name        multiline_java
    Format      regex
    # match a new log line only when it starts with a timestamp
    Regex       /^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}/
```

```javascript
// ❌ In the app: log the stack as separate lines
console.error("Something broke:\n" + err.stack);

// ✅ In the app: single line, JSON-encoded, agent handles the rest
console.error(JSON.stringify({ event: "app.crash", stack: err.stack }));
```

### Clock skew

Your instances have clocks that disagree (especially in containers and spot instances). If your pipeline sorts by *ingestion time* instead of *log timestamp*, a single skewed host scrambles the ordering of everything around it, and your "what happened right before the crash" investigation shows events in the wrong order.

Rules that actually hold up:

- **Always log UTC.** Local time in logs is a debugging tax you pay forever.
- Include both `timestamp` (when it happened) and let the store keep ingestion time.
- Sort/filter by the log's own timestamp, not ingestion time.

### Cardinality explosion

This is the Loki-specific trap, but it applies to any label-based system: labels are cheap to query but expensive to store — every unique label *value* creates a new index series.

```yaml
# ❌ request_id as a label: thousands of unique series, cluster melts
labels:
  request_id: "{request_id}"

# ✅ request_id as a field inside the log line: still queryable, no series explosion
labels:
  service: "checkout"
  env: "prod"
```

Rule of thumb: labels for things with **few values** (service, env, instance, level). Fields for things with **many values** (request IDs, user IDs, paths).

### Volume and cost

Logs are the sneakiest line item in your cloud bill. A busy service easily emits gigabytes a day, and every GB costs you in storage, indexing, and query time. Practical mitigations:

- **Level-based filtering at the agent:** drop `DEBUG` in prod before it ships.
- **Retention tiers:** hot storage for 7 days, cold/archived for 90, delete or archive after. You rarely need to query 6-month-old debug logs.
- **Sampling for high-volume noisy logs** (access logs especially) — sample 1 in 10 and keep the rest in cheap object storage if you truly need it.
- **Don't index what you never query.** In Elasticsearch, mapping every field as `text` + `keyword` triples your index size.

### Dropped logs under load

Agents batch and retry, but when the collector is down for 10 minutes and you're emitting 10k lines/sec, buffers fill and logs get dropped. Plan for it:

- Give agents a **disk buffer**, not just memory (Fluent Bit's `storage.type filesystem`, Filebeat's spool).
- Know your agent's drop behavior: does it block, drop-newest, or drop-oldest? Blocking protects data but backs up your app's stdout — usually fine since stdout is async, but measure it.
- Alert on pipeline lag (`collector lag > 60s`), not just on collector being down.

### Shipping secrets to the aggregator

Your log store is now a high-value target: everyone's logs, all in one place. If an app logs a password once (it shouldn't, but it happens), it's now sitting in the aggregator with weaker access control than the app itself. Treat your log store as sensitive infrastructure: TLS in transit, auth on ingestion, least-privilege read access, and retention rules that match your compliance obligations.

## Self-hosted vs SaaS — the Real Trade-off

Self-hosting ELK looks cheaper on paper and gives you total control. The hidden costs: Elasticsearch is memory-hungry (a 3-node cluster with decent indexing eats 24–48GB RAM before you've stored anything), you own upgrades, disk management, and the 3am "the cluster is red" pager. SaaS shifts all of that to a vendor but adds per-GB pricing that grows with your success — a service that doubles in traffic doubles your log bill.

There's no universally right answer. The honest heuristic:

- Team of 1–5, no dedicated infra person → SaaS (or managed Elasticsearch/Loki).
- You already run Kubernetes and Grafana → self-hosted Loki; it's cheap enough that the ops burden is small.
- You need heavy full-text log analysis, custom parsing, or compliance data residency → self-hosted OpenSearch/Elasticsearch, budget for the RAM.

And whatever you pick: **your pipeline config is code.** Put the agent configs in your repo, version them, and test them in staging. The aggregation layer bit-rots quietly — nobody notices until the day you actually need those logs and they were never arriving.

## Takeaways

- Ship logs with an agent, not from the app — decouple your services from your log vendor and keep the app resilient to log-store outages.
- Backend choice is a trade-off: ELK for query power and ops pain, Loki for cost and Grafana integration, SaaS for zero-ops at a per-GB price.
- Handle rotation and multi-line logs in the agent config before they silently eat your data.
- Log UTC, sort by log timestamps, and keep label cardinality low — request IDs are fields, not labels.
- Plan for volume: drop DEBUG at the agent, tier your retention, and give agents disk buffers so a collector outage doesn't mean lost evidence.
- Treat the aggregator as sensitive infrastructure: TLS, auth, and least-privilege reads.
