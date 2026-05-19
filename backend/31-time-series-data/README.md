# Time Series Data Storage

Regular databases are great until you need to ask "what was the CPU usage every 5 seconds over the past month?" or "show me how many requests per minute we've had for the last 90 days."

That's when you realize your PostgreSQL table with 500 million rows just isn't cutting it anymore.

## What Makes Time Series Data Different

Time series data is any data that has a timestamp and comes in ordered, append-heavy streams. Think:

- Server metrics (CPU, memory, disk I/O)
- Application logs
- IoT sensor readings
- Financial tick data
- User activity events (page views, clicks)

The patterns are fundamentally different from your typical CRUD data:

| Regular Data | Time Series Data |
|---|---|
| Frequently updated/deleted | Mostly appended, rarely modified |
| Random access patterns | Ordered by time |
| Row-level edits common | Bulk inserts at high velocity |
| Can tolerate moderate write latency | Needs fast ingestion (thousands/sec+) |
| Typical retention: indefinite | Often has retention windows (7d, 30d, 1y) |

## The Problem: Vanilla PostgreSQL Won't Cut It

Let's say you're storing server metrics in a regular Postgres table:

```sql
CREATE TABLE metrics (
    id BIGSERIAL PRIMARY KEY,
    server_id INT NOT NULL,
    metric_name VARCHAR(50) NOT NULL,
    value DOUBLE PRECISION NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_metrics_time ON metrics (recorded_at);
```

This works fine for a few thousand rows. But when you're inserting 10,000 metrics per second per server across 50 servers...

```sql
-- ❌ This query becomes painfully slow at scale
SELECT server_id, AVG(value)
FROM metrics
WHERE metric_name = 'cpu_usage'
  AND recorded_at >= NOW() - INTERVAL '30 days'
GROUP BY server_id;

-- ✅ With TimescaleDB's hypertable, it uses automatic partitioning by time
-- Queries prune partitions they don't need, so it stays fast
```

What breaks:
- **Index maintenance** slows down writes as the table grows
- **Auto-vacuum** can't keep up with append-heavy workloads
- **Range queries** scan too many pages because data isn't physically ordered
- **Partition management** is manual and painful to maintain

## Your Options for Time Series Storage

### 1. TimescaleDB (PostgreSQL Extension)

The pragmatic choice if you're already using Postgres. It makes regular Postgres understand time series.

```sql
-- Create a regular table, then convert it
CREATE TABLE conditions (
    time TIMESTAMPTZ NOT NULL,
    location TEXT NOT NULL,
    temperature DOUBLE PRECISION,
    humidity DOUBLE PRECISION
);

SELECT create_hypertable('conditions', 'time', chunk_time_interval => INTERVAL '1 day');
```

**Why it's good:** You keep using Postgres tooling, pgAdmin, your ORM — just faster.
**The catch:** Still Postgres underneath, so memory planning and vacuum strategy need tuning.

### 2. InfluxDB

Purpose-built for time series. Uses its own query language (Flux/InfluxQL).

```
from(bucket: "production")
  |> range(start: -30d)
  |> filter(fn: (r) => r._measurement == "cpu" and r.host == "web-01")
  |> aggregateWindow(every: 5m, fn: mean)
```

**Why it's good:** Blazing fast ingestion, built-in downsampling, data lifecycle management.
**The catch:** It's another database to manage. No joins. Learning curve on Flux.

### 3. Prometheus (Pull-based Metrics)

The de facto standard for infrastructure monitoring.

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'web-app'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9091']
```

**Why it's good:** Dead simple to set up. Built-in alerting (Alertmanager). Huge ecosystem.
**The catch:** Pull model doesn't work for batch jobs. Ephemeral data — not for long-term analytics.

### 4. ClickHouse (OLAP Columnar)

Blazing fast for analytics queries on time series, but it's a columnar store, not a traditional time series DB.

```sql
SELECT
  toStartOfHour(timestamp) AS hour,
  avg(value)
FROM metrics
WHERE timestamp > now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour
```

**Why it's good:** Insane query performance on large datasets. SQL-native.
**The catch:** Not great for point lookups. High memory usage for merges.

## Key Design Decisions

### Retention and Downsampling

Time series data is worth less the older it gets. Design a tiered retention strategy:

```
┌─────────────────────────────────────────────────┐
│ Raw Data (7 days)     │ 1-second resolution     │
├─────────────────────────────────────────────────┤
│ Rollup (30 days)      │ 1-minute aggregates     │
├─────────────────────────────────────────────────┤
│ Archive (1 year)      │ Hourly aggregates       │
└─────────────────────────────────────────────────┘
```

```sql
-- TimescaleDB continuous aggregate example
CREATE MATERIALIZED VIEW conditions_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    location,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM conditions
GROUP BY bucket, location;

SELECT add_continuous_aggregate_policy('conditions_hourly',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### Schema Design: Wide vs Narrow

**Narrow (EAV-style)** — one value per row:

```sql
-- Each metric type is a separate row
| time                | host   | metric  | value |
|---------------------|--------|---------|-------|
| 2026-05-19 10:00:00 | web-01 | cpu     | 45.2  |
| 2026-05-19 10:00:00 | web-01 | memory  | 72.1  |
| 2026-05-19 10:00:00 | web-01 | disk_io | 12.8  |
```

**Wide (columnar)** — each metric type gets its own column:

```sql
| time                | host   | cpu  | memory | disk_io |
|---------------------|--------|------|--------|---------|
| 2026-05-19 10:00:00 | web-01 | 45.2 | 72.1   | 12.8    |
```

Which one's better?
- **Narrow:** Flexible (add metrics without schema changes), works with any TSDB
- **Wide:** Faster queries for fixed metric sets, less storage overhead per row
- **Reality:** Most TSDBs optimize for narrow internally anyway. Use what your tool recommends.

### Write Path Optimization

Time series systems write in batches, not single inserts:

```go
// ❌ Inserting one at a time - slow, lots of overhead
for _, metric := range metrics {
    db.Exec("INSERT INTO metrics (...) VALUES (...)", metric)
}

// ✅ Batch insert - way faster
batch := make([]Metric, 0, 1000)
for _, metric := range metrics {
    batch = append(batch, metric)
    if len(batch) >= 1000 {
        db.BulkInsert(batch) // Single insert of 1000 rows
        batch = batch[:0]
    }
}
if len(batch) > 0 {
    db.BulkInsert(batch)
}
```

## When NOT to Use a Dedicated Time Series DB

Time series databases aren't always the answer. Skip them when:

- **You have <1M data points total** — PostgreSQL with proper indexing is fine
- **You need complex joins across data types** — Postgres + partitioning works better
- **Your queries aren't time-range based** — you're paying for features you don't use
- **You need strong consistency guarantees** — many TSDBs trade consistency for speed
- **Your team already knows Postgres** — the operational complexity of another DB might not be worth it

## Actionable Takeaways

- If you're already on Postgres and growing, start with **TimescaleDB** — it adds partitioning without changing your stack
- For infrastructure monitoring, **Prometheus** is the default choice for a reason (pull model, alerting built-in)
- For high-cardinality analytics (think: millions of unique tag combinations), **InfluxDB** handles it better than Prometheus
- Always plan retention and downsampling upfront — you can't keep raw data forever
- Batch your writes, don't insert one row at a time
- Push vs pull is a real trade-off: pull = simpler for services with stable endpoints, push = better for ephemeral/batch workloads
