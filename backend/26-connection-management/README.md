# HTTP/2 and Connection Management

You've probably noticed your APIs getting slower as traffic grows. Adding more servers helps, but sometimes the bottleneck isn't your code—it's the transport layer itself. Old HTTP/1.1 connections are wasteful. Each request needs its own round trip, and browsers limit concurrent connections per domain to 6. That's where HTTP/2 comes in.

In this article, we'll look at what HTTP/2 changes under the hood, how connection management differs, and what you need to know as a backend engineer.

---

## The HTTP/1.1 Problem

Back in the day, HTTP/1.1 gave us persistent connections (reuse the same TCP connection instead of opening a new one per request). That helped, but not enough:

```
HTTP/1.1 with 100 resources:
├── 6 parallel connections (browser limit)
├── Each connection: 1 request at a time
├── Head-of-line blocking: one slow resource blocks the rest
└── ~200ms per round trip × 17 sequential requests = ~3.4s just for handshakes
```

The workarounds were ugly:
- **Domain sharding** — serving assets from multiple domains just to get more connections
- **Image sprites** — combining dozens of images into one file
- **CSS/JS bundling** — mashing all scripts together to reduce requests

All of these are bandaids. HTTP/2 fixes the root cause.

---

## What HTTP/2 Changes

### Multiplexing (The Big One)

HTTP/2 lets multiple requests and responses share a single TCP connection. No more head-of-line blocking at the application layer.

```
HTTP/2 (single connection):
├── Request 1 (stream)
├── Request 2 (stream) ← sent at the same time
├── Request 3 (stream) ← sent at the same time
├── ... up to 100 concurrent streams
└── Responses arrive in any order, no blocking
```

Each request gets its own **stream** on the same connection. The server can process them in parallel and send responses back as they're ready. The client reassembles everything correctly.

### Stream Prioritization

Not all resources are equal. HTTP/2 lets the client tell the server what matters most:

```
Priority 0 (highest): HTML document
Priority 1: CSS (render-blocking)
Priority 2: JavaScript (interactivity)
Priority 9 (lowest): Analytics scripts, tracking pixels
```

The server uses these hints to allocate bandwidth wisely. If CSS arrives first, the browser can start rendering sooner.

### Header Compression (HPACK)

HTTP headers are repetitive. Every request sends cookies, user-agent, accept-encoding, etc. With HTTP/1.1, that's 500-800 bytes of overhead *per request*, uncompressed.

HTTP/2 compresses headers using HPACK:
- Builds a shared table of headers between client and server
- Static table: common headers (`:method`, `:path`, `content-type`, etc.)
- Dynamic table: headers you send frequently (your cookies, custom headers)
- After first request, subsequent requests can reference headers by index

```
Request 1: Full headers (800 bytes → 600 bytes compressed)
Request 2: Same headers? Just say "same as request 1" (50 bytes)
Request 3: Mostly the same? Just send the delta (30 bytes)
```

### Server Push

The server can send resources the client hasn't asked for yet:

```
Client: "I need /index.html"
Server: "Here's index.html... and also style.css and app.js (you'll need these)"
```

Sounds useful, right? In practice, it's [controversial](https://httpwg.org/specs/rfc7540.html). Browser caches already do a good job. Server push can waste bandwidth if the client already has the resource. Most experts recommend [using `103 Early Hints` instead](https://developer.chrome.com/blog/early-hints/).

```
// ❌ Server Push (controversial)
Server pushes style.css → Client already has it cached → wasted bandwidth

// ✅ 103 Early Hints
Server: "103 Early Hints — you'll want style.css and app.js"
Client: "Have 'em both, cache hit, carry on"
```

---

## Connection Management Changes for Backend Devs

### Your Server Needs to Support It

Most modern servers do:

| Server | HTTP/2 Support | Config |
|--------|---------------|--------|
| Nginx | ✅ Since 1.9.5 | `listen 443 ssl http2;` |
| Caddy | ✅ Default | Auto, no config needed |
| HAProxy | ✅ Since 2.0 | `alpn h2,http/1.1` |
| Node.js | ✅ Native (no http2 module) | `http2.createSecureServer()` |

### TLS Is Mandatory

HTTP/2 over cleartext (`h2c`) exists but almost nobody uses it. Browsers require TLS. You need:

```
// ❌ HTTP/2 without TLS? Not gonna work with browsers
// ✅ You need HTTPS
  
listen 443 ssl http2;
ssl_certificate /etc/ssl/certs/example.com.pem;
ssl_certificate_key /etc/ssl/private/example.com.key;
```

### ALPN Negotiation

Client and server negotiate protocol during the TLS handshake:

```
Client: "I support h2 and http/1.1"
Server: "Let's use h2 (HTTP/2)"
     ↳ One round trip handles both TLS and protocol negotiation
```

No extra latency. The handshake tells you which protocol you're using.

### Connection Coalescing

Since HTTP/2 multiplexes everything on one connection, your DNS setup matters:

```
// ❌ Multiple domains pointing to the same server
assets.example.com → 1.2.3.4
api.example.com    → 1.2.3.4
// Browser opens TWO connections (can't coalesce, different hostnames)

// ✅ Same domain or DNS aliases
assets.example.com CNAME → example.com
api.example.com CNAME    → example.com
// Browser opens ONE connection, all requests share it
```

To coalesce, the server's TLS cert must cover all hostnames (wildcard certs help here).

### Connection Reset Handling

HTTP/2 streams are lightweight. One failing request doesn't kill the connection:

```javascript
// HTTP/1.1: one slow request blocks the connection
const axios = require('axios');

async function fetchAll() {
  // ❌ If request #3 hangs, requests #4-#6 wait behind it
  for (const url of urls) {
    await axios.get(url); // sequential, head-of-line blocked
  }
}

// HTTP/2: multiplexed streams
const http2 = require('http2');
const client = http2.connect('https://api.example.com');

for (const url of urls) {
  // ✅ Each stream is independent
  const stream = client.request({ ':path': url });
  
  stream.on('error', (err) => {
    console.error(`Stream failed: ${url}`, err.message);
    // Other streams are NOT affected
  });
  
  stream.on('response', (headers) => {
    // ✅ Responses arrive in any order
    console.log(`${url}: ${headers[':status']}`);
  });
}
```

---

## What Doesn't Change

HTTP/2 doesn't magically fix everything:

- **Application-level head-of-line blocking** still exists (a slow database query is still slow)
- **TCP head-of-line blocking** still exists (HTTP/2 runs over TCP, and TCP loss affects all streams)
- **HTTP semantics don't change** — methods, status codes, headers, cookies all work the same
- **Caching, CORS, auth** — all unchanged

HTTP/3 (QUIC) solves the TCP HoL blocking by running over UDP instead. But that's a topic for another day.

---

## When You Should Care

| Situation | Impact |
|-----------|--------|
| Serving many small resources (SPA, images, API calls) | ✅ Big improvement — multiplexing shines |
| Single large download (video, binary file) | 😐 Minimal benefit — one stream per connection anyway |
| API with large headers (big JWTs, verbose cookies) | ✅ HPACK compression helps a lot |
| Your users are on high-latency networks (mobile) | ✅ Fewer round trips = faster |
| You already bundle everything into one request | 😕 You won't see much difference |

### What Breaks

```
❌ gRPC? ✓ Works great on HTTP/2
❌ WebSockets? ✓ Still work (upgraded separately)
❌ Server-sent events? ✓ Still work
❌ Long polling? ✓ Still works (but consider WebSockets instead)
```

---

## Practical Takeaways

1. **Enable HTTP/2 on your server** — it's a config change, not a code change. Nginx: add `http2` to your listen directive. Done.

2. **Stop domain sharding** — it was a hack for HTTP/1.1. HTTP/2 multiplexes everything on one connection. Serving assets from different domains just adds more connections without benefit.

3. **Undo over-bundling** — if you merged everything into one giant JS file to save requests, you can split it back. HTTP/2 handles many small requests efficiently. Use code splitting freely.

4. **Watch your connection pool** — HTTP/2 doesn't need 6+ connections. A single persistent connection is usually enough. Update your connection pool configuration (some libraries default to HTTP/1.1-style pooling).

5. **Monitor for TCP loss** — HTTP/2 is more sensitive to packet loss than HTTP/1.1 because one lost packet affects all streams. If you see TCP retransmits, HTTP/3 might be your next move.
