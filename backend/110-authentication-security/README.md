# Authentication Security Best Practices

You know what's terrifying? Realizing your "secure" auth system was built on a couple of assumptions that turned out to be wrong. Passwords stored with MD5. JWTs with a year-long expiry and no way to revoke them. Login endpoints with zero rate limiting. Every one of these is a vulnerability you're handing to attackers on a silver platter.

Authentication is the front door of your entire system. If someone walks through it pretending to be a user, every other security measure you have — encryption, input validation, the works — becomes pointless. They're already inside.

So let's go through what actually matters when you're building auth, from password storage to session handling, and the mistakes I've seen (and made) along the way.

## Password Storage: The Foundation

If you're storing passwords in plain text, or with a fast hash like MD5 or SHA-256, stop what you're doing. Fast hashes are exactly what attackers want — they can try billions of guesses per second with a GPU.

The rule is simple: **use a slow, salted, adaptive hash function**. Your options:

| Algorithm | Salt | Slow by design | Current status |
|-----------|------|----------------|----------------|
| MD5 / SHA-1 / SHA-256 | Optional | ❌ No | ❌ Never use for passwords |
| bcrypt | ✅ Built-in | ✅ Yes (cost factor) | ✅ Still solid |
| Argon2id | ✅ Built-in | ✅ Yes (memory + CPU) | ✅ Recommended (OWASP #1) |
| scrypt | ✅ Built-in | ✅ Yes (memory-hard) | ✅ Good alternative |

Here's what bcrypt looks like in practice:

```js
// ✅ DO THIS - bcrypt handles salting and the cost factor for you
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(password, 12); // cost 12 = ~250ms, tuned for your hardware

// ❌ NEVER DO THIS - no salt, fast hash, rainbow-table fodder
const hash = crypto.createHash('sha256').update(password).digest('hex');

// ❌ ALSO BAD - MD5 with a static salt is still trivial to crack
const hash = md5('my-static-salt' + password);
```

The cost factor matters more than people think. `bcrypt.hash(password, 12)` takes roughly 250ms on modern hardware. That's fine for a login — you're doing one per request — but it makes brute-forcing 10 million passwords take weeks instead of hours.

**Trade-off:** slower hashing means a slightly worse login UX and more CPU under load. That's the point. If your login endpoint can hash 100k passwords/second, so can the attacker's GPU cluster. Aim for ~100-250ms per hash and offload it if you're CPU-bound at scale.

## Never Roll Your Own Auth Logic

Hand-writing a token generator? Writing your own password reset flow from scratch? Please don't. The number of ways this goes wrong is staggering:

- `Math.random()` for tokens instead of a CSPRNG
- Tokens that never expire
- Timing leaks in comparison logic
- Missing lockout on repeated failures

Use battle-tested libraries and frameworks. In the Node world that's something like Passport, Lucia, or your framework's built-in auth. In Django it's `django-allauth`. In Rails it's `has_secure_password` plus Devise if you need full flows. These libraries have been hammered by security researchers for years — your hand-rolled version hasn't.

## JWT: Powerful, but Easy to Get Wrong

JWTs are great for stateless auth, but they come with a specific set of failure modes. The big three:

### 1. Algorithm confusion

```js
// ❌ NEVER DO THIS - accepting whatever algorithm the token claims
const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256', 'HS256'] });

// ✅ DO THIS - pin the algorithm explicitly
const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256'] });
```

If you accept `HS256` (a symmetric algorithm) while expecting `RS256` (asymmetric), an attacker can sign tokens with the public key as the HMAC secret. The `alg: none` attack is a variant of the same family. **Always pin the algorithm.**

### 2. No revocation

JWTs are stateless — that's the appeal. But it means a stolen token is valid until it expires. You can't "log out" a JWT server-side without a blocklist.

Practical mitigations:
- **Short expiry** (15-30 minutes for access tokens)
- **Refresh tokens** with longer life that ARE revocable (stored server-side)
- **Token version** field in the DB — bump it to invalidate all sessions for a user
- **`jti` (JWT ID)** claim for blocklisting specific tokens

### 3. Overstuffed tokens

```js
// ❌ BAD - PII in the token, base64 is NOT encryption
const token = jwt.sign({ name: 'John Doe', email: 'john@example.com', ssn: '123-45-6789' }, secret);

// ✅ GOOD - minimal claims, everything else stays server-side
const token = jwt.sign({ sub: user.id, role: user.role, jti: uuid() }, secret, { expiresIn: '15m' });
```

Anyone with the token can decode the payload. Don't put sensitive data in there.

## Login Endpoints Need Their Own Protection

A login form is the one endpoint where attackers can hammer your system with *legitimate* credentials. That makes it uniquely exposed to:

- **Brute force** — trying many passwords for one account
- **Credential stuffing** — trying username/password combos leaked from other breaches
- **Spraying** — trying one common password across many accounts

What actually works:

| Measure | Stops | Cost |
|---------|-------|------|
| Rate limiting per IP | Brute force, stuffing | Low |
| Rate limiting per account | Spraying | Low |
| Account lockout (with care) | Brute force | Medium |
| MFA | Almost everything | Medium |
| CAPTCHA after N failures | Automated attacks | UX cost |

The lockout caveat: aggressive lockout is a DoS vector. An attacker can lock out your users on purpose. Prefer **temporary, escalating lockouts** (e.g., 5 failures → 15 min, then 1 hour) over permanent bans, and consider a soft lockout that only blocks the password field, not MFA.

## MFA Isn't Optional Anymore

Passwords get breached. It's not a matter of if, it's when. MFA is the difference between "your password leaked" and "your account got taken over."

```js
// ✅ DO THIS - enforce MFA for privileged actions
if (user.mfaEnabled) {
  const valid = await verifyTOTP(user.mfaSecret, code);
  if (!valid) return res.status(401).json({ error: 'Invalid code' });
}

// ❌ NEVER DO THIS - MFA only "if the user feels like it"
if (user.mfaEnabled && req.body.rememberDevice) {
  // skip verification... why did we even build this?
}
```

Enforce MFA for: admins, anyone with billing access, and (strongly consider) all users on high-value platforms. And remember — TOTP recovery codes need secure storage too. Someone with DB access and your recovery codes list has full account takeover.

## Session Management Done Right

If you're using server-side sessions (which you probably should for most apps — they're revocable!), the security is in the details:

- **HttpOnly cookies** — JavaScript can't read them, so XSS can't steal them
- **Secure flag** — only sent over HTTPS
- **SameSite=Strict (or Lax)** — mitigates CSRF
- **Regenerate session ID on login** — prevents session fixation

```js
// ✅ DO THIS
res.cookie('session', sessionId, {
  httpOnly: true,      // XSS can't touch it
  secure: true,        // HTTPS only
  sameSite: 'strict',  // CSRF protection
  maxAge: 7 * 24 * 60 * 60 * 1000
});

// ❌ NEVER DO THIS
res.cookie('session', sessionId); // readable by JS, sent over HTTP, no expiry
```

Session fixation attack: attacker gives you a session ID, you log in, and now they share your session. Regenerating the ID on login kills this dead:

```js
// On successful login
req.session.regenerate(err => {
  req.session.userId = user.id;
  // new session ID, old (attacker-known) one is useless now
});
```

## Constant-Time Comparison for Everything Sensitive

This one's subtle. Comparing strings with `==` or `===` short-circuits on the first mismatched character. Measure the timing across enough attempts and you can leak the value character by character. It's a real attack (see the "Lucky13" and timing attacks on MACs).

```js
// ❌ VULNERABLE - leaks timing information
if (providedToken === storedToken) { /* ... */ }

// ✅ CONSTANT-TIME - always compares every byte
const crypto = require('crypto');
const valid = crypto.timingSafeEqual(
  Buffer.from(providedToken),
  Buffer.from(storedToken)
);
```

Use `timingSafeEqual` for: API tokens, password reset tokens, email verification codes, webhook signatures. Basically anything where a secret is compared.

## Password Reset Flows: The Forgotten Attack Surface

Password reset is one of the most attacked flows in any app — it's account takeover with a user interface. The classics:

1. **Predictable tokens** — `reset_token = user.id + timestamp`. Guessable = game over.
2. **Tokens that never expire** — a token from 2019 still works? That's a breach waiting to happen.
3. **No rate limiting on token validation** — brute-force a 6-digit code? Only 1M combinations, and 100k requests/second makes that minutes.

```js
// ✅ DO THIS - random, single-use, expiring tokens
const token = crypto.randomBytes(32).toString('hex');
const expiresAt = Date.now() + 15 * 60 * 1000; // 15 minutes
await storeResetToken(userId, hash(token), expiresAt); // hash it in the DB too!

// ❌ NEVER DO THIS - predictable and permanent
const token = `${user.id}-${Date.now()}`;
```

Also: hash the reset token in the database (like a password) so a DB leak doesn't hand over working reset tokens. And always show the same response whether the email exists or not — otherwise you're running a user enumeration service.

## Log Everything (Safely)

You need an audit trail: successful logins, failed logins, lockouts, password changes, MFA enrollments. But never log the credentials or tokens themselves:

```js
// ❌ NEVER - full tokens and passwords in logs
logger.info(`Login attempt: ${email} ${password}`);

// ✅ DO THIS - log events and identifiers, not secrets
logger.info('login_failed', { userId: user.id, reason: 'bad_password', ip: req.ip });
```

An audit log is what lets you answer "was this account compromised, and when?" after an incident. Without it you're guessing.

## The Checklist

Here's the practical list to run against any auth system:

- [ ] Passwords hashed with Argon2id or bcrypt, cost tuned to ~100-250ms
- [ ] No custom crypto or hand-rolled token logic
- [ ] JWT algorithm pinned, short expiry, revocable refresh tokens
- [ ] No PII or secrets in token payloads
- [ ] Rate limiting + escalating lockout on login endpoints
- [ ] MFA enforced for privileged accounts
- [ ] Session cookies: HttpOnly, Secure, SameSite, regenerated on login
- [ ] Constant-time comparison for all secret checks
- [ ] Reset tokens: random, expiring, single-use, hashed in DB
- [ ] Audit logging without secrets
- [ ] Secrets (signing keys, DB creds) in a vault, not in code or env files committed to git

## Takeaways

- **Use slow, salted hashes (Argon2id/bcrypt) — never MD5/SHA for passwords.**
- **Pin JWT algorithms and keep access tokens short-lived with revocable refresh tokens.**
- **Protect login endpoints with rate limiting and lockout, and make MFA non-optional for privileged users.**
- **Cookies should be HttpOnly + Secure + SameSite, and session IDs regenerated on login.**
- **Compare secrets in constant time, hash reset tokens, and log everything except the secrets.**
