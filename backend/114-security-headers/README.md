# Security Headers Every App Needs

You deploy a new endpoint, and your security checklist says "add headers." You paste a few lines into the middleware, check the box, move on. Then six months later someone runs your site through a scanner and you find out you shipped with `X-Frame-Options` missing, your HSTS is `max-age=0`, and your session cookies aren't even `Secure`. Headers feel like a formality — until they're the only thing standing between your users' sessions and a clickjacking attack.

The good news: this is one of the cheapest security fixes in existence. No infra changes, no new services, just a few response headers. Let's go through the ones that actually matter, what each one does, and the trade-offs nobody mentions.

## The Headers That Matter

### 1. `Content-Security-Policy` — the big one

I covered CSP in detail in the frontend notes, so here's the short version: CSP tells the browser what's allowed to load on your page — scripts, styles, images, frames. It's your defense against XSS even when your escaping fails.

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; frame-ancestors 'none'
```

The most common real-world mistake is going so strict that you break your own site, then someone "fixes" it with `script-src 'unsafe-inline' 'unsafe-eval' *` — at which point you might as well not have CSP at all. Start strict, use `Content-Security-Policy-Report-Only` in production to collect violations before you enforce:

```http
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

### 2. `Strict-Transport-Security` (HSTS)

Tells the browser: "only ever talk to me over HTTPS, for the next N seconds." This kills the downgrade attack where an attacker on the same Wi-Fi intercepts the first HTTP request and redirects you to a fake site.

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**The trade-off nobody mentions:** once a browser sees HSTS, it *will not* let you serve HTTP for the duration of `max-age`. If you set `max-age=31536000` and then need to serve something over plain HTTP (a captive portal, an old device, a staging environment on the same domain), you're locked out for a year. That's why the safe rollout is: start with a small `max-age` (like 300), verify everything works, then crank it up.

And `preload` is even more permanent — it ships your domain in the browser's built-in HSTS list and you can't easily remove it. Only add `preload` when you're *sure*.

### 3. `X-Content-Type-Options: nosniff`

Browsers sometimes guess the content type of a response if the `Content-Type` header is missing or ambiguous. If you serve a user-uploaded file as `text/html` but the browser decides to sniff it as HTML anyway... you've just served stored XSS on a silver platter. This header says "don't guess, use what I declared."

```http
X-Content-Type-Options: nosniff
```

There is no downside to this header. Ship it everywhere, including API responses — it also makes browsers enforce MIME types strictly on scripts, which blocks some injection tricks.

### 4. `X-Frame-Options` / `frame-ancestors`

Prevents your site from being embedded in an iframe on another domain — the core of clickjacking, where an attacker overlays invisible buttons on top of your "Delete account" form.

```http
# The classic
X-Frame-Options: DENY

# The modern replacement (CSP directive, more flexible)
Content-Security-Policy: frame-ancestors 'none'
```

`frame-ancestors` supersedes `X-Frame-Options` — but you should send both, because older browsers only understand `X-Frame-Options`. One legitimate use case for allowing framing: embedded widgets or payment iframes. If you need that, `frame-ancestors 'self' https://trusted-partner.com` is way more precise than `X-Frame-Options: ALLOW-FROM` ever was.

### 5. `Referrer-Policy`

Controls what goes in the `Referer` header when your users click links to other sites. The problem: a URL like `https://app.com/reset-password?token=abc123` leaks that whole thing — token included — to whatever site the user clicks next.

```http
Referrer-Policy: strict-origin-when-cross-origin
```

That's the sensible default: full URL for same-origin requests, just the origin when going cross-origin, and nothing on HTTPS→HTTP downgrades. Whatever you do, don't use `no-referrer` globally — it breaks analytics and your own backend's referrer-based CSRF checks.

### 6. `Permissions-Policy`

The successor to the old `Feature-Policy`. Lets you say "this site doesn't need the camera, geolocation, or microphone" — so even if a malicious script runs on your page, it can't silently ask for them.

```http
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

**Trade-off:** if you *do* use these features (say, geolocation for a delivery app), don't block them globally — you'll break the product. Only disable what you don't use.

### 7. The Cross-Origin trio: `COOP`, `CORP`, `COEP`

These are the newer, more powerful ones, and they're genuinely confusing, so let's be quick:

- **`Cross-Origin-Opener-Policy: same-origin`** — isolates your page from windows opened via `window.open` or links with `target="_blank"`. This blocks the "window opener" attack where a malicious page gets a reference to your window and navigates it. Bonus: it's a prerequisite for `crossOriginIsolated` (which unlocks `SharedArrayBuffer`).
- **`Cross-Origin-Resource-Policy: same-origin`** — says "only this origin may load my resources," blocking cross-origin embedding of your responses.
- **`Cross-Origin-Embedder-Policy: require-corp`** — says "only load subresources that explicitly allow cross-origin loading." This is the strict one that *will* break third-party scripts and iframes that don't send `Cross-Origin-Resource-Policy`.

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
```

**The honest advice:** COOP is a solid default. COEP is powerful but will break things — you need to audit every third-party script, font, and iframe before enabling it. If your app loads Google Fonts, analytics, or any widget, `require-corp` is going to be a painful afternoon. It's worth it for high-security apps (banking, healthcare); for a content site, probably not.

### 8. Cookie attributes (technically not headers, but close)

`Set-Cookie` deserves a mention here because it's where the most impactful header-adjacent mistakes happen:

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

- `HttpOnly` — JS can't read the cookie, killing the classic XSS-cookie-theft path
- `Secure` — only sent over HTTPS
- `SameSite=Lax` — default in modern browsers, blocks cross-site cookie sending while keeping top-level navigation working
- `SameSite=Strict` if you can tolerate breaking links that arrive from other sites

## What About `X-XSS-Protection`?

You'll see old checklists telling you to add `X-XSS-Protection: 1; mode=block`. Don't. It's deprecated, and in some cases the browser's built-in XSS filter it controls actually *introduced* vulnerabilities (it could be abused to mangle pages or leak data in older Chrome versions). Modern browsers ignore it. Sending it isn't harmful anymore, but it's a signal your checklist is from 2015. CSP is the real protection.

## How to Actually Add These

### Node.js / Express — use helmet

```js
// ❌ Hand-rolling headers, and forgetting half of them
app.use((req, res, next) => {
  res.setHeader('X-Frame-Options', 'DENY');
  next();
});

// ✅ helmet sets the whole sensible set, and you override deliberately
const helmet = require('helmet');
app.use(helmet());

// Override one directive deliberately, not by accident
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      ...helmet.contentSecurityPolicy.getDefaultDirectives(),
      'script-src': ["'self'", 'https://cdn.example.com'],
    },
  },
}));
```

Helmet's defaults are exactly the "good enough for 95% of apps" set: HSTS (with a sane `max-age`), `nosniff`, `X-Frame-Options`, `Referrer-Policy`, CSP, and more.

### Nginx

```nginx
# ❌ Missing headers = default insecure
server {
    listen 443 ssl;
    location / {
        proxy_pass http://backend;
    }
}

# ✅ Explicit, and includes headers for the API responses too
server {
    listen 443 ssl;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Cross-Origin-Opener-Policy "same-origin" always;
    location / {
        proxy_pass http://backend;
    }
}
```

Note the `always` keyword — without it, Nginx only sends `add_header` on 200 responses, and your error pages and redirects go out naked.

### Cloudflare / managed edge

If you're behind Cloudflare (or similar), you can set most of these in the dashboard under Transform Rules / Security Headers, or with a worker. Do it at the edge and you get consistent headers across all origins, including static files your backend never touches.

## Verifying Your Work

Don't trust that you got it right — check:

```bash
# Quick look at actual response headers
curl -sI https://your-app.com | grep -iE "content-security|strict-transport|x-content|x-frame|referrer|permissions"

# Both headers and CSP details
curl -s https://your-app.com -o /dev/null -D - | head -30
```

Run your site through [securityheaders.com](https://securityheaders.com) or use the Observatory (Mozilla) — they grade you and point out what's missing. Also check on a *real* page (not just `/`) since headers can differ per route.

## What Breaks If You Ignore All This

- Your login page gets embedded in an attacker's iframe, and users click "Allow" on invisible overlays — clickjacking
- A user on public Wi-Fi hits the HTTP version first, and the downgrade attack redirects them to a phishing clone — no HSTS
- An uploaded file served with a guessed content type executes as HTML in your origin — stored XSS
- Your analytics, ad, or third-party widget script gets compromised and now has access to camera/geolocation — because you never restricted Permissions-Policy

## Takeaways

- **Install helmet (or the equivalent) and accept the defaults first** — the defaults are better than most hand-rolled configs, and you can relax specific directives later with intent.
- **HSTS rollout is a ramp, not a switch** — small `max-age` first, verify, then grow. Only add `preload` when you're certain you'll never serve HTTP on that domain.
- **Always send `nosniff` and a sane `Referrer-Policy`** — no downsides, free wins.
- **Use both `X-Frame-Options` and `frame-ancestors`** — the CSP version for modern browsers, the legacy header for the stragglers.
- **Skip `X-XSS-Protection`** — it's deprecated; CSP replaced it.
- **Audit before enabling COEP** — it's the only header here likely to break your site, so treat `require-corp` as a project, not a config change.
- **Verify with a scanner and `curl -I`** — headers you didn't verify are headers you probably don't have.
