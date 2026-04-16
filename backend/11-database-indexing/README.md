# Database Indexing: Speed Up Your Queries

Your API is slow. You've optimized your code, added caching, even upgraded the server. But that one endpoint still takes 3 seconds. The culprit? Your database is scanning millions of rows for every request.

Database indexes are the difference between reading a book page-by-page vs. using the index to jump straight to what you need. Without them, your queries perform full table scans—checking every single row. With them, the database can find data in milliseconds.

## What Happens Without Indexes

Let's say you have a `users` table with 10 million rows:

```sql
-- This query scans ALL 10 million rows
SELECT * FROM users WHERE email = 'john@example.com';
```

The database doesn't know where `john@example.com` is. It has to check every single row. On my machine, that's about 8 seconds.

Add an index:

```sql
CREATE INDEX idx_users_email ON users(email);
```

Now the same query takes ~2 milliseconds. The database uses a B-tree structure to navigate directly to the row.

## How Indexes Work

Most databases use **B-tree indexes** by default. Think of it as a sorted tree structure:

```
                    [m]
                   /   \
              [d-g]     [n-s]
              /   \     /   \
           [a-c] [e-f] [n-o] [p-s]
```

To find "p":
1. Start at root [m] → p > m, go right
2. At [n-s] → p is in this range
3. At [p-s] → found it

**Time complexity:** O(log n) instead of O(n)

For 10 million rows:
- Full scan: 10,000,000 comparisons
- B-tree lookup: ~24 comparisons (log₂(10M))

## When to Index

### ✅ Good Candidates

```sql
-- Columns in WHERE clauses
CREATE INDEX idx_orders_status ON orders(status);

-- Columns used in JOINs
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Composite index for common query patterns
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### ❌ Bad Candidates

```sql
-- Low cardinality (few distinct values)
CREATE INDEX idx_users_active ON users(is_active);  -- Only 2 values: true/false

-- Columns that change frequently
-- Every UPDATE to this column requires index updates
CREATE INDEX idx_users_last_login ON users(last_login_at);  -- Updates constantly

-- Small tables
-- Full scan might be faster than index lookup for 100 rows
```

## Composite Indexes: Order Matters

This is where most developers get burned.

```sql
CREATE INDEX idx_users_name ON users(last_name, first_name);
```

This index can help with:

```sql
-- ✅ Uses index
WHERE last_name = 'Smith'

-- ✅ Uses index
WHERE last_name = 'Smith' AND first_name = 'John'

-- ❌ Does NOT use index (leftmost column missing)
WHERE first_name = 'John'
```

**Rule:** A composite index works left-to-right. Skip the first column, and the index is useless.

### Real Example

```sql
-- E-commerce query
SELECT * FROM orders 
WHERE user_id = 12345 
  AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;
```

Which index helps?

```sql
-- ❌ Wrong order
CREATE INDEX idx_orders_status_user ON orders(status, user_id);

-- ✅ Correct order
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at);
```

The second index handles the WHERE clause AND the ORDER BY—no additional sorting needed.

## The Hidden Cost of Indexes

Indexes aren't free. Every `INSERT`, `UPDATE`, or `DELETE` requires updating all relevant indexes.

```sql
-- Table with 5 indexes
INSERT INTO users (email, name, created_at, status, country);
```

This single insert triggers:
- 1 write to the table
- 5 writes to update each index

### Trade-offs

| Scenario | Index Impact |
|----------|-------------|
| Read-heavy workloads | ✅ Positive - faster queries |
| Write-heavy workloads | ⚠️ Negative - slower inserts |
| Small tables (<1000 rows) | ⚠️ Minimal benefit |
| Large tables (>100K rows) | ✅ Huge benefit |

```sql
-- ❌ Over-indexing
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(email, name);  -- Redundant, idx1 covered
CREATE INDEX idx3 ON users(created_at);   -- Rarely queried
CREATE INDEX idx4 ON users(updated_at);   -- Never queried
CREATE INDEX idx5 ON users(is_active);    -- Low cardinality

-- ✅ Thoughtful indexing
CREATE INDEX idx_users_email ON users(email);           -- Login lookups
CREATE INDEX idx_users_created ON users(created_at);     -- Dashboard queries
```

## EXPLAIN: Your Best Friend

Before adding an index, see what the query planner thinks:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345;
```

Look for:

```
❌ Seq Scan on orders  (cost=0.00..45000.00 rows=1 width=100)
   Filter: (user_id = 12345)
   Rows Removed by Filter: 999999

✅ Index Scan using idx_orders_user on orders  (cost=0.42..8.44 rows=1 width=100)
   Index Cond: (user_id = 12345)
```

**Seq Scan** = full table scan (bad for large tables)
**Index Scan** = using index (good)

## Partial Indexes: Smaller, Faster

Sometimes you only need to index a subset of data:

```sql
-- Only index active users
CREATE INDEX idx_active_users_email ON users(email) 
WHERE is_active = true;

-- Only index unpaid orders
CREATE INDEX idx_unpaid_orders ON orders(created_at) 
WHERE status = 'pending' OR status = 'processing';
```

Benefits:
- Smaller index size
- Faster writes (fewer index updates)
- Faster reads (less data to scan)

## Common Mistakes

### 1. Indexing Everything

```sql
-- ❌ Don't do this
CREATE INDEX idx_everything ON orders(user_id, status, total, created_at, updated_at);
```

This creates a massive index that's rarely used. Better to create targeted indexes for specific queries.

### 2. Ignoring Query Patterns

```sql
-- ❌ Index doesn't match how you query
CREATE INDEX idx_users_name ON users(name);

-- But your app queries:
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
```

The index on `name` is useless for email lookups. The `LOWER()` function also prevents index usage unless you create a function-based index:

```sql
-- ✅ Function-based index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

### 3. Not Reindexing

Over time, indexes become fragmented:

```sql
-- PostgreSQL
REINDEX INDEX idx_users_email;

-- Or for the whole table
REINDEX TABLE users;

-- MySQL
ANALYZE TABLE users;
OPTIMIZE TABLE users;
```

## Quick Reference

| Query Pattern | Recommended Index |
|--------------|-------------------|
| `WHERE col = ?` | Single-column index on `col` |
| `WHERE a = ? AND b = ?` | Composite index on `(a, b)` |
| `WHERE a = ? ORDER BY b` | Composite index on `(a, b)` |
| `WHERE a = ? AND b = ? ORDER BY c` | Composite index on `(a, b, c)` |
| `WHERE LOWER(col) = ?` | Function index on `LOWER(col)` |
| Full-text search | GIN or GiST index |

## Actionable Takeaways

1. **Profile before indexing.** Use `EXPLAIN ANALYZE` to find slow queries.

2. **Index for your actual queries.** Don't guess—check what your app actually runs.

3. **One good composite index beats five single-column indexes.** Think about query patterns.

4. **Monitor write performance.** If inserts slow down, you might have too many indexes.

5. **Use partial indexes for filtered queries.** Smaller, faster, less overhead.

6. **Reindex periodically.** Fragmented indexes degrade performance over time.