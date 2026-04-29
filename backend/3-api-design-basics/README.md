# RESTful API Design Basics

Your API got a request for `/users/123`, but what should it return? A user object? An error? What if the user doesn't exist? And should you use `PUT` or `PATCH` to update their email?

These questions seem simple until you're staring at 47 different endpoint patterns across your codebase, each returning errors in a different format. Your frontend team is confused. Your mobile team is frustrated. And debugging? Good luck figuring out what went wrong.

Good API design isn't about following rules for the sake of it. It's about building something predictable, debuggable, and actually usable. Let's cover the fundamentals.

---

## HTTP Methods: Use Them Right

HTTP methods have meanings. Ignoring them makes your API confusing.

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Retrieve data | Yes | Yes |
| POST | Create new resource | No | No |
| PUT | Replace entire resource | Yes | No |
| PATCH | Partial update | No* | No |
| DELETE | Remove resource | Yes | No |

*PATCH can be idempotent, depends on implementation.

### The Common Mistakes

```javascript
// ❌ GET request modifying data
app.get('/users/:id/deactivate', (req, res) => {
  User.deactivate(req.params.id);
  res.json({ success: true });
});

// ✅ POST for modifications
app.post('/users/:id/deactivate', (req, res) => {
  User.deactivate(req.params.id);
  res.json({ success: true });
});
```

Why does it matter? GET requests can be cached, retried, and pre-fetched by browsers. If your GET request deletes data, a browser pre-fetch could accidentally trigger it.

```javascript
// ❌ Using POST for everything
app.post('/get-users', (req, res) => { ... });
app.post('/update-user', (req, res) => { ... });

// ✅ Semantic methods
app.get('/users', (req, res) => { ... });
app.patch('/users/:id', (req, res) => { ... });
```

Using POST for everything works, but you lose the semantics. Future you (and your team) will thank you for following conventions.

---

## Status Codes: More Than 200 and 500

Status codes are your API's way of communicating. Use them.

### The Essentials

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH, DELETE |
| 201 | Created | Successful POST that created something |
| 204 | No Content | Successful request with no response body |
| 400 | Bad Request | Invalid input, malformed JSON |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource, version conflict |
| 422 | Unprocessable Entity | Valid JSON but invalid data |
| 429 | Too Many Requests | Rate limited |
| 500 | Internal Server Error | Something broke on your end |

### Common Patterns

```javascript
// ❌ Everything returns 200 with error in body
app.get('/users/:id', (req, res) => {
  const user = User.find(req.params.id);
  if (!user) {
    res.status(200).json({ error: 'User not found' });
    return;
  }
  res.json(user);
});

// ✅ Correct status codes
app.get('/users/:id', (req, res) => {
  const user = User.find(req.params.id);
  if (!user) {
    res.status(404).json({ error: 'User not found' });
    return;
  }
  res.json(user);
});
```

Why? HTTP libraries, monitoring tools, and caches all understand status codes. A monitoring system sees a 500 spike differently than a 200 spike.

```javascript
// ❌ Using 400 for auth errors
res.status(400).json({ error: 'Invalid API key' });

// ✅ Use correct auth status
res.status(401).json({ error: 'Invalid API key' });
```

400 means "your request was malformed." 401 means "you're not authenticated." Different problems, different codes.

### What About 500 vs 4xx?

```javascript
// ❌ Blaming the client for server errors
app.get('/users/:id', (req, res) => {
  try {
    const user = User.find(req.params.id);
    res.json(user);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// ✅ Server errors are server errors
app.get('/users/:id', (req, res) => {
  try {
    const user = User.find(req.params.id);
    if (!user) {
      res.status(404).json({ error: 'User not found' });
      return;
    }
    res.json(user);
  } catch (error) {
    // This is a server problem, not the client's fault
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

4xx = client made a mistake. 5xx = you made a mistake. Own your errors.

---

## URL Design: Keep It Predictable

### Resource Naming

```javascript
// ❌ Inconsistent, verb-based URLs
app.get('/getAllUsers', ...);
app.post('/createNewUser', ...);
app.delete('/removeUserById/:id', ...);

// ✅ Resource-based, noun URLs
app.get('/users', ...);           // Get all users
app.post('/users', ...);          // Create a user
app.delete('/users/:id', ...);    // Delete a user
```

REST is about resources. URLs identify resources. HTTP methods describe actions.

### Nested Resources

```javascript
// ❌ Flat URLs for related resources
app.get('/posts', ...);                    // All posts
app.get('/user-posts/:userId', ...);       // User's posts

// ✅ Nested resources show relationships
app.get('/users/:userId/posts', ...);      // User's posts
app.post('/users/:userId/posts', ...);    // Create post for user
```

But don't go too deep:

```javascript
// ❌ Deeply nested, hard to parse
app.get('/users/:userId/posts/:postId/comments/:commentId/replies/:replyId');

// ✅ Keep it reasonable, use query params
app.get('/comments/:commentId/replies/:replyId');
// Or query params for filtering
app.get('/comments?postId=123&userId=456');
```

### Query Params for Filtering

```javascript
// ❌ URL params for everything
app.get('/users/active/role/admin');

// ✅ Query params for filtering, sorting, pagination
app.get('/users?status=active&role=admin&sort=createdAt&order=desc&page=1');
```

---

## Request & Response Format

### Consistent Response Structure

```javascript
// ❌ Inconsistent responses across endpoints
app.get('/users/:id', (req, res) => {
  res.json(user);  // Just the user object
});

app.get('/users', (req, res) => {
  res.json({ users, total, page });  // Wrapped in object
});

// ✅ Consistent structure
app.get('/users/:id', (req, res) => {
  res.json({
    data: user,
    success: true
  });
});

app.get('/users', (req, res) => {
  res.json({
    data: users,
    meta: { total, page, perPage },
    success: true
  });
});
```

Consistency lets frontend developers write generic response handlers instead of per-endpoint logic.

### Error Response Format

```javascript
// ❌ Different error formats
res.status(404).json('User not found');
res.status(400).json({ error: 'Invalid email' });
res.status(500).json({ message: 'Database error', code: 'DB001' });

// ✅ Consistent error format
res.status(400).json({
  error: {
    code: 'VALIDATION_ERROR',
    message: 'Invalid email format',
    details: { field: 'email', value: 'not-an-email' }
  }
});

res.status(404).json({
  error: {
    code: 'NOT_FOUND',
    message: 'User not found',
    details: { userId: '123' }
  }
});
```

Include enough detail to debug, but not sensitive info. Error codes help clients handle errors programmatically.

---

## Pagination: Don't Return Everything

```javascript
// ❌ Returning all records
app.get('/posts', (req, res) => {
  const posts = Post.findAll();  // What if there are 100,000?
  res.json(posts);
});

// ✅ Pagination by default
app.get('/posts', (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  
  const posts = Post.findAll({ offset: (page - 1) * limit, limit });
  const total = Post.count();
  
  res.json({
    data: posts,
    meta: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit)
    }
  });
});
```

Always paginate list endpoints. What's 100 records today could be 100,000 next year.

---

## Practical Checklist

| ✓ | Check |
|---|-------|
| ☐ | Using correct HTTP methods for each operation? |
| ☐ | Returning appropriate status codes? |
| ☐ | URLs are noun-based and predictable? |
| ☐ | Response format is consistent across all endpoints? |
| ☐ | Errors follow a consistent structure with codes? |
| ☐ | List endpoints have pagination? |
| ☐ | Input validation happens before processing? |
| ☐ | API versioned (URL or header)? |

---

## Key Takeaways

- **Methods matter** — Use GET for reads, POST for creates, PUT/PATCH for updates, DELETE for deletes. No, GET shouldn't modify data.
- **Status codes communicate** — They're not just for browsers. Monitoring, caching, and client libraries all rely on them.
- **Be consistent** — Response formats, error structures, URL patterns. Pick one style and stick to it.
- **Paginate by default** — List endpoints will grow. Save yourself the headache later.
- **Document edge cases** — What happens with invalid input? Missing resources? Auth failures? Make it predictable.

Good API design is invisible. Developers using your API shouldn't have to guess how it works. They should just know.