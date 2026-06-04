# Docker Compose for Local Dev

Your backend needs PostgreSQL, Redis, a message queue, and that other microservice to run. Installing all of that on your laptop is a nightmare — different versions, conflicting ports, OS-specific setup instructions that always skip one step.

Docker Compose solves this. One file, one command, and your entire dev environment spins up.

## What Docker Compose Actually Is

It's a YAML file that declares all your services, networks, volumes, and config. `docker-compose up` reads it and starts everything together.

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
      REDIS_URL: redis://redis:6379

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

With this file in your project root:

```bash
# Start everything
docker compose up

# Start in background
docker compose up -d

# Tail logs from a specific service
docker compose logs -f app

# Stop everything
docker compose down
```

One command gets you a running app with a database and cache. New teammate joins? Share the file and they're running in 5 minutes.

## The Network Magic

The part that trips people up: **services talk to each other using service names, not IPs**.

Your `app` connects to `db:5432` and `redis:6379`, not `localhost:5432`. Docker Compose creates a default network and each service gets DNS resolution for the service name.

```javascript
// ❌ This fails - services aren't on localhost
const client = new Redis({ host: 'localhost' });

// ✅ This works - Docker DNS resolves 'redis' to the container
const client = new Redis({ host: 'redis' });
```

Host machines can still reach everything via `localhost`, because you mapped ports with the `ports:` directive.

## Why Compose Over Raw Docker Commands?

Without Compose:

```bash
docker network create myapp-net
docker run -d --network myapp-net --name db -e POSTGRES_PASSWORD=pass -v pgdata:/var/lib/postgresql/data postgres:16-alpine
docker run -d --network myapp-net --name redis redis:7-alpine
docker build -t myapp .
docker run -d --network myapp-net -p 3000:3000 -e DATABASE_URL=... -e REDIS_URL=... myapp
```

That's 5 commands to memorize and run in the right order. Compose turns it into one file and one command.

But there's a trade-off: the YAML is coupled to your local setup. What works for local dev shouldn't be the same config you throw at production. More on that below.

## The Local Dev Superpowers

### Hot Reload with Volume Mounts

Mount your source code into the container so changes reflect instantly:

```yaml
services:
  app:
    build: .
    volumes:
      - .:/app          # mount current dir into /app
      - /app/node_modules  # exclude node_modules from mount
    command: npm run dev  # dev server with hot reload
```

No rebuilding the image for every code change. Edit → Save → Refresh.

### Environment Overrides

Don't put production secrets in your base compose file. Use overlays:

```yaml
# docker-compose.override.yml (auto-loaded in dev)
services:
  app:
    environment:
      DEBUG: "true"
      LOG_LEVEL: debug
```

```yaml
# docker-compose.prod.yml
services:
  app:
    environment:
      NODE_ENV: production
```

```bash
# Dev (overrides auto-included)
docker compose up

# Production-like
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

This keeps dev configs and prod configs separate without duplicating files.

## Common Gotchas

### 1. Dependency Order != Readiness

`depends_on` only waits for the container to start, not for the service inside to be ready:

```yaml
services:
  app:
    depends_on:
      - db
      # app starts *after* db container starts,
      # but PostgreSQL might still be initializing
```

If your app connects before Postgres is ready, you'll get connection refused errors. Fix it with a wait script:

```bash
#!/bin/sh
# wait-for-it.sh
until nc -z db 5432; do
  echo "Waiting for postgres..."
  sleep 1
done
exec "$@"
```

Or use [wait-for-it](https://github.com/vishnubob/wait-for-it) / Docker Compose healthchecks:

```yaml
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 5s
      retries: 5

  app:
    depends_on:
      db:
        condition: service_healthy
```

### 2. Data Persistence

Without volumes, your database data disappears when you run `docker compose down`:

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Name your volumes so they persist between `down` and `up` cycles. Use `docker compose down -v` when you deliberately want to wipe everything.

### 3. Port Conflicts

If you already have PostgreSQL running on your machine, it'll clash with the container:

```bash
Error: unable to bind port 5432: address already in use
```

Solutions:
- Map to a different host port: `"5433:5432"`
- Stop your local Postgres service
- Only expose ports that need direct host access (your app doesn't need `db:5432` exposed unless you connect to it from external tools)

## When Not to Use Compose

Docker Compose is great for local dev and CI test environments. It's not designed for production.

For prod, you'd use:
- **Kubernetes** — multi-node orchestration, auto-scaling, self-healing
- **Docker Swarm** — simpler orchestration if you want Docker-native
- **Managed services** — nobody runs their own PostgreSQL in production with Docker unless they have a ops team

The gap becomes obvious when you try to scale: you can't `docker compose scale` across multiple machines. For a single-machine deployment with low traffic, sure, it works. But it's not the intended use case.

## Quick Reference

| Command | What It Does |
|---------|-------------|
| `docker compose up -d` | Start services in background |
| `docker compose down` | Stop and remove containers |
| `docker compose down -v` | Stop and remove containers + volumes |
| `docker compose logs -f` | Tail logs from all services |
| `docker compose build` | Rebuild images without starting |
| `docker compose exec app bash` | Open shell inside running container |
| `docker compose ps` | List running services |

## Actionable Takeaways

- Start every backend project with a `docker-compose.yml` — it's your infrastructure-as-code for local dev
- Mount your source code as a volume during development for hot reload
- Use healthchecks instead of just `depends_on` for services that take time to initialize
- Keep volumes named so your data survives restarts but can be wiped when needed
- Remember: Compose is for dev, not prod — when you need scaling and self-healing, look at Kubernetes

You'll waste a lot less time on "works on my machine" problems.
