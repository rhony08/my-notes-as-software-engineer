# Docker Basics for Backend Devs

You've been in this situation before: you write code on your machine, everything works perfectly. Then you push to staging and... "works on my machine" becomes the standard excuse. Dependencies conflict. OS differences break things. The Python version on prod is somehow different from what you tested on.

Docker solves this. Not by magic — by bundling your app with everything it needs to run, from the OS packages to the runtime to the config files.

## What Docker Actually Does

Docker packages your application into a **container** — think of it as a lightweight, isolated environment that includes your code, runtime, system tools, libraries, and settings. Unlike a virtual machine, it shares the host OS kernel, so it boots in seconds and uses way less memory.

```
┌─────────────────────────────────────┐
│         Container                   │
│  ┌──────────────────────────────┐   │
│  │    Your App Code             │   │
│  ├──────────────────────────────┤   │
│  │    Dependencies (npm, pip)   │   │
│  ├──────────────────────────────┤   │
│  │    Runtime (Node, Python)    │   │
│  ├──────────────────────────────┤   │
│  │    OS Libraries (Alpine)     │   │
│  └──────────────────────────────┘   │
│  Shares host kernel (not a full OS) │
└─────────────────────────────────────┘
```

A VM would include a Guest OS on top instead — that's gigabytes vs megabytes.

## Your First Dockerfile

A `Dockerfile` is just a recipe. Here's one for a Node.js API:

```dockerfile
# Start from a base image with Node 20
FROM node:20-alpine

# Set working directory inside the container
WORKDIR /app

# Copy package files first (caching trick — explained below)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy the rest of the application code
COPY . .

# Tell Docker what port the app listens on
EXPOSE 3000

# Command to run when container starts
CMD ["node", "server.js"]
```

### Why copy package.json separately from the code?

Docker builds in **layers**. Each instruction (`FROM`, `COPY`, `RUN`) creates one layer. Docker caches layers — if nothing changed, it reuses the cache.

So `COPY package*.json ./` + `RUN npm ci` are cached until your `package.json` changes. That means during development, you can rebuild in seconds instead of waiting for `npm install` every time you change a line of code.

❌ **Don't do this:**

```dockerfile
COPY . .
RUN npm ci
```

Every code change triggers a full reinstall. Painfully slow.

✅ **Do this instead:**

```dockerfile
COPY package*.json ./
RUN npm ci
COPY . .
```

Dependencies installed once, cached. Code changes only rebuild the last layer.

## Building and Running

```bash
# Build the image (creates the blueprint)
docker build -t my-api:latest .

# Run a container (creates a running instance from the image)
docker run -p 3000:3000 my-api:latest
```

Now your app is running at `localhost:3000` inside a container. The `-p 3000:3000` maps the container's port 3000 to your machine's port 3000.

## Common Commands You'll Actually Use

| Command | What it does |
|---------|-------------|
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker images` | List downloaded images |
| `docker stop <container>` | Gracefully stop a container |
| `docker rm <container>` | Remove a stopped container |
| `docker rmi <image>` | Delete an image |
| `docker logs <container>` | See stdout/stderr of a running container |
| `docker exec -it <container> sh` | Get a shell inside a running container |

## The Image vs Container Confusion

This trips everyone up at first:

- **Image** = the blueprint (like a class in OOP)
- **Container** = a running instance of an image (like an object)

You can build one image and run 10 containers from it — they're all isolated copies.

## Volumes: Not Losing Your Data

Containers are ephemeral. Stop a container and everything you've written to its filesystem is gone. That's by design — it keeps containers disposable and reproducible.

But what about databases? Or logs you want to persist?

**Volumes** are the answer:

```bash
# Create a named volume that persists
docker run -v my-data:/app/data my-api:latest
```

`my-data` lives on your host machine, managed by Docker. Even if you delete the container, the volume stays.

❌ **Don't store state inside containers.** Your database container will lose all data when it's recreated.

✅ **Always use volumes for persistent data.** Your Postgres container uses a volume for its data directory — that's the standard pattern.

## The Developer Experience

Once you have a Dockerfile, onboarding a new dev goes from "install Node 20, MongoDB, Redis 7, configure environment variables, pray" to:

```bash
git clone <repo>
docker build -t my-api .
docker run -p 3000:3000 my-api
```

That's it. The container has everything. No more setup docs that are already outdated.

## But Docker Isn't Magic

Here's what people don't tell you:

- **Disk space.** Images pile up fast. Run `docker system prune` occasionally to clear unused images, containers, and networks.
- **Permission issues.** Volume mounts between macOS/Linux often cause file ownership problems. Your container runs as root but writes files owned by... someone else.
- **Mac performance.** Docker Desktop on Mac uses a VM under the hood, so file system operations on mounted volumes are noticeably slower than native.
- **Build time.** Large images take forever to push and pull. That's why Alpine-based images (around 5MB base) exist — versus `node:latest` which is over 1GB.

## Takeaway

Docker won't fix bad architecture. It won't make your code better. But it will *absolutely* eliminate the "works on my machine" problem, make deploys reproducible, and let you run databases in isolation without installing them globally.

Start simple — a single `Dockerfile` for one service. Add `docker-compose` when you need multiple services. Don't jump into Kubernetes until you're running more than a handful of containers.

The goal isn't to containerize everything. It's to make environments predictable so you can focus on actually shipping software.
