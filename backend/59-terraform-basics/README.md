# Terraform Basics

You SSH into a server to apply a config change. Six months later, that server gets replaced by auto-scaling, and nobody remembers what manual changes you made. Now your new servers don't work the same way, and you're spending hours trying to reverse-engineer what you did.

This is the problem Infrastructure as Code solves — and Terraform is one of the best tools for it.

## What Terraform Actually Is

Terraform is a declarative tool for managing infrastructure. You tell it *what* you want (three EC2 instances, a load balancer in front of them, a database behind), and it figures out *how* to make that happen. No click-ops, no manual SSH, no undocumented snowflake servers.

The key difference from scripting:

```bash
# ❌ Imperative (shell script) — you specify every step
aws ec2 create-security-group --group-name web-sg --description "Web SG"
aws ec2 authorize-security-group-ingress --group-name web-sg --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 run-instances --image-id ami-123 --instance-type t3.medium --security-groups web-sg

# Problem: What happens when you run this twice? You get duplicate resources.
# What if someone manually deleted a security group? Your script breaks.
```

```hcl
# ✅ Declarative (Terraform) — you specify the desired state
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
}

resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t3.medium"
  vpc_security_group_ids = [aws_security_group.web.id]
}

# Terraform figures out the diff and only applies what's needed.
# Run it 10 times — same result. No duplicates. No surprises.
```

## The Three Core Concepts

### 1. State — The Ground Truth

Terraform maintains a **state file** that maps your config to real-world resources. When you run `terraform apply`, it:

1. Reads your `.tf` files (desired state)
2. Reads the state file (current state)
3. Computes the diff
4. Creates/updates/destroys only what changed

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  .tf files  │     │  State file  │     │  Cloud/AWS   │
│ (desired)   │────▶│  (current)   │────▶│  (reality)   │
└─────────────┘     └──────────────┘     └──────────────┘
                          │
                    [Plan the diff]
                          │
                    [Apply changes]
```

**Critical:** The state file is sensitive — it contains resource IDs, sometimes secrets. Never commit it to git directly. Use remote backends (S3, Terraform Cloud) with locking.

```hcl
# ❌ Local state — no locking, easy to lose
terraform {
  backend "local" {}
}

# ✅ Remote state with S3 + DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/network/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

### 2. Resources — What You Manage

Every piece of infrastructure is a resource. The pattern is always:

```hcl
resource "provider_type_resource_name" "local_label" {
  # configuration arguments
}
```

```hcl
# A simple EC2 instance
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"

  tags = {
    Name = "AppServer"
    Env  = "Production"
  }
}

# An S3 bucket
resource "aws_s3_bucket" "assets" {
  bucket = "my-app-static-assets-2024"
  acl    = "private"
}
```

Resources can reference each other. Terraform builds a dependency graph automatically.

```hcl
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Terraform knows web_sg must be created before this instance
resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.medium"
  vpc_security_group_ids = [aws_security_group.web.id]  # ← implicit dependency
}
```

### 3. Providers — Your Cloud Adapter

Terraform itself doesn't know anything about AWS, GCP, or Azure. Providers are plugins that translate Terraform's generic resource model into cloud-specific API calls.

```hcl
# The AWS provider — your bridge to Amazon
provider "aws" {
  region = "us-west-2"

  # Credentials come from env vars, AWS CLI config, or IAM roles
  # Never hardcode them in .tf files!
}
```

A single config can use multiple providers:

```hcl
provider "aws" {
  region = "us-east-1"
  alias  = "us_east"  # Multiple regions via provider aliases
}

provider "aws" {
  region = "eu-west-1"
  alias  = "eu_west"
}

provider "cloudflare" {
  # Different provider entirely
}

resource "aws_instance" "primary" {
  provider = aws.us_east
  # ...
}

resource "aws_instance" "backup" {
  provider = aws.eu_west
  # ...
}
```

## The Terraform Workflow

It's always the same four commands:

```
terraform init    → Download providers & modules
terraform plan    → Preview changes before applying
terraform apply   → Execute the plan
terraform destroy → Tear everything down
```

### Init — One Time Setup

```bash
terraform init

# Initializing the backend...
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 4.0"...
# - Installing hashicorp/aws v4.67.0...
# Terraform has been successfully initialized!
```

Run this whenever you add a new provider or module. It's the only command that needs network access.

### Plan — Review Before You Rekt

```bash
terraform plan -out=tfplan

# Terraform will perform the following actions:
#
#   # aws_instance.web will be created
#   + resource "aws_instance" "web" {
#       + ami                          = "ami-0c55b159cbfafe1f0"
#       + instance_type                = "t3.medium"
#       + vpc_security_group_ids       = [
#           + (known after apply),
#         ]
#     }
#
# Plan: 1 to add, 0 to change, 0 to destroy.
```

**Always plan before apply.** The plan shows you exactly what'll change. In CI/CD, you can fail the pipeline if the plan shows unexpected destruction.

### Apply — Make It Real

```bash
terraform apply tfplan

# aws_instance.web: Creating...
# aws_instance.web: Still creating... [10s elapsed]
# aws_instance.web: Creation complete after 12s [id=i-0a1b2c3d4e5f]
#
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### Destroy — Cleanup

```bash
terraform destroy

# Plan: 0 to add, 0 to change, 3 to destroy.
# 
# Do you really want to destroy all resources?
#   Enter a value: yes
#
# Destroy complete! Resources: 3 destroyed.
```

**Pro tip:** Use `terraform destroy` on staging environments at the end of the day. It saves cloud costs and forces you to keep things automated.

## Variables and Outputs — Making Configs Reusable

Hardcoding values in Terraform defeats the purpose. Use variables to make configs flexible.

```hcl
# variables.tf — define what can be customized
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  description = "Number of web instances"
  type        = number
  default     = 2
}

# terraform.tfvars — values for this environment
environment    = "staging"
instance_count = 2
```

```hcl
# outputs.tf — surface useful info after apply
output "web_url" {
  description = "Load balancer URL"
  value       = aws_lb.web.dns_name
}

output "web_sg_id" {
  description = "Security group ID"
  value       = aws_security_group.web.id
}
```

Running `terraform output` later gives you:

```bash
$ terraform output web_url
"my-app-staging-1234567890.us-west-2.elb.amazonaws.com"
```

## Modules — Don't Repeat Yourself

You'll end up creating the same patterns — VPC with subnets, web server with ALB, database with replica. Modules let you package and reuse those patterns.

```hcl
# Instead of copy-pasting configs, reference a module
module "web_app" {
  source = "github.com/myteam/tf-modules//web-app"  # or registry.terraform.io

  environment = var.environment
  instance_type = "t3.medium"
  min_size      = 2
  max_size      = 10
}

# The module handles the VPC, security groups, ASG, ALB, and DNS
# All you see is one clean block
```

The [Terraform Registry](https://registry.terraform.io/) has thousands of community modules. For production, pin versions:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"  # ← always pin

  name = "myapp-vpc"
  cidr = "10.0.0.0/16"
}
```

## Common Gotchas

### State Drift

Someone logs into the AWS console and deletes a security group. Terraform still thinks it exists. Next apply, it tries to delete resources that reference it.

**Fix:** Run `terraform plan` regularly (every CI run). Use `terraform refresh` to sync state, but prefer automated detection over manual refresh.

### Accidental Destroys

A misconfigured variable can nuke your production environment.

```hcl
# One wrong value and you destroy the production database
resource "aws_db_instance" "production" {
  skip_final_snapshot = true  # ← no backup before destroy
}
```

**Fix:** Use `prevent_destroy` for critical resources:

```hcl
resource "aws_db_instance" "production" {
  # ...
  lifecycle {
    prevent_destroy = true  # ← Terraform will refuse to destroy this
  }
}
```

### Secrets in State

```hcl
resource "aws_db_instance" "main" {
  password = "supersecret123!"  # ← This ends up in terraform.tfstate in plaintext
}
```

**Fix:** Use a secrets manager or environment variables. Never put secrets in config files.

## When Terraform Makes Sense vs When It Doesn't

| Good Fit | Bad Fit |
|----------|---------|
| Cloud infrastructure (AWS, GCP, Azure) | Application deployment (use CI/CD instead) |
| Network topology (VPC, subnets) | One-off experiments |
| Managed services (RDS, ElastiCache) | Rapidly changing configs |
| Multi-region deployments | Tiny personal projects |

## Key Takeaways

- **Declarative > imperative** for infrastructure. Tell Terraform *what*, not *how*.
- **Always use remote state** with locking. Local state is for tutorials only.
- **Plan before apply.** Every time. Even for "small" changes.
- **Version your Terraform configs** in git. Your `.tf` files are infrastructure documentation.
- **Pin provider and module versions.** `latest` is a time bomb.
- **Treat state as sensitive.** It contains resource IDs, config values, and sometimes secrets.
