# Connection Pool Tuning

Your app is fine in staging. Then production traffic hits, and suddenly every other request takes 5 seconds, the database CPU graph looks like a heartbeat monitor, and the error log is a wall of `timeout: pool is full` / `Connection refused`. You scale up the database. It doesn't help. You scale up the app. It gets *worse*.

This is the connection pool paradox: the thing that's supposed to make your database usage efficient is what's drowning your database. And the fix is rarely "more connections" — it's understanding what the pool is actually doing, and what your database can actually handle. Let's dig into how pools behave under load, how to size them without guessing, and why the default config from your framework's tutorial is probably wrong for your traffic.

## What the Pool Actually Does

A connection pool keeps a set of open database connections alive and hands them out to requests. The point isn't just speed (though opening a Postgres connection costs a few milliseconds and a round trip). The real point is **bounding resource usage**: instead of every request opening its own connection (unbounded, chaotic), you have a fixed number of connections that get reused.

But here's what people miss: the pool is a *queue* in disguise. When all connections are busy, new requests wait. The pool doesn't make your database faster — it makes the waiting *orderly*.

```
Request comes in
      │
      ▼
┌─────────────────┐
│   Pool (idle)   │◄── check out a connection
└─────────────────┘
      │ busy
      ▼
┌─────────────────┐
│  Waiting queue  │── request waits (pool timeout)
└─────────────────┘
      │ timeout
      ▼
   500 error
```

The number you tune — `max_pool_size` — is a trade-off between **latency** (how long requests queue) and **database load** (how many concurrent queries hammer Postgres). More connections = less waiting, but more contention.

## The Default Is a Trap

Every framework ships with a default max pool size, and it's almost always "number of CPUs" or some arbitrary constant. Those defaults were chosen to be *safe*, not *right*.

```js
// ❌ Copy-pasted from a tutorial — 10 connections per instance
const pool = new Pool({ max: 10 });

// ✅ Sized for your actual workload
const pool = new Pool({
  max: 5,        // deliberate, see below
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 2_000,  // fail fast instead of hanging
});
```

The mistake compounds when you run multiple app instances. Say you have 10 instances, each with `max: 10` → 100 connections to a database that comfortably handles 20. The database spends more time context-switching between connections than executing queries, and every query gets slower for everyone. This is the "I scaled my app and the DB got slower" story.

**Rule of thumb that actually works:** pick a per-instance max, then multiply by instance count, and keep the total *under* what your database can handle. For Postgres, a decent starting point is `(cores × 2) + 1` for the *entire* fleet — Postgres's own docs suggest this because each connection is a process with ~5MB of memory and its own cache. For MySQL it's threads, similar math.

## Sizing: The Only Formula That Matters

Everyone asks "what should my pool size be?" The honest answer: **it depends on your query latency, not your traffic.** Here's the reasoning:

**Latency math:** If your average query takes 50ms, one connection can serve ~20 queries/second. Want 200 QPS? You need at least 10 connections — if all queries are serial on one connection, 200 requests × 50ms = 10 seconds of queueing.

```
Required connections ≈ QPS × average query latency (in seconds)
```

But — and this is the trap — that formula assumes queries are short and uniform. Real workloads have slow outliers:

```sql
-- This monster ties up a connection for 2+ seconds
SELECT * FROM analytics_events
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 1000;  -- no index on (user_id, created_at) → seq scan
```

One slow query every few seconds is fine. Ten slow queries at once will saturate a pool sized for the *average* — because the pool can't distinguish "fast query" from "slow query." It just hands out connections.

**The practical approach:**

1. Start with the latency formula as a lower bound.
2. Add headroom for slow queries (or fix the slow queries — usually the better move).
3. Load test with production-like traffic and watch **pool wait time** and **queue depth**.
4. If requests are waiting in the pool, you have two options: add connections *or* reduce query time. Adding connections is the lazy fix that works until it doesn't.

## The Metrics That Tell the Truth

You can't tune what you can't see. Before touching config, get these on a dashboard:

| Metric | What it tells you | Healthy sign |
|---|---|---|
| `pool.wait_time` | How long requests queue for a connection | Near zero; rising = pool too small |
| `pool.usage` / active connections | How much of the pool is in use | Spiky is normal; pegged = saturated |
| `db.connection_utilization` | Database-side connection count vs limit | < 80% of DB max |
| `db.query_latency` (p50/p95/p99) | Actual query cost | Flat under load |
| `db.connections` (server side) | Total connections from all instances | Stable, not climbing |

The key insight: **if pool usage is pegged but DB utilization is low, your pool is too small for your concurrency.** If DB utilization is pegged, your pool is too big (or your queries too slow) — adding more connections will make it worse.

```js
// Node (pg): expose pool stats cheaply
app.get("/metrics/pool", (req, res) => {
  res.json({
    total: pool.totalCount,
    idle: pool.idleCount,
    waiting: pool.waitingCount,   // ← this is the number to watch
  });
});
```

## ❌ vs ✅: Patterns That Break or Save You

### 1. Checking out a connection and never returning it

```js
// ❌ Forgetting to release — pool leaks until exhaustion
app.get("/orders", async (req, res) => {
  const client = await pool.connect();
  const result = await client.query("SELECT * FROM orders");
  res.json(result.rows);
  // client.release() never called → connection stuck "checked out" forever
});

// ✅ Always release, even on errors
app.get("/orders", async (req, res) => {
  const client = await pool.connect();
  try {
    const result = await client.query("SELECT * FROM orders");
    res.json(result.rows);
  } finally {
    client.release();  // runs on success AND error
  }
});
```

### 2. Transactions that hold connections too long

```js
// ❌ Transaction wrapping a slow external call — connection held hostage
await client.query("BEGIN");
const payment = await callPaymentGateway(seconds: 3);  // 3s holding a connection
await client.query("COMMIT");
```

A pool of 10 with 10 concurrent payments = 10 connections held for 3+ seconds = total stall for everything else. **Never hold a connection across I/O that isn't the database.** Do the external call first, then open the transaction.

### 3. Per-request pools (the "connection storm")

```js
// ❌ Creating a pool inside the request handler — connection churn
app.get("/users", async (req, res) => {
  const pool = new Pool(DB_CONFIG);   // new pool per request!
  const result = await pool.query("SELECT * FROM users");
  await pool.end();                    // close everything per request
  res.json(result.rows);
});
```

This defeats the entire purpose of pooling. You're back to open/close per request, plus the overhead of pool bookkeeping. One pool per process, created at startup, shared everywhere.

## Connection Timeout: Fail Fast, Fail Loud

A subtle killer: the pool timeout. If `connectionTimeoutMillis` is 0 (the default in some drivers), a request that can't get a connection waits *forever*. Under a database outage, every request piles up waiting, and your app's memory and thread count balloon. Then when the DB comes back, you get a thundering herd of thousands of queued requests all trying to connect at once.

```js
// ✅ Fail fast, let the caller decide what to do
const pool = new Pool({
  max: 10,
  connectionTimeoutMillis: 2000,  // give up after 2s
});

// ✅ Handle it gracefully at the API layer
try {
  const result = await pool.query("SELECT 1");
} catch (err) {
  if (err.code === "57P01" || err.message.includes("timeout")) {
    // DB is down or pool saturated — return 503, not a 500
    return res.status(503).json({ error: "service unavailable, retry later" });
  }
  throw err;
}
```

The 503 matters: clients and load balancers treat 503s as retryable, while 500s get logged as bugs. Same failure, much kinder downstream behavior.

## Pooling Layers: App Pool vs PgBouncer

Here's a pattern that saves entire teams: **run two pools.** A small pool in your app (for latency) and a connection multiplexer like PgBouncer in front of Postgres (for scale). PgBouncer lets thousands of app connections map to a handful of real database connections, which is how you run 50 app instances against a modest Postgres without melting it.

```
App instances ──► PgBouncer ──► Postgres
   50 × pool(10)      │            max 100
   = 500 connections  │            (only ~50 real ones)
                      ▼
              transaction mode
```

The trade-off: PgBouncer in transaction mode doesn't support session-level features (like `LISTEN/NOTIFY` or session temp tables), and it's another component to operate. For a single app instance, it's overkill — the app pool is enough. The moment you run many instances and your DB says `too many connections`, PgBouncer (or its managed equivalent like RDS Proxy) is the move.

## The "More Connections" Temptation

When things get slow, the instinct is to bump `max` from 10 to 50. Sometimes that's right — if the pool is genuinely saturated and queries are fast, more connections = more throughput. But here's what happens when you keep going:

- Postgres forks a process per connection → memory grows, cache efficiency drops
- More concurrent queries → more lock contention, more I/O thrash
- Each connection competes for the same disk and buffer cache
- p99 latency climbs even as throughput stays flat

At some point, throughput *drops* as you add connections. This is the "hockey stick" of pool sizing — and it's why the formula-based approach beats "add more until it stops helping."

**The rule:** if your pool is saturated *and* queries are fast, add connections. If your pool is saturated *and* queries are slow, fix the queries — more connections just means more slow queries running at once.

## Takeaways

- **Size the pool for query latency, not traffic:** `connections ≈ QPS × avg query seconds`, then multiply by instance count and sanity-check against what your DB can handle.
- **Watch the queue, not the pool:** `waitingCount` / pool wait time rising is the earliest sign of saturation — before errors appear.
- **Always release connections** — `finally` blocks, not afterthoughts. One leaked connection per request kills the pool in minutes.
- **Never hold a connection across non-DB I/O** — external calls, slow business logic, and long transactions are connection hostages.
- **Set a connection timeout** and return 503s (retryable) instead of 500s when the pool is exhausted.
- **Reject per-request pools** — one pool per process, created at startup.
- **When many instances overwhelm the DB, multiplex** with PgBouncer/RDS Proxy instead of raising per-instance maxes.
- **If queries are slow, adding connections makes it worse.** Fix the query, then size the pool.

The mental model that fixes most pool issues: the pool doesn't make your database faster, it makes waiting orderly — and the only way to make waiting shorter is to size the pool to your real concurrency and keep queries fast.
