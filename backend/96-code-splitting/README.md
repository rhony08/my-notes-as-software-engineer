# Code Splitting for SPA

Your single-page app works fine in development. One bundle, everything loads, job's done. Then you deploy to production, and the first-load JavaScript clocking in at 3MB+ is causing Lighthouse to scream at you.

Here's the thing — users don't need *everything* upfront. They need what's on screen *right now*. Code splitting is how you stop shipping the entire app when someone just wants to see the login page.

## The Problem: One Big Bundle

React, Vue, Angular — they all start the same way. One entry point, one massive bundle. Every route, every component, every library gets shipped together:

```
src/
  components/
    Header.tsx
    Dashboard.tsx        # heavy charts library
    AdminPanel.tsx       # huge form builders
    UserProfile.tsx
    Settings.tsx
    SomeRandomFeature.tsx
```

Without code splitting, the user downloads **everything** — including admin panels they may never visit.

```javascript
// ❌ NEVER DO THIS - one import, everything loaded
import { Dashboard, AdminPanel, Settings, UserProfile } from './components';
```

## Dynamic Imports: The Foundation

The core mechanism is JavaScript's dynamic `import()`. Instead of a static import at the top of the file, you load modules on demand:

```javascript
// ✅ Safe approach - lazy load on route change
const Dashboard = React.lazy(() => import('./components/Dashboard'));
const AdminPanel = React.lazy(() => import('./components/AdminPanel'));
```

Webpack, Vite, and other bundlers detect these `import()` calls and automatically split them into separate chunks. The user gets a smaller initial bundle, and additional chunks load as needed.

## Route-Based Splitting (The Easiest Win)

For 90% of SPAs, splitting by route is the most bang for your buck. Users navigate to a route -> load that route's code.

### React Example with React Router

```javascript
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import React, { Suspense } from 'react';

// Lazy-loaded route components
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Settings = React.lazy(() => import('./pages/Settings'));
const AdminPanel = React.lazy(() => import('./pages/AdminPanel'));
const Analytics = React.lazy(() => import('./pages/Analytics'));

// Don't forget Suspense - it's required
function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="/admin" element={<AdminPanel />} />
          <Route path="/analytics" element={<Analytics />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

> **Why Suspense matters:** Without it, React can't show anything while the chunk is loading. You'll either get a blank screen or an error. The `fallback` gives users visual feedback.

### Vue Example

```javascript
const routes = [
  {
    path: '/dashboard',
    component: () => import('./views/Dashboard.vue')
  },
  {
    path: '/settings',
    component: () => import('./views/Settings.vue')
  }
];
```

Vue does this even more cleanly — the dynamic import is right in the route config.

## Component-Level Splitting

Sometimes a page has one particularly heavy component — a rich text editor, a chart, a PDF viewer. Route splitting still loads that component, but you can split even finer.

```javascript
// Before: The entire ArticleEditor (400KB) loads with the route
function CreateArticle() {
  return (
    <div>
      <h1>New Article</h1>
      <ArticleEditor />  // 400KB library - but user might just want to view, not edit!
    </div>
  );
}

// After: Only load editor when user actually starts editing
function CreateArticle() {
  const [isEditing, setIsEditing] = useState(false);

  return (
    <div>
      <h1>New Article</h1>
      {isEditing ? (
        <React.Suspense fallback={<EditorSkeleton />}>
          <ArticleEditor /> // 400KB loaded only when needed
        </React.Suspense>
      ) : (
        <button onClick={() => setIsEditing(true)}>Edit Article</button>
      )}
    </div>
  );
}
```

Same route, much smaller initial load. The editor chunk only downloads when someone clicks "Edit."

## Preloading: Don't Wait Too Long

Route splitting has a trade-off: navigating to a new page means waiting for the chunk to download. The fix is **preloading** — start loading chunks *before* the user clicks.

```javascript
// Preload on hover - tiny delay, huge UX difference
<Link
  to="/dashboard"
  onMouseEnter={() => import('./pages/Dashboard')} // starts loading!
  onFocus={() => import('./pages/Dashboard')}      // handle keyboard users too
>
  Dashboard
</Link>
```

When the user hovers over a link, the chunk starts loading. By the time they actually click, the code is already in cache.

### Chunk Loading Strategies

| Strategy | When to Use | Trade-off |
|----------|------------|-----------|
| **Route-based** | Most apps, start here | One chunk per route, can still be large |
| **Component-level** | Heavy components (editors, charts) | More network requests, more overhead |
| **Preload on hover** | Primary navigation | Slightly more bandwidth, better UX |
| **Preload after idle** | Likely-next-page prediction | Complex logic, hard to get right |
| **Eager (no split)** | Critical, above-the-fold code | No lazy load benefit |

## The Vendor Bundle Trap

A common mistake: splitting your app code but not your dependencies.

```javascript
// ❌ This defeats the purpose
// webpack.config.js
splitChunks: {
  chunks: 'all'
}
```

Without proper config, React, Lodash, Moment.js and 50 other libraries all end up in a single `vendor.chunk.js` that's 1.2MB. Not great.

```javascript
// ✅ Better: split vendor by category
splitChunks: {
  chunks: 'all',
  cacheGroups: {
    react: {
      test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
      name: 'vendor-react',
      chunks: 'all',
    },
    ui: {
      test: /[\\/]node_modules[\\/](antd|@mui)[\\/]/,
      name: 'vendor-ui',
      chunks: 'all',
    },
    // Everything else
    vendors: {
      test: /[\\/]node_modules[\\/]/,
      name: 'vendor-other',
      chunks: 'all',
    },
  },
}
```

Why split vendors? Better caching. If you update your app code, the vendor chunks stay cached. Users don't re-download React every deployment.

## Measuring What Matters

You can't optimize what you don't measure. Before and after code splitting:

| Metric | Before (1 bundle) | After (split) | Impact |
|--------|------------------|---------------|--------|
| Initial JS | 2.8 MB | 180 KB | 93% smaller |
| Time to Interactive | 6.2s | 1.8s | 71% faster |
| Lighthouse score | 42 | 88 | +46 points |

Tools that help:
- **Webpack Bundle Analyzer** — see what's in each chunk
- **React DevTools Profiler** — check component load timings
- **Lighthouse CI** — catch regressions before merge

## When NOT to Code Split

Code splitting isn't free. Every chunk is another network request, another round trip, another chance for something to fail.

Don't split when:
- **Your app is tiny (< 50KB gzipped).** The overhead of chunking and loading screens costs more than it saves.
- **The component is needed immediately.** If it's above the fold and critical for initial render, keep it in the main bundle.
- **Your users are on fast internal networks.** An enterprise app on a corporate LAN doesn't benefit as much.
- **The chunk is smaller than the HTTP overhead.** A 2KB chunk isn't worth the round trip.

## The Takeaway

- **Start with route-based splitting** — it's the easiest win with the least complexity
- **Only split what matters** — tiny components aren't worth the overhead
- **Preload on hover** — kills the navigation delay with almost no downside
- **Split vendor bundles** — better caching, faster deployments
- **Measure before and after** — data beats opinions
- **Don't over-split** — every chunk request has cost
