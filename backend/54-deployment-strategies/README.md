# Deployment Strategies: Blue-Green, Canary

You've got your CI pipeline running, tests are green, and your artifact is ready. Now comes the scary part: getting that code into production without breaking things.

The classic approach—shut down the old version, start the new one—means downtime. And if something goes wrong, you're scrambling to redeploy the previous version while users stare at error pages.

Let's talk about deployment strategies that avoid this mess.

## The Problem with Simple Deployments

A basic rolling update or a straight swap has two issues:

1. **You have no safety net** — if the new version has a bug, it's already live
2. **Users see transient errors** — connections drop, requests fail during the switch

The strategies we'll cover—blue-green and canary—solve these differently.

---

## Blue-Green Deployments

The idea is dead simple: maintain two identical environments. At any time, only one serves production traffic. The other is idle.

```
┌─────────────┐     ┌─────────────┐
│   Blue      │     │   Green     │
│  (live)     │     │  (idle)     │
│  v1.2.0     │     │  v1.3.0     │
└──────┬──────┘     └─────────────┘
       │
       ▼
   Production
    Traffic
```

When you deploy v1.3.0, it goes to the idle environment. You run smoke tests against it, verify everything works, then flip the router.

```bash
# Deploy new version to idle environment
kubectl apply -f k8s/green-deployment.yaml

# Verify
curl -H "Host: internal-staging.example.com" https://green-env/health

# Flip traffic
kubectl patch service my-app -p '{"spec":{"selector":{"env":"green"}}}'
```

### The Magic Part

If v1.3.0 has a problem, you flip back. Instantly.

```bash
# Rollback - just point traffic back to blue
kubectl patch service my-app -p '{"spec":{"selector":{"env":"blue"}}}'
```

That's it. Two commands: deploy to idle, flip traffic. Rollback is just flipping back.

### What You Actually Need

| Component | Purpose |
|-----------|---------|
| Load balancer / router | Controls which environment gets traffic |
| Two identical stacks | Same resources, same config, different versions |
| Database compatibility | Schema changes must work with both versions |
| Monitoring | Alerts on the new environment before full switch |

### ⚠️ The Catch

Blue-green sounds great on paper, but it's expensive. You're paying double for infrastructure. For a microservice with 50 instances, you now run 100 during deployment.

Also, **database migrations** are the weak point. If your new version adds a column, the old version running alongside it shouldn't break. You need backward-compatible schema changes.

```sql
-- ✅ Do this - old code still works because column has a default
ALTER TABLE users ADD COLUMN timezone VARCHAR(50) DEFAULT 'UTC';

-- ❌ Don't do this - old code SELECT * breaks if it doesn't know about this
ALTER TABLE users DROP COLUMN legacy_field;
```

---

## Canary Deployments

Instead of flipping all at once, you route a small percentage of traffic to the new version and watch what happens.

```
    99% ──────────────►  v1.2.0 (stable)
    
     1% ──────────────►  v1.3.0 (canary)
```

If error rates spike or latency increases, you stop the rollout and route everyone back. If metrics look good after 10 minutes, bump it to 10%, then 25%, then 50%, then 100%.

### How It Looks in Practice

With Kubernetes, this is straightforward using a service mesh or weighted services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  ports:
  - port: 80
---
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: my-app-canary
spec:
  service: my-app
  backends:
  - service: my-app-stable
    weight: 990
  - service: my-app-canary
    weight: 10
```

This sends 99% to stable, 1% to canary. You watch your dashboards, and if everything's clean:

```yaml
# Bump canary to 50%
  backends:
  - service: my-app-stable
    weight: 500
  - service: my-app-canary
    weight: 500
```

### Automated Rollback

The real win with canary deployments is automation. Tools like Flagger or Argo Rollouts can watch metrics and auto-rollback.

```
❌ Canary failed: error rate 5.2% (threshold: 2%)
   Duration: 3m 24s
   Action: Rolled back to primary
```

No human needed. Your monitoring tells the orchestrator "this is bad," and traffic returns to the stable version.

### ⚠️ The Catch

Canary deployments are more complex to set up. You need:

- **Traffic splitting capability** — service mesh, ingress controller, or load balancer that supports weighted routing
- **Good observability** — you need real-time metrics to make the call
- **Stateless services work best** — sticky sessions complicate traffic routing
- **Longer deployment time** — rolling out to 100% takes minutes or hours, not seconds

---

## When to Use Which

| Scenario | Pick |
|----------|------|
| Simple API, low traffic | Blue-green |
| High-traffic critical service | Canary |
| Infrastructure change (DB, config) | Blue-green (easier to verify) |
| Machine learning model update | Canary (watch for regression) |
| Tiny team, few resources | Blue-green on a budget (single active, keep previous version around) |
| You have a service mesh already | Canary |

### Hybrid Approaches

They're not mutually exclusive. A common pattern:

1. **Deploy to blue-green environments** — get the code running and smoke-tested
2. **Use canary routing** — slowly shift traffic to the new environment
3. **If all good** — roll to 100%
4. **If bad** — flip back to old environment entirely

```bash
# Step 1: Deploy to idle (green)
kubectl apply -f k8s/green-deployment.yaml

# Step 2: Start canary at 5%
flagger canary my-app --weight=5

# Step 3: Monitor for 5 minutes, then ramp up
flagger canary my-app --weight=50

# Step 4: Full switch
flagger canary my-app --weight=100
```

---

## What Actually Goes Wrong

**Stateful services bleed.** If your app stores session data in memory, users get logged out on flip. Use external session stores (Redis, DB) regardless of deployment strategy.

**DNS caching kills instant rollbacks.** If you're using DNS-level traffic switching (Route53, Cloudflare), TTL means you wait minutes for traffic to shift. Use load balancer-level switching for instant flips.

**Observability isn't ready.** Setting up canaries without proper metrics is dangerous. You won't know something's wrong until users complain.

```
❌ "Hmm, error rate looks... fine? I think?"
   ← This is not a monitoring strategy
```

**Feature flags + deployment strategy** is the ultimate combo. Use flags to disable new features even after deployment, giving you an extra safety layer beyond traffic routing.

---

## Takeaways

- **Blue-green** is for instant rollback — costs double but you can flip back in seconds
- **Canary** is for gradual confidence — complex to set up but catches problems before they reach everyone
- **Database schema changes** are the hardest part of any deployment strategy — always design for backward compatibility
- **Combine both approaches** for the smoothest deployments: blue-green environments + canary traffic routing
- **Automated rollback** should be your goal — let metrics decide, not humans panicking at 3am
- **Test your rollback regularly** — if you only test rollback during incidents, it'll fail when you need it most
