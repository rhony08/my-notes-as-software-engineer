# Monolith vs Microservices: When to Split

"Start with a monolith" is basically gospel these days. But walk into any engineering team meeting and someone's itching to break it apart. The real question isn't *if* you should use microservices—it's *when* the monolith becomes the bottleneck, and whether splitting actually solves your problem.

## The Problem No One Talks About

Here's the thing: most startups don't fail because their monolith can't scale. They fail because they built a distributed system before they had distributed system problems. Microservices introduce real complexity—network latency, eventual consistency, debugging across services, deployment coordination—and they demand operational maturity most teams simply don't have yet.

So when **should** you actually make the jump?

## Why People Want to Split

Let's be honest about what drives the decision:

| Motivation | Real Problem? | Better Solution? |
|------------|--------------|------------------|
| "The codebase is too big" | Yes | Modular monolith first |
| "We need independent deployments" | Yes | Microservices might help |
| "It's slow in production" | No | Optimize the monolith first |
| "Everyone says microservices are better" | No | Ugly reason to refactor |
| "Different teams own different features" | Yes | Microservices make sense |

The last one is the **real** reason. When your team grows beyond 2-3 pizza teams and coordination overhead starts eating your velocity, that's when microservices start looking attractive.

## Monolith: Not the Villain It's Made Out to Be

A well-structured monolith beats poorly designed microservices every single time.

```
// A monolith doesn't mean spaghetti code.
// A modular monolith is just a monolith with clear boundaries.

// ❌ Spaghetti monolith
class OrderService {
    processOrder(orderId) {
        // 300 lines that touch payments, inventory, shipping, emails...
        // Everything mixed together with no boundaries
    }
}

// ✅ Modular monolith
// Separate modules with explicit interfaces
class OrderModule {
    constructor(paymentGateway, inventoryRepo, emailService) {
        this.payment = paymentGateway;
        this.inventory = inventoryRepo;
        this.email = emailService;
    }

    async processOrder(orderId) {
        const order = await this.orderRepo.findById(orderId);
        
        await this.payment.charge(order.total);
        await this.inventory.reserve(order.items);
        await this.email.sendConfirmation(order);
        
        return { status: 'completed', orderId };
    }
}
```

The modular monolith gives you most of the code organization benefits of microservices **without** the distributed system headaches. Your modules are just classes or packages with well-defined interfaces—nothing more.

## The Real Cost of Splitting

Microservices sound elegant in architecture diagrams. In practice:

- **Debugging** goes from "check this one service's logs" to "trace a request across 6 services"
- **Testing** goes from "one deployment" to "coordinate 6 service versions"
- **Latency** goes from "in-process function call" to "network round trip"
- **Consistency** goes from "ACID transactions" to "eventual consistency and sagas"
- **Deployments** go from "one CI/CD pipeline" to "6 CI/CD pipelines, versioning, and compatibility"

```
// ❌ BEFORE: Simple transaction in a monolith
await db.transaction(async (tx) => {
    await tx.createOrder(order);
    await tx.deductInventory(items);
    await tx.chargePayment(amount);
    // If anything fails, everything rolls back. Simple.
});

// ✅ AFTER: Same flow across microservices
await orderService.createOrder(order);
await inventoryService.deductItems(items);  // What if this fails?
await paymentService.charge(amount);        // What if the order creation was actually fine?

// Now you need compensating transactions, sagas, dead letter queues...
// Suddenly "simple" isn't so simple anymore.
```

## When the Monolith Actually Becomes a Problem

You'll feel these pain points long before you need microservices:

**1. Deployment coupling.** One team wants to ship a new checkout flow. Another team is fixing a shipping bug. Both touch the same codebase. Release gets blocked because shipping bug fix needs to go out, but checkout changes aren't ready. **This** is the microservices killer feature—independent deployability.

**2. Team ownership boundaries blur.** Team A owns orders. Team B owns billing. But in the monolith, Team A's change breaks billing. Team B gets paged. Nobody's happy. Clear service boundaries force ownership.

**3. Different scalability requirements.** Billing calls happen once per transaction. Search queries happen 100x more. You want to scale search independently without deploying the entire app. Microservices let you do that.

**4. Technology divergence.** Sometimes you genuinely need a different stack for a specific problem—a Python ML model, a Go websocket server, a Redis-backed rate limiter. A monolith forces one language, one runtime.

## The Conways Law Trap

Here's something most blog posts skip: your service boundaries will mirror your team structure. This is Conway's Law, and it's not optional.

```
// Teams shape services, services shape teams
// If your organization is split by function (frontend, backend, DBAs),
//   → you'll build services that match those functions
//
// If your organization is split by domain (orders, payments, users),
//   → you'll build domain-aligned services
```

This means: before you split your monolith, figure out your team structure first. Let team topology drive service boundaries, not the other way around.

## When to Stay Monolithic

Real talk: you probably don't need microservices if:

- Your team is under 15 people
- You can run your entire app on a single beefy server
- A single deploy takes < 15 minutes
- You don't have zero-downtime deployment requirements
- Your data is small enough that a single database handles it

These aren't "beginner stages." Stripe, GitHub, and Shopify ran monoliths for years with huge teams and massive traffic.

## When Microservices Start Making Sense

Consider splitting when:

- Multiple teams can't ship independently due to code coupling
- Different services need different scaling (read-heavy vs write-heavy vs CPU-bound)
- A single database is becoming a bottleneck
- You've already built a modular monolith and the boundaries are clear
- Your operational maturity supports it (monitoring, tracing, CI/CD, incident response)

## A Practical Decision Framework

Here's how I think about it:

1. **Start monolithic.** Build it right—modular, clean interfaces, tested.
2. **When team coordination hurts more than operational complexity**, consider extracting your first service.
3. **Extract one service at a time.** Don't "big bang" a rewrite. The Strangler Fig pattern exists for a reason.
4. **Live with the hybrid for a while.** You don't need to be all-or-nothing. A monolith with one extracted service is perfectly fine.
5. **Repeat only when it hurts.** Don't extract preemptively. Wait until you feel the pain.

## Key Takeaways

- **Monoliths aren't bad.** Bad monoliths are bad. A modular monolith buys you 80% of the benefits with 20% of the complexity.
- **Microservices solve organizational problems**, not technical ones. If your team doesn't have the org problem, you don't need the architectural solution.
- **Splitting introduces real operational cost.** Network latency, debugging complexity, data consistency—these aren't theoretical.
- **Start monolith, extract services.** Never rewrite. Never big bang. Extract boundaries you already understand.
- **Let team structure drive boundaries.** Conway's Law applies whether you acknowledge it or not.
