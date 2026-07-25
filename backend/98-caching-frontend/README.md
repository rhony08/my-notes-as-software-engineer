# Browser Caching Strategies

You've optimized your images, split your bundles, and deferred your scripts. But every time a user visits your site, their browser downloads everything from scratch. That's frustrating—especially on a mobile connection or spotty Wi-Fi.

This is where browser caching comes in. Get it right, and repeat visits feel instant. Get it wrong, and users stare at a loading spinner—or worse, a broken page with stale assets.

---

## How Browser Caching Actually Works

When a browser requests a resource, it checks if it already has a valid cached copy. If it does, it skips the network request entirely (or does a lightweight validation). The server controls this behavior through HTTP headers.

Two headers run the show:

| Header | What it does |
|--------|-------------|
| `Cache-Control` | Tells the browser *how* and *how long* to cache |
| `ETag` | A fingerprint for the resource—browser asks "has this changed?" |

Let's break each one down.

---

## Cache-Control: The Big Boss

This header has a bunch of directives, but you'll really only use a handful:

```
Cache-Control: public, max-age=3600, immutable
```

The most important ones:

| Directive | Meaning |
|-----------|---------|
| `max-age=N` | Cache for N seconds, no server check needed |
| `no-cache` | Must check with server every time (can still cache) |
| `no-store` | Never cache anything (sensitive data) |
| `public` | CDNs and proxies can cache it too |
| `private` | Only the user's browser can cache it |
| `immutable` | Won't change during `max-age`—don't bother re-checking |

### Common Configuration Patterns

```
# Long-lived, fingerprintable assets (JS/CSS)
Cache-Control: public, max-age=31536000, immutable

# HTML pages that change (SPA)
Cache-Control: no-cache

# API responses with user-specific data
Cache-Control: private, max-age=60

# Never cache sensitive stuff
Cache-Control: no-store
```

### The Pitfall with `max-age`

Setting `max-age` too long is dangerous if you don't version your assets. If a user caches `app.js` for a year and you deploy a bugfix with the same filename... they're stuck with the old version.

**The fix is simple:** use content hashes in filenames.

```
// ❌ BAD: no versioning
app.js → app.js

// ✅ GOOD: content-hash in filename
app.a3b2c1.js → app.a3b2c1.js
```

When the content changes, so does the hash. The new URL forces a fresh download, and the old one expires naturally.

---

## ETag: The Validation Layer

ETags are a fallback for when `max-age` expires. Instead of downloading the full resource again, the browser sends:

```
If-None-Match: "abc123"
```

The server checks if the resource changed. If not, it responds with `304 Not Modified`—zero bytes, just headers. If it changed, the full response comes through.

```
// ❌ Full download every time
200 OK (2MB image)

// ✅ Validated request (if unchanged)
304 Not Modified (140 bytes of headers)
```

Trade-off: ETag validation still requires a round-trip to the server. It saves bandwidth but not latency. That's where `immutable` helps—no request at all until `max-age` expires.

---

## Cache Strategies by Asset Type

Different resources need different approaches:

| Asset | Strategy | Why |
|-------|----------|-----|
| **JS/CSS bundles** | `max-age=1yr, immutable` + content hash | These rarely change independently, and you deploy new versions |
| **Images** | `max-age=1yr, public` | Images don't change often, and you can rename if they do |
| **HTML pages** | `no-cache` or `max-age=0, must-revalidate` | HTML references other assets—you need fresh versions |
| **API responses** | `private, max-age=T` or `no-store` | Depends on sensitivity; user data should be `private` |
| **Fonts** | `public, max-age=1yr, immutable` | Fonts rarely change and serve as blocking resources |
| **Service Workers** | `max-age=0, no-cache` | You want immediate updates to registration logic |

### The Service Worker Gotcha

Service workers have a special caching layer that's independent of HTTP cache headers. If you set `max-age=31536000` on your service worker file, the browser won't check for updates for a year. That's probably not what you want.

```
// For your service worker file:
Cache-Control: max-age=0, no-cache, no-store, must-revalidate
```

---

## Cache Busting: When Things Go Stale

Even with proper headers, you'll run into caching issues. Here are the common ones and how to handle them.

**Problem:** User has an old CSS file cached, and the new HTML references classes that don't exist in the old CSS. Layout is broken.

**Solution:** Content hash in the filename (as mentioned above). When CSS changes, the URL changes. No conflict.

```
// Old version
<link href="/styles.abc123.css" rel="stylesheet">

// After deploy
<link href="/styles.def456.css" rel="stylesheet">
```

**Problem:** A third-party CDN has a long cache header and you pushed a critical bugfix.

**Solution:** Increment a version query parameter as a last resort:

```
https://cdn.example.com/widget.js?v=2
```

It's hacky, but it works. Better to use proper versioning from the start.

---

## Service Workers: Programmatic Caching

Service workers give you full control over caching behavior. You define the strategy in JavaScript:

```javascript
// Cache-first: serve from cache, fall back to network
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request);
    })
  );
});
```

Common service worker caching strategies:

| Strategy | Best for | Trade-off |
|----------|----------|-----------|
| **Cache First** | Static assets, images | May serve slightly stale content |
| **Network First** | API calls, HTML | Slower if network is available |
| **Stale While Revalidate** | Content that can be slightly stale | Best user experience, slightly complex |
| **Network Only** | Sensitive data | No offline support |

**Stale While Revalidate** is the sweet spot for most apps:

```javascript
// Serve cached version immediately, fetch update in background
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open('my-app-v1').then((cache) => {
      return cache.match(event.request).then((cached) => {
        const fetchPromise = fetch(event.request).then((response) => {
          cache.put(event.request, response.clone());
          return response;
        });
        return cached || fetchPromise;
      });
    })
  );
});
```

This gives users an instant load while ensuring content gets updated on the next visit.

---

## Service Worker vs HTTP Cache: They're Different Layers

A common misunderstanding: service worker caching replaces HTTP caching. It doesn't—they work together.

```
Browser → Service Worker Cache → HTTP Cache → Network
```

- **Service Worker cache:** First line of defense, offline support
- **HTTP Cache (browser):** Respects Cache-Control headers
- **Memory Cache:** In-memory, lives until tab closes

You need both. Don't think of them as alternatives.

---

## Measuring Your Cache Hit Rate

You can't improve what you don't measure. The browser's DevTools Network tab shows cache status:

| Status | Meaning |
|--------|---------|
| `200 (from disk cache)` | Served from browser cache, no network request |
| `200 (from memory cache)` | Served from in-memory cache (faster) |
| `304 Not Modified` | Validated with server, resource unchanged |
| `200` | Full network fetch |

You can also use the `Cache-Control` header and `Age` response header to understand caching behavior end-to-end:

```
# CDN response
cache-control: public, max-age=86400
age: 3600  # 1 hour old in the CDN cache
```

A good target: >90% cache hit rate for static assets. If you're below that, check your headers and versioning strategy.

---

## Takeaways

- **Use content hashes in filenames** for JS/CSS—it's the safest long-lived caching strategy
- **Set `max-age=31536000, immutable`** for versioned assets—no server round-trip needed
- **Use `no-cache` or `max-age=0`** for HTML to ensure fresh references
- **Don't set long `max-age` on service workers**—they'll never update
- **ETags are a backup**, not a primary strategy—they save bandwidth, not latency
- **Use service workers for offline support**, HTTP cache for standard performance
- **Monitor cache hit rates** in DevTools and aim for 90%+ on static assets

Browser caching isn't glamorous, but it's one of the highest-ROI optimizations you can make. A properly cached site loads from disk in milliseconds instead of waiting for the network. Your users—especially the ones on mobile—will thank you.
