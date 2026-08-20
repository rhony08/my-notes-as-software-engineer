# Alerting: What to Alert On (and What to Silence)

Your pager goes off at 3:47 AM. You groggily open the alert: `CPU usage > 80% on api-server-3`. You check Grafana, see a spike that lasted four minutes, and go back to bed. This happens three more times that week.

That's not alerting. That's noise with a pager attached.

Bad alerts have a sneaky cost: they train you to ignore the tool that's supposed to wake you up when things actually break. This article is about what deserves a page, what deserves a dashboard, and what deserves to be deleted.

## The Real Problem Isn't Missing Alerts — It's Too Many

Most teams start with the opposite fear. "What if something breaks and we don't know?" So they add alerts for everything: high CPU, low disk, slow queries, failed logins, 5xx responses, JVM heap usage... and now they've built a system that fires constantly.

Here's the uncomfortable truth: **every alert you add is a liability**. Each one costs attention, context-switching, and trust. When 90% of pages turn out to be nothing, your brain starts treating 100% of pages as nothing. That's how real outages slip through — not because the alert didn't fire, but because nobody believed it.

```
❌ Alert on: CPU > 80% for 5 minutes
✅ Alert on: Error rate > 5% for 10 minutes (SLO breach)

❌ Alert on: Disk usage > 70%
✅ Alert on: Disk will fill up in the next 4 hours (projected)
```

See the difference? The left column is about *machines*. The right column is about *users*. Your users don't care about CPU. They care about slow or failing requests.

## Alert on Symptoms, Not Causes

This is the single biggest mental shift in alerting, and it comes straight from Google's SRE book.

- **Symptoms** = what the user experiences (slow responses, errors, timeouts)
- **Causes** = why it's happening (high CPU, connection pool exhausted, a bad deploy)

You want to alert on symptoms. Causes are for dashboards and debugging.

Why? Because causes are noisy and ambiguous. High CPU *might* be a problem — or it might be a batch job that runs every night and is supposed to burn CPU. But a spike in user-facing errors is *always* a problem. You can't argue with it.

The practical flow:

1. Symptom alert fires → you get paged
2. You open the dashboard → you see the cause (CPU, DB latency, whatever)
3. You fix the cause → symptom clears

If you only alert on causes, you'll page people for problems that never reach users — and miss the ones that do.

## The Four Golden Signals

If you're not sure what to alert on, start here. Google's four golden signals cover most of what matters for user-facing services:

| Signal | What it measures | Alert example |
|--------|-----------------|---------------|
| **Latency** | How long requests take | p95 latency > 1s for 10 min |
| **Traffic** | How much demand you're getting | Request rate drops to near zero (or spikes past capacity) |
| **Errors** | Requests that fail | Error rate > 5% for 10 min |
| **Saturation** | How "full" your service is | Queue depth growing faster than it drains |

Traffic is the sneaky one. A sudden *drop* in traffic often means something upstream broke and nobody's getting to you — a silent failure that error rates won't catch because there are no requests to fail.

## SLO-Based Alerting: Let the Numbers Decide

Instead of arguing about thresholds in meetings, let your SLO drive your alerts. The pattern:

1. Define an SLO: "99.9% of requests succeed in under 500ms"
2. Track your error budget: how much you can fail this month and still hit the SLO
3. Alert when you're *burning through* the budget faster than planned

The clever part: **burn rate**. If your SLO is 99.9% and your error rate is 0.1%, you're exactly on budget. But if errors run at 1% for an hour, that's a burn rate of 10x — you're consuming 10 hours of error budget per hour. That deserves a page.

This kills the threshold-guessing game. Instead of "is 5% errors bad?" you ask "how fast are we eating our error budget?" The second question has a concrete answer.

```
// Prometheus-style alert: 99.9% SLO, alert when burning 14.4x budget
// (1% error rate for 1 hour = 10x burn... this one pages fast)
- alert: HighErrorBurnRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[1h])) 
      / sum(rate(http_requests_total[1h])) > 0.014
  for: 15m
  labels:
    severity: page
  annotations:
    summary: "Error budget burning fast ({{ $value | humanizePercentage }})"
```

## Severity Levels That Mean Something

A common failure mode: everything is `severity: high`, so nothing is high. Pick levels that map to *actions*, not feelings:

| Level | What it means | Who gets it | Response |
|-------|---------------|-------------|----------|
| **Page** | Users are affected *right now* | On-call, 24/7 | Wake up, investigate |
| **Ticket** | Degraded but not critical | Team during work hours | Fix within the day |
| **Dashboard** | Worth watching, not acting | Nobody — it's just a chart | Look at it in the morning |

Rule of thumb: if an alert fires and the correct response is "look at it tomorrow morning," it's not a page — it's a ticket. If it doesn't even need tomorrow, it's a dashboard metric.

## Every Alert Needs a Runbook

An alert without a runbook is a riddle. At 3 AM, you don't want to reverse-engineer your own system. The alert itself should tell you what to do:

```
❌ summary: "High error rate detected on checkout service"
✅ summary: "Checkout 5xx rate > 5% for 10 min"
   runbook: https://wiki/runbooks/checkout-high-errors
   - 1. Check deploy status in the last hour (most common cause)
   - 2. Check payment provider status page
   - 3. Check DB connection pool saturation
   - 4. If all clear, roll back the last deploy
```

If you can't write the runbook, you don't understand the failure mode well enough to alert on it. That's a signal to investigate first, alert later.

## The Alert Fatigue Math

Google's SRE book has a brutal rule: **if an alert fires and the recipient can't recall the last time it was useful, it gets deleted.** More precisely, they found that a page is only worth it if there's a meaningful chance something's actually wrong — otherwise you're just generating noise.

Think about it in expected value. If an alert fires 20 times and was actionable once, each page costs ~15 minutes of interrupted sleep and focus. That's 5 hours of human attention for one real incident. The fix isn't "alert louder" — it's either raise the threshold, add a `for:` duration, or delete the alert and track it on a dashboard instead.

```
❌ Keep 40 alerts firing weekly, mostly false positives
✅ Keep 8 alerts, each with a documented runbook and a real incident behind it
```

## When You Should *Not* Alert

Some things feel alert-worthy but aren't:

- **Ephemeral spikes** — a 2-minute CPU blip during a deploy. Add `for: 10m` instead.
- **Known maintenance windows** — silence alerts during deploys, don't make people mentally subtract them.
- **Single-instance hiccups** — if you have 5 replicas and one restarts, that's Tuesday. Alert on the *fleet* error rate, not the individual pod.
- **Things the dashboard already shows** — if nobody acts on it, it's a chart, not an alert.

## The Takeaway

- **Alert on symptoms (user-facing errors, latency, saturation), not causes (CPU, disk, memory)** — causes go on dashboards.
- **Let SLOs and error budgets decide thresholds** — burn-rate alerts beat guessed thresholds.
- **Three severity levels max: page, ticket, dashboard** — anything less than a page during work hours is a ticket, and anything passive is a chart.
- **Every page needs a runbook attached to the alert itself** — you shouldn't have to think at 3 AM.
- **Prune ruthlessly** — if an alert hasn't caught a real problem in months, delete it or demote it. Alert fatigue is how real outages get ignored.
- **Start with the four golden signals** — latency, traffic, errors, saturation. That's 90% of what you need.
