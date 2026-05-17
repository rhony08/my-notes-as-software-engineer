# Backpressure: Handling Overload

Your system is humming along fine. Then a spike hits — Black Friday, a viral tweet, a misconfigured client making 10k requests per second. Your services start slowing down. Then they start dying. And because Service A calls Service B calls Service C, the whole chain collapses like dominoes.

This isn't a hypothetical. It's what happens when you don't think about **backpressure**.

Backpressure is a system's way of saying "I can't keep up." The question isn't whether your system will face overload — it's whether it'll fail gracefully or catastrophically.

## What Actually Breaks

Let's trace what happens under overload in a typical microservice chain:

```
Client → API Gateway → User Service → Order Service → Payment Service → Database
```

When Payment Service gets slow:
1. Order Service holds connections open waiting for responses
2. Connection pool saturates
3. New requests to Order Service queue up or get refused
4. API Gateway runs out of worker threads
5. Clients start seeing timeouts and 502s

**The real danger?** Cascading failure. A slow downstream service can take down the entire upstream chain. By the time you notice, everything's on fire.

## Four Ways to Handle Backpressure

There's no one-size-fits-all approach. Here are the main strategies, with their trade-offs:

### 1. Load Shedding (Drop Requests)

The simplest approach: when you're overwhelmed, start dropping requests.

```
// ❌ NO backpressure - keeps accepting until OOM
app.get('/api/orders', async (req, res) => {
  const orders = await orderService.getAll(); // hangs when overloaded
  res.json(orders);
});

// ✅ With load shedding
const activeRequests = 0;
const MAX_CONCURRENT = 100;

app.get('/api/orders', async (req, res) => {
  if (activeRequests >= MAX_CONCURRENT) {
    return res.status(503).json({
      error: 'Server busy',
      retryAfter: 5
    });
  }
  activeRequests++;
  try {
    const orders = await orderService.getAll();
    res.json(orders);
  } finally {
    activeRequests--;
  }
});
```

**Why this works:** You protect the service from total collapse. 503s are recoverable — crashes aren't.

**The cost:** You're throwing away valid work. Clients need to retry, which means you need proper `Retry-After` headers and exponential backoff on the client side.

### 2. Buffering (Queues)

Instead of dropping, you buffer. This is where message queues shine.

```
Client → API → Queue → Worker Pool → Database
```

```
// Producer - always fast, just pushes to queue
app.post('/api/orders', async (req, res) => {
  await queue.publish('orders', req.body);
  res.status(202).json({ status: 'queued' });
});

// Consumer - processes at its own pace
queue.consume('orders', async (message) => {
  const MAX_BATCH = 50;
  const batch = await message.batch(MAX_BATCH);
  
  // Process at a sustainable rate
  for (const order of batch) {
    await processOrder(order);
  }
  
  await message.ack();
});
```

**Why this works:** The producer and consumer are decoupled. A surge of 10k orders won't crash the worker — it'll just mean 10k messages in the queue.

**The trade-off:** You trade latency for reliability. That order you just placed? It's not processed yet. Could be 100ms, could be 10 seconds depending on queue depth. You also need to monitor queue depth — an unbounded queue during sustained overload is just a memory bomb.

### 3. Throttling (Rate Limiting at Service Level)

Not at the API gateway level — at the *service-to-service* level.

```
class InternalApiClient {
  private rateLimiter: RateLimiter;
  
  async callService(endpoint: string, data: unknown) {
    await this.rateLimiter.wait(); // blocks until token available
    return this.http.post(endpoint, data);
  }
}

// Token bucket with 100 requests/second, burst up to 150
const client = new InternalApiClient({
  tokensPerSecond: 100,
  maxBurst: 150
});
```

**Why this matters:** Distributed rate limiting between services prevents a fast consumer from overwhelming a slow producer. It's like saying "I know you can handle 100 rps, so I'll never send you more than that."

### 4. Reactive / Push-Based Backpressure

This is the most sophisticated pattern — the consumer tells the producer how much it can handle. It's how TCP flow control and Reactive Streams work.

```
// Producer asks "how many can you take?"
Consumer: "I can handle 10 items"

// Producer sends exactly 10
Producer: [1, 2, 3, ..., 10]

// Consumer processes and requests more
Consumer: "Give me 5 more"
Producer: [11, 12, 13, 14, 15]

// Consumer is slowing down
Consumer: "Give me 1 more... okay, now 2 more..."
```

This is the foundation of **Reactive Streams** (RxJava, Project Reactor, Akka Streams). The consumer controls the pace.

```
Flux<Order> orders = orderStream
  .onBackpressureDrop()        // drop if consumer can't keep up
  .limitRate(100)              // request 100 at a time
  .subscribe(processor);
```

**Real-world example:** PostgreSQL's `COPY` protocol uses this. The server sends a "ready for more data" signal, and the client sends the next batch. No signal? No data.

## Backpressure in Practice

Here's a realistic setup combining multiple strategies:

| Layer | Strategy | Why |
|-------|----------|-----|
| API Gateway | Rate limiting + 503 | First line of defense |
| Service A → B | Circuit breaker with queue | Prevent cascading |
| Service B → DB | Connection pooling with wait queue | Don't overwhelm the DB |
| Database | Max connections + query timeout | Last resort protection |

### Kafka Consumer Lag as Backpressure

Kafka is a great example of backpressure in action. The consumer controls how fast it reads:

```java
// Slow consumer - naturally creates backpressure
consumer.subscribe(patterns);
while (true) {
  // Only fetch 100 records at a time
  ConsumerRecords<String, String> records = consumer.poll(100);
  
  for (ConsumerRecord<String, String> record : records) {
    process(record); // slow processing creates lag
  }
  
  // Don't commit until processed
  consumer.commitSync();
}
```

If processing is slow, the consumer just... doesn't poll. The broker holds the data. No connections are wasted, no memory is exhausted. The lag metric tells you exactly how far behind you are.

## The Anti-Pattern: ∞ Buffers

**Never** do this:

```java
// ❌ INFINITE QUEUE - eventual OOM
BlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<>();
ExecutorService executor = new ThreadPoolExecutor(
  10, 10, 0L, TimeUnit.MILLISECONDS, taskQueue
);

// A surge of 1M tasks just sits in memory
for (int i = 0; i < 1_000_000; i++) {
  executor.submit(() -> slowTask()); // BOOM 💥
}
```

Always bound your queues:

```java
// ✅ Bounded queue with rejection policy
BlockingQueue<Runnable> taskQueue = new ArrayBlockingQueue<>(1000);
ExecutorService executor = new ThreadPoolExecutor(
  10, 20, 60L, TimeUnit.SECONDS, 
  taskQueue,
  new ThreadPoolExecutor.CallerRunsPolicy() // runs in caller's thread when full
);
```

The `CallerRunsPolicy` is particularly elegant — when the queue is full, the *caller* thread executes the task, which means whoever submitted the work now has to wait. That creates natural backpressure.

## When You Don't Need This

Not every system needs elaborate backpressure:

- **Synchronous APIs with low traffic:** A simple `maxConcurrentRequests` check is fine
- **Internal tools:** If you control all the clients, buffer and throttle at the source
- **Read-only services:** CDNs and caches handle most of this for you

You need real backpressure strategies when:
- You have a chain of services (3+ hops)
- Traffic is bursty and unpredictable
- You can't afford to lose data but can't always keep up
- Different services have different capacity profiles

---

## Key Takeaways

- **Cascading failure is the real threat** — a slow downstream kills everything upstream
- **Bounded queues over infinite buffers** — always set a max size
- **Combine strategies** — rate limit at the edge, throttle between services, queue for async work
- **Monitor the right signals** — connection pool utilization, queue depth, thread pool saturation, Kafka consumer lag
- **Dropping is sometimes better than waiting** — a fast 503 beats a slow timeout that ties up resources
- **Use reactive streams for fine-grained control** — let the consumer dictate the pace
