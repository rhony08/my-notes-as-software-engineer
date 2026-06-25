# Hexagonal Architecture: Isolating Your Business Logic

You're six months into a project, and the codebase is starting to fight back. Your business logic is tangled up with Express routes, Mongoose models, and Redis client calls. You want to swap PostgreSQL for MySQL? Good luck — it's woven into every layer. Need to test your core domain logic? You'll have to spin up a database and mock HTTP servers first.

This is the problem Hexagonal Architecture solves: **keep your business logic pure and independent from the outside world**.

---

## The Core Idea: Your App Is a Hexagon

Alistair Cockburn came up with this back in 2005. The name "hexagonal" isn't about literal hexagons — it's about having multiple connection points (ports) on the edges. Think of it like a pluggable architecture:

```
                  ┌──────────────────────┐
                  │    Adapters (driven)  │
                  │  (DB, Queue, Email)   │
                  └──────────┬───────────┘
                             │
  ┌──────────────┐  ┌───────┴────────┐  ┌──────────────┐
  │   Adapters   │  │     Ports      │  │   Adapters   │
  │(HTTP, CLI,   │◄─┤  (Interfaces)  ├─►│  (Database,  │
  │  GraphQL)    │  │                │  │   Message)   │
  └──────────────┘  └───────┬────────┘  └──────────────┘
                             │
                  ┌──────────┴───────────┐
                  │   Domain / Core      │
                  │  (Pure Business Logic)│
                  └──────────────────────┘
```

The whole point: **your domain core doesn't know about Express, PostgreSQL, or Redis**. It only knows about *ports* (interfaces). Adapters on the outside implement those ports.

This sounds abstract, so let's make it concrete.

---

## Before Hexagonal Architecture (The Tangled Mess)

Let's say you're building a payment processing service. Here's the "just get it done" approach:

```typescript
// ❌ TIGHTLY COUPLED - business logic mixed with infrastructure

import { db } from '../database/mysql';
import { sendEmail } from '../email/ses';
import { stripe } from '../payment/stripe';

export async function processOrder(orderId: string, userId: string) {
  const order = await db.query('SELECT * FROM orders WHERE id = ?', [orderId]);
  
  if (!order) throw new Error('Order not found');

  const payment = await stripe.charges.create({
    amount: order.total,
    currency: 'usd',
    customer: userId,
  });

  await db.query('UPDATE orders SET status = ? WHERE id = ?', ['paid', orderId]);

  await sendEmail(userId, 'Your payment was successful!');

  return { success: true, paymentId: payment.id };
}
```

This works. But:

- **Can't unit test without** a real MySQL instance, Stripe API keys, and SES credentials
- **Can't swap Stripe for PayPal** without rewriting this function and all its tests
- **Can't understand the business logic** just by reading the code — it's buried in SQL and API calls
- **Database changes** (like moving to Postgres) mean touching every function

---

## Hexagonal Architecture to the Rescue

The fix is to **invert the dependencies**. Your domain defines *what it needs* (ports), and infrastructure provides *how it's done* (adapters).

### Step 1: Define the Ports (Interfaces)

```typescript
// domain/ports/repositories.ts - The "what" not the "how"
export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  updateStatus(id: string, status: OrderStatus): Promise<void>;
}

export interface PaymentGateway {
  charge(amount: number, customerId: string): Promise<PaymentResult>;
}

export interface NotificationService {
  sendPaymentConfirmation(userId: string, amount: number): Promise<void>;
}
```

Notice something: **no imports from external libraries**. No `import { stripe }`. No database connection. Just pure TypeScript interfaces. This is your domain speaking.

### Step 2: Write Business Logic Against Ports

```typescript
// domain/use-cases/process-order.ts - Pure business logic
export class ProcessOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly paymentGateway: PaymentGateway,
    private readonly notificationService: NotificationService,
  ) {}

  async execute(orderId: string, userId: string): Promise<ProcessOrderResult> {
    const order = await this.orderRepo.findById(orderId);
    
    if (!order) {
      // Business decision: throw domain error, not HTTP error
      throw new OrderNotFoundError(orderId);
    }

    // Core business rule: check if order is already paid
    if (order.status === 'paid') {
      return { alreadyProcessed: true };
    }

    const payment = await this.paymentGateway.charge(order.total, userId);

    await this.orderRepo.updateStatus(orderId, 'paid');
    
    await this.notificationService.sendPaymentConfirmation(userId, order.total);

    return { success: true, paymentId: payment.id };
  }
}
```

This is **pure business logic**. No imports from outside the domain layer. You can read this and understand the order processing flow without knowing anything about MySQL or Stripe.

### Step 3: Build Adapters That Implement the Ports

```typescript
// infrastructure/repositories/mysql-order-repository.ts - Database adapter
import { OrderRepository } from '../../domain/ports/repositories';
import { db } from '../database/mysql';

export class MySqlOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order | null> {
    const row = await db.query('SELECT * FROM orders WHERE id = ?', [id]);
    return row ? OrderMapper.toDomain(row) : null;
  }

  async updateStatus(id: string, status: OrderStatus): Promise<void> {
    await db.query('UPDATE orders SET status = ? WHERE id = ?', [status, id]);
  }
}
```

```typescript
// infrastructure/payment/stripe-payment-gateway.ts - Payment adapter
import { PaymentGateway } from '../../domain/ports/repositories';
import { stripe } from './client';

export class StripePaymentGateway implements PaymentGateway {
  async charge(amount: number, customerId: string): Promise<PaymentResult> {
    const charge = await stripe.charges.create({
      amount: Math.round(amount * 100), // cents
      currency: 'usd',
      customer: customerId,
    });
    
    return { id: charge.id, status: charge.status };
  }
}
```

```typescript
// infrastructure/email/ses-notification-service.ts - Notification adapter
import { NotificationService } from '../../domain/ports/repositories';
import { sesClient } from './client';

export class SesNotificationService implements NotificationService {
  async sendPaymentConfirmation(userId: string, amount: number): Promise<void> {
    await sesClient.send({
      to: userId,
      subject: 'Payment confirmed',
      body: `Your payment of $${amount} was successful.`,
    });
  }
}
```

### Step 4: Wire Everything Together (Composition Root)

```typescript
// main.ts - Composition Root (only place where concrete classes meet)
const orderRepo = new MySqlOrderRepository();
const paymentGateway = new StripePaymentGateway();
const notifier = new SesNotificationService();

const processOrder = new ProcessOrderUseCase(orderRepo, paymentGateway, notifier);

// Now wire to HTTP adapter
app.post('/orders/:id/process', async (req, res) => {
  try {
    const result = await processOrder.execute(req.params.id, req.user.id);
    res.json(result);
  } catch (error) {
    if (error instanceof OrderNotFoundError) {
      res.status(404).json({ error: 'Order not found' });
      return;
    }
    res.status(500).json({ error: 'Internal error' });
  }
});
```

---

## Why This Changes Everything

### Testing Becomes Trivial

```typescript
// tests/unit/process-order.test.ts - No infrastructure needed!
import { ProcessOrderUseCase } from '../../domain/use-cases/process-order';

describe('ProcessOrderUseCase', () => {
  it('processes a valid order', async () => {
    // Arrange: use in-memory mocks, no real DB or Stripe
    const orderRepo = new InMemoryOrderRepository();
    const paymentGateway = new FakePaymentGateway();
    const notifier = new FakeNotificationService();

    const useCase = new ProcessOrderUseCase(orderRepo, paymentGateway, notifier);
    
    // Act
    const result = await useCase.execute('order-123', 'user-456');

    // Assert
    expect(result.success).toBe(true);
    expect(paymentGateway.lastCharge?.amount).toBe(99.99);
    expect(notifier.sentTo).toContain('user-456');
  });
});
```

No database needed. No HTTP server. No Stripe mocks that require network access. Pure in-memory tests that run in milliseconds.

### Swapping Infrastructure Is a Day Job

Want to switch from MySQL to PostgreSQL? You write one new adapter:

```typescript
// infrastructure/repositories/pg-order-repository.ts
export class PostgresOrderRepository implements OrderRepository {
  // Implement the same interface with PG client
}
```

Register it in the composition root. Done. **The domain layer never changed**.

Want to move from Stripe to PayPal? Same thing — a new adapter, one line change in `main.ts`.

### Adapters Are Inbound and Outbound

| Type | Direction | Examples |
|------|-----------|---------|
| **Primary (Driving)** | Outside → Inside | HTTP controllers, CLI commands, GraphQL resolvers, message consumers |
| **Secondary (Driven)** | Inside → Outside | Database repos, email senders, payment gateways, queues |

Primary adapters call *into* the domain. Secondary adapters are called *by* the domain. Both implement ports.

---

## The Trade-Offs Nobody Talks About

Let's be real: Hexagonal Architecture isn't free.

**✅ What you gain:**
- Domain logic is independently testable and understandable
- Swapping infrastructure is genuinely low-risk
- Framework changes don't cascade through your codebase
- New team members can understand business rules without reading ORM docs

**❌ What it costs:**
- **More files.** Every port needs at least one interface and one adapter. A simple CRUD endpoint balloons from 1 file to 5+
- **Indirection.** Following the code path means jumping between interfaces, adapters, and the domain layer. IDE "go to implementation" becomes your best friend
- **Over-engineering risk.** For a simple CRUD app with no plans to swap databases, this adds complexity without payoff
- **Learning curve.** New devs used to "grab Express req, call Mongoose, return response" will find this confusing at first

### When to Actually Use This

| Scenario | Hexagonal Architecture? |
|----------|----------------------|
| Building a core domain that will last 5+ years | ✅ Yes, absolutely |
| Microservice with clear business rules | ✅ Yes |
| Simple CRUD wrapper over a database | ❌ Overkill |
| Rapid prototype / MVP | ❌ Wait until you know the domain |
| Library or SDK | ✅ Yes, ports are perfect here |

---

## Real Talk: Hexagonal vs. Clean Architecture vs. Onion

You'll hear these terms thrown around interchangeably. They're all variations of the same principle: **dependency inversion**. The differences are mostly about naming and diagram shape:

- **Hexagonal** (Cockburn, 2005): Multiple ports, symmetrical left/right adapters
- **Onion** (Palermo, 2008): Concentric circles, domain in the center
- **Clean** (Martin, 2012): Similar to Onion, with stricter rules about dependency direction

Pick one, apply it consistently, and don't get hung up on the diagram. The real win is the same: **dependencies point inward**.

---

## Quick Start Checklist

If you're adopting Hexagonal Architecture today:

1. **Start with one use case.** Don't refactor everything at once. Pick `ProcessOrder` or `CreateInvoice` — a single flow with clear business rules
2. **Define the port you need.** What does your domain actually require from the outside? Not what your database provides — what your *domain* needs
3. **Write the domain logic first.** Before any adapter, get the business rules clear in pure code
4. **Mock the adapters in tests.** Verify your domain logic with in-memory implementations
5. **Extract more ports as needed.** Routes, other databases, message queues — compose them later

The biggest mistake is trying to design everything upfront. Hexagonal Architecture emerges as you notice pain points. Start small, extract ports when swapping something actually hurts.