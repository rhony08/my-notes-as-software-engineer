# Stream Processing with Kafka

You've got a message queue set up. Producers are sending events, consumers are reading them. Everything works. Then someone asks: "Can we also compute running totals from that stream?" or "We need real-time alerts based on rolling 5-minute windows."

Suddenly, simple consume-and-process isn't enough. You need **stream processing**.

Kafka isn't just a message broker — it's a stream processing platform. Let's talk about what that actually means.

## The Problem Kafka Solves for Streaming

Most messaging systems (RabbitMQ, SQS) are built around the idea of **consuming and deleting** messages. Once processed, the message is gone. This works great for task queues — but it's terrible for stream processing.

In stream processing, you need to:
- Replay historical data
- Combine multiple streams
- Maintain state across events (counters, aggregations, joins)
- Handle late-arriving events

Kafka's **log-based architecture** makes this possible. Messages persist, consumers control their own position (offsets), and data streams are partitioned for parallel processing.

```python
# A consumer controls its own offset — can replay from any point
consumer.assign([TopicPartition('orders', 0)])
consumer.seek(TopicPartition('orders', 0), 42)  # Skip to event 42

# Or start from the beginning
consumer.seek_to_beginning(TopicPartition('orders', 0))
```

This ability to rewind is what makes Kafka a stream processing platform, not just a queue.

## Consumer Groups — The Foundation of Parallelism

Before you process streams, you need to understand how Kafka distributes work.

```python
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order-processors',  # 👈 same group = shared work
    auto_offset_reset='earliest'
)
```

**How partitioning works:**

| Partition | Consumer | Behavior |
|-----------|----------|----------|
| 1 topic = 1 partition | 1 consumer | Everything goes to 1 process |
| 1 topic = 6 partitions | 2 consumers | Consumer A gets partitions 0,1,2; B gets 3,4,5 |
| 1 topic = 6 partitions | 10 consumers | 4 consumers sit idle — max parallelism = partition count |

**Critical rule:** `max parallelism = number of partitions`. If you need 20 parallel processors, create at least 20 partitions from the start.

```
# ❌ Don't start with 3 partitions and expect 50 consumers
# ✅ Plan ahead: 1 partition per expected consumer + headroom
```

## Kafka Streams — The Native Approach

Kafka Streams is a library (not a separate cluster) that turns your application into a stream processor. It runs in your app's JVM, connects to Kafka, and handles the hard parts — state management, exactly-once, rebalancing.

```java
// Stateless: filter events
KStream<String, Order> orders = builder.stream("orders");
orders.filter((key, order) -> order.value() > 100.0)
      .to("high-value-orders");

// Stateful: count by category (this needs state storage!)
KStream<String, Order> orders = builder.stream("orders");
orders.groupBy((key, order) -> order.getCategory())
      .count(Materialized.as("category-counts"))
      .toStream()
      .to("category-counts");
```

That `.count()` call isn't magic — Kafka Streams keeps a **local state store** (RocksDB by default) that gets changelogged back to a compacted Kafka topic. If your app crashes, it rebuilds state from the changelog.

### Stateful vs Stateless Operations

| Operation | Type | Memory? | Example |
|-----------|------|---------|---------|
| `filter()` | Stateless | No | Keep only error events |
| `map()` | Stateless | No | Transform event format |
| `branch()` | Stateless | No | Route to different topics |
| `count()` | Stateful | Yes | Running count by key |
| `reduce()` | Stateful | Yes | Aggregate over time |
| `join()` | Stateful | Yes | Combine two streams |
| `window()` | Stateful | Yes | Time-bounded aggregation |

**The trade-off:** Stateless ops scale horizontally perfectly. Stateful ops need repartitioning — all events with the same key go to the same partition (and the same local state store). This means hot keys become a bottleneck.

```
# ❌ A single user generates 80% of traffic
# State store for that user's partition grows huge
# Other partitions are idle
```

## Windowing — Making Sense of Time-Based Data

Raw counts are useful, but most stream processing questions are time-bound:

- "Revenue in the last 5 minutes"
- "Errors per minute"
- "Top products this hour"

This is **windowing**. Kafka Streams supports several types:

```java
// Tumbling window — fixed, non-overlapping
// Count orders every 5 minutes
TimeWindows.of(Duration.ofMinutes(5))

// Hopping window — overlapping, better for smoothing
// 5-minute window that advances every 1 minute
TimeWindows.of(Duration.ofMinutes(5))
           .advanceBy(Duration.ofMinutes(1))

// Sliding window — used for joins within a time delta
// Join order events and payment events within 30 seconds
JoinWindows.of(Duration.ofSeconds(30))

// Session window — grouped by activity gaps
// Group clicks into sessions with 15-minute inactivity gap
SessionWindows.with(Duration.ofMinutes(15))
```

**Real example — detecting a spike in 5xx errors:**

```java
KStream<String, HttpRequest> requests = builder.stream("api-requests");

requests.filter((key, req) -> req.statusCode() >= 500)
        .groupBy((key, req) -> req.getEndpoint())  // group by API endpoint
        .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
        .count()
        .toStream()
        .filter((windowedKey, count) -> count > 100)  // alert if >100 errors
        .to("error-alerts");
```

## Exactly-Once Semantics — Not as Scary as It Sounds

The hardest problem in stream processing: **you processed a message, but crashed before recording the offset.** On restart, you reprocess the same message.

Kafka's exactly-once semantics (EOS) address this:

```
Produce to Kafka ──► Process ──► Commit offset ──► Produce output
                                                    ↑
                                           These must be atomic
```

EOS uses a **transactional coordinator** — the produce-to-output-topic and commit-offset happen in a single transaction. If the transaction fails, both roll back.

```java
// Enable EOS — handled automatically by Kafka Streams
Properties props = new Properties();
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, 
          StreamsConfig.EXACTLY_ONCE_V2);

// That's it. The library handles the rest.
```

**But there's a performance cost.** EOS adds about 10-20% latency. For many use cases, **at-least-once** with idempotent downstream consumers is a better trade-off.

```
# Think about your semantics:
# - Payment processing → Exactly once (duplicate payments = bad)
# - Analytics counters → At-least-once (a few extra counts ≈ fine)
# - Log aggregation → At-most-once (dropping a log line ≈ fine)
```

## Beyond Kafka Streams: ksqlDB

If writing Java isn't your thing, **ksqlDB** lets you process streams with SQL:

```sql
-- Create a stream from a Kafka topic
CREATE STREAM orders (
  order_id VARCHAR,
  user_id VARCHAR,
  amount DOUBLE,
  category VARCHAR
) WITH (kafka_topic='orders', value_format='JSON');

-- Continuous query — runs forever, updating results
SELECT category, COUNT(*) AS order_count, SUM(amount) AS revenue
FROM orders
WINDOW TUMBLING (SIZE 5 MINUTES)
GROUP BY category
EMIT CHANGES;
```

This runs as a stream processing job. Every new order updates the output, continuously. ksqlDB handles the underlying Kafka Streams plumbing.

**When to use which:**

| Approach | Best For | Not For |
|----------|----------|---------|
| Kafka Streams (Java) | Custom logic, complex state, low latency | Non-JVM teams |
| ksqlDB | SQL-savvy teams, fast prototyping | Complex multi-step pipelines |
| Kafka Connect | Moving data between systems | Processing/transformations |
| Spark/Flink + Kafka | Heavy analytics, ML pipelines | Simple transformations |

## Common Pitfalls

**1. Increasing partitions is not transparent.**
```python
# ❌ Assumption: partition key = user_id guarantees ordering
# If you repartition from 3 to 10 partitions, 
# the same user_id now lands on a different partition
# Ordering within a user_id group breaks
```

**2. State store size is unbounded without cleanup.**
```java
// Configure a changelog topic with cleanup policy
Materialized.as(Stores.persistentTimestampedKeyValueStore("my-store"))
    .withLoggingConfig(Map.of("cleanup.policy", "compact,delete"));
// Without TTL or compaction, state stores grow forever
```

**3. Rebalancing pauses processing.**
When a consumer joins or leaves a group, Kafka triggers a rebalance. During rebalance, *no processing happens*. With many partitions and stateful operations, this can take minutes.

```
Mitigation:
- Use static group membership (group.instance.id)
- Minimize partition count vs consumer ratio changes
- Avoid deploying during peak hours
```

## When Stream Processing Isn't the Answer

Not everything should be a stream. If you need:

- **Complex historical queries** → Use a database, not a stream
- **Ad-hoc analysis** → Dump to a data warehouse
- **Exactly-once reporting with late data correction** → Batch processing (Spark, Hive) handles this better
- **API-driven request-response** → Kafka adds needless latency

Stream processing shines when you need **continuous, low-latency results** from unbounded data. If you can batch overnight, batch is simpler.

## Actionable Takeaways

1. **Design partitions upfront** — you can't easily increase them without breaking key-based ordering
2. **Choose your processing semantics pragmatically** — exactly once for money, at least once for analytics
3. **Monitor state store size** — RocksDB grows silently if you forget cleanup policies
4. **Plan for rebalancing downtime** — it's not zero-downtime by default
5. **Use Kafka Streams for JVM teams, ksqlDB for quick queries, Flink for heavy analytics**
6. **Test with late-arriving events** — streaming systems behave differently with out-of-order data than batch
