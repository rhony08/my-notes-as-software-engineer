# SLA, SLO, SLI: The Numbers That Define Reliability

You're shipping a new feature. Exciting. But then the question comes from your ops team: "What's our target uptime for this service? What happens when we miss it?"

If you've ever stared blankly at those questions, you're not alone. Most developers know they _should_ care about reliability metrics, but SLA, SLO, and SLI blur together faster than acronym soup at a DevOps conference.

Let's fix that.

## The Two-Second Definitions

- **SLI (Service Level Indicator)** — the _measurement_. What you actually track (latency, error rate, uptime)
- **SLO (Service Level Objective)** — the _target_. The number you want your SLI to meet (99.9% uptime)
- **SLA (Service Level Agreement)** — the _contract_. What you promise your customers, with consequences if you miss

Think of it like a fitness goal:

| | Tech Equivalent | Real World Example |
|---|---|---|
| **SLI** | Your step count for today | Actual p99 latency: 250ms |
| **SLO** | "10,000 steps per day" | "p99 < 500ms" |
| **SLA** | The bet with your friend if you skip leg day | "If p99 exceeds 500ms for 1 hour, customer gets 10% credit" |

## SLIs: What You Actually Measure

Before you can promise anything, you need to track the right things. Common SLIs for a backend service:

**Availability / Uptime**
- How often is the service returning valid responses?
- Formula: `successful requests / total requests`

**Latency**
- How fast are we responding?
- Typically measured at p50, p95, and p99 percentiles

**Error Rate**
- What fraction of requests are failing?
- Formula: `HTTP 5xx / total requests`

**Throughput**
- How much can we handle?
- Requests per second or concurrent connections

### ⚠️ Pick Your SLIs Carefully

More isn't better. Google's SRE book recommends **4-5 SLIs per service**. Track too many and you split your attention. Track too few and you miss blind spots.

```
// ❌ Tracking 15 metrics but defining SLOs for none of them
{
  "metrics": ["cpu", "memory", "disk", "latency_p50", "latency_p95", 
              "latency_p99", "error_rate", "throughput", "gc_pause",
              "connection_pool", "db_query_time", "cache_hit_rate",
              "queue_depth", "thread_count", "heap_usage"],
  "slo": null // 🚩 No actual targets
}

// ✅ 4 tight SLIs with defined measurement windows
{
  "slis": {
    "uptime": { "measurement": "successful_requests / total_requests", "window": "28d" },
    "latency_p99": { "measurement": "p99 response time in ms", "window": "1h" },
    "error_rate": { "measurement": "5xx_rate", "window": "10m" },
    "throughput": { "measurement": "requests_per_second", "window": "5m" }
  }
}
```

## SLOs: Setting Realistic Targets

SLOs are where the real work happens. You've got your SLI data—now what number do you shoot for?

### The 99.9% Trap

Everyone wants 99.999% ("five nines"). But here's the thing nobody talks about: **chasing perfect uptime is expensive and often unnecessary**.

| SLO | Allowed Downtime/Month | Difficulty | Infrastructure Cost |
|---|---|---|---|
| 99% (two nines) | ~7.2 hours | Easy | Low |
| 99.9% (three nines) | ~43 minutes | Moderate | Medium |
| 99.99% (four nines) | ~4.3 minutes | Hard | High |
| 99.999% (five nines) | ~26 seconds | Insane | Very High |

Does your internal API really need five nines? Probably not. An internal reporting dashboard at 99.9% is fine. A payment processing system at 99.99% makes sense.

> **The rule of thumb:** Your SLO should be worse than your best observed SLI. If your system runs at 99.99% naturally, don't set the SLO to 99.99%—that leaves no room for degradation. Set it at 99.9% and give yourself breathing room.

### Error Budgets

This is the killer concept from Google SRE: **once you set an SLO, the difference between 100% and your SLO is your error budget**.

If your SLO is 99.9% uptime, you have a 0.1% error budget. That's 43 minutes of downtime per month. When things go wrong, you spend from that budget.

The magic of error budgets: **they tell you when to stop shipping and focus on reliability**. If you've used up your error budget for the month, you should be in "fix things" mode, not "ship more features" mode.

```python
# Error budget tracking (simplified)
slo_uptime = 0.999  # 99.9%
total_minutes_per_month = 30 * 24 * 60  # 43,200
error_budget_minutes = total_minutes_per_month * (1 - slo_uptime)
# = 43.2 minutes

minutes_down_this_month = 12  # tracking via monitoring
remaining_budget = error_budget_minutes - minutes_down_this_month
# = 31.2 minutes remaining
```

## SLAs: The Legal Bit

SLAs are what you put in a contract. Promises you make to customers (or to other teams). Missing an SLO isn't ideal—but it's a technical problem. Missing an SLA means **real consequences**: credits, refunds, or legal liability.

**Do not set your SLA equal to your SLO.** You need a buffer.

```
SLA: 99.95% (promise to customers)
  ↑ buffer of 0.05%
SLO: 99.9% (internal target)
  ↑ buffer for tracking
SLI: Actual measured values
```

Why? If your SLO == SLA, and your SLO measurement isn't perfect (spoiler: it never is), you'll accidentally breach your contract.

## Real-World Example: An API Service

Let's make this concrete. You're running a payment API.

```
SLIs:
  - Availability: requests returning 2xx / total requests
  - Latency (p99): milliseconds for the slowest 1% of requests
  - Correctness: transactions settled without errors

SLOs (quarterly, rolling 28-day window):
  - 99.9% availability (< 43 min downtime/month)
  - p99 latency < 500ms
  - 99.99% correctness (< 1 error per 10k transactions)

SLA (customer contract):
  - 99.5% availability
  - If breached: 5% credit for affected hours
  - Max credit per month: 25% of monthly fee
```

Notice the SLA (99.5%) is lower than the SLO (99.9%). That's the buffer zone. If the team blows the SLO one month, there's room before customers get credits.

## Common Pitfalls

**Measuring on the wrong time window.** A 30-day rolling window can mask a 30-minute outage. Use shorter windows for alerting, longer windows for reporting.

**Comparing apples to oranges.** Your SLI tracks p99 latency from the server side. Your customer measures p95 from their browser. Those won't match. Define where and how you measure upfront.

**Perfectionism.** It's better to start with a rough SLO and refine it than to wait for perfect SLI instrumentation. Ship a version 1, iterate.

## Takeaways

- **SLI = what you measure**, SLO = what you target, SLA = what you promise
- **Error budgets turn reliability into a ship/release decision** — you spend from the budget, and when it's gone, you stabilize before shipping
- **Your SLA should be looser than your SLO** — give yourself a buffer zone between "internal target" and "customer contract"
- **Not everything needs five nines** — match your reliability targets to the service's actual importance
- **Start simple** — 3-4 SLIs per service, a realistic SLO, no SLA unless you're charging customers
