# Memory Optimization in Backend

Your service is running fine for two weeks straight. Then one night, the container gets OOM-killed. It restarts, fills up again, gets killed again. You're in a crash loop, the on-call phone won't stop buzzing, and nobody knows why memory usage only ever goes up.

That's the thing about memory problems — they're silent until they're catastrophic. A slow query degrades gracefully. A memory leak takes the whole process down. And unlike CPU, which fixes itself once the work is done, memory keeps everything it touches until something explicitly lets it go. This article is about finding and fixing the patterns that eat memory in backend services.

## The First Mistake: Only Watching Heap Usage

Before optimizing anything, you need to know *what* you're measuring. Most people watch one metric and call it a day. That's how you end up "fixing" a problem that isn't there.

| Metric | What it actually is | Why it lies |
|---|---|---|
| **RSS** (resident set size) | Physical memory the OS gave the process | Includes code, stack, off-heap buffers, GC arenas — not just your objects |
| **Heap** | Memory managed by the runtime (JVM, V8, Go) | Doesn't include direct/Native buffers, mmap'd files, thread stacks |
| **Container limit** | What the kernel will kill you at | Doesn't tell you *what* inside is using it |

Here's the classic trap: you set the JVM heap to 2GB, the container limit to 2GB, and wonder why you get OOM-killed at 1.8GB of heap. The JVM needs headroom for native memory — metaspace, thread stacks, JIT, direct buffers. Your heap limit and your container limit are two different budgets, and both count.

```bash
# See the actual resident memory, not just heap
ps -o pid,rss,vsz,cmd -p $(pgrep -f my-service)

# JVM: check native + heap breakdown
jcmd <pid> VM.native_memory summary
```

## The Usual Suspects

In my experience, backend memory blowups come from a short list of patterns. Nine times out of ten, it's one of these:

1. **Unbounded caches** — "let's cache this, it'll be faster!" with no eviction policy and no size cap
2. **Loading everything into memory** — `SELECT * FROM orders` with no pagination, or reading a 500MB file into a string
3. **Accidental retention** — objects kept alive by static collections, event listeners, or closures long after they're "done"
4. **Off-heap / native leaks** — direct byte buffers, unclosed file handles, native libs (Node's Buffer, Java's `DirectByteBuffer`)
5. **String bloat** — building huge strings in loops, storing full payloads in logs

Let's go through the fixes that actually matter.

## Fix 1: Bound Every Cache

Caches are the #1 memory killer because they feel free. "I'll just memoize this function call" — and suddenly every unique request payload is parked in memory forever.

```go
// ❌ Unbounded: this map grows forever, one entry per unique key
var cache = map[string]Result{}

func Get(id string) Result {
    if r, ok := cache[id]; ok {
        return r
    }
    r := expensiveLookup(id)
    cache[id] = r // never evicted. ever.
    return r
}
```

The fix isn't "don't cache" — it's "bound the cache." Every cache needs three answers: max size, eviction policy, and TTL.

```go
// ✅ Bounded LRU with TTL — size capped, oldest entries evicted
cache, _ := lru.NewWithEvict(1000, func(key, value interface{}) {
    log.Printf("evicting %v", key) // visibility into churn
})
```

A good rule of thumb: if you can't name the max size of a cache in a code review, that cache is a memory leak with extra steps.

## Fix 2: Stream Instead of Loading

Loading an entire dataset into memory to process 1% of it is the most expensive way to do almost anything.

```python
# ❌ Pulls every row into memory, then keeps the whole list alive
all_orders = db.execute("SELECT * FROM orders WHERE status = 'pending'")
for order in all_orders:
    process(order)
```

```python
# ✅ Server-side cursor: one row at a time, memory stays flat
with db.execute("SELECT * FROM orders WHERE status = 'pending'") as cursor:
    for order in cursor:   # streamed, not materialized
        process(order)
```

Same idea applies to files:

```python
# ❌ 2GB log file → 2GB string in memory
content = open("huge.log").read()
for line in content.splitlines():
    ...

# ✅ Read line by line — memory stays at ~one line
with open("huge.log") as f:
    for line in f:
        ...
```

The pattern is always the same: **process in chunks, hold one at a time.** Your memory footprint goes from O(dataset) to O(chunk).

## Fix 3: Kill the Accidental Retention

The sneakiest leaks aren't allocations — they're objects the GC *could* collect but can't, because something still references them.

```java
// ❌ Static list grows forever — every request adds an entry
public class RequestLogger {
    private static final List<Request> history = new ArrayList<>();

    public static void log(Request r) {
        history.add(r); // static = never GC'd, app lifetime
    }
}
```

```java
// ✅ Bounded ring buffer: keeps last N, drops the rest
public class RequestLogger {
    private static final ArrayDeque<Request> history = new ArrayDeque<>(1000);

    public static void log(Request r) {
        synchronized (history) {
            if (history.size() >= 1000) history.pollFirst();
            history.add(r);
        }
    }
}
```

Another classic: event listeners and subscriptions that are never removed.

```javascript
// ❌ Every connection registers a listener, never unregisters
function handleConnection(socket) {
    emitter.on('order-updated', (order) => {
        socket.send(order); // closure keeps socket (and its buffers) alive forever
    });
}

// ✅ Track and clean up on close
function handleConnection(socket) {
    const handler = (order) => socket.send(order);
    emitter.on('order-updated', handler);
    socket.on('close', () => emitter.off('order-updated', handler));
}
```

If you register a listener, you own its lifecycle. "It's just one listener" becomes 50,000 one listeners by next month.

## Fix 4: Reuse Objects — But Only Where It Pays

Object pooling is a love-hate topic. Pools reduce allocation pressure and GC churn, but they add complexity: borrow/return discipline, pool sizing, and a whole new class of bugs when someone forgets to return an object.

| Approach | Best for | Costs |
|---|---|---|
| Object pooling | High-frequency, short-lived, expensive-to-create objects (DB connections, byte buffers) | Borrow/return discipline, leak risk if not returned |
| Flyweight | Many objects sharing the same heavy internal state | Shared state must be immutable |
| Just allocate | Most other cases — modern GCs handle short-lived objects cheaply | More GC pressure at very high allocation rates |

For byte buffers in particular, pooling is often worth it:

```java
// ❌ Allocating a 1MB buffer per message in a high-throughput path
byte[] buf = new byte[1024 * 1024];

// ✅ Reuse from a pool (e.g., Netty's ByteBuf allocator or a simple pool)
ByteBuf buf = pool.acquire();
try {
    // ... read/write ...
} finally {
    pool.release(buf); // ALWAYS return, even on exceptions
}
```

The rule: pool things that are **expensive to create and cheap to hold**. Don't pool a 16-byte `Point` object — you'll spend more time managing the pool than you save.

## GC Tuning: The Last Lever, Not the First

When memory problems appear, the instinct is to tweak GC flags. Resist it. GC tuning is what you do *after* you've fixed the leaks and bounded the caches — it tunes behavior, it doesn't fix bugs.

That said, knowing the trade-off matters:

| Goal | Tuning | Trade-off |
|---|---|---|
| Lower latency (fewer pauses) | Larger heap, concurrent GC (ZGC, G1) | More memory reserved, more CPU overhead |
| Lower memory footprint | Smaller heap, aggressive collection | More frequent GC, higher CPU, possible pauses |
| Higher throughput | Larger heap, fewer collections | Longer pauses when GC does run |

And whatever you do, **set container-aware flags** — a JVM that thinks it has 32GB when the container allows 2GB will size its heap accordingly and get OOM-killed:

```bash
# ✅ Tell the JVM about the container limit
java -XX:MaxRAMPercentage=75.0 -jar app.jar

# ❌ Fixed heap that ignores the actual container limit
java -Xmx4g -jar app.jar   # 4GB heap in a 2GB container = guaranteed OOM
```

Leave 20-25% of the container limit for native memory. Heap ≠ total.

## What Actually Breaks Without This

- **Crash loops**: OOM-kill → restart → refill → OOM-kill. Your service is down more than it's up, and Kubernetes keeps "fixing" it by restarting it into the same death.
- **GC death spirals**: memory near the limit → GC runs constantly → CPU spikes → requests slow → more objects pile up → GC runs harder. Throughput collapses before the kill even happens.
- **Cloud bills**: memory is the most expensive resource per GB on most cloud providers. A service holding 8GB when it needs 1GB isn't "being safe" — it's paying for 7GB of waste, every month, forever.
- **Noisy neighbors**: on shared instances, your ballooning RSS can trigger the kernel's OOM killer to pick *another* innocent process.

## Takeaways

- Watch RSS **and** heap — they're different budgets, and containers kill you on the first one.
- Every cache needs a max size, an eviction policy, and a TTL. If you can't name all three, it's a leak.
- Stream large datasets instead of materializing them — memory should scale with chunk size, not dataset size.
- Leaks are usually *retention*, not allocation: static collections and unregistered listeners are the classics. Clean up what you register.
- Pool only expensive-to-create objects, and always return them in `finally`.
- Tune GC flags last, and always size the heap relative to the container (`MaxRAMPercentage`), not a hardcoded `-Xmx`.
- When memory climbs and never comes back down, grab a heap dump *before* restarting — the restart destroys your only evidence.
