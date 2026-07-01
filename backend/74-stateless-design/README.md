# Stateless Design Principles

Your app was running fine. Then you deployed to production with two instances. Suddenly users get logged out randomly, cart items disappear, and someone reports seeing another person's data. Classic "sticky session required" disaster.

This is what happens when you build stateful apps that can't scale horizontally. Stateless design fixes this—but it's not as simple as "just don't store state."

---

## The Core Idea

Stateless means each request contains **everything** the server needs to process it. No "oh wait, let me check what this user was doing earlier" from the server's memory.

```
       ┌─────────────────┐
       │                 │
User ──▶   Any server    ──▶ Response
       │                 │
       └─────────────────┘
       (Server has zero
        user context)
```

The server doesn't care which instance handles the request. Instance A, B, or C? Doesn't matter—they all see the same thing: a self-contained request.

### What Actually Changes

**Stateful:**
```
Server Memory:
 ┌────────────────────────────┐
 │ User 123: logged_in=true   │
 │ User 123: cart=[item1]     │
 │ User 456: logged_in=true   │
 └────────────────────────────┘

Request: GET /cart
Server: "Let me check... User 123 has [item1] in memory"
```

**Stateless:**
```
Request: GET /cart
Cookie/Token: { user_id: 123, cart: [item1] }

Server: "Cart is right here in the request. Return [item1]."
```

---

## Why Stateless Matters

### Horizontal Scaling

This is the big one. Without statelessness, you can't just add more servers and expect things to work.

**With sticky sessions:**
```
                        ┌──────────┐
                     ┌─▶│ Server A │── User 123's state lives here
                     │  └──────────┘
Load Balancer ───────┤  ┌──────────┐
(sticky cookies)     ├─▶│ Server B │── User 456's state lives here
                     │  └──────────┘
                     │  ┌──────────┐
                     └─▶│ Server C │── New users go here
                        └──────────┘
```

Problem: Server A crashes → User 123 loses everything. Or suddenly there's 5x traffic to Server A and it's the bottleneck.

**Stateless:**
```
                        ┌──────────┐
                     ┌─▶│ Server A │── Any server can handle any request
                     │  └──────────┘
Load Balancer ───────┼─▶│ Server B │── Same
(round robin)        │  └──────────┘
                     │  ┌──────────┐
                     └─▶│ Server C │── Same
                        └──────────┘
```

Add 10 more servers? Works immediately. Kill one? No sessions lost.

### Fault Tolerance

Stateless servers can die without drama. Since no instance holds unique state, any other instance picks up the work. Zero session loss, zero user impact.

### Simplifies Deployments

Rolling updates? Canary deployments? Zero-downtime releases? All straightforward when every server is interchangeable.

---

## Where State Actually Goes

Stateless doesn't mean "no state"—it means state lives outside the application server. The obvious answer is a database, but the real art is choosing the right storage for each type of state.

### Session State → Redis/Memcached

```
User logs in → Session created in Redis
                     │
                     ├── Key: session_123
                     └── Value: { user_id, role, expires_at }

Every request: Server checks Redis for session
```

**Why Redis?** In-memory, sub-millisecond reads, built-in TTL expiry. Memcached is lighter but has no persistence if you need it.

### Configuration State → Environment/Config Service

Don't hardcode config. Don't stuff it in session memory either.

| Type | Where | Example |
|------|-------|---------|
| Non-sensitive | Environment variables | `DATABASE_URL` |
| Secrets | Vault/Secret Manager | `API_KEY` |
| Dynamic | Config service (Consul, etcd) | Feature flags |

### User-Generated State → Database

Cart items, saved preferences, documents—this all goes in your primary database. The server fetches it on each request and never caches it locally.

```
// ❌ Stateful - stored in server memory
class CartService {
  private carts = new Map<string, CartItem[]>();
  
  addItem(userId: string, item: CartItem) {
    this.carts.get(userId)?.push(item); // Gone if server restarts!
  }
}

// ✅ Stateless - stored in database
class CartService {
  async addItem(userId: string, item: CartItem) {
    await db.cartItems.insert({ userId, ...item }); // Survives anything
  }
}
```

---

## Making Your API Stateless

The most practical change: move session data into tokens.

### Before (stateful session)
```
Login → Server creates session, stores it in memory, returns session_id cookie

Request → Server looks up session_id in memory → finds user data
```

### After (stateless JWT)
```
Login → Server creates JWT with user data, signs it, returns as token

Request → Server verifies JWT signature → reads user data from token
```

The JWT is cryptographically signed. The server trusts it because it knows the signature is valid and the token hasn't expired.

**The trade-off:** JWTs can't be revoked instantly (unless you maintain a blocklist, which defeats some of the stateless benefit). Good for auth session duration of 15-60 minutes with refresh tokens for longer access.

```
// JWT payload example
{
  "sub": "user_123",
  "name": "Alice",
  "role": "admin",
  "iat": 1719000000,
  "exp": 1719003600
}
// Server verifies signature, trusts the content (until expiry)
```

---

## The Hard Part: File Uploads

File uploads are inherently stateful. You can't fit a 10MB image inside a request token.

**The stateless approach:** Upload directly to object storage.

```
               ┌──────────────┐
User uploads ──▶  Generate    │── Returns signed upload URL
               │  Presigned   │
               │  URL         │
               └──────────────┘
                        │
                        ▼
               ┌──────────────┐
               │  Object      │── User uploads directly to S3/GCS
               │  Storage     │
               └──────────────┘
                        │
                        ▼
               ┌──────────────┐
               │  Database    │── Store just the URL/path
               └──────────────┘
```

Server generates a temporary signed URL, user uploads directly to cloud storage, server stores only the path. The server never touches the file bytes.

---

## Stateless ≠ Stateless Forever

Some things genuinely need state. Don't force statelessness where it doesn't belong:

| Keep Stateful | Make Stateless |
|---------------|----------------|
| WebSocket connections | REST API sessions |
| Long-running computations | Short request-response cycles |
| In-memory caches you can rebuild | User sessions |
| Real-time collaboration state | User preferences |

---

## Quick Wins Checklist

- [ ] Move sessions from local memory to Redis
- [ ] Switch from sticky sessions to stateless load balancing
- [ ] Replace session cookies with JWTs (or token-based auth)
- [ ] Offload file uploads to presigned S3 URLs
- [ ] Store user-specific data in the database, not server memory
- [ ] Make server shutdown safe—no in-flight state to lose
- [ ] Add health checks that don't depend on local state

---

## Takeaways

- **Stateless = any instance handles any request.** No sticky sessions required.
- **State goes to Redis, databases, or object storage**—never server memory.
- **Fault tolerance and scaling** become trivial when servers are interchangeable.
- **JWTs are your friend** for stateless auth, but watch the revocation problem.
- **Don't overdo it**—WebSockets and real-time features are inherently stateful. That's fine.
