# Distributed Systems: CAP Theorem and Trade-offs

Your database just went down. Users are screaming. You check your monitoring—oh wait, you had a replica. Except... the replica is behind by 30 seconds and your app is showing stale data. Do you serve the old data, or fail the request?

This is the kind of choice distributed systems force on you. And the CAP theorem? It's the framework that helps you reason through these decisions.

## What CAP Actually Means

CAP stands for:

- **Consistency** — Every read sees the most recent write (or an error)
- **Availability** — Every request gets a response (success or failure), no matter what
- **Partition Tolerance** — The system keeps working even when network failures split nodes apart

Here's the uncomfortable truth: **you can't have all three.** Not because you're bad at engineering, but because the math doesn't allow it.

### Why Partitions Are Non-Negotiable

In distributed systems, networks *will* fail. Cables get cut. Switches die. Cloud providers have outages. If you're distributing across data centers, partitions aren't a question of *if*—they're a question of *when*.

This means **P is mandatory**. You can't opt out of partition tolerance in a distributed system. The real choice is between C and A.

## The Real Trade-off: CP vs AP

When a partition happens, you have two bad options:

| Choice | What You Do | What Breaks |
|--------|-------------|-------------|
| **CP** (Consistency + Partition Tolerance) | Reject requests that can't reach a quorum | Users get errors, but data is never stale |
| **AP** (Availability + Partition Tolerance) | Serve whatever data you have | Users get responses, but data might be old |

There's no right answer. It depends on what hurts more for your use case.

### When to Choose CP

You need CP when stale data is dangerous:

- **Financial transactions** — You'd rather fail than process with wrong balances
- **Inventory systems** — Overselling inventory is worse than an error message
- **Authentication/authorization** — Revoked access should be immediate

```javascript
// ❌ AP approach for a bank - serving stale balance
async function getBalance(userId) {
  const localData = await localCache.get(userId);
  return localData?.balance ?? 0; // Could be 5 minutes old!
}

// ✅ CP approach - fail if you can't reach the source of truth
async function getBalance(userId) {
  const quorum = await database.getPrimaryConnection();
  if (!quorum) {
    throw new ServiceUnavailableError('Cannot verify current balance');
  }
  return quorum.query('SELECT balance FROM accounts WHERE user_id = ?', [userId]);
}
```

### When to Choose AP

You need AP when uptime matters more than perfect consistency:

- **Social feeds** — Seeing a post 30 seconds late isn't critical
- **Product recommendations** — Slightly outdated recs won't hurt anyone
- **Analytics dashboards** — "Real-time" can tolerate a few minutes of lag

```javascript
// ✅ AP approach for social feed - serve what you have
async function getFeed(userId) {
  // Try primary first, fall back to replica, then cache
  const feed = await primary.getFeed(userId)
    .catch(() => replica.getFeed(userId))
    .catch(() => cache.get(`feed:${userId}`));
  
  if (!feed) {
    // Still try to return something useful
    return getGenericFeed();
  }
  return feed;
}
```

## The "Wait, What About CA?" Trap

You'll sometimes hear people claim their system is CA. That's either:

1. **Not actually distributed** — Single-node databases (PostgreSQL on one server) can be CA because there's no network to partition
2. **Marketing speak** — They're ignoring the inevitability of network failures

If your data lives on more than one machine, you're choosing between CP and AP. Full stop.

## Real Systems and Their Choices

| System | Default | Why |
|--------|---------|-----|
| **PostgreSQL** (single node) | CA | No partition tolerance needed—everything's on one machine |
| **MongoDB** | CP | Primary handles writes; secondaries replicate. If primary is unreachable, you can't write until a new primary is elected |
| **Cassandra** | AP | Any node can accept writes. Reads might return stale data, but the system stays up |
| **Redis Sentinel** | CP | Prioritizes consistency—if master is unreachable, waits for failover before accepting writes |
| **DynamoDB** | AP (configurable) | Default is eventual consistency, but you can request strongly consistent reads (costs more, slower) |

## But Wait—Isn't Consistency a Spectrum?

The CAP theorem makes it sound binary: you're either consistent or you're not. Reality is messier.

Most systems don't live at the extremes. They offer **tunable consistency**:

```javascript
// Cassandra: choose consistency level per query
await client.execute(
  'SELECT * FROM users WHERE id = ?',
  [userId],
  { consistency: ConsistencyLevel.QUORUM } // Wait for majority of replicas
);

await client.execute(
  'SELECT * FROM product_recommendations WHERE id = ?',
  [productId],
  { consistency: ConsistencyLevel.ONE } // Accept first response, might be stale
);
```

```javascript
// DynamoDB: strongly consistent read costs more capacity units
const result = await dynamodb.get({
  TableName: 'Users',
  Key: { id: userId },
  ConsistentRead: true // Extra cost, but guaranteed fresh data
}).promise();
```

### Consistency Levels: A Quick Guide

| Level | What It Means | Latency | Use Case |
|-------|---------------|---------|----------|
| **ONE** | Wait for first replica to respond | Lowest | Non-critical reads, high throughput |
| **QUORUM** | Wait for majority of replicas | Medium | Balance of safety and speed |
| **ALL** | Wait for all replicas | Highest | Critical writes that can't be lost |

## What Breaks When You Choose Wrong

### Choosing AP When You Needed CP

```javascript
// E-commerce inventory with AP approach
async function reserveItem(itemId, quantity) {
  // Wrote to local replica, returned success immediately
  await localReplica.decrementStock(itemId, quantity);
  return { success: true }; // ✅ Responded fast
  
  // Problem: other replicas didn't know yet
  // Other users might also reserve the same item
  // Now you've oversold inventory
}
```

### Choosing CP When You Needed AP

```javascript
// Social media feed with CP approach
async function getFeed(userId) {
  // Must reach primary for guaranteed fresh data
  const feed = await primary.getFeed(userId);
  // ❌ If primary is unreachable, we fail
  // Users see "Feed unavailable" instead of slightly old posts
  // Meanwhile, competitors' apps still work
}
```

## Beyond CAP: Other Trade-offs to Know

CAP isn't the only game in town:

### PACELC

An extension of CAP that says: **If there's a Partition, choose Availability or Consistency. Else (normal operation), choose Latency or Consistency.**

In other words: even when everything's working, you're making trade-offs.

| System | Partition | Normal Operation |
|--------|-----------|------------------|
| MySQL (single node) | N/A | Latency ↔ Consistency |
| Cassandra | AP | Latency over Consistency (tunable) |
| Spanner | CP | Consistency over Latency |

### BASE vs ACID

Traditional databases follow **ACID** (Atomicity, Consistency, Isolation, Durability). Distributed systems often embrace **BASE**:

- **Basically Available** — System responds, but data might be stale
- **Soft state** — Data can change without new writes (replication lag)
- **Eventually consistent** — Given enough time without writes, all replicas converge

```javascript
// ACID: Transaction blocks until fully committed
await db.transaction(async (tx) => {
  await tx.execute('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
  await tx.execute('UPDATE accounts SET balance = balance + 100 WHERE id = 2');
  // Both succeed or both fail, no intermediate state visible
});

// BASE: Accept the write, replicate eventually
await cassandra.execute(
  'UPDATE accounts SET balance = balance - 100 WHERE id = 1'
);
// Other replicas might not see this for a few hundred milliseconds
// That's "eventually consistent"
```

## Practical Takeaways

1. **Accept that you're choosing between CP and AP** — There's no escaping this in distributed systems
2. **Make the choice per use case, not globally** — Your auth service might need CP while your feed service is fine with AP
3. **Question if you need distribution at all** — Single-node PostgreSQL handles a lot of traffic. Complexity has costs
4. **Test your partition behavior** — Use chaos engineering (Chaos Monkey, litmus tests) to see what actually happens
5. **Document your consistency guarantees** — Your team and future-you need to know what's promised

## When This Matters

You'll actually need to think about CAP when:

- **Moving from single database to replicas** — That async replication? It creates windows of inconsistency
- **Going multi-region** — Network latency between regions makes partitions more likely
- **Building financial or safety-critical systems** — Stale data can mean real-world harm
- **Designing APIs that clients depend on** — They need to know: "Will I always see my own writes?"

## Further Reading

- [Brewer's Conjecture](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf) — The original CAP paper
- [PACELC: A Generalization of CAP](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) — Why latency matters even without partitions
- [Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/) — Martin Kleppmann's excellent deep dive