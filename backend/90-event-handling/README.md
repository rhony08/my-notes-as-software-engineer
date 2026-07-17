# Event Handling Patterns

Your app's button click fires 17 handlers, the dropdown closes before its animation finishes, and somewhere a memory leak is slowly growing because nobody unregistered an event listener. Event handling sounds simple—until it isn't.

The DOM event system is more than `element.addEventListener`. Understanding how events flow through the page, how to manage complexity, and how to avoid memory leaks separates a solid frontend from one that falls apart under real use.

---

## The Event Flow: Capturing vs Bubbling

Most developers only see the "bubbling" phase—events travel from the target up through parent elements. But there's a capture phase that runs first.

When you click a button inside a div:

```
Capture phase:    window → document → body → div → button
Target phase:                             button
Bubble phase:     button → div → body → document → window
```

```javascript
// Third parameter controls the phase
element.addEventListener('click', handler, false);  // false = bubbing (default)
element.addEventListener('click', handler, true);   // true = capturing
```

The third parameter can also be an options object:

```javascript
element.addEventListener('click', handler, {
  capture: false,
  once: true,        // auto-remove after first invocation
  passive: true      // promise not to call preventDefault()
});
```

**When you'd use capturing:** Intercepting events before they reach a child. Say you have a drag overlay that needs to swallow all mouse events.

**When bubbling trips you up:** Nested modals. Clicking inside the inner modal registers on the outer modal too. That's why you see `e.stopPropagation()` everywhere.

```javascript
// ❌ Stops propagation unconditionally
function handleClick(e) {
  e.stopPropagation();
  // This also blocks other handlers on the same element
}

// ✅ Be more specific - prevent default when needed
function handleClick(e) {
  if (e.target !== e.currentTarget) return;
  // Only act when target is the element itself
}
```

---

## Event Delegation: The Performance MVP

If you have 100 list items, each with their own click handler, that's 100 event registrations. Or you put one handler on the parent and let bubbling do the work.

```javascript
// ❌ 100 event listeners
document.querySelectorAll('.list-item').forEach(item => {
  item.addEventListener('click', handleItemClick);
});

// ✅ 1 event listener
document.querySelector('.list-container').addEventListener('click', (e) => {
  const item = e.target.closest('.list-item');
  if (!item) return;  // clicked something else inside the container
  handleItemClick(item);
});
```

**Where delegation saves you:**

- **Dynamic content** — items added after page load automatically work without rebinding
- **Large lists** — thousands of rows without thousands of listeners
- **Memory footprint** — fewer closures tracking DOM references

**Trade-off:** You lose access to the exact event phase details. And `closest()` does a DOM traversal, so for deeply nested structures, the check adds some cost. Usually negligible—but know it's there.

---

## Debounce and Throttle: Taming High-Frequency Events

`scroll`, `resize`, `mousemove`, `keydown` — these fire dozens of times per second. Running expensive operations on each tick kills performance.

| Technique | What it does | When it fires |
|-----------|-------------|---------------|
| **Debounce** | Waits for a pause, then fires once | After X ms of inactivity |
| **Throttle** | Ensures at most one fire per interval | Every X ms at most |
| **requestAnimationFrame** | Syncs with browser paint | ~60fps, best for visual updates |

```javascript
// Debounce - waits for the user to stop typing
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

const handleSearch = debounce(async (query) => {
  const results = await fetchSearch(query);
  renderResults(results);
}, 300);

// Throttle - rate-limits scroll handler
function throttle(fn, limit) {
  let inThrottle = false;
  return (...args) => {
    if (inThrottle) return;
    inThrottle = true;
    fn(...args);
    setTimeout(() => inThrottle = false, limit);
  };
}

const handleScroll = throttle(() => {
  updateScrollProgress();
}, 100);
```

**When to use each:**

- **Search autocomplete → debounce** (fire after they stop typing)
- **Infinite scroll → throttle** (check position periodically while scrolling)
- **Animation → `requestAnimationFrame`** (let the browser tell you when to update)

---

## One-Time Handlers: Use `{ once: true }`

Need to handle something exactly once—like a first interaction or a one-off initialization?

```javascript
// ❌ Manual removal
function handleFirstClick(e) {
  // setup stuff
  document.removeEventListener('click', handleFirstClick);
}
document.addEventListener('click', handleFirstClick);

// ✅ Built-in once
document.addEventListener('click', initApp, { once: true });
```

The `{ once: true }` option auto-removes the listener after the first invocation. It's cleaner and eliminates a common source of bugs where the removal logic forgets to reference the exact same function.

---

## Memory Leaks: The Silent App Killer

This is the one that'll bite you in production. Every `addEventListener` creates a reference from the DOM element to your callback function. If the element is removed from the DOM but the listener isn't cleaned up, the closure (and everything it references) stays in memory.

```javascript
function mountComponent() {
  const data = fetchExpensiveData();  // large object

  // ❌ This closure keeps 'data' alive forever
  window.addEventListener('resize', () => {
    adjustLayout(data);
  });
}

// Even after unmountComponent() removes the DOM node,
// the resize handler + data object = memory leak
```

**The fix:** Always clean up. In frameworks, use lifecycle hooks:

```javascript
// Vanilla JS cleanup pattern
function mountComponent(container) {
  const data = fetchExpensiveData();

  function handleResize() {
    adjustLayout(data);
  }

  window.addEventListener('resize', handleResize);

  // Return a cleanup function
  return () => {
    window.removeEventListener('resize', handleResize);
  };
}

const cleanup = mountComponent(root);
// Later:
cleanup();
```

```javascript
// React
useEffect(() => {
  const handleResize = () => adjustLayout(data);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, [data]);
```

### Common Leak Patterns

| Pattern | Problem | Fix |
|---------|---------|-----|
| Anonymous arrow functions | Can't removeEventListener later | Store reference |
| Forgetting cleanup on unmount | Closure keeps dead component alive | Lifecycle cleanup |
| Delegation on non-removed parents | Parent keeps reference to dead children | Clean up parent handler |
| Timer + event combination | setInterval + handler cross-reference | Clear both on teardown |

---

## Custom Events: Decoupling Components

When components need to talk to each other, custom events are cleaner than passing callbacks through layers.

```javascript
// Dispatch a custom event
const event = new CustomEvent('user:login', {
  detail: { userId: 123, name: 'Alice' },
  bubbles: true  // can be captured by parent elements
});
document.dispatchEvent(event);

// Listen anywhere
document.addEventListener('user:login', (e) => {
  console.log('User logged in:', e.detail.name);
  initializeDashboard(e.detail.userId);
});
```

**Naming convention:** Use namespaced names like `namespace:event-name` to avoid collisions. Your analytics library shouldn't clash with your notification system.

---

## Passive Handlers: Free Performance

Touch and wheel events can block scrolling because the browser waits to see if `preventDefault()` is called. If you know you won't cancel the event, mark it passive:

```javascript
// ❌ Browser must wait for this handler before scrolling
document.addEventListener('touchstart', handleTouchStart);

// ✅ Browser can scroll immediately
document.addEventListener('touchstart', handleTouchStart, { passive: true });
```

This is especially important for scroll-linked effects. On mobile, a non-passive touch handler can make your page feel janky.

---

## What You Actually Need to Remember

- **Use event delegation** when you have many elements or dynamic content—one handler beats 100
- **Clean up your listeners** or you'll leak memory like a sieve
- **Debounce or throttle** high-frequency events—your users' CPUs will thank you
- **`{ passive: true }`** on scroll/touch handlers costs nothing and helps the browser optimize
- **`{ once: true }`** is cleaner than manual `removeEventListener` for one-shot handlers
- **Custom events** let components communicate without tight coupling
- **`stopPropagation()` is a code smell**—it suggests your event flow is fighting against you
