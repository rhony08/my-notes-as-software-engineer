# Environment Parity: Dev vs Prod

"It worked on my machine."

You've heard it. You've probably said it. And every time, somewhere in the deployment pipeline, something is quietly different between dev and production.

**Environment parity** is the practice of keeping your environments as identical as possible. The less they diverge, the fewer surprises you'll get when that green checkmark turns into a 500 error.

## What Goes Wrong Without Parity

Here's what happens when environments drift apart:

- **Different OS versions** — Your dev Mac runs perfectly, but the prod Linux box has a different `glibc` version and segfaults
- **Different dependency versions** — Dev has MySQL 8.0, prod is still on 5.7. That `RANK()` window function works locally but fails in prod
- **Different config** — Dev connects to a local Redis with no password, prod has TLS and auth. Your code works in dev, then throws connection errors in prod
- **Different data** — Dev has 10 test records, prod has 10 million. That N+1 query runs fast locally...

The 12-Factor App methodology calls this the **"Dev/Prod Parity"** principle. It's factor #10, and for good reason.

## Containers Fixed a Lot of This

Before Docker, we'd write setup scripts that tried to make every machine look the same. Bash scripts that installed packages, configured services, and prayed nothing changed.

Docker changed the game:

```
# This runs the same everywhere
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
CMD ["node", "server.js"]
```

That image builds exactly the same on your laptop, your CI pipeline, and your production server. The OS, runtime, and system packages are frozen in the image.

But containers aren't a silver bullet. You can still screw up parity in plenty of ways.

## The Big Three: Stacks, Config, Data

Parity breaks down in three main areas. Each needs its own treatment.

### 1. Technology Stack Parity

| Layer | Goal | Common Drift |
|-------|------|--------------|
| OS/Base image | Same distro, same patch level | Dev on macOS, prod on Ubuntu |
| Runtime | Exact same version | Node 18 vs Node 20 |
| Dependencies | Lock file must match | `npm install` vs `npm ci` |
| Infrastructure | Same DB, cache, queue versions | Local SQLite vs prod PostgreSQL |

**The fix:** Use containers. Lock your dependency files (`package-lock.json`, `Gemfile.lock`, `go.sum`). Use `npm ci` (which uses the lock file) instead of `npm install` (which can update it).

### 2. Configuration Parity

This is where things get sneaky. Different environments need different configs—that's expected. The problem is when configs diverge in *structure*:

```
// ❌ DEV and PROD read different config sources
// Dev config
const dbUrl = process.env.DATABASE_URL || 'mysql://root:password@localhost:3306/dev'

// Prod config  
const dbUrl = JSON.parse(process.env.DATABASE_JSON).url
```

Now your code behaves differently in each environment. A field renamed in one config source won't fail in dev—it'll only crash in prod.

**The fix:** Use the same config mechanism everywhere. The values change, the shape doesn't:

```
// ✅ Same structure, different values
// Dev: DATABASE_URL=mysql://root:pass@localhost:3306/dev
// Prod: DATABASE_URL=mysql://admin:secret@prod-db.cluster:3306/prod
const dbUrl = process.env.DATABASE_URL
```

### 3. Data Parity

This is the hardest one. You can't realistically give every developer a copy of production data (security, size, PII concerns).

But dev databases that look nothing like prod will mask real problems:

- **Volume differences:** Dev has 50 users, prod has 500,000. Indexes that work fine in dev fall apart at scale
- **Data shape differences:** Prod data has edge cases dev data doesn't—null fields, strange encodings, corrupted entries
- **Behavior differences:** Dev uses SQLite, prod uses PostgreSQL. That `JSON_EXTRACT()` function exists only in one

**Partial fixes:**
- Use **anonymized production data snapshots** for dev (strip PII, keep the shape)
- Use the **same DB engine** everywhere (no SQLite in dev, PostgreSQL in prod)
- Run a **staging environment** that mirrors production's scale, or at least its relative proportions

## Practical Steps for Better Parity

Here's a checklist I've found useful:

- [ ] **Dockerize everything.** Dev, CI, and prod all run the same container image
- [ ] **Lock your dependencies.** Commit lock files. Use `npm ci` / `pip install -r requirements.txt --no-cache`
- [ ] **Same DB engine everywhere.** No SQLite in dev, PostgreSQL in prod
- [ ] **Don't use different branches of infra.** If prod uses Redis, dev uses Redis (even if installed via Docker)
- [ ] **Env vars only for values, not structure.** Same config loading code in every environment
- [ ] **Seed dev databases with realistic data.** Especially for schema changes and migrations
- [ ] **Run CI with the same steps as deployment.** If prod runs migrations after deploy, CI should too

## The Staging Compromise

Full parity is expensive. Running a full-scale staging environment costs real money. Most teams compromise:

```
Laptop (Docker) → CI (Docker) → Staging (Prod-like) → Prod
```

Staging is your best attempt at production parity without the production traffic. It's where you catch issues that only show up with:
- Realistic data volume
- Actual DNS, load balancers, and TLS termination
- Realistic network latency

If you can't afford a staging environment, at least run your **CI pipeline** in an environment that mirrors production's OS and runtime.

## Trade-offs

The painful truth: **perfect parity isn't achievable.** Production has:
- Actual traffic and scaling
- Real user data (with all its weirdness)
- Different hardware (or at least VM specs)
- Different network topology

The goal isn't perfect parity. It's **minimizing risky differences** while accepting the ones that don't matter.

Most teams over-index on parity early on (trying to mirror production exactly) and under-index on it later (letting environments drift as the codebase grows). The sweet spot is somewhere in the middle—enough parity to catch problems early, but not so much that dev productivity suffers.

---

### What to Do Next

- **Today:** Check if your dev and prod use the same container base image and dependency lock files
- **This week:** Audit your config loading—make sure the mechanism is the same everywhere
- **This sprint:** Set up an anonymized production data snapshot for local development
- **Long term:** If you don't have a staging environment, build one. If you do, verify it's actually close to prod
