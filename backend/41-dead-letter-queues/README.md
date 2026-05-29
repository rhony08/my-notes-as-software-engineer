# Dead Letter Queues: Handling Failed Messages

Messages fail. That's just reality. A malformed payload, a downstream service timing out, a bug in your processing logic—any of these can wreck a message that's already sitting in your queue.

The naive approach? Retry forever until it works. The problem with that is pretty obvious: a message that's *always* going to fail (bad JSON, missing fields) will just clog your queue, eat resources, and delay the messages that *can* be processed.

Enter the **Dead Letter Queue** (DLQ)—a parking lot for messages that couldn't be processed.

## What's a Dead Letter Queue?

A DLQ is exactly what it sounds like: a separate queue where messages go to die. When your consumer tries to process a message and fails (after exhausting its retries), instead of dropping it entirely, you move it to a DLQ.

The idea is dead simple:

```
Main Queue → Consumer → Success ✅ (processed and acknowledged)
                         → Failure → Retry (3-5 times)
                                   → Still failing? → DLQ 🪦
```

Once a message lands in the DLQ, a human or automated process can inspect it, decide what went wrong, fix it, and *replay* it back to the main queue.

## Why Messages Fail

Not all failures are the same. Understanding *what* failed tells you how to handle it:

| Type | Example | Should Retry? | Should DLQ? |
|------|---------|---------------|-------------|
| **Transient** | Database connection timeout, rate limited | ✅ Yes | ❌ Not yet |
| **Poison pill** | Malformed JSON, corrupted payload | ❌ No | ✅ Yes |
| **Business logic** | Missing data in downstream service | ⚠️ Sometimes | ✅ Eventually |
| **Configuration** | Wrong routing key, missing env var | ❌ No | ✅ Yes |

**Poison pill messages** are the classic DLQ candidate. These are messages that will *never* process successfully, no matter how many times you retry.

```
// ❌ DON'T retry a poison pill forever
function processOrder(message) {
  try {
    const order = JSON.parse(message.body);
    // ... process order
    channel.ack(message);
  } catch (err) {
    // This JSON is broken and will ALWAYS be broken.
    // Retrying 100 times wastes 99 more attempts.
    channel.nack(message, false, false); // requeue = false → DLQ
  }
}

// ✅ Better: retry transient errors, DLQ everything else
function processOrder(message) {
  const retryCount = getRetryCount(message);

  try {
    const order = JSON.parse(message.body);
    await saveOrder(order);
    channel.ack(message);
  } catch (err) {
    if (err instanceof SyntaxError) {
      // Poison pill—don't retry
      channel.nack(message, false, false); // send to DLQ
    } else if (retryCount < 3) {
      // Transient failure—retry
      channel.nack(message, false, true); // requeue
    } else {
      // Exhausted retries—DLQ it
      channel.nack(message, false, false);
    }
  }
}
```

## Implementing DLQs

### RabbitMQ

RabbitMQ has first-class DLQ support. You just configure it when declaring your queue:

```javascript
// RabbitMQ setup with DLQ
const amqp = require('amqplib');

async function setup() {
  const conn = await amqp.connect('amqp://localhost');
  const channel = await conn.createChannel();

  // Dead letter exchange
  await channel.assertExchange('orders.dlx', 'direct', { durable: true });

  // Dead letter queue
  await channel.assertQueue('orders.dlq', { durable: true });
  await channel.bindQueue('orders.dlq', 'orders.dlx', '');

  // Main queue—messages that fail go to DLX
  await channel.assertQueue('orders', {
    durable: true,
    deadLetterExchange: 'orders.dlx',
    // Messages expire in DLQ after 24 hours
    deadLetterRoutingKey: ''
  });

  // Consumer
  channel.consume('orders', async (msg) => {
    try {
      await process(msg);
      channel.ack(msg);
    } catch (err) {
      // nack with requeue=false → automatically goes to DLX → DLQ
      channel.nack(msg, false, false);
    }
  });
}
```

The key bit: `deadLetterExchange: 'orders.dlx'`. When a message is nack'd with `requeue = false`, RabbitMQ automatically routes it to the DLX instead of just dropping it.

### AWS SQS

SQS makes it even easier—DLQs are a queue configuration:

```javascript
// AWS CDK—SQS with DLQ
import * as sqs from 'aws-cdk-lib/aws-sqs';

const dlq = new sqs.Queue(this, 'OrdersDLQ', {
  queueName: 'orders-dlq',
  retentionPeriod: Duration.days(14), // keep dead letters 14 days
});

const mainQueue = new sqs.Queue(this, 'OrdersQueue', {
  queueName: 'orders',
  visibilityTimeout: Duration.seconds(30),
  deadLetterQueue: {
    queue: dlq,
    maxReceiveCount: 3, // after 3 receives → DLQ
  },
});
```

That's it. After a message is received 3 times without being deleted, SQS automatically moves it to the DLQ. No code changes needed in your consumer.

## DLQ Monitoring Is Non-Negotiable

Here's the thing about DLQs—they're useless if nobody watches them.

A pile of messages sitting in a DLQ means something's broken. If you're not monitoring, you'll discover it when a customer complains, not when the first message failed.

```yaml
# Prometheus alert rule for DLQ depth
groups:
  - name: dlq-alerts
    rules:
      - alert: DLQMessagesAccumulating
        expr: aws_sqs_approximate_number_of_messages_visible{queue_name="orders-dlq"} > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Messages accumulating in orders DLQ"
          description: "{{ $value }} messages in DLQ—check for processing failures"
```

**What to alert on:**
- **DLQ depth > 0** — any message in DLQ is a potential signal of something wrong
- **DLQ growth rate** — if messages are flooding the DLQ, you've got a systemic issue
- **DLQ age** — messages that sit for hours mean nobody's looking at them

## Processing Dead Letters

When messages land in your DLQ, you need a plan for handling them. Two common approaches:

### 1. Manual Inspection + Replay

A human checks the DLQ, fixes the root cause, and replays messages.

```
DLQ admin panel:
┌──────────────────────────────────────────────────┐
│ DLQ: orders-dlq (14 messages)                    │
│                                                  │
│ ┌────────────────────────────────────────────┐  │
│ │ May 28 10:23 - order-9812 - SyntaxError   │  │
│ │ May 28 10:24 - order-9813 - TimeoutError  │  │
│ │ May 28 10:25 - order-9814 - SyntaxError   │  │
│ └────────────────────────────────────────────┘  │
│                                                  │
│ [View Details] [Replay to Main] [Delete]         │
└──────────────────────────────────────────────────┘
```

This works for low-volume systems where you can manually triage.

### 2. Automated Reprocessing

For higher volumes, build a DLQ consumer that tries to fix messages and replay them:

```javascript
// DLQ consumer—attempts to reprocess with a fix
async function consumeDLQ() {
  const dlqMessages = await sqs.receiveMessage({
    QueueUrl: DLQ_URL,
    MaxNumberOfMessages: 10,
  }).promise();

  for (const message of dlqMessages.Messages) {
    try {
      const fixed = attemptAutoRepair(message);
      await sqs.sendMessage({
        QueueUrl: MAIN_QUEUE_URL,
        MessageBody: fixed,
      }).promise();
      await sqs.deleteMessage({
        QueueUrl: DLQ_URL,
        ReceiptHandle: message.ReceiptHandle,
      }).promise();
    } catch (err) {
      console.error(`Can't auto-fix message ${message.MessageId}:`, err.message);
      // Leave it in the DLQ for manual handling
    }
  }
}
```

## Trade-offs to Know

DLQs aren't free. They come with their own baggage:

| Pro | Con |
|-----|-----|
| Prevents message loss | Adds operational complexity (another queue to manage) |
| Gives visibility into failures | Can mask bugs if nobody monitors |
| Enables replay workflows | Dead letters accumulate if no cleanup policy |
| Decouples error handling | Need alerting to be useful |

**The biggest mistake** teams make: setting up a DLQ, configuring monitoring only to warn on DLQ depth being > 0, then being woken up at 3 AM because a single transient failure caused 1 message to land there. You need a delay and a threshold before alerting. A couple of messages in the DLQ *over time* isn't necessarily an emergency. 50 messages in 5 minutes? That's a problem.

**The second biggest mistake**: never replaying the DLQ. A DLQ with 10,000 messages that nobody ever processes is just a fancy way to lose data. Set a retention period (14 days is common) and have a process for periodic DLQ review.

## What to Keep in Mind

- **DLQ ≠ garbage collector.** Dead letters should be inspected and replayed, not ignored.
- **Set a TTL on DLQ messages.** Otherwise they'll sit forever and you'll pay for storage on data you'll never use.
- **Use separate DLQs for different message types.** An orders DLQ and a notifications DLQ should not be the same queue—the failure patterns and handling are different.
- **Track DLQ reasons.** Log *why* a message ended up in the DLQ so you can quickly triage without inspecting every payload.
- **Automate when you can, manually triage when you must.** Not every failure can be auto-repaired, but your team should handle the edge cases, not the routine ones.

The most pragmatic approach: configure DLQs from day one (it's cheap to add), alert on significant accumulation, and build a simple replay tool before you need it. Future you, getting paged at 2am, will thank present you.
