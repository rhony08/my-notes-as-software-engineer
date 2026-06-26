# Event-Driven Architecture

Your REST API works fine — until it doesn't. A user uploads a CSV of 10,000 rows. The request handler processes every single row before sending back a response. The client times out. The database connection pool fills up. Other users get 503s. All because one synchronous request blocked everything.

Event-driven architecture (EDA) is how you avoid this mess. Instead of doing everything inline, you publish events and let something else deal with them.

## The Core Idea

In a traditional request-response model, **everything happens in sequence**:

```
Client ──POST /orders──→ API ──save to DB──→ Email service ──→ Response
                        ↑________________________________________↓
                        Wait for everything
```

If email takes 3 seconds, the client waits 3 seconds. If the DB is slow, the client waits. You're coupling your API's latency to the slowest downstream system.

**Event-driven flips this:**

```
Client ──POST /orders──→ API ──→ [Event Bus] ──→ Email Service
                           │                      (handles async)
                           └── Response (200 OK, "order placed")
```

The API publishes an `OrderPlaced` event and returns immediately. Email service picks it up when it's ready. The client gets a fast response, and you can scale each service independently.

## Building Blocks

### Events

An event is just a fact that happened. Past tense. Immutable.

```json
{
  "eventType": "order.placed.v1",
  "eventId": "evt_abc123",
  "timestamp": "2026-06-26T07:00:00Z",
  "producer": "order-service",
  "data": {
    "orderId": "ord_456",
    "customerId": "cus_789",
    "total": 49.99,
    "items": ["sku_001", "sku_002"]
  }
}
```

**Key rules:**
- One event type per schema (versioned)
- Events are never modified after creation
- Include enough context so consumers don't need to query you back

### Producers, Consumers, and the Bus

```
  ┌─────────┐   ┌──────────┐   ┌──────────┐
  │ Order   │──→│          │──→│ Email    │
  │ Service │   │  Event   │   │ Service  │
  └─────────┘   │   Bus    │   └──────────┘
  ┌─────────┐   │          │   ┌──────────┐
  │ Payment │──→│          │──→│ Audit    │
  │ Service │   └──────────┘   │ Service  │
  └─────────┘                  └──────────┘
```

**Producers** publish events. They don't know who listens.
**Consumers** subscribe to event types. They don't know who published.
**Event Bus** routes events from producers to consumers. This is your message broker (Kafka, RabbitMQ, SNS/SQS, etc.).

## Patterns in EDA

### 1. Event Notification

The simplest pattern. Something happened, and you're telling the world.

```javascript
// Producer (Order Service)
await eventBus.publish('order.placed.v1', {
  orderId: order.id,
  customerEmail: order.email,
  total: order.total
});

// Consumer (Notification Service)
eventBus.subscribe('order.placed.v1', async (event) => {
  await emailClient.send({
    to: event.data.customerEmail,
    subject: `Order #${event.data.orderId} confirmed!`,
    template: 'order-confirmation'
  });
});
```

⚠️ **Watch out:** Each consumer gets a copy of the event. If two services both email the customer for `order.placed`, that's a problem. Design your events for the intended audience.

### 2. Event-Carried State Transfer

This one's powerful. Instead of sending "hey, something changed", you send the actual data the consumer needs.

```javascript
// ❌ Consumer has to query back
eventBus.subscribe('user.updated.v1', async (event) => {
  const user = await userService.getUser(event.data.userId);
  // ^ Makes a synchronous call back to user-service
  // If user-service is down, this breaks
});

// ✅ Include the state the consumer needs
eventBus.subscribe('user.updated.v1', async (event) => {
  renderPage({
    name: event.data.name,
    email: event.data.email,
    avatar: event.data.avatarUrl
    // ^ Everything already in the event
  });
});
```

**Trade-off:** You're duplicating data across events. If the schema changes, old events still carry the old format. Version your events and plan for migrations.

### 3. Event Sourcing

Instead of storing current state, you store every event. The current state is derived by replaying events.

```
Events (ordered):                          State after replay:
1. AccountCreated(name: "Alice")           { name: "Alice" }
2. EmailChanged("alice@old.com")          { name: "Alice", email: "alice@old.com" }
3. EmailChanged("alice@new.com")          { name: "Alice", email: "alice@new.com" }
4. AccountClosed()                         (deleted)
```

This gives you a complete audit trail and lets you rebuild state at any point in time. But it also means your read model is eventually consistent, and schema changes are trickier.

## Choosing an Event Bus

| Broker | Strengths | When to use |
|--------|-----------|-------------|
| **Kafka** | High throughput, replayable logs, strong durability | Large-scale systems, event sourcing, stream processing |
| **RabbitMQ** | Smart routing, easy DLQ, good for complex workflows | Task queues, RPC-style, moderate scale |
| **SQS** | Fully managed, simple API, no servers | AWS shops, simple decoupling |
| **SNS + SQS** | Fan-out to multiple queues | Broadcast events, multi-consumer |
| **Redis Pub/Sub** | Low latency, simple | Real-time notifications, in-process messaging |

Warning about Redis Pub/Sub: **messages are lost if no consumer is listening**. It has no persistence. Don't use it for important events.

## Common Pitfalls

### Ordering Assumptions

> "Events will arrive in order."

**Nope.** With multiple producers or network retries, ordering is not guaranteed unless you explicitly partition (e.g., Kafka partitions by key). If `UserDeleted` arrives before `OrderLastPlaced`, your consumer might process them wrong.

❌ **Don't** assume implicit ordering.  
✅ **Do** design idempotent consumers that can handle out-of-order events.

### The Dual-Write Problem

You save to the database and publish an event. But what if the DB commits and the event publish fails?

```javascript
// ❌ Not atomic
await db.save(order);
await eventBus.publish('order.placed.v1', order); // If this fails, DB has the order but nobody knows
```

**Solutions:**
- **Outbox pattern** — save events to the same DB (outbox table), then a separate process publishes them
- **Change Data Capture** — read from DB's transaction log instead of manual publish
- **Transactional outbox** — write to DB and outbox in one transaction, publisher polls the outbox

### Over-Engineering

You don't need Kafka for a CRUD app with 50 users. Event-driven adds complexity—event schemas, monitoring, retries, dead letter queues, eventual consistency. Ask yourself:

- Do I actually have multiple services that need this data?
- Is synchronous processing actually causing problems?
- Am I ready to deal with eventual consistency in my UI?

If the answer is "not really", stick with synchronous calls. You can always extract events later.

## When It Shines

This is where EDA genuinely makes your life better:

- **Microservices** — services that shouldn't know about each other
- **Multi-system workflows** — order → payment → shipping → notification
- **Audit logging** — every meaningful action is an event you can replay
- **Real-time features** — dashboards, notifications, live updates
- **Cross-team communication** — Team A publishes events, Team B subscribes. No coordination needed.

---

**Actionable takeaways:**

- Events are immutable facts — name them in past tense and version them
- Use event-carried state transfer to reduce synchronous coupling
- Handle the dual-write problem with an outbox pattern before you go to production
- Never assume event ordering — design idempotent consumers
- Don't reach for Kafka until you actually need Kafka
