# Lazy Loading Strategies

Your app loads in 3 seconds. That's fine, right? Wrong—your users are on a subway with spotty 3G, or worse, they're on a $50 Android phone with limited RAM. Every kilobyte you ship before they actually *need* it is a tax on their patience and battery.

Lazy loading is the answer: defer loading non-critical resources until they're actually required. Let's look at the strategies that actually work.

## The Core Idea

**Load what's visible. Defer the rest.**

Here's the problem with the opposite approach:

```js
// ❌ Load everything upfront
import HeavyComponent from './HeavyComponent';
import ChartLibrary from './chart-library';
import ImageGalleryViewer from './gallery';

function App() {
  return (
    <Layout>
      <HeavyComponent />
      <ChartLibrary />
      <ImageGalleryViewer />
    </Layout>
  );
}

// Bundle size: 2.3 MB — ouch
```

```js
// ✅ Load only what's needed, defer the rest
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));
const ChartLibrary = React.lazy(() => import('./chart-library'));
const ImageGalleryViewer = React.lazy(() => import('./gallery'));

function App() {
  return (
    <Layout>
      <Suspense fallback={<Loader />}>
        <HeavyComponent />
        {/* These won't load until rendered */}
      </Suspense>
    </Layout>
  );
}

// Initial bundle: 400 KB — big difference
```

## Strategy 1: Image Lazy Loading

The most common and most impactful one. Images make up ~50% of page weight on average.

### Native Lazy Loading (The Easy Win)

```html
<!-- ✅ Modern browsers handle this natively -->
<img src="hero.jpg" loading="lazy" alt="Hero image" class="hero" />

<!-- ❌ Eager loading blocks rendering -->
<img src="everything.jpg" alt="Everything all at once" />
```

`loading="lazy"` defers off-screen images until the user scrolls near them. It's supported in Chrome, Firefox, Edge, and Safari 16.4+. That covers ~95% of users.

**When this is NOT enough:** The native `loading="lazy"` doesn't give you fine-grained control over load order, placeholder states, or fade-in animations. For that, you need Intersection Observer.

### Intersection Observer (The Pro Approach)

```js
// ✅ Intersection Observer gives full control
const lazyImages = document.querySelectorAll('img[data-src]');

const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const img = entry.target;
        // Swap placeholder for real image
        img.src = img.dataset.src;
        img.onload = () => img.classList.add('loaded');
        observer.unobserve(img); // Clean up
      }
    });
  },
  { rootMargin: '200px' } // Start loading 200px before visible
);

lazyImages.forEach(img => observer.observe(img));
```

```js
// ❌ Scroll-based detection is janky
window.addEventListener('scroll', () => {
  lazyImages.forEach(img => {
    if (img.getBoundingClientRect().top < window.innerHeight) {
      img.src = img.dataset.src; // Fires on EVERY scroll event
    }
  });
});
```

The scroll listener approach triggers on every scroll event—even at 60fps. Intersection Observer is callback-based and runs asynchronously, so it doesn't block the main thread.

**Caveat:** Set `rootMargin` to `'200px'` or `'500px'` to start loading images *before* they enter the viewport. Otherwise users see placeholders flickering in as they scroll.

## Strategy 2: Route-Based Lazy Loading

Single-page apps load a massive bundle by default. Route-based splitting fixes that.

```js
// ❌ Single massive bundle for the whole app
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';
import Reports from './pages/Reports';
import Admin from './pages/Admin';
// This loads EVERYTHING, even if user only needs login

// ✅ Load pages as users navigate to them
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Settings = React.lazy(() => import('./pages/Settings'));
const Reports = React.lazy(() => import('./pages/Reports'));

// React Router v6 style
<Routes>
  <Route path="/dashboard" element={
    <Suspense fallback={<PageSkeleton />}>
      <Dashboard />
    </Suspense>
  } />
  <Route path="/settings" element={
    <Suspense fallback={<PageSkeleton />}>
      <Settings />
    </Suspense>
  } />
</Routes>
```

**What breaks without Suspense:** Without a `<Suspense>` boundary, `React.lazy` components will crash at runtime:

```
❌ Error: A React component suspended while rendering...
   Add a <Suspense> fallback to this component.
```

Always wrap lazy components in Suspense with a meaningful fallback—preferably a skeleton loader, not a spinner.

## Strategy 3: Conditional Lazy Loading

Sometimes you don't know what the user needs until they interact.

```js
// ❌ Load a heavy chart library even if user never views charts
import { Chart } from 'chart.js';

function AnalyticsPanel({ showChart }) {
  return (
    <div>
      {showChart && <Chart data={data} />}
    </div>
  );
}
// Chart.js is in your bundle whether the user clicks or not

// ✅ Dynamic import on interaction
function AnalyticsPanel({ showChart }) {
  const [ChartComponent, setChartComponent] = useState(null);

  useEffect(() => {
    if (showChart && !ChartComponent) {
      // Only download chart.js when user actually requests it
      import('chart.js').then(mod => {
        setChartComponent(() => mod.Chart);
      });
    }
  }, [showChart]);

  return (
    <div>
      {showChart && ChartComponent ? <ChartComponent data={data} /> : null}
    </div>
  );
}
```

This pattern works great for:
- Modal dialogs containing rich editors
- "Show more" expandable sections
- Tab panels with different tooling
- User-uploaded media previews

## The Trade-offs (Because There Are Always Trade-offs)

| Strategy | Benefit | Cost |
|----------|---------|------|
| Image lazy loading | Saves bandwidth, faster initial paint | Images pop in as you scroll (layout shift risk) |
| Route-based splitting | Smaller initial JS bundle | Delay on first navigation to each route |
| Conditional loading | Users who never need it never pay | Loading spinner on first interaction |

**Layout shift is the silent killer.** When a lazy-loaded images finally loads and expands into its space, everything below it jumps. Fix this:

```css
/* ✅ Reserve space before the image loads */
.lazy-image-wrapper {
  width: 100%;
  aspect-ratio: 16 / 9;
  background: #f0f0f0; /* Placeholder background */
  overflow: hidden;
}

.lazy-image-wrapper img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

Without explicit dimensions, lazy-loaded images cause Cumulative Layout Shift (CLS) penalties—and your Core Web Vitals score takes a hit.

## The Loading State Problem

Lazy loading means loading states. Handle them:

```jsx
function PageRoutes() {
  return (
    <Routes>
      <Route path="/reports" element={
        // ❌ Just a spinner
        <Suspense fallback={<Spinner />}>
          <Reports />
        </Suspense>
      } />
    </Routes>
  );
}
```

```jsx
function PageRoutes() {
  return (
    <Routes>
      <Route path="/reports" element={
        // ✅ Skeleton that matches page structure
        <Suspense fallback={<ReportSkeleton />}>
          <Reports />
        </Suspense>
      } />
    </Routes>
  );
}
```

Skeleton loaders communicate "something is happening" without the anxiety of a spinning circle. Users scan the skeleton, it fills in, and they barely notice the delay.

## When NOT to Lazy Load

Lazy loading isn't free. Here's when to avoid it:

- **Critical above-the-fold content** — don't lazy load your hero image or primary CTA
- **Small assets** — lazy loading a 2KB icon adds more overhead than it saves
- **SEO-critical images** — search crawlers may not execute JS, so `loading="lazy"` images might not get indexed
- **Slow connections** — paradoxically, lazy loading on slow connections can feel worse because every scroll triggers a new network request

## Actionable Takeaways

- **Start with images.** Add `loading="lazy"` to all below-the-fold images today—it's a one-line performance win
- **Split at route boundaries.** Every "page" in your app should be its own chunk
- **Defer heavy libraries.** If the user needs to click a button before they see a chart, don't load the chart library until they do
- **Reserve space for lazy images** to prevent layout shift
- **Use skeleton loaders** over spinners for Suspense fallbacks
- **Don't lazy load critical above-the-fold assets** — you'll make your initial paint *worse*
