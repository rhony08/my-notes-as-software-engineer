# Kubernetes: Pods, Services, Deployments

You've got your app running in a Docker container. Works great on your laptop. Now deploy it to production and suddenly "it works on my machine" becomes "why are there five replicas but only one is serving traffic, and by the way the database can't reach the app because the IPs changed on restart."

That's what Kubernetes solves. It gives you three fundamental building blocks that turn container wrangling into declarative configuration.

## Pods — The Smallest Deployable Unit

A Pod is a wrapper around one or more containers. Think of it like a pod of peas — multiple peas (containers) sharing a single shell (network namespace, storage volumes).

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
    tier: backend
spec:
  containers:
    - name: app
      image: my-app:1.2.3
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
```

**Key things about Pods:**

- Each Pod gets a unique IP inside the cluster
- Containers in the same Pod share `localhost` — useful for sidecar patterns (e.g., a logging sidecar reading from the app's log file)
- Pods are **ephemeral** — they die, get rescheduled, and you never address them directly

```bash
# ❌ Don't create bare Pods in production
kubectl run my-app --image=my-app:1.2.3

# ✅ Create Pods through a Deployment instead
kubectl create deployment my-app --image=my-app:1.2.3
```

> **Why this matters:** Bare Pods don't self-heal. If the node dies, your Pod is gone. A Deployment recreates it automatically.

## Deployments — The Self-Healing Abstraction

You almost never create Pods directly. You create a **Deployment**, and it manages the Pod lifecycle for you.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:1.2.3
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

| Deployment Feature | What It Does |
|---|---|
| **Replica management** | Keeps N Pods running at all times |
| **Rolling updates** | Replaces old Pods with new ones gradually |
| **Rollback** | Reverts to a previous revision if something breaks |
| **Self-healing** | Recreates Pods that crash or get deleted |

```bash
# Rollout a new version
kubectl set image deployment/my-app my-app=my-app:2.0.0

# Check rollout status
kubectl rollout status deployment/my-app

# Whoops — rollback
kubectl rollout undo deployment/my-app
```

### The Problem Deployments Don't Solve

Deployments manage Pods, but they don't give you a stable way to reach those Pods. Pod IPs change when they're rescheduled. You need a permanent address.

## Services — Stable Networking for Ephemeral Pods

A Service is a stable endpoint that load-balances traffic to a set of Pods matching a label selector.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### Service Types

| Type | Accessible From | Use Case |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Internal communication between services |
| **NodePort** | External via `<NodeIP>:<NodePort>` | Dev/testing, not production |
| **LoadBalancer** | External via cloud LB | Production traffic from the internet |

```bash
# Check which Pods the Service routes to
kubectl get endpoints my-app-service

# Port-forward to test locally
kubectl port-forward service/my-app-service 8080:80
```

### How Services Find Pods

Services don't track Pod IPs directly. They use **labels and selectors**:

```yaml
# Pod with matching labels will receive traffic
# Pod has: labels: { app: my-app }
# Service has: selector: { app: my-app }
```

This is loose coupling at its finest. If a Pod restarts and gets a new IP, the Service just updates its endpoint list and keeps routing traffic.

## Putting It All Together

```yaml
# complete-app.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:1.2.3
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

### The Flow

```
User → LoadBalancer → Service (my-app-service) → Pod (my-app-abc123)
                                                 → Pod (my-app-def456)
                                                 → Pod (my-app-ghi789)
```

- **Deployment** keeps 3 Pods running
- **Service** gives them a stable IP + DNS name
- **LoadBalancer type** exposes it externally
- When a Pod dies, Deployment recreates it, and the Service updates its endpoints automatically

## Common Gotchas

```yaml
# ❌ Wrong — Pod and Service labels don't match
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app-v2  # <-- mismatched label

---
apiVersion: v1
kind: Service
spec:
  selector:
    app: my-app  # <-- expects 'app: my-app' label
```

**Result:** Service finds zero endpoints. Traffic goes nowhere.

```bash
# Debug mismatches
kubectl get pods --selector=app=my-app       # Check what Pods match the Service selector
kubectl describe service my-app-service       # Check Endpoints field
kubectl get endpoints my-app-service          # Empty = no matching Pods
```

### Readiness vs Liveness Probes

```yaml
# 📌 Know the difference — they're not the same
spec:
  containers:
    - name: app
      readinessProbe:          # Is this Pod ready to receive traffic?
        httpGet:
          path: /ready
          port: 8080
      livenessProbe:           # Is this Pod healthy? Restart if not.
        httpGet:
          path: /live
          port: 8080
```

- **Readiness probe** fails → Service removes the Pod from rotation
- **Liveness probe** fails → Pod gets restarted
- Confusing them is a classic mistake. A high-load app might be alive but not ready to accept more traffic. Don't restart it — just stop sending traffic.

## Actionable Takeaways

- **Don't create bare Pods** — always use Deployments for self-healing and rollbacks
- **Services are how apps find each other** in the cluster — use DNS names (`my-app-service.namespace.svc.cluster.local`), not IPs
- **Labels and selectors** are the glue — if Pods and Services don't share label values, you have an unreachable app
- **Readiness probes matter more than you think** — without them, Services route traffic to Pods that are still starting up
- **`kubectl get endpoints`** is your best debugging friend when traffic doesn't reach your Pods
