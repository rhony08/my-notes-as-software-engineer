# Event Sourcing and CQRS Patterns

Your database just corrupted some data. A customer's order shows shipped, but the shipping service has no record. The finance report doesn't match the actual transactions. When you try to debug, all you have is the current state — no history, no audit trail, no way to understand *how* you got here.

This is the problem Event Sourcing solves. Instead of storing current state, you store every change as an immutable event. Want to know the current state? Replay the events. Want to know what happened last Tuesday at 3pm? Replay up to that point. Need to debug a weird edge case? You have the complete history.

## The Core Idea: Events as Source of Truth

Traditional approach: store current state, overwrite on updates.

```
orders table:
| id | status   | total | customer_id |
|----|----------|-------|-------------|
| 42 | shipped  | 99.00 | 123         |
```

Event Sourcing: store every change that ever happened.

```
events table:
| id | aggregate_id | event_type          | data                    | timestamp  |
|----|--------------|---------------------|-------------------------|------------|
| 1  | order-42     | OrderCreated        | {total: 99, customer:123} | 10:00:00 |
| 2  | order-42     | PaymentReceived    | {method: "card"}       | 10:05:00   |
| 3  | order-42     | OrderShipped        | {tracking: "ABC123"}   | 10:30:00   |
```

To get current state, replay events 1, 2, 3 in order. The aggregate ID groups events for the same entity.

### Why This Matters

- **Audit trail:** Every change is recorded. No more "who changed this and why?"
- **Time travel:** Replay to any point in history. Debug production issues with actual data.
- **Undo:** Reverse events to rollback changes (compensating actions).
- **Separation of concerns:** Business logic operates on events, not database schema.

## CQRS: Command Query Responsibility Segregation

Event Sourcing pairs naturally with CQRS. The idea: separate your writes (commands) from your reads (queries).

```
┌─────────────┐     Command      ┌─────────────┐
│   Client    │ ──────────────► │ Write Model │
└─────────────┘                 │ (Events)    │
      ▲                         └──────┬──────┘
      │                                │
      │                          Publish Events
      │                                │
      │                                ▼
      │                         ┌─────────────┐
      │        Query            │  Read Model │
      └──────────────────────── │ (Projections)│
                                └─────────────┘
```

**Write model:** Handles commands, validates business rules, emits events. Optimized for writes, normalized.

**Read model:** Built from events, optimized for queries. Denormalized, can have multiple projections for different use cases.

```javascript
// ❌ Traditional: Same model for reads and writes
// Often leads to anemic domain models and complex queries
const order = await db.orders.findUnique({ where: { id } });

// ✅ CQRS: Separate models for different concerns
// Write side
await commandBus.execute(new ShipOrderCommand(orderId, tracking));

// Read side (could be from a completely different database)
const orderSummary = await readDb.orderSummaries.findById(orderId);
```

## When to Use Event Sourcing

It's not for everything. Event Sourcing adds complexity.

### Good Fit

- **Financial systems:** Need complete audit trail, regulatory compliance
- **Collaboration apps:** Multiple users, conflict resolution (think Google Docs)
- **Event-driven architectures:** Already using message queues, async processing
- **Complex business logic:** Rules depend on history, not just current state

### Bad Fit

- **Simple CRUD:** If you're just fetching and saving, don't over-engineer
- **High-frequency reads:** Replaying events is slower than a direct query (without snapshots)
- **GDPR/right-to-forget:** Events are immutable by design, deletion requires extra work

```javascript
// ❌ DON'T: "We'll use Event Sourcing for everything"
// A blog post with title and body? Just use a regular database.

// ✅ DO: Use when the history matters
// Payment processing? Every state change is legally significant.
```

## Practical Implementation

### Event Store

You need somewhere to store events. Options:

| Option | Pros | Cons |
|--------|------|------|
| PostgreSQL/MySQL | Familiar, ACID guarantees | Need to build event logic yourself |
| EventStoreDB | Purpose-built, subscriptions, projections | Another database to manage |
| Kafka | High throughput, distributed | No ordering guarantees per key without care |
| DynamoDB | Serverless, scalable | Need to design partition key carefully |

### Event Schema

```javascript
// Keep events small and focused
const event = {
  id: "evt_12345",
  aggregateId: "order-42",
  aggregateType: "Order",
  eventType: "OrderShipped",
  data: {
    trackingNumber: "ABC123",
    carrier: "FedEx",
    shippedAt: "2024-01-15T10:30:00Z"
  },
  metadata: {
    causationId: "cmd_999",  // Which command caused this
    correlationId: "req_777", // Track the whole operation
    userId: "user-5"
  },
  version: 3,  // For optimistic concurrency
  timestamp: "2024-01-15T10:30:00Z"
};
```

### Rebuilding State (Aggregates)

```javascript
class Order {
  static fromEvents(events) {
    const order = new Order();
    for (const event of events) {
      order.apply(event);
    }
    return order;
  }
  
  apply(event) {
    switch (event.eventType) {
      case 'OrderCreated':
        this.id = event.aggregateId;
        this.total = event.data.total;
        this.status = 'pending';
        break;
      case 'PaymentReceived':
        this.paidAt = event.timestamp;
        this.status = 'paid';
        break;
      case 'OrderShipped':
        this.trackingNumber = event.data.trackingNumber;
        this.status = 'shipped';
        break;
      // Don't forget to handle unknown events gracefully
    }
  }
  
  // Business logic lives in command handlers
  ship(trackingNumber) {
    if (this.status !== 'paid') {
      throw new Error('Cannot ship unpaid order');
    }
    return {
      eventType: 'OrderShipped',
      data: { trackingNumber }
    };
  }
}
```

## Snapshots: Solving the Replay Problem

Replaying 10,000 events to check an order's status is slow. Snapshots solve this.

```javascript
// Store periodic snapshots
{
  aggregateId: "order-42",
  version: 1000,
  state: {
    id: "order-42",
    status: "shipped",
    total: 99.00,
    // ... full state at version 1000
  },
  timestamp: "2024-01-15T12:00:00Z"
}

// To rebuild: load snapshot, replay only events after version 1000
const snapshot = await snapshotStore.load('order-42');
const events = await eventStore.getEvents('order-42', { afterVersion: snapshot.version });
const order = Order.fromSnapshot(snapshot, events);
```

## Projections: Building Read Models

Events are the source of truth, but they're not great for queries. Projections transform events into query-friendly structures.

```javascript
// Project events into a summary table for quick lookups
class OrderSummaryProjection {
  async handle(event) {
    switch (event.eventType) {
      case 'OrderCreated':
        await db.orderSummaries.insert({
          orderId: event.aggregateId,
          total: event.data.total,
          status: 'pending',
          createdAt: event.timestamp
        });
        break;
      case 'OrderShipped':
        await db.orderSummaries.update(
          { orderId: event.aggregateId },
          { status: 'shipped', shippedAt: event.timestamp }
        );
        break;
    }
  }
}

// Now queries are fast
const recentOrders = await db.orderSummaries
  .where({ status: 'shipped' })
  .orderBy('shippedAt', 'desc')
  .limit(20);
```

Multiple projections for different use cases:

```javascript
// Projection 1: For the orders list page
db.orderListItems.find({ customerId });

// Projection 2: For analytics
db.dailyOrderStats.find({ date });

// Projection 3: For the shipping dashboard
db.pendingShipments.find({ warehouseId });
```

## Handling Event Schema Changes

Events are stored forever. But schemas evolve. What happens when you need to change an event?

```javascript
// ❌ NEVER: Delete or modify existing events
// They might be used by other services, projections, or audits

// ✅ Option 1: Version your events
// Old: { type: 'OrderCreated', version: 1, data: { total: 99 } }
// New: { type: 'OrderCreated', version: 2, data: { total: 99, currency: 'USD' } }

// ✅ Option 2: Upcasters (transform old events on read)
function upcastOrderCreated(event) {
  if (event.version === 1) {
    return {
      ...event,
      version: 2,
      data: { ...event.data, currency: 'USD' }  // Default value
    };
  }
  return event;
}
```

## Eventual Consistency Trade-offs

CQRS introduces eventual consistency between write and read models.

```javascript
// Command succeeds
await commandBus.execute(new ShipOrderCommand('order-42'));

// But projection might not be updated yet!
const summary = await readDb.orderSummaries.findById('order-42');
console.log(summary.status); // Could still be 'paid', not 'shipped'
```

### Strategies to Handle This

1. **Accept it:** UI shows "updating..." or optimistic updates
2. **Wait for projection:** Subscribe to events, wait for confirmation
3. **Read from write model:** For critical data, query the aggregate directly

```javascript
// UI pattern: Optimistic update + eventual consistency
async function shipOrder(orderId) {
  // Optimistically update UI
  setOrderStatus(orderId, 'shipped');
  
  try {
    await commandBus.execute(new ShipOrderCommand(orderId));
  } catch (err) {
    // Revert on failure
    setOrderStatus(orderId, 'paid');
    showError(err.message);
  }
}
```

## Common Pitfalls

```javascript
// ❌ Event with calculated data
// What if the calculation changes? You can't replay correctly.
{
  type: 'OrderTotalCalculated',
  data: { total: 99.00, discount: 10.00 }  // Discount logic might change
}

// ✅ Store raw data, calculate on replay
{
  type: 'OrderItemsAdded',
  data: {
    items: [
      { productId: 'prod-1', quantity: 2, price: 49.50 }
    ],
    discountCode: 'SAVE10'
  }
}

// ❌ Events that reference external state
// What if the external data changes?
{
  type: 'CustomerEmailUpdated',
  data: { customerEmail: 'old@example.com' }  // Use ID instead
}

// ✅ Reference by ID
{
  type: 'CustomerEmailUpdated',
  data: { customerId: 'cust-123' }
}
```

## Quick Reference

| Concept | What It Does |
|---------|--------------|
| Event Store | Append-only log of all events |
| Aggregate | Entity that rebuilds state from events |
| Command | Intent to change state (can be rejected) |
| Event | Fact that something happened (immutable) |
| Projection | Read model built from events |
| Snapshot | Cached aggregate state at a point in time |
| Upcaster | Transforms old event versions to new |

## When You Actually Need This

- Your system has complex business rules that depend on history
- Audit requirements demand a complete trail
- You're building an event-driven architecture anyway
- Multiple teams need different views of the same data

If you're building a simple CRUD app with basic state, Event Sourcing is overkill. But when you need to answer "how did we get here?" and "what happened exactly?", it's invaluable.

---

**Key takeaways:**
- Store events, derive state — events are immutable, state is calculated
- Use CQRS to separate write concerns from read concerns
- Snapshots prevent replay performance issues for long-lived aggregates
- Projections give you query-friendly views from your event log
- Accept eventual consistency, design for it in the UI