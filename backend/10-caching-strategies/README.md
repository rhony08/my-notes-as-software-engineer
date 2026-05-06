# Caching Strategies: From In-Memory to Distributed

Your database query takes 500ms. You run it again. Another 500ms. Same query, same result. That's wasted time on every request—and your users feel it.

Caching solves this by storing the result somewhere faster to access. But where? And for how long? And what happens when the data changes? These decisions separate a quick fix from a production-ready solution.

## Why Caching Matters

Every millisecond counts when you're handling thousands of requests. Without caching:

- Database becomes a bottleneck under load
- Slow API responses frustrate users
- You waste compute on repeated work
- Third-party API quotas get exhausted

The right caching strategy can reduce response times by 90%+ and dramatically lower your infrastructure costs.

## The Caching Spectrum

### 1. In-Memory Cache (Local)

The simplest approach: store data in your application's memory.

```javascript
// Simple in-memory cache
const cache = new Map();

async function getUser(id) {
  if (cache.has(id)) {
    return cache.get(id); // ~0.1ms
  }
  
  const user = await db.query('SELECT * FROM users WHERE id = ?', [id]); // ~50ms
  cache.set(id, user);
  return user;
}
```

**Pros:**
- Zero latency (same process)
- No external dependencies
- Simple to implement

**Cons:**
- Lost on restart or deploy
- Each server has its own cache (no sharing)
- Memory limited to one instance

**When to use:** Small datasets, single-instance apps, development environments.

### 2. Distributed Cache (Redis, Memcached)

A separate cache server that all your app instances share.

```javascript
// ❌ Cache miss, but you forgot to set TTL
await redis.set('user:123', JSON.stringify(user));

// ✅ Always set expiration
await redis.set('user:123', JSON.stringify(user), 'EX', 3600); // 1 hour
```

**Pros:**
- Shared across all instances
- Persists across deploys
- Can scale independently
- Rich data structures (Redis)

**Cons:**
- Network latency (~1-3ms)
- Additional infrastructure
- Cache invalidation complexity
- Need to handle failures gracefully

**When to use:** Multi-instance deployments, session storage, frequently accessed data.

### 3. CDN / Edge Caching

Cache at the edge, closest to users.

```
User → CDN Edge (cached?) → Load Balancer → App → Database
         ↑
    Response cached here for next user
```

**Pros:**
- Lowest latency for users globally
- Reduces load on your servers
- Handles massive traffic spikes

**Cons:**
- Only works for cacheable responses (GET, public data)
- Cache invalidation takes time to propagate
- Less control over cache behavior

**When to use:** Static assets, public API responses, geographically distributed users.

## Cache Invalidation: The Hard Part

There are only two hard things in Computer Science: cache invalidation and naming things.

### Strategies

**1. Time-based (TTL)**

Simplest approach. Set an expiration time.

```javascript
// Cache for 5 minutes
await redis.set('products:featured', products, 'EX', 300);
```

**Trade-off:** Users might see stale data for up to 5 minutes. Choose TTL based on how quickly your data changes and how critical freshness is.

**2. Write-through**

Update cache immediately when data changes.

```javascript
async function updateUser(id, data) {
  await db.query('UPDATE users SET ? WHERE id = ?', [data, id]);
  await redis.del(`user:${id}`); // Invalidate cache
}
```

**Trade-off:** Extra latency on writes. Every update now involves cache operations.

**3. Write-behind**

Update cache first, persist to database later.

```javascript
// Fast response
await redis.set(`user:${id}`, updatedUser);
queue.push({ type: 'update', id, data: updatedUser }); // Async DB write

return updatedUser;
```

**Trade-off:** Risk of data loss if cache fails before DB write completes. Use for non-critical data.

**4. Cache-aside (Lazy Loading)**

Only load into cache when requested.

```javascript
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);
  
  const user = await db.query('SELECT * FROM users WHERE id = ?', [id]);
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 3600);
  
  return user;
}
```

**Trade-off:** First request after cache miss is slow. Good for read-heavy workloads.

## Common Patterns

### Pattern: Cache Warming

Pre-populate cache before users need it.

```javascript
// On startup, load frequently accessed data
async function warmCache() {
  const featured = await db.query('SELECT * FROM products WHERE featured = true');
  await redis.set('products:featured', JSON.stringify(featured), 'EX', 3600);
}
```

### Pattern: Stampede Protection

When cache expires, multiple requests might hit the database simultaneously.

```javascript
// ❌ Multiple requests can trigger expensive query
async function getExpensiveData() {
  const cached = await redis.get('expensive:data');
  if (cached) return cached;
  
  // Race condition: multiple requests could reach here
  const data = await expensiveComputation();
  await redis.set('expensive:data', data, 'EX', 300);
  return data;
}

// ✅ Use lock to prevent stampede
async function getExpensiveDataSafe() {
  const cached = await redis.get('expensive:data');
  if (cached) return cached;
  
  const lockKey = 'lock:expensive:data';
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 10);
  
  if (acquired) {
    const data = await expensiveComputation();
    await redis.set('expensive:data', data, 'EX', 300);
    await redis.del(lockKey);
    return data;
  }
  
  // Wait and retry
  await sleep(100);
  return getExpensiveDataSafe();
}
```

### Pattern: Graceful Degradation

If cache fails, your app should still work.

```javascript
async function getUser(id) {
  try {
    const cached = await redis.get(`user:${id}`);
    if (cached) return JSON.parse(cached);
  } catch (err) {
    console.error('Cache error:', err);
    // Continue to database
  }
  
  return db.query('SELECT * FROM users WHERE id = ?', [id]);
}
```

## What to Cache

| Data Type | Cache? | TTL | Notes |
|-----------|--------|-----|-------|
| User profiles | ✅ | 5-15 min | Invalidate on update |
| Product catalog | ✅ | 1-24 hours | Warm on startup |
| Search results | ✅ | 1-5 min | Key includes query params |
| User-specific recommendations | ✅ | 15-60 min | Personalized, can be stale |
| User sessions | ✅ | 7-30 days | Use Redis with refresh |
| Real-time inventory | ⚠️ | 10-30 sec | Risk of overselling |
| Passwords, auth tokens | ❌ | Never | Security risk |
| One-time data (payment status) | ❌ | Never | Must be fresh |

## Monitoring Your Cache

Track these metrics:

```
Cache Hit Ratio = Hits / (Hits + Misses)

Target: > 80% for most use cases
```

If hit ratio is low:
- TTL might be too short
- Keys might not be reused
- You might be caching the wrong data

```javascript
// Simple hit ratio tracking
let hits = 0, misses = 0;

async function getWithMetrics(key) {
  const cached = await redis.get(key);
  if (cached) {
    hits++;
    return JSON.parse(cached);
  }
  misses++;
  // ... fetch from DB
}
```

## Choosing Your Strategy

| Scenario | Strategy | Why |
|----------|----------|-----|
| Single server, simple app | In-memory Map | No complexity needed |
| Multiple instances | Redis/Memcached | Shared cache state |
| Global API with static responses | CDN + Redis | Multi-layer protection |
| User sessions | Redis with TTL | Fast, shared across instances |
| Frequently changing data | Short TTL + write-through | Balance freshness vs performance |
| Expensive computations | Cache-aside with stampede protection | Prevent thundering herd |

## Key Takeaways

1. **Start simple.** In-memory cache works for small apps. Add Redis when you need shared state across instances.

2. **Always set TTL.** Without expiration, stale data lives forever and memory grows unbounded.

3. **Handle cache failures.** Your app should degrade gracefully when Redis is down.

4. **Measure hit ratio.** If it's below 80%, you're either caching the wrong data or expiring too quickly.

5. **Invalidate on writes.** Time-based expiration isn't enough for critical data—actively invalidate when data changes.

6. **Protect against stampedes.** Use locks or probabilistic early expiration when cache misses trigger expensive operations.

7. **Layer your caches.** CDN → Redis → Database. Each layer handles different access patterns.

Caching isn't free—it adds complexity, infrastructure, and new failure modes. But when your database is the bottleneck, it's often the difference between a sluggish system and a responsive one.