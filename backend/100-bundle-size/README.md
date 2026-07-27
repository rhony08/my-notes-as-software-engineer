# Bundle Size Optimization

Your app's JavaScript bundle is 3MB. Users on a 4G connection wait 8 seconds before seeing anything useful. 53% of mobile users leave if a page takes longer than 3 seconds to load. That's not just bad UX — that's lost revenue.

Bundle size isn't just a "nice to optimize" thing. It directly impacts how fast your app loads, how quickly it's interactive, and whether users stick around. Let's talk about what's bloating your bundles and how to fix it.

## What's Eating Your Bundle?

Before optimizing, you need to know what's in there. The usual suspects:

- **Heavy dependencies** — Lodash, Moment.js, entire UI frameworks when you only need a few components
- **Duplicate code** — Same utility function in 5 places across different modules
- **Dead code** — Unused exports, components you removed but forgot to delete
- **Large assets** — Unoptimized images embedded as base64, huge fonts
- **Duplicate dependencies** — Multiple versions of the same library (hello, npm dedupe issues)

### First Step: Find Out

Use tools to visualize your bundle:

```bash
# Webpack
webpack --profile --json > stats.json
npx webpack-bundle-analyzer stats.json

# Vite (built-in)
npx vite build --mode analyze

# Rollup
npx rollup-plugin-visualizer
```

If you're not running a bundle analyzer, you're optimizing blind. It's like trying to clean a messy room in the dark.

## Code Splitting: The Big One

Don't ship everything at once. Split your bundle so users only download what they need for the current page.

### Route-Based Splitting

```javascript
// ❌ Single massive bundle
import AdminDashboard from './pages/AdminDashboard';

// ✅ Split by route so admin code only loads on /admin
const AdminDashboard = React.lazy(() => import('./pages/AdminDashboard'));

// In your router:
<Route
  path="/admin"
  element={
    <Suspense fallback={<Spinner />}>
      <AdminDashboard />
    </Suspense>
  }
/>
```

The admin dashboard code? It only downloads when someone actually visits `/admin`. For everyone else, that's hundreds of KB they never need.

### Component-Level Splitting

Not ready for route splitting? Split at the component level:

```javascript
// ❌ Heavy chart library loaded even if user never opens the chart tab
import HeavyChart from './HeavyChart';

// ✅ Loaded only when the chart tab activates
const ChartModal = React.lazy(() =>
  import('./ChartModal').then(module => ({ default: module.HeavyChart }))
);
```

**Why this matters:** A chart library like D3 or ECharts can be 200-500KB. If only 10% of users ever see charts, the other 90% are paying for it.

## Tree Shaking: Dead Code Elimination

Tree shaking removes unused exports. Modern bundlers do this automatically — if you set it up right.

### What Breaks Tree Shaking

```javascript
// ❌ Barrel imports kill tree shaking
import { format, parse, addDays, subMonths } from 'date-fns';

// ✅ Direct imports keep tree shaking working
import format from 'date-fns/format';
import parse from 'date-fns/parse';
```

Barrel files (`index.js` that re-exports everything) are convenient but destructive for bundle size. Each file in a barrel import forces the bundler to include everything from every re-exported file because it can't statically analyze which exports are actually used across the chain.

| Approach | Bundle Impact | Maintainability |
|----------|--------------|-----------------|
| Barrel imports | Easy to write, bigger bundles | Convenient but costly |
| Direct imports | Smaller bundles | More import lines |
| Path-based imports from date-fns/lodash | Best size | Ugly but effective |

### Side Effects: The Hidden Killer

```json
// package.json - tell the bundler it's safe to shake
{
  "sideEffects": false
}
```

If your package has CSS imports:

```json
{
  "sideEffects": [
    "*.css",
    "*.scss"
  ]
}
```

Without telling the bundler about side effects, it plays it safe and includes code it could otherwise remove. This single config change often shaves off 10-30%.

## Replace Heavy Dependencies

### Moment.js → date-fns or dayjs

Moment.js is 231KB minified. It's been deprecated for years. Stop using it.

```javascript
// ❌ 231KB
import moment from 'moment';
moment().format('YYYY-MM-DD');

// ✅ 2KB
import dayjs from 'dayjs';
dayjs().format('YYYY-MM-DD');

// ✅ 16KB (tree-shakable)
import { format } from 'date-fns';
format(new Date(), 'yyyy-MM-dd');
```

### Lodash → Native JS or lodash-es

```javascript
// ❌ Full lodash: ~71KB
import _ from 'lodash';
_.debounce(fn, 300);

// ✅ lodash-es (tree-shakable)
import debounce from 'lodash-es/debounce';

// ✅ Native: 0KB (for modern browsers)
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

| Library | Size | Better Alternative | Size Saved |
|---------|------|-------------------|------------|
| Moment.js | 231KB | dayjs | 229KB |
| Lodash (full) | 71KB | lodash-es | ~55KB (tree-shaken) |
| jQuery (full) | 87KB | Vanilla JS | 87KB |
| Axios | 32KB | fetch API | 32KB |

## Dynamic Imports for Heavy Features

Features like rich text editors, PDF viewers, and image editors are massive. Don't load them eagerly.

```javascript
// ❌ Loads on every page
import QuillEditor from 'react-quill';

// ✅ Only loads when user clicks "Edit"
const handleEdit = async () => {
  const { default: QuillEditor } = await import('react-quill');
  setEditor(<QuillEditor />);
};
```

The `import()` pattern works with any bundler. It creates a separate chunk automatically. Users who never click "Edit" never pay for Quill's 300KB.

## Image Optimization

Bundle size isn't just JS. Images often make up way more than JavaScript ever will.

```javascript
// ❌ Full-res image embedded in bundle
import heroImage from './assets/hero.jpg';  // 2MB

// ✅ Responsive, lazy-loaded, WebP
<img
  src="hero-800w.webp"
  srcSet="hero-400w.webp 400w, hero-800w.webp 800w, hero-1200w.webp 1200w"
  sizes="(max-width: 600px) 400px, 800px"
  loading="lazy"
  alt="Hero section"
/>
```

### Quick Wins:

- **WebP/AVIF** — 25-35% smaller than JPEG/PNG with same quality
- **Responsive images** — Don't ship 2000px images for mobile
- **Lazy loading** — `loading="lazy"` on all below-the-fold images
- **CSS sprites** or **inline SVGs** for icons (avoid icon fonts)

## Gzip / Brotli Compression

This isn't just about making files smaller — it's about making your CDN serve compressed bundles.

```nginx
# Nginx config
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_min_length 256;
gzip_comp_level 6;

# Brotli (better, newer)
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

Brotli typically gives 20-30% better compression than gzip. A 500KB bundle becomes ~150KB with Brotli. Your users get that 500KB file in the time they'd normally wait for 150KB.

## Monitoring Bundle Size in CI

Don't let regressions slip through. Set up bundle size checks in CI:

```javascript
// bundle-size.config.js
module.exports = {
  entries: {
    'main': { path: './dist/main.js', maxSize: '150KB' },
    'vendor': { path: './dist/vendor.js', maxSize: '80KB' },
    'admin': { path: './dist/admin.js', maxSize: '100KB' }
  }
};
```

```yaml
# GitHub Action
- name: Check bundle size
  run: npx bundlesize
```

When someone accidentally imports the full lodash instead of tree-shaken, the CI fails. Way better than discovering it in a Lighthouse audit.

## Trade-offs: What You Lose

Code splitting isn't free:

| Optimization | Cost | Benefit |
|-------------|------|---------|
| More chunks = more network requests | Slower on HTTP/1.1 | Faster on HTTP/2+ |
| Lazy loading = loading spinners | Worse UX for slow connections | Better initial load |
| Direct imports = more import lines | More code churn | Smaller bundles |
| Tree shaking = config complexity | Harder to debug | ~10-30% size reduction |
| Image optimization = build time | Slower CI | Massive bandwidth savings |

For HTTP/1.1, too many chunks actually hurt. For HTTP/2 (which is basically standard now), multiplexing makes multiple chunks fine. But if you're supporting old infrastructure, balance chunk count with size.

## What a Good Optimization Looks Like

Real numbers from a React SPA I worked on:

| Before | After | Technique |
|--------|-------|-----------|
| 482KB JS | 185KB JS | Route splitting + tree shaking |
| 1.2MB images | 340KB images | WebP + responsive + lazy loading |
| 231KB Moment.js | 2KB dayjs | Library swap |
| 87KB unused components | removed | Dead code audit |
| **2MB total** | **~527KB total** | **~74% reduction** |

First meaningful paint went from 4.2s to 1.1s. Not bad for a weekend of work.

---

## Actionable Takeaways

- Run a bundle analyzer before guessing what's big — data beats intuition
- Set up route-based code splitting from day one — retrofitting is painful
- Replace Moment.js with dayjs or date-fns — it's an easy 229KB win
- Use `sideEffects: false` in package.json — it's a config change that works for free
- Set a bundle size budget in CI — prevent regressions before they ship
- Prefer Brotli over gzip for compression — better savings, same effort
- Audit images — they're usually the biggest hidden cost in your bundle
