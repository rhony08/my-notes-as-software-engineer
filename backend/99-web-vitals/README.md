# Core Web Vitals Explained

You've built a fast app. Your server responds in 50ms, database queries are indexed, CDN is in place. But users are still complaining it "feels slow."

Welcome to the world of Core Web Vitals — Google's set of metrics that measure what users actually experience, not what your server logs say.

## The Problem: Server Metrics Lie

Server-side metrics look great when everything's fine on your end. But they don't capture:

- A slow network on a 3G connection
- A JavaScript-heavy page that takes 6 seconds to become interactive
- Layout shifts that make users accidentally click the wrong button

Core Web Vitals aim to fix that by measuring things from the user's browser.

## The Three Core Metrics

Google settled on three metrics that cover the main aspects of user experience:

| Metric | What It Measures | Good Score | Why It Matters |
|--------|-----------------|-----------|----------------|
| **LCP** (Largest Contentful Paint) | Loading speed — when the main content appears | ≤ 2.5s | Users see a blank screen until LCP happens |
| **FID** (First Input Delay) | Interactivity — how fast the page responds to clicks | ≤ 100ms | Frustrating when you tap something and nothing happens |
| **CLS** (Cumulative Layout Shift) | Visual stability — elements not jumping around | ≤ 0.1 | Prevents accidental clicks and annoyance |

There's a fourth one rolling out — **INP (Interaction to Next Paint)** — which replaces FID. It measures latency for *all* interactions, not just the first one. More on that later.

## LCP: What Users Actually See First

Largest Contentful Paint measures when the biggest visible element (hero image, heading, video thumbnail) finishes rendering.

**Typical culprits when LCP is slow:**

- Large images that aren't optimized
- Render-blocking CSS/JS
- Slow server response times (TTFB)
- Client-side rendered content that takes time to hydrate

```
// ❌ Unoptimized hero image
<img src="hero.jpg" alt="Hero" />  // hero.jpg might be 5MB

// ✅ Optimized with responsive sizes + WebP + lazy loading
<img 
  src="hero-400w.webp" 
  srcset="hero-400w.webp 400w, hero-800w.webp 800w, hero-1200w.webp 1200w"
  sizes="(max-width: 600px) 400px, 100vw"
  alt="Hero" 
  fetchpriority="high"
/>
```

**Quick wins to improve LCP:**

- Preload the LCP image: `<link rel="preload" as="image" href="hero.webp">`
- Use a CDN for static assets
- Minimize render-blocking resources (inline critical CSS, defer non-critical JS)
- Optimize TTFB — if your server takes 1.5s to respond, you've already lost

## FID / INP: Why Your App Feels Laggy

First Input Delay measures the time between a user's first interaction (click, tap, key press) and the browser actually handling it.

This happens because the main thread is busy — JavaScript is parsing, executing, or garbage collecting. The browser can't respond to user input until it finishes.

```
// ❌ Long task blocking the main thread
function loadHeavyData() {
  const data = [];
  for (let i = 0; i < 10000000; i++) {
    data.push(expensiveCalculation(i));  // blocks the main thread for 500ms
  }
  renderTable(data);
}

// ✅ Break it up or defer
function loadHeavyData() {
  // Use requestIdleCallback to avoid blocking
  requestIdleCallback(() => {
    const data = [];
    for (let i = 0; i < 10000000; i++) {
      data.push(expensiveCalculation(i));
    }
    renderTable(data);
  });
}
```

**What causes bad FID/INP:**

- Heavy JavaScript execution on page load
- Large third-party scripts (analytics, ads, tracking)
- Complex component re-rendering on interaction
- Unoptimized event handlers doing too much work

### INP: The New Kid

INP (Interaction to Next Paint) measures the latency of *every* click, tap, and keyboard interaction throughout the page lifecycle — not just the first one. A "good" INP is ≤ 200ms.

Why Google is switching: a page might have a great FID (first interaction is fast) but terrible INP (clicking "submit" takes 1 second because of a heavy form handler). INP catches that.

## CLS: The Annoying Layout Shifts

You've definitely experienced this: you're about to click a link, then an ad loads below it, pushing everything down. You click the wrong thing instead.

That's CLS — Cumulative Layout Shift. It measures how much the visible content shifts unexpectedly.

```
/* ❌ No dimensions set — layout shifts when content loads */
<img src="cat.jpg" alt="Cat" />  /* <-- page jumps when this loads */

/* ✅ Reserve space — no unexpected shifts */
<img src="cat.jpg" alt="Cat" width="800" height="600" style="max-width: 100%; height: auto;" />

/* Even better with aspect-ratio (CSS) */
img {
  aspect-ratio: 800 / 600;  /* browser reserves the space */
  max-width: 100%;
  height: auto;
}
```

**Common CLS causes:**

- Images/videos without explicit dimensions
- Ads, embeds, or iframes that inject content late
- Dynamically injected content (popups, banners)
- Web fonts causing FOIT/FOUT (Flash of Invisible/Unstyled Text)

```
/* ❌ Web font causes invisible text — then layout shifts when it renders */
@font-face {
  font-family: 'FancyFont';
  src: url('/fonts/fancy.woff2');
}

/* ✅ Use font-display: swap to prevent invisible text */
@font-face {
  font-family: 'FancyFont';
  src: url('/fonts/fancy.woff2');
  font-display: swap;  /* Show fallback font immediately, swap when custom loads */
}
```

## Measuring Core Web Vitals

You have a few options. Pick the one that fits your stack:

| Tool | What It Does | Best For |
|------|-------------|----------|
| **Lighthouse** | Lab-based audit in Chrome DevTools | Development, CI/CD checks |
| **PageSpeed Insights** | Combines lab + field data | Quick URL checks |
| **CrUX (Chrome UX Report)** | Real user data from Chrome users | Understanding actual user experience |
| **Web Vitals Extension** | Real-time metrics in your browser | Debugging during development |
| **RUM (Real User Monitoring)** | Custom in-app tracking | Production monitoring at scale |

For the most accurate picture, *you need field data* — what real users experience in the wild. Lab tests (Lighthouse) are useful for catching issues during development, but they won't tell you about slow 3G connections in rural areas or low-end devices.

### Collecting Field Data

```javascript
// Minimal RUM implementation using the Web Vitals library
import { onLCP, onFID, onCLS } from 'web-vitals';

function sendToAnalytics(metric) {
  const body = {
    name: metric.name,
    value: metric.value,
    rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
    delta: metric.delta,
    id: metric.id,  // unique identifier for this metric instance
  };
  
  // Send to your analytics (don't block on this)
  navigator.sendBeacon('/analytics', JSON.stringify(body));
}

onLCP(sendToAnalytics);
onFID(sendToAnalytics);
onCLS(sendToAnalytics);
```

**Why this matters:** Without field data, you're flying blind. Your development machine on a gigabit fiber connection tells you nothing about a user on a $50 Android phone in a congested 4G area.

## The Practical Optimization Order

If you're looking at a page with poor Core Web Vitals, here's the order I'd tackle things:

1. **Fix CLS first** — it's usually the easiest win. Add dimensions to images, use `font-display: swap`, reserve space for ads.
2. **Improve LCP** — optimize your hero image, preload it, fix render-blocking resources.
3. **Address INP/FID** — split long tasks, optimize event handlers, defer non-critical JS.

CLS is often "free" to fix (just add some attributes). LCP takes some effort. INP can be harder — it might require architectural changes to how your app handles JavaScript.

## Trade-offs Worth Knowing

Core Web Vitals aren't the whole story, and chasing perfect scores can backfire:

- **Aggressive lazy loading can hurt LCP** — lazy-loading everything pushes critical content down the priority queue. Be selective.
- **Third-party scripts are hard to control** — that analytics tool your marketing team insists on? It might chunk 200ms onto your INP. Sometimes the "perfect" score isn't realistic.
- **SPA trade-offs** — single-page apps trade initial load time for fast subsequent navigation. CrUX measures individual page loads, so SPAs often score worse on LCP than equivalent server-rendered pages.
- **Not everything is in your control** — slow networks, device capabilities, and user behavior all affect real-world Core Web Vitals. You can optimize, but you can't eliminate external factors.

## Actionable Takeaways

- ✅ **Measure with field data** — CrUX or your own RUM. Lab data catches bugs, field data catches reality.
- ✅ **Fix CLS first** — easiest wins: dimensions on images/videos, `aspect-ratio` in CSS, `font-display: swap`.
- ✅ **Optimize LCP by preloading** critical assets, using responsive images, and improving TTFB.
- ✅ **Split long JS tasks** to improve INP — `requestIdleCallback`, web workers, or code splitting.
- ✅ **Monitor in CI** — add Lighthouse CI or a Web Vitals threshold to your pipeline so regressions get caught before deploy.
- ❌ **Don't obsess over the perfect 100** — a CLS of 0.05 is fine. An LCP of 2.4s is fine. Diminishing returns are real.
