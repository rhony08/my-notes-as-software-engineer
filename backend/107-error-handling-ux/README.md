# Error Handling UX: Design for When Things Go Wrong

Your app works perfectly — right up until it doesn't. The API times out, the payment webhook fails, the CDN drops a bundle. And in that moment, the user isn't looking at your carefully crafted UI. They're looking at whatever you shipped for the failure case. For most apps, that's a sad gray box with "Something went wrong" and a button that doesn't do anything.

Here's the uncomfortable truth: **users judge your product by how it fails, not how it works.** A flawless happy path is the baseline. But an error experience that explains what happened and gets them back on track? That's what turns a frustrated user into a loyal one. This article covers the error states you need, how to write error messages humans actually read, and when to use inline errors vs toasts vs full screens.

## The Four States Every Screen Needs

Here's a mental model that's saved me more times than I can count: every screen in your app has four states — **loading, empty, error, and success**. Teams build the success state, sprinkle in a loading spinner, and call it done. Then the first real user hits an empty list and sees a blank page, and nobody knows if it's broken or if there's just no data.

```
// ❌ Assuming data always exists
function OrdersList({ orders }) {
  return (
    <ul>
      {orders.map((o) => <OrderCard key={o.id} order={o} />)}
    </ul>
  );
}

// ✅ Explicit states for every outcome
function OrdersList({ orders, status, error, onRetry }) {
  if (status === 'loading') return <SkeletonRows />;
  if (status === 'error') return <ErrorState error={error} onRetry={onRetry} />;
  if (!orders.length) return <EmptyState message="No orders yet" />;
  return <ul>{orders.map((o) => <OrderCard key={o.id} order={o} />)}</ul>;
}
```

The empty state is the one people forget most. "You have no orders yet" with a CTA to browse is a feature. A blank white page is a bug report waiting to happen. And for error states — more on those below — the retry button isn't optional. It's the whole point.

## Error Boundaries: Stop the Domino Effect

In React, a single uncaught error in one component used to take down the entire app — white screen of death, React unmounts everything. One bad render in a comments widget and your whole product page is gone. That's not a bug in the comments widget anymore, that's a bug in your *entire* app.

```jsx
// ❌ No boundary — one throw and the whole tree unmounts
function App() {
  return (
    <Header />
    <ProductPage />  // if this throws: white screen for EVERYTHING
    <Footer />
  );
}

// ✅ Isolate failures so the rest of the app keeps working
function App() {
  return (
    <Header />
    <ErrorBoundary fallback={<WidgetCrash />}>
      <ProductPage />
    </ErrorBoundary>
    <Footer />
  );
}
```

```jsx
// A minimal class-based ErrorBoundary (React still requires class components here)
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };  // re-render with fallback, don't crash
  }

  componentDidCatch(error, info) {
    reportErrorToSentry(error, info);  // log it — silently swallowing is worse
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

Two things matter here. First, **scope your boundaries to the right size** — wrapping the whole app means one widget crash blanks everything anyway; wrapping too granularly means a thousand tiny fallbacks to maintain. Second, log the error to your monitoring. An error boundary that swallows the problem and never tells anyone is how bugs live in production for months.

## What Actually Goes in an Error Message

"Something went wrong" is the most useless sentence in software. It tells the user nothing: what went wrong, whether it's their fault, whether it'll happen again, or what to do about it. A good error message answers three questions:

1. **What happened?** — in plain language, not error codes
2. **Why did it happen?** — is it on their end or yours?
3. **What can they do about it?** — retry, fix input, contact support

```
❌ "Error 500: Internal Server Error"
❌ "Something went wrong. Please try again later."
❌ "An unexpected error occurred (code: 0x7F3A)."

✅ "We couldn't load your payment history.
   This is on our end — it's not something you did.
   [Try again] [Contact support]"
```

Notice the difference in ownership. If it's your server, say so. "This is on our end" builds trust because it's honest. Users blame themselves when things fail — "did I do something wrong?" — and a message that takes responsibility removes that anxiety. And never, ever show a raw error code to a non-technical user. Log it internally, show them the friendly version.

## Inline, Toast, or Full Screen?

Not all errors are created equal, and neither are the ways to surface them. Pick based on severity and context:

| Type | Good for | Bad for | Example |
|------|----------|---------|---------|
| **Inline** | Form fields, action-specific failures | System-wide problems | "That email is already registered" |
| **Toast/banner** | Transient, non-blocking failures | Critical errors that need attention | "Couldn't save — check your connection" |
| **Full page** | Fatal, unrecoverable states | Minor hiccups | 404, "app can't load", session expired |

The trade-off that nobody talks about: **toasts are easy to ignore.** They auto-dismiss, they're off to the side, and users trained on 2010s web have learned to tune them out. So the rule is: if the user *must* do something about the error, don't put it in a toast. If the error is destructive (a failed save, a lost update), a toast is actively dangerous — the user assumes it worked and moves on.

```js
// ❌ Critical failure in a toast that vanishes in 4 seconds
toast.success('Order submitted!');   // it actually failed silently...

// ✅ Make destructive failures impossible to miss
if (orderSaveFailed) {
  showBanner({
    tone: 'error',
    sticky: true,             // doesn't auto-dismiss
    message: 'Your order was NOT submitted.',
    action: { label: 'Review order', onClick: openCart }
  });
}
```

For form errors, inline next to the field beats anything else — that's where the user's eyes are. We covered the full pattern in the form validation notes, but the short version: error goes next to the field, not stacked at the top of the page.

## Network Failures: The One You'll Hit Most

Client-side, the most common error isn't a bug — it's the network. Flaky wifi, a train going through a tunnel, the user's VPN acting up. And here's the thing: **most network failures are transient.** A retry that succeeds in 2 seconds is a much better experience than a scary error screen for a problem that already resolved itself.

```
// ❌ Kill the whole flow on the first failure
async function submitOrder() {
  try {
    await api.post('/orders', order);
    navigate('/success');
  } catch (e) {
    showErrorPage();  // one blip of connectivity and the user is staring at this
  }
}

// ✅ Distinguish "try again" failures from real ones
async function submitOrder() {
  try {
    await api.post('/orders', order, { retry: 3 });  // idempotent, safe to retry
    navigate('/success');
  } catch (e) {
    if (e.isNetworkError) {
      showInlineError('Connection lost. Your order is saved — retry when you're back online.');
    } else {
      showFullError(e);  // 4xx/5xx — genuinely needs attention
    }
  }
}
```

A few patterns worth stealing from the pros:

- **Retry with backoff** — automatic retry on network blips, but don't hammer the server (that's the backend's job to rate-limit, but don't be the client that abuses it)
- **Preserve user input** — if a form submit fails, the user's typed data must survive the retry. Losing a 400-word support message to a network error is rage-quit territory
- **Offline detection** — `navigator.onLine` + the `online`/`offline` events let you tell users "you're offline" *before* they try something and fail
- **Idempotency on retries** — if you're retrying a POST, make sure double-submission is safe (idempotency keys, or disable the button while in-flight). Retrying a payment twice is the worst error UX there is

## 404s and "This Page Doesn't Exist" Done Right

A 404 is a dead end. The user asked for something that isn't there — and if you just show "Page not found" with no way forward, you've handed them an exit. Good 404 pages do three things: admit the page is gone, offer a path back, and search for what they wanted.

```
❌ "404 Not Found"
❌ A 404 page with no links — a literal dead end

✅ "We couldn't find that page.
   [Go to dashboard] [Search] [Browse popular articles]"
```

The same logic applies to empty search results, expired links, and deleted items. Every dead end needs a way back. The worst UX pattern in the industry: dead ends with a browser back button as the only escape.

## When Errors Are Actually Features

Not every failure needs to be disguised. Sometimes the error state *is* the product — the "we couldn't find flights on that date" screen that suggests nearby airports, or the "you're offline, here's your downloaded content" state in streaming apps. Netflix's offline mode, Spotify's "you're offline" with cached playlists — these are error states designed so well they're features.

The lesson: **think about what the user's goal is, not just what broke.** If their goal was "read an article" and the network is down, showing cached content serves the goal. If their goal was "book a flight" and the search failed, showing alternative dates serves the goal. Error UX is just UX with constraints.

## Actionable Takeaways

- **Ship all four states** — loading, empty, error, success — for every data-driven screen. Empty states aren't optional.
- **Wrap components in error boundaries** at meaningful scopes, and wire `componentDidCatch` to your error monitoring. Never let one widget kill the page.
- **Write error messages that answer what / why / what now**, take ownership when it's your fault, and never show raw codes to users.
- **Match the surface to the severity** — inline for forms, sticky banners for destructive failures, full pages for fatal states. Toasts are for the unimportant stuff.
- **Treat network errors as transient** — retry with backoff, preserve user input, make retries idempotent.
- **Never leave a dead end** — every 404, empty state, and failure screen needs a path forward.
