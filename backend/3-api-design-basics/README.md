# RESTful API Design Basics: Methods, Status Codes, and Best Practices

REST (Representational State Transfer) has become the de facto standard for building web APIs. A well-designed REST API is intuitive to use, easy to maintain, and scales gracefully. However, many APIs suffer from inconsistent design choices that make them frustrating to work with. In this article, we'll explore the fundamentals of RESTful API design, covering HTTP methods, status codes, and best practices that will help you build APIs developers actually enjoy using.

## Table of Contents

- [What is REST?](#what-is-rest)
- [HTTP Methods](#http-methods)
- [HTTP Status Codes](#http-status-codes)
- [Resource Naming Conventions](#resource-naming-conventions)
- [Request and Response Formats](#request-and-response-formats)
- [Error Handling](#error-handling)
- [Pagination and Filtering](#pagination-and-filtering)
- [Best Practices Summary](#best-practices-summary)
- [Conclusion](#conclusion)

## What is REST?

REST is an architectural style for designing networked applications. It was introduced by Roy Fielding in his 2000 doctoral dissertation. RESTful APIs have several key principles:

### Core Principles

**1. Client-Server Separation**
The client and server are independent. The client doesn't need to know how the server stores data, and the server doesn't need to know how the client displays it.

**2. Stateless**
Each request contains all information needed to process it. The server doesn't store client context between requests.

**3. Cacheable**
Responses should indicate whether they can be cached, improving performance and reducing server load.

**4. Uniform Interface**
A consistent way of interacting with resources through:
- Resource identification (URIs)
- Resource manipulation through representations
- Self-descriptive messages
- Hypermedia as the engine of application state (HATEOAS)

**5. Layered System**
The client can't tell whether it's connected directly to the server or through intermediaries (load balancers, proxies, etc.).

## HTTP Methods

HTTP methods define the action to perform on a resource. Using them correctly is fundamental to RESTful design.

### GET - Retrieve Resources

Retrieve a representation of a resource without modifying it.

```
GET /users           # Get all users
GET /users/123       # Get user with ID 123
GET /users/123/orders # Get orders for user 123
```

**Key characteristics:**
- Safe: Should not modify server state
- Idempotent: Multiple identical requests have the same effect
- Cacheable: Responses can be cached
- Should not have a request body (though technically allowed)

**Example request:**
```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
```

### POST - Create Resources

Create a new resource. The server assigns the resource ID.

```
POST /users          # Create a new user
POST /users/123/orders # Create a new order for user 123
```

**Key characteristics:**
- Not safe: Modifies server state
- Not idempotent: Multiple identical requests create multiple resources
- Request body contains the resource data

**Example request:**
```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

**Example response:**
```http
HTTP/1.1 201 Created
Location: /api/users/123
Content-Type: application/json

{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

### PUT - Replace Resources

Replace an entire resource with new data. All fields are replaced; missing fields are removed.

```
PUT /users/123       # Replace user 123 entirely
```

**Key characteristics:**
- Not safe: Modifies server state
- Idempotent: Multiple identical requests have the same effect
- Requires the complete resource in the request body

**Example request:**
```http
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```

### PATCH - Partial Update

Update part of a resource. Only specified fields are modified.

```
PATCH /users/123     # Update specific fields of user 123
```

**Key characteristics:**
- Not safe: Modifies server state
- Not necessarily idempotent (depends on implementation)
- Only requires fields being updated

**Example request:**
```http
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "email": "newemail@example.com"
}
```

### DELETE - Remove Resources

Remove a resource from the server.

```
DELETE /users/123     # Delete user 123
```

**Key characteristics:**
- Not safe: Modifies server state
- Idempotent: Deleting the same resource multiple times has the same effect

**Example request:**
```http
DELETE /api/users/123 HTTP/1.1
Host: api.example.com
```

**Example response:**
```http
HTTP/1.1 204 No Content
```

### Method Summary Table

| Method | Safe | Idempotent | Use Case |
|--------|------|------------|----------|
| GET | Yes | Yes | Retrieve resources |
| POST | No | No | Create new resources |
| PUT | No | Yes | Replace entire resources |
| PATCH | No | Sometimes | Partial updates |
| DELETE | No | Yes | Remove resources |

## HTTP Status Codes

Status codes communicate the result of a request. Using the correct code helps clients understand what happened and how to respond.

### 2xx Success

**200 OK**
- Generic success response
- Used for GET, PUT, PATCH, DELETE

```json
{
  "id": 123,
  "name": "John Doe"
}
```

**201 Created**
- Resource successfully created
- Used for POST
- Include `Location` header with new resource URI

```http
HTTP/1.1 201 Created
Location: /api/users/123
```

**204 No Content**
- Successful request with no response body
- Used for DELETE or PUT/PATCH when no body is returned

```http
HTTP/1.1 204 No Content
```

### 3xx Redirection

**301 Moved Permanently**
- Resource has a new permanent URL
- Include new URL in `Location` header

**304 Not Modified**
- Resource hasn't changed since last request
- Used with conditional headers (If-Modified-Since, If-None-Match)

### 4xx Client Errors

**400 Bad Request**
- Malformed request syntax
- Invalid request parameters
- Missing required fields

```json
{
  "error": "Bad Request",
  "message": "Email is required",
  "field": "email"
}
```

**401 Unauthorized**
- Authentication required
- Client hasn't provided valid credentials

```json
{
  "error": "Unauthorized",
  "message": "Authentication required"
}
```

**403 Forbidden**
- Authenticated but not authorized
- Client lacks permission for this resource

```json
{
  "error": "Forbidden",
  "message": "You don't have permission to access this resource"
}
```

**404 Not Found**
- Resource doesn't exist
- URL path is incorrect

```json
{
  "error": "Not Found",
  "message": "User with ID 999 not found"
}
```

**405 Method Not Allowed**
- HTTP method not supported for this resource
- Include `Allow` header with permitted methods

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST
```

**409 Conflict**
- Request conflicts with current state
- Duplicate resource, constraint violation

```json
{
  "error": "Conflict",
  "message": "Email already exists"
}
```

**422 Unprocessable Entity**
- Well-formed request but semantic errors
- Validation failures

```json
{
  "error": "Unprocessable Entity",
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "age", "message": "Must be at least 18" }
  ]
}
```

**429 Too Many Requests**
- Rate limit exceeded
- Include `Retry-After` header

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

### 5xx Server Errors

**500 Internal Server Error**
- Generic server error
- Something unexpected went wrong

```json
{
  "error": "Internal Server Error",
  "message": "An unexpected error occurred"
}
```

**502 Bad Gateway**
- Upstream server error
- Invalid response from another service

**503 Service Unavailable**
- Server temporarily overloaded or under maintenance
- Include `Retry-After` header

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 300
```

**504 Gateway Timeout**
- Upstream server timeout
- Another service didn't respond in time

### Status Code Summary

| Range | Category | Meaning |
|-------|----------|---------|
| 2xx | Success | Request processed successfully |
| 3xx | Redirection | Further action needed |
| 4xx | Client Error | Client made a mistake |
| 5xx | Server Error | Server failed to fulfill valid request |

## Resource Naming Conventions

### Use Nouns, Not Verbs

```
❌ /getUsers
❌ /createUser
❌ /deleteUser/123

✅ GET    /users
✅ POST   /users
✅ DELETE /users/123
```

### Use Plural Nouns

```
❌ /user
❌ /user/123

✅ /users
✅ /users/123
```

### Use Lowercase and Hyphens

```
❌ /UserProfiles
❌ /user_profiles
❌ /userProfiles

✅ /user-profiles
```

### Represent Hierarchy

```
✅ GET /users/123/orders              # Orders for user 123
✅ GET /users/123/orders/456          # Order 456 for user 123
✅ POST /users/123/orders             # Create order for user 123
```

### Avoid Deep Nesting

```
❌ /users/123/orders/456/items/789/...

✅ GET /orders/456/items              # If order context is clear
✅ GET /users/123/orders              # Then use direct order access
✅ GET /order-items/789               # Or use sub-resources with IDs
```

### Use Query Parameters for Filtering

```
✅ GET /users?status=active&role=admin
✅ GET /orders?status=pending&created_after=2024-01-01
✅ GET /products?category=electronics&price_min=100&price_max=500
```

## Request and Response Formats

### Content Negotiation

Use the `Accept` header to request a specific format:

```http
GET /api/users/123 HTTP/1.1
Accept: application/json
```

```http
GET /api/users/123 HTTP/1.1
Accept: application/xml
```

### Content-Type Header

Always specify the content type for request bodies:

```http
POST /api/users HTTP/1.1
Content-Type: application/json

{
  "name": "John Doe"
}
```

### Consistent Response Structure

**Success response:**
```json
{
  "data": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

**List response with pagination:**
```json
{
  "data": [
    { "id": 1, "name": "User One" },
    { "id": 2, "name": "User Two" }
  ],
  "meta": {
    "total": 100,
    "page": 1,
    "perPage": 20,
    "totalPages": 5
  },
  "links": {
    "self": "/users?page=1",
    "next": "/users?page=2",
    "last": "/users?page=5"
  }
}
```

## Error Handling

### Consistent Error Format

Always return errors in a consistent format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request was invalid",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "age",
        "message": "Must be at least 18"
      }
    ],
    "requestId": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Use Error Codes

Error codes help clients handle errors programmatically:

| Code | Description |
|------|-------------|
| VALIDATION_ERROR | Invalid input data |
| AUTHENTICATION_ERROR | Invalid or missing credentials |
| AUTHORIZATION_ERROR | Insufficient permissions |
| RESOURCE_NOT_FOUND | Resource doesn't exist |
| RESOURCE_EXISTS | Resource already exists |
| RATE_LIMIT_EXCEEDED | Too many requests |
| INTERNAL_ERROR | Server error |

### Include Helpful Information

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 100,
      "remaining": 0,
      "resetAt": "2024-01-15T10:35:00Z"
    }
  }
}
```

## Pagination and Filtering

### Offset-Based Pagination

Simple but can have issues with concurrent updates:

```
GET /users?page=2&perPage=20
```

```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "perPage": 20,
    "total": 100,
    "totalPages": 5
  },
  "links": {
    "first": "/users?page=1",
    "prev": "/users?page=1",
    "self": "/users?page=2",
    "next": "/users?page=3",
    "last": "/users?page=5"
  }
}
```

### Cursor-Based Pagination

Better for real-time data and large datasets:

```
GET /users?cursor=eyJpZCI6MTAwfQ&limit=20
```

```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

### Filtering and Sorting

```
GET /products?category=electronics&sort=price&order=asc
GET /orders?status=shipped&created_after=2024-01-01&sort=created_at:desc
```

## Best Practices Summary

### DO:
- Use nouns for resource names
- Use plural nouns for collections
- Use HTTP methods semantically
- Return appropriate status codes
- Provide consistent error responses
- Use versioning (e.g., `/api/v1/users`)
- Document your API
- Support pagination for collections
- Include hypermedia links (HATEOAS)
- Use HTTPS everywhere
- Implement rate limiting
- Validate and sanitize input

### DON'T:
- Use verbs in URLs
- Use deep nesting (more than 2-3 levels)
- Return HTML errors for API endpoints
- Use 200 for errors
- Expose internal IDs or implementation details
- Use GET for state-changing operations
- Ignore security headers
- Make breaking changes without versioning

## Conclusion

Designing RESTful APIs is about creating intuitive, consistent interfaces that developers can easily understand and use. By following these principles:

- **Use HTTP methods correctly** to express intent
- **Return appropriate status codes** to communicate results
- **Name resources consistently** with nouns and hierarchy
- **Handle errors gracefully** with consistent formats
- **Support pagination and filtering** for large datasets
- **Document everything** so developers can be successful

Your APIs will be more maintainable, easier to consume, and enjoyable to work with. Remember: good API design is an investment that pays off in developer productivity and system reliability.