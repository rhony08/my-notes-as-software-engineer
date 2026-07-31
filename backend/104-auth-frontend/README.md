# Authentication in Frontend Apps

Your login page works. The token arrives, the user is "authenticated," and you ship it. Then the first bug report comes in: *"I refreshed the page and got logged out."* Next: *"My token shows up in a screenshot someone took of my DevTools."* Then the classic: *"Why does the app flash the login screen every time I open it?"*

Frontend auth is a minefield of subtle decisions — where to store the token, what happens when it expires mid-request, how to keep the UI in sync with auth state. Get these wrong and you get security holes, or an app that logs people out at the worst possible moment. Let's walk through the decisions you actually have to make.

## The First Decision: Where Does the Token Live?

This is the debate that starts every frontend auth discussion. Two main options:

**localStorage** — simple, accessible from JavaScript, survives reloads.
**HttpOnly cookie** — invisible to JavaScript, sent automatically with requests.

The trade-off is between convenience and attack surface:

| Storage | XSS exposure | CSRF exposure | JS access | DevTools visibility |
|---------|-------------|---------------|-----------|---------------------|
| localStorage | High — any XSS reads your token | None (not auto-sent) | Yes | Visible |
| HttpOnly cookie | None — JS can't read it | Possible (needs CSRF protection) | No | Hidden |
| In-memory (variable) | Low | None | Yes | Hidden, but lost on reload |

```js
// ❌ Token readable by any injected script
localStorage.setItem('access_token', token);

// ✅ HttpOnly cookie — the browser handles it, JS never sees it
// Set by the server: Set-Cookie: access_token=...; HttpOnly; Secure; SameSite=Strict
```

Here's the honest take: if your app is an SPA with a separate API server, the common middle ground is an **access token in memory + refresh token in an HttpOnly cookie**. In-memory means a page reload loses it — but that's what the refresh token is for. The XSS attacker gets nothing, and you only need CSRF protection on the refresh endpoint.

If you *must* use localStorage (say, you're integrating with a third-party auth SDK that requires it), at least keep the access token short-lived so a leak has a small window.

## The Token Lifecycle: Access + Refresh

A single long-lived token is a liability — if it leaks, the attacker has access forever. So production apps split it:

- **Access token** — short-lived (5–15 min), sent with every API call
- **Refresh token** — long-lived (days/weeks), only used to get new access tokens

```js
// The happy path
const res = await fetch('/api/me', {
  headers: { Authorization: `Bearer ${accessToken}` }
});

// 401 → token expired → silently refresh → retry once
if (res.status === 401) {
  const { accessToken: fresh } = await fetch('/auth/refresh', {
    method: 'POST',
    credentials: 'include'   // refresh token cookie goes along
  }).then(r => r.json());

  // retry the original request with the fresh token
}
```

The ugly part: **multiple requests can 401 at the same time**, and if they each fire a refresh, you've got a refresh race — and possibly a bunch of useless refreshes. The fix is a single shared refresh promise:

```js
let refreshPromise = null;

function refreshToken() {
  if (!refreshPromise) {
    refreshPromise = fetch('/auth/refresh', { method: 'POST', credentials: 'include' })
      .then(r => r.json())
      .finally(() => { refreshPromise = null; });
  }
  return refreshPromise;
}
```

Now ten simultaneous 401s all await the *same* refresh call. One refresh, ten retries.

## Handling Expiry Without Destroying the UX

The worst possible auth UX: user is typing a long form, hits submit, and the app bounces them to the login page because the token expired two minutes ago.

```js
// ❌ Kill the session on the first 401
if (res.status === 401) {
  logout();
  router.push('/login');
}

// ✅ Only redirect on real auth failures (refresh also fails)
if (res.status === 401) {
  try {
    await refreshToken();
    return retryOriginalRequest();
  } catch {
    logout();           // refresh failed → session is truly dead
    router.push('/login');
  }
}
```

The refresh token is the thing that keeps sessions alive across tab closes and coffee breaks. Only when *it* expires (or is revoked) should the user see the login screen again.

And a tip that saves real headaches: **start the refresh before the token actually dies.** If you know your access token lives 15 minutes, check expiry at ~14 minutes and refresh proactively. It's the difference between a smooth app and one with a periodic "hiccup" every 15 minutes.

## Auth State in the UI: The Loading Problem

Every auth'd app has this bug: open the app, and for a split second it renders the logged-out state — navbar without your avatar, a flash of the login page. That's the app rendering before it knows who you are.

The fix is boring but essential: treat "checking auth" as a real state, not a boolean.

```js
// ❌ Assumes logged out until proven otherwise
const isLoggedIn = !!user;

// ✅ Three states: unknown → authenticated / unauthenticated
const authState = user === undefined ? 'loading'
                : user === null ? 'unauthenticated'
                : 'authenticated';

if (authState === 'loading') return <Spinner />;  // don't render anything auth-dependent
```

Why this matters beyond looks: if you render a "login" button for 200ms on every page load, analytics get polluted, and — worse — if an unauthenticated user somehow clicks it, they might hit a redirect loop. The spinner is the honest answer.

## OAuth and "Login with Google"

Most apps eventually add social login. That's OAuth 2.0, and the flow that matters for SPAs is the **Authorization Code + PKCE** flow:

1. App redirects to Google: `https://accounts.google.com/o/oauth2/auth?client_id=...&redirect_uri=...&code_challenge=...`
2. User logs into Google, gets redirected back with a `?code=...`
3. App exchanges the code (plus the PKCE verifier) for tokens — usually via a backend endpoint, not directly from the browser

Two things people get wrong:

- **Never put your client secret in the frontend.** It's not secret once it ships. With PKCE you don't even need a secret for the code exchange — that's the whole point.
- **Validate the `redirect_uri` carefully.** It's a common attack vector — if the auth provider accepts a slightly-off URI, an attacker can redirect the code to themselves.

Also, the "state" parameter isn't optional. It's your CSRF defense for OAuth — without it, an attacker can log you into *their* account and you'll happily send them your data.

```js
// Generate and stash state before redirecting
const state = crypto.randomUUID();
sessionStorage.setItem('oauth_state', state);

// ...after redirect back...
const returnedState = new URLSearchParams(location.search).get('state');
// ❌ Skip the check → CSRF-able login flow
// ✅ Verify it matches what we stored
if (returnedState !== sessionStorage.getItem('oauth_state')) {
  throw new Error('OAuth state mismatch — possible CSRF');
}
```

## Logout: More Than Clearing the Token

Logout is where sloppy apps leak sessions. Clearing the token on the client *looks* done, but the refresh token cookie is still valid — the session lives on server-side.

```js
// ❌ Client-only logout — refresh token survives, session still valid
localStorage.removeItem('access_token');
router.push('/login');

// ✅ Revoke server-side, then clear local state
await fetch('/auth/logout', { method: 'POST', credentials: 'include' });
clearAuthState();
router.push('/login');
```

If you don't revoke, someone with a stolen refresh cookie stays logged in forever — your "logout" button is a lie. (This is also why revoking on password change matters: a leaked session should die when the user changes their password.)

## Trade-offs Worth Admitting

- **In-memory tokens are annoying for tabs.** Each tab has its own memory — log in on one tab, the other doesn't know. Solutions exist (BroadcastChannel, storage events) but they add complexity.
- **HttpOnly cookies need CSRF protection.** You've traded XSS risk for CSRF risk. SameSite=Strict/Lax cookies handle most of it, but same-site subdomains can still bite you.
- **Refresh tokens are a bigger target.** A long-lived refresh token is gold for attackers. Rotate them on every refresh, and consider revoking on suspicious activity (new device, different IP).
- **Session expiry UX is a product decision.** "You've been logged out" at 30 minutes might be correct for a bank and infuriating for a music player. Pick lifetimes based on what the app holds.

## Takeaways

- Store access tokens in memory, refresh tokens in HttpOnly cookies — and keep access tokens short-lived.
- Treat 401 as "refresh and retry," not "kill the session." Only redirect to login when the refresh itself fails.
- Render a loading state while auth is being checked — don't flash logged-out UI.
- Use Authorization Code + PKCE for social login, and never trust a client-side secret.
- Logout must revoke server-side, or it's not a logout.
- Rotate refresh tokens and keep session lifetimes aligned with the sensitivity of your app's data.
