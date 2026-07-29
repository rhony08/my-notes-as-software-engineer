# Content Security Policies

You secure your backend with authentication, rate limits, and input validation. Then a user pastes a malicious script in a comment field, your app renders it unsanitized, and suddenly someone's session cookie is headed to a server you don't control.

That's XSS. And CSP is your second line of defense — a browser-level safety net that says "my app should only load scripts from these trusted sources, and anything else gets blocked."

## How CSP Works

CSP works through an HTTP header — `Content-Security-Policy` — that tells the browser what's allowed and what isn't. Think of it as a whitelist for your page's resources.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://apis.google.com
```

This header says: "Only load content from this origin by default. Scripts can also come from Google's APIs. Everything else? Block it."

No header means your browser trusts *everything* — inline scripts, `eval()`, scripts from any third-party domain. That's a lot of surface area for attackers.

## The Policies You'll Actually Use

CSP has a dozen+ directives, but you'll use these most often:

| Directive | What It Controls | Why It Matters |
|-----------|-----------------|----------------|
| `default-src` | Fallback for all resource types | Set this as your baseline |
| `script-src` | JavaScript sources | Block inline scripts & `eval()` |
| `style-src` | CSS sources | Prevent CSS injection |
| `img-src` | Image sources | Block tracking pixels |
| `connect-src` | XHR/fetch destinations | Control API calls |
| `font-src` | Font sources | Prevent font-based attacks |
| `frame-ancestors` | Who can iframe your page | Clickjacking protection |

## ❌/✅ CSP Headers — What to Actually Ship

Let's walk through the most common patterns.

### The Locked-Down API App

This is tight. No external scripts. No inline anything.

```http
// ❌ Too permissive — might as well not have CSP
Content-Security-Policy: default-src *

// ✅ Locked down — only same-origin
Content-Security-Policy: default-src 'self'
```

### The App Using Google Analytics + a CDN

You need external scripts but want to be specific about it.

```http
// ❌ Wildcard on scripts — defeats the purpose
Content-Security-Policy: script-src 'self' *

// ✅ Explicit origins only
Content-Security-Policy: default-src 'self'; script-src 'self' https://www.googletagmanager.com https://cdn.jsdelivr.net; img-src 'self' https://www.google-analytics.com
```

### The SPA with React/Vue

Modern SPAs use inline scripts for bundling. This is where people get confused.

```http
// ❌ Using 'unsafe-inline' opens the door for XSS
Content-Security-Policy: script-src 'self' 'unsafe-inline'

// ✅ Better: use nonces for known-good inline scripts
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'
```

With nonces, your server generates a random value per request and embeds it in both the CSP header and the script tag:

```html
<script nonce="abc123">
  // This runs — it matches the nonce in the header
</script>

<script>
  // This gets blocked — no nonce
</script>
```

## The Hash Alternative

Instead of generating a nonce per request, you can hash the script content and whitelist that hash:

```http
Content-Security-Policy: script-src 'sha256-ABC123DEF456...'
```

This is great for static apps — the hash changes when the code changes, and nothing else runs. But it's impractical for apps that generate dynamic inline scripts.

## The Reality Check

CSP sounds straightforward. It's not.

**Biggest pain point:** third-party scripts. Google Analytics, Facebook Pixel, YouTube embeds, Stripe — they all need different allowances, and their CDNs change URLs. You'll spend time in report-only mode (we'll get to that) watching what gets blocked.

**Second biggest:** inline event handlers. `onclick="doStuff()"` in HTML? That's inline script. `style="color: red"` in HTML? That's inline style. CSP blocks both unless you explicitly allow `'unsafe-inline'`. The fix is to move event handlers to external JS files and use CSS classes instead of inline styles.

## Report-Only Mode: Test Before You Enforce

This is your best friend when rolling out CSP:

```http
// Reports violations without blocking anything
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report
```

Run in report-only for a week. Collect violations. Fix the ones that break legitimate functionality. *Then* switch to the enforce header.

### Handling Violations

Set up a `report-uri` (or the newer `report-to`) endpoint to collect violations:

```json
// POST body to your /csp-report endpoint
{
  "csp-report": {
    "document-uri": "https://example.com/dashboard",
    "blocked-uri": "https://evil.com/tracker.js",
    "violated-directive": "script-src 'self'",
    "effective-directive": "script-src"
  }
}
```

This tells you exactly what got blocked. Use it to tune your policy before enforcing.

## Trade-offs Worth Knowing

- **Report-only doesn't protect users** — it's for testing, not production safety
- **Nonces are great but add complexity** — your server needs to inject them into HTML templates
- **Hashes are static but fragile** — change a single character in the script and the hash breaks
- **`'strict-dynamic'`** passes trust to loaded scripts (great for SPAs), but old browsers ignore it, so you still need `'unsafe-inline'` as a fallback for them
- **`eval()` in your codebase?** You'll have to refactor it or accept `'unsafe-eval'` — there's no clean workaround

## What to Do This Week

1. **Add a report-only policy** to your app today — even a basic one
2. **Monitor violations** for a few days — see what breaks
3. **Audit your inline scripts** — move them to external files where possible
4. **Generate nonces** from your server or framework (most modern frameworks support this)
5. **Switch to enforce** once the violation noise is manageable

Start with `default-src 'self'`. You'll inevitably loosen it for third-party services, but at least you'll know exactly *what* you're allowing and *why*.
