# Strangler Fig Pattern: Legacy Migration

You've got this massive legacy system. Nobody on the team fully understands it, the tests are sparse, and every deployment feels like defusing a bomb. Yet the business needs new features. You can't just rewrite it all at once — that's a recipe for a 18-month project that'll never ship.

This is exactly where the Strangler Fig pattern comes in.

The name comes from strangler figs — vines that grow around a host tree, slowly replacing it until the original tree is gone. In software, you incrementally replace pieces of a legacy system with new services, routing traffic to the new code piece by piece, until you can safely kill the old system.

## How It Actually Works

Instead of a big-bang rewrite, you intercept calls to the legacy system and gradually redirect them:

```
User Request
     │
     ▼
  [Router / Gateway]
     │              │
     ▼              ▼
  New Service    Legacy System
  (migrated)     (being replaced)
```

You start with a single endpoint—maybe login, or search—build a new service for it, test it with internal traffic, then flip the switch. Next endpoint. Repeat.

### The Router Is Everything

The key piece is the routing layer. It decides whether a request goes to the legacy system or the new one. This can be:

- An **API gateway** (most common)
- A **reverse proxy** (nginx, HAProxy)
- **Feature flags** at the application level
- A **load balancer** with weighted routing

```nginx
# nginx example — incremental migration
upstream legacy {
    server legacy-app:8080;
}

upstream new_search {
    server search-service:8081;
}

server {
    listen 80;

    # Migrated endpoints go to new services
    location /search {
        proxy_pass http://new_search;
    }
    location /api/v2/products {
        proxy_pass http://new_search;
    }

    # Everything else stays on legacy
    location / {
        proxy_pass http://legacy;
    }
}
```

## The Migration Flow

Here's what a typical strangler fig migration looks like in practice:

```
Phase 1: Identify + Isolate
  ┌────────────────────────────────┐
  │ Pick a bounded context         │ <- start with search, not "the whole auth module"
  │ Build the new service          │ <- clean code, proper tests
  │ Route test traffic to it       │ <- internal only at first
  └────────────────────────────────┘
  
Phase 2: Validate + Switch
  ┌────────────────────────────────┐
  │ Compare outputs (legacy vs new)│ <- run both, check for diffs
  │ Route real traffic slowly      │ <- 10% → 50% → 100%
  │ Monitor everything             │ <- errors, latency, data consistency
  └────────────────────────────────┘
  
Phase 3: Clean Up
  ┌────────────────────────────────┐
  │ Verify old system gets no calls│
  │ Remove dead code/routes        │
  │ Archive or decom legacy infra  │
  └────────────────────────────────┘
```

### Parallel Run Pattern

One technique worth knowing: **parallel run**. You route requests to both systems but only serve the legacy response to users. The new service's response gets logged and compared. This gives you confidence without risk.

```
User → Gateway
  ├──→ Legacy System → User (actual response)
  └──→ New Service  → Log (for comparison only)
```

Set up monitoring to alert when outputs diverge:

```python
# Pseudo-code for comparison logic
def handle_search_request(query: str):
    legacy_result = legacy_search(query)
    new_result = new_search(query)
    
    # Log any differences for investigation
    if legacy_result != new_result:
        logger.warning(
            "Output mismatch for search query",
            extra={
                "query": query,
                "legacy_result": truncate(legacy_result),
                "new_result": truncate(new_result),
                "diff_score": compute_diff(legacy_result, new_result),
            }
        )
        # Flag for manual review
        alert_team("search_mismatch", query)
    
    return legacy_result  # Keep serving old until validated
```

## When It Works (and When It Doesn't)

Let's be real about the trade-offs.

**Good fit for:**
- Large monolithic apps that grew organically over years
- Systems with clear bounded contexts you can carve out
- Teams that can't afford a full rewrite
- Migration of specific features (search, payments, user profiles)

**Not so great for:**
- Tiny apps — just rewrite it, it'll be faster
- Systems where you can't safely route individual endpoints (tightly coupled monolith with shared state everywhere)
- Teams without decent monitoring — you'll be flying blind
- When the legacy data model is a tangled mess that's coupled with everything

```python
# ❌ Problem: shared database makes strangling harder
# The legacy DB is accessed by both old and new code
def create_order_legacy(order_data):
    legacy_db.execute("INSERT INTO orders ...")
    legacy_db.execute("INSERT INTO order_items ...")
    legacy_db.execute("UPDATE inventory ...")  # Coupled!

# ✅ Better: introduce a data sync layer or facade
def create_order_new(order_data):
    # New service owns its own data
    new_db.execute("INSERT INTO orders ...")
    
    # Sync back to legacy for remaining consumers
    event_bus.publish(OrderCreatedEvent(order_data))
    # Eventually, migrate all consumers off legacy
```

## The Big Gotcha: Data

Code migration is the easy part. Data migration is where things get ugly.

When you move a feature to a new service, you usually need to bring its data too. Now you've got two problems: keeping data in sync during the migration, and handling writes that happen on both sides during the transition.

**Strategies:**

| Approach | How It Works | Best For |
|----------|-------------|----------|
| Dual writes | Write to both systems, read from new | Short transitions, simple schemas |
| Async sync | Write to one, pipeline to the other | When availability > consistency |
| Shared DB | Both systems read/write same DB | When splitting is gradual and you accept coupling |
| CDC pipeline | Use change data capture to sync | Complex schemas, zero-touch legacy |

## Practical Steps to Start

1. **Find your first victim** — Look for an endpoint that's self-contained and low-risk. Maybe something read-only. Search is a classic first pick.

2. **Build a shadow copy** — Stand up the new service, point it at a replica of the legacy data. Route test traffic only.

3. **Compare outputs for a week** — You'll find edge cases you never anticipated. Fix them. Iterate.

4. **Cut over carefully** — Start with 1% of real traffic. Watch dashboards. Slowly ramp up.

5. **Kill the old code** — Once the old endpoint gets zero traffic, delete it. No dead code. No zombie routes.

## Things I Wish I Knew Earlier

- **Use feature flags** alongside the strangler. You'll want the ability to toggle individual users or tenants to the new system without redeploying.
- **Don't touch the data model on day one.** Keep the same schema in the new service. You can optimize later — the migration is hard enough without schema changes.
- **Monitoring is non-negotiable.** Before you strangler anything, make sure you can see error rates, latency, and data drift for both systems side by side.
- **Plan for rollback.** Every endpoint you migrate needs a kill switch that can fail back to legacy in under a minute.

## Take Away

- The Strangler Fig pattern lets you replace legacy systems incrementally instead of risky big-bang rewrites
- The routing layer (gateway/proxy) is the critical piece — it directs traffic to old or new code
- Start with low-risk, self-contained endpoints (read-only is ideal)
- Run both systems in parallel during migration to catch discrepancies
- Data migration is harder than code migration — plan for it upfront
- Monitoring, rollback plans, and feature flags are your safety net