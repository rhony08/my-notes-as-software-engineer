# Dashboard Design: Why Your Dashboards Collect Dust

Someone on your team spent a week building a dashboard with 47 panels. It tracks everything — CPU, memory, request rates, JVM heap, disk I/O, the phase of the moon. And yet, when the pager goes off at 3am, the on-call engineer doesn't open it. They SSH into the box and run `curl` and `top` like it's 2005.

Sound familiar? The dashboard isn't broken — it's useless. It was built to show *everything*, so it shows *nothing*. Good dashboard design isn't about pretty charts. It's about building a tool that answers the question "is this broken, and if so, where?" in under ten seconds. We'll cover how to pick what goes on a dashboard, how to lay it out, and the anti-patterns that turn dashboards into wallpaper.

## Every Panel Must Answer a Question

Here's the test I use: if someone asks "why is this panel here?" and you can't answer with a specific decision it helps you make, delete it.

A panel exists to answer one of these:
- Is the service healthy right now?
- What broke, and where do I look next?
- Is this trending toward a problem (disk filling up, queue backing up)?
- Did my last deploy actually help or hurt?

If a chart doesn't drive one of those, it's decoration. And decoration on an ops dashboard is worse than nothing — it buries the one signal that matters under noise.

```
# ❌ Dashboard filler
A panel showing "total requests since deploy" as a huge number.
Nobody can act on this. It's a counter plotted raw, forever climbing.

# ✅ Actually useful
Request rate over the last hour, colored by HTTP status.
If the error rate spikes, you see it immediately and know where to click.
```

A good rule of thumb: a dashboard should fit on one screen and be readable in one glance. If you need to scroll, you've got a chart wall, not a dashboard.

## Know Who's Looking

Dashboards aren't one-size-fits-all. The on-call engineer, the backend team, and the VP of Engineering all need different things — and mixing them into one dashboard guarantees it serves nobody.

| Audience | Question they ask | What they need |
|----------|------------------|----------------|
| **On-call** | "What's broken right now?" | Error rates, latency percentiles, queue depth, links to logs/traces |
| **Team** | "Did our change help?" | Deploy markers, throughput, latency before/after, resource usage |
| **Management** | "Are we healthy and trending well?" | Uptime %, SLO burn, cost, capacity — daily/weekly view, not seconds |

The classic mistake: giving execs a real-time dashboard with latency percentiles. They don't care about p99 at 2:14pm — they care about whether last month's uptime hit the SLO. And giving on-call a monthly trend chart when the pager fires is equally useless.

Practical takeaway: keep one **ops dashboard** (realtime, actionable, linked to runbooks) and one **team/business dashboard** (trends, weekly aggregates). Don't merge them.

## Start With the Golden Signals

If you're staring at a blank Grafana and don't know where to begin, the golden signals are your checklist. Two frameworks cover almost everything:

**RED** — for request-driven services (APIs, web apps):
- **R**ate — requests per second
- **E**rrors — % of requests failing (5xx, timeouts)
- **D**uration — latency percentiles (p50, p95, p99)

**USE** — for resources (databases, servers, queues):
- **U**tilization — % busy (CPU, memory, disk, connections)
- **S**aturation — how much *waiting* (queue depth, thread pool backlog)
- **E**rrors — hardware/software errors (disk failures, dropped connections)

That's it. Most services can be covered with one RED dashboard plus a USE dashboard for the database. Everything else is context: deploy markers, config changes, correlated business metrics.

## Layout: Respect the Reading Order

People read dashboards like they read a newspaper — top-left to bottom-right, skimming. Design for that.

- **Most important panel goes top-left.** That's the first thing eyes land on. For most services: overall error rate and latency.
- **Left-to-right flow = the story.** On the left, "is it broken" (RED). Middle: where to look (per-endpoint breakdown). Right: supporting context (resources, queue depth).
- **Group related panels.** Don't scatter latency charts around the page. Cluster by subsystem: API → DB → workers → infrastructure.
- **Keep the header calm.** Service name, environment, and a time-range selector. Not eleven logos and a rainbow.

```
# ❌ Scattered chaos
Latency panel top-left, error rate bottom-right,
disk usage middle, no grouping, 14 colors.

# ✅ One-glance story
Row 1: Error rate | Latency p50/p95/p99 | Request rate
Row 2: Per-endpoint latency table | Per-endpoint error table
Row 3: DB connections | Queue depth | Deploy markers
```

## Time Ranges and Aggregation Are the Silent Killers

You can build a perfect dashboard and still have it be useless because the defaults are wrong.

- **Default range matters.** An on-call dashboard should default to the last 15–30 minutes, not 24 hours. A 24-hour view averages away the incident you're trying to see. The team dashboard can default to a week.
- **Aggregation hides spikes.** If you graph 30 days with `avg` and a 1-day step, a 20-minute outage becomes a barely visible blip. For incident hunting you want raw or `max` at a fine granularity; averages are for trends.
- **Percentiles over averages.** Average latency is a lie — one slow request drags it up, and it hides the fact that 95% of users are fine. p50/p95/p99 tells you the real shape. Plot p95 and p99 by default.

```
# ❌ Averaging away the incident
avg(latency_seconds[1d]) — an outage looks like a small bump

# ✅ Seeing the spike
max(latency_seconds[1m]) — the 20-minute outage is a wall
```

## Anti-Patterns That Scream "Amateur Dashboard"

These are the things that make operators distrust a dashboard, consciously or not:

| ❌ Anti-pattern | Why it's bad | ✅ Instead |
|---|---|---|
| Rainbow colors (10+ series) | Can't tell anything apart | Max 3–4 colors, consistent meaning (green=ok, red=error) |
| 3D charts / gradients | Distort values, look flashy | Flat lines, plain bars |
| Dual y-axes | Misleads — correlation is in your head | Separate panels with shared time axis |
| Raw counters plotted directly | Always climbing, unreadable | `rate()` or `irate()` |
| Gauge showing "average" | Hides the tail | Percentiles or max |
| No alert links | Dashboard says "broken", then what? | Link each panel to logs/traces/runbook |

The last one deserves emphasis: a dashboard that tells you something is wrong but not *where to look next* is only half a tool. Every error panel should deep-link to the logs or traces for that time window. That's the difference between a dashboard and a dead end.

## A Minimal Grafana Panel That Doesn't Suck

Here's a tiny example of what a sane error-rate panel looks like — the kind of thing that actually helps during an incident:

```json
{
  "title": "Error rate (5xx)",
  "type": "timeseries",
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 1 },
          { "color": "red", "value": 5 }
        ]
      }
    }
  },
  "targets": [
    {
      "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
      "legendFormat": "5xx %"
    }
  ],
  "links": [
    {
      "title": "Open logs",
      "url": "${LOKI_URL}/explore?from=${__from}&to=${__to}&query={service=\"api\"}"
    }
  ]
}
```

Notes: thresholds color the line so it turns red *before* it's a crisis, the range variables (`${__from}`/`${__to}`) carry the incident window into the log query, and the link turns the panel into a jump-off point. That's the whole job.

## The Trade-offs Nobody Mentions

Good dashboard design is full of trade-offs, and pretending otherwise leads to dogma:

- **One big dashboard vs many small ones.** A single "everything" dashboard is easy to find but impossible to read. Per-service dashboards are readable but multiply to dozens. Compromise: one landing dashboard with links to per-service ones — the index page pattern.
- **Standardized templates vs custom builds.** Templates (SRE's Grafana dashboards, for example) get you 80% of the way and make dashboards consistent across services. But the last 20% — your queue depths, your weird async workers — is always custom. Don't force a template where it doesn't fit, and don't hand-roll what a template already covers.
- **Realtime fidelity vs cost.** Storing 1-second granularity for everything is expensive. Fine-grained data for the last hour, downsampled forever — that's what most systems do. Accept that historical dashboards are trend tools, not incident tools.
- **Self-service vs controlled.** Letting every engineer add panels makes dashboards rot into chart walls. Locking it down makes them stale. A review process ("dashboard changes go in the same PR as the code") works better than either extreme.

## What to Do Next

- **Audit your existing dashboards this week.** Open each one and delete every panel that doesn't answer a question. You'll probably cut 30–50%. That's a feature, not vandalism.
- **Build (or rebuild) your ops dashboard around RED or USE.** Pick one framework per dashboard, top-left = most important signal, default time range = 15 minutes.
- **Wire panels to the next step.** Every error panel gets a link to logs/traces for the same time window. If you can't deep-link, fix that before adding more charts.
- **Add deploy markers.** Nothing explains a latency spike faster than "oh, we shipped at 2pm." One annotation line, huge payoff.
- **Agree on the audience first.** Before building anything, write down who looks at this dashboard and what decision it helps them make. If you can't, you're not ready to build it.
