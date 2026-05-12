# Read Replicas: Scaling Database Reads

Too many slow reads can make your app feel like it's crawling — users complain, pages time out, and your primary DB becomes a choke point. Read replicas are a common way to scale read-heavy workloads, but they're not a magic bullet. We'll look at what they solve, how they behave, and practical patterns you can use today.

Why this matters

- If your primary database is doing both reads and writes, heavy analytic queries or cache misses can steal CPU and I/O from critical transactional work.
- Read replicas let you offload non-critical reads (reports, dashboards, some API endpoints) so the primary can focus on writes.

How read replicas actually work (short)

- The primary accepts writes and ships changes to replicas via replication (streaming WAL, binlog, etc.).
- Replicas apply those changes and serve reads.
- Replication is usually asynchronous: replicas lag behind the primary by some milliseconds (or more under load).

❌ vs ✅ — quick comparison

- ❌ Single primary doing reads+writes: simplest, but you'll hit limits fast.
- ✅ Primary + read replicas: better read throughput, but adds complexity (lag, routing, failover).

Common approaches and examples

1) Simple routing (app-level)

- Pattern: Send all writes to primary. Send reads to a replica pool.
- Use-case: Non-critical reads like reporting, user timelines where eventual consistency is okay.

Example (pseudo-JavaScript):

```js
// ❌ DON'T route strong-consistency reads here
const writeDb = getPrimaryDb();
const readPool = getReadReplicaPool();

// Safe write
await writeDb.query('INSERT INTO orders ...');

// Read that can tolerate lag
const recentOrders = await readPool.query('SELECT * FROM orders ORDER BY created_at DESC LIMIT 20');
```

Why this works: it's simple, low-op cost. Trade-offs: eventual consistency — a row you just inserted might not appear on replicas immediately.

2) Fallback reads for low-latency reads

- Pattern: Try replica(s) first, on consistent-required paths fallback to primary if replica returns older data or empty.
- Use-case: Search pages where stale results are tolerable but completeness matters.

Example flow:

- Query replica
- If row not found OR timestamp < threshold → query primary

```js
const r = await replica.query('SELECT * FROM product WHERE id=$1', [id]);
if (!r.rowCount || r.rows[0].updated_at < Date.now() - 2000) {
  // replica is stale, fetch from primary
  const p = await primary.query('SELECT * FROM product WHERE id=$1', [id]);
  return p.rows[0];
}
return r.rows[0];
```

3) Session-level consistency window

- Pattern: If a user performs a write and expects to see the change immediately (e.g., after creating a post), route subsequent reads for that session to primary for a short TTL (e.g., 1–5s).
- Use-case: UX flows where immediate visibility matters.

Implementation note:

- Store a short-lived flag in session cache (Redis) saying "stick-to-primary-until: ts". Your read layer checks this flag.

Trade-offs and pitfalls

- Replication lag
  - Under heavy write or network issues, lag can spike from milliseconds to seconds. If you rely on replicas for critical reads, you'll expose stale data.
- Schema changes
  - Some DDLs block or slow replication. Plan migrations carefully: rolling schema changes, read-only windows, or tools that coordinate migrations across replicas.
- Failover complexity
  - Promoting a replica to primary is possible, but you need to handle in-flight writes, and clients may have cached DNS/ip for the old primary.
- Write amplification
  - Some architectures attempt to replicate writes differently; be mindful of WAL generation and disk I/O.

Operational checklist

- Monitor replication lag (seconds, not just error rates).
- Tag which queries are safe for replicas.
- Add health checks for replicas and remove unhealthy nodes from the read pool.
- Run migrations with a plan that tolerates replicas (avoid destructive DDLs without coordination).

Practical Postgres example (PG logical/streaming replication)

- Primary streams WAL to replicas via replication slots.
- On replicas: configure hot_standby = on to allow reads.
- Monitor pg_stat_replication and replay_lag.

Sample check (psql):

```sql
-- On primary
SELECT client_addr, state, sync_state, sent_lsn, write_lsn, replay_lsn,
  (pg_current_wal_lsn() - replay_lsn) AS wal_lag_bytes
FROM pg_stat_replication;
```

Actionable takeaways

- Don't send every read to replicas by default. Classify reads: tolerant vs strict-consistency.
- Implement a session-stickiness fallback for user flows that need immediate visibility.
- Monitor lag and fail fast to primary if replicas are unhealthy or lagging.
- Plan schema changes with replication in mind: avoid blocking DDL or use online-migration tools.
- Keep a small runbook: how to promote a replica, DNS updates, and how to re-sync a broken replica.

References & further reading

- PostgreSQL docs: replication
- Cloud provider guides (RDS, Cloud SQL) for managed replica behavior



