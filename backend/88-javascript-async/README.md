# JavaScript Async Patterns

Everything in JavaScript runs on a single thread. That's not a limitation — it's the fundamental constraint that shapes how we write async code. If you're coming from a language with real threads or goroutines, JS async patterns will feel weird at first. But once you understand the event loop, it clicks.

## The Problem: Single Thread, Many Things to Do

Imagine you're cooking. You've only got one burner, but you need to boil water, chop veggies, and check the oven. A multi-threaded language would give you more burners. JavaScript gives you a smarter way to juggle: start the water boiling, then chop while it heats up.

This is the non-blocking I/O model. Instead of waiting, JS says "okay I'll come back to that later" and keeps working.

```
// ❌ This blocks the entire thread
const data = readFileSync('big-file.json')
console.log(data) // nothing happens until file is fully read

// ✅ This doesn't block
readFile('big-file.json', (err, data) => {
  console.log(data)
})
console.log('This prints first')
```

## Callbacks: The Original Pattern

Callbacks are the oldest async mechanism in JS. You pass a function that gets called when the operation completes.

```javascript
// Classic callback pattern
function getUser(id, callback) {
  setTimeout(() => {
    callback(null, { id, name: 'Alice' })
  }, 100)
}

function getPosts(userId, callback) {
  setTimeout(() => {
    callback(null, ['Post 1', 'Post 2'])
  }, 100)
}

// This works, but...
getUser(1, (err, user) => {
  if (err) return console.error(err)
  getPosts(user.id, (err, posts) => {
    if (err) return console.error(err)
    console.log(user.name, posts)
  })
})
```

**The problem?** Nesting. Add three more levels and you've got callback hell — that infamous pyramid shape where code indents so far right it falls off the screen. Error handling gets messy too. Every callback needs its own `if (err)` check.

Callbacks are fine for simple cases but break down fast with any real logic.

## Promises: Flattening the Pyramid

Promises give us a way to chain async operations instead of nesting them.

```javascript
function getUser(id) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve({ id, name: 'Alice' })
    }, 100)
  })
}

function getPosts(userId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(['Post 1', 'Post 2'])
    }, 100)
  })
}

// Flat chain instead of nesting
getUser(1)
  .then(user => getPosts(user.id))
  .then(posts => console.log(posts))
  .catch(err => console.error(err))
```

The key improvement: error handling is centralized in one `.catch()`. No more scattered `if (err)` checks. And chaining keeps things flat.

### Promise Pitfalls to Watch For

Promises aren't magic. People still create callback hell — they just use `.then()` versions of it:

```javascript
// ❌ This is still callback hell, just with more syntax
getUser(1)
  .then(user => {
    getPosts(user.id).then(posts => {
      getComments(posts[0]).then(comments => {
        console.log(comments)
      })
    })
  })

// ✅ Return the promise and keep it flat
getUser(1)
  .then(user => getPosts(user.id))
  .then(posts => getComments(posts[0]))
  .then(comments => console.log(comments))
```

**Trade-off:** Promises add memory overhead compared to callbacks. For most apps it doesn't matter, but in resource-constrained environments (IoT, browser extensions) it's worth knowing.

## Async/Await: Synchronous-Looking Async

Async/await is syntactic sugar over promises. It lets you write async code that *looks* synchronous — which makes it way easier to reason about.

```javascript
async function loadUserData(userId) {
  try {
    const user = await getUser(userId)
    const posts = await getPosts(user.id)
    return { user, posts }
  } catch (err) {
    console.error('Failed to load user data:', err)
    throw err // re-throw so caller can handle too
  }
}

// Usage
const data = await loadUserData(1)
```

### The Golden Rule: Await in Sequence or Parallel?

This is where most people trip up. `await` pauses execution. If you have independent async operations, don't wait for one before starting the other.

```javascript
// ❌ Sequential — slow! Posts and comments are independent
const posts = await getPosts(userId)
const comments = await getComments(postId)

// ✅ Parallel — both start at the same time
const [posts, comments] = await Promise.all([
  getPosts(userId),
  getComments(postId),
])
```

**When to use each:**

| Pattern | Use Case | Time | 
|---------|----------|------|
| Sequential `await` | Operations depend on each other | Sum of all operations |
| `Promise.all` | Independent operations | Slowest operation only |
| `Promise.allSettled` | Need all results even if some fail | Slowest operation only |
| `Promise.race` | Need fastest result (timeouts) | Fastest operation |

### Common Async/Await Mistakes

**1. Forgetting error handling**

```javascript
// ❌ Unhandled promise rejection hiding in plain sight
app.get('/user/:id', async (req, res) => {
  const user = await getUser(req.params.id)
  res.json(user)
})
// If getUser throws, Express doesn't catch it. The promise rejects silently.

// ✅ Always wrap in try/catch or use a wrapper
app.get('/user/:id', async (req, res, next) => {
  try {
    const user = await getUser(req.params.id)
    res.json(user)
  } catch (err) {
    next(err) // pass to Express error handler
  }
})
```

**2. Sequential when you should parallel**

```javascript
// ❌ Sequential — 300ms total
const a = await fetch('/api/a')
const b = await fetch('/api/b')
const c = await fetch('/api/c')

// ✅ Parallel — ~100ms total
const [a, b, c] = await Promise.all([
  fetch('/api/a'),
  fetch('/api/b'),
  fetch('/api/c'),
])
```

**Trade-off:** `Promise.all` fails fast — if one promise rejects, everything fails. That's often what you want, but sometimes you want the other promises to still resolve. That's when you reach for `Promise.allSettled`.

```javascript
const results = await Promise.allSettled([
  fetch('/api/users'),
  fetch('/api/posts'),  // might fail
  fetch('/api/comments'),
])

results.forEach(result => {
  if (result.status === 'fulfilled') {
    console.log('Success:', result.value)
  } else {
    console.log('Failed:', result.reason)
    // Keep going — we still have users and comments
  }
})
```

## The Event Loop: What's Actually Happening

Understanding the event loop helps you predict *when* your async code runs. Here's the mental model:

```
┌──────────────────────────────┐
│         Call Stack           │ ← Where your synchronous code runs
├──────────────────────────────┤
│         Microtask Queue      │ ← Promise callbacks, queueMicrotask
├──────────────────────────────┤
│         Task Queue           │ ← setTimeout, setInterval, I/O
└──────────────────────────────┘
```

The event loop checks: Is the call stack empty? If so, drain the microtask queue first, then run one task from the task queue. Repeat.

This matters because microtasks (promises) run *before* tasks (timeouts):

```javascript
console.log('1')                   // sync

setTimeout(() => console.log('2'), 0)  // task queue

Promise.resolve().then(() => console.log('3'))  // microtask queue

console.log('4')                   // sync

// Output: 1, 4, 3, 2
```

Why this order? Promises are microtasks. They're higher priority than `setTimeout` callbacks. So even though both are "async," promises resolve before timers.

## When Things Get Nasty: Starving the Event Loop

Here's a real problem: blocking the event loop with heavy synchronous work.

```javascript
// ❌ This freezes your entire app for seconds
function processHugeDataSet(items) {
  for (const item of items) {
    doExpensiveSyncWork(item)  // blocks everything — no UI updates, no API responses
  }
}

// ✅ Break it up with setTimeout or schedule microtasks
function processHugeDataSet(items, index = 0) {
  if (index >= items.length) return
  
  doExpensiveSyncWork(items[index])
  
  // Yield to the event loop, then continue
  setTimeout(() => processHugeDataSet(items, index + 1), 0)
}
```

Or better, offload heavy work to Web Workers (browser) or worker threads (Node.js):

```javascript
// Node.js worker thread
const { Worker } = require('worker_threads')

function processInBackground(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./processor.js', { workerData: data })
    worker.on('message', resolve)
    worker.on('error', reject)
  })
}
```

## Patterns Worth Knowing

### Async Generator Pattern

For streaming data or paginated APIs:

```javascript
async function* paginate(url, maxPages = 10) {
  let page = 1
  while (page <= maxPages) {
    const response = await fetch(`${url}?page=${page}`)
    const data = await response.json()
    yield data
    if (!data.hasMore) break
    page++
  }
}

// Usage
for await (const page of paginate('/api/users')) {
  renderPage(page)
}
```

### Timeout Wrapper

Async operations can hang forever. Don't let them:

```javascript
function withTimeout(promise, ms = 5000) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  })
  return Promise.race([promise, timeout])
}

// Usage
try {
  const data = await withTimeout(fetch('/api/slow'), 3000)
} catch (err) {
  console.error('Request timed out or failed:', err.message)
}
```

### Retry with Backoff

Because networks fail. Retry is your friend:

```javascript
async function fetchWithRetry(url, options = {}) {
  const { retries = 3, baseDelay = 1000 } = options
  
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const response = await fetch(url)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      return await response.json()
    } catch (err) {
      if (attempt === retries) throw err
      const delay = baseDelay * Math.pow(2, attempt - 1)
      console.warn(`Attempt ${attempt} failed, retrying in ${delay}ms...`)
      await new Promise(r => setTimeout(r, delay))
    }
  }
}
```

## Takeaways

- **Callbacks** are fine for simple cases, but Promises or async/await scale better
- **Promise.all** for parallel ops, **sequential await** for dependent ones — know the difference
- **Always handle promise rejections** — unhandled rejections crash Node.js processes
- **The event loop** runs microtasks (promises) before tasks (timeouts) — this affects execution order
- **Don't starve the event loop** — heavy sync work blocks everything, break it up or use workers
- **`Promise.allSettled`** exists for when you want results regardless of failures
- **Timing out and retrying** async operations should be part of your standard toolkit, not an afterthought
