# Caching at Scale: Redis Clusters

A single Redis instance can handle ~100K ops/second. That sounds like a lot — until your app serves millions of users and every page load triggers a dozen cache lookups. Suddenly that single node becomes a bottleneck, a single point of failure, and a memory constraint all at once.

Scale up your cache, and you need a Redis cluster. Let's talk about how it works, what breaks, and how to actually make it production-ready.

---

## The Single-Node Problem

Running Redis on one box works fine until it doesn't. Here's what bites you:

| Problem | What Happens |
|---------|-------------|
| **Memory cap** | Redis is in-memory. One box = one RAM ceiling. Your dataset outgrows it. |
| **Throughput ceiling** | One CPU core for event loop. You max out at ~100K commands/sec. |
| **Single point of failure** | Node goes down = entire cache disappears. Say hello to a thundering herd of requests hitting your DB. |
| **No fault tolerance** | Restart Redis, and unless you configured persistence, your cache is cold. |

You could vertically scale — bigger box, more RAM. But at some point that stops being cost-effective, and you still have the SPOF problem.

---

## How Redis Cluster Actually Works

Redis Cluster isn't just "multiple Redis instances." It's a specific topology with built-in sharding and replication.

### Slot-Based Sharding

The cluster splits the keyspace into **16,384 hash slots**. Every key gets assigned to a slot:

```
HASH_SLOT = CRC16(key) % 16384
```

Each node owns a range of these slots. When you add a node, slots get redistributed. Remove a node, same thing.

```
Node A:  slots 0–5460
Node B:  slots 5461–10922
Node C:  slots 10923–16383
```

This means every key lives on exactly one node. Your app (or Redis client) needs to know which node holds which slot.

### Replication Groups

Each master node can have one or more replicas. If the master goes down, a replica gets promoted. This gives you:

- **High availability** — node loss doesn't mean data loss
- **Read scaling** — replicas can serve read-only queries
- **Automatic failover** — the cluster detects failures and promotes replicas

### The No-Cross-Slot Gotcha

Here's where people get tripped up. Redis Cluster doesn't allow multi-key operations across different slots:

```redis
# ❌ This will fail — keys are on different nodes
MGET user:123 user:456 user:789

# ✅ This works — same slot (hash tags)
MGET user:{123} user:{456} user:{789}
```

**Hash tags** (`{...}`) force multiple keys into the same slot. The cluster only hashes what's inside the braces. Use them wisely — cramming everything into one slot defeats the purpose of sharding.

---

## Setting Up a Redis Cluster

### Minimal Cluster (3 Masters, 3 Replicas)

```bash
# Create 6 Redis instances (3 masters + 3 replicas)
for port in 7000 7001 7002 7003 7004 7005; do
  mkdir -p /redis/cluster/$port
  cat > /redis/cluster/$port/redis.conf << CONF
port $port
cluster-enabled yes
cluster-config-file nodes-$port.conf
cluster-node-timeout 5000
appendonly yes
dir /redis/cluster/$port
CONF
  redis-server /redis/cluster/$port/redis.conf &
done

# Create the cluster (one command to rule them all)
redis-cli --cluster create \
  127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
  127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1
```

The `--cluster-replicas 1` flag tells Redis to assign one replica per master automatically.

### What Happens at Startup

```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
>>> Nodes configuration updated
>>> Cluster state changed to OK
```

---

## Client-Side: Talking to the Cluster

You can't just point your app at one node anymore. The client needs to be cluster-aware.

### Node.js Example (ioredis)

```javascript
const Redis = require('ioredis');

// Cluster-aware client — it handles slot routing internally
const cluster = new Redis.Cluster([
  { host: '10.0.1.1', port: 7000 },
  { host: '10.0.1.2', port: 7001 },
  { host: '10.0.1.3', port: 7002 },
]);

// This looks like any other Redis call
await cluster.set('session:abc123', JSON.stringify(sessionData), 'EX', 3600);
const data = await cluster.get('session:abc123');

// But under the hood, ioredis:
// 1. Hashes the key → finds the slot
// 2. Routes the command to the right node
// 3. Handles MOVED redirections transparently
// 4. Retries on failure with built-in backoff
```

### What Happens When a Key Moves (MOVED Redirect)

```redis
# Client asks Node A for "session:xyz"
127.0.0.1:7000> GET session:xyz
-MOVED 12182 127.0.0.1:7002
# "Hey, slot 12182 is on Node C, not me. Ask them."
```

Smart clients handle this transparently. Dumb clients need to re-issue the command to the right node. **Use a cluster-aware client.**

---

## Scaling Up: Adding Nodes

One of the selling points of Redis Cluster — scale horizontally without downtime.

```bash
# Add a new master node
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# Rebalance slots (this is slow and happens online)
redis-cli --cluster rebalance 127.0.0.1:7000 --cluster-threshold 1

# Add a replica for the new master
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7006 --cluster-slave
```

**During resharding**, some keys are being migrated. The cluster handles this with ASK redirects — the source node temporarily forwards requests. Your client needs to handle this (most modern ones do).

---

## Production Gotchas

### Memory: Don't Fill It Up

Redis Cluster has no per-node memory limits by default. One node fills up → entire cluster degrades. Set `maxmemory` on every node:

```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

Without `maxmemory-policy`, an OOM node stops accepting writes. All writes to that node's slots fail. Your app gets partial write failures — fun to debug at 3am.

### Network Partitions: Split-Brain

Redis Cluster uses a **gossip protocol** for node discovery and failure detection. If a network partition splits your cluster:

- **Majority side** — continues operating normally
- **Minority side** — stops accepting writes (cluster-required-full-coverage)

By default, if any slot is unreachable, the entire cluster stops. You can disable this:

```
cluster-require-full-coverage no
```

But think carefully before doing this — you're trading availability for consistency.

### Batch Operations: Pipelines with Care

```javascript
// ❌ Naive — sends each command separately
for (const userId of userIds) {
  await cluster.get(`user:${userId}`);
}

// ✅ Pipeline with cluster awareness — batches by node
const pipeline = cluster.pipeline();
for (const userId of userIds) {
  pipeline.get(`user:${userId}`);
}
const results = await pipeline.exec();
// ioredis routes commands to the right nodes internally
```

### Cold Start Problem

When your Redis Cluster restarts from scratch (e.g., after a full outage), every key is gone. All cache misses hit your database simultaneously. This is the **thundering herd** problem.

Mitigations:
- Gradual warmup — preload popular keys before serving traffic
- Rate-limited DB fallback — don't let all cache misses hit DB at once
- Local L1 cache (caffeine/caffine) — buffers the fallback

---

## Redis Cluster vs Alternatives

| Solution | Sharding | HA | Latency | Ops | Best For |
|----------|----------|----|---------|-----|----------|
| **Redis Cluster** | Built-in | Built-in | ~1ms | ~100K/node | Large datasets, HA needed |
| **Redis Sentinel** | Manual | Built-in | ~1ms | ~100K | HA without sharding |
| **Memcached** | Client-side | None | <1ms | ~200K | Simple key-value, no persistence |
| **Dragonfly** | Built-in | In progress | <1ms | ~1M+ | Extreme throughput |
| **KeyDB** | Multi-threaded | Sentinel | <1ms | ~200K | Drop-in Redis replacement |

**Trade-off**: Redis Cluster adds operational complexity (monitoring nodes, resharding, slot management) but gives you horizontal scaling and HA out of the box. For datasets under ~50GB, a single Redis instance with Sentinel might be simpler and faster.

---

## Monitoring What Matters

```bash
# Cluster info
redis-cli -h 10.0.1.1 -p 7000 cluster info

# Node health
redis-cli -h 10.0.1.1 -p 7000 info stats

# Key metrics to watch
# - cluster_state:fail → bad times
# - cluster_slots_assigned: 16384 → full coverage
# - cluster_known_nodes → expected count
# - Keyspace misses → rising = cold cache = DB getting hammered
# - Evicted keys → maxmemory too low
```

---

## Actionable Takeaways

- **Use Redis Cluster when** your dataset exceeds ~50GB or you need HA without a separate Sentinel setup
- **Hash tags are a sharp tool** — they're essential for multi-key ops but too much aggregation defeats sharding
- **Set maxmemory + eviction policy** on every node — a single OOM node breaks writes for its slots
- **Use cluster-aware clients** (ioredis, redis-py-cluster, Lettuce) — don't roll your own slot routing
- **Plan for cold starts** — have a cache-warming strategy or rate-limited DB fallback
- **Monitor cluster state** — `CLUSTER INFO` is your first diagnostic when something feels wrong
