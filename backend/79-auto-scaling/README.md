# Auto-scaling Strategies

You've built a service, it's getting traffic, and you're manually waking up at 3 AM to add more servers. That's not sustainable. Auto-scaling is the answer—but getting it right is harder than it looks.

The wrong scaling strategy leads to two equally bad outcomes: paying for idle capacity or dropping requests during traffic spikes. Let's talk about how to do this properly.

---

## Reactive vs Predictive Scaling

There are two fundamental approaches to auto-scaling. Most systems start with one and evolve toward the other.

### Reactive Scaling

React to what's already happening. CPU hits 80% → spin up another instance.

**Simple, but slow.** Here's the problem:

```
Traffic spike starts ──→ CPU rises ──→ Threshold crossed ──→ Instance provisioning ──→ Ready
      ↓                       ↓                  ↓                      ↓                     ↓
    T+0s                   T+30s              T+60s                  T+90s                T+120s
```

By the time your new instance is serving traffic, the spike might already be over. You've paid for a server that arrived late to the party.

```python
# ❌ Naive reactive approach
if cpu_percent > 80:
    add_instance()  # Already too late

# ✅ Better: use shorter metrics windows + proactive buffer
if cpu_percent > 70 and trend_is_up:
    add_instance()  # Start scaling before you hit the wall
```

### Predictive Scaling

Use historical patterns to anticipate demand. Your traffic graphs from the last 30 days are a goldmine.

```
Monday 9 AM: traffic doubles. Every week.
Friday 6 PM: traffic drops by half. Every week.
```

Predictive scaling says: "It's Monday 8:45 AM, let's pre-warm some instances." AWS predictive scaling, for example, analyzes up to 14 days of history to forecast demand.

**Trade-off:** Predictive scaling needs enough historical data. A brand-new service has no patterns to learn from. You'll need reactive fallbacks.

| Approach | Latency | Cost Efficiency | Setup Complexity |
|----------|---------|----------------|------------------|
| Reactive | High (lag behind spikes) | Good (scale down fast) | Low |
| Predictive | Low (ready before spike) | Moderate (may over-provision) | High |
| Hybrid | Low (predict + react) | Best (fine-tuned) | Highest |

---

## Horizontal vs Vertical Scaling

This decision shapes everything else about your architecture.

### Horizontal (Scale Out)

Add more instances. 3 servers become 4. Your load balancer distributes traffic.

**Pros:**
- No upper limit (theoretically infinite)
- Better fault tolerance (lose one, others keep serving)
- Often cheaper (many smaller instances vs few big ones)

**Cons:**
- Your app must be stateless or have shared session storage
- More complex networking, service discovery
- Database connections multiply (connection pool tuning needed)

### Vertical (Scale Up)

Make your instance bigger. More CPU, more RAM. Same server, bigger engine.

```yaml
# Horizontal: add more pods
replicas: 5  # was 3

# Vertical: give each pod more resources
resources:
  requests:
    cpu: "2"    # was "1"
    memory: "4Gi"  # was "2Gi"
```

**Pros:**
- Simple—no architecture changes needed
- No connection management headaches
- Works for stateful workloads (databases)

**Cons:**
- Hard cap (max instance size at your cloud provider)
- Single point of failure (bigger instance = bigger blast radius)
- Diminishing returns (going from 64GB to 128GB isn't 2x the throughput)

**The reality:** Most production systems use both. Horizontal for stateless app servers, vertical for databases.

---

## Key Metrics That Drive Scaling

What should you actually measure? CPU isn't always the answer.

### CPU Utilization

**Works well when:** Your service is CPU-bound (compute workloads, image processing).

**Fails when:** Your service is I/O bound (database queries, external API calls). CPU sits at 30% while requests queue up.

### Request Queue Depth

This is often better than CPU. Measure how many requests are waiting to be processed.

```yaml
# ❌ Only tracking CPU
scaling_policy:
  metric: cpu_utilization
  target: 70%  # Won't catch queue buildup

# ✅ Track both queue and CPU
scaling_policy:
  metrics:
    - queue_depth
    - cpu_utilization
  cooldown: 120  # seconds between scaling actions
```

### Custom Business Metrics

Sometimes standard metrics miss the point.

- **Requests per second** — straightforward, but doesn't account for varying request complexity
- **P50/P99 latency** — if latency is climbing, you're over capacity regardless of CPU
- **Concurrent connections** — especially relevant for WebSocket-heavy services
- **Error rate** — spike in 5xx errors might mean you need more capacity (or something else is broken)

### Kubernetes HPA Example

Kubernetes Horizontal Pod Autoscaler can use custom metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: 500
```

---

## The Gotchas Nobody Warns You About

### Thrashing (Scale-Out / Scale-In Oscillation)

The classic mistake: scale up when CPU > 70%, scale down when CPU < 30%. But as soon as a new instance comes online, CPU drops below 30% on the old ones, triggering scale-in. Now you're cycling instances constantly.

**Fix:** Cooldown periods and stabilization windows.

```python
# ❌ No cooldown — constant churn
if cpu > 70: scale_up()
if cpu < 30: scale_down()

# ✅ With stabilization window
stabilization_window = 300  # Wait 5 minutes before re-evaluating
```

### Cold Starts

New instances aren't immediately useful. They need to:
1. Start the OS
2. Pull container images
3. Initialize your app
4. Warm caches
5. Pass health checks

This takes **30 seconds to several minutes**. If your scaling decision is based on 1-minute metrics, you're making decisions based on already-stale data.

### Connection Draining

When scaling in (removing instances), you need to gracefully terminate connections. Otherwise, active users get dropped.

AWS ALB calls this "connection draining"; Kubernetes has `terminationGracePeriodSeconds`.

```yaml
# Kubernetes: give pods time to finish
spec:
  terminationGracePeriodSeconds: 60
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 30"]
```

---

## Cloud-Native Implementations

### AWS Auto Scaling Groups

The most mature offering. Supports:
- Dynamic scaling (step, simple, target tracking)
- Scheduled scaling (predictable events like Black Friday)
- Predictive scaling (ML-based forecasts)

```bash
# Target tracking — easiest to get right
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    }
  }'
```

### Kubernetes HPA + VPA

- **HPA** (Horizontal Pod Autoscaler): add/remove pods
- **VPA** (Vertical Pod Autoscaler): resize pod resources
- **Cluster Autoscaler**: add/remove nodes when pods can't schedule

Together they handle the full picture: HPA manages app-level scaling, Cluster Autoscaler manages infrastructure capacity.

---

## Actionable Takeaways

- **Start with reactive, evolve to hybrid.** Reactive gets you 80% of the way and teaches you your traffic patterns.
- **Don't rely on CPU alone.** Track queue depth, latency, and request rate—CPU lies when your service is I/O bound.
- **Set cooldown windows.** 3-5 minutes minimum between scaling actions to prevent thrashing.
- **Pre-warm if you can.** Predictive scaling or scheduled scaling for known traffic patterns saves you from cold start pain.
- **Test your scaling.** Simulate traffic spikes and watch how your system responds. You don't want to discover on Black Friday that your auto-scaling config is wrong.
- **Plan for scale-in.** Connection draining isn't optional—dropping active users because you scaled down is a bad user experience.
