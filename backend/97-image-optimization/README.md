# Image Optimization Techniques

Your users bounce. Your Lighthouse score is in the red. You check the Network tab and — oof. There they are: 2MB JPEGs, PNGs nobody compressed, and that hero image you uploaded straight from the camera roll. Images are almost always the biggest thing on any web page, and treating them like afterthoughts is costing you real performance.

Image optimization isn't about squeezing every byte until the picture looks like a potato. It's about delivering the *right quality at the right size at the right time* — without making the user wait.

## What Goes Wrong Without Optimization

Without a strategy, you'll hit every one of these:

- **Load time kills conversions** — a 1-second delay drops conversions by 7%. A 5-second page? You've already lost half your visitors.
- **Data bills on mobile** — that 3MB hero image is someone's monthly allowance on a prepaid plan
- **Layout shift** — images loading late push content around, wrecking the user experience
- **CDN costs** — serving giant files to millions of visitors gets expensive fast

The fix isn't "just use WebP." It's a pipeline of techniques that work together.

---

## Technique 1: Choose the Right Format

This is where most people stop. But format choice depends on what you're serving:

| Format | Best For | Notes |
|--------|----------|-------|
| **WebP** | Photos, illustrations, everything general | 25-35% smaller than JPEG at same quality. Supported everywhere modern. |
| **AVIF** | High-quality photos | Even smaller than WebP, but encoding is slow. AVIF can go 50% smaller than JPEG. |
| **JPEG** | Photos (fallback) | Universal. Use when you need maximum compatibility. |
| **PNG** | Screenshots, logos with transparency | Lossless, but huge. Only for cases where pixel-perfect matters. |
| **SVG** | Icons, logos, illustrations | Vector. Infinitely scalable. Tiny file size for simple shapes. |
| **GIF** | ...just don't | Seriously, use video instead. More on that below. |

**The practical rule:** Ship WebP with a JPEG fallback in `<picture>` tags, use AVIF if you can afford the encoding time, and never serve a raw PNG for photos.

```html
<!-- ✅ Serve WebP with JPEG fallback -->
<picture>
  <source srcset="hero.avif" type="image/avif">
  <source srcset="hero.webp" type="image/webp">
  <img src="hero.jpg" alt="Hero banner" loading="lazy" width="1200" height="600">
</picture>

<!-- ❌ Just hoping the browser figures it out -->
<img src="hero.jpg" alt="Hero banner">
```

## Technique 2: Properly Size Images

Serving a 4000px-wide image into a 300px card is wasteful. The browser *will* downscale it, but it still downloads the full 4000px worth of data first.

**Responsive images with `srcset`:**

```html
<img
  src="photo-800.jpg"
  srcset="
    photo-400.jpg 400w,
    photo-800.jpg 800w,
    photo-1200.jpg 1200w
  "
  sizes="(max-width: 600px) 100vw, 50vw"
  alt="..."
>
```

This tells the browser: "Here are three sizes. Pick the one that fits the user's viewport." The browser handles the rest.

**Always set explicit width and height:**

```html
<!-- ❌ Causes layout shift -->
<img src="photo.jpg" alt="...">

<!-- ✅ Reserves space, no CLS impact -->
<img src="photo.jpg" alt="..." width="800" height="600">
```

If you don't set dimensions, the browser doesn't know the aspect ratio until the image loads. That means content below the image jumps down when it finally arrives — Cumulative Layout Shift penalty.

## Technique 3: Lazy Loading

Never load images the user can't see yet.

```html
<!-- ✅ Native lazy loading — it's this simple -->
<img src="photo.jpg" loading="lazy" alt="...">

<!-- ✅ For above-the-fold priority images -->
<img src="hero.jpg" fetchpriority="high" alt="Hero">
```

**Rules of thumb:**
- Above the fold → `fetchpriority="high"`, no lazy loading
- Below the fold → `loading="lazy"` on everything
- First 1-2 images in the viewport → eager, everything else → lazy

The browser handles intersection observer natively now. No JS library needed.

## Technique 4: Compression Without the Ugly

Every image tool gives you a quality slider. The trick is finding the *sweet spot* where the file shrinks but nobody notices.

```bash
# Using sharp in a Node.js build script
const sharp = require('sharp');

// ✅ Lossy WebP at quality 80 — barely visible difference, huge savings
await sharp('input.jpg')
  .resize(1200)
  .webp({ quality: 80 })
  .toFile('output.webp');

// ❌ quality 100 — negligible visual gain, double the file size
await sharp('input.jpg')
  .webp({ quality: 100 })
  .toFile('output.webp');
```

```bash
# Using squoosh-cli for batch processing
npx @squoosh/cli --webp '{"quality":75}' --output-dir . images/*.jpg
```

**What quality 75-85 looks like in practice:**
- A 2MB raw JPEG → ~150KB WebP
- A 500KB PNG screenshot → ~40KB WebP
- Visual difference? Essentially zero in a blind test

## Technique 5: Stop Using GIFs

GIF is a 1987 format. It has 256 colors. It's terrible at video. And your users are downloading multi-megabyte GIFs that could be 10% the size as video.

```html
<!-- ❌ 5MB looping GIF -->
<img src="cat-dancing.gif" alt="Cat dancing">

<!-- ✅ 200KB video, autoplays, loops, no audio -->
<video autoplay loop muted playsinline>
  <source src="cat-dancing.webm" type="video/webm">
  <source src="cat-dancing.mp4" type="video/mp4">
</video>
```

**Savings from GIF → video:**
- MP4: typically 80-90% smaller than the equivalent GIF
- WebM: even better compression
- Bonus: you get 24-bit color and proper transparency

## Technique 6: Serve via CDN with On-the-Fly Processing

This is the cheat code. Modern image CDNs (Cloudinary, imgix, Cloudflare Images) let you transform images at the edge with URL parameters:

```
https://cdn.example.com/photo.jpg?w=800&h=600&fit=cover&q=80&f=webp
```

One URL. The CDN:
1. Resizes to 800×600
2. Crops to fill
3. Compresses to quality 80
4. Converts to WebP
5. Caches the result

No build step. No pre-generation of 50 variants. Just transform on request and cache aggressively.

If you're on a CDN that supports it, this is the single biggest bang for your effort. You can store one high-resolution original and transform it on delivery.

## Technique 7: Image CDNs and the Cache Strategy

Even with on-the-fly transforms, you need a caching strategy:

```
Client → CDN Edge → (cache hit? serve cached version) → Origin Server → Transform → Cache → Serve
```

**Cache headers for images:**
```
Cache-Control: public, max-age=31536000, immutable
```

Images rarely change. Set a year-long cache. If you need to update, change the URL (cache-busting with a hash in the filename).

---

## Trade-offs Worth Knowing

| Technique | Benefit | Cost |
|-----------|---------|------|
| WebP/AVIF conversion | Smaller files | Server-side conversion time, encoding CPU |
| Multiple format variants | Browser picks the best | Storage for multiple files per image |
| Responsive srcset | Right size for viewport | More HTML, more variants to generate |
| On-the-fly CDN transforms | One source, many outputs | CDN cost per transform |
| Aggressive compression | Tiny files | Visible artifacts if you push too far |
| Lazy loading | Faster initial paint | Brief flash of empty space if not handled |

There's no free lunch. Aggressive compression saves bytes but at quality 50 you'll see artifacts. Multiple formats save bandwidth but cost build time. Pick what matches your audience — if they're mostly on fast Wi-Fi with retina screens, your priorities shift vs mobile-first in emerging markets.

---

## What You Can Do Right Now

Start with the 80/20:

1. **Switch your default image format to WebP** — build tooling, fallback to JPEG
2. **Add `loading="lazy"` to every image below the fold** — it's one attribute
3. **Resize images to their display dimensions** — don't serve 4000px for a 400px card
4. **Set explicit width and height** — stop the layout shifts
5. **Convert GIFs to video** — it's the easiest 90% bandwidth saving you'll ever make
6. **Use `srcset` on your hero/featured images** — let the browser pick
7. **Look into an image CDN** — if your traffic is significant, it pays for itself in CDN bandwidth savings alone

Run the images through a compressor in your build pipeline. Add a Lighthouse CI check that flags oversized images. Make it part of your definition of done — not an afterthought.
