# Retry Patterns: Exponential Backoff

You've got a service calling another service. Maybe it's your API hitting a database, or your microservice talking to a downstream partner. Network hiccups happen. The database has a brief blip. The downstream service is throttling you for a second.

So you retry. Makes sense, right?

But here's the thing — **naive retries can make everything worse**. If all your clients retry at the same cadence, you get what's called the *thundering herd problem*: everyone slams the recovering service at once, taking it right back down.

## What Goes Wrong With Simple Retries

Say you're calling an API that's having a bad moment:

```python
# ❌ NEVER DO THIS - immediate retries amplify the problem
def fetch_data(url):
    response = requests.get(url)
    if response.status_code == 429 or response.status_code >= 500:
        response = requests.get(url)  # Hammer again immediately!
    return response.json()
```

If 100 instances of your service all hit the same failing endpoint and retry at the same time, the downstream gets 100x the traffic instead of 100. You've just turned a hiccup into an outage.

The fix? **Wait between retries. Wait longer each time.**

## Exponential Backoff — The Core Idea

Instead of retrying instantly, you wait. And every successive retry, you increase the wait. The classic formula:

```
delay = base_delay * (2 ^ attempt)
```

So with a 1-second base delay:

| Attempt | Wait Time |
|---------|-----------|
| 1       | 1 second  |
| 2       | 2 seconds |
| 3       | 4 seconds |
| 4       | 8 seconds |
| 5       | 16 seconds |

Each retry gives the system more time to recover. You're being polite instead of panicking.

```python
# ✅ Better: exponential backoff
import time

def fetch_with_backoff(url, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code < 500:
            return response.json()
        
        # Wait before retrying — exponentially
        delay = base_delay * (2 ** attempt)
        print(f"Attempt {attempt + 1} failed, retrying in {delay}s...")
        time.sleep(delay)
    
    raise Exception("Max retries exceeded")
```

### But Wait — There's a Problem

Imagine 50 services all using this exact same formula. Attempt 1 fails → everyone waits 1 second. Attempt 2 fails → everyone waits 2 seconds. Attempt 3 → 4 seconds... and then at the same time, they all retry simultaneously.

You've still got the thundering herd, just slightly delayed.

The fix: **add jitter**.

## Jitter: The Secret Sauce

Jitter introduces randomness to break the synchronization:

```python
import random

def fetch_with_jitter(url, max_retries=5, base_delay=1):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code < 500:
            return response.json()
        
        # Full jitter: random between 0 and the exponential delay
        delay = random.uniform(0, base_delay * (2 ** attempt))
        print(f"Retry {attempt + 1}, waiting {delay:.2f}s...")
        time.sleep(delay)
    
    raise Exception("Max retries exceeded")
```

This spreads out the retries. Instead of everyone hitting at the same time, they arrive like a gentle drizzle instead of a thunderstorm.

### Common Jitter Strategies

| Strategy | Formula | When to Use |
|----------|---------|-------------|
| **Full jitter** | `random(0, cap)` | General purpose — smooths load best |
| **Equal jitter** | `cap/2 + random(0, cap/2)` | When you want a minimum delay |
| **Decorrelated jitter** | `min(cap, random(base, previous * 3))` | Long-running retries with high variance |

**Stripe** uses full jitter. **Google** recommends it in their cloud client libraries. It's the safest default.

```python
# Full jitter implementation with cap
def retry_with_full_jitter(url, max_retries=5, base_delay=1, max_delay=60):
    for attempt in range(max_retries):
        response = requests.get(url)
        
        if response.status_code < 500:
            return response.json()
        
        # Cap the delay so we don't wait forever
        cap = min(max_delay, base_delay * (2 ** attempt))
        delay = random.uniform(0, cap)
        
        time.sleep(delay)
    
    raise Exception("Max retries exceeded")
```

## When NOT to Retry

Not every failure deserves a retry.

**Retry when:**
- You got a `429 Too Many Requests` (server says "back off")
- You got a `5xx` error (temporary server issue)
- Connection timed out (network hiccup)
- DNS resolution failed (transient)

**Don't retry when:**
- You got a `4xx` error (bad request, auth error — won't help)
- The response says "not found" (404 won't become 200)
- The server says "payment required" — that's not transient

```python
# ✅ Smart retry: only for transient failures
def smart_retry(url, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=5)
        except (requests.ConnectionError, requests.Timeout) as e:
            # Transient — worth retrying
            delay = random.uniform(0, 2 ** attempt)
            time.sleep(delay)
            continue
        
        if response.status_code == 429:
            # Rate limited — retry after suggested wait
            retry_after = int(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(retry_after)
            continue
        
        if response.status_code >= 500:
            delay = random.uniform(0, 2 ** attempt)
            time.sleep(delay)
            continue  # Server-side blip
        
        return response.json()
    
    raise Exception(f"Failed after {max_retries} retries")
```

## Trade-offs You Need to Know

**Exponential backoff isn't free:**

- **Latency**: Your request now takes potentially minutes instead of milliseconds. Design for that — don't call an API with retries inside a hot request path.
- **Resource usage**: Each retry keeps connections, memory, and CPU active. If you've got thousands of concurrent requests all retrying, that's a lot of sustained load.
- **Deadlines**: Retries don't help if the whole operation needs to complete in 100ms. Sometimes failing fast is better than retrying slowly.

```python
# ✅ Adding a deadline to bounded retries
def retry_with_deadline(url, deadline_ms=5000, max_retries=3):
    start = time.monotonic()
    for attempt in range(max_retries):
        elapsed = (time.monotonic() - start) * 1000
        if elapsed >= deadline_ms:
            break  # We're out of time
        
        response = requests.get(url)
        if response.status_code < 500:
            return response.json()
        
        delay = random.uniform(0, 2 ** attempt)
        if elapsed + (delay * 1000) >= deadline_ms:
            break
        time.sleep(delay)
    
    raise Exception("Failed within deadline")
```

## Real-World Headers to Watch

When dealing with APIs that rate-limit (OpenAI, GitHub, Stripe), they'll tell you what to do:

```
Retry-After: 120        # Wait this many seconds
x-ratelimit-remaining: 0 # You're out of quota
x-ratelimit-reset: 1685029200 # Unix timestamp when quota resets
```

Always respect `Retry-After`. It's the server telling you exactly how long to wait — no math needed.

```python
response = requests.get(url)
if response.status_code == 429:
    retry_after = int(response.headers.get("Retry-After", 5))
    time.sleep(retry_after)
```

## What To Take Away

- **Never retry without a delay** — you'll turn a blip into a blackout
- **Exponential + jitter** is the standard pattern for a reason
- **Cap your delays** — don't wait 128 seconds for a 500ms operation
- **Only retry transient failures** — 4xx errors aren't going away
- **Set a deadline** — retries shouldn't outlive your request's useful lifetime
- **Respect `Retry-After`** when the server tells you what it wants
- **Consider the alternative**: sometimes failing fast and showing a degraded response is better than hanging your user for 30 seconds
