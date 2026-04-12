# Request Validation: Don't Trust User Input

Every piece of data that enters your backend from external sources—form submissions, API calls, URL parameters, file uploads—is a potential attack vector. The single most important security practice in backend development is simple: never trust user input. In this article, we'll explore how to implement robust request validation that protects your application from malicious data, injection attacks, and unexpected errors.

## Table of Contents

- [Why Validation Matters](#why-validation-matters)
- [The OWASP Top 10 Connection](#the-owasp-top-10-connection)
- [Types of Input to Validate](#types-of-input-to-validate)
- [Validation Strategies](#validation-strategies)
- [Common Validation Patterns](#common-validation-patterns)
- [Validation Libraries](#validation-libraries)
- [Error Handling and User Feedback](#error-handling-and-user-feedback)
- [Security-Specific Validations](#security-specific-validations)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)

## Why Validation Matters

### The Cost of Missing Validation

Without proper validation, your application is vulnerable to:

1. **SQL Injection** - Malicious SQL code in input fields
2. **Cross-Site Scripting (XSS)** - JavaScript code injection
3. **Command Injection** - System command execution
4. **Buffer Overflows** - Memory corruption attacks
5. **Logic Errors** - Unexpected application behavior
6. **Data Corruption** - Invalid data stored in databases

### A Simple Example

```javascript
// ❌ Dangerous - no validation
app.post('/api/users', (req, res) => {
  const { email, age, role } = req.body;
  
  // What if email is not an email?
  // What if age is -100 or "admin"?
  // What if role is "administrator" instead of "user"?
  
  db.query('INSERT INTO users SET ?', { email, age, role });
  res.json({ success: true });
});
```

```javascript
// ✅ Safe - with validation
app.post('/api/users', (req, res) => {
  const { error, value } = validateUser(req.body);
  
  if (error) {
    return res.status(400).json({ error: error.details });
  }
  
  db.query('INSERT INTO users SET ?', value);
  res.json({ success: true });
});
```

## The OWASP Top 10 Connection

Request validation directly addresses several OWASP Top 10 security risks:

| OWASP Risk | How Validation Helps |
|------------|---------------------|
| A03: Injection | Prevents SQL, NoSQL, OS command injection |
| A04: Insecure Design | Enforces business rules at input boundaries |
| A05: Security Misconfiguration | Reduces attack surface |
| A08: Software and Data Integrity | Validates data before processing |

## Types of Input to Validate

### 1. Request Body

The most common source of user input:

```javascript
// JSON body
{
  "email": "user@example.com",
  "password": "secretpass123",
  "age": 25,
  "preferences": { "newsletter": true }
}
```

### 2. URL Parameters

```javascript
// Path parameters
app.get('/api/users/:id', (req, res) => {
  const id = req.params.id; // Must validate!
});

// Query parameters
app.get('/api/products', (req, res) => {
  const { page, limit, sort } = req.query; // Must validate!
});
```

### 3. HTTP Headers

```javascript
app.post('/api/upload', (req, res) => {
  const contentType = req.headers['content-type'];
  const contentLength = parseInt(req.headers['content-length'], 10);
  const authorization = req.headers['authorization'];
  // All must be validated!
});
```

### 4. File Uploads

```javascript
app.post('/api/avatar', upload.single('avatar'), (req, res) => {
  const file = req.file;
  // Validate: file type, size, dimensions, content
});
```

### 5. Cookies

```javascript
app.use((req, res, next) => {
  const sessionId = req.cookies.sessionId;
  // Validate before using!
});
```

## Validation Strategies

### Allowlist vs Blocklist

**Blocklist (Less Secure):** Block known bad patterns

```javascript
// ❌ Blocklist approach - fragile
const blockedChars = ['<', '>', '"', "'", '&'];
function sanitize(input) {
  let result = input;
  blockedChars.forEach(char => {
    result = result.replace(char, '');
  });
  return result;
}
// Problem: Attackers find ways around blocklists
```

**Allowlist (More Secure):** Only accept known good patterns

```javascript
// ✅ Allowlist approach - robust
function validateUsername(input) {
  // Only allow alphanumeric and underscores
  if (!/^[a-zA-Z0-9_]{3,20}$/.test(input)) {
    throw new Error('Invalid username format');
  }
  return input;
}
```

### Validation Layers

Apply validation at multiple layers:

```
Client → API Gateway → Application → Database
   │         │              │             │
   └─────────┴──────────────┴─────────────┘
              Defense in Depth
```

1. **Client-side:** UX feedback (never trust alone!)
2. **API Gateway:** Basic sanitization, rate limiting
3. **Application:** Business logic validation
4. **Database:** Data integrity constraints

## Common Validation Patterns

### String Validation

```javascript
// Length constraints
const isValidName = (name) => 
  typeof name === 'string' && 
  name.length >= 2 && 
  name.length <= 100;

// Pattern matching
const isValidEmail = (email) =>
  /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);

// Alphanumeric only
const isValidCode = (code) =>
  /^[A-Z0-9]{6}$/.test(code);
```

### Number Validation

```javascript
// Range validation
const isValidAge = (age) => {
  const num = Number(age);
  return Number.isInteger(num) && num >= 0 && num <= 150;
};

// Positive number
const isValidPrice = (price) => {
  const num = parseFloat(price);
  return !isNaN(num) && num > 0;
};

// Enum values
const isValidStatus = (status) =>
  ['pending', 'active', 'completed', 'cancelled'].includes(status);
```

### Date Validation

```javascript
const isValidDate = (dateString) => {
  const date = new Date(dateString);
  return date instanceof Date && !isNaN(date);
};

const isFutureDate = (dateString) => {
  const date = new Date(dateString);
  return isValidDate(dateString) && date > new Date();
};

const isAgeValid = (birthDate) => {
  const birth = new Date(birthDate);
  const minAge = new Date();
  minAge.setFullYear(minAge.getFullYear() - 13);
  return birth <= minAge;
};
```

### Array Validation

```javascript
const isValidTags = (tags) => {
  return Array.isArray(tags) && 
    tags.length <= 10 &&
    tags.every(tag => 
      typeof tag === 'string' && 
      tag.length <= 30
    );
};

const isValidProductIds = (ids) => {
  return Array.isArray(ids) &&
    ids.length > 0 &&
    ids.every(id => /^[a-f0-9]{24}$/.test(id));
};
```

### Object Validation

```javascript
const isValidAddress = (address) => {
  if (typeof address !== 'object' || address === null) {
    return false;
  }
  
  const required = ['street', 'city', 'country', 'postalCode'];
  const hasAll = required.every(field => 
    typeof address[field] === 'string' && 
    address[field].length > 0
  );
  
  return hasAll;
};
```

## Validation Libraries

### Joi (Node.js)

```javascript
const Joi = require('joi');

const userSchema = Joi.object({
  email: Joi.string()
    .email()
    .required()
    .max(255),
  
  password: Joi.string()
    .min(8)
    .max(100)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .required()
    .messages({
      'string.pattern.base': 'Password must contain uppercase, lowercase, and number'
    }),
  
  age: Joi.number()
    .integer()
    .min(0)
    .max(150),
  
  role: Joi.string()
    .valid('user', 'moderator', 'admin')
    .default('user'),
  
  preferences: Joi.object({
    newsletter: Joi.boolean().default(false),
    notifications: Joi.boolean().default(true)
  }).default()
});

// Usage
const { error, value } = userSchema.validate(req.body, {
  abortEarly: false, // Return all errors
  stripUnknown: true  // Remove unknown fields
});

if (error) {
  return res.status(400).json({ 
    errors: error.details.map(d => ({
      field: d.path.join('.'),
      message: d.message
    }))
  });
}
```

### Zod (TypeScript/JavaScript)

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
    .default('user'),
  
  preferences: z.object({
    newsletter: z.boolean().default(false),
    notifications: z.boolean().default(true)
  }).default({})
});

type User = z.infer<typeof userSchema>;

// Usage
const result = userSchema.safeParse(req.body);

if (!result.success) {
  return res.status(400).json({ 
    errors: result.error.issues.map(issue => ({
      field: issue.path.join('.'),
      message: issue.message
    }))
  });
}

const validatedUser = result.data;
```

### Pydantic (Python)

```python
from pydantic import BaseModel, EmailStr, validator, Field
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

# Usage in FastAPI
from fastapi import HTTPException

@app.post('/api/users')
async def create_user(user: UserCreate):
    # Validation happens automatically
    return await create_user_in_db(user)
```

## Error Handling and User Feedback

### Clear Error Messages

```javascript
// ❌ Vague error
return res.status(400).json({ error: 'Invalid input' });

// ✅ Specific, actionable error
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

### Internationalization Support

```javascript
const errorMessages = {
  en: {
    'email.invalid': 'Please enter a valid email address',
    'password.min': 'Password must be at least {min} characters',
    'password.pattern': 'Password must contain uppercase, lowercase, and number'
  },
  de: {
    'email.invalid': 'Bitte geben Sie eine gültige E-Mail-Adresse ein',
    'password.min': 'Passwort muss mindestens {min} Zeichen haben',
    'password.pattern': 'Passwort muss Großbuchstaben, Kleinbuchstaben und Zahl enthalten'
  }
};

function getErrorMessage(code, language = 'en', params = {}) {
  let message = errorMessages[language][code] || errorMessages['en'][code];
  Object.entries(params).forEach(([key, value]) => {
    message = message.replace(`{${key}}`, value);
  });
  return message;
}
```

### Grouped Errors

```javascript
// Group errors by field for cleaner display
function groupErrors(errors) {
  return errors.reduce((acc, error) => {
    const field = error.field;
    if (!acc[field]) {
      acc[field] = [];
    }
    acc[field].push(error.message);
    return acc;
  }, {});
}

// Result:
{
  "email": ["Please enter a valid email address"],
  "password": [
    "Password must be at least 8 characters",
    "Password must contain uppercase letter"
  ]
}
```

## Security-Specific Validations

### SQL Injection Prevention

```javascript
// ❌ NEVER do this
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// ✅ Use parameterized queries
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [userId]);

// ✅ Or use an ORM with automatic escaping
const user = await User.findById(userId);
```

### XSS Prevention

```javascript
// Sanitize HTML input
const sanitizeHtml = require('sanitize-html');

function validateHtmlContent(content) {
  const clean = sanitizeHtml(content, {
    allowedTags: ['p', 'b', 'i', 'em', 'strong', 'a'],
    allowedAttributes: {
      'a': ['href']
    }
  });
  return clean;
}
```

### File Upload Validation

```javascript
const ALLOWED_MIME_TYPES = ['image/jpeg', 'image/png', 'image/gif'];
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

function validateFileUpload(file) {
  // Check file exists
  if (!file) {
    throw new Error('No file provided');
  }
  
  // Check MIME type
  if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) {
    throw new Error('Invalid file type. Only JPEG, PNG, and GIF allowed.');
  }
  
  // Check file size
  if (file.size > MAX_FILE_SIZE) {
    throw new Error('File too large. Maximum size is 5MB.');
  }
  
  // Verify actual content (not just extension)
  const buffer = fs.readFileSync(file.path);
  const magicNumbers = buffer.slice(0, 4).toString('hex');
  
  const validSignatures = {
    'ffd8ffe0': 'jpeg',
    'ffd8ffe1': 'jpeg',
    '89504e47': 'png',
    '47494638': 'gif'
  };
  
  const isValid = Object.keys(validSignatures).some(sig => 
    magicNumbers.startsWith(sig)
  );
  
  if (!isValid) {
    throw new Error('File content does not match extension.');
  }
  
  return true;
}
```

### NoSQL Injection Prevention

```javascript
// ❌ Vulnerable to NoSQL injection
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  // Attacker sends: { "username": "admin", "password": { "$ne": "" } }
  db.users.findOne({ username, password });
});

// ✅ Validate types explicitly
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  
  if (typeof username !== 'string' || typeof password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
  }
  
  db.users.findOne({ username, password });
});
```

## Best Practices

### 1. Validate Early, Validate Often

```javascript
// Validate at the entry point (middleware)
const validateUser = (req, res, next) => {
  const { error } = userSchema.validate(req.body);
  if (error) {
    return res.status(400).json({ error: error.details });
  }
  next();
};

app.post('/api/users', validateUser, createUser);
app.put('/api/users/:id', validateUser, updateUser);
```

### 2. Use Type Coercion Carefully

```javascript
// Be explicit about type conversion
const schema = Joi.object({
  age: Joi.number().integer().min(0),
  active: Joi.boolean(),
  tags: Joi.array().items(Joi.string())
});

// String "25" becomes number 25
// String "true" becomes boolean true
// But invalid strings become errors
```

### 3. Sanitize After Validation

```javascript
// Validate first
const { error, value } = schema.validate(input);
if (error) return res.status(400).json({ error });

// Then sanitize for storage
const sanitized = {
  ...value,
  name: value.name.trim(),
  email: value.email.toLowerCase(),
  bio: sanitizeHtml(value.bio)
};
```

### 4. Don't Reveal Internal Structure

```javascript
// ❌ Reveals internal structure
return res.status(400).json({
  error: `User with id ${userId} does not exist in table users`
});

// ✅ Generic error message
return res.status(404).json({
  error: 'Resource not found'
});
```

### 5. Log Validation Failures

```javascript
app.use((err, req, res, next) => {
  if (err.name === 'ValidationError') {
    logger.warn('Validation failed', {
      path: req.path,
      method: req.method,
      errors: err.details,
      ip: req.ip,
      userAgent: req.headers['user-agent']
    });
    
    return res.status(400).json({ error: 'Validation failed' });
  }
  next(err);
});
```

### 6. Create Reusable Validation Schemas

```javascript
// Common schemas
const schemas = {
  id: Joi.string().pattern(/^[a-f0-9]{24}$/),
  email: Joi.string().email().max(255),
  password: Joi.string().min(8).max(100)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  pagination: Joi.object({
    page: Joi.number().integer().min(1).default(1),
    limit: Joi.number().integer().min(1).max(100).default(20),
    sort: Joi.string().valid('createdAt', 'updatedAt', 'name').default('createdAt'),
    order: Joi.string().valid('asc', 'desc').default('desc')
  })
};

// Reuse across routes
const getUserSchema = Joi.object({
  id: schemas.id.required()
});

const listUsersSchema = Joi.object({
  ...schemas.pagination,
  search: Joi.string().max(100)
});
```

### 7. Handle Edge Cases

```javascript
// Empty strings
const schema = Joi.object({
  name: Joi.string().trim().min(1).required(),
  bio: Joi.string().allow('').max(500)
});

// Null values
const schema = Joi.object({
  middleName: Joi.string().allow(null),
  nickname: Joi.string().optional()
});

// Unexpected types
const schema = Joi.object({
  age: Joi.number().integer().min(0).strict() // Rejects string "25"
});
```

## Conclusion

Request validation is your first line of defense against security vulnerabilities and data corruption. By following these practices:

- **Never trust user input** — validate everything from every source
- **Use allowlists over blocklists** — accept only known good patterns
- **Validate at multiple layers** — defense in depth
- **Use established libraries** — Joi, Zod, Pydantic — don't roll your own
- **Provide clear error messages** — help users fix their input
- **Log validation failures** — detect attack patterns
- **Sanitize after validation** — clean data before storage

Remember: every field in your API is an attack surface. Treat it accordingly. Proper validation isn't just about preventing errors — it's about protecting your users, your data, and your application from malicious actors.

The few minutes you spend implementing thorough validation will save you hours of debugging, security incident response, and data recovery down the line.