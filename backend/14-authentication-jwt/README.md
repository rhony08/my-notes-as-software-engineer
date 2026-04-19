# Authentication with JWT: Implementation and Security

Every request your API receives needs to answer one question: *who is this user, and what can they do?* Session-based auth works, but it requires server-side storage and gets messy with multiple servers. JSON Web Tokens (JWT) take a different approach—instead of the server remembering who's logged in, the client carries proof.

But JWTs come with their own set of pitfalls. Implemented poorly, they create security holes that are worse than no auth at all. Let's explore how to do it right.

## How JWTs Work

A JWT has three parts separated by dots: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjM0NSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTY4MDAwMDAwMH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

- **Header**: Algorithm and token type
- **Payload**: Claims (user ID, role, expiration)
- **Signature**: Verifies the token wasn't tampered with

The server creates the token using a secret key. The client sends it back with each request. The server verifies the signature—no database lookup required.

```javascript
// Creating a JWT (Node.js with jsonwebtoken)
const jwt = require('jsonwebtoken');

const token = jwt.sign(
  { userId: '12345', role: 'admin' },  // payload
  process.env.JWT_SECRET,               // secret key
  { expiresIn: '1h' }                   // options
);
```

## What Can Go Wrong

### ❌ Weak or Hardcoded Secrets

```javascript
// ❌ NEVER DO THIS
const token = jwt.sign(payload, 'my-secret-key');
const token = jwt.sign(payload, 'password123');
const token = jwt.sign(payload, process.env.JWT_SECRET || 'fallback-secret');
```

```javascript
// ✅ Proper secret management
// Generate a strong secret: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
const token = jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: '15m' });

// .env file (never commit this!)
JWT_SECRET=a1b2c3d4e5f6...256_bits_of_randomness
```

A weak secret means attackers can forge valid tokens. If they guess your secret, they can impersonate any user—including admins.

### ❌ Sensitive Data in Payload

```javascript
// ❌ NEVER PUT THIS IN A JWT
const token = jwt.sign({
  email: 'user@example.com',
  password: 'hashed_password',    // absolutely not
  ssn: '123-45-6789',             // no
  creditCard: '4111-1111-1111-1111' // stop
}, secret);
```

JWT payloads are **base64 encoded, not encrypted**. Anyone who gets the token can read its contents by pasting it into jwt.io. The signature only proves it wasn't modified—it doesn't hide the data.

```javascript
// ✅ Minimal payload
const token = jwt.sign({
  sub: 'user-uuid-123',    // subject (user ID)
  role: 'viewer',
  iat: Math.floor(Date.now() / 1000)  // issued at
}, secret, { expiresIn: '15m' });
```

### ❌ Never-Expiring Tokens

```javascript
// ❌ Token valid forever
const token = jwt.sign({ userId: '12345' }, secret);
```

If an attacker steals this token, they have permanent access. There's no way to revoke it without changing your secret (which invalidates *all* users' tokens).

```javascript
// ✅ Short-lived tokens with refresh mechanism
const accessToken = jwt.sign({ userId: '12345' }, secret, { expiresIn: '15m' });
const refreshToken = jwt.sign({ userId: '12345', type: 'refresh' }, refreshSecret, { expiresIn: '7d' });
```

## Refresh Token Pattern

Short-lived access tokens + long-lived refresh tokens = best of both worlds.

| Token Type | Lifetime | Stored In | Purpose |
|------------|----------|-----------|---------|
| Access Token | 5-15 minutes | Memory (JS variable) | Authorize API requests |
| Refresh Token | 7-30 days | HttpOnly cookie or secure storage | Get new access tokens |

```javascript
// Login endpoint returns both
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body);
  
  const accessToken = jwt.sign(
    { sub: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }
  );
  
  const refreshToken = jwt.sign(
    { sub: user.id, type: 'refresh' },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  );
  
  // Store refresh token in DB for revocation capability
  await storeRefreshToken(user.id, refreshToken);
  
  res.json({ accessToken });
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: true,        // HTTPS only
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days
  });
});
```

```javascript
// Refresh endpoint
app.post('/refresh', async (req, res) => {
  const { refreshToken } = req.cookies;
  
  if (!refreshToken) return res.sendStatus(401);
  
  try {
    const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    
    // Check if token was revoked
    if (await isTokenRevoked(refreshToken)) {
      return res.sendStatus(403);
    }
    
    const newAccessToken = jwt.sign(
      { sub: payload.sub, role: payload.role },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }
    );
    
    res.json({ accessToken: newAccessToken });
  } catch (err) {
    res.sendStatus(403);
  }
});
```

## Token Storage Trade-offs

Where you store tokens on the client matters—a lot.

| Storage Method | XSS Vulnerable | CSRF Vulnerable | Recommendation |
|----------------|----------------|-----------------|----------------|
| localStorage | ❌ Yes | ✅ No | Avoid for sensitive tokens |
| Memory (JS variable) | ❌ Yes | ✅ No | Best for access tokens (lost on refresh) |
| HttpOnly Cookie | ✅ No | ❌ Yes | Good for refresh tokens |
| HttpOnly + SameSite Cookie | ✅ No | ✅ Mostly | Best for refresh tokens |

The pattern I use:
- **Access token**: Store in memory (JS variable), lost on page refresh but that's fine—it's short-lived
- **Refresh token**: HttpOnly cookie with `sameSite: 'strict'`

```javascript
// Client-side token handling
let accessToken = null;

async function login(credentials) {
  const response = await fetch('/login', {
    method: 'POST',
    body: JSON.stringify(credentials),
    credentials: 'include'  // send/receive cookies
  });
  
  const data = await response.json();
  accessToken = data.accessToken;
}

async function fetchWithAuth(url, options = {}) {
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${accessToken}`
    },
    credentials: 'include'
  });
  
  if (response.status === 401) {
    // Try to refresh
    const refreshResponse = await fetch('/refresh', {
      method: 'POST',
      credentials: 'include'
    });
    
    if (refreshResponse.ok) {
      const { accessToken: newToken } = await refreshResponse.json();
      accessToken = newToken;
      
      // Retry original request
      return fetch(url, {
        ...options,
        headers: {
          ...options.headers,
          'Authorization': `Bearer ${accessToken}`
        },
        credentials: 'include'
      });
    }
    
    // Refresh failed - redirect to login
    window.location.href = '/login';
    return;
  }
  
  return response;
}
```

## Token Revocation

Here's the ugly truth about JWTs: **you can't revoke them without additional infrastructure**. The token is valid until it expires.

### Options for Revocation

| Approach | Pros | Cons |
|----------|------|------|
| Short expiration + refresh | Simple, limits exposure | User logged out on expiration |
| Token blacklist in Redis | Fast, precise | Requires infrastructure |
| Refresh token rotation | Good security posture | More complex |
| Version claim in token | Easy per-user revocation | Requires DB check (defeats stateless purpose) |

```javascript
// Redis-based blacklist
async function revokeToken(token) {
  const decoded = jwt.decode(token);
  const ttl = decoded.exp - Math.floor(Date.now() / 1000);
  
  if (ttl > 0) {
    await redis.setex(`blacklist:${token}`, ttl, '1');
  }
}

// Verify with blacklist check
async function verifyToken(token) {
  const isBlacklisted = await redis.exists(`blacklist:${token}`);
  if (isBlacklisted) {
    throw new Error('Token revoked');
  }
  
  return jwt.verify(token, process.env.JWT_SECRET);
}
```

## Logout Implementation

```javascript
app.post('/logout', async (req, res) => {
  const { refreshToken } = req.cookies;
  
  // Blacklist the refresh token
  if (refreshToken) {
    await revokeRefreshToken(refreshToken);
  }
  
  // Clear the cookie
  res.clearCookie('refreshToken', {
    httpOnly: true,
    secure: true,
    sameSite: 'strict'
  });
  
  res.sendStatus(200);
});
```

## Common Attack Vectors

### Algorithm Confusion

Some JWT libraries accept `alg: 'none'` as valid, bypassing signature verification entirely.

```javascript
// ❌ Vulnerable library might accept this
const maliciousToken = base64url(header) + '.' + base64url({ sub: 'admin' }) + '.';
```

**Fix**: Always specify allowed algorithms explicitly.

```javascript
// ✅ Explicit algorithm verification
jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256']  // Only allow HMAC-SHA256
});
```

### Timing Attacks

Comparing tokens character-by-character can leak information through timing differences.

```javascript
// ❌ Vulnerable to timing attacks
if (token === expectedToken) { ... }

// ✅ Use constant-time comparison
const crypto = require('crypto');
if (crypto.timingSafeEqual(Buffer.from(token), Buffer.from(expectedToken))) { ... }
```

## Middleware Pattern

```javascript
function authMiddleware(requiredRole = null) {
  return async (req, res, next) => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'Missing or invalid authorization header' });
    }
    
    const token = authHeader.split(' ')[1];
    
    try {
      const decoded = await verifyToken(token);
      
      // Optional role check
      if (requiredRole && decoded.role !== requiredRole) {
        return res.status(403).json({ error: 'Insufficient permissions' });
      }
      
      req.user = decoded;
      next();
    } catch (err) {
      if (err.name === 'TokenExpiredError') {
        return res.status(401).json({ error: 'Token expired' });
      }
      if (err.name === 'JsonWebTokenError') {
        return res.status(401).json({ error: 'Invalid token' });
      }
      return res.status(500).json({ error: 'Authentication failed' });
    }
  };
}

// Usage
app.get('/profile', authMiddleware(), getProfile);
app.delete('/users/:id', authMiddleware('admin'), deleteUser);
```

## Key Takeaways

- **Keep payloads minimal**—no sensitive data, JWTs are readable
- **Use strong secrets** (256+ bits) from environment variables, never hardcoded
- **Short-lived access tokens** (5-15 minutes) with refresh token rotation
- **Store access tokens in memory**, refresh tokens in HttpOnly cookies
- **Implement revocation if you need logout**—pure JWT can't be revoked
- **Always specify algorithms** when verifying to prevent confusion attacks
- **Set reasonable expiration**—balance security with user experience
- **Use refresh tokens with rotation** for sessions that need to persist

JWTs are a tool, not a silver bullet. For simple apps with few users, session-based auth might be easier. For distributed systems, APIs, and microservices, JWTs shine—but only when implemented with care.