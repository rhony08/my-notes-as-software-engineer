# Effective Logging for Backend Applications

Logging is often treated as an afterthought in backend development—something we add when debugging or when something goes wrong. But effective logging is a critical component of production-ready applications. It provides visibility into system behavior, helps diagnose issues, and enables monitoring and alerting. In this article, we'll explore how to implement logging that actually helps you when things go wrong.

## Table of Contents

- [Why Logging Matters](#why-logging-matters)
- [Log Levels](#log-levels)
- [Structured vs Unstructured Logs](#structured-vs-unstructured-logs)
- [What to Log and What Not to Log](#what-to-log-and-what-not-to-log)
- [Correlation IDs for Request Tracing](#correlation-ids-for-request-tracing)
- [Log Aggregation Basics](#log-aggregation-basics)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## Why Logging Matters

Without logs, debugging production issues is like flying blind. Good logs provide:

### 1. Debugging Information

When errors occur, logs tell you:
- What happened
- When it happened
- Where in the code it happened
- The context (user, request, data)

### 2. Performance Insights

Logs can reveal:
- Slow database queries
- API response times
- Bottlenecks in request processing
- Resource usage patterns

### 3. Security Monitoring

Logs help detect:
- Unusual access patterns
- Failed authentication attempts
- Suspicious API usage
- Data access anomalies

### 4. Business Intelligence

Beyond debugging, logs can track:
- User behavior
- Feature usage
- Transaction flows
- System health trends

## Log Levels

Log levels indicate the severity and importance of a message. Using them correctly is crucial for filtering noise from signal.

### DEBUG

**Use for:** Detailed information for development and debugging

```javascript
logger.debug('Processing payment', { 
  userId: user.id, 
  amount: payment.amount,
  paymentMethod: payment.method 
});
```

**When to use:** During development or when tracing specific issues. Usually disabled in production.

### INFO

**Use for:** General application flow and significant events

```javascript
logger.info('User registered successfully', { userId: user.id });
logger.info('Payment processed', { orderId: order.id, amount: order.total });
logger.info('Server started', { port: 3000, environment: 'production' });
```

**When to use:** Normal application operations that you want to track. The default level for production.

### WARN

**Use for:** Unexpected situations that don't prevent operation

```javascript
logger.warn('Database connection slow', { 
  queryTime: 2500, 
  threshold: 1000,
  query: 'SELECT * FROM large_table' 
});
logger.warn('Retrying failed request', { attempt: 2, maxRetries: 3 });
```

**When to use:** Issues that should be investigated but don't break functionality.

### ERROR

**Use for:** Errors that prevent the current operation from completing

```javascript
logger.error('Payment processing failed', {
  error: error.message,
  stack: error.stack,
  userId: user.id,
  orderId: order.id
});
```

**When to use:** Exceptions, failed operations, and conditions that require immediate attention.

### Log Level Guidelines

| Level | Production | Development | Purpose |
|-------|------------|-------------|---------|
| DEBUG | Off | On | Detailed debugging |
| INFO | On | On | Normal operations |
| WARN | On | On | Warnings |
| ERROR | On | On | Errors |

## Structured vs Unstructured Logs

### Unstructured Logs (Avoid)

```
User John logged in at 2024-01-15 10:30:00
Payment failed: insufficient funds
Database error: connection timeout after 30s
```

**Problems:**
- Hard to parse programmatically
- Inconsistent formats
- Difficult to query
- No standard fields

### Structured Logs (Preferred)

```json
{"level":"info","message":"User logged in","timestamp":"2024-01-15T10:30:00Z","userId":123,"action":"login"}
{"level":"error","message":"Payment failed","timestamp":"2024-01-15T10:31:00Z","orderId":"ORD-456","reason":"insufficient_funds","amount":99.99}
```

**Benefits:**
- Machine-readable
- Consistent schema
- Easy to query and filter
- Supports aggregation and analysis

### Implementing Structured Logging

**Node.js (Winston):**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});

logger.info('User action', { userId: 123, action: 'purchase' });
```

**Python (structlog):**
```python
import structlog

logger = structlog.get_logger()

logger.info("user_action", user_id=123, action="purchase")
```

**Go (zap):**
```go
import "go.uber.org/zap"

logger, _ := zap.NewProduction()
defer logger.Sync()

logger.Info("user_action",
    zap.Int("user_id", 123),
    zap.String("action", "purchase"),
)
```

## What to Log and What Not to Log

### Log These

**1. Application Lifecycle Events**
```javascript
logger.info('Server starting', { port: 3000, env: 'production' });
logger.info('Database connected', { host: 'db.example.com', poolSize: 10 });
logger.info('Server shutting down', { signal: 'SIGTERM' });
```

**2. Request/Response Summary**
```javascript
logger.info('Request completed', {
  method: 'POST',
  path: '/api/orders',
  statusCode: 201,
  durationMs: 245,
  userId: 123
});
```

**3. Business Events**
```javascript
logger.info('Order created', { orderId: 'ORD-789', userId: 123, total: 99.99 });
logger.info('Payment received', { paymentId: 'PAY-456', amount: 99.99, method: 'credit_card' });
```

**4. Errors with Context**
```javascript
logger.error('Payment failed', {
  error: error.message,
  orderId: 'ORD-789',
  userId: 123,
  amount: 99.99,
  retryCount: 3
});
```

**5. Performance Metrics**
```javascript
logger.info('Slow query detected', {
  query: 'SELECT * FROM orders WHERE...',
  durationMs: 2500,
  thresholdMs: 1000
});
```

### Never Log These

**1. Passwords and Secrets**
```javascript
// ❌ NEVER
logger.info('User login', { username: 'john', password: 'secret123' });

// ✅ DO THIS
logger.info('User login attempt', { username: 'john', success: true });
```

**2. Personal Identifiable Information (PII)**
```javascript
// ❌ NEVER
logger.info('User registered', { 
  email: 'john@example.com',
  ssn: '123-45-6789',
  creditCard: '4532-1234-5678-9012'
});

// ✅ DO THIS
logger.info('User registered', { 
  userId: 'usr_123',
  emailDomain: 'example.com' // If needed for analytics
});
```

**3. Sensitive Business Data**
```javascript
// ❌ NEVER
logger.debug('API response', { apiKey: 'sk-live-abc123', secretToken: 'xyz789' });

// ✅ DO THIS
logger.debug('API response received', { 
  endpoint: '/api/data',
  statusCode: 200,
  responseSize: 1024
});
```

**4. Large Data Payloads**
```javascript
// ❌ NEVER - can overwhelm logs
logger.info('Request body', { body: hugeJsonObject });

// ✅ DO THIS
logger.info('Request received', { 
  contentType: 'application/json',
  contentLength: 5242880,
  fields: Object.keys(hugeJsonObject)
});
```

## Correlation IDs for Request Tracing

In distributed systems, a single user request may flow through multiple services. Correlation IDs tie these scattered logs together.

### What is a Correlation ID?

A unique identifier assigned to a request that propagates through all services involved in processing that request.

### Implementation

**1. Generate on Entry**
```javascript
const { v4: uuidv4 } = require('uuid');

app.use((req, res, next) => {
  req.correlationId = req.headers['x-correlation-id'] || uuidv4();
  res.setHeader('X-Correlation-Id', req.correlationId);
  next();
});
```

**2. Include in All Logs**
```javascript
app.use((req, res, next) => {
  req.logger = logger.child({ correlationId: req.correlationId });
  next();
});

// Usage
app.get('/api/orders', (req, res) => {
  req.logger.info('Fetching orders', { userId: req.user.id });
  // All logs will include correlationId
});
```

**3. Pass to Downstream Services**
```javascript
const response = await fetch('http://payment-service/process', {
  headers: {
    'X-Correlation-Id': req.correlationId,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(paymentData)
});
```

### Benefits

With correlation IDs, you can trace a complete request flow:

```
[API Gateway]      correlationId: abc-123 | Received request POST /orders
[Order Service]    correlationId: abc-123 | Creating order for user 456
[Payment Service]  correlationId: abc-123 | Processing payment $99.99
[Payment Service]  correlationId: abc-123 | Payment successful
[Order Service]    correlationId: abc-123 | Order ORD-789 created
[API Gateway]      correlationId: abc-123 | Response 201 in 245ms
```

## Log Aggregation Basics

In production, logs are spread across multiple servers and containers. Log aggregation collects them in one place for analysis.

### Popular Solutions

**Cloud-Native:**
- AWS CloudWatch
- Google Cloud Logging
- Azure Monitor
- Datadog

**Self-Hosted:**
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Grafana Loki
- Fluentd + Elasticsearch

### Basic Setup (ELK Stack)

**1. Application → Filebeat**
```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  paths:
    - /var/log/myapp/*.log
  json.keys_under_root: true

output.elasticsearch:
  hosts: ["localhost:9200"]
```

**2. Query in Kibana**
```
level: error AND service: "order-service"
correlationId: "abc-123"
durationMs > 1000
```

### Log Retention

Define retention policies based on log level:

| Level | Retention | Reason |
|-------|-----------|--------|
| ERROR | 90 days | Need for incident investigation |
| WARN | 30 days | Troubleshooting reference |
| INFO | 7 days | Operational visibility |
| DEBUG | 1 day | Temporary debugging only |

## Best Practices

### 1. Use a Consistent Schema

Define standard fields for all logs:

```javascript
const baseLogFields = {
  timestamp: new Date().toISOString(),
  service: 'order-service',
  version: '1.2.3',
  environment: process.env.NODE_ENV
};
```

### 2. Log at the Right Level

```javascript
// ❌ Wrong - this is normal operation
logger.error('User logged in', { userId: 123 });

// ✅ Correct
logger.info('User logged in', { userId: 123 });
```

### 3. Include Context, Not Just Messages

```javascript
// ❌ Not helpful
logger.error('Something went wrong');

// ✅ Actionable
logger.error('Payment processing failed', {
  error: error.message,
  orderId: 'ORD-789',
  userId: 123,
  paymentMethod: 'credit_card',
  attempt: 3,
  correlationId: 'abc-123'
});
```

### 4. Use Appropriate Log Destinations

```javascript
const logger = winston.createLogger({
  transports: [
    // Console for development
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    // File for persistence
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'logs/combined.log' 
    })
  ]
});
```

### 5. Handle Async Context

```javascript
const { AsyncLocalStorage } = require('async_hooks');
const asyncLocalStorage = new AsyncLocalStorage();

// Middleware
app.use((req, res, next) => {
  asyncLocalStorage.run({ correlationId: req.correlationId }, next);
});

// Logger that picks up context
function getLogger() {
  const context = asyncLocalStorage.getStore() || {};
  return logger.child(context);
}

// Usage anywhere in the call stack
getLogger().info('Processing payment');
```

## Conclusion

Effective logging transforms debugging from guesswork into science. By following these practices:

- **Use appropriate log levels** to control verbosity
- **Structure your logs** for machine parsing
- **Include correlation IDs** for distributed tracing
- **Log context, not just messages**
- **Never log sensitive data**
- **Aggregate logs** for centralized analysis

You'll gain visibility into your application's behavior, catch issues faster, and spend less time hunting for bugs in production.

Remember: logs are your eyes into running systems. Invest in good logging practices, and they'll pay dividends when things go wrong.