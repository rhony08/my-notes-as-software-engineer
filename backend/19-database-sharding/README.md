# Database Sharding: Scaling Beyond Single Node

Your database is getting slow. Queries that used to take 10ms now take 500ms. You've added indexes, tuned queries, and thrown more RAM at the server. But the data keeps growing—millions of rows becoming billions—and one machine just can't keep up anymore.

That's when sharding enters the conversation. It's not a magic bullet, and it comes with real costs. But when you've exhausted vertical scaling options, sharding is how you keep growing.

## What Is Sharding, Really?

Sharding is horizontal partitioning across multiple database instances. Each "shard" holds a subset of your data, and together they form your complete dataset.

```
┌─────────────────────────────────────────┐
│           Before: Single Node            │
│                                         │
│    [ All 100M users in one database ]   │
│              ↓                          │
│         Queries slow                    │
│         Writes bottlenecked             │
│         One point of failure            │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│           After: Sharded                 │
│                                         │
│  Shard 1    Shard 2    Shard 3           │
│  [1-33M]    [34-66M]   [67-100M]         │
│     ↓          ↓          ↓              │
│  Fast reads, parallel writes            │
│  No single point of failure             │
└─────────────────────────────────────────┘
```

The key difference from regular partitioning: shards are **separate database instances**, not just tables on the same server.

## The Shard Key: Make or Break Decision

Your shard key determines where each row lives. Pick the wrong one, and you'll create more problems than you solve.

### What Makes a Good Shard Key?

- **High cardinality** — many distinct values to distribute evenly
- **Uniform distribution** — no hotspots where one shard gets hammered
- **Query-aligned** — most queries include the shard key, so you don't have to query all shards
- **Immutable** — changing a row's shard key means moving data between shards

### Common Shard Key Strategies

| Strategy | How It Works | Pros | Cons |
|----------|--------------|------|------|
| **Hash-based** | `shard = hash(user_id) % num_shards` | Even distribution | Can't range query |
| **Range-based** | Shard 1: users 1-1M, Shard 2: 1M-2M | Efficient range queries | Hotspots (new users go to last shard) |
| **Directory-based** | Lookup table maps keys to shards | Flexible, supports re-sharding | Extra lookup, single point of failure |
| **Geography** | US users → US shard, EU users → EU shard | Latency, data sovereignty | Uneven distribution |

### Hash-Based Sharding Example

```python
# ❌ BAD: Simple modulo causes problems when adding shards
def get_shard(user_id):
    return user_id % 4  # Shards 0, 1, 2, 3

# When you add shard 4, MOST users move → massive rebalancing


# ✅ GOOD: Consistent hashing minimizes movement
import hashlib

def get_shard(user_id, num_shards=4):
    # Hash the key and map to ring position
    hash_val = int(hashlib.md5(str(user_id).encode()).hexdigest(), 16)
    return hash_val % num_shards

# Adding a shard? Only ~1/num_shards of data moves
```

Consistent hashing is your friend here. Instead of `key % n`, you hash both the key and the shard onto a ring. Adding or removing shards only affects adjacent data.

## Real-World Sharding Patterns

### User-Centric Sharding (Most Common)

```sql
-- All of a user's data lives on the same shard
-- Queries are fast because you hit one shard

-- ❌ BAD: Querying without shard key hits ALL shards
SELECT * FROM orders WHERE created_at > '2024-01-01';
-- This queries every shard → slow, defeats the purpose

-- ✅ GOOD: Include shard key in query
SELECT * FROM orders 
WHERE user_id = 12345 AND created_at > '2024-01-01';
-- Hits only shard 2, much faster
```

This is why most successful sharding strategies are **user-centric**. A user's data stays together, and most queries are per-user.

### Multi-Tenant Sharding

```
┌────────────────────────────────────────┐
│  Tenant A → Shard 1 (dedicated)         │
│  Tenant B → Shard 2 (dedicated)         │
│  Tenants C,D,E → Shard 3 (shared, small)│
│  Tenants F,G,H → Shard 4 (shared, small)│
└────────────────────────────────────────┘
```

Big tenants get their own shards. Small tenants share. You can even tier by pricing.

## The Trade-offs Nobody Tells You About

Sharding isn't free. Here's what you give up:

### 1. Cross-Shard Queries Become Painful

```sql
-- ❌ This query now has to hit EVERY shard
SELECT COUNT(*) FROM users WHERE created_at > '2024-01-01';

-- Then aggregate the results in your application
-- Or maintain a separate analytics database
```

You often need:
- Pre-computed aggregates
- Separate analytics databases
- Application-level joins

### 2. Transactions Get Complicated

```python
# ❌ Before sharding: easy atomic transaction
with db.transaction():
    user.balance -= 100
    merchant.balance += 100
    # Both updates are atomic


# ✅ After sharding: distributed transaction (expensive!)
# User on Shard 1, merchant on Shard 2
# You need two-phase commit or eventual consistency
```

Most sharded systems move toward **eventual consistency** and **compensating transactions** rather than distributed ACID transactions.

### 3. Rebalancing Is Hard

When one shard gets too big, you need to split it. This means:
- Downtime (if not done carefully)
- Data migration while serving traffic
- Updating your shard directory

### 4. Operational Complexity

```bash
# Before sharding: one database to back up, monitor, upgrade
# After sharding: manage n databases

# Rolling upgrades become tricky
# Monitoring needs aggregation
# Backups must be coordinated
# Failover is per-shard
```

## When Should You Shard?

Don't shard prematurely. Consider it when:

- **Data size exceeds single-node capacity** — tables with billions of rows
- **Write throughput is bottlenecked** — single master can't keep up
- **Query latency is unacceptable** — despite optimization
- **You need geographic distribution** — data sovereignty or latency requirements

**You probably DON'T need sharding if:**
- Your data fits in memory with room to grow
- Your queries are fast enough
- You can solve the problem with read replicas
- A bigger machine costs less than the engineering time to shard

## Sharding in the Wild

### How Big Companies Do It

| Company | Approach | Scale |
|---------|----------|-------|
| **Instagram** | User ID hash-based, PostgreSQL | 500M+ users |
| **Uber** | City-based sharding, Cassandra + Postgres | 100M+ trips/day |
| **Discord** | Guild (server) ID sharding, ScyllaDB | Billions of messages |
| **Shopify** | Shop ID sharding, MySQL | Millions of shops |

Common pattern: **shard by the primary entity in your domain** (user, shop, guild, city).

### Managed Sharding Options

If you don't want to build it yourself:

```
┌─────────────────────────────────────────────────┐
│ Service          │ Sharding Support             │
├─────────────────────────────────────────────────┤
│ Amazon Aurora    │ Automatic horizontal scaling │
│ Google Spanner   │ Auto-sharding, global        │
│ Azure SQL        │ Elastic pool, shard maps     │
│ Citus (Postgres) │ Extension for sharding       │
│ Vitess (MySQL)   │ Sharding proxy layer         │
│ MongoDB          │ Built-in sharding            │
│ Cassandra        │ Partition key = shard key    │
└─────────────────────────────────────────────────┘
```

## Practical Sharding Checklist

Before you shard:

- [ ] **Profile your data** — which tables are biggest?
- [ ] **Profile your queries** — what's the primary access pattern?
- [ ] **Pick a shard key** — usually the primary entity (user_id, shop_id)
- [ ] **Plan for cross-shard queries** — denormalize or accept limitations
- [ ] **Decide on transaction strategy** — eventual consistency? Saga pattern?
- [ ] **Test with realistic data volumes** — sharding bugs are painful to fix in production
- [ ] **Have a rebalancing plan** — you'll need it eventually

## Quick Reference: Sharding Decision Matrix

```
┌─────────────────────────────────────────────────────────┐
│ Situation                    │ Recommendation           │
├─────────────────────────────────────────────────────────┤
│ < 10M rows                   │ Don't shard. Optimize.   │
│ 10M-100M rows, reads slow    │ Read replicas first.     │
│ 100M+ rows, writes slow      │ Consider sharding.       │
│ Multi-tenant SaaS            │ Tenant-based sharding.   │
│ Global users, latency issues │ Geo-sharding.            │
│ Need ACID across shards      │ Think twice. Really.     │
└─────────────────────────────────────────────────────────┘
```

---

Sharding is an architectural commitment. Once you shard, migrating away is expensive. Start with the question: "Have I exhausted simpler options?" If the answer is yes, and your growth trajectory demands it, shard with your eyes open to the trade-offs.