# Health Checks and Readiness Probes

When you deploy a backend service, how do you know it's actually working? A process might be running but stuck in an infinite loop, deadlocked, or unable to connect to its database. Health checks solve this problem by providing a standardized way for infrastructure and monitoring systems to verify your application is functioning correctly.

## Table of Contents

- [What is a Health Check?](#what-is-a-health-check)
- [Health Check Endpoints](#health-check-endpoints)
- [Types of Health Checks](#types-of-health-checks)
- [Readiness vs Liveness Probes](#readiness-vs-liveness-probes)
- [Implementing Health Checks](#implementing-health-checks)
- [Kubernetes Probes](#kubernetes-probes)
- [Load Balancer Health Checks](#load-balancer-health-checks)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## What is a Health Check?

A health check is an endpoint or mechanism that reports whether a service is operational. It's the "pulse check" of your application—answering the question: "Can this service handle requests?"

### Why Health Checks Matter

**1. Zero-Downtime Deployments**

When rolling out a new version, load balancers and orchestration systems need to know when a service is ready to accept traffic:

```
Old version: Still healthy → Keep receiving traffic
New version: Starting → Not ready → Don't send traffic
New version: Healthy → Ready → Start receiving traffic
Old version: Draining → Stop receiving traffic
```

**2. Automatic Recovery**

When a service becomes unhealthy, orchestration platforms can:
- Restart the container
- Route traffic to healthy instances
- Alert operators

**3. Monitoring and Alerting**

Health check failures trigger alerts before users notice problems:
```
Service: payment-api
Health: UNHEALTHY
Reason: Database connection pool exhausted
Action: Alert on-call engineer
```

## Health Check Endpoints

The most common implementation is an HTTP endpoint that returns a status.

### Basic Health Check

```javascript
// Express.js
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});
```

```python
# FastAPI
@app.get("/health")
async def health():
    return {"status": "healthy"}
```

```go
// Gin
router.GET("/health", func(c *gin.Context) {
    c.JSON(200, gin.H{"status": "healthy"})
})
```

### Response Codes Convention

| Status Code | Meaning |
|-------------|---------|
| 200 OK | Service is healthy |
| 503 Service Unavailable | Service is unhealthy |

Avoid using redirect codes or non-standard responses—orchestration tools expect simple 200/503 responses.

## Types of Health Checks

### 1. Shallow Health Check (Liveness)

Checks if the process is running and can respond to HTTP requests.

```javascript
app.get('/health/live', (req, res) => {
  // Just checking if the process is alive
  res.status(200).json({ status: 'alive' });
});
```

**Pros:** Fast, no dependencies
**Cons:** Doesn't verify the service can actually do work

**Use for:** Detecting deadlocks, hung processes, or crashes

### 2. Deep Health Check (Readiness)

Verifies all dependencies are available and the service can process requests.

```javascript
app.get('/health/ready', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    externalApi: await checkExternalApi()
  };
  
  const allHealthy = Object.values(checks).every(c => c.status === 'ok');
  
  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'healthy' : 'unhealthy',
    checks
  });
});

async function checkDatabase() {
  try {
    await db.query('SELECT 1');
    return { status: 'ok' };
  } catch (error) {
    return { status: 'error', message: error.message };
  }
}

async function checkRedis() {
  try {
    await redis.ping();
    return { status: 'ok' };
  } catch (error) {
    return { status: 'error', message: error.message };
  }
}
```

**Pros:** Validates actual functionality
**Cons:** Slower, can cause cascading failures if not careful

### 3. Startup Probe

Checks if the application has finished starting up. Used to delay other probes during initialization.

```javascript
let isReady = false;

// During startup
async function initialize() {
  await connectDatabase();
  await warmCache();
  await loadConfig();
  isReady = true;
}

app.get('/health/startup', (req, res) => {
  if (isReady) {
    res.status(200).json({ status: 'started' });
  } else {
    res.status(503).json({ status: 'starting' });
  }
});
```

## Readiness vs Liveness Probes

Understanding the difference is critical for production reliability.

### Liveness Probe: "Should this container be restarted?"

**Purpose:** Detect unrecoverable states
- Deadlocks
- Memory corruption
- Infinite loops

**Action on failure:** Restart the container

```yaml
# Kubernetes liveness probe
livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

If this fails 3 times, Kubernetes restarts the container.

### Readiness Probe: "Should this container receive traffic?"

**Purpose:** Detect temporary unavailability
- Database connection lost
- Cache warming in progress
- Dependent service down

**Action on failure:** Stop sending traffic (don't restart)

```yaml
# Kubernetes readiness probe
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

If this fails, Kubernetes removes the pod from service endpoints but doesn't restart it.

### Key Difference

| Scenario | Liveness | Readiness |
|----------|----------|-----------|
| Database connection lost | No restart needed | Stop traffic |
| Memory leak / deadlock | Restart | N/A |
| Slow startup | N/A | Don't send traffic yet |
| External API slow | No restart | Maybe stop traffic |

## Implementing Health Checks

### Express.js Implementation

```javascript
const express = require('express');
const app = express();

// Track application state
let isShuttingDown = false;
let dbConnected = false;
let redisConnected = false;

// Graceful shutdown flag
process.on('SIGTERM', () => {
  console.log('Received SIGTERM, starting graceful shutdown');
  isShuttingDown = true;
});

// Liveness - just check if process is responsive
app.get('/health/live', (req, res) => {
  if (isShuttingDown) {
    // During shutdown, return 503 so traffic stops
    return res.status(503).json({ status: 'shutting down' });
  }
  res.status(200).json({ status: 'alive' });
});

// Readiness - check if we can handle requests
app.get('/health/ready', async (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting down' });
  }
  
  const checks = {
    database: dbConnected ? 'ok' : 'error',
    redis: redisConnected ? 'ok' : 'error'
  };
  
  const healthy = dbConnected && redisConnected;
  
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks
  });
});

// Combined endpoint for simple setups
app.get('/health', (req, res) => {
  res.redirect('/health/ready');
});
```

### FastAPI Implementation

```python
from fastapi import FastAPI, Response
from pydantic import BaseModel
import asyncio

app = FastAPI()

class HealthStatus(BaseModel):
    status: str
    checks: dict = None

# Application state
app_state = {
    "is_shutting_down": False,
    "db_connected": False,
    "redis_connected": False
}

@app.get("/health/live")
async def liveness():
    if app_state["is_shutting_down"]:
        return Response(content='{"status":"shutting down"}', 
                       status_code=503,
                       media_type="application/json")
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    if app_state["is_shutting_down"]:
        return Response(content='{"status":"shutting down"}',
                       status_code=503,
                       media_type="application/json")
    
    checks = {
        "database": "ok" if app_state["db_connected"] else "error",
        "redis": "ok" if app_state["redis_connected"] else "error"
    }
    
    healthy = all(v == "ok" for v in checks.values())
    
    if not healthy:
        return Response(
            content=f'{{"status":"unhealthy","checks":{checks}}}',
            status_code=503,
            media_type="application/json"
        )
    
    return {"status": "healthy", "checks": checks}
```

### Go Implementation

```go
package main

import (
    "net/http"
    "sync"
    
    "github.com/gin-gonic/gin"
)

type HealthChecker struct {
    mu             sync.RWMutex
    shuttingDown   bool
    dbHealthy      bool
    redisHealthy   bool
}

func (h *HealthChecker) Liveness(c *gin.Context) {
    h.mu.RLock()
    defer h.mu.RUnlock()
    
    if h.shuttingDown {
        c.JSON(503, gin.H{"status": "shutting down"})
        return
    }
    c.JSON(200, gin.H{"status": "alive"})
}

func (h *HealthChecker) Readiness(c *gin.Context) {
    h.mu.RLock()
    defer h.mu.RUnlock()
    
    if h.shuttingDown {
        c.JSON(503, gin.H{"status": "shutting down"})
        return
    }
    
    checks := map[string]string{
        "database": "ok",
        "redis":   "ok",
    }
    
    if !h.dbHealthy {
        checks["database"] = "error"
    }
    if !h.redisHealthy {
        checks["redis"] = "error"
    }
    
    healthy := h.dbHealthy && h.redisHealthy
    
    status := 200
    if !healthy {
        status = 503
    }
    
    c.JSON(status, gin.H{
        "status": map[bool]string{true: "healthy", false: "unhealthy"}[healthy],
        "checks": checks,
    })
}
```

## Kubernetes Probes

Kubernetes has built-in support for health checks through probes.

### Complete Pod Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
      - name: api
        image: my-api:v1.0.0
        ports:
        - containerPort: 3000
        
        # Startup probe - give the app time to start
        startupProbe:
          httpGet:
            path: /health/startup
            port: 3000
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 30  # 30 * 5 = 150s max startup time
        
        # Liveness probe - restart if unhealthy
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Readiness probe - don't send traffic if unhealthy
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          periodSeconds: 5
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
```

### Probe Types

| Probe | Purpose | Action on Failure |
|-------|---------|-------------------|
| `startupProbe` | App initialization | Don't start other probes yet |
| `livenessProbe` | Deadlock/crash detection | Restart container |
| `readinessProbe` | Traffic routing | Remove from service |

### Probe Parameters

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
    httpHeaders:              # Optional headers
    - name: X-Health-Check
      value: "true"
  
  initialDelaySeconds: 5     # Wait before first check
  periodSeconds: 10          # Check every 10 seconds
  timeoutSeconds: 5          # Timeout for response
  failureThreshold: 3        # Failures before unhealthy
  successThreshold: 1        # Successes before healthy
```

### Alternative Probe Types

**TCP Probe:**
```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  periodSeconds: 10
```

**Command Probe:**
```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "pg_isready -h localhost"
  periodSeconds: 10
```

## Load Balancer Health Checks

Load balancers (AWS ALB, Nginx, HAProxy) also use health checks.

### AWS Application Load Balancer

```yaml
# Target Group Configuration
HealthCheckPath: /health
HealthCheckIntervalSeconds: 30
HealthCheckTimeoutSeconds: 5
HealthyThresholdCount: 2
UnhealthyThresholdCount: 3
Matcher:
  HttpCode: 200
```

### Nginx Upstream Health Check

```nginx
upstream backend {
  server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
  server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
  
  # Active health check (requires nginx-plus or openresty)
  check interval=5000 rise=2 fall=3 timeout=1000 type=http;
  check_http_send "GET /health HTTP/1.0\r\n\r\n";
  check_http_expect_alive http_2xx http_3xx;
}
```

### HAProxy Health Check

```
backend myapi
  balance roundrobin
  option httpchk GET /health
  http-check expect status 200
  server app1 10.0.0.1:3000 check
  server app2 10.0.0.2:3000 check
```

## Best Practices

### 1. Keep Liveness Checks Simple

```javascript
// ✅ Good - fast and simple
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// ❌ Bad - includes dependency checks
app.get('/health/live', async (req, res) => {
  await db.query('SELECT 1');  // Don't do this
  await redis.ping();          // Don't do this
  res.status(200).json({ status: 'alive' });
});
```

**Why:** If your database is slow, liveness probe fails, container restarts, but database is still slow—causing a restart loop.

### 2. Use Meaningful Response Details

```javascript
app.get('/health/ready', async (req, res) => {
  const checks = {
    database: { status: 'ok', latencyMs: 5 },
    redis: { status: 'ok', latencyMs: 2 },
    externalApi: { status: 'degraded', latencyMs: 2500 }
  };
  
  res.status(200).json({
    status: 'healthy',
    version: process.env.APP_VERSION,
    uptime: process.uptime(),
    checks
  });
});
```

### 3. Implement Graceful Shutdown

```javascript
const server = app.listen(3000);

process.on('SIGTERM', async () => {
  console.log('Shutting down gracefully...');
  
  // Stop accepting new connections
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Close database connections
  await db.close();
  await redis.quit();
  
  process.exit(0);
});
```

### 4. Set Appropriate Timeouts

```yaml
# Kubernetes
livenessProbe:
  timeoutSeconds: 5      # Should be fast
  failureThreshold: 3    # 3 failures = restart

readinessProbe:
  timeoutSeconds: 10     # Can be longer for deep checks
  failureThreshold: 3    # 3 failures = stop traffic
```

### 5. Don't Check External Dependencies in Liveness

```
Liveness Probe: Is my process alive?
Readiness Probe: Can I handle requests?

External APIs: May be temporarily slow/down
- Don't restart your container because someone else is slow
- Use readiness to route away traffic instead
```

### 6. Include Health Check in All Services

Every service should have health checks, even "internal" ones:

```yaml
# Even internal services need health checks
services:
  internal-metrics:
    healthCheck: /health
    # If this fails, dependent services can react
```

### 7. Use a Dedicated Health Check Port (Optional)

Separate health checks from application traffic:

```javascript
// Main app on port 3000
const app = express();
app.listen(3000);

// Health on port 3001 (internal only)
const healthApp = express();
healthApp.get('/health', (req, res) => res.json({ status: 'ok' }));
healthApp.listen(3001);
```

**Benefits:**
- Health checks not affected by app load
- Can bind to different network interface
- Simpler monitoring

## Conclusion

Health checks are fundamental to reliable distributed systems:

- **Liveness probes** detect when a container needs to be restarted
- **Readiness probes** determine if a container should receive traffic
- **Startup probes** handle slow-starting applications

By implementing proper health checks, you enable:

- Zero-downtime deployments
- Automatic recovery from failures
- Better monitoring and alerting
- Smoother scaling operations

Remember: A simple `/health` endpoint returning `200 OK` is infinitely better than no health check at all. Start simple, then add depth as your needs evolve.