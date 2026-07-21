# Critical Rendering Path Optimization

You've done the hard work — built a beautiful site, added interactions, polished the UI. Then you load it on a real device and stare at a white screen for 2.5 seconds. What happened?

The browser had to work through your HTML, CSS, and JavaScript before it could paint a single pixel. That pipeline is the **critical rendering path** (CRP), and if you don't optimize it, your users are staring at nothing while your code processes.

---

## The Pipeline: What the Browser Does

Every page load goes through the same sequence:

```
HTML → DOM
CSS → CSSOM
DOM + CSSOM → Render Tree
Render Tree → Layout → Paint
```

Each step blocks the next. You can't build the render tree without both DOM and CSSOM. You can't paint before layout finishes. And JavaScript? It can block the whole thing.

### Step 1: Bytes → DOM (HTML Parsing)

The browser reads raw HTML bytes, tokenizes them, builds nodes, and constructs the Document Object Model.

```html
<!-- When the browser sees this: -->
<html>
  <head>
    <link href="styles.css" rel="stylesheet">
  </head>
  <body>
    <h1>Hello</h1>
  </body>
</html>
```

The parser works incrementally — it can start building the DOM as soon as it receives partial HTML. But it pauses when it hits certain resources.

### Step 2: Bytes → CSSOM (CSS Parsing)

CSS is **render-blocking** by default. The browser won't render anything until it's downloaded and parsed all CSS.

```html
<!-- ❌ This blocks rendering -->
<link href="styles.css" rel="stylesheet">

<!-- ✅ This does NOT block rendering -->
<link href="non-critical.css" rel="stylesheet" media="print">
<link href="mobile.css" rel="stylesheet" media="(max-width: 600px)">
```

The `media` attribute is your friend — the browser only loads print stylesheets as non-blocking, and media-query-specific sheets only block rendering when the query matches.

### Step 3: Render Tree Construction

The browser combines DOM + CSSOM, but it only includes visible elements:

- `display: none` elements are excluded
- `head` and its children are excluded
- Invisible elements (`script`, `meta`) are excluded

```css
/* ❌ This element is in the DOM but not the render tree */
.hidden { display: none; }

/* ✅ This IS in the render tree (just invisible) */
.invisible { visibility: hidden; opacity: 0; }
```

Subtle difference: `display: none` removes the element from the render tree entirely. `visibility: hidden` keeps it in the tree — it just doesn't paint. Important for layout calculations.

### Step 4: Layout (Recalculate Styles)

Now the browser figures out where everything goes: widths, heights, positions. This is expensive for complex pages.

```css
/* Cheap — just changes paint */
.color-change { color: red; background-color: blue; }

/* Expensive — triggers layout recalculation */
.width-change { width: 75%; margin-left: 20px; }
```

**Layout triggers** include: width, height, padding, margin, border, position, top/left/right/bottom, font-size, font-weight, display, float, clear, etc.

### Step 5: Paint

The browser fills in pixels. This is fast for simple backgrounds, slow for shadows, gradients, and filters.

| Property | Cost | Triggers |
|----------|------|----------|
| `background-color` | Cheap | Paint only |
| `box-shadow` | Expensive | Paint |
| `transform` | Cheap | Composite only |
| `opacity` | Cheap | Composite only |
| `width`/`height` | Expensive | Layout + Paint |

---

## Where JavaScript Messes Everything Up

Here's the problem: **JavaScript can modify the DOM and CSSOM**. So the browser has to wait for JavaScript to run before it continues rendering.

```
HTML → DOM → (parser blocked) → JS runs → continue DOM → Render Tree
```

And there's a subtlety the docs don't always emphasize:

```html
<!-- The parser waits for this CSS before executing JS -->
<link href="styles.css" rel="stylesheet">
<script src="app.js"></script>
```

Even if `app.js` doesn't touch styles, the browser waits for `styles.css` to load first. Why? Because `app.js` might ask for computed styles. If the CSS hasn't loaded yet, the JS would get wrong values. This creates a **critical request chain**:

```
HTML → styles.css → app.js → rest of HTML
```

### The Fix: Defer and Async

```html
<!-- ❌ Blocks parser and rendering -->
<script src="heavy.js"></script>

<!-- ✅ Downloads in parallel, executes after HTML parsing -->
<script src="heavy.js" defer></script>

<!-- ✅ Downloads in parallel, executes as soon as it's available -->
<script src="analytics.js" async></script>
```

| Strategy | Download | Execution | Order Preserved |
|----------|----------|-----------|-----------------|
| Default | Blocks parser | Immediate | Yes |
| `defer` | Parallel, non-blocking | After HTML parsed | Yes |
| `async` | Parallel, non-blocking | Whenever loaded | No |

**Rule of thumb:** Use `defer` for your own scripts (dependency order matters), `async` for third-party scripts (analytics, ads) where order doesn't.

---

## The Metrics That Matter

For CRP optimization, watch these three:

### First Contentful Paint (FCP)

When the first text or image appears. Optimized CRP = fast FCP.

### Largest Contentful Paint (LCP)

When the main content is visible. This includes the hero image, headline, or main content block.

| LCP Time | Rating |
|----------|--------|
| < 1.5s | Good ✅ |
| 1.5 - 2.5s | Needs work ⚠️ |
| > 2.5s | Poor ❌ |

### Time to Interactive (TTI)

When the page is fully interactive — event handlers attached, JS ready.

---

## Practical Optimization Tactics

### 1. Inline Critical CSS

For the CSS needed above the fold (what users see first), inline it directly in the `<head>`:

```html
<!-- ❌ Extra round trip for above-the-fold styles -->
<link href="styles.css" rel="stylesheet">

<!-- ✅ Inline critical, defer the rest -->
<head>
  <style>
    /* Critical styles — everything needed for hero/header/nav */
    header { display: flex; padding: 1rem; background: #fff; }
    .hero { font-size: 2rem; color: #333; }
  </style>
  <link href="styles.css" rel="stylesheet" media="print" onload="this.media='all'">
</head>
```

The `onload` trick loads non-critical CSS without blocking — it starts as print-only (non-blocking), then switches to `all` after loading.

**Trade-off:** Inlined CSS means bigger HTML. If your critical CSS is over 14KB (one TCP round trip), you're actually making things worse for HTTP/1.1. For HTTP/2, the threshold changes — but the principle stays: keep critical CSS minimal.

### 2. Preload Key Resources

Tell the browser about critical resources before the parser finds them:

```html
<!-- Preload hero image so it starts loading early -->
<link rel="preload" href="hero.webp" as="image">

<!-- Preload critical fonts -->
<link rel="preload" href="/fonts/inter-var.woff2" as="font" crossorigin>

<!-- Preload render-critical CSS -->
<link rel="preload" href="critical.css" as="style">
```

### 3. Minimize Render-Blocking Resources

Use Lighthouse or Chrome DevTools to audit your critical request chain. The goal: **2-3 round trips max** before the first paint.

```bash
# Run Lighthouse from CLI for CRP audit
lighthouse https://yoursite.com --preset=desktop --quiet
```

Look for:
- Any `<script>` without `defer`/`async` in the `<head>`
- Non-critical CSS loaded in the `<head>` without `media` trick
- Font files blocking render

### 4. Optimize Font Loading

Fonts are often invisible render blockers. The browser won't render text until the font loads — that's the **Flash of Invisible Text (FOIT)**.

```css
/* ✅ Use font-display to control behavior */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-display: swap; /* Show fallback text immediately, swap when font loads */
}
```

| `font-display` | Behavior |
|----------------|----------|
| `auto` | Browser decides (usually FOIT) |
| `swap` | Show fallback, swap when font arrives |
| `optional` | Show fallback if font doesn't arrive quickly |
| `block` | Short FOIT period, then swap |

**Trade-off:** `font-display: swap` gives you instant text but a layout shift when the font loads (CLS penalty). `optional` avoids layout shifts but some users might never see your custom font.

---

## Putting It All Together: Before and After

### Before (Slow)

```html
<head>
  <link href="bootstrap.css" rel="stylesheet">
  <link href="app.css" rel="stylesheet">
  <script src="jquery.js"></script>
  <script src="app.js"></script>
</head>
```

Critical request chain: `HTML → bootstrap.css → app.css → jquery.js → app.js → first paint`

That's 4 round trips before anything visible. On a 3G connection, that's easily 2+ seconds of white screen.

### After (Fast)

```html
<head>
  <style>
    /* ~3KB of critical CSS inlined */
  </style>
  <link rel="preload" href="hero.webp" as="image">
  <link href="bootstrap.css" rel="stylesheet" media="print" onload="this.media='all'">
  <link href="app.css" rel="stylesheet" media="print" onload="this.media='all'">
  <script src="jquery.js" defer></script>
  <script src="app.js" defer></script>
</head>
```

Now: `HTML → first paint`. The critical CSS is in the HTML itself. Everything else loads in the background.

---

## What to Take Away

- **CSS blocks rendering** → inline critical CSS, defer the rest
- **JavaScript blocks parsing** → use `defer` for your scripts, `async` for third-party
- **The critical request chain is your enemy** → audit it with DevTools, keep it short
- **Fonts need love too** → use `font-display: swap` and preload key fonts
- **Measure what matters** → track FCP, LCP, and TTI, not just "page load"

The CRP isn't something you optimize once and forget. Every new library, stylesheet, or script you add extends the chain. Treat it like a budget — you've got limited time before the user sees something, and every KB counts.
