# API Security: Rate Limiting, CORS, Headers

You've got authentication sorted. Sessions are locked down. Then someone hits your `/api/v1/login` endpoint 50,000 times in an hour with passwords from a breach list, and one of them — one — matches a user who reused their password. That's not a hypothetical, that's a Tuesday.

API security is a weird discipline because the boring stuff does most of the work. Not clever cryptography — boring configuration. Rate limits, CORS policies, and HTTP headers won't make a splash in your architecture diagram, but they're the difference between an API that gets scraped, brute-forced, and abused cross-origin, and one that doesn't. Let's go through each one from an attacker's point of view.

## Rate Limiting: The Security Angle

We covered the algorithms (fixed window, token bucket, sliding log) in the rate limiting notes — I won't re-litigate those here. What's worth a second look is *why* rate limiting is a security control, not just a traffic management tool.

Without limits, these attacks are trivially automated:

- **Credential stuffing** — attacker tries millions of leaked username/password pairs against `/login`
- **Brute force** — same endpoint, dictionary passwords, one account
- **Account enumeration** — hammering `/signup` or `/forgot-password` to learn which emails are registered (look at the differing responses)
- **Scraping** — someone pulls your entire catalog or user directory via public endpoints
- **OTP/SMS bombing** — abusing "send code" endpoints to burn through your SMS budget or annoy a victim

```js
// ❌ Login endpoint with zero limits - free brute force for anyone
app.post('/api/login', async (req, res) => {
  const user = await authenticate(req.body.email, req.body.password);
  res.json({ ok: !!user });
});

// ✅ Rate limited per-account AND per-IP
app.post('/api/login',
  rateLimit({
    windowMs: 15 * 60 * 1000,        // 15 minutes
    max: 5,                           // 5 attempts per email
    keyGenerator: (req) => req.body.email, // limit per account
    skipSuccessfulRequests: true      // don't count successful logins
  }),
  rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 100,                         // 100 attempts per IP, regardless of account
    keyGenerator: (req) => req.ip
  }),
  async (req, res) => { /* ... */ }
);
```

The per-account limit stops credential stuffing — one account, five tries, done. The per-IP limit stops distributed attempts across many accounts from the same source. You need both, because stuffing attacks rotate accounts *and* often come from botnets with many IPs.

**Trade-off alert:** per-account rate limiting needs the account identifier *before* authentication, and that identifier is attacker-controlled. An attacker can deliberately lock out a victim by failing 5 logins on *their* email. Real systems mitigate with:

- **Progressive delays** instead of hard blocks (fails silently, no lockout signal)
- **CAPTCHA after N failures** — humans pass, bots stall
- **Don't reveal account existence** — identical response for "wrong password" and "no such user"
- **Email verification on suspicious activity** instead of permanent lockout

And remember the trickiest part: **the response itself is information**. If your API returns `429 Too Many Requests` for a locked account but `401 Unauthorized` for a wrong password on a live one, you've just built an enumeration oracle. Keep status codes consistent.

## CORS: The Misconfiguration Hall of Fame

CORS exists because browsers enforce the same-origin policy — a script on `evil.com` can't just read responses from `yourapi.com`. CORS headers are your explicit opt-out. And developers opt out *hard*.

```js
// ❌ THE classic. "Allow everything" so the frontend works during dev.
//    Shipped to prod. Now ANY website can read your users' data.
app.use(cors());

// ❌ Slightly better but still bad - reflects any origin back
app.use(cors({
  origin: (origin, cb) => cb(null, origin) // echoes attacker's origin!
}));

// ✅ Explicit allowlist of origins you control
const ALLOWED_ORIGINS = new Set([
  'https://app.example.com',
  'https://admin.example.com'
]);
app.use(cors({
  origin: (origin, cb) => {
    if (!origin || ALLOWED_ORIGINS.has(origin)) return cb(null, true);
    cb(new Error('Not allowed by CORS'));
  },
  credentials: true // only needed if you send cookies/auth headers
}));
```

Why the reflected-origin one is so dangerous: an attacker on `evil.com` makes a request with `Origin: https://evil.com`, your server echoes it back, the browser happily exposes the response. Combined with `credentials: true`, that's a full cross-origin read of authenticated data. It's one of the most common critical findings in API security reviews, and it almost always starts as "we needed to fix a CORS error in dev."

Rules that actually hold up:

- **Default deny.** No `*` for origins when credentials are involved. `Access-Control-Allow-Origin: *` + `credentials: true` is invalid in browsers, but misconfigured servers get it wrong in ways that leak.
- **Never reflect arbitrary origins.** Allowlist or nothing.
- **`credentials: true` only where you actually send cookies.** APIs that use bearer tokens in headers don't need it at all — and removing it shrinks the attack surface.
- **Validate `Origin`, not `Referer`.** Origin is set by the browser and can't be forged by page JavaScript; Referer can be stripped or spoofed.
- **Keep the allowlist narrow and exact.** `https://app.example.com` doesn't cover `https://app.example.com.evil.com`.

And don't forget: **CORS is a browser enforcement, not an API enforcement.** Curl, Postman, mobile apps, and server-to-server calls ignore it entirely. CORS protects *browsers* from cross-origin reads; it does nothing against direct API abuse. That's why you pair it with the other stuff on this page.

## Headers: The Free Wins

A handful of response headers cost nothing to set and close real attack classes. For APIs, the big ones:

| Header | What it does | API value |
|---|---|---|
| `Strict-Transport-Security` | Forces HTTPS for a period | Kills downgrade & MITM on API calls |
| `X-Content-Type-Options: nosniff` | Stops MIME sniffing | Blocks content-type confusion attacks |
| `Content-Security-Policy` | Restricts script sources | Mostly for pages; relax for pure APIs |
| `Cache-Control: no-store` | Stops caching of sensitive responses | Auth tokens don't end up in shared caches |
| `X-Frame-Options` / `frame-ancestors` | Blocks framing | Clickjacking on API-driven UIs |
| `Referrer-Policy` | Controls what leaks in `Referer` | Tokens in URLs don't leak cross-origin |

```js
// Express example - the practical set for an API
app.use((req, res, next) => {
  res.set({
    'Strict-Transport-Security': 'max-age=63072000; includeSubDomains',
    'X-Content-Type-Options': 'nosniff',
    'Cache-Control': 'no-store',        // responses may contain auth data
    'Referrer-Policy': 'no-referrer'    // never leak tokens via Referer
  });
  next();
});
```

**The one that surprises people: `Cache-Control: no-store`.** If your API returns tokens, user profiles, or anything sensitive, and a shared cache (browser, corporate proxy, CDN) decides to store it, the next person on that machine can read it. `no-store` is the explicit "do not persist this anywhere" instruction.

Two headers deserve extra care because they're commonly *mis*understood:

- **`Access-Control-Allow-Origin` is not a security header in the way people think.** It's a *feature* header — it defines what the browser lets cross-origin pages read. Setting it wrong is dangerous (see above), but setting it "correctly" doesn't protect your API from non-browser clients.
- **`Content-Security-Policy` on a pure JSON API is mostly noise.** CSP protects against XSS by constraining what the *browser* executes as scripts/styles. A JSON endpoint has nothing to execute. You still want CSP on any HTML pages that consume your API, but don't let a scanner checklist convince you it's a critical API control.

You can check your headers with a single command:

```bash
# What does your API actually announce to the world?
curl -sI https://api.example.com/v1/health

HTTP/2 200
strict-transport-security: max-age=63072000
x-content-type-options: nosniff
cache-control: no-store
referrer-policy: no-referrer
```

If that output is empty, so is your defense.

## Putting It Together: What An Attacker Sees

A quick mental model: your API is a house. Auth is the front door lock. Rate limiting is the bouncer who stops someone trying 10,000 keys. CORS is the rule that only lets *your* windows look in — but it does nothing if someone just walks up and knocks. Headers are the signage and the closed blinds on sensitive rooms.

An attacker probing your API will first check, in this order:

1. Is rate limiting present? (`429` responses, `RateLimit-*` headers, or unlimited retries)
2. What does CORS allow? (send `Origin: https://evil.com`, read the response)
3. What headers come back? (HSTS missing, no `no-store`, `Server` banner leaking versions)
4. Do failures leak information? (different status codes for existing vs non-existing resources)

Each check is cheap for them and each fix is cheap for you.

## The Checklist

- [ ] Rate limit authentication endpoints per-account AND per-IP
- [ ] Consistent responses — don't let status codes reveal account existence
- [ ] Use progressive delays/CAPTCHA instead of instant lockouts (or accept the DoS trade-off)
- [ ] CORS: explicit origin allowlist, no `*`, no reflection, `credentials` only when needed
- [ ] Remember CORS only constrains browsers — non-browser clients ignore it
- [ ] `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store`, `Referrer-Policy` on every response
- [ ] `no-store` on anything carrying auth data or personal info
- [ ] Verify with `curl -sI` and an automated header check in CI
- [ ] Audit CORS config before every release — the "works in dev" config ships too often

## Takeaways

- **Rate limiting is a security control: it's what makes credential stuffing, brute force, enumeration, and scraping uneconomical — apply it per-account and per-IP, and keep failure responses indistinguishable.**
- **CORS is a browser policy, not an API firewall — allowlist exact origins, never reflect arbitrary ones, and know it does nothing against non-browser clients.**
- **Set `Strict-Transport-Security`, `nosniff`, `Cache-Control: no-store`, and `Referrer-Policy` on every response; they're free and they close real attack classes.**
- **Test from an attacker's perspective: probe your own API with `curl`, check the headers, try a reflected origin, and see what your error responses reveal.**
