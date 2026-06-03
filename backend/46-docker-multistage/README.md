# Multi-stage Docker Builds

You've built a Docker image for your Go service. It works. But it's 1.2GB. Pulling that image on every deploy takes forever, and you're storing gigabytes of build tools in production that your runtime doesn't even need.

Multi-stage builds fix this. They let you use one Dockerfile to build your app in a "builder" stage (with all the compilers, SDKs, and dev dependencies), then copy only the compiled artifact into a slim production image.

## The Problem: Bloated Images

Here's the naive approach — a single-stage Dockerfile for a Go app:

```dockerfile
FROM golang:1.22

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o myapp .

EXPOSE 8080
CMD ["./myapp"]
```

What's in this image?

- Go compiler (hundreds of MB)
- Package manager cache
- Build tools (gcc, make, etc.)
- Your source code (exposed!)
- The compiled binary

A `golang:1.22` base image alone is ~800MB **compressed**. Your app is just a 15MB binary sitting inside all that dead weight.

❌ **This wastes bandwidth, disk, and startup time — and unnecessarily exposes source code.**

## Multi-stage to the Rescue

Multi-stage builds let you define multiple `FROM` statements in one Dockerfile. Each `FROM` starts a new stage. You copy artifacts from earlier stages into later ones.

```dockerfile
# Stage 1: Build the binary
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/myapp .

# Stage 2: Runtime image — tiny
FROM alpine:3.20
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

The final image? Around **20MB** instead of 800MB+. The compiler, source code, and build tools never make it into production.

```
┌─────────── Build Stage ───────────┐
│ golang:1.22                       │
│  ┌─ go.mod, go.sum ───────────┐   │
│  ┌─ go mod download ──────────┘   │
│  ┌─ go build → myapp ─────────┐   │
└────────────┬───────────────────────┘
             │ COPY --from=builder
             ▼
┌───────── Runtime Stage ───────────┐
│ alpine:3.20                       │
│  ┌─ myapp (compiled binary) ──┐   │
│  ┌─ ca-certificates ──────────┘   │
└───────────────────────────────────┘
```

## Beyond Go: Works for Any Compiled Language

**Rust:**
```dockerfile
FROM rust:1.77 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN cargo fetch
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/
CMD ["myapp"]
```

**Node.js (full build then slim runtime):**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
# Only copy what's needed at runtime
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
CMD ["node", "dist/server.js"]
```

For Node.js, the savings are less dramatic (Node runtime is needed in both stages), but you still avoid shipping `node_modules` with dev dependencies or source TypeScript files.

## Not Just for Compiled Languages

Even interpreted languages benefit when you have build steps:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt ./
RUN pip install --user -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY ./app ./app
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app/main.py"]
```

**Why this matters:** If a Python package requires compiling native extensions (like `numpy` or `psycopg2`), the builder stage handles that. The runtime stage only keeps the installed packages, not the compiler toolchain.

## Advanced Patterns

### Multiple Build Artifacts

Some apps need multiple binaries from one Dockerfile:

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /app/server ./cmd/server
RUN go build -o /app/worker ./cmd/worker

FROM alpine:3.20
COPY --from=builder /app/server /app/server
COPY --from=builder /app/worker /app/worker
CMD ["/app/server"]
```

### Named Stages for Clarity

You can reference stages by name (`AS builder`) or by index (`COPY --from=0`). **Always use names.** Indexes break when you reorder stages.

### Build Arguments for Flexibility

```dockerfile
ARG APP_VERSION=dev

FROM node:20-alpine AS builder
ARG APP_VERSION
RUN echo "Building version: $APP_VERSION"
# ... build steps

FROM node:20-alpine
COPY --from=builder /app/dist ./dist
ENV APP_VERSION=${APP_VERSION}
```

```bash
docker build --build-arg APP_VERSION=1.2.3 -t myapp .
```

### Using Distroless for Maximum Slimming

For extra paranoia about attack surface, use Google's distroless images:

```dockerfile
FROM golang:1.22 AS builder
# ... build steps

FROM gcr.io/distroless/base-debian12
COPY --from=builder /app/myapp /myapp
EXPOSE 8080
CMD ["/myapp"]
```

No shell, no package manager, no utilities — just your binary and libc. About 5MB total. Great for security-conscious deployments.

## When Multi-stage Doesn't Help

Multi-stage isn't always the answer:

| Case | Why not |
|------|---------|
| **Scripting languages (single file)** | A Python script with no native deps just needs `python:3.12-slim` — no build stage needed |
| **PHP + Apache** | The runtime already includes everything; multi-stage adds complexity for no gain |
| **Small apps on fast infra** | If your deploy pipeline is sub-10s and storage is cheap, the complexity may not be worth it |

✅ **Use multi-stage when:** You have a compilation/transpilation step, native dependencies, or you care about image size for slow connections or cold starts.

❌ **Skip it when:** Your app doesn't build anything and the runtime image is already small enough.

## Practical Comparison: Before vs After

Here's a real-world example from a Go service I worked on:

**Before (single stage):**
```
REPOSITORY   TAG       IMAGE ID       SIZE
myapp        latest    a1b2c3d4e5f6   812MB
```
Deploy time (from CI to container runtime): ~45 seconds for image pull on a 200Mbps connection.

**After (multi-stage):**
```
REPOSITORY   TAG       IMAGE ID       SIZE
myapp        latest    f6e5d4c3b2a1   18MB
```
Deploy time: ~3 seconds.

That's 42 seconds saved *per deploy*. On a team deploying 20 times a day, that's 14 minutes per day *per service*.

## A Word on Caching

Multi-stage builds still benefit from layer caching within each stage. The key rules from Day 1 still apply:

```dockerfile
# ❌ Breaks cache on every code change
FROM golang:1.22 AS builder
COPY . .
RUN go mod download
RUN go build -o /app/myapp .

# ✅ Caches dependencies separately
FROM golang:1.22 AS builder
COPY go.mod go.sum ./
RUN go mod download      ← cached until go.mod changes
COPY . .
RUN go build -o /app/myapp .  ← only re-runs when code changes
```

Same principle, just applied inside the builder stage.

## Takeaway

- Multi-stage builds give you a single `Dockerfile` that produces a tiny production image
- The compile-time dependencies (Go, Rust, native extensions) stay in the builder stage
- Final images go from hundreds of MB to tens of MB — faster deploys, less disk, smaller attack surface
- Works best for compiled languages, but also useful for any build step (TypeScript → JS, SASS → CSS)
- Not needed for simple interpreted apps — start with a slim base image first
