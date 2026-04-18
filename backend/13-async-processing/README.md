# Asynchronous Processing: Queues and Background Jobs

Your API just received a request to send 10,000 marketing emails. If you process them synchronously, the client waits 5 minutes for a response—or until the connection times out. Either way, you've got a problem.

Some operations just take too long to run during a request. Email delivery, image processing, report generation, third-party API calls—they can't all happen in the 200ms your users expect. That's where async processing comes in.

We'll cover:
- When to offload work to background jobs
- Queue patterns: push vs pull
- Popular tools (Redis-based, cloud-native, message brokers)
- Handling failures and retries
- Trade-offs and gotchas

## What Goes Wrong Without Async

Let's say you have an endpoint that processes video uploads:

```python
# ❌ Everything in the request handler
@app.post("/upload")
def upload_video(file):
    video = save_file(file)                    # ~500ms
    thumbnail = generate_thumbnail(video)       # ~3s
    transcoded = transcode_to_mp4(video)        # ~30s
    upload_to_cdn(transcoded)                   # ~5s
    notify_user(email, "Video ready!")          # ~1s
    
    return {"status": "done"}  # User waited 40+ seconds
```

What happens?

1. **Timeouts** — Load balancers, proxies, and browsers all have default timeouts (30-60 seconds). Long jobs get killed.
2. **Resource exhaustion** — Your web server processes are blocked, not serving other requests.
3. **Poor UX** — Users stare at spinners, wondering if it's working.
4. **Retry chaos** — Users click "submit" again, starting duplicate jobs.

## The Pattern: Offload and Respond

Instead of doing the work during the request, queue it:

```python
# ✅ Queue the work, respond immediately
@app.post("/upload")
def upload_video(file):
    video = save_file(file)  # Quick, we need the path
    
    # Push job to queue
    queue.enqueue(
        "process_video",
        video_path=video.path,
        user_email=request.user.email
    )
    
    return {"status": "processing", "video_id": video.id}  # ~50ms
```

The client gets an immediate response. The heavy lifting happens elsewhere, at its own pace.

## Queue Patterns: Push vs Pull

### Push Queues (Job Queues)

You explicitly push a job to a specific queue. A worker pulls it and executes.

```
[API] --enqueue--> [Queue] <--dequeue-- [Worker]
```

**Examples:** Sidekiq, Bull, RQ, Celery, AWS SQS

```python
# Push queue example (Python/RQ)
from rq import Queue
from redis import Redis

redis = Redis()
queue = Queue("videos", connection=redis)

# Enqueue the job
job = queue.enqueue(
    process_video,
    video_path="/uploads/video-123.mp4",
    user_email="user@example.com"
)

# Check status later
job.get_status()  # 'queued', 'started', 'finished', 'failed'
```

**When to use:** Most background jobs in web apps—email, notifications, file processing, scheduled tasks.

### Pull Queues (Message Brokers)

Producers publish messages to topics/exchanges. Consumers subscribe and pull at their own pace.

```
[Producer] --publish--> [Exchange] --route--> [Queue] <--consume-- [Consumer]
```

**Examples:** RabbitMQ, Kafka, AWS SNS/SQS

```python
# Message broker example (RabbitMQ/pika)
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare exchange and queue
channel.exchange_declare(exchange='videos', exchange_type='direct')
channel.queue_declare(queue='video_processing')
channel.queue_bind(exchange='videos', queue='video_processing', routing_key='process')

# Publish
channel.basic_publish(
    exchange='videos',
    routing_key='process',
    body=json.dumps({"video_id": 123, "email": "user@example.com"})
)
```

**When to use:** Event-driven architectures, microservices communication, event sourcing, when you need pub/sub semantics.

## Popular Tools

| Tool | Type | Best For | Language Support |
|------|------|----------|------------------|
| **Sidekiq** | Redis-based | Ruby apps, simple jobs | Ruby |
| **Bull** | Redis-based | Node.js, reliable queues | Node.js |
| **RQ** | Redis-based | Python, simple to start | Python |
| **Celery** | Multi-backend | Python, complex workflows | Python |
| **AWS SQS** | Cloud queue | Serverless, no infra | Any |
| **RabbitMQ** | Message broker | Complex routing, pub/sub | Any |
| **Kafka** | Event stream | High throughput, events | Any |

### Redis-based Queues (Simplest to Start)

```python
# RQ (Python) - minimal setup
from rq import Queue, Worker
from redis import Redis

# Define the job
def send_welcome_email(user_id):
    user = User.get(user_id)
    mailer.send(user.email, "Welcome!", "...")

# Enqueue
q = Queue(connection=Redis())
job = q.enqueue(send_welcome_email, user_id=123)
```

### Celery (More Features, More Complexity)

```python
# Celery - workflows, retries, scheduling
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task(bind=True, max_retries=3, default_retry_delay=60)
def process_video(self, video_path):
    try:
        video = transcode(video_path)
        upload_to_cdn(video)
    except TranscodeError as e:
        self.retry(exc=e)  # Retry with exponential backoff
    except CDNAError as e:
        raise self.retry(exc=e, countdown=300)  # Retry in 5 minutes
```

## Handling Failures: Retries and Dead Letters

Jobs will fail. Networks flake. APIs go down. You need a strategy.

### Retry with Backoff

```python
# ❌ Immediate retry on failure
job.retry()  # Might overwhelm a struggling service

# ✅ Exponential backoff
@app.task(bind=True, autoretry_for=(ConnectionError,), retry_backoff=True, max_retries=5)
def call_third_party_api(url, data):
    return requests.post(url, json=data)
```

### Dead Letter Queues

After max retries, move the job somewhere you can inspect it:

```
[Job Queue] --(max retries)--> [Dead Letter Queue]
```

```python
# Bull (Node.js) - dead letter handling
const queue = new Queue('emails', {
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
    removeOnFail: false  // Keep failed jobs
  }
});

// Later, inspect failed jobs
const failed = await queue.getFailed();
```

## Trade-offs

### Latency vs Reliability

| Approach | Latency | Reliability | Use Case |
|----------|---------|-------------|----------|
| In-process | None | Low (crash = lost) | Trivial tasks |
| Redis queue | ~ms | Medium (Redis down = stuck) | Most web apps |
| Persistent queue | ~10ms | High (persisted to disk) | Financial, critical jobs |
| Outbox pattern | ~ms | Very High (DB transaction) | Must not lose jobs |

### The Outbox Pattern (Maximum Reliability)

When you *cannot* lose a job, store it in your database first:

```python
# ✅ Outbox pattern - atomic job creation
@app.post("/orders")
def create_order(items):
    # Both happen in the same transaction
    order = db.execute("INSERT INTO orders ...")
    db.execute("""
        INSERT INTO outbox (job_type, payload, status)
        VALUES ('process_order', ?, 'pending')
    """, json.dumps({"order_id": order.id}))
    db.commit()
    
    return {"order_id": order.id}

# Separate process picks up from outbox
def outbox_worker():
    while True:
        jobs = db.execute("SELECT * FROM outbox WHERE status = 'pending' LIMIT 10")
        for job in jobs:
            process_job(job)
            db.execute("UPDATE outbox SET status = 'done' WHERE id = ?", job.id)
            db.commit()
```

This guarantees no jobs are lost—even if the queue crashes after the transaction commits.

## Common Gotchas

### 1. Job Payload Size

```python
# ❌ Storing full objects (bloated, stale)
queue.enqueue(process_video, video=video_object)  # Serialized, could be MBs

# ✅ Store references only
queue.enqueue(process_video, video_id=video.id)
```

### 2. Implicit Dependencies

```python
# ❌ Worker might not have the same context
queue.enqueue(send_email, user=current_user)  # current_user is request-scoped!

# ✅ Pass everything explicitly
queue.enqueue(send_email, user_id=current_user.id, locale=request.locale)
```

### 3. Queue Backpressure

```python
# ❌ Unbounded queue growth
for item in items:
    queue.enqueue(process, item)  # Could create millions of jobs

# ✅ Check queue depth first
if queue.count < 10000:
    queue.enqueue(process, item)
else:
    # Handle overflow - log, shed load, or alert
    logger.warning("Queue full, shedding load")
```

### 4. Poison Pills

```python
# ❌ Infinite retry on permanent failure
queue.enqueue(parse_json, '{"malformed": ')  # Always fails, retries forever

# ✅ Validate before enqueuing, or have a max retry
def parse_json(data):
    try:
        return json.loads(data)
    except json.JSONDecodeError:
        logger.error(f"Invalid JSON: {data[:100]}")
        raise  # Let it fail permanently after retries
```

## Monitoring

You can't manage what you can't see:

```python
# Log job lifecycle
@app.task(bind=True)
def important_job(self, data):
    logger.info(f"Job {self.request.id} started")
    result = do_work(data)
    logger.info(f"Job {self.request.id} completed")
    return result
```

Key metrics to track:
- **Queue depth** — How many jobs waiting?
- **Processing time** — P50, P95, P99
- **Failure rate** — % of jobs failing
- **Age of oldest job** — Stuck jobs?

## When to Use Async Processing

✅ **Good candidates:**
- Email sending
- Image/video processing
- Report generation
- Third-party API calls
- Batch data imports
- Scheduled cleanups

❌ **Bad candidates:**
- Quick database reads
- Auth validation
- Anything the user needs immediately

## Takeaways

1. **Offload heavy work** — If it takes >1 second, queue it.
2. **Respond fast** — Give users immediate feedback, process in background.
3. **Plan for failure** — Retries with backoff, dead letter queues, monitoring.
4. **Keep payloads small** — IDs, not objects.
5. **Match tool to use case** — Redis queues for simple jobs, message brokers for events, outbox for critical jobs.
6. **Monitor queue health** — Depth, latency, failures. Silent queues hide problems.

---

*When your API starts timing out on slow operations, that's your cue: move it to a background job.*