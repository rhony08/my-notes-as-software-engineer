# Pagination: Offset vs Cursor-Based

Your API returns 50,000 records in a single response. The client times out, the database struggles, and users stare at a loading spinner. This isn't a hypothetical—it's what happens when you skip pagination.

But here's the thing: not all pagination is created equal. The approach you choose affects performance, consistency, and user experience. Let's break down the two main strategies and when each makes sense.

---

## The Two Approaches at a Glance

| Strategy | How It Works | Best For |
|----------|--------------|----------|
| Offset | `LIMIT 20 OFFSET 40` | Static data, simple UIs, page numbers |
| Cursor | `WHERE id > last_id LIMIT 20` | Real-time data, infinite scroll, large datasets |

---

## Offset-Based Pagination

The classic approach: skip N records, take the next M.

```
GET /users?page=2&limit=20
```

Translates to:
```sql
SELECT * FROM users LIMIT 20 OFFSET 20;
```

### How It Looks

```python
# Flask example
@app.route('/users', methods=['GET'])
def get_users():
    page = int(request.args.get('page', 1))
    limit = int(request.args.get('limit', 20))
    offset = (page - 1) * limit
    
    users = db.query("SELECT * FROM users LIMIT ? OFFSET ?", [limit, offset])
    total = db.query("SELECT COUNT(*) FROM users")[0]['count']
    
    return jsonify({
        "data": users,
        "pagination": {
            "page": page,
            "limit": limit,
            "total": total,
            "total_pages": (total + limit - 1) // limit
        }
    })
```

### Why It's Popular

✅ Simple to implement—every SQL database supports `LIMIT/OFFSET`  
✅ Jump to any page—users can click "page 5" directly  
✅ Easy to understand—you get page numbers, total counts, all the UI goodies  

### The Problem with Offset

Here's where it breaks down:

**1. Performance degrades with offset**

```sql
-- Fast: offset 0
SELECT * FROM users LIMIT 20 OFFSET 0;        -- ~1ms

-- Slow: offset 100000
SELECT * FROM users LIMIT 20 OFFSET 100000;   -- ~500ms
```

The database still has to scan and discard 100,000 rows before returning 20. On large tables, this kills performance.

**2. Inconsistent results with real-time data**

```
Page 1: Users [1, 2, 3, ..., 20]
-- Someone inserts a new user at position 5 --
Page 2: Users [21, 22, 23, ..., 40]  -- Wait, where did user 21 go?
```

User 21 shifted to page 1. Your user just saw it twice or missed it entirely. This is the "drifting page" problem.

**3. Count queries are expensive**

```sql
SELECT COUNT(*) FROM users;  -- On 10M rows, this takes seconds
```

You need that count for "Page 1 of 500." But counting millions of rows isn't free.

---

## Cursor-Based Pagination

Instead of skipping records, you start from where you left off.

```
GET /users?cursor=abc123&limit=20
```

Translates to:
```sql
SELECT * FROM users WHERE id > 12345 ORDER BY id LIMIT 20;
```

### How It Looks

```python
import base64
import json

@app.route('/users', methods=['GET'])
def get_users():
    limit = int(request.args.get('limit', 20))
    cursor = request.args.get('cursor')
    
    query = "SELECT * FROM users ORDER BY id LIMIT ?"
    params = [limit + 1]  # Fetch one extra to check if there's more
    
    if cursor:
        # Decode cursor to get the last ID
        last_id = int(base64.urlsafe_b64decode(cursor).decode())
        query = "SELECT * FROM users WHERE id > ? ORDER BY id LIMIT ?"
        params = [last_id, limit + 1]
    
    users = db.query(query, params)
    
    has_more = len(users) > limit
    if has_more:
        users = users[:limit]  # Remove the extra one
    
    next_cursor = None
    if has_more:
        # Encode the last ID as cursor
        next_cursor = base64.urlsafe_b64encode(
            str(users[-1]['id']).encode()
        ).decode()
    
    return jsonify({
        "data": users,
        "pagination": {
            "next_cursor": next_cursor,
            "has_more": has_more
        }
    })
```

### Why It's Better for Large Datasets

✅ Consistent O(1) performance—doesn't matter if you're on page 1 or 10000  
✅ No drifting—new records appear at the end, not in the middle  
✅ No expensive count queries—you only need `has_more`, not total pages  

### The Trade-offs

❌ No jumping to arbitrary pages—can't click "page 5" directly  
❌ No total count—harder to show "Page 1 of 50"  
❌ Cursors can be opaque—`abc123` means nothing to the user  
❌ Sorting changes everything—if you sort by `created_at` instead of `id`, you need a different cursor  

---

## When to Use Each

| Situation | Recommended |
|-----------|-------------|
| Admin dashboard with page numbers | Offset |
| Public API with small dataset (<10k records) | Offset |
| Infinite scroll feed | Cursor |
| Real-time data (chat messages, activity streams) | Cursor |
| Large dataset (>100k records) | Cursor |
| Need to show "Page X of Y" | Offset (or accept expensive count) |
| Mobile app with pull-to-refresh | Cursor |

---

## Real-World Examples

**GitHub API** — Cursor-based for lists
```
Link: <https://api.github.com/repos/.../issues?page=2>; rel="next"
```
GitHub uses offset for simplicity but recommends cursors for large collections.

**Twitter/X API** — Cursor-based
```json
{
  "data": [...],
  "meta": {
    "next_token": "dfhduf84h4h",
    "previous_token": "hd8374g73"
  }
}
```
Twitter's data is too dynamic for offset pagination. Cursors are the only sane choice.

**Stripe API** — Cursor-based
```json
{
  "object": "list",
  "data": [...],
  "has_more": true,
  "starting_after": "prod_abc123"
}
```
Stripe's `starting_after` is essentially a cursor—the ID of the last item you saw.

---

## Handling Edge Cases

### Cursors with Non-ID Sorting

What if you want to sort by `created_at` instead of `id`?

```python
# Cursor encodes both the sort value and ID
cursor_data = json.dumps({
    "last_value": users[-1]['created_at'],
    "last_id": users[-1]['id']
})
cursor = base64.urlsafe_b64encode(cursor_data.encode()).decode()

# Query uses both for pagination
SELECT * FROM users 
WHERE (created_at, id) > (?, ?)
ORDER BY created_at, id
LIMIT 20
```

This handles ties—if two records have the same `created_at`, the ID breaks the tie.

### Bidirectional Pagination

For "previous" and "next" buttons, you need both cursors:

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6IDEwMH0=",
    "prev_cursor": "eyJpZCI6IDF9"
  }
}
```

The previous cursor works by reversing the sort direction and using `WHERE id < ?`.

---

## Common Mistakes

### ❌ Using Offset for Everything

```python
# This will hurt at scale
@app.route('/messages')
def get_messages():
    page = request.args.get('page', 1)
    return db.query(f"SELECT * FROM messages LIMIT 50 OFFSET {(page-1)*50}")
```

Works fine with 100 messages. Falls apart with 10 million.

### ❌ Exposing Database IDs as Cursors

```python
# Cursor = user_id directly
cursor = str(users[-1]['id'])
```

This leaks information about your data volume and structure. Encode it.

### ❌ Ignoring the Sort Order

```sql
-- If you're sorting by created_at, don't page by id!
SELECT * FROM users WHERE id > ? ORDER BY created_at LIMIT 20;  -- WRONG
SELECT * FROM users WHERE created_at > ? ORDER BY created_at LIMIT 20;  -- RIGHT
```

The cursor must match the sort column.

---

## Quick Reference: Response Formats

**Offset pagination:**
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 1500,
    "total_pages": 75
  }
}
```

**Cursor pagination:**
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6IDIxfQ==",
    "has_more": true
  }
}
```

---

## Takeaways

- **Offset pagination** is simple and supports page numbers, but degrades with large offsets and real-time data
- **Cursor pagination** is consistent and performant, but can't jump to arbitrary pages
- Use **offset** for admin panels, small datasets, and when you need page numbers
- Use **cursor** for feeds, large datasets, and real-time data
- Cursors should encode the last seen value (and ID for tie-breaking)
- Don't expose raw database IDs—encode your cursors
- Match your cursor to your sort order

Pick the one that fits your use case. Your database (and users) will thank you.