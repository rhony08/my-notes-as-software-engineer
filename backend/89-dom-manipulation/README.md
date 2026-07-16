# Efficient DOM Manipulation

The DOM is slow. Like, *painfully* slow compared to plain JavaScript operations. Reading a property off a DOM node costs hundreds of times more than reading a variable. Writing to the DOM triggers layout recalculations that can freeze your UI. And the worst part? Most of the time you're doing way more DOM work than you realize.

But here's the thing — the DOM isn't slow because it's badly designed. It's slow because it has a hard job: reconciling a tree of elements with the browser's rendering pipeline, painting pixels to screen, and doing it all 60 times per second. Treat it with respect, and it'll perform. Abuse it with a thousand individual updates, and your users get jank.

## Reflows and Repaints: The Cost You Don't See

Every time you change something visible, the browser has to recalculate positions (reflow) or redraw pixels (repaint). These are expensive.

```javascript
// ❌ Three separate writes — three reflows
const el = document.getElementById('card')
el.style.width = '200px'
el.style.height = '300px'
el.style.marginTop = '20px'

// ✅ One batch write — one reflow
el.style.cssText = 'width: 200px; height: 300px; margin-top: 20px'
// Or use classList for more maintainable code
el.classList.add('card--expanded')
```

**Reflow** happens when you change layout-affecting properties (width, height, margin, position, font-size). It's the expensive one because the browser has to recalculate positions for the changed element *and everything below it in the DOM tree*.

**Repaint** happens for visual-only changes (color, background, visibility). Cheaper than reflow, but still not free — especially on low-powered devices.

### What Triggers a Reflow?

Almost everything you do with the DOM. Reading certain properties forces a reflow too, because the browser needs up-to-date values:

```javascript
// ❌ Reading layout properties forces a reflow
const height = element.offsetHeight  // forces reflow
element.style.height = '200px'       // forces reflow
const width = element.offsetWidth    // forces reflow AGAIN

// ✅ Batch reads together, batch writes together
// Read first
const height = element.offsetHeight
const width = element.offsetWidth
// Then write
element.style.height = '200px'
element.style.width = '300px'
```

This is called **layout thrashing** — alternating reads and writes that force the browser to reflow over and over. It's the #1 cause of janky DOM interactions.

## Batch Updates with Document Fragments

Adding multiple elements to the DOM one by one is a classic performance killer.

```javascript
// ❌ Adding items one at a time — triggers reflow for each
const list = document.getElementById('items')
for (let i = 0; i < 1000; i++) {
  const li = document.createElement('li')
  li.textContent = `Item ${i}`
  list.appendChild(li)  // reflow every time
}

// ✅ Using a document fragment — one reflow at the end
const list = document.getElementById('items')
const fragment = document.createDocumentFragment()
for (let i = 0; i < 1000; i++) {
  const li = document.createElement('li')
  li.textContent = `Item ${i}`
  fragment.appendChild(li)  // no reflow — not in the DOM yet
}
list.appendChild(fragment)  // single reflow
```

Document fragments are lightweight — they don't have a parent, so adding children to them doesn't cost anything. You build your structure off-screen, then inject it in one shot.

Same pattern works for hiding an element while you work on it:

```javascript
// ❌ Modifying a visible element triggers layout updates for each change
card.style.display = 'block'
card.style.width = '400px'
card.innerHTML = '...'

// ✅ Detach, modify, reattach
const parent = card.parentNode
const next = card.nextSibling
parent.removeChild(card)  // remove from layout tree
// Now modify freely — no reflow cost
card.style.width = '400px'
card.innerHTML = '...'
parent.insertBefore(card, next)  // single reflow when reattached
```

**Trade-off:** Detaching/reattaching is great for bulk work but can cause a flash if users are already looking at the element. Use `visibility: hidden` as an alternative — it hides visually without removing from flow (only triggers repaint, not reflow).

## Event Delegation: One Handler to Rule Them All

Attaching event listeners to every element in a list is a waste.

```javascript
// ❌ 1000 event listeners
document.querySelectorAll('.list-item').forEach(item => {
  item.addEventListener('click', handleClick)
})

// ✅ One event listener, thanks to bubbling
document.querySelector('.list').addEventListener('click', (e) => {
  const item = e.target.closest('.list-item')
  if (!item) return // not a list item click
  handleClick(item)
})
```

Event delegation works because most DOM events bubble up from the target element to the root. Instead of attaching a handler to 1000 elements, attach one to their parent and use `event.target` to figure out what was clicked.

| Approach | Listeners | Memory | Dynamic Elements |
|----------|-----------|--------|-----------------|
| Direct | N | Higher | Not supported |
| Delegation | 1 | Lower | Supported automatically |

The delegation approach also handles dynamically added elements — since the listener is on the parent, it catches clicks on items added later without any extra work.

## Reading and Writing Layout Separately

This is the most practical optimization you can make. **Batch all your reads, then batch all your writes.**

```javascript
// ❌ Layout thrashing — read/write/read/write
for (const box of boxes) {
  const width = box.offsetWidth  // read
  box.style.width = width * 2 + 'px'  // write
  const height = box.offsetHeight  // read (forces reflow again!)
  box.style.height = height * 2 + 'px'  // write
}

// ✅ Batch reads, then batch writes
const widths = boxes.map(box => box.offsetWidth)
const heights = boxes.map(box => box.offsetHeight)

boxes.forEach((box, i) => {
  box.style.width = widths[i] * 2 + 'px'
  box.style.height = heights[i] * 2 + 'px'
})
```

If you can't batch manually, use `requestAnimationFrame` to schedule work:

```javascript
// schedule a write in the next frame
requestAnimationFrame(() => {
  element.style.transform = 'translateX(100px)'
})
```

## innerHTML vs createElement: Choose Your Weapon

`innerHTML` is convenient but has baggage.

```javascript
// ❌ innerHTML destroys and recreates everything
list.innerHTML = `
  <li class="item" data-id="1">Item 1</li>
  <li class="item" data-id="2">Item 2</li>
`

// ✅ createElement preserves existing references
const items = [
  { id: 1, text: 'Item 1' },
  { id: 2, text: 'Item 2' },
]

const fragment = document.createDocumentFragment()
items.forEach(item => {
  const li = document.createElement('li')
  li.className = 'item'
  li.dataset.id = item.id
  li.textContent = item.text
  fragment.appendChild(li)
})
list.appendChild(fragment)
```

**Why not innerHTML?** It serializes the existing content to HTML, parses the new string, and replaces everything. That destroys event listeners, form state, and references. It's also a security risk if you include user-generated content — XSS waiting to happen.

**But it's not always bad.** For initial render of static content, `innerHTML` can actually be faster because the browser's HTML parser is optimized C++ code, while `createElement` requires more JS function calls.

```javascript
// innerHTML is fine for static initial render
const container = document.getElementById('app')
container.innerHTML = '<h1>Hello World</h1><p>Static content</p>'

// createElement/style when you need to update later
const btn = document.createElement('button')
btn.textContent = 'Click me'
btn.addEventListener('click', handleClick)
container.appendChild(btn)
```

## Modern DOM APIs That Make Life Easier

The DOM API has gotten better. Use these instead of the old ways:

```javascript
// Old way — verbose, easy to mess up
const div = document.createElement('div')
div.className = 'card'
div.setAttribute('data-id', '123')
div.style.backgroundColor = 'blue'

// Modern — cleaner, faster to write
const div = Object.assign(document.createElement('div'), {
  className: 'card',
  dataset: { id: '123' },
  style: { backgroundColor: 'blue' },
})

// Insert adjacent — better than innerHTML for partial updates
list.insertAdjacentHTML('beforeend', '<li>New item</li>')

// replaceChildren — atomic swap
container.replaceChildren(...newElements)
```

The `insertAdjacentHTML` is a hidden gem. It parses HTML just like innerHTML, but doesn't touch existing children:

- `beforebegin` — before the element itself
- `afterbegin` — inside the element, before first child
- `beforeend` — inside the element, after last child
- `afterend` — after the element itself

## When Virtual DOM Helps (and When It Doesn't)

Frameworks like React and Vue use a virtual DOM to batch updates. The idea: you describe what the UI should look like, and the framework figures out the minimal set of DOM operations to get there.

```javascript
// React example — you just describe the state
function ItemList({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.text}</li>
      ))}
    </ul>
  )
}
```

**When it helps:**
- Complex UIs with lots of state
- Apps where you don't want to manually manage DOM updates
- Team projects where declarative code is easier to reason about

**When it doesn't:**
- Simple pages with minimal interactivity
- Performance-critical animations (raw DOM or canvas still wins)
- Micro-interactions where framework overhead dominates

**The reality check:** Virtual DOM overhead is real. A React diff on a large tree takes time. For most apps it doesn't matter. For apps with thousands of elements (chat lists, data tables, feeds), you'll need manual optimizations anyway (windowing, memoization, skipping reconciliation).

## Takeaways

- **Batch your DOM operations** — use document fragments, detach-edit-reattach, or `replaceChildren` to minimize reflows
- **Separate reads from writes** — layout thrashing is the #1 cause of jank
- **Use event delegation** — one listener on a parent beats N listeners on children
- **Prefer `createElement` over `innerHTML`** for dynamic content — you keep references and avoid XSS
- **`insertAdjacentHTML`** is the unsung hero for targeted HTML injection without destroying existing elements
- **Virtual DOM is a tool, not magic** — it abstracts away DOM management but adds its own cost
- **Real DOM APIs are fine** — you don't need a framework to manipulate the DOM efficiently, just know the patterns
