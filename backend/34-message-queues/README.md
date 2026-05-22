# Message Queues: RabbitMQ vs Kafka

You've built a service, and now another team wants their service to talk to yours. Direct HTTP calls work for a while, but then things get messy — one service goes down, requests pile up, retries time out, and suddenly nothing works. You need something in between. Something that decouples your services and handles the chaos.

That's where message queues come in.

But here's the thing — there are *a lot* of options. RabbitMQ and Kafka are the two most common. And people love to argue about which is better. The real answer? They solve *different problems*. Picking the wrong one will make your life miserable.

## What They Actually Are

**RabbitMQ** is a message broker. Think postal service — it delivers messages from point A to point B, confirms delivery, and reroutes if something goes wrong. It's built around the idea of **queues** and **exchanges**.

**Kafka** is a distributed log. Think of it like a newspaper subscription — producers write to a log (a topic), and consumers read from it at their own pace. Nobody tracks who got the message. You get what's published, and you keep track of where you left off.

```
RabbitMQ:  Producer → [Exchange] → Queue → Consumer   ✅ Delivery confirmed
Kafka:     Producer → [Topic/Partition] ← Consumer     🔄 Read at your pace
```

That fundamental difference drives everything else.

## When RabbitMQ Shines

RabbitMQ is your friend when you need **reliable task distribution**.

Let's say you have an e-commerce app where users upload product images. You need to:
1. Resize images to multiple formats
2. Generate thumbnails
3. Apply watermark
4. Store to CDN

This is a perfect RabbitMQ job. Each image upload becomes a message. Multiple worker services pick up messages from the queue. If a worker crashes mid-process, RabbitMQ knows and redelivers the message to another worker.

```python
# Pseudocode — publishing a task to RabbitMQ
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='image_processing', durable=True)

message = {
    'image_id': 'abc-123',
    'format': 'jpeg',
    'actions': ['resize', 'thumbnail', 'watermark']
}

channel.basic_publish(
    exchange='',
    routing_key='image_processing',
    body=json.dumps(message),
    properties=pika.BasicProperties(delivery_mode=2)  # Make message persistent
)

print(f"Queued image processing for {message['image_id']}")
connection.close()
```

The worker side:

```python
# Worker — picks up tasks and processes them
def callback(ch, method, properties, body):
    task = json.loads(body)
    try:
        process_image(task['image_id'], task['actions'])
        ch.basic_ack(delivery_tag=method.delivery_tag)  # ✅ Acknowledge completion
    except ProcessingError:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)  # 🔄 Try again later

channel.basic_consume(queue='image_processing', on_message_callback=callback)
channel.start_consuming()
```

**What RabbitMQ handles for you:**
- **Delivery acknowledgements** — knows if a consumer crashed and requeues
- **Dead lettering** — messages that keep failing go to a dead letter queue for inspection
- **Flexible routing** — fanout, direct, topic exchanges for different routing patterns
- **TTL and delays** — messages can expire or be delayed

### ❌/✅ RabbitMQ Approach

```
❌ Need to process 10M events per second across 100 consumer groups?
→ RabbitMQ will struggle. It's not built for that throughput.

✅ Need to dispatch 1,000 image-processing tasks to 5 workers, 
   making sure each task is processed exactly once?
→ RabbitMQ is perfect for this.
```

## When Kafka Wins

Kafka comes into play when you're dealing with **streams of events** that multiple consumers need to process independently.

Think about a ride-sharing app. Every second, thousands of drivers send their GPS coordinates. You need:
- A real-time map showing all drivers
- A pricing service that adjusts surge pricing
- An analytics pipeline tracking driver hours
- An ML model training on trip patterns

With RabbitMQ, you'd have to publish the same GPS data to four separate queues. With Kafka, all of that data sits in one topic, and each consumer reads at their own pace.

```python
# Kafka producer — publishing GPS events
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

gps_event = {
    'driver_id': 42,
    'lat': 40.7128,
    'lng': -74.0060,
    'timestamp': 1704101234,
    'speed_kmh': 35
}

# Kafka handles partitioning — events for same driver go to same partition
producer.send('driver_locations', key=b'driver_42', value=gps_event)
producer.flush()
```

Consumers can process at completely different speeds:

```python
# Consumer 1 — real-time map (reads latest, doesn't care about history)
consumer_map = KafkaConsumer(
    'driver_locations',
    bootstrap_servers=['localhost:9092'],
    group_id='real-time-map',
    auto_offset_reset='latest'  # Only care about new events
)

# Consumer 2 — analytics (processes everything going back 30 days)
consumer_analytics = KafkaConsumer(
    'driver_locations',
    bootstrap_servers=['localhost:9092'],
    group_id='analytics-pipeline',
    auto_offset_reset='earliest'  # Read from the beginning
)
```

**What Kafka handles for you:**
- **Massive throughput** — tens of thousands of messages per second per partition
- **Log retention** — messages persist for days/weeks, consumers can replay
- **Consumer groups** — multiple consumers can read independently
- **Exactly-once semantics** — when configured properly, no duplicates
- **Partitioning** — messages with the same key land in the same partition, preserving order

### ❌/✅ Kafka Approach

```
❌ Need simple task distribution with guaranteed delivery to ONE worker?
→ Kafka adds complexity you don't need. Use RabbitMQ.

✅ Need to process 50K events/second and have 3 different services 
   consume the same event stream independently?
→ Kafka is your tool.
```

## The Showdown

| Dimension | RabbitMQ | Kafka |
|-----------|----------|-------|
| **Core concept** | Message broker, queues | Distributed commit log |
| **Delivery model** | Push to consumer | Consumer pulls |
| **Message tracking** | Tracks acknowledgements, removes on ack | Consumer tracks offset, messages persist |
| **Throughput** | ~10K-50K msg/s per node | ~100K-1M+ msg/s per node |
| **Message lifecycle** | Deleted after consumer acknowledges | Configurable retention (time/size) |
| **Routing** | Complex (direct, topic, fanout, headers) | Simple (topic-based with partitioning) |
| **Message order** | Per queue | Per partition |
| **Persistence** | Optional (memory or disk) | Always disk-based, append-only |
| **Maturity** | 2007, known for stability | 2011, known for scale |
| **Operations** | Simple to run, good UI | Needs ZooKR or KRaft, more ops overhead |
| **Latency** | Microseconds | Milliseconds (batches for throughput) |

## When to Pick What

### Pick RabbitMQ when:
- You need **reliable task distribution** (job queues, work dispatch)
- You want **flexible routing** (messages need to go to different places based on content)
- Your **throughput needs are moderate** (under ~50K messages/second)
- You need **guaranteed delivery to one consumer** (not fan-out to many)
- Your team is smaller and wants something that "just works" out of the box

### Pick Kafka when:
- You need **event streaming / data pipelines** (CDC, logs, metrics)
- **Multiple consumers** need the same data independently
- You need to **replay history** — re-process last 30 days of events
- Your **throughput is high** (100K+ messages/second)
- You need **long-term storage** of event data

## The Third Option No One Talks About

If you're on AWS and just need something that works, **SQS + SNS** often beats both. SQS gives you RabbitMQ-like queue semantics (dead letter queues, retries, visibility timeouts) and SNS handles the fan-out. No servers to manage, scales infinitely, and you pay per request.

But if you need RabbitMQ or Kafka's features specifically (complex routing, log retention, etc.), managed versions exist: Amazon MQ for RabbitMQ, MSK for Kafka.

## Things I Wished I Knew Sooner

**RabbitMQ gotchas:**
- Default vhost and guest/guest credentials should be changed immediately
- `durable=true` on queues doesn't mean much if your messages aren't persistent
- The management plugin is incredibly useful — enable it from day one
- Clustering isn't as straightforward as you'd think for HA

**Kafka gotchas:**
- ZooKeeper is a real operational burden (KRaft mode improves this)
- Message order is only guaranteed *within a partition*, not across partitions
- You need to think about partitioning strategy up front — changing it later means reprocessing
- Monitoring consumer lag is essential — use Burrow or built-in Kafka tools

---

### What You Can Do Right Now

1. **Audit your current architecture** — are you using a message queue where a direct call would work? Or vice versa?
2. **Map your traffic patterns** — task distribution or event streaming? That's your main decision point.
3. **Start simple** — RabbitMQ for task queues, Kafka for event streams. Don't overthink it.
4. **Monitor your queues** — RabbitMQ queue depth growing? Kafka consumer lag increasing? Those are early warning signs.
5. **Don't ignore managed services** — SQS/SNS on AWS, Cloud Pub/Sub on GCP. Sometimes the best message queue is the one you don't have to manage.
