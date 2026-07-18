# State Management Basics

Ever clicked a button in a web app and got a stale page? Or filled out a form, navigated away, and lost everything? That's state management (or lack thereof) biting you in the ass.

Here's the deal: every frontend app needs to know *what's happening right now*. Is the user logged in? Are we loading data? Did the API call fail? What's in the shopping cart? Without a plan for managing this stuff, your UI will show wrong data, components won't stay in sync, and users will have a bad time.

## What Even Is "State"?

State is just data that changes over time. In a frontend app, it's basically everything that isn't static HTML:

- **Server state** — data from an API (user profile, product list)
- **UI state** — which modal is open, current tab selected
- **Form state** — what the user typed before pressing submit
- **Auth state** — is the user logged in? what's their token?
- **Route state** — which page are we on? any URL params?

The difference between a decent app and a dumpster fire is how well you track and update these pieces.

## The Naive Approach (and Why It Breaks)

```javascript
// ❌ Global variables — seems convenient, but...
let currentUser = null;
let cartItems = [];
let isLoading = false;

function addToCart(item) {
  cartItems.push(item);
  // Problem: the UI doesn't know cartItems changed
  // You'd need to manually re-render everything
}
```

This works in a 1998 GeoCities page. In a modern app with dozens of components, global variables become a spaghetti nightmare. Component A updates `cartItems`, but Component B (showing the cart badge) has no idea. You'll end up writing janky event-emitter glue code that nobody wants to maintain.

## The Core Problem: Keeping UI in Sync

The fundamental challenge of state management is this: **when state changes, the UI needs to reflect that change**. Every approach to state management is just a different answer to that problem.

Here's what actually happens in a well-managed app:

```
User clicks "Add to Cart"
  → State: cartItems = ['item-42']
  → Notification sent to subscribed components
  → Cart icon badge re-renders to show "1"
  → Add button changes to "In Cart"
  → Checkout button becomes enabled
```

All of that happens automatically. You don't manually tell each component to update — the state management layer handles it.

## Lifting State Up

One of the first patterns you'll encounter: **lift state up to the closest common ancestor**.

Say you have two sibling components — a search input and a results list. They need to share the search query:

```javascript
// ❌ Each component manages its own state
function SearchInput() {
  const [query, setQuery] = useState('');
  // Results list can't access query!
}

function ResultsList() {
  const [results, setResults] = useState([]);
  // This can't react to query changes from SearchInput
}

// ✅ Lift state to parent
function SearchPage() {
  const [query, setQuery] = useState('');  // state lives here
  const [results, setResults] = useState([]);

  return (
    <>
      <SearchInput query={query} onQueryChange={setQuery} />
      <ResultsList results={results} />
    </>
  );
}
```

This works well for small component trees. But what happens when your app has 50 components spread across the page, and they all need to know the current user?

## Prop Drilling — the Pain of Deep Trees

When state lives at the top and you thread it through layers of components that don't even use it:

```
App → Dashboard → MainContent → Sidebar → UserProfile
  ↑                                                                 ↓
  └────── state threaded through EVERYTHING ──────────────────────┘
```

Those middle components (`Dashboard`, `MainContent`, `Sidebar`) don't need `currentUser` — they're just forced to pass it along. This is **prop drilling**, and it's a code smell.

```javascript
// ❌ Prop drilling — Sidebar doesn't need user but has to pass it
function Sidebar({ currentUser, onLogout }) {
  return (
    <div className="sidebar">
      <NavigationMenu />
      <UserProfile currentUser={currentUser} onLogout={onLogout} />
    </div>
  );
}
```

## Solutions to Prop Drilling

There are three common approaches:

| Approach | How it Works | When to Use |
|----------|-------------|-------------|
| **Context** | Provides state to entire subtree without passing through props | App-wide concerns (theme, auth, locale) |
| **Flux/CQRS pattern** | Single store, unidirectional data flow | Medium-to-large apps with complex state |
| **Signals/observables** | Fine-grained reactive updates | Performance-critical, frequent updates |

### Context API (React Example)

```javascript
// Create a context
const AuthContext = React.createContext(null);

// Provider wraps the app, makes state available everywhere
function App() {
  const [currentUser, setCurrentUser] = useState(null);

  return (
    <AuthContext.Provider value={{ currentUser, setCurrentUser }}>
      <Dashboard />
    </AuthContext.Provider>
  );
}

// Any nested component can consume it — no prop drilling
function UserProfile() {
  const { currentUser } = useContext(AuthContext);
  // ...direct access to currentUser
}
```

**Trade-off:** Context isn't a silver bullet. Every component that consumes context re-renders when the value changes, even if the specific piece it reads didn't change. For high-frequency updates (like real-time cursors in a collaborative editor), this causes performance problems.

### State Libraries

When apps grow beyond a certain size, context alone isn't enough. Libraries like Redux, Zustand, or Pinia (Vue) give you:

- A single store with predictable updates
- DevTools for debugging state changes
- Middleware for side effects (logging, API calls)
- Selectors for efficient re-renders

```javascript
// Zustand example — simple, minimal boilerplate
import { create } from 'zustand';

const useCartStore = create((set) => ({
  items: [],
  total: 0,

  addItem: (item) => set((state) => ({
    items: [...state.items, item],
    total: state.total + item.price,
  })),

  removeItem: (id) => set((state) => ({
    items: state.items.filter((i) => i.id !== id),
    total: state.total - state.items.find((i) => i.id === id).price,
  })),
}));

// Component usage
function CartBadge() {
  const items = useCartStore((state) => state.items);
  // Only re-renders when items changes — not when total changes
  return <span>{items.length} items</span>;
}
```

The key improvement: **selectors**. Your component only re-renders if the specific slice of state it reads actually changed. That's way more efficient than re-rendering every context consumer.

## A Mental Model That Helps

Think of state like this:

- **Local state** is a notepad on your desk — scratch notes for one component
- **Lifted state** is a whiteboard shared by a team — coordinated between a few components
- **Global store** is the company wiki — everyone reads from the same source of truth

Don't put everything in the global store. That's like using the company wiki to track what you had for lunch. Keep local stuff local, share what needs sharing, and only globalize what truly matters app-wide.

## Common Pitfalls

**Not distinguishing between derived and stored state.** If you can compute something from existing state, do that instead of storing it separately.

```javascript
// ❌ Storing derived data separately
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);
// Now total can get out of sync with items!

// ✅ Compute it
const [items, setItems] = useState([]);
const total = items.reduce((sum, i) => sum + i.price, 0);
```

**Mutating state directly.** This is the #1 bug in junior frontend code.

```javascript
// ❌ Mutation — React can't detect this change
state.items.push(newItem);
setState(state);  // Re-render won't happen

// ✅ Immutable update
setState({
  ...state,
  items: [...state.items, newItem],
});
```

**Over-engineering too early.** You don't need Redux for a todo app. Start with local state. Lift when it hurts. Reach for a library when prop drilling becomes painful.

## Takeaways

- State is just data that changes over time — but managing it well is what separates good apps from broken ones
- Lift state to the closest common ancestor, but avoid prop drilling through too many layers
- Use context for app-wide concerns, state libraries for complex apps with many consumers
- Derive what you can compute; store only the minimal necessary data
- Never mutate state directly — always create new objects
- Start simple and add complexity only when you feel the pain of the current approach
