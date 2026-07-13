# Metrics That Matter for Availability

You've built a distributed system. It runs across multiple regions, load balancers spread traffic evenly, databases replicate without a hitch. But then someone asks: "How available is it, really?"

And you freeze.

Because you realize you're staring at dashboards full of numbers — CPU graphs, memory curves, request counts — but none of them actually answer the question. What does "available" look like in numbers? And more importantly, what's *worth* tracking before your pager goes off at 3 AM?

Let's separate signal from noise.

---

## The Trinity: SLI, SLO, SLA

Before we talk about specific metrics, you need to understand how they fit together. These three form the backbone of any availability measurement:

| Term | What It Is | Example |
|------|-----------|---------|
| **SLI** (Service Level Indicator) | A raw measurement | "99.95% of requests complete in under 500ms" |
| **SLO** (Service Level Objective) | A target you aim for | "99.9% of requests should be successful" |
| **SLA** (Service Level Agreement) | A contractual commitment | "If availability drops below 99.9%, here's your credit" |

Think of it as: **measure (SLI) → set a target (SLO) → commit to it (SLA)**.

You can't have meaningful SLOs without good SLIs. And you shouldn't write SLAs before you understand your SLOs. A lot of teams skip straight to "we need 99.99% availability" without knowing whether they can even measure uptime properly.

---

## The Metrics That Actually Matter

### 1. Request Success Rate (The Obvious One)

```yaml
# SLI definition: percentage of requests returning 2xx/4xx vs 5xx
slis:
  request_success_rate:
    good: "status_code < 500"
    bad: "status_code >= 500"
    target: "99.9% over rolling 1h window"
```

This is the most basic availability metric. If you track nothing else, track this.

```python
# ❌ Treating all errors equally
error_rate = failed_requests / total_requests  # 4xx + 5xx together

# ✅ Separate client from server errors
server_error_rate = server_errors / total_requests  # 5xx only
client_error_rate = client_errors / total_requests  # 4xx only
```

Why? A 429 (rate limited) and a 503 (service unavailable) tell very different stories. Grouping them hides both.

**Watch out for:** Successful requests that are actually empty results. A 200 with `null` body isn't really success.

---

### 2. Latency at the Edge (P50, P95, P99)

Averages lie. If your average response time is 200ms, that could mean:

- Every request takes ~200ms ✅
- 90% of requests take 50ms, 10% take 1.5 seconds ❌

That's why percentiles matter:

```
// ❌ Average hides problems
avg_latency_ms: 210

// ✅ Percentiles tell the real story
p50: 45ms   # Half your users are happy
p95: 380ms  # Your 95th percentile is still ok
p99: 2800ms # Something is very wrong for 1% of users
```

**Why p99 matters:** In a system handling 10,000 requests/second, a p99 of 2.8s means 100 users are waiting almost 3 seconds every second. That's a lot of unhappy people.

### 3. Error Budget (The Practical One)

Error budget is what makes SLOs actionable. Here's the idea:

- Your SLO is 99.9% uptime over 30 days
- 99.9% of 30 days = 43 minutes of allowed downtime
- That's your **error budget** — the total acceptable failure time

```
Error Budget = (1 - SLO) × Time Period

99.9% over 30 days = 0.001 × 30 × 24 × 60 = 43.2 minutes
99.99% over 30 days = 0.0001 × 30 × 24 × 60 = 4.3 minutes
```

When your error budget is exhausted:

- ❌ No risky deployments until it recovers
- ✅ Focus on stabilization, not new features
- 📊 Track: `remaining_budget = budget - consumed`

Google's SRE teams treat error budget like a financial budget — once it's spent, you stop spending until the next cycle.

---

### 4. Time-Based Metrics: Uptime vs Availability

These get confused constantly. Let's be clear:

- **Uptime:** System is *running* (the process hasn't crashed)
- **Availability:** System is *serving traffic correctly*

```
Uptime = (total_time - downtime) / total_time × 100
Availability = successful_requests / total_requests × 100
```

You can have 100% uptime and 0% availability — if your service runs but returns errors for every request. That's why availability is the real metric.

| Scenario | Uptime | Availability |
|----------|--------|-------------|
| Server running, responding fine | 100% | 100% |
| Server running, all requests return 503 | 100% | 0% |
| Server crashed twice for 5 min each | 99.98% | 99.5% |

---

### 5. The Four Golden Signals

Google SRE popularized these four metrics as the minimum set for any service:

**1. Latency** — Time to serve a request
**2. Traffic** — How many requests are coming in
**3. Errors** — Rate of failed requests
**4. Saturation** — How close to capacity you are

Most teams track the first three. The one that's consistently forgotten is **saturation**.

```
# Tracking saturation for a database connection pool
pool_saturation_pct = (active_connections / max_connections) × 100

if pool_saturation_pct > 80:
  print(f"🚨 Connection pool running hot: {pool_saturation_pct}%")
  # Alert: you're about to hit connection limits
```

Saturation is your early warning system. By the time latency spikes or errors appear, you're already in trouble. Saturation tells you *you're about to run out* before you do.

---

## What NOT to Track (The Noise)

More dashboards ≠ better visibility. Here's what to stop wasting space on:

- ❌ **CPU utilization** (unless you're CPU-bound — most services aren't)
- ❌ **Memory usage** (unless you have a leak or GC issues)
- ❌ **Disk I/O** (for most stateless services, this is irrelevant)
- ❌ **99.999th percentile** (too much noise from random network jitter)

These aren't useless, but they're infrastructure health metrics, not availability metrics. If someone asks "is the service available?", you shouldn't need to check CPU graphs to answer.

---

## Putting It Together: A Minimal Dashboard

Here's what a focused availability dashboard looks like:

```
┌─────────────────────────────────────────────┐
│  AVAILABILITY OVERVIEW                     │
│                                             │
│  Success Rate:    99.92%  ✅                │
│  Error Budget:    32/43 min remaining       │
│  Current Burn:    ~2 min/hour              │
│                                             │
│  P50: 45ms  │  P95: 210ms  │  P99: 890ms  │
│                                             │
│  Request Rate: 12,340 req/s  ↑ 3.2%        │
│  Pool Saturation: 67%                       │
└─────────────────────────────────────────────┘
```

That's it. Six numbers tell you everything you need to know about whether your service is healthy.

---

## The 3 Things That Matter Most

1. **Track success rate and p95/p99 latency** — everything else is supporting context
2. **Set an error budget** and treat it like real money — once it's spent, stop deploying
3. **Watch saturation, not CPU** — it'll tell you you're about to have a problem before you actually have one

The rest is noise. If your success rate is 99.9%+, your p99 is under 1 second, and your error budget hasn't burned through, your service is available. Everything else is just looking busy.
