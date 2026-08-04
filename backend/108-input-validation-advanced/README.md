# Input Validation: Beyond Basics

Your API validates email formats, checks that `name` is a string, and makes sure `quantity` is a positive integer. Great. Now a user sends `quantity: "999999999999999999999"` and your inventory system silently overflows. Another sends `email: "admin@example.com "` with a sneaky trailing space and bypasses your duplicate-account check. Someone else sends `role: "admin"` in a signup payload and your ORM just... accepts it.

Basic validation keeps honest users honest. It does nothing against anyone who's actually trying to break your system. This article covers the validation that actually matters: canonicalization, type confusion, mass assignment, and the business-logic layer most of us forget entirely.

## What "valid" even means

Here's the thing — validation isn't one check. It's a pipeline, and each stage answers a different question:

| Layer | Question | Example |
|-------|----------|---------|
| Syntax | Is it the right shape? | `email` is a string matching a sane pattern |
| Type | Is it the right kind? | `quantity` is an integer, not a float or string |
| Range | Is it in acceptable bounds? | `quantity` between 1 and 10,000 |
| Business | Is it allowed *right now*? | user can only order 5 of an item with stock 3 |
| Authorization | Is this user allowed to set this field? | a normal user can't set `role: "admin"` |

Most validation libraries stop after the first two rows. That's where the trouble starts.

## Whitelist, don't blacklist

Blacklist validation says "reject anything that contains `' OR 1=1 --`". Whitelist says "accept only things that match this exact pattern". One of these has a maintenance problem, and it's not the whitelist.

```
// ❌ Blacklist: you're always one bypass away from a hole
if (input.includes("'") || input.includes(";") || input.includes("--")) {
  throw new Error("Invalid input");
}

// ✅ Whitelist: only characters you explicitly allow
if (!/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/.test(email)) {
  throw new Error("Invalid email");
}
```

Blacklists are reactive by design — you discover the next weird input after someone exploits it. Whitelists are proactive: if it's not in the set, it doesn't get in, period. This applies everywhere, not just strings. Enums, HTTP methods, sort directions, status transitions — define the allowed set explicitly.

## Normalize before you validate

This is the one that bites everyone eventually. Two strings can look identical to a user but be completely different bytes.

- `"café"` in NFD form vs NFC form (decomposed vs composed Unicode)
- `"admin@example.com"` vs `"admin@example.com\n"` vs `" admin@example.com"`
- `"/etc/passwd"` vs `"/etc/../etc/passwd"` (path traversal, hello)

If you validate first and normalize after, your validation is theater. The rule: **normalize, then validate, then use**. Trim whitespace, canonicalize Unicode, resolve paths — *then* check the pattern.

```
// ❌ Validate raw input, store it unnormalized → duplicates + bypasses
if (isValidEmail(email)) {
  await db.users.create({ email }); // "Bob@x.com" and "bob@x.com" are now two users
}

// ✅ Normalize → validate → use
const normalized = email.trim().toLowerCase();
if (isValidEmail(normalized)) {
  await db.users.create({ email: normalized });
}
```

## Type confusion: strings that lie

JSON gives you strings, numbers, booleans, arrays, and objects. But what arrives on the wire is often a string that *claims* to be a number. `"1e3"`, `"+5"`, `"0x10"`, `"1.0"`, and `"01"` are all strings that parse into very different things depending on your coercion rules.

```
// ❌ PHP-style loose coercion: "1e3" becomes 1000, "abc" becomes 0
if (quantity > 0) { /* "abc" passes because it coerces to 0... wait, no, 0 > 0 is false.
                       But "1e3" → 1000 passes and blows past your max */ }

// ✅ Explicit type + range check
const qty = Number.parseInt(quantity, 10);        // radix 10, always
if (!Number.isInteger(qty) || qty < 1 || qty > 10000) {
  throw new ValidationError("quantity must be an integer between 1 and 10000");
}
```

Same story with booleans: `"false"` is truthy in most languages. If you accept booleans from JSON, check `typeof value === "boolean"` instead of `if (value)`.

And integer overflow is real: a client sending `quantity: 9223372036854775807` might pass your `> 0` check, then overflow server-side arithmetic and wrap around to a negative — or worse, zero. Always check upper bounds, not just lower ones.

## Mass assignment: stop trusting the payload

Here's a classic. Your `User` model has a `role` column. Your signup endpoint does:

```
// ❌ Spread the whole body into the model — role: "admin" sails right through
const user = await User.create(req.body);

// ✅ Explicit allowlist of what the client may set
const { email, password, name } = req.body;
const user = await User.create({ email, password, name });
```

This is the mass-assignment vulnerability that has burned Rails, Laravel, and Django apps for a decade. The fix is structural: never bind request bodies directly to models, and always maintain an explicit allowlist of client-settable fields. Same for query params and headers — `?admin=true` shouldn't change anything about a user's permissions.

## Business logic: the layer validators don't know about

Syntax validation can't catch a user ordering −5 items, transferring more money than they have, or booking a flight in the past. That's business logic validation, and it lives in your service layer, close to the data — not in a schema at the API edge.

```
// ❌ Schema validates shape, then the service trusts it blindly
function createOrder(userId, items) {
  return db.orders.create({ userId, items });
}

// ✅ Business rules enforced where the data lives
function createOrder(userId, items) {
  const user = await db.users.find(userId);
  for (const item of items) {
    const product = await db.products.find(item.productId);
    if (product.stock < item.quantity) {
      throw new BusinessRuleError(`${product.name} only has ${product.stock} left`);
    }
    if (!user.canAfford(product.price * item.quantity)) {
      throw new BusinessRuleError("Insufficient funds");
    }
  }
  // ... create the order, decrement stock, all in a transaction
}
```

This is also where you check ownership ("can *this* user edit *this* resource?") — the validation that stops IDOR attacks. A schema library can't know that resource 1234 belongs to user 42. Only your service layer can.

A good pattern: **validate at the edge for fast failure (shape), validate in the service for correctness (business rules), validate at the DB for integrity (constraints, uniqueness)**. All three, always. Defense in depth isn't a buzzword, it's the difference between "caught at the API" and "corrupted data in prod".

## Performance trade-off

There's a real cost here, and pretending otherwise is dishonest. Business-logic validation means extra DB queries per request. For high-traffic endpoints you'll want to:

- Cache product stock and prices (with short TTLs)
- Batch validation queries instead of N+1 lookups
- Move expensive checks to async jobs when the check isn't on the critical path (e.g., "is this email on a blocklist" can happen after the response)

On the flip side: validation errors are cheap. A rejected request costs a few milliseconds. A bad row in production costs hours of debugging, a data migration, or a security incident. Spend the milliseconds.

## Actionable takeaways

- **Whitelist everything**: allowed characters, allowed enums, allowed fields. Blacklists are a treadmill.
- **Normalize before validating**: trim, lowercase, canonicalize Unicode and paths. Validate what you actually store.
- **Be explicit about types**: check `typeof`/`Number.isInteger` instead of relying on coercion, and always bound the upper range, not just the lower.
- **Never spread request bodies into models**: maintain an explicit allowlist of client-settable fields.
- **Validate in three layers**: syntax at the edge, business rules in the service, constraints at the database.
- **Ownership checks are validation**: every mutating endpoint should verify the caller is allowed to touch that resource.

The goal isn't to build an impenetrable validation fortress — it's to make sure the data you persist is the data you *intended* to persist. Everything else is just how you get there.
