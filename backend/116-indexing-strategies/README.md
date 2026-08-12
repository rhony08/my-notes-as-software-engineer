# Advanced Indexing Strategies

You added an index to every column that appeared in a `WHERE` clause, and somehow your writes got 3x slower while the queries barely improved. Sound familiar?

Indexes are the classic "easy to add, hard to master" tool. A single-column index on everything is the indexing equivalent of throwing spaghetti at the wall. The good stuff — composite indexes, covering indexes, partial indexes — is where the real wins live. That's what we'll dig into here.

## Why "Just Index Everything" Fails

Every index is a separate data structure the database has to maintain on **every write**. Insert, update, delete — all of them now touch the index too. That's the write amplification nobody mentions when they say "just add an index."

Let's say you have a `users` table:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email TEXT NOT NULL,
    first_name TEXT,
    last_name TEXT,
    country TEXT,
    last_login_at TIMESTAMPTZ,
    is_active BOOLEAN DEFAULT true
);
```

Now imagine this query:

```sql
SELECT id, first_name, last_name
FROM users
WHERE country = 'DE' AND last_login_at > now() - interval '30 days'
ORDER BY last_login_at DESC;
```

The naive fix: index on `country` and another on `last_login_at`.

```sql
-- ❌ Two separate single-column indexes
CREATE INDEX idx_users_country ON users (country);
CREATE INDEX idx_users_last_login ON users (last_login_at);
```

Postgres can only use one index per table in a plain scan (bitmap scans can combine them, but it's rarely as efficient as one good composite index). You're paying maintenance cost for **two** indexes and getting a half-baked plan. One of them is basically dead weight for this query.

## Composite Indexes: Order Matters

The right move is a single composite index:

```sql
-- ✅ One composite index that matches the query
CREATE INDEX idx_users_country_login
ON users (country, last_login_at DESC);
```

Composite indexes follow the **leftmost prefix rule**: the index can be used for lookups on `country`, on `country + last_login_at`, but **not** on `last_login_at` alone. Column order is the whole game.

| Column order | Works for | Doesn't help |
|---|---|---|
| `(country, last_login_at)` | `WHERE country = ?`, `WHERE country = ? AND last_login_at > ?` | `WHERE last_login_at > ?` alone |
| `(last_login_at, country)` | `WHERE last_login_at > ?`, range + filter on country | equality on country alone is less useful |

General rule of thumb:

1. **Equality conditions first** (`=`, `IN`) — they narrow the search space most predictably.
2. **Range conditions last** (`>`, `<`, `BETWEEN`) — the index is only useful for the range on the *last* column; everything after it in the index is dead for that query.

That's why `(country, last_login_at)` beats `(last_login_at, country)` for our query: `country = 'DE'` is equality, `last_login_at > ...` is a range.

> **Trade-off:** you can't optimize for every query shape. You pick the index that serves your hottest queries and accept that the occasional cold query does a seq scan. Trying to serve all query shapes = index explosion.

## Covering Indexes: Skip the Table Round-Trip

Here's a subtle one. Our query selects `first_name` and `last_name` — columns that aren't in the index. So Postgres has to:

1. Find matching rows in the index (fast)
2. Jump to the heap (the actual table) to fetch `first_name` and `last_name` (slower)

That second step is a **lookup per row**. For big result sets, it dominates. The fix: a covering index with `INCLUDE`.

```sql
-- ✅ Covering index: all needed columns live in the index
CREATE INDEX idx_users_country_login_covering
ON users (country, last_login_at DESC)
INCLUDE (first_name, last_name);
```

Now the query can be answered entirely from the index — an **index-only scan**. No table round-trip at all. Postgres will tell you in `EXPLAIN` when it can do this:

```
Index Only Scan using idx_users_country_login_covering on users
```

The `INCLUDE` columns are stored in the index but **don't participate in ordering or lookups**. They're just payload. So don't put columns in `INCLUDE` that you filter or sort on — put those in the main column list.

**When does this matter?** When you have a query that runs thousands of times per second and the table is too big to fit in cache. Shaving one heap lookup per row is a massive win. For a query that runs once a day? Don't bother — the index bloat isn't worth it.

## Partial Indexes: Index Less, Win More

This is the most underused trick in the book. Why index 10 million rows when your queries only ever touch 100,000?

```sql
-- ❌ Indexing everything
CREATE INDEX idx_users_active ON users (last_login_at);

-- ✅ Only index the rows that matter
CREATE INDEX idx_users_active_login
ON users (last_login_at)
WHERE is_active = true;
```

Now the index only contains active users. It's:

- **Smaller** → fits in memory, faster scans
- **Cheaper to maintain** → fewer writes to update
- **More selective** → less noise for the planner

The query needs to match the predicate though:

```sql
-- ✅ Uses the partial index
SELECT * FROM users WHERE is_active = true AND last_login_at < now() - interval '90 days';

-- ❌ Won't use it (missing the predicate)
SELECT * FROM users WHERE last_login_at < now() - interval '90 days';
```

Partial indexes shine for things like: soft-deleted rows, current vs archived data, `status = 'pending'` queues. Any time a small slice of the table gets hammered by queries, a partial index is a free win.

## Expression Indexes: When Your WHERE Clause Lies

```sql
-- ❌ This query can't use an index on email
SELECT * FROM users WHERE lower(email) = 'rhon@example.com';
```

The `lower()` call makes the plain `email` index useless — the database would have to compute `lower(email)` for every row. The fix is to index the expression itself:

```sql
-- ✅ Index the expression, not the column
CREATE INDEX idx_users_email_lower ON users (lower(email));
```

Now the query planner recognizes the expression and uses the index. Same trick works for JSONB fields:

```sql
CREATE INDEX idx_orders_customer_id
ON orders ((payload ->> 'customer_id'));
```

**Gotcha:** the expression in the query must match the index definition **exactly**. `lower(email)` in the query, `lower(email)` in the index. Write it slightly differently (`LOWER(email)` vs `lower(email)` is fine, but add a cast and it breaks), and the planner won't match it.

## Cardinality: The Selectivity Trap

An index on a column where 95% of values are the same is nearly useless:

```sql
CREATE INDEX idx_users_is_active ON users (is_active);
```

If 9.5M of 10M users are active, the planner looks at that index and says "nah, I'll just scan the table." And it's right! An index that matches half the table isn't an index, it's an expensive copy of the table.

**High cardinality** (many distinct values: `id`, `email`, `created_at`) → indexes are gold.
**Low cardinality** (few distinct values: `is_active`, `status` with 3 values) → index rarely helps alone; it only makes sense as part of a composite or partial index.

Check your data before creating indexes:

```sql
SELECT count(DISTINCT country) FROM users;        -- if this is 5, skip it
SELECT count(DISTINCT email) FROM users;          -- if this is ~row count, index it
```

## Index Maintenance: The Silent Killer

Indexes degrade over time. Every update leaves dead tuples behind, and the index accumulates bloat. The symptoms: queries get slower for no obvious reason, disk usage creeps up, and `EXPLAIN` shows the same plan but slower execution.

```sql
-- Check for bloat and unused indexes
SELECT relname, indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

Indexes with `idx_scan = 0` (or near zero) are candidates for deletion. They're pure overhead — paid for on every write, used by nothing.

Regular maintenance matters:

- **Postgres:** `VACUUM` keeps things tidy; `REINDEX` (or `pg_repack`) rebuilds bloated indexes. Autovacuum handles the routine stuff, but heavily-updated tables need attention.
- **MySQL/InnoDB:** `OPTIMIZE TABLE` rebuilds the table and its indexes.
- **MongoDB:** `collStats` shows index sizes; drop unused ones.

## The Practical Workflow

When a query is slow, this is the order of operations I actually use:

1. **`EXPLAIN ANALYZE` the query.** See what the planner is doing before touching anything. Maybe it's a seq scan on a huge table, or maybe it's a tiny N+1 problem that no index fixes.
2. **Look at existing indexes.** Maybe the index you need already exists but the query is written in a way that bypasses it (that `lower(email)` trap).
3. **Design for the hottest query.** One composite index beats five single-column ones.
4. **Add `INCLUDE` columns if it's a high-frequency query.**
5. **Check for partial-index opportunities.** Only index the slice of data you actually query.
6. **Measure again.** `EXPLAIN ANALYZE` before and after. If the plan didn't change, the index isn't helping — drop it.

## Actionable Takeaways

- **One composite index beats multiple single-column ones.** Put equality columns first, range columns last.
- **Use `INCLUDE` to build covering indexes** for hot queries and skip the heap round-trip entirely.
- **Partial indexes are free wins** — index only the rows your queries actually touch.
- **Watch cardinality.** Indexing a near-constant column is wasted write amplification.
- **`EXPLAIN ANALYZE` before and after.** If the plan doesn't change, the index isn't earning its keep.
- **Audit periodically.** `pg_stat_user_indexes` (or equivalent) will show you the dead weight. Drop indexes nobody scans.
