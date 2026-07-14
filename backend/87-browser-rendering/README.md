# Browser Rendering Pipeline

You type a URL, hit Enter, and a few hundred milliseconds later — pixels. A fully rendered page.

But what actually happens between the browser getting HTML and showing you something useful? If you're coming from backend (like me), this whole process feels like magic. It's not. It's a surprisingly well-defined pipeline, and understanding it is the difference between "the page is slow, let's throw more hardware at it" and knowing exactly which part of the chain is choking.

Let's walk through it.

---

## The Pipeline at a Glance

The browser doesn't paint a page all at once. It goes through stages, and each stage depends on the one before it:

```
HTML  →  DOM Tree
         CSS  →  CSSOM
                  DOM + CSSOM  →  Render Tree
                                   Layout (Flow)
                                    Paint
                                     Composite
```

Each step can block the next. Some steps can be skipped if nothing changed. That's where performance optimization lives.

---

## Stage 1: HTML → DOM Tree

The browser receives raw HTML bytes. The first thing it does is parse them into tokens — tags like `<div>`, attributes like `class="container"`, text nodes. These tokens form a tree: the **Document Object Model (DOM)**.

```html
<!-- Input HTML -->
<div class="card">
  <h1>Hello</h1>
  <p>World</p>
</div>
```

Becomes:

```
Document
 └── html
      └── body
           └── div.card
                ├── h1 ("Hello")
                └── p ("World")
```

**The catch:** Parsing is blocking. When the parser hits a `<script>` tag (without `async` or `defer`), it stops. Downloads the script. Executes it. Then continues.

```
<!-- ❌ This blocks DOM parsing -->
<script src="heavy-analytics.js"></script>

<!-- ✅ These don't block -->
<script async src="heavy-analytics.js"></script>
<script defer src="heavy-analytics.js"></script>
```

### Why this matters
If you're loading a dozen render-blocking scripts, your user stares at a white screen until all of them download and run. On a 3G connection, that's seconds. Done.

---

## Stage 2: CSS → CSSOM

While the HTML parser is building the DOM, another process is parsing CSS into the **CSS Object Model (CSSOM)**. Same idea: tokens → tree, but this tree describes styles, not structure.

The difference is subtle but critical:

- **DOM** can be built incrementally — the browser can start rendering partial content
- **CSSOM** is built all-or-nothing — you can't paint until all CSS is parsed, because the browser needs to know the *final* computed styles before it draws anything

```html
<!-- ❌ This blocks rendering -->
<link rel="stylesheet" href="styles.css">

<!-- This also blocks rendering, even though it loads async -->
<link rel="stylesheet" href="print.css" media="print">
```

**The CSSOM is why CSS is considered "render blocking."** The browser will not paint a single pixel until it's done.

> **Trade-off:** You could inline critical CSS (under 14KB) in the `<head>` to get pixels on screen faster, but that increases initial HTML size. There's no free lunch.

---

## Stage 3: DOM + CSSOM → Render Tree

The browser takes the DOM and the CSSOM and merges them into the **Render Tree**.

This isn't just "DOM plus styles." It's filtered:
- `display: none` elements are excluded (they take no visual space)
- `head`, `script`, `meta` tags are excluded
- Only visible elements make it in

```
DOM                          CSSOM                        Render Tree
html                         html { display: block }      [html block]
 ├── head (excluded)                                       ├── [body block]
 ├── body                    body { margin: 8px }          │    ├── [div block]
 │    ├── div.card                                          │    │    ├── [h1 block]
 │    │    ├── h1            h1 { font-size: 24px }        │    │    └── [p block]
 │    │    └── p             p { font-size: 16px }         └── (script excluded)
 │    └── script (excluded)
```

**Important:** The Render Tree only has *visible* nodes. If you hide something with `display: none`, it's not in the render tree. If you hide with `visibility: hidden`, it *is* (it occupies space, just invisible).

---

## Stage 4: Layout (Reflow)

Now the browser knows *what* to draw and *how* it should look. But it doesn't know *where* to put things. That's layout.

The browser calculates:
- Element positions (x, y coordinates)
- Element sizes (width, height)
- How elements relate to each other (normal flow, flexbox, grid)

```css
.container {
  width: 80%;
  margin: 0 auto;
  display: flex;
  gap: 16px;
}
```

The browser needs to figure out what "80%" means in pixels. Then calculate margins. Then distribute flex children. Then compute gaps. All of this is layout.

**Layout is expensive.** Changing element geometry triggers a **reflow** (or layout recalculation), which cascades — change the width of an element, and all its children may need new positions too.

```javascript
// ❌ Causes reflow — multiple times
element.style.width = '200px';
console.log(element.offsetHeight);  // Forces reflow to read layout
element.style.height = '100px';
console.log(element.offsetWidth);   // Forces reflow again

// ✅ Batch reads and writes separately
const height = element.offsetHeight;  // Read once
element.style.width = '200px';
element.style.height = '100px';       // Write once — one reflow
```

---

## Stage 5: Paint

Now the browser knows *where* everything goes and *how* it should look. Time to fill in the pixels.

Painting is drawing: filling backgrounds, drawing borders, rendering text, applying shadows. The browser converts the render tree into actual pixel data.

**Paint is also expensive.** Things that trigger a repaint:
- Changing colors (`color`, `background-color`)
- Changing shadows (`box-shadow`, `text-shadow`)
- Changing visibility (`visibility: hidden`)
- Border changes

If only visual properties change (not layout), the browser can repaint without reflowing — which is cheaper, but not free.

```css
/* ❌ Triggers both layout AND paint */
.element {
  width: 50%;  /* layout change */
}

/* ✅ Only triggers paint */
.element {
  background-color: red;  /* paint only, no layout change */
}

/* ✅ Only triggers compositing (cheapest) */
.element {
  transform: translateX(100px);  /* compositor-only on GPU */
}
```

---

## Stage 6: Compositing

The browser splits the page into **layers** (like layers in Photoshop) and composites them together. This is what makes scrolling and animations smooth — the compositor thread runs separately from the main thread.

Things that promote an element to its own layer:
- `transform` and `opacity` animations
- `will-change: transform`
- `<video>`, `<canvas>`, `<iframe>` elements

```css
/* ✅ Promoted to its own compositor layer */
.animated-card {
  will-change: transform;
  transition: transform 0.3s ease;
}

.animated-card:hover {
  transform: scale(1.05);
}
```

**Why this matters:** Layers are composited on the GPU. Animating transforms or opacity doesn't trigger layout or paint — it just shifts pixels around in the compositing step. That's why GPU-accelerated animations are buttery smooth.

But layers aren't free. Each layer uses GPU memory. Too many layers can overload memory on low-end devices.

> **Trade-off:** Promoting everything to its own layer (`translateZ(0)` on every element) used to be a "performance hack." Now it's considered an anti-pattern — it wastes memory and can actually hurt performance on mobile.

---

## The Critical Rendering Path

The **critical rendering path** is the sequence of steps the browser must complete before it can paint *anything*:

1. Get HTML bytes
2. Build DOM
3. Get CSS bytes
4. Build CSSOM
5. Create Render Tree
6. Layout
7. Paint
8. Composite

Anything that blocks any of these steps delays the **first meaningful paint** — the moment the user sees something useful.

### How to Optimize It

| Problem | Solution |
|---------|----------|
| Render-blocking CSS | Inline critical CSS, defer non-critical |
| Render-blocking JS | Use `async` or `defer` on scripts |
| Large images | Lazy-load, use responsive images |
| Too many DOM nodes | Reduce markup complexity |
| Layout thrashing | Batch DOM reads/writes |
| Expensive paint | Prefer `transform`/`opacity` animations |

---

## What Breaks Without Understanding This

- You add a third-party widget script in the `<head>`, and the page goes blank for 2 seconds
- You animate `left` and `top` instead of `transform`, and your 60fps animation drops to 15fps
- You set `will-change: all` on every element, and Chrome eats 300MB of GPU memory on a budget phone
- You load CSS at the bottom of `<body>` "for speed," and the page flashes unstyled content (FOUC)

All of these are just pipeline problems. Know the pipeline, fix the pipeline.

---

## The Takeaway

- **DOM ≠ Render Tree.** Not everything in the DOM appears on screen.
- **CSS blocks rendering.** Get critical CSS to the browser fast.
- **Layout is expensive.** Avoid triggering reflows unnecessarily.
- **Paint is also expensive.** Prefer compositor-only properties like `transform` and `opacity`.
- **Compositing is the cheapest.** Use layers wisely, don't overdo it.
- **The critical rendering path is your bottleneck.** Optimize the first paint, everything else comes after.

The browser rendering pipeline isn't abstract trivia. It's the difference between a snappy app and one that feels sluggish. And once you see it, you can't unsee it.
