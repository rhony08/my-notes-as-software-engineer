# Testing Infrastructure Code

You'd never push application code without running tests. Yet somehow, infrastructure code gets a pass all the time. We write Terraform modules, Ansible playbooks, and Kubernetes manifests — and deploy them into production hoping they work.

I've been there. `terraform apply` hits you with "Error: creating EC2 instance: InvalidParameterValue", and you realize your variable validation was missing. Or worse — it applies successfully but misconfigures a security group, leaving a port open.

Infrastructure code is code. It should be tested like code.

---

## Why Testing Infra Matters

When application code breaks, it usually affects a single feature or endpoint. When infrastructure breaks, **everything** goes down. A bad Terraform change can:

- Delete production databases
- Open security holes
- Misconfigure load balancers, dropping traffic
- Incur massive cloud bills from misconfigured resources

The kicker? Infra changes often look fine in `plan` output but break in subtle ways. That's why we need multiple layers of testing.

---

## The Four Layers of Infrastructure Testing

Not all tests are created equal. Here's the pyramid:

```
         /\
        /  \      Layer 4: E2E / Smoke Tests (prod-like env)
       /    \
      /------\    Layer 3: Integration Tests (real resources)
     /        \
    /----------\  Layer 2: Compliance / Policy Tests
   /            \
  /--------------\ Layer 1: Unit / Static Analysis (fastest, cheapest)
```

### Layer 1: Static Analysis & Linting

This is the easiest win. Before you even run anything, catch common mistakes:

```hcl
// ❌ No validation - you can pass any string
variable "instance_type" {
  description = "EC2 instance type"
}

// ✅ Validation prevents obvious mistakes at plan time
variable "instance_type" {
  description = "EC2 instance type"
  type = string

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium", "m5.large"], var.instance_type)
    error_message = "Instance type must be one of: t3.micro, t3.small, t3.medium, m5.large."
  }
}
```

**Tools for static analysis:**

| Tool | Language | What it catches |
|------|----------|-----------------|
| `tflint` | Terraform | Provider-specific issues, deprecated syntax |
| `checkov` | Multi-IaC | Security misconfigurations (open ports, unencrypted S3) |
| `tfsec` | Terraform | Security-focused static analysis |
| `ansible-lint` | Ansible | Best practice violations |
| `conftest` | OPA/Rego | Custom policy enforcement |

Run these in CI on every PR:

```yaml
# .github/workflows/terraform-lint.yml
name: Terraform Static Analysis

on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
      - run: terraform fmt -check -recursive
      - run: terraform validate
      - uses: aquasecurity/tfsec-action@v1.0.0
      - uses: bridgecrewio/checkov-action@v12
```

### Layer 2: Policy / Compliance Testing

Your infra might be syntactically correct, but does it violate company policies? Things like:

- "All S3 buckets must have encryption enabled"
- "No security groups with 0.0.0.0/0 ingress"
- "All RDS instances must be encrypted at rest"

This is where policy-as-code tools shine:

```hcl
# policy/require_encryption.rego (OPA / conftest)
package main

deny[msg] {
  resource := input.resource.aws_s3_bucket[name]
  not resource.server_side_encryption_configuration
  msg := sprintf("Bucket %v must have encryption enabled", [name])
}

deny[msg] {
  resource := input.resource.aws_security_group[name]
  rule := resource.ingress[_]
  rule.cidr_blocks[_] == "0.0.0.0/0"
  msg := sprintf("Security group %v allows traffic from 0.0.0.0/0", [name])
}
```

Run with:

```bash
conftest test --policy policy/ terraform/plan.json
```

**Why this matters:** These checks are repeatable, automated, and don't rely on human code review catching the 50th security group that was opened "temporarily".

### Layer 3: Integration Tests with Real Resources

This is where things get real. You actually provision infrastructure, run tests against it, then tear it down.

**The golden rule:** Infrastructure tests must be **idempotent** and **self-cleaning**. If a test fails halfway, it shouldn't leave orphaned resources.

Here's a pattern using Terratest (Go):

```go
// test/terraform_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
)

func TestVPCDeployment(t *testing.T) {
  terraformOptions := &terraform.Options{
    TerraformDir: "../",

    // Use isolated test variables
    Vars: map[string]interface{}{
      "environment":   "test",
      "vpc_cidr":      "10.0.0.0/16",
      "enable_nat":    false, // Skip NAT to save $$$
    },
  }

  // Defer cleanup — runs even if test fails
  defer terraform.Destroy(t, terraformOptions)

  // Apply
  terraform.InitAndApply(t, terraformOptions)

  // Assert: VPC exists and has correct CIDR
  vpcId := terraform.Output(t, terraformOptions, "vpc_id")
  actualCidr := aws.GetVpcCidrBlock(t, vpcId, "us-east-1")
  assert.Equal(t, "10.0.0.0/16", actualCidr)

  // Assert: Subnets exist (expected 3 private, 3 public)
  expectedSubnetCount := 6
  actualSubnetCount := aws.GetSubnetsForVpc(t, vpcId, "us-east-1")
  assert.Equal(t, expectedSubnetCount, len(actualSubnetCount))
}
```

**Trade-off:** Integration tests cost money (you're creating real resources). But for critical infrastructure — VPCs, databases, load balancers — the confidence gained is worth every penny.

Pro tip: **Use a dedicated AWS account** for testing. And always tag resources so you can clean up stragglers:

```bash
# Lambda that runs daily to nuke orphaned test resources
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment,Values=test \
  --query 'ResourceTagMappingList[].ResourceARN'
```

### Layer 4: E2E / Smoke Tests

Once your infrastructure is deployed to a staging environment, validate it actually works from the outside:

```bash
#!/bin/bash
# smoke_test.sh

set -euo pipefail

# Test the load balancer responds
LB_DNS=$(terraform output -raw lb_dns_name)

# HTTP health check
curl -f -s -o /dev/null -w "%{http_code}" "https://$LB_DNS/health"

# Database connectivity test
PGPASSWORD=$DB_PASSWORD psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT 1"

# Redis ping test
redis-cli -h $REDIS_ENDPOINT ping

# DNS resolution
dig +short app.mydomain.com | grep -q . || exit 1
```

These tests are simple but catch things integration tests miss: DNS propagation, TLS certificate issues, and real-world latency.

---

## Testing Ansible Playbooks

Ansible has its own testing ecosystem. The key insight: test your playbooks **without** running against real servers.

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: instance
    image: "geerlingguy/docker-ubuntu2204-ansible:latest"
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```

```bash
# Test locally with Molecule — no cloud resources needed
molecule create     # Spin up Docker containers
molecule converge   # Run your playbook
molecule verify     # Assert the state
molecule destroy    # Clean up
```

```yaml
# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  tasks:
    - name: Check Nginx is running
      ansible.builtin.systemd:
        name: nginx
        state: started
      register: nginx_status
      failed_when: nginx_status.status.ActiveState != "active"

    - name: Check port 80 is listening
      ansible.builtin.wait_for:
        port: 80
        state: started
        timeout: 10
```

---

## Testing Kubernetes Manifests

Kubernetes configs are notorious for "works on my cluster" syndrome. Some ways to test them:

```bash
# Dry-run against a real API server (validates schema)
kubectl apply --dry-run=server -f deployment.yaml

# Official validation
kubectl create --validate=true --dry-run=client -f deployment.yaml

# Custom resource validation with kubeconform
kubeconform -summary deployment.yaml
```

**Kyverno / OPA Gatekeeper policies** can also validate manifests before they hit the cluster:

```yaml
# kyverno/require-resource-limits.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: enforce
  rules:
    - name: check-resources
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "All containers must have resource limits set."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

---

## A Practical Testing Pipeline

Here's what a complete infrastructure CI/CD pipeline looks like:

```
PR Opened
    │
    ▼
Static Analysis ──► tflint, tfsec, checkov, terraform fmt
    │
    ▼
Policy Check ──► conftest / OPA
    │
    ▼
Plan ──► terraform plan (validate logic)
    │
    ▼
Unit Tests ──► Terratest (with mocks, no real infra)
    │
    ▼
Integration Tests ──► Terratest (real resources, test account)
    │
    ▼
Approval Gate
    │
    ▼
Apply ──► terraform apply (staging)
    │
    ▼
Smoke Tests ──► curl, psql, redis-cli checks
    │
    ▼
Promote to Production
```

Each step catches different failure modes earlier. Static analysis costs nothing and runs in seconds. Integration tests cost some money but catch real issues.

---

## What You Can Do Today

Not every team has a full testing pipeline. Here's where to start without getting overwhelmed:

1. **Add `tflint` and `checkov` to your CI** — Zero cost, catches real issues
2. **Add `terraform validate` to your PR workflow** — Catches syntax errors early
3. **Write one critical-rule OPA policy** — Start with "no public S3 buckets"
4. **Use `terraform plan` as a diff review tool** — Make plan output part of your PR review process
5. **When you break something, add a test** — Infrastructure testing is iterative. Don't try to cover everything at once

The goal isn't 100% test coverage on day one. It's making sure your next `terraform apply` doesn't accidentally open a port to the world.

Your infrastructure deploys itself. Your tests should make sure it deploys safely.
