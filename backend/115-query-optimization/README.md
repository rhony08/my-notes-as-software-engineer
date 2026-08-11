# Database Query Optimization

Your endpoint was fine in staging. 40ms, no complaints. Then you shipped, real traffic hit, and the same query is taking 4 seconds in production — and it's the one behind your dashboard's main page. You bump the instance size, it gets better for a week, then it's slow again. Sound familiar?

Here's the uncomfortable truth: most slow queries aren't slow because of hardware. They're slow because of how they're written — or how the data around them is structured. Before you throw money (or more CPUs) at the problem, learn to read what the database is actually doing. That's what this note is about: the process of finding a slow query, understanding *why* it's slow, and fixing the root cause instead of the symptom.

## Start With the Query Plan, Not the Query

The single most valuable skill in query optimization is reading an `EXPLAIN` output. It's the database telling you exactly how it plans to execute your query — which tables it scans, which indexes it uses (or ignores), how many rows it *thinks* it'll touch, and where the expensive parts are.

```sql
-- PostgreSQL. The ANALYZE actually runs the query, so be careful on prod data
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

```text
Limit  (cost=0.43..12.55 rows=20 width=120) (actual time=0.051..0.253 rows=20 loops=1)
  ->  Index Scan Backward using orders_user_created_idx on orders  (cost=0.43..12.55 rows=20 width=120)
        Index Cond: (user_id = 42)
```

Read it like this:

- **`Seq Scan` (full table scan)** — the database is reading every row. Fine for a 100-row table, a disaster at 10 million rows.
- **`Index Scan` / `Index Only Scan`** — it's using an index. Good. `Index Only Scan` means it never even touches the table — even better.
- **`rows=...` vs `actual rows=...`** — the estimate vs reality. Big gaps here mean stale statistics, which means `ANALYZE`/`VACUUM` (or equivalent) needs to run.
- **`actual time`** — where the milliseconds actually go. That's your bottleneck map.

If you see `Seq Scan` on a big table, that's usually your answer. The query isn't slow because it's complex — it's slow because the database is reading 10 million rows to find 20.

## The Usual Suspects

Most slow queries in real apps are one (or more) of these five things.

### 1. Selecting Columns You Don't Need

```sql
-- ❌ Pulls every column, including any blobs/text you stored
SELECT * FROM users WHERE email = 'x@example.com';

-- ✅ Only what the app actually uses
SELECT id, name, email FROM users WHERE email = 'x@example.com';
```

`SELECT *` looks lazy, not harmful. But it becomes harmful when: the table has wide columns (JSON blobs, long text), the query runs in a hot path, or — the sneaky one — the table gets a new column later and your "cheap" query suddenly transfers way more data than before. Also, the moment you have a *covering index* (more on that below), `SELECT *` silently destroys its benefit because the DB has to go back to the table for the extra columns.

### 2. Functions on Indexed Columns (Non-Sargable Queries)

This is the classic one that confuses everyone, because the query *looks* indexed:

```sql
-- ❌ This cannot use an index on created_at. The DB must compute DATE() for every row
SELECT * FROM orders WHERE DATE(created_at) = '2026-08-11';

-- ✅ Range condition — index-friendly, and actually equivalent
SELECT * FROM orders
WHERE created_at >= '2026-08-11 00:00:00'
  AND created_at <  '2026-08-12 00:00:00';

-- ❌ Also kills index usage — same problem, different disguise
SELECT * FROM users WHERE LOWER(email) = 'x@example.com';

-- ✅ If you need case-insensitive lookup, index the expression itself
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'x@example.com';
```

The rule: **when you wrap a column in a function, the index on the raw column is useless** — the database would have to compute the function for every row before it can compare. Either rewrite the query as a range, or create an index on the *expression*.

### 3. The N+1 Problem

Not strictly a SQL problem — it's an application problem that shows up as thousands of tiny queries. Classic ORM code:

```python
# ❌ 1 query for users + 1 query PER USER for their orders = N+1 queries
users = User.objects.all()
for user in users:
    orders = user.orders.all()  # fires a query every iteration
    print(user.name, len(orders))
```

With 500 users, that's 501 round trips. Each one might be 1ms locally, but network latency makes it 1ms + ~0.5ms per round trip, and now your "simple" loop takes seconds.

```python
# ✅ 2 queries total: one for users, one for all their orders, joined in memory
from django.db.models import Prefetch

users = User.objects.prefetch_related(
    Prefetch('orders', queryset=Order.objects.all())
).all()
```

The same idea exists in every ORM (eager loading in Rails, `Include` in EF Core, `JOIN FETCH` in JPA). And if you're writing raw SQL, the equivalent mistake is running a query in a loop that could be a single `JOIN` or `IN` clause.

### 4. Implicit Type Coercion

The column is a `varchar`, the parameter is a number — or vice versa. The database silently casts, which often means it can't use the index:

```sql
-- ❌ column is varchar, param is int — DB casts every row, index ignored
SELECT * FROM customers WHERE phone = 123456789;

-- ✅ match the type, index works
SELECT * FROM customers WHERE phone = '123456789';
```

If you can't change the query (it comes from a legacy client), at least know this is happening — it shows up in `EXPLAIN` as a `Seq Scan` with a cast node.

### 5. Leading Wildcards

```sql
-- ❌ Can't use a regular B-tree index — must scan everything
SELECT * FROM products WHERE name LIKE '%iphone%';

-- ✅ prefix match uses the index
SELECT * FROM products WHERE name LIKE 'iphone%';
```

Full-text search is the real answer for `%needle%` searches on text (PostgreSQL `tsvector`, or a dedicated search engine). A `LIKE '%x%'` on a big table is a full scan, no matter how many indexes you add.

## Indexes: The Good, The Bad, and The Ordering

Indexes are the first thing people reach for, so let's be precise about them.

### Composite Index Column Order Matters

For `WHERE user_id = ? ORDER BY created_at DESC`, a composite index on `(user_id, created_at)` is perfect — the DB finds the user's rows via the first column, and the second column already provides the sort order, so no sort step. But swap the columns and it falls apart:

```sql
-- ✅ Index that serves this exact query
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at);

-- ❌ The reverse order helps queries filtering by created_at, not by user_id
CREATE INDEX idx_orders_created_user ON orders (created_at, user_id);
```

Rule of thumb: **equality columns first, then range/sort columns.** `WHERE user_id = ? AND status = ? ORDER BY created_at` → index `(user_id, status, created_at)`.

### Covering Indexes

An index that contains *all* the columns a query needs. The database answers the query entirely from the index — no trip back to the table:

```sql
-- Query only needs these three columns
CREATE INDEX idx_users_email_cover ON users (email) INCLUDE (name, avatar_url);

-- Now this is an Index Only Scan — index alone answers it
SELECT name, avatar_url FROM users WHERE email = 'x@example.com';
```

This is a genuinely useful trick for hot read paths. The cost: every write to that table now maintains the extra columns in the index too. Use it sparingly, for the queries that actually matter.

### The Trade-Off Nobody Puts on the Slide

Indexes are not free. Every index you add:

- Slows down `INSERT`/`UPDATE`/`DELETE` (the DB maintains it on every write)
- Takes disk space (and memory, if it's hot)
- Adds planner overhead (more options to consider)

A table with 15 indexes for 15 different one-off queries is a table whose writes are paying for queries that run twice a year. **Measure first, index second.** If a query runs once a day in a cron job, a full scan might be completely fine — don't optimize what doesn't matter.

## Pagination: OFFSET Is a Trap

```sql
-- ❌ Works great on page 1. Page 10,000 means reading and discarding 200,000 rows
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 199980;

-- ✅ Keyset pagination: no OFFSET, the index does all the work
SELECT * FROM posts
WHERE created_at < '2026-08-01 12:00:00'   -- last row's value from previous page
ORDER BY created_at DESC
LIMIT 20;
```

Offset pagination gets quadratically slower as you go deeper — the database still has to scan and discard all the skipped rows. Keyset (or "seek") pagination uses the index to jump straight past them. The trade-off: you can't jump to page 5,000 anymore, and you need a stable sort key (tie-break with the primary key if `created_at` isn't unique). For infinite-scroll feeds — which is what most apps actually have — keyset is strictly better.

## EXISTS vs IN vs JOIN

```sql
-- ❌ DISTINCT + JOIN to find users with orders — sorts a potentially huge result
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id;

-- ✅ EXISTS short-circuits: stops at the first matching order per user
SELECT u.id, u.name
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

`EXISTS` stops as soon as it finds one match; `IN`/`JOIN` don't necessarily. Modern optimizers often rewrite these into the same plan anyway, but `EXISTS` communicates intent and behaves predictably — and with a good index on `orders(user_id)`, it's hard to beat.

The flip side: if you *do* need data from the joined table, `JOIN` is the right tool. `EXISTS` doesn't return columns from the other table. Pick by what you need, not by what you heard is "faster."

## The Optimization Workflow

Optimizing queries without a process is how you end up with 15 indexes and the same slow query. Do it in order:

1. **Find the slow queries first.** Enable the slow query log (`log_min_duration_statement` in Postgres, `slow_query_log` in MySQL) or use `pg_stat_statements` to see what's actually slow in production — not what you *guess* is slow.
2. **`EXPLAIN ANALYZE` the worst offender.** Look for `Seq Scan` on big tables, huge gaps between estimated and actual rows, and sort/hash steps.
3. **Fix the query first** (sargability, N+1, `SELECT *`), then consider indexes.
4. **Verify with the same `EXPLAIN ANALYZE`** — did `actual time` drop? Did the plan change? If not, you fixed the wrong thing.
5. **Re-check after a while.** Data distribution changes, and so do plans. A query that used an index beautifully at 1M rows may get re-planned badly at 10M.

```bash
# Quick way to find the worst offenders in Postgres (needs pg_stat_statements enabled)
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

## What Breaks If You Skip This

- Your "cheap" dashboard endpoint scans a 50M-row table on every page load, and the DB CPU pegs at 100% during peak hours
- The N+1 loop that was fine with 50 test users becomes 5,000 queries per request in production, and your API latency goes from 80ms to 8 seconds
- You add indexes blindly, writes slow down, and the DB disk fills up — and the query you were trying to fix is still slow, because it was never index-bound in the first place

## Takeaways

- **Read `EXPLAIN ANALYZE` before touching anything** — the plan tells you where the time goes; guessing is how you fix the wrong thing.
- **Fix the query before adding indexes** — check for `SELECT *`, functions on columns, type mismatches, and leading wildcards first.
- **Kill N+1 with eager loading** — it's the most common "slow app" cause that has nothing to do with SQL.
- **Composite index order: equality columns first, then range/sort** — and remember every index taxes your writes.
- **Use keyset pagination for deep pages** — `OFFSET` gets quadratically worse the deeper you go.
- **Measure from production data** — slow query logs and `pg_stat_statements` beat your intuition every time.
