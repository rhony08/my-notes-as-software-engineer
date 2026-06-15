# Feature Flags for Safe Releases

You shipped a new checkout flow. Everything looked fine in staging. Then five minutes into production, your on-call phone lights up — users can't complete purchases, revenue is tanking, and you're scrambling to revert. The rollback takes fifteen minutes. Each minute costs real money.

Now imagine flipping a switch instead. No deploy. No rollback. Just a toggle that turns the new flow off while you figure out what went wrong. That's the promise of feature flags.

## What's a Feature Flag?

It's a conditional in your code that controls whether a feature is active — without deploying new code.

```python
# The flag decides which checkout version runs
if feature_flags.is_enabled("new_checkout_flow"):
    return render_new_checkout(user)
else:
    return render_legacy_checkout(user)
```

That's it. A simple if/else. But the implications are huge.

## Why Bother?

### Decouple Deploy from Release

This is the big one. Without flags, "deploy" and "release" are the same thing. You push code, it goes live. Maybe you're fine with that for trivial changes, but for anything risky?

| Approach | Deploy | Release | Rollback |
|----------|--------|---------|----------|
| No flags | Ship code → users see it | Same moment | Revert + redeploy |
| With flags | Ship code hidden | Flip flag when ready | Flip flag back |

You can deploy dark code all day. Release it when you're confident. Roll it back in seconds if things go sideways.

### Progressive Rollouts

Instead of releasing to 100% of users at once, you can ramp up:

```python
# Roll out to 10% of users, then increase gradually
if feature_flags.is_enabled("new_checkout_flow", user_id=user.id):
    rollout_pct = feature_flags.get_percentage("new_checkout_flow")
    if hash(user.id) % 100 < rollout_pct:
        return render_new_checkout(user)
```

Watch metrics at 1%. Then 5%. Then 25%. At any point, if error rates spike — pause or rollback.

### Kill Switches

This alone is worth the investment. Every risky feature should have a kill switch that doesn't require a deploy.

```python
# ❌ No kill switch — must redeploy to disable
if os.getenv("ENABLE_NEW_FEATURE") == "true":
    run_experimental_module()

# ✅ Kill switch via feature flag service
if feature_flags.is_enabled("experimental_module"):
    run_experimental_module()
```

Why `os.getenv` isn't a kill switch: you still need to redeploy or restart the process to pick up the env change. Feature flags are live — you toggle, the app picks it up on the next check.

## Flag Categories

Not all flags are the same. Mixing them up causes chaos.

| Type | Lifetime | Example | Who Controls |
|------|----------|---------|-------------|
| **Release toggle** | Short (days/weeks) | Enable new checkout for 10% → 100% | Engineering |
| **Experiment flag** | Medium (weeks/months) | A/B test two recommendation algorithms | Product/Data |
| **Permission flag** | Permanent | Enable premium feature for beta users | Product/Sales |
| **Ops flag** | Long-lived | Toggle debug logging for specific tenants | Operations |

### The Trap: Permanent Flags

Release flags should die. If "new_checkout_flow" is at 100% for six months and nobody removed it, congratulations — you have technical debt called a flag.

```python
# ❌ Dead flag left in code
if feature_flags.is_enabled("new_checkout_flow"):
    # This always runs now. Flag is useless.
    return render_new_checkout(user)
```

**Fix:** Once a feature is fully rolled out, remove the flag and the old code path. Your team should have a cleanup checklist as part of release.

## Implementation Approaches

### 1. Config-Based (Simple)

Read flags from a config file that your app periodically refreshes.

```json
{
  "flags": {
    "new_checkout_flow": {
      "enabled": true,
      "percentage": 25
    }
  }
}
```

**Pros:** Dead simple, no extra infra needed.
**Cons:** Requires file deploy or config reload, no targeting per user.

Good for early-stage projects or internal tools. Bad for anything with real traffic.

### 2. Database-Backed

Store flags in a DB, cache aggressively.

```sql
-- flags table
CREATE TABLE feature_flags (
    name VARCHAR(100) PRIMARY KEY,
    enabled BOOLEAN NOT NULL DEFAULT false,
    percentage INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP
);

-- user flag overrides
CREATE TABLE user_flags (
    user_id VARCHAR(50),
    flag_name VARCHAR(100),
    enabled BOOLEAN NOT NULL,
    PRIMARY KEY (user_id, flag_name)
);
```

**Pros:** Full control, user targeting, easy to audit.
**Cons:** You're now responsible for a flag management system. Cache invalidation. Race conditions.

### 3. Dedicated Service (Recommended for serious use)

Use something like LaunchDarkly, Unleash, or Split.

**Pros:** Real-time updates, gradual rollouts, targeting rules, SDK handles caching, audit logs, experiments.
**Cons:** It's a SaaS bill. But usually cheaper than building your own.

### Trade-off at Scale

> "We built our own feature flag system using a Redis-backed config service. It worked fine until someone accidentally toggled a flag that affected 50M users with the wrong targeting rule. The audit log was, let's say, insufficient."

If you have more than a handful of services, pay for a proper solution or invest heavily in your own. Half-baked flag systems cause more incidents than they prevent.

## Real-World Patterns

### Kill Switch Pattern

Every flag service should have this. Period.

```python
# Emergency: disable ALL flags with a global kill switch
if feature_flags.global_kill_switch():
    log.warning("Global kill switch activated")
    return run_legacy_code()
```

Laurent_durieux: This flag runs old code for everything. When things go catastrophically wrong, you pull the big red lever.

### Sticky Flags

For checkout flows and other state-sensitive features, a user shouldn't flip between old and new code on page reload.

```python
# Assign user to a variant, then stick to it
if feature_flags.is_enabled("new_checkout", user_id=user.id):
    assigned_variant = feature_flags.get_variant("new_checkout", user.id)
    # Store this in session so it doesn't change mid-flow
    session["checkout_variant"] = assigned_variant
```

### Flag Health Checks

Every flag should die. Track it.

```python
# Monitor dead flags
check_flags_health()
# Check: was this flag at 100% for 30+ days?
# Check: is the old code path still being tested?
```

## What Can Go Wrong

- **Flag debt.** Hundreds of unused flags bogging down code, making it impossible to remove old paths.
- **Overlapping flags.** Flag A enables feature X, flag B disables it. Good luck debugging that.
- **Caching issues.** You toggle a flag, but the app doesn't poll for updates for 5 minutes. The kill switch feels dead.
- **Too many flags.** Every minor change behind a flag becomes noise. Use flags for risky changes, not every CSS tweak.

## When NOT to Use Feature Flags

- **Tiny, low-risk changes.** Changing button color? Just ship it.
- **Security patches.** Don't hide a CVE fix behind a flag. Ship immediately.
- **Database migrations.** Flags can gate *code paths*, but they don't prevent your migration from running.

## Key Takeaways

- Feature flags decouple deploy from release — ship code dark, toggle when ready
- Every risky feature needs a kill switch, and it shouldn't require a deploy
- Release flags are temporary — clean them up after rollout
- Use a dedicated service at scale; DIY only for simple cases
- Monitor flag health, enforce flag cleanup, and avoid permanent flag debt
- Not every change needs a flag — use them where speed of rollback matters
