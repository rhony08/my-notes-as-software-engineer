# Container Security Best Practices

You've Dockerized your app. It builds. It runs. You push it to production and sleep easy.

Then someone points out your container is running as root. Your base image hasn't been updated in six months and has 14 known CVEs. And that `.env` file you baked into the image? It's sitting in the registry for anyone to pull.

Container security isn't optional — it's what separates "we use Docker" from "we run Docker safely." Let's go through the practices that actually matter.

---

## Don't Run as Root (Seriously)

This is the #1 mistake. By default, containers can run as root inside the container, which means if an attacker escapes the container, they've got root on your host.

```dockerfile
# ❌ Default root user
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]

# ✅ Non-root user
FROM node:20
WORKDIR /app
COPY . .
RUN npm install && \
    groupadd -r appuser && \
    useradd -r -g appuser appuser && \
    chown -R appuser:appuser /app
USER appuser
CMD ["node", "server.js"]
```

**But what about port 80/443?** Non-root users can't bind to privileged ports (<1024). Either use a reverse proxy (like nginx running on a different container) or map the port: `docker run -p 8080:3000 ...`.

> **Trade-off:** Adding a user adds one more layer to your Dockerfile. Some minimal images (like `distroless`) don't include `groupadd`/`useradd` — you'll need a multi-stage build where the first stage creates the user.

---

## Pick Your Base Image Carefully

`ubuntu:latest` is comfortable. It's also 200MB and ships with hundreds of packages you don't need — each one a potential attack surface.

```dockerfile
# ❌ Bloated and full of attack surface
FROM ubuntu:latest

# ✅ Minimal and focused
FROM node:20-alpine

# ✅ Even better - only what your app needs at runtime
FROM gcr.io/distroless/nodejs20-debian12
```

| Image | Size | Packages | CVE Surface | Good for |
|-------|------|----------|-------------|----------|
| `ubuntu:latest` | ~200MB | Hundreds | High | Dev environments |
| `node:20-alpine` | ~120MB | Minimal | Medium | Most apps |
| `distroless` | ~40MB | Almost none | Low | Production |
| `scratch` | ~0MB | Zero | None | Static binaries |

**The rule:** Start minimal, add only what's needed. Don't settle for "it works" — check if it's the minimum that works.

---

## Scan Images for Vulnerabilities

You wouldn't add an npm package without checking for known vulnerabilities. Why treat your base image differently?

```bash
# With Trivy (my go-to — fast and free)
trivy image my-app:latest

# With Docker Scout (built into Docker Desktop)
docker scout quickview my-app:latest

# With Snyk (also good, but rate-limited on free tier)
snyk container test my-app:latest
```

**Real example output from Trivy:**

```
node:20-alpine (alpine 3.19)
─────────────────────────────────
Total: 3 (HIGH: 1, CRITICAL: 0)

┌──────────────┬────────────────┬──────────┬───────────────────┐
│   Library    │ Vulnerability  │ Severity │      Fixed In     │
├──────────────┼────────────────┼──────────┼───────────────────┤
│ libssl3      │ CVE-2024-XXXX  │ HIGH     │ 3.19.2-r0         │
└──────────────┴────────────────┴──────────┴───────────────────┘
```

**Scan in CI, block builds that fail.** Your Dockerfile should never ship critical vulnerabilities to production.

```yaml
# GitHub Actions snippet
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-app:${{ github.sha }}'
    severity: 'HIGH,CRITICAL'
    exit-code: '1'
```

> **Heads up:** Scanning adds time to your CI pipeline. Scanning a 200MB image can take 30+ seconds. Distroless images scan in under 5 seconds. Yet another reason to go minimal.

---

## Secrets? Don't Bake Them In

This still happens more than it should:

```dockerfile
# ❌ NEVER DO THIS
ENV DB_PASSWORD=supersecret123
COPY .env ./

# ✅ Use build args for build-time secrets
ARG DB_PASSWORD
ENV DB_PASSWORD=$DB_PASSWORD

# ✅ Even better: use secrets mount (Docker BuildKit)
RUN --mount=type=secret,id=db_password \
    DB_PASSWORD=$(cat /run/secrets/db_password) \
    ./configure.sh
```

For runtime secrets, never embed them in the image:

```bash
# ❌ Bad - secret is in the image
docker run -e DB_PASSWORD=supersecret123 my-app

# ✅ Better - use Docker secrets (Swarm/Kubernetes)
echo "supersecret123" | docker secret create db_password -
docker service create --secret db_password my-app

# ✅ Also good - use a secrets manager at runtime
docker run -e DB_PASSWORD=$(aws secretsmanager get-secret-value ...) my-app
```

**The golden rule:** If your image needs a rebuild to change a secret, you're doing it wrong. Secrets are runtime concerns.

---

## Read-Only Root Filesystem

By default, containers have a writable filesystem. If an attacker gains code execution, they can drop malware, modify binaries, or persist.

```bash
# Make root filesystem read-only
docker run --read-only --tmpfs /tmp my-app
```

```yaml
# In Docker Compose
services:
  app:
    image: my-app
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

```yaml
# In Kubernetes
securityContext:
  readOnlyRootFilesystem: true
```

This won't break most apps — they just need writable `/tmp` or `/var/run`. The `--tmpfs` flag gives them exactly that without sacrificing security.

> **When this breaks:** Apps that write logs to arbitrary paths will fail. Fix it by making your app write to stdout (for container logs) or to a specific mounted volume.

---

## Resource Limits Prevent DoS

A container with no resource limits can eat your entire host. One memory leak in your app and suddenly all other containers get OOM-killed.

```bash
docker run --memory="512m" --cpus="0.5" my-app
```

```yaml
# Kubernetes
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"
```

**Set both requests and limits.** Requests guarantee minimum resources. Limits prevent runaway usage. Without limits, a single container can tank your entire node.

---

## Capabilities: Drop Everything, Add Back Only What's Needed

Containers run with a default set of Linux capabilities. Most apps don't need any of them.

```bash
# Run with ZERO capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE my-app
```

```yaml
# Kubernetes
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

| Capability | What it allows | When you need it |
|------------|---------------|------------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 | If mapping to port 80/443 in-container |
| `CHOWN` | Change file ownership | If running as non-root user |
| `SYS_PTRACE` | Debug processes | Development only |
| `SYS_ADMIN` | Mount, namespace ops | Almost never — huge red flag |

**Most apps work with `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`.** If your app needs more, ask yourself why — or fix the app.

---

## Networking: Don't Expose More Than You Need

Every exposed port is a potential entry point.

```yaml
# ❌ Exposes entire container network stack
services:
  app:
    network_mode: host

# ✅ Only expose what's necessary
services:
  app:
    ports:
      - "3000:3000"  # Only the app port
    expose:
      - "3000"

# ✅ Even better: use internal networks
services:
  app:
    networks:
      - internal
  db:
    networks:
      - internal  # Not exposed at all

networks:
  internal:
    internal: true  # No external access
```

**In Kubernetes,** use NetworkPolicies to restrict pod-to-pod communication:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - port: 3000
```

Default-deny ingress + explicit allow is the pattern. Don't assume "it works" is good enough.

---

## Keep Images Small (Fewer Moving Parts)

Smaller images = smaller attack surface. Every package in your image is code that could have a vulnerability.

```dockerfile
# ❌ Dev dependencies in production
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install  # Installs devDependencies too
COPY . .
CMD ["node", "server.js"]

# ✅ Multi-stage: build separately from runtime
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

| Approach | Image Size | Security |
|----------|-----------|----------|
| Single stage, full npm install | ~350MB | All devDeps + npm cache in image |
| Multi-stage, npm ci --production | ~130MB | Only production deps |
| Multi-stage + distroless | ~45MB | Bare minimum |

---

## Putting It All Together: A Secure Dockerfile

Here's what a production-grade secure Dockerfile looks like:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
USER 1001  # Non-root user in distroless
EXPOSE 3000
CMD ["server.js"]
```

Build and run safely:

```bash
# Build
docker build -t my-app:secure .

# Scan
trivy image my-app:secure

# Run with security best practices
docker run \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --memory="512m" \
  --cpus="0.5" \
  -p 3000:3000 \
  my-app:secure
```

---

## Actionable Takeaways

- **Never run as root.** Add a `USER` directive in your Dockerfile. It's two lines and it mitigates an entire class of attacks.
- **Minimize your base image.** Start with alpine or distroless, not ubuntu. Smaller = safer.
- **Scan everything, block criticals.** Integrate Trivy or Docker Scout into CI. Don't deploy known vulnerabilities.
- **Secrets are runtime values, not build artifacts.** Use BuildKit secrets mounts, Docker secrets, or a vault — never env vars in Dockerfiles.
- **Drop capabilities you don't need.** Start with `--cap-drop=ALL` and add back only what your app actually uses.
- **Set resource limits.** Memory and CPU limits prevent a single container from DoS-ing your host.
- **Audit regularly.** `docker scout` and `trivy` aren't one-time checks. Run them on every build and periodically on running images.
