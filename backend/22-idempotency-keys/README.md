# Idempotency Keys: Preventing Duplicate Operations

Your payment API just got called twice for the same transaction. The customer was charged $100... twice. They're furious. Support is overwhelmed. And it wasn't even a bug in your code—it was a network timeout.

The client's request timed out. They retried. Your server processed both.

This is why idempotency keys exist.

## What's Idempotency?

An operation is **idempotent** if calling it multiple times produces the same result as calling it once.

```
GET /users/123  →  idempotent (reading doesn't change state)
POST /orders    →  NOT idempotent (each call creates a new order)
PUT /users/123  →  idempotent (setting name="John" 5 times = same result)
DELETE /users/123 → idempotent (delete once or 100 times, result is the same)
```

POST requests are the problem. By design, they create things. Retrying a POST shouldn't create duplicates—but without idempotency keys, it will.

## The Problem: Retries Are Everywhere

Networks fail. Timeouts happen. Clients retry. These aren't bugs—they're expected behavior:

- **Mobile networks** drop connections constantly
- **Load balancers** have timeout thresholds
- **Browsers** auto-retry on 5xx errors
- **Users** click "Submit" twice when impatient

You can't prevent retries. You can only make them safe.

### What Goes Wrong

```javascript
// ❌ No idempotency protection
app.post('/api/charges', async (req, res) => {
  const charge = await stripe.charges.create({
    amount: req.body.amount,
    source: req.body.source
  });
  res.json(charge);
});

// If this times out at 29 seconds, client retries...
// Customer gets charged twice. Oops.
```

## Idempotency Keys: The Solution

The client sends a unique key with each request. Your server stores the key and the response. If the same key comes again, return the stored response—don't reprocess.

```javascript
// ✅ With idempotency key
app.post('/api/charges', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  // Check if we've seen this key before
  const existing = await db.getIdempotencyResult(idempotencyKey);
  if (existing) {
    return res.json(existing.response); // Return cached result
  }
  
  // Process the request
  const charge = await stripe.charges.create({
    amount: req.body.amount,
    source: req.body.source
  });
  
  // Store the result for future retries
  await db.storeIdempotencyResult(idempotencyKey, charge, 24 * 60 * 60); // 24h TTL
  
  res.json(charge);
});
```

Now when the client retries, they get the same response without creating a duplicate charge.

## How Stripe Does It

Stripe's API is the gold standard for idempotency. Here's what they do:

**Headers:**
```
Idempotency-Key: <unique-key>
```

**Behavior:**
- Keys are scoped to a specific customer/merchant
- Results are cached for 24 hours
- The same key always returns the same response
- Keys are stored with the request parameters (to detect mismatches)

**Error handling:**
```javascript
// If you send the same key with DIFFERENT parameters:
POST /v1/charges (Idempotency-Key: abc123, amount: 1000)
POST /v1/charges (Idempotency-Key: abc123, amount: 2000)  // ❌ Error!

// Stripe returns 409 Conflict:
{
  "error": {
    "type": "idempotency_error",
    "message": "Keys for idempotent requests must only be used once"
  }
}
```

This prevents accidental key reuse across different operations.

## Implementation Patterns

### Pattern 1: Database Storage

Store keys with your main database:

```sql
CREATE TABLE idempotency_keys (
  key VARCHAR(255) PRIMARY KEY,
  request_hash VARCHAR(64),  -- hash of request body
  response JSONB,
  created_at TIMESTAMP,
  expires_at TIMESTAMP
);
```

```javascript
async function handleIdempotentRequest(key, requestParams, handler) {
  const existing = await db.query(
    'SELECT * FROM idempotency_keys WHERE key = $1',
    [key]
  );
  
  if (existing.row) {
    // Check if request params match
    if (existing.row.request_hash !== hash(requestParams)) {
      throw new ConflictError('Idempotency key already used for different request');
    }
    return existing.row.response;
  }
  
  // Process and store
  const result = await handler();
  
  await db.query(
    `INSERT INTO idempotency_keys (key, request_hash, response, expires_at)
     VALUES ($1, $2, $3, NOW() + INTERVAL '24 hours')`,
    [key, hash(requestParams), result]
  );
  
  return result;
}
```

**Pros:** Simple, transactional with your main data
**Cons:** Database becomes a bottleneck, cleanup needed

### Pattern 2: Redis Storage

Use Redis for fast key lookup with automatic expiration:

```javascript
async function handleIdempotentRequest(key, requestParams, handler) {
  const redisKey = `idempotency:${key}`;
  
  // Check existing
  const cached = await redis.get(redisKey);
  if (cached) {
    const parsed = JSON.parse(cached);
    if (parsed.requestHash !== hash(requestParams)) {
      throw new ConflictError('Idempotency key conflict');
    }
    return parsed.response;
  }
  
  // Use Redis SETNX for atomic lock
  const locked = await redis.set(
    redisKey,
    JSON.stringify({ processing: true }),
    'NX',
    'EX', 86400  // 24 hours
  );
  
  if (!locked) {
    // Another request is processing this key
    // Wait and retry, or return 409
    throw new ConflictError('Request already processing');
  }
  
  try {
    const result = await handler();
    
    // Update with actual response
    await redis.set(
      redisKey,
      JSON.stringify({ requestHash: hash(requestParams), response: result }),
      'EX', 86400
    );
    
    return result;
  } catch (error) {
    // On failure, remove the key so client can retry
    await redis.del(redisKey);
    throw error;
  }
}
```

**Pros:** Fast, automatic expiration, no DB pollution
**Cons:** Separate infrastructure, eventual consistency concerns

### Pattern 3: Idempotency in the Resource Itself

For some operations, you can bake idempotency into the resource design:

```javascript
// Instead of:
POST /api/charges
{ amount: 1000, source: "card_123" }

// Use client-generated ID:
POST /api/charges/charge_abc123
{ amount: 1000, source: "card_123" }

// Or use an idempotency key as part of the request:
POST /api/charges
{ 
  idempotency_key: "order-456-payment",
  amount: 1000,
  source: "card_123"
}
```

If the client generates a stable ID for their operation, retries naturally deduplicate:

```javascript
app.post('/api/charges/:id', async (req, res) => {
  const { id } = req.params;
  
  // Upsert: create if not exists, return existing if it does
  const charge = await db.query(
    `INSERT INTO charges (id, amount, source, status)
     VALUES ($1, $2, $3, 'pending')
     ON CONFLICT (id) DO NOTHING
     RETURNING *`,
    [id, req.body.amount, req.body.source]
  );
  
  if (!charge.row) {
    // Already existed, fetch it
    const existing = await db.query('SELECT * FROM charges WHERE id = $1', [id]);
    return res.json(existing.row);
  }
  
  // Process new charge...
});
```

**Pros:** No separate storage, natural deduplication
**Cons:** Requires client to generate IDs, changes API design

## Key Generation

Who generates the key? The client.

```javascript
// Client-side key generation
function generateIdempotencyKey() {
  // Option 1: UUID
  return crypto.randomUUID();
  
  // Option 2: Deterministic (same operation = same key)
  return hash(`${orderId}-${operation}-${timestamp}`);
}

// Usage
fetch('/api/charges', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Idempotency-Key': generateIdempotencyKey()
  },
  body: JSON.stringify({ amount: 1000, source: 'card_123' })
});
```

**Deterministic keys** (Option 2) are useful when the same logical operation should always produce the same result:
- Paying for order #123 → key: `order-123-payment`
- This way, even across different sessions, the payment can't be duplicated

**Random keys** (Option 1) are safer when you want each form submission to be independent.

## Edge Cases and Gotchas

### Partial Failures

What if the request fails after processing but before storing the idempotency result?

```javascript
// ❌ Problem: crash between process and store
const result = await processPayment();  // Success
// CRASH! Key not stored
await storeIdempotencyResult(key, result);  // Never executed

// Client retries... processes payment again!
```

**Solution:** Use a transaction or atomic operation:

```javascript
// ✅ Atomic: store key first, then process
await db.query('BEGIN');
await db.query('INSERT INTO idempotency_keys (key, status) VALUES ($1, $processing)', [key]);
const result = await processPayment();
await db.query('UPDATE idempotency_keys SET response = $1 WHERE key = $2', [result, key]);
await db.query('COMMIT');
```

### TTL and Expiration

How long should keys last?

| Duration | Use Case |
|----------|----------|
| 24 hours | Payment operations (Stripe's default) |
| 1 hour | High-volume APIs, rate-sensitive operations |
| 7 days | Long-running processes, workflows |
| Forever | Idempotent resources (upsert by ID) |

Too short: legitimate retries might re-process.
Too long: storage bloat, potential privacy issues.

### Concurrent Requests with Same Key

What if two requests with the same key arrive simultaneously?

```
Request A: POST /charges (Idempotency-Key: abc123) → starts processing
Request B: POST /charges (Idempotency-Key: abc123) → arrives 100ms later
```

**Options:**

1. **Lock and wait:** B waits for A to complete, then returns A's result
2. **Fail fast:** B immediately returns 409 Conflict
3. **Return in-progress status:** B returns a status indicating A is processing

```javascript
// Redis with lock pattern
const lockKey = `idempotency-lock:${key}`;
const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 30);

if (!acquired) {
  // Another request is processing
  // Option: poll and wait, or return 409
  return res.status(409).json({ error: 'Request already processing' });
}

try {
  const result = await handler();
  await storeResult(key, result);
  return res.json(result);
} finally {
  await redis.del(lockKey);
}
```

## When You Need Idempotency (And When You Don't)

### ✅ Always Use Idempotency Keys

- **Payment processing** — duplicate charges are unacceptable
- **Order creation** — customers hate duplicate orders
- **Email sending** — don't spam users
- **Resource creation with side effects** — webhooks, notifications, external API calls

### ⚠️ Consider Using Idempotency

- **Database writes** — depends on if duplicates are acceptable
- **File uploads** — depends on storage costs
- **State changes** — "mark as read" is naturally idempotent

### ❌ Not Needed

- **GET requests** — already idempotent
- **PUT requests** — naturally idempotent (same input = same result)
- **DELETE requests** — naturally idempotent
- **Read-only operations** — no state change

## Real-World Checklist

Before shipping idempotency to production:

- [ ] Keys are validated (non-empty, reasonable length)
- [ ] Keys are stored with request hash (detect mismatched retries)
- [ ] Keys have TTL (don't keep forever)
- [ ] Handles concurrent requests with same key
- [ ] Errors don't leave orphaned keys
- [ ] Key storage failure doesn't block the operation
- [ ] Client SDK generates keys automatically
- [ ] Documentation explains key format and expectations

---

**The bottom line:** Networks will fail. Clients will retry. Your API should handle it gracefully. Idempotency keys are a few extra lines of code that prevent real customer pain.

Start with payments. Then expand to any operation where duplicates hurt.