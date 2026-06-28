# Saga Pattern for Distributed Transactions

You're placing an order. The system needs to:

1. Reserve inventory
2. Charge the payment
3. Send a confirmation email

In a monolith, this is a single database transaction. One commit handles it all. If the payment fails — roll back the inventory reservation, no harm done.

But in microservices, each of those steps belongs to a different service with its own database. There's no `BEGIN TRANSACTION` that spans services. No `ROLLBACK` when step 2 fails.

You've got a distributed transaction problem. And the classical ACID approach (two-phase commit) doesn't play well in distributed systems — it's slow, complex, and holds locks that kill scalability.

Enter the **Saga pattern**.

## What's a Saga?

A saga is a sequence of local transactions where each step publishes an event or message that triggers the next step. If a step fails, the saga runs *compensating transactions* to undo the previous steps.

```
Normal flow:
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Reserve  │ ──→ │  Charge  │ ──→ │  Send    │
│Inventory │     │ Payment  │     │  Email   │
└──────────┘     └──────────┘     └──────────┘

Failure flow:
┌──────────┐     ┌──────────┐     ✗
│ Reserve  │ ──→ │  Charge  │ ──→ Payment fails!
│Inventory │     │ Payment  │
└──────────┘     └────┬─────┘
                       │
                       ▼
┌──────────┐     ┌──────────┐
│  Release │ ←── │  Compensate  │
│Inventory │     │  (refund)    │
└──────────┘     └──────────┘
```

No global transaction. No locks held across services. Just local transactions with compensation logic.

## Choreography vs Orchestration

There are two ways to coordinate a saga, and they solve different problems.

### Choreography: Each Service Knows Its Next Step

Each service publishes events. Other services listen and react. There's no central coordinator.

```
Order Service ──→ "Order Created" event
                      │
                      ▼
            Inventory Service reserves stock
                      │
                      ▼
            "Stock Reserved" event ──→ Payment Service charges card
                                            │
                                            ▼
                                    "Payment Charged" event ──→ Notification Service sends email
```

```typescript
// Service just publishes events after its local transaction
class OrderService {
  async createOrder(cart: Cart): Promise<void> {
    const order = await this.orderRepo.save(Order.create(cart));

    // Publish event — don't care who listens
    await this.eventBus.publish(new OrderCreatedEvent(order.id, cart));
  }
}

class InventoryService {
  @Subscribe('OrderCreated')
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.inventoryRepo.reserve(event.orderId, event.cart.items);
      await this.eventBus.publish(new StockReservedEvent(event.orderId));
    } catch (err) {
      // Can't reserve — publish failure
      await this.eventBus.publish(new StockReservationFailedEvent(event.orderId));
    }
  }
}

class PaymentService {
  @Subscribe('StockReserved')
  async onStockReserved(event: StockReservedEvent): Promise<void> {
    try {
      await this.paymentRepo.charge(event.orderId, event.amount);
      await this.eventBus.publish(new PaymentChargedEvent(event.orderId));
    } catch (err) {
      await this.eventBus.publish(new PaymentFailedEvent(event.orderId));
      // Someone else will handle compensation
    }
  }
}
```

**The problem with choreography:** Business logic is scattered across services. To understand the full order flow, you have to read every event handler in every service. It looks elegant in diagrams but gets messy fast when you add more steps.

| Pros | Cons |
|------|------|
| Simple to implement each service | Flow is implicit — hard to see end-to-end |
| No single point of failure | Cyclic dependencies between services |
| Each service is independent | Hard to add new steps or change flow |
| Scales naturally | Debugging is a nightmare |

### Orchestration: One Coordinator Calls the Shots

A dedicated "saga orchestrator" tells each service what to do and when. It knows the full flow and handles the state machine.

```
                     ┌──────────────────┐
                     │  Order Saga       │
                     │  Orchestrator     │
                     └────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
       ┌──────────┐    ┌──────────┐    ┌──────────────┐
       │Inventory │    │ Payment  │    │Notification  │
       │ Service  │    │ Service  │    │   Service    │
       └──────────┘    └──────────┘    └──────────────┘
```

```typescript
// Orchestrator — knows the full choreography
class OrderSagaOrchestrator {
  async execute(orderId: string, cart: Cart): Promise<void> {
    // Step 1: Reserve inventory
    const inventoryResult = await this.inventoryClient.reserve(orderId, cart.items);

    if (!inventoryResult.success) {
      // Saga failed at step 1 — nothing to compensate
      await this.orderClient.markFailed(orderId, 'inventory_unavailable');
      return;
    }

    // Step 2: Charge payment
    const paymentResult = await this.paymentClient.charge(orderId, cart.total);

    if (!paymentResult.success) {
      // Step 2 failed — compensate step 1
      await this.inventoryClient.release(orderId);
      await this.orderClient.markFailed(orderId, 'payment_failed');
      return;
    }

    // Step 3: Send email
    const emailResult = await this.notificationClient.sendConfirmation(orderId, cart);

    if (!emailResult.success) {
      // Step 3 failed — compensate steps 1 and 2
      await this.inventoryClient.release(orderId);
      await this.paymentClient.refund(orderId);
      await this.orderClient.markFailed(orderId, 'notification_failed');
      return;
    }

    // All steps succeeded
    await this.orderClient.markCompleted(orderId);
  }
}
```

**The orchestrator acts as a state machine:**

```
          ┌──────────┐
          │ Pending  │
          └────┬─────┘
               │
          ┌────▼─────┐
          │ Reserving│──── failure ──→ ┌──────────┐
          │Inventory │                 │  Failed  │
          └────┬─────┘                 └──────────┘
               │ success
          ┌────▼─────┐
          │ Charging │──── failure ──→ ┌───────────┐
          │ Payment  │                 │Releasing  │
          └────┬─────┘                 │Inventory  │──→ ┌──────────┐
               │ success               └───────────┘    │  Failed  │
          ┌────▼──────┐                                 └──────────┘
          │  Sending  │──── failure ──→ ┌───────────┐
          │   Email   │                 │  Refund   │
          └────┬──────┘                 │  Payment  │
               │ success                └───────────┘
          ┌────▼──────┐                      │
          │Completed  │                   ┌──▼────┐
          └───────────┘                   │Failed │
                                          └───────┘
```

```typescript
// More robust orchestrator with state persistence
class OrderSagaOrchestrator {
  async execute(orderId: string, cart: Cart): Promise<void> {
    const saga = await this.sagaStore.create({
      orderId,
      status: 'pending',
      steps: []
    });

    try {
      // Step 1
      await saga.executeStep('reserve_inventory', () =>
        this.inventoryClient.reserve(orderId, cart.items)
      );

      // Step 2
      await saga.executeStep('charge_payment', () =>
        this.paymentClient.charge(orderId, cart.total)
      );

      // Step 3
      await saga.executeStep('send_email', () =>
        this.notificationClient.sendConfirmation(orderId, cart)
      );

      await saga.complete();
    } catch (err) {
      // Roll back in reverse order of completed steps
      await saga.compensate(async (step) => {
        switch (step.name) {
          case 'reserve_inventory':
            return this.inventoryClient.release(orderId);
          case 'charge_payment':
            return this.paymentClient.refund(orderId);
          case 'send_email':
            // Emails can't be unsent — idempotent skip
            return;
        }
      });

      await saga.markFailed(err.message);
    }
  }
}

// Saga state persisted to DB — survives crashes
interface SagaState {
  id: string;
  orderId: string;
  status: 'pending' | 'in_progress' | 'completed' | 'compensating' | 'failed';
  completedSteps: string[];
  compensationSteps: string[];
  createdAt: Date;
  updatedAt: Date;
}
```

| Pros | Cons |
|------|------|
| Clear, centralized flow | Single point of failure (the orchestrator) |
| Easy to add/modify steps | Orchestrator can become a god service |
| Simple error handling | More code to write upfront |
| State is visible and trackable | Adds latency (one more hop per step) |

### ❌ Choreography vs ✅ Orchestration — Which to Pick?

**Start with choreography when:**
- You have 2-3 services in the saga
- The flow rarely changes
- Each service can handle its own compensation

**Go with orchestration when:**
- You have 4+ services in the saga
- The flow needs to change without redeploying every service
- You need visibility into saga state (for debugging/auditing)
- Different teams own different services

**Reality check:** Most production sagas I've seen use orchestration. Choreography looks elegant in blog posts but becomes spaghetti in practice. The orchestrator overhead is worth it once you have more than a couple of steps.

## Two Saga Flavors

### 1. Choreography Saga (Event-Based)

We already covered this — services react to events. Compensation is handled by publishing "failure" events that trigger undo logic.

**Where it shines:** Simple flows with few services. Each service is truly independent.

### 2. Orchestration Saga (Command-Based)

The orchestrator sends explicit commands. Compensation is handled by sending "undo" commands.

**Where it shines:** Complex flows with multiple steps, branching logic, and strict ordering requirements.

---

```
// ❌ Choreography — implicit flow, hard to trace
OrderService.publish('OrderCreated')
  → InventoryService.subscribe('OrderCreated')
    → publishes 'StockReserved' or 'ReservationFailed'
      → PaymentService reacts...

// ✅ Orchestration — explicit flow, easy to trace
SagaOrchestrator.send('reserve_inventory')
  → wait for response
  → send('charge_payment')
  → wait for response
  → send('send_email')
  → complete
```

## Compensating Transactions Are Not Rollbacks

This is the most important thing to understand about sagas: **Compensating transactions are NOT database rollbacks. They're business-level reversals.**

A real rollback restores previous state exactly. A compensating transaction reverts the *business effect* — and it might not be symmetric.

```typescript
// Step 1: Reserve inventory — easy to compensate
const reserve = await inventoryClient.reserve(orderId, items);
// Compensate: release the stock
await inventoryClient.release(orderId);

// Step 2: Charge payment — compensation has side effects
const charge = await paymentClient.charge(orderId, 100.00);
// Compensate: refund the charge (costs merchant fee!)
await paymentClient.refund(orderId);
// ^^^ This costs money and might take 5-7 business days

// Step 3: Send confirmation email — can't compensate
const email = await emailClient.sendOrderConfirmation(orderId);
// Compensate: ??? You can't unsend an email
// Best you can do: send a follow-up "Order Cancelled" email
```

**Key insights about compensation:**

- Compensations are **not guaranteed to succeed** — what if the refund fails too?
- Compensations might have **different semantics** than the forward action
- Some things **can't be undone** (sending an email, printing a shipping label)
- Your saga needs to handle **partial failures** at the compensation level too

## Idempotency Is Non-Negotiable

Every step in a saga must be idempotent. Services crash. Networks timeout. Messages get delivered twice.

```typescript
// ❌ Not idempotent — charging twice
class PaymentService {
  async charge(orderId: string, amount: number): Promise<void> {
    await this.db.query(
      `INSERT INTO charges (order_id, amount) VALUES ($1, $2)`,
      [orderId, amount]
    );
  }
}

// ✅ Idempotent — ignores duplicates
class PaymentService {
  async charge(orderId: string, amount: number, idempotencyKey: string): Promise<void> {
    const existing = await this.db.query(
      `SELECT id FROM charges WHERE idempotency_key = $1`,
      [idempotencyKey]
    );

    if (existing.rows.length > 0) {
      return; // Already processed — skip
    }

    await this.db.query(
      `INSERT INTO charges (order_id, amount, idempotency_key) VALUES ($1, $2, $3)`,
      [orderId, amount, idempotencyKey]
    );
  }
}
```

## Handling Saga Failures (The Nuanced Part)

A saga can fail in three ways, and each requires different handling:

### 1. Forward Failure — Step N Fails Before Processing

No compensation needed — nothing happened yet. Just stop the saga.

### 2. Partial Failure — Step N Fails After Previous Steps Succeeded

Run compensating transactions for all completed steps. But:

```typescript
// What happens when compensation fails?
async compensateOrder(orderId: string): Promise<void> {
  const steps = await this.sagaStore.getCompletedSteps(orderId);

  for (const step of steps.reverse()) {
    try {
      await this.executeCompensation(step);
      await this.sagaStore.markCompensated(step.id);
    } catch (err) {
      // Compensation failed!
      // Log it, alert someone, and try again later
      await this.sagaStore.markCompensationPending(step.id);
      await this.failureQueue.enqueue({
        type: 'compensation_retry',
        stepId: step.id,
        orderId,
        attempts: 0,
        maxAttempts: 5
      });
      // Don't stop — try to compensate the rest
    }
  }
}
```

A saga that's stuck between "not fully succeeded" and "not fully compensated" is in a **saga recovery** state. You need a background worker that spots these and retries compensation.

### 3. Recovery Failure — Compensation Itself Fails

This is the nightmare scenario. You've charged a customer but can't refund them. You've shipped inventory but can't recall it. Now you need:

- **Automatic retry** with exponential backoff
- **Manual intervention** when retries are exhausted
- **Audit logs** that tell an operator exactly what happened

```
Saga failures often require human intervention.
Don't design a system where compensation failures are silent.
```

### Recovery Patterns

```typescript
// Retry with backoff
class RecoveryWorker {
  async processDeadSagas(): Promise<void> {
    const stuck = await this.sagaStore.findStuck();

    for (const saga of stuck) {
      console.warn(`Recovering saga ${saga.id}, step ${saga.currentStep}`);

      try {
        if (saga.status === 'compensation_pending') {
          await this.retryCompensation(saga);
        } else if (saga.status === 'forward_pending') {
          await this.retryForward(saga);
        }
      } catch (err) {
        console.error(`Recovery failed for saga ${saga.id}: ${err.message}`);
        // Notify on-call — this needs human eyes
        await this.alertService.pageteam(
          `Saga ${saga.id} unrecoverable after ${saga.retryCount} attempts`
        );
      }
    }
  }

  private async retryCompensation(saga: SagaState): Promise<void> {
    if (saga.retryCount >= 5) {
      await this.sagaStore.markUnrecoverable(saga.id);
      await this.alertService.pageteam(`Saga ${saga.id} requires manual resolution`);
      return;
    }

    await this.compensationWorker.execute(saga);
    await this.sagaStore.incrementRetry(saga.id);
  }
}
```

## Real-World Example: Booking System Saga

Let's make this concrete. You're building a travel booking system:

```
1. Book flight (Airline Service)
2. Book hotel (Hotel Service)
3. Reserve rental car (Car Service)
4. Charge customer (Payment Service)
5. Send confirmation (Notification Service)
```

```typescript
class BookingSagaOrchestrator {
  async bookTrip(userId: string, booking: TripBooking): Promise<void> {
    const sagaId = crypto.randomUUID();
    await this.sagaStore.init(sagaId, booking);

    try {
      // Step 1: Book flight — must succeed before hotel
      const flight = await this.flightClient.book(booking.flight);
      await this.sagaStore.recordStep(sagaId, 'flight_booked', flight);

      // Step 2: Book hotel
      const hotel = await this.hotelClient.book(booking.hotel);
      await this.sagaStore.recordStep(sagaId, 'hotel_booked', hotel);

      // Step 3: Reserve car (optional — skip if not requested)
      let car = null;
      if (booking.rentalCar) {
        car = await this.carClient.reserve(booking.rentalCar);
        await this.sagaStore.recordStep(sagaId, 'car_reserved', car);
      }

      // Step 4: Charge the card
      const total = flight.price + hotel.price + (car?.price ?? 0);
      await this.paymentClient.charge(userId, total, `booking_${sagaId}`);
      await this.sagaStore.recordStep(sagaId, 'payment_charged', { total });

      // Step 5: Send confirmation
      await this.notificationClient.sendItinerary(userId, { flight, hotel, car });
      await this.sagaStore.complete(sagaId);

    } catch (err) {
      await this.compensateBooking(sagaId, err.message);
    }
  }

  private async compensateBooking(sagaId: string, reason: string): Promise<void> {
    const steps = await this.sagaStore.getCompletedSteps(sagaId);

    // Compensate in reverse order
    for (const step of steps.reverse()) {
      try {
        switch (step.name) {
          case 'flight_booked':
            await this.flightClient.cancel(step.data.flight.id);
            break;
          case 'hotel_booked':
            await this.hotelClient.cancel(step.data.hotel.id);
            break;
          case 'car_reserved':
            await this.carClient.release(step.data.car.id);
            break;
          case 'payment_charged':
            await this.paymentClient.refund(userId, step.data.total);
            break;
        }
      } catch (compErr) {
        console.error(`Compensation failed for step ${step.name}:`, compErr);
        // Enqueue for retry
        await this.failureQueue.enqueue({
          type: 'compensate_saga_step',
          sagaId,
          stepName: step.name
        });
      }
    }

    await this.sagaStore.markFailed(sagaId, reason);
  }
}
```

### Compensating Transactions in This Example

| Step | Forward Action | Compensating Action | Notes |
|------|---------------|-------------------|-------|
| Book flight | Reserve flight seat | Cancel reservation | Usually free cancellation within 24 hours |
| Book hotel | Reserve hotel room | Cancel reservation | Check cancellation policy — might incur fee |
| Reserve car | Reserve rental car | Release reservation | Typically free cancellation |
| Charge payment | Capture $X on credit card | Issue refund | Refund takes 3-5 business days |
| Send email | Send itinerary | Send cancellation email | Can't unsend — just send a follow-up |

**Notice:** The compensation for "send email" isn't "unsend" — it's "send a cancellation email." This is what makes sagas different from database transactions. You're working with business operations, not data states.

## When Not to Use Sagas

| You don't need a saga if... | You might need a saga if... |
|----------------------------|---------------------------|
| All your data is in one database | Each service has its own database |
| You can use a regular DB transaction | The transaction spans multiple services |
| Eventual consistency is unacceptable | Milliseconds of delay are fine |
| You have 2 services with simple needs | 5+ services with complex orchestration |
| You can't implement compensation logic | Each step has a clear reversal operation |

**The honest truth:** Sagas make your system more resilient at the cost of complexity. The compensating transaction logic is rarely as clean as the forward logic. If you can keep your data in one database and use regular transactions, do that first. Sagas are a distributed systems tool, not a default choice.

## Takeaways

- **Sagas break distributed transactions into local transactions** — each step commits independently, and failures trigger compensation
- **Compensating transactions are business-level reversals, not database rollbacks** — they have different semantics, costs, and failure modes
- **Choreography works for simple flows** (2-3 services), but **orchestration scales better** for complex ones
- **Every saga step must be idempotent** — retries are guaranteed, deduplication is your responsibility
- **Saga recovery needs a background worker** — stuck sagas don't resolve themselves
- **Don't use sagas unless you have to** — one database with ACID transactions is simpler, safer, and faster
