# CPU Optimization Patterns

Your API was fine at 9am. By 10:30, latency charts look like a ski jump, the autoscaler has spun up twice as many instances, and your cloud bill is having a bad day. Nobody changed a line of code. You open the metrics dashboard and there it is: **CPU pinned at 100% on every instance**.

CPU is the nastiest resource to debug because it doesn't fail gracefully. Memory gets you OOM-kills and crash loops. Disk fills up with warnings. But CPU just makes *everything* slower — database calls, network responses, even the health checks that are supposed to save you. And by the time you notice, the problem has been compounding for hours. The good news: CPU waste follows a handful of repeatable patterns. Find the pattern, and you've found the fix.

## Measure First, Guess Never

I'll say it once and mean it: **do not optimize CPU until you've profiled.** Every time I've skipped this step, I've "fixed" the wrong thing. You'll swear the problem is the database query when it's actually the JSON serializer eating 40% of your CPU. You'll rewrite a function to be "more efficient" when it was already 0.01% of your runtime.

Your first move is always a profiler. Not `top` (that just tells you *something* is hot, not *what*).

| Tool | Language | What it gives you |
|---|---|---|
| `perf` + flamegraphs | Native (C, Go, Rust, JVM via async-profiler) | Kernel-level CPU attribution, call stacks |
| `async-profiler` | JVM | CPU + allocation flamegraphs, no JVM restarts |
| `py-spy` | Python | Attach to a running process, no code changes |
| `pprof` | Go | Built-in, dead simple: `go tool pprof` |
| Node's `--cpu-prof` | Node.js | CPU profile files with V8's profiler |
| `cProfile` | Python | Function-level call counts and times |

The workflow is always the same: reproduce the load → capture a profile → look at the *widest* part of the flamegraph → fix that → re-profile. If you fix the second-widest thing first, congratulations, you just made the widest thing a bigger percentage.

```bash
# Go: capture 30s of CPU profile from a running service
curl -o cpu.prof "http://localhost:6060/debug/pprof/profile?seconds=30"
go tool pprof -top cpu.prof

# Node: profile a process, then open the .cpuprofile in Chrome DevTools
node --cpu-prof --cpu-prof-dir=./profiles server.js
```

## The Algorithmic Sins

CPU problems are usually not "the machine is slow" — they're "we're doing way more work than we need to." The three classics:

### 1. The N+1 Query

The most common CPU sink in backend code, and it doesn't even look like CPU — it looks like database time.

```js
// ❌ One query per order, then one per customer — 101 queries for 100 orders
const orders = await db.query("SELECT * FROM orders WHERE user_id = $1", [userId]);
for (const order of orders) {
  const customer = await db.query("SELECT * FROM customers WHERE id = $1", [order.customer_id]);
  // ... 
}
```

That's 101 round trips. Each one carries network latency, serialization, connection pool churn, and query planning. The CPU cost hides in all of it.

```js
// ✅ One query. JOIN does the work in the database, which is better at it.
const rows = await db.query(`
  SELECT o.*, c.name AS customer_name
  FROM orders o
  JOIN customers c ON c.id = o.customer_id
  WHERE o.user_id = $1
`, [userId]);
```

### 2. Accidental O(n²)

Anything that looks like "loop inside a loop" deserves a skeptical glance. The sneaky version is doing a linear scan inside a loop when a hash map would do:

```js
// ❌ For every item, scan the whole list — O(n²) as the list grows
for (const item of items) {
  const match = allUsers.find(u => u.id === item.userId); // linear scan
  // ...
}
```

```js
// ✅ Build a map once, lookups become O(1) — O(n) total
const userById = new Map(allUsers.map(u => [u.id, u]));
for (const item of items) {
  const match = userById.get(item.userId);
  // ...
}
```

At 100 items, nobody notices. At 100k items — which is where production data always ends up — the difference is 10 billion comparisons vs 100k lookups. That's the whole story of "it was fast in staging."

### 3. Catastrophic Regex Backtracking

This one is a sleeper. A regex that runs fine on normal input can take *minutes* on crafted input, because the engine backtracks exponentially. It's also a known DoS vector (ReDoS).

```python
# ❌ Nested quantifiers — (a+)+ backtracks like crazy on "aaaa...b"
pattern = re.compile(r"^(a+)+$")
pattern.match("a" * 30 + "b")  # hangs for seconds, gets worse exponentially
```

```python
# ✅ Same intent, no nested quantifier explosion
pattern = re.compile(r"^a+$")
pattern.match("a" * 30 + "b")  # instant fail
```

Rule of thumb: if your regex has nested `+`, `*`, or `{m,n}` inside groups, test it against adversarial input *before* shipping. Tools like `recheck` or regex101's debugger will show you the backtracking steps.

## Allocation Pressure: The Silent CPU Tax

Here's something most people don't realize: **allocating memory costs CPU.** Every object you create has to be allocated, and eventually garbage-collected (or reference-counted). A service that "only" allocates a lot can burn 20–40% of its CPU in the garbage collector alone — and GC time shows up as CPU, not memory.

```go
// ❌ Building a string in a loop — each += allocates a new string, old ones get GC'd
var s string
for _, part := range parts {
    s += part
}
```

```go
// ✅ strings.Builder amortizes the allocation — one buffer, no GC churn
var sb strings.Builder
for _, part := range parts {
    sb.WriteString(part)
}
s := sb.String()
```

Same story in every language: use `StringBuilder`/`bytes.Buffer` instead of concatenation in loops, reuse buffers where you can, and watch out for frameworks that allocate per-request objects you don't need.

The trade-off: micro-optimizing allocations everywhere makes code uglier and harder to read. The right approach is *targeted* — profile first, find the hot loop, fix the allocations in the hot loop, leave the rest alone. Prematurely optimizing cold paths is how you end up with unreadable code and no measurable win.

## Blocking the Event Loop (and Thread Pools)

If you're on Node.js (or Python's asyncio, or any single-threaded event loop), CPU-bound work is a different beast: it doesn't just slow *that request* down — it blocks *everything*.

```js
// ❌ CPU-heavy work on the event loop — every other request stalls behind it
app.post("/reports", async (req, res) => {
  const data = await loadData(req.body.id);
  const result = heavyComputation(data); // synchronous, blocks the loop!
  res.json(result);
});
```

While `heavyComputation` runs, zero other requests get processed. Health checks time out. The load balancer marks you dead. It's like one person doing the dishes in a tiny kitchen — nobody else can even get to the sink.

```js
// ✅ Offload to a worker thread — event loop stays free
const { Worker } = require("worker_threads");

app.post("/reports", async (req, res) => {
  const data = await loadData(req.body.id);
  const result = await runInWorker("heavyComputation", data); // async
  res.json(result);
});
```

The same pattern exists everywhere: JVM and Go handle this automatically with thread pools and goroutines, but if you're hammering a shared resource (like a lock or a DB connection pool), you can still end up with *thread pool exhaustion* — threads all blocked, CPU idle, latency exploding. The fix there is usually not more threads; it's fewer blocking calls, shorter transactions, or async I/O.

## Busy-Waiting: Burning CPU While Doing Nothing

This is the most embarrassing pattern to find in your own code, because it looks like activity but does zero useful work:

```python
# ❌ Polling every 10ms — 100 wakeups per second, CPU pegged, doing nothing useful
while not queue.has_items():
    time.sleep(0.01)  # "checking" = busy loop with extra steps
process_next_item()
```

That 10ms sleep still wakes the process 100 times a second. Multiply by a few services and you've got a full core of CPU being burned by *waiting*.

```python
# ✅ Block on the queue — the OS parks the thread until something arrives
item = queue.get()  # blocks efficiently, zero CPU while waiting
process(item)
```

The general rule: **never poll for something that can notify you.** Queues, condition variables, channels, `select`/`epoll` — your platform has a primitive for waiting without burning CPU. Use it. The trade-off is that event-driven code is harder to reason about than a simple loop, but that's a small price for reclaiming a whole core.

## Lock Contention: When Threads Fight

Locks are supposed to protect data, but under load they become a CPU problem of their own. When threads contend for a lock, they either spin (burning CPU while waiting) or block (context-switching, which also costs CPU). Either way, you pay.

```java
// ❌ One global lock — every request serializes, threads queue up spinning
public class Counter {
    private static final Object LOCK = new Object();
    private int count;
    public synchronized void increment() { count++; } // everyone fights for this
}
```

```java
// ✅ Fine-grained locking (or a lock-free primitive) — contention drops
public class Counter {
    private final AtomicInteger count = new AtomicInteger();
    public void increment() { count.incrementAndGet(); } // CAS, no lock
}
```

The trade-off: fine-grained locking is more complex, and lock-free code is genuinely hard to get right. But if your profile shows threads stuck in `park`/`lock`/`spin` for a big chunk of time, contention is your CPU problem — and the fix is less locking, not faster code.

## Wasteful Work in Hot Paths

Some CPU sinks are just... unnecessary work you didn't notice. Three favorites:

- **Logging in hot loops** — string interpolation, timestamp formatting, and JSON serialization in a `logger.debug()` that's *enabled in production*. Check log levels before building the message.
- **Serializing the same thing twice** — deserializing a payload, then re-serializing it to forward it, when you could pass the raw bytes through.
- **Compression at the wrong layer** — compressing a 200-byte JSON response saves 40 bytes and costs 2ms of CPU per request. At high throughput, that's a real tax for no benefit. Measure whether compression actually helps before turning it on everywhere.

```go
// ❌ Building the log message even when debug is disabled
log.Debugf("order %s payload: %s", order.ID, prettyPrint(order)) // prettyPrint runs ALWAYS

// ✅ Guard it — zero cost when debug is off
if log.IsDebugEnabled() {
    log.Debugf("order %s payload: %s", order.ID, prettyPrint(order))
}
```

## When to Offload: The Queue Trade-off

Sometimes the right answer isn't "make it faster" — it's "don't do it in the request path at all." CPU-heavy work like image processing, PDF generation, or bulk computation is a classic candidate for a background queue:

- Request comes in → enqueue a job → respond "accepted" → worker processes it → webhook/status endpoint when done.

This smooths CPU spikes (workers process at a sustainable rate instead of all-at-once) and protects your API latency. But it's not free: you now own a queue, retry logic, idempotency, and a way for users to check job status. For a report that runs twice a day, a queue is overkill. For image thumbnails on every upload, it's the difference between a stable service and a melting one. Know which side you're on.

## Takeaways

- **Profile before you optimize.** Flamegraphs or nothing. Fix the widest frame first, re-profile after every change.
- **Kill algorithmic waste first** — N+1 queries, accidental O(n²), catastrophic regexes. They're the biggest wins with the smallest code changes.
- **Watch allocation pressure in managed languages** — GC time is CPU time. Target hot loops, not cold paths.
- **Never block the event loop** with synchronous CPU work; offload to workers or queues.
- **Don't poll what can notify you** — blocking waits are free, busy waits cost a core.
- **If threads are fighting, reduce locking instead of adding threads.**
- **Re-check hot paths for waste** — log guards, double serialization, pointless compression.

The pattern to remember: CPU problems are almost never "the machine is too slow." They're "we're doing too much work, or doing it in the wrong place." Find the excess work, and the CPU takes care of itself.
