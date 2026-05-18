# Graceful Degradation Patterns

Your payment service is down. What happens to your e-commerce site?

If you built things right, users can still browse products, add things to their cart, and maybe even check out — it just won't charge until payment comes back. Annoying for the business, but users get their shopping done.

If you didn't? The whole site crashes. Every page. Every feature. Including the login page.

That's the difference between a system that degrades gracefully and one that falls over like a Jenga tower the moment you pull the wrong block.

## What Goes Wrong Without Graceful Degradation

The classic failure cascade in a microservice architecture:

```
User tries to view product page
  → Product service needs user info → calls User Service
  → User Service returns 500
  → Product service throws exception
  → API Gateway returns 500
  → User sees error page
```

One failed dependency just killed the entire user experience. Even the parts that don't need that dependency (like showing cached product images) are gone.

This is the **cascading failure** pattern, and it's surprisingly easy to trigger.

## The Core Idea: Every Dependency Is Optional

The mental shift you need: **treat every external dependency as potentially unavailable**. Not "this might fail" — "this WILL fail, and I need to decide what happens when it does."

```python
# ❌ Everything depends on everything working
def get_product_page(product_id):
    user = user_service.get_user(current_user_id())
    recommendations = recommendation_engine.get_for_user(user.id)
    reviews = review_service.get_reviews(product_id)
    inventory = inventory_service.check_stock(product_id)
    
    return render_product(product_id, user, recommendations, reviews, inventory)

# ✅ Failures are isolated — each feature handles its own fallback
def get_product_page(product_id):
    user = _try_get_user()           # Returns None or cached version
    recommendations = _try_get_recs() # Returns empty list on failure
    reviews = _try_get_reviews()      # Returns cached or empty
    inventory = _try_get_stock()      # Returns "unknown" or cached
    
    return render_product(product_id, user, recommendations, reviews, inventory)
```

Each call now has a fallback. The page renders even if three of four services are down.

## Degradation Patterns

There are a few common patterns for handling partial failure. Pick based on your use case.

### 1. Fail Silent (Return Defaults)

The simplest approach: when a dependency fails, return a safe default and move on.

```javascript
// Graceful degradation — return empty state instead of error
async function getRecommendations(userId) {
  try {
    const response = await fetch(`/recommendations/${userId}`);
    return await response.json();
  } catch (err) {
    logger.warn('Recommendations unavailable, returning empty', { userId, error: err.message });
    return []; // Empty array — the page just won't show recommendations
  }
}
```

**When to use:** Non-critical features — recommendations, "you might also like", social features.

**When to avoid:** Anything that affects correctness or security.

### 2. Stale Cache Fallback

If you can't get fresh data, serve what you have — even if it's outdated.

```python
def get_inventory_status(product_id):
    try:
        status = inventory_service.check(product_id)
        cache.set(f"inventory:{product_id}", status, ttl=300)
        return status
    except InventoryServiceError:
        cached = cache.get(f"inventory:{product_id}")
        if cached:
            logger.info(f"Using cached inventory for {product_id}")
            return cached
        return {"available": True, "quantity": None}  # Assume available
```

The user doesn't see an error. They might see slightly outdated stock info, but they can still add to cart.

**When to use:** Read-heavy endpoints where stale data is acceptable.

**When to avoid:** Real-time systems (trading, live monitoring) where stale data is dangerous.

### 3. Feature Toggle Degradation

Not all features are equal. Define tiers of functionality:

| Tier | Features | Behavior on Failure |
|------|----------|-------------------|
| **Critical** | Login, Checkout, Payment | Block user, show clear error |
| **Important** | Product Search, Cart | Show degraded version |
| **Nice-to-have** | Recommendations, Reviews | Silently hide component |
| **Cosmetic** | Avatars, Themes | Show placeholder |

A key insight: **even critical features can degrade**. If checkout is down but the payment gateway is up, you can accept orders and process them later.

```javascript
// Feature flag based degradation
const featureFlags = {
  recommendations: true,   // Toggle this off if rec service is unhealthy
  reviews: true,
  inventory: true,
};

function getPageDegradationLevel(failedServices) {
  if (failedServices.includes('payment') && failedServices.includes('auth')) {
    return 'read_only'; // Can browse but can't purchase
  }
  if (failedServices.includes('search')) {
    return 'degraded_search'; // Can browse by category but not search
  }
  return 'normal';
}
```

### 4. Circuit Breaker with Fallback

The circuit breaker pattern (from the microservices patterns article) pairs perfectly with graceful degradation.

```python
# Circuit breaker wraps dependency calls with fallback
from pybreaker import CircuitBreaker

breaker = CircuitBreaker(fail_max=5, reset_timeout=30)

def get_user_profile(user_id):
    try:
        return breaker.call(user_service.get_profile, user_id)
    except CircuitBreakerError:
        # Circuit is open — use cached profile
        cached = cache.get(f"profile:{user_id}")
        if cached:
            return cached
        return {"display_name": "User", "avatar_url": DEFAULT_AVATAR}
```

The circuit breaker prevents cascading failures. The fallback provides a degraded experience.

### 5. Request Collapsing for Degraded Reads

When things are slow, batch requests instead of hammering the failing service.

```python
# Instead of N requests to a struggling service, batch them
request_queue = {}
batch_timer = None

def get_user_batch(user_ids):
    """Collect requests for 50ms then batch them"""
    for uid in user_ids:
        request_queue[uid] = True
    
    def flush():
        ids = list(request_queue.keys())
        request_queue.clear()
        return user_service.batch_get(ids)
    
    # Schedule flush in 50ms
    return schedule_batch(flush, delay_ms=50)
```

Reduces load on the struggling service, making it more likely to recover.

## Real‑World Example: Netflix's Hystrix

Netflix literally wrote the book on this. Their Hystrix library (now in maintenance mode, replaced by resilience4j) pioneered most of these patterns:

- **Thread isolation** — each dependency gets its own thread pool
- **Circuit breakers** — fail fast when a dependency is unhealthy
- **Fallbacks** — return cached or default data
- **Request collapsing** — batch requests automatically
- **Metrics** — monitor every dependency's health

The key insight from Hystrix: **fail fast is better than fail slow**. A 100ms timeout that returns cached data is better than a 10-second timeout that eventually errors out.

## When NOT to Degrade Gracefully

Not everything should degrade. Some things must fail visibly:

- **Financial transactions** — don't silently "degrade" a payment
- **Security checks** — don't bypass auth because Auth Service is down
- **Data mutations** — don't show "saved" when the write actually failed

For these, use the **Fail loud** pattern: clear error messages that tell the user exactly what happened and what to do next.

```
❌ "Something went wrong" → User confused, no action they can take
✅ "Payment is temporarily unavailable. Your cart items are saved — try again in a few minutes."
```

## Takeaways

- **Every dependency WILL fail.** Plan for it before it happens.
- **Return defaults, not errors.** An empty recommendation section is better than a 500 page.
- **Cache aggressively.** That stale data from 5 minutes ago is usually better than nothing.
- **Use circuit breakers.** They turn a 10-second timeout into a 10ms fallback.
- **Degrade feature-by-feature, not page-by-page.** One broken service shouldn't break your whole UI.
- **Know what's critical.** Financial transactions and security checks should fail loud, not degrade.
