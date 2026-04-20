# API Versioning Strategies

Your API just broke all your mobile clients. You changed a field name from `userId` to `user_id`, deployed to production, and suddenly thousands of users can't log in. Sound familiar?

Versioning isn't just about being "professional"—it's about not breaking things for people relying on your code. In this article, we'll cover the main strategies for versioning REST APIs, their trade-offs, and when to use each.

---

## Why Versioning Matters

APIs are contracts. When you expose an endpoint, clients depend on it staying stable. Changing that contract without versioning is like changing the locks on a house while people are still living inside.

**What breaks without versioning:**

- Field renames → client code crashes
- Removing fields → clients get null/undefined
- Changing response structure → parsing errors
- Adding required fields → old clients fail
- Changing behavior → subtle bugs

**When you actually need versioning:**

- You have external consumers (mobile apps, third-party integrations)
- Your API is consumed by code you don't control
- You're iterating rapidly and can't coordinate every change with consumers

---

## The Three Main Approaches

There's no "best" approach—each has trade-offs. Here's how they compare:

| Strategy | Where Version Lives | Pros | Cons |
|----------|---------------------|------|------|
| URI Path | `/v1/users` | Explicit, cacheable, visible in logs | Breaks all URLs on version change |
| Query Param | `/users?version=1` | Optional, backward-compatible | Easy to miss, not cache-friendly |
| Header | `Accept: application/vnd.api+json;version=1` | Clean URLs, RESTful | Hidden, harder to debug/test |

---

## 1. URI Path Versioning

The most common approach: embed the version right in the URL.

```
GET /v1/users
GET /v2/users
```

### How It Looks

```python
# Flask example
@app.route('/v1/users', methods=['GET'])
def get_users_v1():
    return jsonify({
        "users": [
            {"id": 1, "name": "Alice"}
        ]
    })

@app.route('/v2/users', methods=['GET'])
def get_users_v2():
    return jsonify({
        "data": [
            {"id": 1, "full_name": "Alice Smith", "created_at": "2024-01-15"}
        ]
    })
```

### When to Use It

✅ You have public APIs with external consumers  
✅ You want versions to be obvious in logs and documentation  
✅ You need CDN caching per version  

### Trade-offs

❌ Version is everywhere—in your docs, client code, URLs shared in emails  
❌ Changing versions means updating all URLs  
❌ Feels "impure" to REST purists (the resource didn't change, just the representation)

---

## 2. Query Parameter Versioning

Keep URLs clean, specify version as a query param.

```
GET /users?version=1
GET /users?version=2
```

### How It Looks

```python
@app.route('/users', methods=['GET'])
def get_users():
    version = request.args.get('version', '1')  # Default to v1
    
    if version == '1':
        return jsonify({"users": [{"id": 1, "name": "Alice"}]})
    elif version == '2':
        return jsonify({"data": [{"id": 1, "full_name": "Alice Smith"}]})
```

### When to Use It

✅ You want backward compatibility by default (omit version = latest)  
✅ You're versioning internally and can control clients  
✅ You want minimal URL changes  

### Trade-offs

❌ Easy for clients to forget the param  
❌ Not cache-friendly—same URL with different params = cache miss  
❌ Query params are meant for filtering, not versioning (semantic confusion)  

---

## 3. Header Versioning

The "cleanest" URLs, version lives in the request header.

```
GET /users
Accept: application/vnd.myapi+json;version=1
```

### How It Looks

```python
@app.route('/users', methods=['GET'])
def get_users():
    accept_header = request.headers.get('Accept', '')
    
    if 'version=1' in accept_header:
        return jsonify({"users": [{"id": 1, "name": "Alice"}]})
    elif 'version=2' in accept_header or 'version=' not in accept_header:
        return jsonify({"data": [{"id": 1, "full_name": "Alice Smith"}]})
```

### When to Use It

✅ You want pure RESTful URLs (the resource is `/users`, not `/v1/users`)  
✅ You're building a professional API with documentation standards  
✅ You control both client and server (can ensure headers are set)  

### Trade-offs

❌ Hidden—can't see version in browser URL bar  
❌ Harder to test with curl/Postman (more typing)  
❌ Browser requests won't have the header by default  

---

## Which One Should You Pick?

**Default choice: URI Path (`/v1/users`)**

Why? It's the most pragmatic:
- Works everywhere (browsers, curl, mobile apps)
- Obvious in logs and monitoring
- Easy to document and test
- Most widely adopted (GitHub, Stripe, Twitter all use it)

**Consider alternatives when:**

| Situation | Better Choice |
|-----------|---------------|
| Internal-only API | Query param is fine |
| You control all clients | Header versioning works |
| Need to hide version complexity | Header or default-to-latest query param |

---

## Version Lifecycle Management

Versioning isn't just about the "how"—it's about managing the lifecycle.

### The Deprecation Pattern

```python
@app.route('/v1/users', methods=['GET'])
def get_users_v1():
    response = jsonify({"users": [...]})
    response.headers['X-API-Deprecated'] = 'true'
    response.headers['X-API-Sunset'] = '2024-06-01'
    response.headers['X-API-Migration-Guide'] = 'https://api.example.com/docs/v2-migration'
    return response
```

**Good practices:**

- Return deprecation headers on old versions
- Set a sunset date (3-6 months is reasonable)
- Provide migration documentation
- Notify clients via email/announcements

### How Many Versions to Support?

Real talk: supporting every version forever is unsustainable.

```
v1 → Deprecated (sunset in 3 months)
v2 → Stable (current)
v3 → Beta (upcoming)
```

**Rule of thumb:** Support at most 2-3 versions concurrently. Older versions get deprecated with clear timelines.

---

## Common Mistakes

### ❌ Versioning Everything

Not every change needs a version bump. Additive changes (new fields, new endpoints) are backward-compatible.

```json
// v1 response
{"id": 1, "name": "Alice"}

// v2 response - ADDITIVE, no version needed
{"id": 1, "name": "Alice", "email": "alice@example.com"}
```

### ❌ Versioning by Behavior

Version numbers should reflect API structure, not implementation details.

```python
# ❌ DON'T do this
if version == '1':
    return slow_database_query()
elif version == '2':
    return cached_query()

# ✅ Version structure, not implementation
# Both versions return the same structure, just optimized differently
```

### ❌ No Communication Plan

Deprecating an API version without telling anyone is how you break production apps.

---

## Quick Reference

| Change Type | Version Bump Needed? |
|-------------|---------------------|
| Add new field | ❌ No |
| Add new endpoint | ❌ No |
| Remove field | ✅ Yes |
| Rename field | ✅ Yes |
| Change field type | ✅ Yes |
| Add required field | ✅ Yes (or make optional with default) |
| Change error format | ✅ Yes |
| Performance improvement | ❌ No |

---

## Takeaways

- **URI path versioning** is the pragmatic default—it's visible, cacheable, and works everywhere
- **Header versioning** is cleaner but harder to debug—use for APIs where you control clients
- **Query params** work for internal APIs but aren't great for public consumption
- Additive changes don't need versioning—only breaking changes do
- Support 2-3 versions max, with clear deprecation timelines
- Communicate deprecations through headers, docs, and announcements

Your future self (and your API consumers) will thank you.