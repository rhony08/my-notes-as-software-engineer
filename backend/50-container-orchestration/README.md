# Container Orchestration Choices

You've got your app containerized. Dockerfile's clean, images are small, `docker run` works great on your machine. Now you need to run this thing in production — across multiple servers, handle failures, scale up when traffic spikes, roll out updates without downtime.

That's the moment you need an orchestrator.

But which one? The container orchestration landscape has more players than a tournament bracket. Let's cut through the noise.

---

## The Orchestrator's Job

First, what's an orchestrator actually doing?

- **Scheduling** — deciding which server runs which container
- **Service discovery** — containers talking to each other without hardcoded IPs
- **Scaling** — spinning up or down containers based on load
- **Health monitoring** — restarting failed containers
- **Rolling updates** — updating without downtime
- **Networking & load balancing** — routing traffic to healthy containers

Every orchestrator handles these. The differences are in *how* and *how well*.

---

## The Main Players

### Kubernetes (K8s)

Kubernetes is the 800-pound gorilla. It's what everyone compares everything to.

**The good:**
- Massive ecosystem — Helm charts, Operators, CRDs, service meshes
- Portable — runs on AWS, GCP, Azure, bare metal, your laptop
- Rich primitives — Deployments, StatefulSets, DaemonSets, Jobs
- Self-healing at every level
- Huge community and job market

**The bad:**
- Genuinely complex — the learning curve is a vertical cliff
- Overkill for small setups
- Requires dedicated operational knowledge
- Upgrade challenges between minor versions

```
// A minimal Kubernetes Deployment — even this has a lot of moving parts
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: app
        image: myapp:1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:   # Minimum guaranteed resources
            cpu: 250m
            memory: 256Mi
          limits:     # Maximum allowed resources
            cpu: 500m
            memory: 512Mi
        readinessProbe:  # Don't send traffic until this passes
          httpGet:
            path: /health
            port: 8080
```

**Verdict:** If your team has the ops muscle and you need the flexibility, K8s is the long-term play.

---

### Docker Swarm

Docker Swarm is built into the Docker engine itself. It's Kubernetes' simpler cousin.

**The good:**
- Dead simple to set up — `docker swarm init` and you're done
- Uses the same `docker-compose.yml` syntax you already know
- Much lower learning curve
- Good enough for small-to-medium deployments

**The bad:**
- Limited ecosystem compared to K8s
- Fewer features — no built-in service mesh, CRDs, or Operators
- Smaller community, less tooling
- Docker Inc. has essentially stopped major feature development

```
// Docker Swarm stack — familiar compose syntax
version: '3.8'
services:
  api:
    image: myapp:1.2.3
    ports:
      - "8080:8080"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

**Verdict:** Great if you just need "distributed Docker" and don't care about advanced orchestration. Losing steam though.

---

### HashiCorp Nomad

Nomad is the unsung hero. It orchestrates containers but also standalone apps, Java JARs, and raw executables.

**The good:**
- Single binary — dead simple to install and operate
- Handles non-container workloads too (raw processes, Java, QEMU VMs)
- Easier to operate than Kubernetes — no control plane complexity
- Integrates with Consul (service discovery) and Vault (secrets)
- Great for batch and cron workloads

**The bad:**
- Smaller ecosystem and community
- Fewer out-of-box features (no built-in ingress, limited auto-scaling)
- You'll need to piece together some infrastructure yourself

```
# Nomad job specification
job "api-server" {
  datacenters = ["dc1"]
  type = "service"

  group "api" {
    count = 3

    task "server" {
      driver = "docker"
      config {
        image = "myapp:1.2.3"
        port_map {
          http = 8080
        }
      }
      resources {
        cpu    = 500
        memory = 256
      }
      service {
        name = "api-server"
        port = "http"
        check {
          type     = "http"
          path     = "/health"
          interval = "10s"
        }
      }
    }
  }
}
```

**Verdict:** Underrated choice for teams already using HashiCorp stack, or for mixed container/bare-process workloads.

---

### Amazon ECS / Fargate

ECS is AWS's managed container service. Fargate is a serverless compute engine for containers.

**The good:**
- No control plane to manage — AWS handles it
- Deep AWS integration (ALB, IAM, CloudWatch, VPC)
- Fargate means no EC2 instances to manage at all
- AWS manages upgrades, patching, and availability

**The bad:**
- Locked to AWS — you can't easily migrate
- Less flexibility than Kubernetes
- Complex IAM permission model
- Fargate pricing can surprise you at scale

```
// ECS task definition (JSON, because of course)
{
  "family": "api-server",
  "taskRoleArn": "arn:aws:iam::123456789:role/api-task-role",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:1.2.3",
      "memory": 512,
      "cpu": 256,
      "portMappings": [
        { "containerPort": 8080, "hostPort": 8080 }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-server",
          "awslogs-region": "us-east-1"
        }
      }
    }
  ]
}
```

**Verdict:** Best if you're already all-in on AWS and want to minimize operational overhead.

---

## How to Choose

| Factor | K8s | Swarm | Nomad | ECS |
|--------|-----|-------|-------|-----|
| Setup complexity | High | Low | Low | Low |
| Learning curve | Steep | Gentle | Moderate | Moderate |
| Ecosystem | Massive | Small | Growing | AWS-only |
| Multi-cloud | ✅ | ✅ | ✅ | ❌ |
| Non-container workloads | ❌ (sorta) | ❌ | ✅ | ❌ |
| Operational overhead | High | Low | Low | Very low (managed) |
| Best for | Big teams, complex needs | Simple deploys | Ops teams, mixed workloads | AWS native stacks |

---

## Common Mistakes

❌ **Starting with Kubernetes for a 2-service app.** You'll spend more time on the cluster than the app.

✅ **Use Docker Compose or a lightweight orchestrator first.** Graduate to K8s when you actually need what it offers.

❌ **Thinking managed K8s (EKS/GKE/AKS) means "no ops."** You still need to manage node pools, upgrades, RBAC, networking, and monitoring.

✅ **Managed K8s reduces infra burden but doesn't eliminate ops.** You're trading some infrastructure work for Kubernetes complexity.

❌ **Treating all orchestrators as interchangeable.** Swarm and K8s have fundamentally different architectures — migration between them is not a trivial lift-and-shift.

---

## When Does It Actually Matter?

**You probably don't need an orchestrator until:**
- You're running more than 3-5 servers
- You need rolling updates with zero downtime
- You're dealing with multiple microservices that need to find each other
- Manual `docker run` on servers has caused incidents

**You definitely need one when:**
- A server died at 3am and nobody noticed until users complained
- You're afraid to deploy on Friday
- Your deployment process is a bash script that a colleague wrote and nobody understands

---

## Takeaways

- **Kubernetes** is the industry standard — pay the learning tax if you're going big
- **Docker Swarm** is fine for small teams that want simplicity, but it's not getting new features
- **Nomad** is a solid middle ground, especially for mixed workloads
- **ECS/Fargate** is the easy button if you're AWS-native
- Don't over-engineer. Your orchestrator choice should match your team size, ops maturity, and actual scaling needs
- The orchestrator is a tool, not a goal. Pick what lets you ship reliably with the least friction
