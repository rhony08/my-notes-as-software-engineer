# Rollback Strategies

You rolled out a shiny new deploy. Five minutes later, error rates spike, SRE is pinging you, and your customers are seeing 500s instead of cat pictures.

This is the moment rollback strategies earn their keep. A good rollback plan doesn't just undo your deploy—it does it fast, safely, and without making things worse.

## The Worst Option: Revert-and-Reploy

Here's the naive approach everyone's tried at least once:

```bash
# Someone realizes the deploy is broken
# They run:
git revert HEAD
git push
# Wait for CI... wait for CD... wait for approval...
# 15 minutes later, the fix is live. If you're lucky.
```

The problem? **Time to recovery isn't zero**. Every minute your broken code runs, more damage piles up—corrupted data, angry users, lost revenue.

**Rollbacks should take seconds, not minutes.**

## Deployment Strategy Dictates Your Rollback Options

The rollback strategy you can use depends entirely on how you deployed in the first place.

### 1. Rolling Deploy — The Slow Undo

**How it works:** Instances are updated one-by-one (or batch-by-batch). A new instance replaces an old one until all are updated.

```
Before: [A][A][A][A][A]  (version 1.0)
During: [B][B][A][A][A]  (version 2.0 rolling)
Full:   [B][B][B][B][B]  (version 2.0 complete)
```

**Rollback process:** Re-run the deploy pipeline with the previous version. Instances get replaced again, one-by-one.

| Pros | Cons |
|------|------|
| No additional infra cost | Rollback is slow (same as deploy time) |
| Works with existing tooling | During rollback, some users hit broken version |
| Simple to implement | Requires artifact of old version to still exist |

```
❌ Bad: "We'll just redeploy the old code"
   — Requires old artifacts to exist; won't help with DB schema changes

✅ Better: Keep the last N versions in your artifact registry
   — Kubernetes: `kubectl rollout undo deployment/my-app`
   — Most CI tools have "rollback to previous" buttons
```

**Cleanest approach:** Kubernetes does this natively with `rollout undo`:

```bash
# Check rollout history
kubectl rollout history deployment/api-service

# Rollback to previous revision
kubectl rollout undo deployment/api-service

# Rollback to a specific revision
kubectl rollout undo deployment/api-service --to-revision=3
```

This re-applies the previous ReplicaSet. Quick, reliable, and it keeps your audit trail intact.

### 2. Blue-Green — Instant Cutover

**How it works:** You run two identical environments ("blue" and "green"). Only one serves live traffic at any time. New version deploys to the idle environment, then you switch traffic.

```
Green: [v2.0] ← idle
Blue:  [v1.0] ← serving live traffic  → switch traffic → Blue: [v1.0] ← idle
```

**Rollback process:** Switch traffic back to the previous environment. That's it. No redeploy needed.

```nginx
# Load balancer config for blue-green traffic switch
upstream backend {
    # Switch this line to revert
    server blue-cluster.example.com;  # current live
    # server green-cluster.example.com;  # rollback target
}
```

**This is the gold standard for speed.** Rollback is a literal config flip.

| Pros | Cons |
|------|------|
| Sub-second rollback | 2x infrastructure cost |
| Zero downtime during rollback | State/DB compatibility still matters |
| Easy to test in idle env first | Data schema changes complicate things |
| Traffic can be split for canary testing | Takes longer to provision full environments |

### 3. Canary Deploy — Incremental Trust

**How it works:** Roll out to a small percentage of users first (say 5%). Monitor metrics. Gradually increase if things look good.

```
Traffic: 100% → v1
Traffic: 95% v1, 5% v2  → metrics show errors!
Rollback: redirect that 5% back to v1
```

**Rollback process:** Tell the load balancer to stop routing traffic to canary instances. The 5% gets redirected back to the stable version.

```yaml
# Canary config (Istio-style VirtualService)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-service
spec:
  hosts:
  - api.example.com
  http:
  - match:
    - headers:
        x-canary: "true"
    route:
    - destination:
        host: api-service
        subset: v2  # canary
      weight: 5
  - route:
    - destination:
        host: api-service
        subset: v1  # stable
      weight: 95
```

**Rollback:** Change it back to 100% v1. Done.

| Pros | Cons |
|------|------|
| Fast rollback (config change) | Complex to set up (requires service mesh or advanced LB) |
| Minimal blast radius | Telemetry must be good enough to detect issues early |
| Real-world testing on live traffic | Canary users may have inconsistent experience |
| Metrics-driven rollout decisions | Statistically significant metrics need enough traffic |

## The Hard Part: Database Rollbacks

Here's where things get messy. Application code is easy to revert. But your database doesn't "undo" easily.

```
❌ You changed a column type from INT to VARCHAR
❌ You dropped an unused column that was apparently very used
❌ A migration deleted data (which is now gone forever)
```

**Common approaches for DB rollbacks:**

### Forward-fix (recommended)

Instead of rolling back the DB, write a new migration that reverses the change:

```sql
-- Migration that broke things
ALTER TABLE users DROP COLUMN legacy_api_key;

-- Forward-fix migration
ALTER TABLE users ADD COLUMN legacy_api_key VARCHAR(255) NULL;
```

This keeps the schema moving forward instead of backward, which avoids complications with feature branches that depend on the new schema.

### Defensive migrations (even better)

Structure your migrations so they're safe to have running alongside old code:

```sql
-- Phase 1: Add new column (safe to add, old code ignores)
ALTER TABLE users ADD COLUMN new_api_key VARCHAR(255) NULL;

-- Phase 2: Backfill data in batches (run after deploy stabilizes)
UPDATE users SET new_api_key = legacy_api_key WHERE new_api_key IS NULL LIMIT 1000;

-- Phase 3: Old code is gone, now we can drop (next deploy, if everything is stable)
ALTER TABLE users DROP COLUMN legacy_api_key;
```

**Key principle:** Deploy changes in phases where each phase is reversible.

### Feature flags to the rescue

The cleanest approach combines feature flags with deploy safety:

```python
# Feature flag gates the behavior, not the deploy
if feature_flags.is_enabled("new_api_key_system"):
    return generate_new_api_key(user)
else:
    return user.legacy_api_key  # old path still works
```

If the new system breaks, flip the flag off. No rollback needed. The old code path is still there, still tested, still working.

## When NOT to Rollback

Counter-intuitive, but sometimes rolling back is the wrong call:

| Don't rollback when... | Why |
|------------------------|-----|
| DB migration already ran | Reverting the code doesn't revert the schema. You'll get errors from code that expects old columns |
| Data was transformed irreversibly | If you normalized, deduplicated, or encrypted data — rolling back doesn't restore the original |
| The bug involves data integrity | Rolling back might introduce false confidence. The damage already happened |
| Security fix | Undoing a security patch is worse than living with a performance regression |

In these cases, a **forward-fix** (hotfix deploy that fixes the bug) is better than a rollback.

## Actionable Takeaways

- **If you can choose, use blue-green or canary** — rollback is a config flip, not a redeploy
- **Keep old artifacts** — purge old containers, but keep at least the last 3 versions
- **DB changes need forward-fix migrations** — never write irreversible migrations in a single step
- **Test your rollbacks** — rollback is a recovery procedure. If you never practice it, it won't work when you need it
- **Feature flags are safer than rollbacks** — if you need to undo behavior without touching infra, flags win
- **Know when NOT to rollback** — sometimes a hotfix is the lesser evil
