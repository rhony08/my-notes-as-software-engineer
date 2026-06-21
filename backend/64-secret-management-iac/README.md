# Secret Management in IaC

We've all seen it—somewhere in your repo history, there's a `.env` file or a hardcoded password. Maybe you caught it before pushing. Maybe you didn't. The hard truth is: **secrets in your IaC repo are one accidental push away from a breach**.

Terraform state files, Ansible playbooks, and CloudFormation templates all need secrets at some point—database passwords, API keys, TLS certificates. The question isn't *if* you'll need them, but *how* you'll store them without leaking.

Let's look at what actually works in production.

---

## The Obvious (Wrong) Answer

I know, I know—you'd never hardcode secrets. But "I'll just use environment variables" isn't much better.

```hcl
# ❌ NEVER DO THIS
resource "aws_db_instance" "main" {
  username = "admin"
  password = "SuperSecretP@ssword123!"
}

# ❌ Also not great - env vars checked into CI pipelines
resource "aws_db_instance" "main" {
  username = var.db_user     # from a .env file committed "just for dev"
  password = var.db_password
}
```

Environment variables at build time get written into logs, CI artifacts, and state files. They're better than plaintext in code, but they still end up visible in too many places.

---

## The Problem: Secrets Leak Everywhere

Here's what actually happens when you pass secrets the naive way to Terraform:

```bash
# You run this in CI
TF_VAR_db_password="hunter2" terraform apply

# Terraform happily writes that password into the state file
# The state file? Probably in S3. Maybe encrypted. Maybe not.
# Anyone with read access to the state bucket now has your DB password.
```

```
# Ansible equivalent
ansible-playbook deploy.yml -e "db_password=hunter2"
# Shows up in process list, logs, and any debug output
```

Three leakage points:
1. **CLI history** — visible to anyone on the CI runner
2. **State files** — Terraform state contains all resolved values
3. **CI logs** — one `echo` or debug statement, and it's game over

---

## The Right Approaches

### 1. Backend-Native Secret Managers

The cloud providers all have secret stores. Use them as the source of truth.

```hcl
# ✅ Terraform + AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

No secrets in code. No secrets in variables. The reference is in IaC, but the actual value lives in the secret store with proper audit trails and rotation.

| Provider | Service | Rotation | Cost |
|----------|---------|----------|------|
| AWS | Secrets Manager | Built-in | ~$0.40/secret/month |
| AWS | Parameter Store (SecureString) | Manual | Free (with limitations) |
| GCP | Secret Manager | Built-in | $0.06/version/month |
| Azure | Key Vault | Built-in | ~$0.03/10k ops |

**Trade-off:** You now depend on that secret store being accessible during deployment. If your pipeline can't reach Secrets Manager (network outage, permissions issue), your deploy fails.

---

### 2. SOPS (Secrets OPerationS) + Git

Sometimes you want to version-control your secrets (encrypted) for disaster recovery.

```yaml
# ✅ secrets.enc.yaml - encrypted with SOPS
db_password: ENC[AES256_GCM,data:3H8BnQFZ0Q==,iv:...]
api_key: ENC[AES256_GCM,data:XYZ...,iv:...]
sops:
  kms:
    - arn: arn:aws:kms:us-east-1:xxx:key/yyy
```

```bash
# Decrypt at deployment time
sops decrypt secrets.enc.yaml > secrets.yaml
terraform apply -var-file=secrets.yaml
# Wipe after
rm secrets.yaml
```

SOPS supports KMS, age, PGP, and GCP KMS. It's the most popular open-source approach for encrypted-in-git secrets.

**Trade-off:** The decryption key must be available at deploy time (e.g., CI role with KMS access). And decrypted files sitting on disk are a short-lived but real risk.

---

### 3. Vault Dynamic Secrets

HashiCorp Vault takes it further—it generates short-lived credentials on demand.

```hcl
# ✅ Vault + Terraform provider
data "vault_generic_secret" "aws_creds" {
  path = "aws/creds/deploy-role"
}

provider "aws" {
  access_key = data.vault_generic_secret.aws_creds.data["access_key"]
  secret_key = data.vault_generic_secret.aws_creds.data["secret_key"]
}
```

Vault creates a temporary AWS credential that expires in (say) 1 hour. If it leaks, it's useless thirty minutes later.

**But** you now run Vault. That's a whole distributed system to operate, back up, and secure. Worth it at scale. Overkill for a side project.

| Approach | Setup Effort | Security Level | Best For |
|----------|-------------|---------------|----------|
| Cloud Secret Manager | Low | High | Most production setups |
| SOPS + Git | Medium | High | GitOps/cold-start scenarios |
| Vault | High | Very High | Large teams, compliance-heavy |
| Env Vars (CI secrets) | Low | Medium | Small projects, no state concern |

---

## Terraform State: The Elephant in the Room

Even if you do everything right with secret inputs, **Terraform state files contain all resolved values**. Your DB password ends up in the state JSON, plain as day.

```json
{
  "resources": [
    {
      "type": "aws_db_instance",
      "instances": [
        {
          "attributes": {
            "password": "hunter2"
          }
        }
      ]
    }
  ]
}
```

You **must** encrypt your state:

```hcl
# ✅ Terraform backend with encryption
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "alias/terraform-state-key"
    dynamodb_table = "terraform-locks"
  }
}
```

Without encrypted state, all your secret management was a waste of time.

---

## GitOps + Secrets: The Hard Mode

In pure GitOps (ArgoCD, Flux), everything comes from git. But secrets can't be plaintext in git. The solution is usually:

1. **Sealed Secrets** (Kubernetes) — encrypt Secret resources with a controller key
2. **External Secrets Operator** — sync from cloud secret managers into K8s Secrets
3. **Vault Agent Injector** — sidecar that pulls secrets at pod startup

```
┌─────────────┐     ┌──────────────┐     ┌────────────┐
│  Git Repo   │ ──→ │  Controller  │ ──→ │  Cluster   │
│ (encrypted) │     │ (has key)    │     │ (decrypts) │
└─────────────┘     └──────────────┘     └────────────┘
```

External Secrets Operator is my current favorite—Chef's Kiss pattern: define a `SecretStore` resource that references AWS Secrets Manager, then create `ExternalSecret` resources that sync automatically. No encrypted files, no key management dance, just "here's what I need" and the operator handles the rest.

---

## Actionable Takeaways

- **Never hardcode secrets in IaC code** — this includes default values, test fixtures, and examples in docs
- **Always encrypt Terraform state** — it stores all resolved values, including secrets
- **Use cloud secret managers** for most projects (AWS Secrets Manager, GCP Secret Manager)
- **Use SOPS** when you need encrypted-in-git patterns for GitOps
- **Use Vault** when you need dynamic, short-lived credentials at scale
- **Audit your state files** — run `terraform state pull | grep -i password` and have a strong coffee if anything shows up
- **Set up secret rotation** from day one — retrofitting is painful

And seriously—check your git history for that accidentally committed `.env` file. We've all got one. Fix it before someone finds yours.
