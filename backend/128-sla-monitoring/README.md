# SLA Monitoring: Proving You Keep Your Promises

Your contract says 99.9% uptime. Your dashboard says everything's green. Your customer's ticket says "your API was down for 40 minutes on Tuesday." One of these is a lie, and it's probably your dashboard.

This is the classic gap: you *have* an SLA, but you don't actually monitor against it. You track uptime as a percentage, sure — but that's not the same as tracking whether you met the commitment. An SLA isn't a number you print on a marketing page. It's a promise with a penalty attached. If you're not measuring against it continuously, you'll find out you missed it the same way your customer did: after the fact, in an angry email.

So let's talk about what it actually takes to monitor and report against an SLA: defining the right metrics, measuring availability in a way that matches the contract, and building reports that don't hide the truth.

## First, Decide What You're Actually Promising

You can't monitor an SLA you haven't defined precisely. And "99.9% uptime" is *not* precise. It raises more questions than it answers:

- Uptime of *what*? The API gateway? The database? The whole stack?
- Over what window? Calendar month? Rolling 30 days?
- What counts as "down"? Any 5xx? Timeouts? Responses slower than 2 seconds?
- Does planned maintenance count against it?

Every one of those ambiguities is a future argument. The moment your SLA and your monitoring disagree, the contract wins — and the contract is whatever the customer's lawyer can twist it into.

```
# ❌ Vague promise
"We guarantee 99.9% availability."

# ✅ Measurable promise
"The API will return a successful response within 2 seconds
for 99.9% of requests in any calendar month. Excluded:
scheduled maintenance windows, announced at least 48h in advance."
```

This is why SLOs (service level objectives) exist as a layer under the SLA. The SLA is the legal commitment; the SLO is the engineering target you actually monitor. Most teams set the SLO *stricter* than the SLA — say, 99.95% SLO against a 99.9% SLA — so you have a buffer to burn before you're in breach. The SLO is your early-warning system. The SLA is the cliff you don't want to fall off.

## Measure Availability Like a Customer Would

Here's the trap: a lot of uptime monitoring measures *infrastructure*, not *service*. Your host is up, your process is running, your load balancer reports healthy — and yet every request is timing out because the connection pool is exhausted.

Availability should be measured from the user's perspective, which means request-based, not host-based. The standard formula:

```
Availability % = (Successful requests / Total requests) × 100
```

Where "successful" means exactly what your SLA says — typically a 2xx response under the latency threshold. Everything else — 5xx, timeouts, connection failures — counts against you. This is a fundamentally different number from "host is up," and it's the only one your customer cares about.

```
# ❌ Infrastructure uptime
"Server has been up for 30 days" → 100% availability, 
but the app was returning 503s all afternoon.

# ✅ Request-based availability
Total requests: 1,204,332
Successful (< 2s, 2xx): 1,203,128
Availability: 99.90% — right at the line. Need to dig in.
```

The tricky part is where you measure from. A synthetic probe from outside your network (like a health check hitting a public endpoint every minute) catches outages your internal monitors miss — DNS issues, CDN problems, region-wide networking failures. Real user monitoring (RUM) captures what actual customers experience. Best answer: both. Synthetic for consistent, always-on signal; RUM for ground truth.

## The Error Budget Is Your Real Dashboard

Once you have request-based availability, convert it into an **error budget** — the amount of failure you're allowed before you miss the SLO. This reframes everything from "is it down?" to "how much runway do we have left?"

For a 99.9% SLO over a month, your error budget is:

```
Total requests in a month:  10,000,000
Allowed failures (0.1%):    10,000
Used so far (this month):   6,340
Remaining budget:           3,660 requests
```

This is powerful because it makes SLOs *actionable*. Teams that only track uptime react to outages. Teams with error budgets make decisions:

- Can we ship this risky deploy? Check the budget. If we're at 90% used, no — hold it.
- Should we roll back this new feature? The budget says we're burning through it, so yes.
- Do we need to declare a freeze on launches? The budget decides.

This is the Google SRE model, and the reason it works is simple: it converts an abstract percentage into a concrete number everyone understands. "We have 3,660 requests of failure left this month" is a sentence a product manager can act on. "Our availability is 99.94%" is not.

| Approach | What it tells you | The problem |
|----------|-------------------|-------------|
| Uptime % | Host was running | Misses app-level failures entirely |
| Request availability | Service worked for users | Better, but no sense of "how much left" |
| Error budget | Remaining allowed failures | Actionable — drives deploy decisions |

## Reporting: The Monthly Reckoning

If you're not reporting against your SLA, the monitoring is theater. But "reporting" doesn't mean a PDF that gets buried in a drive. It means a repeatable process where you can answer, at any point: *are we meeting the commitment?*

The report itself doesn't need to be fancy. It needs to be honest. That's harder than it sounds, because the honest version exposes things teams like to hide — the incident in week 2, the SLO breach nobody acknowledged, the slow degradation that never triggered an alert.

A useful SLA report includes:

- **The headline number**: availability for the period, vs. the target
- **Trend**: last 6 months, so a slow slide is visible
- **The breaches**: every period where you missed, with a one-line cause
- **Error budget status**: current burn rate and projected exhaustion date
- **Exclusions used**: maintenance windows claimed, so nobody argues later

```
# ❌ Cherry-picked reporting
"99.95% uptime this quarter!" — computed only from the 
weeks that went well, maintenance window conveniently "forgotten".

# ✅ Full picture
July availability: 99.87% (target 99.9%) — MISSED
  Cause: 2h connection pool exhaustion incident on Jul 14
  Budget impact: 1,120 requests over allowance
  Action: pool sizing review + load test scheduled
```

Automate this. A monthly report generated by hand will be generated late, then forgotten, then fudged. Wire the numbers into a dashboard that computes availability, error budget burn, and projected exhaustion continuously — then the monthly report is just a snapshot of something that's always on.

## Where Most Teams Go Wrong

Three mistakes I see constantly:

**1. Monitoring the SLA you wish you had, not the one you signed.** If the contract excludes scheduled maintenance but your monitor counts it as downtime, you'll think you're failing when you're not. If the contract includes a latency clause but your monitor only tracks status codes, you'll think you're passing when you're not. Align the monitor to the contract, line by line.

**2. No burn-rate visibility.** Availability that's drifting down toward the SLO line is the most dangerous state — nothing's technically broken, so nobody reacts. An error budget with a projected exhaustion date turns "slowly getting worse" into "we will miss our SLA on September 14 if nothing changes." That's a sentence that gets meetings scheduled.

**3. Treating the SLA as engineering's problem.** The SLA is a business commitment. Finance, sales, and product all have opinions about it — usually when it's breached and a customer wants a credit. Get them involved *before* that: agree on what counts as downtime, what the exclusions are, and what happens when you miss. The monitoring is the easy part. The agreement is where the real work lives.

## The Tooling Question

You don't need a dedicated "SLA monitoring" product. What you need is honest data flowing into something that can compute percentages and trends:

- **Metrics**: Prometheus (or your cloud provider's equivalent), with a `requests_total{status, route}` counter and a latency histogram. Everything else builds on these two.
- **Availability computation**: a query that derives success ratio per window. PromQL, Grafana, or a small scheduled job — whatever's already in your stack.
- **Synthetic probes**: a cron hitting your public endpoint from outside. Free and catches what internal checks miss.
- **Alerting**: alert on *burn rate*, not raw availability. "We're consuming budget 3x faster than allowed over the last hour" is actionable. "Availability dropped below 99.9%" is already too late — you only find out after the fact.

The honest truth: SLA monitoring is 20% tooling and 80% discipline. The tools are commodity. The discipline — defining the commitment precisely, measuring it like a customer would, and reporting it without spin — is the part that actually keeps you out of breach.

---

## Takeaways

- Define the SLA precisely before you monitor anything: what counts as downtime, over what window, with what exclusions. Ambiguity is a future argument.
- Measure availability request-based (successful requests / total requests), not host-based. Your customer doesn't care if the server is up.
- Track an error budget, not just uptime. It turns "are we meeting the SLA?" into "how much failure budget do we have left?" — a number people can make decisions on.
- Set your SLO stricter than your SLA so you have a buffer before you're in breach territory.
- Report continuously and honestly, including the breaches. Automate the report; a hand-built monthly PDF will always be late.
- Alert on error budget burn rate, not on crossing the availability line. By the time you're below the line, it's already an incident.
