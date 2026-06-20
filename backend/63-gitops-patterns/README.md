# GitOps Patterns

Your deployment pipeline is fragile. Someone ran a `kubectl apply` on production, and now nobody knows what's actually running out there. The team's been asking "who changed the replica count?" for three hours.

GitOps fixes this mess. Let's walk through what it is, how it works, and when you'd actually want it.

---

## The Problem: Who Changed What?

In traditional deployment workflows, infrastructure drifts constantly:

- Someone SSH's into a box and hotfixes config
- A CI pipeline pushes a new image but forgets to update the manifests
- Terraform state gets out of sync because someone ran a patch manually

The result: **nobody can trust what's actually deployed**.

```bash
# ❌ DON'T DO THIS - blind trust in manual changes
ssh prod-server-01 "apt install nginx && systemctl start nginx"

# ✅ What you want - every change is in Git
# Open a PR, get it reviewed, merge, let automation apply it
```

---

## What Is GitOps, Really?

GitOps is a deployment pattern where **Git is the single source of truth** for your entire infrastructure. If it's not in Git, it doesn't exist. If it's in Git, the system makes it real.

Three core principles:

| Principle | What It Means |
|-----------|---------------|
| **Declarative** | You describe the desired state (not the steps to get there) |
| **Version-controlled** | Every change goes through Git — full audit trail |
| **Automated reconciliation** | A controller constantly ensures reality matches Git |

### The Reconciliation Loop

This is the heart of GitOps:

```
  Git Repo (desired state)
       │
       ▼
  ┌──────────────┐
  │  Operator /  │◄──── continuous comparison
  │  Controller  │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Kubernetes  │ (or whatever you're deploying to)
  │  Cluster     │
  └──────────────┘
         │
         └──── reports back status to Git
```

The controller doesn't push changes. It pulls from Git and makes reality match.

---

## Pull-Based vs Push-Based GitOps

This is the big architectural decision.

### Push-Based (Old School)

Your CI pipeline runs, builds an image, and then *pushes* the deployment to the cluster.

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    steps:
      - run: docker build -t app:${{ github.sha }}
      - run: docker push app:${{ github.sha }}
      - run: kubectl set image deployment/app app=app:${{ github.sha }}  # push!
```

**Problem:** Your CI needs cluster credentials. If CI gets compromised, your cluster is toast.

### Pull-Based (True GitOps)

Your CI only builds and pushes the image. A separate GitOps operator in the cluster *pulls* the desired state.

```yaml
# Application manifest that Flux/ArgoCD watches
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: app:v1.0.0  # updated via PR, not CI pushing
```

**Flow:**
1. Dev updates image tag in Git → PR → merge to main
2. GitOps operator detects change
3. Operator applies to cluster

```
✅ Safer: CI doesn't have cluster access
✅ Better audit trail: every change is a commit
✅ Auto-remediation: if someone kubectl apply's outside Git, operator reverts it
```

---

## Setting Up a GitOps Workflow

Let's look at how this works with ArgoCD, one of the most popular tools.

### Directory Structure

```
infra/
├── apps/
│   ├── production/
│   │   ├── api/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── web/
│   │       └── ...
│   └── staging/
│       └── ...
└── clusters/
    └── prod-cluster/
        └── argocd/
            └── applications.yaml  # defines what apps to sync
```

### ArgoCD Application Definition

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/team/infra.git
    targetRevision: main
    path: apps/production/api
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true          # remove resources no longer in Git
      selfHeal: true       # revert manual changes
    syncOptions:
      - Validate=true
      - CreateNamespace=true
```

The key line here is `automated: { prune: true, selfHeal: true }`. This means:

- **Prune:** If a manifest is deleted from Git, ArgoCD deletes it from the cluster
- **Self-heal:** If someone runs `kubectl delete pod` or `kubectl scale deployment x --replicas=5`, the operator reverts it

---

## GitOps Patterns You'll Actually Use

### 1. Environment Promotion

Don't manage dev, staging, and prod separately. Promote images through environments using Git.

```yaml
# staging/deployment.yaml
image: app:1.2.3

# When ready for prod, create a PR that updates:
# prod/deployment.yaml
image: app:1.2.3
```

```
❌ Without GitOps:
  dev → deploy script A → staging → manual copy → prod

✅ With GitOps:
  dev merged → staging auto-syncs → PR to prod → reviewed → merged → prod auto-syncs
```

### 2. Multi-Cluster Management

One Git repo, many clusters, same config with small overrides.

```yaml
# apps/api/overlays/
├── dev/
│   └── patch-replicas.yaml     # replicas: 1
├── staging/
│   └── patch-replicas.yaml     # replicas: 3
└── production/
    └── patch-replicas.yaml     # replicas: 10
```

Use Kustomize overlays or Helm values per environment. The GitOps operator picks the right overlay based on the path.

### 3. Drift Detection and Remediation

This is where GitOps really shines. The operator constantly compares real state vs desired state.

```
Operator check cycle:
  1. Fetch desired state from Git
  2. Compare against cluster state
  3. If different → log the diff and apply desired state
  4. Wait N seconds, repeat
```

**What happens when someone goes rogue:**

```bash
# Someone runs this in panic:
kubectl scale deployment api --replicas=20

# 3 minutes later (default sync interval):
# ArgoCD sees: Git says replicas=5, cluster says replicas=20
# ArgoCD scales back to 5

# Logs show:
# argocd-server[123]: Detected drift. api/production: spec.replicas: 5 != 20
# argocd-server[123]: Applying desired state...
```

---

## When GitOps Is Terrible

Let's be real — GitOps isn't a silver bullet.

### ✅ GitOps Is Great For

- Production deployments with audit requirements
- Multi-cluster environments
- Teams that need change control
- Infrastructure that doesn't change every minute

### ❌ GitOps Can Suck For

- **Ephemeral environments** — spinning up 50 preview environments per PR means 50 ArgoCD apps
- **Stateful workloads** — databases with auto-scaling that changes replica counts constantly (the GitOps operator fights the autoscaler)
- **Emergency rollbacks** — waiting for a PR to merge during an outage is painful. You need a branch strategy that makes rollbacks a fast-forward
- **Teams unfamiliar with Git workflows** — GitOps enforces discipline. If your team can't handle PR workflows, GitOps won't fix that

### The Database Problem

```yaml
# Git says:
replicas: 3

# But your HPA says:
# "We need 7 replicas for this DB right now!"
```

The GitOps operator sees 3, HPA sets 7, operator reverts to 3, HPA sets 7... **tug of war**.

**Workaround:** Exclude certain resources from GitOps reconciliation, or use "observe mode" for autoscaled workloads.

---

## Tools Compared

| Tool | Approach | Best For |
|------|----------|----------|
| **ArgoCD** | Pull-based, full UI, SSO | Teams that want visibility and control |
| **Flux** | Pull-based, lightweight, no UI | K8s-native teams, simpler setups |
| **Jenkins X** | Push-based with GitOps wrappers | Existing Jenkins shops |
| **Terraform Cloud** | Push with Git-driven runs | Infrastructure provisioning (not app deployment) |

**My take:** ArgoCD if you want visual feedback and RBAC. Flux if you want minimal moving parts.

---

## Takeaway Patterns

1. **Keep your Git repos clean** — one repo per team or app, not a monorepo of 200 apps unless you have solid CI
2. **Use Kustomize or Helm** — raw YAML becomes unmanageable past 5 manifests
3. **Don't GitOps stateful stuff** — databases, caches, anything with HPA that changes replicas rapidly
4. **Commit messages matter** — "fix prod" is useless. "fix: increase api replicas to handle traffic spike" is gold
5. **Automate rollbacks** — tag your commits, and have a "rollback-to-v1.2.3" script that's faster than a manual revert
