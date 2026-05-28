# Failover Patterns: Active-Active vs Active-Passive

Your database goes down. What happens next? If you're running a single instance, the answer is simple: your app breaks. Users get 500s. Someone gets paged at 3 AM. You scramble to bring it back up.

The fix is redundancy—running multiple instances so when one dies, another takes over. But *how* they take over is where it gets interesting. There are two main camps: **Active-Active** and **Active-Passive**. They're fundamentally different approaches to the same problem, and picking the wrong one will haunt you.

---

## The Core Difference

**Active-Passive** keeps a standby instance that does nothing until the primary fails. **Active-Active** puts every instance to work, all the time.

Sounds simple, but the implications ripple through your entire architecture.

---

## Active-Passive: The Hot Standby

Also called **hot standby** or **leader-follower with failover**. One instance handles traffic, the other just... waits.

```
┌──────────────┐         ┌──────────────┐
│   Primary    │ ──sync──▶│   Standby    │
│  (handles    │         │   (idle,     │
│   traffic)   │         │  waiting)    │
└──────────────┘         └──────────────┘
       │
       ▼
    Users
```

When the primary dies, a health checker (or manual intervention) promotes the standby to primary. DNS records or load balancer configs get updated to point at the new primary.

### Where it shines

- **Databases** — PostgreSQL with Patroni, MySQL with Orchestrator, MongoDB replica sets. Active-passive (or single-primary) is the standard for stateful workloads.
- **Simple to reason about** — No conflicts, no split-brain, no concurrent writes to manage.
- **Lower resource needs** — You're paying for one active node plus one cheaper standby instead of two full nodes.

### Where it hurts

- **Failover isn't instant.** DNS propagation takes time. Even with automatic failover, there's a window where the service is down.
- **The standby is wasted.** You're paying for compute that does nothing useful 99.9% of the time.
- **Failover complexity is hidden.** Testing failover is hard because you rarely do it. When you *need* it, it might not work.

```
// ❌ Active-passive failover in practice is never as clean as the diagram
// What actually happens during a PostgreSQL failover:

1. Primary crashes (or becomes unreachable)
2. Patroni detects the failure (~5-10 seconds)
3. Standby checks: "Should I promote myself?"
4. Patroni promotes the standby to primary
5. Standby replays remaining WAL logs (may take seconds to minutes)
6. Applications reconnect with new connection string
7. Old primary comes back — now it's a replica that needs to catch up

// That's 10-60+ seconds of downtime depending on WAL replay time
```

### Making it work

```python
# Pattern: Active-passive with health checks and automatic failover
# Using a simple consul-based approach

class FailoverManager:
    def __init__(self, primary, standby):
        self.primary = primary
        self.standby = standby
        self.current_primary = primary
    
    def check_health(self):
        """Health check with a 3-strike threshold to avoid flapping"""
        failures = 0
        for _ in range(3):
            try:
                resp = requests.get(f"{self.primary}/health", timeout=2)
                if resp.status_code == 200:
                    return True
            except requests.RequestException:
                failures += 1
        
        return failures >= 3  # Only failover after 3 consecutive failures
    
    def failover(self):
        """Promote standby to primary"""
        if self.check_health():
            return  # Primary is fine, no action needed
        
        # Promote standby
        requests.post(f"{self.standby}/promote")
        self.current_primary = self.standby
        
        # Update DNS or config registry
        self.update_service_discovery(self.standby)
        
        # Log the event for compliance/audit
        logger.warning("FAILOVER: Promoted %s to primary", self.standby)
```

---

## Active-Active: Everyone Works

Every node handles traffic simultaneously. There's no idle capacity—you're running N instances, and all N are serving requests.

```
        ┌──── Request ────┐
        ▼                  ▼
┌──────────────┐   ┌──────────────┐
│  Node A      │   │  Node B      │
│  (active)    │   │  (active)    │
└──────┬───────┘   └──────┬───────┘
       │                  │
       └────── DB ────────┘
         (shared or sharded)
```

### Where it shines

- **Zero downtime on single-node failure.** Traffic just redistributes to remaining nodes.
- **Better resource utilization.** Every dollar spent on compute serves traffic.
- **Linear scalability.** Need more capacity? Add another node.

### Where it hurts

- **Stateful workloads are painful.** Two databases writing to the same data? That's how you get conflicts, lost writes, and your Friday night ruined.
- **You need session-aware routing.** Unless you use distributed sessions (Redis, etc.), user sessions break when they hit a different node.
- **Caching becomes distributed cache invalidation.** Each node has its own cache, so stale data creeps in.

```
// ✅ Active-Active works great for stateless services (API servers, workers)
// ❌ Active-Active is a nightmare for relational databases without careful design

// OK for active-active:
┌─────────┐   ┌─────────┐
│ API Svr │   │ API Svr │
│  (stateless,  │  (stateless,
│  shares Redis)│  shares Redis)
└────┬────┘   └────┬────┘
     └─── Redis ───┘

// Much harder for active-active:
┌─────────┐   ┌─────────┐
│ MySQL A │   │ MySQL B │
│  (writes)│   │  (writes)│
└────┬────┘   └────┬────┘
     └─── Conflict ───┘
       (multi-master replication is fragile)
```

### Making it work

```python
# Pattern: Active-Active with idempotency and distributed sessions
# Assumes stateless compute + shared state via Redis

from flask import Flask, request, session
from flask_session import Session  # Server-side sessions in Redis
import uuid

app = Flask(__name__)
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_PERMANENT'] = False
Session(app)

@app.route('/api/orders', methods=['POST'])
def create_order():
    # Generate idempotency key at the client level
    idempotency_key = request.headers.get('Idempotency-Key', str(uuid.uuid4()))
    
    # Check if this was already processed (idempotent across all nodes)
    # Because all nodes share Redis, any node can check this
    if redis_client.exists(f"processed:{idempotency_key}"):
        return {"message": "Already processed"}, 200
    
    order = create_order_from_request(request.json)
    redis_client.setex(f"processed:{idempotency_key}", 86400, order.id)
    
    return {"order_id": order.id}, 201

# Without idempotency keys, a retry hitting a different node = duplicate order
```

---

## The Decision Matrix

| Factor | Active-Passive | Active-Active |
|--------|---------------|---------------|
| **Failover time** | Seconds to minutes | Instant (traffic redistributes) |
| **Resource cost** | Higher per-request (wasted standby) | Efficient (all nodes serve traffic) |
| **Stateful workloads** | Natural fit | Painful (needs conflict resolution) |
| **Complexity** | Simple to understand | Complex (caching, sessions, writes) |
| **Split-brain risk** | Low (one active writer) | Higher (multi-writer conflicts) |
| **Scaling** | Vertical or promote-to-primary | Horizontal (add nodes) |
| **Maintenance** | Downtime during standby switch | No downtime (drain one node at a time) |

---

## Hybrid Approaches (Real World)

Most production systems aren't pure one-or-the-other. They're mixed:

**Active-active for compute, active-passive for data.**

```
           Load Balancer
          /              \
    ┌─────┐            ┌─────┐
    │ API │   active   │ API │
    │  A  │────────────│  B  │
    └──┬──┘            └──┬──┘
       │                  │
       └────────┬─────────┘
                │
          ┌─────┴──────┐
          │ PostgreSQL │  (active-passive
          │  + Patroni │   with streaming
          └────────────┘   replication)
```

This combo gives you:
- **Stateless layer:** Active-active API servers. Any node dies? Traffic just hits the others.
- **Stateful layer:** Active-passive database. One writer, one standby. Simple, reliable.

### Another hybrid: Active-active with leader-based writes

```yaml
# Kubernetes-style: multiple read replicas, single write primary
# This is what production setups actually look like

Write API → Primary DB (single writer)
Read API  → Replica 1 (read-only) 
          → Replica 2 (read-only)
          → Replica 3 (read-only)

# Primary dies:
# - Writes fail until failover completes (active-passive behavior)
# - Reads continue uninterrupted (replicas still serve)
# - This is called a "read-scale" or "primary-replica" setup
```

---

## Split-Brain: The Nightmare Scenario

Both patterns can suffer from split-brain. In active-passive, if both nodes think they're the primary, you get conflicting writes. In active-active, it's a constant risk anyway.

### Preventing it

- **Quorum-based decision making** — Use tools like etcd, Consul, or Zookeeper with a majority-consensus protocol (Raft, Paxos).
- **Fencing (STONITH)** — Shoot The Other Node In The Head. If a node can't confirm its peer is dead, forcibly shut the peer down before taking over.
- **Lease mechanisms** — Writes require a lease from a coordinator. Expired lease = no writes.

```
// SQL-level fencing for PostgreSQL active-passive
// Patroni + etcd handle this automatically:

1. Primary maintains a leader key in etcd with a TTL (typically 30s)
2. Primary refreshes the key every ~10 seconds
3. If primary crashes, etcd key expires after TTL
4. Standby detects expired key, creates its own leader key
5. Old primary restarts — sees it doesn't hold the leader key
6. Old primary refuses to accept writes (fencing works!)
```

---

## What You'll Actually Deal With

**Start with active-passive for databases.** It's the most battle-tested. PostgreSQL with Patroni, MySQL with Orchestrator, MongoDB replica sets — these are proven patterns that thousands of teams run in production.

**Go active-active for stateless services.** API servers, workers, event consumers — these are trivially active-active. A load balancer and you're done.

**Only do active-active databases if you have to.** Multi-master MySQL (Galera), CockroachDB, and Cassandra are active-active by design, but they trade consistency for availability and come with sharp edges. You'll need conflict resolution, clock synchronization, and careful schema design. Most apps don't need this.

## Actionable Takeaways

- **Stateless layers should always be active-active.** If your API server isn't active-active, you're wasting capacity.
- **Databases default to active-passive.** This is fine. Don't over-engineer for a scaling problem you don't have.
- **Test your failover.** If you haven't tested failover in the last 3 months, assume it doesn't work. Run a game day. Kill the primary. See what breaks.
- **Split-brain is the real enemy.** Use quorum-based tools (etcd, Consul) and fence aggressively. `STONITH` isn't a joke—it's a survival tactic.
- **Design for the pattern your data needs.** If you need multi-region writes, you've chosen active-active by default. Understand the trade-offs *before* you commit.
