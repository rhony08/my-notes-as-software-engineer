# Web Accessibility Basics

Try using your own app for five minutes with only a keyboard. No mouse, no trackpad. If you can't get past the first page — or worse, you *can* but it takes you forty seconds — that's not a "nice-to-have" problem. That's a product bug, and it's actively shutting people out.

Accessibility (a11y for short) is one of those things that's easy to skip because it's invisible. When everything works, you never see it. But roughly **15% of the world's population** lives with some form of disability — visual, motor, auditory, cognitive. That's not a niche audience; that's a huge chunk of your users. And accessibility isn't just for them. Captions help people in loud cafés. Keyboard shortcuts help power users. High-contrast text helps everyone on a sunny day.

The good news: you don't need to be a specialist. Most accessibility problems are the same handful of mistakes, repeated. Fix those, and you cover the vast majority of cases. Let's go through the ones that actually matter.

## Semantic HTML Is 80% of the Work

Here's the dirty secret of accessibility: if you use HTML elements for what they're *for*, you get most of the way there for free. Screen readers, keyboard navigation, and assistive tech all hook into the browser's accessibility tree — which is built from your HTML. If your HTML is meaningless, the accessibility tree is empty, and no amount of ARIA can fully patch that.

```html
<!-- ❌ Div soup — a screen reader hears nothing but "group, group, group" -->
<div class="header">
  <div class="title" onclick="goHome()">My Store</div>
  <div class="nav">
    <div onclick="showCart()">Cart</div>
  </div>
</div>

<!-- ✅ Semantic structure — the browser does the work for you -->
<header>
  <h1><a href="/">My Store</a></h1>
  <nav>
    <ul>
      <li><a href="/cart">Cart</a></li>
    </ul>
  </nav>
</header>
```

Notice what changed: a `<nav>` announces itself as navigation. A heading gives structure that screen reader users can jump between. A link is focusable and clickable with Enter. You didn't write *any* accessibility code — you just used the right tags.

A few rules of thumb:

- **One `<h1>` per page**, and heading levels should nest (h1 → h2 → h3), not skip around. Screen reader users navigate by headings — skipping levels is like a book where chapters are numbered 1, 3, 7.
- **Use `<button>` for actions, `<a>` for navigation.** If it does something on the page, it's a button. If it takes you somewhere, it's a link. They have different keyboard behavior and screen reader announcements for a reason.
- **Wrap related form fields in `<fieldset>`** (with a `<legend>`), so screen readers announce the group context.

## Keyboard Navigation: The Second-Place User

Not everyone uses a mouse — and I don't just mean people with motor impairments. Think about power users, or someone with a repetitive strain injury who can't use a trackpad all day. The keyboard is their primary input, and the Tab key is their mouse.

The basics that must work:

- **Every interactive element is reachable and operable with the keyboard.** Tab to it, Enter/Space to activate it.
- **Focus is visible.** When you tab to an element, you need to *see* where you are. Browsers give you a default outline — don't remove it. (More on that below.)
- **No keyboard traps.** Tab should never get stuck in one element or one widget. The classic trap: a modal dialog where focus can enter but never leave.

The most common bug I see is people removing focus outlines:

```css
/* ❌ "Cleaner" look — now nobody can see where they are */
:focus {
  outline: none;
}

/* ✅ Visible focus, still looks decent */
:focus-visible {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
}
```

`outline: none` is the accessibility equivalent of hiding the cursor. If you must restyle it (and you're not replacing it with something equally visible), you've made the site unusable for keyboard users. `:focus-visible` is the modern answer — it shows the ring only for keyboard users, not when someone clicks with a mouse.

**Skip links** are another tiny fix with a huge payoff. A screen reader or keyboard user tabbing through a site has to press Tab through every nav link before reaching content — every single page. A skip link jumps them straight to the main content:

```html
<!-- ✅ First focusable element on the page -->
<a class="skip-link" href="#main">Skip to content</a>

<main id="main">
  <!-- actual content -->
</main>
```

```css
.skip-link {
  position: absolute;
  left: -9999px; /* hidden until focused */
}

.skip-link:focus {
  left: 0; /* appears when tabbed to */
}
```

## Forms: Where Accessibility Goes to Die

Forms are the most interacted-with part of most apps, and the most commonly broken. The fix is embarrassingly simple: **every input needs a label, and the label needs to be connected to the input.**

```html
<!-- ❌ Placeholder is not a label -->
<input type="email" placeholder="Email address">

<!-- ✅ Explicit label, connected via `for` -->
<label for="email">Email address</label>
<input type="email" id="email" name="email">

<!-- ✅ Or wrap it — same effect, less boilerplate -->
<label>
  Email address
  <input type="email" name="email">
</label>
```

Why does the connection matter? A screen reader announces the label when the input gets focus. Without it, a blind user tabs into a field and hears "edit text" — with zero clue what goes there. Placeholders don't help because screen readers often skip them (and they disappear the moment you type).

Errors are the other half. When a form fails validation, the error needs to be:

1. **Associated with the field** (so the screen reader announces it)
2. **Programmatically announced** (so the user knows *something* failed, even if the error is elsewhere on the page)

```html
<!-- ✅ aria-describedby links the input to its error message -->
<label for="email">Email address</label>
<input
  type="email"
  id="email"
  name="email"
  aria-describedby="email-error"
  aria-invalid="true"
>
<p id="email-error" role="alert">Please enter a valid email address.</p>
```

The `role="alert"` is what makes the error *announced* — without it, a screen reader user might be looking at the submit button and never know the form didn't go through. Also: don't rely on color alone to mark errors. "The field turned red" means nothing to someone who can't see red. Pair it with text, an icon, or both.

## Images and Non-Text Content

Every image needs an alternative. But "alt text" isn't a one-size-fits-all thing — it depends on what the image *is*:

| Image type | Alt text strategy | Example |
|------------|-------------------|---------|
| Informative (chart, diagram, product photo) | Describe the information it conveys | `alt="Bar chart: sales up 20% in Q2"` |
| Decorative (spacer, background flourish) | Empty alt — hide it from AT | `alt=""` |
| Functional (icon that links/acts) | Describe the *action*, not the picture | `alt="Search"` (not `alt="magnifying glass"`) |
| Text in an image (logo, banner) | The text itself | `alt="Acme Corp"` |

```html
<!-- ❌ Screen reader: "Image. Image. Image." -->
<img src="team-photo.jpg">
<img src="spacer.gif">
<img src="settings-icon.png">

<!-- ✅ Useful, hidden, actionable -->
<img src="team-photo.jpg" alt="The engineering team at the 2025 offsite">
<img src="spacer.gif" alt="">
<img src="settings-icon.png" alt="Open settings">
```

And a pet peeve: **never put text inside images**. "Click here to download our guide" as a JPEG is unreadable to screen readers, unselectable, and un-translatable. If you need text, use text.

## Color: Contrast and Colorblindness

Around 8% of men have some form of color vision deficiency. That's not rare — that's a lot of users. The rules:

1. **Don't communicate with color alone.** "Red = error, green = success" fails for colorblind users. Pair color with text/icons.
2. **Meet contrast ratios.** Text needs enough contrast against its background. WCAG requires:

| Text type | Minimum contrast ratio |
|-----------|------------------------|
| Normal text | 4.5:1 |
| Large text (18pt+ / 14pt bold+) | 3:1 |
| UI components & icons | 3:1 |

```css
/* ❌ #999 on white — about 2.8:1, fails even large text */
color: #999999;

/* ✅ #595959 on white — about 7:1, passes AAA */
color: #595959;
```

There are free tools (WebAIM Contrast Checker, browser DevTools) that compute this for you. If you use a design system, check contrast once at the token level instead of per-component — one fix, applied everywhere.

## ARIA: Powerful, and Powerfully Misused

ARIA (Accessible Rich Internet Applications) lets you tell assistive tech about things HTML can't express — like "this div is a dialog" or "this region just updated." But there's a golden rule:

> **Don't use ARIA when a native HTML element does the job.** ARIA doesn't magically add behavior — it only changes what screen readers *announce*. A `<div role="button">` announces itself as a button but still doesn't respond to Enter, doesn't get focus, and doesn't handle clicks. You've built a worse button.

```html
<!-- ❌ ARIA can't fix a div -->
<div role="button" onclick="submit()">Submit</div>

<!-- ✅ The real thing, all behavior included -->
<button onclick="submit()">Submit</button>
```

Where ARIA genuinely shines:

- **`aria-label`** — when there's no visible text to announce (icon buttons):

```html
<button aria-label="Close dialog">✕</button>
```

- **`aria-live`** — announcing dynamic updates, like a chat message arriving or a cart total changing:

```html
<!-- ✅ Screen reader announces new content here automatically -->
<div aria-live="polite">
  <!-- injected messages -->
</div>
```

`polite` waits for the user to finish what they're doing; `assertive` interrupts immediately (use sparingly — it's the screen reader equivalent of someone yelling).

- **`role="dialog"` + `aria-modal="true"`** — for custom modals, *combined with* focus management (see below).

The #1 ARIA mistake is over-application. A common smell: `role="navigation"` on a `<nav>` (redundant — `<nav>` already implies it), or `aria-label` on everything in sight. Audits with tools like axe will flag most of this for you.

## Modals: The Hardest Common Pattern

Modals are where accessibility goes to die, because there's no native HTML modal. Every framework's modal component needs manual work:

1. **Focus moves into the modal** when it opens.
2. **Focus is trapped inside** while it's open (Tab from the last element cycles back to the first).
3. **Focus returns to the trigger** when it closes.
4. **Esc closes it**, and the close button is reachable.

```js
// Opening: save where focus was, move it into the dialog
const trigger = document.activeElement;
dialog.show();
closeButton.focus();

// Closing: put focus right back where it was
function closeModal() {
  dialog.hidden = true;
  trigger.focus(); // ✅ never leave the user stranded
}

document.addEventListener('keydown', (e) => {
  if (e.key === 'Escape') closeModal();
});
```

A related bug: **background content stays reachable.** While a modal is open, users can still Tab into the page behind it — that's a focus leak, and it's confusing as hell ("where did I just go?"). The simplest fix: use the native `<dialog>` element with `showModal()`, which handles focus trapping and the top-layer for you. In 2026, there's no good excuse for hand-rolling a modal.

## Testing: You Don't Need a Lab

You don't need a team of specialists to find most a11y issues. A 15-minute routine catches the majority:

1. **Tab through the whole page.** Can you reach everything? Is focus visible? Does anything trap you?
2. **Run an automated scanner.** axe DevTools (free browser extension) or Lighthouse's accessibility audit. These catch contrast, missing labels, missing alt text, and ARIA misuse automatically.
3. **Try a screen reader once.** On macOS, VoiceOver is built-in (Cmd+F5); on Windows, NVDA is free. You don't need to master it — just experience what a sighted-mouse user takes for granted and what your app sounds like without it. That empathy alone will change how you build.
4. **Test zoom.** Browsers handle most of this now, but a 200% zoom should never break your layout or overlap elements.

```bash
# Lighthouse accessibility audit, no browser needed
npx lighthouse https://your-site.com --only-categories=accessibility --output=json
```

Automated tools catch maybe 30–40% of issues. The rest — focus order, screen reader flow, keyboard traps — only show up when a human actually uses the thing. That's why the keyboard walkthrough is non-negotiable.

## The Honest Trade-offs

- **Fancy custom components are the enemy.** A custom dropdown with animations and virtual scrolling is 10x the a11y work of a native `<select>`. If you must build one, budget for the accessibility work up front — it's not free.
- **Focus rings can clash with design.** Yes, the outline looks "unpolished" next to your rounded-corner aesthetic. That's a design problem to solve, not a license to delete the ring. Style it, don't remove it.
- **Perfection is a moving target.** WCAG has A, AA, and AAA levels. AA is the practical standard most companies target (and the one legal requirements reference). Chasing AAA everywhere usually means degrading the experience for everyone else — dim animations, simplified language. Start at AA, revisit later.
- **It's an ongoing thing, not a one-time fix.** New features ship, new components land, and each one can reintroduce the same bugs. The fix is process, not heroics: make a11y part of code review, keep axe in your CI, and treat "I can't reach this with a keyboard" as a bug with the same severity as a crash.

## Takeaways

- Build with semantic HTML first — `<button>`, `<nav>`, proper headings. It's 80% of accessibility, for free.
- Never remove focus outlines, and add a skip link so keyboard users can jump to content.
- Every input needs a connected `<label>`, and errors need `aria-describedby` + `role="alert"`.
- Write alt text that matches the image's *purpose* — and use empty `alt=""` for decoration.
- Respect contrast ratios and never rely on color alone to convey meaning.
- Use ARIA sparingly — if a native element exists, use it. And if you build modals, manage focus properly (or use `<dialog>`).
- Add a keyboard tab-through + an automated scanner to your routine. Accessibility is a habit, not a project.
