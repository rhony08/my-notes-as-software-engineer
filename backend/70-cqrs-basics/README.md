# CQRS: Command Query Responsibility Segregation

You've got a CRUD API. It's clean. Users create orders, admins query them. Then the business team wants a dashboard with 12 aggregations. And analytics. And a report that shows "orders per user per region per month." Pretty soon, that simple `GET /orders` endpoint is crunching 50 million rows across 7 joins — and it's taking 30 seconds.

You need to make it fast. But the same model you use for **writing** data is the one you're using for **reading** it. That's the problem.

CQRS says: **Use a different model for reads than writes.** It sounds obvious when you say it, but most of us don't do it.

## The Problem with One Model

Your typical repository looks something like this:

```typescript
// One model to rule them all
class OrderRepository {
  async create(order: Order): Promise<void> { /* insert into DB */ }
  async getById(id: string): Promise<Order> { /* simple fetch */ }
  async getDashboardReport(region: string): Promise<DashboardReport> { /* nightmare */ }
}
```

This works fine when reads and writes care about the same shape of data. But that rarely holds at scale.

**The tension:**
- Writes need **consistency, validation, and normalization**
- Reads need **speed, denormalization, and aggregation**

Trying to satisfy both with one model means neither is optimized. Your writes carry the weight of read-friendly structures, and your reads fight through write-friendly constraints.

```typescript
// ❌ Same model, different needs
class User {
  async createUser(data: UserData): Promise<void> {
    // Validates email, checks uniqueness, encrypts password, fires events...
    // This is write-optimized
  }

  async getActiveUsersReport(): Promise<Report[]> {
    // JOINs across 8 tables, aggregates millions of rows
    // Why does my write model care about report queries?
  }
}

// ✅ CQRS separates them
class CreateUserCommand {
  async handle(data: UserData): Promise<void> {
    // Pure write: validate, persist, emit event
  }
}

class UserReportQuery {
  async handle(filter: ReportFilter): Promise<Report[]> {
    // Pure read: pre-joined, pre-aggregated, denormalized
  }
}
```

## How CQRS Works

At its simplest, CQRS splits your system into two sides:

```
                 ┌─────────────────┐
                 │   Command Side  │
                 │  (Writes/Write  │
                 │   Model)        │
                 └────────┬────────┘
                          │
              (data flows one way)
                          │
                          ▼
                 ┌─────────────────┐
                 │   Query Side    │
                 │  (Reads/Read    │
                 │   Model)        │
                 └─────────────────┘
```

**Commands:**
- Named after intent: `PlaceOrder`, `UpdateInvoice`, `CancelSubscription`
- Validate, mutate state, emit events
- Should return **no data** (or just a success/fail status) — if you return data, listeners get confused
- Run exactly once (idempotency matters)

**Queries:**
- Named after what's being fetched: `GetOrderHistory`, `SearchProducts`
- Never mutate state
- Return data however it's most convenient — pre-joined, denormalized, cached

```typescript
// Command — mutations only, no data returned
class PlaceOrderCommand {
  constructor(
    private orderRepo: OrderWriteRepository,
    private eventBus: EventBus
  ) {}

  async handle(orderId: string, items: CartItem[]): Promise<void> {
    const order = Order.create(orderId, items);
    await this.orderRepo.save(order);
    await this.eventBus.publish(new OrderPlacedEvent(orderId, items));
    // No return value — just success/error
  }
}

// Query — reads only, no mutations
class GetOrderSummaryQuery {
  constructor(private summaryRepo: OrderReadRepository) {}

  async handle(userId: string): Promise<OrderSummary[]> {
    return this.summaryRepo.findByUser(userId);
    // Returns pre-built denormalized projections
  }
}
```

## Separate Databases? Sometimes.

Here's where CQRS gets controversial. Some folks take it all the way to separate databases — a write-optimized DB for commands and a read-optimized DB for queries.

```
                  ┌─────────────┐
      Command ──→ │ Write Store │ ──┐
                  └─────────────┘   │
                                    │ sync mechanism
                                    │ (events, CDC, batch)
                  ┌─────────────┐   │
      Query  ──→  │  Read Store │ ◄─┘
                  └─────────────┘
```

| Approach | How sync works | When it makes sense |
|----------|---------------|---------------------|
| **Same DB, different models** | Views, materialized views, query-specific tables | Starting out, modest read complexity |
| **Same DB type, separate instances** | Write → event → read replica with projection | Medium scale, read-heavy workloads |
| **Different DB types** | Write to PostgreSQL → CDC → Elasticsearch | Full-text search, analytics, reporting |
| **Event-sourced** | Store events as source of truth, project to read models | Audit trails, complex business logic |

**The reality check:** Separate databases add operational complexity. You're now managing data sync, eventual consistency, and two sets of migrations. Start with different models in the same database. Only split the database when you have a concrete reason.

### Keeping Data in Sync

The write side publishes events. The read side subscribes and updates its projections.

```typescript
// Read model projection — updates when write side changes
class OrderProjection {
  constructor(private readRepo: OrderReadRepository) {}

  async onOrderPlaced(event: OrderPlacedEvent): Promise<void> {
    // Build denormalized read model
    const summary = {
      orderId: event.orderId,
      items: event.items,
      total: event.items.reduce((sum, i) => sum + i.price, 0),
      itemCount: event.items.length,
      createdAt: new Date(),
      // Pre-join additional data
      status: 'pending',
    };

    await this.readRepo.save(summary);
  }

  async onOrderShipped(event: OrderShippedEvent): Promise<void> {
    await this.readRepo.update(event.orderId, { status: 'shipped' });
  }
}
```

## What CQRS Unlocks

### 1. Different Read Models for Different Consumers

Your admin dashboard and your customer portal have completely different data needs. With CQRS, they get their own read models:

```typescript
// Admin read model — deep, aggregated
interface AdminOrderReadModel {
  orderId: string;
  customer: { id: string; email: string; lifetimeValue: number };
  items: { sku: string; name: string; qty: number; cost: number; margin: number }[];
  fulfillmentStatus: 'pending' | 'packed' | 'shipped' | 'delivered';
  paymentStatus: 'pending' | 'paid' | 'refunded';
  fraudScore: number;
}

// Customer read model — minimal, safe
interface CustomerOrderReadModel {
  orderId: string;
  status: string;
  estimatedDelivery: Date;
  items: { name: string; qty: number }[];
  trackingUrl?: string;
}
```

No more shipping the entire user object when all you need is a name and an avatar.

### 2. Optimized Query Performance

You can build your read tables however you want:

```sql
-- Write side: normalized, third normal form
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  created_at TIMESTAMP NOT NULL,
  status VARCHAR(20) NOT NULL
);

CREATE TABLE order_items (
  id UUID PRIMARY KEY,
  order_id UUID REFERENCES orders(id),
  product_id UUID NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL
);

-- Read side: denormalized for fast queries
CREATE TABLE order_summaries (
  order_id UUID PRIMARY KEY,
  user_name VARCHAR(100),
  item_count INT,
  total DECIMAL(10,2),
  first_item_name VARCHAR(200),
  status VARCHAR(20),
  created_at TIMESTAMP
);
```

A 7-table JOIN becomes a single table scan. That's the whole point.

### 3. Independent Scaling

Write heavy? Scale your command handlers. Read heavy? Spin up more read replicas. They don't interfere with each other because they're separate code paths with separate resources.

```
Traffic spike on the dashboard → scale read model horizontally
Black Friday order flood → command handlers stay fast because reads don't slow them down
```

## When CQRS Is Overkill

Let's be honest — you probably don't need CQRS for most apps.

| You don't need it if... | You might need it if... |
|-------------------------|------------------------|
| Simple CRUD with basic queries | Complex read models vs write models |
| < 100K users | Multiple data consumers with different needs |
| Single database is fine | Different teams own different read models |
| Real-time consistency matters | Eventual consistency is acceptable |
| 90% of queries are simple key lookups | You need full-text search + OLTP + analytics |

**The trade-off:** You trade simplicity for scalability. CQRS adds complexity — separate models, eventual consistency, extra code to maintain. Don't pay that cost if you don't have the problem.

## Common Implementation Patterns

### Separation Within the Same Service

Start simple: different classes/modules for commands and queries, same database.

```
src/
├── commands/
│   ├── place-order.handler.ts
│   └── cancel-order.handler.ts
├── queries/
│   ├── get-order-summary.handler.ts
│   └── search-orders.handler.ts
├── models/
│   ├── order-write.model.ts    # Normalized write model
│   └── order-read.model.ts     # Denormalized read model
└── projections/
    └── order.projection.ts     # Keeps read model in sync
```

### Separate Services

When the complexity justifies it, commands and queries become separate services or even separate teams.

```
orders-api/
├── commands-service/     # Accepts commands, publishes events
├── queries-service/      # Serves read models, subscribes to events
└── shared/               # Event schemas, types
```

## Practical Example: Order Dashboard

Here's a concrete scenario. You've got an e-commerce system. The `OrderService` handles everything — creating orders and serving dashboard queries. The dashboard query is killing performance:

```typescript
// ❌ Bloated single service
class OrderService {
  async placeOrder(cart: Cart): Promise<void> { /* ... */ }

  async getDashboardData(filters: DashboardFilters): Promise<DashboardDTO> {
    const orders = await this.db.query(`
      SELECT o.*, u.name, u.email, u.lifetime_orders,
             COUNT(oi.id) as item_count,
             SUM(oi.unit_price * oi.quantity) as total,
             AVG(oi.unit_price) as avg_item_price
      FROM orders o
      JOIN users u ON o.user_id = u.id
      JOIN order_items oi ON o.id = oi.order_id
      WHERE o.created_at BETWEEN $1 AND $2
      GROUP BY o.id, u.name, u.email, u.lifetime_orders
      ORDER BY o.created_at DESC
      LIMIT 100
    `, [filters.startDate, filters.endDate]);

    return DashboardDTO.fromRows(rows);
  }
}
```

With CQRS, you build a dedicated read model that's pre-aggregated:

```typescript
// ✅ CQRS — read model syncs from events
class DashboardReadRepository {
  async getDashboard(filters: Filters): Promise<DashboardDTO> {
    // Single table, no JOINs, pre-computed
    const rows = await this.db.query(`
      SELECT * FROM dashboard_order_summary
      WHERE created_at BETWEEN $1 AND $2
      ORDER BY created_at DESC
      LIMIT 100
    `, [filters.startDate, filters.endDate]);

    return DashboardDTO.fromRows(rows);
  }
}
```

The query goes from 7 joins to 1. Response time drops from 4 seconds to 50ms. All because you separated the concerns.

## Things That'll Bite You

### Eventual Consistency

The read model is always behind the write model. By milliseconds in most cases, but during failures it could be minutes.

```typescript
// User places an order
await commandBus.send(new PlaceOrderCommand(cart));

// User immediately refreshes the dashboard
const dashboard = await queryBus.send(new GetDashboardQuery(/* ... */));

// Order might not be in the read model yet!
// Dashboard shows old data for a brief moment
```

**How to handle it:**
- Accept it (most users don't notice sub-second delays)
- Show a "just placed" state from the write model for recently modified data
- Use optimistic UI that shows the expected state immediately

### Eventual consistency is fine for dashboards. It's terrible for banking.

If a user transfers money and the balance doesn't update — that's a bug. CQRS isn't the right pattern for systems where read-after-write consistency is critical.

### Adding CQRS Later Is Painful

You can't just "bolt on" CQRS. It affects your entire data flow. If you're starting simple, keep your models clean (separation of concerns) but don't go full CQRS until you need it.

**A middle ground:** Separate your service methods conceptually before separating databases.

## Takeaways

- **CQRS separates read models from write models** — they don't have to look the same or live in the same place
- **Commands mutate state and return no data** — they're about intent, not queries
- **Queries fetch data and mutate nothing** — they're optimized for the consumer, not the writer
- **Separate models before separate databases** — start with different code paths, escalate to different DBs only when needed
- **Eventual consistency is a feature, not a bug** — but know where you can't tolerate it
- **Don't over-engineer it** — if a simple `GET /orders` serves your dashboard fine, you don't need CQRS
