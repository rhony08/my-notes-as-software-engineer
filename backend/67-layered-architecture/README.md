# Layered Architecture Pattern

You've probably seen it a hundred times. A project with `controllers/`, `services/`, and `repositories/` folders. Maybe an `routes/` on top and a `models/` on the bottom. That's layered architecture—the most widely used pattern in backend development.

But just because it's common doesn't mean it's used well. Let's talk about what makes it work, where it breaks, and how to keep your layers from turning into a tangled mess.

## What Are We Even Talking About?

Layered architecture organizes code into horizontal stacks. Each layer has one job, and it only talks to the layer directly below it.

```
┌─────────────────┐
│  Routes        │  ← HTTP, auth, request parsing
├─────────────────┤
│  Controllers   │  ← Orchestration, response formatting
├─────────────────┤
│  Services      │  ← Business logic, validation, workflows
├─────────────────┤
│  Repositories  │  ← Data access, queries, DB interaction
├─────────────────┤
│  Database      │  ← Your actual data store
└─────────────────┘
```

The golden rule: **dependencies point inward**. Routes know about controllers. Controllers know about services. Services know about repositories. Never the other way around.

## Why This Pattern Won

Most backend frameworks push you toward this structure out of the box. Rails has MVC. Express projects naturally grow `routes/`, `controllers/`, `services/`. Spring Boot's annotations practically demand it.

The reasons it's so popular:

- **Familiarity** — every dev who's worked on a web backend recognizes this
- **Separation of concerns** — each layer has a clear reason to exist
- **Testability** — you can mock a layer and test the one above it
- **Onboarding** — new devs can find things quickly

## The Hard Part: Keeping Layers Clean

This is where most projects fall apart. Here's what I see all the time:

### ❌ Controllers Doing Business Logic

```javascript
// ❌ Controller knows too much
async function createUser(req, res) {
  const db = getDatabase();
  
  // This logic belongs in a service
  if (req.body.email && !isValidEmail(req.body.email)) {
    return res.status(400).json({ error: 'Invalid email' });
  }
  
  // And this belongs in a repository
  const existing = await db.query('SELECT * FROM users WHERE email = $1', [req.body.email]);
  if (existing.rows.length > 0) {
    return res.status(409).json({ error: 'User already exists' });
  }
  
  const result = await db.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id',
    [req.body.name, req.body.email]
  );
  
  // And formatting logic doesn't belong here either
  res.status(201).json({ 
    userId: result.rows[0].id,
    message: 'User created successfully'
  });
}
```

### ✅ Clean Separation

```javascript
// routes/userRoutes.js — Just routing
router.post('/users', validateSchema(createUserSchema), userController.create);

// controllers/userController.js — Orchestration
async function create(req, res) {
  const user = await userService.create(req.body);
  res.status(201).json(userSerializer.serialize(user));
}

// services/userService.js — Business logic
async function create(data) {
  const existing = await userRepo.findByEmail(data.email);
  // ↑ Let the repository handle the query, not us
  if (existing) {
    throw new ConflictError('User already exists');
  }
  const user = new User(data.name, data.email);
  return userRepo.save(user);
}

// repositories/userRepo.js — Data access
async function findByEmail(email) {
  const result = await db.query('SELECT * FROM users WHERE email = $1', [email]);
  return result.rows[0] ? UserMapper.toDomain(result.rows[0]) : null;
}
```

The difference? Each function stays in its lane. If I want to change how I validate emails, I touch the service. If I want to change the query, I touch the repo. If I want to change the API response format, I touch the controller.

## When Layers Start Leaking

Nobody draws dependency arrows wrong on purpose. It happens gradually.

**Abstraction leakage** is the classic trap. Your repository layer starts returning raw database rows, so your service layer has to know about column names. Your controller starts making decisions based on error details from the ORM.

The fix is usually a **domain model** that lives between services and repositories:

```javascript
// ❌ Leaky — controller knows about database errors
try {
  await userService.create(data);
} catch (err) {
  if (err.code === '23505') {  // PostgreSQL unique violation
    return res.status(409).json({ error: 'Duplicate' });
  }
}

// ✅ Clean — error is abstracted
try {
  await userService.create(data);
} catch (err) {
  if (err instanceof DuplicateResourceError) {
    return res.status(409).json({ error: 'Duplicate' });
  }
}
```

## Where This Pattern Struggles

Layered architecture is great for CRUD apps. But it starts creaking under certain conditions:

| Situation | What Happens |
|-----------|-------------|
| **Complex business logic** | Services get fat. You end up with 500-line service files. |
| **Cross-cutting concerns** | Logging, caching, metrics get sprinkled everywhere. |
| **Simple features** | You write 4 files for a basic CRUD endpoint. Feels like overkill. |
| **Microservices** | Each service is already small—do you really need 4 layers inside it? |

The "simple feature" problem is real. For a straightforward read endpoint, you end up routing through 4 layers just to fetch a row. Some teams collapse layers for simple operations:

```javascript
// Sometimes it's okay to skip layers for simple reads
router.get('/categories', async (req, res) => {
  const categories = await categoryRepo.findAll();
  res.json(categorySerializer.serialize(categories));
});
```

Is this "pure" layered architecture? No. Is it pragmatic? Yes. The layers are still there if you need them—you just don't force every request through every layer.

## The Trade-offs (Real Talk)

**What you gain:**
- Predictable code organization
- Easy to unit test (each layer in isolation)
- Low cognitive overhead for new team members
- Works well with dependency injection

**What you pay:**
- More files, more boilerplate
- Performance overhead from all the indirection (usually negligible, but it adds up)
- Can encourage anemic domain models (services become "doers", models become "bags of getters")
- Hard to enforce purely by convention—teams need discipline

## Practical Tips

Some things I've learned the hard way:

1. **Define layer boundaries explicitly in your README.** A short doc saying "controllers parse requests, services run logic, repos query data" saves hours of arguments in code review.

2. **Use custom exceptions per layer.** `NotFoundError` from a service is cleaner than `null` bubbling up from a repo.

3. **Don't be religious about it.** If a simple read endpoint needs 4 files, collapse it. The pattern serves you, not the other way around.

4. **Watch the service layer growth.** If a service file passes 300 lines, it's probably doing too much. Extract use-case objects or domain services.

5. **Test at the right level.** Test controllers with HTTP integration tests, test services with unit tests, test repos with integration tests against a real DB. Don't test everything at every layer.

## The Bottom Line

Layered architecture is the default for a reason. It's simple, it works, and most teams already know it. But it's not a silver bullet—knowing when to bend the rules is what separates a pattern that helps from one that gets in the way.

The goal isn't purity. It's **maintainability**. If your layers make the code easier to change, you're doing it right. If they're getting in the way, it's time to rethink.
