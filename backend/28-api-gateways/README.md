# API Gateways: Pattern and Implementation

Your frontend calls three microservices for a single page. That means three round trips, three auth checks, three different error formats, and three SSL handshakes. The user's phone on a slow network? They'll be staring at a spinner for seconds.

This is the exact problem an API gateway solves. It sits between your clients and your services, acting as a single entry point that handles cross-cutting concerns so your services don't have to.

## What Actually Happens Without a Gateway

Let's say you're building an e-commerce app. A product page needs:

```
Frontend → Product Service    (product details)
Frontend → Review Service     (ratings)
Frontend → Inventory Service  (stock status)
Frontend → Recommendation API (similar items)
```

Without a gateway, the frontend manages:

- Four different URLs and auth tokens
- Four separate SSL/TLS handshakes
- Error handling for each service independently
- Protocol differences (gRPC here, REST there)
- Rate limiting, request logging, response transformation

❌ **No gateway** — frontend knows too much about backend internals
✅ **With gateway** — frontend calls one endpoint, gateway handles the rest

The frontend should NOT need to know your internal service topology. An API gateway is how you enforce that boundary.

## What an API Gateway Actually Does

At its simplest, an API gateway is a reverse proxy with superpowers. Here's what it typically handles:

| Concern | Without Gateway | With Gateway |
|---------|----------------|--------------|
| Routing | Client knows service URLs | Single entry point, gateway routes |
| Auth | Each service validates tokens | Gateway validates once, passes identity |
| Rate limiting | Each service implements separately | Centralized at gateway |
| Response aggregation | Client makes multiple requests | Gateway aggregates internal calls |
| Protocol translation | Client must support gRPC + REST | Gateway translates between protocols |
| Request logging | Every service logs differently | Gateway provides unified access logs |
| Circuit breaking | Each service implements | Gateway protects downstream services |

## The Request/Response Pipeline

Here's what a typical request looks like going through a gateway:

```
Client → Gateway → [Auth Plugin → Rate Limiter → Router → Circuit Breaker] → Service
         Gateway ← [Response Transformer ← Cache ← Aggregator] ← Service
```

Let's look at each step:

### 1. Auth (Authentication & Authorization)

The gateway extracts and validates the auth token **before** the request reaches your service. If the token is expired or invalid, the gateway returns 401 immediately—no need to waste service resources.

```javascript
// Gateway auth middleware (pseudo-code)
async function authMiddleware(req) {
  const token = extractToken(req.headers.authorization);
  
  if (!token) {
    // ❌ NO AUTH: Reject early, save service cycles
    return { status: 401, body: { error: 'Missing auth token' } };
  }
  
  try {
    const payload = await verifyJWT(token, process.env.JWT_SECRET);
    // ✅ PASS AUTH: Attach user context for downstream services
    req.context.user = { id: payload.sub, role: payload.role };
    return null; // continue
  } catch (err) {
    return { status: 401, body: { error: 'Invalid or expired token' } };
  }
}
```

### 2. Rate Limiting

Instead of implementing rate limiting in every service, do it at the gateway level. This protects ALL your services at once.

```yaml
# gateway rate limit config (Kong-style)
rate_limiting:
  routes:
    /api/v1/orders:
      limit: 100
      window: 60s
    /api/v1/products:
      limit: 1000
      window: 60s
  global:
    limit: 5000
    window: 60s
```

❌ **Per-service rate limiting** — each service reimplements the same logic, inconsistent limits
✅ **Gateway rate limiting** — single source of truth, consistent enforcement, easier to audit

### 3. Routing

The gateway routes requests based on path, headers, or even request body content:

```javascript
// Route table — declarative, easy to read
const routes = [
  // Service-based routing (most common)
  { path: '/api/users/*',      upstream: 'http://user-service:3001' },
  { path: '/api/orders/*',     upstream: 'http://order-service:3002' },
  { path: '/api/products/*',   upstream: 'http://product-service:3003' },
  
  // Version-based routing
  { path: '/api/v1/*',         upstream: 'http://legacy-app:8080' },
  { path: '/api/v2/*',         upstream: 'http://modern-app:8081' },
  
  // Header-based routing (canary deployments)
  { path: '/api/*',            upstream: 'http://stable:3000', 
    headers: { 'X-Canary': null } },
  { path: '/api/*',            upstream: 'http://canary-v2:3000', 
    headers: { 'X-Canary': 'v2' } },
];
```

### 4. Response Aggregation (The "Backend for Frontend" Pattern)

One of the most powerful features: the gateway can call multiple services and combine their responses into a single payload. This is the **Backend for Frontend (BFF)** pattern.

```javascript
// Gateway aggregation — single endpoint, multiple backend calls
async function getProductPage(req) {
  const productId = req.params.id;
  
  // Fire all requests in parallel
  const [product, reviews, stock] = await Promise.all([
    fetch(`http://product-service:3001/products/${productId}`),
    fetch(`http://review-service:3002/products/${productId}/reviews`),
    fetch(`http://inventory-service:3003/products/${productId}/stock`),
  ]);
  
  // Aggregate responses into a single payload
  return {
    product: product.json(),
    reviews: reviews.json(),
    stock: stock.json(),
    // ✅ BFF pattern: frontend gets exactly what it needs, one request
  };
}
```

**Trade-off:** Aggregation increases latency for the gateway itself. The gateway now holds open connections to multiple services. If one service is slow, the entire response is delayed. Use timeouts and parallel requests (like above) to mitigate this.

## Popular API Gateway Solutions

| Tool | Language | Best For | Notes |
|------|----------|----------|-------|
| **Kong** | Lua/OpenResty | Enterprise, plugins ecosystem | Mature, huge plugin library |
| **AWS API Gateway** | Managed | Serverless, AWS stack | Pay-per-use, tight Lambda integration |
| **NGINX + Lua** | C/Lua | Custom, high performance | Full control, steep learning curve |
| **Traefik** | Go | Kubernetes-native | Auto-discovers services, automatic HTTPS |
| **Envoy** | C++ | Service mesh | L7 proxy, sidecar pattern, advanced |
| **Express Gateway** | Node.js | JS-heavy teams | Familiar Express API |
| **Spring Cloud Gateway** | Java | Java/Spring shops | Reactive, non-blocking |

The right choice depends on your stack. If you're on Kubernetes, Traefik or Envoy are no-brainers. If you're on AWS Lambda, use API Gateway. If you want maximum control and performance, NGINX.

## Common Pitfalls

### Gateway Becomes a Monolith

❌ A gateway that has too much business logic is just a monolith with a new name. Keep business logic in your services.

✅ The gateway should handle **cross-cutting concerns only**: auth, routing, rate limiting, logging. Not order processing, not payment validation.

### Single Point of Failure

Your gateway is now the entry point for ALL traffic. If it goes down, everything goes down.

```yaml
# ❌ Single gateway instance
services:
  gateway:
    replicas: 1  # 🚩 One crash = total outage

# ✅ Multiple gateway instances with load balancer
services:
  gateway:
    replicas: 3  # HA — one fails, others keep going
```

Run at least 2-3 gateway instances behind a load balancer. If you're on Kubernetes, rely on its built-in service mesh or run Envoy as a sidecar.

### Performance Overhead

Every request now passes through an extra hop. That adds latency:

```
Direct: Client → Service (1 hop)
Gateway: Client → Gateway → Service (2 hops)
```

For latency-sensitive applications, measure the overhead. In practice, gateway latency is typically 1-10ms per request—negligible for most APIs, but meaningful for real-time systems (gaming, trading, WebSocket-heavy apps).

## When You Don't Need a Gateway

Honestly? Sometimes you don't.

- **Monolithic app** — You have one service. Adding a gateway is unnecessary complexity.
- **Small team, 2-3 services** — A reverse proxy (NGINX/Caddy) might be enough.
- **Internal-only APIs** — If there's no external access, a gateway adds overhead without benefit.

Don't reach for an API gateway because it's trendy. Reach for it when the pain of not having one exceeds the complexity of running one.

## Takeaways

- An API gateway is a single entry point that handles auth, routing, rate limiting, and aggregation
- It decouples clients from backend internals — frontends don't need to know your service topology
- The BFF pattern (response aggregation) reduces client round-trips significantly
- Run multiple instances — never let the gateway become a single point of failure
- Keep business logic in your services, not in the gateway
- Start without one, add it when the pain of not having it is real
