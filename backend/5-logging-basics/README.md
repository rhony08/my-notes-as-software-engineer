# Effective Logging for Backend Applications

Your API is throwing 500 errors. Users are complaining. You check the logs and see:

```
[2024-01-15 14:23:01] ERROR: Something went wrong
[2024-01-15 14:23:01] ERROR: Something went wrong
[2024-01-15 14:23:01] ERROR: Something went wrong
```

Good luck debugging that.

Logging is often treated as an afterthought—something we add when debugging. But good logs are like having a black box recorder for your app. When something breaks at 3am, you'll wish you had structured logs instead of grep-ing through `User John did something at some time`.

## Why Logging Matters

Bad logs don't just waste time—they hide problems:

- **Production incidents drag on** because you can't reproduce the issue
- **Performance problems go unnoticed** until users complain
- **Security breaches slip through** because suspicious patterns weren't logged
- **On-call becomes a nightmare** of guessing what happened

## Structured vs Unstructured Logging

The difference isn't aesthetics. It's whether your logs are actually useful.

### ❌ Unstructured Logs

```
[2024-01-15 14:23:01] INFO: User john@example.com logged in from 192.168.1.1
[2024-01-15 14:23:05] INFO: User john@example.com created order #12345
[2024-01-15 14:23:10] ERROR: Payment failed for order #12345
```

To find all payment failures, you'd need regex. To correlate the user with the failed payment, you'd need complex parsing. This doesn't scale.

### ✅ Structured Logs (JSON)

```json
{"timestamp": "2024-01-15T14:23:01Z", "level": "INFO", "event": "user.login", "user_id": "u_123", "email": "john@example.com", "ip": "192.168.1.1"}
{"timestamp": "2024-01-15T14:23:05Z", "level": "INFO", "event": "order.created", "user_id": "u_123", "order_id": "12345", "amount": 99.99}
{"timestamp": "2024-01-15T14:23:10Z", "level": "ERROR", "event": "payment.failed", "user_id": "u_123", "order_id": "12345", "error_code": "card_declined", "reason": "insufficient_funds"}
```

Now you can filter by `event=payment.failed`, group by `user_id`, and see the full picture. Most log aggregators (Datadog, Logtail, Papertrail) can query JSON fields directly.

## What to Log (and What Not To)

### ✅ Always Log

- **Request IDs** — correlate logs across services for a single request
- **User IDs** — trace what a specific user did (audit trails, debugging)
- **Timestamps** — ISO 8601 with timezone, always
- **Event names** — `user.login`, `order.created`, `payment.failed`
- **Duration** — how long did database queries, external API calls take?
- **Error details** — stack traces, error codes, not just "error occurred"

### ❌ Never Log

- **Passwords** — even hashed, just don't
- **API keys / tokens** — if you must log something, log `token_id` or first/last 4 chars
- **Credit card numbers** — PCI compliance 101
- **PII in some contexts** — depends on GDPR/CCPA requirements
- **Full request bodies** — can contain sensitive data

```javascript
// ❌ NEVER DO THIS - logs the full secret
console.log('API response:', response);

// ✅ Safe approach - log reference, not content
console.log('API response:', { 
  status: response.status, 
  durationMs: 245,
  requestId: response.headers['x-request-id']
});
```

## Log Levels: Use Them Properly

Log levels exist for a reason. Misusing them creates noise.

| Level | When to Use | Example |
|-------|-------------|---------|
| DEBUG | Development details, never in prod | "Cache key generated: user:123:profile" |
| INFO | Normal operations | "User logged in", "Order created" |
| WARN | Something unexpected, but recoverable | "Retry attempt 2/3 for external API" |
| ERROR | Something failed, needs attention | "Payment gateway timeout", "Database connection lost" |
| FATAL | App can't continue | "Config missing", "Database migration failed" |

**Common mistake:** Using ERROR for things that aren't errors.

```javascript
// ❌ This isn't an error - user just didn't exist
logger.error('User not found', { user_id: 'u_123' });

// ✅ This is correct - it's expected behavior
logger.info('User lookup returned empty', { user_id: 'u_123', result: 'not_found' });
```

## Request IDs: Your Debugging Best Friend

Every request should have a unique ID. Pass it through your entire stack.

```javascript
// middleware to generate or extract request ID
app.use((req, res, next) => {
  req.requestId = req.headers['x-request-id'] || crypto.randomUUID();
  res.setHeader('x-request-id', req.requestId);
  
  // Add to all logs for this request
  logger.defaultMeta = { request_id: req.requestId };
  next();
});

// Now all logs include request_id automatically
logger.info('Processing payment', { user_id: 'u_123' });
// {"request_id": "abc-123", "level": "INFO", "msg": "Processing payment", "user_id": "u_123"}
```

When a user reports "I got an error at 2pm", you can search for their request ID and see every log line from that request—database queries, external API calls, the error itself.

## Logging External Dependencies

When your app calls an external API, log what matters:

```javascript
// ✅ Good - useful info, no secrets
logger.info('External API call', {
  service: 'stripe',
  endpoint: '/v1/charges',
  method: 'POST',
  duration_ms: 234,
  status: 200,
  request_id: requestId
});

// ❌ Bad - logs sensitive data
logger.info('Stripe charge', {
  card_number: '4242424242424242',  // NEVER
  cvc: '123',                       // NEVER
  amount: 9999
});
```

## Performance: Don't Let Logging Kill Your App

Logging has overhead. Bad logging has *significant* overhead.

### Synchronous Logging Blocks

```javascript
// ❌ Blocks the event loop
fs.appendFileSync('/var/log/app.log', JSON.stringify(logEntry) + '\n');
```

### Asynchronous is Better

```javascript
// ✅ Non-blocking
logger.info('User action', { user_id: 'u_123' }); // Winston, Pino, etc. handle async
```

### Structured Logging Libraries

For Node.js, use a structured logging library:

| Library | Speed | Features |
|---------|-------|----------|
| **Pino** | Fastest (~5x Winston) | JSON, pretty-print, child loggers |
| **Winston** | Moderate | Transports, formats, widely used |
| **Bunyan** | Fast | JSON, CLI tool for pretty-print |

```javascript
// Pino example - extremely fast
const pino = require('pino');
const logger = pino({ 
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label })
  }
});

// Child loggers for context
const userLogger = logger.child({ user_id: 'u_123' });
userLogger.info('Created order'); // Automatically includes user_id
```

## Common Patterns

### Log at Service Boundaries

Every time your code crosses a boundary (database, external API, cache), log it:

```javascript
async function fetchUser(userId) {
  const start = Date.now();
  try {
    const user = await db.users.findById(userId);
    logger.info('Database query', {
      operation: 'users.findById',
      duration_ms: Date.now() - start,
      result: user ? 'found' : 'not_found'
    });
    return user;
  } catch (error) {
    logger.error('Database query failed', {
      operation: 'users.findById',
      duration_ms: Date.now() - start,
      error: error.message
    });
    throw error;
  }
}
```

### Don't Log in Hot Paths (Every Request)

If your API handles 10k requests/second, don't log every single one at INFO level:

```javascript
// ❌ Creates massive log volume
app.get('/api/health', (req, res) => {
  logger.info('Health check'); // 10k logs per second!
  res.json({ status: 'ok' });
});

// ✅ Only log anomalies
app.get('/api/health', (req, res) => {
  res.json({ status: 'ok' });
  // No log needed - this is noise
});
```

### Sampling for High-Volume Logs

For high-traffic endpoints, use sampling:

```javascript
// Log 1% of requests for metrics
if (Math.random() < 0.01) {
  logger.info('Request processed', { 
    endpoint: '/api/search', 
    duration_ms: elapsed,
    sample_rate: 0.01 
  });
}
```

## Trade-offs

Structured logging isn't free:

- **Disk space:** JSON logs are larger than plain text. Plan your log retention.
- **Parsing overhead:** Your log aggregator needs to parse JSON. Test at scale.
- **Developer friction:** `console.log` is easy; structured logging requires discipline.

But the alternative—debugging production issues with unstructured logs—costs far more in engineering time.

## Actionable Takeaways

1. **Switch to structured JSON logging** — Pino, Winston, or Bunyan for Node.js
2. **Add request IDs to every request** — pass them through your entire stack
3. **Log at service boundaries** — database queries, external APIs, cache misses
4. **Use log levels correctly** — ERROR isn't for "user not found"
5. **Never log secrets** — passwords, tokens, credit cards, PII
6. **Set up log aggregation** — local logs don't scale; use Datadog, Logtail, or similar
7. **Add timing to external calls** — duration_ms is invaluable for debugging slow requests