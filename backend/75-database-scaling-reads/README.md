# Scaling Database Reads

Your app's working fine. Tables are indexed, queries are fast, everyone's happy. Then traffic doubles. Then doubles again. Suddenly your primary database is sweating bullets just to serve SELECT queries, and writes are backing up because there's no room left to breathe.

That's the read scaling problem. And it's one of those "good problems to have" — until it's not.

## What Actually Happens Under Load

Most databases handle reads and writes through a single primary node. Every SELECT, every JOIN, every aggregation competes for the same CPU, memory, and disk I/O as your INSERTs and UPDATEs. On a busy system:

- Each read steals resources from writes
- Heavy analytical queries can stall transactional traffic
- Connection pools hit their ceiling faster
- Page cache gets thrashed by competing access patterns

The solution isn't "buy a bigger server." Not the first one, anyway. There's a whole ladder of options before you need to throw hardware at it.

---

## Strategy 1: Query Optimization (The Free Lunch)

Before you add any infrastructure, make sure your queries aren't the problem. Optimizing a bad query costs nothing and can 10x your throughput.

**What to check:**

```sql
-- ❌ Slow: Full table scan on every SELECT
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Fast: Uses index, fetches only needed columns
SELECT id, customer_id, total, created_at
FROM orders
WHERE status = 'pending'
  AND created_at > NOW() - INTERVAL '30 days';
```

**The usual suspects:**
- **Missing indexes** — run `EXPLAIN ANALYZE` and look for sequential scans
- **SELECT \*** — you're paying to transfer columns you don't use
- **N+1 queries** — ORMs love hiding these. Watch your logs.
- **Unnecessary joins** — do you really need that 6-table join on every request?

> Performance tip: Add a query log with duration thresholds. If anything takes >100ms regularly, investigate. Don't guess what's slow — measure it.

---

## Strategy 2: Read Replicas

This is the flagship strategy for scaling reads. You promote read-replica instances that asynchronously replicate from the primary and serve read-only traffic.

```
                   ┌─────────────┐
                   │   Primary   │ (writes)
                   │  (writer)   │
                   └──────┬──────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
         ┌────┴────┐ ┌───┴────┐ ┌───┴────┐
         │Replica 1│ │Replica 2│ │Replica 3│
         │ (reads) │ │ (reads) │ │ (reads) │
         └─────────┘ └─────────┘ └─────────┘
```

**How your app uses them:**

```python
# database.py - read/write splitting
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

WRITER_URL = os.environ["DATABASE_WRITER_URL"]
READER_URLS = os.environ["DATABASE_READER_URLS"].split(",")

writer_engine = create_engine(WRITER_URL, pool_size=10)
reader_engines = [create_engine(url, pool_size=20) for url in READER_URLS]
reader_index = 0

def get_writer_session():
    return Session(writer_engine)

def get_reader_session():
    global reader_index
    engine = reader_engines[reader_index % len(reader_engines)]
    reader_index += 1
    return Session(engine)

# Usage
def get_order(order_id: int):
    db = get_reader_session()
    return db.query(Order).filter(Order.id == order_id).first()

def create_order(data: dict):
    db = get_writer_session()
    order = Order(**data)
    db.add(order)
    db.commit()
    return order
```

**Trade-offs you need to know:**

| Concern | Reality |
|---------|---------|
| **Replication lag** | Replicas can be seconds behind. If you write an order then immediately read it from a replica, it might not exist yet. |
| **Stale reads** | Reports, analytics, dashboards — usually fine with stale data. User-facing "my profile" views are not. |
| **Cost** | Each replica costs money. You're trading compute for query throughput. |
| **Failover** | If the primary dies, a replica needs promotion. That's not instant. |

> **The read-your-writes problem:** The classic fix is routing the current user's own writes to the primary for 1-2 seconds after any write. Or just always serve authenticated user data from the primary and route anonymous/public views to replicas.

---

## Strategy 3: Caching (Reduce Reads at the Source)

Replicas help distribute load, but they don't reduce the total number of read requests hitting your database. Caching does.

**The caching stack:**

```
Client ──► CDN (static assets)
              │
              ▼
        Load Balancer
              │
              ▼
   ┌─────────────────────┐
   │  Application Server │
   │  (in-memory cache)  │ ◄── L1: local LRU cache
   └─────────┬───────────┘
             │
             ▼
   ┌─────────────────────┐
   │    Redis / Memcached│ ◄── L2: distributed cache
   └─────────┬───────────┘
             │
             ▼
   ┌─────────────────────┐
   │   Read Replicas     │ ◄── L3: database
   └─────────────────────┘
```

**Cache-aside pattern (most common):**

```python
import redis
import json

cache = redis.Redis(host="redis-cluster", decode_responses=True)

def get_user_profile(user_id: str):
    cache_key = f"user:profile:{user_id}"

    # Try cache first
    cached = cache.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss — hit the database
    profile = query_db("SELECT * FROM users WHERE id = %s", user_id)
    if profile:
        # Write to cache with TTL
        cache.setex(cache_key, 300, json.dumps(profile))

    return profile
```

**What to cache:**
- User profiles (TTL: 5-15 min)
- Product catalog / reference data (TTL: 1 hour+)
- Aggregated stats (TTL: 1-5 min)
- Session data (TTL: session duration)
- Configuration / feature flags (long TTL with invalidation)

> **Cache invalidation warning:** A stale cache is better than a cache stampede. When a popular key expires, 100 concurrent requests might all hit the database at once. Use a **mutex** or **probabilistic early expiration** to prevent this.

---

## Strategy 4: Connection Pool Tuning

Sometimes you don't need more database capacity — you need to use what you have more efficiently.

**The problem:** Each new database connection consumes ~2-10 MB of memory on the database side. If 100 app servers each open 50 connections, that's 5000 connections the database has to manage. Most of them are idle.

```python
# ❌ Opening a new connection per request
def get_user(user_id):
    conn = psycopg2.connect(DATABASE_URL)  # Slow!
    # ... query ...
    conn.close()

# ✅ Using a connection pool
from sqlalchemy import create_engine

engine = create_engine(
    DATABASE_URL,
    pool_size=20,        # Connections kept open
    max_overflow=10,      # Extra if under load
    pool_timeout=30,      # Wait 30s before failing
    pool_pre_ping=True,   # Check connection health
)
```

**Quick pool sizing guide:**

| App servers | Max connections per pool | Total connections |
|-------------|------------------------|-------------------|
| 2 | 25 | 50 |
| 5 | 15 | 75 |
| 10 | 10 | 100 |
| 20 | 5 | 100 |

The formula: `(CPU cores × 2) + effective_spindrive_spindles` is a rough starting point. But really — benchmark your workload and tune from there.

---

## Strategy 5: Denormalization and Materialized Views

Joins are expensive, especially at read scale. Sometimes the right answer is to pre-join your data.

**Materialized views** are database snapshots that get refreshed on a schedule:

```sql
CREATE MATERIALIZED VIEW dashboard_daily_stats AS
SELECT
    d.id AS department_id,
    d.name AS department_name,
    COUNT(e.id) AS employee_count,
    AVG(e.salary) AS avg_salary,
    SUM(p.amount) AS total_payroll
FROM departments d
JOIN employees e ON e.department_id = d.id
JOIN payroll p ON p.employee_id = e.id
WHERE p.paid_at > NOW() - INTERVAL '30 days'
GROUP BY d.id, d.name;

-- Refresh periodically
REFRESH MATERIALIZED VIEW dashboard_daily_stats;
```

This turns a query that joins 3 tables and aggregates millions of rows into a simple SELECT on a pre-computed table. Your BI dashboard goes from 30 seconds to 30 milliseconds.

**When to denormalize:**
- Read-heavy reporting queries
- Dashboard aggregations
- Data that doesn't need real-time freshness
- Complex joins with predictable access patterns

---

## Putting It All Together

Here's a realistic progression for a growing app:

**Stage 1 — One database, no replicas**
- Optimize queries, add indexes, tune pool
- Works up to ~1000 req/s on a modest instance

**Stage 2 — Single read replica**
- Route report queries and background jobs to replica
- Keeps primary free for user-facing writes
- Works up to ~5000 req/s

**Stage 3 — Multiple read replicas + Redis cache**
- 3-5 replicas behind a simple round-robin balancer
- Cache hot data (profiles, catalog, sessions)
- Works up to ~50,000 req/s

**Stage 4 — Sharded reads (beyond this article)**
- Reads are distributed across shards geographically
- Local replicas per region
- Global CDN for static data

---

## What's Next After Read Scaling

Once reads are handled, writes become the bottleneck. That's where sharding, partitioning, and CQRS come in. But that's a topic for another day.

### Takeaways

- **Optimize queries first** — it's free and often gives the biggest win
- **Add read replicas** for horizontal read scaling, but watch for replication lag
- **Layer caching** to reduce database load at every level of the stack
- **Tune connection pools** — more connections ≠ more performance
- **Denormalize** for read-heavy reporting and dashboards
- **Route smartly** — treat "read your own writes" differently from anonymous reads
