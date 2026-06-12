# Artifact Management: Registries

You've built your application, Dockerized it, and it runs perfectly on your machine. Now how do you get that exact image onto a production server without `scp`-ing tarballs around like it's 1999?

That's where artifact registries come in. They're the "package manager" for your deployments — a central place to store, version, and distribute your build outputs.

## What's an Artifact Anyway?

An artifact is anything your build process produces that needs to be consumed later:

- Docker images
- JAR/WAR files
- npm packages
- Python wheels
- Compiled binaries
- Helm charts
- Terraform modules

You could stash these on an S3 bucket with versioned keys, but registries give you something better: **semantic metadata, access control, vulnerability scanning, and lifecycle policies**.

## The Three Types You'll Deal With

### 1. Container Registries

This is the one you'll interact with most. Docker Hub is the most famous, but there are better options for production.

**The big players:**

| Registry | Best For | Gotcha |
|----------|----------|--------|
| Docker Hub | Open-source projects, public images | Rate limits on free tier (200 pulls/6h) |
| GitHub Container Registry (ghcr.io) | Tight GitHub integration | Requires PAT for auth |
| Amazon ECR | AWS-native workloads | IAM complexity |
| Google Artifact Registry | GCP-native, multi-format | Networking costs |
| Self-hosted (Harbor, Nexus) | Air-gapped, compliance | You maintain it |

**The flow is always the same:**

```bash
# Build
docker build -t my-app:1.0.0 .

# Tag with registry URL
docker tag my-app:1.0.0 ghcr.io/myorg/my-app:1.0.0

# Authenticate
echo $PAT | docker login ghcr.io -u myuser --password-stdin

# Push
docker push ghcr.io/myorg/my-app:1.0.0
```

### 2. Package Registries

These handle language-specific artifacts. npm for JavaScript, PyPI for Python, Maven Central for Java.

The ecosystem is fragmented, but the pattern is the same everywhere:

```
┌─────────────┐     publish     ┌──────────────────┐
│  Your CI     │ ──────────────> │  Package Registry │
│  (build)     │                 │  (npm, PyPI, etc) │
└─────────────┘                 └──────────────────┘
                                         │
                                    download
                                         │
                                         ▼
┌─────────────┐                 ┌──────────────────┐
│  Production  │ <────────────── │  Package Registry │
│  (install)   │                 │                   │
└─────────────┘                 └──────────────────┘
```

### 3. Generic Artifact Stores

For everything else — binary releases, firmware, datasets. Most cloud registries support "generic" artifacts now.

```
❌ Scp-ing a .jar to a server
✅ Publishing it to Artifactory with version metadata
```

## Tagging Strategies — Don't Just Use `latest`

`latest` is a trap. It's mutable, ambiguous, and breaks reproducibility.

**Good tagging patterns:**

```yaml
# ✅ Immutable tags
my-app:1.0.0           # SemVer
my-app:1.0.0-b123      # SemVer + build number
my-app:sha-abc1234     # Git commit SHA

# ✅ Environment tags (mutable but useful for rollouts)
my-app:staging-1.0.0
my-app:production-1.0.0

# ❌ Avoid these
my-app:latest          # What's in it today? Tomorrow?
my-app:v1              # Too vague
my-app:fix-login-bug   # Never use branch names
```

**The safest approach:** Use Git SHA as your primary tag, with SemVer as a human-readable alias.

```bash
# CI pipeline
VERSION="1.0.0"
SHA=$(git rev-parse --short HEAD)

docker build -t my-app:${SHA} .
docker tag my-app:${SHA} my-app:${VERSION}
docker tag my-app:${SHA} my-app:${VERSION}-${SHA}

docker push my-app:${SHA}
docker push my-app:${VERSION}
docker push my-app:${VERSION}-${SHA}
```

This way every image is immutable (SHA-based) but you can reference them by version too.

## Vulnerability Scanning

This is the killer feature of modern registries. Every major registry scans your images for known CVEs before you deploy them.

```
# Using Docker Scout
docker scout cves my-app:1.0.0

# Output example:
#  ✗ Critical: 2 (CVE-2024-xxx)
#  ✗ High: 5
#  ✗ Medium: 12
#  ✓ Low: 3
#  ✓ Negligible: 8
```

**Why this matters:** That base image you pulled from Docker Hub three months ago? It likely has vulnerabilities now. Registries with automated scanning catch this.

❌ Pulling a base image once and never re-scanning
✅ Setting up registry scanning + webhook alerts for critical CVEs

## Lifecycle Policies

Registries grow fast. A CI pipeline pushing images on every commit can quickly rack up thousands of unused images.

**You need cleanup rules:**

```hcl
# Example ECR lifecycle policy
rules = [
  {
    rulePriority = 1
    description  = "Keep last 10 tagged images"
    selection = {
      tagStatus   = "tagged"
      countType   = "imageCountMoreThan"
      countNumber = 10
    }
    action = { type = "expire" }
  },
  {
    rulePriority = 2
    description  = "Expire untagged images after 7 days"
    selection = {
      tagStatus   = "untagged"
      countType   = "sinceImagePushed"
      countNumber = 7
    }
    action = { type = "expire" }
  }
]
```

**Key rules of thumb:**

- **Untagged images:** Expire after 7-14 days (they're orphaned layers from failed pushes or retagged images)
- **Tagged old images:** Keep last N versions (nobody needs 200 versions of `my-app:1.0.0-${SHA}`)
- **Production images:** Never auto-delete. Protect them with immutability settings

## Proxying and Caching

Every time you pull an ubuntu:22.04 base image in CI, you're downloading ~80MB from Docker Hub. Do this 50 times a day and you're burning bandwidth — and hitting rate limits.

**Solution:** Set up a pull-through cache in your registry.

```yaml
# Harbor proxy cache configuration
repositories:
  - endpoint: https://registry-1.docker.io
    cache: harbor.internal/docker-hub-cache

# Now your CI pulls from:
FROM harbor.internal/docker-hub-cache/ubuntu:22.04
# Instead of:
FROM ubuntu:22.04
```

You get:
- Faster pulls (local network vs internet)
- No rate limit issues
- Immutable cache (pinned versions don't change)
- Air-gapped builds work offline

## What Breaks Without a Registry

Let's be real about this:

- **Deployments become fragile** — you're pulling "whatever is latest" instead of a pinned artifact
- **Rollback is impossible** — you overwrote the good image with a bad one
- **Audit trail is missing** — who pushed what, when, and with which vulnerabilities?
- **Dev/Prod drift** — the image in dev isn't the same as the one in prod because someone rebuilt it
- **CI takes longer** — rebuilding from source everywhere instead of pulling a pre-built artifact

## The Setup I'd Recommend

For a small team starting out:

```
src/ ─> CI (GitHub Actions) ─> ghcr.io ─> Production
                                │
                          Package registry (npm for JS, PyPI for Python)
                                │
                          npm install/pip install in services
```

For a larger team or regulated environment:

```
src/ ─> CI ─> Harbor (self-hosted) ─> Kubernetes
              │
              ├── Docker images
              ├── Helm charts
              ├── npm packages (proxied cache)
              └── Vulnerability scanner (Trivy)
```

## Takeaways

- **Use a registry, not an S3 bucket** — metadata, scanning, and lifecycle matter more than storage cost
- **Tag immutably** — Git SHA + SemVer, never `latest`
- **Scan everything** — CVEs in base images are the most common supply chain risk
- **Set lifecycle policies early** — cleanup is cheaper than storage bills
- **Cache upstream images** — faster builds, no rate limits, works offline
- **Protect production tags** — make them immutable so nobody overwrites a deployed image
