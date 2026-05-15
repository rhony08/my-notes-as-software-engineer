# Service Discovery in Distributed Systems

You've got 10 microservices. Each one talks to three others. You deploy a new version of the auth service, and suddenly it's on a different IP. Now the gateway can't find it. Everything breaks.

This is the problem service discovery solves: **how does one service know where to find another?**

In a monolithic app, it's simple—everything runs in the same process, same address space. But once you go distributed, services move around. Containers restart, instances scale up and down, IPs change. Hardcoding addresses doesn't work.

## The Manual Approach (And Why It Hurts)

Let's start with what most people try first:

```yaml
# config.yml — the "please don't fail" approach
services:
  auth: http://192.168.1.50:3000
  payment: http://192.168.1.51:4000
  notification: http://192.168.1.52:5000
```

This works until:

- A container restarts on a different host → new IP
- You scale from 1 to 3 auth instances → which one do you call?
- You migrate to a new cluster → every config file needs updating

❌ **Hardcoded addresses** — brittle, manual, breaks on any infra change
✅ **Service discovery** — dynamic, auto-adapts, zero manual config updates

## The Core Idea

Service discovery is a directory service for your services. Instead of asking "what's the IP of auth?" at deploy time, you ask at runtime:

```
Gateway needs to call Auth →
  ❌ Hardcoded: http://192.168.1.50:3000
  ✅ Discovered: query registry → get "auth" → http://10.0.3.12:3000
```

The key difference is **dynamic resolution**. The registry always knows where healthy instances are.

## Two Flavors: Client-Side vs Server-Side

### Client-Side Discovery

The client (your service) directly queries the registry and picks an instance.

```
Service A → queries Registry → gets [IP1, IP2, IP3] → picks one → calls it
```

**How it works:**
1. Service A asks the registry "where's Service B?"
2. Registry returns a list of healthy B instances
3. Service A picks one (round-robin, random, least-loaded)
4. Makes the call directly

```python
# Client-side discovery with Consul
import consul
import requests

def call_service(service_name, endpoint):
    # Query the registry
    consul_client = consul.Consul(host='consul.service.consul')
    _, services = consul_client.health.service(service_name, passing=True)
    
    if not services:
        raise Exception(f"No healthy instances of {service_name}")
    
    # Pick one (simple round-robin)
    instance = services[current_index % len(services)]
    current_index += 1
    
    url = f"http://{instance['Service']['Address']}:{instance['Service']['Port']}/{endpoint}"
    return requests.get(url)
```

**Pros:**
- Fewer moving parts (no proxy between)
- Direct calls = lower latency
- Simple to understand

**Cons:**
- Every service needs discovery logic
- You're maintaining a client library per language
- Instance selection logic is scattered everywhere

### Server-Side Discovery

The client hits a load balancer, and the LB handles registry lookup + routing.

```
Service A → calls Load Balancer → LB queries Registry → routes to instance
```

**The classic example:** AWS ALB + target groups

```python
# Server-side discovery — client doesn't need to know anything
def call_service(endpoint):
    # Just call the load balancer DNS name
    url = f"http://internal-api-alb-123456.elb.amazonaws.com/{endpoint}"
    return requests.get(url)
    
# The ALB (server side) handles:
# 1. Health checks against registered instances
# 2. Routing to healthy targets
# 3. Load balancing across instances
# 4. Removing failed instances automatically
```

**Pros:**
- Clients are dumb (just make HTTP calls)
- Centralized routing logic
- Works with any client, any language

**Cons:**
- Extra hop adds latency (usually <5ms, but still)
- Load balancer can become a bottleneck
- More infrastructure to manage

| Approach | Latency | Complexity | Flexibility |
|----------|---------|------------|-------------|
| Client-side | Lower (direct call) | Higher (per-service) | Maximum |
| Server-side | Slightly higher (extra hop) | Lower (centralized) | Limited by LB |

## Popular Service Discovery Tools

### Consul

HashiCorp's Consul is probably the most popular dedicated service discovery tool.

```
// Register a service with Consul
{
  "service": {
    "name": "payment-api",
    "port": 4000,
    "checks": [
      {
        "name": "health check",
        "http": "http://localhost:4000/health",
        "interval": "10s",
        "timeout": "2s"
      }
    ],
    "tags": ["v2", "prod"]
  }
}
```

Consul uses **health checks** to track which instances are alive. If a health check fails 3 times, the instance is removed from the registry automatically. No manual cleanup needed.

**The trade-off:** Consul is another stateful service you need to run. It uses Raft consensus (like etcd), so you need 3 or 5 nodes for fault tolerance. That's infrastructure you're now responsible for.

### Kubernetes DNS

If you're on Kubernetes, you get service discovery built-in. No extra tools needed:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  selector:
    app: payment-api
  ports:
    - port: 80
      targetPort: 4000
```

K8s automatically creates DNS records: `payment-api.default.svc.cluster.local`. Any pod in the cluster can reach the service at that DNS name. K8s handles load balancing across pods.

**The trade-off:** Works great inside K8s. Terrible outside it. If you have services running outside the cluster, they can't resolve `*.svc.cluster.local` without extra DNS configuration.

### etcd / ZooKeeper

These are distributed key-value stores that you can use for service discovery. They're the building blocks—you implement the discovery logic yourself.

```go
// etcd-based service registration (Go)
func Register(serviceName, addr string, ttl int64) error {
    lease, err := cli.Grant(context.Background(), ttl)
    if err != nil {
        return err
    }
    
    key := fmt.Sprintf("/services/%s/%s", serviceName, addr)
    _, err = cli.Put(context.Background(), key, addr, 
        clientv3.WithLease(lease.ID))
    
    // Keep the lease alive (heartbeat)
    ch, err := cli.KeepAlive(context.Background(), lease.ID)
    go func() {
        for range ch {}  // consume keepalive responses
    }()
    
    return err
}
```

**The trade-off:** More flexibility, more work. You're building discovery from primitives. Only worth it if you have specific requirements that Consul or K8s DNS can't meet.

## Handling the Tricky Stuff

### Service Registration

Two approaches:

**Self-registration:** The service registers itself on startup and deregisters on shutdown.
```
Service starts → POST /v1/agent/service/register (Consul API)
Service stops → POST /v1/agent/service/deregister
```

**Third-party registration:** An external watcher detects new instances and registers them.
```
K8s watcher → detects new pod → registers with external Consul
```

Self-registration is simpler. Third-party is more reliable (even if the service crashes before deregistering, the health check will catch it).

### Health Checks Matter

Without health checks, service discovery is just a phone book with dead people in it.

```python
# Bad: registration without health checking
{
  "service": { "name": "api", "address": "10.0.1.5" }
  # No health check — if this goes down, traffic still goes there
}

# Good: registration + health check
{
  "service": { "name": "api", "address": "10.0.1.5" },
  "check": {
    "http": "http://10.0.1.5/health",
    "interval": "10s",
    "deregister_critical_service_after": "60s"
  }
}
```

The `deregister_critical_service_after` setting is crucial. It says "if this service has been unhealthy for 60 seconds, remove it entirely." Without this, your registry fills up with dead instances.

### Caching: Don't Query Every Time

Querying the registry on every request adds latency and load. **Cache the results.**

```python
import time
from functools import lru_cache

class ServiceCache:
    def __init__(self, ttl_seconds=30):
        self.cache = {}
        self.ttl = ttl_seconds
    
    def get_instances(self, service_name):
        if service_name in self.cache:
            instances, timestamp = self.cache[service_name]
            if time.time() - timestamp < self.ttl:
                return instances
        
        # Cache miss — query the registry
        instances = query_consul(service_name)
        self.cache[service_name] = (instances, time.time())
        return instances
```

30 seconds TTL is a good default. Low enough that changes propagate reasonably fast. High enough that you're not hammering the registry.

## What Service Discovery Doesn't Solve

Service discovery tells you *where* a service is. It doesn't handle:

- **Authentication** — "Is this instance actually my payment service?" Add mTLS for that.
- **Load balancing** — Discovery gives you a list; you still need to pick one wisely.
- **Circuit breaking** — If an instance is failing, discovery alone won't stop calling it. You need circuit breakers on top.
- **Configuration** — Discovery is about *location*, not *settings*. Keep config separate (ConfigMaps, Vault, etc.).

## Actionable Takeaways

1. **Use DNS-based discovery (K8s DNS or Consul DNS) as your default** — it's the simplest, works with existing code, and requires no client libraries
2. **Always pair service discovery with health checks** — without health checks, you're just routing to dead instances
3. **Cache registry lookups** — querying every request is wasteful; 30-second TTL is usually fine
4. **Set `deregister_critical_service_after`** — dead instances should be cleaned up, not left in the registry
5. **If you're on Kubernetes, use built-in Service DNS** — there's no reason to run Consul on top of K8s unless you have services outside the cluster
6. **For hybrid infra (K8s + VMs + bare metal), use a dedicated tool** — Consul or etcd with proper DNS integration
