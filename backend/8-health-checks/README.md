# Health Checks and Readiness Probes

Your service is running. Containers are up. But is it actually *ready* to handle requests?

At 3am, you get paged. The load balancer is sending traffic to your freshly deployed pods, but they're still warming up caches and connecting to databases. Users see 500 errors for 30 seconds after every deployment. Kubernetes *thinks* everything's fine—it just doesn't know the difference between "running" and "ready."

Health checks are how you tell the truth. Let's sort out liveness vs readiness, and how to implement them without shooting yourself in the foot.

---

## The Problem: Running ≠ Ready

Containers and orchestrators have a simple view of the world: process exists = healthy. But your app needs time to:

- Establish database connections
- Warm up caches
- Load configuration
- Connect to message queues
- Complete initialization logic

Without proper health checks, your orchestrator will route traffic to containers that aren't prepared, causing errors and failed requests.

---

## Liveness vs Readiness: They're Different Things

| Probe Type | Purpose | Failure Behavior |
|------------|---------|------------------|
| **Liveness** | "Is this container dead?" | Restart the container |
| **Readiness** | "Can this container handle traffic?" | Stop sending traffic, keep running |

**Liveness** catches deadlocks, infinite loops, or stuck processes. If it fails, Kubernetes kills and restarts your pod.

**Readiness** catches initialization, dependency issues, or overload. If it fails, Kubernetes stops routing traffic but lets the container run.

```yaml
# Kubernetes probe configuration
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30   # Wait before first check
  periodSeconds: 10         # Check every 10 seconds
  timeoutSeconds: 5         # Fail if no response in 5 seconds
  failureThreshold: 3       # Restart after 3 failures

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3       # Stop traffic after 3 failures
```

---

## What to Check

### Liveness: Keep It Simple

Your liveness endpoint should answer one question: "Can this process respond to HTTP requests?"

```python
# ✅ Good liveness check - minimal dependencies
@app.get("/health/live")
async def liveness():
    return {"status": "alive"}
```

```python
# ❌ Bad liveness check - too many dependencies
@app.get("/health/live")
async def liveness():
    # If database is slow, this will timeout and restart the pod
    # But the database being slow doesn't mean the pod is dead!
    await db.execute("SELECT 1")
    await redis.ping()
    await external_api.check()
    return {"status": "alive"}
```

**Why:** If your liveness check depends on external services, you might restart healthy pods when those services are slow. That makes problems worse.

### Readiness: Check Everything You Need

Your readiness endpoint should verify all dependencies needed to serve requests:

```python
# ✅ Good readiness check - verify dependencies
@app.get("/health/ready")
async def readiness():
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
        "cache": cache.is_warmed()
    }
    
    all_healthy = all(checks.values())
    
    if not all_healthy:
        # Return 503 so orchestrator knows to back off
        raise HTTPException(503, detail=checks)
    
    return {"status": "ready", "checks": checks}
```

---

## Startup Probes: For Slow Starters

Some apps need a long time to initialize. Maybe you're loading ML models, warming large caches, or running migrations. Liveness probes will kill your pod before it finishes starting.

**Startup probes** solve this: they disable liveness checks until the app is fully started.

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30      # Allow up to 5 minutes (30 * 10s)
  
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 0    # Will wait for startup probe first
  periodSeconds: 10
  failureThreshold: 3
```

---

## Common Mistakes

### ❌ Health Checks That Lie

```python
# Always returns 200 even when broken
@app.get("/health")
async def health():
    return {"status": "ok"}  # Does this actually check anything?
```

This is worse than no health check. You're telling the orchestrator everything's fine when it's not.

### ❌ Checking External Dependencies in Liveness

```python
# If external API is down, your pods get restarted
@app.get("/health/live")
async def liveness():
    await external_payment_api.ping()  # DO NOT DO THIS
    return {"status": "alive"}
```

Result: Payment API has issues → your pods restart → more traffic to already-stressed API → cascade failure.

### ❌ Too Aggressive Timeouts

```yaml
livenessProbe:
  timeoutSeconds: 1    # Too short for realistic workloads
```

If your app is under load and responses take 1.5 seconds, your pods will restart repeatedly. Set realistic timeouts.

### ❌ No Graceful Shutdown

```python
# When readiness fails, stop accepting new requests
# but finish in-flight requests before dying
@app.on_event("shutdown")
async def shutdown():
    logger.info("Shutting down, draining connections...")
    await drain_connections(timeout=30)
```

---

## Implementation Patterns

### Structured Health Responses

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class HealthStatus:
    status: str
    timestamp: str
    version: str
    checks: dict

@app.get("/health/ready")
async def readiness():
    checks = {
        "database": await db.health_check(),
        "redis": await redis.ping(),
        "external_api": await external_api.health_check()
    }
    
    return HealthStatus(
        status="healthy" if all(checks.values()) else "degraded",
        timestamp=datetime.utcnow().isoformat(),
        version=os.getenv("APP_VERSION", "unknown"),
        checks=checks
    )
```

### Startup Logic

```python
class StartupState:
    is_ready = False
    
    async def mark_ready(self):
        self.is_ready = True

startup_state = StartupState()

@app.on_event("startup")
async def startup():
    await connect_database()
    await warm_cache()
    startup_state.is_ready = True

@app.get("/health/ready")
async def readiness():
    if not startup_state.is_ready:
        raise HTTPException(503, detail="Still initializing")
    return {"status": "ready"}
```

---

## Load Balancer Health Checks

If you're behind a load balancer (ALB, nginx, Cloudflare), it also needs to know when your app is unhealthy:

```
# nginx upstream health check
upstream backend {
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
}
```

```
# AWS ALB target group health check
HealthCheckPath: /health/ready
HealthCheckIntervalSeconds: 30
HealthCheckTimeoutSeconds: 5
HealthyThresholdCount: 2
UnhealthyThresholdCount: 3
```

Match your orchestrator and load balancer settings to avoid conflicting behaviors.

---

## Monitoring Your Health Checks

Health check failures should be observable:

- **Metrics:** Track `/health/*` response times and status codes
- **Logs:** Log when readiness fails and why
- **Alerts:** Alert on repeated liveness failures (pods restarting)

```python
@app.get("/health/ready")
async def readiness():
    checks = await run_health_checks()
    
    if not all(checks.values()):
        logger.warning("Readiness check failed", extra={"checks": checks})
        metrics.increment("health.ready.failed")
        raise HTTPException(503, detail=checks)
    
    return {"status": "ready"}
```

---

## Quick Reference

| Check Type | Endpoint | Dependencies | Failure = |
|------------|----------|--------------|-----------|
| Liveness | `/health/live` | None | Restart pod |
| Readiness | `/health/ready` | All required | Stop traffic |
| Startup | `/health/startup` | None (just initialization) | Keep waiting |

---

## Takeaways

1. **Liveness checks if your process is dead** — don't check external dependencies
2. **Readiness checks if you can serve traffic** — verify all dependencies
3. **Startup probes buy you time** for slow initialization
4. **Return proper status codes** — 200 for healthy, 503 for not ready
5. **Log and monitor failures** — health checks are your early warning system
6. **Test your health checks** — they're critical infrastructure, treat them that way