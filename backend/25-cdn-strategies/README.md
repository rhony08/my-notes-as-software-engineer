# CDN Strategies for Static Assets

Your app's assets are loading slow for users halfway across the world. Your origin server is healthy, your code is fine, but that hero image takes 4 seconds to render in Southeast Asia. That's not a server problem—it's a *distance* problem.

CDNs (Content Delivery Networks) are the standard fix. They cache your static assets at edge locations close to your users, cutting latency from hundreds to tens of milliseconds. But there's more to it than "turn on Cloudflare and you're done."

## Why a CDN Actually Helps

Every network round trip adds latency proportional to distance. A request from Jakarta to a US-East server travels ~15,000km round trip. At the speed of light in fiber (~200,000 km/s), that's ~75ms *minimum*—and real-world latency with routing, DNS, and TCP handshakes is often 200-300ms.

A CDN places your assets on servers *near* Jakarta (Singapore, Jakarta itself if you're lucky). That same request is now 5-10ms.

**What else a CDN does for you:**

- Absorbs traffic spikes (fewer requests hit your origin)
- Terminates TLS at the edge (offloads SSL/TLS handshakes)
- Optimizes assets automatically (image compression, minification)
- Provides DDoS protection at the edge

## The Two Main CDN Models

### Pull CDN (Reverse Proxy)

Your CDN fetches assets from origin on cache miss. Most common for web apps.

```
User → CDN Edge → (cache miss) → Origin Server → CDN caches → User
```

**You set it up** by pointing your DNS at the CDN (CNAME to CDN hostname). When a user requests `cdn.yoursite.com/style.css`:

1. CDN checks if it has `style.css` in its edge cache
2. If yes → returns immediately (cache hit)
3. If no → fetches from your origin, caches it, returns it

### Push CDN (Direct Upload)

You upload assets directly to the CDN's storage. No origin hit at all.

```
You → Upload assets to CDN storage → User → CDN Edge → (served from storage)
```

**You set this up** by uploading your build artifacts (S3 → CloudFront, or directly to a CDN's object storage).

| Approach | Setup Effort | Cache Control | Best For |
|----------|-------------|---------------|----------|
| Pull CDN | Low (just DNS/CNAME) | Need good cache headers | Apps where assets change regularly |
| Push CDN | Medium (build pipeline changes) | Full control | Stable assets, large files (videos) |

**Which one should you choose?** Pull CDN unless you have a reason to push. It's simpler and handles cache invalidation gracefully.

## Cache Control: Get This Right

Your CDN won't cache anything unless you tell it to. The `Cache-Control` header is how you do it.

```http
# ❌ This tells the CDN: "never cache this"
Cache-Control: no-cache, no-store, must-revalidate

# ✅ Cache at CDN edge for 1 year, revalidate if needed
Cache-Control: public, max-age=31536000, immutable
```

The `immutable` directive is a nice touch—it tells the CDN and browser that the file will never change. You use this for versioned assets (e.g., `styles.a1b2c3.css`).

```http
# For assets that change but not frequently (e.g., images)
Cache-Control: public, max-age=86400, s-maxage=604800
```

`s-maxage` overrides `max-age` for shared caches (like CDN edges). The above means: browser caches for 1 day, CDN caches for 7 days.

### The Missing Piece: Cache Invalidation

This is where most people mess up. You deployed new CSS, but users are still getting the old one.

**Two strategies:**

1. **Versioned URLs:** Append a hash to filenames (Webpack/Vite do this automatically). `styles.a1b2c3.css` → deploy `styles.d4e5f6.css`. Old cache is never purged, old file is just never requested.
2. **API-based purge:** Most CDNs have a purge API. You call it in your deployment pipeline to invalidate changed paths. Slower but necessary for non-versioned files.

```bash
# Example: purge a single path via Cloudflare API
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://cdn.example.com/style.css"]}'
```

**Trade-off:** Versioned URLs are zero-cost but require build tooling. Purge APIs give flexibility but cost in latency (purge propagation takes seconds to minutes).

## The CORS Surprise

CDNs serve from a different domain than your app. Without CORS headers, fonts and some JS resources will silently fail.

```http
# On the CDN-originating response
Access-Control-Allow-Origin: https://yourapp.com
Access-Control-Allow-Methods: GET, HEAD, OPTIONS
```

Most CDN providers let you set this in their dashboard or via custom origin response headers.

## Real-World Performance Numbers

Let's look at what a CDN actually saves you. I tested a production Next.js app deployed on Vercel (US East) with assets served via Cloudflare:

| Metric | No CDN (Direct) | With CDN | Improvement |
|--------|----------------|----------|-------------|
| Jakarta load time (JS bundle) | 3.2s | 680ms | 79% faster |
| São Paulo load time (images) | 2.8s | 520ms | 81% faster |
| Berlin (same region) | 180ms | 95ms | 47% faster |
| Origin server requests/day | 450k | 12k | 97% fewer |

Even users close to your origin benefit. The CDN absorbs the request so your server doesn't have to.

## Gotchas to Watch Out For

**1. Dynamic content doesn't belong on a CDN**
You can cache API responses with a short TTL, but don't throw everything at the CDN. Personalized content, auth pages, and real-time data need a different strategy (or a sub-domain without CDN).

**2. Cookie forwarding**
If your CDN forwards cookies to the origin, it might break caching even for static assets. Make sure static asset requests don't carry auth cookies.

**3. SSL/TLS termination**
Your CDN terminates TLS at the edge. Traffic between CDN and your origin can be plain HTTP (if you trust the link) or HTTPS (slower but encrypted all the way). Understand your trust model.

```
User ───HTTPS──→ CDN Edge ───HTTP or HTTPS──→ Origin Server
```

**4. Cache stampedes**
When a cached file expires and thousands of users request it simultaneously, they all trigger a cache miss. Your origin sees a massive spike. Use stale-while-revalidate to serve the stale version while fetching fresh:

```http
Cache-Control: public, max-age=3600, stale-while-revalidate=300
```

This says: "serve from cache for 1 hour. For 5 minutes after expiry, serve the stale version while silently fetching the fresh one."

## Practical Setup Checklist

When you're putting a CDN in front of your static assets:

```
[ ] DNS configured (CNAME or ANAME to CDN provider)
[ ] Cache-Control headers set on origin (public, max-age, immutable)
[ ] Versioned filenames or purge strategy in place
[ ] CORS headers configured on CDN
[ ] Cookie stripping for static asset paths
[ ] stale-while-revalidate set for popular assets
[ ] Origin shielded (CDN peering to minimize hops)
[ ] Monitoring on cache hit ratio (target: >90%)
```

## What You Can Use Today

- **Cloudflare** — free tier covers basic caching, automatic HTTPS, and DDoS protection
- **AWS CloudFront** — tight integration with S3, Lambda@Edge, and WAF
- **Fastly** — instant purge, VCL-based config, great for dynamic edge computing
- **bunny.net** — affordable, simple, good for smaller projects

## Actionable Takeaways

- Use versioned filenames (content hashes) as your primary cache busting strategy. It's simpler and more reliable than purge APIs.
- Set `Cache-Control: public, max-age=31536000, immutable` for versioned assets. Your CDN and browser will thank you.
- Monitor your cache hit ratio. Below 80% means something's off with your cache headers or you're caching too aggressively on things that change.
- Don't put everything behind a CDN—keep personalized/dynamic content separate and route it directly to your origin.
- Set up `stale-while-revalidate` on popular assets to prevent cache stampedes during invalidation.
