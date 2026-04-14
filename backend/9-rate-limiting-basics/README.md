# Rate Limiting: Protecting Your APIs

Every API has limits—server resources, database connections, third-party quotas. Without rate limiting, a single misbehaving client can overwhelm your entire system. Rate limiting is your first line of defense, ensuring fair resource distribution and protecting your infrastructure from abuse. In this article, we'll explore why rate limiting matters, common algorithms, and how to implement it effectively.

## Table of Contents

- [Why Rate Limiting Matters](#why-rate-limiting-matters)
- [Rate Limiting vs Throttling](#rate-limiting-vs-throttling)
- [Common Algorithms](#common-algorithms)
- [Identifying Clients](#identifying-clients)
- [HTTP Headers and Responses](#http-headers-and-responses)
- [Implementation Examples](#implementation-examples)
- [Distributed Rate Limiting](#distributed-rate-limiting)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## Why Rate Limiting Matters

### Protecting Resources

Your servers have finite capacity. Without limits:

- A buggy client loop can DDOS your API
- A malicious actor can scrape all your data
- One heavy user can degrade experience for everyone
- Third-party API quotas get exhausted

### Ensuring Fairness

Rate limiting enforces the principle that all users get fair access:

```
User A: 10 requests/second  ✅ Within limit
User B: 1000 requests/second ❌ Exceeds limit → Rejected
```

### Controlling Costs

Many backend services charge per request:

- Database queries (AWS RDS, MongoDB Atlas)
- Third-party APIs (OpenAI, Stripe, Twilio)
- Cloud functions (AWS Lambda, Cloudflare Workers)

Rate limiting prevents runaway costs from bugs or abuse.

### Preventing Abuse

Common attack patterns that rate limiting mitigates:

| Attack Type | How Rate Limiting Helps |
|-------------|------------------------|
| Brute force login | Limit attempts per IP/user |
| Credential stuffing | Block rapid auth attempts |
| API scraping | Limit data extraction rate |
| Resource exhaustion | Cap concurrent requests |

## Rate Limiting vs Throttling

These terms are often confused but have different meanings:

**Rate Limiting:** Hard rejection when limit exceeded
```
Client → Request #101 → Server
Server: "429 Too Many Requests" (immediate rejection)
```

**Throttling:** Slowing down requests when limit exceeded
```
Client → Request #101 → Server
Server: (queues request, processes after delay)
```

**Rate limiting** is about **protection**—stopping abuse.
**Throttling** is about **traffic shaping**—managing flow.

## Common Algorithms

### 1. Fixed Window

Simplest approach: count requests in fixed time windows.

```
Window: 1 minute
Limit: 100 requests

00:00 - 00:59: 100 requests ✅ (request 101 rejected)
01:00 - 01:59: Counter resets to 0
```

**Implementation:**
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  message: 'Too many requests, please try again later'
});

app.use('/api/', limiter);
```

**Problem: Window boundary attacks**
```
User makes 100 requests at 00:59
User makes 100 requests at 01:00
Result: 200 requests in 2 seconds! (at boundary)
```

### 2. Sliding Window Log

Track timestamp of each request and count within sliding window.

```
Current time: 10:00:30
Window: 1 minute
Count requests from: 09:59:30 to 10:00:30
```

**Implementation (Redis):**
```javascript
const redis = require('redis');
const client = redis.createClient();

async function isRateLimited(userId) {
  const now = Date.now();
  const windowStart = now - 60000; // 1 minute ago
  const key = `rate_limit:${userId}`;
  
  // Add current request timestamp
  await client.zAdd(key, { score: now, value: now.toString() });
  
  // Remove old entries outside window
  await client.zRemRangeByScore(key, 0, windowStart);
  
  // Count requests in window
  const count = await client.zCard(key);
  
  // Set expiry for cleanup
  await client.expire(key, 60);
  
  return count > 100;
}
```

**Pros:** Precise, no boundary issues
**Cons:** Memory usage scales with request count

### 3. Token Bucket

Maintain a "bucket" of tokens that refills over time.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Each request consumes 1 token
If bucket empty → request rejected
```

**Implementation:**
```javascript
class TokenBucket {
  constructor(capacity, refillRate) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillRate = refillRate; // tokens per second
    this.lastRefill = Date.now();
  }
  
  consume(tokens = 1) {
    this.refill();
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true; // Request allowed
    }
    return false; // Rate limited
  }
  
  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(
      this.capacity,
      this.tokens + elapsed * this.refillRate
    );
    this.lastRefill = now;
  }
}

// Usage
const bucket = new TokenBucket(100, 10); // 100 capacity, 10/sec refill
if (bucket.consume()) {
  // Process request
} else {
  // Return 429
}
```

**Pros:** Allows bursting, smooth traffic flow
**Cons:** More complex to implement

### 4. Leaky Bucket

Requests enter a queue and "leak" out at a constant rate.

```
Queue size: 100 requests
Processing rate: 10 requests/second

Requests queue up and are processed FIFO
If queue full → new requests rejected
```

**Implementation:**
```javascript
class LeakyBucket {
  constructor(capacity, leakRate) {
    this.capacity = capacity;
    this.queue = [];
    this.leakRate = leakRate; // requests per second
    this.leaking = false;
  }
  
  addRequest(callback) {
    if (this.queue.length >= this.capacity) {
      return false; // Bucket full
    }
    this.queue.push(callback);
    this.leak();
    return true;
  }
  
  leak() {
    if (this.leaking) return;
    this.leaking = true;
    
    const interval = setInterval(() => {
      if (this.queue.length === 0) {
        clearInterval(interval);
        this.leaking = false;
        return;
      }
      const callback = this.queue.shift();
      callback();
    }, 1000 / this.leakRate);
  }
}
```

**Pros:** Smooths traffic bursts, good for downstream protection
**Cons:** Adds latency, requests queue instead of immediate response

### Algorithm Comparison

| Algorithm | Memory | Precision | Burst Handling | Complexity |
|-----------|--------|-----------|----------------|------------|
| Fixed Window | Low | Low (boundary issues) | Poor | Simple |
| Sliding Log | High | High | Good | Medium |
| Token Bucket | Low | High | Excellent | Medium |
| Leaky Bucket | Medium | High | Good | Medium |

## Identifying Clients

Rate limiting is only effective if you can identify clients consistently.

### By IP Address

Simplest approach, works for anonymous users.

```javascript
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  keyGenerator: (req) => req.ip // Default behavior
});
```

**Issues:**
- Multiple users behind same NAT/proxy
- Users on mobile networks with rotating IPs
- Can be bypassed with IP rotation

### By User ID

Best for authenticated requests.

```javascript
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  keyGenerator: (req) => req.user?.id || req.ip,
  skip: (req) => !req.user // Skip if not authenticated
});
```

### By API Key

Common for public APIs.

```javascript
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  keyGenerator: (req) => req.headers['x-api-key'] || req.ip
});
```

### Hierarchical Limits

Combine multiple identifiers:

```javascript
// Rate limit by user (authenticated)
// Fall back to IP (unauthenticated)
function getKey(req) {
  if (req.user) {
    return `user:${req.user.id}`;
  }
  if (req.headers['x-api-key']) {
    return `api_key:${req.headers['x-api-key']}`;
  }
  return `ip:${req.ip}`;
}
```

## HTTP Headers and Responses

### Standard Headers

Always communicate rate limit status to clients:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704067200
```

**Header meanings:**
- `X-RateLimit-Limit`: Maximum requests allowed
- `X-RateLimit-Remaining`: Requests left in current window
- `X-RateLimit-Reset`: Unix timestamp when limit resets

### 429 Too Many Requests

When limit is exceeded, return proper status:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 60

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please retry after 60 seconds.",
  "retryAfter": 60
}
```

**Implementation:**
```javascript
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  headers: true, // Send X-RateLimit-* headers
  handler: (req, res) => {
    res.status(429).json({
      error: 'rate_limit_exceeded',
      message: 'Too many requests. Please slow down.',
      retryAfter: 60
    });
  }
});
```

### Retry-After Header

The `Retry-After` header tells clients when to retry:

```http
Retry-After: 120
```
Seconds to wait, or:

```http
Retry-After: Wed, 21 Oct 2025 07:28:00 GMT
```
Specific time to retry.

## Implementation Examples

### Express.js with express-rate-limit

```javascript
const express = require('express');
const rateLimit = require('express-rate-limit');

const app = express();

// Global limiter
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per 15 minutes
  message: 'Too many requests from this IP'
});

// Stricter limiter for auth endpoints
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5, // 5 failed attempts per hour
  message: 'Too many login attempts, please try again later',
  skipSuccessfulRequests: true // Only count failed attempts
});

// Apply globally
app.use('/api/', globalLimiter);

// Apply to auth routes
app.post('/api/login', authLimiter, loginHandler);
app.post('/api/register', authLimiter, registerHandler);
```

### Redis-Based Distributed Rate Limiting

```javascript
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

class RedisRateLimiter {
  constructor(key, limit, windowMs) {
    this.key = key;
    this.limit = limit;
    this.windowMs = windowMs;
  }
  
  async check(identifier) {
    const fullKey = `${this.key}:${identifier}`;
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    // Use Redis transaction for atomicity
    const result = await redis
      .multi()
      .zRemRangeByScore(fullKey, 0, windowStart)
      .zCard(fullKey)
      .zAdd(fullKey, { score: now, value: `${now}-${Math.random()}` })
      .pExpire(fullKey, this.windowMs)
      .exec();
    
    const count = result[1][1]; // zCard result
    const remaining = Math.max(0, this.limit - count);
    const resetTime = now + this.windowMs;
    
    return {
      allowed: count < this.limit,
      remaining,
      resetTime,
      retryAfter: count >= this.limit ? Math.ceil(this.windowMs / 1000) : 0
    };
  }
}

// Usage
const limiter = new RedisRateLimiter('api_limit', 100, 60000);

app.use(async (req, res, next) => {
  const identifier = req.user?.id || req.ip;
  const result = await limiter.check(identifier);
  
  res.set('X-RateLimit-Limit', limiter.limit);
  res.set('X-RateLimit-Remaining', result.remaining);
  res.set('X-RateLimit-Reset', result.resetTime);
  
  if (!result.allowed) {
    return res.status(429).json({
      error: 'rate_limit_exceeded',
      retryAfter: result.retryAfter
    });
  }
  
  next();
});
```

### Python Flask with Flask-Limiter

```python
from flask import Flask
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)
limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route("/api/data")
@limiter.limit("10 per minute")
def get_data():
    return {"data": "response"}

@app.route("/api/login", methods=["POST"])
@limiter.limit("5 per hour")
def login():
    # Login logic
    pass
```

## Distributed Rate Limiting

For applications with multiple instances, in-memory rate limiting doesn't work—each instance counts independently.

### Options for Distributed Rate Limiting

**1. Redis (Most Common)**

```javascript
// Shared Redis instance across all app servers
const redis = new Redis('redis://rate-limiter-cache:6379');
```

**2. Database-Based**

```sql
CREATE TABLE rate_limits (
  identifier VARCHAR(255),
  request_count INT,
  window_start TIMESTAMP,
  PRIMARY KEY (identifier)
);

-- Check and increment atomically
UPDATE rate_limits 
SET request_count = request_count + 1 
WHERE identifier = ? AND window_start > NOW() - INTERVAL '1 minute';
```

**3. API Gateway (Kong, Nginx, AWS API Gateway)**

Let the infrastructure handle it:

```nginx
# nginx.conf
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
  location /api/ {
    limit_req zone=api_limit burst=20 nodelay;
    proxy_pass http://backend;
  }
}
```

### Trade-offs

| Approach | Latency | Complexity | Cost |
|----------|---------|------------|------|
| In-Memory | Lowest | Simple | Free |
| Redis | Low (1-5ms) | Medium | Redis cost |
| Database | Higher (10-50ms) | Medium | DB load |
| API Gateway | Lowest | Highest | Infra cost |

## Best Practices

### 1. Set Appropriate Limits

Different endpoints need different limits:

```javascript
const limits = {
  // Public APIs - generous
  '/api/public': { max: 1000, window: 60000 },
  
  // Auth endpoints - strict
  '/api/login': { max: 5, window: 3600000 },
  '/api/register': { max: 3, window: 3600000 },
  
  // Data endpoints - moderate
  '/api/users': { max: 100, window: 60000 },
  
  // Expensive operations - very strict
  '/api/reports': { max: 10, window: 60000 }
};
```

### 2. Communicate Limits Clearly

Document your rate limits:

```markdown
## Rate Limits

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/api/public/*` | 1000 | 1 minute |
| `/api/auth/*` | 10 | 1 hour |
| `/api/users/*` | 100 | 1 minute |

Rate limit headers are included in every response.
```

### 3. Graceful Degradation

Handle rate limits client-side:

```javascript
async function fetchWithRetry(url, options = {}, retries = 3) {
  const response = await fetch(url, options);
  
  if (response.status === 429) {
    const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
    
    if (retries > 0) {
      await new Promise(r => setTimeout(r, retryAfter * 1000));
      return fetchWithRetry(url, options, retries - 1);
    }
    
    throw new Error('Rate limit exceeded, max retries reached');
  }
  
  return response;
}
```

### 4. Whitelist Internal Services

Don't rate limit your own services:

```javascript
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  skip: (req) => {
    // Skip internal services
    return req.ip === '10.0.0.1' || 
           req.headers['x-internal-service'] === 'true';
  }
});
```

### 5. Monitor and Alert

Track rate limit hits:

```javascript
app.use(rateLimit({
  // ...config
  handler: (req, res, next) => {
    metrics.increment('rate_limit.exceeded', {
      endpoint: req.path,
      identifier: req.user?.id || req.ip
    });
    
    res.status(429).json({ error: 'rate_limit_exceeded' });
  }
}));
```

Set alerts for unusual patterns:
- Sudden spike in rate limit hits → potential attack
- Steady increase over time → need higher limits or more capacity

### 6. Consider User Tiers

Different limits for different users:

```javascript
function getLimitForUser(user) {
  if (!user) return 100; // Anonymous
  if (user.tier === 'free') return 100;
  if (user.tier === 'pro') return 1000;
  if (user.tier === 'enterprise') return 10000;
  return 100;
}

const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: (req) => getLimitForUser(req.user)
});
```

## Conclusion

Rate limiting is essential infrastructure for any production API. It protects your resources, ensures fairness, and prevents abuse. By implementing appropriate rate limiting:

- **Choose the right algorithm** for your use case (Token Bucket for flexibility, Fixed Window for simplicity)
- **Identify clients properly** (user ID for authenticated, IP/API key for anonymous)
- **Communicate limits** via HTTP headers and 429 responses
- **Use Redis or similar** for distributed rate limiting
- **Set appropriate limits** per endpoint based on cost and importance
- **Monitor and adjust** based on real-world usage

Remember: rate limiting isn't just about blocking requests—it's about ensuring your API remains available and responsive for all your users.

Start simple with a fixed window implementation, then evolve to more sophisticated algorithms as your needs grow.