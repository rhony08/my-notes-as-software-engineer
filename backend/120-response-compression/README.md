# Response Compression Strategies

Your API returns JSON, and it's slow. Not slow-slow, but you can feel it — the network tab shows a 400KB payload taking 800ms on a decent connection. Your server CPU is at 5%. Your database is happy. The bottleneck isn't your code, it's the wire. Every byte you ship has to travel through the user's ISP, across whatever mobile network they're on, through a VPN, into a coffee shop's Wi-Fi that's somehow slower than dial-up.

Compression is the cheapest performance win in existence. One header, sometimes zero code changes, and your payloads shrink by 70-90%. Yet tons of backends skip it entirely, or enable gzip and call it a day without realizing they're leaving 30% more on the table. Let's look at how HTTP compression actually works, when to use gzip vs Brotli vs zstd, and the traps that'll make your compression pointless (or worse, slower).

## How Compression Fits Into HTTP

The flow is dead simple. The client says what it can decode, the server picks something and compresses, the client decompresses. Nothing about it changes your application logic.

```
Client                              Server
  │  GET /api/users                   │
  │  Accept-Encoding: gzip, br, zstd  │
  │──────────────────────────────────▶│
  │                                   │  compress with Brotli (best match)
  │  200 OK                           │
  │  Content-Encoding: br             │
  │  Content-Length: 38,412           │
  │◀──────────────────────────────────│
  │  decompress, render               │
```

Three headers matter:

- **`Accept-Encoding`** — sent by the client, listing what it supports, usually with quality values: `gzip, deflate, br, zstd`
- **`Content-Encoding`** — sent by the server, telling the client what was actually used
- **`Vary: Accept-Encoding`** — tells caches "the response varies based on this header, so don't serve the gzip version to a client that asked for br"

The third one is the one everyone forgets, and it causes the classic "compression works locally but the CDN serves raw JSON" mystery. More on that below.

## Which Algorithm Should You Use?

The short answer in 2026: **Brotli for text, zstd if both sides support it, gzip as the floor.** But let's look at the actual trade-offs, because "best compression ratio" isn't the whole story.

| Algorithm | Compression ratio (text/JSON) | Speed (compress) | Speed (decompress) | Browser support |
|-----------|------------------------------|------------------|--------------------|-----------------|
| gzip (level 6) | ~3-4x | Fast | Very fast | Everywhere, forever |
| Brotli (level 5) | ~4-5x | Medium | Fast | All modern browsers, HTTPS-only in some old setups |
| zstd | ~4-5x | Very fast | Extremely fast | Growing; Chrome/Firefox/Safari 18+, not universal yet |

A few things people get wrong here:

**Brotli isn't automatically better.** It depends heavily on the compression level. `br -q 1` compresses fast but barely beats gzip. `br -q 11` gets the best ratio but can be 10x slower to compress than gzip — fine for static assets you compress once, terrible for compressing every API response on the fly.

**gzip level 9 is rarely worth it.** Going from level 6 to 9 buys you maybe 2-3% more compression for several times the CPU cost. For dynamic responses, level 4-6 is the sweet spot. The marginal bytes aren't worth the latency spike under load.

```nginx
# ❌ Maximum compression everywhere — burns CPU on every request
gzip_comp_level 9;

# ✅ Pragmatic: high ratio for static, cheap for dynamic
gzip_comp_level 5;
gzip_static on;  # serve pre-compressed .gz files for static assets
```

**zstd is the dark horse.** It compresses at speeds that make gzip look slow while hitting Brotli-level ratios. If you control both ends of the connection (internal services, your own mobile app, a BFF talking to your backend), zstd is genuinely the best default. The catch is browser support isn't 100% yet, so you can't *only* serve zstd to the public.

```nginx
# Serve zstd when the client asks for it, fall back to gzip
gzip on;
gzip_types application/json text/css application/javascript;

# nginx 1.26+ with zstd module
zstd on;
zstd_comp_level 3;
```

## Compress Everything? No. Here's What NOT to Compress

Compression isn't free. It costs CPU, and for data that's already compressed, it *adds* size and wastes time. The classic mistake:

```nginx
# ❌ Compressing binary formats that are already compressed
gzip_types application/json text/html image/jpeg image/png image/webp video/mp4;
```

JPEG, PNG, WebP, AVIF, MP4, ZIP files — they're already compressed internally. Running gzip on them gives you nothing (often a *bigger* file due to header overhead) while burning CPU on every single request. Serving a 2MB JPEG through gzip can actually slow things down.

**The "compressible" list is basically text:**

- JSON, XML, CSV — compress fantastically (JSON is repetitive as hell, which is exactly what compression loves)
- HTML, CSS, JavaScript
- SVG (it's XML, so yes)
- Logs, if you're serving them

Rule of thumb: if you can open it in a text editor and read it, compress it. If it's a binary format, skip it.

## The Static vs Dynamic Split

This is where you get the biggest win-to-effort ratio. Most backends serve a mix: static assets (JS bundles, CSS, images, fonts) and dynamic responses (API JSON, rendered HTML). They need different compression strategies.

**Static assets: pre-compress at build time.** You compress once, store the compressed file, and serve it with zero runtime CPU cost. This is what `gzip_static` does — nginx checks for `foo.js.gz` next to `foo.js` and serves it directly.

```bash
# Build step: pre-compress with the best ratio, since it only happens once
brotli -q 11 -f dist/js/app.js -o dist/js/app.js.br
gzip -9 -k dist/js/app.js
```

You can also let your bundler do it. Vite, webpack, and esbuild all have plugins that emit `.gz` and `.br` variants at build time. Then your web server or CDN picks the best one based on `Accept-Encoding`.

**Dynamic responses: compress on the fly, but cheaply.** For API JSON, use a low/medium level. The CPU cost of Brotli level 4-5 or gzip level 5 on a few KB of JSON is negligible; the bandwidth savings are huge. And here's a nice trick — if your JSON is big and you control the API consumers, consider zstd for internal endpoints.

## The CDN Gotcha: Vary: Accept-Encoding

You enable compression, it works locally, then you put a CDN in front and suddenly some users get uncompressed responses. Or worse: a client that doesn't support Brotli gets served a Brotli response and the page breaks.

This is the `Vary` header problem. A CDN caches one version of `/api/users` and serves it to everyone unless you tell it the response depends on request headers. `Accept-Encoding` is exactly such a header.

```http
# ❌ CDN caches the first response (say, gzip) and serves it to everyone
HTTP/1.1 200 OK
Content-Type: application/json
Content-Encoding: gzip

# ✅ Cache knows: br clients get br, gzip clients get gzip
HTTP/1.1 200 OK
Content-Type: application/json
Content-Encoding: br
Vary: Accept-Encoding
```

Without `Vary: Accept-Encoding`, the CDN might serve a compressed response to a client that never asked for it (and can't decode it). The fix is one line, but it's the most commonly missing header in production APIs. Same story if you ever do per-device-type responses (`Vary: User-Agent`) — anything that changes the response based on a request header needs to be in `Vary`.

## Compression Level Micro-Benchmarks

Let's make this concrete. Take a 200KB JSON response (a realistic "list of orders" payload). Rough numbers on a modern server:

| Setup | Payload size | Time to compress | Notes |
|-------|-------------|------------------|-------|
| No compression | 200 KB | 0 ms | 200KB over the wire |
| gzip level 6 | 38 KB | ~2 ms | The safe default |
| Brotli level 5 | 31 KB | ~6 ms | Best ratio per CPU spent |
| Brotli level 11 | 28 KB | ~90 ms | Only for pre-compressed static |
| zstd level 3 | 32 KB | ~1.5 ms | Fastest, close to Brotli ratio |

For a dynamic API, 6ms of CPU to save 170KB of transfer is a *great* trade. For a high-throughput endpoint (10k req/s), 6ms each starts to matter — that's 60 seconds of CPU per second of wall time across your fleet. That's when you drop to zstd level 3 or gzip level 4, or move compression to the edge/CDN where it's someone else's CPU.

## When Compression Bites You

Honest list of downsides, because nothing's free:

- **CPU on the server.** Every compressed response costs compute. Under massive load, you can trade bandwidth problems for CPU problems.
- **Latency for tiny payloads.** Compressing a 200-byte response adds more time than it saves. Most servers handle this by only compressing above a size threshold (`gzip_min_length 1024` or similar).
- **Breaks your manual debugging.** You can't just `curl` an endpoint and read the body anymore — you need `curl --compressed` or `Accept-Encoding: identity`. A minor annoyance, but real.
- **CDN + Vary cache fragmentation.** With `Vary: Accept-Encoding`, your CDN stores multiple copies of each response (one per encoding). Slightly more cache storage, usually irrelevant, but it exists.
- **Timing attacks.** Compression ratio leaks information (that's the basis of BREACH attacks on secrets in HTML). If you're embedding tokens in responses, be aware that response size now correlates with content. For APIs returning JSON, this is mostly a theoretical concern — for HTML pages with CSRF tokens, it's worth thinking about.

## A Practical Checklist

Here's what a sane setup looks like, roughly in order of effort:

1. **Enable gzip at level 5-6** on your reverse proxy for text types, with `gzip_min_length` set. One config block, immediate win.
2. **Add `Vary: Accept-Encoding`** to every response your CDN caches. Do this the same day.
3. **Pre-compress static assets** at build time (`.br` and `.gz` variants) and serve them with `gzip_static`/equivalent.
4. **Add Brotli** at level 5 for dynamic responses once you've confirmed your proxy supports it (nginx, Caddy, and most CDNs do).
5. **Use zstd for internal service-to-service traffic** where you control both ends.
6. **Measure before tuning levels.** Check the actual payload sizes in your access logs or a proxy. If your biggest JSON is 5KB, stop optimizing compression and go do something more useful.

## Takeaways

- Compression is the cheapest perf win: one header, 70-90% smaller payloads for text.
- Brotli beats gzip on ratio; zstd beats both on speed. gzip remains the universal floor.
- Compress text, never already-compressed binaries like JPEG/WebP/MP4.
- Static = pre-compress at build time with max level. Dynamic = compress on the fly at a low level.
- `Vary: Accept-Encoding` is non-negotiable if a CDN sits in front of you.
- Set a minimum size threshold — compressing tiny responses is pure overhead.
