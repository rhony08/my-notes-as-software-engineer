# Circuit Breaker: Advanced Patterns

Every distributed system has that one dependency that's flaky. Maybe it's a legacy service that occasionally times out after 30 seconds. Or a third-party API that goes down every other Tuesday. You throw a basic circuit breaker on it, problem solved, right?

Well, mostly. But as your system grows, the simple "open/closed/half-open" state machine starts showing cracks. Latency spikes that don't trigger error thresholds. Partial failures where some endpoints work and others don't. Cascading failures across multiple circuit breakers. The basics get you 80% of the way. The remaining 20% is where production systems live or die.

## Why the Basic Pattern Isn't Enough

The classic circuit breaker works on a simple premise: count failures past a threshold → open the circuit → reject requests → try again after a timeout. Here's what that looks like in practice:

```python
# Basic circuit breaker - works but has blind spots
class SimpleCircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.state = "closed"
        self.last_failure_time = None

    def call(self, fn):
        if self.state == "open":
            if time_since(self.last_failure_time) > self.recovery_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()

        try:
            result = fn()
            if self.state == "half-open":
                self.state = "closed"
                self.failure_count = 0
            return result
        except Exception:
            self.failure_count += 1
            self.last_failure_time = now()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"
            raise
```

This works, but there's a bunch of things it doesn't handle well:

- **Only counts errors, not slow requests** — A service that takes 10 seconds (but doesn't error) will still hold up your threads
- **Same threshold for everything** — Your critical auth endpoint gets the same treatment as your analytics endpoint
- **Binary state transitions** — No gradual throttling, just all-or-nothing
- **No recovery probing** — In half-open state, it lets through one request and makes you wait the full timeout

## Advanced Pattern 1: Timeout-Based Circuit Breaking

Slow requests are worse than fast failures. A fast "503" costs you 50ms. A slow timeout costs you 10 seconds of holding a thread/connection/connection_pool_slot. Basic circuit breakers miss this entirely.

```python
# ❌ Only counts exceptions
if exception:
    self.failure_count += 1

# ✅ Counts slow responses too
def call(self, fn, timeout_ms=500):
    start = time.monotonic()
    try:
        result = fn(timeout=timeout_ms / 1000)
        elapsed = (time.monotonic() - start) * 1000

        # Count slow responses as failures
        if elapsed > self.slow_threshold_ms:
            self.slow_count += 1
            if self.slow_count >= self.slow_failure_threshold:
                self.trip_circuit("slow responses")
        return result
    except TimeoutError:
        self.failure_count += 1
        if self.failure_count >= self.failure_threshold:
            self.trip_circuit("timeouts")
        raise
```

**The key insight:** You want to detect degradation *before* it becomes a hard failure. If response times go from 50ms to 2s, that's a warning sign, not a crisis — but only if you're watching.

## Advanced Pattern 2: Per-Endpoint Circuit Breakers

Not all endpoints are equal. Your `/api/orders` endpoint might need 99.99% availability. Your `/api/analytics/track` endpoint might be nice-to-have. Treating them the same means either:

- Setting thresholds too low → unnecessary outages on important endpoints
- Setting thresholds too high → slow failures cascade on less important ones

```java
// Circuit breaker per endpoint
public class EndpointAwareCircuitBreaker {
    private Map<String, CircuitBreakerState> endpoints = new ConcurrentHashMap<>();

    public <T> T call(String endpoint, Supplier<T> fn, EndpointConfig config) {
        var breaker = endpoints.computeIfAbsent(endpoint,
            k -> new CircuitBreakerState(config.threshold(), config.timeout()));

        if (breaker.getState() == State.OPEN) {
            if (config.isCritical()) {
                // For critical endpoints, retry via fallback
                return fn.get();
            }
            throw new CircuitOpenException(endpoint);
        }
        // ... proceed with call
    }
}
```

| Endpoint | Threshold | Timeout | Fallback | Priority |
|----------|-----------|---------|----------|----------|
| GET /orders/list | 5 errors | 500ms | Cache | Critical |
| POST /orders | 3 errors | 2s | Reject + queue | Critical |
| POST /analytics | 20 errors | 5s | Drop | Best-effort |
| GET /reports | 10 errors | 10s | Serve stale | Medium |

This lets you be aggressive on endpoints that can degrade safely and conservative on the ones that can't.

## Advanced Pattern 3: Adaptive Circuit Breaker (Metrics-Driven)

Static thresholds are a lie. Your system's behavior changes throughout the day — more traffic at peak, different failure modes during deploys, varying downstream latency. A circuit breaker that adapts is more useful than one that's perfectly tuned for last Tuesday.

```python
class AdaptiveCircuitBreaker:
    def __init__(self):
        # Track rolling window of success/failure/latency
        self.metrics = RollingWindow(size_seconds=60)
        self.base_latency = None  # Learned baseline

    def calculate_threshold(self):
        # Dynamic threshold: 3 standard deviations above baseline
        window_stats = self.metrics.get_stats()
        if window_stats.total_requests < 100:
            return self.default_threshold  # Not enough data

        # Adjust threshold based on historical error rate
        error_rate = window_stats.errors / window_stats.total_requests
        if error_rate > 0.01:  # 1% error rate → tighten threshold
            return max(self.default_threshold * 0.5, 3)
        elif error_rate < 0.001:  # 0.1% error rate → loosen
            return min(self.default_threshold * 1.5, 20)
        return self.default_threshold
```

**When this saves you:** Imagine a deployment that introduces a 2% error rate — barely enough to trip a standard breaker with threshold=10. An adaptive breaker notices the sustained error rate increase and gradually tightens the threshold. It catches the slow degradation that static breakers miss.

## Advanced Pattern 4: Bulkhead + Circuit Breaker Combo

Circuit breakers protect against cascading failures at the network level. Bulkheads protect at the resource level. Together, they're a power couple.

```
┌─────────────────────────────────────────┐
│           Application                    │
│  ┌────────┐  ┌────────┐  ┌────────┐    │
│  │ Auth   │  │ Orders │  │ Reports│    │
│  │ Pool=5 │  │ Pool=10│  │ Pool=3 │    │
│  │  CB ✓  │  │  CB ✓  │  │  CB ✓  │    │
│  └────┬───┘  └────┬───┘  └────┬───┘    │
│       │           │           │         │
│  ┌────▼───────────▼───────────▼────┐    │
│  │      Downstream Services        │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

The pattern: each dependency gets its own thread pool (bulkhead) and its own circuit breaker. If the auth service starts timing out:

1. The circuit breaker opens for auth calls ✓
2. The auth bulkhead stops consuming threads ✓
3. Orders and reports bulkheads keep working ✓
4. The overall app is degraded (auth is down) but not dead ✓

```java
// ❌ Single thread pool - one slow dependency blocks everything
ExecutorService sharedPool = Executors.newFixedThreadPool(20);

// ✅ Bulkhead per dependency
ExecutorService authPool = Executors.newFixedThreadPool(5);
ExecutorService ordersPool = Executors.newFixedThreadPool(10);
ExecutorService reportsPool = Executors.newFixedThreadPool(3);
```

## Advanced Pattern 5: Probing Recovery

Standard half-open state sends one request through and waits for the full recovery timeout. That's slow and wasteful. Better approaches:

**Gradual recovery:** In half-open, gradually increase the request percentage:
- Second 0: 5% of requests
- Second 10: 25% of requests
- Second 30: 50% of requests
- Second 60: 100% of requests (fully closed)

If any batch fails above threshold, snap back to open immediately.

**Shadow probes:** Send synthetic "health check" requests to the dependency even when the circuit is open. When you see a successful shadow probe, transition to half-open faster.

```python
class ProbingCircuitBreaker:
    def __init__(self):
        self.state = "open"
        self.probe_interval = 5  # seconds
        self.last_probe = 0

    def probe_recovery(self):
        # Send lightweight health check while circuit is open
        if time.monotonic() - self.last_probe > self.probe_interval:
            try:
                # Lightweight ping - don't charge actual user requests
                self.dependency.health_check(timeout_ms=100)
                self.evaluate_recovery()
            except Exception:
                pass  # Still down, keep waiting
            self.last_probe = time.monotonic()

    def evaluate_recovery(self):
        # Consider recovery based on multiple successful probes
        self.successful_probes += 1
        if self.successful_probes >= 3:  # 3 successful probes before letting traffic
            self.state = "half-open"
            self.successful_probes = 0
```

## Pattern Comparison

| Pattern | What it solves | Complexity | Trade-off |
|---------|---------------|------------|-----------|
| Timeout-based | Slow degradation | Low | Need to tune slow threshold |
| Per-endpoint | Unequal criticality | Medium | More state to manage |
| Adaptive | Changing conditions | High | Risk of wrong adaptation |
| Bulkhead combo | Resource exhaustion | Medium | More threads, more memory |
| Probing recovery | Slow recovery | Medium | Extra health check traffic |

## When You've Gone Too Far

Circuit breakers are great until you have circuit breakers on your circuit breakers. Things to watch for:

- **Too many breakers** — An endpoint that calls 10 downstream services could have 10 circuit breakers. Now your error handling is more complex than the actual business logic.
- **False confidence** — A circuit breaker doesn't make a flaky service reliable. It makes the flakiness *contained*. You still need to fix the root cause.
- **Recovery thrashing** — If your half-open recovery lets traffic through too aggressively, you get repeated open → half-open → open cycles that look like system hiccups.

## What You Actually Need in Production

From real experience running circuit breakers in production (mostly via [resilience4j](https://resilience4j.readme.io/) on the JVM side or custom implementations in Go):

1. **Start with per-endpoint configuration** — It's the highest ROI for the complexity
2. **Add timeout-based breaking before adaptive** — Slow requests are usually your biggest problem
3. **Wire up monitoring immediately** — A circuit breaker you can't see opening isn't protecting you, it's hiding problems
4. **Test your recovery** — The half-open → closed transition is when failures actually matter. If your dependency recovers but your breaker doesn't let it, you're inventing outages

### Actionable Takeaways

- Don't just count exceptions — timeouts and slow responses are early warning signals
- Isolate per-endpoint thresholds; critical and best-effort paths shouldn't share the same breaker
- Pair circuit breakers with bulkheads for real isolation, not just network-level protection
- Use gradual recovery with shadow probes instead of fixed-time half-open windows
- Monitor circuit breaker state changes — unexpected state transitions point to deeper issues
- Know when to stop: circuit breakers are containment, not a substitute for fixing flaky dependencies
