# Caching Strategies: From In-Memory to Distributed

Your API response time just jumped from 50ms to 2 seconds. The database is pegged at 100% CPU. Users are complaining. You check the queries and realize—the same data is being fetched hundreds of times per second, and it barely changes.

That's when caching stops being a "nice to have" and becomes critical infrastructure.

In this guide, we'll cover when to cache, what to cache, and the trade-offs between in-memory, distributed, and CDN-level caching.

---

## Why Caching Matters

Every database query has a cost. Sometimes it's milliseconds, sometimes it's seconds. When you're serving the same data repeatedly, caching gives you:

- **Lower latency** — fetching from memory is orders of magnitude faster than disk or network
- **Reduced load** — fewer database queries means more headroom for traffic spikes
- **Cost savings** — less database CPU and I/O translates to smaller infrastructure bills

But caching isn't free. It adds complexity, introduces consistency challenges, and can become a source of bugs if you're not careful.

---

## What Should You Cache?

Not everything benefits from caching. Here's a quick decision framework:

| Data Type | Cache It? | Why |
|-----------|-----------|-----|
| User profile data | ✅ Yes | Read frequently, changes rarely |
| Product catalog | ✅ Yes | High read volume, updates are controlled |
| Personalized feeds | ⚠️ Maybe | Depends on staleness tolerance |
| Real-time prices | ❌ No | Needs to be accurate to the second |
| User session tokens | ✅ Yes | Checked on every request |
| One-time calculations | ❌ No | Won't be reused |

### The Cache-Worthiness Test

Ask yourself three questions:

1. **Is it read often?** — If data is accessed once per hour, caching won't help much.
2. **Does it change rarely?** — If it updates every second, you'll have cache invalidation headaches.
3. **Is it expensive to compute?** — Complex queries or API calls benefit most from caching.

If you answered yes to at least two, consider caching it.

---

## In-Memory Caching: The Simple Approach

For single-server applications, in-memory caches like Node.js's `node-cache` or Python's `functools.lru_cache` are dead simple to implement.

### Example: Simple In-Memory Cache

```javascript
// Using node-cache for simple caching
const NodeCache = require('node-cache');

// Cache with 10-minute TTL
const cache = new NodeCache({ stdTTL: 600, checkperiod: 120 });

async function getUser(userId) {
  // Check cache first
  const cached = cache.get(`user:${userId}`);
  if (cached) {
    return cached;
  }

  // Cache miss — fetch from database
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  
  // Store in cache for future requests
  cache.set(`user:${userId}`, user);
  return user;
}
```

### When In-Memory Works

- Single server or low traffic
- Development and testing environments
- Data that can tolerate occasional loss on restart
- Session storage with sticky sessions

### The Problem with In-Memory

Every server has its own cache. If you have 3 instances behind a load balancer:

```
Request 1 → Server A (cache miss, fetches data, caches it)
Request 2 → Server B (cache miss again, fetches data, caches it)
Request 3 → Server A (cache hit! 🎉)
Request 4 → Server B (cache hit! 🎉)
Request 5 → Server C (cache miss... again 😞)
```

You've just tripled your database load and wasted memory storing the same data three times.

---

## Distributed Caching: Redis and Friends

When you have multiple servers, you need a shared cache. Redis is the industry standard—fast, feature-rich, and battle-tested.

### Basic Redis Caching

```javascript
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

async function getUser(userId) {
  // Check Redis first
  const cached = await redis.get(`user:${userId}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  
  // Cache for 10 minutes
  await redis.setex(`user:${userId}`, 600, JSON.stringify(user));
  return user;
}
```

### Redis Data Structures

Redis isn't just a key-value store. It offers data structures that solve real problems:

| Type | Use Case |
|------|----------|
| String | Simple cached values, counters |
| Hash | User profiles, product details |
| List | Activity feeds, queues |
| Set | Unique visitors, tags |
| Sorted Set | Leaderboards, rate limiting |
| Bitfield | Feature flags, permissions |

### Example: Caching a Leaderboard

```javascript
// ❌ Bad: Fetch from database every time
async function getLeaderboard() {
  return await db.query(`
    SELECT user_id, score 
    FROM game_scores 
    ORDER BY score DESC 
    LIMIT 100
  `);
}

// ✅ Better: Cache with Redis sorted set
async function updateLeaderboard(userId, score) {
  // Add to sorted set (score is the sort key)
  await redis.zadd('leaderboard', score, userId);
}

async function getLeaderboard() {
  // Get top 100 (cached, no database hit)
  const results = await redis.zrevrange('leaderboard', 0, 99, 'WITHSCORES');
  
  // Convert to friendly format
  const leaderboard = [];
  for (let i = 0; i < results.length; i += 2) {
    leaderboard.push({
      userId: results[i],
      score: parseInt(results[i + 1])
    });
  }
  return leaderboard;
}
```

---

## Cache Invalidation: The Hard Part

Phil Karlton famously said: "There are only two hard things in Computer Science: cache invalidation and naming things."

Stale cache is worse than no cache. Users see outdated data and make bad decisions.

### Strategies for Invalidation

**1. Time-Based Expiration (TTL)**

Simplest approach—every cache entry has an expiration time.

```javascript
// Cache for 5 minutes, then automatically expire
await redis.setex('product:123', 300, JSON.stringify(product));
```

✅ Pros: Simple, works for most cases  
❌ Cons: Data can be stale for up to TTL duration

**2. Write-Through Cache**

Update the cache whenever you update the database.

```javascript
async function updateProduct(productId, data) {
  // Update database
  await db.query('UPDATE products SET name = $1 WHERE id = $2', [data.name, productId]);
  
  // Update cache immediately
  await redis.set(`product:${productId}`, JSON.stringify(data));
}
```

✅ Pros: Cache always fresh  
❌ Cons: More writes, slower updates, need to handle failures

**3. Write-Behind (Lazy Invalidation)**

Mark cache as invalid on write, let the next read fetch fresh data.

```javascript
async function updateProduct(productId, data) {
  // Update database
  await db.query('UPDATE products SET name = $1 WHERE id = $2', [data.name, productId]);
  
  // Delete cache entry — next read will fetch fresh data
  await redis.del(`product:${productId}`);
}
```

✅ Pros: Fast writes, fresh data on next read  
❌ Cons: First read after update is slower

**4. Cache Versioning**

Add a version number to cache keys. When data changes, increment the version.

```javascript
// Cache key includes version: product:123:v5
async function getProduct(productId) {
  const version = await redis.get(`product:${productId}:version`) || 'v1';
  const cached = await redis.get(`product:${productId}:${version}`);
  
  if (cached) return JSON.parse(cached);
  
  // Fetch and cache
  const product = await db.query('SELECT * FROM products WHERE id = $1', [productId]);
  await redis.set(`product:${productId}:${version}`, JSON.stringify(product), 'EX', 3600);
  return product;
}

async function updateProduct(productId, data) {
  await db.query('UPDATE products SET name = $1 WHERE id = $2', [data.name, productId]);
  
  // Increment version — old cache keys are now orphaned
  await redis.incr(`product:${productId}:version`);
}
```

---

## Cache Stampede Protection

Here's a classic scenario: your cache expires, and suddenly 100 requests hit your database simultaneously.

```
Cache expires → Request 1, 2, 3... 100 all see cache miss → 100 DB queries → 💥
```

### Solution: Locking / Single-Flight

Only let one request fetch the data; others wait.

```javascript
const locks = new Map();

async function getUserWithLock(userId) {
  const cacheKey = `user:${userId}`;
  
  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);
  
  // Check if another request is already fetching
  if (locks.has(cacheKey)) {
    // Wait for the other request to finish
    return await locks.get(cacheKey);
  }
  
  // Acquire lock
  const fetchPromise = (async () => {
    const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    await redis.setex(cacheKey, 600, JSON.stringify(user));
    locks.delete(cacheKey);
    return user;
  })();
  
  locks.set(cacheKey, fetchPromise);
  return fetchPromise;
}
```

Or use Redis's `SETNX` for distributed locking:

```javascript
async function getUserWithDistributedLock(userId) {
  const cacheKey = `user:${userId}`;
  const lockKey = `lock:${cacheKey}`;
  
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);
  
  // Try to acquire lock (expires in 5 seconds to prevent deadlocks)
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 5);
  
  if (acquired) {
    // We got the lock — fetch and cache
    const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    await redis.setex(cacheKey, 600, JSON.stringify(user));
    await redis.del(lockKey);
    return user;
  } else {
    // Another request is fetching — wait and retry
    await new Promise(r => setTimeout(r, 100));
    return getUserWithDistributedLock(userId);
  }
}
```

---

## CDN-Level Caching

For static assets and some API responses, caching at the CDN level is the most efficient.

### HTTP Caching Headers

```javascript
// Express.js example
app.get('/api/products/:id', async (req, res) => {
  const product = await getProduct(req.params.id);
  
  // Tell CDN and browser to cache for 5 minutes
  res.set('Cache-Control', 'public, max-age=300');
  
  // Enable conditional requests
  res.set('ETag', generateETag(product));
  
  res.json(product);
});
```

### Cache-Control Options

| Header | Meaning |
|--------|---------|
| `public` | CDN/proxy can cache this |
| `private` | Only browser can cache (not CDN) |
| `max-age=300` | Cache for 300 seconds |
| `no-cache` | Must revalidate before use |
| `no-store` | Don't cache at all |
| `stale-while-revalidate=60` | Serve stale for 60s while revalidating |

### Stale-While-Revalidate

This header is magical for APIs. It lets you serve slightly stale data while fetching fresh data in the background.

```
Cache-Control: public, max-age=300, stale-while-revalidate=60
```

- First 5 minutes: serve from cache, fresh
- Minutes 5-6: serve from cache (stale), fetch fresh data in background
- After 6 minutes: wait for fresh data

---

## Caching Anti-Patterns

### ❌ Caching User-Specific Data Without Namespacing

```javascript
// BAD: Different users' data will collide
await redis.set('user-preferences', JSON.stringify(prefs));
```

```javascript
// GOOD: Include user ID in key
await redis.set(`user:${userId}:preferences`, JSON.stringify(prefs));
```

### ❌ Caching Sensitive Data Without Expiration

```javascript
// BAD: Password hash lives forever in cache
await redis.set('user:' + userId, JSON.stringify(user));
```

```javascript
// GOOD: Never cache sensitive fields, always set TTL
const { passwordHash, ...safeUser } = user;
await redis.setex(`user:${userId}`, 3600, JSON.stringify(safeUser));
```

### ❌ Assuming Cache Will Always Be There

```javascript
// BAD: Will crash if Redis is down
const user = JSON.parse(await redis.get(`user:${userId}`));
```

```javascript
// GOOD: Graceful fallback
async function getUser(userId) {
  try {
    const cached = await redis.get(`user:${userId}`);
    if (cached) return JSON.parse(cached);
  } catch (err) {
    console.error('Cache error, falling back to database:', err.message);
  }
  
  return await db.query('SELECT * FROM users WHERE id = $1', [userId]);
}
```

---

## Choosing the Right Strategy

| Scenario | Recommended Approach |
|----------|----------------------|
| Single server, simple app | In-memory cache (node-cache, LRU cache) |
| Multiple servers | Redis distributed cache |
| Static assets, public APIs | CDN caching with Cache-Control headers |
| Frequently updated data | Short TTL + write-behind invalidation |
| Rarely updated reference data | Long TTL + versioned keys |
| High-traffic, cost-sensitive | Multi-tier: CDN → Redis → Database |

---

## Key Takeaways

- **Start simple** — In-memory cache for single servers, Redis for distributed systems
- **Set TTLs** — Always have expiration; stale data is better than wrong data, but both are bad
- **Handle cache misses gracefully** — The cache will fail; design for it
- **Protect against stampedes** — Use locking or single-flight when cache expires under load
- **Namespace your keys** — Avoid collisions between different data types
- **Never cache sensitive data** — Or at minimum, exclude it before caching
- **Measure before optimizing** — Add caching where it matters, not everywhere

Caching is a tool, not a silver bullet. Use it to solve real performance problems, not to paper over inefficient queries or missing indexes.