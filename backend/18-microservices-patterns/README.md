# Microservices Patterns: Circuit Breaker, Bulkhead, Retry

Your payment service is down. But now your order service is hanging too, waiting on connections that'll never complete. Users are seeing timeouts everywhere, and your entire system is grinding to a halt—all because one service failed.

That's the reality of distributed systems. Services depend on other services, and when one fails, the failure cascades. Let's look at three patterns that prevent a single failure from taking down everything.

## The Problem: Cascading Failures

Here's what happens without resilience patterns:

```
User Request → Order Service → Payment Service (DOWN)
                    ↓
            Thread waits... waits... waits...
                    ↓
            All threads exhausted
                    ↓
            Order Service can't accept new requests
                    ↓
            Entire system affected
```

One slow or failing service creates a thread pool exhaustion domino effect. The caller keeps allocating resources waiting for responses that'll never come.

## Circuit Breaker

The circuit breaker pattern prevents your service from hammering a failing dependency. It works like an electrical circuit breaker—when things go wrong, it trips to protect the system.

### Three States

| State | Behavior | When It Transitions |
|-------|----------|---------------------|
| **Closed** | Requests pass through normally | Opens after X failures |
| **Open** | Requests fail fast, no calls made | Transitions to half-open after timeout |
| **Half-Open** | Limited test requests allowed | Closes on success, opens on failure |

### Implementation Example (with Resilience4j)

```java
// Circuit breaker configuration
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)                    // Open after 50% failures
    .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait before retry
    .slidingWindowSize(10)                        // Evaluate last 10 calls
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("paymentService", config);

// Wrap your calls
Supplier<Order> supplier = CircuitBreaker.decorateSupplier(
    circuitBreaker, 
    () -> paymentClient.processPayment(order)
);

// Execute with circuit breaker protection
Try<Order> result = Try.ofSupplier(supplier);
```

### What It Looks Like in Practice

```
Normal operation (CLOSED):
Client → Order Service → Payment Service → Response

After failures detected (OPEN):
Client → Order Service → ✗ (fail fast, no call made)
                            ↓
                    Return fallback immediately

After timeout (HALF-OPEN):
Client → Order Service → Payment Service (test call)
                            ↓
                    Success? → CLOSE circuit
                    Failure? → OPEN circuit again
```

### ❌ vs ✅ Fallback Handling

```java
// ❌ Bad: No fallback, exception propagates
Order result = circuitBreaker.executeSupplier(() -> 
    paymentClient.processPayment(order)
);

// ✅ Good: Graceful fallback
Order result = circuitBreaker.executeSupplier(() -> 
    paymentClient.processPayment(order)
);
// With fallback:
Order result = circuitBreaker.executeSupplier(
    () -> paymentClient.processPayment(order),
    () -> Order.pendingPayment(order)  // Fallback: queue for later
);
```

## Bulkhead

Named after ship compartments that prevent one breach from sinking the entire vessel, bulkhead isolation limits the resources dedicated to any one dependency.

### Why You Need It

Without bulkheads, a slow dependency consumes all your threads:

```
Payment Service (slow): 80 threads waiting
Inventory Service (slow): 15 threads waiting
User Service (healthy): 5 threads remaining
                    ↓
Your entire app is degraded
```

With bulkheads, each dependency gets its own thread pool:

```
Payment Service: 30 thread limit (all busy)
Inventory Service: 15 thread limit (all busy)
User Service: 20 thread limit (available!)
                    ↓
User Service still works despite other failures
```

### Implementation

```java
// Thread pool bulkhead
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(20)       // Max concurrent calls
    .coreThreadPoolSize(10)
    .queueCapacity(5)            // Queue before rejection
    .build();

ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("paymentBulkhead", config);

// Use it
Supplier<Future<Order>> supplier = ThreadPoolBulkhead.decorateSupplier(
    bulkhead,
    () -> paymentClient.processPayment(order)
);
```

### Semaphore vs Thread Pool Bulkheads

| Type | How It Works | When to Use |
|------|--------------|-------------|
| **Semaphore** | Limits concurrent access, no queuing | Lightweight, no async overhead |
| **Thread Pool** | Dedicated thread pool per dependency | Need queuing, timeout per call |

## Retry

Sometimes things just fail transiently—a network blip, a momentary overload. Retrying makes sense, but only if done carefully.

### The Danger of Naive Retries

```java
// ❌ NEVER DO THIS
for (int i = 0; i < 10; i++) {
    try {
        return paymentClient.process(order);
    } catch (Exception e) {
        // Immediate retry - hammers the struggling service
        continue;
    }
}
```

This is how you DDOS your own services. When payment service is struggling, 10 services each retry 10 times instantly—that's 100x load on an already struggling system.

### Smart Retry with Exponential Backoff

```java
// ✅ Exponential backoff with jitter
RetryConfig config = RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofMillis(500))
    .exponentialBackoffMultiplier(2)  // 500ms, 1s, 2s
    .retryExceptions(IOException.class, TimeoutException.class)
    .ignoreExceptions(BusinessException.class)  // Don't retry these
    .build();

Retry retry = Retry.of("paymentRetry", config);

// Execute with retry
Supplier<Order> supplier = Retry.decorateSupplier(
    retry,
    () -> paymentClient.processPayment(order)
);
```

### Why Jitter Matters

Without jitter, all retrying clients hit at the same moment:

```
Without jitter:
Client A: retry at 1.0s, 2.0s, 4.0s
Client B: retry at 1.0s, 2.0s, 4.0s
Client C: retry at 1.0s, 2.0s, 4.0s
                    ↓
Synchronized load spikes

With jitter:
Client A: retry at 0.9s, 1.8s, 4.2s
Client B: retry at 1.1s, 2.3s, 3.9s
Client C: retry at 1.0s, 1.7s, 4.4s
                    ↓
Spread out, gentler on the system
```

```java
// Adding jitter in Resilience4j
RetryConfig config = RetryConfig.custom()
    .waitDuration(Duration.ofMillis(500))
    .exponentialBackoffMultiplier(2)
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(500, 2, 0.5))
    .build();
```

## Combining All Three

The real power comes from layering these patterns:

```java
// Full resilience stack
CircuitBreaker circuitBreaker = CircuitBreaker.of("payment", circuitBreakerConfig);
ThreadPoolBulkhead bulkhead = ThreadPoolBulkhead.of("payment", bulkheadConfig);
Retry retry = Retry.of("payment", retryConfig);

Supplier<Order> supplier = Supplier.of(() -> paymentClient.processPayment(order))
    .andThen(ThreadPoolBulkhead.decorateSupplier(bulkhead))
    .andThen(Retry.decorateSupplier(retry))
    .andThen(CircuitBreaker.decorateSupplier(circuitBreaker));

// Order matters: bulkhead → retry → circuit breaker
```

### Why This Order?

1. **Bulkhead first**: Limit how many threads even attempt the call
2. **Retry second**: Handle transient failures within the allowed threads
3. **Circuit breaker last**: Stop calling if it's truly down

## Trade-offs

These patterns aren't free:

| Pattern | Cost | Consideration |
|---------|------|---------------|
| Circuit Breaker | State management, tuning complexity | Worth it for critical external dependencies |
| Bulkhead | Thread pool overhead, underutilization | Best for high-risk, high-traffic dependencies |
| Retry | Increased latency on failures, load spikes | Only for transient errors, never for 4xx |

### Don't Apply These Everywhere

```java
// ❌ Over-engineering: retry + bulkhead + circuit breaker for a simple cache lookup
// ✅ Just let it fail fast with a fallback
Supplier<String> cacheLookup = () -> cache.get(key);
// If cache fails, return empty - no need for full resilience stack

// ✅ Use the full stack for critical, potentially slow operations
Supplier<Order> paymentProcessing = () -> paymentClient.processPayment(order);
```

## When to Use Each Pattern

| Scenario | Pattern | Why |
|----------|---------|-----|
| External API that occasionally times out | Retry (with backoff) | Transient failures, usually recovers |
| Database that sometimes gets slow | Circuit Breaker | Fail fast, don't accumulate waiting threads |
| Third-party service with known instability | All three | High risk, protect everything |
| Internal cache lookup | None (or simple timeout) | Fast, local, should just work |
| Payment processing | All three + timeout | Critical path, must be protected |

## Actionable Takeaways

1. **Start with timeouts**: Set reasonable timeouts on all external calls before adding complexity
2. **Add circuit breakers**: For any external dependency that could fail or become slow
3. **Use bulkheads for critical paths**: When one slow dependency shouldn't affect the rest
4. **Retry only transient failures**: Never retry 4xx errors, only network issues and 5xx
5. **Always add jitter**: Prevent synchronized retry storms
6. **Monitor your circuit breakers**: Track open/close transitions—they're signals something's wrong
7. **Test your resilience**: Use chaos engineering ( Chaos Monkey, etc.) to verify patterns work

The goal isn't to prevent failures—they'll happen. The goal is to fail gracefully and keep as much of your system running as possible when they do.