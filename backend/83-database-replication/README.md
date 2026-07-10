# Database Replication Strategies

Your database is the single source of truth. It's also a single point of failure.

Here's a scenario you've probably lived through: the primary database goes down. Maybe it's a hardware failure, maybe a corrupted page, maybe someone ran a `DROP TABLE` on the wrong server. However it happens, the app stops working. Every request fails. Users start tweeting about you.

Replication is how you avoid that. But not all replication's the same—each strategy has trade-offs in consistency, latency, throughput, and complexity.

Let's walk through the real options.

## The Problem Replication Solves

Without replication, you've got one server holding all your data. One disk failure, one network partition, one bad deploy, and you're restoring from last night's backup. That's hours of downtime at best.

Replication gives you:

- **High availability** — if primary goes down, a replica takes over
- **Read scalability** — spread read queries across replicas
- **Disaster recovery** — data survives a data center failure
- **Analytics isolation** — run heavy queries on replicas without hurting production

But here's the catch: each benefit comes with a trade-off.

## Synchronous vs Asynchronous: The Fundamental Choice

Every replication strategy boils down to one question: **how fast does the data need to arrive at the replica?**

### Synchronous Replication

The primary waits for the replica to confirm before telling the client "write successful."

```
Client → Primary → Writes to disk → Sends to Replica → Replica confirms → "OK"
```

**Pros:**
- Zero data loss (committed writes survive a primary crash)
- Always consistent

**Cons:**
- Writes are slower (waiting for network round-trip)
- If the replica is down, the primary blocks or fails the write
- Secondary is always a bottleneck

### Asynchronous Replication

The primary writes locally, tells the client "done," and sends the data to the replica whenever it gets around to it.

```
Client → Primary → Writes to disk → "OK"  ──(later)──→ Sends to Replica
```

**Pros:**
- Fast writes (no waiting)
- Primary isn't affected by replica latency or failures

**Cons:**
- Data loss window — if primary dies before the replica catches up, those writes are gone
- Replica is always slightly behind (staleness)

### The Reality

Most production systems use **async replication** for performance, then layer on safeguards. The key metric to track is **replication lag** — how far behind the replica is. If it's consistently > 5 seconds, you've got a problem.

## Replication Topologies

Which servers talk to which servers? Here are the common patterns:

### Leader-Based (Single Primary)

```
┌──────────┐      ┌──────────┐
│  Leader   │─────│ Replica 1 │
│ (Primary) │─────│ Replica 2 │
└──────────┘      │ Replica N │
                  └──────────┘
```

**How it works:** One node handles all writes. Followers replicate from it. Reads can hit any node.

You're probably already using this. PostgreSQL streaming replication, MySQL group replication—same pattern.

**Gotchas:**
- Leader is still a single point of failure for writes
- If replicas fall too far behind, reads return stale data

### Multi-Leader (Active-Active)

```
┌──────────┐         ┌──────────┐
│ Leader A  │◄──────►│ Leader B  │
└──────────┘         └──────────┘
     │                    │
┌──────────┐         ┌──────────┐
│ Replica  │         │ Replica  │
└──────────┘         └──────────┘
```

**How it works:** Multiple nodes accept writes and replicate to each other. Used in multi-datacenter setups where you want local writes in each region.

**Real-world example:** MySQL with multi-source replication. CockroachDB. Google Spanner.

**Gotchas:**
- Write conflicts — two users editing the same record in different leaders
- Requires conflict resolution (last-write-wins, CRDTs, or app-level handling)
- Much harder to reason about consistency

### Leaderless

**How it works:** Any node can accept reads and writes. Quorum ensures consistency.

**Real-world example:** Cassandra, Amazon DynamoDB, Riak.

Every client writes to N nodes, reads from R nodes. If W + R > N, you get strong consistency.

**Gotchas:**
- Complex conflict resolution
- Need to tune W and R values carefully
- Operations are harder (repair, tombstone management)

## Database-Specific Approaches

### PostgreSQL: Streaming Replication

PostgreSQL's built-in replication is solid but opinionated:

```sql
-- On the primary
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 5;

-- On the replica (recovery.conf or pg_hba.conf change)
primary_conninfo = 'host=primary-db port=5432 user=replicator'
hot_standby = on     -- allows read-only queries on replica
```

**Key detail:** PostgreSQL uses WAL (Write-Ahead Log) shipping. The primary ships WAL segments to replicas, which replay them. This means:

- Replicas are **exact** copies at the transaction level
- You can promote a replica to primary with `pg_ctl promote`
- Sync replication is optional (`synchronous_standby_names`)

**Gotcha:** PostgreSQL synchronous replication blocks ALL transactions after `synchronous_commit = on` if any configured standby is down. Many teams use "quorum sync" to mitigate this.

### MySQL: Group Replication

MySQL's group replication creates a cluster that handles consensus internally:

```sql
-- On each node
INSTALL PLUGIN group_replication SONAME 'group_replication.so';

-- Configure
SET GLOBAL group_replication_group_seeds = 'node1:3306,node2:3306,node3:3306';
SET GLOBAL group_replication_bootstrap_group = ON;

START GROUP_REPLICATION;
```

**Key detail:** MySQL Group Replication uses Paxos internally. The cluster auto-elects a primary. If the primary dies, a new one is elected automatically.

**Pros:** Auto-failover, built-in conflict detection
**Cons:** All nodes need to be in the same network (high latency kills performance), 9-node limit

### MongoDB: Replica Sets

MongoDB's replica sets are a mature leader-based system:

```json
// rs.conf() output
{
  "_id": "myReplicaSet",
  "members": [
    { "_id": 0, "host": "node1:27017", "priority": 2 },
    { "_id": 1, "host": "node2:27017", "priority": 1 },
    { "_id": 2, "host": "node3:27017", "priority": 1, "arbiterOnly": true }
  ]
}
```

**Key detail:** A replica set has one primary, multiple secondaries, and optionally an arbiter (votes but doesn't store data, useful for odd-numbered voting).

**Read preferences matter:**
```
// ❌ Always read from primary — defeats scalability
db.collection.find().readPref('primary')

// ✅ Read from nearest replica — lower latency, potentially stale
db.collection.find().readPref('nearest')

// ✅ Read from secondary — isolate analytics from primary
db.collection.find().readPref('secondary')
```

Simple gotcha: with `readPref('secondary')`, a slow secondary could serve 5-second-old data and your users see inconsistent results.

## Choosing Your Strategy

There's no universal answer. Here's how to decide:

| Priority | Strategy | Database Fit |
|----------|----------|-------------|
| **Zero data loss** | Sync replication | PostgreSQL with `synchronous_commit = on` |
| **Fast writes** | Async replication | Any (PostgreSQL, MySQL, MongoDB) |
| **Global writes** | Multi-leader | CockroachDB, MySQL multi-source |
| **Read scaling** | Leader + many followers | PostgreSQL streaming, MongoDB replica sets |
| **Maximum availability** | Leaderless (quorum) | Cassandra, DynamoDB |
| **Simple setup** | Leader with semi-sync | MySQL Group Replication |

## Monitoring Replication Health

If you're running replication, you must monitor these or you will get burned:

```
// PostgreSQL - check replication lag
SELECT
  pid,
  application_name,
  state,
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;

// MySQL - check seconds behind master
SHOW SLAVE STATUS\G
-- Look at: Seconds_Behind_Master, Slave_IO_Running, Slave_SQL_Running

// MongoDB - check opLog lag
rs.printSlaveReplicationInfo()
```

**What to alert on:**

- **Replication lag > 10 seconds** (for async) — your replica is falling behind
- **Replication stopped** — IO or SQL thread stopped on MySQL
- **Replica disconnected** — network issue or primary can't reach replica
- **Disk space on replica** — replication stops if replica runs out of space

## Real Talk: What Breaks in Practice

A few things that'll ruin your day:

**1. Schema changes run on the primary, but the replica wasn't ready**

You add a column on the primary, the DDL runs, and suddenly replication breaks because the replica can't apply the change. Solution: use tools like `gh-ost` (MySQL) or `pgroll` (PostgreSQL) for online migrations.

**2. A replica falls behind during a heavy batch job**

Your nightly ETL runs on a replica. Except it's so heavy the replica can't keep up with the primary. Reads start returning 5-minute-old data. Solution: use a dedicated read-only replica for analytics, not the same ones serving production reads.

**3. Failover doesn't go as planned**

Primary dies. You promote the replica. But the old primary comes back online with stale data, and now you've got two primaries accepting writes — a split-brain scenario. Solution: use fencing (STONITH) or database features like PostgreSQL's `recovery_target_timeline`.

## The Bottom Line

**Start with leader-based async replication.** It's the easiest to set up, well-supported by every major database, and covers 80% of use cases. Add synchronous protection for your critical writes. Monitor replication lag like it's your job (because it is).

Once you outgrow that, you'll know — because you'll feel the pain of a specific trade-off. That's when multi-leader or leaderless becomes the right choice.

<hr/>

### What Now?

- **Monitor your replication lag today** — run the query above and see where you stand
- **Test a failover** — don't wait for an emergency to learn if it works
- **For global writes** → Research [CockroachDB's multi-region patterns](https://www.cockroachlabs.com/docs/stable/multiregion-overview.html)
- **For MySQL** → Check out [MySQL InnoDB Cluster](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-introduction.html) for auto-failover
- **For PostgreSQL** → Look into [Patroni](https://patroni.readthedocs.io/) for production HA
