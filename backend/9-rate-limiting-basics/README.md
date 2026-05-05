# Rate Limiting: Protecting Your APIs

Every API has limits—server resources, database connections, third-party quotas. Without rate limiting, a single misbehaving client can overwhelm your entire system.

I learned this the hard way. A buggy mobile app was retrying failed requests in an infinite loop, and within minutes, our database connection pool was exhausted. Every other user got 503 errors. Rate limiting would've prevented this entirely.

## Why Rate Limiting Matters

Your servers aren't infinite. Neither are your budgets.

**What breaks without limits:**

- A buggy client loop DDOSes your API
- A malicious actor scrapes all your data
- One heavy user degrades experience for everyone
- You blow through third-party API quotas (expensive!)
- Your cloud bill spikes unexpectedly

Rate limiting is your first line of defense—like a bouncer at a club, letting people in at a sustainable pace instead of letting everyone rush in at once.

## Common Algorithms

There's no "best" algorithm. Each has trade-offs.

### Fixed Window

Simplest approach: count requests within a time window (e.g., 1 minute).

```
User A: 10 requests in 00:00-00:59 → allowed (limit: 10/min)
User A: 1 request in 01:00-01:59 → allowed (limit: 10/min)
```

**The problem:** Edge-case abuse. A user can make 10 requests at 00:59 and 10 more at 01:00—20 requests in 2 seconds.

```
Timeline: |-- Window 1 --|-- Window 2 --|
Requests:         10    10
                  ↑     ↑
               00:59  01:00  ← 20 requests in 2 seconds!
```

**When to use:** Low-risk APIs, internal tools, where simplicity matters more than precision.

### Token Bucket

My favorite for most use cases. You get a bucket of tokens that refills over time.

**How it works:**

```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

- Each request costs 1 token
- Bucket starts full (10 tokens)
- Burst of 10 requests? Allowed (bucket empties)
- After burst: limited to 1 request/second until refilled
- Wait 5 seconds? Bucket has 5 tokens again
```

**Why it's great:**

- Handles bursts naturally (unlike fixed window)
- Memory efficient—just track token count + last refill time
- Predictable average rate with burst tolerance

**Trade-off:** If you need strict "no more than X per minute" guarantees, this might not fit—bursts are allowed.

### Sliding Window Log

Most precise, but expensive.

Store every request timestamp in a sorted set:

```
Requests in last 60 seconds:
[user:123:requests] = [1712345671, 1712345690, 1712345720, ...]

Count: 3 requests in window → allow
Count: 100 requests in window → deny
```

**Pros:**

- Exact enforcement (no edge cases)
- Natural "request count in last N seconds" queries

**Cons:**

- Memory grows with request count
- Each request = a write operation
- Expensive at high traffic

**When to use:** Financial APIs, strict compliance requirements, where precision matters more than cost.

### Sliding Window Counter (Hybrid)

Fixed window's simplicity + sliding window's accuracy.

Track counts for current and previous windows:

```
Current window (30-60s): 5 requests
Previous window (0-30s): 8 requests

Weighted count = 5 + (8 * 0.5) = 9 requests
                ↑     ↑
           current  weighted previous
```

Good balance of accuracy and performance.

## Implementation with Redis

Token bucket is straightforward to implement:

```javascript
// ❌ NEVER DO THIS - in-memory rate limiting
// Resets on every deployment, doesn't work across servers
const requestCounts = {};

function checkRateLimit(userId) {
  const count = requestCounts[userId] || 0;
  if (count >= 10) return false;
  requestCounts[userId] = count + 1;
  return true;
}

// ✅ Redis-based rate limiting
async function checkRateLimit(userId, limit = 10, windowSec = 60) {
  const key = `ratelimit:${userId}`;
  
  const current = await redis.get(key);
  
  if (current && parseInt(current) >= limit) {
    return { allowed: false, remaining: 0 };
  }
  
  const multi = redis.multi();
  multi.incr(key);
  multi.expire(key, windowSec);
  await multi.exec();
  
  return { 
    allowed: true, 
    remaining: limit - parseInt(current || '0') - 1 
  };
}
```

**For token bucket:**

```javascript
async function tokenBucket(userId, capacity = 10, refillRate = 1) {
  const key = `bucket:${userId}`;
  const now = Date.now();
  
  // Get current bucket state
  const [tokens, lastRefill] = await redis.hmget(key, 'tokens', 'last_refill');
  
  let currentTokens = parseInt(tokens || capacity);
  const lastRefillTime = parseInt(lastRefill || now);
  
  // Calculate tokens to add based on elapsed time
  const elapsed = Math.floor((now - lastRefillTime) / 1000);
  currentTokens = Math.min(capacity, currentTokens + elapsed * refillRate);
  
  if (currentTokens < 1) {
    return { allowed: false, retryAfter: Math.ceil((1 - currentTokens) / refillRate) };
  }
  
  // Consume a token
  await redis.hmset(key, {
    tokens: currentTokens - 1,
    last_refill: now
  });
  redis.expire(key, 3600); // Clean up inactive buckets
  
  return { allowed: true, remaining: currentTokens - 1 };
}
```

## Communicating Limits to Clients

Don't leave clients guessing. Use standard headers:

```javascript
app.use((req, res, next) => {
  const remaining = getRemainingRequests(req.user.id);
  const resetTime = getResetTime();
  
  res.set({
    'X-RateLimit-Limit': 100,
    'X-RateLimit-Remaining': remaining,
    'X-RateLimit-Reset': resetTime,
    'Retry-After': remaining === 0 ? Math.ceil((resetTime - Date.now()) / 1000) : 0
  });
  
  next();
});
```

**When rate limited, return 429:**

```javascript
if (!allowed) {
  return res.status(429).json({
    error: 'Too Many Requests',
    retryAfter: retryAfterSeconds,
    limit: 100
  });
}
```

## What to Rate Limit

You can't rate limit everything—pick your battles.

| What to Limit | Why |
|---------------|-----|
| Authentication endpoints | Brute force protection |
| Password reset | Prevent abuse |
| Search queries | Database-heavy |
| File uploads | Bandwidth and storage |
| Third-party API proxies | Quota protection |
| Write operations | Data integrity |

| What NOT to Limit | Why |
|-------------------|-----|
| Static assets | CDN handles this |
| Health checks | Breaks orchestration |
| Webhook callbacks | You initiated these |
| Internal service calls | Use different auth |

## Handling Rate Limit Exceeded

Clients will hit limits. Help them recover gracefully:

```javascript
// Client-side retry with exponential backoff
async function fetchWithRetry(url, options = {}, retries = 3) {
  const response = await fetch(url, options);
  
  if (response.status === 429) {
    const retryAfter = parseInt(response.headers.get('Retry-After') || '1');
    const backoff = Math.min(retryAfter * 1000, 30000); // Cap at 30s
    
    if (retries > 0) {
      await sleep(backoff);
      return fetchWithRetry(url, options, retries - 1);
    }
  }
  
  return response;
}
```

## Choosing the Right Limits

No magic numbers—tune based on your system:

**For read-heavy APIs:**

- Higher limits (100-1000 req/min per user)
- Consider IP-based limits as backup

**For write-heavy APIs:**

- Lower limits (10-50 req/min per user)
- Per-resource limits (max 5 posts/hour)

**For authentication:**

- Very low limits (5-10 attempts/min)
- Progressive lockout (failed attempts increase wait time)

## Trade-offs Summary

| Algorithm | Memory | Precision | Burst Handling | Use Case |
|-----------|--------|-----------|----------------|----------|
| Fixed Window | Low | Low | Poor | Internal APIs |
| Token Bucket | Low | Medium | Good | Most APIs |
| Sliding Log | High | High | Exact | Financial/Compliance |
| Sliding Counter | Low | High | Good | Balance of accuracy/cost |

## Key Takeaways

1. **Rate limiting isn't optional**—it's essential for production APIs
2. **Token bucket** is usually the right choice (simple + burst handling)
3. **Use Redis** (or similar) for distributed rate limiting across servers
4. **Communicate limits** via headers (`X-RateLimit-*`, `Retry-After`)
5. **Return 429** with helpful retry information
6. **Client-side retry logic** should use exponential backoff
7. **Tune limits** based on your actual traffic patterns and resource constraints
8. **Start strict, loosen later**—easier to relax limits than deal with abuse

---

Rate limiting is one of those things you don't think about until it's too late. Add it early. Your future self will thank you at 3am when something goes wrong and your API stays up.