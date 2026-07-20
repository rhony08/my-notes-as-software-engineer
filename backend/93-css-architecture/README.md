# CSS Architecture Patterns

Ever opened a project and found 12,000 lines of CSS in a single file — with class names like `.wrapper`, `.wrapper2`, `.wrapper-new`, and `.wrapper-final-v2`? Yeah, we've all been there.

CSS is deceptively simple. It's easy to write, but incredibly hard to *maintain* at scale. Without a solid architecture, your stylesheets become tangled, unpredictable, and feared by everyone on the team. Changing one thing breaks three others, and `!important` starts popping up like weeds.

Let's talk about the patterns that solve this mess.

## Why CSS Needs Architecture

CSS is global by nature. Every rule you write can affect every element on every page. In a small project, that's manageable. But once you have:

- Multiple developers touching the same styles
- Hundreds of components
- Responsive breakpoints everywhere
- Dark mode, print styles, accessibility overrides

…that global scope becomes your worst enemy. You need conventions, namespacing, and organization to keep things sane.

## BEM: The Workhorse

**BEM** (Block Element Modifier) is the most widely adopted CSS methodology. It's simple, strict, and works in virtually any project.

```
.block {}
.block__element {}
.block--modifier {}
```

- **Block** — the parent component (`.card`)
- **Element** — a child of the block (`.card__title`)
- **Modifier** — a variant state (`.card--featured`)

```css
/* ✅ BEM naming — clear, scoped, maintainable */
.card { }
.card__title { }
.card__description { }
.card--featured { }
.card--featured .card__title { color: gold; }
```

The magic isn't in the syntax — it's in the **specificity**. Every BEM selector has the same specificity (one class), which means you never battle inheritance. No more "I'll add another class and hope it overrides."

```css
/* ❌ Deep nesting hell — fragile and hard to override */
.card { }
.card .header .title { }
.card .header .title.featured { }

/* ✅ BEM — flat, predictable specificity */
.card__title { }
.card__title--featured { }
```

**Trade-off:** Your HTML gets verbose. `<div class="card__title card__title--featured">` is ugly but maintainable. Some people hate how it looks — I think it beats debugging specificity wars at 2am.

## SMACSS: Categories for Sanity

**SMACSS** (Scalable and Modular Architecture for CSS) by Jonathan Snook organizes styles into five categories:

| Category | Prefix | What Goes Here |
|----------|--------|----------------|
| Base | (none) | Reset, typography defaults |
| Layout | `.l-` | Grid, header, footer, sidebar |
| Module | (component name) | Cards, buttons, navigation |
| State | `.is-` | `.is-active`, `.is-hidden`, `.is-loading` |
| Theme | `.theme-` | Color schemes, dark mode |

```scss
/* Base — element selectors only, no classes */
body { font-family: system-ui; }
a { color: blue; }

/* Layout */
.l-container { max-width: 1200px; margin: 0 auto; }

/* Module — reusable component */
.media { display: flex; }
.media__image { flex: 0 0 auto; }
.media__body { flex: 1; }

/* State */
.is-active { font-weight: bold; }
.is-loading { opacity: 0.5; pointer-events: none; }
```

SMACSS is less prescriptive than BEM — it gives you a mental model, not strict syntax rules. You can combine it with BEM naming inside modules.

**Trade-off:** Categories can blur. When does a "module" become a "layout"? If your team doesn't agree on the boundaries, you'll end up with inconsistent categorization.

## OOCSS: Object-Oriented CSS

**OOCSS** (Object-Oriented CSS) by Nicole Sullivan has two core principles:

1. **Separate structure from skin** — keep layout properties (width, margin, padding) separate from visual properties (color, background, border)
2. **Separate container from content** — don't couple a component's styles to a specific location

```css
/* ❌ Coupled to location */
.sidebar .button { width: 100%; }
.main-content .button { width: auto; }

/* ✅ Separated — button has its own styles, width is applied via a utility */
.button { 
  padding: 8px 16px;
  border-radius: 4px;
  background: blue;
  color: white;
}
.button--full { width: 100%; }
```

The "separate container from content" rule is especially powerful. A `.media` object should look the same whether it's in the sidebar, the main content, or a modal. Don't write `.sidebar .media { font-size: 14px; }` — create a modifier or a context class instead.

**Trade-off:** OOCSS can lead to lots of tiny classes in your HTML. That's by design, but it means your markup becomes more descriptive and your CSS becomes more reusable. It's a trade of brevity for maintainability.

## CSS Modules: Scoping at Build Time

CSS Modules solve the global scope problem at the build level. Every class name is locally scoped to its component by default.

```css
/* Component.module.css */
.title { font-size: 1.5rem; }
.description { color: #666; }
```

```jsx
import styles from './Component.module.css';

function Component() {
  return (
    <div>
      <h2 className={styles.title}>Hello</h2>
      <p className={styles.description}>World</p>
    </div>
  );
}
```

Webpack or Vite transforms `.title` into something like `.Component_title_1a2b3` — guaranteed unique. No collisions, no nesting, no specificity battles.

```css
/* What gets compiled — automatically unique */
.Component_title_1a2b3 { font-size: 1.5rem; }
.Component_description_4d5e6 { color: #666; }
```

**Trade-off:** You lose the ability to globally override styles from outside the component. That's actually the *point*, but it means you need to consciously create CSS custom properties (variables) for theming instead of relying on cascading.

## CSS-in-JS: Colocation Done Right

Libraries like styled-components and Emotion let you write CSS directly in your JavaScript/TypeScript files.

```jsx
const Button = styled.button`
  padding: 8px 16px;
  background: ${props => props.primary ? 'blue' : 'gray'};
  color: white;
  border-radius: 4px;
  
  &:hover {
    opacity: 0.9;
  }
`;

function App() {
  return <Button primary>Click me</Button>;
}
```

The huge win: **dead code elimination** and **dynamic styling** based on props without any selector overhead. When a component isn't used, its styles don't exist. When a prop changes, only the relevant style updates.

**Trade-off:** Runtime cost. Every styled component generates a unique class at runtime. For large apps with thousands of components, that adds up. Zero-runtime alternatives like Linaria or vanilla-extract exist but have their own trade-offs.

## Atomic CSS / Utility-First (Tailwind)

Tailwind CSS popularized the "utility-first" approach: hundreds of small, single-purpose classes that you compose directly in HTML.

```html
<!-- Tailwind — composing utilities -->
<button class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition-colors">
  Click me
</button>
```

No naming, no context switching between HTML and CSS files, no worrying about specificity. You build everything by combining atomic classes.

```css
/* The equivalent CSS you'd need to write */
.button {
  padding-left: 1rem;
  padding-right: 1rem;
  padding-top: 0.5rem;
  padding-bottom: 0.5rem;
  background-color: #3b82f6;
  color: white;
  border-radius: 0.25rem;
  transition: background-color 0.2s;
}
.button:hover {
  background-color: #2563eb;
}
```

**Trade-off:** Your HTML looks like a laundry list of classes. It works incredibly well once you learn the class names, but new team members have a steep learning curve. Also, extracting repeated patterns into components is *mandatory* — otherwise you're repeating 20+ class strings everywhere.

## Which One Should You Pick?

There's no universal winner. Here's how I think about it:

| Scenario | Go With |
|----------|---------|
| Traditional multi-page app, server-rendered | BEM + SMACSS categories |
| React / Vue / SPA with build tools | CSS Modules or styled-components |
| Rapid prototyping, utility-focused team | Tailwind CSS |
| Design system / component library | CSS-in-JS (theming capability) |
| Legacy project cleanup | BEM (lowest friction to adopt incrementally) |

## Practical Tips That Apply Everywhere

**1. Use CSS custom properties for theming**
No matter which methodology you pick, CSS variables (`--color-primary`, `--spacing-md`) give you a single source of truth for design tokens.

**2. Keep specificity flat**
Avoid nested selectors, ID selectors, and `!important`. Flat specificity = predictable overrides = happy developers.

```css
/* ❌ Specificity nightmare */
#main .sidebar ul li a { }

/* ✅ Simple, predictable */
.nav-link { }
```

**3. Enforce with linting**
Use Stylelint with `stylelint-config-standard` or a BEM-specific configuration. Automated enforcement beats code review arguments.

**4. One pattern per project**
Mixing BEM with Tailwind with CSS Modules with styled-components in the same project is chaos. Pick one primary pattern and stick with it. You can use utilities for edge cases, but have a clear primary approach.

## What This Means for You

Next time you start a project, don't just start writing CSS. Spend 30 minutes deciding your architecture up front:

- Pick a naming convention (even if it's "no convention" — that's a decision too)
- Decide how you'll handle global styles vs component styles
- Agree on how variants and states are expressed

CSS doesn't scale by accident. But with the right architecture, it can scale just fine.
