# Session Management Security

Here's a scenario that still gives me nightmares: user logs in, closes the laptop, and their session stays valid for a month. Someone steals the cookie (or the session ID from a log file), and now they're browsing as that user — reading their emails, changing their password, buying stuff with their saved card. Nobody gets kicked out. Nobody gets notified. The session just... lives on.

Session management is one of those things that looks trivial until it isn't. You set a cookie, you check it on every request, done. But every decision — how the ID is generated, where it's stored, when it expires, what happens on logout — is a security decision. Get one wrong and you've handed over the keys.

Let's walk through the whole lifecycle: generating session IDs, storing sessions safely, cookie flags that actually matter, expiry, logout, and the trade-offs you'll have to make.

## Session IDs: The Foundation Is Entropy

Your session ID is the only thing separating an attacker from a logged-in user. It needs to be **unguessable**, which means it needs real randomness.

```js
// ✅ DO THIS - 256 bits from a CSPRNG
const crypto = require('crypto');
const sessionId = crypto.randomBytes(32).toString('hex');

// ❌ NEVER DO THIS - predictable, incrementable, guessable
const sessionId = `${user.id}-${Date.now()}`;

// ❌ ALSO BAD - Math.random is NOT cryptographically secure
const sessionId = Math.random().toString(36).slice(2);
```

The `Math.random()` one is the sneaky one — it looks random, it passes your tests, and then someone publishes a paper showing how to predict V8's PRNG state after enough samples. Always reach for `crypto.randomBytes` (or `uuid` v4 / `secrets.token_urlsafe` in Python — anything backed by a CSPRNG).

Rule of thumb: **at least 128 bits of entropy** (16 random bytes). If you can enumerate all possible session IDs in any reasonable time, they're not secure.

## Where Do Sessions Live?

The big architectural fork: server-side sessions (opaque ID → data in a store) vs client-side sessions (everything in the token itself, JWT-style). Each has real trade-offs, and neither is "wrong" — but they behave very differently under attack.

| | Server-side (opaque token) | Client-side (JWT) |
|---|---|---|
| Revocation | ✅ Instant — delete the record | ❌ Can't until expiry, need a blocklist |
| Data stored | Server-side, invisible to client | In the token, base64-readable by anyone |
| Scaling | Needs shared store (Redis) | Stateless, easy to scale |
| Leak impact | Single session dies | Token valid everywhere until expiry |
| Complexity | Session store to run | Revocation infrastructure to build |

Most apps should start server-side. The instant-revocation property alone is worth it — "log out everywhere" is a one-liner instead of a blocklist migration.

## Cookie Flags: The Boring Checklist That Saves You

If you're using cookies (and for browser apps you should be), four flags decide how much damage a leaked or forged cookie can do:

```js
// ✅ DO THIS - the full armor set
res.cookie('session', sessionId, {
  httpOnly: true,      // JS can't read it -> XSS can't steal it
  secure: true,        // only over HTTPS -> no sniffing on public wifi
  sameSite: 'lax',     // stops most CSRF -> browser won't send it cross-site
  path: '/',           // whole app needs it
  maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days absolute expiry
});

// ❌ NEVER DO THIS - everything wrong at once
res.cookie('session', sessionId);
```

Breaking down why each matters:

- **`httpOnly`** — without it, any XSS vulnerability becomes instant session theft. `document.cookie` is right there. This flag is the cheapest defense you'll ever deploy.
- **`secure`** — without it, the cookie rides along on plain HTTP. Coffee-shop wifi + a packet capture = session hijack.
- **`sameSite`** — `lax` blocks cookies on cross-site POST requests (kills most CSRF), `strict` also blocks them on cross-site GETs (more secure, but breaks some legit flows like following an external link into an authed page). For most apps `lax` is the sweet spot.
- **`maxAge` / `expires`** — a cookie without an expiry is a *session cookie* that dies with the browser tab. That's sometimes what you want (banking apps do this), but you need to decide deliberately.

One more: don't set `domain` on your cookie unless you genuinely need subdomain sharing. A cookie scoped to `.example.com` is sent to *every* subdomain — including the marketing site someone else controls.

## Expiry: Absolute vs Idle

Two clocks are running on every session, and people mix them up:

- **Absolute expiry** — the session dies N hours/days after login, no matter what. Hard upper bound on damage if a token leaks.
- **Idle timeout** — the session dies after N minutes of inactivity. Kills abandoned sessions on shared computers.

```js
// Session data (e.g. in Redis)
{
  userId: 42,
  createdAt: 1723000000000,
  lastActiveAt: 1723000000000,
  absoluteExpiry: 1723000000000 + 24 * 3600 * 1000, // 24h hard cap
  idleTimeout: 30 * 60 * 1000 // 30 min of inactivity
}
```

You want **both**. Absolute expiry bounds worst-case damage; idle timeout cleans up forgotten sessions. And the idle timeout needs a sliding update:

```js
// Middleware: touch the session on activity
app.use((req, res, next) => {
  const session = req.session;
  if (session && Date.now() - session.lastActiveAt > session.idleTimeout) {
    return destroySession(req, res); // expired by inactivity
  }
  session.lastActiveAt = Date.now(); // sliding window
  next();
});
```

**Trade-off:** sliding idle timeouts are friendlier (users don't get randomly logged out mid-work) but they extend the *effective* lifetime of a stolen session as long as the thief keeps using it. For high-security apps, prefer *fixed* idle expiry: count from `createdAt`, don't slide. Annoying? Yes. Safer? Yes.

## Session Fixation: The Attack People Forget

Session fixation is nasty because it doesn't steal your session — it *gives you theirs*.

1. Attacker creates a session on your app and gets a valid session ID
2. Attacker tricks the victim into using that ID (URL parameter, crafted link, cookie injection on a subdomain)
3. Victim logs in — and the login "upgrades" the attacker's session to an authenticated one
4. Attacker now shares the victim's session

The fix is dead simple: **never keep the same session ID across a privilege change**:

```js
// On successful login (Express + express-session)
req.session.regenerate(err => {
  if (err) return next(err);
  req.session.userId = user.id;
  // old session ID (possibly attacker-controlled) is now useless
});

// ❌ If you do this instead, you're vulnerable:
req.session.userId = user.id; // same ID, now authenticated!
```

Regenerate on login, on privilege escalation (becoming admin), and on password change. It's one line and it kills an entire attack class.

## Logout: Make It Actually Destroy Things

A logout that doesn't destroy the server-side session is a lie. The cookie's gone from the browser, but the session record still exists — and anyone who captured the ID earlier can still use it.

```js
// ✅ DO THIS - destroy server-side, then clear the cookie
await sessionStore.destroy(sessionId);
res.clearCookie('session', { httpOnly: true, secure: true, sameSite: 'lax' });

// ❌ INCOMPLETE - cookie gone, session record alive
res.clearCookie('session');
```

If your app has a "log out of all devices" feature, that's just `sessionStore.destroyByUserId(user.id)`. If you *can't* do that instantly — hello, JWT — you need a blocklist or a token version bump. Which brings us to:

## Revocation: The JWT Elephant

JWTs are stateless, which is exactly why they're hard to kill. A leaked access token is valid until expiry, full stop. Your options, in order of preference:

1. **Short access-token lifetime (15 min or less)** + long-lived refresh token that *is* revocable (stored server-side). Stolen access token dies quickly; stolen refresh token can be killed.
2. **Token version field** — store `tokenVersion` on the user, include it in the JWT, check it on every request. Bump the version = all that user's tokens die instantly.
3. **Blocklist (`jti`)** — store revoked token IDs in Redis with TTL until natural expiry. Works, but now you're running a session store anyway, so... why did you pick JWTs?

Honest take: if you need revocation, and you're building it from scratch, server-side sessions are usually less work. JWTs shine when you genuinely need statelessness (distributed services that can't share a session store easily) — just know what you're giving up.

## Session Storage: Redis, and the Trade-offs

For server-side sessions you need a shared store the moment you run more than one app instance. Options ranked roughly by commonness:

| Store | Pros | Cons |
|---|---|---|
| In-memory (default) | Zero setup | Dies on restart, per-instance only ❌ |
| Redis | Fast, TTL built-in, shared | Another service to run, needs auth |
| Database table | Durable, transactional | Slower per request, DB load |
| Memcached | Fast | No persistence, TTL fiddly |

Redis is the classic choice and it fits sessions perfectly: TTL = expiry, `DEL` = logout, `SCAN` by user = log out everywhere. Just remember:

- **Set a TTL on every key.** Redis eviction under memory pressure will happily kill your sessions if you didn't.
- **Don't store anything sensitive you don't need.** The session record doesn't need the user's full profile or their credit card. Store `userId` and a few flags, fetch the rest per request.
- **Secure the Redis connection** — an open Redis on the default port is a well-known way to get your sessions dumped. Require auth, don't expose it publicly.

## Spotting Hijacked Sessions

You can't prevent every leak, but you can detect them. Cheap signals that pay off:

- **IP / user-agent changes mid-session** — flag it, require re-auth for sensitive actions
- **Session used from two geographies at once** — classic stolen-cookie pattern
- **Unusual times** — a session that's active at 4am from a country you don't serve
- **Simultaneous sessions on many devices** — for most apps, a user logging in from 14 devices in an hour is either a bot or a reseller, not a person

Don't overreact to IP changes though — mobile users hop between wifi and cellular constantly. Alert, don't nuke. Require re-auth for high-risk actions (password change, new payout method) rather than killing the session outright.

## The Checklist

- [ ] Session IDs: ≥128 bits from a CSPRNG, never `Math.random()` or predictable patterns
- [ ] Cookies: `httpOnly` + `secure` + `sameSite` (lax/strict) + deliberate expiry
- [ ] Regenerate session ID on login, privilege change, password change
- [ ] Absolute expiry AND idle timeout (fixed, not sliding, for high-security apps)
- [ ] Logout destroys the server-side session, not just the cookie
- [ ] Revocation story: server-side sessions, or short-lived JWTs + revocable refresh tokens
- [ ] Session store: shared (Redis), TTL on everything, secured, minimal data
- [ ] Anomaly detection on IP/user-agent/geo changes
- [ ] Re-auth required for sensitive actions on suspicious sessions

## Takeaways

- **Generate session IDs with a CSPRNG and at least 128 bits of entropy — predictable IDs are game over.**
- **Set `httpOnly`, `secure`, and `sameSite` on every session cookie; these three flags stop most theft and CSRF.**
- **Regenerate the session ID on login and privilege changes to kill session fixation.**
- **Run both an absolute expiry and an idle timeout — one bounds damage, the other cleans up.**
- **Make logout and "log out everywhere" actually destroy server-side sessions; if you use JWTs, accept that revocation requires a blocklist or token versions.**
