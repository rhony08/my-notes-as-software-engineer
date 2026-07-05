# Global Distribution: Multi-Region Architecture

Your app runs beautifully in one region. Users in that region get sub-20ms responses. Your database is happy. Your cache is warm. Life is good.

Then someone in Singapore signs up. Then someone in Brazil. Then half your traffic is coming from Europe and Australia — and every request crosses an ocean. That 20ms response is now 300ms. Your users notice. They leave.

Multi-region architecture isn't about showing off. It's about making your app fast and available *everywhere*, not just where you happen to be hosting it.

---

## Why Single-Region Breaks at Scale

Let's be clear about the problems before looking at solutions.

**Latency kills conversions.** Amazon calculated that every 100ms of latency costs them 1% in revenue. Google found that an extra 500ms dropped search traffic by 20%. That's not theoretical — that's real money.

**Regions go down.** In 2021, an AWS us-east-1 outage took down half the internet. If your entire infrastructure lives in one region, a single failure domain takes everything offline.

**Data has to live somewhere.** GDPR says European user data should stay in Europe. If your only region is us-west-2, you're either breaking the law or routing traffic halfway around the world.

```
// ❌ Single-region: everything in us-east-1
User in Tokyo → us-east-1 → 200ms round trip
User in London → us-east-1 → 90ms round trip
us-east-1 goes down → 100% downtime for everyone

// ✅ Multi-region: traffic routes to nearest region
User in Tokyo → ap-northeast-1 → 10ms round trip
User in London → eu-west-1 → 15ms round trip
us-east-1 goes down → ap-northeast-1 still serves Asian users
```

---

## Two Strategies: Active-Passive vs Active-Active

There's no one-size-fits-all approach. The choice depends on your data model, consistency needs, and budget.

### Active-Passive (Hot-Standby)

One region handles all traffic. The other region(s) sit idle, ready to take over.

| Aspect | Details |
|--------|---------|
| **Cost** | Lower — passive regions don't need full compute |
| **Complexity** | Lower — primary region owns all writes |
| **Failover time** | Minutes (DNS propagation + region switch) |
| **Consistency** | Simple — single source of truth |
| **Read performance** | Slow for remote users (one origin) |

**Good for:** Apps where writes dominate, or where strong consistency is non-negotiable (banking, inventory).

### Active-Active (Multi-Master)

Multiple regions handle traffic simultaneously. Each region serves local users, and data syncs behind the scenes.

| Aspect | Details |
|--------|---------|
| **Cost** | Higher — every region runs full infrastructure |
| **Complexity** | Much higher — conflict resolution, eventual consistency |
| **Failover time** | Instant — traffic just shifts to other active regions |
| **Consistency** | Eventual — conflicts happen in multi-writer scenarios |
| **Read performance** | Excellent — users hit nearest region |

**Good for:** Global read-heavy apps, social media, content platforms where eventual consistency is fine.

---

## DNS Routing: Getting Users to the Right Region

Users don't know which region to hit. You need a routing layer to point them to the nearest healthy endpoint.

### GeoDNS / Latency-Based Routing

Instead of returning a single IP for `api.yourapp.com`, DNS returns the IP of the nearest region based on the user's location.

```dns
# User in Germany queries api.yourapp.com
# Route53 returns eu-west-1 IP
# Result: 25ms round trip

# User in Singapore queries api.yourapp.com
# Route53 returns ap-southeast-1 IP
# Result: 30ms round trip
```

**Health checks are critical.** If ap-southeast-1 goes down, DNS should reroute Singapore traffic to the next closest region (ap-northeast-1 or us-west-2). Without health checks, you're sending users to a dead endpoint.

### Anycast

With anycast, a single IP address is advertised from multiple regions. BGP routing naturally directs users to the nearest region.

```
// How anycast works
User in Sydney → connects to IP 1.2.3.4
  → BGP routes to closest region (ap-southeast-2)
User in Ireland → connects to same IP 1.2.3.4
  → BGP routes to closest region (eu-west-1)

// Same IP, different physical server based on geography
```

Anycast is cleaner than GeoDNS because there's no DNS caching lag. But it requires BGP control, which means owning your IP space — or using a provider like Cloudflare or AWS Global Accelerator.

---

## The Database Problem: Cross-Region Replication

Here's where things get hard. Your database lives in one region. Users in other regions need to read and write. How?

### Option 1: Primary-Replica with Read Replicas

One primary region handles writes. Other regions have read replicas.

```
us-east-1 (Primary)    eu-west-1 (Replica)    ap-southeast-1 (Replica)
    |                        |                        |
  Writes                Reads only               Reads only
    |                        |                        |
    └────────── Replication ────────────┘
```

**Pros:** Simple. Strong consistency on writes. Works with MySQL, PostgreSQL, Aurora.

**Cons:** Writes are always slow for remote users. Write to us-east-1 from Tokyo = painful. Replication lag means stale reads.

### Option 2: Multi-Region Write (Active-Active)

Multiple regions accept writes and reconcile differences. This is where you need:

- **CRDTs (Conflict-free Replicated Data Types)** — data structures that merge automatically
- **Application-level conflict resolution** — "last write wins" or custom merge logic
- **Eventually consistent databases** — DynamoDB Global Tables, CockroachDB, Cassandra

```
// ❌ LWW (Last Write Wins) disaster
User A (us-east-1): Update cart quantity to 5 at 10:00:01
User B (eu-west-1): Update cart quantity to 2 at 10:00:02

// Both regions accept the write, then sync
// LWW gives us: cart has 2 items (User B "won")
// But maybe User A just added 3 more items to their existing 2!

// ✅ Solution: Use CRDTs or application-level merging
// Operation-based CRDT: {add: [itemA, itemB, itemC]}, {remove: [itemA]}
// Merging these gives the correct result regardless of order
```

**Real talk:** Active-active databases are hard. You'll face issues like:
- Auto-increment IDs colliding (use UUIDs or Snowflake IDs)
- Cross-region latency on sync (can't cheat physics)
- Conflict resolution that's rarely perfect

---

## Caching Without Borders

A single Redis cluster won't work across regions. Here's what to do instead:

```
// Regional caches + global cache layer
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  us-east-1   │    │  eu-west-1   │    │ap-southeast-1│
│  Redis       │    │  Redis       │    │  Redis       │
│  (hot cache) │    │  (hot cache) │    │  (hot cache) │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                   Global Cache
                   (CDN + Global Datastore)
```

**Pattern: Local cache first, then regional, then global.**

1. User request hits nearest region
2. Check regional Redis — 90% hit rate for popular data
3. On miss, check global cache (CDN or global DynamoDB)
4. Still miss? Hit the primary region's database

This keeps hot data local. Cold data requires a cross-region round trip, but that's the exception, not the rule.

---

## Trade-Offs You Can't Ignore

Multi-region comes with real costs — both financial and operational.

| Trade-off | The Reality |
|-----------|-------------|
| **Cost** | You're paying for 2-3x infrastructure. Every region needs compute, storage, bandwidth. |
| **Complexity** | Your deployment pipeline, observability, and testing all multiply in complexity. |
| **Consistency** | CP or AP? You pick. Multi-region means eventual consistency or degraded writes. |
| **Debugging** | "It works in us-east-1 but not eu-west-1" — congratulations, you now have distributed debugging. |
| **Team overhead** | Someone needs to understand networking, geo-routing, cross-region replication. This isn't a side project. |

### When Multi-Region Actually Makes Sense

- **You have users on 3+ continents** — that's 100ms+ latency if you're in one region
- **You need 99.99%+ uptime** — one region can't give you that
- **Regulatory requirements** — GDPR, data sovereignty laws
- **You're already handling millions of DAU** — at this scale, minimizing latency is a revenue multiplier

### When It Doesn't

- **You haven't optimized your single-region app yet** — fix your slow queries first
- **You're serving one geographic area** — pick the closest region, you're fine
- **Your budget is tight** — multi-region doubles or triples infra costs

---

## What You Can Do Right Now

1. **Use a CDN** — this is multi-region-lite. Static assets served globally, zero infra changes on your end.

2. **Design stateless services** — if your app servers have no local state, failover between regions becomes a DNS change.

3. **Global load balancing** — Route53 latency-based routing or Cloudflare Argo. Easy setup, immediate latency improvement for reads.

4. **Read replicas in other regions** — start with read-only replicas in the regions where your users are. Writes still go to primary, but reads are fast.

5. **Run a Game Day** — simulate a region failure. Turn off your primary region. See what breaks. Fix it before it happens in production.

6. **Re-evaluate yearly** — if your users are distributed globally, there's a threshold where multi-region becomes unavoidable. Don't wait until you're scrambling during an outage.
