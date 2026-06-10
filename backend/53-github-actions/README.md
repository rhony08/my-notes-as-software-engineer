# GitHub Actions Basics

Remember the last time you forgot to run `npm test` before pushing, and your teammate caught it during code review? Or worse — it hit production and a customer found it first? That's the whole reason CI exists. But setting up CI shouldn't feel like a project of its own.

GitHub Actions makes this trivial. You drop a YAML file in `.github/workflows/`, and GitHub handles the rest. No Jenkins server to maintain, no GitLab runners to configure, no CircleCI context to learn. Just a file.

## The Anatomy of a Workflow

A workflow is just a YAML file in `.github/workflows/` with a few sections:

```yaml
name: CI  # Shows up in the GitHub UI — make it descriptive

on:  # When does this run?
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4  # Built-in action — pulls your code
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm  # Auto-caches node_modules
      
      - run: npm ci
      - run: npm test
```

That's it. Push this file, and every PR to `main` gets tested automatically.

## What Each Piece Does

### Triggers (`on`)

This controls when your workflow fires. Here are the common patterns:

```yaml
# Run on every push to any branch
on: push

# Run only on specific branches and events
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Every Monday at 6AM UTC

# ❌ DON'T trigger on everything — you'll burn through minutes
on: [push, pull_request, workflow_dispatch]
```

**The trap:** `on: push` fires for every branch, every commit. If you push 10 commits, that's 10 workflow runs. Use branch filtering.

### Jobs — Compose Your Pipeline

Each job runs in a fresh VM. Jobs run in parallel by default — you need `needs` to create dependencies:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint  # Only runs if lint passes
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: [lint, test]  # Waits for both
    if: github.ref == 'refs/heads/main'  # Only deploy from main
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - run: echo "Deploying..."
```

**Why `needs` matters:** Without it, `deploy` runs the same time as `lint` and `test`. You'd deploy untested code — defeating the point of CI.

### Steps and Actions

Steps are either `run` (shell commands) or `uses` (reusable actions from the marketplace):

```yaml
steps:
  # Built-in action — check out your repo
  - uses: actions/checkout@v4
  
  # Community action — deploy to AWS
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/deploy-role
  
  # Raw shell — sometimes that's all you need
  - name: Deploy to S3
    run: aws s3 sync ./build s3://my-app-bucket
```

### Environment Variables and Secrets

Never hardcode credentials in workflows:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:  # Available to all steps
      NODE_ENV: production
    
    steps:
      - run: echo "Deploying to ${{ env.NODE_ENV }}"
      
      - name: Deploy
        env:  # Step-specific
          API_KEY: ${{ secrets.PROD_API_KEY }}
        run: npm run deploy
```

Secrets are stored in the repo under **Settings → Secrets and variables → Actions**. They're masked in logs automatically. No `echo $API_KEY` won't expose it in the logs — GitHub detects and masks it.

## A Real Workflow That Actually Helps

Here's the one I use on almost every project:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # ❗ Cancel stale runs

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check  # TypeScript checking
      - run: npm test -- --run  # Vitest/Jest — run once
      - run: npm run build
```

Three things worth calling out:

1. **`concurrency` with cancel-in-progress** — if you push again while a run is still going, it cancels the old one. Saves minutes and prevents confusion.
2. **`cache: npm`** — setup-node automatically caches `node_modules`. Your second and third runs are dramatically faster.
3. **`npm ci` vs `npm install`** — `ci` fails if `package-lock.json` is out of sync. That's intentional — it catches people who forgot to commit their lockfile changes.

## What Actions Can't Do (And That's Okay)

Actions is great for CI. But there are boundaries:

| Area | Actions | Better Tool |
|------|---------|-------------|
| Long-running builds (30+ min) | Times out at 6h, but minutes are expensive | Self-hosted runner |
| Fine-grained permissions per step | Limited — it's job-level by default | Custom scripts |
| Complex pipeline DAGs | Gets messy with many `needs` | Jenkins, Buildkite |
| Deployments to corporate infrastructure | Possible, but needs self-hosted runner | Dedicated CD tool |

**The self-hosted runner gap:** If your build needs to access an internal database or VPN-only resources, you need a self-hosted runner. GitHub hosted runners can't reach your VPC. Don't open security groups just for CI — run a self-hosted runner instead.

## Caching — The Speed Multiplier

Nothing kills developer happiness faster than waiting 10 minutes for CI. Caching is how you stay under 2 minutes:

```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

For most Node.js setups, `setup-node` with `cache: npm` handles this. For Python, Docker layers, or custom folders, use the `cache` action directly.

## Actionable Takeaways

- **Start with one YAML file** in `.github/workflows/ci.yml` — lint + test + build. That's 90% of the value.
- **Use `concurrency.cancel-in-progress`** on every workflow. Saves minutes and frustration on rapid commit pushes.
- **Store secrets in GitHub Settings**, not in workflow files. Even if the repo is private, secrets should never be in YAML or code.
- **Add caching from day one.** Without it, every run downloads the full internet. With it, runs are seconds.
- **Pin action versions to SHAs in production.** `@v4` is a moving tag — someone could push a new v4 release that breaks your workflow. Use `@abc123def`
