# Load Balancing: Distributing Traffic Like a Pro

Your server just crashed. Not because of a bug—because it couldn't handle the traffic spike. One server, whether it's handling 100 requests or 10,000, has a ceiling. When you hit it, everything goes down.

Load balancing is the art of spreading that load across multiple servers so no single machine becomes a bottleneck. We'll cover the algorithms that decide *how* traffic gets distributed, when to use each, and the gotchas that'll bite you in production.

---

## Why Load Balancing Matters

Without a load balancer, you're putting all your eggs in one basket:

- **Single point of failure** — Server dies? Everything's down
- **Uneven resource usage** — Some servers idle while others sweat
- **No graceful scaling** — Adding capacity means DNS changes and praying for TTL

A load balancer sits between clients and your servers, acting as traffic cop. Clients talk to the load balancer; the load balancer decides which backend gets the request.

```
Client A ────┐
Client B ────┼──→ [Load Balancer] ──→ Server 1
Client C ────┤                        Server 2
Client D ────┘                        Server 3
```

---

## Core Algorithms Explained

### Round Robin

The simplest approach: take turns.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (cycle repeats)
```

**Pros:**
- Dead simple to implement
- Works great when all servers have similar capacity
- No state to track (stateless)

**Cons:**
- Ignores server load—Server A could be drowning while Server B is idle
- Doesn't account for request complexity (a heavy API call costs the same as a lightweight one)

```nginx
# Nginx round robin (default)
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

**When to use:** Homogeneous servers with similar request patterns. Your "hello world" apps and simple CRUD APIs.

---

### Weighted Round Robin

Same as round robin, but servers get "votes" based on their capacity.

```
Server A: weight 5  → gets 5 requests
Server B: weight 3  → gets 3 requests
Server C: weight 1  → gets 1 request
Pattern: A A A A A B B B C (repeat)
```

**Use case:** You've got a mix of hardware. Your beefy 32-core server should handle more traffic than the legacy 4-core box.

```nginx
upstream backend {
    server 10.0.0.1:8080 weight=5;  # 5x traffic
    server 10.0.0.2:8080 weight=3;  # 3x traffic
    server 10.0.0.3:8080 weight=1;  # 1x traffic
}
```

**Gotcha:** Weights are static. If Server A starts struggling, the load balancer won't know—it'll keep sending 5x traffic.

---

### Least Connections

Route to the server with fewest active connections.

```
Server A: 47 connections
Server B: 12 connections  ← gets the next request
Server C: 89 connections
```

**Pros:**
- Responds to real-time load
- Handles heterogeneous request durations well

**Cons:**
- Requires tracking connection state (memory overhead)
- Doesn't account for server capacity—a weak server with few connections will still get hammered

```nginx
upstream backend {
    least_conn;  # Least connections algorithm
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

**When to use:** When request durations vary wildly (think file uploads, streaming, long-running queries). Long requests shouldn't block short ones from being distributed.

---

### IP Hash

Hash the client's IP address and use it to determine the server.

```
Client IP 192.168.1.100 → hash → Server A (always)
Client IP 10.0.0.50     → hash → Server B (always)
Client IP 172.16.0.25   → hash → Server C (always)
```

**The good:** Session stickiness without cookies. Same client always hits the same server.

**The bad:**
- Uneven distribution—some IPs might generate tons of traffic
- When a server goes down, its hash bucket redistributes, breaking stickiness
- Doesn't work well with IPv6 or NAT'd clients (everyone behind corporate firewall = same IP)

```nginx
upstream backend {
    ip_hash;  # Sticky sessions by IP
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

**When to use:** Legacy apps that store session state locally. But honestly, you should probably fix the session storage instead.

---

### Least Response Time

Route to the server with the fastest response time *and* fewest active connections. It's like least connections, but smarter.

```
Server A: 12 connections, avg response 450ms
Server B: 8 connections, avg response 50ms   ← winner
Server C: 5 connections, avg response 200ms
```

**When to use:** When you care about latency. API gateways, real-time systems, anything user-facing where milliseconds matter.

**Note:** This requires the load balancer to track response times per server, which adds overhead.

---

### Random

Just... pick one randomly.

```
Request arrives → roll dice → Server B
```

Surprisingly effective at scale. With enough requests, distribution converges to uniform.

**Pros:**
- Zero coordination overhead
- Scales infinitely (no shared state)
- Works across multiple load balancers

**Cons:**
- Variance at low traffic volumes
- Doesn't account for server differences

**When to use:** Distributed systems where multiple load balancers need to work independently. Combined with health checks, it's robust.

---

## Algorithm Comparison

| Algorithm | State Required | Session Affinity | Best For |
|-----------|---------------|------------------|----------|
| Round Robin | None | ❌ | Equal servers, uniform requests |
| Weighted Round Robin | None | ❌ | Heterogeneous servers |
| Least Connections | Per-server count | ❌ | Varied request durations |
| IP Hash | None | ✅ | Sticky sessions (legacy apps) |
| Least Response Time | Per-server metrics | ❌ | Latency-sensitive APIs |
| Random | None | ❌ | Distributed load balancers |

---

## Health Checks: The Missing Piece

None of these algorithms matter if you're routing traffic to dead servers.

Load balancers need health checks—periodic pings to verify servers are alive:

```
Every 5 seconds:
  GET /health on Server A → 200 OK ✓
  GET /health on Server B → timeout ✗ (mark as down)
  GET /health on Server C → 503 ✗ (mark as down)
```

### Active vs Passive Health Checks

**Active:** Load balancer actively polls servers

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
}
# If 3 requests fail within 30s, server is marked down for 30s
```

**Passive:** Load balancer watches real traffic

```nginx
# Nginx passive check
proxy_next_upstream error timeout http_502 http_503 http_504;
# Automatically retry on another server if upstream fails
```

### What Makes a Good Health Check?

```javascript
// ❌ Terrible health check - always returns 200
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// ✅ Meaningful health check
app.get('/health', async (req, res) => {
  try {
    await database.ping();          // Check DB connection
    await redis.ping();             // Check cache
    res.status(200).json({ 
      status: 'healthy',
      db: 'connected',
      cache: 'connected'
    });
  } catch (err) {
    res.status(503).json({ 
      status: 'unhealthy',
      error: err.message 
    });
  }
});
```

**The key:** Health checks should verify *dependencies*, not just the process. Your app returning 200 while the database is down isn't "healthy."

---

## Graceful Server Removal

When you need to take a server offline (deploy, maintenance, scaling down):

```
1. Stop sending NEW traffic (mark as draining)
2. Wait for existing connections to complete
3. Once connections hit 0, safe to kill
```

```nginx
# Mark server as draining (Nginx Plus)
# curl -X POST http://localhost/api/4/upstreams/backend/servers/1?drain=true

# HAProxy
server backend1 10.0.0.1:8080 state drain
```

Without graceful draining, active users get disconnected mid-request. That's a bad experience.

---

## Load Balancer Placement Strategies

### Layer 4 (Transport Layer)

Routes based on IP and port. Doesn't inspect packet contents.

```
Client IP:Port → Load Balancer → Backend IP:Port
```

**Pros:**
- Fast (no packet inspection)
- Works for any protocol
- Lower latency

**Cons:**
- Can't route based on URL, headers, or content
- No SSL termination at load balancer

**Use case:** Database load balancing, TCP services, maximum performance.

### Layer 7 (Application Layer)

Routes based on HTTP path, headers, cookies, query params.

```
/api/users    → User service
/api/orders   → Order service
/api/payments → Payment service
```

**Pros:**
- Content-based routing
- SSL termination
- URL rewrites, redirects, caching

**Cons:**
- Higher latency (must parse requests)
- More resource-intensive

**Use case:** Microservices, API gateways, web applications.

```nginx
# Layer 7 routing example
upstream users {
    server 10.0.0.1:8080;
}

upstream orders {
    server 10.0.0.2:8080;
}

server {
    location /api/users {
        proxy_pass http://users;
    }
    
    location /api/orders {
        proxy_pass http://orders;
    }
}
```

---

## Common Gotchas

### 1. Thundering Herd

When a server comes back online after being down, ALL waiting requests flood it simultaneously.

**Solution:** Add jitter to health check intervals, or use slow-start mode:

```nginx
server 10.0.0.1:8080 slow_start=30s;  # Gradually ramp up traffic
```

### 2. Connection Pool Exhaustion

If backend servers are slow, the load balancer runs out of connections waiting for responses.

```
Max connections: 1000
All 1000 waiting on slow backends → new requests rejected
```

**Solution:** Set aggressive timeouts and connection limits.

```nginx
proxy_connect_timeout 5s;
proxy_read_timeout 10s;
proxy_send_timeout 10s;
```

### 3. Sticky Session Pitfalls

IP hash breaks when:
- Server count changes (rehash everything)
- Clients are behind NAT (everyone looks like same IP)
- IPv6 privacy addresses rotate

**Better approach:** Use consistent hashing with a proper session store.

### 4. Ignoring Upstream Latency

Load balancers add a hop. If you're routing across regions, that's measurable latency.

```
Client (US) → LB (EU) → Server (US) → LB (EU) → Client (US)
```

**Solution:** Deploy load balancers close to your servers, not close to clients (unless you're doing global anycast).

---

## Real-World Architecture

Here's how a production setup typically looks:

```
                    ┌─────────────────┐
                    │   DNS (Route53) │
                    │  Round Robin    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                              ▼
     ┌────────────────┐            ┌────────────────┐
     │  LB (Region A) │            │  LB (Region B) │
     │  Layer 7, SSL  │            │  Layer 7, SSL  │
     └───────┬────────┘            └───────┬────────┘
             │                              │
     ┌───────┴───────┐              ┌───────┴───────┐
     ▼               ▼              ▼               ▼
┌─────────┐   ┌─────────┐    ┌─────────┐   ┌─────────┐
│ App Svr │   │ App Svr │    │ App Svr │   │ App Svr │
│ (Zone 1)│   │ (Zone 2)│    │ (Zone 1)│   │ (Zone 2)│
└─────────┘   └─────────┘    └─────────┘   └─────────┘
```

**Multiple layers:**
1. **DNS load balancing** — Geographic distribution
2. **L7 load balancer** — SSL termination, routing, rate limiting
3. **Service mesh** — Internal load balancing between services

---

## Actionable Takeaways

1. **Start with round robin** for simple apps—it's usually good enough
2. **Add health checks immediately**—no excuses for routing to dead servers
3. **Use least connections** when request durations vary significantly
4. **Implement graceful draining** before you need it (deploy time isn't the time to figure this out)
5. **Layer 7 for microservices** (content-based routing), Layer 4 for databases
6. **Monitor your load balancer**—it's a critical piece of infrastructure
7. **Don't use IP hash for session stickiness** unless you have no other choice; fix your session storage instead

The best load balancing strategy is the one that stays invisible—users never notice it's there, and when servers fail, traffic just keeps flowing.