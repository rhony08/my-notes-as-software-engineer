# Multi-region Architecture Design

Your app is running fine in `us-east-1`. Users are happy. Then AWS has a networking issue in that region for 45 minutes, and suddenly your entire platform is dark. That's the problem multi-region architecture solves — and it's deceptively hard to get right.

## Why Bother with Multiple Regions?

Single-region setups are simple. One cluster, one database, one set of worries. But as soon as uptime matters beyond 99.9%, or your users spread across continents, you'll feel the pain:

- **A regional outage sinks you** — one AZ failing is manageable, but an entire region going dark? That's game over for a single-region app
- **Latency for global users** — hitting `us-east-1` from Southeast Asia adds 200-300ms. That's not just slow, it changes user behavior
- **Data sovereignty requirements** — GDPR in Europe, PDPB in India, LGPD in Brazil. You can't serve EU users from a US-only stack

Multi-region solves all three, but introduces a whole set of new problems.

## The Core Tension: Latency vs. Consistency

Here's the fundamental trade-off in multi-region architecture:

```
Multi-region is about picking the right trade-off between
   ⚡ Low Latency   ↔   🔒 Data Consistency
```

Fast reads everywhere usually means stale data somewhere. Perfect consistency usually means everything slows to single-region speed. You can't have both.

### What Breaks in Multi-region

Things that work fine in one region that become painful across regions:

| Challenge | Why It Hurts |
|-----------|-------------|
| **Database writes** | Can't have a single write master in `us-east-1` if your users in `eu-west-1` are writing to their local copy |
| **Session state** | User logs in on `us-east-1`, next request hits `eu-west-1` — they're logged out |
| **Cache invalidation** | Redis in one region doesn't know about Redis in another region |
| **DNS propagation** | Routing users to the right region isn't instant |
| **Conflict resolution** | User edits the same record in two regions — whose version wins? |

## Three Patterns for Multi-region Architecture

Let's walk through the real approaches, from simplest to most complex.

### Pattern 1: Active-Passive (Least Painful)

One region handles all traffic. The second region is a warm standby — databases are replicating, app servers are running, but not serving traffic.

```
User → Route53 → us-east-1 (active)
                         ↓
                    (replication)
                         ↓
                    eu-west-1 (passive)
                    ↑ DNS failover
```

**When to use:** You're mostly worried about disaster recovery, not latency. Most users are in one region.

**The trade-off:** Failover takes 1-10 minutes. DNS needs to propagate, databases need to catch up. You'll lose some writes that were in-flight.

```
// ❌ Active-passive with async replication
// Problem: If primary region goes down, un-replicated writes are gone

// ✅ Better: synchronous replication for critical data
// But this adds latency to every write
```

### Pattern 2: Active-Active with Local Writes (High Performance)

Every region accepts reads and writes. Each region has its own database. Data syncs between regions asynchronously.

```
User in EU → eu-west-1 (reads & writes)
User in US → us-east-1 (reads & writes)
        ↕             ↕
    (async replication between regions)
```

**When to use:** Global user base, low latency is critical, and you can tolerate eventual consistency.

**The trade-off:** Users will occasionally see stale data. If someone books a flight in the US and checks it in the EU 2 seconds later, it might not show up yet.

```
// Multi-region user creation — handling ID conflicts
// Each region gets a unique ID prefix to avoid collisions
function generateUserId(region: string): string {
  const prefix = REGION_PREFIXES[region]; // 'us1', 'eu2', 'ap3'
  const timestamp = Date.now().toString(36);
  const random = Math.random().toString(36).substr(2, 5);
  return `${prefix}_${timestamp}_${random}`;
}
// User from EU → "eu2_lx5a2j_3f8k2"
// User from US → "us1_lx5a2j_7dm1q"
```

### Pattern 3: Active-Active with CRDTs (Eventual + Automatic Conflict Resolution)

The hardest pattern, but the most resilient. Each region is fully independent and merges changes using CRDTs (Conflict-free Replicated Data Types).

```
eu-west-1 (full stack) ←→ us-east-1 (full stack)
    |                          |
  (CRDT merge)            (CRDT merge)
    |                          |
ap-southeast-1 (full stack) ←→ sa-east-1 (full stack)
```

**When to use:** You need 100% uptime across regions, offline-first apps, or collaborative editing (like Google Docs).

**The trade-off:** CRDTs add complexity. You need data structures designed for conflict resolution, and not every use case maps well to CRDTs. Also, history size can grow unbounded.

Real-world example — a shopping cart with CRDT merge:

```javascript
// A shopping cart that can merge from different regions
class MergeableCart {
  constructor(userId) {
    this.userId = userId;
    this.items = new Map(); // itemId → { count: 0, version: 0 }
  }

  addItem(itemId, count, regionVersion) {
    const current = this.items.get(itemId) || { count: 0, version: 0 };
    
    if (regionVersion > current.version) {
      // Newer version wins
      this.items.set(itemId, { count, version: regionVersion });
    } else {
      // Merge: take the higher count
      this.items.set(itemId, { 
        count: Math.max(current.count, count), 
        version: current.version 
      });
    }
  }

  // ❌ Simple approach: last-write-wins
  // User adds item in US but the EU overwrite removes it

  // ✅ CRDT approach: take max count
  // Both regions' additions are preserved even if they conflict
}
```

## Routing Users to the Right Region

DNS-based routing is the most common approach, but it's not instant. Here's where the latency comes from:

```
User request → DNS lookup → Geo-routed to nearest region → App → DB

DNS time: 50-200ms (depending on TTL)
Network round-trip: 50-300ms (depending on distance)
Processing: 20-200ms (depending on complexity)
```

To minimize DNS lag:

```yaml
# Route53 latency-based routing config
# Route users to region with lowest latency, not just "closest"
Records:
  - Name: api.myapp.com
    Type: A
    SetIdentifier: us-east-1
    Routing: Latency
    Failover: Primary
    TTL: 10  # Low TTL for quick failover
  - Name: api.myapp.com
    Type: A
    SetIdentifier: eu-west-1
    Routing: Latency
    Failover: Secondary
    TTL: 10
```

## Database Replication Between Regions

This is where most multi-region architectures fail. Let's look at the options.

### Option A: Active-Passive with Cross-Region Replication

```sql
-- Primary region (us-east-1)
CREATE DATABASE app_db;
-- All writes go here

-- Secondary region (eu-west-1)
-- Read replica of us-east-1
-- Accepts SELECT only
-- Failover promotes this to primary
```

**Pros:** Simple, no conflict resolution needed, consistent reads
**Cons:** Cross-region replication lag (100ms-5s), writes are single-region

### Option B: Multi-Active with Conflict Resolution

```sql
-- Each region has its own primary database
-- Tables need to handle conflicts

CREATE TABLE user_profiles (
  user_id UUID PRIMARY KEY,
  display_name TEXT,
  region TEXT NOT NULL DEFAULT current_region(),
  last_updated TIMESTAMP WITH TIME ZONE,
  version INT DEFAULT 1
);

-- Conflict resolution: last-writer-wins by region priority
-- us-east-1 > eu-west-1 > ap-southeast-1
```

## The Infrastructure Cost Reality

Multi-region isn't free. Here's a rough estimate for a modest setup:

| Component | Single Region | Two Regions | Delta |
|-----------|--------------|-------------|-------|
| EC2 (4 nodes) | $400/mo | $800/mo | 2x |
| RDS (db.r6g.large) | $300/mo | $600/mo + replication | 2x+ |
| Data transfer | $100/mo | $200-400/mo | 2-4x |
| Route53 latency routing | $0 | ~$100/mo | added |
| DevOps complexity | 1x | ~3-5x | not $ but time |

**The real cost:** Cross-region data transfer. Moving data between regions is expensive — AWS charges $0.02-0.09/GB depending on regions. If you're syncing terabytes, that adds up fast.

## Practical Patterns for Getting Started

Don't go full multi-region on day one. Here's a sane migration path:

```
Phase 1: Active-Passive with manual failover (this week)
  → Deploy app in region 2, validate DNS failover manually
  
Phase 2: Active-Passive with automated health checks (next month)
  → Route53 health checks, auto-failover when primary goes dark
  
Phase 3: Active-Active for read-only workloads (quarter 2)
  → Read replicas in multiple regions, writes go to primary
  
Phase 4: Active-Active for writes (quarter 3+)
  → Only after you've tested, validated, and understood your data model
```

## When Multi-region Isn't Worth It

Be honest with yourself:

- **Your uptime SLA is 99.9%?** Multi-region is overkill. A well-architected single-region setup with multi-AZ handles this.
- **You have < 50K users, all in one country?** Spend that complexity budget on faster queries and better caching instead.
- **Your data has strict consistency requirements?** Multi-region eventual consistency will cause customer-facing bugs.
- **You don't have dedicated ops?** Multi-region complexity will bury a team of less than 5-7 engineers.

## Takeaways

- **Active-passive is the safest starting point** — one region handles traffic, another waits. Failover takes minutes but it's reliable
- **For global latency, go active-active with local writes** — but accept eventual consistency. Users get fast reads, occasional stale data
- **Database replication is the hardest part** — sync replication adds latency, async replication creates conflicts. Pick your poison
- **Cost scales non-linearly** — two regions cost more than 2x. Cross-region data transfer burns money fast
- **Start simple and iterate** — active-passive → read replicas in other regions → full active-active. Don't try to build Google in a week
