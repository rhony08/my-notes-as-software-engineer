# CI Fundamentals: Automated Testing

You know that feeling when you merge something on Friday afternoon, and by Monday morning your PM is asking why the staging environment is broken? Yeah. That's exactly what CI is designed to prevent.

Continuous Integration is simple in theory: every time you push code, the machine checks your work. No waiting for code review. No "it works on my machine." Just automated feedback, fast.

## The Problem CI Solves

Without CI, here's the typical timeline:

```
Day 1: Write code, test locally, it works ✓
Day 3: Push to shared branch, merge conflicts resolved ✓
Day 5: Deploy to staging → 🔥 everything breaks
Day 7: Find out it was a missing env variable. Again.
```

The gap between "it works on my machine" and "it breaks in production" is where CI lives. Automated testing closes that gap by catching issues before they reach anyone else.

## Types of Automated Tests (And What Each Catches)

You don't need everything on day one. Here's what matters:

### Unit Tests — The Safety Net

Fast, focused, isolated. These test a single function or method in complete isolation.

```python
# ❌ Testing database queries in a unit test
def test_create_user():
    user = User.create(name="John")  # Hits the DB
    assert user.id is not None

# ✅ Isolated unit test with mocked dependencies
def test_calculate_discount():
    # No DB, no network — pure logic
    result = calculate_discount(subtotal=100, coupon_code="SAVE10")
    assert result == 90
```

**What they catch:** Logic errors, edge cases in functions, wrong return values.

### Integration Tests — The Reality Check

These verify that your components actually talk to each other correctly.

```python
def test_user_repository():
    # Using a real test database, not mocks
    repo = UserRepository(database_url="sqlite:///:memory:")
    repo.save(User(name="Alice"))
    user = repo.find_by_name("Alice")
    assert user is not None
    assert user.name == "Alice"
```

**What they catch:** Schema mismatches, API contract violations, wrong query parameters.

### End-to-End Tests — The Insurance Policy

These simulate real user flows through your entire system. Slow but valuable for critical paths.

```python
def test_checkout_flow():
    # Browser automation testing
    page.goto("https://staging.example.com")
    page.click("#add-to-cart")
    page.fill("#email", "test@example.com")
    page.click("#checkout")
    assert page.text_content(".order-confirmation") == "Order placed!"
```

**What they catch:** Full workflow breaks, UI issues, auth problems, payment flow failures.

## The CI Pipeline: What Actually Runs

Here's a minimal but effective pipeline:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup
        run: npm ci  # Clean install — catches lockfile drift
      
      - name: Lint
        run: npm run lint  # Catch formatting/style issues fast
      
      - name: Unit Tests
        run: npm run test:unit  # Fast feedback loop
      
      - name: Integration Tests
        run: npm run test:integration  # DB/API contract checks
```

Three things worth noticing:

1. **`npm ci` not `npm install`** — it fails if `package-lock.json` is out of sync. That's a feature, not a bug.
2. **Lint runs first** — cheap check that catches formatting before you waste time on tests.
3. **Unit before integration** — failing fast means faster feedback.

### A Real-World Failure This Catches

I've seen this exact scenario play out more times than I'd like to admit:

```javascript
// PR #423 - "Refactor payment module"
// dev A changes the payment service response
function processPayment(amount) {
  return { status: "success", txId: "abc123" };  // New: txId added
}

// dev B (same sprint, different branch) consumes it
function handlePayment(res) {
  if (res.status === "success") {
    // Old code — expects res.transactionId
    redirectTo(`/confirmation/${res.transactionId}`);
  }
}
```

Merge both to main → integration test catches the mismatch → CI fails → no deployment → no debugging at 2am. That's the whole point.

## Trade-offs Nobody Talks About

### Speed vs. Coverage

Your first CI pipeline will probably take 3-5 minutes. Add enough tests, and you're looking at 30+ minutes. Fast feedback is useless if it isn't fast.

**What to do:** Layer your tests. Run unit tests on every push, integration tests on PRs, and E2E tests on merges to main.

```yaml
# Different triggers for different speeds
on:
  push:          # Unit tests only
    paths: ['src/**']
  pull_request:  # Full suite
  push:
    branches: [main]  # Add E2E on merge
```

### Flaky Tests

Tests that pass one run and fail the next are worse than no tests at all. They train your team to ignore CI failures.

**What to do:** When you spot a flaky test, fix or delete it immediately. Don't let flaky tests pile up.

### CI Doesn't Replace Code Review

We've all been there: "Tests pass ✅, ship it!" — and the code is terrible. CI catches regressions. Code review catches design problems. They're complementary, not alternatives.

## Actionable Takeaways

- **Start with unit tests** for your core business logic. They're fast and catch most bugs.
- **Add one integration test per API endpoint** — just the happy path. You'll catch schema mismatches immediately.
- **Ship the CI config alongside your first PR.** Don't "add CI later" — you never will.
- **Keep the pipeline under 10 minutes.** If tests take longer, split them or parallelize.
- **Every red CI means someone fixes it before lunch.** Broken builds that stay broken erode trust in the process.
