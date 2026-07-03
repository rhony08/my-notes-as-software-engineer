# Scaling Database Writes

Reading from a database is easy to scale—throw in read replicas, cache aggressively, and you're good. Writes are a different beast. Every write has to go somewhere authoritative, and that "somewhere" becomes a bottleneck fast.

Let's talk about what happens when your database can't keep up with writes, and what you can do about it.

---

## The Core Problem: One Leader

In most database setups, writes go through a single leader node. Even if you have 50 read replicas hooked up, all writes funnel through that one primary. That means:

- **CPU bottleneck** — the leader processes every INSERT, UPDATE, DELETE
- **Disk I/O bottleneck** — every write hits disk (WAL, data files, indexes)
- **Lock contention** — concurrent writes fight over row/table locks
- **Replication lag** — the leader can only ship changes as fast as it can write them

Here's what this looks like in practice:

```sql
-- Single leader setup — ALL writes go here
INSERT INTO orders (user_id, total, items) VALUES (42, 299.99, 3);
-- That's one row, on one machine, fighting for disk with everyone else
```

At some point, throwing more CPU at your single leader stops helping. That's when you need to scale writes.

---

## Strategy 1: Sharding (Horizontal Partitioning)

Sharding splits your data across multiple database instances. Each shard is responsible for a subset of data, so write load distributes.

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Shard A │  │ Shard B │  │ Shard C │
│ users   │  │ users   │  │ users   │
│ 0-10000 │  │10001-   │  │20001-   │
│         │  │ 20000   │  │ 30000   │
└─────────┘  └─────────┘  └─────────┘
    ↑             ↑             ↑
    └─────────────┼─────────────┘
               App Layer
          (routes by user_id hash)
```

### How it works

You pick a shard key (user_id, order_id, tenant_id), hash it, and route writes to the right shard.

```python
def get_shard(user_id: int) -> int:
    return user_id % NUM_SHARDS  # Simple modulo

def write_order(user_id: int, order_data: dict):
    shard = get_shard(user_id)
    db = get_shard_connection(shard)
    db.execute("INSERT INTO orders VALUES (%s)", order_data)
```

### ❌ What goes wrong

```python
# ❌ DON'T use a bad shard key
def get_shard(timestamp):
    return timestamp.hour  # All writes at 2PM hit shard 14, others are idle

# ✅ DO use something well-distributed
def get_shard(user_id):
    return hash(user_id) % NUM_SHARDS  # Even distribution
```

### The trade-offs

| Pro | Con |
|-----|-----|
| Writes scale linearly with shard count | Cross-shard queries are painful—joining data across shards means app-level work |
| Each shard is smaller, so indexes stay fast | Rebalancing shards when you add capacity is complex |
| Isolation—one noisy shard doesn't take down the whole system | Transactions can't span shards easily |

**When it works best:** Multi-tenant systems, user-centric data (orders, profiles, messages). Each user's data stays on one shard.

---

## Strategy 2: Batch Writes

Instead of writing one row at a time, buffer writes and flush them in batches. This is huge for throughput.

```sql
-- ❌ One row per round-trip
INSERT INTO logs (level, message, timestamp) VALUES ('INFO', 'Request processed', NOW());
INSERT INTO logs (level, message, timestamp) VALUES ('ERROR', 'Timeout exceeded', NOW());
-- 2 round-trips, 2 transactions

-- ✅ Batch insert
INSERT INTO logs (level, message, timestamp) VALUES 
  ('INFO', 'Request processed', NOW()),
  ('ERROR', 'Timeout exceeded', NOW()),
  ('WARN', 'Slow query detected', NOW());
-- 1 round-trip, 1 transaction, 3x the throughput
```

In application code:

```python
class LogBatcher:
    def __init__(self, batch_size=100):
        self.buffer = []
        self.batch_size = batch_size

    def write(self, entry):
        self.buffer.append(entry)
        if len(self.buffer) >= self.batch_size:
            self.flush()

    def flush(self):
        if not self.buffer:
            return
        db.execute("INSERT INTO logs VALUES %s", self.buffer)
        self.buffer = []
```

The trade-off: you're trading **durability** for **throughput**. If the app crashes between flushes, you lose buffered data. That's fine for analytics logs, not for payment transactions.

---

## Strategy 3: Async Writes (Queue-Offload)

Don't write to the database during the HTTP request. Push to a queue and let workers handle the writes.

```
Client → API → Queue (Redis/Kafka) → Worker → Database
```

This decouples the write from the request lifecycle:

```python
@app.post("/orders")
def create_order(order_data):
    # ❌ Synchronous write — request waits for DB
    order = db.execute("INSERT INTO orders VALUES (%s)", order_data)
    return {"order_id": order.id}

@app.post("/orders")
def create_order(order_data):
    # ✅ Async write — queue and return immediately
    queue.push("order_created", order_data)
    return {"status": "queued", "order_id": order_data["id"]}
```

### Why this matters

- **Peak load handling** — the queue acts as a shock absorber. Write spikes don't crash the DB; they just build up in the queue
- **Worker scaling** — add more workers when the queue grows
- **Retry-friendly** — failed writes go to a DLQ instead of returning 500s to clients

### But there's a catch

> **You lose read-after-write consistency.** The client gets "order created" but a subsequent read from a replica might not see it yet. If your business logic depends on immediate visibility, this approach needs careful design (like returning a correlation ID that workers can use to update the client).

---

## Strategy 4: Write-Behind Cache

Cache writes in Redis/Memcached and persist them to the database asynchronously.

```
Write → Redis (fast write, TTL) → Background sync → Database
```

```python
# Write to cache, then asynchronously flush to DB
def write_user_session(user_id, session_data):
    redis.set(f"session:{user_id}", session_data, ttl=3600)
    # Background worker picks this up
    queue.push("persist_session", {"user_id": user_id, "data": session_data})
```

### ❌ What can go wrong

```
1. User creates session → saved to Redis ✅
2. Redis node dies before sync → session lost ❌
3. User tries to use session → not in Redis, not in DB → error
```

**Mitigations:**
- Write-through (sync to DB first, then cache) for critical data
- Accept data loss for non-critical data (sessions are less critical than payments)
- Use Redis with AOF persistence to reduce the window

---

## Strategy 5: CQRS (Command Query Responsibility Segregation)

Separate the write model from the read model entirely. Writes go to one data store, reads go to another.

```
Write API → Write Database (normalized, optimized for writes)
                                                    ↓
                                              Event/Sync
                                                    ↓
Read API  → Read Database (denormalized, optimized for queries)
```

```python
# Command (write)
class OrderCommandHandler:
    def create_order(self, command):
        order = Order.create(command.data)
        event_store.save(OrderCreatedEvent(order.id, command.data))
        return order.id

# Query (read — eventually consistent)
class OrderQueryHandler:
    def get_order(self, query):
        return read_db.execute(
            "SELECT * FROM order_summaries WHERE id = %s", query.order_id
        )
```

### The trade-off

CQRS is **powerful but complex**. You're now running two databases, keeping them in sync, handling eventual consistency. Don't reach for CQRS unless you genuinely have different read and write patterns that are hard to optimize within one store.

**Good fit:** Reporting systems, audit trails, analytics platforms where writes are append-only and reads are complex aggregations.

---

## Strategy 6: Columnar Databases for Analytics Writes

If you're doing analytics writes (time-series, logs, events), columnar databases like ClickHouse or TimescaleDB handle high-ingest writes better than PostgreSQL for this use case.

```sql
-- ClickHouse — optimized for high-frequency inserts
INSERT INTO events (timestamp, event_type, user_id, properties) VALUES
  (NOW(), 'page_view', 42, '{"page": "/pricing"}'),
  (NOW(), 'click', 73, '{"button": "signup"}');
-- Millions of rows/second on modest hardware
```

PostgreSQL rows are stored row-by-row (all columns together). ClickHouse stores column-by-column. For writes, this means:

- **Row stores** (Postgres, MySQL): Great for row-level writes, but the WAL + B-tree index maintenance limits throughput
- **Column stores** (ClickHouse, Redshift): Optimized for batch appends. Each insert is just appending to column files

---

## Which Strategy Should You Use?

Here's a practical decision tree:

| Scenario | Approach |
|----------|----------|
| Your single DB can't keep up with INSERT volume | **Shard** — distribute writes across instances |
| Write spikes during peak hours | **Async queue** — buffer spikes, process steadily |
| Analytics/event data, millions of rows/day | **Columnar DB** — ClickHouse or TimescaleDB |
| Logging or audit trails (can tolerate loss) | **Write-behind cache** — batch + async |
| Different read/write patterns hurting both | **CQRS** — separate stores for each concern |
| High-volume event logging | **Batch writes** — 100 rows at a time, not 1 |

---

## Takeaways

- **Read replicas don't help with writes** — you need to solve write scaling separately
- **Sharding is the most common approach** but requires careful key selection and makes cross-shard operations painful
- **Async writes improve throughput but sacrifice consistency** — know your business requirements before choosing
- **Batch writes are the simplest improvement** — even 10-row batches can be 5-10x faster than individual inserts
- **CQRS and columnar stores are specialized tools** — great when you need them, overkill otherwise
- **Always measure first** — your bottleneck might be index maintenance, not INSERT throughput. Profile before scaling
