# CSRF Protection Patterns

You're logged into your bank's website. Somewhere else, you click a link that loads an image — except that image URL is actually a request to `https://your-bank.com/transfer?amount=1000&to=attacker`. Your browser automatically includes your session cookie. The bank's server sees a legitimate-looking request from an authenticated user. Money gone.

That's CSRF — Cross-Site Request Forgery. And it's been one of the most overlooked vulnerabilities in web apps for years.

## How CSRF Actually Works

CSRF exploits a fundamental feature of how browsers handle authentication. When you log into a site, the server sets a cookie. Your browser sends that cookie along with *every* request to that domain — even if the request came from a completely different website.

```html
<!-- Attacker's page, not your bank's page -->
<img src="https://your-bank.com/transfer?amount=1000&to=attacker" 
     style="display:none" />
```

The browser doesn't care *who* made the request. It just attaches the cookie and sends it. The server checks the cookie, sees an authenticated user, and processes the request. It has no way of knowing this request came from a malicious site.

**The key insight:** CSRF isn't about stealing your data. It's about *tricking the server into performing actions on your behalf* — transfers, password changes, email updates, purchases.

## ❌/✅ The Three Protection Patterns

There are three main ways to defend against CSRF. Each has trade-offs.

### 1. Synchronizer Token Pattern (The Classic)

This is the traditional approach. Every form that performs a state-changing action includes a hidden field with a CSRF token — a random value that's tied to the user's session.

```html
<!-- ❌ No CSRF token — vulnerable -->
<form action="/transfer" method="POST">
  <input name="amount" value="1000" />
  <input name="to" value="attacker" />
  <button>Submit</button>
</form>

<!-- ✅ With CSRF token — protected -->
<form action="/transfer" method="POST">
  <input type="hidden" name="_csrf" value="a3f8c2d1e5b..." />
  <input name="amount" value="1000" />
  <input name="to" value="recipient" />
  <button>Submit</button>
</form>
```

The server embeds the token in the form when rendering the page. When the form is submitted, the server validates that the submitted token matches the one stored in the session.

```javascript
// Server-side token generation (Express with csurf-style pattern)
app.get('/transfer', (req, res) => {
  const token = crypto.randomBytes(32).toString('hex');
  req.session.csrfToken = token;
  res.render('transfer', { csrfToken: token });
});

app.post('/transfer', (req, res) => {
  // ❌ BAD — just trusting the session
  // processTransfer(req.user, req.body.amount, req.body.to);

  // ✅ GOOD — validate the token
  if (req.body._csrf !== req.session.csrfToken) {
    return res.status(403).send('Invalid CSRF token');
  }
  processTransfer(req.user, req.body.amount, req.body.to);
});
```

**This works because** the attacker's page can't read the token from your site — it's behind the same-origin policy. The attacker's `<img>` or `<form>` can't include the hidden field because they don't know its value.

**Trade-off:** You need server-side session state and you have to inject the token into every form. For SPAs making API calls (not form submissions), you'll need to send the token as a custom header instead.

### 2. SameSite Cookies (The Modern Savior)

This is the easiest protection — and it's built into browsers now. The `SameSite` attribute on cookies tells the browser when to send the cookie.

```http
// ❌ No SameSite — cookie sent on all requests (CSRF-friendly)
Set-Cookie: session=abc123; Path=/; HttpOnly

// ✅ SameSite=Lax — cookie sent only for top-level navigations
Set-Cookie: session=abc123; Path=/; HttpOnly; SameSite=Lax

// ✅ SameSite=Strict — cookie never sent for cross-site requests
Set-Cookie: session=abc123; Path=/; HttpOnly; SameSite=Strict
```

| SameSite Value | Cookie Sent For | Cookie NOT Sent For |
|----------------|-----------------|---------------------|
| `None` | All requests | Nothing — this is the old default |
| `Lax` | Top-level navigations (clicking a link), GET requests | POST forms, img/script/fetch from other sites |
| `Strict` | Only same-site requests | Everything cross-origin |

`SameSite=Lax` is the sweet spot for most apps. It blocks CSRF on state-changing POST requests while still allowing users to click links to your site without losing their session.

**Reality check:** `SameSite=Lax` is now the default in Chrome, Firefox, and Safari. This means if you're not explicitly setting `SameSite`, modern browsers default to `Lax` automatically. But relying on browser defaults is still risky — older browsers don't enforce it, and you might have explicitly set `SameSite=None` for cross-site use cases (like embeds or third-party widgets).

### 3. Custom Header Pattern (For APIs)

If you're building a single-page app that talks to a REST API, the sync token pattern gets awkward — you don't have forms to embed tokens in. The common approach is to use a custom header.

The idea: the browser sends a `X-Requested-With` or `X-CSRF-Token` header with AJAX requests. Cross-origin requests can't set custom headers without CORS preflight — and the preflight check gives the server a chance to reject it.

```javascript
// ✅ SPA sending CSRF token as custom header
fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': getCookie('csrf-token')  // Read from cookie
  },
  body: JSON.stringify({ amount: 1000, to: 'recipient' })
});
```

```javascript
// ✅ Server validates the custom header
app.post('/api/transfer', (req, res) => {
  const token = req.headers['x-csrf-token'];
  const expected = req.cookies['csrf-token'];
  
  if (!token || token !== expected) {
    return res.status(403).json({ error: 'CSRF validation failed' });
  }
  
  processTransfer(req.user, req.body);
});
```

This is the **double-submit cookie pattern** — the token is in both a cookie and a header. The attacker can't read the cookie from another domain, and they can't set a custom header on a cross-origin request without triggering CORS checks.

## What About CORS?

CORS and CSRF are related but not the same thing. CORS controls *reading* the response. CSRF controls *sending* the request.

If your API uses CORS with credentials (`Access-Control-Allow-Credentials: true`) and a permissive origin (`Access-Control-Allow-Origin: *`), you've opened the door for CSRF. The attacker's site can make credentialed requests AND read the response.

**The fix:** Be specific with CORS origins, and always pair CORS with CSRF protection.

```
// ❌ This + credentials = CSRF disaster
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

// ✅ Specific origin + CSRF protection
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Credentials: true
```

## Trade-offs at a Glance

| Pattern | Complexity | Session Needed | Works with SPAs | Browser Support |
|---------|-----------|----------------|-----------------|-----------------|
| Sync Token | Medium | Yes | Awkward | Universal |
| SameSite=Lax | None (config) | No | Yes | Modern browsers only |
| Custom Header | Low | No | Natural | Universal |
| Double-Submit Cookie | Medium | No | Yes | Universal |

## The Layered Approach

Don't pick one. Use multiple layers:

**Layer 1:** Set `SameSite=Lax` on all your cookies. This handles the vast majority of CSRF attacks for free.

**Layer 2:** For state-changing requests (POST, PUT, DELETE), validate a custom header or CSRF token.

**Layer 3:** For highly sensitive actions (password change, money transfer), add a re-authentication step — ask the user for their password again or send a confirmation code via email.

This layered approach means a single browser quirk or misconfiguration won't leave you exposed.

## What to Do This Week

1. **Audit your cookies.** Are they using `SameSite`? If not, set `SameSite=Lax` as the default.
2. **Check your state-changing endpoints.** Do POST/PUT/DELETE routes have any CSRF validation? Even basic header checks make a difference.
3. **Review CORS settings.** If you're using credentials with overly broad origins, lock it down.
4. **Add a CSRF token for sensitive actions.** Money transfers, password resets, email changes — these deserve more than just cookie-based auth.
5. **Test with a CSRF PoC.** Create a simple HTML page that tries to POST to your app. See if it works. If it does, you know what to fix.
