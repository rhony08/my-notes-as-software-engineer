# Request Validation: Don't Trust User Input

Your API just accepted a negative age, an email that's 500 characters long, and a "role" field set to "admin" from a new user registration. Congratulations—you've just opened the door to data corruption, potential privilege escalation, and a database full of garbage. Request validation isn't optional. It's your first line of defense.

## What Happens Without Validation

Every field in your API is an attack surface. Let's look at what goes wrong:

```javascript
// ❌ No validation - trusting user input
app.post('/api/users', async (req, res) => {
  await db.query('INSERT INTO users SET ?', req.body);
  res.json({ success: true });
});
```

This code accepts **anything**:
- `age: -500` → Negative ages in your database
- `role: "admin"` → User just made themselves an admin
- `email: "<script>alert('xss')</script>"` → Stored XSS attack
- `id: { "$ne": "" }` → NoSQL injection (MongoDB)

The fix isn't complex. You just need to validate before processing.

## Validation Fundamentals

### Allowlist, Don't Blocklist

Blocklists try to catch bad input. Allowlists only accept known-good input.

```javascript
// ❌ Blocklist - fragile, easy to bypass
function sanitizeUsername(input) {
  const badChars = ['<', '>', '"', "'", '&', ';', '|'];
  let result = input;
  badChars.forEach(char => {
    result = result.replaceAll(char, '');
  });
  return result;
}
// Problem: What about null bytes? Unicode tricks? New encodings?

// ✅ Allowlist - robust by default
function validateUsername(input) {
  if (!/^[a-zA-Z0-9_]{3,20}$/.test(input)) {
    throw new Error('Username must be 3-20 characters: letters, numbers, underscore only');
  }
  return input;
}
```

Allowlists are safer because they **fail closed**—anything unexpected gets rejected.

### Validate at the Boundary

Check input at the edge of your application, before it reaches your business logic:

```
Request → [Validation Middleware] → [Business Logic] → [Database]
                ↑ Check here
```

This keeps your internal code clean and ensures consistent validation across all entry points.

```javascript
// ✅ Validation middleware
const validateUser = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, { abortEarly: false });
  if (error) {
    return res.status(400).json({
      error: 'Validation failed',
      details: error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message
      }))
    });
  }
  req.body = value; // Use sanitized value
  next();
};

app.post('/api/users', validateUser(userSchema), createUser);
```

## What to Validate

### Request Body

The most common source of user input:

```javascript
const Joi = require('joi');

const userSchema = Joi.object({
  email: Joi.string().email().required().max(255),
  password: Joi.string().min(8).max(100)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .message('Password needs uppercase, lowercase, and number'),
  age: Joi.number().integer().min(0).max(150).optional(),
  role: Joi.string().valid('user', 'moderator').default('user')
});
```

### URL Parameters

Never trust path or query params:

```javascript
// ❌ Direct use - SQL injection risk
app.get('/api/users/:id', (req, res) => {
  db.query(`SELECT * FROM users WHERE id = ${req.params.id}`);
});

// ✅ Validated
const idSchema = Joi.string().pattern(/^[a-f0-9]{24}$/);

app.get('/api/users/:id', (req, res) => {
  const { error, value } = idSchema.validate(req.params.id);
  if (error) return res.status(400).json({ error: 'Invalid user ID' });
  
  // Use parameterized query with validated value
  db.query('SELECT * FROM users WHERE id = ?', [value]);
});
```

### Query Parameters

Pagination, filtering, sorting—all need validation:

```javascript
const listSchema = Joi.object({
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(20),
  sort: Joi.string().valid('createdAt', 'name', 'email').default('createdAt'),
  order: Joi.string().valid('asc', 'desc').default('desc'),
  search: Joi.string().max(100).optional()
});

app.get('/api/users', (req, res) => {
  const { error, value } = listSchema.validate(req.query);
  if (error) return res.status(400).json({ error: error.details });
  // Use validated value
});
```

### HTTP Headers

Headers can be spoofed. Validate anything you use:

```javascript
// ❌ Trusting header directly
const userId = req.headers['x-user-id'];

// ✅ Validated
const userId = Joi.string().uuid().validate(req.headers['x-user-id']).value;
```

### File Uploads

Files are a huge attack vector:

```javascript
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

function validateFile(file) {
  if (!file) throw new Error('No file provided');
  if (!ALLOWED_TYPES.includes(file.mimetype)) {
    throw new Error('Only JPEG, PNG, and WebP allowed');
  }
  if (file.size > MAX_SIZE) {
    throw new Error('File too large (max 5MB)');
  }
  
  // Verify actual content, not just extension
  const magic = file.buffer.slice(0, 4);
  const validSignatures = {
    'ffd8ffe': 'jpeg',
    '89504e47': 'png',
    '52494646': 'webp'
  };
  
  const hex = magic.toString('hex');
  if (!Object.keys(validSignatures).some(sig => hex.startsWith(sig))) {
    throw new Error('File content doesn\'t match extension');
  }
  
  return file;
}
```

## Using Validation Libraries

Don't roll your own validation. Use battle-tested libraries.

### Joi (Node.js)

```javascript
const Joi = require('joi');

const createUserSchema = Joi.object({
  email: Joi.string()
    .email()
    .required()
    .max(255)
    .messages({
      'string.email': 'Please enter a valid email address',
      'string.max': 'Email must be less than 255 characters'
    }),
    
  password: Joi.string()
    .min(8)
    .max(100)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .required()
    .messages({
      'string.pattern.base': 'Password must contain uppercase, lowercase, and number'
    }),
    
  role: Joi.string()
    .valid('user', 'moderator', 'admin')
    .default('user'),
    
  preferences: Joi.object({
    newsletter: Joi.boolean().default(false),
    notifications: Joi.boolean().default(true)
  }).default()
});

// Validate with options
const { error, value } = createUserSchema.validate(req.body, {
  abortEarly: false,  // Get all errors, not just first
  stripUnknown: true, // Remove unknown fields
  convert: true       // Type coercion (string "25" → number 25)
});
```

### Zod (TypeScript)

```typescript
import { z } from 'zod';

const userSchema = z.object({
  email: z.string()
    .email()
    .max(255),
    
  password: z.string()
    .min(8)
    .max(100)
    .regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, 
      'Password must contain uppercase, lowercase, and number'),
    
  age: z.number()
    .int()
    .min(0)
    .max(150)
    .optional(),
    
  role: z.enum(['user', 'moderator', 'admin'])
    .default('user')
});

type User = z.infer<typeof userSchema>; // Auto-generated type

// Safe parse - doesn't throw
const result = userSchema.safeParse(req.body);
if (!result.success) {
  return res.status(400).json({
    error: 'Validation failed',
    issues: result.error.issues
  });
}
const validatedUser = result.data;
```

### Pydantic (Python/FastAPI)

```python
from pydantic import BaseModel, EmailStr, Field, validator
from typing import Optional
import re

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=100)
    age: Optional[int] = Field(None, ge=0, le=150)
    role: str = 'user'
    
    @validator('password')
    def validate_password(cls, v):
        if not re.search(r'[A-Z]', v):
            raise ValueError('Must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Must contain a number')
        return v
    
    @validator('role')
    def validate_role(cls, v):
        if v not in ['user', 'moderator', 'admin']:
            raise ValueError('Invalid role')
        return v

# FastAPI auto-validates
@app.post('/api/users')
async def create_user(user: UserCreate):
    # user is already validated here
    return await save_user(user)
```

## Common Validation Patterns

### Strings

```javascript
// Length constraints
name: Joi.string().min(2).max(100).trim().required()

// Email
email: Joi.string().email().required()

// URL
website: Joi.string().uri().optional()

// Pattern matching
slug: Joi.string().pattern(/^[a-z0-9-]+$/)

// Enum
status: Joi.string().valid('draft', 'published', 'archived')
```

### Numbers

```javascript
// Range
age: Joi.number().integer().min(0).max(150)

// Positive only
price: Joi.number().positive().precision(2)

// UUID count
count: Joi.number().integer().min(1).max(100)
```

### Dates

```javascript
// ISO date
birthDate: Joi.date().iso()

// Range
eventDate: Joi.date().min('now').max('2099-12-31')

// Age validation
birthDate: Joi.date().max('now').custom((value) => {
  const age = Math.floor((Date.now() - value) / (365.25 * 24 * 60 * 60 * 1000));
  if (age < 13) throw new Error('Must be at least 13 years old');
  return value;
})
```

### Arrays

```javascript
// List of strings
tags: Joi.array().items(Joi.string().max(30)).max(10)

// List of IDs
userIds: Joi.array().items(Joi.string().pattern(/^[a-f0-9]{24}$/)).min(1)

// Unique values
categories: Joi.array().items(Joi.string()).unique()
```

### Nested Objects

```javascript
address: Joi.object({
  street: Joi.string().required(),
  city: Joi.string().required(),
  country: Joi.string().required(),
  postalCode: Joi.string().pattern(/^[A-Z0-9-]{3,10}$/).required()
}).required()
```

## Security-Specific Validations

### SQL Injection Prevention

```javascript
// ❌ NEVER concatenate user input into SQL
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// ✅ Use parameterized queries
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [validatedId]);

// ✅ Or use an ORM
const user = await User.findById(validatedId);
```

### NoSQL Injection Prevention

```javascript
// ❌ Vulnerable - attacker sends { "$ne": "" } as password
db.users.findOne({ username, password });

// ✅ Validate types explicitly
if (typeof username !== 'string' || typeof password !== 'string') {
  return res.status(400).json({ error: 'Invalid input' });
}
```

### XSS Prevention

```javascript
import sanitizeHtml from 'sanitize-html';

function validateHtml(content) {
  return sanitizeHtml(content, {
    allowedTags: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li'],
    allowedAttributes: { 'a': ['href'] }
  });
}

// In schema
bio: Joi.string().custom(validateHtml)
```

### Path Traversal Prevention

```javascript
// ❌ User can request ../../../etc/passwd
app.get('/api/files/:filename', (req, res) => {
  res.sendFile(`/uploads/${req.params.filename}`);
});

// ✅ Validate and sanitize path
const filename = req.params.filename.replace(/\.\./g, '');
if (!/^[a-zA-Z0-9_.-]+$/.test(filename)) {
  return res.status(400).json({ error: 'Invalid filename' });
}
res.sendFile(path.join('/uploads', filename));
```

## Error Handling

### Clear, Actionable Messages

```javascript
// ❌ Vague
return res.status(400).json({ error: 'Invalid input' });

// ✅ Specific and helpful
return res.status(400).json({
  error: 'Validation failed',
  details: [
    {
      field: 'email',
      message: 'Please enter a valid email address',
      example: 'user@example.com'
    },
    {
      field: 'password',
      message: 'Password must be at least 8 characters with uppercase, lowercase, and number',
      hint: 'Try combining words with numbers and symbols'
    }
  ]
});
```

### Don't Leak Internal Structure

```javascript
// ❌ Reveals database structure
return res.status(404).json({
  error: `User with id ${userId} does not exist in table users`
});

// ✅ Generic message
return res.status(404).json({
  error: 'Resource not found'
});
```

### Log Suspicious Activity

```javascript
app.use((err, req, res, next) => {
  if (err.name === 'ValidationError') {
    // Check for potential attacks
    const suspicious = err.details.some(d => 
      /<script|javascript:|onerror|union select/i.test(d.value)
    );
    
    if (suspicious) {
      logger.warn('Suspicious input detected', {
        ip: req.ip,
        path: req.path,
        errors: err.details
      });
    }
    
    return res.status(400).json({ error: 'Validation failed' });
  }
  next(err);
});
```

## Best Practices

### 1. Validate Early, at the Edge

Use middleware to validate before your handlers:

```javascript
const validate = (schema) => (req, res, next) => {
  const { error, value } = schema.validate(req.body, { abortEarly: false });
  if (error) return res.status(400).json({ errors: error.details });
  req.body = value;
  next();
};

app.post('/api/users', validate(userSchema), createUser);
```

### 2. Be Strict by Default

```javascript
const schema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required()
}).options({ 
  stripUnknown: true,  // Remove unknown fields
  presence: 'required' // All fields required by default
});
```

### 3. Sanitize After Validation

```javascript
const { error, value } = schema.validate(input);
if (error) return res.status(400).json({ error });

const sanitized = {
  ...value,
  email: value.email.toLowerCase().trim(),
  name: value.name.trim(),
  bio: sanitizeHtml(value.bio)
};
```

### 4. Create Reusable Schemas

```javascript
const schemas = {
  id: Joi.string().pattern(/^[a-f0-9]{24}$/),
  email: Joi.string().email().max(255),
  password: Joi.string().min(8).max(100)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  pagination: Joi.object({
    page: Joi.number().integer().min(1).default(1),
    limit: Joi.number().integer().min(1).max(100).default(20)
  })
};

// Reuse across routes
const getUserSchema = Joi.object({ id: schemas.id.required() });
const listSchema = schemas.pagination.keys({ search: Joi.string().max(100) });
```

## Takeaways

1. **Never trust user input**—every field is an attack surface
2. **Use allowlists, not blocklists**—accept only known-good patterns
3. **Validate at the boundary**—middleware before handlers
4. **Use established libraries**—Joi, Zod, Pydantic—don't roll your own
5. **Provide clear errors**—help users fix their input
6. **Log suspicious patterns**—catch attacks early
7. **Sanitize after validation**—clean data before storage

Request validation isn't just about preventing errors. It's about protecting your users, your data, and your reputation. The few minutes you spend implementing thorough validation will save you hours of debugging security incidents and data corruption down the line.