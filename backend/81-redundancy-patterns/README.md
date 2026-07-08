# Redundancy Patterns: Don't Put All Your Eggs in One Server

Your service is running in production. Traffic is fine. Life is good. Then the hosting provider has a network hiccup, the VM your app runs on goes down, and suddenly your customers can't access anything.

"This shouldn't happen," you mutter at 2 AM, SSHing into a fresh instance.

The fix isn't better luck. It's redundancy—running multiple copies of your infrastructure so that when one thing fails, another takes over without anyone noticing.

Let's look at the patterns that make that possible.

## What We Mean by Redundancy

Redundancy means having backup. Not as an afterthought, but baked into your architecture. The goal: **no single component failure takes down the whole system**.

There are trade-offs with every approach:

| Pattern | Cost | Complexity | Failover Speed | Data Loss Risk |
|---------|------|------------|----------------|----------------|
| Active-Passive | Medium | Low | Seconds to minutes | Low |
| Active-Active | High | High | Instant | Low |
| N+1 | Low | Very Low | Minutes | Medium |
| Database Replica | Medium | Medium | Seconds | Very Low |
| Geographic | Very High | Very High | Minutes | Variable |

## Active-Passive (Hot Standby)

The simplest pattern. You run a primary server and a standby server. The standby sits there, ready to take over. It might be running the same software but not serving traffic. When the primary fails, you fail over to the standby.

```
┌──────────┐        ┌──────────┐
│  Primary  │ ─────  │  Standby  │
│ (Active)  │        │ (Passive) │
└─────┬────┘        └─────┬────┘
      │                   │
      │      Shared       │
      │     Storage/DB    │
      └──────────┬────────┘
                 │
          ┌──────┴──────┐
          │   Clients    │
          └─────────────┘
```

```python
# ❌ No redundancy: one server, one point of failure
servers = ["api-1.prod.internal"]
current_server = servers[0]  # If this goes down, game over

# ✅ Active-Passive: health check triggers failover
servers = ["api-1.prod.internal", "api-2.standby.internal"]
active_server = servers[0]
standby_server = servers[1]

def get_active_server():
    if not is_healthy(active_server):
        print(f"Failing over from {active_server} to {standby_server}")
        active_server, standby_server = standby_server, active_server
        # Swap references so the standby becomes active
    return active_server
```

**The catch:** Failover isn't instant. You need health checks to detect failure, and there's a window where the service is down while the standby boots up. DNS propagation can make this worse if you're not using a load balancer.

**Good for:** Databases, stateful services, anything that needs a warm backup without the complexity of running both simultaneously.

## Active-Active (Multi-Primary)

Both servers handle traffic at the same time. If one fails, the other keeps going at 50% capacity (or whatever share it was handling).

```
      ┌──────────┐
      │  Load     │
      │  Balancer  │
      └─────┬────┘
        ┌───┴───┐
   ┌────┴────┐ ┌┴────────┐
   │  Node A  │ │  Node B  │
   │ (Active) │ │ (Active) │
   └─────────┘ └─────────┘
        │           │
        └───┬───┬───┘
            │ DB │
            └────┘
```

This is the workhorse pattern for stateless APIs. Spin up multiple instances, put a load balancer in front, and you get both redundancy _and_ scalability.

```yaml
# Kubernetes-style: 3 replicas, all serving
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3  # 🔁 If one pod dies, the other two keep serving
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: my-api:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

**The gotcha:** This works great for stateless services. But if your app has state (sessions, WebSocket connections, in-memory caches), you need to handle that. A user connected to Node A won't magically have their session on Node B if they get routed there.

Solutions: sticky sessions (but they're fragile), shared session stores (Redis), or make everything stateless.

## N+1 Redundancy

This is what most teams start with without realizing it has a name. "I've got 3 servers handling traffic. Let me add 1 more for safety."

N+1 means: if you need N instances to handle your peak load, run N+1. When one goes down, N others handle the load—at capacity, but still up.

```
Normal:  [A] [B] [C] [D]  ← (N=3, +1 spare)
Failure: [✗] [B] [C] [D]  ← 3 remaining, still handles peak
```

It's cheap, simple, and doesn't require any special architecture beyond what you already have. But there's a risk: **cascading failures**.

```
// ❌ The cascade scenario:
// 4 servers at 70% load each → total healthy
// 1 server dies → remaining 3 servers jump to 93% load
// 93% triggers GC pauses + timeouts
// 2nd server starts failing → remaining 2 servers at 140%
// Both servers die under overload
```

```python
# ✅ N+1 with graceful degradation and circuit breakers
from circuitbreaker import circuit

class NPlusOnePool:
    def __init__(self, min_servers: int, extra: int = 1):
        self.min_servers = min_servers  # N: what you need
        self.extra = extra              # +1: the safety margin
        self.servers = [start_server() for _ in range(min_servers + extra)]
    
    @circuit(failure_threshold=5, recovery_timeout=30)
    def handle_request(self, request):
        # Try servers in round-robin, skip unhealthy ones
        for server in self.servers:
            if server.is_healthy():
                return server.process(request)
        raise Exception("No healthy servers available (circuit open)")
```

**The fix:** Add proper load shedding. When a server goes down, the others should shed low-priority traffic (or just serve with degraded behavior) instead of blindly accepting everything.

## Database Redundancy: Primary-Replica

Stateless servers are easy to duplicate. Databases are the tricky part. Your app servers can be stateless, but the data needs to survive server failures.

The standard pattern: **one primary, multiple replicas**.

```
┌──────────┐     ┌──────────┐
│  Primary  │ ──→ │ Replica 1 │  (sync replication)
│ (writes)  │ ──→ ├──────────┤
└──────────┘     │ Replica 2 │  (async, for reads)
Writes go here    └──────────┘
                  Reads come from here
```

```sql
-- ❌ Single database, single point of failure
CONNECT TO db-primary.prod.internal:5432

-- ✅ Read/write splitting with replica failover
-- Write to primary
INSERT INTO orders (user_id, total) VALUES (42, 29.99);
-- Read from replica
SELECT * FROM products;  -- Goes to replica-1 or replica-2

-- If replica-1 fails: route reads to replica-2
-- If primary fails: promote replica-2 to primary
```

**Common setup:** PostgreSQL with streaming replication or MySQL with group replication. Write to the primary, read from replicas. If the primary dies, one of the replicas gets promoted.

> **⚠️ Replication lag is real.** Your replica might be 500ms behind the primary. If a user creates an order and immediately queries for it, they might not see it yet. For most apps this is fine. For payment processing—not so much.

### The Split-Brain Problem

When you have multiple nodes that can accept writes, and network partitions them, both sides might think the other is dead and promote themselves to primary. Now you've got two primaries accepting different writes.

```
Time: 01:23:45 - Network partition happens
  Node A on subnet 10.0.1.x  → "Node B is dead, I'm primary now"
  Node B on subnet 10.0.2.x  → "Node A is dead, I'm primary now"

Time: 01:23:50 - Both accept writes
  Node A: "User 42 ordered Item X - order_id: 5001"
  Node B: "User 42 ordered Item Y - order_id: 5001"  ← same order_id!

Time: 01:24:00 - Network recovers
  → Conflicting data that needs manual reconciliation
```

Solutions:
- **Majority quorum** (consensus-based systems like etcd, Raft)
- **Fencing** (a "kill switch" that shuts down the old primary before promoting)
- **Lease-based systems** (primary must renew a lease; if it can't, it steps down)

## Geographic Redundancy

The highest level of redundancy. You run your service in multiple data centers or cloud regions.

```
┌─────────────────┐       ┌─────────────────┐
│  us-east-1       │       │  eu-west-1       │
│ ┌─────────────┐ │       │ ┌─────────────┐ │
│ │ API Servers  │ │       │ │ API Servers  │ │
│ │ DB (Primary) │ │       │ │ DB (Replica) │ │
│ └─────────────┘ │       │ └─────────────┘ │
│  Load Balancer   │       │  Load Balancer   │
└────────┬────────┘       └────────┬────────┘
         └─────── Global DNS ──────┘
                     │
              ┌──────┴──────┐
              │   Clients    │
              └─────────────┘
```

Global DNS routing (Route53, Cloudflare, etc.) directs users to the nearest region. If an entire region goes down, DNS shifts traffic to the healthy one.

**The hard parts:**

- **Database sync across regions:** Latency between US-East to EU-West is ~80ms. Syncing every write synchronously would kill performance. Most teams use async replication.
- **Data sovereignty laws:** You might not be allowed to store EU user data in US servers (GDPR).
- **Cost:** You're paying for _everything_ twice. Or three times.

## How to Choose

| Your Situation | Start With | Upgrade To |
|---|---|---|
| Side project, one server | N+1 (just add a small backup) | Active-Passive |
| SaaS product, 99.9% target | Active-Active (2-3 instances) | Active-Active + DB replica |
| Enterprise, 99.99%+ target | Active-Active + DB replica | Geographic redundancy |
| Database-dependent app | Primary-Replica setup | Multi-region replication |

## Takeaways

- **Active-Passive is the simplest failover pattern** — one server stands by, takes over when the primary fails. Failover isn't instant but it's predictable
- **Active-Active gives you both redundancy and scalability** — works great for stateless services behind a load balancer
- **N+1 is cheap but needs load shedding** — when a server goes down, the rest must gracefully handle the extra load instead of cascading
- **Database redundancy is harder than app redundancy** — replication lag, split-brain, and failover orchestration are real problems
- **Geographic redundancy is the endgame** — entire-region failure protection, but costs double and adds significant complexity
- **Always test your failover** — redundancy you've never tested is just expensive wishful thinking
